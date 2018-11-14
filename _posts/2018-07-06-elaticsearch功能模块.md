elaticsearch功能模块
=====

#### 源码分析

版本：2.4.4

启动类：org.elasticsearch.bootstrap.Elasticsearch，该类进行参数校验，环境变量设置等，最后调用start函数

```java
private void start() {
    node.start();
    keepAliveThread.start();
}
```

1.node.start()启动节点，内部通过反射机制对索引，快照，路由，查询，Gateway，cluster等各个service进行启动。

2.keepAliveThread.start()启动一个线程，Bootstrap类构造函数中创建线程，run()方法中keepAliveLatch为CountDownLatch(同步工具类),keepAliveThread可以保证节点运行期间Bootstrap会一直存在，接受shutdown后才会关闭。

```java
/** creates a new instance */
Bootstrap() {
    keepAliveThread = new Thread(new Runnable() {
        @Override
        public void run() {
            try {
                keepAliveLatch.await();
            } catch (InterruptedException e) {
                // bail out
            }
        }
    }, "elasticsearch[keepAlive/" + Version.CURRENT + "]");
    keepAliveThread.setDaemon(false);
    // keep this thread alive (non daemon thread) until we shutdown
    Runtime.getRuntime().addShutdownHook(new Thread() {
        @Override
        public void run() {
            keepAliveLatch.countDown();
        }
    });
}
```



Bulk操作相关源码

在start()之前的setup()中会添加Shutdown的钩子函数，配置环境参数等，主要会创建一个Node实例，Node的构造函数中会加载各个module,其中就包含bulkAction的module（ActionModule）

```java
modules.add(new ActionModule(false));
modules.add(new MonitorModule(settings));
modules.add(new GatewayModule(settings));
modules.add(new NodeClientModule());
```



```java
INSTANCE.setup(true, settings, environment);

INSTANCE.start();
```

类处理路径：RestBulkAction->TransportBulkAction->TransportShardBulkAction

入口为RestBulkAction对象，一个请求会构建一个BulkRequest对象，

![image-20181107150435627](/var/folders/9j/3sfy__g120x1lx36zhx_n03m0000gp/T/abnerworks.Typora/image-20181107150435627.png)

BulkRequest.add方法会解析文本，构建对应的对象。接着会通过NodeClient将请求发送到TransportBulkAction类

####  Rest 模块分析

请求会被转发到RestController ,RestController类似一个微型的Controller层框架，可以根据注册关系执行请求对应的controller功能。

以Rest*Action命名的类都是提供http服务的，会在RestActionModule中被初始化，

UnAssigned 原因：

1）INDEX_CREATED：由于创建索引的API导致未分配。 

2）CLUSTER_RECOVERED ：由于完全集群恢复导致未分配。

 3）INDEX_REOPENED ：由于打开open或关闭close一个索引导致未分配。

 4）DANGLING_INDEX_IMPORTED ：由于导入dangling索引的结果导致未分配。 

5）NEW_INDEX_RESTORED ：由于恢复到新索引导致未分配。 

6）EXISTING_INDEX_RESTORED ：由于恢复到已关闭的索引导致未分配。 

7）REPLICA_ADDED：由于显式添加副本分片导致未分配。

 8）ALLOCATION_FAILED ：由于分片分配失败导致未分配。

 9）NODE_LEFT ：由于承载该分片的节点离开集群导致未分配。 

10）REINITIALIZED ：由于当分片从开始移动到初始化时导致未分配（例如，使用影子shadow副本分片）。

11）REROUTE_CANCELLED ：作为显式取消重新路由命令的结果取消分配。 

12）REALLOCATED_REPLICA ：确定更好的副本位置被标定使用，导致现有的副本分配被取消，出现未分配。

discovery自动发现模块
-----
该模块用于发现集群中新加入的节点，以及master的自动选举。

zen为默认的发现机制，形式为单播，当master节点down掉后，集群节点会互相ping选举出新节点，互相ping可以防止一个节点与master节点网络不同而误认为master节点有问题，因为它可以从其他节点获取master信息。

当集群不能产生一个master的时候，有两种配置配置策略：

1.所有节点的读写操作全部拒绝。

2.写操作拒绝，读操作正常。

ElectMasterService类：

```java
public ElectMasterService.MasterCandidate electMaster(Collection<ElectMasterService.MasterCandidate> candidates) {
    assert this.hasEnoughCandidates(candidates);

    List<ElectMasterService.MasterCandidate> sortedCandidates = new ArrayList(candidates);
    sortedCandidates.sort(ElectMasterService.MasterCandidate::compare);
    return (ElectMasterService.MasterCandidate)sortedCandidates.get(0);
}
```

没有master节点，则将ping到的所有节点进行排序，选取排序后的第一个投票其为master。

```java
public DiscoveryNode tieBreakActiveMasters(Collection<DiscoveryNode> activeMasters) {
    return (DiscoveryNode)activeMasters.stream().min(ElectMasterService::compareNodes).get();
}
```

如果ping到有master节点，选取其中节点id最小的投票。

#### 针对分布式

Luence是单机的，支持分布式需要自己实现。es通过增加一个系统字段_routing来支持分布式，通过该字段将Doc分布到不的shard中。

#### ES对Luence不足的解决方案

luence的不足：

1.Lucene是一个单机的搜索库，如何能以分布式形式支持海量数据。

解决方案：es增加了一个系统字段_routing，通过 _routing将Doc分发到不同的Shard，不同的Shard可以位于不同的机器上。

2.Lucene中没有更新，每次都是Append一个新文档，如何做部分字段的更新。

解决方案：es增加了系统字段_source，部分更新时

3.Lucene中没有主键索引，如何处理同一个Doc的多次写入。

解决方案：es增加一个系统字段_id来实现逐渐，每次写入的时候都会先查询id，如果有，则说明已经有相同Doc存在了。

4.在稀疏列数据中，如何判断某些文档是否存在特定字段。

解决方案：_field_names 字段会索引某个Field的名称，用来判断某个Doc中是否存在某个Field。

5.Lucen中生成完整Segment后，该Segment就不能再被更改，此时该Segment才能被搜索，这种情况下，如何做实时搜索。

elasticseach中通过分区实现分部式，数据写入时根据_routing规则将数据写入某一个shard中

查询近实时原因主要是由于内存中的index数据需要一段时间才会刷新为segment

elasticsearch中的查询主要分为两类，get请求（通过id查询特定Doc） search请求（通过Query查询匹配Doc），对于Search请求，查询的时候查询内存和磁盘上的segment，最后将结果合并返回。对于Get请求，查询的时候是先查询内存中的TransLog，如果找到就立即返回，若果没有找到再查询磁盘上的TransLog，如果还没有则再去查询磁盘上的segment

elasticsearch 查询流程：

Client Node：

1.判断是否需要跨集群访问，如果需要，则获取到要访问的Shard列表

2.获取当前cluster中要访问的shard，和上一步中的shard列表合并，构建出最终要访问的完整shard列表

3.遍历每个shard，对每个shard执行后面的逻辑

4.将查询阶段请求发送给相应的shard（会执行查询步骤，Query Phase）

5.异步等待返回结果，然后对结果合并，合并策略是维护一个Top N大小的优先级队列，每当收到一个shard的返回，就把结果放入优先级队列做一次排序，直到所有的shard都返回。翻页逻辑也在这一步进行，有两种策略

6.选出Top N个Doc ID后发送给这些Doc ID所在的Shard执行Fetch Phase，最后返回Top N的内容。(执行fetch phase)



Query Phase

1.创建Search Context，之后Search过程中的所有中间状态都会存在Context中，这些状态总共有50多个。

2.解析query的source，将结果存入search context，这里会根据请求中query类型的不同创建不同的Query对象。

3.判断请求是否允许被Cache，如果允许，则检查Cache中是否已经有结果，有则直接读，没有则继续执行，并将结果加入Cache。

4.添加Collectors(收集查询结果，实现排序)。

5.调用lucene的search接口，执行真正的搜索逻辑。

6.根据request中是否包含rescore配置决定是否进行二阶段排序。

7.有推荐请求，则执行推荐请求。

8.执行聚合统计请求。

Fetch Phase

该阶段根据DocID取完整Doc内容。该阶段是由数据分布导致的，因为查询时大多是根据非主字段来查询，而shard是根据主字段来分的（默认为_id）,所以需要遍历查找每个shard，为了降低cpu和io，索引会在选出Top N后进行fetch phase。



### Lucene  源码

倒排索引的实现，主要为Term Dictionary(后缀tim)和Term Index(后缀tip)文件,FST(finite state transducers) 是一个带输出的有限状态机（现理解为字典树结构）




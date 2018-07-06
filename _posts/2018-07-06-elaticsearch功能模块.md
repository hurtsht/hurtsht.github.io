elaticsearch功能模块
=====
discovery自动发现模块
-----
该模块用于发现集群中新加入的节点，以及master的自动选举。

zen为默认的发现机制，形式为单播，当master节点down掉后，集群节点会互相ping选举出新节点，互相ping可以防止一个节点与master节点网络不同而误认为master节点有问题，因为它可以从其他节点获取master信息。

当集群不能产生一个master的时候，有两种配置配置策略：

1.所有节点的读写操作全部拒绝。

2.写操作拒绝，读操作正常。

ElectMasterService类：

```
public ElectMasterService.MasterCandidate electMaster(Collection<ElectMasterService.MasterCandidate> candidates) {
    assert this.hasEnoughCandidates(candidates);

    List<ElectMasterService.MasterCandidate> sortedCandidates = new ArrayList(candidates);
    sortedCandidates.sort(ElectMasterService.MasterCandidate::compare);
    return (ElectMasterService.MasterCandidate)sortedCandidates.get(0);
}
```

没有master节点，则将ping到的所有节点进行排序，选取排序后的第一个投票其为master。

```
public DiscoveryNode tieBreakActiveMasters(Collection<DiscoveryNode> activeMasters) {
    return (DiscoveryNode)activeMasters.stream().min(ElectMasterService::compareNodes).get();
}
```

如果ping到有master节点，选取其中节点id最小的投票。
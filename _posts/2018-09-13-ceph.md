Ceph分布式存储，对象存储，块设备存储和文件系统服务

核心组件：Ceph OSD，Ceph Monitor和Ceph MDS

osd功能是存储数据，复制数据，平衡数据，恢复数据等

monitor负责监视集群，维护cluster map

mds保存文件系统服务的元数据。但对象存储和块存储设备是不需要使用该服务的。

ceph基础架构：

最底层为rados，rados是ceph的核心，本身是一个完整的分布式对象存储系统，主要有osd和monitor组成。rados的上一层为librados，librados是一个库，允许应用程序通过访问该库来与rados系统进行交互。基于librados层有radossgw，rbd，ceph fs。

radosgw是一套基于当前restful协议的网关。rdb提供一个分布式块设备，ceph fs提供一个兼容posix的文件系统。

Ceph数据分布算法：Crush 伪随机的数据分布，复制算法






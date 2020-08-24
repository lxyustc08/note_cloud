- [Ceph Basic Concept](#ceph-basic-concept)

# Ceph Basic Concept

一个Ceph存储集群至少需要1个`Ceph monitors`，1个`Ceph Manager`，以及1个`Ceph OSD` (Object Storage Daemon)，对于Ceph FS而言，`Ceph Metadata Server`同样需要一个。

![Alt Text](../ceph_pictures/ceph基础组件.png)

+ **Monitors:** 维护集群状态的映射(The maps of the cluster state)，包括`monitor map`, `manager map`, `OSD map`, `MDS map`以及`CRUSH map` 。这些映射对于Ceph daemons而言是集群的重要状态信息。Monitors同样负责管理Daemons与Clients之间的认证。为保证集群的高可靠，至少需要部署3个Monitors
+ **Managers:** 负责跟踪当前集群状态以及运行时状态指标，包括存储利用率，当前性能指标以及系统负载。同时manager daemons运行一个基于Python的模块，负责管理及导出Ceph集群信息，包括基于web的`Ceph Dashboard`以及RESTful 风格的API，从高可靠性角度出发，至少需要部署2个`Managers`
+ **Ceph OSDs:** 每个Ceph OSD负责存储数据，处理数据副本，恢复，平衡以及通过检测其他Ceph OSD的心跳来向`Monitors`与`Managers`提供监控信息。至少需要提供3个Ceph OSDs来实现数据冗余与高可用
+ **MDSs:** Ceph Metadata Server存储Ceph File System的元数据。Ceph Metadata Servers支持POSIX标准的文件系统用户执行基本的文件操作，如ls, find等

Ceph存储数据时，将数据视为`logical storage pool`中的对象。通过使用CRUSH算法，Ceph计算哪个`Placement Group`（`PG`）来存储盖对象，然后进一步计算哪个Ceph OSD Daemon存储该`Placement Group`（`PG`）。

CRUSH算法允许Ceph Storage Cluster动态伸缩，动态再平衡以及动态恢复

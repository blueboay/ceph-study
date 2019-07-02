# Ceph存储（一）介绍
---
### Ceph存储简介
Ceph是一个可靠、自动均衡、自动恢复的分布式存储系统，通常可用于对象存储，块设备存储和文件系统存储。
Ceph在存储的时候充分利用存储节点的计算能力，在存储每一个数据时都会通过计算得出该数据的位置，尽量的分布均衡。
### Ceph存储组件
Ceph核心组件包括：
  * OSD
  * Monitor
  * MDS

OSD：英文全称为Object Storage Device，主要功能用于数据的存储，当直接使用硬盘作为存储目标的时候，一块硬盘称之为OSD。当使用一个目录作为存储目标的时候，这个目录也称之为OSD。
Monitor：负责监视整个Ceph集群运行Map图，维护Ceph集群的状态。还包括了集群中客户端的认证与授权。
MDS：英文全称为Metadata Server，主要文件系统服务的元数据，对象存储和块设备存储不需要元数据服务。如果集群中使用CephFS接口，那么至少集群中至少需要部署一个MDS服务。
在新版本的Ceph还有其它组件，如：Manager。此组件在L版（Luminous）和之后的版本支持，是一个Ceph守护进程，用于跟踪运行时指标和集群的当前状态，如存储利用率，系统负载。通过基于Python插件来管理和公开这些信息，可基于Web UI和Rest API。如果要对此服务进行高可用至少需要两个节点。 
### Rados存储集群
RADOS为一个Ceph名词，通常指ceph存储集群，英文全名Reliable Automatic Distributed Object Store。即可靠的、自动化的、分布式对象存储系统。 
### Librados介绍
Librados是RADOS存储集群的API，支持常见的编程语言，如：C、C++、Java、Python、Ruby和PHP等。
通常客户端在操作RADOS的时候需要通过与Librados API接口交互，支持下面几种客户端：
  * RBD
  * CephFS
  * RadosGW

RBD：Rados Block Devices此客户端接口基于Librados API开发，通常用于块设备存储，如虚拟机硬盘。支持快照功能。
RadosGW：此客户端接口同样基于Librados API开发，是一个基于HTTP Restful风格的接口。
CephFS：此客户端原生的支持，通常文件系统存储的操作使用CephFS客户端。如：NFS挂载。 

[![](http://121.43.168.35/wp-content/uploads/2019/05/1.png)](https://www.linux-note.cn/wp-content/uploads/2019/05/1.png)
### 管理节点
Ceph常用管理接口通常都是命令行工具，如rados、ceph、rbd等命令，另外Ceph还有可以有一个专用的管理节点，在此节点上面部署专用的管理工具来实现近乎集群的一些管理工作，如集群部署，集群组件管理等。
### Pool与PG
Pool是一个存储对象的逻辑分区，它通常规定了数据冗余的类型与副本数，默认为3副本。对于不同类型的存储，需要单独的Pool，如RBD。每个Pool内包含很多个PG。
PG是归置组，英文名为Placement Group，它是一个对象的集合，服务端数据均衡和恢复的最小单位就是PG。
### FileStore与BlueStore
FileStore是老版本默认使用的后端存储引擎，如果使用FileStore，建议使用xfs文件系统。
BlueStore是一个新的后端存储引擎，可以直接管理裸硬盘，抛弃了ext4与xfs等本地文件系统。可直接对物理硬盘进行操作，同时效率也高出很多。 
### Object对象
在Ceph集群中，一条数据、一个配置都为一个对象。这些都是存储在Ceph集群，以对象进行存储。每个对象应包含ID、Binary Data和Metadata。 

[![](http://121.43.168.35/wp-content/uploads/2019/05/2.png)](https://www.linux-note.cn/wp-content/uploads/2019/05/2.png)

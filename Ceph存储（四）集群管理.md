# Ceph存储（四）集群管理
---
## 状态获取
关于集群其它的一些操作参考《[Ceph存储](https://www.linux-note.cn/?cat=48)》系列其它相关文章。
### 整体运行状态
执行ceph -s命令可以查看ceph集群整体运行状态，如。

<pre>
[cephadmin@ceph-monitor-1 ~]$ ceph -s
  cluster:
    id: 7e1c1695-1b01-4151-beea-70c008cffd8c
    health: HEALTH_WARN
    2 pools have pg_num > pgp_num
 
  services:
    mon: 3 daemons, quorum ceph-monitor-1,ceph-monitor-2,ceph-monitor-3
    mgr: ceph-monitor-2(active), standbys: ceph-monitor-1
    mds: cephfs-1/1/1 up  {0=ceph-monitor-1=up:active}
    osd: 7 osds: 7 up, 7 in
    rgw: 1 daemon active
 
  data:
    pools:   8 pools, 224 pgs
    objects: 246  objects, 4.1 KiB
    usage:   16 GiB used, 304 GiB / 320 GiB avail
    pgs: 224 active+clean
</pre>
  1. id：集群ID
  2. health：集群运行状态，这里有一个警告，说明是有问题，意思是pg数大于pgp数，通常此数值相等。
  3. mon：Monitors运行状态。
  4. osd：OSDs运行状态。
  5. mgr：Managers运行状态。
  6. mds：Metadatas运行状态。
  7. pools：存储池与PGs的数量。
  8. objects：存储对象的数量。
  9. usage：存储的理论用量。 
  10. pgs：PGs的运行状态
### PG状态
查看pg状态查看通常使用下面两个命令即可，dump可以查看更详细信息，如。

<pre>
~]$ ceph pg dump
~]$ ceph pg stat
</pre>
### Pool状态
查看整个pool状态。

<pre>
~]$ ceph osd pool stats
</pre>
查看指定pool状态。

<pre>
~]$ ceph osd pool stats <pool name>
</pre>
### OSD状态
以下3条命令查看，dump显示出更详细信息，tree可以让信息显示得更漂亮。

<pre>
~]$ ceph osd stat
~]$ ceph osd dump
~]$ ceph osd tree
</pre>
### Monitor状态
基本状态查看，使用dump可以查看更详细信息。

<pre>
~]$ ceph mon stat
~]$ ceph mon dump
</pre>
查看仲裁状态。

<pre>
~]$ ceph quorum_status
</pre>
### 集群空间用量
主要用到df子命令，后面跟detail可以显示更详细信息。 

<pre>
~]$ ceph df
~]$ ceph df detail
</pre>
## ceph.conf配置文件
该配置文件采用init文件语法，#和;为注释，ceph集群在启动的时候会按照顺序加载所有的conf配置文件。
配置文件分为以下几大块配置。
  * global：全局配置。
  * osd：osd专用配置，可以使用osd.N，来表示某一个OSD专用配置，N为osd的编号，如0、2、1等。
  * mon：mon专用配置，也可以使用mon.A来为某一个monitor节点做专用配置，其中A为该节点的名称，ceph-monitor-2、ceph-monitor-1等。使用命令 ceph mon dump可以获取节点的名称。
  * client：客户端专用配置。

配置文件可以从多个地方进行顺序加载，如果冲突将使用最新加载的配置，其加载顺序为。
  1. $CEPH_CONF环境变量
  2. -c 指定的位置
  3. /etc/ceph/ceph.conf
  4. ~/.ceph/ceph.conf
  5. ./ceph.conf 

配置文件还可以使用一些元变量应用到配置文件，如。
  * $cluster：当前集群名。
  * $type：当前服务类型。
  * $id：进程的标识符。
  * $host：守护进程所在的主机名。
  * $name：值为$type.$id。 
## 套接字文件
所谓套接字相当于守护进程的socket文件，默认存储于/var/run/ceph/目录下面，后缀名为.asok。可以使用此套接字完成对此守护进程所对应用的配置。但只能在本地执行，无法远程执行，套接字通常用于查询类。 
命令使用格式如。

<pre>
~]$ sudo ceph --admin-daemon /path/to/file <COMMAND>
</pre>
获取指定套接字可使用的命令帮助。

<pre>
~]$ sudo ceph --admin-daemon /var/run/ceph/ceph-mgr.ceph-monitor-1.asok help
</pre>
使用可使用的命令，如status查询状态。

<pre>
~]$ sudo ceph --admin-daemon /var/run/ceph/ceph-mgr.ceph-monitor-1.asok status
</pre>
## 服务平滑重启
有时候需要更改服务的配置，但不想重启服务，或者是临时修改。这时候就可以使用tell和daemon子命令来完成此需求。
### tell子命令
命令使用格式如下。

<pre>
~]$ ceph tell {daemon-type}.{daemon id or *} injectargs --{name}={value} [--{name}={value}]
</pre>
  * daemon-type：为要操作的对象类型如osd、mon等。
  * daemon id：该对象的名称，osd通常为0、1等，mon为ceph -s显示的名称，这里可以输入*表示全部。
  * injectargs：表示参数注入，后面必须跟一个参数，也可以跟多个。

例。

<pre>
~]$ ceph tell mon.ceph-monitor-1 injectargs --mon_allow_pool_delete=true
</pre>
mon_allow_pool_delete此选项的值默认为false，表示不允许删除pool，只有此选项打开后方可删除，记得改回去！！！
这里使用mon.ceph-monitor-1表示只对ceph-monitor-1设置，可以使用*
### daemon子命令
命令格式如下。

<pre>
~]$ ceph daemon {daemon-type}.{id} config set {name}={value}
</pre>
例。

<pre>
~]$ sudo ceph daemon mon.ceph-monitor-1 config set mon_allow_pool_delete false
</pre>

# Ceph存储（七）CephFS详解
---
## CephFS简介
CephFS作为Ceph集群最早原生支持的客户端，但成熟的最晚，而最成熟的还是RBD。要想在集中中可以使用CephFS客户端，需要创建至少一个metadata Pool，该Pool用来管理元数据信息，并向客户端输出 一个倒置树状的层级结构，这里面存放了真实数据的对应关系，相当于一个索引。而还需要创建至少1个Data Pool用来存储真正的数据。
## CephFS客户端
在开始使用CephFS之前需要创建CephFS接口，参考《[集群部署之创建CephFS](https://www.linux-note.cn/?p=85#cephfs)》。 
Ceph支持两种类型的客户端，一种是基于内核模块ceph.ko，这种需要客户端安装ceph-common程序包，并在/etc/ceph/目录下有ceph.conf集群配置文件和用于认证的密钥文件。
另外一种客户端为FUSE客户端，当有些操作系统无法使用ceph.ko内核的时候就可以使用此客户端连接。要求需要安装ceph-fuse程序软件，该客户端也需要从从/etc/ceph目录加载ceph集群配置文件和keyring文件进行连接。 
关于如何授权这里也不做过多介绍，参考《[认证与授权](https://www.linux-note.cn/?p=179)》，本案例所使用授权如下。

<pre>
[client.cephfs]
	key = AQCoKNVcATH5FxAAQ3m90iGjtLvsTtiKEFGl7g==
	caps mds = "allow *"
	caps mon = "allow r"
	caps osd = "allow * pool=cehpfs-metadata, allow * pool=cehpfs-data"
</pre>
授权命令如下。

<pre>
~]$ ceph auth add client.cephfs mds "allow *" mon "allow r" osd "allow * pool=cehpfs-metadata, allow * pool=cehpfs-data"
</pre>
需要指定对源数据存储池和实际数据存储池都有权限。
### 内核客户端
安装ceph-common软件包，复制集群配置文件和用户keyring文件至目标节点/etc/ceph目录。
#### 命令行挂载

<pre>
~]# mount -t ceph 192.168.6.126:6789,192.168.6.127:6789,192.168.6.128:6789:/ /mnt -o name=cephfs,secretfile=/etc/ceph/cephfs.key
</pre>
  * -t：指定使用ceph协议，接着后面指定mon节点的访问IP与端口，可以指定多个，使用逗号分隔，默认端口为6789。最后使用  / 代表要挂载根目录，挂载到本地的/mnt目录。
  * -o：指定挂载的选项，name为用户名，这里的用户为用户ID，不可使用用户标识。secretfile指定该用户的Key，可以指定Key文件路径，也可以直接指定Key字符串。

注意：cephfs.key文件内只能包含用户的Key。
挂载成功后在客户端节点使用下面命令可以查看挂载的信息。
~]# stat -f /mnt
#### 通过fstab客户端挂载
在/etc/fstab文件内写下如面格式的内容即可。

<pre>
192.168.6.126:6789,192.168.6.127:6789,192.168.6.128:6789:/	/mnt	ceph	name=cephfs,secretfile=/etc/ceph/cephfs.key,_netdev,noatime 0 0
</pre>
#### 卸载

<pre>
~]# umount /mnt
</pre>
### FUSE客户端
安装ceph-fuse软件包，复制集群配置文件和用户keyring文件至目标节点/etc/ceph目录。 
#### 命令行挂载

<pre>
~]# ceph-fuse -n client.cephfs -m 192.168.6.126:6789,192.168.6.127:6789,192.168.6.128:6789 /mnt/
</pre>
  * -n：指定用户标识。
  * -m：指定monitor节点信息，之后再指定挂载的目标 。
#### 通过fatab文件挂载
在/etc/fstab文件内写下如面格式的内容即可。 

<pre>
none/mntfuse.ceph   ceph.id=cephfs,ceph.conf=/etc/ceph/ceph.conf,_netdev,noatime 0 0
</pre>
ceph.id需要为用户的ID，而不是用户标识。
#### 卸载

<pre>
~]# fusermount /mnt/
</pre>
## Rank
在MDS集群中每一个MDS进程由一个Rank进行管理，Rank数量由max_mds参数配置，默认为1。每个Rank都有一个编号。编号从0开始。
rank有三种状态：
  * up：代表 Rank已经由某个MDS守护进程接管。
  * failed：代表未被接管。
  * damaged：代表损坏，元数据丢失或崩溃，可以使用命令ceph mds repaired修复，在未被修复之前Rank不会被分配给任何守护进程。

如果要对MDS进程做高可用，就可以启动多个MDS，然后设置多个Rank，这时候每个MDS就会关联至对应的Rank来实现高用。通常MDS的数量为Rank数量的两倍，这样可以保证任何一个Rank出现问题（Rank出现问题也就相当于MDS出现问题）有另外的MDS进程马上进行替换。
### 设置Rank数量

<pre>
~]$ ceph fs set cephfs max_mds 2
</pre>
### 减少Rank数量
减少Rank数量就也使用max_mds参数设置，设置的数量小于当前数量就相当于减少。如果Rank数减少需要手动关闭被停止的Rank，可以使用命令ceph fs status查看减少后被停止的Rank，然后执行如下命令关闭被停止的Rank。

<pre>
~]$ ceph mds deactivate 1
</pre>
这里的 1为rank的编号。
### Rank状态查看

<pre>
[cephadmin@ceph-monitor-1 ~]$ ceph mds stat
cephfs-2/2/2 up  {0=ceph-monitor-1=up:active,1=ceph-monitor-2=up:active}, 1 up:standby
</pre>
  * cephfs-2/2/2 up：cephfs是CephFS的名称，后面3个分别是已经分配的 Rank数、正常up的Rank数和设置的最大Rank数。
  * 0=ceph-monitor-1=up:active：代表第0个rank关联到了ceph-monitor-1节点，并且状态为up。
  * 1 up:standby：代表有1个备份。 
### 高可用配置
假设启动4个MDS进程，设置2个Rank。这时候有2个MDS进程会分配给两个Rank，还剩下2个MDS进程分别作为另外个的备份。
1.  增加MDS实例数量与Rank数量。
2.  增加完成后再查看CephFS状态如下。

[![](http://121.43.168.35/wp-content/uploads/2019/05/1-4-1024x479.png)](https://www.linux-note.cn/wp-content/uploads/2019/05/1-4.png)
通过上图可以看出有两个Rank，分别关联到了ceph-monitor-1和ceph-monitor-2的MDS，并且状态为active。在最后面有两个备份的MDS，分别为ceph-monitor-3和ceph-storage-2。 
3.  设置每个Rank的备份MDS，也就是如果此Rank当前的MDS出现问题马上切换到另个MDS。设置备份的方法有很多，常用选项如下。
  * mds_standby_replay：值为true或false，true表示开启replay模式，这种模式下主MDS内的数量将实时与从MDS同步。如果主宕机，从可以快速的切换。如果为false只有宕机的时候才去同步数据，这样会有一段时间的中断。
  * mds_standby_for_name：设置当前MDS进程只用于备份于指定名称的MDS。
  * mds_standby_for_rank：设置当前MDS进程只用于备份于哪个Rank，通常为Rank编号。另外在存在之个CephFS文件系统中，还可以使用mds_standby_for_fscid参数来为指定不同的文件系统。
  * mds_standby_for_fscid：指定CephFS文件系统ID，需要联合mds_standby_for_rank生效，如果设置mds_standby_for_rank，那么就是用于指定文件系统的指定Rank，如果没有设置，就是指定文件系统的所有Rank。

上面的配置需要写在ceph.conf配置文件，按本实验目标，在管理节点上修改ceph.conf配置文件，配置如下。

<pre>
[mds.ceph-monitor-3]
mds_standby_for_name = mon.ceph-monitor-2
mds_standby_replay = true

[mds.ceph-storage-2]
mds_standby_for_name = mon.ceph-monitor-1
mds_standby_replay = true
</pre>
通过以上的配置mon.ceph-monitor-3就作为了mon.ceph-monitor-2的备份，并且实时的同步数据。而mon.ceph-storage-2为mon.ceph-monitor-1的备份，也是实时的同步数据。如果想让某一个MDS可以作用于多个MDS的备份，可配置如下。

<pre>
[mds.ceph-storage-2]
mds_standby_for_fscid = 1
</pre>
指定CephFS文件系统的ID为1，如果不指定mds_standby_for_rank，代表备份于编号1的文件系统下面的所有MDS，此方法无法实际同步数据。
修完配置后需要同步到指定的节点，然后重启MDS进程所对应的服务。

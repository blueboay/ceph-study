# Ceph存储（二）集群部署
---
## 部署环境准备
以下为测试环境部署案例，如果是生产环境推荐osd所在服务器是物理机，根据情况配置不同的硬盘。其它的节点可以是虚拟机，但必须是高可用，如monitor和manager。管理节点administrator必须单独在一台机器，防止集群出现问题时无法进行管理。 
### 主机说明
<table class="wp-block-table" ><tbody ><tr >
<td >主机IP地址
</td>
<td >主机名
</td>
<td >部署角色 
</td></tr><tr >
<td >192.168.6.114/18.50.129.111
</td>
<td >ceph-admin.gogen.cn
</td>
<td >administrator
</td></tr><tr >
<td >192.168.6.115/18.50.129.112
</td>
<td >ceph-stroage-1.gogen.cn
</td>
<td >osd
</td></tr><tr >
<td >192.168.6.116/18.50.129.113 
</td>
<td >ceph-stroage-2.gogen.cn 
</td>
<td >osd
</td></tr><tr >
<td >192.168.6.117/18.50.129.114 
</td>
<td >ceph-stroage-3.gogen.cn 
</td>
<td >osd
</td></tr><tr >
<td >192.168.6.118/18.50.129.115 
</td>
<td >ceph-stroage-4.gogen.cn 
</td>
<td >osd
</td></tr><tr >
<td >192.168.6.126/18.50.129.116 
</td>
<td >ceph-monitor-1.gogen.cn 
</td>
<td >monitor、manager、mds
</td></tr><tr >
<td >192.168.6.127/18.50.129.117 
</td>
<td >ceph-monitor-2.gogen.cn 
</td>
<td >monitor、manager
</td></tr><tr >
<td >192.168.6.128/18.50.129.118 
</td>
<td >ceph-monitor-3.gogen.cn 
</td>
<td >monitor 
</td></tr></tbody></table>

### 硬盘说明
在ceph集群中。Administrators、Monitors、Managers和MDSs节点节点对服务器硬盘都没有要求，只要系统能正常运行即可。但OSD节点不一样，通常一个OSD就代表一块物理硬盘，作为分布式存储，OSD越多越好，这里有的4个OSD节点，分别再每节点添加两块硬盘，分别是40GB和 50GB，作为OSD的硬盘不需要做任何操作，需要OSD初始化的工具来回初始化。
查看ceph-storage-1节点硬盘信息如下。 

<pre>
[root@ceph-storage-1 ~]# fdisk -l /dev/sd*

Disk /dev/sda: 21.5 GB, 21474836480 bytes, 41943040 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x00024e19

   Device Boot  Start End  Blocks   Id  System
/dev/sda1   *2048 2099199 1048576   83  Linux
/dev/sda2 20992004194303919921920   8e  Linux LVM

Disk /dev/sda1: 1073 MB, 1073741824 bytes, 2097152 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sda2: 20.4 GB, 20400046080 bytes, 39843840 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sdb: 42.9 GB, 42949672960 bytes, 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sdc: 53.7 GB, 53687091200 bytes, 104857600 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
</pre>
### 网络说明
部署Ceph集群推荐所有节点都有两张网卡，一张用于外部通讯，承载业务数据流量，另外一张用于集群内部了数据交换。防止在进行内部数据交换的时候占用业务通道，影响业务通讯。
在此案例中192.168.6.0/24为业务使用，而 18.50.129.0/24为集群内部使用。 
注意：如果一台服务器有几个网卡都配置了IP地址与网关，那么将会存在几条默认路由。正常情况下我们只需要一条默认路由，而其它不需要的路由则删除，再根据情况添加静态路由。
### 配置时间同步
如果节点可以访问互联网，直接启动chronyd服务即可，CentOS  7自带此服务。

<pre>
~]# systemctl enable chronyd && systemctl start chronyd
</pre>
注意：如果是CentOS 6系统，默认不使用chronyd时间同步服务。 
一般推荐使用本地的时间服务器，修改/etc/chrony.conf配置文件，将server后配置修改为指定时间服务器地址即可，如果有多个，指定多个server即可，server后面跟时间服务器的地址，如。 

<pre>
server 0.centos.pool.ntp.org iburst
</pre>
可以使用以下命令可以查看时间同步的信息。 

<pre>
~]# chronyc sources -v
</pre>
### 配置主机名解析
本所本案例，在每个节点都配置此hosts，文件内容如下。 

<pre>
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1 localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.6.114 ceph-admin.gogen.cn ceph-admin
192.168.6.115 ceph-storage-1.gogen.cn ceph-storage-1
192.168.6.116 ceph-storage-2.gogen.cn ceph-storage-2
192.168.6.117 ceph-storage-3.gogen.cn ceph-storage-3
192.168.6.118 ceph-storage-4.gogen.cn ceph-storage-4
192.168.6.126 ceph-monitor-1.gogen.cn ceph-monitor-1
192.168.6.127 ceph-monitor-2.gogen.cn ceph-monitor-2
192.168.6.128 ceph-monitor-3.gogen.cn ceph-monitor-3
</pre>
### 关闭防火墙
为防止防火墙的干扰，推荐将防火墙关闭，操作如下。 

<pre>
~]# systemctl disable firewalld && systemctl stop firewalld
</pre>
### 关闭SELinux
同样为了防止SELinux干扰，默认系统是启用了SELinux，需要将此关闭。修改/etc/sysconfig/selinux文件来禁用SELinux，修改配置如下。 

<pre>
~]# sed -i "/^SELINUX/s@enforcing@disabled@" /etc/sysconfig/selinux
</pre>
修改完SELinux配置后需要重启系统。 
## 准备工作
这里主要是准备好administrator节点，和部署前所需要的一些配置准备配置。
### 配置yum源
使用Ceph官方仓库安装下载较慢，国内阿里云镜像站也提供Ceph相关镜像，地址如下。

<pre>
https://mirrors.aliyun.com/ceph/
</pre>
安装阿里云Ceph的repo源，在所有节点上安装此源。

<pre>
~]# rpm -ivh https://mirrors.aliyun.com/ceph/rpm-mimic/el7/noarch/ceph-release-1-1.el7.noarch.rpm
</pre>
### 准备普通用户
Ceph集群各节点推荐使用一个普通用户管理所有进程，也就是ceph-deploy在部署的时候需要以普通用户的身份登录到Ceph各节点，目标用户需要可以无密码使用sudo命令权限，以便在安装的过程中无需中断。并且在SSH的时候也需要配置为密钥登录，且无需输入密码。 
在所有节点上创建用户并设置密码。

<pre>
~]# useradd cephadmin && echo "cephadmin" | passwd cephadmin --stdin
</pre>
在administrator节点生成并分发密钥。 注意：此操作需要在cephadmin用户下进行。 

<pre>
~]$ ssh-keygen -t rsa -P ''
~]$ ssh-copy-id -i .ssh/id_rsa.pub cephadmin@ceph-monitor-1
~]$ ssh-copy-id -i .ssh/id_rsa.pub cephadmin@ceph-monitor-2
~]$ ssh-copy-id -i .ssh/id_rsa.pub cephadmin@ceph-monitor-3
~]$ ssh-copy-id -i .ssh/id_rsa.pub cephadmin@ceph-storage-1
~]$ ssh-copy-id -i .ssh/id_rsa.pub cephadmin@ceph-storage -2
~]$ ssh-copy-id -i .ssh/id_rsa.pub cephadmin@ceph-storage -3
~]$ ssh-copy-id -i .ssh/id_rsa.pub cephadmin@ceph-storage -4
</pre>
### 配置sudo权限
注意：此操作需要在root用户下执行。 
创建文件/etc/sudoers.d/cephadmin，文件内容如下。 

<pre>
~]# echo "cephadmin ALL = (root) NOPASSWD: ALL" | tee /etc/sudoers.d/cephadmin
</pre>
更改cephadmin文件的权限。 

<pre>
~]# chmod 0440 /etc/sudoers.d/cephadmin
</pre>
### 部署ceph-deploy
接下来将部署ceph-deploy节点，主要就是安装ceph-deploy软件包及其依赖包。此在administrator节点上面执行即可。

<pre>
~]# yum install ceph-deploy python-setuptools python2-subprocess32
</pre>
## 部署Rados集群
只要是ceph-deploy命令的操作都需要在administrator节点上的cephadmin用户上执行，并且必需在ceph-cluster目录进行操作。
ceph-deploy命令只能在administrator节点执行，而ceph命令可在集群任何节点执行，无论是在哪个节点执行都需要确保在administrator用户下。
### 初始化集群
1.  首先创建配置文件目录。 

<pre>
~]$ mkdir ceph-cluster
~]$ cd ceph-cluster
</pre>
2.  创建一个集群配置信息和密钥环境文件等，初化时需要指定集群中一个monitor节点。 

<pre>
~]$ ceph-deploy new --cluster-network 18.50.129.0/24 --public-network 192.168.6.0/24 ceph-monitor-1 
</pre>
ceph-deploy为部署集群的主要命令。
  * new：ceph-deploy子命令，ceph-deploy有很多子命令可以完成更多管理功能，可以使用ceph-deploy --help命令查看。其中new功能为创建一个新的集群，会在当前目录生成集群配置文件与keryring文件。
  * --cluster-network：new子命令的可用选项，可使用ceph-deploy new --help获取所有选项，其中--cluster-network为指定集群内部使用网络
  * --public-network：指定集群公用网络，也就是业务使用网络。
  * ceph-monitor-1：指定初始的集群的第一个monitor节点的主机名，网络必须可达。 

创建集群生成的文件如下。 

<pre>
[cephadmin@ceph-admin ceph-cluster]$ ls
ceph.conf  ceph-deploy-ceph.log  ceph.mon.keyring
</pre>
ceph.conf主要包含集群的基本配置信息。

<pre>
[cephadmin@ceph-admin ceph-cluster]$ cat ceph.conf 
[global]
fsid = 7e1c1695-1b01-4151-beea-70c008cffd8c
public_network = 192.168.6.0/24
cluster_network = 18.50.129.0/24
mon_initial_members = ceph-monitor-1
mon_host = 192.168.6.126
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
</pre>
3.  安装ceph集群
安装ceph集群需要使用ceph-deploy的install子命令，首先初始化所有monitor节点。

<pre>
~]$ ceph-deploy install ceph-monitor-1 ceph-monitor-2 ceph-monitor-3
</pre>
接下来初始化所有storage节点，执行下面的命令。

<pre>
~]$ ceph-deploy install ceph-storage-1 ceph-storage-2 ceph-storage-3 ceph-storage-4
</pre>
注意：初始化的操作主要就是SSH到各节点上面通过yum安装好需要的软件包，默认使用的是官方源。这个过程也可以手动登录到各节点自行安装，这样在初始化的时候速度将会快些。如果使用ceph-deploy执行的时候是一台一台执行，而手动执行可多台一起执行，并且可使用阿里源。手动安装方法如下。
在各节点上面执行安装阿里yum源，然后安装需要的软件包。

<pre>
~]$ rpm -ivh https://mirrors.aliyun.com/ceph/rpm-mimic/el7/noarch/ceph-release-1-1.el7.noarch.rpm
~]$ yum install ceph ceph-radosgw
</pre>
注意：安装手动完成软件包安装后还必须再执行ceph-deploy install来初始化，因为还有其它的初始化操作，但此时这个过程相对较快。在执行的时候可以使用选项--no-adjust-repos指定不修改本地yum源即可，否则还是会使用官方源。 
4.  配置第一个monitor节点。

<pre>
~]$ ceph-deploy mon create-initial
</pre>
以上命令执行成功后，在第2步初始化的那个monitor节点上会启动一个ceph-mon进程，并且以ceph用户运行。在/etc/ceph目录会生成一些对应的配置文件，其中ceph.conf文件就是从前面初始化生成的文件直接copy过去的，此文件可以直接进行修改。 
5.  配置第一个manager节点。
根据前面主机说明，我们将ceph-monitor-1和ceph-monitor-2节点也作为manager节点，所以我们将在ceph-monitor-1部署第一个manager节点 。执行如下命令配置。

<pre>
~]$ ceph-deploy mgr create ceph-monitor-1
</pre>
上面的命令运行成功后会在当前目录生成一些keryring密钥，并且在指定的集群节点上面会产生一个ceph-mgr进程，运行用户为ceph。 
6.  将配置文件和密钥复制到集群各节点。 
配置文件就是第2步操作生成的ceph.conf，而密钥是ceph.client.admin.keyring，当使用ceph客户端连接至ceph集群时需要使用的密默认密钥，这里我们所有节点都要复制，命令如下。 

<pre>
~]$ ceph-deploy admin ceph-monitor-1 ceph-monitor-2 ceph-monitor-3
~]$ ceph-deploy admin ceph-storage-1 ceph-storage-2 ceph-storage-3 ceph-storage-4
</pre>
7.  更改ceph.client.admin.keyring权限。 
默认情况下ceph.client.admin.keyring文件的权限为600，属主和属组为root，如果在集群内节点使用cephadmin用户直接直接ceph命令，将会提示无法找到/etc/ceph/ceph.client.admin.keyring文件，因为权限不足。当然如果使用sudo ceph不存在此问题，为方便直接使用ceph命令，可将权限设置为644。在集群节点上面cephadmin用户下执行下面命令。 

<pre>
~]$ sudo chmod 644 /etc/ceph/ceph.client.admin.keyring 
</pre>
8.  集群健康状态检查。 
 在集群节点上面使用ceph -s命令可看集群状态，如。

<pre>
[cephadmin@ceph-monitor-1 root]$ ceph -s
  cluster:
    id: 7e1c1695-1b01-4151-beea-70c008cffd8c
    health: HEALTH_OK
 
  services:
    mon: 1 daemons, quorum ceph-monitor-1
    mgr: ceph-monitor-1(active)
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0  objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:

</pre>
  * cluster：集群相关，其中health表示集群的健康状态，id为集群的ID。
  * services：集群内服务的状态，mon代表monitor，mgr为manager，osd就是指osd。如果有多个mon或mgr的时候这里也会体现出。
  * data：实在存储数据的状态，有多少个pool，多少个pg，有多少个对象，以即大小。包括使用了多少等。因为我们还没有添加osd，所以这里全是0。 
### 添加OSD
在Ceph集群组件里面OSD主要作用数据存储，每个OSD可以是一块物理硬盘也可以是一个分区目录，推荐使用的是物理硬盘。 
1.  列出storage节点上面可用的物理硬盘。 

<pre>
~]$ ceph-deploy disk list ceph-storage-1
</pre>
我们分别对每个storage节点添加了两块硬盘，分别为是40GB和50GB，正常情况下是可以看到添加的硬盘信息。

<pre>
[ceph-storage-1][INFO  ] Running command: sudo fdisk -l
[ceph-storage-1][INFO  ] Disk /dev/sda: 21.5 GB, 21474836480 bytes, 41943040 sectors
[ceph-storage-1][INFO  ] Disk /dev/sdb: 42.9 GB, 42949672960 bytes, 83886080 sectors
[ceph-storage-1][INFO  ] Disk /dev/sdc: 53.7 GB, 53687091200 bytes, 104857600 sectors
</pre>
2.  擦除指定storage节点，指定硬盘的数据。

<pre>
~]$ ceph-deploy disk zap ceph-storage-1 /dev/sdb
</pre>
3.  添加指定storage节点，指定硬盘为OSD，相当于创建一个OSD。

<pre>
~]$ ceph-deploy osd create ceph-storage-1 --data /dev/sdb
</pre>
添加成功后提示如下信息。

<pre>
[ceph-storage-1][INFO  ] Running command: sudo /bin/ceph --cluster=ceph osd stat --format=json
[ceph_deploy.osd][DEBUG ] Host ceph-storage-1 is now ready for osd use.
</pre>
指定的storage节点会运行一个ceph-osd的进程，启动用户为ceph，如果一台storage上面有多个osd，那么会有多个进程。
使用上面操作将4个storage节点共8块硬盘都添加为OSD。 
4.  查看指定OSD的信息。
首先可以通过ceph -s命令查看集群状态，也可以输出OSD的基本信息。

<pre>
[cephadmin@ceph-monitor-1 ~]$ ceph -s
  cluster:
    id: 7e1c1695-1b01-4151-beea-70c008cffd8c
    health: HEALTH_OK
 
  services:
    mon: 1 daemons, quorum ceph-monitor-1
    mgr: ceph-monitor-1(active)
    osd: 8 osds: 8 up, 8 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0  objects, 0 B
    usage:   8.0 GiB used, 352 GiB / 360 GiB avail
   pgs:

</pre>
从上面可以看出总共有8个OSD，并且8个已就绪，总空间为360GB，已使用8GB，每个OSD默认会使用1GB的空间。
还可以使用以下命令分别查看指定storage节点上面的OSD信息。

<pre>
~]$ ceph-deploy osd list ceph-storage-1
</pre>
还可以使用ceph命令获取OSD的信息，此命令需要在集群任何节点上面运行。

<pre>
[cephadmin@ceph-monitor-1 ~]$ ceph osd stat
8 osds: 8 up, 8 in; epoch: e33
[cephadmin@ceph-monitor-1 ~]$ ceph osd ls
0
1
2
3
4
5
6
7
[cephadmin@ceph-monitor-1 ~]$ ceph osd dump
</pre>
  * dump可以查看更详细信息。
  * ls列出所有OSD的编号。
  * stat查看基本状态
### 上传与下载
1.  首先创建一个用于数据存储的存储池。

<pre>
ceph osd pool create testpool 32 32
</pre>
创建了一个名为testpool的存储池，并指定有32个PG和32个用于归置的PG，第二个代表PGP（归置PG），如果不指定默认与PG相等，推荐相等。
2.  上传文件至指定存储池。

<pre>
~]$ rados put fstab /etc/fstab --pool=testpool
</pre>
rados命令用于rados集群管理，而put子命令用于上传文件，fstab为指定的对象名，而/etc/fstab为要上传的文件，最后使用--pool指定要上传到哪个存储池。 
3.  查看存储池中的所有对象。

<pre>
[cephadmin@ceph-monitor-1 ~]$ rados ls --pool=testpool
fstab
</pre>
查看指定对象更详细的信息。

<pre>
[cephadmin@ceph-monitor-1 ~]$ ceph osd map testpool fstab
osdmap e43 pool 'testpool' (1) object 'fstab' -> pg 1.45a53d91 (1.11) -> up ([5,2,1], p5) acting ([5,2,1], p5)
</pre>
4.  删除存储池中指定的对象。

<pre>
~]$ rados rm fstab --pool=testpool
</pre>
5.  删除存储池。
一般情况下不推荐删除存储池，Ceph集群默认是禁用此操作，需要管理员在ceph.conf配置文件中启用支持删除存储池的操作后才可能删除，默认执行删除命令会报错。删除命令如下： 

<pre>
[cephadmin@ceph-monitor-1 ~]$ ceph osd pool rm testpool testpool --yes-i-really-really-mean-it
Error EPERM: pool deletion is disabled; you must first set the mon_allow_pool_delete config option to true before you can destroy a pool
</pre>
在删除的时候需要输入两遍存储池的名称和更长的确认删除选项，就是以防止误删除。Ceph集群默认禁用删除功能，需要指定mon_allow_pool_delete选项为false时才可以删除。 
## 集群其它管理
### 扩展monitor节点
monitor作为Ceph集群中重要组件之一，Ceph集群一般部署3个以上奇数节点的monitor集群。
在administrator节点执行执行下面命令把另外两台monitor节点主机加入到monitor集群中。

<pre>
~]$ ceph-deploy mon add ceph-monitor-2
~]$ ceph-deploy mon add ceph-monitor-3
</pre>
这时候再集群健康状态如下。

<pre>
[cephadmin@ceph-monitor-1 ~]$ ceph -s
  cluster:
    id: 7e1c1695-1b01-4151-beea-70c008cffd8c
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum ceph-monitor-1,ceph-monitor-2,ceph-monitor-3
    mgr: ceph-monitor-1(active)
    osd: 7 osds: 7 up, 7 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0  objects, 0 B
    usage:   7.0 GiB used, 313 GiB / 320 GiB avail
    pgs:

</pre>
还可以使用quorum_status子命令查看monitor更详细的信息。

<pre>
~]$ ceph quorum_status --format json-pretty
</pre>
  * --format：指定输出的格式，输入为普通字符串，不便于阅读，可指定输出为json。
### 扩展manager节点
manager节点以守护进程Active和Standby模式运行，当Active节点上面守护进程故障的时候，其中一个Standby实例可以在不中断服务的情况下接管任务。一般有两个manager节点即可，一个为Active，另一个为Standby。集群中第一个manager节点为Active。 
执行下面的命令可添加另一个manager节点。

<pre>
~]$ ceph-deploy mgr create ceph-monitor-2
</pre>
创建成功后查看集群状态如下。

<pre>
[cephadmin@ceph-monitor-1 ~]$ ceph -s
  cluster:
    id: 7e1c1695-1b01-4151-beea-70c008cffd8c
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum ceph-monitor-1,ceph-monitor-2,ceph-monitor-3
    mgr: ceph-monitor-1(active), standbys: ceph-monitor-2
    osd: 7 osds: 7 up, 7 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0  objects, 0 B
    usage:   7.0 GiB used, 313 GiB / 320 GiB avail
    pgs:

</pre>
### 移除monitor节点
1.  在管理节点上面执行destroy子命令删除指定的monitor节点。

<pre>
~]$ ceph-deploy mon destroy ceph-monitor-2
</pre>
如果该服务器还需要启用mon，重新添加即可。 
### 移除manager节点
1.  在指定的manager节点上面将mgr服务停止，并关闭开机自启动。

<pre>
~]$ sudo systemctl stop ceph-mgr@ceph-monitor-1
~]$ sudo systemctl disable ceph-mgr@ceph-monitor-1
</pre>
服务名为：ceph-mgr@<mgr id>。
注意：如果该服务器还需要启用mgr，把服务直接启动即可。但如果是新机器要启用mrg需要通过扩展mgr重新创建。
### 移除OSD硬盘
1.  使用ceph命令停用指定的OSD。

<pre>
~]$ ceph osd out 0
</pre>
0代表第0个OSD设备，编号可以使用ceph osd dump获取对应OSD的编号。 
2.  在被信用的OSD节点上停止对应的服务。
服务名为：ceph-osd@<osd id>。

<pre>
~]$ sudo systemctl stop ceph-osd@0
</pre>
3.  再使用ceph命令将该OSD从集群中移除。

<pre>
~]$ ceph osd purge 0 --yes-i-really-mean-it
</pre>
注意：此方法不适用于L版本之前的版本。 
### 创建RBD接口
RBD主要是块设备接口，通常RBD为KVM等虚拟化OS提供高并发到有无限可扩展性的后端存储。创建RBD接口需要创建专门用于RBD的存储池，其方法如下。
1.  创建一个普通存储池。

<pre>
~]$ ceph osd pool create kvm 32 32
</pre>
  * kvm：定义一个名为kvm的存储池，并指定pg与pgp都为32。
2.  将存储池转换为RBD模式。

<pre>
~]$ ceph osd pool application enable kvm rbd
</pre>
相当于开启存储池的rbd模式，还有其它模式。
3.  初始化存储池。

<pre>
~]$ rbd pool init -p kvm
</pre>
4.  创建镜像。

<pre>
~]$ rbd create img1 --size 10240 --pool kvm
</pre>
  * img1：指定镜像名称
  * --size：指定镜像大小。
  * --pool：指定创建到哪个pool。
5.  查看镜像相关信息。

<pre>
[cephadmin@ceph-monitor-1 ~]$ rbd --image img1 --pool kvm info
rbd image 'img1':
	size 10 GiB in 2560 objects
	order 22 (4 MiB objects)
	id: 85e96b8b4567
	block_name_prefix: rbd_data.85e96b8b4567
	format: 2
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	op_features: 
	flags: 
	create_timestamp: Tue Apr 30 05:25:34 2019
</pre>
### 创建radosgw接口
如果使用到类似S3或者Swift接口时候才需要部署，通常用于对象存储OSS，类似于阿里云OSS。
1.  创建rgw守护进程，可以创建在集群任何节点。

<pre>
~]$ ceph-deploy rgw create ceph-storage-1
</pre>
注意：生产环境下此进程一般需要高可用，方法将在以后的文章介绍。创建成功后默认情况下会创建一系列用于rgw的存储池，如。

<pre>
[cephadmin@ceph-storage-1 ~]$ ceph osd pool ls
testpool
kvm
.rgw.root
default.rgw.control
default.rgw.meta
default.rgw.log
</pre>
2.  通过ceph -s查看集群状态，这时候会出现rgw服务。

<pre>
[cephadmin@ceph-monitor-1 ~]$ ceph -s
  cluster:
    id: 7e1c1695-1b01-4151-beea-70c008cffd8c
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum ceph-monitor-1,ceph-monitor-2,ceph-monitor-3
    mgr: ceph-monitor-2(active), standbys: ceph-monitor-1
    osd: 7 osds: 7 up, 7 in
    rgw: 1 daemon active
 
  data:
    pools:   6 pools, 96 pgs
    objects: 224  objects, 1.8 KiB
    usage:   7.1 GiB used, 313 GiB / 320 GiB avail
    pgs: 96 active+clean
</pre>
3.  默认情况下rgw监听7480号端口，在创建完成后日志有会显示。这时候访问该节点的rgw端口，打开如下图所示，说明部署成功。
[![](http://121.43.168.35/wp-content/uploads/2019/05/1-1.png)](https://www.linux-note.cn/wp-content/uploads/2019/05/1-1.png)
### 创建cephfs接口
使用CephFS接口需要至少运行一个元数据服务mds实例。
1.  使用以下命令创建mds实例，可以创建在集群任何节点，这里根据主机说明创建在ceph-monitor-1节点。

<pre>
~]$ ceph-deploy mds create ceph-monitor-1
</pre>
创建成功后会在ceph-monitor-1会运行一个ceph-mds的进程。在集群内任何节点内，使用以下命令可以查看mds的运行状态。

<pre>
[cephadmin@ceph-storage-1 ~]$ ceph mds stat
, 1 up:standby
</pre>
2.  创建好mds后，接下来需要创建用于cephfs的数据相关的pool池，和元数据相关的pool池，并将二者进行关联。在集群内任何节点内执行，操作如下。

<pre>
~]$ ceph osd pool create cehpfs-metadata 64 64
~]$ ceph osd pool create cehpfs-data 64 64	
~]$ ceph fs new cephfs cehpfs-metadata cehpfs-data
</pre>
3.  使用下面命令可以查看cephfs运行状态。
~]$ ceph fs status cephfs
如果State为active说明状态正常。
### 集群的停止与启动
1.  停止。
  1. 告知Ceph集群不要将OSD标记为out。
  2. 停止存储客户端。
  3. 停止网关，例如：对象网关、NFS Ganesha等。
  4. 停止元数据服务mds。
  5. 停止OSD。
  6. 停止Mgr。
  7. 停止Mon。 
备注：告知Ceph集群不要将OSD标记为out，可以使用以下命令。

<pre>
~]$ ceph osd set noout
</pre>
2.  启动。
与停止反方向。最后需要告诉Ceph集群可以将OSD标记为out，命令如下。

<pre>
~]$ ceph osd unset noout
</pre>

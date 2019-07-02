# Ceph存储（二）集群部署

## 部署环境准备

以下为测试环境部署案例，如果是生产环境推荐osd所在服务器是物理机，根据情况配置不同的硬盘。其它的节点可以是虚拟机，但必须是高可用，如monitor和manager。管理节点administrator必须单独在一台机器，防止集群出现问题时无法进行管理。 

### 主机说明

主机IP地址
主机名
部署角色 

192.168.6.114/18.50.129.111
ceph-admin.gogen.cn
administrator

192.168.6.115/18.50.129.112
ceph-stroage-1.gogen.cn
osd

192.168.6.116/18.50.129.113 
ceph-stroage-2.gogen.cn 
osd

192.168.6.117/18.50.129.114 
ceph-stroage-3.gogen.cn 
osd

192.168.6.118/18.50.129.115 
ceph-stroage-4.gogen.cn 
osd

192.168.6.126/18.50.129.116 
ceph-monitor-1.gogen.cn 
monitor、manager、mds

192.168.6.127/18.50.129.117 
ceph-monitor-2.gogen.cn 
monitor、manager

192.168.6.128/18.50.129.118 
ceph-monitor-3.gogen.cn 
monitor 

### 硬盘说明

在ceph集群中。Administrators、Monitors、Managers和MDSs节点节点对服务器硬盘都没有要求，只要系统能正常运行即可。但OSD节点不一样，通常一个OSD就代表一块物理硬盘，作为分布式存储，OSD越多越好，这里有的4个OSD节点，分别再每节点添加两块硬盘，分别是40GB和 50GB，作为OSD的硬盘不需要做任何操作，需要OSD初始化的工具来回初始化。

查看ceph-storage-1节点硬盘信息如下。 
    
    
    [root@ceph-storage-1 ~]# fdisk -l /dev/sd*
    
    Disk /dev/sda: 21.5 GB, 21474836480 bytes, 41943040 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk label type: dos
    Disk identifier: 0x00024e19
    
       Device Boot      Start         End      Blocks   Id  System
    /dev/sda1   *        2048     2099199     1048576   83  Linux
    /dev/sda2         2099200    41943039    19921920   8e  Linux LVM
    
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

### 网络说明

部署Ceph集群推荐所有节点都有两张网卡，一张用于外部通讯，承载业务数据流量，另外一张用于集群内部了数据交换。防止在进行内部数据交换的时候占用业务通道，影响业务通讯。

在此案例中192.168.6.0/24为业务使用，而 18.50.129.0/24为集群内部使用。         

注意：如果一台服务器有几个网卡都配置了IP地址与网关，那么将会存在几条默认路由。正常情况下我们只需要一条默认路由，而其它不需要的路由则删除，再根据情况添加静态路由。

### 配置时间同步

如果节点可以访问互联网，直接启动chronyd服务即可，CentOS 7自带此服务。
    
    
    ~]# systemctl enable chronyd && systemctl start chronyd

注意：如果是CentOS 6系统，默认不使用chronyd时间同步服务。 

一般推荐使用本地的时间服务器，修改/etc/chrony.conf配置文件，将server后配置修改为指定时间服务器地址即可，如果有多个，指定多个server即可，server后面跟时间服务器的地址，如。 
    
    
    server 0.centos.pool.ntp.org iburst

可以使用以下命令可以查看时间同步的信息。 
    
    
    ~]# chronyc sources -v

### 配置主机名解析

本所本案例，在每个节点都配置此hosts，文件内容如下。 
    
    
    127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
    ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
    192.168.6.114 ceph-admin.gogen.cn ceph-admin
    192.168.6.115 ceph-storage-1.gogen.cn ceph-storage-1
    192.168.6.116 ceph-storage-2.gogen.cn ceph-storage-2
    192.168.6.117 ceph-storage-3.gogen.cn ceph-storage-3
    192.168.6.118 ceph-storage-4.gogen.cn ceph-storage-4
    192.168.6.126 ceph-monitor-1.gogen.cn ceph-monitor-1
    192.168.6.127 ceph-monitor-2.gogen.cn ceph-monitor-2
    192.168.6.128 ceph-monitor-3.gogen.cn ceph-monitor-3

### 关闭防火墙

为防止防火墙的干扰，推荐将防火墙关闭，操作如下。 
    
    
    ~]# systemctl disable firewalld && systemctl stop firewalld

### 关闭SELinux

同样为了防止SELinux干扰，默认系统是启用了SELinux，需要将此关闭。修改/etc/sysconfig/selinux文件来禁用SELinux，修改配置如下。 
    
    
    ~]# sed -i "/^SELINUX/s@enforcing@disabled@" /etc/sysconfig/selinux

修改完SELinux配置后需要重启系统。 

## 准备工作

这里主要是准备好administrator节点，和部署前所需要的一些配置准备配置。

### 配置yum源

使用Ceph官方仓库安装下载较慢，国内阿里云镜像站也提供Ceph相关镜像，地址如下。
    
    
    https://mirrors.aliyun.com/ceph/

安装阿里云Ceph的repo源，在所有节点上安装此源。
    
    
    ~]# rpm -ivh https://mirrors.aliyun.com/ceph/rpm-mimic/el7/noarch/ceph-release-1-1.el7.noarch.rpm

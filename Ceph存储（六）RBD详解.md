# Ceph存储（六）RBD详解
---
## RBD介绍
RBD全称为RADOS Block Device，是一种构建在RADOS集群之上为客户端提供块设备接口的存储服务中间层。这类的客户端包括虚拟机KVM和云计算操作系统OpenStack、CloudStack等。
RBD为条带化，支持存储空间的动态扩容等特性，并可以借助RADOS实现快照，副本和一致性。
客户端访问RBD有两种方式。
  * 通过内核模块rbd.ko将镜像映射为本地块设备，通常设置文件一般为：/dev/rbd*
  * 另一种是通过librbd接口，KVM虚拟机就是使用这种接口。 
## rbd命令使用
对于RBD相关的管理有创建、删除、修改和列出等基本的CRUD操作，另外也有分组，镜像，快照和回收等相关的管理需求，这些所有操作都可以使用rbd命令完成。
在开始管理rdb之前首先初始化至少一个RBD的存储池，参考《[集群部署之创建RBD接口](https://www.linux-note.cn/?p=85#RBD)》。
### 创建镜像
创建image。

<pre>
~]$ rbd create --size 5G --pool <pool name> <image name>
</pre>
  * --size：指定镜像大小。
  * --pool：指定存储池名称。
### 查看镜像信息
列出所有镜像。

<pre>
~]$ rbd ls --pool <pool name>
</pre>
另外还可以使用-l选项输出更多的信息。 

<pre>
~]$ rbd ls --pool <pool name> -l
</pre>
查看指定image信息。

<pre>
~]$ rbd info --pool <pool name> --image <image name>
</pre>
可以将镜像写成spec路径模式，这样就不需要指定--pool和--image，如。

<pre>
~]$ rbd info <pool name>/<image name>
</pre>
info输出信息出下所示。

<pre>
rbd image 'image01':
	size 30 GiB in 7680 objects
	order 22 (4 MiB objects)
	id: 85e96b8b4567
	block_name_prefix: rbd_data.85e96b8b4567
	format: 2
	features: layering, exclusive-lock
	op_features: 
	flags: 
	create_timestamp: Tue Apr 30 05:25:34 2019
</pre>
  * size：镜像的大小与被分割成的条带数。
  * order 22：条带的编号，有效范围是12到25，对应4K到32M，而22代表2的22次方，这样刚好是4M。
  * id：镜像的ID标识。
  * block_name_prefix：名称前缀。
  * format：使用的镜像格式，默认为2。
  * features：当前镜像的功能特性。
  * op_features：可选的功能特性。
### 调整镜像大小

<pre>
~]$ rbd resize <pool name>/<image name> --size <size number>
</pre>
使用resize就可以调整镜像的大小，一般建议只增不减，如果是减少的话需要加一个选项--allow-shrink，如。

<pre>
~]$ rbd resize kvm/image01 --size 5G --allow-shrink
</pre>
### 删除镜像

<pre>
~]$ rbd remove <pool name>/<image name>
</pre>
使用remove或者rm都可以，通常不推荐直接删除镜像，这种方法删除镜像后，镜像将不可恢复。推荐使用trash命令，这个命令删除是将镜像移动一回收站，如果想找回还可以恢复，如。

<pre>
~]$ rbd trash move <pool name>/<image name>
~]$ rbd trash list --pool <pool name>
~]$ rbd trash restore <pool name>/<id>
</pre>
trash move代表将镜像移动到回收站，还可以使用list查看回收站所有镜像、remove从回收站删除镜像，restore从回收站恢复、purge清空回收站。使用trash在操作的时候同样需要指定--pool，也可以使用spec格式，在restore的时候可以指定镜像的ID，在list的时候可以显示出镜像的ID。 
### image特性
  * layering: 是否支持克隆
  * striping: 是否支持数据对象间的数据条带化
  * exclusive-lock: 是否支持分布式排他锁机制以限制同时仅能一个客户端访问当前镜像。
  * object-map: 是否支持object位图， 主要用于加速导入、 导出及已用容量统计等操作， 依赖于exclusive-lock特性。
  * fast-diff: 是否支持快照间的快速比较操作， 依赖于object-map特性。
  * deep-flatten: 是否支持克隆分离时解除在克隆image时创建的快照与其父image之间的关联关系。
  * journaling: 是否支持日志IO， 即是否支持记录image的修改操作至日志对象； 依赖于exclusive-lock特性。
  * data-pool: 是否支持将image的数据对象存储于纠删码存储池， 主要用于将image 的元数据与数据放置于不同的存储池。 
## 快照
对rbd镜像进行快照，可以保留镜像的状态历史，另外还可以利用快照的分层技术，通过将快照克隆为新的镜像使用。
### 创建快照

<pre>
~]$ rbd snap create --pool <pool name> --image <image name> --snap <snap name>
</pre>
  * --pool：指定哪个存储池。
  * --image：指定哪个镜像。
  * --snap：指定快照的名称。

还可以使用下面的命令创建快照，更方便，简单。

<pre>
~]$ rbd snap create <pool name>/<image name>@<snap name>
</pre>
### 查看快照
列出指定镜像所有快照。

<pre>
~]$ rbd snap list <pool name>/<image name>
</pre>
还可以用json格式输出。

<pre>
~]$ rbd snap list <pool name>/<image name> --format json --pretty-format
</pre>
### 回滚快照
回滚镜像到指定快照。

<pre>
~]$ rbd snap rollback <pool name>/<image name>@<snap name>
</pre>
注意： 在回滚快照之需要将镜像取消镜像的映射，然后再回滚。
### 限制快照数
限制镜像可创建快照数。

<pre>
~]$ rbd snap limit set <pool name>/<image name> --limit 3
</pre>
解除限制。

<pre>
~]$ rbd snap limit clear <pool name>/<image name>
</pre>
### 删除快照
删除指定快照。

<pre>
~]$ rbd snap rm <pool name>/<image name>@<snap name>
</pre>
删除所有快照。

<pre>
~]$ rbd snap purge <pool name>/<image name>
</pre>
### 快照分层
快照分层支持将快照克隆生成新的镜像，这种镜像与直接创建的镜像几乎完全一样，支持镜像的所有操作。唯一不同的是克隆镜像引用了一个只读的上游快照，而且此快照必须要设置保护模式。
#### 快照克隆
1.   将上游快照设置为保护模式。

<pre>
~]$ rbd snap protect <pool name>/<image name>@<snap name>
</pre>
2.   克隆快照为新的镜像。

<pre>
~]$ rbd clone <pool name>/<image name>@<snap name> --dest <pool name>/<image name>
</pre>
  * --dest：指定新的镜像名。
克隆完成后可以通过下面命令查看快照的子项。

<pre>
~]$ rbd children <pool name>/<image name>@<snap name>
</pre>
#### 快照展平
通常情况下通过快照克隆的镜像会保留对父快照的引用，这时候不可以删除该父快照，否则会有影响。如果要删除必须先展平其子镜像，展平的时间取决于镜像的大小，操作方法如下。
1.  展平子镜像。

<pre>
~]$ rbd flatten <pool name>/<image name>
</pre>
当展平完成后，正常情况下该快照还是受保护的，无法删除，所以还需要先取消保护。
2.  取消快照保护。

<pre>
~]$ rbd snap unprotect <pool name>/<image name>@<snap name>
</pre>
3.  删除快照

<pre>
~]$ rbd snap rm <pool name>/<image name>@<snap name>
</pre>
## Linux客户端使用
本例主要是使用Linux客户端挂载RBD镜像为本地磁盘使用。
开始之前需要在所需要客户端节点上面安装ceph-common软件包，因为客户端需要调用rbd命令将RBD镜像映射到本地当作一块普通硬盘使用。并还需要把ceph.conf配置文件和授权keyring文件复制到对应的节点。 
### 映射
1.  安装ceph-common软件包。

<pre>
~]# yum install ceph-common
</pre>
注意：安装ceph-common软件包推荐使用软件包源与Ceph集群源相同，软件版本一致。
2.  创建并授权一个用户可访问指定的rbd存储池，这里指定存储池为kvm。

<pre>
~]$ ceph auth caps client.osd-mount osd "allow * pool=kvm" mon "allow r"
</pre>
关于授权相关的知识参考《[Ceph认证与授权](https://www.linux-note.cn/?p=179)》，以上是本案例所需要的权限，指定用户标识为client.osd-mount，可以为其它的标识。对另对OSD有所有的权限，对Mon有只读的权限
3.  将用户的keyring文件和ceph.conf文件复制到目标主机的/etc/ceph目录。
关于如何获取用户的keyring文件也可以参考《[Ceph认证与授权](https://www.linux-note.cn/?p=179)》
4.  修改RBD镜像特性，需要提前创建一个镜像。
默认情况下只支持layering和striping特性，需要将其它的特性关闭。

<pre>
~]$ rbd feature disable <pool name>/<image name> object-map, fast-diff, deep-flatten
</pre>
5.  执行客户端映射。

<pre>
~]$ rbd map --pool <pool name> --image <image name> --keyring /path/to/keyring --user <userType.id>
</pre>
  * --keyring：指定密钥环文件，如果不指定默认找/etc/ceph/ceph.client.admin.keyring。
  * --user：指定访问的用户 
如所下示：

<pre>
~]$ rbd map --pool kvm --image image01 --keyring /etc/ceph/ceph.client.osd-mount.keyring --user osd-mount
</pre>
6.  格式化并挂载。

<pre>
~]# mkfs.xfs /dev/rbd0
~]# mount /dev/rbd0 <mount point>
</pre>
正常情况下会返回映射的配置文件为/dev/rbd* ，可以使用下面的查看目标主机rbd映射的信息，需要在目标主机上面执行。

<pre>
~]# rbd showmapped
</pre>
7.  空间扩容， 如果在调整了image的大小，需要在pod运行所在节点执行下面的命令使其对应的镜像生效。 

<pre>
~]# resize2fs /dev/rbd0
~]# xfs_growfs /dev/rbd0
</pre>
### 断开
1.   umount设备挂载点。

<pre>
~]# umount <mount point>
</pre>
2.  断开与rbd的连接。

<pre>
~]# rbd unmap <pool name>/<image name>
</pre>
## KVM虚拟机使用
在开始之前首先需要部署一台KVM虚拟，然后指定虚拟机xml配置文件，最后修改虚拟机配置文件指定从 Ceph集群加载磁盘即可。
同样使用KVM作为Ceph集群客户端，那么客户端节点必须安装ceph-common软件包，并把ceph.conf配置文件和授权keyring文件复制到对应的节点。 
### 准备KVM虚拟机环境
部署KVM的服务器需要支持虚拟化，否则无法正常使用。使用下面命令可判断服务器是否支持虚拟化，如果有输出表示支持，否则可能需要另外开启。

<pre>
~]# lsmod |grep kvm
</pre>
1.  假设底层支持环境没有问题，安装 虚拟机。

<pre>
~]# yum -y install qemu-kvm qemu-kvm-tools virt-manager virt-install libvirt
</pre>
2.  启动服务

<pre>
~]# systemctl enable libvirtd.service
~]# systemctl start libvirtd.service
</pre>
### 准备用户授权
授权与《[ Linux客户端 ](https://www.linux-note.cn/?p=216#Linux)》使用一样，参考第2步的操作。
本实验如使用如下授权。

<pre>
~]$ ceph auth caps client.osd-mount osd "allow * pool=kvm" mon "allow r"
</pre>
### 创建KVM使用Secret
1.   创建一个xml文件，文件名称随意，内容如下。

<pre>
&lt;secret ephemeral='no' private='no'&gt;
  &lt;usage type='ceph'&gt;
    &lt;name&gt;client.kvm secret&lt;/name&gt;
  &lt;/usage&gt;
&lt;/secret&gt;
</pre>
client.kvm代表Ceph集群的用户，type为ceph表示用户类型为Ceph用户。 
2.  将创建的xml文件转换为Secret。

<pre>
~]# virsh secret-define --file secret.xml
</pre>
  * --file：指定前面的xml文件，该命令会返回一串Secret字符串。Secret相当于KVM去访问Ceph集群时使用的认证，但这个里面只包含了用户名。 
3.  将对应用的Ceph用户的Key与Secret合并。

<pre>
~]# virsh secret-set-value --secret 7b64bd4d-ed43-4ddf-8e36-97c82397e718 --base64 AQCpE9VcOrHbBhAASQ36s2IWCEdw+8wbjoyU/w==
</pre>
  * --secret：指定第2步生成的Secret值。
  * --base64：指定Ceph用户的Key信息。
### 创建虚拟机模板
原则上可以直接通过创建一个虚拟机模板，该模板内指定使用RBD镜像作为启动盘，然后通过定义了这个模板来直接启动虚拟机。但有有可能每个人所定义的配置不一样，这时候可以通过自定义创建一台虚拟机出来，然后在此虚拟机的模板基础上修改得到新的虚拟机。
1.  在Ceph集群创建KVM虚拟机使用的RBD镜像，同样也需要修改该镜像的特性，只支持 layering和striping特性。这里直接使用Linux客户端挂载时创建的镜像，镜像名为img1。
2.  在本地创建KVM虚拟机使用的本地镜像。作为虚拟机首次创建获取模板所用，后面将不使用此镜像启动。

<pre>
~]# qemu-img create -f raw /opt/CentOS7.raw 2G
</pre>
  * -f：指定镜像格式，raw相当于全镜像模式，性能最好，另外还有一种格式为qcow2，这种为稀疏格式，性能较差。
  * 镜像存储在/opt/CentOS7.raw，并指定大小为2G。
3.  使用刚创建的本地镜像，并将一个系统ISO挂载到虚拟驱内一起启动。

<pre>
~]# virt-install --name CentOS7 --virt-type kvm --ram 1024 --cdrom=/opt/CentOS-7-x86_64-Minimal-1810.iso --disk path=/opt/CentOS7.raw --network network=default --graphics vnc,listen=0.0.0.0 --noautoconsole
</pre>
  * --cdrom：指定系统ISO镜像。
  * --disk：指定本地硬盘镜像，第2步创建的镜像。
  * --name：虚拟机名称。
  * --virt-type：虚拟机类型。
  * --network：虚拟机使用的网络类型。
  * --graphics：指定图形相关配置，启动成功后监听0.0.0.0:5900号端口，如果第二台就是0.0.0.0:5901，以此类推。可以使用VNC客户端进行连接。
  * --noautoconsole：不自动连接至控制吧。
4.  虚拟机创建成功后，查看监听端口，这时候使用VNC客户端连接至该端口，正常情况该虚拟机已经启动，并进入系统安装界面，如下图所示。

[![](http://121.43.168.35/wp-content/uploads/2019/05/TTMHU1JSNKF4O8_XEHYU1.png)](https://www.linux-note.cn/wp-content/uploads/2019/05/TTMHU1JSNKF4O8_XEHYU1.png)
5.  命令下面的命令可以查看所有虚拟机运行状态。

<pre>
~]# virsh list --all
</pre>
6.  关闭虚拟机，并获取获取机的配置文件。

<pre>
~]# virsh destroy CentOS7
</pre>
默认虚拟机配置文件存放在/etc/libvirt/qemu/下以虚拟机名称命名，只需要将该配置文件复制一份即可。

<pre>
~]# cp /etc/libvirt/qemu/CentOS7.xml ./CentOS7-RBD.xml
</pre>
7.  将新配置文件修改如下，前面的2到6步操作主要是为了演示配置文件的由来，如果在实验的情况下，此配置文件可直接根据自己的信息修改后使用。

<pre>
&lt;domain type='kvm'&gt;
  &lt;name&gt;CentOS7-RBD&lt;/name&gt;
  &lt;memory unit='KiB'&gt;1048576&lt;/memory&gt;
  &lt;vcpu placement='static'&gt;1&lt;/vcpu&gt;
  &lt;os&gt;
    &lt;type arch='x86_64'&gt;hvm&lt;/type&gt;
    &lt;boot dev="cdrom"/&gt;
  &lt;/os&gt;
  &lt;devices&gt;
    &lt;emulator&gt;/usr/libexec/qemu-kvm&lt;/emulator&gt;
    &lt;disk type='network' device='disk'&gt;
      &lt;source protocol='rbd' name='kvm/img1'&gt;
        &lt;host name='192.168.6.126,192.168.6.127,192.168.6.128' port='6789'/&gt;
      &lt;/source&gt;
     &lt;auth username='osd-mount'&gt;
       &lt;secret type='ceph' uuid='6c5a16cc-7b16-4bdc-be85-d9d664b14570'/&gt;
     &lt;/auth&gt;
     &lt;target dev='vda' bus='virtio'/&gt;
    &lt;/disk&gt;
    &lt;disk type='file' device='cdrom'&gt;
      &lt;driver name='qemu' type='raw'/&gt;
      &lt;target dev='hda' bus='ide'/&gt;
      &lt;source file='/opt/CentOS-7-x86_64-Minimal-1810.iso'/&gt;
      &lt;readonly/&gt;
    &lt;/disk&gt;
    &lt;interface type='network'&gt;
      &lt;source network='default'/&gt;
      &lt;model type='virtio'/&gt;
    &lt;/interface&gt;
    &lt;graphics type='vnc' port='-1' autoport='yes' listen='0.0.0.0'&gt;
      &lt;listen type='address' address='0.0.0.0'/&gt;
    &lt;/graphics&gt;
  &lt;/devices&gt;
&lt;/domain&gt;
</pre>
将配置文件必须的部分保留，注意更改了最上面的<name/>，删除了<mac/>与虚拟机<uuid/>相关的配置。在<os/>内指定了引导启动设备为cdrom，并且在<disk/>内指定了cdrom相关的配置。
主要配置为<disk/>，有两个<disk/>配置，一个为ISO镜像的配置，另一个配置如下。

<pre>
&lt;disk type='network' device='disk'&gt;
  &lt;source protocol='rbd' name='kvm/img1'&gt;
    &lt;host name='192.168.6.126,192.168.6.127,192.168.6.128' port='6789'/&gt;
  &lt;/source&gt;
 &lt;auth username='osd-mount'&gt;
   &lt;secret type='ceph' uuid='6c5a16cc-7b16-4bdc-be85-d9d664b14570'/&gt;
 &lt;/auth&gt;
 &lt;target dev='vda' bus='virtio'/&gt;
&lt;/disk&gt;
</pre>
  * type：指定磁盘从网络获取。
  * <source/>：指定源协议为rbd，镜像名为kvm/img1。相当于<pool name>/<image name>。
  * <host/>：指定Mon主机的IP与端口，多个主机使用逗号分隔，端口单独指定。
  * <auth/>：指定认证信息，username用来指定用户名。secret指定密码，其中type指定为ceph类型，而uuid为前面创建的Secret的ID，可以使用命令virsh secret-list获取。
8.  将配置文件定义为虚拟机。

<pre>
~]# virsh define CentOS7-RBD.xml
</pre>
9.  这时候通过virsh list --all查看新创建的虚拟机，此时状态为关闭。接下来就是开机安装系统即可。

<pre>
~]# virsh start CentOS7-RBD
</pre>
正常情况下虚拟机启动成功后就可以用VNC客户端连接上面正常安装系统即可，关于虚拟机网络即其它相关配置这里不过多介绍，有兴趣可查阅更多相关资料。
系统安装完成后将disk设置为开机引导项即可，删除cdrom的<disk/>块即可。可再使用一份新的配置文件重新定义一台虚拟机，如。

<pre>
&lt;domain type='kvm'&gt;
  &lt;name&gt;CentOS7-RBD&lt;/name&gt;
  &lt;memory unit='KiB'&gt;1048576&lt;/memory&gt;
  &lt;vcpu placement='static'&gt;1&lt;/vcpu&gt;
  &lt;os&gt;
    &lt;type arch='x86_64'&gt;hvm&lt;/type&gt;
  &lt;/os&gt;
  &lt;devices&gt;
    &lt;emulator&gt;/usr/libexec/qemu-kvm&lt;/emulator&gt;
    &lt;disk type='network' device='disk'&gt;
      &lt;source protocol='rbd' name='kvm/img1'&gt;
        &lt;host name='192.168.6.126,192.168.6.127,192.168.6.128' port='6789'/&gt;
      &lt;/source&gt;
     &lt;auth username='osd-mount'&gt;
       &lt;secret type='ceph' uuid='6c5a16cc-7b16-4bdc-be85-d9d664b14570'/&gt;
     &lt;/auth&gt;
     &lt;target dev='vda' bus='virtio'/&gt;
    &lt;/disk&gt;
    &lt;interface type='network'&gt;
      &lt;source network='default'/&gt;
      &lt;model type='virtio'/&gt;
    &lt;/interface&gt;
    &lt;graphics type='vnc' port='-1' autoport='yes' listen='0.0.0.0'&gt;
      &lt;listen type='address' address='0.0.0.0'/&gt;
    &lt;/graphics&gt;
  &lt;/devices&gt;
&lt;/domain&gt;
</pre>

# Ceph存储（三）Pool与PG
---
## 存储池
存储池（Pool）通常情况下可以为特定的应用程序或者不同类型的数据需求创建专用的存储池，如rdb存储池，rgw存储池，个人专用存储池，某部门专用存储池等。
客户端在连接到Ceph集群的时候必须指定一个存储池的名称，并完成用户名和密钥的认证方可连接至指定的存储池。 
### 存储池类型
通过存储池的使用类型有rbd存储池，个人存储池等。按存储池的官方类型可以将存储池分为副本池（replicated）和纠删码池（ erasure code ）
#### 副本池
默认的存储池类型，把每个存入的对象（Object）存储为多个副本，其中分为主副本和从副本，从副本相当于备份副本。如果客户端在上传对象的时候不指定副本数，默认为3个副本。关于副本池数据存储流程如下图所示。
[![](http://121.43.168.35/wp-content/uploads/2019/05/2-1.png)](https://www.linux-note.cn/wp-content/uploads/2019/05/2-1.png)
在开始存数据之前会计算出该对象存储的主副本与从副本的位置，首先会将数据存入到主副本，然后主副本再将数据分别同步到从副本。主副本与从副本同步完毕后会通知主副本，这时候主副本再响应客户端，并表示数据上传成功。所以如果客户端收到存储成功的请求后，说明数据已经完成了所有副本的存储。
#### 纠删码池
此类型会将数据存储为K+M，其中K数据块数量。每个对象存储到Ceph集群的时候会分成多个数据块分开进行存储。而M为为编码块，也代表最多容忍可坏的数据块数量。类似于磁盘阵列RAID5，在最大化利用空间的同时，还能保证数据丢失可恢复性，相比副本池更节约磁盘的空间。
因为副本池很浪费存储空间，如果Ceph集群总容量为100T，如果使用副本池，那么实际可用空间按3个副本算，那么只有30T出头。而使用纠删码池就可以更大化的利用空间，但纠删码池更浪费计算资源。
如存储一个100M的资源，如果使用副本池，按3副本计算实际上要使用300M的空间。而使用纠删码池，如果将100M资源分为25块，如果将M指定为2，那么总共只需要108M空间即可，计算公式为100+100/25*2。
注意：如果存储RBD镜像，那么不支持纠删码池。关于此类型存储池使用不多，不做过多介绍。
## PG
PG英文全称 Placement Group，中文称之为归置组。
### PG的作用
PG相当于一个虚拟组件，出于集群伸缩，性能方面的考虑。Ceph将每个存储池分为多个PG，如果存储池为副本池类型，并会给该存储池每个PG分配一个主OSD和多个从OSD，当数据量大的时候PG将均衡的分布行不通集群中的每个OSD上面。
### 如何正确设置PG数量
一个Ceph集群中，PG的数量不能随便的设定。而应该合理的设定。通常如果一个集群有个超过50个OSD，建议每个OSD大约有50到100到PG。如果有更大规模的集群，建议每个OSD大约100到200个PG。
总PG的数量基本计算公式为： (总OSD数*每个OSD计划PG数)/副本数 => 总PG数
因为总PG数推荐为2的N次幂，计算出来的结果不一定为2的N次幂，需要取比计算结果小的，最近的一个数，并且这个数为2的N次幂。因为2的N次幂计算结果最快，这样可以减少CPU内存的消耗。
说明：总的PG数=所有Pool中定义的PG数的总和
假设集群中有50个OSD，按推荐超过50个OSD的RADOS集群推荐的每个OSD的PG数为50到100。计划PG数为60，这时候公式为：(50*60)/3 = 1000 => 512，所以这时候集群中总的PG数量推荐为512个。
为什么我们定义每个OSD的PG数为60，而不是100?
如果一个OSD的PG数越多，那么在移动数据的时候会更少，但更浪费CPU的内存。如果一个OSD的PG数越少，这样移动的数据会越多，会对正常的性能产生负面影响。而在OSD之间进行数据持久存储和数据分布需要较多的PG，它们的数据应该减少到最大性能所需要的最小值，也减少CPU和内存资源。这个性能需要根据自身情况而定。 备注：PG所对应的就是实际的存储对象。移动PG就相当于移动数据。 
### PG的常见状态
PG的状态可以由以下命令获取。

<pre>
~]$ ceph pg stat
</pre>
通常状态为active+clean表示正常，还可以使用以下命令查看详细信息。

<pre>
~]$ ceph pg dump
</pre>
  * Active：表示主OSD和从OSD都处理就绪状态，可常用提供客户端请求。 
  * Clean：表示主OSD和从OSD都处理就绪状态，所有对象的副本均符合期望。
  * Peering：通常此状态表示正在将主OSD和从OSD的对象同步一致的过程，如果这个过程完成后，通过状态就为Active。
  * Degraded：当我某OSD标记为down的时候，这时候映射到此的OSD的PG将进入Degraded（降级）状态，当OSD重新up，并完成Peering后，将重回正常状态。 一旦标记为down超过5分钟，这时候此OSD将被T出集群，Ceph将启动自恢复操作，相当于重新分配PG，直到状态正常。 有时候某个OSD不可用，崩溃的时候也会处此此状态。
  * Stale：每个OSD都要周期性的向Monitor报千其主OSD所持有的PG最新统计数据。如果因为任何原因某个主OSD主法正常向Monitor报告，或由其它OSD报告某个OSD已经挂了，这时候以以OSD为主的其它OSD都将标记为此状态。
  * Undersized：当PG中的副本数少于其存储池指定的个数的时候，就为此状态。
  * Scrubbing：各OSD还会周期性的检查其持有的数据对象的完性，以确保主和从的数据一致，这时候状态就为此状态。 另外PG偶尔还需要查检确保一个对象的OSD上能按位匹配，这时候状态为scrubbing+deep。
  * Recovering：当添加一个新的OSD到集中中，或者某个OSD宕掉时，PG有要嗵会被重新映射，而这些处理同步过各中的PG则会标记为recovering。
  * Backfilling：新OSD加放到集群后，Ceph会进入数据重新均衡的状态，即一些数据会从现有OSD迁移到新的OSD，这些操作过程即为backfill。 
## 存储池的相关操作
前面介绍了存储池与PG的理论知识，下面主要介绍一些实际的操作。
### 创建
基本格式为。

<pre>
~]$ ceph osd pool create <pool name> <pg num> <pgp num> [type]
</pre>
  * pool name：存储池名称，必须唯一。
  * pg num：存储池中的pg数量。
  * pgp num：用于归置的pg数量，默认与pg数量相等。
  * type：指定存储池的类型，有replicated和erasure， 默认为replicated。
创建一个副本池如下。

<pre>
~]$ ceph osd pool create test01 64
</pre>
### 查看
列出存储池的相关信息。

<pre>
~]$ ceph osd pool ls
~]$ ceph osd pool ls detail
</pre>
使用detail可以获取更详细的信息。
获取存储池的统计数据。

<pre>
~]$ ceph osd pool stats
~]$ ceph osd pool stats detail
</pre>
显示存储池的用量信息。

<pre>
~]$ rados df
</pre>
### 重命名
格式如下。

<pre>
~]$ ceph osd pool rename <old name> <new name>
</pre>
### 删除
通常有两个机制防止存储池被删除，第一个为nodelete标志，该标志的值需要为false才可以删除，默认也是false，如果为true就不可以删除该存储池。可以使用以下命令查看该标志的值。

<pre>
~]$ ceph osd pool get <pool name> nodelete
</pre>
修改的方法如下。

<pre>
~]$ ceph osd pool set <pool name> nodelete false
</pre>
另一个机制是作用于集群范围，配置参数mon_allow_pool_delete，需要将此参数的值设置为true才可以删除，默认为false，修改方法。

<pre>
~]$ ceph tell mon.* injectargs --mon_allow_pool_delete=true
</pre>
上面的命令相当于对mon所有节点注入一个配置mon_allow_pool_delete=true，更改完成后立刻生效。
删除pool。

<pre>
~]$ ceph osd pool rm <pool name> <pool name> --yes-i-really-really-mean-it
</pre>
需要输入两遍存储池的名称，和--yes-i-really-really-mean-it此选项可删除，删除完成后记得把mon_allow_pool_delete改回去。 
### 配额
当我们有很多存储池的时候，有些作为公共存储池，这时候就有必要为这些存储池做一些配额，限制可存放的文件数，或者空间大小，以免无限的增大影响到集群的正常运行。 
设置配额。

<pre>
~]$ ceph osd pool set-quota <pool name> max_objects|max_bytes <value>
</pre>
  * max_objects：代表对对象个数进行配额。
  * max_bytes：代表对磁盘大小进行配额。 
获取配额。

<pre>
~]$ ceph osd pool get-quota <pool name>
</pre>
### 配置参数
对于存储池的配置参数可以通过下面命令获取。

<pre>
~]$ ceph osd pool get <pool name> [key name]
</pre>
如。

<pre>
~]$ ceph osd pool get <pool name> size
</pre>
如果不跟个key名称，会输出所有参数，但有个报错。
设置参数。

<pre>
~]$ ceph osd pool set <pool name> <key> <value>
</pre>
常用的可用配置参数有。
  * size：存储池中的对象副本数
  * min_size：提供服务所需要的最小副本数，如果定义size为3，min_size也为3，坏掉一个OSD，如果pool池中有副本在此块OSD上面，那么此pool将不提供服务，如果将min_size定义为2，那么还可以提供服务，如果提供为1，表示只要有一块副本都提供服务。
  * pg_num：定义PG的数量
  * pgp_num：定义归置时使用的PG数量
  * crush_ruleset：设置crush算法规则
  * nodelete：控制是否可删除，默认可以
  * nopgchange：控制是否可更改存储池的pg num和pgp num
  * nosizechange：控制是否可以更改存储池的大小
  * noscrub和nodeep-scrub：控制是否整理或深层整理存储池，可临时解决高I/O问题
  * scrub_min_interval：集群负载较低时整理存储池的最小时间间隔
  * scrub_max_interval：整理存储池的最大时间间隔
  * deep_scrub_interval：深层整理存储池的时间间隔
### 快照
创建存储池快照需要大量的存储空间，取决于存储池的大小。
创建快照，以下两条命令都可以 。

<pre>
~]$ ceph osd pool mksnap <pool name> <snap name>
~]$ rados -p <pool name> mksnap <snap name>
</pre>
列出快照。

<pre>
~]$ rados -p <pool name> lssnap
</pre>
回滚至存储池快照。

<pre>
~]$ rados -p <pool name> rollback <snap name>
</pre>
删除存储池快照，以下两条命令都可以删除。

<pre>
~]$ ceph osd pool rmsnap <pool name> <snap name>
~]$ rados -p <pool name> rmsnap <snap name>
</pre>
### 压缩
如果使用bulestore存储引擎，默认提供数据压缩，以节约磁盘空间。 
启用压缩。

<pre>
~]$ ceph osd pool set <pool name> compression_algorithm snappy
</pre>
snappy：压缩使用的算法，还有有none、zlib、lz4、zstd和snappy等算法。默认为sanppy。zstd压缩比好，但消耗CPU，lz4和snappy对CPU占用较低，不建议使用zlib。

<pre>
~]$ ceph osd pool set <pool name> compression_mode aggressive
</pre>
aggressive：压缩的模式，有none、aggressive、passive和force，默认none。表示不压缩，passive表示提示COMPRESSIBLE才压缩，aggressive表示提示INCOMPRESSIBLE不压缩，其它都压缩。force表示始终压缩。
压缩参数。
  * compression_max_blob_size：压缩对象的最大体积，超过此体积不压缩。默认为0。
  * compression_min_blob_size：压缩对象的最小体积，小于此体积不压缩。默认为0。
全局压缩选项，这些可以配置到ceph.conf配置文件，作用于所有存储池。
  * bluestore_compression_algorithm
  * bluestore_compression_mode
  * bluestore_compression_required_ratio
  * bluestore_compression_min_blob_size
  * bluestore_compression_max_blob_size
  * bluestore_compression_min_blob_size_ssd
  * bluestore_compression_max_blob_size_ssd
  * bluestore_compression_min_blob_size_hdd
  * bluestore_compression_max_blob_size_hdd
## CRUSH运行图
在Ceph中有很多的运行图，比如Monitor运行图，OSD运行图，集群运行图，MDS运行图和CRUSH运行图。
### 什么是CRUSH
CRUSH是一种类似于一致性hash的算法，用于为RADOS存储集群控制数据分布，全称为：Controlled Replication Under Scalable Hashing
### CRUSH在Ceph集群中的作用
负责数据从PG到OSD的存取。
### 故障域
可以把故障域理解为一个单元，这个单元有大有小，成一个倒树。其中最小为OSD，OSD之上为具体的服务器，服务器通过是放置在机架上，所以再者是机架，机排（一排机架），一个配电单元，机房，数据中心，根（root）。
正确的设置故障域可以降低数据丢失的风险，如果将故障域设置为OSD，那么同一条数据肯定将分布在不同OSD，有台服务器有可能会有多个OSD，有可能这条数据都在这台服务器上面，如果这台服务器宕机，有可能造成丢失。如果将故障域设置为host，那么一条数据肯定分布在不同的host，那么这时候如果有一台host宕机不会造成数据丢失。
Ceph集群中默认的故障域有。
  * osd：硬盘
  * host：服务器
  * chassis：机箱
  * rack：机架（一个机架包含多个机箱）
  * row：机排
  * pdu：配电单元（有可能多个机排共用一个配电单元）
  * pod：多个机排
  * room：机房
  * datacenter：数据中心（有可能多个机房组成一个数据中心）
  * region：区域（华东1，华东2等）
  * root：最顶级，必须存在
注意：这些故障域也称之为Bucket，但些Bucket非radowsgw里面的bucket。
### 故障域算法
每个故障域都自己的算法，比如可以对每个故障域内的对象设置权重，这时候数据将以权重的大小比例将数据均分。比如硬盘大些的可能权重会高些，而硬盘小些的权重将低些。这样才可以保证数据存放到每个OSD的比例都差不多，而不是占用空间大小差不多。通常这样的过程需要一个算法来支持，一般有下面的一些算法。
  * uniform
  * list
  * tree
  * straw
  * straw2：straw的升级版，也是现在默认使用的版本，也推荐使用这个，其它的作为了解即可。
### CRUSH算法流程
  1. take：这一步选择一个根节点，这个节点不一定是root，这个节点可以是任何一个故障域，从指定选择的这个节点开始执行。
  2. select：这里开始选择合适的OSD，如果是副本池那么默认使用firstn的算法，如果是纠删码池默认使用indep算法。
  3. emit：返回最终结果。 
### 自定义CRUSH运行图
#### 获取CRUSH运行图
获取运行图。

<pre>
~]$ ceph osd getcrushmap -o crushmap.bin
</pre>
默认情况下crush运行图是一个二进制文件，还需要将二进制转换为文本，方法如下。

<pre>
~]$ crushtool -d crushmap.bin -o crushmap.txt
</pre>
#### CRUSH运行图解读

<pre>
# begin crush map
tunable choose_local_tries 0
tunable choose_local_fallback_tries 0
tunable choose_total_tries 50
tunable chooseleaf_descend_once 1
tunable chooseleaf_vary_r 1
tunable chooseleaf_stable 1
tunable straw_calc_version 1
tunable allowed_bucket_algs 54

# devices
device 1 osd.1 class hdd
device 2 osd.2 class hdd
device 3 osd.3 class hdd
device 4 osd.4 class hdd
device 5 osd.5 class hdd
device 6 osd.6 class hdd
device 7 osd.7 class hdd

# types
type 0 osd
type 1 host
type 2 chassis
type 3 rack
type 4 row
type 5 pdu
type 6 pod
type 7 room
type 8 datacenter
type 9 region
type 10 root

# buckets
host ceph-storage-1 {
	id -3		# do not change unnecessarily
	id -4 class hdd		# do not change unnecessarily
	# weight 0.049
	alg straw2
	hash 0	# rjenkins1
	item osd.1 weight 0.049
}
host ceph-storage-2 {
	id -5		# do not change unnecessarily
	id -6 class hdd		# do not change unnecessarily
	# weight 0.088
	alg straw2
	hash 0	# rjenkins1
	item osd.2 weight 0.049
	item osd.3 weight 0.039
}
host ceph-storage-3 {
	id -7		# do not change unnecessarily
	id -8 class hdd		# do not change unnecessarily
	# weight 0.088
	alg straw2
	hash 0	# rjenkins1
	item osd.4 weight 0.039
	item osd.5 weight 0.049
}
host ceph-storage-4 {
	id -9		# do not change unnecessarily
	id -10 class hdd		# do not change unnecessarily
	# weight 0.088
	alg straw2
	hash 0	# rjenkins1
	item osd.6 weight 0.049
	item osd.7 weight 0.039
}
root default {
	id -1		# do not change unnecessarily
	id -2 class hdd		# do not change unnecessarily
	# weight 0.312
	alg straw2
	hash 0	# rjenkins1
	item ceph-storage-1 weight 0.049
	item ceph-storage-2 weight 0.088
	item ceph-storage-3 weight 0.088
	item ceph-storage-4 weight 0.088
}

# rules
rule replicated_rule {
	id 0
	type replicated
	min_size 1
	max_size 10
	step take default
	step chooseleaf firstn 0 type host
	step emit
}

# end crush map
</pre>
  * tunable：这里都是一些微调的参数，通常不建议修改，一般得集群规模够大，到了专家级别才会去修改。
  * devices：这下面将列出集群中的所有OSD基本信息。
  * types：这里列出集中可用的故障域，可以自定义。
  * buckets：这里就是定义故障域名。 
  * rules：这里定义的是存储池的规则，type为存储池的类型，replicated代表副本池。如果有纠删码池也会创建出一个默认的配置，这里没有。min_size代表允许的最少副本数，max_size允许的最大副本数。step代表每一个步骤，基本第二步为选择OSD，在选择OSD的时候就定义了需要使用哪种故障域级别，这里定义为host，如果有机房或者其它的，可以将故障域定义为更高的级别。

关于buckets定义格式如下。

<pre>
&lt;bucket name&gt; &lt;name&gt; {
    ……
}

</pre>
  * bucket name：与故障域的某一名称相同。
  * name：自定义名称。

buckets里面需要定义算法还有此故障域的子故障域，比如host下面就一定是osd，没有其它的。而pdu下面可以有row、rack、chassis等，根据# types由大到小。root作为最低层是必须有的，因为我们这里没有机房，机排之内的，所以root里面直接就是主机。 
#### 保存CRUSH运行图
通过自定义修改了CRUSH运行图，现在需要将修改保存，首先是将文本转换为二进制，然后再导入到集群中。
首先将文本文件保存为二进制文件，然后再将二进制文件导入至集群。

<pre>
~]$ crushtool -c crushmap.txt -o crushmap-v2.bin
~]$ ceph osd setcrushmap -i crushmap-v2.bin
</pre>

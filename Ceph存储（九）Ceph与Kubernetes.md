# Ceph存储（九）Ceph与Kubernetes
---
## 创建RBD存储池
关于RBD存储池的创建参考《[Ceph集群部署之创建RBD接口](https://www.linux-note.cn/?p=85#RBD)》章节
## 用户授权
关于用户授权参考《[Ceph认证与授权](https://www.linux-note.cn/?p=179)》此文章
此案例授权信息如下。

<pre>
~]$ ceph auth add client.osd-mount osd "allow * pool=kvm" mon "allow r"
</pre>
## Kubernetes集群前期准备
1.  需要在 kubernetes所有节点上面安装ceph-common软件包，然后需要将用户keyring文件与ceph.conf集群配置文件复制到kubernetes集群所有节点上的/etc/ceph目录下面，包括master节点与node节点。
注意：安装ceph-common软件包推荐使用软件包源与Ceph集群源相同，软件版本一致。
2.  在kubernetes任意节点验证用户能否正常接入ceph集群，并可以创建镜像，如果可以说明说明授权正常。

<pre>
~]# rbd --user osd-mount create --size 5G kvm/k8s-test01
</pre>
在kubernetes节点上面安装好ceph-common软件包后就有rbd命令。
  * --user：指定访问ceph集群时使用的用户名，如果不指定默认使用admin。如果使用admin，默认是寻找/etc/admin/cluster.client.admin.keyring文件。此例子会寻找/etc/admin/cluster.client.osd-mount.keyring文件。
3.  修改RBD镜像的特性，kubernetes使用rbd镜像默认只支持layering和striping特性。而通过第2步创建的镜像默认有5个特性，现在需要将另外的3个特性关闭，本操作在ceph集群节点上面执行。

<pre>
~]$ rbd feature disable kvm/k8s-test01 object-map fast-diff deep-flatten
</pre>
## 容器直接挂载使用
在创建pod的时候直接定义volumes，将volumes指定为ceph集群。
### 使用Ceph密钥环文件
创建一个配置资源清单，定义内容如下。

<pre>
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  containers:
  - image: nginx
    name:  test
    volumeMounts:
    - name: rbd-image
      mountPath: /data
  volumes:
  - name: rbd-image
    rbd:
      monitors:
      - "192.168.6.126:6789"
      - "192.168.6.127:6789"
      - "192.168.6.128:6789"
      pool: kvm
      image: k8s-test01
      fsType: ext4
      readOnly: false
      user: osd-mount
      keyring: /etc/ceph/ceph.client.osd-mount.keyring
</pre>
  * monitors：指定mon节点的地址与端口，最好使用域名，默认monitor监听6789号端口。
  * pool：指定存储池名称。
  * image：指定要挂载的镜像名称.
  * fsType：指定文件系统类型，如果使用xfs有可能会有问题。
  * readOnly：指定只读为false，也就是可读可写。
  * user：指定访问ceph集群的用户名，也就是授权的用户名。
  * keyring：指定用户keyring文件位置。

创建好资源清单后直接apply，这时候进入pod内该容器查看分区，内容如下。

<pre>
$ kubectl exec -ti test -- df -h
Filesystem   Size  Used Avail Use% Mounted on
overlay  196G  7.4G  189G   4% /
tmpfs1.9G 0  1.9G   0% /dev
tmpfs1.9G 0  1.9G   0% /sys/fs/cgroup
/dev/rbd04.8G   20M  4.8G   1% /data
/dev/mapper/centos-root  196G  7.4G  189G   4% /etc/hosts
shm   64M 0   64M   0% /dev/shm
tmpfs1.9G   12K  1.9G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs1.9G 0  1.9G   0% /sys/firmware
</pre>
可以看到pod内挂载了一块设置/dev/rbd0，并且挂到了指定的位置。 
pod在挂载的时候是调用pod所在宿主机的rbd命令挂载，然后映射到容器内，所以在pod运行所在主机执行“rbd showmapped”命令可以看到挂载的信息。
如果在调整了image的大小，需要在pod运行所在节点执行下面的命令让其生效。 

<pre>
~]# resize2fs /dev/rbd0
~]# xfs_growfs /dev/rbd0
</pre>
如果是ext4文件系统执行resize2fs，如果是xfs文件系统执行xfs_growf命令。
### 使用Kubernetes Secret配置清单
如果直接将keyring文件写在yaml文件这样不安全，这时候可以通过将用户的key通过base64计算过后存入k8s的secret清单中，然后在pod中引用这个secret即可，这样也可以不用把keyring文件复制到k8s所有节点。
1.  获取用户key然后进行base64计算。

<pre>
~]$ ceph auth print-key client.osd-mount | base64
</pre>
2.  创建secret资源清单。

<pre>
apiVersion: v1
kind: Secret
metadata:
  name: osd-mount-secret
type: "kubernetes.io/rbd"
data:
  key: QVFDVWo5SmNWWWUvS1JBQXZCMFl3bnY1eHJlZ1d5Wjc4SmUyQ2c9PQ==
</pre>
  * type：指定类型为kubernetes.io/rbd。
  * data：key名称就为key，value为用户的key进行base64后的值。
3.  创建pod资源清单。

<pre>
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  containers:
  - image: nginx
    name:  test
    volumeMounts:
    - name: rbd-image
      mountPath: /data
  volumes:
  - name: rbd-image
    rbd:
      monitors:
      - "192.168.6.126:6789"
      - "192.168.6.127:6789"
      - "192.168.6.128:6789"
      pool: kvm
      image: k8s-test01
      fsType: ext4
      readOnly: false
      user: osd-mount
      secretRef:
        name:  osd-mount-secret
</pre>
在最后使用secretRef指定加载对应的secret。
## 通过StorageClass使用
动态申请资源默认情不支持kubeadm部署的集群，因为k8s动态申请pv是由kube-controller组件完成。而kubeadm默认部署的kube-controller是容器，这时候容器内需要去访问ceph集群，能够管理rbd，所以这里时候需要在容器内安装ceph-common，还需要把keyring和ceph.conf等信息复制进去，不建议直接改变kube-controller容器。如果使用的是二进制部署的kube-controller，那么就没有任何问题，在kube-controller所在节点部署ceph-common，把keyring文件的ceph.conf文件复制过去即可。如果kubeadm部署的集群想要支持，需要外部external-provision的方式提供rbd管理工具。
1.   部署rbd-provision，这里使用github的一个项目《[项目地址](https://github.com/kubernetes-incubator/external-storage/tree/master/ceph/rbd/deploy)》。
该项目分为rbac方式部署和非rbac方式部署，因为k8s默认支持rbac授权，所以这里使用rbac方式，执行下面命令将所有rbac所需要的配置清单下载到本地。

<pre>
~]# for url in {clusterrole.yaml,clusterrolebinding.yaml,deployment.yaml,role.yaml,rolebinding.yaml,serviceaccount.yaml};do wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/ceph/rbd/deploy/rbac/$url; done
</pre>
2.   获取ceph集群admin账号的key，并使用base64计算后的结果。 此账号用来管理rbd，在使用storageclass的时候动态创建与删除。也可以都用同一个账号，只要都有权限即可。
3.  获取一个普通用户账号的key，此账号就来挂载rbd镜像。
备注：关于如何创建用户参考本文《[用户授权](https://www.linux-note.cn/?p=152#i)》，关于如何获取用户key并进行base64计算，参考上一小节。
4.  创建两个secret，分别对应管理账号和普通账号，如下所示。

<pre>
apiVersion: v1
kind: Secret
metadata:
  name: ceph-osd-mount-secret
type: "kubernetes.io/rbd"
data:
  key: QVFDVWo5SmNWWWUvS1JBQXZCMFl3bnY1eHJlZ1d5Wjc4SmUyQ2c9PQ==
---
apiVersion: v1
kind: Secret
metadata:
  name: ceph-admin-secret
type: "kubernetes.io/rbd"
data:
  key: QVFCSXhzZGNhb0M5QVJBQUtzNWYzRTJIam1RMTYwL3lUNzdCdEE9PQ==
</pre>
5.  创建StorageCalss配置清单。

<pre>
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rbd
  annotations:
    storageclass.beta.kubernetes.io/os-default-class: "true"
provisioner: ceph.com/rbd
parameters:
  monitors: 192.168.6.126:6789,192.168.6.127:6789,192.168.6.128:6789
  pool: kvm
  adminId: admin
  adminSecretName: ceph-admin-secret
  adminSecretNamespace: default
  userId: osd-mount
  userSecretName: ceph-osd-mount-secret
  userSecretNamespace: default
  imageFormat: "2"
  fsType: ext4
  imageFeatures: "layering"
reclaimPolicy: Retain
</pre>
  * storageclass.beta.kubernetes.io/os-default-class：指定该存储类为默认存储类，如果pvc没有指定存储类名，那么就会从默认存储类中获取，一个集群中只能有一个默认存储类。
  * monitors：指定monitor的地址，多个用逗号分开。
  * pool：指定存储池名。
  * admin*：指定用于管理的用户信息，该用户会创建删除rbd镜像。
  * user*：指定普通用户信息，该用户会去挂载rbd镜像。
  * imageFormat：指定rbd镜像的版本，默认也为2。
  * fsType：指定rbd镜像挂载文件系统。
  * imageFeatures：设置创建的rbd镜像的功能，目前只支持layering，默认为空，什么都没有。并且只有指定imageFormat为2的时候才可以使用。
  * reclaimPolicy： 默认情况下pvc的回收策略为“Delete”，也就是删除pvc后pv也会删除，pv删除那么也会触发ceph集群中对应的镜像删除，所以在定义存储类的的时候应该指定创建的pv回收策略为“Retain”。

关于StorageClass使用Ceph存储《[官方文档](https://kubernetes.io/docs/concepts/storage/storage-classes/#ceph-rbd)》
6.  创建PVC。

<pre>
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: rbd-claim01
spec:
  storageClassName: rbd
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
</pre>
apply成功后，会动态的申请一块rbd镜像。ceph集群中查看。会申请一块5GB大小的RBD镜像。
7.  配置pod使用pvc。

<pre>
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  containers:
  - image: nginx
    name:  test
    volumeMounts:
    - name: rbd-image
      mountPath: /data
  volumes:
  - name: rbd-image
    persistentVolumeClaim:
      claimName: rbd-claim01
</pre>
只需要指定pvc的名称即可，空间扩容与《[容器直接挂载使用](https://www.linux-note.cn/?p=152#i-2)》方法一样。 

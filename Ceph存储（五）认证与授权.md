# Ceph存储（五）认证与授权
---
## 认证与授权
Ceph使用cephx协议对客户端进行身份验证，集群中每一个Monitor节点都可以对客户端进行身份验证，所以不存在单点故障。cephx仅用于Ceph集群中的各组件，而不能用于非Ceph组件。它并不解决数据传输加密问题，但也可以提高访问控制安全性问题。
## 认证授权流程
  1. 客户端向Monitor请求创建用户。
  2. Monitor返回用户共享密钥给客户端，并将此用户信息共享给MDS和OSD。
  3. 客户端使用此共享密钥向Monitor进行认证。
  4. Monitor返回一个session key给客户端，并且此session key与对应客户端密钥进行加密。此session key过一段时间后就会失效，需要重新请求。
  5. 客户端对此session key进行解密，如果密钥不匹配无法解密，这时候认证失败。
  6. 如果认证成功，客户端向服务器申请访问的令牌。
  7. 服务端返回令牌给客户端。
  8. 这时候客户端就可以拿着令牌访问到MDS和OSD，并进行数据的交互。因为MDS和Monitor之间有共享此用户的信息，所以当客户端拿到令牌后就可以直接访问。

认证授权流程如下图所示：

[![](http://121.43.168.35/wp-content/uploads/2019/05/1-2.png)](https://www.linux-note.cn/wp-content/uploads/2019/05/1-2.png)
## 用户
用户通常指定个人或某个应用，个人就是指定实际的人，比如管理员。而应用就是指客户端或Ceph集群中的某个组件，通过用户可以控制谁可以如何访问Ceph集群中的哪块数据。
Ceph支持多种类型的用户，个人与某应用都属于client类型。还有mds、osd、mgr一些专用类型。
## 用户标识
用户标识由“TYPE.ID”组成，通常ID也代表用户名，如client.admin、osd.1等。
## 使能caps
使能表示用户可以行使的能力，通俗点也可以理解为用户所拥有的权限。 对于不同的对象所能使用的权限也不一样，大致如下所示。
  * Monitor权限有：r、w、x和allow、profile、cap。
  * OSD权限有：r、w、x、class-read、class-wirte和profile osd。另外OSD还可以指定单个存储池或者名称空间，如果不指定存储池，默认为整个存储池。 
  * MDS权限有：allow或者留空。

关于各权限的意义：
  * allow：对mds表示rw的意思，其它的表示“允许”。
  * r：读取。
  * w：写入。
  * x：同时拥有读取和写入，相当于可以调用类方法，并且可以在monitor上面执行auth操作。
  * class-read：可以读取类方法，x的子集。
  * class-wirte：可以调用类方法，x的子集。
  * *：这个比较特殊，代表指定对象的所有权限。
  * profile：类似于Linux下sudo，比如profile osd表示授予用户以某个osd身份连接到其它OSD或者Monitor的权限。profile bootstrap-osd表示授予用户引导OSD的权限，关于此处可查阅更多资料。 
## 用户管理
关于用户管理将介绍如何创建、修改与删除用户。
### 查看用户信息
查看所有用户信息。

<pre>
~]$ ceph auth list
</pre>
list可以简写为ls，通过此命令可以获取所有用户的key与权限相关信息。如果只需要某个用户的信息可以使用get子命令，如。

<pre>
~]$ ceph auth get client.admin
</pre>
这里clinet.admin为用户标识，如果只需要某个用户的key信息，可以使用pring-key子命令，如。

<pre>
~]$ ceph auth print-key client.admin 
</pre>
### 创建用户

<pre>
~]$ ceph auth add client.test mon "allow r" osd "allow rw" 
</pre>
上面创建了一个用户，用户标识为client.test。指定该用户对mon有r的权限，对osd有rw的权限，osd没有指定存储池，所以是对所有存储池都有rw的权限。在创建用户的时候还会自动创建用户的密钥。 

<pre>
~]$ ceph auth import -i ceph.client.test.keyring
</pre>
使用import可以从某个keyring文件中导入用户信息，-i指定keyring文件的位置。有导入就有导出，导出用户信息命令如下。

<pre>
~]$ ceph auth get client.test -o ceph.client.test.keyring
</pre>
获取到用户信息后，指定-o选项即可将用户信息保存至keyring文件。
### 修改用户权限

<pre>
ceph auth caps client.test mon "allow r" osd "allow rw pool=kvm"
</pre>
修改权限不等于增加权限，修改后的权限会完全覆盖之前的所有权限。这里在osd上面指定了pool=kvm，如果不指定默认就是所有存储池。 
### 删除用户

<pre>
ceph auth del client.test
</pre>
## 关于keyring密钥环
keyring文件是一个包含密码，key，证书等内容的一个集合。一个keyring文件可以包含多个用户的信息，也就是可以将多个用户信息存储到一个keyring文件。 
### keyring自动加载顺序
当访问Ceph集群时候默认会从以下四个地方加载keyring文件。
  * /etc/ceph/cluster-name.user-name.keyring：通过这种类型的文件用来保存单个用户信息，文件名格式固定：集群名.用户标识.keyring。如ceph.client.admin.keyring。这个代表ceph这个集群，这里的ceph是集群名，而client.admin为admin用户的标识。
  * /etc/ceph/cluster.keyring：通用来保存多个用户的keyring信息。
  * /etc/ceph/keyring：也用来保存多个用户的keyring信息。
  * /etc/ceph/keyring.bin：二进制keyring文件，也用来保存多个用户的keyring信息。 
### keyring文件管理
创建keyring文件。

<pre>
ceph-authtool --create-keyring /etc/ceph/cluster.keyring
</pre>
通过ceph-authtool命令也可以用来管理用户，比ceph更底层。通常keyring文件应该保存至/etc/ceph目录下，并且文件名应按照规范。上面创建的多用户的keryring文件。 
将用户添加至keryring。
ceph-authtool /etc/ceph/cluster.keyring --import-keyring /etc/ceph/ceph.client.test.keyring 
  * --import-keyring：指定被导入的信息。

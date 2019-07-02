# Ceph存储（八）RadosGW详解
---
## 简介
RadosGW通常作为对象存储（Object Storage）使用，类型于阿里云OSS。对象存储通常数据于同一平面，一般用于云计算环境中。每一条数据都作为单独的对象存储，拥有唯一的地址来识别数据对象，专为使用在API应用级别进行访问而设计。另外对象存储中的对象通常不需要再修改，如果需要修改只能下载下来修改再重新上传，无法直接修改。
一般来说，一个对象存储的核心资源有用户（User）、存储桶（Bucket）和对象（Object）。三者的关系是对用户将对象存储到存储系统上的存储桶，存储桶属于某个用户并可以容纳对象，一个用户可以拥有多个存储桶，而一个存储桶用于存储多个对象。大多数对象存储的核心资源类型大同小异，如亚马逊S3、OpenStack Swift与RadosGW。这其中S3与Swift互不兼容，而RadosGW兼容S3与Swift。
RadosGW为了兼容S3与Swift，Ceph在RadosGW集群的基础上提供了RGW（RadosGateWay）数据抽象层和管理层，它可以原生兼容S3和Swift的API。S3和Swift它们可基于http或https完成数据交换，由RadosGW内建的Civeweb提供服务。它还可以支持主流的Web服务器程序以代理的形式接收用户请求，再转发至RadosGW进程，这些代理服务器包括nginx、haproxy等。  RGW的功能依赖于对象网关守护进程实现，负责向客户端提供REST API接口。出于冗余负载均衡的需求，一个Ceph集群上通常不止一个RadosGW守护进程。在云计算机环境中还会在多个Ceph集群中定义出多个Zone，这些Zone之间通过同步实现冗余功能,在本地环境中通常不需要Zone。 
## 关于Civeweb
RadosGW守护进程内部就由Civeweb实现，通过对Civeweb的配置可以完成对RadosGW的基本管理。
### 服务管理
服的名格式为： ceph-radosgw@rgw.<rgw node name> ，如。

<pre>
~]$ systemctl restart ceph-radosgw@rgw.ceph-storage-1
</pre>
### 更改监听端口
Civeweb默认监听在7480端口并提供http协议，如果需要修改配置需要编辑ceph.conf配置文件，在管理节点编辑ceph.conf，新增如下配置。

<pre>
[client.rgw.ceph-storage-1]
rgw_host = ceph-storage-1
rgw_frontends = "civetweb port=8080 num_threads=500 request_timeout_ms=60000"
</pre>
RadosGW也作为一个客户端，所以配置项应该为[client.rgw.ceph-storage-1]，其中client代表客户端配置，RadosGW客户端配置，而最后ceph-storage-1代表对某一个节点的RadosGW配置。
  * rgw_host：对应的RadosGW名称或者IP地址。
  * rgw_frontends：这里配置监听的端口，是否使用https，以即一些选项信息。 

常用的rgw_frontends配置选项有。
  * num_threads：最大并发连接数，默认为50，根据需求调整，通常在生产集群环境中此值应该更大。
  * request_timeout_ms：发送与接收超时时长，以ms为单位，默认为30000。
  * access_log_file：访问日志路径，默认为空。
  * error_log_file：错误日志路径，默认为空。

注意：ceph.conf配置文件一般应该修改管理节点的配置文件，然后由管理节点统一推送到指定的节点。也可以直接修改对应的节点的文件，无论以何种方式修改完成后都需要重启对应的RadosGW服务。通过管理节推送配置文件的命令如下。

<pre>
~]$ ceph-deploy --overwrite-conf config  push ceph-storage-1
</pre>
  * --overwrite-conf：表示强制覆盖，如果修改了管理节点的ceph.conf配置文件后，这样管理节点与被推送的节点的配置文件不一致，这时候如果确认没有问题就需要强制覆盖。
### HTTPS配置
首先需要生成证书，CA签名证书或者自签证书都可以，测试使用这里生成自签名证书即可。
1.  生成私钥。

<pre>
~]$ openssl genrsa -out civetweb.key 2048
</pre>
2.  生成公钥。

<pre>
~]$ openssl req -new -x509 -key civetweb.key -out civetweb.crt -days 3650 -subj "/CN=oss.gogen.cn"
</pre>
3.  将生成的证书合并为pem。

<pre>
~]$ cat civetweb.key civetweb.crt > civetweb.pem
</pre>
4.  修改ceph.conf配置文件，指定配置如下。

<pre>
[client.rgw.ceph-storage-1]
rgw_host = ceph-storage-1
rgw_frontends = "civetweb port=443s ssl_certificate=/etc/ceph/civetweb.pem num_threads=500 request_timeout_ms=60000"
rgw_dns_name = oss.gogen.cn
</pre>
  * port：如果是https端口，需要在端口后面加一个s。
  * ssl_certificate：指定证书的路径。
  * rgw_dns_name：如果使用域名，这里需要指定域名。

还可以同时开启http+https，将port指定为如下配置。

<pre>
rgw_frontends = "civetweb port=80+443s ssl_certificate=/etc/ceph/civetweb.pem num_threads=500 request_timeout_ms=60000"
</pre>
5.  验证，如果是http+https那么两个协议都可以访问，正常返回结果如下。

[![](http://121.43.168.35/wp-content/uploads/2019/05/375R9NDV7OZB50RZ4.png)](https://www.linux-note.cn/wp-content/uploads/2019/05/375R9NDV7OZB50RZ4.png)

[![](http://121.43.168.35/wp-content/uploads/2019/05/I9NR2AY7K1ZRJRC24W.png)](https://www.linux-note.cn/wp-content/uploads/2019/05/I9NR2AY7K1ZRJRC24W.png)
### 配置泛域名解析
通常每个对象都是存储在特定的存储桶之中，而每个对象都要直接通过REST API基于URL进行访问，URL的格式为： http(s)://bucket-name.domain-name:port/path/to/object。以本实验为例，如果bucket-name名称为test，而对象的路径为/image/1.jpg，那么URL为：https://test.oss.gogen.cn/image/1.jpg 
如果Bucket有很多的话，这时候就需要使用泛域名解释，将*.oss.gogen.cn都解析到同一个地址，*代表任何主机名。而在云计算环境中提供对象存储服务，对于Bucket的需要的数量和名称是未知的，所以需要泛域名解析。 
泛域名就是指主机名为*的域名，比如：*.oss.gogen.cn，一般添加一条*为主机名的A记录指向即可完成泛域名解析。
### 创建RadosGW用户
创建RadosGW用户需要使用radosgw-admin命令，在集群中任何可管理节点执行下面命令。

<pre>
~]$ radosgw-admin user create --uid="test" --display-name="test user"
</pre>
  * --uid：指定用户名。
  * --display-name：指定用户描述信息。
创建成功后将输入用户的基本信息，基本最重要的两项为access_key和secret_key。关于更我用户管理可以使用选项--help查看。
用户创建成后功，如果忘记用户信息可以使用下面的命令查看。

<pre>
~]$ radosgw-admin user info --uid="test"
</pre>
## S3接口访问测试
一般情况下访问RadosGW集群大多情况都是使用的S3的接口方法进行访问，这里将使用一个Linux下的S3的客户端工具进行访问测试。
### 安装客户端工具
测试S3的API接口，需要使用s3cmd命令，该命令需要安装软件包s3cmd，该软件包在epel源。

<pre>
~]# yum install s3cmd
</pre>
### 初始化客户端配置
执行下面命令进行初始化客户端配置。

<pre>
[root@localhost ~]# s3cmd --configure

Enter new values or accept defaults in brackets with Enter.
Refer to user manual for detailed description of all options.

Access key and Secret key are your identifiers for Amazon S3. Leave them empty for using the env variables.
Access Key: DBX185L1FBOHZFC5ZBP3
Secret Key: 9MRABYIYrLNqogyD3qgQVzGX7iBAZwQbvLVGBk4o
Default Region [US]: 

Use "s3.amazonaws.com" for S3 Endpoint and not modify it to the target Amazon S3.
S3 Endpoint [s3.amazonaws.com]: oss.gogen.cn:80

Use "%(bucket)s.s3.amazonaws.com" to the target Amazon S3. "%(bucket)s" and "%(location)s" vars can be used
if the target S3 system supports dns based buckets.
DNS-style bucket+hostname:port template for accessing a bucket [%(bucket)s.s3.amazonaws.com]: %(bucket)s.oss.gogen.cn:80

Encryption password is used to protect your files from reading
by unauthorized persons while in transfer to S3
Encryption password: 
Path to GPG program [/usr/bin/gpg]: 

When using secure HTTPS protocol all communication with Amazon S3
servers is protected from 3rd party eavesdropping. This method is
slower than plain HTTP, and can only be proxied with Python 2.7 or newer
Use HTTPS protocol [Yes]: No

On some networks all internet access must go through a HTTP proxy.
Try setting it here if you can't connect to S3 directly
HTTP Proxy server name: 

New settings:
  Access Key: DBX185L1FBOHZFC5ZBP3
  Secret Key: 9MRABYIYrLNqogyD3qgQVzGX7iBAZwQbvLVGBk4o
  Default Region: US
  S3 Endpoint: oss.gogen.cn:80
  DNS-style bucket+hostname:port template for accessing a bucket: %(bucket)s.oss.gogen.cn:80
  Encryption password: 
  Path to GPG program: /usr/bin/gpg
  Use HTTPS protocol: False
  HTTP Proxy server name: 
  HTTP Proxy server port: 0

Test access with supplied credentials? [Y/n] 
Please wait, attempting to list all buckets...
Success. Your access key and secret key worked fine :-)

Now verifying that encryption works...
Not configured. Never mind.

Save settings? [y/N] y
Configuration saved to '/root/.s3cfg'
</pre>
  * Access Key：创建用户时生成的Access Key。
  * Secret Key：创建用户时生成的Secret Key。
  * Default Region：选择区域，默认为US，本环境没有区域，通常用于云环境，所以这里默认即可。
  * S3 Endpoint：输入Endpoit的地址，域名+端口，这里使用http，所以需要使用oss.gogen.cn:80。
  * DNS-style bucket+hostname:port template for accessing a bucket：这里的DNS样式，格式为%(bucket)s.oss.gogen.cn:80。也就是bucket.hostname:port的模式样式。
  * Encryption password：加密的密码，这里没有，直接回车留空。
  * Path to GPG program：这里是加密需要用到的程序包，默认Linux客户端已经安装。
  * Use HTTPS protocol：是否使用HTTPS，默认为使用，不使用输入No。如果使用证书需要为可信任证书，否则无法使用。比如Endpoint的域名为oss.gogen.cn，那么需要申请一个*.oss.gogen.cn的泛域名证书。
  * HTTP Proxy server name：代理服务器名称，这里没有代理服务器，默认留空。
  * New settings：配置信息确认。
  * Test access with supplied credentials：是否进行访问测试，默认为Y。
  * Save settings：如果测试成功会有些提示，是否将配置信息保存到文件，默认在/root/.s3cfg。后面在使用s3cmd命令的时候会自动从此目录读取配置，后续也可以对这里直接进行修改。
### 创建与查看Bucket
创建Bucket。

<pre>
~]# s3cmd mb s3://images
</pre>
如果报ERROR: S3 error: 403 (SignatureDoesNotMatch)错误，编辑s3cfg文件，将里面signature_v2的值改为True即可。 
  * mb：表示创建Bucket，为s3cmd子命令，可以使用s3cmd --help查看更多命令使用方法。
  * s3://：代表使用的协议，协议后面紧跟着的就是Bucket的名称。
查看Bucket。

<pre>
~]# s3cmd ls s3://
</pre>
### 上传对象
上传对象命令格式。

<pre>
~]# s3cmd put &lt;object path&gt; s3://&lt;bucket name&gt;/&lt;save path&gt;
~]# s3cmd put ./ceph-release-1-1.el7.noarch.rpm s3://images/ceph-release-1-1.el7.noarch.rpm
</pre>
  * put：代表上传对象。
  * &lt;object path&gt;：需要上传的对象的路径。
  * &lt;buketc name&gt;：需要上传到哪个Bucket。
  * &lt;save path&gt;：保存的路径，可以有多层目录。

上传保存的时候指定目录，目录不存在会自行创建，如。

<pre>
~]# s3cmd put ./ceph-release-1-1.el7.noarch.rpm s3://images/rpm/ceph-repo.rpm
</pre>
说明：上传的文件会通过md5校验，如果文件一样，无需重新上传新文件，而是直接创建一个类型于硬链接。
### 下载对象
格式如下。

<pre>
~]# s3cmd get s3://&lt;bucket name&gt;/&lt;save path&gt;
</pre>
### 查看对象
查看指定bucket下所有文件。

<pre>
~]# s3cmd ls s3://&lt;;bucket name&gt;
</pre>
## Swift接口访问测试
关于Swfi接口测试不做过多介绍。Swift接口测试可以使用Python Swiftclient模块，可以使用下面命令进行安装。

<pre>
~]# pip install --upgrade python-swiftclient
</pre>
安装成功后也会有一个命令行工具，命令行swift。命令的格式为。

<pre>
~]# swift [-A auth url] [-U username] [-K password] subcommond
</pre>
另外Swfit接口与S3不同，Swift使用子账号访问，子账号隶属于某一个存在的RadosGW账号。子账号创建方法如下。

<pre>
~]$ radosgw-admin subuser create --uid=test --subuser=test:swifttest --access=full
</pre>
  * --uid：指定RadosGW所存在的账号。
  * --subuser：指定子账号，格式为<radosgw user>:<sub user>。
  * --full：指定权限，full代表完全控制。

创建完成后同样会给子账号也生成一个secret_key。
## RadosGW负载均衡+高可用
在RadosGW前面加一层代理，同时还可以使用使用Keepalived做高可用，代理后端地址为RadosGW主机的IP+PORT，可以起多个RadosGW进程在不同的主机，这样便实现了负载均衡+高可用。
这时候HTTPS证书需要配置在前端代理，在使用s3cmd --configure的时候在“HTTP Proxy server name”这里需要填写代理服务器访问的域名。
在ceph.conf配置文件内不需要指定rgw_dns_name选项，配置为HTTP即可。

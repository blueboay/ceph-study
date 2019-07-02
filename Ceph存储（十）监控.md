# Ceph存储（十）监控
---
## 启用Dashboard面板
默认新版本Ceph集群已集成一个Dashboard，并作为一个模块在manager组件里面，只需要启动这个模块，配置其监听的地址与端口，最后创建一个用户即可登录。
1.  启用模块。

<pre>
~]$ ceph mgr module enable dashboard
</pre>
2.  生成证书。

<pre>
~]$ openssl req -new -nodes -x509 -subj "/CN=dashboard.gogen.cn" -days 3650 -keyout dashboard.key -out dashboard.crt -extensions v3_ca
</pre>
上面生成的是自签证书，如果有自己的CA服务器，可以通过CA颁发证书。也可以使用可信任购买的证书，只要满足需求即可。
3.  导入证书。

<pre>
~]$ ceph config-key set mgr mgr/dashboard/crt -i dashboard.crt
~]$ ceph config-key set mgr mgr/dashboard/key -i dashboard.key
</pre>
4.   配置监听的端口与地址。

<pre>
~]$ ceph config set mgr mgr/dashboard/server_addr 0.0.0.0
~]$ ceph config set mgr mgr/dashboard/server_port 8443
</pre>
5.  创建用户。

<pre>
~]$ ceph dashboard set-login-credentials admin 123456
</pre>
用户名为admin，密码为123456。

6.   登录，访问任何一个mgr节点上面的8443即可，如图所示。

[![](http://121.43.168.35/wp-content/uploads/2019/05/M1LKCTS4YSOKZMUV@J-1024x557.png)](https://www.linux-note.cn/wp-content/uploads/2019/05/M1LKCTS4YSOKZMUV@J.png)
## 启用Prometheus监控接口
Ceph集群已集成Prometheus监控Clientlib，集成于mgr组件，模块名称为prometheus，只需要开启即可，默认监听9283号端口。
启用模块。

<pre>
~]$ ceph mgr module enable prometheus
</pre>
通过访问任何一个mgr节点的此端口即可获取到数据，如。

[![](http://121.43.168.35/wp-content/uploads/2019/05/1-3.png)](https://www.linux-note.cn/wp-content/uploads/2019/05/1-3.png)

# heartbeat
以httpd为例实现高可用集群
## 1 测试服务（http服务器为例）
### 1.1 安装http服务器
在主服务器上安装http

```
yum install httpd
```

添加开启启动

```
systemctl start httpd.service
systemctl enable httpd.service
```

测试
执行curl命令能请求到网页内容时，说明服务部署成功。

```
curl 192.168.6.192 (或 curl 127.0.0.1)
```

开发80端口，供远程访问 

```
firewall-cmd --zone=public --add-port=80/tcp --permanent
systemctl restart firewalld.service
```
打开浏览器，输入`http://192.168.6.192`打开如下所示页面时，说明远程访问配置完成。
<center> ![HP](Images/http-test-website.png)</center> 

创建测试页面
在`/var/www/html/`目录下创建`index.html`页面,输入内容`Hello Host-A`，刷新浏览器，显示如下，则说明测试页面成功运行：
<center> ![HP](Images/http-host-website.png)</center>
同理，按照上述方式在备用服务器上部署httpd服务用于测试。

测试正常后关闭httpd服务并关闭自启动。
```
systemctl disable httpd.service
systemctl stop httpd.service
```
## 2 安装heartbeat模块
用 root 用户操作
### 2.1 环境说明
```
节点1：node1 node1.creation.com 192.168.6.192
节点2：node2 node2.creation.com 192.168.6.193
vip地址：node2.creation.com 192.168.6.190
```
### 2.2 前提条件
在某个节点上做一下配置：
- 关闭firewalld
- 关闭selinux
- 同步时间
- 配置主机名
- 配置主机间ssh互信，免密钥认证
以下配置节点1（192）
```
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
sed -i s#SELINUX=enforcing#SELINUX=disabled# /etc/selinux/config
ntpdate 10.0.0.100
hostname node1.creation.com
echo "node1.creation.com" >> /etc/hostname
ssh-keygen -q -t rsa -N '' -f ~/.ssh/id_rsa
ssh-copy-id root@192.168.6.193
echo -e "192.168.6.192 node1.creation.com node1\n192.168.6.193 node2.creation.com node2" >> /etc/hosts

```
以下配置节点2（193）
```
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
sed -i s#SELINUX=enforcing#SELINUX=disabled# /etc/selinux/config
ntpdate 10.0.0.100
hostname node2.creation.com
echo "node2.creation.com" >> /etc/hostname
ssh-keygen -q -t rsa -N '' -f ~/.ssh/id_rsa
ssh-copy-id root@192.168.6.192
echo -e "192.168.6.192 node1.creation.com node1\n192.168.6.193 node2.creation.com node2" >> /etc/hosts

```
### 2.3 开始安装
安装基础环境
```
yum install gcc gcc-c++ autoconf automake libtool glib2-devel libxml2-devel bzip2 bzip2-devel e2fsprogs-devel libxslt-devel libtool-ltdl-devel asciidoc
```
创建用户和组
```
groupadd haclient
useradd -g haclient hacluster
```
下载软件包：`Reusable-Components-glue、resource-agents、heartbeat`
```
wget http://hg.linux-ha.org/heartbeat-STABLE_3_0/archive/958e11be8686.tar.bz2
wget http://hg.linux-ha.org/glue/archive/0a7add1d9996.tar.bz2
wget https://github.com/ClusterLabs/resource-agents/archive/v3.9.6.tar.gz
```
安装glue
```
tar xf 0a7add1d9996.tar.bz2
cd Reusable-Cluster-Components-glue--0a7add1d9996/
./autogen.sh
./configure --prefix=/usr/local/heartbeat --with-daemon-user=hacluster --with-daemon-group=haclient --enable-fatal-warnings=no LIBS='/lib64/libuuid.so.1'
make && make install
echo $?
```
安装Resource Agents
```
tar xf v3.9.6.tar.gz
cd resource-agents-3.9.6/
./autogen.sh 
./configure --prefix=/usr/local/heartbeat --with-daemon-user=hacluster --with-daemon-group=haclient --enable-fatal-warnings=no LIBS='/lib64/libuuid.so.1'
make && make install
echo $?
```

安装HeartBeat
```
tar xf 958e11be8686.tar.bz2
cd Heartbeat-3-0-958e11be8686/
./bootstrap
export CFLAGS="$CFLAGS -I/usr/local/heartbeat/include -L/usr/local/heartbeat/lib"
./configure --prefix=/usr/local/heartbeat --with-daemon-user=hacluster --with-daemon-group=haclient --enable-fatal-warnings=no LIBS='/lib64/libuuid.so.1'
make && make install
echo $?
```

配置网卡支持插件文件
```
mkdir -pv /usr/local/heartbeat/usr/lib/ocf/lib/heartbeat/
cp /usr/lib/ocf/lib/heartbeat/ocf-* /usr/local/heartbeat/usr/lib/ocf/lib/heartbeat/
```
#注意：一般启动时会报错因为 ping和ucast这些配置都需要插件支持 需要将lib64下面的插件软连接到lib目录 才不会抛出异常
```
ln -svf /usr/local/heartbeat/lib64/heartbeat/plugins/RAExec/* /usr/local/heartbeat/lib/heartbeat/plugins/RAExec/
ln -svf /usr/local/heartbeat/lib64/heartbeat/plugins/* /usr/local/heartbeat/lib/heartbeat/plugins/
```
#以上在节点1上安装完成，在节点2上执行以上同样的步骤，此处省略...

#下面开始配置

配置heartbeat
拷贝三个模版配置文件到 /usr/local/heartbeat/etc/ha.d 目录下 
```
cp doc/{ha.cf,haresources,authkeys} /usr/local/heartbeat/etc/ha.d/
```
配置ha.cf配置文件
#该配置文件用于配置 心跳的核心配置
```
vim /usr/local/heartbeat/etc/ha.d/ha.cf
 
debugfile /var/log/ha-debug  #表示调试的日志文件 一般测试建议开启
logfile /var/log/ha-log  #表示系统的的日志文件路径
logfacility     local0  #表示使用系统日志与上面只能开启一个
keepalive 2  #主备之间的心跳间隔时间单位:s
deadtime 30  #表示如果连接对方30s还无法连接，表示节点死亡需要考虑vip转移
warntime 10  #表示10s时间未收到心跳时发出警告日志
initdead 120  #有时机器启动后需要一段时间网卡才能正常工作 需要预留一定的时间后，再开始判断心跳检测
udpport 694  #多播的udp端口
#baud   19200  #串行端口的波特率
#serial /dev/ttyS0      # Linux  #串口的接口名
#serial /dev/cuaa0      # FreeBSD
#serial /dev/cuad0      # FreeBSD 6.x
#serial /dev/cua/a      # Solaris
#bcast  eth0            # Linux #传播心跳的广播网卡信息
#bcast  eth1 eth2       # Linux
#bcast  le0             # Solaris
#bcast  le1 le2         # Solaris
#mcast eth0 225.0.0.1 694 1 0  #多播传送心跳的网卡 多播组 端口 跃点数 是否回环内传送
ucast ens33 192.168.6.150 #设置单播心跳，设置对方的ip地址,此处使用单播
auto_failback on  #表示如果主机停止后，从机接管设置为on当主机从新启动后，主机立即接管vip off从机不会释放vip给主机
node    node1.pjy.com  #配置主从的节点信息，要与uname -n保持一致
node    node2.pjy.com
#############################################
#使用ping模式 有时当主机挂掉或者heartbeat挂掉后vip才会转移  有时出现某个进程挂掉 切换需要使用脚本 
#ping模式用于测试 如果网卡ping不同 某个主机 就认为当前断网 需要转移vip 
#respawn root     /usr/local/heartbeat/libexec/heartbeat/ipfail 表示当ping不通时 自动调用 ipfail这个脚本
#apiauth ipfail gid=haclient uid=hacluster 表示有权限操作ipfail脚本的组和用户
############################################
ping 192.168.6.1
#ping组的所有主机
#ping_group group1 10.10.10.254 10.10.10.253
#respawn userid /path/name/to/run
#指定与heartbeat一同启动和关闭的进程，该进程被自动监视，遇到故障则重新启动。最常用的进程是ipfail，该进程用于检测和处理网络故障，需要配合ping语句指定的ping node来检测网络连接。如果你的系统是64bit，请注意该文件的路径。
#respawn hacluster /usr/local/heartbeat/libexec/heartbeat/ipfail
#apiauth ipfail gid=haclient uid=hacluster
```
配置authkeys配置文件
#该文件表示发送心跳时 机器用于验证的key的hash算法，节点之间必须配置成一致的密码
```
vim /usr/local/heartbeat/etc/ha.d/authkeys
```
```
auth 2  #表示使用id为2的验证 下边需要定义一个2的验证算法 
2 sha1 1a2b3c  #ID 2的验证加密为shal,并添加密码
```
#更改权限为600
```
chmod 600 /usr/local/heartbeat/etc/ha.d/authkeys
```
配置haresources配置文件
#该文件表示资源的管理，如果是主机，当主机启动后自动加载该文件中配置的所有启动资源，资源脚本默认在haresources同级目录下的resource.d目录下
```
vim /usr/local/heartbeat/etc/ha.d/haresources
#指定节点主机名，和VIP地址，以双冒号分隔资源，此处以apache为例进行配置
node1.creaion.com  192.168.6.190 apache::/etc/httpd/conf/httpd.conf
```
节点2上准备配置文件

#拷贝三个配置好的文件到节点2上，只需修改ha.cf配置文件中的单播地址为对方地址即可(ucast ens33 192.168.6.192)。
```
scp authkeys ha.cf haresources root@node2:/usr/local/heartbeat/etc/ha.d/
```
启动服务
#启动每个节点上heartbeat服务
```
systemctl enable heartbeat
systemctl start heartbeat
ssh node2 'systemctl start heartbeat'
```
测试VIP可用性
此时查看网络配置情况，可以看到enp5s0:0配置出现，实现了资源转移。
`<center> ![HP](Images/heartbeat-vip-ficonfig.png)</center>`
且VIP可以联通
`<center> ![HP](Images/heartbeat-vip-ficonfig.png)</center>`
```
# curl http://192.168.6.190
Hello Host-B
#使用heartbeat自带脚本切换主备节点
# /usr/local/heartbeat/share/heartbeat/hb_standby
Going standby [all].
# curl http://192.168.6.190
Hello Host-A
```

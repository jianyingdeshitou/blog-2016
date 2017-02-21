
# Docker基础与高级

---

来源：http://17173ops.com/2014/10/13/docker%E5%9F%BA%E7%A1%80%E4%B8%8E%E9%AB%98%E7%BA%A7.shtml

陈承 (cyent@163.com, QQ:57237382)

---

## 目录

* 1.Docker安装

* 2.devicemapper

* 3.玩转net namespace
	* 3.1. ENV（环境变量）

* 4.port map

* 5.直接使用docker默认分配的IP对外提供服务（测试中）
	* 5.1. 使用参数以及将docker0的ip配为机房内网网段

* 6.Docker COMMAND
	* 6.1. docker参数
	* 6.2. run

* 7.搭建私有Registry注册中心
	* 7.1. 下载软件
	* 7.2. 启动服务
	* 7.3. 使用
	* 7.4. 套一层透明代理（不推荐，有bug）
	* 7.5. Web UI

* 8.docker with HTTPS
	* 8.1. 原理
	* 8.2. 使用
		* 8.2.1. 创建CA（证书颁发中心）
		* 8.2.2. 创建服务端公钥和私钥
		* 8.2.3. 创建客户端公钥和私钥
		* 8.2.4. 移除服务端私钥、客户端私钥密码
		* 8.2.5. 使用
		* 8.2.6. 管理

* 9.Docker Web-UI(shipyard)
	* 9.1. 工作原理
	* 9.2. server配置
		* 9.2.1. 下载镜像
		* 9.2.2. 启动容器（自动完成部署）
		* 9.2.3. 验证
	* 9.3. agent配置
		* 9.3.1. 下载镜像
		* 9.3.2. 启动容器（自动注册到server）
	* 9.4. 页面配置
		* 9.4.1. 登录页面
		* 9.4.2. 接受agent注册
	* 9.5. 注意
		* 9.5.1. 页面上的Images(http://192.168.1.1:8000/images/)进行镜像删除要注意
		* 9.5.2. server端管理
		* 9.5.3. 不建议生产使用，可作为学习借鉴

* 10.镜像制作
	* 10.1. 远程编译Dockerfile

* 11.内置bridge（nat）和自建网络桥接使用区别

* 12.Docker Event事件监听
	* 12.1. 方法1：使用remote api
	* 12.2. 方法2：使用unix socket
	* 12.3. 方法3: 使用docker events命令

* 13.神器
	* 13.1. nsenter（无需sshd、无需attach也可以登录容器）

* 14.FAQ
	* 14.1. sshd服务起不来
	* 14.2. ulimit无法更改open-file、max processes
	* 14.3. 改变/var/lib/docker路径
	* 14.4. 将指定镜像标识为latest
	* 14.5. docker push报错
		* 14.5.1. HTTP code 403 while uploading metadata: invalid character ‘<‘ looking for beginning of value
		* 14.5.2. dial tcp 127.0.0.1:5000: connection refused
	* 14.6. CMD 和 ENTRYPOINT的区别

---

## 1. Docker安装

* 按照官方说明：红帽6、centos均通过epel源，执行yum install docker-io进行docker安装，启动服务是service docker start

* 经测试，红帽6、centos也可以通过在官网上下载编译好的二进制文件到本地也可以使用，但需要提前手动执行service cgconfig start来挂载cgroup，然后./docker-latest -d来启动服务。

```
下载地址：https://get.docker.io/builds/Linux/x86_64/docker-latest

https://get.docker.io/builds/Linux/x86_64/docker-1.0.1
```

但是官方提示需要内核大于3.8版本，否则可能会有问题。el>3.8

## 2. devicemapper


* 扩容存储池大小、扩容容器文件系统大小

```
https://www.dockboard.org/resizing-docker-containers-with-the-device-mapper-plugin/

实验成功，但是一旦容器关闭再启动，就会报错，还得根据文档再做一次dmsetup load; dmsetup resume才能成功启动容器（但是如果不先启动容器，就无法使用dmsetup命令来resume），因此能否用于生产环境有待继续探索
```

## 3. 玩转net namespace

* 首先要支持ip netns指令。而红帽6及epel均不支持，解决方法：

```
yum install -y http://rdo.fedorapeople.org/rdo-release.rpm
yum update -y iproute
```

* ip netns

```
直接执行这个命令（或ip netns list）读取的是/var/run/netns下的文件名，因此若不存在/var/run/netns，需要mkdir -p /var/run/netns
```

* 配置像LXC一样的网络

```
I. 宿主配置
  1. 宿主上升级iproute包，以便支持ip netns指令：
    yum install -y http://rdo.fedorapeople.org/rdo-release.rpm
    yum update -y iproute

  2. 在宿主上配置好桥接：
    一. 方法1（不推荐）: 敲命令配置桥接（很容易导致网络中断，需要ILO连上操作）
      1) 创建桥接网卡br1并激活：brctl addbr br1; ip link set br1 up
      2) 配置br1的mac地址，和宿主准备桥接的网卡mac相同，通常为内网网卡eth1：ip link set br1 address xx:xx:xx:xx:xx:xx
      3) 给br1配置一个ip地址，或者将eth1的ip地址配置在br1上，2种方法任选其一都可行：
       前者：
       ifconfig br1 192.168.2.1 netmask 255.255.255.0
       后者：
       ifconfig eth1 0.0.0.0; ifconfig br1 192.168.2.2 netmask 255.255.255.0
      4) 配置宿主网关，从br1出
       ip ro del default
       ip ro add default via 192.168.2.254 dev br1
      5) 将eth1桥接至br1：
       brctl addif br1 eth1
    二. 方法2（推荐）：写网卡配置文件
      ifcfg-br1：
		DEVICE="br1"
		TYPE="Bridge"
		NM_CONTROLLED="no"
		ONBOOT="yes"
		BOOTPROTO="static"
		IPADDR=192.168.2.2
		NETMASK=255.255.255.0

      ifcfg-eth1：
		DEVICE="eth1"
		BRIDGE="br1"
		BOOTPROTO="none"
		NM_CONTROLLED="no"
		ONBOOT="yes"
		TYPE="Ethernet"

      注意：要在/etc/sysconfig/network-scripts/ifup-eth里if [ "${TYPE}" = "Bridge" ]; then -> fi段落最后（fi前）加个ip link set br1 address $(get_hwaddr eth1)，防止桥接网卡mac地址随机生成导致网络暂时中断

    service network restart		# 重启网络生效

II. 容器配置：
  1. 启动docker容器：
       docker run -t -i -d --name="net_test" --net=none centos:latest /bin/bash
       记录下输出（即CONTAINER ID），然后通过docker inspect -f '{{.State.Pid}}' CONTAINER ID获得该容器的pid（也即容器首进程在宿主上的pid），假设为1000
  2. 为容器创建网卡命名空间，建立点对点连接（容器命名空间网卡和宿主上生成的网卡点对点）
       mkdir -p /var/run/netns		#创建网络命名空间目录，ip netns会读取该目录下的文件名
       ln -s /proc/1000/ns/net /var/run/netns/1000		#将网络命名空间文件软链接到/var/run/netns，以便ip netns能够读取
       ip link add vethA type veth peer name vethB		#在宿主上创建2张直连网卡（vethA与vethB直连），将vethA作为容器里的网卡，vethB作为宿主上能看到的网卡
       ip link set vethB up			# 激活网卡vethB
       ip link set vethA netns 1000		# 将刚才创建的网卡归位网络命名空间
       配置vethA网卡参数：
         ip netns exec 1000 ip link set vethA name eth1
         ip netns exec 1000 ip addr add 192.168.2.3/24 dev eth1
         ip netns exec 1000 ip link set eth1 up
         ip netns exec 1000 ip route add default via 192.168.2.254 dev eth1
       brctl addif br1 vethB			# 将eth1桥接至br1
  3. 测试：
      docker attach登录容器，查看是否能ping通网关及其他子网或公网
```

### 3.1. ENV（环境变量）

* Dockerfile支持ENV参数，表示启动容器时候设置变量。

只在CMD启动的进程export设置变量，而不是将变量赋值命令写入/etc/profile等脚本里，因此通过ssh方式登录容器获得的shell是没有这个变量的

## 4. port map

* docker支持端口映射，通过iptables DNAT将宿主上的端口转发至容器ip对应端口。

虽然配置了端口映射后，在宿主上通过netstat -lntpu可以看到docker进程会监听这个端口，但是还没发现其作用，因为流量直接从iptables就进入容器里。

* docker run -p、docker run -P、docker port作用：

```
docker run -P 就是将image定好的port给做个端口映射（若没指定-p，则外部端口随机）
docker run -p "8080:80" 启动容器时候做端口映射：宿主的0.0.0.0:8080 -> 容器80
docker run -P -p "8080:80" 假如image已经有一个port 22的配置，那么就会做2个端口映射：宿主0.0.0.0:xxxxx -> 容器22、宿主0.0.0.0:8080 -> 容器80
docker port 是查看容器已经做了端口映射的端口被映射到了哪个端口上，其实直接用docker ps就能看到，使用docker port可能是为了方便二次开发
```

## 5. 直接使用docker默认分配的IP对外提供服务（测试中）

### 5.1. 使用参数以及将docker0的ip配为机房内网网段

* 将宿主eth1桥接到docker0上，将docker0的ip更改为原来eth1的ip（机房内网网段）

存在一个问题：docker run时候分配的ip是从1开始，到254。因此存在和网关或者其他机器ip冲突的可能，无法避免。因此docker分配ip不会做ping检查是否存活

docker run分配出去的ip，docker kill并且docker rm都不会收回并重新使用，而是重启docker daemon才会将ip收回

* –iptables=false

```
使用这个参数后，就不会再往iptables里生成nat、forward等信息了。

这样启动的容器，登录容器能看到网关是宿主docker0的ip，这样网络是通的，是可以访问外网，但路是这么走的：
1. 容器里的数据包将数据经过point-to-point网卡传送到宿主的对应veth网卡上
2. 宿主veth网卡接收到数据包后发现网关是254，于是通过icmp数据包告知网关是254，然后容器发送数据包时自动将网关更改为254，可以从ping的输出看到：
[ 17:37:23-root@21e77bf38fc0:~ ]#ping www.baidu.com
PING www.a.shifen.com (115.239.210.27) 56(84) bytes of data.
64 bytes from 115.239.210.27: icmp_seq=1 ttl=55 time=13.9 ms
From 192.168.3.1: icmp_seq=2 Redirect Host(New nexthop: 192.168.3.254)
64 bytes from 115.239.210.27: icmp_seq=2 ttl=55 time=13.6 ms
From 192.168.3.1: icmp_seq=3 Redirect Host(New nexthop: 192.168.3.254)
64 bytes from 115.239.210.27: icmp_seq=3 ttl=55 time=13.6 ms
From 192.168.3.1: icmp_seq=4 Redirect Host(New nexthop: 192.168.3.254)
64 bytes from 115.239.210.27: icmp_seq=4 ttl=55 time=14.0 ms
From 192.168.3.1: icmp_seq=5 Redirect Host(New nexthop: 192.168.3.254)
64 bytes from 115.239.210.27: icmp_seq=5 ttl=55 time=14.7 ms
From 192.168.3.1: icmp_seq=6 Redirect Host(New nexthop: 192.168.3.254)
64 bytes from 115.239.210.27: icmp_seq=6 ttl=55 time=13.8 ms
64 bytes from 115.239.210.27: icmp_seq=7 ttl=55 time=13.7 ms
From192.168.3.1: icmp_seq=8 Redirect Host(New nexthop: 192.168.3.254)
64 bytes from 115.239.210.27: icmp_seq=8 ttl=55 time=13.8 ms
64 bytes from 115.239.210.27: icmp_seq=9 ttl=55 time=13.6 ms
64 bytes from 115.239.210.27: icmp_seq=10 ttl=55 time=13.5 ms
From 192.168.3.1: icmp_seq=11 Redirect Host(New nexthop: 192.168.3.254)
64 bytes from 115.239.210.27: icmp_seq=11 ttl=55 time=13.8 ms
64 bytes from 115.239.210.27: icmp_seq=12 ttl=55 time=14.1 ms
64 bytes from 115.239.210.27: icmp_seq=13 ttl=55 time=13.8 ms
64 bytes from 115.239.210.27: icmp_seq=14 ttl=55 time=13.6 ms
64 bytes from 115.239.210.27: icmp_seq=15 ttl=55 time=13.7 ms
64 bytes from 115.239.210.27: icmp_seq=16 ttl=55 time=13.8 ms
From 192.168.3.1: icmp_seq=17 Redirect Host(New nexthop: 192.168.3.254)
64 bytes from 115.239.210.27: icmp_seq=17 ttl=55 time=13.6 ms
64 bytes from 115.239.210.27: icmp_seq=18 ttl=55 time=13.8 ms
64 bytes from 115.239.210.27: icmp_seq=19 ttl=55 time=13.7 ms
64 bytes from 115.239.210.27: icmp_seq=20 ttl=55 time=14.1 ms
64 bytes from 115.239.210.27: icmp_seq=21 ttl=55 time=13.7 ms
64 bytes from 115.239.210.27: icmp_seq=22 ttl=55 time=13.8 ms
64 bytes from 115.239.210.27: icmp_seq=23 ttl=55 time=13.7 ms
64 bytes from 115.239.210.27: icmp_seq=24 ttl=55 time=13.9 ms
64 bytes from 115.239.210.27: icmp_seq=25 ttl=55 time=14.2 ms
64 bytes from 115.239.210.27: icmp_seq=26 ttl=55 time=13.7 ms
64 bytes from 115.239.210.27: icmp_seq=27 ttl=55 time=13.7 ms
64 bytes from 115.239.210.27: icmp_seq=28 ttl=55 time=13.8 ms
64 bytes from 115.239.210.27: icmp_seq=29 ttl=55 time=13.9 ms
64 bytes from 115.239.210.27: icmp_seq=30 ttl=55 time=14.1 ms
64 bytes from 115.239.210.27: icmp_seq=31 ttl=55 time=13.9 ms
64 bytes from 115.239.210.27: icmp_seq=32 ttl=55 time=13.9 ms
64 bytes from 115.239.210.27: icmp_seq=33 ttl=55 time=14.0 ms
64 bytes from 115.239.210.27: icmp_seq=34 ttl=55 time=13.7 ms
64 bytes from 115.239.210.27: icmp_seq=35 ttl=55 time=14.0 ms
64 bytes from 115.239.210.27: icmp_seq=36 ttl=55 time=14.4 ms
64 bytes from 115.239.210.27: icmp_seq=37 ttl=55 time=13.9 ms
```

docker服务启动时候会把内核参数ip.forward给打开（数据包转发）

## 6. Docker COMMAND

### 6.1. docker参数


* –api-enable-cors

```
开启cors，以便浏览器能够通过ajax调用。但是若开启了tls，使用cors就变得困难了，目前网络上还未找到解决方案
```

### 6.2. run

* –link：2个容器互通

```
其实就做3件事：
1. 若有端口映射，则在iptables的FORWARD链里将端口ACCEPT
2. /etc/hosts：做link的容器的/etc/hosts能看到被link的容器的hosts条目
3. 环境变量：做link的容器可以看到被link的容器的环境变量（仅为--env变量），如：ALIAS_ENV_变量名、ALIAS_NAME=xxx
```

* –volume: 目录共享

```
支持2种模式：
1. 从宿主挂载：-v /tmp:/tmp/foo 表示将宿主的/tmp目录挂载至容器的/tmp/foo目录，可读可写，和mount --bind的效果类似
2. 容器之间共享：
	启动第一个容器时带参数-v /tmp/foo表示在宿主上创建/var/lib/docker/vfs/dir/xxxxx（id，但不是容器id），然后挂进容器的/tmp/foo目录;
	启动第二个容器时带参数--volumes-from=b5f8320cf019（*第一个容器id）表示和第一个容器共享挂载，因此第二个容器启动后也能从df -h看到/tmp/foo目录被挂载。从inspect也可以很容易看出来（2个容器的inspect以下内容相同）：
	"Volumes": {
		"/opt": "/var/lib/docker/vfs/dir/a7b1b03773d9391718b8524e7ac001bb877eb6d0596fa2a4328435d8c49f2415"
	},
	"VolumesRW": {
		"/opt": true
	}
```

## 7. 搭建私有Registry注册中心

### 7.1. 下载软件

* 安装pip：

```
yum install python-devel libevent-devel python-pip gcc xz-devel
```

* 安装registry：

```
pip install docker-registry
pip install docker-registry[BUGSNAG]
pip install -i http://pypi.douban.com/simple/ backports.lzma
pip install --upgrade -i http://pypi.douban.com/simple/ backports.lzma
```

### 7.2. 启动服务

* 使用默认配置文件启动服务：

```
cd /usr/lib/python2.6/site-packages/config
cp config_sample.yml config.yml
mkdir /tmp/registry                 # config_sample.yml的默认配置
gunicorn --access-logfile - --debug -k gevent -b 0.0.0.0:5000 -w 1 docker_registry.wsgi:application
```

* 加入开机启动

```
可以将以下内容写入/etc/rc.local，或者放在一个脚本里，/etc/rc.local调用这个脚本
pkill gunicorn
sleep 1
rm -f /data/docker-registry.db
sleep 1
/usr/bin/gunicorn --access-logfile /opt/logs/docker-registry/access.log --error-logfile /opt/logs/docker-registry/error.log --daemon --debug -k gevent -b 0.0.0.0:5000 -w 8 docker_registry.wsgi:application
sleep 3
pkill gunicorn
sleep 1
/usr/bin/gunicorn --access-logfile /opt/logs/docker-registry/access.log --error-logfile /opt/logs/docker-registry/error.log --daemon --debug -k gevent -b 0.0.0.0:5000 -w 8 docker_registry.wsgi:application
```

* 启动不成功FAQ

```
1. 日志目录没有创建
2. .db文件已存在，启动前删掉就行
排错可通过error.log分析
```

### 7.3. 使用

* 上传镜像

```
docker tag busybox localhost:5000/busybox
docker push localhost:5000/busybox

上传成功后在/tmp/registry上应该能看到2个目录：images和repositories
```

* 下载镜像

```
docker pull localhost:5000/busybox

若想给下载的镜像取个别名：
docker tag localhost:5000/busybox aliasname 或 docker tag id aliasname  #id为localhost:5000/busybox的image id

删除别名：
docker rmi aliasname
当然也可以选择把原始名字删掉：
docker rmi localhost:5000/busybox
```

若一个镜像有至少2个tag，那么通过docker rmi删除的只是别名，若只有1个名字（没有别名），那么删除的是真正的镜像。

已测试：当镜像还在上传过程中时，其他机器是无法pull下来的。因此不会生成不完整的镜像

* 搜索镜像

```
curl --silent "http://192.168.1.1:5000/v1/search?q="  | json_reformat
```

### 7.4. 套一层透明代理（不推荐，有bug）

* 配置nginx，vhost内容如下

```
upstream docker-registry {
      server 127.0.0.1:5000;
}

server {
      listen 192.168.1.1:80;
      server_name registry.17173ops.com;

      proxy_set_header Host       $host;
      proxy_set_header X-Real-IP  $remote_addr;

      client_max_body_size 0; # disable any limits to avoid HTTP 413 for large image uploads

      # required to avoid HTTP 411: see Issue #1486 (https://github.com/dotcloud/docker/issues/1486)
      chunked_transfer_encoding on;

      location / {
            proxy_pass http://docker-registry;
      }
}
```

* 注意：只有以下2种情况，套一层透明代理是行的通的：

```
1. nginx监听80端口，gunicorn监听0.0.0.0
2. nginx监听非80端口，且gunicorn监听在网络可通的ip，如192.168.1.1这样的内网ip，而不能监听在127.0.0.1

测试发现，当端口不为80时，在docker push时候，会首先连接image name中的ip、端口，如registry.17173ops.com:80，然后连接实际的gunicorn端口。因此当gunicorn端口无法访问时，就会报错。
而端口为80时候就不会这样，原因未知，应该是代码逻辑。
<=1.1.2版本都有如上共性，>1.1.2的尚未测试。
```

### 7.5. Web UI

* 下载

```
docker pull atcol/docker-registry-ui
```

* 启动

```
docker run --name "registry_UI" -tid -p 127.0.0.1:5001:8080 -e REG1=http://registry.17173ops.com/v1/ atcol/docker-registry-ui:latest
```

* 开机启动：

```
echo 'docker start registry_UI' >> /etc/rc.local
```

* 套一层透明代理

```
配置nginx，vhost内容如下
upstream docker-registry-web {
      server 127.0.0.1:5001;
}

server {
      listen 192.168.1.1:80;
      server_name registry-web.17173ops.com;

      location / {
            proxy_pass http://docker-registry-web;
      }
}
```

* 使用

```
http://registry-web.17173ops.com/
```

* 注意：页面上的删除镜像只是删除tag标签，实际id和image不会删除

```
docker镜像的元数据里记录着FROM哪个镜像，如果真的删除镜像所有数据，正常逻辑应该是A->B B->C -C->D D->E，这样一连串都给删除了，但是其中的某个镜像可能又被其他镜像给依赖了，猜测正是基于这种逻辑，才只删除了tag
```

## 8. docker with HTTPS

### 8.1. 原理

* Docker HTTPS原理：双向验证。官方说明（https://docs.docker.com/articles/https/ ）：

```
In daemon mode, it will only allow connections from clients authenticated by a certificate signed by that CA.
In client mode, it will only connect to servers with a certificate signed by that CA.
核心：服务端和客户端的数字证书都由同一个CA签发，因此双方在认证通讯时使用和签发时的同一个CA就能互相认证。
```

### 8.2. 使用

#### 8.2.1. 创建CA（证书颁发中心）

* 测试

```
echo 01 > ca.srl
openssl genrsa -des3 -out ca-key.pem 2048
openssl req -new -x509 -days 3650 -key ca-key.pem -out ca.pem
```

#### 8.2.2. 创建服务端公钥和私钥

* 生成私钥：

```
openssl genrsa -des3 -out server-key.pem 2048
```

* 生成公钥（数字签名证书）：

```
1. 生成CSR文件（Certificate Signing Request 证书签名请求）：openssl req -subj '/CN=docker.17173ops.com' -new -key server-key.pem -out server.csr
2. 编写openssl.conf文件，内容如下：
------------- BEGIN ---------------------------------
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req

[req_distinguished_name]

[ v3_req ]
subjectAltName = @alt_names

[alt_names]
DNS.1 = *.h.173ops.com
DNS.2 = *.docker.17173ops.com
------------- END ------------------------------------
```

* 生成公钥（数字签名证书）：

```
openssl x509 -req -days 3650 -in server.csr -CA ca.pem -CAkey ca-key.pem -out server-cert.pem -extensions v3_req -extfile openssl.conf
```

#### 8.2.3. 创建客户端公钥和私钥

* 生成私钥：

```
openssl genrsa -des3 -out client-key.pem 2048
```

* 生成公钥（数字签名证书）：

```
1. 生成CSR文件（Certificate Signing Request 证书签名请求）：openssl req -subj '/CN=client' -new -key client-key.pem -out client.csr
2. echo extendedKeyUsage = clientAuth > extfile.cnf
3. 生成公钥（数字签名证书）：openssl x509 -req -days 3650 -in client.csr -CA ca.pem -CAkey ca-key.pem -out client-cert.pem -extfile extfile.cnf
```

#### 8.2.4. 移除服务端私钥、客户端私钥密码

* 服务端

```
openssl rsa -in server-key.pem -out server-key.pem
```

* 客户端

```
openssl rsa -in client-key.pem -out client-key.pem
```

#### 8.2.5. 使用

将服务端3个文件拷贝到docker daemon（假设为192.168.1.2）的/root/.docker/下：

```
1. scp ca.pem server-cert.pem server-key.pem 192.168.1.2:/root/.docker/
2. 登录192.168.1.2
	cd /root/.docker/
	chmod 0600 *
3. 添加启动参数：
	修改/etc/sysconfig/docker中的other_args值，添加以下内容，如：other_args="--graph /opt/docker --tlsverify --tlscacert=/root/.docker/ca.pem --tlscert=/root/.docker/server-cert.pem --tlskey=/root/.docker/server-key.pem -H unix:///var/run/docker.sock -H tcp://192.168.1.2:2376 -H tcp://127.0.0.1:2376"
```

* 将客户端3个文件拷贝至”中控机”，然后就可以通过HTTPS远程对docker daemon进行操作：

```
docker --tlsverify --tlscacert=/opt/docker_tls/ca.pem --tlscert=/opt/docker_tls/client-cert.pem --tlskey=/opt/docker_tls/client-key.pem -H localhost:2376 images
```

#### 8.2.6. 管理

* 查看CSR文件：

```
openssl req -noout -text -in server.csr
```

* 查看签名证书（server-cert.pem、client-cert.pem）：

```
openssl x509 -noout -text -in server-cert.pem
```

## 9. Docker Web-UI(shipyard)

### 9.1. 工作原理

* 在每台docker宿主机上启动一个容器（shipyard/agent），这个容器通过挂载宿主的/var/run/docker.sock文件来获取该宿主上容器、镜像的信息，同时在容器启动时将agent注册到管理中心（shipyard/deploy）上，实现从Web查看和操作docker容器与镜像

* 每个docker宿主启动一个agent（shipyard/agent），管理中心启动一个server（shipyard/deploy）

### 9.2. server配置

#### 9.2.1. 下载镜像

```
docker pull shipyard/deploy
```

#### 9.2.2. 启动容器（自动完成部署）

* docker run -i -t -v /var/run/docker.sock:/docker.sock shipyard/deploy setup

```
执行该命令时，实际做了如下操作：
在本地宿主上依次下载并启动
1. shipyard/redis
2. shipyard/router
3. shipyard/lb（Load Balance的意思）
4. shipyard/db
5. shipyard/shipyard（Web-UI）
```

如何做到上述的：docker run -i -t -v /var/run/docker.sock:/docker.sock shipyard/deploy setup会将socket文件挂载进容器里，这样在容器里就能对宿主的镜像和容器进行管理，而shipyard/deploy的CMD是一个脚本，这个脚本就是进行上述操作

#### 9.2.3. 验证

* docker ps查看是否都已启动

注意docker logs shipyard_router可能会看到大量报错，并且占用大量CPU，负载也会升高。暂时不知原因，已提交至github.com，等待答复，详见https://github.com/shipyard/docker-shipyard-router/issues/3

### 9.3. agent配置

#### 9.3.1. 下载镜像

* docker pull shipyard/agent

#### 9.3.2. 启动容器（自动注册到server）


* docker run -i -t -v /var/run/docker.sock:/docker.sock -e IP=`ip -4 address show br1 | grep ‘inet ‘ | sed ‘s/.*inet \([0-9\.]\+\).*/\1/’` -e URL=http://192.168.1.1:8000 -p 4500:4500 shipyard/agent

```
如果看不懂，可以这么写：
docker run -i -t -v /var/run/docker.sock:/docker.sock -e IP=192.168.1.2 -e URL=http://192.168.1.1:8000 -p 4500:4500 shipyard/agent
（192.168.1.2是docker宿主ip）

注意：要先启动server，才能启动agent，否则agent注册到server可能会失败。
```

## 9.4. 页面配置

#### 9.4.1. 登录页面

* http://192.168.1.1:8000

```
默认账号：admin
默认密码：shipyard
```

#### 9.4.2. 接受agent注册

* http://192.168.1.1:8000/hosts/ ，在打开网页的右部分点击按钮，选择authorize host

### 9.5. 注意

#### 9.5.1. 页面上的Images(http://192.168.1.1:8000/images/)进行镜像删除要注意

* 自动进行了去重（根据image id），因此从页面上删除镜像时会将相同image id的全部删除

#### 9.5.2. server端管理

* 移除：docker run -i -t -v /var/run/docker.sock:/docker.sock shipyard/deploy cleanup

* 重启：docker run -i -t -v /var/run/docker.sock:/docker.sock shipyard/deploy restart

* 升级：docker run -i -t -v /var/run/docker.sock:/docker.sock shipyard/deploy upgrade

#### 9.5.3. 不建议生产使用，可作为学习借鉴

* 原因1：封装太多，用户无法定制修改（20140903）

* 原因2：一些细节功能方面，例如不同host的容器打印在一张表里，连排序功能都没有（20140903）

## 10. 镜像制作

### 10.1. 远程编译Dockerfile

* docker -H tcp://xxx:2376 build –force-rm –no-cache -t foo/rhel6.5:1.0 /path/to/ 那么这个/path/to/指的是本地文件，而非远程编译机上的文件。

* /path/to/Dockerfile文件会在命令执行之初就通过远端2376端口将/path/to/*（Dockerfile所在目录下的所有文件）传送到编译机上

```
Sending build context to Docker daemon xx MB过程就是将文件发送过去
```

* Dockerfile除了支持文件方式外，还支持URL，即/path/to/可以改为http://xxx （尚未测试过）

## 11. 内置bridge（nat）和自建网络桥接使用区别

* 内置bridge（nat）

```
优点：
1. 节省IP

缺点：
1. 需要配套服务注册/发现，否则宿主上端口分配困难，容易冲突。
2. 由于每个容器暴露的端口都不一致，造成前端路由层nginx配置（proxy_pass）里无法使用dns的方式。
3. 端口映射要在容器启动时就指定好，后期无法变更。
4. 测试发现nat不支持websocket。
```

* 自建桥接网络

```
优点：
1. 每个容器都有独立ip，对外提供服务，如nginx+php，nginx+resin，都可以使用默认的80端口
2. 由于容器暴露端口都可以使用80端口，因此前端路由层nginx配置（proxy_pass）里可以使用dns的方式。
3. 无需为了后期端口映射添加而烦恼
4. 桥接支持websocket

缺点：
1. 每个容器都需要一个IP（内网ip是不需要钱的）
```

## 12. Docker Event事件监听

### 12.1. 方法1：使用remote api

* 还未找出靠谱的阻塞方式

### 12.2. 方法2：使用unix socket

* 测试发现：docker服务关闭时，/var/run/docker.sock文件并不会自动删掉

```
使用python连接/var/run/docker.sock文件会一直连着，docker daemon停止后，文件仍在，连接也仍在，因此不适用于事件监听。
```

### 12.3. 方法3: 使用docker events命令

* 测试发现：输出内容与api的输出不完全一样，比如docker events输出的时间格式为可读的格式，而api输出的是unix timestamp。

* 可以通过python的模块os.popen(‘docker events’)来建立监听连接

```
	# -*- coding: utf-8 -*-
	import os
	handler = os.popen('docker -H 127.0.0.1:2376 events')
	while True:
		res = handler.readline()
		if res:
			print res
		else:
			print "Null"
			exit(1)
```

## 13. 神器

### 13.1. nsenter（无需sshd、无需attach也可以登录容器）


* 原理：进入namespace（通过/proc/xxxx/ns/）

* 安装：docker run -v /usr/local/bin:/target registry.17173ops.com:5000/jpetazzo/nsenter:latest

```
执行成功后会在宿主的/usr/local/bin下生成1个二进制程序nsenter和1个shell脚本docker-enter

docker-enter是对nsenter用法的封装，让使用更加简单
```

* 使用：docker-enter 容器id(或容器名)

```
docker-enter是一个shell脚本，其实是调用nsenter（二进制文件），因此可以直接使用nsenter：
nsenter --target $PID --mount --uts --ipc --net --pid		# 这个$PID指容器里的任意进程在宿主上的真实PID
```

## 14. FAQ

### 14.1. sshd服务起不来

* docker在源码里就关掉了audit_control（linux Capabilities），而/etc/pam.d/sshd里有一行session required pam_loginuid.so，把这行删掉就可以了

### 14.2. ulimit无法更改open-file、max processes

* docker容器默认移除sys_resource（Linux能力），因而ulimit -n设置只能改小无法改大，改大会报错：ulimit: open files: cannot modify limit: Operation not permitted。

* 红帽7下docker run可以使用–privileged选项来不移除Linux能力，但docker默认移除这个Linux能力肯定是有安全方面的考量，因此尽量别用该选项

红帽6下要使用–privileged，docker版本不能>=1.0.1，否则会报错；stat /dev/.udev/db/cpuid:cpu0: no such file or directory。

* 解决方法：

```
若启动docker使用sysV服务，则在/etc/init.d/functions最开头添加一行：ulimit -u 204800 -HSn 204800

若启动docker使用命令，如docker -d，那么在启动之前先执行ulimit -u 204800 -HSn 204800即可
```

### 14.3. 改变/var/lib/docker路径

* 使用–graph参数：docker –graph=/opt/docker -d，会自动生成/opt/docker目录（0700），并在该目录下创建docker相关文件

原来的镜像和容器都找不到了，因为路径改了（原来的镜像是在/var/lib/docker/devicemapper/devicemapper/{data,metadata}）

### 14.4. 将指定镜像标识为latest

* docker tag 镜像id cyent/rhel6.5:latest

### 14.5. docker push报错

#### 14.5.1. HTTP code 403 while uploading metadata: invalid character ‘<‘ looking for beginning of value

* 报错示例：

```
[ 14:50:44-root@localhost:vhosts ]#docker push registry.17173ops.com:82/crosbymichael/dockerui
The push refers to a repository [registry.17173ops.com:82/crosbymichael/dockerui] (len: 1)
Sending image list
Pushing repository registry.17173ops.com:82/crosbymichael/dockerui (1 tags)
511136ea3c5a: Pushing
2014/09/01 14:50:46 HTTP code 403 while uploading metadata: invalid character '<' looking for beginning of value
```

* 解决方法：在nginx配置里注释掉”proxy_set_header Host $host;”

#### 14.5.2. dial tcp 127.0.0.1:5000: connection refused

* 报错示例：

```
[ 18:24:35-root@localhost:~ ]#docker push registry.17173ops.com:81/17173/as6.5-ng1.4:1.4
The push refers to a repository [registry.17173ops.com:81/17173/as6.5-ng1.4] (len: 1)
Sending image list
Pushing repository registry.17173ops.com:81/17173/as6.5-ng1.4 (1 tags)
511136ea3c5a: Pushing
2014/09/01 18:27:27 Failed to upload metadata: Put http://127.0.0.1:5000/v1/images/511136ea3c5a64f264b78b5433614aec563103b4d4702f3ba7d4d2698e22c158/json: dial tcp 127.0.0.1:5000: connection refused
```

* 解决方法：不要这么用，不要套nginx，gunicorn直接对外使用

### 14.6. CMD 和 ENTRYPOINT的区别

* CMD是可以在docker run时候被覆盖的，而ENTRYPOINT无法被覆盖

* CMD通常被用于调试，可以选择不同的CMD，可支持带参数

* 而ENTRYPOINT通常被用于应用发布，也可支持带参数

* 举例说明：

```
假设CMD和ENTRYPOINT都是/bin/run.sh，run.sh的内容是echo "Hello,$1"
那么，若为CMD，则docker run -t -i xxx/yyy /bin/ls /root就会列出/root目录下的内容
若为ENTRYPOINT，则docker run -t -i xxx/yyy /bin/ls /root就会打印出Hello,/bin/ls
```

* 最重要的区别：ENTRYPOINT是会被继承下去的

```
比如做镜像A的时候Dockerfile里写了一行ENTRYPOINT /usr/local/bin/run.sh，那么之后做的镜像（FROM:）的Dockerfile里如果不指定ENTRYPOINT，而是指定CMD，那么这个CMD虽然从inspect里看是存在的，但却是无效的。
```
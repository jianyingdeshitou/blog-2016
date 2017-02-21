
# 关于Docker的15个提示

> 来源：http://my.oschina.net/zjzhai/blog/213401

## 目录

1. 得到最后执行的容器的ID
1. 尝试在shell制作Docker镜像
1. 去除sudo
1. 删除所有已经停止的容器
1. 转化docker inspect输出
1. 查看镜像内的环境变量有哪些
1. RUN 与 CMD的区别
1. CMD 与ENTRYPOINT的区别
1. 查看Docker容器的IP
1. Docker构架：薄CLI客户端，建立在UNIX socket上的提供REST服务的守护进程
1. 以图像的方式查看你的镜像的依赖
1. Docker的东西到底存在哪里？
1. Docker源码
1. 不要在你的Dockerfile中的RUN指令中执行守护进程
1. 容器之间的通信

---

> 这些我从网上学到的，我只负责搬砖，原文地址：http://sssslide.com/speakerdeck.com/bmorearty/15-docker-tips-in-5-minutes

## 1. 得到最后执行的容器的ID

```shell
$ ID=$(docker run ubuntu echo hello world)
hello world
$ docker commit $ID helloword
```

如果这样觉得麻烦，你可以这样：

```shell
$ alias dl='docker ps -l -q'
$ docker run ubuntu echo hello word
hello world
$ dl
1904cf045887
$ docker commit `dl` helloworld
fd08a884dc79
```

## 2. 尝试在shell制作Docker镜像

```shell
$ docker run -i -t ubuntu bash
root@db0c3978af8:/# apt-get install postgresql

root@db0c3978af8:/# exit

$ docker commit -run='{"Cmd":["postgres","-too -many -opts"]}' `dl` postgres
507611232efc0
```

## 3. 去除sudo

```shell
# 添加docker组
$ sudo groupadd docker

# 将自己添加到组中
$ sudo gpasswd -a myusername docker

# 重启Docker守护进程
$ sudo service docker restart

# 退出，再登录
$ exit
```

## 4. 删除所有已经停止的容器

```shell
$ docker rm $(docker ps -a -q)
```

## 5. 转化docker inspect输出

```shell
$ docker inspect `dl` | grep IPAddress | cut -d '"' -f 4
172.17.0.52

$ docker inspect `dl1` | jq -r '.[0].NetworkSettings.IPAddress'
172.17.0.52
```

## 6. 查看镜像内的环境变量有哪些

```shell
$ docker run ubuntu env
HOME=/
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
container=lxc
HOSTNAME=5e150b7fef22
```

## 7. RUN 与 CMD的区别

```Dockerfile
FROM ubuntu

# 执行docker build时会执行下面这些：
RUN apt-get update
RUN apt-get install softwres

# 执行docker run时会执行默认执行:
CMD ["softwares"]
```

## 8. CMD 与ENTRYPOINT的区别

```shell
$ cat Dockerfile
FROM ubuntu
CMD ["echo"]

$ coker run imagename echo hello
hello

$ cat Dockerfile
FROM ubuntu
ENTRYPOINT ["echo"]

$ coker run imagename echo hello
echo hello
```

## 9. 查看Docker容器的IP

```shell
$ ip -4 -o addr show eth0
2: eth0    inet 10.108.1.107/24 brd 10.108.1.255 scope global eth0

$ docker run ubuntu ip -4 -o addr show eth0
83: eth0    inet 172.17.0.4/16 scope global eth0
```

## 10. Docker构架：薄CLI客户端，建立在UNIX socket上的提供REST服务的守护进程

```shell
#像HTTP客户端一样连接并使用UNIX socket
$ nc -U //var/run/docker.sock GET /images/json HTTP/1.1
```

> 注：我执行这条命令，没见反应。不知道为什么。

## 11. 以图像的方式查看你的镜像的依赖

```shell
$ docker images -viz | dot -Tpng -o docker.png

$ python -m SimpleHTTPServer

#打开浏览器
# http://machinename:8000/docker.png
```

> 注：个人不是很明白这里

## 12. Docker的东西到底存在哪里？

```shell
$ sudo su
# cd /var/lib/docker
# ls -F
containers/ graph/ repositories volumes/
```

graph下存的是镜像，而文件系统存在 `graph/imagesid/layer`

## 13. Docker源码

* commands.go

负责命令行客户端

* api.go

REST API路由

* server.go

一个REST API的实现

* buildfile.go

Dockerfile的解析器

## 14. 不要在你的Dockerfile中的RUN指令中执行守护进程

```shell
$ cat Dockerfile
FROM ubuntu:12.04
MAINTAINER Brian Morearty
..
RUN pg_ctl start ...

$ docker run -i -t postgresimage bash
root@4432fe2dd3:/# ps aux

# Doesn't show postgres daemon
```

> 注：事实上，可以这样执行 `RUN pg_ctl start &`

## 15. 容器之间的通信

```shell
# 执行一个容器，并分配一个名字给它
$ docker run -d -name loldb loldbimage

# 执行第二个容器，并连接上第一个容器，同时使用别名
$ docker run -link /loldb:cheez otherimage env
CHEEZ_PORT=tcp://172.17.0.8:6379
CHEEZ_PORT_1337_TCP=tcp://172.17.0.8:6379
CHEEZ_PORT_1337_TCP_ADDR=172.17.0.12
CHEEZ_PORT_1337_TCP_PORT=6379
CHEEZ_PORT_1337_TCP_PROTO=tcp
```

> 注：我不是很明白显示的那些env是什么意思。

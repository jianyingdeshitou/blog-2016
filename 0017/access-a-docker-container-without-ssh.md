# 不通过SSH接入Docker

> 来源：http://openstack.wiaapp.com/?p=565

> 翻译自：http://www.sebastien-han.fr/blog/2014/01/27/access-a-container-without-ssh/

先运行一个简单的memcache容器：

```shell
$ sudo docker run -d -p 11211 bacongobbler/memcached memcached /usr/bin/memcached -m 64 -p 11211 -u memcache -l 0.0.0.0

$ sudo docker ps
CONTAINER ID        IMAGE                                      COMMAND                CREATED             STATUS              PORTS                      NAMES
0a9856723f90        192.168.0.127:5042/memcached:latest        memcached /usr/bin/m   2 seconds ago       Up 2 seconds        0.0.0.0:49153-&gt;11211/tcp   pensive_pasteur
```

获得运行在docker中进程的pid:

```shell
root@docker:~# ps faux |grep memcached
syslog   29123  0.0  0.0 323216  1184 ?        Sl   22:40   0:00          _ memcached /usr/bin/memcached -m 64 -p 11211 -u memcache -l 0.0.0.0
```

安装`nsenter`命令（`util-linux`包中），需要2.23release的linux。

```shell
$ wget https://www.kernel.org/pub/linux/utils/util-linux/v2.24/util-linux-2.24.tar.bz2
$ bzip2 -d -c util-linux-2.24.tar.bz2 | tar xvf -
$ cd util-linux-2.24/
$ sudo ./configure --without-ncurses
$ make nsenter
$ cp nsenter /usr/local/bin
```

然后，我们连接进入docker容器。

```shell
$ sudo nsenter -m -u -i -n -p -t 29123 /bin/sh
# ps faux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root        11  0.0  0.0   4396   608 ?        S    22:41   0:00 /bin/sh
root        12  0.0  0.0  15272  1100 ?        R+   22:41   0:00  _ ps faux
memcache     1  0.0  0.0 323216  1184 ?        Sl   22:40   0:00 memcached /usr/bin/memcached -m 64 -p 11211 -u memcache -l 0.0.0.0
```

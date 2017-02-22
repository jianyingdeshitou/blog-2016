# 隐藏Apache版本号的必要性与方法

> 来源：http://www.ha97.com/2505.html

一般情况下，软件的漏洞信息和特定版本是相关的，因此，软件的版本号对攻击者来说是很有价值的。

在默认情况下，系统会把Apache版本模块都显示出来（http返回头信息）。如果列举目录的话，会显示域名信息（文件列表正文），如：

```shell
    [root@localhost tmp]# curl -I 192.168.80.128:88
    HTTP/1.1 403 Forbidden
    Date: Wed, 21 Jul 2010 13:09:33 GMT
    Server: Apache/2.2.15 (CentOS)
    Accept-Ranges: bytes
    Content-Length: 5043
    Connection: close
    Content-Type: text/html; charset=UTF-8
```

隐藏方法：

1、隐藏Apache版本号的方法是修改Apache的配置文件，如RedHat系的Linux默认是：

```shell
    vim /etc/httpd/conf/httpd.conf
```

分别搜索关键字ServerTokens和ServerSignature，修改：

```shell
    ServerTokens OS 修改为 ServerTokens ProductOnly
    ServerSignature On 修改为 ServerSignature Off
```

2、重启或重新加载Apache就可以了。

```shell
    apachectl restart
```

测试一下，如下：

```shell
    [root@localhost tmp]# curl -I 192.168.80.128:88
    HTTP/1.1 403 Forbidden
    Date: Wed, 21 Jul 2010 13:23:22 GMT
    Server: Apache
    Accept-Ranges: bytes
    Content-Length: 5043
    Connection: close
    Content-Type: text/html; charset=UTF-8
```

版本号与操作系统信息已经隐藏了。

3、上面的方法是默认情况下安装的Apache，如果是编译安装的，还可以用修改源码编译的方法：

进入Apache的源码目录下的include目录，然后编辑`ap_release.h`这个文件，你会看到有如下变量：

```c
    #define AP_SERVER_BASEVENDOR “Apache Software Foundation”
    #define AP_SERVER_BASEPROJECT “Apache HTTP Server”
    #define AP_SERVER_BASEPRODUCT “Apache”

    #define AP_SERVER_MAJORVERSION_NUMBER 2
    #define AP_SERVER_MINORVERSION_NUMBER 2
    #define AP_SERVER_PATCHLEVEL_NUMBER 15
    #define AP_SERVER_DEVBUILD_BOOLEAN 0
```

可以根据自己喜好，修改或隐藏版本号与名字。

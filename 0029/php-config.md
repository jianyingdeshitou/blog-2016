# 隐藏PHP版本与PHP基本安全设置

> 来源：http://www.ha97.com/2058.html

为了安全起见，最好还是将PHP版本隐藏，以避免一些因PHP版本漏洞而引起的攻击。

## 1、隐藏PHP版本就是隐藏 “X-Powered-By: PHP/5.2.13” 这个信息。

方法很简单：

编辑`php.ini`配置文件，修改或加入: `expose_php = Off` 保存后重新启动Nginx或Apache等相应的Web服务器即可。

```shell
[root@bkjz /]# curl -I www.ha97.com
HTTP/1.1 200 OK
Server: nginx
Date: Tue, 20 Jul 2010 05:45:13 GMT
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
Vary: Accept-Encoding
```

已经彻底隐藏了PHP版本。

## 2、其它几个PHP的基本安全设置：

```shell
disable_functions = phpinfo,system,exec,shell_exec,passthru,popen,dl,proc_open,popen,curl_exec,curl_multi_exec,parse_ini_file,show_source
#该指令接受一个用逗号分隔的函数名列表，以禁用特定的函数。
```

```shell
display_errors = Off
#是否将错误信息作为输出的一部分显示。在最终发布的web站点上，强烈建议你关掉这个特性，并使用错误日志代替。打开这个特性可能暴露一些安全信息，例如你的web服务上的文件路径、数据库规划或别的信息。
```

```shell
allow_url_fopen = Off
#是否允许打开远程文件，建议关闭，如果网站需要采集功能就打开。
```

```shell
safe_mode = On
#是否启用安全模式。打开时，PHP将检查当前脚本的拥有者是否和被操作的文件的拥有者相同，相同则允许操作，不同则拒绝操作。开启安全模式的前提是你的目录文件权限已完全分配正确。
```

```shell
open_basedir = /var/www/html/ha97:/var/www/html/168pc
#目录权限控制，ha97目录中的php程序就无法访问168pc目录中的内容。反过来也不行。在Linux/UNIX系统中用冒号分隔目录，Windows中用分号分隔目录。
```

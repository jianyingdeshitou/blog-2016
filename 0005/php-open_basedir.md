# PHP 配置文件中open_basedir选项作用

> 来源：http://www.jb51.net/article/19231.htm

> `open_basedir`: 将用户可操作的文件限制在某目录下

如下是`php.ini`中的原文说明以及默认配置:

```shell
; open_basedir, if set, limits all file operations to the defined directory
; and below. This directive makes most sense if used in a per-directory or
; per-virtualhost web server configuration file. This directive is
; *NOT* affected by whether Safe Mode is turned On or Off.
open_basedir = .
```

`open_basedir`可将用户访问文件的活动范围限制在指定的区域，通常是其家目录的路径，也
可用符号"`.`"来代表当前目录。注意用`open_basedir`指定的限制实际上是前缀,而不是目录名。
举例来说: 若"`open_basedir = /dir/user`", 那么目录 "`/dir/user`" 和 "`/dir/user1`"都是
可以访问的。所以如果要将访问限制在仅为指定的目录，请用斜线结束路径名。例如设置成:

```shell
"open_basedir = /dir/user/"
```

`open_basedir`也可以同时设置多个目录, 在Windows中用分号分隔目录,在任何其它系统中用
冒号分隔目录。当其作用于Apache模块时，父目录中的`open_basedir`路径自动被继承。

有三种方法可以在Apache中为指定的用户做独立的设置:

* (a) 在Apache的`httpd.conf`中`Directory`的相应设置方法:

```shell
php_admin_value open_basedir /usr/local/apache/htdocs/
#设置多个目录可以参考如下:
php_admin_value open_basedir /usr/local/apache/htdocs/:/tmp/
```

* (b) 在Apache的`httpd.conf`中`VirtualHost`的相应设置方法:

```shell
php_admin_value open_basedir /usr/local/apache/htdocs/
#设置多个目录可以参考如下:
php_admin_value open_basedir /var/www/html/:/var/tmp/
```

* (c) 因为`VirtualHost`中设置了`open_basedir`之后, 这个虚拟用户就不会再自动继承`php.ini`中的`open_basedir`设置值了,这就难以达到灵活的配置措施, 所以建议您不要在`VirtualHost`中设置此项限制. 例如,可以在`php.ini`中设置`open_basedir = .:/tmp/`, 这个设置表示允许访问当前目录(即PHP脚本文件所在之目录)和`/tmp/`目录.

> 请注意: 若在`php.ini`所设置的上传文件临时目录为`/tmp/`, 那么设置`open_basedir`时就必须
包含`/tmp/`,否则会导致上传失败. 新版php则会提示"`open_basedir restriction in effect`"
警告信息, 但`move_uploaded_file()`函数仍然可以成功取出`/tmp/`目录下的上传文件,不知道
这是漏洞还是新功能.

针对ShopEx472版本的配置：

```shell
open_basedir = "D:/Server;../catalog;../include;../../home;../syssite;../templates;../language;../../language;../../../language;../../../../language"
```

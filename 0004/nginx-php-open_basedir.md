# nginx+php使用open_basedir限制站点目录防止跨站

> http://www.server110.com/nginx/201308/477.html

以下三种设置方法均需要PHP版本为5.3或者以上。

## 方法1）

在Nginx配置文件中加入

```shell
fastcgi_param  PHP_VALUE  "open_basedir=$document_root:/tmp/:/proc/";
```

通常nginx的站点配置文件里用了`include fastcgi.conf`;，这样的，把这行加在`fastcgi.conf`里就OK了。
如果某个站点需要单独设置额外的目录，把上面的代码写在`include fastcgi.conf;`这行下面就OK了，会把`fastcgi.conf`中的设置覆盖掉。
这种方式的设置需要重启nginx后生效。

## 方法2）

在`php.ini`中加入：

```shell
[HOST=www.server110.com]
open_basedir=/home/www/www.server110.com:/tmp/:/proc/
[PATH=/home/www/www.server110.com]
open_basedir=/home/www/www.server110.com:/tmp/:/proc/
```

这种方式的设置需要重启`php-fpm`后生效。

## 方法3）

在网站根目录下创建`.user.ini`并写入：

```shell
open_basedir=/home/www/www.server110.com:/tmp/:/proc/
```

这种方式不需要重启nginx或`php-fpm`服务。安全起见应当取消掉`.user.ini`文件的写权限。
关于`.user.ini`文件的详细说明：
http://php.net/manual/zh/configuration.file.per-user.php

设置`open_basedir`的同时最好禁止下执行命令的函数，比如：

* `shell_exec('ls /etc')`仍然查看到`/etc`目录的文件列表
* `shell_exec('cat /etc/passwd')`仍可查看到`/etc/passwd`文件的内容

建议禁止的函数如下：

```shell
disable_functions = pcntl_alarm, pcntl_fork, pcntl_waitpid, pcntl_w
```

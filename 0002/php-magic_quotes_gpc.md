# PHP magic_quotes_gpc的正确处理方式

> 来源：http://blog.csdn.net/lyjtynet/article/details/6261169

大多的 PHP 程序，都有这样的逻辑：

如果发现 `php.ini` 配置为不给 GPC 变量自动添加转义斜线，则 PHP 自动为 GPC 添加转义斜线

但是事实上，这是错误的，因为它改变了 GPC 变量原来的值．

有这个遗留习惯的原因是 PHP 程序使用往往配合 MySQL, 而 mysql 对特殊字符的转义，采取的是添加转义斜线，但是其它数据如 mssql,oci 呢，不一定是这样的．

如果使用其它类型数据库，如 mssql,oci,sybase　那么，给 GPC 添加转义斜线，更是个错误

进一步，如果 GPC 数据不需要存入数据库，而保存到文件系统，或转发给其它程序呢？更是很严重的错误逻辑．

所以，正确的做法是：

1. PHP 程序入口去掉转义斜线（若 `php.ini` 配置为自动添加转义斜线）
1. 在写入 mysql 时，使用 `mysql_real_escape_string` 而不是 `addcslashes` 来转义变量，因为前者比后者更为安全(字符集相关的)

db 类中已考虑到这个问题，详情参阅 `db_mysql.class.php`,搜寻 `mysql_real_escape_string`

目前有以下案例：

* 积分商城的 `php.ini` 配置为自动添加转义斜线，用户提交的数据写入 cookie 时，需要及时去掉斜线
* `discuz 6.0` 的论坛，特殊用户名中的＂頫＂经过 `addcslashes` 处理后，竟然变成＂頫/＂,后面多了一个斜线，这是 `discuz 6` 的一个 bug．

那么，综述一下：

* 针对系统管理员，应该配置 `php.ini`

```ini
magic_quotes_gpc=Off
magic_quotes_runtime=Off
magic_quotes_sybase=Off
```

* 针对 php 开发人员，更准确的逻辑：

1. 检查 php 环境是否配置为自动添加转义斜线，若是，应该调用 `stripslashes` 去掉 `$_REQUEST`,`$_GET`,`$_POST`,`$_COOKIE` 的转义斜线
1. 查询/写入/修改数据至 mysql 时，再使用 `mysql_real_escape_string` 转义之

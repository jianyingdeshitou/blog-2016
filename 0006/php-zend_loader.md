# 谁知道PHP加密方面的事情？（Zend Guard Loader）

> 来源：http://www.hostloc.com/thread-245741-1-1.html

## 常识：

### 1

* Zend Guard，大家常用的PHP代码加密工具。
* Zend Optimizer（ZO），对应的解密工具
* Zend Guard Loader（ZGL）  是 Zend Optimizer的代替品
* Zend Guard Loader只能安装在PHP5.3.x及以上的版本里，Zend Optimizer只能安装在PHP5.2.x及以下的版本中。

### 2

* 由于PHP 的版本（5.4.16）的原因，PHP 5.4.x 对应的是Zend Guard Loader
* PHP 5.3.x 及以上版本，使用Zend Guard Loader
* PHP 5.2.x 及以下版本，使用Zend Optimizer

### 3

* Zend Guard 5.0.1 版本编译后的代码，只能在Zend Optimizer （版本3.3.3） 下运行。
* Zend Guard 5.5.0 版本编译后的代码，只能在Zend Guard Loader （版本5.5.0）下运行。

### 4

Zend Guard 和Zend Guard Loader的下载链接：http://www.zend.com/en/products/guard/downloads

---

Zend Guard Loader不支持TS版本，只支持NTS版本

---

## PHP For Windows：http://windows.php.net/

官方解释如下：

### IIS

If you are using PHP as `FastCGI with IIS` you should use the `Non-Thread Safe (NTS)` versions of PHP.

### Apache

Please use the Apache builds provided byApache Lounge. They also provide VC11 builds of Apache for x86 and x64. We use their binaries to build the Apache SAPIs.
If you are using PHP with Apache 1 or Apache2 from apache.org (not recommended) you need to use the older VC6 versions of PHP compiled with the legacy Visual Studio 6 compiler. Do NOTuse VC9+ versions of PHP with the apache.org binaries.
With Apache you `have to` use the `Thread Safe (TS)` versions of PHP.

### VC9 and VC11

More recent versions of PHP are built with VC9 or VC11 (Visual Studio 2008 and 2012 compiler respectively) and include improvements in performance and stability.
The VC9 builds require you to have theVisual C++ Redistributable for Visual Studio 2008 SP1 x86 or x64 installed.
The VC11 builds require to have the Visual C++ Redistributable for Visual Studio 2012x86 or x64 installed.

### TS and NTS

`TS refers to multithread capable builds.NTS refers to single thread only builds.` Use case for TS binaries involves interaction with a multithreaded SAPI and PHP loaded as a module into a web server. For NTSbinaries the widespread use case is interaction with a web server through the FastCGI protocol, utilizing no multithreading (but also for example CLI).

---

`php.ini`有个选项 开了才是真正开启ZendGuard

```shell
zend_loader.enable = 1
zend_loader.disable_licensing = 0
zend_loader.obfuscation_level_support = 3
```

---

请帮忙看看，这个配置有没有问题，包括注明的问题哦，谢谢啦

```shell
[Zend.loader]
zend_extension=
"D:\wamp\bin\php\php5.4.16\ext\ZendLoader.dll"
;启用加载编码脚本。默认开启 ,【默认为1】
zend_loader.enable=1
;是否检查授权，【0为禁止，1为允许】
zend_loader.disable_licensing=1
;配置混淆水平，这里不懂？
zend_loader.obfuscation_level_support=3
;寻找授权文件的路径，这里怎么填，是必选项吗？
zend_loader.license_path=
```

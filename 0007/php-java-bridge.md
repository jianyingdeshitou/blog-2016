# 利用php-java-bridge包实现PHP调用JAVA类

> http://blog.csdn.net/qhdcsj/article/details/49131387

在工作中需要将office文档软件转为pdf文件，查找了很多技术发现使用openoffice可以完成这个要求，但调用它的功能全部是Java写的，所有需要在PHP中调用java类，就有了下面调试过程。

1、安装java，这个过程我就不写了，网上有很多，讲的也很细。（其实可以不用安装，我就是将原机器上的java整个目录保存了下来直接使用，实现软件的绿化）

2、下载`php-java-bridge`包。地址：http://sourceforge.NET/projects/php-java-bridge/files，在Binary package栏目中下载`JavaBridgeTemplate621.war`。使用winrar解压，找到`WEB-INF\lib`下的`JavaBridge.jar`文件。

3、使用`java.exe`打开这个文件：`java -jar JavaBridge.jar`(也可以使用`javaw.exe`这样执行后可以马上退出命令窗口),在弹出的窗口中选择`8080`端口。

4、新建一个php文件测试是否成功。文件内容如下：

```php
<?php
require_once("http://localhost:8080/JavaBridge/java/Java.inc");
$System = java("java.lang.System");
echo $System->getProperties();
?>
```

在上面中使用URL地址包含，所以需要在`php.ini`文件`allow_url_include`设为`On`。

5、编辑自己的java类，并使用`jar.exe`打包。将所有需要的包放入`jre7/lib/ext`目录下。（有新的包放入时需要重新启动`JavaBridge.jar`。

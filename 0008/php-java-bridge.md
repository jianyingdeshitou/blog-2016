# PHP-Java-Bridge使用笔记

> http://blog.csdn.net/jiben1002/article/details/40210859

最近因为需要封装一个jar包供PHP调用，在网上搜索了下，基本上讲都是使用`php-Java-bridge`，说实话，网上的教程有很多是不行的，但是功夫不负有心人，找到了一篇文章，也很感谢那篇文章的作者，链接是 http://www.jb51.net/article/55451.htm ，大部分是参考这里的，折腾了这一两天，有一点心得跟大家分享了，少走一点弯路，特别是刚接触者一块的，我自己也是新人。

现在进入正题，我写的是基于windows平台下的，第一部分当然就是下载`Php-java-bridge`，自己也是没经验，花了一点时间在上面，第一次直接下载的是`JavaBridge.jar`，其实不是单独下载这个，首先进入这个网站 http://sourceforge.net/projects/php-java-bridge/files ，选择Binary package，然后选择最新的版本`Php-java-bridge_6.2.1`，下载`JavaBridgeTemplate621.war`，下载好以后就要用到tomcat了，首先把`JavaBridgeTemplate621.war`放到tomcat下`webapps`，启动tomcat，tomcat就会解析该文件，然后产生一个同名文件夹，tomcat的在这里的主要作用就是这个（用完关掉），然后把该文件夹复制到Apache中使用，接下来就是把自己写好的jar包放到Java虚拟机下面，也就是jre安装下面，比如我的是`C:\Program Files\Java\jre6\lib\ext`下面（在高版本已经不能使用`java_require`了，把自己写的jar包放虚拟机下就不需要引入包了），接着就是双击运行`JavaBridge.jar`（这文件可以单独下载，也可以在刚才的`JavaBridgeTemplate621\WEB-INF\lib`目录下找到这文件），选择`8080`端口，在这里需要注意下顺序，是先放写好的jar包，然后运行`JavaBridge.jar`，否则会提示找不到class文件，如果要有新的jar包写好后放到java虚拟机目录，先把虚拟机停掉（我直接任务管理器结束`java.exe`），然后启动`JavaBridge.jar`，接着就可以启动Apache测试了，就写到这里了。

上面是一点简单的经验，其实我自己还是有很多不懂，如果上面有误，欢迎交流指正，谢谢。

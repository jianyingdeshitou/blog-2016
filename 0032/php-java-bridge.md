# PHP-Java-Bridge使用笔记

> http://www.jb51.net/article/55451.htm

> 这是我在做平安银行开发的时候，本地使用PHP环境，平安银行接口为Java接口的时候，采用PHP-Java-Bridge的方式调用接口的笔记。因为现在网上的教程基本上都不行了，所以在这里贴出我能使用的而且目前网上最新的版本（2014-09-19），如果有错，请通过邮件联系，谢谢。

> * @author  ken(695093513@qq.com)
> * @date    2014-09-09

## 版本与环境

版本：

1、PHP版本：最高为5.4,当前测试为5.4/5.3

2、JDK：官方最新版本,当前测试为1.8

3、php-java-bridge：官方最新版本,当前测试为6.2.1

4、操作系统：Windows7 32位/64位 | Linux(Centos6.5)

## 安装和使用

### 第一步：安装

1、JDK的安装：正常安装即可，并配置好环境变量

2、PHP的安装：正常安装即可

3、php-java-bridge的安装：

①先下载Java服务器Tomcat正常安装，安装好后，开启Tomcat服务器

②将下载的php-java-bridge包放到webapps下面

③等待Tomcat执行解析，会在该目录下面生成相同名字的文件夹

④将该文件夹拷贝到Apache服务器下面使用

(注：网上的教程可以正常使用，调用java系统函数和简单的jar包，但是对于复杂的jar包会遇到各种各样的问题，所以建议使用这种方式)

### 第二步：使用

1、不需要开启Tomcat(最好关闭掉)，开启apache服务器，双击运行javabridge.jar,选择8080端口(javabridge.jar也需要放到java虚拟机下面，参见下面第二点规则)。

2、尽可能的将jar包放到java虚拟机下面，即jre安装下面(比如：C:\Program Files\Java\jre1.8.0_20\lib\ext)

3、在PHP文件中不需要再引用jar包，因为放到虚拟机下面去了，java会自动调用

(注：第1点中的javabridge.jar是在第一步:安装中第3点中获得的)

## 其他使用方法和注意事项

### 关于PHP-Java-Bridge的各种函数使用

1、高版本的java_require不再使用,也无法使用,由于放到java虚拟机下面,则不需要再手动引入包文件

2、java_value()用于获取值,而且必须使用该函数获取值

(特别注意：如果该值需要存入数据库，那么必须使用该java_value函数,不然会报错，或者无法存入数据库)

3、java_inspect()对实例化或者方法进行print_r类似的输出

(注：请不要直接使用var_dump这样的输出方法输出java的类、方法、变量，需要使用java_inspect或者java_value，例如：var_dump(java_inspect($abc)))

4、实例化使用 $test = new Java("Test")的方式,如果实例化的方法中存在参数,可以这样new Java("Test","pram")

### 注意事项

1、务必确保对java.inc的引用，确保引用正确

2、务必确保对jar包放在能引用的地方，比如java虚拟机jre下面

3、在PHP中调用Java使用PHP的的写法即可

## 附录1：各种报错问题处理

1、参照上面的“其他使用方法和注意事项”,大多数问题都是路径引用的问题,只要处理好了,正确获得了,就不会出问题

## 附录2：PHP实例代码

```php
require_once("/java/Java.inc");

$util = new Java("com.sdb.payclient.core.PayclientInterfaceUtil");
$input = new Java("com.ecc.emp.data.KeyedCollection");
$signDataput = new Java("com.ecc.emp.data.KeyedCollection");

$input->put("masterId","111111");
$input->put("orderId","222222");

$signDataput = $util->getSignData($input);

$orig = java_values($signDataput->getDataValue("orig"));
$sign = $signDataput->getDataValue("sign");

echo java_values($sign);
```

## 附录3：PHP-Java-bridge文件包解压后目录图

```text
bridge
 --java
 java.inc
 JavaProxy.php
 --WEB-INF
 --cgi
 --...
 --lib
 php-script.jar
 php-servlet.jar
 --pear
 web.xml
 weblogic.xml
```

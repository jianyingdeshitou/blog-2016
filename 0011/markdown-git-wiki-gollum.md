# markdown + git 最适合程序员的wiki系统：gollum

> 来源：http://www.open-open.com/lib/view/open1422014636421.html

gollum 是 github 的使用的一个基于markdown的 wiki系统。 最重要的是gollum 直接和 git 集成不需要数据库，你可以选择在Web页面撰写文档，也可以用你喜欢的markdown工具编辑文档在命令行进行提交。 “markdown+git = wiki” 这对程序员来讲绝对是最棒的选择。

Gollum wiki 的截图如下

![](001.png)

Mac下安装的适合遇到了一些小问题，所以在这里记录一下安装过程 。

## 基本的环境

在安装之前，假定我们已经拥有了mac 下的包管理工具 homebrew 及 ruby 运行环境。我当前的工作环境如下:

* Mac 10.9
* homebrew 0.9
* ruby 2.0

## 安装 Gollum

我们可以选择通过二进制或者源码的方式进行安装。

### 二进制安装

在安装的时候会出现找不到`libiconv`所以需要安装下面的依赖库

```shell
$ brew install libxml2 libxslt
$ brew link libxml2 libxslt --force
$ wget http://ftp.gnu.org/pub/gnu/libiconv/libiconv-1.13.1.tar.gz
$ tar xvfz libiconv-1.13.1.tar.gz
$ cd libiconv-1.13.1
$ ./configure --prefix=/usr/local/Cellar/libiconv/1.13.1
$ make
$ sudo make install
```

通过Gem安装 gollum

```shell
$ [sudo] gem install gollum
```

### 源码安装

```shell
$ git clone https://github.com/gollum/gollum
$ cd gollum
$ bundle install
```

安装成功后尝试着在终端输入下面的命令，如果能够正常显示版本号则说明安装成功.

## 创建自己的wiki系统

接下来我们就可以建立自己的wiki系统了，建立一个名字为”`wiki`”的目录使用git进行管理。进入到`wiki`目录，在wiki目录下启动gollum

```shell
$ mkdir wiki
$ cd wiki
$ git init
$ gollum
```

显示如下：

```shell
[2015-01-21 14:03:45] INFO  WEBrick 1.3.1
[2015-01-21 14:03:45] INFO  ruby 2.0.0 (2013-05-14) [x86_64-darwin12.4.1]
== Sinatra/1.4.5 has taken the stage on 4567 for development with backup from WEBrick
[2015-01-21 14:03:45] INFO  WEBrick::HTTPServer
```

接下就可以在浏览器中访问了 http://localhost:4567/

创建wiki文档可以选择在全部Web界面进行操作，也可以选择在终端命令行进行提交管理markdown文件，在浏览器中进行查看。

来自：http://examplecode.github.io/tools/2014/09/26/install-gollum-in-mac-109/

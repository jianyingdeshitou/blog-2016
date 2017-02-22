# 使用Gitbook制作电子书

> 来源：http://www.ituring.com.cn/article/127645

Gitbook是一个命令行工具，可以把你的Markdown文件汇集成电子书，并提供`PDF`等多种格式输出。你可以把Gitbook生成的`HTML`发布出来，就形成了一个简单的静态网站。Gitbook还有一个同名的平台（gitbook.io），可以发布和销售电子书，并提供了一个Markdown客户端工具（支持Mac、Windows和Linux）帮助写作。以下是我在使用Gitbook中的笔记。

首先Gitbook和`Git/Github`都没有什么关系。它只是一个build book的工具而已。但它的Git前缀的确引起了许多人的迷惑，起初我认为至少它也是个和`Github`类似的Git平台吧，但其实没什么关系，你只要懂几条markdown语法，不必理解任何与Git相关的东西就能用Gitbook了，不要为其名字迷惑。

## 第0步 安装npm（Node Package Manager）。

从`node.js`的 官网 上下载安装程序，即可完成`Node.js`和`npm`的安装。

## 第1步 通过npm安装Gitbook。

```shell
$ npm install gitbook -g
```

完成后花10分钟阅读下Gitbook的 帮助文档 。如果你没耐心看手册，那就继续往下读吧 :D

## 第2步 了解Gitbook的基本规则。

Gitbook需要2个基本文件：

```shell
README.md
SUMMARY.md
```

`README.md`是关于你的书的介绍，而`SUMMARY.md`中则包含了书目，即章节结构，它的格式大致是：

```markdown
* [第1章](c1.md)
    * [第1节](c1s1.md)
    * [第2节](c1s2.md)
* [第2章](c2.md)
```

剩下的东西就很好理解了，你只需要编写相应章节即可。在编辑完`README.md`和`SUMMARY.md`后，你可以运行以下命令：

```shell
$ gitbook serve -p 8080 .
```

Gitbook首先把你的Markdown文件编译为`HTML`文件，并根据`SUMMARY.md`生成书的目录。所有生存的文件都保存在当前目录下的一个名为`_book`的子目录中。完成这些工作后，Gitbook会作为一个`HTTP Server`运行，并在`8080`端口监听`HTTP`请求。

运行以上命令后，打开浏览器，在地址栏输入： `http://localhost:8080` 即可看到你的书页了。

其中位于左侧书目顶部的 Introduction 一节就编译自`README.md`，而书目本身自编译自`SUMMARY.md`。你要在自己的网站上发布新书，只需把`_book`目录复制到服务器相应目录即可。至此Gitbook的基本用法就介绍完毕。下面简单讨论下Gitbook的其他应用，包括Gitbook的插件、与`Github`的融合、Gitbook客户端、Gitbook平台，以及Gitbook的问题。

## Gitbook的插件支持

Gitbook可以生成`HTML`，因此它支持一些外部的`JavaScript`文件嵌入到`HTML`中，例如Google统计、Disqus评论系统等。以下以页面中嵌入Disqus评论为例。

首先是安装Gitbook的Disqus插件。

```shell
$ npm install gitbook-plugin-disqus
```

然后建立一个`book.json`文件，其格式如下：

```json
{
  "plugins": ["disqus"],
  "pluginsConfig": {
    "disqus": {
      "shortName": "NAME-FROM-DISQUS"
    }
  }
}
```

把上面的 `NAME-FROM-DISQUS` 修改为你在Disqus上的项目名即可。

再次运行命令：

```shell
$ gitbook serve -p 8080 .
```

并刷新浏览器，即可看到附加了Disqus评论的页面。

## 与Github的融合

Gitbook的博客上说`Github`提供了对Gitbook的特殊支持，但我没有测试。只是依然把源文件保存在`Github`上，然后用Gitbook去编译。期待Gitbook做的更好。

## Gitbook客户端

Gitbook客户端支持Mac、Windows、Linux。我在Mac和Windows简单尝试了这个客户端，总体而言可以用。但也仅仅是可以用而已。你可以在客户端里编辑Markdown文件，并提供一个实时的预览窗口；可以关联到你的Gitbook账户，并把内容同步到gitbook.io，并为你生成PDF等。说句题外话，如果你要Markdown的客户端的话，飞象马克更好用，至少Vim编辑模式你得支持啊。

## Gitbook的问题

Gitbook网站的访问速度很慢。可以在生成`_book`目录后，把其中的HTML文件和gitbook子目录（包含字体和js文件等）复制到自己的网站上。

Gitbook提供的`push`功能不能用。`push.gitbook.io`这个地址无法访问，不知是否是临时性服务故障。

Gitbook生成`PDF`的中文字体极其难看。万分期待改进。话说Gitbook生存的`HTML`上的中文非常漂亮。

在我的手机上看Gitbook的页面时，会让浏览器挂掉。

末，话说我也是个Gitbook新手呢，有理解不对的请大家指教 :-)

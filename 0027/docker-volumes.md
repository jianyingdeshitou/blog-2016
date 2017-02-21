# Docker 中管理数据

> 来源：http://www.oschina.net/translate/dockervolumes

> 英文原文：Managing data in containers(http://docs.docker.com/userguide/dockervolumes/)

---

## 目录

* 添加数据卷
* 安装一个目录作为数据卷
* 创建并安装数据卷容器
* 备份、恢复和移植数据卷

---

到目前我们介绍了一些Docker的基础概念, 知道了如何使用Docker的image， 也知道了如何在多个container间通过网络通讯. 在这章里我们将介绍如何在docker的container内管理数据以及如何在不同的container间共享数据。

我们将介绍两种主要的在docker中管理数据的方法：

* Data volumes
* Data volume container

## Data volumes

一个 data volume 就是一个在一个或者多个container里的特殊用途的目录。它绕过了 Union File System (译者： 这里不确定， 需要研究)为持久化数据、共享数据提供了下面这一些有用的特性：

* Data volumes 可以在不同的container之间共享和重用数据
* 对 Data volume 的修改及时生效（译者：data volumn是一个目录， 多个container都挂载这个目录， 具体的可以通过 docker inspect 看 volumne的信息)
* 对 data volume 修改内容在升级image的时候不会被包括进去 (译者：在docker的整个设计中image是一个无状态的， 这样对升级重用非常有利。而标记状态的数据， 比如数据库的数据， 生产的log之类的应该放到volume里。volume的持久化和恢复在下面有介绍， 是通过文件的形式的， 而不是通过image)
* Volumes 的持久化直到没有container使用他们

## 添加数据卷

你可以在docker run 的时候使用 -v 来添加一个 data volume。这个参数在docker run 的时候可以多次使用来添加多个 data volumes。让我们为我们的web application container挂载一个 volume。

```bash
$ sudo docker run -d -P --name web -v /webapp training/webapp python app.py
```

这里一个新的volume会创建到container里的 /webapp. （译者：如果你通过ssh或者通过 -i 登陆到你的container的一个shell里， 使用 ls /webapp 可以验证挂载成功了)

>注意: 你也可以在Dockerfile里添加 VOLUME 字段，这样在创建一个新的image的 container是就会自动的创建新的volume.

## 安装一个目录作为数据卷

使用 -v 不仅能创建一个新的 volume， 还可以把宿主机一个目录mount到container里。

```bash
$ sudo docker run -d -P --name web -v /src/webapp:/opt/webapp training/webapp python app.py
```

这条命令会把本地目录 /src/webapp mount到container里的 /opt/webapp 目录上。用这个方法来测试程序非常方便， 比如我们可以把我们的源代码通过这个方法mount到container里， 修改本地代码后立即就可以看到修改后的代码是如何在container里工作的了。宿主机的目录必须是绝对路径， 如果这个目录不存在docker会为你自动创建。

>注意 这里是没法用 Dockerfile实现的， 因为这样的用法有悖于可移植性和共享. 因为本地目录就像他名字告诉我们的， 是和本地相关的， 不一定可以在所有的宿主机上工作.(译者: 鬼知道你在使用image的时候的host是啥样子的）

Docker默认设置volume是可读写的，但是我们也可以mount一个目录为只读:

```bash
$ sudo docker run -d -P --name web -v /src/webapp:/opt/webapp:ro training/webapp python app.py
```

这里我们同样mount了 /src/webapp 目录， 但是我们加上了 ro 参数， 告诉docker这个volume是只读的.

## 创建并安装数据卷容器

如果你有一些持久化的数据， 并且想在不同的container之间共享这些数据， 或者想在一些没有持久化的container中使用， 最好的方法就是使用 Data Volumn Container, 在把数据mount到你的container里.(译者：如开篇译者提到的docker的container是无状态的， 也就是说标记状态的数据，例如：数据库数据， 应用程序的log 等等， 是不应该放到container里的， 而是放到 Data Volume Container里, 这点和funcational programming很像， 所以我喜欢把一般的docker container 叫做 functional container用来区分 data volume container ）

让我们创建一个有名字的 Data Volume Container 来共享数据.

```bash
$ docker run -d -v /dbdata --name dbdata training/postgres
```

这样做之后就可以通过 --volumes-from 把 /dbdata mount到其他的container里了

```bash
$ docker run -d --volumes-from dbdata --name db1 training/postgres
```

还可以继续共享到另外一个container里

```bash
$ docker run -d --volumes-from dbdata --name db2 training/postgres
```

-volumes-from 可以多次使用来 mount 多个conatainer里的多个volumes。

这个操作是链式的， 我们在db1 中通过 --volumes-from mount进来的 volume可以继续被其他container使用

```bash
$ docker run -d --name db3 --volumes-from db1 training/postgres
```

>(译者: 这里我们不是直接使用 volume container， 而是使用db1 这个functional container 把volume 挂载到另外一个 funcational container上的，所谓的链式就是 dbdata -> db1 -> db3)

如果你把所有mount volumes的container都移除掉， 包括初始化的那个 dbdata container， volume才会被移除掉。通过这个属性可以方便的升级升级数据或者在不同container间migrate数据.

## 备份、恢复和移植数据卷

Volume的另外一个用处就是备份、恢复和migrate数据。 具体的做法如下，使用 --volumes-from 来创建一个新的container mount这个volume

```bash
$ sudo docker run --volumes-from dbdata -v $(pwd):/backup ubuntu tar cvf /backup/backup.tar /dbdata
```

这里我们启动了一个新的container， 从 dbdata 挂载了一个volume。同时挂载了一个本地目录到这个container里。最后我们通过一个 tar命令把 dbdata 里的数据备份到了 /backup 里。命令结束并且停止这个container后我们就在本地得到了一个备份的数据.

>(译者: 这里使用的 ubuntu container， 就是为了把volume中的数据打包备份到host的某一个目录里。)

备份的数据可以恢复到这个container， 或者其他使用这个volume的container。首先创建一个container

```bash
$ sudo docker run -v /dbdata --name dbdata2 ubuntu
```

之后un-tar备份文件到 data volume 里

```bash
$ sudo docker run --volumes-from dbdata2 -v $(pwd):/backup busybox tar xvf /backup/backup.tar
```

你可以使用你喜欢的工具加上上面的技术来自动备份，迁移和恢复数据

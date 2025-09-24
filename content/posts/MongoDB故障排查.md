+++
date = '2025-09-24T15:50:21+08:00'
draft = false
title = 'MongoDB错误处理【1053】【1067】（意外断开读写中的数据库）'

tags = ["MongoDB","故障"]

categories = ["故障排查"]

+++

**摘要：** 本文记录了Windows服务器因意外断电导致MongoDB服务无法启动的解决方案。主要处理了两个错误：1053错误（通过删除并重建MongoDB服务解决）和1067错误（通过修复受损数据文件解决）。重点介绍了使用repair命令修复数据的方法，该过程可能耗时较长（千万级数据需近一天）。最后强调在数据库读写过程中切勿随意断开连接，以免造成数据损坏。解决方案同样适用于Linux系统。

[toc]

### 起因

本人的MongoDB服务在**windows服务器**运行，**且一直有读写的操作**。因为服务器突然断电导致出错，再次重启服务器后无法正常运行MongoDB服务。

查阅了网上的众多资料后觉得实在很碎片化，尤其是有的文章互相抄袭，难以参考，于是记录下自己遇到的问题和解决成功的办法，如果你同样遇到了**意外断开MongoDB服务（尤其是有读写操作运行的时候）**，那么极有可能遇到这两个错误，希望以下错误处理的方法对你有用。

（本人服务器环境是windows系统，但是本操作流程可以同样适用于linux系统，具体的命令可以查询ai）

**读写操作执行过程中千万不要随便让数据库断开连接！**
### 解决问题1：无法启动MongoDB服务[1053错误]

使用命令行工具或者windows服务启动MongoDB遇到如下报错，无法启动：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/d43472bf45954a8ab778b650018dd743.png)


这个错误的解决办法很简单：删除该服务并重新创建一个服务。
**MongoDB服务相当于注册表的一条命令，并不影响本地的数据，可以随便进行删除和重新创建。**

1.  删除原有的MongoDB服务

```cmd
 sc.exe delete MongoDB
```

2.  创建新的MongoDB服务
    这个方式有多种，一般分为两种创建的方式：

*   一种直接命令输入指定数据库的存储路径 `dbpath`（可以有本地数据，不会影响） 和日志的路径`logpath`；
*   另一种则是使用`config`文件（位于MongoDB安装目录的bin目录下的`mongod.cfg`文件，一般位于`C:\Program Files\MongoDB\Server\5.0\bin>`），在config文件里面写入配置如数据库存储路径和日志，以及是否开启安全验证等，**这样就可以避免多次直接在命令行中输入路径**。

为了方便使用，这里我们编辑一个简单的cfg文件（也可以编辑原本的`mongod.cfg`），在其中写入路径配置

```cfg
##数据文件
dbpath=E:\MongoDB\data

##日志文件
logpath=E:\MongoDB\log\mongo.log
```

然后在**当前的bin目录**下运行以下命令，表示按照config创建一个MongoDB服务。

```cmd
 mongod.exe --config "C:\Program Files\MongoDB\Server\5.0\bin\mongod.cfg" --install --serviceName "MongoDB" --serviceDisplayName "MongoDB"
```

创建成功是没有返回结果的，此时可以通过`任务管理器`-`服务`-`打开服务`-`MongoDB`）看到在启动的状态。

其实就是创建了一个新的命令，如果你的本地数据没有重要的数据，可以直接创建新的服务时候指定一个新的`dbpath`，这样基本上不会遇到后面1063错误了。

![windows查看服务是否开启](https://i-blog.csdnimg.cn/direct/ae8af03fabac47a1a5dad8455fc6aa7f.png)


3.  尝试启动MongoDB
    这里可以可以尝试启动一下，如果直接可以启动说明数据库没有数据文件损坏。**如果没有什么数据文件，推荐可以直接在新的文件夹下存储数据，也就是将`dbpath`路径置为新的文件夹。**

```cmd
net start MongoDB
```

### 解决问题2： 进程意外错误[1067错误]

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/6dd656995fef421ba9ba7d50ea211e0f.png)


这个问题有多个原因，网上的处理方式一般是不针对数据受损的情况的，如果数据不受损，那么可以直接将`dbpath`路径下的`mongod.lock`文件删除（锁文件，不影响实际数据），然后再次使用net start命令启动。

如果还是不行那么考虑数据受损的情况，那就使用`repair`命令修复。

##### 使用repair命令，扫描所有的本地数据修复（耗时长）

**（使用前删除lock文件）**

该命令是MongoDB专门应对数据文件受损的一种修复机制，原理是删除所有搜索的索引和数据文档。所以最好在运行该命令前对数据文件进行备份！

直接**在MongoDB的bin目录下**运行以下命令：

```cmd
mongod.exe --dbpath E:\MongoDB\data --repair
```

该命令会对`dbpath`中的所有的数据文件进行扫描、检查、索引的重构。如果数据量大的话（尤其是索引很多需要重新构建），时间耗费也会很久。**参考本人千万级别的数据文档，运行了近一天一夜才结束。**

在结束后会详细报告修复的集合。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/6ccb9b4c71634ae08fee41dcd742ee73.png)


最后运行执行启动MongoDB，这次一般都可以顺利进行执行。

```cmd
net start MongoDB
```

**最后一定不要随便给正在读写的MongoDB数据库意外断开！！**

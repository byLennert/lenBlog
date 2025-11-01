+++

date = '2025-11-01'
draft = false
title = 'WSL迁移Ubuntu到D盘'

tags = ["Linux"]

categories = ["技术记录"]

+++

## **1.准备工作**

打开[CMD](https://zhida.zhihu.com/search?content_id=226325634&content_type=Article&match_order=1&q=CMD&zhida_source=entity)，输入`wsl -l -v`查看wsl虚拟机的名称与状态。

了解到本机的WSL全称为Ubuntu-20.04，以下的操作都将围绕这个来进行。

输入 `wsl --shutdown` 使其停止运行，再次使用`wsl -l -v`确保其处于stopped状态。

## **2.导出/恢复备份**

在D盘创建一个目录用来存放新的WSL，比如我创建了一个 `D:\Ubuntu_WSL` 。

①导出它的备份（比如命名为Ubuntu.tar)

```text
wsl --export Ubuntu-20.04 D:\Ubuntu_WSL\Ubuntu.tar
```

②确定在此目录下可以看见备份Ubuntu.tar文件之后，注销原有的wsl



```text
wsl --unregister Ubuntu-20.04
```

③将备份文件恢复到`D:\Ubuntu_WSL`中去

```text
wsl --import Ubuntu-20.04 D:\Ubuntu_WSL D:\Ubuntu_WSL\Ubuntu.tar
```

这时候启动WSL，发现好像已经恢复正常了，但是用户变成了root，之前使用过的文件也看不见了。

## **3.恢复默认用户**

在CMD中，输入 `Linux发行版名称 config --default-user 原本用户名`

例如：lennert是我的用户名

```bash
Ubuntu2004 config --default-user lennert
```

请注意，这里的发行版名称的版本号是纯数字，比如Ubuntu-22.04就是Ubuntu2204。

如果显示以下说明环境变量里没有添加Ubuntu2004，没关系，我们可以直接进入到该程序的目录下运行cmd再次运行这样的命令

**Ubuntu.exe的路径在\AppData\Local\Microsoft\WindowsApps下面**，找到后就可以确认没有问题，再次运行

```bash
Ubuntu : 无法将“Ubuntu”项识别为 cmdlet、函数、脚本文件或可运行程序的名称。请检查名称的拼写，如果包括路径，请确保路径
正确，然后再试一次。
```

这时候再次打开WSL，你会发现一切都恢复正常了。

D盘里有 ext4.vhdx就代表成功了。这时候你可以看到C盘增加了不少空间。
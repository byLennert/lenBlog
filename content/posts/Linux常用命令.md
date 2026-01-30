+++
date = '2025-10-29'
draft = false
title = 'Linux部署python项目'

tags = ["Linux"]

categories = ["技术记录"]

+++

### 使用conda创建Python3.12版本环境

下载python3.12并创建conda环境名称为py312_env

```bash
conda create -n py312_env python=3.12
```

激活新环境

```bash
conda activate py312_env
```

查看当前环境的python版本（验证）

```bash
python --version
```

退出环境

```bash
conda deactivate
```

查看当前所有的环境

```bash
conda env list
```

git clone -b linux https://gitee.com/lennert/ark-ts.git

查看当前的pip

```bash
# 在激活的 py312_env 环境中执行
which pip #如果是当前的环境下的pip，就可以正常使用，下载包到当前环境中，否则就需要强制用当前环境的pip
# 示例：requirements.txt 在 ~/docs 目录下 
/home/dl2/anaconda3/envs/py312_env/bin/pip install -r requirements.txt
```

\# 示例：requirements.txt 在 ~/docs 目录下 



### 迁移windows数据到linux上（内网更适合）

使用基于ssh的scp工具进行，一般windows会默认自带，只要linux支持sshd就可以（常用的都支持）

1.检查是否windows可以使用scp

```cmd
ssh -V
```

2.测试 ping 通(在windows服务器上进行ping通linux)

```text
ping linux的ip
```

测试 SSH 连通性（SCP 依赖 SSH）

```cmd
# 格式：ssh Linux用户名@LinuxIP
ssh root@192.168.1.100
```

登录后输入`exit`退出 Linux 终端，回到 Windows PowerShell 即可

#### 前提满足后使用

迁移文件

```cmd
scp D:\... linux_username@202.23.1.165:/home/...
```

windows的文件路径必须要完整，linux也一样。

输入命令后需要输入linux的用户的密码。

之后会显示传输的进度条。

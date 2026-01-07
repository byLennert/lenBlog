+++
date = '2026-1-07'
draft = false
title = '排查linux服务器根目录空间满的问题'

tags = ["linux","清理空间"]

categories = ["故障排查"]

+++

## 清理linux服务器根目录空间

现象：代码无法复制到新文件，创建的新文件无法正常写入内容，大小保存后为0kb。无法上传新的文件。这说明根目录磁盘空间满了。

### 排查步骤

检查是否确实根目录空间满：

**ssh连接工具如MobaXterm可以设置右下角显示内存空间信息，随时进行观察。**![image-20260107143304560](https://picgo-lenblog.oss-cn-beijing.aliyuncs.com/lenblog/202601071433691.png)

####  **使用命令先查看根分区整体使用情况**

```bash
df -h /
```

输出重点看 `Use%`（使用率）和 `Avail`（可用空间），确认是否真的满了（Use%≥95%）。

![image-20260107143739501](https://picgo-lenblog.oss-cn-beijing.aliyuncs.com/lenblog/202601071437382.png)

这是根目录的使用情况。

**为什么/dev/nvme0n1p3 是根目录？**

> Linux 中所有硬件设备都以文件形式存在于 `/dev` 目录下，这个名称是 NVMe 固态硬盘的分区标识。`/dev/nvme0n1p3` 是第一块 NVMe 硬盘的第 3 个分区，**它成为根目录的唯一原因是系统将其挂载到 `/` 挂载点**；根目录（`/`）是逻辑概念，根分区（`/dev/nvme0n1p3`）是物理存储，二者通过 “挂载” 关联；

#### 用 ncdu 可视化分析

```bash
# 安装ncdu（Ubuntu/Debian） 
sudo apt update && sudo apt install -y ncdu 
# 扫描根分区，跳过无用系统目录（提速） 
sudo ncdu -x / --exclude=/proc --exclude=/sys --exclude=/dev
```

扫描完成后，按 `↑/↓` 导航、`Enter` 进入子目录，直观看到各目录占用占比；

重点关注 `/tmp`、`/home`、`/var`、`/usr` 等目录（根分区的主要占用来源）。

#### 分类清理 / 迁移

| 高占用目录                     | 典型内容                      | 处理方案（优先级从高到低）                                   |
| ------------------------------ | ----------------------------- | ------------------------------------------------------------ |
| /tmp                           | 程序临时文件、大文件夹        | 1. 移动到大容量挂载盘（如 `/home/dl2/data`）；2. 确认无用后直接删除（`sudo rm -rf /tmp/xxx`）；3. 长期：建立软链接 `/tmp` → 大容量盘，避免再次占满。 |
| /home/xxx/anaconda3/miniconda3 | 冗余 Conda 环境               | 1. 用 `which conda` 确认当前使用的版本（如 miniconda3）；删除无用 Conda 环境：`conda env remove -n 环境名`。 |
| /var/log                       | 系统 / 应用日志               | 删除旧日志（`sudo rm -rf /var/log/*.gz`）；                  |
| /var/cache/apt/archives        | apt 安装包缓存                | 清理：`sudo apt clean`（释放数 G 空间）。                    |
| /var/lib/docker                | Docker 镜像 / 容器 / 日志占用 | 查看 Docker 整体空间使用 docker system df                    |

#### 长期优化（避免再次满盘）

这类文件夹的特点是：**存储的是用户数据 / 临时数据 / 应用数据，不是系统核心文件**，迁移后不影响系统启动和基础功能。

#####  不建议操作的文件夹（系统核心目录）

这类文件夹是系统启动 / 运行的核心，迁移 / 改软链接会导致系统崩溃，绝对不能动：

- `/bin`、`/sbin`、`/lib`、`/lib64`（系统核心命令 / 库文件）；
- `/etc`（系统配置文件）；
- `/boot`（系统启动文件）；
- `/proc`、`/sys`、`/dev`（虚拟文件系统，不是实际存储）。

##### 通用操作流程（所有可迁移文件夹都适用）

不管是 Docker 目录、Conda 环境还是日志目录，都遵循 “**迁移数据 → 建立链接 → 验证生效**” 的三步逻辑，以迁移 `/var/lib/docker` 为例（可替换为其他目录）：

步骤 1：停止关联程序（避免文件被占用）

```bash
# 以 Docker 为例，先停止服务
sudo systemctl stop docker
```

步骤 2：迁移原目录数据到大容量盘

```bash
# 1. 在大容量盘创建目标目录
sudo mkdir -p /home/dl2/data/docker

# 2. 迁移原目录数据（rsync 比 cp 更稳定，保留权限）
sudo rsync -avz /var/lib/docker/ /home/dl2/data/docker/
```

步骤 3：备份并删除原目录（可选，确保干净）

```bash
# 备份原目录（防止迁移失败）
sudo mv /var/lib/docker /var/lib/docker.bak

# 或直接删除（确认迁移成功后）
sudo rm -rf /var/lib/docker
```

步骤 4：建立软链接

```bash
sudo ln -s /home/dl2/data/docker /var/lib/docker
```

步骤 5：重启程序并验证

```bash
# 重启 Docker 服务
sudo systemctl start docker

# 验证：查看目录属性（显示软链接）
ls -ld /var/lib/docker

# 验证：程序功能正常（如 Docker 拉取镜像）
docker pull ubuntu:latest
```

##### 关键规则（避坑核心）

**1.权限必须一致：**

迁移后的目录要和原目录权限完全相同（比如 `/var/lib/docker` 是 `root:root`，`/home/dl2/data/docker` 也要设置 `sudo chown root:root /home/dl2/data/docker`），否则程序会因权限不足报错。

**优先用软链接，特殊场景用绑定挂载**：

- 软链接（`ln -s`）：简单易操作，适合普通用户、非系统核心目录；
- 绑定挂载（`/etc/fstab` 配置）：更稳定，适合 Docker、数据库等对路径敏感的程序

**先验证再删除原数据**：

迁移后先启动程序、验证功能正常（比如 Docker 能运行容器、Conda 能激活环境），再删除原目录的备份（如 `/var/lib/docker.bak`），避免数据丢失。

**避免 “嵌套迁移”**：

不要把 `/a/b` 迁移到 `/a/b/c`，也不要把 `/var/log` 迁移到 `/tmp`（如果 `/tmp` 已是软链接），会导致循环引用，程序无法访问。

+++
date = '2025-09-23T12:40:52+08:00'
draft = false
title = '利用Hugo搭建个人博客【极低成本+省运维】'

tags = ["Hugo","博客","教程"]

categories = ["技术记录"]

+++

>  很久以前本人其实搭建过一个博客，服务器域名等等应有尽有，但是无奈写个笔记还得很很繁琐地进行更新，而且服务器次年续费真的好贵！所以现在搭建的新的博客不需要任何的服务器，而且写完笔记更新到网站也不需要自己操作，直接一键更新！

本站点使用Hugo + 阿里云OSS对象服务 + Github Action 实现自动构建静态网页和更新。

### 搭建博客

本教程不需要任何的服务器和复杂代码撰写，完成后也可以拥有很快的访问速度。当你的博客保证更新到一定的频率和文章质量时候，也可以被百度收录和推送。

#### 你将获得的效果

当你完成一个markdown笔记的时候，你可以将其置于Hugo项目的某个文件夹下，然后右键打开终端，运行某个命令，稍等几秒之后你的笔记就将可以通过输入网址访问到（速度很快哦）。

> 这里甚至可以使用windows11自带的定时执行功能执行一个git脚本，每天到点自动帮你推送更新。

### 搭建步骤

#### 前置准备

:signal_strength: 至少需要有一个实名认证的域名，最好是备案过的域名，这样可以使用中国内地区域的OSS对象存储，否则如果不备案就只能使用香港区域的OSS对象存储。

> PS：某种意义上来说，还是使用香港的OSS更好一点，因为域名的备案是不允许使用博客等字眼作为用途的，而厂商会检查域名绑定的网页是否有正常使用备案时候的用途（浏览器标题就得是xx日记、xx生活等）。

域名就像网络上的“门牌号”，让人们能更容易地找到特定的网站。这里我们使用的是OSS对象存储，是不需要服务器的，我们的域名是用来访问到OSS对象存储的Bucket中的资源。推荐注册阿里云域名，便宜的一个也就十几块一年。阿里云购买域名详见：[域名注册_域名查询_域名申请_域名购买_域名续费_国际域名-万网-阿里云品牌](https://wanwang.aliyun.com/domain/?spm=5176.30275541.J_ZGek9Blx07Hclc3Ddt9dg.1.50112f3dY79azu&scm=20140722.S_card@@产品@@3417315.S_new~UND~card.ID_card@@产品@@3417315-RL_域名-LOC_2024SPSearchCard-OR_ser-PAR1_213e371917587893608198175e5d6a-V_4-RE_new6-P0_0-P1_0)

:cloud:需要开通阿里云的OSS对象存储服务（免费开通）。该服务主要用来存储和管理数据，比如图片、视频、文档等。你可以通过它来进行上传、下载和分享文件，还能设置访问权限，保障数据安全。可以理解成一个可以访问资源的网盘。要注意的是阿里云的OSS对象存储服务是按量付费的，所以推荐购买资源包（5元 40GB 一年）。

#### 运行原理

在具体开始之前我们先简单聊一下为什么可以轻松地完成博客的搭建。当前搭建博客的选择已经有很多如Hugo、Hexo等等，其实都是直接将markdown文档与项目结构一起打包，构建好一个静态的HTML网页（包含样式文件），然后将其部署到可被外网访问到的平台上(如服务器或者是OSS对象存储或者是Github Pages)，通过绑定或解析域名访问网页。

![](https://picgo-lenblog.oss-cn-beijing.aliyuncs.com/lenblog/202509251658259.png)

而这其中最繁琐的一步就是，每次写完笔记之后都需要重新完成构建和部署的操作，这会让写博客这个过程变得很啰嗦。所以为了解决频繁的重新部署静态网页的操作，我们这里**将Hugo的项目文件夹存储于Github仓库中，并使用仓库的`Github Action`来自动化帮我们完成部署网页更新的操作**。

![](https://picgo-lenblog.oss-cn-beijing.aliyuncs.com/lenblog/202509251717824.png)

> GitHub Actions 就像是给代码仓库配了一个“智能管家”。当你的代码有新提交、合并请求或者发布新版本时，这个“管家”就会自动帮你运行一系列任务，比如自动测试代码、自动打包、自动部署到服务器，让你不用手动去做这些重复的事，省心又高效。

所以我们最终只需要使用git将本地文件push到代码仓库，后面的流程就会自动执行了。

> push的命令也可以合成一句代码，最终只需要运行一个命令就可以！

#### 搭建过程

##### 阿里云创建OSS对象存储桶

> Bucket 是阿里云 OSS 中用于存储数据的容器

静态站点的html、css、js和图片等都是放在Bucket里的，阿里云的存储是付费的，存储的费用大概在0.12元每月每GB，流量费用大概在0.5元每GB。流量费用如果真的有人看的话，也可以配置CDN缓存，能减少70%-80%的流量费用。但是对于刚开始搭建博客的人来说实在没必要。

> 我们首先保证正常运行，不需要做过多的繁琐配置，如果的确访问量很大可以后续去阿里云配置CDN，操作也非常简单。操作方法我更新在了第二篇博客当中。

打开阿里云OSS对象存储的页面并开通服务，然后创建一个Bucket。

> Bucket的名称和Endpoint我们之后还会用到，所以这里可以留意下（之后可以在控制台看到）。

如果没有备案的域名就创建地域选择香港，有备案就选个离自己近点的地方（比如北京）。存储冗余类型本地冗余就可以的，剩下的配置默认就行，见图所示。Bucket的读取权限这些我们需要创建完之后再设置。

![](https://picgo-lenblog.oss-cn-beijing.aliyuncs.com/lenblog/202509251733158.png)

创建完之后就可以看到Bucket的基本信息了

![image-20250925174434569](https://picgo-lenblog.oss-cn-beijing.aliyuncs.com/lenblog/202509251744621.png)

点击`数据管理` - `静态页面` 配置Bucket中资源默认访问的首页和404页面，在默认情况下，如果你访问站点的根路径时，OSS会返回给你一个错误页面，通常情况下我们希望访问静态站点的某个路径的时候，OSS返回路径下的index.html，这样就能在浏览器上显示内容了，同时地址栏上显示某个路径，而不是具体的文件名，地址栏上显示.html扩展会显得多余，虽然也可以访问。所以“默认首页”需要填上index.html，子目录也保持同样的行为。

> 这些页面不需要自己创建，Hugo之后都会生成，这里只是配置一下而已。

![](https://fookwood.com/posts/publish-hugo-to-aliyun/setup-static-site.png)

##### 域名和证书的设置

通过OSS默认域名访问html等静态资源的时候，浏览器会自动当成文件下载，而不是直接在页面中加载。设置完自定义域名之后，就可以作为页面正常访问了。在OSS对象服务控制台的`Bucket 配置`-`域名管理`–`绑定域名`处进行设置，这里会要求你进行域名的验证。

> 阿里云购买域名后一定要先进行实名认证，在域名相关的控制台可以看到，申请了大概也就一天以内就可以完成。

![image-20250926214455409](https://picgo-lenblog.oss-cn-beijing.aliyuncs.com/lenblog/202509262144456.png)

设置分为两部，第一步是验证域名的所有权，另一步是添加CNAME记录，就是将你的域名映射到默认域名。这里的验证和添加，都是到你的域名解析提供商那里进行配置。

如果你是阿里云的域名可以直接点击后提示自动添加，本人的域名在腾讯云中购买，所以在腾讯云的DNS解析控制台中添加对应的记录。

> 要注意的是，我这里绑定的OSS是香港地区的，所以是没有要求备案的！

![image-20250926214406053](https://picgo-lenblog.oss-cn-beijing.aliyuncs.com/lenblog/202509262144112.png)

![image-20250926214059716](https://picgo-lenblog.oss-cn-beijing.aliyuncs.com/lenblog/202509262141812.png)

> 如果你申请的域名是xxx.com这种类型的，这里只需要填写*_dnsauth*字段就可以了，因为你的域名本来就是顶级域名而不是二级域名。

接下来就是配置SSL证书了。当然你也可以不用，但是浏览器直接访问你的域名网址时候就会提示不安全的连接。这里我们可以申请阿里云的免费个人测试SSL证书，阿里云每年有20个免费证书可以申领，每个证书有效期三个月。

> 根据CA平台规定，现在免费的SSL证书已经没有一次申请用一年的了。好在每次申请很快也很简单。

在申请证书的时候，也是需要进行验证的，就是搞个DNS记录，如果你的域名是放在云解析的，可以自动进行DNS验证，不用自己手动去填写了。有了证书再将证书配置到Bucket，找到刚才配置自定义域名的地方，点击“证书托管”，选择你刚才申请的证书就可以了。你也可以上传在其他地方申请的证书，不麻烦。

![img](https://fookwood.com/posts/publish-hugo-to-aliyun/setup-ssl.png)

如果你的域名解析是在云解析的，整个过程还是非常方便的，可能2分钟就搞定自定义域名和证书配置了。就算你的域名是在其他地方解析的，证书是在其他地方购买的，也可以轻松完成整个配置流程。

> 直接在域名购买处的解析DNS的地方按照阿里云的验证要求添加一条记录就可以了。

##### 简单验证

在Bucket中随便上传一个index.html文件，访问你的域名，就可以浏览静态站点了。

接下来我们看看如何将静态资源文件自动上传到Bucket。

##### Github配置

创建个仓库，用来保存你的文章源码，也就是Hugo的原项目，这里我们可以随便塞一个文件测试Github Action，之后再将Hugo项目的文章源码传上来。

因为流水线在运行的时候，需要权限才能将文件上传到Bucket。首先，你需要在阿里云上创建一个RAM 用户，授予它AliyunOSSFullAccess权限，这样RAM用户就能访问OSS了，你有空的话，可以将这个权限设置的粒度设置更小一点，比如只授权某个Bucket的读写权限，遵循[最小权限原则](https://zh.wikipedia.org/wiki/最小权限原则)。然后生成这个用户的AccessKey，这个是用来完成API操作的。

对于Bucket的访问你需要在仓库上配置以下四个数据：

- OSS_BUCKET: Bucket的名称
- OSS_ENDPOINT: 上面提到的和区域相关的Endpoint
- OSS_ACCESS_KEY_ID: AccessKey的ID
- OSS_ACCESS_KEY_SECRET: AccessKey的Secret

这些敏感数据，放到代码仓库里不太安全，Github提供了在仓库里配置敏感信息的能力，可以配置流水线专用的secret：

![](https://fookwood.com/posts/publish-hugo-to-aliyun/setup-secrets.png)

添加secret之后，就可以在流水线中通过secret的名称访问secret的值了。

[Github Actions](https://github.com/features/actions)是Github推出的自动化工作流工具，主要用来完成CI/CD。它的工作流是通过yaml文件定义的，放在要部署的仓库的`.github/workflow`目录下。如果你没有空去看Github的文档来看yaml文件格式的定义，那么直接看下面的代码,这个可以直接拿来粘贴，只要配置的secret和我一样就行。

```yaml
name: 发布博客到OSS

on:
  push:
    branches: ["main"]
  workflow_dispatch:

jobs:
  publish:
    runs-on: ubuntu-latest
    
    steps:
      - name: 验证Secrets配置
        run: |
          echo "=== 开始验证Secrets配置 ==="
          echo "OSS_ENDPOINT: ${{ secrets.OSS_ENDPOINT }}"
          echo "OSS_BUCKET: ${{ secrets.OSS_BUCKET }}"
          echo "OSS_ACCESS_KEY_ID: ${{ secrets.OSS_ACCESS_KEY_ID }}"
          echo "OSS_ACCESS_KEY_SECRET: ${{ secrets.OSS_ACCESS_KEY_SECRET }}"
          
          # 检查每个secret是否为空
          if [ -z "${{ secrets.OSS_ENDPOINT }}" ]; then
            echo "❌ OSS_ENDPOINT 为空或未设置"
            exit 1
          else
            echo "✅ OSS_ENDPOINT 已配置"
          fi
          
          if [ -z "${{ secrets.OSS_BUCKET }}" ]; then
            echo "❌ OSS_BUCKET 为空或未设置"
            exit 1
          else
            echo "✅ OSS_BUCKET 已配置"
          fi
          
          if [ -z "${{ secrets.OSS_ACCESS_KEY_ID }}" ]; then
            echo "❌ OSS_ACCESS_KEY_ID 为空或未设置"
            exit 1
          else
            echo "✅ OSS_ACCESS_KEY_ID 已配置"
          fi
          
          if [ -z "${{ secrets.OSS_ACCESS_KEY_SECRET }}" ]; then
            echo "❌ OSS_ACCESS_KEY_SECRET 为空或未设置"
            exit 1
          else
            echo "✅ OSS_ACCESS_KEY_SECRET 已配置"
          fi
          
          echo "=== Secrets验证完成 ==="

      - name: 检出代码
        uses: actions/checkout@v4
        with:
          submodules: true
          fetch-depth: 0

      - name: 安装 Hugo v0.145.0
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: '0.145.0'
          extended: true

      - name: 构建网站
        run: hugo --minify


      - name: 配置阿里云OSS 
        uses: yizhoumo/setup-ossutil@v2
        with:
          ossutil-version: 'latest'
          endpoint: ${{ secrets.OSS_ENDPOINT }}
          access-key-id: ${{ secrets.OSS_ACCESS_KEY_ID }}
          access-key-secret: ${{ secrets.OSS_ACCESS_KEY_SECRET }}

      - name: 上传资源
        run: |
          ossutil rm -rf oss://${{ secrets.OSS_BUCKET }}
          ossutil cp -rf public oss://${{ secrets.OSS_BUCKET }}
```

> 这段 YAML 配置定义了一个 GitHub Actions 工作流，用于将博客发布到阿里云 OSS。它在 `main` 分支推送或手动触发时运行，验证必要的 Secrets 配置，检出代码，安装 Hugo 并构建网站，配置阿里云 OSS 工具，并将生成的网站资源上传到指定的 OSS Bucket。

你把流水线的配置文件添加到仓库里之后，push下代码，流水线就可以执行了。你自己在操作的时候多少会出现一些问题，不过看看错误报告基本都能很快解决。整个流程非常之快，只需要不到10s就可以自动执行完毕。





#### 可选功能 图床

之后再来更新这个

#### 可选功能 评论

见这个

在 Windows 上安装 Hugo 来搭建博客，步骤清晰。虽然你提供的 Hugo 官方安装页面是权威参考，下面我为你整合了更具体的安装流程和后续操作步骤，并用一个表格来汇总关键信息，方便你快速把握全局。

| 组件名称            | 是否必需       | 主要作用                         | 备注                                               |
| :------------------ | :------------- | :------------------------------- | :------------------------------------------------- |
| **Hugo 二进制文件** | **是**         | 静态网站生成器的核心程序         | 建议下载 **扩展版（extended）** 以支持更多主题功能 |
| **Git**             | 强烈推荐       | 用于下载和管理主题               | 许多主题通过 Git 仓库分发                          |
| **Go 语言**         | 仅源码安装需要 | 如果你选择从源码编译 Hugo 才需要 | 大多数用户直接使用预编译二进制文件，无需安装 Go    |

🧰 安装前的准备：获取必要工具

1.  **安装 Git**：前往 [Git 官网](https://git-scm.com/downloads) 下载 Windows 版本的安装程序，并按照提示完成安装。安装后，你可以在命令行中使用 `git` 命令，这对于后续拉取主题非常方便。

📥 安装 Hugo 核心程序

对于大多数用户，最推荐的方法是直接使用**预编译的二进制文件**，过程非常简单：

1.  **下载 Hugo**：访问 Hugo 在 GitHub 上的 [发布页面](https://github.com/gohugoio/hugo/releases)。寻找名称中包含 **`hugo_extended`** 以及 `windows-amd64` 的最新版本压缩包（例如 `hugo_extended_0.128.2_windows-amd64.zip`）并下载。
2.  **解压文件**：将下载的 ZIP 文件解压到一个你打算长期存放的目录，例如 `C:\hugo\bin` 或 `D:\hugo\bin`。解压后，你通常会得到一个 `hugo.exe` 可执行文件。
3.  **配置环境变量**：这是关键一步，为了让系统在任何路径下都能识别 `hugo` 命令。
    *   在 Windows 搜索栏输入“环境变量”，选择“编辑系统环境变量”。
    *   在弹出的系统属性窗口中，点击“环境变量...”按钮。
    *   在“系统变量”区域，找到并选中名为 `Path` 的变量，然后点击“编辑...”。
    *   点击“新建”，将你刚才存放 `hugo.exe` 的目录路径（例如 `C:\hugo\bin`）添加进去。
    *   依次点击“确定”关闭所有窗口。
4.  **验证安装**：打开一个新的命令提示符（CMD）或 PowerShell 窗口，输入以下命令并回车：
    ```bash
    hugo version
    ```
    如果安装成功，屏幕上会显示 Hugo 的版本信息，例如 `Hugo Static Site Generator v0.128.2`。

🚀 快速启动你的第一个博客站点

安装好 Hugo 后，你可以快速在本地创建和预览一个博客网站。

1.  **创建站点**：在命令行中，导航到你希望创建博客项目的目录，然后运行以下命令（`myblog` 可以替换为你喜欢的任何文件夹名）：
    ```bash
    hugo new site myblog
    ```
    这会在当前目录下生成一个名为 `myblog` 的文件夹，里面包含了 Hugo 站点的基本结构。
2.  **添加主题**：Hugo 的强大之处在于有丰富的主题可选。以添加一个名为 `ananke` 的流行主题为例：
    *   进入你的站点目录：
        ```bash
        cd myblog
        ```
    *   使用 Git 将主题克隆到项目的 `themes` 目录下：
        ```bash
        git clone https://github.com/budparr/gohugo-theme-ananke.git themes/ananke
        ```
    *   编辑站点根目录下的配置文件 `config.toml`（或 `hugo.toml`），添加一行来指定主题：
        ```toml
        theme = "ananke"
        ```
3.  **创建内容与预览**：
    *   使用命令创建你的第一篇文章：
        ```bash
        hugo new posts/my-first-post.md
        ```
      文章文件（Markdown 格式）会生成在 `content/posts/` 目录下。用文本编辑器打开它，你会看到文件顶部有类似这样的前置元数据（Front Matter）：
        ```markdown
        ---
        title: "My First Post"
        date: 2025-09-23T15:30:00+08:00
        draft: true
        ---
        ```
      将 `draft: true` 改为 `draft: false` 表示这不是草稿，然后可以在 `---` 下方用 Markdown 语法撰写正文。
    *   在站点根目录下启动本地服务器：
        ```bash
        hugo server -D
        ```
        （`-D` 参数表示包含标记为草稿的文章）。
    *   命令行中会输出一个本地访问地址（通常是 `http://localhost:1313`），在浏览器中打开它，你就能看到你的博客站点了！

⚠️ 常见问题与小贴士

*   **命令未找到**：如果在输入 `hugo version` 时提示“无法识别”，请检查环境变量是否设置正确，并确保已经**重新启动了命令行窗口**。
*   **主题不显示**：请确保在 `config.toml` 文件中正确指定了主题名称，且主题已成功下载到 `themes` 目录下对应的文件夹中。
*   **使用包管理器（可选）**：如果你熟悉包管理器，也可以使用 **Chocolatey** (`choco install hugo -confirm`) 或 **Scoop** (`scoop install hugo`) 来安装 Hugo，这可能更方便后续更新。




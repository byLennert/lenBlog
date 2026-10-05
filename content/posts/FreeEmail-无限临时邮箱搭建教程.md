+++
date = '2026-10-06'
draft = false
title = ' 0 成本搭建一个「无限」临时邮箱：Cloudflare + FreeEmail 完整实战教程'

tags = ["cloudflare","临时邮箱注册"]

categories = ["教程"]

+++

从域名申请到收到第一封邮件，全程免费，全程踩坑复盘。

> 本文基于 [idinging/freemail](https://github.com/idinging/freemail) 项目（Apache-2.0 开源）。

---

## 目录

- [一、这是什么？能干什么？](#一这是什么能干什么)
- [二、原理：为什么它敢叫「无限」](#二原理为什么它敢叫无限)
- [三、准备工作](#三准备工作)
- [四、第一步：把域名接入 Cloudflare](#四第一步把域名接入-cloudflare)
- [五、第二步：部署 FreeEmail](#五第二步部署-freemail)
- [六、第三步：打通收信链路](#六第三步打通收信链路)
- [七、第四步：开始使用](#七第四步开始使用)
- [八、踩坑记录（真实报错复盘）](#八踩坑记录真实报错复盘)
- [九、安全须知（必读）](#九安全须知必读)
- [十、成本与限额](#十成本与限额)
- [十一、进阶：绑定自定义域名收发信](#十一进阶绑定自定义域名收发信)
- [十二、总结](#十二总结)

---

## 一、这是什么？能干什么？

一个**属于你自己的临时邮箱系统**。

和网上那些公共临时邮箱（比如 temp-mail 之类）最大的区别是：

|                              | 公共临时邮箱       | 自建（本文方案）                 |
| ---------------------------- | ------------------ | -------------------------------- |
| 域名                         | 别人的             | **你自己的**                     |
| 地址                         | 别人分配           | **想叫什么就叫什么**             |
| 隐私                         | 邮件在别人服务器上 | 在**你自己的** Cloudflare 账号里 |
| 能不能查到是谁泄露了你的邮箱 | 不能               | **能**（按网站命名即可）         |
| 成本                         | 免费               | 免费                             |

**典型用途**：

- 注册各种网站 / 试用服务，不想暴露真实邮箱
- 收验证码（这是最核心的场景）
- 隔离营销邮件、垃圾邮件
- 给每个网站分配**独立邮箱**，谁泄露了一眼就知道

**我实际用下来的感受**：最适合的场景就是「按网站起名」——比如 `github@你的域名`、`taobao@你的域名`。哪天某个地址开始收到垃圾邮件，你就知道是谁把数据卖了。

---

## 二、原理：为什么它敢叫「无限」

先看架构：

```
       别人给你发邮件
              ↓
   ┌──────────────────────┐
   │  Cloudflare 收信服务器 │  MX 记录指向这里
   └──────────┬───────────┘
              ↓  Catch-all（全收）规则
   ┌──────────────────────┐
   │  Cloudflare Worker    │  ← FreeEmail 程序跑在这里
   │   ├── 解析邮件         │
   │   ├── 提取验证码       │
   │   ├── 邮箱不存在就自动建 │  ← 这就是「无限」的来源
   │   └── 存进数据库       │
   └──────────┬───────────┘
              ↓
   ┌──────────┴───────────┐
   │  D1 数据库  │  R2 存储  │
   │  （索引）    │ （邮件原文）│
   └──────────────────────┘
              ↓
      你打开网页看验证码
```

**「无限」的两个含义**：

**① 地址无限**

你**不需要提前创建邮箱**。项目的收信逻辑是：

```
收到邮件 → 查这个地址存在吗 → 不存在就直接建一个
```

所以你随手编一个 `whatever@你的域名` 去注册网站，来信照样收得到，后台也会自动出现这个邮箱。

**② 成本为 0**

整套东西跑在 Cloudflare 的免费额度里，个人使用**根本用不完**。

> ⚠️ 诚实说明：所谓"无限"是**地址数量无限、日常够用**，不是数学意义的无限。Cloudflare 免费额度、单个邮件大小限制等依然存在，详见[第十章](#十成本与限额)。

---

## 三、准备工作

你需要三样东西，**全部免费**：

### 1. Cloudflare 账号

👉 https://dash.cloudflare.com/sign-up

### 2. GitHub 账号

👉 https://github.com/signup

（用来托管代码并触发自动部署，不用会写代码）

### 3. 一个域名 ⚠️ 这里有个大坑

**这是整个教程最关键的前置知识，搞错了后面全白搭。**

Cloudflare 的**全收（Catch-all）规则只支持 Zone 的根域**，不支持子域：

> "Once the records propagate, you can create literal routing rules for addresses on the subdomain. **Catch-all rules are only available for the apex domain.**"
> —— [Cloudflare 官方文档](https://developers.cloudflare.com/email-service/configuration/subdomains/)

而 FreeEmail 恰恰**依赖全收**（任意地址自动建邮箱）。所以：

#### ✅ 方案 A：用自己的域名（推荐）

任何你拥有的域名都行，`example.com` 直接拿来当邮箱域。

#### ✅ 方案 B：免费域名（0 成本方案）

[DigitalPlat FreeDomain](https://dash.domain.digitalplat.org/) 提供免费二级域名，可选后缀：
`.dpdns.org`、`.us.kg`、`.qzz.io`、`.xx.kg`、`.qd.je`

**但是有个硬性条件**：这个后缀必须被收录进 **Public Suffix List（PSL）**。

为什么？因为 Cloudflare 判断"你是不是一个可注册的根域"靠的就是 PSL。如果后缀不在 PSL 里，你添加站点时会直接报错：

```
We were unable to identify xxx as a registered domain.
Please ensure you are providing the root domain and not any subdomains (Code: 1099)
```

**好消息**：`dpdns.org` **已经在 PSL 里**（[Vercel 官方已确认](https://community.vercel.com/t/request-to-add-dpdns-org-to-the-public-suffix-list/9202)），所以你可以注册 `yourname.dpdns.org`，它能作为独立 Zone 接进 Cloudflare。

> 💡 **为什么 `yourname.dpdns.org` 可以，但 `mail.你的域名.com` 不行？**
> 因为前者在 PSL 眼里**自己就是一个根域**（它是它自己 Zone 的 apex，全收规则可用）；
> 而后者只是某个 Zone 下的子域，全收规则用不了。

#### ❌ 方案 C：用已有域名的子域

**不可行**，原因如上。别在这上面浪费时间。

> 📌 本文后续统一用 `yourname.dpdns.org` 作为示例，请自行替换成你的域名。

---

## 四、第一步：把域名接入 Cloudflare

Cloudflare Email Routing **要求域名的 DNS 由 Cloudflare 托管**，所以必须先做 NS 委派。

### 4.1 在 Cloudflare 添加站点

1. 打开 https://dash.cloudflare.com
2. 首页点 **+ Add** → **Add a site**（不同版本可能叫 "Connect a domain"）
3. 输入 **`yourname.dpdns.org`**
   - ⚠️ **填完整域名**，不要只填 `dpdns.org`
4. 计划选 **Free**
5. 扫描 DNS 记录时会说没找到——正常，直接继续
6. 最后它会给你**两个 NS 地址**，形如：

```
alexis.ns.cloudflare.com
tia.ns.cloudflare.com
```

**把这两个复制下来。**

### 4.2 去域名商填写 NS

以 DigitalPlat 为例：

1. 打开 https://dash.domain.digitalplat.org/
2. 登录 → 找到你的域名
3. 找到 **Nameservers**（自定义 NS）设置
4. 把上面两个 NS **都填进去**
   - ⚠️ **两个都要填**
   - ⚠️ **不要填 IP 地址**
5. 保存

> 📖 官方说明：[Connect External Nameservers](https://github.com/DigitalPlatDev/FreeDomain/blob/main/documents/tutorial/platform/1.3-connect-nameservers.md)（DigitalPlat 只负责委派，不提供普通 DNS 记录编辑）

### 4.3 等待生效

通常 5~30 分钟，慢的话几小时。

**检查方法**：👉 https://www.whatsmydns.net/#NS/yourname.dpdns.org

看到所有节点都返回 Cloudflare 的 NS 就成了（个别节点打 ✗ 是缓存问题，正常）。

也可以在命令行查：

```bash
dig NS yourname.dpdns.org
```

### 4.4 确认状态为 Active

回 Cloudflare 控制台，点你的域名：

- 显示 **Active** ✅ → 继续下一步
- 显示 **Pending** → 页面上有 **Check nameservers now** 按钮，点一下触发重新检查

---

## 五、第二步：部署 FreeEmail

### 5.1 Fork 项目

1. 打开 https://github.com/idinging/freemail
2. 点右上角 **Fork** → **Create fork**

完成后你就有了 `你的用户名/freemail`。

### 5.2 获取 Cloudflare API Token ⚠️ 关键步骤

**这里是最容易踩的坑。** 项目文档明确要求：

> **核心权限要求**：请务必确保授权清单中包含对 `Workers Scripts`、`D1`、`R2` 以及 `Workers KV Storage` 的**编辑**权限。

而 Cloudflare 自带的 **"Edit Cloudflare Workers" 模板默认不含 D1 权限**，用了它部署会卡在数据库那一步报 `Authentication error [code: 10000]`。

**所以必须自建自定义 Token：**

1. 打开 👉 https://dash.cloudflare.com/profile/api-tokens
2. 点 **Create Token**
3. **不要选上面的模板**，拉到最底部点 **Create Custom Token** → **Get started**
4. **Token name**：`freemail-deploy`
5. **Permissions** 点 **+ Add more**，逐条添加：

| 类别    | 资源               | 权限     |
| ------- | ------------------ | -------- |
| Account | Workers Scripts    | **Edit** |
| Account | D1                 | **Edit** |
| Account | Workers R2 Storage | **Edit** |
| Account | Workers KV Storage | **Edit** |
| Account | Account Settings   | **Read** |

6. **Account Resources** → `Include` → 选你的账号
7. **Continue to summary** → **Create Token**
8. **复制 Token**（只显示一次）

同时记下 **Account ID**（在 Cloudflare 首页右侧栏，或域名 Overview 页右下角）。

> ⚠️ **Token 复制时注意首尾不要带空格或换行**，多一个空格就会报一模一样的认证错误。

### 5.3 开通 R2 存储 ⚠️ 需要绑定支付方式

FreeEmail 用 R2 存邮件原文，所以**必须开通 R2**。

未开通时报错：

```
Please enable R2 through the Cloudflare Dashboard. [code: 10042]
```

**开通步骤**：

1. Cloudflare 控制台 → 左侧 **R2**
2. 点 **Enable R2** / **Purchase R2 Plan**
3. 按提示**添加支付方式**（信用卡或 PayPal）
4. 同意条款 → 开通
5. 看到 **Create bucket** 按钮 = 成功

> 💰 **关于收费**：R2 需要绑卡才能开通（这是 Cloudflare 的官方政策），但**免费额度内不扣费**——10 GB 存储 + 每月百万次写入。收验证码一年也用不到 1 MB，不用担心。

### 5.4 配置 GitHub Secrets

打开你 Fork 的仓库 → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

添加以下 5 条（名字必须**完全一致**）：

| Name                    | Value                | 说明                           |
| ----------------------- | -------------------- | ------------------------------ |
| `CLOUDFLARE_API_TOKEN`  | 5.2 生成的 Token     | **必填**                       |
| `CLOUDFLARE_ACCOUNT_ID` | 你的 Account ID      | **必填**                       |
| `ADMIN_PASSWORD`        | 你自己设的强密码     | **必填**，留空会导致登不进后台 |
| `JWT_TOKEN`             | 一串长随机字符串     | **必填，务必换掉默认值**       |
| `MAIL_DOMAIN`           | `yourname.dpdns.org` | **必填**                       |

**可选**（不填就用默认值）：

| Name                  | 默认值         | 作用                    |
| --------------------- | -------------- | ----------------------- |
| `NAME`                | `mailfree`     | Worker 名称             |
| `D1_DB_NAME`          | `mail_free_db` | 数据库名                |
| `R2_BUCKET_NAME`      | `mail-eml`     | 存储桶名                |
| `ADMIN_NAME`          | `admin`        | 后台用户名              |
| `SESSION_EXPIRE_DAYS` | `365`          | 会话有效期（建议改 30） |
| `GUEST_PASSWORD`      | 空             | 留空 = 关闭访客模式     |

#### 生成一个安全的 JWT_TOKEN

**绝对不要用仓库里的默认值**（`dfvhluhdaslufhvdv`，它是公开的）。

生成方法（任选）：

```bash
# Linux / macOS
openssl rand -hex 32

# 或者用密码管理器自带的生成器（推荐）
```

得到一串 64 位十六进制字符即可。

### 5.5 手动触发部署 ⚠️ 这是个坑

**这个项目不会自动部署。** 工作流里写的是：

```yaml
on:
  push:
    branches: [ main ]   # ← 但仓库默认分支是 master，永远匹配不上
```

所以 push 代码**不会触发**，必须手动运行：

1. 仓库 → **Actions** 标签
2. 左侧选 **🚀 Deploy freemail to Cloudflare Workers**
3. 右侧点 **Run workflow** → 绿色按钮
4. 等待 2~3 分钟，看到 ✅ 绿勾

**工作流会自动完成**：

- 创建 D1 数据库（不存在则新建）
- 创建 R2 存储桶（不存在则新建）
- 执行建表 SQL
- 部署 Worker
- 注入你配置的环境变量

> 💡 部署日志里**看不到 Worker URL**——工作流故意把它过滤并屏蔽了（`grep -v` + `::add-mask::`）。别在日志里找。

### 5.6 找到你的网址

1. Cloudflare 控制台 → **Workers & Pages**
2. 找到 **`mailfree`** 项目，点进去
3. 页面右上角就是你的地址：

```
https://mailfree.你的子域.workers.dev
```

**打开它，用 `admin` + 你设的 `ADMIN_PASSWORD` 登录。**

能进后台 = 部署成功 ✅

---

## 六、第三步：打通收信链路

**这一步不做，邮件收不到。**（我就在这里卡了一轮，收到退信才反应过来）

### 6.1 启用 Email Routing

1. Cloudflare 控制台 → 点 **`yourname.dpdns.org`**
2. 左侧 **Email** → **Email Routing**
3. 点 **Enable Email Routing** / **启用**
4. 它列出需要的 MX 记录 → 点 **Add records automatically** / **自动添加记录**
5. 确认

### 6.2 配置全收（Catch-all）🔴 最关键的一步

1. 在 Email Routing 页面点 **Routing rules**（路由规则）标签
2. 找到 **Catch-all address**（全收 / 兜底地址）
3. 点 **Edit**（编辑）
4. 设置：

| 项目                    | 选择                 |
| ----------------------- | -------------------- |
| **Action**（操作）      | **Send to a Worker** |
| **Destination**（目标） | **`mailfree`**       |

5. 点 **Save**

6. 保存后应该看到这样一行：

```
路由规则    操作         状态
全收        mailfree     活跃 ✅
```

> 🔴 **这一步是整个项目的命门。** MX 记录只负责让邮件"到达 Cloudflare"，全收规则才决定邮件"送进你的程序"。漏了它，你会收到 `550 5.1.1 Address does not exist` 退信。

### 6.3 测试收信

1. 用 QQ / Gmail **发一封**到：

```
任意名字@yourname.dpdns.org
```

2. 等 10~30 秒
3. 回 FreeEmail 后台 → **所有邮箱** → 应该能看到这个地址和那封信

**收到了 = 全线打通 🎉**

---

## 七、第四步：开始使用

### 7.1 三种创建邮箱的方式

| 方式           | 操作               | 适合                            |
| -------------- | ------------------ | ------------------------------- |
| **随机生成**   | 首页点生成         | 一次性用完就扔                  |
| **自定义创建** | 输入你记得住的名字 | **推荐**，如 `github`、`taobao` |
| **直接不创建** | 拿任意地址去注册   | 最省事，来信自动建              |

### 7.2 推荐用法：按网站起名

这是自用最舒服的模式：

```
① 创建 github@yourname.dpdns.org
② 设置「转发到」= 你的真实邮箱
③ 拿这个地址去注册 GitHub
④ 之后 GitHub 的来信：
   - FreeEmail 后台能看到（验证码已自动提取）
   - 同时转发一份到你的真实邮箱 ← 不用一直开着网页
```

**好处**：哪个网站开始发垃圾邮件，你一眼就知道是谁泄露的。

### 7.3 配置转发到真实邮箱

转发目标**必须先在 Cloudflare 验证**，否则转发会被拒：

1. Cloudflare → `yourname.dpdns.org` → **Email Routing** → **Destination addresses**（目标地址）
2. 点 **Add destination address** → 输入你的 QQ / Gmail
3. **去那个邮箱点验证链接**
4. 回 FreeEmail 后台 → 进某个邮箱 → 设置 **转发到** → 填已验证的地址

> 也可以配置全局规则（环境变量 `FORWARD_RULES`），支持前缀匹配，`*` 为兜底：
>
> ```
> FORWARD_RULES="vip=me@qq.com,*=fallback@qq.com"
> ```

### 7.4 验证码自动提取

这是这个项目最实用的功能。收到邮件时，程序会自动从主题和正文里提取 4~8 位数字验证码，直接显示在邮件列表里，**还会顺手过滤掉年份、邮编之类的干扰数字**。

邮件详情页有**一键复制**按钮。

### 7.5 用 API 做自动化

带上你的 `JWT_TOKEN` 就能直接调 API，不需要登录、不需要 Cookie：

```bash
TOKEN="你的JWT_TOKEN"
BASE="https://mailfree.你的子域.workers.dev"

# 查看所有邮箱
curl -H "Authorization: Bearer $TOKEN" "$BASE/api/mailboxes"

# 收验证码（列表里直接带 verification_code）
curl -H "Authorization: Bearer $TOKEN" \
  "$BASE/api/emails?mailbox=github@yourname.dpdns.org&limit=10"

# 读某封邮件全文
curl -H "Authorization: Bearer $TOKEN" "$BASE/api/email/123"

# 创建指定邮箱
curl -H "Authorization: Bearer $TOKEN" -X POST "$BASE/api/create" \
  -H 'Content-Type: application/json' \
  -d '{"local":"github","domainIndex":0}'

# 清空某邮箱邮件（会连带删除 R2 原文）
curl -H "Authorization: Bearer $TOKEN" -X DELETE \
  "$BASE/api/emails?mailbox=github@yourname.dpdns.org"
```

> ⚠️ `JWT_TOKEN` 同时也是**超管令牌**，只能在自己可信的机器上使用，**绝对不要写进前端代码或公开仓库**。

---

## 八、踩坑记录（真实报错复盘）

这一章是我实际搭建过程中踩过的所有坑，按发生顺序排列：

| 报错 / 现象                                                  | 根本原因                                               | 解决办法                                          |
| ------------------------------------------------------------ | ------------------------------------------------------ | ------------------------------------------------- |
| `Authentication error [code: 10000]`                         | "Edit Cloudflare Workers" 模板**不含 D1 权限**         | 改用自定义 Token，手动勾上 D1 / R2 / KV 的 Edit   |
| `Please enable R2 through the Cloudflare Dashboard. [code: 10042]` | 账号还没开通 R2                                        | 去控制台开通 R2（需绑支付方式，免费额度内不收费） |
| 添加站点报 `Code: 1099`                                      | 域名后缀不在 PSL 里，Cloudflare 不认它是根域           | 换一个已在 PSL 的域名（如 `.dpdns.org`）          |
| push 代码后工作流不触发                                      | 工作流写的是 `branches: [main]`，但默认分支是 `master` | 手动 **Actions → Run workflow**                   |
| 收信退信 `550 5.1.1 Address does not exist`                  | MX 通了，但**没配全收规则**                            | 配 Catch-all → Send to a Worker                   |
| 部署日志里找不到 Worker URL                                  | 工作流主动屏蔽了它                                     | 去 Cloudflare → Workers & Pages 里看              |
| 邮件日志里看不到记录                                         | Cloudflare 日志**有几分钟延迟**                        | 等几分钟再刷新，别急着下结论                      |
| 后台发信报「未找到域名对应的发件 API Key」                   | 没配发信渠道                                           | 见[第十一章](#十一进阶绑定自定义域名收发信)       |

---

## 九、安全须知（必读）

这套系统**架构上是自用工具**，直接公网开放给陌生人用会有一堆问题。以下几条务必做到：

### 9.1 必须改掉 `JWT_TOKEN` 的默认值 🔴

仓库的 `wrangler.toml` 里带着一个**公开的默认值**：

```toml
JWT_TOKEN="dfvhluhdaslufhvdv"
```

而代码里有个「根管理员令牌」机制：

```js
// src/middleware/auth.js
if (bearer && bearer === JWT_TOKEN) return { role: 'admin', username: '__root__', userId: 0 };
```

**也就是说：任何知道你域名的人，只要带上这个公开字符串，就能免密成为最高管理员，读走你所有邮件。**

```bash
# 攻击者只需这一条命令
curl -H "Authorization: Bearer dfvhluhdaslufhvdv" https://你的域名/api/mailboxes
```

**必须换成强随机值**，这是整套部署里唯一不能省的一步。

### 9.2 不要给不信任的人开普通用户账号 🔴

项目存在一个**横向越权漏洞**（截至本文写作时 master 分支仍未修复）：

- `POST /api/create` 允许用户指定任意地址
- 但 `assignMailboxToUser()` **没有检查该邮箱是否已被他人绑定**
- 而 `user_mailboxes` 表唯一约束是 `(user_id, mailbox_id)`，**不阻止第二个用户绑定同一个邮箱**

**利用链**：

```
普通用户 POST /api/create {"local":"受害者"} 
  → 认领受害者邮箱
  → GET /api/emails?mailbox=受害者 读全部来信（含验证码）
  → POST /api/mailbox/forward 静默把后续来信转发到攻击者邮箱
```

**规避方法**：只留 `admin` 一个账号自用；确实要分享，用「单个邮箱登录」（开启某邮箱的 `can_login` 并设密码），别开 `user` 角色账号。

### 9.3 其他注意事项

| 项                        | 说明                                                         |
| ------------------------- | ------------------------------------------------------------ |
| `ADMIN_PASSWORD` 不能留空 | 留空等于**禁用管理员口令登录**，你自己会进不去               |
| 关闭访客模式              | `GUEST_PASSWORD` 留空即可（访客模式走的是假数据，但没必要开） |
| 会话时长                  | `SESSION_EXPIRE_DAYS` 默认 365 天，建议改 30                 |
| 收信无配额                | 任意地址来信都会自动建邮箱，**垃圾邮件会堆积**，建议定期清理 |
| 删邮箱不删文件            | `DELETE /api/mailboxes` **只删数据库记录，不删 R2 原文**。想彻底清理：先 `DELETE /api/emails` 清空邮件，再删邮箱 |
| 没有自动过期              | `expires_at` 字段写了但从不生效，没有任何清理任务，建议每几个月手动清一次 |

---

## 十、成本与限额

### 10.1 实际花费

| 项目                     | 花费                                 |
| ------------------------ | ------------------------------------ |
| Cloudflare Workers       | **0 元**（免费额度内）               |
| Cloudflare D1            | **0 元**                             |
| Cloudflare R2            | **0 元**（需绑卡，免费额度内不扣费） |
| Cloudflare Email Routing | **0 元**（收信免费）                 |
| 域名                     | 自有域名 ¥0 额外成本 / 免费域名 ¥0   |
| GitHub Actions           | **0 元**（公开仓库免费）             |
| **合计**                 | **0 元**                             |

### 10.2 免费额度参考

| 服务          | 免费额度                                  | 备注                      |
| ------------- | ----------------------------------------- | ------------------------- |
| Workers       | 10 万请求/天                              | 个人用绰绰有余            |
| D1            | 5 GB 存储、500 万行读/天、10 万行写/天    | —                         |
| R2            | 10 GB 存储、100 万次写/月、1000 万次读/月 | —                         |
| Email Routing | 单封最大 25 MiB                           | 每域名最多 200 条路由规则 |

### 10.3 发信要额外花钱吗？

**不用。** 但需要配第三方渠道（默认没配，所以只能收不能发）。

| 渠道                               | 免费额度                                   | 付费            |
| ---------------------------------- | ------------------------------------------ | --------------- |
| [Resend](https://resend.com)       | 3,000 封/月（**每天限 100 封**），1 个域名 | $20/月 = 5 万封 |
| [SendFlare](https://sendflare.com) | 3,000 封/月，2 个域名                      | 有付费版        |
| Cyberpersons                       | CyberPanel 的邮件投递服务                  | —               |

> ⚠️ **但是——如果你用的是免费域名后缀（如 `.dpdns.org`），发信送达率会很差。**
> 这类公共后缀在 Gmail / Outlook 眼里信誉天然偏低，邮件容易进垃圾箱甚至被直接拒收。这是免费域名的固有问题，配置解决不了。
>
> **结论：用它收验证码就好，发信还是用真实邮箱。**

---

## 十一、进阶：绑定自定义域名收发信

### 11.1 给后台配一个好记的地址

默认的 `xxx.workers.dev` 不好记，可以绑自定义域名：

1. Cloudflare → **Workers & Pages** → `mailfree`
2. **Settings** → **Domains & Routes** → **Add** → **Custom domain**
3. 输入 `mail.yourname.dpdns.org` → 添加

> 这是普通的网站记录，跟邮箱的 MX 记录**不冲突**，可以同时存在。

### 11.2 配置发信（可选）

以 Resend 为例：

1. 注册 https://resend.com
2. **Domains** → **Add Domain** → 填 `yourname.dpdns.org`
3. 按提示去 Cloudflare DNS 添加它给的 **DKIM / SPF** 记录
4. 回 Resend 点 **Verify**，等验证通过
5. 生成 **API Key** → 复制
6. Cloudflare → Workers & Pages → `mailfree` → **Settings** → **Variables**
   → 添加变量 `RESEND_API_KEY` = 你的 Key
7. 按提示**保存并部署**（Dashboard 改完变量需要重新部署才生效）

---

## 十二、总结

### 这套方案的优势

- ✅ **真·0 成本**，跑在 Cloudflare 免费额度里
- ✅ **地址无限**，来信自动建邮箱，不用提前准备
- ✅ **验证码自动提取**，这是最省事的功能
- ✅ **发件渠道插件化**，支持 Resend / SendFlare / Cyberpersons，可按域名路由
- ✅ **数据在你自己账号里**，不像公共临时邮箱那样把邮件交给第三方

### 需要接受的不足

- ⚠️ 免费域名后缀**发信送达率差**，只适合收
- ⚠️ 免费域名的**父域不在你手上**，运营方策略变化或域名过期会导致邮箱全失效
- ⚠️ 项目存在未修复的**越权漏洞**，不适合开放给多人使用
- ⚠️ 没有**自动清理机制**，需要定期手动清理 D1 / R2

### 最后的建议

**适合**：个人自用收验证码、隔离垃圾邮件、给每个网站分配独立邮箱。

**不适合**：承载重要账号的找回邮箱、作为正式的对外收发邮箱、开放给不特定用户注册使用。

**尤其是免费域名**——别把它当成重要账号的唯一邮箱，哪天域名没了就全完了。

---

## 附录：常用链接

| 用途                      | 链接                                                         |
| ------------------------- | ------------------------------------------------------------ |
| FreeEmail 项目            | https://github.com/idinging/freemail                         |
| Cloudflare 控制台         | https://dash.cloudflare.com                                  |
| Cloudflare API Token 管理 | https://dash.cloudflare.com/profile/api-tokens               |
| DigitalPlat 免费域名      | https://dash.domain.digitalplat.org/                         |
| DNS 传播检查              | https://www.whatsmydns.net/                                  |
| Resend（发信，可选）      | https://resend.com                                           |
| 官方部署文档              | https://github.com/idinging/freemail/blob/master/docs/action-deployment.md |

---

*本文基于实际搭建过程记录整理，所有报错均为真实遇到。项目版本：V5.3.1。*

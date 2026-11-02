# VPNGate SSTP 家宽节点（edgetunnel 链式代理） 🚀

自动抓取 [VPN Gate](https://www.vpngate.net/) 的 SSTP 家宽/机房节点，调用检测 Worker 逐个验证可用性，按国家分组、标注住宅/机房，生成供 edgetunnel 读取的节点清单。工作流配置为每 30 分钟尝试更新，成功检测并发布后，客户端刷新订阅才能获取新节点。

---

## 架构（数据流向）

~~~text
VPN Gate 官方源
      │  (GitHub Actions 定时抓取)
      ▼
筛选 SSTP 节点 → 去重
      │
      ▼
检测 Worker (CheckSocks5，部署在 Cloudflare)
      │  GET /check?sstp=vpn:vpn@host:port
      │  返回 success + 出口 IP(住宅/机房判定)
      ▼
保留成功节点 → 按国家分组 → 住宅/机房标注 → 延迟排序
      │
      ▼
生成 nodes.txt (GitHub Pages 发布)
      │
      ▼
edgetunnel 后台「自定义优选IP」框填 https://…/nodes.txt
      │  edgetunnel 每次生成订阅时自动 fetch → 解析 $sstp:// → 套链式代理
      ▼
客户端刷新 edgetunnel 订阅 → 使用最新 SSTP 节点
~~~

---

## 此 Fork 的持续运行配置

工作流已设置默认检测地址：

~~~text
https://cm-workers-checksocks5.asfhjbcueirew.workers.dev/check?sstp=vpn:vpn@
~~~

配置优先级为：手动运行的 `check_worker` 输入 → 仓库变量 `CHECK_WORKER` → 上述默认地址。手动输入可以留空；未设置仓库变量时，定时任务也会使用默认地址。输入域名或根 URL 时，工作流会自动补齐 HTTPS 和 `/check?sstp=vpn:vpn@`。非空但错误的覆盖值会报错，不会静默改用另一个服务。

工作流使用 `17,47 * * * *`，即每小时第 17、47 分钟尝试执行。修改工作流或 `vpngate.py` 并推送到 `main` 也会触发检测。GitHub 调度可能延迟；公开仓库长期无活动时，GitHub 可能停用定时工作流，应在 Actions 中检查状态。

### 地址配置成功不等于 Worker 已部署成功

工作流先请求不带代理参数的 `/check`。本项目正确的检测代码会返回 **HTTP 400 JSON**，其中 `success` 为 `false`，错误提示包含 `Missing proxy parameter` 和 SSTP 参数格式。这只是确认检测 API 存在，不代表任何 VPN 节点已经可用。

- **404**：地址没有提供预期检测接口；检查 Cloudflare 的生产部署、Worker 名称和 workers.dev 路由。不能把它归因于全部 VPN Gate 节点失效。
- **HTML 或登录跳转**：可能部署了错误入口或启用了访问控制。
- **接口通过、节点检测失败**：再检查实际 SSTP 服务端和握手错误。

检测服务仓库为 [Yeon-886/cm-Workers-CheckSocks5](https://github.com/Yeon-886/cm-Workers-CheckSocks5)，`wrangler.toml` 已对齐为 `name = "cm-workers-checksocks5"`、`main = "./_worker.js"`、`workers_dev = true`。GitHub 中修改配置不等于 Cloudflare 已完成部署；需在 Cloudflare 查看生产部署状态。

GitHub Pages 的 Source 应设为 **GitHub Actions**。工作流拒绝发布空的 SSTP 清单。检测与部署成功后，节点文件为：

~~~text
https://Yeon-886.github.io/gate/nodes.txt
~~~

原有 edgetunnel Worker 负责转发流量，不能替代独立的 CheckSocks5 检测服务。

---

## 一、完整部署教程（从零开始）

### 前置条件
- 一个 Cloudflare 账号（免费即可）
- 一个 GitHub 账号
- 一个已转入 Cloudflare 的域名（可选，但强烈推荐）

### 第 1 步：部署 edgetunnel（核心使用端）

1. 登录 Cloudflare 控制台，点击左侧 **Workers 和 Pages**
2. 点击 **创建** → 选择 **创建 Worker**，起名 `edgetunnel`，点击 **部署**
3. 打开 https://github.com/cmliu/edgetunnel/blob/main/_worker.js ，复制全部代码
4. 回到 Cloudflare Worker 编辑器，粘贴代码，点击 **保存并部署**
5. 在 **设置** → **变量** 中，添加变量 `ADMIN`，值填你的管理员密码
6. 在 **绑定** 中，添加 KV 命名空间绑定，变量名称填 `KV`
7. （推荐）在 **触发器** 中绑定自定义域名
8. 浏览器访问 `https://你的域名/admin`，登录后台
9. **在后台首页记下你的 UUID 和节点域名**（后面要用）

> **关键**：`UUID` 和 `节点域名` 是 edgetunnel 自己的配置，**不需要**在 `vpngate.py` 中设置。`vpngate.py` 只负责生成 `nodes.txt`，edgetunnel 会用它自己的 UUID/域名去生成最终订阅。

### 第 2 步：部署检测 Worker（CheckSocks5）

1. 打开 https://github.com/lsh8848/cm-Workers-CheckSocks5 ，点 **Fork**
2. 进 Cloudflare 控制台 → Workers 和 Pages → 创建 → 创建 Worker
3. 把 `_worker.js` 的全部内容粘贴进编辑器，点「部署」
4. 记下这个 Worker 的域名，形如 `https://xxx.你的用户名.workers.dev`
5. 验证：浏览器打开 `https://你的Worker域名/check`，确认返回上述缺少参数的 JSON。然后用真实节点请求 `/check?sstp=vpn:vpn@节点:端口`，仅 `success: true` 表示该次节点检测成功。

### 第 3 步：Fork 本仓库

在 GitHub 上打开本仓库，点 **Fork**，复制到你账号下。

### 第 4 步：修改配置（重点）

此 Fork 已配置默认检测地址。更换检测服务或再次 Fork 时，按下表覆盖配置：

| 文件 | 位置 | 改成什么 | 为什么 |
| :--- | :--- | :--- | :--- |
| 仓库 Settings → Secrets and variables → Actions → Variables | `CHECK_WORKER` | 你的检测 Worker URL，形如 `https://xxx.workers.dev/check?sstp=vpn:vpn@` | 覆盖默认地址，供手动及定时运行使用 |
| vpngate.py / 工作流环境变量 | `NODES_URL` | 你自己的 Pages 节点文件地址 | 用于输出使用提示 |

> **注意**：`vpngate.py` 中**不需要**配置 `EDT_UUID` 和 `EDT_DOMAIN`。你之前看到的这两个变量是旧版遗留，现已删除。edgetunnel 后台会自己处理 UUID 和域名。

### 第 5 步：开启 GitHub Pages 与 Actions

1. 进你 fork 的仓库 → Settings → Pages，Source 设为 **GitHub Actions**
2. 进 Actions 页，若提示启用 Actions 就点启用
3. 手动触发一次：Actions → VPN Gate Node Check → Run workflow → Run workflow
4. 检查检测、产物上传及部署步骤是否全部成功，不要只看另一个 `pages build and deployment` 工作流的状态。

### 第 6 步：确认产物

跑完后，你的站点地址是：
~~~text
https://你的GitHub用户名.github.io/仓库名/nodes.txt
~~~
浏览器打开，应看到 `优选域名:443#国家-住宅-XX$sstp://...` 格式的节点行。随后还需在客户端验证实际连接。

---

## 二、使用教程（URL 更新）

1. 进 edgetunnel 后台（你的域名/admin），找到「自定义优选IP」文本框
2. 粘贴**一行网址**：
   ~~~text
   https://你的GitHub用户名.github.io/仓库名/nodes.txt
   ~~~
3. 点保存
4. 客户端刷新订阅 → edgetunnel 生成订阅时读取 nodes.txt

> 原理：`nodes.txt` 是纯节点行版本（无注释头），每行 `入口域名:443#国家-住宅-01$sstp://vpn:vpn@节点:端口`。edgetunnel 生成订阅时读取这个网址，解析为优选入口及链式代理指令。GitHub 更新清单不会自动替换客户端已经缓存的节点，客户端仍需刷新订阅。

---

## 三、如何更换优选域名

入口地址用的是「优选域名」，决定客户端连 Cloudflare 用哪个 IP、稳不稳。

### 在哪个文件改
- 文件：vpngate.py
- 位置：`EDGE_HOSTS = [ ... ]`

### 改法
1. 用测速工具（如 bestcf）测一批 Cloudflare 优选域名，挑「延迟低 + 实际能连通」的
2. 打开 vpngate.py，把 `EDGE_HOSTS` 里的域名列表换成你测出来的（逗号分隔，格式 `域名:443`）
3. 提交推送，等下一次自动运行或手动触发 Action

---

## 四、配置速查表（vpngate.py）

| 常量 | 说明 |
| :--- | :--- |
| `EDGE_HOSTS` | 入口优选域名（换域名改这里） |
| `WORKER_CHECK_URL` | 检测 Worker（本地运行默认值，Action 里用 workflow 的 `CHECK_WORKER` 覆盖） |
| `NODES_URL` | 输出使用提示中的清单地址 |

> 再次强调：`vpngate.py` **不需要**配置 `EDT_UUID` 和 `EDT_DOMAIN`，这两个参数属于 edgetunnel 本身。

---

## 五、常见问题

### 只有几个节点能连
分别检查 Cloudflare 入口与 SSTP 出口，必要时替换 `EDGE_HOSTS` 为实测能通的地址。

### 全部 -1
检查：edgetunnel 是否部署好、域名是否解析到 Cloudflare、UUID 是否填对、传输协议是否对得上。

### 30 分钟没更新
到 Actions 页看最近一次运行是否成功、cron 是否还在、工作流是否被停用。

### 检测 Worker 报错
先看接口预检结果。404 时检查生产部署和 workers.dev 路由；不要反复更改一个已经通过格式检查的 `CHECK_WORKER` 值。

### 不知道 UUID 和节点域名在哪里看
登录 edgetunnel 后台（`https://你的域名/admin`），在后台首页就能看到。

---

*流水线：GitHub Actions → vpngate.py → 检测 Worker → GitHub Pages*

---

## 引用与致谢

本项目的实现离不开以下开源项目和服务的支持，在此表示衷心的感谢：

| 项目 | 用途 | 链接 |
| :--- | :--- | :--- |
| **cmliu/edgetunnel** | VLESS 代理 + 链式代理，节点最终通过它使用 | https://github.com/cmliu/edgetunnel |
| **lsh8848/cm-Workers-CheckSocks5** | 检测 Worker：验证 SSTP 节点可用性并读取出口 IP | https://github.com/lsh8848/cm-Workers-CheckSocks5 |
| **fdciabdul/Vpngate-Scraper-API** | VPN Gate 节点数据的 GitHub 镜像（官方源失效时回退） | https://github.com/fdciabdul/Vpngate-Scraper-API |
| **VPN Gate** | SSTP 节点数据源 | https://www.vpngate.net/ |
| **Star History** | 提供项目热度曲线图生成服务 | https://star-history.com/ |

---

## 项目热度

[![Star History Chart](https://api.star-history.com/svg?repos=hezhanleiok/gate&type=Date)](https://star-history.com/#hezhanleiok/gate&Date)

---

**特别感谢**：
感谢所有为开源社区做出贡献的开发者们！没有你们的无私奉献，就没有这个项目的诞生。也感谢每一位使用、测试和反馈问题的用户，是你们的支持让这个项目不断完善。

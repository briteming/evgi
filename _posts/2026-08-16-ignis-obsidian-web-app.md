---
layout: post
title: "Ignis：把 Obsidian 变成真正的自托管网页应用"
aliases: ["Ignis：把 Obsidian 变成真正的自托管网页应用"]
tagline: "不是远程桌面，而是在浏览器里原生运行的 Obsidian"
description: "Ignis 通过为 Electron API 提供浏览器兼容层，让 Obsidian 以真正 Web 应用的形式自托管运行。本文介绍 Ignis 的工作原理、Docker 部署步骤、远程访问与安全配置，以及实际使用中的限制与避坑经验。"
category: 产品体验
tags: [obsidian, ignis, self-hosted, docker, knowledge-management]
create_time: 2026-08-16 10:00:00
last_updated: 2026-08-16 10:00:00
---

用 [[Obsidian]] 记笔记这么多年，有一个需求一直没有被很好地解决，那就是在浏览器里直接访问自己的笔记库。Obsidian 官方一直没有推出 Web 版本，过去想要在别人的电脑上、或者在不方便安装客户端的环境里查看和编辑笔记，要么依赖远程桌面这种笨重的方案，要么就只能把笔记渲染成[静态网站](https://blog.einverne.info/post/2024/06/quartz-obsidian-publish.html)，牺牲掉编辑能力，或者通过 SSH 登录我的 macOS 通过命令行方式访问。最近我发现了一个叫 [Ignis](https://ignis.thiefling.com/) 的开源项目，它的口号非常直接：Run Obsidian as a self-hosted web app. Not remote desktop, an actual web app。它不是远程桌面，而是让 Obsidian 真正跑在浏览器里，体验下来确实让我眼前一亮，这篇文章就来聊聊它。

![Ignis 让 Obsidian 在浏览器中运行](https://pic.einverne.info/images/2026-08-16-10-05-00-ignis-obsidian-web-app-cover.png)

## 为什么浏览器访问 Obsidian 一直是个难题

Obsidian 是一个基于 Electron 的桌面应用，它的编辑器、插件系统、文件访问都建立在 Electron 提供的 Node.js 能力之上，而浏览器出于安全考虑并不提供这些 API，这是官方迟迟没有 Web 版的根本原因。在 Ignis 出现之前，想远程访问自己的笔记库大致有几类办法。

第一类是远程桌面方案，比如用 KasmVNC 或者类似 linuxserver 的 Obsidian 容器镜像，把整个桌面版 Obsidian 的画面通过 VNC 串流到浏览器。这类方案功能上最完整，但体验很差，字体渲染模糊、剪贴板不通、延迟明显，在手机上更是几乎不可用。

第二类是发布类方案，比如 Obsidian Publish、[[Quartz]]、Flowershow 这些工具，把笔记库渲染成静态网站。它们适合对外分享，但本质上是只读的，无法在浏览器里编辑笔记，也用不了任何插件。

第三类是换用天生就是 Web 应用的笔记工具，比如 SiYuan、AFFiNE 之类，但这意味着放弃 Obsidian 的整个插件生态和已经养成的工作流，迁移成本太高。

Ignis 走的是第四条路：它是一个兼容层（compatibility shim），为 Obsidian 所依赖的 Electron API 提供了浏览器端的实现，让原版 Obsidian 的代码直接在浏览器里运行，笔记库则保存在服务器上。值得一提的是，Ignis 本身不包含也不分发任何 Obsidian 的代码和资源，Docker 容器在首次启动时会从 Obsidian 官方源下载程序本体。项目采用 AGPL-3.0 协议开源，作者还专门写了一份 LEGAL.md，援引欧盟软件指令中关于互操作性的条款说明合法性，并明确表示无意损害 Obsidian 官方的商业利益，这种认真程度在同类项目里并不多见。

## Ignis 能做到什么

我最关心的当然是兼容性，毕竟一个残缺的 Obsidian 没有意义。实际情况比我预期的好很多，Obsidian 的核心功能基本都能用：编辑器、Canvas 白板、Bases 数据库视图、命令面板、右键菜单、主题和 CSS 片段都正常工作，绝大多数基于 Obsidian 插件 API 开发的社区插件也能直接加载。图谱视图、大纲这些功能在正确配置 HTTPS 之后也都可用，这一点后面讲部署时会展开。

在 Web 化之后，Ignis 还带来了一些桌面版没有的能力。它支持通过工具栏、右键菜单或者直接拖拽来上传文件到笔记库，也可以把单个文件或整个文件夹打包成 ZIP 下载下来。多仓库支持做得很完整，可以创建、切换、重命名、删除 vault，不同的浏览器标签页甚至可以打开不同的 vault。多个标签页之间通过 WebSocket 实时同步，在一个标签页里的编辑会在一秒内出现在另一个标签页中。另外还有两个很实用的 URL 参数：`?workspace=` 可以在独立标签页中打开某个保存好的工作区布局，`?file=` 可以通过 URL 直接打开某篇笔记，这让 Obsidian 的笔记第一次拥有了可以分享给自己其他设备的链接。小屏幕设备上 Ignis 会切换到移动端 UI，手机浏览器里的体验接近 Obsidian 移动客户端。

同步方面，官方的 Obsidian Sync 可以在登录的标签页里正常工作，Ignis 还提供了服务端的 Headless Sync，即使浏览器标签页全部关闭，服务器也能继续在后台同步，这个设计解决了 Web 应用"关掉页面就停止工作"的天然缺陷。对于我这种用第三方方案同步的用户，obsidian-livesync 这类走 WebSocket 或 HTTP 的插件也能配置成功，只是要注意一些网络上的细节，后面避坑部分会提到。我在 [[2020-11-23-obsidian-sync-acrose-devices-solution|我的 Obsidian 笔记跨设备同步方案]] 里梳理过各种同步方式，Ignis 相当于给这些方案又加了一个随时可用的 Web 入口。

性能上作者也下了功夫。Ignis 用一次预压缩的 bootstrap 请求就把 vault 信息、元数据树、插件列表全部交付给浏览器，配合索引器预取（indexer pre-fetch）预热内容缓存，让 Obsidian 启动时的索引过程命中缓存而不是反复走网络。服务端用 LRU 缓存控制内存占用，默认 50MB，不会把整个笔记库都加载进内存，这些参数都可以在设置面板里调整。我的笔记库有几千个文件，加载速度完全在可接受范围内。

## 用 Docker 部署 Ignis

Ignis 的部署非常简单，官方提供了 Docker 镜像，一个 docker-compose 文件就能跑起来：

```yaml
services:
  ignis:
    image: nobbe/ignis:latest
    ports:
      - "8080:8080"
    environment:
      # 运行 id 命令查看自己的 uid/gid 并填入
      - PUID=1000
      - PGID=1000
    volumes:
      - ./vaults:/vaults
      - ./data:/app/data
      - obsidian-app:/app/obsidian-app
    restart: unless-stopped

volumes:
  obsidian-app:
```

保存为 `docker-compose.yml` 之后执行 `docker compose up -d`，首次启动时容器会从官方源下载 Obsidian 和 obsidian-headless CLI，大概需要一两分钟，可以用 `docker compose logs -f` 观察进度。之后访问 `http://localhost:8080`，如果 `vaults` 目录下已经有笔记库会自动加载，否则会打开 vault 管理器引导创建第一个。

有几个部署细节值得注意。PUID 和 PGID 要和宿主机用户匹配，用 `id` 命令查一下自己的 uid 和 gid 填进去，否则 Ignis 写入的文件归属会出问题。如果笔记库放在 NAS 挂载或者 NFS 上，可以直接把外部目录挂载到 `/vaults` 下面的子目录；对于 rclone mount、FUSE、NFS、SMB 这类较慢的文件系统，还可以设置 `WRITE_COALESCE_MS` 环境变量开启写入合并去抖，减少频繁的小写入。

我自己的做法是把 Ignis 指向已有的同步目录，这样桌面版 Obsidian、手机客户端和 Ignis 操作的是同一份数据，Ignis 只是多出来的一个访问入口，不需要改变原有的同步链路。

## 远程访问与安全：最重要的避坑点

这一部分是使用 Ignis 之前必须搞清楚的，官方文档也用了醒目的警告来强调。核心有两点：Ignis 没有内置任何身份验证，以及浏览器的安全上下文（secure context）要求。

先说安全上下文。Obsidian 依赖的一些浏览器 API，比如加密和剪贴板相关的接口，只在 HTTPS 或者 localhost 环境下可用。所以如果你通过 `http://192.168.1.10:8080` 这样的局域网地址裸访问 Ignis，会发现图谱视图、大纲、Sync 等一系列功能默默失效，这不是 bug，而是浏览器的安全策略。解决办法有两类：正经的做法是在前面加一层 TLS，用 Caddy、nginx 或 Traefik 做反向代理（官方 examples 目录里有现成配置），或者用 `tailscale serve`、Cloudflare Tunnel 这类免证书管理的方案；偷懒的做法是在每个客户端浏览器里把 Ignis 的地址加入安全源白名单，Chromium 系浏览器在 `chrome://flags/#unsafely-treat-insecure-origin-as-secure` 设置，但这种方式只适合局域网，Safari 没有对应选项只能上 TLS。

再说身份验证。Ignis 默认监听纯 HTTP 且没有登录机制，任何能访问到这个端口的人都可以读写你的整个笔记库。所以绝对不要把 Ignis 直接暴露到公网。如果需要在外网访问，务必在前面加一层认证：反向代理的 Basic Auth 是最简单的，Authelia、Authentik、OAuth2 Proxy 这类 SSO 方案更完善，也可以用 Cloudflare Access 配合 Tunnel，或者干脆走 Tailscale、WireGuard 这样的 VPN 只在私有网络里访问。官方 examples 里提供了两套完整的 Caddy 配置，分别对应 Basic Auth 和 Authelia，可以直接拿来用。路线图里提到未来会支持内置认证和多用户 OIDC，但在那之前，认证完全是自己的责任。

我个人的建议是家庭网络内用 `tailscale serve` 一条命令解决 HTTPS 和访问控制两个问题，既不用管证书，也天然只有自己的设备能访问，是最省心的组合。

还有一个容易踩的坑是第三方同步插件的连通性。出于防止恶意网络扫描的考虑，Ignis 服务端默认拒绝中继指向私有地址、回环地址的 HTTP 请求，所以如果你的 CouchDB 或者其他同步服务器跑在局域网或同一台 Docker 主机上，需要通过 `PROXY_ALLOW_PRIVATE_HOSTS` 环境变量显式放行对应的 IP 或 CIDR（注意只接受 IP 不接受主机名），或者在设置里配置 direct-fetch 让浏览器直连（这要求同步服务器开启 CORS）。走 WebSocket 的同步插件则是浏览器直连，当 Ignis 本身是 HTTPS 时，浏览器会拒绝明文的 `ws://` 连接，同步服务器也需要提供 `wss://`，用受信任的证书或者同样套一层 `tailscale serve` 就能解决。

## 限制与不完美的地方

把 Electron 应用塞进浏览器不可能没有代价，有些限制需要提前知晓。最主要的是需要 Node 原生模块或 `child_process` 的插件无法加载，比如依赖本地执行命令的插件（典型如调用本地 Git 二进制、执行 shell 脚本的那一类）在 Ignis 里是跑不起来的，官方文档维护了一个插件兼容性页面，重度依赖某个插件的话建议先去查一下。

另一个需要留意的是密钥存储。桌面版 Obsidian 的插件可以用 Electron 的 safeStorage 借助操作系统加密敏感数据，浏览器没有等价能力，所以 Ignis 里插件存储的 API key 之类的秘密目前是明文保存的，服务端加密在计划中但尚未实现。在共享或安全性存疑的服务器上部署时，这一点要纳入考虑。

还有一些小的差异：浏览器无法弹出真正的本地文件选择器，所以像 Importer 这类插件导入文件要分两步操作，先选择文件暂存再重新执行动作；拼写检查语言跟随浏览器设置而不是应用内设置；依赖 Electron 菜单 API 的原生菜单选项被禁用。这些都属于可以接受的妥协。

最后要提醒的是，Ignis 还是一个很年轻的项目，虽然作者自己已经把它当作日常笔记工具在用，GitHub 上也已经收获了超过 1200 个 star，但处于活跃开发阶段意味着可能遇到未记录的问题。好在它只是数据的一个访问层，笔记本体始终是磁盘上的 Markdown 文件，就算 Ignis 出问题，数据本身也不会受影响，这也是我敢直接把它指向主力笔记库的原因。当然，任何时候都不要忘了备份。

## 最后

Ignis 解决的是一个存在了很多年的真实痛点：Obsidian 的本地优先哲学和随时随地访问之间的矛盾。此前的答案要么是体验糟糕的远程桌面，要么是丧失编辑能力的静态发布，而 Ignis 用兼容层的思路给出了第三种答案，让你在浏览器里得到一个接近原生的、插件可用的、可编辑的 Obsidian，而数据依然完整地躺在自己服务器的文件系统里。

对我来说，它最大的价值是让 Obsidian 的访问入口从"装了客户端的设备"扩展到了"任何一个有浏览器的地方"，配合 Tailscale 之后，在公司的电脑、朋友的电脑、甚至 iPad 的浏览器里打开自己的笔记库都只是一个 URL 的事情。如果你也是 Obsidian 的自托管爱好者，手边有一台跑着 Docker 的小主机或 NAS，非常值得花十几分钟把 Ignis 跑起来体验一下。项目的 [GitHub 仓库](https://github.com/Nystik-gh/ignis) 和[官方文档](https://ignis.thiefling.com/docs/)都写得相当清楚，部署前把安全章节读一遍，就可以放心使用了。

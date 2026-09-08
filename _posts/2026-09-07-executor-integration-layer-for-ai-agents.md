---
layout: post
title: "Executor：多 Agents 和外部世界之间的统一代理层"
aliases: 
- "Executor：多 Agents 和外部世界之间的统一代理层"
tagline: "把散落在各个客户端里的 MCP 配置和 API Key 收拢到一个目录里"
description: "Executor 是一个开源的 AI Agent 集成层，把 MCP server、OpenAPI 规范、GraphQL 接口统一成一个工具目录，配置一次、认证一次、设定一次策略，然后所有 MCP 客户端共享。这篇文章讲清楚它的核心概念、MCP 代理机制、四种部署方式，以及我认为它真正解决的问题和目前的局限。"
category: "产品体验"
tags: [executor, mcp, ai-agent, claude-code, openapi, graphql, self-hosted, developer-tools, api-gateway, integration]
create_time: 2026-09-07 10:00:00
last_updated: 2026-09-07 10:00:00
---

同时使用 Claude Code，Codex，Pi，OpenClaw 等等 Agent 工具时有一个非常尴尬的状况，就是不同的 Agent 工具维护了不同的配置格式，如果要配置 MCP server 就需要分别到不同的配置文件中定义，相同的配置散落在系统的各个角落中。同一份 API Key 也需要重复粘贴到不同的配置中，更麻烦的是如果一旦有更新或者 API Key 变动就需要重复修改多个地方，我之前还调查过如何[[跨平台管理 MCP]]，主要的思路还是通过脚本和同步来对齐配置，但本质上还是在管理多个副本，直到我看到了 Executor 这个项目。

[Executor](https://github.com/UsefulSoftwareCo/executor) 项目的目的是为了简化各个 Agent 配置，在 Agent 和外部 API 之间做一层代理。集成只加一次，凭据只给一次，权限策略只设一次，然后所有 MCP 兼容的客户端都指向这一层，共享同一个工具目录。项目地址的作者是 Rhys Sullivan，之前在 Vercel、Microsoft、Epic Games 待过。

![AI Agent 与外部 API 之间的统一集成层](https://pic.einverne.info/images/2026-09-07-10-00-00-executor-integration-layer.png)

## 它到底解决的是什么问题

先说清楚 Executor 不是什么。它不是又一个 MCP server，也不是某个特定服务的封装。它的定位是集成层，或者说 MCP 网关，README 里的原话是 "The missing integration layer for AI agents"。

今天 agent 生态的现状是：每个客户端都是一座孤岛。你在 Claude Code 里接了 Linear 的 MCP server，Cursor 想用就得再接一遍；你给某个 agent 配了公司内部 API 的 token，另一个 agent 想调同一个接口，你得再找一遍那个 token 存在哪。这套模式在只有一个 agent 的时候完全没问题，但当你手上同时跑着桌面客户端、终端 CLI、云端 agent 的时候，配置的重复度和凭据的扩散范围就开始失控了。

更深一层的问题是权限。MCP 协议本身没有定义工具级别的授权模型，一个 MCP server 暴露出来的所有工具，对客户端来说要么全都能调，要么整个 server 不接。你没有办法说"这个 server 的读操作随便调，写操作必须先问我"。实际使用中这个粒度是不够的，尤其当 agent 有能力调用会产生真实副作用的接口时。

Executor 把这三件事——集成定义、凭据存储、权限策略——从客户端里抽出来，放到一个独立的服务里，然后通过 MCP 协议统一暴露出去。

## 三个核心概念

Executor 的设计很简单，只有三个概念：

Integration 是集成本身，也就是你想接入的东西的定义。Executor 支持 MCP server、OpenAPI 规范、GraphQL 接口，以及 Google Discovery。README 里有一句话我觉得概括得很好：只要能用 JSON Schema 描述，它就能成为一个 integration。这也意味着接入方式非常直接，你有一份 OpenAPI 的 YAML 或者 JSON，扔进去就完事了，不需要为它单独写一个 MCP server 的包装层。这一点其实很关键，因为现实中大量内部服务都有 OpenAPI 文档，但几乎没有人会为它专门写 MCP server。

Connection 是集成的一个已配置实例。这里的设计有点意思：integration 和 connection 是一对多的。同一个 GitHub 的 OpenAPI 定义，你可以建三个 connection，分别用不同账号的 token；同一个内部 API，你可以建 staging 和 production 两个 connection，指向不同的 baseUrl。凭据是挂在 connection 上的，而不是挂在 integration 上。

Policy 是每个工具的权限级别，一共三档：allow 直接放行，require approval 调用时暂停等人工批准，block 完全禁止。关键在于默认值不是手工一个个点出来的，而是从规范里推导的。文档给的例子是 OpenAPI：GET 这类只读操作默认允许，写操作可以设成需要批准。这个默认值的推导逻辑很实用，因为一份稍具规模的 OpenAPI 文档动辄上百个 endpoint，指望人工逐个设策略是不现实的，能按 HTTP 方法自动分出安全和危险两档，剩下的手工微调量就小得多了。

## 一个端点，所有 agent

Executor 对外的形态就是一个 MCP endpoint。agent 说 MCP，Executor 在后面把请求路由到具体的集成上：对上游 MCP server 说 MCP，对 OpenAPI 和 GraphQL 说 HTTP，然后把结果原路返回。

这个代理结构带来的第一个好处是配置的解耦。因为客户端只认识 Executor 这一个端点，你在 Executor 里增删改上游服务，客户端完全不需要动。加了一个新集成，agent 那边自动就能看到新工具，不需要重启、不需要改配置文件、不需要在 5 个客户端里重复这个动作。这一点对我来说是最直接的收益，我加一个内部 API，Claude Code 和 Cursor 同时就有了。

第二个好处更重要，是凭据的隔离。文档里的说法是 credentials stay out，凭据存在 Executor 持有的 connection 上，在实际发起上游调用的那一刻才附加到请求里。agent 从头到尾看不到 token，也就不存在 token 意外进入模型上下文、被写进日志、或者被 prompt injection 套出来的风险。如果你的 agent 跑在沙箱里，或者是一个你不完全信任的第三方客户端，这个隔离是有实际意义的。

第三是策略在每次调用时统一执行。不管请求从哪个 agent 来，走的都是同一套 policy 判断。你不需要在每个客户端里分别配一遍权限，也不会出现某个客户端漏配导致权限失控的情况。新接入的上游服务自动继承同一套策略机制。

需要提醒一点，大部分 MCP 客户端只在启动时加载 server 列表，所以第一次把 Executor 接进去之后，通常需要重启客户端或者开一个新会话，工具才会出现。这个坑文档里明确提到了，我也确实踩了一次，以为是配置写错了。

## 部署方式的选择

Executor 提供了 4 种运行形态，功能完全一致，区别只在打包方式。

本地 CLI 是最轻的方式，`npm install -g executor` 装上，然后 `executor install` 把它注册成常驻后台服务，`executor web` 打开网页控制台。这个后台服务会跨重启保持运行，如果你只想临时跑一下不留痕迹，用 `executor web --foreground` 起一个前台进程就行。默认监听 `127.0.0.1:4788`，端口被占用时会自动挑一个空闲端口。需要 Node.js 20 以上。

桌面应用是同一个运行时套了个原生壳，Mac、Windows、Linux 都有，适合日常桌面环境；CLI 更适合无头服务器。

Executor Cloud 是官方托管版本，有免费额度，什么都不用装，直接登录、加集成、把 agent 指向托管端点。如果你用的是云端 agent，这条路是唯一能走通的，因为云端 agent 连不到你本机的 127.0.0.1。

自托管有两条路径。[[Docker]] 版本是 `ghcr.io/usefulsoftwareco/executor-selfhost:latest`，单个容器里打包了 API、MCP、认证、代码执行和 Web UI，数据落在一个 SQLite 文件里，暴露 4788 端口，零配置起步。[[Cloudflare]] 版本是部署到你自己账号下的一个 Worker，用 Cloudflare Access 做认证，用 D1 做存储。

选择的判断标准其实很清晰：如果所有 agent 都在本机，用本地版本，桌面环境选 App，服务器选 CLI；如果需要多台机器或者云端 agent 访问同一份目录，就上托管或自托管。我自己的用法是 Docker 自托管，跑在家里的 NAS 上，这样笔记本、台式机和几个服务器脚本共享同一份集成目录，同时数据完全在自己手上。

## 上手的实际流程

接入客户端用的是 `add-mcp` 这个工具，它会自动检测你当前的 MCP 客户端并写入配置：

```bash
npx add-mcp http://127.0.0.1:4788/mcp --transport http --name executor
```

如果要走 stdio 传输：

```bash
npx add-mcp "executor mcp" --name executor
```

添加集成可以在 Web UI 里点 Add Integration，也可以走 CLI。这里有个细节值得注意：当 OpenAPI 文档里的 `servers` 字段用的是相对路径时，需要显式传 `baseUrl`，否则请求不知道该发到哪里。很多内部服务生成的 OpenAPI 文档都有这个问题，第一次接的时候容易卡住。

日常会用到的 CLI 命令大概是这几个：

```bash
executor tools search <query>        # 在目录里搜工具
executor call <path...>              # 直接调用某个工具
executor tools integrations          # 列出所有集成
executor tools describe <tool>       # 查看工具的详细定义
executor resume --execution-id <id>  # 恢复一次被暂停的执行
executor daemon status               # 查看后台服务状态
```

`executor resume` 对应的就是 policy 里 require approval 那一档，调用被挂起之后用它继续。这些命令都会自动拉起本地 daemon，不需要手动先启动。

如果要在自己的代码里用，官方提供了 TypeScript SDK，同时给了 Promise 和 Effect 两套 API：

```ts
import { createExecutor } from "@executor-js/sdk/promise";
import { openApiPlugin } from "@executor-js/plugin-openapi/promise";
```

## 使用中的一些判断

用下来我觉得它最适合的场景，是你手上有多个 agent 客户端，并且有一批自己的、非公开的 API 需要接进去。如果你只用一个 Claude Code，接的也都是现成的公开 MCP server，那 Executor 引入的这一层收益不大，反而多了一个需要维护的服务。它的价值随着 agent 数量和自有 API 数量的增长而放大。

OpenAPI 直接导入这条路是我认为最被低估的能力。写一个 MCP server 需要理解协议、搭脚手架、处理传输层，而扔一份 OpenAPI 文档进去是零成本的。公司内部服务基本都有 Swagger 文档，这意味着让 agent 接触内部系统的门槛一下子降到了几乎为零。当然这也是双刃剑，接得越容易，就越需要认真对待 policy 那一层。

关于 policy ，默认按 HTTP 方法分档。但是 GET 也可能是危险的，比如一个导出全量用户数据的 GET 接口；POST 也可能完全无害，比如一个搜索接口。所以自动推导出来的默认值应该被当成待审阅的草稿，而不是最终配置。特别是刚导入一份大的 OpenAPI 文档之后，值得花点时间把明显敏感的接口手动降级。目前文档里对批准流程的细节说得不多，谁能批准、请求怎么呈现、有没有超时、批准是否会被记住，这些都还不清楚，需要自己试。

另一个需要清醒认识的是，加这一层意味着多了一个单点。Executor 挂了，所有 agent 的所有工具一起挂。本地跑还好，如果是团队共享的自托管实例，这个可用性问题得认真考虑。相应地，它也成了一个高价值目标——一个集中存放了你所有 API 凭据的服务，本身的安全边界需要被认真对待，尤其是自托管暴露到公网的情况。Cloudflare 版本用 Cloudflare Access 做认证这个选择，某种程度上也是在回应这个问题。

从生态位上看，Executor 经常被拿来和 Composio 这类服务比较，核心差异在于开源和可自托管。Composio 是托管的商业服务，你的凭据存在它那里；Executor 你可以完全跑在自己的机器或者自己的 Cloudflare 账号里。对于凭据敏感的场景，这个差异是决定性的。

## 最后

我觉得 Executor 真正抓住的，是 agent 生态里一个还没被认真对待的结构性问题：随着一个人同时使用的 agent 从 1 个变成 5 个，集成配置的复杂度不是线性增长，而是集成数乘以客户端数的乘积增长。用同步配置文件的方式去对抗这个乘法，只能缓解，不能解决。把集成提取成一个独立的、被所有客户端共享的层，才是从根上消掉那个乘数。

它现在还很年轻，2026 年才起步，文档里不少地方——尤其是批准流程和沙箱执行——还没写透，policy 模型也还比较粗。但方向我认为是对的。MCP 协议解决了 agent 和工具之间的通信标准，但没有解决工具的管理、认证和授权，那部分空白总归需要有东西来填。Executor 是目前我看到的填法里比较完整的一个：概念足够少，部署方式足够多，而且完全开源可自托管。

## related

- [[FumaDB]]
- [[Pi]]
- [[Emdash]]
- [[Composio]]

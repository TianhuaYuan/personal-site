---
title: "MCP Transport：Streamable HTTP → ASGI 子应用挂载 + JSON-RPC 2.0"
tags:
  - 技术学习
  - ai
  - agent
created: "2026-07-21"
---

> **一句话**：Streamable HTTP是MCP的默认传输——单端点/mcp、POST为主、按需开SSE流，可无状态水平扩容，用ASGI mount挂进现有服务一个进程搞定。

# MCP Transport：Streamable HTTP → ASGI 子应用挂载 + JSON-RPC 2.0



## 二、它是什么 / 为什么需要它

### 1.1 传输层 = 信封和管道，不关心信里写啥

MCP 可以拆成三层看（别被术语吓到，就是一个"信封—管道—内容"的关系）：

| 层 | 是什么 | 生活类比 |
|-|-|-|
| **协议层（Protocol）** | 定义消息长啥样、有哪些方法（tools/call、resources/read…） | 信封上写"收件人/寄件人/正文格式"的规矩 |
| **传输层（Transport）** | 定义消息**怎么在网线上跑** | 用平信、快递还是当面递 |
| **能力层（Capabilities）** | 服务端具体提供哪些 Tools/Resources/Prompts | 仓库里实际有哪些货 |

> **关键认知**：传输层只负责"把一段 JSON-RPC 文本可靠送达"，**不关心里面是调用工具还是读资源**。所以换传输层（stdio → Streamable HTTP）时，上层协议和能力**一行都不用改**——这正是"分层"的价值。（来源：MCP 官方规范 Transports 章节）

### 1.2 stdio 的天花板：本地 demo 无敌，一上远程就露怯

stdio 在本地调试无敌，但生产场景一上"远程"就三个硬伤：

1. **必须同机同进程**：老板和管理员得在一台机器、用子进程拉起。想让云端大模型调你本地的 Server？不行。
2. **不跨网络**：没有端口、没有 HTTP，防火墙/CDN/网关全用不上。
3. **一对一**：一个 Server 只能被一个 Client 当儿子养，没法多租户。

而真实生产是：**云端 Agent → 远程 MCP Server（部署在服务器、多客户端连、前面有 nginx/网关）**。这就必须上 HTTP 系传输。

## 三、三种传输横向对比

| 维度 | stdio | HTTP+SSE（旧，2024-11-05） | **Streamable HTTP（新，2025-03-26）** |
|-|-|-|-|
| 端点数 | 无（走 stdin/stdout） | 2 个：`/sse` 长连接 + `/message` POST | **1 个：`/mcp`（POST 为主）** |
| 长连接 | 不需要 | **必须常年保持 /sse** | 不需要，按需开 SSE 流 |
| 断流后果 | 进程级，整段断开 | **丢消息、要重连重发** | 每段请求独立，断哪段重哪段 |
| 负载均衡 | 不适用 | 脆弱（长连接绑死实例） | **友好（可无状态）** |
| 是否远程 | 否（本地） | 是 | 是 |
| 会话 | 无会话概念 | 有会话但要维持 | **Mcp-Session-Id header 可选会话（稳定版）** |
| 现状 | 仍在用 | **已废弃（2025-11-25 官方标记）** | **官方推荐默认** |

> ⚠️ **版本红线**：网上不少老教程说"HTTP+SSE 在 2025-05 废弃"，**准确说法是 2025-11-25 规范正式把 HTTP+SSE 标记为废弃**（来源：MCP 规范版本史 / yuuine 生态盘点 2026）。写代码请以你依赖的 SDK 实际支持版本为准。

```mermaid
graph LR
    A["stdio 本地父子进程 stdin/stdout"] --> B["HTTP+SSE 2024-11-05 双端点"]
    B -.->|"已废弃 2025-11-25"| C["Streamable HTTP 2025-03-26 单端点 /mcp"]
    C --> D["draft 2026-07-28 无状态核心 去会话去握手"]

```

## 四、Streamable HTTP 核心机制拆解

### 3.1 单端点 + POST 为主

客户端**每个 JSON-RPC 消息都是一次新的 HTTP POST** 到 `/mcp`。服务器看情况二选一返回：

- 能立刻答：返回 `Content-Type: application/json` 的**单个 JSON 对象**；
- 要慢慢推（进度、多消息）：返回 `Content-Type: text/event-stream` 的 **SSE 流**，报完关掉。

就像去窗口办事：能马上办完就当场给回执（单 JSON）；要排队慢慢办的，就给你开个"实时进度广播"（SSE 流），办完广播就关。

### 3.2 会话用 `Mcp-Session-Id` header（稳定版 2025-03-26 \~ 2025-11-25）

`initialize`（初始化握手）时，服务器**可能**在响应头回一个 `Mcp-Session-Id`；客户端后续所有请求**必须原样带上**。服务器用它挂"每会话状态"（限流计数、作用域凭证、审计日志）。这个 ID 必须是全局唯一且密码学安全的（UUID 或 JWT），且只含可见 ASCII（0x21–0x7E）。

> Auth0 / 官方安全建议把它编码成 **JWT（JSON Web Token，一种自包含的凭证令牌）**，这样服务器不用查库就能验真——呼应第 ⑨ 篇的 JWT 话题。

### 3.3 🔴 2026-07-28 无状态化巨变（当前是 RC，最终版 2026-07-28 发布）

这是 🔴 高优必须展开的最新演进。**2026-07-28 发布候选（Release Candidate）是 MCP 自发布以来最大的一次修订**，核心是"协议层无状态化"。它**移除了**：

- `initialize` / `notifications/initialized` 握手（SEP-2575）—— 协议版本、clientInfo、capabilities 改由每个请求的 `_meta` 携带；新增 `server/discover` 让客户端主动拉取服务端能力。
- 协议级会话与 `Mcp-Session-Id` 头（SEP-2567）—— 任意请求可落到任意实例，之前 sticky session、共享会话存储全不需要了。
- `GET` 流端点、`resources/subscribe`/`unsubscribe` —— 改成 `subscriptions/listen` 单一长流。
- **Resumable SSE（可恢复 SSE 流）被移除** —— 注意！稳定版里服务器可用 `Last-Event-ID` 做断线重放，但 2026-07-28 草案明确"Resumable SSE streams via Last-Event-ID are not supported"。

它**新增了**：

- **MRTR（Multi Round-Trip Requests，多轮往返请求，SEP-2322）**：服务器不再在 SSE 上反向发请求（sampling/elicitation/roots），而是返回一个 `InputRequiredResult`（含 `inputRequests` + `requestState`），客户端拿答案后**带着 `inputResponses` 重发原请求**。
- **可路由头 `Mcp-Method` / `Mcp-Name`**：负载均衡、网关、限流器不用读 body 就能按"方法"路由（来源：MCP 2026-07-28 RC 官方博客）。
- **缓存 `ttlMs` / `cacheScope`**：`tools/list`、`resources/read` 等返回带新鲜度提示，客户端可缓存（来源：hivebook / Mamezou 解读）。
- **JSON Schema 默认 2020-12**（SEP-1613）。

> 🔴 **实战含义**：这是 RC（候选版），最终规范 2026-07-28 才发布。写代码**务必以你用的 SDK 版本为准**：用 FastMCP 新版时，别再依赖 GET 长流、别假设 `Mcp-Session-Id` 一定存在；生产系统在上 RC 前建议**锁在 2025-11-25 稳定版**。

### 3.4 无状态模式（水平扩容关键）

若你的工具是**纯函数**（无会话状态，状态都在数据库），可开 `stateless_http=True`：不要求 `Mcp-Session-Id`，**任意 worker 都能处理任意请求**，直接多进程 `--workers 4` 扩容。代价是客户端不能"续上"上一个会话。（来源：FastMCP 部署文档 / MCP Python SDK）

## 五、JSON-RPC 2.0 在 HTTP 上怎么跑

MCP 所有消息都是 **JSON-RPC 2.0**（一种无状态的远程调用格式：一个 `jsonrpc:"2.0"` 字段 + `method` + 可选 `id` + `params`）。在 Streamable HTTP 上：

- **请求**：`POST /mcp`，`Accept: application/json, text/event-stream`（必须同时声明两种，表明"你回单个 JSON 或开流我都接"）。
- **响应**：单 JSON 对象，或 SSE 流（流里先发 `notifications/progress` 这类进度通知，最后发最终响应，随后关流）。
- **通知（notification，无 id 的消息）**：服务器收下回 `202 Accepted` 空 body；拒收回 4xx。
- **取消**：客户端关掉 SSE 响应流 = 取消该请求（每段请求独立，断哪段取哪段，不会误伤别的）。

```mermaid
graph TD
    C["Client"] -->|"POST initialize Accept json,event-stream"| S["MCP Server /mcp"]
    S -->|"200 JSON + Mcp-Session-Id"| C
    C -->|"POST tools/call 带 Session-Id"| S
    S -->|"立刻答 200 application/json"| C
    S -->|"流式 200 text/event-stream"| P["进度 notifications/progress"]
    P -->|"最终 response 后关流"| C
    C -->|"POST notification 无id"| S
    S -->|"202 Accepted 空body"| C

```

⚠️ **准确性硬约束（必看）**：当前**稳定版（2025-11-25）不支持 JSON-RPC 批处理（batching）**。批处理曾在 2025-03-26 短暂引入，但**2025-06-18 即被移除**（官方理由："没有 compelling use case"）。所以你在 Streamable HTTP 上**每个请求都是单个 JSON-RPC 消息**，不要写"body 可放 JSON 数组批量发"的过时代码。（来源：MCP 官方 release notes / Speakeasy 版本对照 / 掘金 2025-06-18 详解）

```json
POST /mcp  HTTP/1.1
Content-Type: application/json
Accept: application/json, text/event-stream
Mcp-Session-Id: 1a2b3c4d

{ "jsonrpc": "2.0", "id": 2, "method": "tools/list" }
```

## 六、ASGI 子应用挂载

### 5.1 为什么"挂载"而不是"另起服务"

很多教程把 MCP 当**独立服务**跑：单独端口、单独 auth、单独 CORS、单独监控。可你**本来就有个 FastAPI 主服务**啊——再加一个 MCP 服务，等于多一套"攻击面（surface area）"：多一个端口、多一个健康检查、多一套鉴权路径。生产环境本来就够吵了。

**正解：把 MCP 当子应用（sub-application）挂进现有 FastAPI/Starlette 里**——一个进程、一个端口、一套 auth/监控。MCP 不是兄弟服务，是**挂在你 App 下的一个挂载点（mount）**。

### 5.2 代码骨架（极简，只留感觉）

```python
from fastapi import FastAPI
from fastmcp import FastMCP

mcp = FastMCP("API Tools")

@mcp.tool
def get_status() -> str:
    return "ok"

# 1) 生成 MCP 的 ASGI app，路径用根 "/"
mcp_app = mcp.http_app(path="/", transport="streamable-http")

# 2) 父应用复用 MCP 的 lifespan（生命周期），无需额外包装
app = FastAPI(lifespan=mcp_app.lifespan)

@app.get("/health")
def health():
    return {"status": "ok"}

# 3) 把 MCP 挂到 /mcp 下
app.mount("/mcp", mcp_app)

# 跑父应用即可：uvicorn main:app --host 0.0.0.0 --port 8000
```

挂载后：你的 API 仍答 `/health`，MCP 答 `/mcp`。**auth/CORS/日志全走外层 FastAPI 中间件**，MCP 自动复用——不用给 MCP 单独再写一套。

> **ASGI（Asynchronous Server Gateway Interface，异步服务器网关接口）** 是 Python Web 的异步标准接口；FastAPI/Starlette 都是 ASGI 应用，`app.mount()` 就是把一个 ASGI 应用挂到另一个的路径下。`mcp.http_app()` 吐出的正是个标准 ASGI 应用，所以能被挂。（来源：FastMCP 部署文档 / ekky.dev《Mount, Stream, Authenticate FastMCP with FastAPI》2026-05）

### 5.3 可挂多个 MCP Server

```mermaid
graph TD
    U["外部请求"] --> F["FastAPI 父应用 共享 auth/CORS/日志/lifespan"]
    F --> H["/health 等自有路由"]
    F --> M["/mcp 挂载点 ASGI 子应用"]
    M --> A["FastMCP http_app transport=streamable-http"]
    A --> T1["Tool get_status"]
    A --> T2["Tool search_chroma"]
    A --> T3["..."]

```

`FastAPI`/`Starlette` 下能 `mount` 多个 MCP 子应用（如 `/mcp-echo`、`/mcp-math`），各自独立工具集，共享外层 auth——多租户/多能力模块化就靠它。（来源：orchome《Python 使用 FastAPI 挂载多个 MCP 服务》/ MCP Python SDK 多 Server 示例）

## 七、演进时间线（🔴 高优）

| 时间 | 版本 | 关键变化 |
|-|-|-|
| 2024-11-05 | 初版 | 引入 `stdio` + `HTTP+SSE` 双端点 |
| 2025-03-26 | 大改 | **Streamable HTTP** 登场，单端点 `/mcp`；引入 tool annotations；短暂加 JSON-RPC 批处理 |
| 2025-06-18 | 精修 | 结构化输出 `outputSchema`/`structuredContent`/`title`；**移除批处理**；Elicitation；强制 `MCP-Protocol-Version` 头 |
| 2025-11-25 | 当前稳定版 | HTTP+SSE **正式废弃**；OAuth/OIDC 增强；治理结构化；JSON Schema 2020-12 默认 |
| 2026-07-28 | RC（最终版同日发布） | **无状态化**：去握手、去会话；MRTR；`Mcp-Method`/`Mcp-Name` 头；`ttlMs`/`cacheScope` 缓存；**移除 Resumable SSE**；Roots/Sampling/Logging 标记废弃 |

> 实战：Atlassian 已于 2026-06-30 关闭其远程 MCP 的 SSE 端点——旧传输退场是实打实的（来源：agenticwire《FastMCP Streamable HTTP》）。

## 八、生产落地细节（🔴 高优）

这些是高优才展开、上线都用得上的"坑位清单"：

1. **Origin 校验防 DNS rebinding（DNS 重绑定攻击）**：服务器**必须校验 `Origin` 请求头**，非法就回 `403 Forbidden`。否则攻击者可用 DNS 重绑定从远程网页调你本地的 MCP Server。本地运行**只绑 `127.0.0.1`**，别绑 `0.0.0.0`。（来源：MCP 规范 Security / mcp.directory 审计——"Origin 校验是自定义 Server 最常被漏掉的一行"）
2. **CORS（Cross-Origin Resource Sharing，跨域资源共享）头**：浏览器/多源场景要放行 `mcp-session-id`、`mcp-protocol-version`、`Authorization`、`Content-Type`，并 `expose_headers=["mcp-session-id"]`。
3. **nginx 反代**：`proxy_buffering off`（缓冲会毁掉流式增量响应；或用 `X-Accel-Buffering: no` 头让 nginx 主动关缓冲）、`proxy_read_timeout` 调大到 300s（默认 60s 长工具会断）。
4. **EventStore（事件仓库）**：长耗时工具 > 客户端读超时，需要它存进度事件以便断线重连重放（稳定版特性）；多 worker 用 **Redis 后端**让事件跨进程存活。
5. **stateless_http 水平扩容**：纯函数工具开无状态，直接 `--workers 4`；否则多实例下会话在内存、请求被负载均衡分到不同实例会 404。
6. **sticky session（会话保持）不靠谱**：Cursor、Claude Code 等客户端内部用 `fetch()`，**不转发 Cookie**，负载均衡器认不出该路由到哪台——所以优先无状态，而非依赖粘性会话。（2026-07-28 之后连这个烦恼都没了，协议层直接无状态。）
7. **断线重连（稳定版）**：SSE 事件带 `id`，断了用 `Last-Event-ID` 头重连，服务器只重放此后的事件（不重不漏）；重试用指数退避。官方 Python SDK 的 `StreamableHTTPTransport` 已内置自动重连（`DEFAULT_RECONNECTION_DELAY_MS=1000`、`MAX_RECONNECTION_ATTEMPTS=2`）。（来源：deepwiki Python SDK 4.3）

## 
> ▶ 对应原理：[[49-MCP协议实现-JSON-RPC与传输层|49-MCP协议实现-JSON-RPC与传输层]]


> ▶ 对应原理：[[51-MCP-Streamable-HTTP与SSE全链路|51-MCP-Streamable-HTTP与SSE全链路]]

相关链接

- 项目实践：ai-resume: MCP协议
- 项目实践：[[笔记/技术学习/项目开发笔记/cr-agent笔记/01-技术研读/07-MCP协议实战.md#🔴 记忆级|cr-agent: MCP协议实战]]

---
→ [[技术学习清单#Agent 架构（核心）]]

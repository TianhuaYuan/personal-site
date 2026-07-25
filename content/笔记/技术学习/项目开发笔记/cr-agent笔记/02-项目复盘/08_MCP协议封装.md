---
title: "MCP 协议封装"
tags:
  - 项目笔记
  - cr-agent
created: "2026-07-21"
---

# MCP 协议封装

> FastMCP 选型、5 Tool + 2 Resource 设计，以及两个最深最隐蔽的坑——mount 双前缀和 lifespan 不触发。

## 一、背景（为什么要做这个）

MCP（Model Context Protocol）正在成为 Agent 系统的事实标准——Claude Desktop、VS Code、Cline 都支持。cr-agent 的审查能力通过 MCP 暴露后，外部 Agent 可以直接调用。但 MCP 测试一直全挂，被标记为"环境问题"搁置了 3 个 Phase。Phase 6 的目标就是把 MCP 从"架构就绪"变成"测试通过"。

## 二、逐个讲：问题 → 怎么想 → 怎么解

### 1. FastMCP 选型：5 Tool + 2 Resource

**问题**：MCP 服务器用哪个框架？工具（Tool）和资源（Resource）怎么分？

**怎么想**：FastMCP 是 Python 生态最成熟的 MCP 实现，streamable-http 传输内置，FastAPI 友好。Tool 适合主动调用的操作，Resource 适合被动读取的数据。按这个原则：`ping`、`review_code`、`decompose_code`、`worker_review`、`aggregate_report` 是 Tool；`review://history` 和 `review://stats` 是 Resource。

**怎么解**：5 Tool + 2 Resource，所有工具基于现有的 `services/` 模块封装（不复写审查逻辑）。Tool 注入 `AsyncSessionLocal` 和数据访问依赖，调真实核心服务。Resource 只读数据库，无副作用。

### 2. 11 个 MCP 测试从全挂到全过（conftest 假模块抢注）

**问题**：`from fastmcp import Client` 报 `ImportError: cannot import name 'Client'`。7 个 in-memory 测试全挂，3 个 HTTP 集成测试报 `ModuleNotFoundError: No module named 'asgi_lifespan'`。仅 1 个 `test_mcp_instance_exists` 通过。

**怎么想**：第一反应是 fastmcp 版本变了。但用 venv Python 直接 `from fastmcp import Client` 是成功的。说明是 pytest 环境的问题——conftest 有问题。

**怎么解**：追到 conftest.py 的 `_install_fake_fastmcp()`。它是模块级执行的，在真实 fastmcp 被 import 之前就注册了假模块（只有 FastMCP，没有 Client），导致所有 Client 测试失败。修改：先 `try: import fastmcp; return`，成功就用真的，失败再注册假的。同时发现 `backend/mcp/` 目录和 PyPI `mcp` 包同名——从 `backend/` 目录跑时 `import mcp` 被本地目录劫持，但 pytest 从项目根跑不受影响。

### 3. mount 双前缀坑（/mcp vs /mcp/ 405 问题）

**问题**：`POST /mcp` 返回 405 Method Not Allowed。但直接用 FastMCP 的 http_app 测 `POST /` 是 200。

**怎么想**：不是 MCP 的问题，是 FastAPI mount 的问题。`app.mount("/mcp", mcp_app)` 后，访问 `/mcp` 不带尾斜杠时，Starlette 不会重定向到子应用——直接返回 405，因为主应用没有 `/mcp` 的 POST handler。

**怎么解**：所有测试 URL 从 `/mcp` 改成 `/mcp/`（带尾斜杠）。FastAPI mount 的语义：访问 `/mcp/` 才会转发给子应用的 `/`。不带斜杠是"主应用自己的路径"，没有 POST 处理器就返回 405。

### 4. lifespan 不触发 → ASGI transport 的寿命管理

**问题**：HTTP 集成测试里，MCP 的 lifespan（初始化 task_group）没有被触发。测试报 `RuntimeError: Lifespan not started`。

**怎么想**：HTTPX 的 `ASGITransport` 默认不调用 lifespan。需要用 `LifespanManager`（来自 `asgi_lifespan` 包）手动触发 start/stop。但 `asgi_lifespan` 没在 venv 里。

**怎么解**：不用 `asgi_lifespan`，改用 FastMCP 自带的 `_mcp_app.router.lifespan_context(_mcp_app)`——这是 Starlette 原生方法，返回一个 async context manager，`async with` 包裹即可触发 lifespan 的 startup/shutdown。零额外依赖，且和 `main.py` 的实现一致。

## 三、关键决策与取舍

| 决策 | 选择 | 取舍 |
|------|------|------|
| MCP 框架 | FastMCP（streamable-http） | 比手动实现 MCP spec 少很多代码，但依赖 fastmcp 版本稳定 |
| Tool vs Resource 划分 | 5 Tool + 2 Resource | Resource 的概念额外学习成本，但语义上更精确 |
| 假模块策略 | 先试真实，失败再假 | 比"删干净"更安全——沙箱/CI 没 fastmcp 时还有兜底 |
| lifespan 方案 | `router.lifespan_context()` | 比 asgi_lifespan 更少依赖，且和 main.py 一致 |
| mount 路径 | 改测试 URL 而非加中间件 | 不改生产代码更安全，但 MCP 客户端必须用 `/mcp/` |

## 四、踩坑（值得讲的故事）

**1. 假模块"先发制人"**

现象 → 根因 → 解法 → 教训

conftest.py 在模块级执行 `_install_fake_fastmcp()`，此时 pytest 的 sys.modules 里还没有 fastmcp，假模块抢先注册。之后 `from fastmcp import Client` 拿到的是假模块（只有 FastMCP），所有测试失败。归类为"环境问题"搁置了 3 个 Phase，其实全是自己代码的 bug。教训：**monkeypatch 要做"后发制人"**——只在真实模块不可用时才注册假的。

**2. mount 的尾斜杠陷阱**

现象 → 根因 → 解法 → 教训

`POST /mcp` → 405。"HTTP 方法不对"的误导方向：第一反应是 FastMCP 的 HTTP 传输不支持 POST，或者要用 GET/SSE。查了一小时才发现是路径不对——`/mcp` 不带斜杠根本没转发到 MCP app。Starlette 的 mount 行为和直觉完全不一样：带斜杠转发，不带斜杠直接 405。教训：**FastAPI mount 子应用后，请求 URL 一定要带尾斜杠**。

**3. `asgi_lifespan` 缺失的隐藏假设**

现象 → 根因 → 解法 → 教训

测试报 `ModuleNotFoundError: No module named 'asgi_lifespan'`——这是"测试依赖了一个不被任何 install_requires 声明的包"。隐患：换环境必然挂。解法：原生 `router.lifespan_context()` 替代。教训：**测试不要隐式依赖不被项目声明的包**——要么加进 dev-dependencies，要么用原生替代方案。

## 五、常见疑问

**Q1：Tool 和 Resource 的边界怎么定的？为什么 history/stats 不做成 Tool？**

A：Tool 是主动调用的——你调它，它执行操作（可能还有副作用）。Resource 是被动读取的——像读文件一样读内容，无副作用。history 和 stats 是纯只读的，做成 Resource 语义更精确：MCP 客户端可以像订阅资源一样订阅，缓存内容。做成 Tool 会暗示"调了有影响"，不合适。

**Q2：为什么不直接用 FastMCP 自带的运行方式，要挂到 FastAPI 下面？**

A：主应用是 FastAPI（审查 API、评测 API、静态文件），MCP 只是其中一个模块。挂到 `/mcp` 路径下统一端口、统一部署、统一中间件（鉴权、CORS、日志）。这是微服务里"模块化单体"的思路——一个进程里多个功能模块。如果 MCP 单独起服务，多了部署成本和端口管理。

**Q3：mount 的尾斜杠问题是怎么发现的？**

---

## 相关笔记

- [[01-技术研读/00-17条架构决策总览|00 17条架构决策总览]]
- [[01-技术研读/07-MCP协议实战|07 MCP协议实战]]
- [[10_LLM评测体系搭建|10 LLM评测体系搭建]]

## 技术学习笔记

- [[26-MCP协议核心概念|MCP协议基础]]
- [[20-MCP Server 实现：FastMCP + JWT 认证中间件 + contextvars|MCP Server实现]]
- MCP Transport
- [[22-MCP Client：自定义客户端 + JSON-RPC over HTTP + 连接池 + 幂等重连|MCP Client]]
- [[23-MCP Tool-Resource 定义与注册模式|MCP Tool-Resource]]

A：用一个"排除法"链——先用 FastMCP 的 http_app 直接测（`POST /` 正常），确认 MCP 本身没问题。再用 FastAPI 测（`POST /mcp` 405），确认问题在挂载层。试了 `POST /mcp/` 就通了。最后翻 Starlette 源码确认 mount 语义：`mount("/mcp", app)` 只在 `/mcp/` 前缀匹配时转发，`/mcp` 视为独立路径。

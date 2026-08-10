---
title: "07：MCP 协议实战"
tags:
  - 项目笔记
  - cr-agent
created: "2026-07-21"
---

# 07：MCP 协议实战

## 概述

把 W1 的代码审查能力封装为 **MCP Tools**，外部客户端（Claude Desktop、GitHub Actions、其他 Agent）可通过 MCP 协议调用。MCP 是"壳"，W1 引擎是"核"。

**源码**：`mcp/server.py`（210 行）

---

## 记忆级

### 实例化

📍 **位置**：`mcp/server.py:13-18`

```python
from fastmcp import FastMCP
mcp = FastMCP("cr-agent")
```

### 挂载方式（`main.py`）

📍 **位置**：`main.py:XX-XX`

```text
_mcp_app = mcp.http_app(transport="streamable-http", stateless_http=True, path="/")
_mcp_auth_app = _BearerAuthMiddleware(_mcp_app, _verify_mcp_token)
app.mount("/mcp", _mcp_auth_app)
```

- 经 BearerAuthMiddleware ASGI 层鉴权（开发态放行，生产态验证 JWT）
- Starlette mount 剥离 `/mcp` 前缀后转发到 MCP 内部根路由

### 关键坑：Mounted app lifespan 不自动触发

MCP 的 StreamableHTTPASGIApp 需要 lifespan 初始化 session_manager/task_group。挂载的 sub-app 生命周期不会自动触发，**需要在 FastAPI lifespan 中手动触发**：

📍 **位置**：`main.py:lifespan`

```python
async with _mcp_app.router.lifespan_context(_mcp_app):
    yield
```

---

## 原理级

| Tool | 功能 | 内部调用 | 粒度 |
|------|------|----------|------|
| `ping` | 健康检查 | 直接返回 `{"pong": True}` | 最细—无需引擎 |
| `review_code` | 完整代码审查 | `build_supervisor_graph().ainvoke()` | 最粗—终端用户用 |
| `decompose_code` | 审查任务拆解 | `decompose_node(state)` | 中—查看拆解结果 |
| `worker_review` | 单 Worker 审查 | `WORKERS[role].review()` | 细—编排层用 |
| `aggregate_report` | 聚合去重生成报告 | `aggregate_findings()` + `generate_report()` | 中—配合 worker_review |

**Tool 粒度设计**：review_code（粗粒度）面向终端用户；worker_review 与 aggregate_report（细粒度）面向编排层。外部客户端不需要知道 LangGraph/Worker/Aggregator 的内部实现。

**错误处理**：worker_review 遇到未知 role 返回 info 级别错误 finding，而非抛异常——保持 MCP Tool 的输出类型一致。

---

## 了解级

### Resource：review://history

📍 **位置**：`mcp/server.py:XX-XX`

```python
@mcp.resource("review://history")
async def review_history() -> list:
    """返回最近 10 条审查记录（id / 语言 / 状态 / 创建时间）。"""
```

查 `Review` 表，按 `created_at` 倒序取 10 条。

### Resource：review://stats

📍 **位置**：`mcp/server.py:XX-XX`

```python
@mcp.resource("review://stats")
async def review_stats() -> dict:
    """返回审查统计信息（总数 / 各状态数量）。"""
```

查 `Review` 表 COUNT + GROUP BY status。

### FastMCP vs 官方 low-level SDK

| 维度 | FastMCP（本项目选用） | 官方 low-level SDK |
|------|----------------------|-------------------|
| 代码量 | 装饰器声明 Tool/Resource，~50 行 | 手动 Server + handlers + request/response 序列化 |
| 流式传输 | `http_app(transport="streamable-http")` 一行切换 | 需实现 ASGI lifespan + SSE/streamable 协议 |
| Resource 声明 | `@mcp.resource("uri")` 直接映射 | 需注册 `list_resource_templates` + `read_resource` handler |
| 测试 | FastMCP Client in-memory 测试 | 需启动 MCP 子进程+stdio 通信 |
| 灵活性 | 自动处理 JSON-RPC 序列化/反序列化 | 完全控制每个字段 |
| 结论 | **够用就近**：本项目不涉及自定义传输层或非标准 JSON-RPC 扩展，FastMCP 足以覆盖全部需求 |

---

## Q&A

**Q**：为什么不直接用 FastAPI 路由暴露 API，非要套 MCP？  
**A**：MCP 是标准协议，Claude Desktop 原生支持 MCP 连接。如果未来想"把代码审查接入 Claude Desktop / Cursor / Windsurf"，MCP 是标准通路。

**Q**：FastMCP 是否有生产环境坑？  
**A**：FastMCP 较新，坑主要在 lifespan 触发 + path 映射。本项目已验证：`mount("/mcp")` + `http_app(path="/")` 正确，lifespan 手动触发可用。

**Q**：Tools 和 Resources 的鉴权怎么做的？  

---

## 相关笔记

- [[00-17条架构决策总览|00 17条架构决策总览]]
- [[05-API与CLI接口契约|05 API与CLI接口契约]]
- [[08-GitHub集成|08 GitHub集成]]

## 技术学习笔记

- [[26-MCP协议核心概念|MCP协议基础]]
- [[20-MCP Server 实现：FastMCP + JWT 认证中间件 + contextvars|MCP Server实现]]
- MCP Transport
- [[22-MCP Client：自定义客户端 + JSON-RPC over HTTP + 连接池 + 幂等重连|MCP Client]]
- [[23-MCP Tool-Resource 定义与注册模式|MCP Tool-Resource]]
**A**：在 ASGI 层统一拦截（`_BearerAuthMiddleware`），不依赖 MCP 本身鉴权。验证 `Authorization: Bearer <JWT>`，开发态放行，生产态必须有效 token。

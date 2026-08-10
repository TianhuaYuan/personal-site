---
title: "Spec: cr-agent — 多Agent代码审查协作平台"
tags:
  - 项目笔记
  - cr-agent
created: "2026-07-21"
---

# Spec: cr-agent — 多Agent代码审查协作平台

## Objective

### 是什么
一个基于 LangGraph Supervisor-Worker 架构的多 Agent 代码审查系统。用户提交代码（本地文件/GitHub PR）→ Supervisor 拆解审查任务 → 4 个专业 Worker Agent 并行审查 → Aggregator 合并去重 → 输出结构化审查报告。

### 用户是谁
我（求职展示）+ （3 分钟内理解架构 + 追问技术决策）。

### 成功标准
- 第 1 周结束时：**提交一段 Python/JS 代码 → 返回一份 Markdown 审查报告**，端到端链路跑通
- 看完能追问 "你的 Supervisor 怎么拆任务的？" "死循环检测怎么做的？" "为什么不用 CrewAI？"——每个问题我都能用亲手写的代码回答

### 不做什么
- 不做 GitHub PR 自动发布（第 2 周做）
- 不做复杂前端 UI（只有 API + CLI 入口）
- 不用 CrewAI / AutoGen / Google ADK
- 不追求 100% 准确率（第 3 周评测）

---

## Tech Stack

| 层 | 技术 | 版本 | 与 1 号项目的关系 |
|---|------|------|------------------|
| 编排 | LangGraph (StateGraph) | 与 1 号项目同版 | 复用经验，模式从串行升到并行 |
| LLM | Chat API（同 1 号项目配置） | — | 直接复用 `rag/clients.py` 的 Chat 客户端 |
| 后端 | FastAPI + async | 0.115+ | 复用 main.py 模式 + 中间件链 |
| 数据 | SQLAlchemy async + SQLite（开发）/ MySQL（生产） | — | 直接拷 `core/database.py` |
| MCP | FastMCP + JSON-RPC 2.0 | — | 复用 MCP Server/Client 实现 |
| 工具 | httpx, Pydantic v2 | — | 复用 |
| 容器 | Docker + Compose | — | 复制 Dockerfile 模板 |
| CI/CD | GitHub Actions | — | 复制 ci.yml 模板 |
| 测试 | pytest + pytest-asyncio | — | 复制 pyproject.toml |

---

## Commands

```text
# 开发环境
启动:     docker compose -f docker-compose.dev.yml up -d
停止:     docker compose -f docker-compose.dev.yml down
安装依赖: pip install -r backend/requirements.txt
配置:     cp backend/.env.example backend/.env.dev

# 测试
全部测试:  pytest backend/tests/ -v
单文件:    pytest backend/tests/test_supervisor.py -v
覆盖率:    pytest backend/tests/ --cov=backend --cov-report=term-missing

# 代码质量
lint:     ruff check backend/
format:   ruff format backend/
类型检查:  mypy backend/

# CLI 审查（核心入口）
审查文件:  python -m backend.cli review --file path/to/code.py
审查 PR:   python -m backend.cli review --pr https://github.com/owner/repo/pull/1

# 构建
构建镜像:  docker build -t cr-agent backend/
启动全部:  docker compose up -d
```

---

## Project Structure

```text
D:\Project\cr-agent/
├── README.md
├── .gitignore
├── .claude/
│   └── settings.json
├── .github/
│   └── workflows/
│       └── ci.yml
├── docker-compose.yml
├── docker-compose.dev.yml
│
├── backend/
│   ├── .env.example
│   ├── .env.dev
│   ├── requirements.txt
│   ├── pyproject.toml
│   ├── Dockerfile
│   ├── main.py                  # FastAPI 入口
│   │
│   ├── core/                    # ← 从 1 号项目拷贝（不改逻辑）
│   │   ├── config.py
│   │   ├── database.py
│   │   ├── security.py
│   │   ├── exceptions.py
│   │   ├── retry.py
│   │   ├── trace.py
│   │   ├── metrics.py
│   │   ├── cache.py
│   │   ├── limiter.py
│   │   ├── request_id.py
│   │   ├── logging_config.py
│   │   └── error_types.py
│   │
│   ├── models/                  # 审查相关数据模型
│   │   ├── __init__.py
│   │   ├── review.py            # Review（id, status, code_content, report）
│   │   └── task.py              # ReviewTask（supervisor 拆解的子任务）
│   │
│   ├── schemas/                 # Pydantic 请求/响应
│   │   ├── __init__.py
│   │   ├── review.py            # ReviewRequest / ReviewResponse / ReviewReport
│   │   └── task.py              # TaskSchema / TaskResult
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   ├── deps.py              # 依赖注入
│   │   └── reviews.py           # POST /reviews, GET /reviews/{id}
│   │
│   ├── cli/
│   │   ├── __init__.py
│   │   └── main.py              # CLI 入口：python -m backend.cli review
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   ├── supervisor/          # Supervisor Agent
│   │   │   ├── __init__.py
│   │   │   ├── state.py         # SupervisorState TypedDict
│   │   │   ├── graph.py         # StateGraph 装配
│   │   │   ├── decompose.py     # 任务拆解节点
│   │   │   └── route.py         # 路由/分发节点
│   │   │
│   │   ├── workers/             # Worker Agent（4 个专业审查器）
│   │   │   ├── __init__.py
│   │   │   ├── base.py          # BaseWorker 抽象
│   │   │   ├── quality.py       # 代码规范/风格
│   │   │   ├── security.py      # 安全漏洞/依赖
│   │   │   ├── performance.py   # 性能瓶颈/复杂度
│   │   │   └── structure.py     # 设计模式/架构
│   │   │
│   │   ├── aggregator/          # 结果聚合
│   │   │   ├── __init__.py
│   │   │   ├── merge.py         # 合并去重
│   │   │   └── report.py        # 生成 Markdown 报告
│   │   │
│   │   └── mcp/                 # MCP Gateway（先占位，第 2 周实现）
│   │       └── __init__.py
│   │
│   ├── tests/
│   │   ├── __init__.py
│   │   ├── conftest.py
│   │   ├── test_supervisor.py
│   │   ├── test_worker_quality.py
│   │   ├── test_worker_security.py
│   │   ├── test_aggregator.py
│   │   └── test_e2e.py
│   │
│   └── alembic/
│       ├── env.py
│       └── versions/
│
├── tasks/
│   ├── spec.md                  # 本文件
│   ├── plan.md                  # 实现计划
│   └── todo.md                  # 任务清单
│
└── samples/                     # 测试用代码样本
    ├── sample_bad_python.py     # 有问题的 Python 代码
    ├── sample_bad_js.js          # 有问题的 JS 代码
    └── sample_good_python.py    # 没问题的 Python 代码（对照组）
```

---

## Code Style

### State 定义（TypedDict，与 1 号项目同模式）

```python
from typing import TypedDict, Annotated, Sequence
from langgraph.graph.message import add_messages
from langchain_core.messages import BaseMessage

class SupervisorState(TypedDict):
    """Supervisor 工作流状态

    所有节点共享，通过 StateGraph 的条件边路由。
    """

    # 输入
    code: str                          # 原始代码内容
    language: str                      # 代码语言（python/javascript/typescript）
    review_id: str                     # 审查任务 ID（UUID）

    # Supervisor 拆解
    tasks: list[dict]                  # [{role: "security", description: "...", priority: 1}, ...]

    # Worker 结果（Annotated reducer 累积）
    worker_results: Annotated[list[dict], add_messages]  # 各 Worker 返回的审查发现

    # 聚合
    report: str                        # 最终 Markdown 审查报告

    # 控制
    iteration_count: int               # 当前迭代次数（防死循环）
    max_iterations: int                # 最大迭代次数（默认 3）
    errors: list[str]                  # 聚合的错误信息
```

### 节点签名（统一规范）

```python
# 每个节点都是一个 async 函数，接收 state、返回 dict（局部更新）
async def decompose_node(state: SupervisorState) -> dict:
    """Supervisor: 分析代码，拆解审查任务列表。"""
    ...

# 图装配
from langgraph.graph import StateGraph, END

def build_supervisor_graph() -> StateGraph:
    workflow = StateGraph(SupervisorState)

    workflow.add_node("decompose", decompose_node)
    workflow.add_node("quality_worker", quality_worker_node)
    workflow.add_node("security_worker", security_worker_node)
    workflow.add_node("performance_worker", performance_worker_node)
    workflow.add_node("structure_worker", structure_worker_node)
    workflow.add_node("aggregate", aggregate_node)

    workflow.set_entry_point("decompose")

    # 拆解后 → 并行分发到 4 个 Worker
    workflow.add_edge("decompose", "quality_worker")
    workflow.add_edge("decompose", "security_worker")
    workflow.add_edge("decompose", "performance_worker")
    workflow.add_edge("decompose", "structure_worker")

    # 所有 Worker → 聚合
    workflow.add_edge("quality_worker", "aggregate")
    workflow.add_edge("security_worker", "aggregate")
    workflow.add_edge("performance_worker", "aggregate")
    workflow.add_edge("structure_worker", "aggregate")

    workflow.add_edge("aggregate", END)

    return workflow.compile()
```

### Worker 基类

```python
from abc import ABC, abstractmethod

class BaseWorker(ABC):
    """审查 Worker 基类。

    每个 Worker 接收代码内容，返回结构化的审查发现列表。
    """

    role: str                  # "security" | "quality" | "performance" | "structure"
    system_prompt: str         # Worker 专属的 system prompt

    @abstractmethod
    async def review(self, code: str, language: str) -> list[dict]:
        """审查代码，返回发现列表。

        返回格式：[{"severity": "high", "line": 12, "category": "...",
                    "description": "...", "suggestion": "...", "code_snippet": "..."}]
        """
        ...

    async def _call_llm(self, prompt: str) -> str:
        """统一的 LLM 调用封装，复用 1 号项目的 Chat 客户端。"""
        ...
```

### API 路由（FastAPI，复用 1 号项目模式）

```python
# backend/api/reviews.py
from fastapi import APIRouter, Depends, HTTPException
from backend.schemas.review import ReviewRequest, ReviewResponse

router = APIRouter(prefix="/reviews", tags=["reviews"])

@router.post("", response_model=ReviewResponse, status_code=202)
async def create_review(req: ReviewRequest):
    """提交代码审查。返回 202 Accepted + review_id，后台异步执行。"""
    ...

@router.get("/{review_id}", response_model=ReviewResponse)
async def get_review(review_id: str):
    """查询审查结果。status: pending → running → completed/failed。"""
    ...
```

### 命名约定

- 文件：`snake_case`，模块按功能拆分（`supervisor/`, `workers/`, `aggregator/`）
- 函数：`async def xxx_node(state) -> dict`（LangGraph 节点统一后缀 `_node`）
- 类：`PascalCase`（`BaseWorker`, `SupervisorState`）
- 数据模型：Pydantic `BaseModel`，SQLAlchemy `Base`
- Worker 名称：`quality`, `security`, `performance`, `structure`（全小写）

---

## Testing Strategy

| 层级 | 框架 | 位置 | 覆盖率目标 | 说明 |
|------|------|------|-----------|------|
| 单元测试 | pytest + pytest-asyncio | `backend/tests/test_worker_*.py` | >80% | 每个 Worker 的 review 逻辑 |
| 集成测试 | pytest | `backend/tests/test_supervisor.py` | 核心路径 | Supervisor 拆解 + 分发 |
| 端到端 | pytest | `backend/tests/test_e2e.py` | 至少 5 条 | 提交代码 → 返回报告 |
| LLM Mock | unittest.mock | conftest.py | — | 所有 LLM 调用在测试中 mock |

### 测试原则

- 每个测试函数**不超过 10 行设置代码**
- LLM 调用**全部 mock**——测试测的是**图结构的逻辑**，不是 API 调用
- 关键边界：空代码、超长代码、单行代码、多语言混用
- 错误路径：LLM 返回格式错误 → Worker 降级处理

---

## Boundaries

### Always do
- 节点签名统一 `async (state) -> dict`
- LLM 调用通过 Chat 客户端单例（不直接 `openai.ChatCompletion`）
- 异常透传到 State.errors（不在节点内部吞异常）
- commit 前跑 `pytest backend/tests/ -v`

### Ask first
- 修改 State 定义（影响所有节点）
- 添加新 Worker 类型
- 添加新依赖（pip install）
- 修改数据库 schema（需要 alembic 迁移）
- 修改 CI/CD 配置

### Never do
- 在 Worker 内部直接 `print()`（用 logging）
- 硬编码 API Key 或 Secret
- 在节点内做同步阻塞 I/O
- 超过 3 次迭代不熔断（max_iterations 硬限制）

---

## Success Criteria (Week 1)

- [x] `python -m backend.cli review --file samples/sample_bad_python.py` 返回 Markdown 报告
- [x] 报告包含至少 3 个维度的审查发现（质量/安全/性能/结构）
- [x] 4 个 Worker 并行执行（日志可验证）
- [x] Supervisor 拆解了 ≥2 个子任务
- [x] max_iterations=3 熔断生效
- [x] 端到端耗时 < 60s（含 LLM 调用）
- [x] `pytest backend/tests/` 全绿，覆盖率 > 70%

---

## Resolved Decisions

完整 17 条决策记录见 `01-架构决策总览.md`。以下是 W1 起点的 3 条核心决策：

| # | 问题 | 决策 | 理由 |
|---|------|------|------|
| 1 | Worker LLM 路由 | 统一模型 | 第 1 周先跑通链路；第 3 周评测后按数据做分级路由 |
| 2 | 结果存储 | SQLite | 复用 1 号项目 `core/database.py`，支持按 severity/category 回溯 |

---

## 相关笔记

- [[00-17条架构决策总览|00 17条架构决策总览]]
- [[06-项目搭建与TDD实践|06 项目搭建与TDD实践]]
- [[09-LLM评测体系搭建|09 LLM评测体系搭建]]

## 技术学习笔记

- [[15-Agent架构与工具调用|Agent架构与工具调用]]
- [[44-LangChain框架入门|LangChain框架入门]]
- [[26-MCP协议核心概念|MCP协议基础]]
- [[语言与框架/MySQL/实战/06-SQLAlchemy与ORM实战|SQLAlchemy与ORM实战]]
- [[工程化与运维/工程化部署/01-Docker多阶段构建|Docker多阶段构建]]
| 3 | Worker 失败策略 | 跳过 + 记录 error | 渐进演进：第 1 周 skip → 第 2 周加 timeout 重试 + 熔断

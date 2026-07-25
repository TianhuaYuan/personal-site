---
title: "06 — 项目搭建与 TDD 实践"
tags:
  - 项目笔记
  - cr-agent
created: "2026-07-21"
---

# 06 — 项目搭建与 TDD 实践

## 了解级

### `core/config.py`（129行）

**一句话**：Pydantic Settings 双层 `.env` 加载（通用 + 环境特定），所有环境变量统一前缀 `CR_AGENT_`，启动时 `validate_required_settings()` fail-fast 校验。

📍位置：
- `env_prefix="CR_AGENT_"`（`config.py:97`）
- 双层 env_file：`.env` + `.env.{APP_ENV}`（`config.py:99-103`）
- `_resolve_role_model_defaults`：空 model 字段回退到 `CHAT_MODEL`（`config.py:43`）
- `validate_required_settings`：检查 `CHAT_API_KEY` / `CHAT_BASE_URL` / `CHAT_MODEL` 非空（`config.py:117`）

### `core/database.py`（39行）

**一句话**：模块级 `engine` 由 `settings.DATABASE_URL`（默认 SQLite async）构建，`get_db` yield session，`init_db` 连通性验证 + 自动建表。

📍位置：
- `engine = create_async_engine(settings.DATABASE_URL, ...)`（`database.py:12`）
- `get_db`：`async with AsyncSessionLocal() as session: yield session`（`database.py:29`）
- `init_db`：`SELECT 1` + `Base.metadata.create_all`（`database.py:35`）
- `expire_on_commit=False`：commit 后对象还能用（`database.py:21`）

### `core/llm.py`（42行）

**一句话**：`get_chat_client()` 惰性创建 `AsyncOpenAI` 单例，显式禁用环境代理（`trust_env=False`），`reset_chat_client()` 供测试重置。

📍位置：
- `get_chat_client`：全局 `_chat_client` 为 None 时创建（`llm.py:20`）
- `http_client=httpx.AsyncClient(trust_env=False)`：避免 Clash/V2Ray 系统代理干扰（`llm.py:34`）
- `reset_chat_client`：`_chat_client = None`（`llm.py:39`）

### `core/common.py`（68行）

**一句话**：`extract_json_array` 从 LLM 文本剥 markdown 围栏后正则提取 JSON 数组；`fetch_pr_code` 从 PR URL 获取代码和语言。

📍位置：
- `extract_json_array`：先剥 ` ```json ` → 正则 `\[.*\]` → JSON 解析 → 兜底匹配「无问题」关键词（`common.py:13`）
- `fetch_pr_code`：GitHubClient → parse_pr_url → get_pr_patch → parse_patch_to_code → detect_language（`common.py:56`）

### `models/review.py`（34行）

**一句话**：`Review` ORM，主键自增 id，`status` 流转 pending→running→completed/failed，`report` Text 存完整 Markdown。

📍位置：
- `__tablename__ = "reviews"`（`review.py:19`）
- 字段：id / status / code_content / language / report / created_at / updated_at（`review.py:21-33`）
- `_utcnow`：`datetime.now(timezone.utc)`（`review.py:14`）

### `models/task.py`（25行）

**一句话**：`ReviewTask` ORM，外键关联 Review，role 标记子任务类型（quality/security/performance/structure），`findings` 用 SQLAlchemy JSON 列自动反序列化。

📍位置：
- `ForeignKey("reviews.id", ondelete="CASCADE")`（`task.py:16`）
- `findings: Mapped[list] = mapped_column(JSON, ...)`：用 JSON 类型而非 Text，避免 `[]` 存成字符串（`task.py:24`）

### `schemas/task.py`（32行）

**一句话**：`TaskSchema` 定义子任务（role + description + priority），`TaskResult` 回传审查结果（findings + error + status）。

📍位置：
- `TaskSchema.role`：决定派给哪个 Worker（`task.py:16`）
- `TaskResult.findings`：结构化审查发现列表（`task.py:30`）
- `TaskResult.error`：非空表示该 Worker 挂了，不影响其他 Worker（`task.py:31`）

### `tests/conftest.py`（137行）

**一句话**：`_install_fake_fastmcp` 绕过 slim 版 fastmcp 的 import 错误；`async_session` 每个测试独立内存 SQLite + 自动建表；`mock_llm` 用 `_FakeChatClient` 替换 `get_chat_client` 免真实 API key。

📍位置：
- `_install_fake_fastmcp`：`del sys.modules[key]` 卸载 slim → 注册 `_FakeFastMCP`（`conftest.py:14`）
- `_FakeChatClient`：实现 `chat.completions.create` 返回可控 `response_text`（`conftest.py:77`）
- `async_session` fixture：`StaticPool` 单一连接保证内存库多连接可见（`conftest.py:98`）
- `mock_llm` fixture：`monkeypatch.setattr("backend.core.llm.get_chat_client", ...)`（`conftest.py:116`）
- `fake_llm_factory` fixture：可定制 response_text 的工厂（`conftest.py:126`）

---

## 🎯 追问集

### Q1: 为什么 ORM 选择 SQLAlchemy 2.0 async 而非 Tortoise ORM 或 Django ORM？

**A**：三个原因：① SQLAlchemy 2.0 的 `Mapped` / `mapped_column` 声明式语法已接近现代 ORM 的简洁度，学习成本不高；② FastAPI 生态原生配合 SQLAlchemy async（`AsyncSession`/`async_sessionmaker`），Dependency Injection 的 `get_db` yield session 模式是标准实践；③ SQLAlchemy 的 JSON 列原生支持（`mapped_column(JSON)`）比 Tortoise 的 JSONField 在类型提示和反序列化一致性上更成熟——本项目 `ReviewTask.findings` 直接用 SQLAlchemy JSON 类型，避免 `字符串 "[ ]" vs 列表 []"` 的坑。

### Q2: `ReviewTask.findings` 用了 JSON 列而不是子表，为什么？什么场景下该拆成子表？

---

## 相关笔记

- [[00-17条架构决策总览|00 17条架构决策总览]]
- [[05-API与CLI接口契约|05 API与CLI接口契约]]
- [[08-GitHub集成|08 GitHub集成]]

## 技术学习笔记

- [[15-Agent架构与工具调用|Agent架构与工具调用]]
- [[笔记/技术学习/MySQL 实战/06-SQLAlchemy与ORM实战|SQLAlchemy与ORM实战]]
- [[笔记/技术学习/工程化与部署/08-pytest异步测试基础设施|pytest异步测试]]
- unittest.mock
- [[笔记/技术学习/工程化与部署/07-pre-commit代码规范|pre-commit代码规范]]

**A**：当前用 JSON 列是因为 findings 的结构是**写后即读**（Worker 产出 → aggregate 合并 → report 渲染），不需要对单个 finding 做独立查询或关联。JSON 列省去子表的 CRUD 样板代码和 JOIN 开销。如果后续需求变为：① 按 severity 筛选 finding（"只看 critical 级别的"），② 按代码行号定位，③ finding 需要独立的状态流转或评论，那就该拆成 `Finding` 子表 + `review_id` 外键，并加对应索引。

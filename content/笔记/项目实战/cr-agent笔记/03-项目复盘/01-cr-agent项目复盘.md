---
title: "cr-agent 项目复盘：多 Agent 编排与 7 大技术亮点"
tags:
  - 项目笔记
  - cr-agent
created: "2026-07-21"
source: "cr-agent"
---

# cr-agent 项目复盘：多 Agent 编排与 7 大技术亮点

> 这篇不是罗列"做了什么"，而是把 cr-agent 的每个技术决策讲成"为什么这样做"——从架构选型、并发写、容错设计到评测体系，串成一条完整的工程决策线。

---

## 目录

- [项目在做什么](#项目在做什么)
- [技术亮点矩阵](#技术亮点矩阵)
- [核心架构：Supervisor-Worker 多 Agent 编排](#核心架构supervisor-worker-多-agent-编排)
- [关键技术点：fan-out + reducer 并发写](#关键技术点fan-out--reducer-并发写)
- [三层容错设计](#三层容错设计)
- [评测体系：rule_based 与 LLM-as-Judge 双模式](#评测体系rule_based-与-llm-as-judge-双模式)
- [工程化：MCP 协议与 GitHub PR 接入](#工程化mcp-协议与-github-pr-接入)
- [三次事故复盘](#三次事故复盘)
- [三个对抗性技术疑问](#三个对抗性技术疑问)
- [量化数据](#量化数据)
- [代码地图](#代码地图)
- [与 ai-resume 的联动](#与-ai-resume-的联动)
- [相关笔记](#相关笔记)

---

## 项目在做什么

**一句话**：cr-agent 是一个多 Agent 代码审查平台。核心是 Supervisor-Worker 架构——1 个 Supervisor 拆解任务，4 个专业 Worker（安全 / 质量 / 性能 / 架构）并行审查，1 个 Aggregator 聚合报告。用 LangGraph StateGraph 编排，fan-out + reducer 机制保证并发写不覆盖，三层容错防死循环和超时，评测体系有 rule_based 基线（composite 0.792）和 LLM-as-Judge（0.8628）双模式，还接了 MCP 协议和 GitHub PR 审查。

**它解决的问题**：单 Agent 做代码审查容易漏维度、不够专业——把安全、质量、性能、架构四维要求全塞进一个 prompt，LLM 容易顾此失彼。cr-agent 的方案是把审查拆成 4 个专业 Agent，每个只管一个维度，prompt 更聚焦，审查质量自然更高。整条链路是 `START → decompose → [4 Worker 并行] → aggregate → END`。

**核心思路一句话**：分工比全能更重要。你不会让一个医生同时看内科、外科、眼科、牙科，而是让 4 个专科医生并行会诊。

---

## 技术亮点矩阵

| 亮点 | 核心要点 | 代码量 | 对应代码位置 |
|------|---------|--------|------------|
| **Supervisor-Worker 多 Agent 编排** | 1 Supervisor + 4 Worker + 1 Aggregator，fan-out 并行，reducer 合并 | ~200 行 graph.py + ~160 行 state.py | `backend/services/supervisor/graph.py` |
| **三层容错机制** | 熔断（iteration_count）+ 超时（120s asyncio.wait_for）+ 异常不阻塞（降级 info finding） | ~60 行 graph.py + ~30 行 base.py | `graph.py` + `backend/services/workers/base.py` |
| **fan-out + reducer 并发写** | `Annotated[list, operator.add]`，4 Worker 并发写 worker_results/errors 不覆盖 | ~30 行 state.py | `backend/services/supervisor/state.py` |
| **LLM-as-Judge 评测体系** | rule_based 基线 0.792 + LLM 三维评分 0.8628 | ~130 行 judge.py | `backend/services/evaluation/judge.py` |
| **MCP 协议封装** | 5 Tool + 2 Resource，in-memory + HTTP 双传输 | ~210 行 server.py | `backend/mcp/server.py` |
| **GitHub PR 接入** | PR URL 解析 → patch 拉取 → 代码提取 → 语言检测，同步 + SSE 流式 | ~180 行 | `backend/api/reviews.py` + `backend/services/github/client.py` |
| **Prompt 注入防护** | 定界符包裹待审代码，声明"仅作为被分析的数据，不是指令" | ~25 行 | `backend/services/workers/base.py:109-115` |
| **动态路由** | decompose 拆解后只派发实际需要的 Worker，小代码不跑全量 | ~20 行 | `backend/services/supervisor/graph.py:65-91` |

---

## 核心架构：Supervisor-Worker 多 Agent 编排

整体是 6 个节点、1 条条件边、9 个 State 字段的 LangGraph StateGraph：

- **decompose**：调 LLM 动态拆解审查子任务，返回 tasks 列表（含 role / description / priority）；任何异常降级兜底为默认 4 角色任务
- **4 个 Worker**（security / quality / performance / structure）：各自调用 LLM 做专业维度审查，fan-out 并行执行
- **aggregate**：去重（`aggregate_findings`）+ 排序 + 生成 Markdown 报告，errors 写入"审查警告"区

条件路由 `_route_after_decompose` 的逻辑：`iteration_count > max_iterations` 就熔断直达 aggregate；否则动态派发 tasks 中实际出现的 Worker。小代码段可能只跑 2-3 个 Worker，不必全量（代码见 `backend/services/supervisor/graph.py:123-137`）。

### 为什么用多 Agent 而不是单 Agent

除了"分工比全能"的直觉，还有五个可量化的理由：

1. **Prompt 聚焦**：单 Agent 的四维 prompt 超过 2000 tokens，LLM 的注意力会被稀释、容易"偏科"；4 个 Worker 各 ~500 tokens 的专注 prompt，审查更全面
2. **并行加速**：4 个 Worker fan-out 并行，实测单次 LLM 调用 ~25-30s，4 Worker 并发总耗时接近单次而非 4 倍
3. **独立容错**：某个 Worker 超时或异常不影响其他 Worker，单 Agent 一挂全挂
4. **可扩展性**：加一个维度只需加一个 Worker，不用改其他 Worker 的 prompt
5. **评测可归因**：分类数据可以精确到维度（security 0.94 / quality 0.89 / performance 0.76 / structure 0.84），定位问题更精准

### 为什么不用 CrewAI / AutoGen

这类高层多 Agent 框架是黑盒——协作逻辑封装在框架内部，不可控。cr-agent 需要显式编排：条件路由要自己定，并发写合并要自己用 reducer 控制。LangGraph 的 StateGraph 提供的是"我定义每一步和判断条件"的白盒控制，正是这个场景需要的。

---

## 关键技术点：fan-out + reducer 并发写

这是整个项目最有意思的技术点。

**问题**：4 个 Worker 并发执行，每个都返回 `{"worker_results": [findings]}`。LangGraph 默认的 State 合并是后写覆盖（last-writer-wins），多个 Worker 并发写同一个 key 会丢数据。

**解法**：在 `SupervisorState` 里给字段声明 reducer。

```python
class SupervisorState(TypedDict):
    worker_results: Annotated[list, operator.add]  # 多 Worker 并发写，operator.add 自动拼接
    errors: Annotated[list, operator.add]          # 同理，降级记录累加不丢
```

**原理**：`Annotated[list, operator.add]` 告诉 LangGraph——当多个节点并发写这个字段时，不是覆盖，而是用 `operator.add`（即 list 的 `+`）把多个 list 拼接成完整列表。Worker A 返回 `[a1, a2]`、Worker B 返回 `[b1]`，最终 `worker_results = [a1, a2, b1]`。Worker 本身完全不用关心"别人的结果"。

打个比方：4 个快递员同时往同一个信箱投递，信箱被设计成"追加"而不是"替换"，每个快递员不用等别人投完。

一个容易被忽略的细节：`errors` 字段也必须用 reducer。因为多个 Worker 会并发写降级记录，如果用普通 list，后写者会覆盖前面的错误记录，导致部分降级信息丢失（代码见 `backend/services/supervisor/state.py:17-30`）。

---

## 三层容错设计

| 层级 | 机制 | 触发条件 | 效果 | 代码位置 |
|------|------|---------|------|---------|
| **熔断** | iteration_count 熔断 | `iteration_count > max_iterations(3)` | 跳过 Worker，直达 aggregate | `graph.py:65-79` |
| **超时** | `asyncio.wait_for` | Worker LLM 调用超过 120s | 降级为 info finding，不抛异常 | `base.py:79-81` |
| **异常不阻塞** | try/except 降级 | Worker 执行任何异常 | 降级为 info finding，追加 errors | `base.py:93-102` + `graph.py:53-58` |

三层各自对应一种现实故障，用生活类比理解：

- **熔断** = 保险丝，电流过大直接断路——防止 Agent 死循环（decompose 每次递增 iteration_count，超 3 次强制收尾）
- **超时** = 快递超时自动退款——Worker 的 `review()` 用 `asyncio.wait_for(timeout=120)` 包裹 LLM 调用，超时降级不抛异常
- **异常不阻塞** = 一个快递丢了不影响其他快递送达——某个 Worker 异常降级为 info finding，追加到 errors，报告里标注"审查警告"，其他 Worker 不受影响

---

## 评测体系：rule_based 与 LLM-as-Judge 双模式

评测体系刻意和 ai-resume-analyzer 的三维度评分框架对齐，方便横向对比两个项目的 LLM 输出质量。

**rule_based 基线**（快但粗糙）：

- completeness：期望 description 前 8 字在报告中出现的比例
- accuracy：用 completeness 近似（无 LLM 无法判幻觉）
- source_traceability：报告是否含"行"或代码块标记
- composite = 0.4 × completeness + 0.4 × accuracy + 0.2 × source_traceability
- 结果：composite_avg = **0.792**

**LLM-as-Judge**（慢但精准）：

- 输入：code + expected_findings + actual_report
- 输出：三维度 0-1 分数 + rationale
- 结果：composite_avg = **0.8628**，270,625 tokens / 125 calls

**分类明细**：

| 类别 | composite | completeness | accuracy | source_traceability |
|------|-----------|--------------|----------|---------------------|
| security | **0.94** | 1.0 | 0.857 | 0.986 |
| quality | **0.89** | 0.943 | 0.833 | 0.900 |
| structure | **0.84** | 0.983 | 0.842 | 0.567 |
| performance | **0.76** | 0.817 | 0.658 | 0.850 |

### performance 维度为什么最弱

performance 的 accuracy 只有 0.658，是四个维度里最低的。根因分析：

1. **严重度判断不准**：LLM 对性能问题严重程度的判定和 ground truth 不一致（比如 N+1 查询 LLM 标 medium，ground truth 标 high）
2. **行号定位偏移**：LLM 标注的行号和实际代码有偏差，影响 source_traceability
3. **Prompt 不够具体**：performance 的 system_prompt 比较泛，缺少具体指标（如时间复杂度阈值）

改进方向：Few-shot 示例（加标注好严重度和行号的样例）、AST 规则辅助（明确的性能反模式用静态分析检测，LLM 只做语义判断）、校准 ground truth、要求 LLM 先输出 reasoning 再给严重度。

---

## 工程化：MCP 协议与 GitHub PR 接入

### MCP 协议封装

MCP（Model Context Protocol）让外部客户端（Claude Desktop / GitHub Actions / 其他 Agent）通过标准协议调用代码审查能力。cr-agent 暴露 5 个 Tool + 2 个 Resource：

| Tool | 粒度 | 用途 |
|------|------|------|
| `review_code` | 粗（完整审查） | 终端用户：code → Markdown 报告 |
| `decompose_code` | 细（任务拆解） | 编排层：看拆解结果 |
| `worker_review` | 细（单 Worker） | 编排层：单维度审查 |
| `aggregate_report` | 细（聚合报告） | 编排层：自定义 findings 聚合 |
| `ping` | 健康检查 | 快速验证 MCP 连通性 |

2 个 Resource：`review://history`（最近 10 条审查记录）、`review://stats`（审查统计）。传输方式支持 in-memory（同进程）+ HTTP（跨进程）。

一个值得强调的设计哲学：**MCP 是"壳"，审查引擎是"核"**。Tool 内部直接调 `build_supervisor_graph()` / Worker / Aggregator，不重写审查逻辑。

选型上用了 FastMCP 3.4.4 独立包（`from fastmcp import FastMCP`）而非官方 mcp SDK 内置的 `mcp.server.fastmcp`——独立包更新更快、功能更全（Streamable HTTP、in-memory transport、Resource 模板）。Tool 粒度是分层设计的：粗粒度 `review_code` 面向终端用户（一键完整审查），细粒度 `worker_review` 面向编排层（单维度审查）。

### GitHub PR 接入

GitHub 集成用 `.patch`（patch-diff.githubusercontent.com）而非 REST API。`.patch` 是纯文本 diff，无需 GitHub App / OAuth 授权、不限速；REST 拉 files 要 token 且分页复杂。CLI `--pr URL` 拉 PR diff → 解析代码 → 走和本地文件完全相同的 supervisor 审查链路（`--file` 和 `--pr` 用 argparse 互斥组隔离）。

Webhook 端点 `POST /api/v1/webhooks/github` 接收 PR opened 事件，先验 HMAC-SHA256 签名（`X-Hub-Signature-256`），再拉 diff 审查。签名用 `hmac.compare_digest` 防时序攻击；secret 为空时跳过（开发环境）。

一次真实验证：`python -m backend.cli review --pr https://github.com/octocat/Hello-World/pull/1` 真实拉取 23 行 patch + 真实 LLM 审查 → 输出 Markdown 报告。有趣的是这个 PR 实际是 README 文本，`detect_language` 默认判为 python，但 Worker 智能识别出"这不是 Python 代码"并给了文档格式建议——三层容错兜住了这个边界情况。

### Token 成本计量

用 TokenMeter 包住 LLM client，拦截每次 completion 的 usage 累加，把"一次审查花多少 token"量化出来。实测：单次审查约 8,850 token（prompt ~1,430 + completion ~7,420），26 样本评测共约 23 万 token。一次审查 = decompose(1) + 4 Worker + judge(1) ≈ 5-6 次 LLM 调用。当前用免费模型 mimo-v2.5 成本为 ¥0，但计量框架已就位——换成收费模型，单价 × token 直接出成本。

优化成本有三条路：缓存（相同代码段不重复审）、分级路由（简单维度用便宜小模型）、控制上下文（只喂相关代码段）。这三条都依赖 token 计量先有数据——不计量就不知道一次审查的代价。

---

## 三次事故复盘

### 事故一：httpx 默认走系统代理导致 Connection error

**现象**：开发环境调用 LLM API 报 `Connection error`，但同网络其他机器正常。

**排查过程**：确认 LLM API 地址可 ping 通（网络没问题）→ 怀疑 httpx 默认行为 → 查文档发现 `httpx.AsyncClient()` 默认 `trust_env=True`，会读系统环境变量 `HTTP_PROXY` / `HTTPS_PROXY` → 开发机装了 Clash，系统代理指向 127.0.0.1:7890，但 Clash 没启动，代理转发失败。

**修复**：`httpx.AsyncClient(trust_env=False)` 禁用环境变量代理，直连 LLM API（`backend/core/llm.py:34`）。

**价值**：问题看似小，但体现"不迷信默认行为"的排查思路。httpx 和 requests 都默认走系统代理，在开发 / CI 环境经常踩坑。

### 事故二：conftest.py 假模块覆盖真实模块

**现象**：装了真实 fastmcp 后，测试里 import fastmcp 拿到的还是假的。

**根因**：conftest.py 的 `_install_fake_fastmcp()` 在模块加载时执行，先于真实 fastmcp 的 import，把假模块注册到 `sys.modules["fastmcp"]`。后续 import 读的就是 `sys.modules` 里已注册的假模块。

**修复**：先 try import 真实模块，失败再注册假模块。

```python
def _install_fake_fastmcp():
    if "fastmcp" in sys.modules:
        return  # 已经有了（不管真假），不重复装
    try:
        import fastmcp  # 先试真的
        return  # 真实 fastmcp 可用，不需要假的
    except ImportError:
        pass  # 没有真实的，继续装假的
    # ... 注册假模块到 sys.modules
```

**价值**：Python 的 import 机制是 `sys.modules` 缓存优先。测试框架的 fixture / mock 如果在模块级操作 `sys.modules`，容易和真实模块冲突——测试不应该修改全局 import 行为（`backend/tests/conftest.py:14-67`）。

### 事故三：MCP mount 双前缀 404 + lifespan 不触发

**现象**：FastAPI `app.mount("/mcp", mcp.http_app())` 后，请求变成 `/mcp/mcp/` 双前缀 404；mounted sub-app 的 lifespan 不自动触发，报 `RuntimeError: Task group is not initialized`。

**根因**：FastAPI mount 剥离 `/mcp` 前缀转发给子 app，子 app 内部又路由 `/mcp`，于是双前缀；而 Starlette 的 mount 只转发 HTTP 请求，不转发 ASGI lifespan 事件。

**修复**：用 `http_app(path="/")` 让 MCP 内部路由挂在根；在 FastAPI 的 lifespan 里手动 `async with _mcp_app.router.lifespan_context(_mcp_app)` 触发 sub-app 的 lifespan。

**价值**：这两个坑让我真正理解了 **ASGI mount 只转发请求，不转发生命周期事件**——这是很多人挂载子应用时会栽的地方。

---

## 三个对抗性技术疑问

### "多 Agent 的开销值得吗？单 Agent 加长 prompt 不行吗？"

先承认开销（4 次 LLM 调用，token 和延迟都更高），再论证值得：

1. **质量碾压**：单 Agent 四维 prompt 稀释注意力，会"偏科"；多 Agent 各自专注，completeness 明显更高
2. **并行抵消延迟**：fan-out 并行使 4 Worker 总耗时接近单次调用，端到端 ~50-90s，而不是单次的 4 倍
3. **容错不对称**：LLM API 可用性不是 100%，多 Agent 的部分失败容忍度远高于单 Agent
4. **可扩展 + 可归因**：加维度只加 Worker；分类评测能精确定位是哪个维度不行

数据支撑：LLM-as-Judge composite 0.8628，各维度分差明显（0.76~0.94），正是多 Agent 分工的证据。

### "LLM-as-Judge 的 Precision 只有 0.086，评测可信吗？"

先解释数字，再论证可信：

1. **P=0.086 的原因**：Precision 按"实际报告与 expected 精确匹配的关键词占比"计算。LLM 用自然语言描述（"此处存在 SQL 注入风险"），expected 用精炼关键词（"SQL injection"），精确匹配率自然低——这是评估粒度不匹配，不是审查质量差
2. **R=0.808 说明覆盖广**：expected 中 80% 的问题都被覆盖，只是描述方式不同
3. **composite 规避了 PRF 局限**：LLM-as-Judge 不靠关键词匹配，而是让 LLM 判断"是否覆盖期望发现"和"是否正确"，composite 0.8628 高于 rule_based 0.792
4. **双模式互验**：两种模式对同一样本集评估，趋势一致（security > quality > structure > performance），说明评测本身稳定
5. **承认不足**：PRF 确实不适合评估自然语言报告，后续可改用语义相似度（embedding cosine）替代精确匹配

### "和 SonarQube / CodeRabbit 比有什么优势？"

先承认生态，再定位差异化：

1. **不是替代 SonarQube**：SonarQube 做规则扫描（静态分析），cr-agent 做语义理解（LLM 审查）。SonarQube 能检测 `eval()` 调用，但理解不了"这个 eval 是在安全沙箱里用的"，LLM 可以
2. **不是替代 CodeRabbit**：CodeRabbit 是 SaaS，cr-agent 是自部署的 MCP 服务。数据合规要求代码不出域时，cr-agent 可部署内网、用私有 LLM
3. **核心差异——多 Agent 分工**：现有工具大多单模型审查，cr-agent 每个维度有专门优化（全科医生 vs 专科会诊）
4. **MCP 可组合**：暴露为 MCP 服务，可被任何 MCP 兼容工作流调用，不是孤岛
5. **评测闭环**：rule_based + LLM-as-Judge 双模式评测、分类归因、量化迭代——工程化思维而非 demo 思维
6. **诚实承认**：评测样本只有 26 条、performance accuracy 0.66 需优化、没有人工评测校准。它是"概念验证 + 工程化框架"，不是成品

---

## 量化数据

| 指标 | 数值 |
|------|------|
| **架构** | 6 节点 / 1 条条件边 / 9 个 State 字段 |
| **Worker 数** | 4 个（security / quality / performance / structure） |
| **rule_based composite** | 0.792 |
| **LLM-as-Judge composite** | 0.8628 |
| **评测样本** | 26 条 |
| **Token 用量** | 270,625 tokens / 125 calls |
| **security / quality / structure / performance** | 0.94 / 0.89 / 0.84 / 0.76 |
| **PRF（LLM 模式）** | P=0.086 / R=0.808 / F1=0.154 |
| **composite 公式** | 0.4×completeness + 0.4×accuracy + 0.2×source_traceability |
| **熔断阈值 / Worker 超时** | max_iterations=3 / 120s |
| **MCP Tool / Resource** | 5 个 / 2 个 |
| **测试数** | 64 个（53 核心 + 11 MCP） |
| **代码规模** | 后端 ~3500 行 Python，前端 ~2500 行单文件 HTML/CSS/JS |

---

## 代码地图

| 想看 | 位置 |
|-------|------|
| LangGraph 图怎么装配的 | `backend/services/supervisor/graph.py:123-137`（`build_supervisor_graph`） |
| 熔断逻辑 | `backend/services/supervisor/graph.py:65-79`（`_route_after_decompose`） |
| Worker 超时 | `backend/services/workers/base.py:79-81`（`asyncio.wait_for`） |
| Worker 异常降级 | `backend/services/workers/base.py:93-102`（try/except → info finding） |
| reducer 机制 | `backend/services/supervisor/state.py:17-30`（`Annotated[list, operator.add]`） |
| LLM-as-Judge | `backend/services/evaluation/judge.py:105-127`（`judge_with_llm`） |
| rule_based 基线 | `backend/services/evaluation/judge.py:83-102`（`judge_rule_based`） |
| composite 公式 | `backend/services/evaluation/judge.py:39-40`（`_composite`） |
| httpx 代理修复 | `backend/core/llm.py:34`（`trust_env=False`） |
| 假 fastmcp 逻辑 | `backend/tests/conftest.py:14-67`（`_install_fake_fastmcp`） |
| PR 审查 API | `backend/api/reviews.py:206-228`（`create_review_from_pr`） |
| SSE 流式 | `backend/api/reviews.py:64-119`（`_run_review_stream`） |
| MCP Server | `backend/mcp/server.py`（5 Tool + 2 Resource） |
| Prompt 注入防护 | `backend/services/workers/base.py:109-115`（定界符包裹） |
| decompose 降级 | `backend/services/supervisor/decompose.py:83-103`（except → `_default_tasks`） |
| 前端页面 | `backend/static/index.html`（单文件 ~2500 行） |

---

## 与 ai-resume 的联动

两个项目不是重复造轮子，而是从"用 AI 做问答"到"用 AI 做专业审查"的能力扩展。可以从三个层次理解它们的关系。

**技术栈演进**：ai-resume 是单 Agent 架构（9 节点 DAG + Reflexion 自纠正循环），cr-agent 是多 Agent 架构（Supervisor-Worker + fan-out 并行）。从单 Agent 到多 Agent，核心区别是**并发写和容错**——ai-resume 不需要 reducer（单 Agent 串行写），cr-agent 必须用 `Annotated[list, operator.add]` 解决多 Worker 并发写覆盖问题。

**评测体系对齐**：两个项目的 LLM-as-Judge 框架是刻意对齐的——都是三维度评分（completeness / accuracy / source_traceability），权重一致（40% / 40% / 20%）。这样同一套评测框架可以横向对比两个项目的 LLM 输出质量，本质上是一套可复用的 Agent 评测方法论。

**能力互补**：ai-resume 侧重 RAG 与工程化（混合检索、RRF 融合、Reflexion、MCP + JWT、Prometheus 可观测性）；cr-agent 侧重多 Agent 编排与系统设计（Supervisor-Worker、三层容错、fan-out + reducer、双模式评测、GitHub PR 接入）。

几个常见延伸问题：

| 问题 | 回答 |
|------|------|
| 两个项目的 MCP 有什么区别？ | ai-resume：5 Tool + JWT 认证 + HTTP 传输；cr-agent：5 Tool + 2 Resource + in-memory + HTTP 双传输。cr-agent 的 Resource 概念（review://history / review://stats）是 ai-resume 没有的。 |
| 为什么不把两个项目合一起？ | 领域不同（简历问答 vs 代码审查），强行合并会模糊核心叙事。分开更清晰：ai-resume = RAG 能力，cr-agent = Multi-Agent 能力。 |
| ai-resume 的 Reflexion 能用到 cr-agent 吗？ | 理论上可以（Worker 审查后加 Reflexion 节点自纠正）。但 cr-agent 的评测显示主要瓶颈是 performance 的 accuracy（0.66），不是 completeness；Reflexion 提升的是覆盖度，解决不了准确度问题。 |
| 两个项目哪个更难？ | ai-resume 难在 RAG 链路长（检索 + 融合 + 重排 + 生成 + 反思），任何一环拉胯都影响最终效果；cr-agent 难在系统设计（并发写、容错、评测归因）。两者是不同类型的复杂度——工程链路复杂度 vs 系统设计复杂度。 |

---

## 相关笔记

- [[01-技术研读/00-17条架构决策总览|00 17条架构决策总览]]
- [[01-技术研读/12-项目规格与TechStack|12 项目规格与TechStack]]
- [[02-代码审查报告展示样例|02 代码审查报告展示样例]]

### 技术学习笔记

- [[15-Agent架构与工具调用|Agent架构与工具调用]]
- [[37-Multi-Agent协作模式|Multi-Agent协作模式]]
- [[39-Agent评测体系：benchmark-case-指标设计|Agent评测体系]]
- [[26-MCP协议核心概念|MCP协议基础]]
- Agent韧性工程概述

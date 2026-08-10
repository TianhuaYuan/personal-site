---
title: "LLM 评测体系搭建"
tags:
  - 项目笔记
  - cr-agent
created: "2026-07-21"
---

# LLM 评测体系搭建

> rule-based 基线秒出结果、LLM-as-Judge 三维度语义评分、PRF 硬指标暴露出 precision 0.08 的残酷现实，以及 TokenMeter 的从零到一。

## 一、背景（为什么要做这个）

审查质量不能只靠"看起来不错"。一个必然会想到的问题："你怎么知道你的 Agent 确实变好了？"——没有评测体系，任何优化都只是拍脑袋。目标是建立双模式评测：rule_based 做快速基线（秒级），LLM-as-Judge 做深度评估（真实 LLM 评判），外加 token 计量做成本感知。

## 二、逐个讲：问题 → 怎么想 → 怎么解

### 1. 评测数据从哪来——从 mock 到真实

**问题**：评测面板需要数据，但真实评测要跑 26 条样本 × 多次 LLM 调用，费钱又费时。我在 Phase 3 时用了 mock 数据（确定性伪随机），但开发到 Phase 8 时我必须上真的。

**怎么想**：分两步走。我 Phase 3 时先用 mock 数据把 UI 和 API 跑通（接口契约先定），到 Phase 8 时再替换为真实评测逻辑（数据源换但格式不变）。

**怎么解**：Phase 3 时我写 `_build_mock_results()` 基于 sample id 哈希生成确定性伪随机结果。到 Phase 8 时我替换为 `_run_evaluation(mode)`——mode=rule_based 用规则秒出、mode=llm 用 LLM-as-Judge 评分。前端格式（total/composite_avg/prf_avg/by_category/per_sample）完全不变，零迁移成本。

### 2. rule-based 基线设计

**问题**：什么是最低可接受的评审标准？如果 LLM 连关键词都没覆盖，那肯定不行。

**怎么想**：rule_based 是"最低标准"——completeness（期望 findings 的关键词命中率）、accuracy（completeness 近似，规则无法判幻觉）、source_traceability（报告里有没有行号或代码块标记）。

**怎么解**：三条规则——completeness=期望 description 前 8 字在报告中出现的比例；accuracy=completeness（规则无法判幻觉，所以用 completeness 近似）；source_traceability=报告里有没有"行"或代码块标记。26 样本 composite_avg=0.792。糙但够用。

### 3. LLM-as-Judge 三维度

**问题**：rule_based 太糙——关键词命中不相等"准确"，行号标记不相等"可追溯"。需要真正理解语义的评判。

**怎么想**：LLM 当裁判，给定 code + expected_findings + actual_report，从三个维度打分 0-1：
- **completeness**：实际报告是否覆盖了期望发现的所有问题（漏报扣分）
- **accuracy**：实际报告是否准确描述了问题（误报、严重度错误、行号错误都扣分）
- **source_traceability**：每项发现是否追溯到具体的行号或代码

**怎么解**：3 个 LLM judge 调用 × 26 样本 = 78 次 LLM 调用，加上 6 个审查节点的 47 次，共 125 次调用，270k tokens。结果：composite_avg=0.8628，security 0.94 / quality 0.89 / performance 0.76 / structure 0.84。

### 4. PRF 硬指标——precision 0.08 的暴击

**问题**：评测一跑，precision=0.08——LLM 每报 10 个问题只有 0.8 个是真的，误报严重。recall=0.77 尚可，但 F1 被 precision 拖到 0.15。

**怎么想**：PRF 用的是关键词精确匹配——expected 的 description 和 actual 的 description 做字符串匹配。LLM 生成描述是自然语言（"SQL injection" vs "SQL注入风险"），精确匹配天然低。

**怎么解**：PRF 作为硬指标保留——它诚实地反映了"字符串匹配"场景下的表现。但这不是我真正关心的，三维度 LLM-as-Judge 才是主评测。PRF 的 0.15 是个 Feature 不是 Bug——它说明 LLM 的语义表达和精确描述之间有 gap，这正好引出为什么需要 LLM-as-Judge。

### 5. TokenMeter——从零到一的 token 计量

**问题**：评测跑了 125 次 LLM 调用、270k tokens，但我没有计量工具——不知道每次审查花了多少钱、哪个节点最贵、什么场景最费 token。

**怎么想**：不能改 LLM client 的 create 方法（范围太大），要包装一层——拦截 create 调用记录 usage，其余属性原样代理。

**怎么解**：`MeteredClient`——包装 AsyncOpenAI 风格 client，拦截 `.chat.completions.create` 记录 usage（prompt_tokens + completion_tokens），其余属性直接穿透到真实 client。我在 CLI 加了 `--tokens` 开关（默认关，日常评测不关心 token），开箱即计量。

## 三、关键决策与取舍

| 决策 | 选择 | 取舍 |
|------|------|------|
| 评测模式 | rule_based + LLM-as-Judge 双模式 | 开发成本更高，但有基线和深度两套工具可用 |
| 三维度评分 | completeness / accuracy / source_traceability | 比单一分数更细粒度，但 3 次 LLM judge 调用 |
| PRF 精度 | 关键词精确匹配 | 数据诚实地低（0.15），但暴露了自然语言 gap |
| mock 策略 | 确定性伪随机（基于 id 哈希） | 不是真实数据，但接口契约 100% 可验证 |
| TokenMeter 注入 | MeteredClient 包装 + `--tokens` 开关 | 比改 LLM client 代码更安全、更可测试 |

## 四、踩坑（值得讲的故事）

**1. httpx 代理劫持——trust_env=False**

现象 → 根因 → 解法 → 教训

26 条样本 LLM-as-Judge 跑下来全部 "Connection error"，composite_avg=0.0。但 PowerShell 调 API 是正常的。追踪堆栈看到 `httpcore._async.http_proxy.py`——httpx 默认读 HTTP_PROXY 走本地代理（Clash/V2Ray），代理没运行所以全部失败。解法：`AsyncOpenAI(http_client=httpx.AsyncClient(trust_env=False))`。教训：**用 httpx/openai 调外部 API，永远显式 trust_env=False**——开发机代理不是给 LLM API 用的。

**2. rule_based 的 accuracy 用 completeness 近似的取舍**

现象 → 根因 → 解法 → 教训

rule_based 的 accuracy 直接等于 completeness——关键词命中高 ≈ 准确率高。但幻觉不命中关键词但看起来合理，这种情况 rule_based 无法检测。解法：接受这个局限，在文档里写明"rule_based 的 accuracy 不代表真实准确率，请参考 LLM-as-Judge"。教训：**确定性的规则评测有明确的上限**——语义理解必须靠 LLM。

**3. TokenMeter 的"函数引用污染"**

现象 → 根因 → 解法 → 教训

修复 CLI 漏传 meter 的时候，用 `monkeypatch.setattr` 替换了 `llm_mod.get_chat_client` 函数引用。但 restore 时只保存了原函数的 `id`，两次跑之间重用同一个 meter 实例，计量数据交叉污染。解法：存函数引用不存 id，`monkeypatch.undo()` 后真正还原。教训：**monkeypatch 的还原必须是"用原函数引用覆盖"，不是"删掉我改过的东西"**——后者会留下其他测试的 mock。

## 五、常见疑问

**Q1：为什么 rule_based 和 LLM-as-Judge 分数差这么多（0.792 vs 0.8628）？**

A：两个原因。第一，rule_based 的 completeness 用"期望 description 前 8 字在报告中出现"判断——LLM 用自然语言，前 8 字不一定匹配，导致被低估。第二，rule_based 的 accuracy = completeness，无法区分"对但表述不同"和"错但碰巧匹配"。LLM-as-Judge 能理解语义相似性，所以更准确。

**Q2：为什么 performance 维度分数最低（0.76）？**

A：accuracy 只有 0.66，集中在：严重度误判（如 low 判为中危）、行号不匹配（行 3 vs 行 4）。说明 Worker 在 performance 维度的严重度校准和行号定位需要优化。改进方向：prompt 里加严重度判断规则 + few-shot 示例。

**Q3：PRF 的 F1 为什么只有 0.15？**

A：PRF 用的是关键词精确匹配——expected 的 "SQL injection" 和 actual 的 "SQL注入风险" 不匹配，precision 低到 0.086。但 recall 0.81 说明大部分问题被发现了。这正好是为什么要做 LLM-as-Judge——它能理解语义相似性。

**Q4：125 次 LLM 调用是怎么算的？**

---

## 相关笔记

- [[01-技术研读/00-17条架构决策总览|00 17条架构决策总览]]
- [[01-技术研读/09-LLM评测体系搭建|09 LLM评测体系搭建]]
- [[11_安全加固四道防线|11 安全加固四道防线]]

## 技术学习笔记

- [[39-Agent评测体系：benchmark-case-指标设计|Agent评测体系]]
- [[10-RAG 评估：检索指标 + 生成指标|RAG评估]]
- [[01-Token计费原理-Temperature控制-SystemPrompt层级|Token计费原理]]
- [[工程化与运维/可观测性/01-Prometheus四层指标|Prometheus四层指标]]
- [[工程化与运维/可观测性/02-Grafana-Dashboard预置面板|Grafana Dashboard]]

A：26 条样本 ×（1 decompose + 4 Worker + 1 aggregate + 1 judge）= 182 次。实际 125 次是因为部分调用复用了缓存或降级。Token 270k，平均每次 ~2160 tokens。成本大约 ¥2-3。

---
title: "多Agent代码审查系统设计"
created: "2026-07-21"
tags:
  - 八股文
  - ai
  - 系统设计
  - 多agent
---

# 多 Agent 代码审查系统设计

> 典型**多 Agent 协作**落地：用 Supervisor-Worker 架构把一次 Code Review 拆给多个专职 Agent 并行审，再聚合报告。协作模式总览见 [[AI与Agent/知识/八股/29-多Agent协作基础与框架选型|多Agent协作Multi-Agent]]、[[AI与Agent/知识/八股/23-Agent架构与核心组件|Agent架构与ReAct]]。

## 为什么用多 Agent 而非单 Agent
单个 Agent 一次审完，容易顾此失彼、上下文爆炸、视角单一。拆成多个专职 Worker（安全/性能/规范/逻辑），**并行 + 专业分工**，质量与速度都更好。但引入了协调、聚合、一致性成本。

## Supervisor-Worker 架构

```mermaid
flowchart TB
    PR[Pull Request 变更] --> Sup[Supervisor 调度]
    Sup --> W1[Worker-安全<br/>漏洞/注入]
    Sup --> W2[Worker-性能<br/>复杂度/NDB]
    Sup --> W3[Worker-规范<br/>风格/约定]
    Sup --> W4[Worker-逻辑<br/>边界/空指针]
    W1 --> Agg[Aggregator 聚合]
    W2 --> Agg
    W3 --> Agg
    W4 --> Agg
    Agg --> Dedup[去重/去误报/定级]
    Dedup --> Rep[审查报告 + 行内评论]
```

## 模块设计
- **Supervisor**：切分变更文件、按文件/维度派发任务、控制并发与超时、收集结果。
- **Worker（专职）**：每个 Worker 只负责一个维度，System Prompt 聚焦，工具各取所需（AST 解析、grep、单测运行）。
- **Aggregator**：合并多个 Worker 的结论，**去重、过滤误报、按严重度定级**（阻塞/建议/提示）。
- **反馈闭环**：被开发者标记为"误报"的结论回流，用于优化 Worker Prompt/规则。

## 工程要点
- **并行限界**：Worker 数受 LLM 并发与成本限制，按变更规模动态决定并行度。
- **上下文隔离**：每个 Worker 只看自己负责的文件片段，避免上下文溢出。
- **确定性**：同一 PR 多次审阅结果应稳定，需固定模型与去随机。
- 多 Agent 的通信与一致性问题见 [[AI与Agent/知识/八股/29-多Agent协作基础与框架选型|多Agent协作Multi-Agent]]、[[AI与Agent/知识/八股/43-多Agent编排四种模式|多Agent编排模式]]、[[45-多Agent幻觉放大问题与解决方案|多Agent幻觉放大]]。

## 关键权衡

| 问题 | 设计回答要点 |
| ------ | ------ |
| 多 Agent 比单 Agent 好在哪？ | 专业分工 + 并行，视角更全、质量更高；代价是协调/聚合/一致性成本 |
| 怎么处理多个 Worker 结论冲突？ | Aggregator 去重+定级，按严重度与证据强度裁决；冲突项升级人工 |
| Worker 太多会不会很贵？ | 按变更规模动态并行度、上下文隔离控 token、误报回流降后续成本 |
| 如何防止误报淹没开发者？ | 严格定级（阻塞/建议/提示）+ 去重 + 误报反馈闭环，只把高置信结论评论到 PR |

---


## 速记卡（面试闪卡）

**Q1：一句话讲清「多 Agent 代码审查系统设计」到底是什么？**
A：用 Supervisor-Worker 架构把 Code Review 拆给多个专职 Agent 并行审，再聚合去重、按严重度定级生成报告。

**Q2：为什么多 Agent 而非单 Agent —— 怎么理解？ —— 怎么理解？**
A：单个 Agent 一次审完全部，容易顾此失彼、上下文爆炸、视角单一。拆成多个专职审查员并行，质量与速度都更好。生活类比：一个人又写代码又审又测像身兼数职忙中出错；多个专职审查员（安全/性能/规范/逻辑）各看一摊，视角全、漏得少。代价是引入了协调、聚合、一致性成本——不是免费午餐。英文术语：single-agent、multi-agent、specialization（专业化分工）。

**Q3：Supervisor-Worker 架构 —— 怎么理解？ —— 怎么理解？**
A：Supervisor 切分变更文件、按文件/维度派发任务、控并发与超时、收结果；多个 Worker 各负责一个维度，System Prompt 聚焦、工具各取所需（AST 解析、grep、单测）。生活类比：主管像工地包工头，把活拆给四个工种（水电/泥瓦/木工/油漆），每人只精一行，并行干活。英文术语：Supervisor（调度者）、Worker（工作者）、fan-out（扇出）。

**Q4：模块设计：Aggregator 聚合去重定级 —— 怎么理解？ —— 怎么理解？**
A：Aggregator 合并多个 Worker 的结论，去重、过滤误报、按严重度定级（阻塞/建议/提示），只把高置信结论评论到 PR。生活类比：Aggregator 像主编，把四份审稿合成一篇，删掉重复的、撤掉不靠谱的、按重要性排版面。被标"误报"的结论回流，用于优化 Worker 的 Prompt/规则，形成反馈闭环。英文术语：Aggregator（聚合器）、false positive（误报）、feedback loop（反馈闭环）。

**Q5：工程要点与关键权衡 —— 怎么理解？ —— 怎么理解？**
A：并行度受 LLM 并发与成本限制，按变更规模动态决定（不是越多越好）；上下文隔离让每个 Worker 只看自己负责的片段，避免上下文溢出；确定性要求同一 PR 多次审阅结果稳定，需固定模型、去随机。生活类比：像餐厅按客流排班（动态并行）、每个厨师只看自己工位（隔离）、同一道菜每次口味一致（确定性）。英文术语：parallelism bound（并行限界）、context isolation（上下文隔离）、determinism（确定性）。

**Q6：核心速记主线有哪些？**
- 架构：Supervisor 派发 → 多 Worker 并行审 → Aggregator 聚合
- 分工：安全/性能/规范/逻辑四个专职维度
- 聚合：去重、过滤误报、按严重度定级
- 工程：动态并行度、上下文隔离、确定性可复现

**口诀**
A：代码审查上多 Agent，分工并行质量强；
Supervisor 派任务，四工种各守一岗；
Aggregator 合报告，去重定级误报降；
动态并行控成本，上下文隔离不溢仓。

## 相关链接
- [[AI与Agent/知识/八股/29-多Agent协作基础与框架选型|多Agent协作Multi-Agent]]
- [[AI与Agent/知识/八股/23-Agent架构与核心组件|Agent架构与ReAct]]
- [[AI与Agent/知识/八股/43-多Agent编排四种模式|多Agent编排四种模式]]
- [[AI与Agent/知识/八股/45-多Agent幻觉放大问题与解决方案|多Agent幻觉放大问题]]
- [[八股文学习路线图]]
- [[00-全局导航|全局导航]]

---

## 快速问答

| 问题 | 参考答案 |
| ------ | --------- |
| 多 Agent 代码审查的架构？ | Supervisor 派发 → 多个专职 Worker 并行审（安全/性能/规范/逻辑）→ Aggregator 去重定级 → 报告。 |
| 为什么不直接让一个 Agent 审完？ | 单 Agent 上下文易溢出、视角单一、易遗漏；多 Worker 专业分工+并行，质量与速度更优。 |
| 多个 Worker 结论冲突怎么办？ | Aggregator 按严重度与证据强度裁决，冲突项升级人工；最终只评论高置信结论。 |
| 如何控制成本与误报？ | 动态并行度 + 上下文隔离控 token；去重定级 + 误报反馈闭环降噪声。 |

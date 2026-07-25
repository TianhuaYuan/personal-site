---
title: "ReAct框架与规划能力"
created: "2026-07-13"
tags:
  - 八股文
  - ai
---

# ReAct 框架与规划能力

> ReAct（Reasoning + Acting）是目前最主流的 Agent 推理模式，由 Yao et al. (2022) 提出。核心思想：将**推理（Thought）**和**行动（Action）**交替进行，让 Agent 像人一样"边想边做"。

## ReAct 核心思想

将**推理（Thought）**和**行动（Action）**交替进行：

```mermaid
graph LR
    T1["Thought: 分析当前情况，决定下一步"] --> A1["Action: 选择并执行一个工具"]
    A1 --> O1["Observation: 获取工具返回的结果"]
    O1 --> T2["Thought: 分析结果，判断是否需要继续"]
    T2 -->|继续循环| A1
    T2 -->|完成| DONE[完成任务]
```

## ReAct 示例

```text
问题：北京和上海哪个城市人口更多？

Thought 1: 我需要查找两个城市的人口数据
Action 1: search("北京 2024 人口")
Observation 1: 北京常住人口约2185万

Thought 2: 获得了北京的数据，现在查上海
Action 2: search("上海 2024 人口")
Observation 2: 上海常住人口约2487万

Thought 3: 两个数据都有了，可以比较了
Answer: 上海人口（约2487万）多于北京（约2185万）
```

## ReAct vs 其他模式

| 模式 | 说明 | 优缺点 |
| ------ | ------ | -------- |
| ReAct | 推理+行动交替 | 灵活，但可能循环 |
| CoT only | 只推理不行动 | 无法获取外部信息 |
| Act only (Tool use) | 只行动不推理 | 缺乏规划能力 |
| Plan & Execute | 先规划再执行 | 适合复杂任务，但不够灵活 |

## Planning 规划能力

### 简单任务：ReAct 逐步规划

对于简单任务，每一步动态决策即可。

### 复杂任务：分层规划

```mermaid
graph TD
    Plan["Plan: 将大任务分解为子任务"] --> ST1[Sub-task 1]
    Plan --> ST2[Sub-task 2]
    Plan --> ST3[Sub-task 3]
    ST1 --> E1[Execute] --> R1[Result 1]
    ST2 --> E2[Execute] --> R2[Result 2]
    ST3 --> E3[Execute] --> R3[Result 3]
    R1 --> Synth[Synthesize: 综合子任务结果]
    R2 --> Synth
    R3 --> Synth
```

### 规划策略

| 策略 | 说明 | 适用场景 |
| ------ | ------ | --------- |
| 递归分解 | 将任务不断拆分为更小的子任务 | 复杂多步任务 |
| 任务图 | 构建有向无环图 (DAG)，明确依赖关系 | 并行可执行的任务 |
| 动态重规划 | 根据执行结果调整后续计划 | 不确定性高的任务 |

---

## 
> ▶ 对应实操：[[24-Human-in-the-loop：人工介入兜底与敏感操作审批机制|24-Human-in-the-loop：人工介入兜底与敏感操作审批机制]]


> ▶ 对应实操：[[15-Agent架构与工具调用|15-Agent架构与工具调用]]

相关链接
- [[八股文学习清单]]
- [[00-全局导航|全局导航]]

---

## 快速问答

| 问题 | 参考答案 |
| ------ | --------- |
| ReAct 模式的核心思想？ | 将推理（Thought）和行动（Action）交替进行。每步先思考再执行，根据执行结果继续思考，直到完成任务 |
| 如何解决 Agent 的循环问题？ | ① 设置最大步数限制 ② 检测重复动作 ③ 使用 LangGraph 的状态机控制 ④ 加入终止条件判断 |
| 多 Agent 协作的挑战是什么？ | ① 通信开销 ② 角色分工 ③ 冲突解决 ④ 全局一致性 ⑤ 成本控制 |

---

title: "AI Agent 方法论与产品思维路线图

"

tags: [学习路线图]

created: "2026-08-12"

---



# AI Agent 方法论与产品思维路线图

## 优先级图例



| 标记 | 含义 |
| :-: | --- |
| 🔴 重点 | 核心心智模型，必须内化 |
| 🟡 进阶 | 实战技巧，能落地 |
| 🟢 了解 | 概念认知 |



## Top 20 核心方法



| 排名 | 方法 / 认知 | 模块 |
| :-: | --- | --- |
| 1 | 四层模型演进：Prompt→Context→Harness→Loop | 认知 |
| 2 | Prompt 2.0 四大黄金法则 | Prompt |
| 3 | 给目标不给步骤 | Prompt |
| 4 | 设计反馈闭环（execute→evaluate→adjust） | Prompt |
| 5 | Lost in the Middle 与首末重注入 | Context |
| 6 | Attention Budget 与指令天花板（~150 条） | Context |
| 7 | 渐进式加载 + 上下文压缩三级策略 | Context |
| 8 | Recency Effect 利用 | Context |
| 9 | 记忆污染识别与纠错 | Context |
| 10 | Skill 九要素结构 + Gotchas 优先 | Skill |
| 11 | CLAUDE.md / AGENTS.md 分工 | Skill |
| 12 | MCP / A2A / ACP 协议区别 | 协同 |
| 13 | 终止条件必须机器可检查 | Loop |
| 14 | 预算护栏必设（Token/步数） | Loop |
| 15 | 断路器 / 看门狗熔断 | Loop |
| 16 | Workflow vs Agent 选型判断 | 产品化 |
| 17 | 四步翻译（需求→方案） | 产品化 |
| 18 | 降级路径设计 | 产品化 |
| 19 | 三维评估 + LLM-as-Judge | 产品化 |
| 20 | Human-in-the-Loop 时机判断 | 产品化 |



---



### 四层模型演进全貌

- 2024 Prompt Engineering 时代：把话说清楚 🔴 重点

- 2025 Context Engineering 兴起：管好上下文比写好 prompt 更关键 🔴 重点

- 2026 初 Harness Engineering：把 Agent 跑起来的工程脚手架 🔴 重点

- 2026 中 Loop Engineering 爆发：长期自主循环的安全护栏 🔴 重点



### 核心认知

- 四层模型的嵌套关系：上层依赖下层，权重随阶段变化 🔴 重点

- Prompt 在整体中的权重变化：从 100% 降到工程化主导 🟡 进阶

- 一句话说清四层区别、画出演进图 🟡 进阶

- 设计 Agent 的思考框架：先定边界再定能力 🔴 重点



---



### 四大黄金法则

- 给目标不给步骤：让模型自己规划路径 🔴 重点

- 评估标准写进指令：可判分才有闭环 🔴 重点

- 分离规划与执行：Planner 与 Executor 解耦 🔴 重点

- 设计反馈闭环：execute → evaluate → adjust 🔴 重点



### 结构化与约束

- 结构化 Prompt 六段式：角色/目标/约束/步骤/输出/示例 🔴 重点

- 约束数量控制：指令不是越多越好（~150 条天花板）🟡 进阶

- 反例约束与禁止模糊词：用「不要做 X」补边界 🟡 进阶



### Maker-Checker 模式

- Maker-Checker 概念：生成与校验分离 🟡 进阶

- 落地方式、Checker 成本考量、Maker vs Checker 自检 🟡 进阶



### 框架选型判断力

- ReAct / Plan-and-Execute / Reflexion 的适用边界 🟡 进阶

- 选型判断标准：任务确定性、是否需要反思 🟡 进阶



---



### 注意力陷阱

- Lost in the Middle：长 Prompt 中间指令遵循率仅 ~60% 🔴 重点

- Attention Budget：指令也占注意力额度 🔴 重点

- 150 条指令天花板：超量后边际效用骤降 🔴 重点

- 文件大小与注意力衰减：1-100 行 ~95% 遵循，600+ 行 ~45% 🔴 重点



### 注入与加载策略

- 首末重注入：关键指令放开头与结尾 🔴 重点

- 渐进式加载：按需取片段而非全量 🔴 重点

- Recency Effect 利用：硬性 gate 放末尾 🔴 重点

- 路径作用域：只加载当前任务相关文件 🟡 进阶



### 压缩与纠错

- 摘要压缩：任务/文件/过程三级摘要 🔴 重点

- 长任务重注入：周期性回填关键状态 🟡 进阶

- 污染征兆识别：错误被上下文放大 🔴 重点

- 找到临界点、compact 总结、重注入格式设计 🟡 进阶



---



## 一、能力工程（Skill Engineering）



- 读外部 Skills、分析优秀 skill、对比工具/Skill 系统 🟡 进阶

- Skill 九要素结构：name/description/trigger/步骤/gotchas/示例… 🔴 重点

- Gotchas 优先：把踩坑写在第一位 🔴 重点

- 写第一个 SKILL、压缩到 500 行以内 🟡 进阶

- CLAUDE.md 与 AGENTS.md 分工：项目约定 vs 行为规范 🔴 重点

- 延迟加载机制、多文件架构、Skill 版本管理 🟡 进阶



---



## 二、多工具协同



- Claude Code / OpenCode / Codex 的特性差异 🟡 进阶

- 工具选型原则：稳定任务用确定性工具 🟴 重点

- MCP / A2A / ACP 协议区别与选型 🔴 重点

- 工具箱管理、混用 Agent 策略、工具切换成本 🟡 进阶



---



### 六大构件

- Automations（定时/事件触发）、Worktrees（隔离）、Skills、Connectors、Sub-agents、State（跨 session 持久化）🟡 进阶



### 安全护栏

- 终止条件必须机器可检查：避免无限循环 🔴 重点

- 预算护栏必设：Token / 步数上限 🔴 重点

- 15 个工具调用后注意力衰减：及时压缩或交接 🟴 重点

- Loop 高风险警告、爆炸半径评估 🔴 重点



### 容错机制

- 断路器（Circuit Breaker）：连续失败快速拒绝 🔴 重点

- 存活检查、Closed Loop 脚本、看门狗熔断 🟡 进阶



---



## 四、Agent 产品化思维



- Workflow vs Agent 选型：路径固定→Workflow；不确定→Agent；混合更优 🔴 重点

- 简单任务最简方案：别为简单事上 Agent 🔴 重点

- 幻觉递归陷阱：错误自我强化，需外部校验打断 🔴 重点

- 四步翻译：把模糊需求翻译成可执行的 Agent 方案 🔴 重点

- 失败模式预判：提前设计降级而非事后救火 🔴 重点

- 降级路径设计：核心不可用时回退次优 🔴 重点

- 三维评估框架 + LLM-as-Judge：效果可量化 🔴 重点

- A-B 测试对照：版本间客观比较 🟡 进阶

- Human-in-the-Loop 时机判断：敏感/不可逆操作必介入 🔴 重点

- 权限分级、决策可追溯、错误恢复路径 🟴 重点

- 四维乘积模型：把产品化拆成可权衡的维度 🟡 进阶



---



## 学习顺序



1. 认知建立：第一章四层模型。

2. 指令工程：第二章 Prompt 2.0。

3. 上下文工程：第三章 Context Engineering。

4. 能力沉淀：第四章 Skill Engineering + 第五章多工具协同。

5. 长循环：第六章 Loop Engineering。

6. 产品化：第七章产品化思维（降级/HITL/评估）。

## 相关链接



- [[学习路线图/技术学习路线图|技术学习路线图（AI Agent 开发）]]

- [[学习路线图/八股文学习路线图|八股文学习路线图]]

- [[学习路线图/LeetCode学习路线图|LeetCode 学习路线图（186 题）]]


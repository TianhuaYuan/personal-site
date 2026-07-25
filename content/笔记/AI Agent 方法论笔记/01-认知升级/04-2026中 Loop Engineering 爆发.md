---
title: "2026中 Loop Engineering 爆发"
tags:
  - agent方法论
  - 认知升级
  - 四层模型
  - loop
created: "2026-07-21"
---

# 2026 中 Loop Engineering 爆发：从手动调 Prompt 到设计自主循环

> 本条目是「认知升级：四层模型」的第 4 部分，对应学习清单条目 1.1.4。
>
> **前置依赖**：Prompt Engineering、Context Engineering、Harness Engineering
> **为以下铺垫**：（系列顶层，无后续依赖）

---

## 一、核心观点

> **Loop Engineering 研究「让一套系统去 prompt Agent，而不是你逐次手动 prompt」——你不再是聊天框里那个人，而是造聊天框机器的人。**

---

## 二、定义与出处（可验证）

### 2.1 时间线（2026 年 6 月「 Loop 周」）

| 时间 | 人物 | 事件 |
|------|------|------|
| 2026-06-02 | **Boris Cherny**（Claude Code 负责人） | WorkOS Acquired Unplugged 活动上说：「I don't prompt Claude anymore. I have loops running that prompt Claude and figuring out what to do. My job is to write loops.」 |
| 2026-06-07 | **Peter Steinberger**（OpenClaw 创始人） | 推文 24 小时内约 **220 万**浏览，数日破 **650 万**：「You should be designing loops that prompt your agents.」 |
| 2026-06-08 | **Addy Osmani**（Google Cloud AI） | 发文《Loop Engineering》正式命名，拆出六大构件 |

### 2.2 前置思想渊源

- **Anthropic《Building Effective Agents》（2024-12）** 已定义 Agent 本质：「LLMs using tools based on environmental feedback in a loop.」
- Simon Willison 反复强调：「Agents are models using tools in a loop.」
- 2025-08 Braintrust 文章标题即《The canonical agent architecture: A while loop with tools.》

> **关键判断**：「Loop 本身不是新东西，新的是——loop 成了工程的首要对象，是你设计、调优、拥有、测试的东西，而不只是聊天框背后的实现细节。」

---

## 三、核心机制

### 3.1 自校正循环

```text
定义目标 → 生成 → 执行 → 验证 → 读结果 → 决定下一步 → 循环
```

### 3.2 四个独立的退出机制（绝不「问 LLM 该不该停」）

1. **确定性验证器（verifier）**：用代码判（测试/lint/typecheck），非模型自评
2. **步骤上限（MAX_STEPS）**：硬迭代次数上限
3. **无进展检测（no-progress）**：最近若干步产出相同错误/状态无变化
4. **成本/时间上限（budget）**：token 或美元或墙钟时间到顶

> 金句：「只要你的终止条件是『问 LLM 该不该停』，loop 就回到了 vibes（凭感觉）。」

### 3.3 六大构件（Addy Osmani）

| 构件 | 是什么 | 大白话 |
|------|--------|--------|
| **Automations** | 定时/事件触发的任务发现 | 人没醒，系统已扫完 CI 失败、open issues |
| **Worktrees** | 隔离工作目录 | 多 Agent 并行不撞同一文件 |
| **Skills** | 写死的可复用知识 | Agent 不用每次重推「这项目怎么测」 |
| **Connectors** | 经 MCP 接外部系统 | 让 Agent 真能读营收、改 DNS |
| **Sub-agents** | 写的人不审自己 | Maker 写、Checker 独立验，消除自评偏差 |
| **State** | 跨轮持久记忆 | 模型两轮之间会忘，状态落盘才存活 |

---

## 四、Maker-Checker：Loop 的核心技术重点

- **铁律**：生成器（Maker）**绝不能**给自己的产物打分。Anthropic 工程团队发现，「调教一个持怀疑态度的独立评估器，远比让生成器批评自己更可行」。
- **实证**：Claude Fable 5（2026-06-09 发布）中，独立上下文窗口的 verifier sub-agents 持续优于 self-critique；rubric 驱动的 loop 让某训练流水线较上一代提升约 **6 倍**。
- 自检会形成「点头循环」（self-congratulation loop），长期累积隐性故障。

---

## 五、最新研究与企业数据（2024–2026）

- **Anthropic《When AI builds itself》（2026-06）真实生产力数据**：
  - Claude 写的代码占合并总量 **>80%**（2026-05）
  - 工程师日均代码合并量较 2024 年 **增长 8 倍**
  - 开放式任务成功率 **76%**（2026-05），半年前仅 **26%**
  - 训练代码自主优化加速：从 **3x**（2025-05）到 **52x**（2026-04）
- **Boris Cherny 个人工作流**：最近 30 天，Claude Code 仓库 **100%** 贡献（259 个 PR）由 Claude Code 自己完成；他 2025-11 删除 IDE 后再未打开。

---

## 六、优势与局限

- ✅ 人从「操作者」变「设计者」，杠杆点转移；可并行、可无人值守
- ❌ 四大隐性风险：验证债务、理解衰退、认知投降、token 井喷；成本可差数量级
- ❌ 仍早：Verification 比以往更依赖工程师本人；必须保留人工审核点

---

## 七、学习资源

- **原始信号**
  - Boris Cherny @ WorkOS Acquired Unplugged (2026-06-02)
  - Peter Steinberger 推文 (2026-06-07)；Addy Osmani《Loop Engineering》(2026-06-08)
  - Anthropic《Building Effective Agents》(2024-12)；《When AI builds itself》(2026-06)
- **深度综述**
  - agentconn.com《Loop Engineering: The Week the Industry Stopped Prompting》
  - 腾讯云开发者《Loop Engineering：当你不再是那个敲 Prompt 的人》(2026-06)
- **进阶阅读**
  - 本系列第 6 篇：Loop Engineering 方法论详解

---

**下一篇**：[[../02-Prompt 2.0/01-给目标不给步骤|Prompt 2.0 方法论]]——四层「地图」讲完，进入 Prompt 层「放大图」。

---

## 参考来源（一手链接 · 可溯源深挖）

- **Boris Cherny (2026-06-02)** WorkOS Acquired Unplugged 演讲——原始视频待核实（WorkOS / Acquired 播客）
- **Peter Steinberger (2026-06-07)** X 推文「designing loops that prompt your agents」——原始帖待核实（@steipete）
- **Addy Osmani (2026-06-08)**《Loop Engineering》正式命名：原博 [addyosmani.com/blog/loop-engineering](https://addyosmani.com/blog/loop-engineering)（待核实）；可靠综述 [tosea.ai/blog/loop-engineering-ai-agents-complete-guide-2026](https://tosea.ai/blog/loop-engineering-ai-agents-complete-guide-2026)
- **Anthropic (2024-12)**《Building Effective Agents》：[anthropic.com/engineering/building-effective-agents](https://www.anthropic.com/engineering/building-effective-agents)
- **Simon Willison**「Agents are models using tools in a loop」——原始博客待核实
- **Braintrust (2025-08)**《The canonical agent architecture: A while loop with tools》——原始链接待核实
- **Anthropic (2026-06)**《When AI builds itself》（>80% 代码、8× 合并、76% 成功率）：[anthropic.com/institute/recursive-self-improvement](https://anthropic.com/institute/recursive-self-improvement)
- **Claude Fable 5 (2026-06-09)**「verifier sub-agents 优于 self-critique、rubric loop 6×」——具体 6× 数据待核实（见 Anthropic Fable 5 发布报道）

**相关链接**：
- 系列清单：[[学习计划/Agent 方法论与产品思维学习清单|Agent 方法论与产品思维学习清单]]
- 上一层级：[[00-Agent 方法论与产品思维|认知升级：四层模型 · 索引]]
- 同主题：[[03-2026初 Harness Engineering|Harness Engineering]] · [[08-Boris Cherny语录|Boris Cherny 语录]]

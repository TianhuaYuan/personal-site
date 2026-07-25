---
title: "Boris Cherny语录"
tags:
  - agent方法论
  - 认知升级
  - 四层模型
  - loop
created: "2026-07-21"
---

# Boris Cherny 语录与范式转移信号

> 本条目是「认知升级：四层模型」的第 8 部分，对应学习清单条目 1.2.4。
>
> **前置依赖**：四层模型整体认知
> **为以下铺垫**：后续所有方法论学习

---

## 一、核心观点

> **Boris 的语录是分水岭信号**：当造工具的人都说「我不 prompt 了，我写 loop」，说明范式真的转移了——人的角色从「操作者」变成「设计者」。

---

## 二、Boris Cherny 是谁

- **职位**：Anthropic Claude Code 负责人（Claude Code 最初是他在 2024-09 的副项目）
- **影响力**：AI 编程领域重要话语权；据披露 Claude Code 背后接近 **4%** 的公开 GitHub commit

---

## 三、标志性语录（可验证）

> **"I don't prompt Claude anymore. I have loops running that prompt Claude and figuring out what to do. My job is to write loops."**

- **来源**：WorkOS 赞助的 Acquired Unplugged 活动，**2026-06-02**
- **翻译**：「我已经不 prompt Claude 了。我有一堆 loop 在跑、在 prompt Claude、在判断下一步该做什么。我的工作是写 loop。」

**个人工作流演进**（他自己披露）：
1. 一年前：用自动补全手写代码
2. 中间：并行跑 5–10 个 Claude 会话逐个提示
3. 现在：完全不再提示，数百个 Claude 实例自动读 GitHub issues、扫 Twitter 反馈、理 Slack，自行决定下一步构建什么

**成果数据**：过去 30 天，Claude Code 仓库 **100%** 贡献（259 个 PR）由 Claude Code 自己完成；2025-11 删 IDE 后再未打开。

---

## 四、为什么这句话重要

| 信号 | 含义 |
|------|------|
| 「我不 prompt 了」 | Prompt 不再是核心工作 |
| 「我有 loop 在跑」 | Loop Engineering 已落地生产 |
| 「我的工作是写 loop」 | 人的角色从「操作者」变「设计者」 |

Boris 是**造工具的人**，他的实践即行业风向标——这不是概念，是生产实践。

---

## 五、同期佐证（2026 年 6 月）

| 时间 | 人物 | 内容 |
|------|------|------|
| 2026-06-07 | Peter Steinberger | 「You should be designing loops that prompt your agents.」24h 约 220 万浏览，数日破 650 万 |
| 2026-06-08 | Addy Osmani | 发文《Loop Engineering》正式命名，拆出六大构件 |
| 2026-06 | 多家媒体 | tosea.ai、腾讯云开发者、juejin 等一致确认 Loop Engineering 成 2026 主流趋势 |

---

## 六、支撑数据（2024–2026）

- **Anthropic《When AI builds itself》(2026-06)**：Claude 写代码占合并量 >80%（2026-05）；工程师日均合并量较 2024 年 **8 倍**；开放式任务成功率 **76%**（半年前 26%）。
- 这些数字印证核心假设：**当 Agent 能在 Loop 中自主运行，人的产出不取决于写代码速度，而取决于设计 Loop 的质量。**

---

## 七、学习资源

- Boris Cherny @ WorkOS Acquired Unplugged (2026-06-02)
- Peter Steinberger 推文 (2026-06-07)；Addy Osmani《Loop Engineering》(2026-06-08)
- 进阶：[[04-2026中 Loop Engineering 爆发|Loop Engineering 爆发]] · [[09-一句话说清区别|一句话说清区别]]

---

## 参考来源（一手链接 · 可溯源深挖）

- **Boris Cherny** 披露 ~4% 公开 GitHub commit 来自 Claude Code（二手报道，原始待核实）
- **Boris Cherny (2026-06-02)** WorkOS Acquired Unplugged——原始视频待核实
- **Peter Steinberger (2026-06-07)** X 推文（@steipete，待核实）
- **Addy Osmani (2026-06-08)**《Loop Engineering》：[tosea.ai 综述](https://tosea.ai/blog/loop-engineering-ai-agents-complete-guide-2026)
- **Anthropic (2026-06)**《When AI builds itself》：[anthropic.com/institute/recursive-self-improvement](https://anthropic.com/institute/recursive-self-improvement)

**相关链接**：
- 系列清单：[[学习计划/Agent 方法论与产品思维学习清单|Agent 方法论与产品思维学习清单]]
- 上一层级：[[00-Agent 方法论与产品思维|认知升级：四层模型 · 索引]]
- 同主题：[[04-2026中 Loop Engineering 爆发|Loop Engineering 爆发]] · [[10-画出四层模型演进图|四层模型演进图]]

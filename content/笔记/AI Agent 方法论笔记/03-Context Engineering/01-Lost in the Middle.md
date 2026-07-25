---
title: "Lost in the Middle"
tags:
  - agent方法论
  - 理论根基
created: "2026-07-21"
---

# Lost in the Middle：模型也会「翻到中间就走神」

> 本条目是「Context Engineering」的第 1 部分，对应学习清单条目 3.1.1。
>
> **前置依赖**：认知升级：四层模型、Prompt 2.0 方法论
> **为以下铺垫**：Attention Budget、首末重注入、Context Rot

---

## 核心观点

> **Lost in the Middle（中间迷失）**：把关键信息塞进超长上下文的「中间」，模型的指令遵循率会显著塌方——开头和结尾最可靠，中间最容易「被略过不看」。

想象你参加一场 2 小时的马拉松会议，20 个人依次发言 5 分钟。散会后有人问你：「第 8 个发言的人说了啥？」——你大概率记得开头那几位（首因效应 primacy）和最后几位（近因效应 recency），中间那群人基本糊成一团。LLM 读长上下文时，和人类开长会一模一样：**开头和结尾记得牢，中间基本「听了但没完全听」**。只不过模型不会承认走神，它会很自信地答错。

## 定义 / 原理 / 实践 / 示例 / 优劣势

### 定义与原理

Lost in the Middle 由 Liu 等（Stanford、UC Berkeley、Samaya AI）在 2023 年的论文《Lost in the Middle: How Language Models Use Long Contexts》提出。核心发现是 **U 型注意力曲线（U-shaped attention curve）**：当相关信息出现在上下文开头或结尾时模型表现最好，被埋在中间时表现骤降。

### 为什么会发生（注意力分布）

```mermaid
xychart-beta
    title "U 型注意力曲线：位置 → 信息利用率"
    x-axis ["开头" "1/4" "中间" "3/4" "结尾"]
    y-axis "信息利用率（越高越不易被忽略）" 0 --> 100
    line [95, 80, 40, 80, 95]
```

底层原因有两点（Anthropic 2025 也重申过）：
1. **架构层面**：Transformer 是「每个 token 都要和上下文里所有其他 token 两两互相注意」的结构，产生 n² 的成对关系。上下文一长，这层关系被拉得越来越薄，中间 token 分到的注意力最少。
2. **训练分布层面**：训练数据里短序列更常见，且指令、摘要往往落在开头/结尾，模型对「中间」天生经验最少。

### 工程实践（怎么绕开它）

- **关键信息放首尾**：系统规则放开头，当前任务放结尾，长文档/历史堆中间。
- **重排序检索结果**：RAG 里把最相关 chunk 摆到首尾，别按相关性顺序一路铺下来。
- **少而精**：宁可少放几条高质量片段，也不要堆一堆 mediocre（平庸）的中间噪声——加无关中间内容会主动伤害表现。

### 优劣势

- ✅ 这个规律跨模型一致（GPT-3.5、GPT-4、Claude、Llama 2 都中招），可预测、可对抗。
- ❌ 不是「硬悬崖」而是「性能梯度」，不同模型严重程度不同；CLAUDE.md / System Prompt 越长越容易把真规则挤到中间被忽略。

## 核心要点

| 概念 | 一句话 |
|------|--------|
| Lost in the Middle | 长上下文中间的信息最容易被模型忽略 |
| U 型曲线 | 开头、结尾高，中间低 |
| 代价 | 关键信息埋中间，准确率可掉 15–25 个百分点（arXiv:2307.03172） |
| 对策 | 关键规则放首尾 + 检索结果重排序 |

## 最新研究与企业数据

- **原始论文（Liu et al., 2023）**：多文档 QA 中，把含答案的文档从「开头」挪到「中间」，模型准确率下降 **15–25 个百分点**；GPT-3.5-Turbo 在 20/30 文档设置下甚至**跌破无文档的闭卷基线（56.1%）**（来源：arXiv:2307.03172）。
- **性能是梯度而非悬崖**：Anthropic（2025-09）指出 context rot（上下文衰减）在所有模型上都出现，只是有的模型衰减更平缓；注意力是「有限预算、边际收益递减」（来源：Anthropic Effective Context Engineering）。
- **实践拐点观测**：从业者复盘称质量在窗口约 **40–50%** 处开始明显下滑，但未到硬限制（来源：crystl.dev 2026 从业者博客，属次级来源，建议自测校准）。

## 学习资源

- **必读论文**：Liu et al. (2023)《Lost in the Middle》：[arXiv:2307.03172](https://arxiv.org/abs/2307.03172)
- **延伸**：Anthropic《Effective context engineering for AI agents》(2025-09)
- **进阶**：[[02-Attention Budget|Attention Budget]] · [[05-首末重注入|首末重注入]] · [[08-摘要压缩|摘要压缩]]

## 相关链接

- 系列清单：[[学习计划/Agent 方法论与产品思维学习清单|Agent 方法论与产品思维学习清单]]
- 上一层级：[[00-Agent 方法论与产品思维|Context Engineering · 索引]]
- 关联理论：[[../01-认知升级/07-四层嵌套关系|四层嵌套关系]]

**下一篇**：[[02-Attention Budget|Attention Budget]]——中间迷失讲完，下篇讲为什么每个 token 都在偷偷消耗模型的「注意力预算」。

---

## 参考来源（一手链接 · 可溯源深挖）

- **Liu et al. (2023)**《Lost in the Middle: How Language Models Use Long Contexts》(Stanford / UC Berkeley / Samaya AI)：[arXiv:2307.03172](https://arxiv.org/abs/2307.03172)
- **Anthropic (2025-09)**《Effective context engineering for AI agents》：[anthropic.com/engineering/effective-context-engineering-for-ai-agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- **crystl.dev (2026)**《Optimize Claude Code's Context Window》：[crystl.dev/blog/optimize-claude-code-context](https://crystl.dev/blog/optimize-claude-code-context) —— 次级从业者博客，40–50% 拐点为经验观测，非受控实验。

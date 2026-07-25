---
title: "2025 Context Engineering 兴起"
tags:
  - agent方法论
  - 认知升级
  - 四层模型
  - context
created: "2026-07-21"
---

# 2025 Context Engineering 兴起：RAG + 信息筛选 + 注意力管理

> 本条目是「认知升级：四层模型」的第 2 部分，对应学习清单条目 1.1.2。
>
> **前置依赖**：Prompt Engineering
> **为以下铺垫**：Harness Engineering、Loop Engineering

---

## 一、核心观点

> **Context Engineering 研究「模型在推理那一刻能看到的所有 token」——料不对，问得再好也白搭。**

---

## 二、定义与出处（可验证）

### 2.1 定义

Context（上下文）指从 LLM 中采样时所包含的**整个 token 集合**，不只是你写的 Prompt，还包括：

- System Prompt、检索结果、对话历史
- 工具描述、Agent 状态、外部知识与 MCP 数据

Context Engineering 即**筛选与维护**这组 token、以持续产生期望行为的策略集合。

### 2.2 出处时间线

| 时间 | 事件 | 来源 |
|------|------|------|
| 2025-06 | Shopify CEO **Tobi Lütke** 在 X 提出「核心技能是 Context Engineering 而非 Prompt Engineering」 | X / Tobi Lütke |
| 2025 年中 | **Andrej Karpathy** 给出经典定义：「一门微妙的艺术与科学，旨在为下一步推理填入恰到好处的信息」 | Karpathy |
| 2025-09-29 | **Anthropic** 正式发文《Effective context engineering for AI agents》系统化该学科 | Anthropic Applied AI 团队 |

> Anthropic 将 Context Engineering 视为 Prompt Engineering 的**自然演进**，而非替代。

---

## 三、核心技术

### 3.1 RAG（Retrieval-Augmented Generation）

```text
用户查询 → 检索相关文档 → 注入上下文 → LLM 生成回答
```

- **向量数据库**：ChromaDB、Pinecone、Weaviate
- **嵌入模型**：OpenAI Ada、BGE、E5
- **检索策略**：相似度搜索、混合检索、重排序（Rerank）

### 3.2 注意力管理（Anthropic 实证要点）

| 现象 | 含义 | 工程对策 |
|------|------|----------|
| **Lost in the Middle** | 长上下文中间信息遵循率仅 ~60%（Liu et al., 2023） | 关键规则放首尾 |
| **Recency / Primacy Bias** | 开头与末尾权重更高 | hard gate 放末尾 |
| **Attention Budget** | 每 token 注意力有限，超阈值开始忽略 | 控制总量、渐进式加载 |

### 3.3 Anthropic 推荐的长期策略

1. **Compaction（压缩）**：接近窗口上限时总结历史，最简单的形式是清除早期原始工具调用结果
2. **Structured note-taking（结构化笔记 / Agentic Memory）**：Agent 将进度写进上下文之外的 `NOTES.md` 等文件，按需读回——如 Claude Code 的 to-do、Claude 玩 Pokémon 跨数千步保持 tally
3. **Sub-agent 架构**：子 Agent 在独立上下文窗口深挖，只回传 1k–2k token 的精炼摘要（Anthropic 多 Agent 研究系统实证优于单 Agent）
4. **Just-in-time 上下文**：运行时用工具按需拉取，而非预加载全部（渐进式披露）

---

## 四、与 Prompt Engineering 的对比

| 维度 | Prompt Engineering | Context Engineering |
|------|-------------------|---------------------|
| **关注点** | 怎么问 | 模型看到什么 |
| **优化目标** | 单次调用措辞 | 整个推理过程的信息 |
| **核心技术** | Few-shot、CoT、格式 | RAG、窗口管理、注意力管理 |
| **局限** | 只管输入 | 需更多工程基础设施 |

---

## 五、局限与下一步

Context Engineering 解决了「模型看到什么」，但没解决：

- Agent 在**什么环境**运行（引出 [[03-2026初 Harness Engineering|Harness Engineering]]）
- 工具如何调用、错误如何处理、循环如何设计（引出 [[04-2026中 Loop Engineering 爆发|Loop Engineering]]）

---

## 六、最新研究与企业数据（2024–2026）

- **Context Rot 被量化**：Transformer 架构下「上下文规模」与「注意力集中度」天然矛盾，token 越多召回越差（Anthropic，2025-09；Liu et al. *Lost in the Middle*, 2023）。
- **渐进式披露成为主流实践**：Anthropic、LangChain 均将「按需加载」列为 Agent 可靠性第一原则。
- **基准进步间接印证**：Stanford HAI（2026-04）报告 OSWorld 计算机任务 Agent 成功率从 2024 年 ~12% 升至 **66.3%**（接近人类 ~72%）——长上下文管理与工具调用可靠性的提升是关键贡献之一。

---

## 七、学习资源

- **官方/权威**
  - Anthropic《Effective context engineering for AI agents》(2025-09)
  - *Lost in the Middle: How Language Models Use Long Contexts*（Liu et al., 2023）
- **实践指南**
  - LangChain RAG Tutorial；Anthropic Memory & Context Management Cookbook
- **进阶阅读**
  - [[03-2026初 Harness Engineering|2026 初 Harness Engineering]]——Context 策展好了，Agent 还需要「工作台」

---

**下一篇**：[[03-2026初 Harness Engineering|2026 初 Harness Engineering]]——Context 策展好了，但 Agent 还需要「工作台」才能干活。

---

## 参考来源（一手链接 · 可溯源深挖）

- **Tobi Lütke (2025-06)** 提出「context engineering」一词（X / @tobi，原始帖待核实）
- **Karpathy (2025-06-25)** 转发背书 context engineering（X / @karpathy，原始帖待核实）
- **Anthropic (2025-09-29)**《Effective context engineering for AI agents》：[anthropic.com/engineering/effective-context-engineering-for-ai-agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- **Liu et al. (2023)** Lost in the Middle：[arXiv:2307.03172](https://arxiv.org/abs/2307.03172)
- **Anthropic** 多 Agent 研究系统实证：[anthropic.com/engineering/built-multi-agent-research-system](https://www.anthropic.com/engineering/built-multi-agent-research-system)
- **Stanford HAI (2026-04)** AI Index：OSWorld 成功率 12%→66.3%：[Stanford HAI AI Index 2026](https://hai.stanford.edu/ai-index)
- **LangChain RAG Tutorial** / **Anthropic Memory & Context Management Cookbook**：[python.langchain.com/docs/tutorials/rag](https://python.langchain.com/docs/tutorials/rag/)（RAG 教程）；[docs.anthropic.com](https://docs.anthropic.com/en/docs/build-with-claude/context-management)（Context 管理 Cookbook）

**相关链接**：
- 系列清单：[[学习计划/Agent 方法论与产品思维学习清单|Agent 方法论与产品思维学习清单]]
- 上一层级：[[00-Agent 方法论与产品思维|认知升级：四层模型 · 索引]]
- 同主题：[[01-2024 Prompt Engineering 时代|Prompt Engineering 时代]] · [[10-画出四层模型演进图|四层模型演进图]]

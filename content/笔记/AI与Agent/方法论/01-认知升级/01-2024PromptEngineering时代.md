---
title: "2024 Prompt Engineering 时代"
tags:
  - agent方法论
  - 认知升级
  - 四层模型
  - prompt
created: "2026-07-21"
---

# 2024 Prompt Engineering 时代：ChatGPT 级提示词技巧

> 本条目是「认知升级：四层模型」的第 1 部分，对应学习清单条目 1.1.1。
>
> **前置依赖**：无（系列起点）
> **为以下铺垫**：Context Engineering、Harness Engineering、Loop Engineering

---

## 一、核心观点

> **Prompt Engineering 解决「怎么把一句话问清楚」，是四层模型的底座，但只是 Context 的子集。**
> 料不对，问得再好也白搭。

---

### 2.1 定义

Prompt Engineering（提示词工程）是研究「**怎么把一句话问清楚**」的技术。

- **核心动作**：设计输入文本，引导大语言模型（LLM）生成期望输出
- **本质**：人与 AI 的沟通界面，类似「给实习生写操作手册」
- **时间窗口**：2022–2024 年，随 ChatGPT 普及成为最核心的 AI 工程技能

### 2.2 核心技术

| 技术 | 说明 | 示例 |
|------|------|------|
| **角色设定** | 给模型一个身份 | "你是一个资深 Python 工程师" |
| **Few-shot 示例** | 提供输入输出示例 | "输入：天气真好 → 输出：Positive" |
| **Chain-of-Thought (CoT)** | 引导模型逐步推理 | "请一步一步思考……" |
| **输出格式约束** | 指定返回结构 | "请用 JSON 格式返回" |
| **温度控制** | 调节随机性 | `temperature=0.7` |

### 2.3 演进三阶段

| 阶段 | 时间 | 特征 | 代表 |
|------|------|------|------|
| 手工调优 | 2022–2023 | 人工试错，难复用 | ChatGPT Playground |
| 框架化 | 2023–2024 | 模板化、可优化 | LangChain PromptTemplate、DSPy、CRISPE |
| 系统化 | 2024–2025 | 与系统设计结合 | System Prompt、Function Calling、多轮上下文管理 |

> **关键转变**：从「手艺」走向「工程」，但优化边界始终停在**单次 LLM 调用**。

---

## 三、为什么 Prompt 仍然是基础

尽管 2025–2026 出现了 Context / Harness / Loop 三层，Prompt 并未过时：

1. **它是 Context 的子集**：`Context ⊃ Prompt`，写好 Prompt 是做好 Context Engineering 的前提
2. **它是单次调用的最优解**：简单任务（问答、分类、生成）只需好 Prompt，无需复杂架构
3. **工程铁律——从最简方案起步**：不是所有场景都需要 Agent 化

---

### 4.1 只管「怎么问」，不管「模型看到什么」

Prompt 只优化了输入措辞，但模型**实际能看到的信息**可能不完整：

> **类比**：你给实习生写了一份完美的操作手册，但他桌上没有相关文档、没有访问权限、没有工具——手册再好也没用。

### 4.2 单次调用的天花板

- ✅ 可优化：措辞、格式、示例、推理步骤
- ❌ 无法优化：信息缺失、环境限制、系统性问题
- ❌ 难以扩展：每次新任务都要重新设计 Prompt

### 4.3 脆弱且难维护

- 对措辞敏感，微小改动可能导致输出剧变
- 不同模型、不同场景需要重新设计
- 散落在代码各处，缺乏版本管理

---

## 五、从 Prompt 到 Context 的认知跃迁

| 维度 | Prompt Engineering | Context Engineering |
|------|-------------------|---------------------|
| **关注点** | 怎么问 | 模型看到什么 |
| **优化目标** | 单次调用的措辞 | 整个推理过程的信息 |
| **类比** | 写操作手册 | 准备工作台 + 文档 + 权限 |
| **局限** | 只管输入 | 管理整个信息环境 |

> **一句话总结**：Prompt Engineering 解决「怎么问」，Context Engineering 解决「模型看到什么」。

---

## 六、最新研究与企业数据（2024–2026）

- **权重转移已成共识**：2026 年行业从 "写 Prompt" 转向 "设计 Loop"（详见 [[04-2026中LoopEngineering爆发|Loop Engineering]] 与 [[05-Prompt权重变化|Prompt 权重变化]]）。Stanford HAI（2026-04）调研显示，已部署 AI 的企业在软件开发上获得 **26%** 生产率提升——但这主要源于系统化（Context/Harness/Loop）而非单点 Prompt 优化。
- **模型越强，Prompt 越「轻」**：Anthropic 在《Effective context engineering for AI agents》（2025-09）中指出，更聪明的模型需要更少规定性（prescriptive）的 Prompt，但「把上下文当作有限资源」这一原则始终成立。

---

## 七、学习资源

- **经典论文**
  - *Chain-of-Thought Prompting Elicits Reasoning in Large Language Models*（Wei et al., 2022, Google）
  - *Large Language Models are Zero-Shot Reasoners*（Kojima et al., 2022）
  - *DSPy: Compiling Declarative Language Model Calls into Self-Improving Pipelines*（Khattab et al., 2023）
- **实践指南**
  - OpenAI Prompt Engineering Guide
  - Anthropic Prompt Engineering Documentation
- **进阶阅读**
  - [[02-2025ContextEngineering兴起|2025 Context Engineering 兴起]]——Prompt 写得再好，上下文被污染也会失效

---

**下一篇**：[[02-2025ContextEngineering兴起|2025 Context Engineering 兴起]]——下一层解决「模型看到什么」。

---

## 参考来源（一手链接 · 可溯源深挖）

- **Stanford HAI (2026-04)** AI Index：已部署 AI 企业软件开发生产率 +26%：[Stanford HAI AI Index 2026](https://hai.stanford.edu/ai-index)
- **Anthropic (2025-09)**《Effective context engineering for AI agents》：[anthropic.com/engineering/effective-context-engineering-for-ai-agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- **Wei et al. (2022)** Chain-of-Thought Prompting（Google）：[arXiv:2201.11903](https://arxiv.org/abs/2201.11903)
- **Kojima et al. (2022)** Large Language Models are Zero-Shot Reasoners：[arXiv:2205.11916](https://arxiv.org/abs/2205.11916)
- **Khattab et al. (2023)** DSPy：[arXiv:2310.03714](https://arxiv.org/abs/2310.03714)
- **OpenAI Prompt Engineering Guide**：[platform.openai.com/docs/guides/prompt-engineering](https://platform.openai.com/docs/guides/prompt-engineering)
- **Anthropic Prompt Engineering Documentation**：[docs.anthropic.com](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering)


## 速记卡（面试闪卡）

**Q1：一句话讲清「2024 Prompt Engineering 时代：ChatGPT 级提示词技巧」到底是什么？**
A：Prompt Engineering 是四层模型的底座：研究"怎么把一句话问清楚"，但只是 Context 的子集，管不了模型实际看到什么。

**Q2：它包含哪些核心技术 —— 怎么理解？**
A：像给实习生写操作手册的几种写法：角色设定（"你是资深 Python 工程师"）、Few-shot 示例（给输入输出样例）、Chain-of-Thought（CoT，引导逐步推理）、输出格式约束（"用 JSON 返回"）、温度控制（调节随机性）。演进从 2022 手工试错，到框架化（LangChain/DSPy），再到 2024 与系统设计结合。

**Q3：为什么 Prompt 仍然是基础 —— 怎么理解？**
A：三个理由：① 它是 Context 的子集（`Context ⊃ Prompt`），写好 Prompt 是做好上下文工程的前提；② 简单任务（问答、分类、生成）只需好 Prompt，无需复杂 Agent 架构；③ 工程铁律——从最简方案起步，不是所有场景都要 Agent 化。模型越强，Prompt 反而越"轻"，但"上下文是有限资源"始终成立。

**Q4：它的致命局限 —— 怎么理解？**
A：只管"怎么问"，不管"模型看到什么"——像给实习生写了完美手册，但他桌上没文档、没权限、没工具，手册再好也白搭。而且单次调用天花板明显：信息缺失、环境限制它优化不了；对措辞还敏感，微小改动可能输出剧变，散落代码难维护。

**Q5：从 Prompt 到 Context 的认知跃迁 —— 怎么理解？**
A：Prompt Engineering 解决"怎么问"，Context Engineering（上下文工程）解决"模型看到什么"——前者优化单次调用的措辞，后者管理整个推理过程的信息环境（工作台+文档+权限）。2026 行业共识已从"写 Prompt"转向"设计 Loop"，但 Prompt 仍是不可跳过的地基。

**Q6：核心速记主线有哪些？**
- Prompt 管"怎么问"，是四层模型底座
- 核心技术：角色/Few-shot/CoT/格式/温度
- 局限：不管模型看到什么、单次天花板
- Context ⊃ Prompt；2026 转向设计 Loop

**口诀**
A：Prompt 底座问清楚，角色示例 CoT 格式温
Context 包含 Prompt，模型看到才关键
只管怎么问不够，缺文档权限白搭
2026 转设计 Loop，地基 Prompt 仍要打

## 相关链接
- 系列清单：[[学习路线图/Agent方法论与产品思维学习路线图|Agent 方法论与产品思维学习路线图]]
- 上一层级：[[00-Agent方法论与产品思维|认知升级：四层模型 · 索引]]
- 同主题：[[05-Prompt权重变化|Prompt 权重变化]] · [[06-设计Agent思考框架|设计 Agent 思考框架]]

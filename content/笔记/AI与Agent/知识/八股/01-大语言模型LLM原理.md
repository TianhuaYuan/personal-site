---
title: "大语言模型LLM原理"
created: "2026-07-13"
tags:
  - 八股文
  - ai
---

# 大语言模型LLM原理

> **生活化类比**：把 LLM 想象成一位"读了海量书籍的助理"——预训练是它通读整个图书馆积累常识，SFT 是岗前培训教它按指令办事，RLHF/DPO 对齐则是老员工带教，让它说话更合人意。

大语言模型（Large Language Model, LLM）是基于 Transformer 架构、在海量文本上预训练的神经网络模型。理解 LLM 的训练流程和推理优化是 AI 岗位的核心考点。

## LLM 生命周期总览

```mermaid
graph LR
    RT[原始文本] --> TK[Tokenization 分词]
    TK --> PT[Pre-training 预训练<br/>学会语言能力]
    PT --> SF[SFT 监督微调<br/>学会遵循指令]
    SF --> AL[RLHF / DPO 对齐<br/>学会人类偏好]
    AL --> DP[部署推理]
```

> **一句话串联**：预训练（Pre-training）让模型"懂语言"，SFT（Supervised Fine-Tuning 监督微调）让模型"听指令"，RLHF（Reinforcement Learning from Human Feedback 基于人类反馈的强化学习）/ DPO（Direct Preference Optimization 直接偏好优化）让模型"合人意"。三者逐级收窄能力目标。

## Tokenization 分词

LLM 不直接处理原始文本，而是先将文本拆分为 **Token**（子词单元）。

### BPE（Byte-Pair Encoding）算法

BPE 是当前主流的分词算法，GPT 系列和 LLaMA 均使用。

**核心思想：** 从字符级别开始，反复合并出现频率最高的相邻 token 对，直到达到目标词表大小。

```text
初始：所有单个字符都在词表中
第1轮：统计所有相邻字符对频率，合并最高频的（如 "t" + "h" → "th"）
第2轮：在合并后的序列上继续统计，合并最高频的（如 "th" + "e" → "the"）
...重复直到词表达到预设大小
```

**为什么不用词级别分词？**

| 方式 | 优点 | 缺点 |
| ------ | ------ | ------ |
| 词级别 | 语义清晰 | 词表巨大（几十万），OOV 问题严重 |
| 字符级别 | 词表小，无 OV 问题 | 序列太长，难以捕获语义 |
| 子词（BPE） | 平衡词表大小和序列长度 | 需要训练分词器 |

### 主流模型的词表大小

| 模型 | 词表大小 | 分词算法 |
| ------ | --------- | --------- |
| GPT-2 | 50,257 | BPE |
| GPT-3/4 | 100,256 | BPE |
| LLaMA | 32,000 | SentencePiece BPE |
| LLaMA 3 | 128,256 | Tiktoken BPE |
| Qwen | 151,936 | Tiktoken BPE |
| BERT | 30,522 | WordPiece |

## Pre-training 预训练

预训练阶段让模型学习语言的基本能力。

### 训练目标

| 目标 | 说明 | 代表模型 |
| ------ | ------ | --------- |
| Causal LM（自回归） | 预测下一个 token：P(x_t | x_1, ..., x_{t-1}) | GPT, LLaMA, Qwen |
| Masked LM（掩码） | 预测被 mask 的 token | BERT, RoBERTa |
| Prefix LM | 前缀双向编码 + 后缀自回归 | T5, PaLM |

**为什么 Decoder-Only + Causal LM 成为主流？**
- 自然适配生成任务
- Scaling Law 表明在大参数量下效果最好
- 训练效率高（每个 token 都贡献一次梯度）

### 训练数据

| 数据来源 | 规模 | 特点 |
| --------- | ------ | ------ |
| Common Crawl | 数 TB | 网页数据，质量参差不齐 |
| Wikipedia | ~20GB | 高质量百科知识 |
| Books | ~100GB | 长文本，叙事结构 |
| Code (GitHub) | 数 TB | 增强逻辑推理能力 |
| StackExchange | ~50GB | 问答格式，有助于指令理解 |

> **要点**：数据质量比数据量更重要。LLaMA 2 使用了经过精心筛选的 2 万亿 token 数据集，效果优于使用 4 倍以上数据但质量较差的模型。

### Scaling Laws 缩放定律

Kaplan et al. (2020) 和 Chinchilla (Hoffmann et al., 2022) 发现了 LLM 的缩放规律：

```text
性能（Loss）≈ f(参数量 N, 数据量 D, 计算量 C)

Chinchilla 最优分配：
C ≈ 6 × N × D（计算量固定时）
最优比例：N ∝ D（参数量和数据量应同步增长）
```

| 模型 | 参数量 | 训练 Token 数 | 是否符合 Chinchilla |
| ------ | -------- | ------------- | ------------------- |
| GPT-3 | 175B | 300B | 数据不足（过参数化） |
| LLaMA | 7B-65B | 1T-1.4T | 接近最优 |
| Chinchilla | 70B | 1.4T | 最优比例 |
| LLaMA 3 | 8B-405B | 15T | 过度训练（为了推理效率） |

## Supervised Fine-Tuning (SFT)

SFT 使用高质量的「指令-回复」数据对模型进行微调，让模型学会遵循人类指令。

### 训练数据格式

```json
{
  "instruction": "请解释什么是机器学习",
  "input": "",
  "output": "机器学习是人工智能的一个分支，它让计算机能够从数据中学习规律，而不是被显式编程..."
}
```

### SFT 的关键技巧

| 技巧 | 说明 |
| ------ | ------ |
| 数据质量 > 数量 | 1000 条高质量数据可能超过 10 万条低质量数据 |
| 多样性 | 覆盖多种任务类型（问答、翻译、代码、数学等） |
| 对话模板 | 使用 ChatML 等标准模板，区分 system/user/assistant |
| Loss Masking | 只在 assistant 回复部分计算 loss，不计算 user 输入部分 |

## RLHF 人类反馈强化学习

RLHF 的目标是让模型的输出更符合人类偏好。

### 三阶段流程

```text
阶段1: Reward Model 训练
  收集人类偏好数据（给同一 prompt 的两个回复排序）
  训练奖励模型：R(prompt, response) → 奖励分数

阶段2: PPO 强化学习
  使用 PPO 算法优化策略模型，最大化奖励同时用 KL 散度约束不要偏离原始模型太远
  目标：max R(x,y) - β·KL(π_θ || π_ref)
```

### DPO 直接偏好优化

DPO 是 RLHF 的简化替代方案，无需训练单独的 Reward Model：

| 特性 | RLHF (PPO) | DPO |
| ------ | ----------- | ----- |
| 需要 Reward Model | 是 | 否 |
| 训练稳定性 | 较差（超参敏感） | 更好 |
| 计算开销 | 大（需要多个模型） | 小 |
| 效果 | 上限更高 | 接近 RLHF |

DPO 的核心公式：`L_DPO = -log σ(β(log π_θ(y_w|x)/π_ref(y_w|x) - log π_θ(y_l|x)/π_ref(y_l|x)))`

其中 y_w 是人类偏好的回复（winner），y_l 是不偏好的回复（loser）。

## Inference 推理优化

### Temperature / Top-p / Top-k 采样

| 参数 | 范围 | 作用 |
| ------ | ------ | ------ |
| Temperature | 0-2 | 控制随机性。=1 原始分布；→0 趋近 greedy；→∞ 趋近均匀 |
| Top-k | 1-100 | 只从概率最高的 k 个 token 中采样 |
| Top-p (Nucleus) | 0-1 | 只从累积概率达到 p 的最小 token 集合中采样 |

**组合策略：**
- 代码生成：temperature=0, top_p=1（确定性输出）
- 创意写作：temperature=0.7~1.0, top_p=0.9
- 通用对话：temperature=0.7, top_p=0.9

### KV Cache

参见 [[03-自注意力机制与Transformer基础]] 中的 KV Cache 章节。

### Speculative Decoding 推测解码

用一个小模型（draft model）快速生成多个候选 token，然后用大模型并行验证。

```text
Draft Model: 快速生成 K 个 token → [t1, t2, t3, t4, t5]
Target Model: 并行验证这 K 个 token → 接受前 3 个，拒绝第 4 个
速度提升：2-3x（大模型只做一次前向传播，而非 5 次）
```

### 量化 (Quantization)

将模型权重从 FP16 压缩到 INT8 或 INT4，减少显存占用和推理延迟。

| 精度 | 每参数字节 | 7B 模型显存 | 效果损失 |
| ------ | ---------- | ----------- | --------- |
| FP16 | 2 bytes | ~14 GB | 无 |
| INT8 | 1 byte | ~7 GB | 极小 |
| INT4 | 0.5 bytes | ~3.5 GB | 可接受 |

**主流量化方法：** GPTQ（训练后量化）、AWQ（激活感知量化）、GGUF（llama.cpp 格式）

---

## 
> ▶ 对应实操：[[01-Token计费原理-Temperature控制-SystemPrompt层级|01-Token计费原理-Temperature控制-SystemPrompt层级]]


> ▶ 对应实操：[[02-LLM本质-API调用封装|02-LLM本质-API调用封装]]

相关链接
- [[八股文学习清单]]
- [[00-全局导航|全局导航]]

---

## 快速问答

| 问题 | 参考答案 |
| ------ | --------- |
| GPT 和 BERT 的核心区别？ | GPT 是 Decoder-Only，自回归生成；BERT 是 Encoder-Only，双向编码。GPT 适合生成任务，BERT 适合理解任务 |
| 为什么现在都用 Decoder-Only？ | ① Scaling Law 表明大参数下效果好 ② 统一了理解和生成 ③ 训练效率高 |
| 预训练和微调的区别？ | 预训练在海量无标注数据上学习语言能力（自监督）；微调在特定任务数据上调整模型行为（监督） |
| RLHF 解决了什么问题？ | SFT 模型可能生成有害/不准确/冗长的内容，RLHF 通过人类偏好数据让模型输出更符合人类期望 |
| 什么是幻觉（Hallucination）？ | 模型自信地生成看似合理但实际错误的内容。原因：训练数据噪声、解码策略、知识边界模糊 |
| 如何减少幻觉？ | ① RAG 引入外部知识 ② RLHF 对齐训练 ③ 低 temperature ④ 后处理验证 ⑤ 思维链推理 |
| LoRA 的核心思想是什么？ | 冻结原始权重，只训练低秩分解的增量矩阵 ΔW = A×B，A(d×r), B(r×d)，r<<d，参数量大幅减少 |
| 如何评估 LLM？ | 基准测试（MMLU、HumanEval、GSM8K）、人工评估（Chatbot Arena）、特定任务评估 |

## 速记卡（面试闪卡）

**Q1：一句话讲清「大语言模型LLM原理」到底是什么？**
A：**一句话串联**：预训练（Pre-training）让模型"懂语言"，SFT（Supervised Fine-Tuning 监督微调）让模型"听指令"，RLHF（Reinforcement Learning from Human Feedback 基于人类反馈的强化学习）/ DPO（Direct Preference Optimization 直接偏好优化）让模型"合人意"。

**Q2：LLM 生命周期总览 —— 怎么理解？**
A：**一句话串联**：预训练（Pre-training）让模型"懂语言"，SFT（Supervised Fine-Tuning 监督微调）让模型"听指令"，RLHF（Reinforcement Learning from Human Feedback 基于人类反馈的强化学习）/ DPO（Direct Preference Optimization 直接偏好优化）让模型"合人意"。三者逐级收窄能力目标。

**Q3：Tokenization 分词 —— 怎么理解？**
A：LLM 不直接处理原始文本，而是先将文本拆分为 **Token**（子词单元）。
BPE 是当前主流的分词算法，GPT 系列和 LLaMA 均使用。
**核心思想：** 从字符级别开始，反复合并出现频率最高的相邻 token 对，直到达到目标词表大小。

**Q4：Pre-training 预训练 —— 怎么理解？**
A：预训练阶段让模型学习语言的基本能力。
| 目标 | 说明 | 代表模型 |
| ------ | ------ | --------- |
| Causal LM（自回归） | 预测下一个 token：P(x_t | x_1, ...

**Q5：Supervised Fine-Tuning (SFT) —— 怎么理解？**
A：SFT 使用高质量的「指令-回复」数据对模型进行微调，让模型学会遵循人类指令。
| 技巧 | 说明 |
| ------ | ------ |
| 数据质量 > 数量 | 1000 条高质量数据可能超过 10 万条低质量数据 |
| 多样性 | 覆盖多种任务类型（问答、翻译、代码、数学等） |
| 对话模板 | 使用 ChatML 等标准模板，区分 system/user/assistant |

**Q6：核心速记主线有哪些？**
A：抓住这几根：LLM 生命周期总览、Tokenization 分词、Pre-training 预训练、Supervised Fine-Tuning (SFT)、RLHF 人类反馈强化学习、Inference 推理优化。


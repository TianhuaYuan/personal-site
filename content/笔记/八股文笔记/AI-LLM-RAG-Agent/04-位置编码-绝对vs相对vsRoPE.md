---
title: "位置编码-绝对vs相对vsRoPE"
created: "2026-07-13"
tags:
  - 八股文
  - ai
---

# 位置编码：绝对 vs 相对 vs RoPE

> Self-Attention 本身是**排列不变**的（permutation invariant），即打乱输入顺序不影响输出。需要额外注入位置信息——这就是位置编码的使命。

## 正弦位置编码（绝对位置编码）

原始 Transformer 使用的正弦位置编码（Sinusoidal Positional Encoding）：

```text
PE(pos, 2i)   = sin(pos / 10000^(2i/d_model))
PE(pos, 2i+1) = cos(pos / 10000^(2i/d_model))
```

**特点：**
- 偶数维度用 sin，奇数维度用 cos
- 不同频率的正弦/余弦组合可以编码任意位置
- 理论上能泛化到训练时未见过的序列长度
- 注入方式：加法（PE + Embedding）

## 可学习绝对位置编码

BERT 和 GPT-2 使用可学习的位置嵌入：

```text
Input = Token Embedding + Learned Position Embedding
```

- 每个位置有一个可训练的向量
- 简单直接，但无法泛化到训练时未见过的长度

## 旋转位置编码 (RoPE)

现代 LLM（LLaMA、Qwen 等）普遍使用 RoPE（Rotary Position Embedding）：

**核心思想**：将位置信息编码为旋转角度，通过旋转 Q/K 向量来注入位置信息。

| 特性 | 正弦编码 | 可学习编码 | RoPE |
| ------ | --------- |----------- | ------ |
| 注入方式 | 加法 | 加法 | 旋转（修改 Q/K 向量） |
| 远距离衰减 | 无 | 无 | 天然衰减 |
| 外推能力 | 有限 | 无法外推 | 通过 NTK-aware 插值可扩展 |
| 参数开销 | 无 | 需要位置嵌入参数 | 无额外参数 |
| 使用模型 | 原始 Transformer、BERT | BERT、GPT-2 | LLaMA、GPT-NeoX、Qwen |

### RoPE 的优势

1. **相对位置感知**：通过旋转角度差来编码相对位置，天然支持相对位置
2. **远程衰减**：远距离 token 的注意力分数自然衰减，符合直觉
3. **外推能力**：通过 NTK-aware 插值或 YaRN 可以扩展到更长上下文
4. **无额外参数**：不需要学习位置嵌入向量

### RoPE 在长上下文中的应用

| 技术 | 说明 | 效果 |
| ------ | ------ | ------ |
| NTK-aware 插值 | 修改 RoPE 的 base frequency | 平滑扩展上下文长度 |
| YaRN | 结合 NTK 和注意力缩放 | 支持 128K+ 上下文 |
| Dynamic NTK | 根据实际序列长度动态调整 | 灵活适配不同长度 |

---

## 
> ▶ 对应实操：[[04-Embedding向量化原理+语义搜索场景|04-Embedding向量化原理+语义搜索场景]]

相关链接
- [[八股文学习清单]]
- [[00-全局导航|全局导航]]

---

## 快速问答

| 问题 | 参考答案 |
| ------ | --------- |
| RoPE 相比绝对位置编码的优势？ | RoPE 将位置信息编码为旋转角度，天然具有相对位置感知能力，且通过 NTK-aware 插值可以扩展到更长上下文 |
| 为什么 Transformer 需要位置编码？ | Self-Attention 是排列不变的（permutation invariant），打乱输入顺序不影响输出。需要额外注入位置信息让模型知道 token 的顺序 |
| RoPE 如何实现长上下文扩展？ | 通过 NTK-aware 插值、YaRN 等技术修改 RoPE 的 base frequency，实现平滑的上下文长度扩展 |

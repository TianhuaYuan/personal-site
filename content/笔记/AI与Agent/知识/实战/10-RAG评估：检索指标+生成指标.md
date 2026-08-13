---
title: "RAG 评估"
tags:
  - 技术学习
  - ai
  - agent
created: "2026-07-21"
---

# RAG 评估：检索指标 + 生成指标

> **一句话**：RAG 评估分两层——检索层看"找得准不准"（Recall/Precision/MRR/NDCG），生成层看"答得好不好"（Faithfulness/Relevance）。两层拆开测，才能定位问题根因。

---

### 1.1 混在一起测 = 盲人摸象

RAG 系统 = **检索器（Retriever）**检索器（Retriever） + **生成器（Generator）**生成器（Generator）。一条 Fail 的用例，可能是：

- 检索没召回对内容（检索层问题）
- LLM 没用好召回的片段（生成层问题）
- 两者都有问题

**混在一起只能得到模糊结论**混在一起只能得到模糊结论，无法指导优化方向。

### 1.2 分层评估的诊断价值

| 症状 | 检索层指标 | 生成层指标 | 问题定位 |
| --- | --- | --- | --- |
| 回答完全错误 | Recall 低 | Faithfulness 低 | 检索没召回 |
| 回答部分正确但有幻觉 | Recall 高 | Faithfulness 低 | LLM 编造 |
| 回答正确但不完整 | Recall 中 | Faithfulness 高 | TopK 太小 |
| 回答跑题 | Recall 高 | Relevance 低 | Prompt 问题 |

> [!note]
> 90% 的 RAG 问题出在检索环节。如果 Hit@3 < 0.8，先修 embedding 或 chunk 策略。

---

## 二、检索层指标详解

检索层的核心问题：**给定 query，系统能否从文档库中召回最相关的片段？**给定 query，系统能否从文档库中召回最相关的片段？

### 2.1 混淆矩阵基础

在信息检索中，我们把结果分为四类：

|  | 实际相关 | 实际不相关 |
| --- | --- | --- |
| 被系统检索 | TP（真正例） | FP（假正例） |
| 未被检索 | FN（假负例） | TN（真负例） |

### 2.2 核心检索指标

> [!note]
> 📌 **① Precision（精确率 / 查准率）**

**定义**定义：检索到的文档中，有多大比例是真正相关的？

**公式**公式：Precision = TP / (TP + FP)

**通俗理解**通俗理解：你抓回来 10 条结果，有几条是真的有用的？

**RAG 场景意义**RAG 场景意义：Precision 低 = 大量无关片段被召回，干扰大模型，易产生幻觉。必须加 Rerank 过滤。

**理想标准**理想标准：≥ 70%

---

> [!note]
> 📌 **② Recall（召回率 / 查全率）**

**定义**定义：所有相关文档中，成功检索到了多大比例？

**公式**公式：Recall = TP / (TP + FN)

**通俗理解**通俗理解：库里有 20 条相关内容，你找回来几条？

**RAG 场景意义**RAG 场景意义：Recall 低 = chunk 切割不合理、Embedding 能力弱、缺少关键词检索。

**理想标准**理想标准：≥ 90%

---

> [!note]
> 📌 **③ Precision@K / Recall@K**

**定义**定义：只看 Top-K 结果中的 Precision 和 Recall。

**RAG 场景意义**RAG 场景意义：K 通常取 3 或 5（因为 LLM 上下文有限）。**Recall@K 是 RAG 检索评估的核心指标**Recall@K 是 RAG 检索评估的核心指标。

---

> [!note]
> 📌 **④ F1-Score**

**定义**定义：Precision 和 Recall 的调和平均数。

**公式**公式：F1 = 2 × (Precision × Recall) / (Precision + Recall)

**适用场景**适用场景：需要平衡精确率和召回率时。单一指标看 F1，避免"偏科"。

**理想标准**理想标准：≥ 0.8

---

> [!note]
> 📌 **⑤ Hit@K / Hit Rate（命中率）**

**定义**定义：有多大比例的查询，在前 K 个结果中至少检索到了一个相关文档？

**公式**公式：Hit@K = 至少有一个相关文档出现在 TopK 的查询数 / 查询总数

**通俗理解**通俗理解：用户提问后，前 K 条结果里有没有"正确答案"？

**适用场景**适用场景：快速验证检索器是否有基本召回能力。

---

> [!note]
> 📌 **⑥ MRR（Mean Reciprocal Rank，平均倒数排名）**

**定义**定义：第一个相关文档的排名的倒数，取平均值。

**公式**公式：MRR = (1/|Q|) × Σ(1/rank_i)

其中 rank_i 是第 i 个查询的第一个相关文档的排名。

**通俗理解**通俗理解：第一个正确答案平均排在第几位？排第 1 得 1 分，排第 2 得 0.5 分，排第 3 得 0.33 分...

**RAG 场景意义**RAG 场景意义：问答场景中，通常第一个片段命中最重要。MRR 越高，说明正确答案越靠前。

| rank | 得分 |
| --- | --- |
| 1 | 1.0 |
| 2 | 0.5 |
| 3 | 0.33 |
| 4 | 0.25 |
| 5 | 0.2 |

---

> [!note]
> 📌 **⑦ NDCG（Normalized Discounted Cumulative Gain，归一化折损累积增益）**

**定义**定义：综合衡量排序质量，考虑相关性的"分级"（不是简单的是/否）。

**计算步骤**计算步骤：

1. **CG@K**CG@K（Cumulative Gain）：前 K 个结果的相关性得分累加
2. **DCG@K**DCG@K（Discounted CG）：排名越靠后，权重越低（折损）
- 公式：DCG@K = Σ((2^rel_i - 1) / log2(i + 1))
1. **IDCG@K**IDCG@K（Ideal DCG）：最优排序下的 DCG
2. **NDCG@K**NDCG@K = DCG@K / IDCG@K

**适用场景**适用场景：搜索结果需要分级相关性（如：不相关=0，部分相关=1，完全相关=2）。

> [!note]
> 💡 **注意**注意：如果只标相关/不相关（0/1），NDCG 退化为简单的排序评估。

---

### 2.3 检索指标对比速查表

| 指标 | 关注重点 | 适用场景 | 公式复杂度 | 是否需要分级标注 |
| --- | --- | --- | --- | --- |
| Precision | 结果纯度 | 控制噪声传入 LLM | 简单 | 否 |
| Recall | 结果覆盖 | 确保不遗漏关键信息 | 简单 | 否 |
| F1 | 平衡性 | 综合评估检索能力 | 简单 | 否 |
| Hit@K | 是否有正确答案 | 快速验证召回能力 | 简单 | 否 |
| MRR | 正确答案位置 | 问答场景 | 简单 | 否 |
| NDCG | 排序质量 | 搜索结果排序 | 复杂 | 是 |

---

### 2.4 检索常见问题诊断

| 指标异常 | 可能原因 | 优化方向 |
| --- | --- | --- |
| Recall 低 | chunk 过大/过小、Embedding 弱、无关键词检索 | 调 chunk_size、换 Embedding 模型、加 BM25 混合检索 |
| Precision 低 | 大量无关片段召回 | 加 Rerank、调相似度阈值、优化 query 改写 |
| MRR 低 | 正确答案排名靠后 | 换更强的 Rerank 模型、调向量检索参数 |
| Hit@3 < 0.8 | 检索器基础能力不够 | 重新选型检索方案 |

---

## 三、生成层指标详解

生成层的核心问题：**基于召回的上下文，LLM 生成的回答是否忠实、相关、有用？**基于召回的上下文，LLM 生成的回答是否忠实、相关、有用？

### 3.1 生成评估的难点

- **没有参考答案**没有参考答案：同一问题可以有多种正确回答
- **语义理解**语义理解：文字相似不代表语义正确
- **幻觉检测**幻觉检测：LLM 可能编造不存在的信息

**解决方案**解决方案：LLM-as-Judge（用大模型当裁判打分）

---

### 3.2 核心生成指标

> [!note]
> 📌 **① Faithfulness（忠实度 / 真实性）**

**定义**定义：回答中的每个陈述，是否都能被检索到的文档支持？

**通俗理解**通俗理解：LLM 有没有"胡说八道"？

**评估方式**评估方式：

1. **声明提取**声明提取：把回答拆成一个个事实声明
2. **证据匹配**证据匹配：逐个检查声明是否能在 context 中找到支持
3. **打分**打分：支持的声明数 / 总声明数

**RAG 场景意义**RAG 场景意义：**最核心的生成指标**最核心的生成指标。回答可以不完整，但不能编造。

**理想标准**理想标准：≥ 0.9（生产环境要求幻觉率 ≤ 5%）

> [!note]
> Faithfulness 是 RAG 的第一生成指标。优先级：Faithfulness > Answer Relevance > Context Utilization。

---

> [!note]
> 📌 **② Answer Relevance（答案相关性）**

**定义**定义：回答是否贴合用户的问题？有没有答非所问？

**评估方式**评估方式：

1. 把回答反向生成问题（Reverse Query Generation）
2. 比较生成的问题和原始问题的语义相似度
3. 或者用 LLM 直接判断"这个回答是否解决了用户的问题"

**RAG 场景意义**RAG 场景意义：检测"跑题"现象——虽然用了正确的文档，但回答方向偏了。

---

> [!note]
> 📌 **③ Answer Correctness（答案正确性）**

**定义**定义：回答与参考答案（Ground Truth）的一致性。

**评估方式**评估方式：

- **语义相似度**语义相似度：用 Embedding 计算回答与参考答案的相似度
- **LLM 判断**LLM 判断：让 LLM 对比回答和参考答案，给出 0~1 的分数

**适用场景**适用场景：有参考答案的测试集（如考试题、FAQ）。

> [!note]
> 💡 **注意**注意：无参考评估时不能用这个指标，需要依赖 Faithfulness + Relevance。

---

> [!note]
> 📌 **④ Context Precision（上下文精确度）**

**定义**定义：检索到的上下文中，相关片段的比例。反映检索结果的整体相关性密度。

**RAGAS 版本**RAGAS 版本：用 LLM 判断每个片段是否与 query 相关，计算比例。

---

> [!note]
> 📌 **⑤ Context Recall（上下文召回率）**

**定义**定义：参考答案所需的参考信息，有多少被检索到了？

**评估方式**评估方式：

1. 把参考答案拆成关键事实点
2. 逐个检查这些事实点是否能在检索到的 context 中找到
3. 计算比例

---

### 3.3 生成指标对比速查表

| 指标 | 评估对象 | 是否有参考 | 核心用途 | 评估方式 |
| --- | --- | --- | --- | --- |
| Faithfulness | 回答 vs Context | 无参考 | 防幻觉 | LLM-as-Judge |
| Answer Relevance | 回答 vs Query | 无参考 | 防跑题 | LLM-as-Judge |
| Answer Correctness | 回答 vs Ground Truth | 有参考 | 测准确性 | Embedding/LLM |
| Context Precision | Context vs Query | 无参考 | 测检索纯度 | LLM-as-Judge |
| Context Recall | Context vs Ground Truth | 有参考 | 测检索覆盖 | LLM-as-Judge |

---

### 3.4 幻觉检测专项

**幻觉类型**幻觉类型：

| 类型 | 定义 | 示例 |
| --- | --- | --- |
| 事实性幻觉 | 编造不存在的事实 | "公司成立于2010年"（实际是2015年） |
| 逻辑性幻觉 | 推理过程错误 | "A>B, B>C, 所以 C>A" |
| 引用性幻觉 | 声称引用文档但文档无此内容 | "根据第3条..."（实际没有） |

**幻觉率计算**幻觉率计算：幻觉率 = 含幻觉的样本数 / 总测试样本数 × 100%

**生产要求**生产要求：幻觉率 ≤ 5%

---

### 4.1 RAGAS 简介

RAGAS（Retrieval-Augmented Generation Assessment）是 RAG 评估的行业标准框架，提供**无参考评估**无参考评估能力。

**核心指标**核心指标：

| 指标 | 类别 | 说明 |
| --- | --- | --- |
| context_precision | 检索 | 上下文精确度 |
| context_recall | 检索 | 上下文召回率 |
| faithfulness | 生成 | 忠实度 |
| answer_relevancy | 生成 | 答案相关性 |
| answer_correctness | 生成 | 答案正确性（需参考） |

### 4.2 安装与基础使用

```python
from ragas import evaluate
from ragas.metrics import (
    context_precision,
    context_recall,
    faithfulness,
    answer_relevancy,
    answer_correctness
)
from datasets import Dataset

# 构造测试数据
data = {
    "question": ["公司年假最多可以休几天？", "如何申请报销？"],
    "answer": ["正式员工每年最多休10天年假", "需要填写报销单并提交发票"],
    "contexts": [
        ["员工手册规定正式员工年假上限10天"],
        ["报销流程：填写报销单 → 提交发票 → 部门审批 → 财务打款"]
    ],
    "ground_truth": ["正式员工年假每年最多10天", "填写报销单并提交发票即可申请报销"]
}

ds = Dataset.from_dict(data)

# 执行评估
result = evaluate(
    ds,
    metrics=[
        context_precision,
        context_recall,
        faithfulness,
        answer_relevancy,
    ]
)

print(result)
#  'faithfulness': 0.88, 'answer_relevancy': 0.85}
```

### 4.3 手写评估指标计算

核心指标的纯 Python 实现：

```python
import numpy as np
from typing import List, Dict

# ========== 检索层指标 ==========

def precision_at_k(retrieved_ids: List[str], relevant_ids: List[str], k: int) -> float:
    """Precision@K：前 K 个检索结果中有多少是相关的？

    Args:
        retrieved_ids: 检索返回的文档 ID 列表（已排序）
        relevant_ids: 标注的相关文档 ID 列表
        k: 只看前 K 个结果

    Returns:
        精确率，范围 [0, 1]
    """
    retrieved_k = retrieved_ids[:k]                          # 取前 K 个
    tp = len(set(retrieved_k) & set(relevant_ids))           # 相关且被检索 = 真阳性
    return tp / k if k > 0 else 0.0                          # 除以 K 归一化


def recall_at_k(retrieved_ids: List[str], relevant_ids: List[str], k: int) -> float:
    """Recall@K：所有相关文档中，有多少被检索到了前 K？

    Args:
        retrieved_ids: 检索返回的文档 ID 列表
        relevant_ids: 标注的相关文档 ID 列表
        k: 只看前 K 个结果

    Returns:
        召回率，范围 [0, 1]
    """
    retrieved_k = retrieved_ids[:k]                          # 取前 K 个
    tp = len(set(retrieved_k) & set(relevant_ids))           # 相关且被检索
    return tp / len(relevant_ids) if relevant_ids else 0.0   # 除以总相关数


def hit_at_k(retrieved_ids: List[str], relevant_ids: List[str], k: int) -> float:
    """Hit@K：前 K 个里是否至少命中一个相关文档？（二元的）

    Returns:
        1.0 表示命中，0.0 表示未命中
    """
    retrieved_k = set(retrieved_ids[:k])
    return 1.0 if retrieved_k & set(relevant_ids) else 0.0   # 有交集即命中


def mrr(retrieved_ids: List[str], relevant_ids: List[str]) -> float:
    """MRR（Mean Reciprocal Rank）：第一个相关文档排名的倒数

    排第 1 得 1.0，排第 2 得 0.5，排第 3 得 0.33...
    """
    for rank, doc_id in enumerate(retrieved_ids, start=1):
        if doc_id in relevant_ids:                           # 找到第一个相关文档
            return 1.0 / rank                                # 返回其排名倒数
    return 0.0                                               # 没找到 → 0


def f1_score(precision: float, recall: float) -> float:
    """F1-Score：精确率和召回率的调和平均"""
    if precision + recall == 0:
        return 0.0
    return 2 * (precision * recall) / (precision + recall)


# ========== 生成层指标（LLM-as-Judge 简化版） ==========

def faithfulness_simple(answer: str, context_chunks: List[str]) -> float:
    """忠实度简化计算：检查回答中的句子能否在上下文中找到支撑

    生产环境建议用 RAGAS 的 faithfulness（基于声明提取+证据匹配），
    这里给一个简化版说明原理。

    Args:
        answer: LLM 生成的回答
        context_chunks: 检索到的上下文片段列表

    Returns:
        忠实度分数，范围 [0, 1]——有支撑的句子占比
    """
    import re
    # 按句号/分号/换行拆分回答为独立声明
    claims = [s.strip() for s in re.split(r'[。；\n]', answer) if len(s.strip()) > 5]
    if not claims:
        return 0.0

    context_text = ' '.join(context_chunks)                  # 拼接所有上下文
    supported = 0
    for claim in claims:
        # 简化判断：声明中的关键词是否在上下文中出现
        # 生产环境应用 embedding 相似度或 LLM 逐条判断
        keywords = [w for w in claim if len(w) >= 2]         # 取长度≥2的词
        match_count = sum(1 for kw in keywords if kw in context_text)
        if len(keywords) > 0 and match_count / len(keywords) > 0.3:
            supported += 1                                   # 关键词覆盖超 30% → 有支撑

    return supported / len(claims)                           # 有支撑声明占比


# ========== 使用示例 ==========
if __name__ == "__main__":
    # 模拟一次检索
    retrieved = ["doc_3", "doc_1", "doc_7", "doc_2", "doc_5"]  # 检索返回的排序
    relevant = ["doc_1", "doc_2", "doc_9"]                     # 人工标注的相关文档

    k = 3
    print(f"Precision@{k}: {precision_at_k(retrieved, relevant, k):.3f}")
    print(f"Recall@{k}:    {recall_at_k(retrieved, relevant, k):.3f}")
    print(f"Hit@{k}:       {hit_at_k(retrieved, relevant, k):.3f}")
    print(f"MRR:           {mrr(retrieved, relevant):.3f}")
    print(f"F1:            {f1_score(precision_at_k(retrieved, relevant, k),
                                      recall_at_k(retrieved, relevant, k)):.3f}")

    # 模拟生成评估
    answer = "正式员工每年最多休10天年假。需要填写报销单并提交发票。"
    contexts = [
        "员工手册规定正式员工年假上限10天。",
        "报销流程：填写报销单 → 提交发票 → 部门审批。",
    ]
    print(f"Faithfulness:  {faithfulness_simple(answer, contexts):.3f}")
```

---

### Pipeline 架构

> [!note]
> 📈 `[Mermaid 图表]
> flowchart TD
>     A[测试数据集<br/>question + ground_truth + reference_chunks] --> B[检索模块]
>     B --> C[检索层指标<br/>Recall@K / Precision@K / MRR / NDCG]
>     B --> D[召回的 Contexts]
>     D --> E[生成模块<br/>LLM生成Answer]
>     E --> F[生成层指标<br/>Faithfulness / Relevance / Correctness]
>     C --> G[评估报告]
>     F --> G
>     G --> H{指标达标?}
>     H -->|Yes| I[通过评估]
>     H -->|No| J[诊断根因]
>     J --> K[优化检索 / 优化生成]
>     K --> A`

---

### 行业参考标准

| 指标 | 合格线 | 优秀线 | 说明 |
| --- | --- | --- | --- |
| Recall@K | ≥ 80% | ≥ 90% | 关键信息不能漏 |
| Precision@K | ≥ 60% | ≥ 75% | 噪声控制 |
| F1-Score | ≥ 0.7 | ≥ 0.85 | 综合平衡 |
| Hit@3 | ≥ 0.7 | ≥ 0.85 | 基础召回能力 |
| MRR | ≥ 0.6 | ≥ 0.8 | 正确答案位置 |
| Faithfulness | ≥ 0.85 | ≥ 0.95 | 防幻觉核心 |
| Answer Relevance | ≥ 0.8 | ≥ 0.9 | 贴合问题 |
| Hallucination Rate | ≤ 10% | ≤ 5% | 越低越好 |

### 诊断矩阵

> [!note]
> 📈 `[Mermaid 图表]
> quadrantChart
>     title RAG 问题诊断四象限（Recall vs Faithfulness）
>     x-axis Low Recall --> High Recall
>     y-axis Low Faithfulness --> High Faithfulness
>     quadrant-1 检索强 + 生成弱：调 Prompt / 换模型
>     quadrant-2 双高：理想状态，保持监控
>     quadrant-3 双低：先修检索，再调生成
>     quadrant-4 检索弱 + 生成强：优化 Embedding / Chunk / 检索策略`

| 象限 | Recall | Faithfulness | 问题定位 | 优化优先级 |
| --- | --- | --- | --- | --- |
| 第一象限 | 高 | 低 | LLM 没用好文档 / Prompt 问题 | 调 Prompt、加约束 |
| 第二象限 | 高 | 高 | 理想状态 | 保持监控 |
| 第三象限 | 低 | 低 | 检索器根本不行 | 换检索方案 |
| 第四象限 | 低 | 高 | 检索召回不够 | 调 chunk、加混合检索 |

---

---

## 八、参考资料

| 来源 | 链接 | 内容 |
| --- | --- | --- |
| CSDN - RAG测评系统全链路搭建指南 | https://blog.csdn.net/m0_58868237/article/details/162827441 | 检索+生成指标详解与代码实现 |
| CSDN - RAG评估指标理论篇 | https://blog.csdn.net/ngadminq/article/details/147857146 | 指标公式与定义 |
| CSDN - RAG项目完整效果评估体系 | https://blog.csdn.net/2501_92990656/article/details/162809923 | 三层评估体系与 RAGAS 实战 |
| CSDN - RAGAS 框架详解 | https://blog.csdn.net/u013538542/article/details/158312178 | RAGAS 指标原理与使用 |
| 51CTO - LLM-as-Judge 实现 | https://www.51cto.com/article/816568.html | LLM 评估系统实践 |
| RAGAS 官方文档 | https://docs.ragas.io/ | 官方指标定义与 API |

---

> [!note]
> 💡 **关联笔记**关联笔记：

> [!note]
> 💡 - [[08-混合检索：向量+BM25+RRF融合]]

> [!note]
> 💡 - [[07-分块策略：chunk_sizeoverlap选择、语义分块]]

> [!note]
> 💡 - [[AI与Agent/知识/八股/19-Query改写与多轮对话指代消解]]

##
> ▶ 对应原理：[[10-LLM幻觉与可靠性|10-LLM幻觉与可靠性]]


> ▶ 对应原理：[[22-RAG评估指标|22-RAG评估指标]]


## 速记卡（面试闪卡）

**Q1：一句话讲清「RAG 评估：检索指标 + 生成指标」到底是什么？**
A：RAG 评估分两层：检索层看找得准不准，生成层看答得好不好，分层才能定位根因。

**Q2：一、为什么分两层 —— 怎么理解？**
A：RAG = 检索器 + 生成器。答错可能是没召回对内容，或 LLM 没用好片段。混测像盲人摸象，只知"不行"不知改哪。分开测：Recall 低修检索，Faithfulness 低管幻觉。

**Q3：二、检索层指标 —— 怎么理解？**
A：Precision 看抓回的有用比例（低了加 Rerank），Recall 看相关找回多少（低了调 chunk/换 Embedding），MRR 看正确答案排第几，NDCG 看排序质量。90% 的 RAG 问题出在检索。

**Q4：三、生成层指标 —— 怎么理解？**
A：Faithfulness（忠实度）是头号指标：每句话能否在 context 找到支撑，可不完整但不能编。Answer Relevance 防跑题，Answer Correctness 需参考答案。无参考时用 LLM-as-Judge 打分。

**Q5：四、RAGAS 与评估流水线 —— 怎么理解？**
A：RAGAS 是行业标准，提供无参考评估（context_precision/recall、faithfulness、answer_relevancy）。Pipeline：测试集→检索指标→生成指标→报告→不达标诊断根因。合格线：Recall@K≥80%、幻觉率≤5%。

**Q6：核心速记主线有哪些？**
- 分层：检索层 + 生成层
- 检索：Precision/Recall/MRR/NDCG
- 生成：Faithfulness 为首，防幻觉
- RAGAS 无参考；Recall@K≥80%、幻觉≤5%

**口诀**
A：RAG 评估分两层，召回生成各问症；
检索不准先去整，忠实不编是死坑。
达标线记心间，幻觉五个点莫轻；
分层定位好下手，优化不偏方向明。

相关链接


---

→ [[技术学习路线图#RAG]]
## 相关链接

- [[笔记/AI与Agent/知识/实战/13-RAG参数调优：网格搜索实验框架|RAG 参数调优]]
- [[笔记/AI与Agent/知识/实战/37-Multi-Agent协作模式|Multi-Agent 协作模式]]
- [[笔记/AI与Agent/知识/实战/40-LLM-as-Judge评测工具链：Ragas-DeepEval-Langfuse配置与接入|LLM-as-Judge评测工具链：Ragas / DeepEval / Langfuse 配置与接入]]
- [[笔记/AI与Agent/知识/实战/36-Reflexion：带自我反思的Agent|Reflexion：带自我反思的Agent]]
- [[笔记/AI与Agent/知识/实战/11-AgenticRAG：LLM主动决策多次检索vs传统单次被动检索|Agentic RAG：LLM 主动决策多次检索 vs 传统单次被动检索]]

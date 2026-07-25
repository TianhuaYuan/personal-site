---
title: "RAG 参数调优"
tags:
  - 技术学习
  - ai
  - agent
created: "2026-07-21"
---

# RAG 参数调优：网格搜索实验框架

> **一句话**：RAG 参数调优不是"炼丹"，而是科学实验——先定义参数空间和评估指标，再用网格搜索自动化遍历组合，最终用数据驱动选最优配置。

---

## 一、为什么 RAG 需要参数调优？

### 1.1 RAG 有多少参数可调？

一个典型的 RAG 系统涉及**三类可调参数**三类可调参数，组合起来搜索空间爆炸：

| 参数类别 | 具体参数 | 典型取值范围 |
| --- | --- | --- |
| 分块参数 | chunk_size | 128 / 256 / 512 / 1024 / 2048 |
|  | chunk_overlap | 0 / 64 / 128 / 256 |
|  | 分块策略 | 递归切分 / 语义分块 / 结构化 |
| 检索参数 | top_k | 1 / 2 / 3 / 5 / 8 / 10 |
|  | 检索类型 | 纯向量 / 纯BM25 / 混合检索 |
|  | rerank | 无 / bge-reranker / cohere |
|  | similarity_threshold | 0.5 / 0.6 / 0.7 / 0.8 |
| 生成参数 | temperature | 0 / 0.3 / 0.7 |
|  | LLM 模型 | GPT-4o / Claude / 开源模型 |

> [!note]
> 💡 **关键洞察**关键洞察：仅 chunk_size(5) × top_k(5) × 检索类型(3) = **75 种组合**75 种组合。加上 rerank、embedding 模型，组合数轻松破百。手动调参 = 在 Excel 里记几百行数据然后凭感觉选，这不科学。

### 1.2 调优的核心原则

> [!note]
> RAG 调优的核心原则——先测检索，再调生成。90% 的效果问题出在检索环节。

调优顺序必须遵循：

> [!note]
> 📈 `[Mermaid 图表]
> flowchart TD
>     A[确定评估基准线<br/>Baseline] --> B[第一阶段：优化分块策略]
>     B --> C[第二阶段：优化检索参数]
>     C --> D[第三阶段：优化生成参数]
>     D --> E{指标达标?}
>     E -->|Yes| F[锁定配置，上线监控]
>     E -->|No| G[回退到最弱环节重新调]
>     G --> B`

---

## 二、分块参数调优

### 2.1 chunk_size 的黄金区间

chunk_size 是 RAG 调优中**影响最大的单一参数**影响最大的单一参数，没有之一。

| chunk_size | 优势 | 劣势 | 适用场景 |
| --- | --- | --- | --- |
| 128~256 | 检索精度极高，噪声少 | 上下文碎片化，语义不完整 | 精确问答、FAQ、事实检索 |
| 256~512 | 精度与上下文的最佳平衡点 | — | 大多数 RAG 场景的首选 |
| 512~1024 | 上下文完整，适合长答案 | 可能引入噪声，检索精度下降 | 报告生成、综合分析 |
| 1024~2048 | 信息覆盖最全 | 噪声严重，LLM 注意力分散 | 探索性搜索、知识浏览 |

> [!note]
> 💡 **经验法则**经验法则：chunk_size 优先从 **512**512 开始试，这是大多数 Embedding 模型的最佳输入范围。

### 2.2 chunk_overlap 的选择

| overlap | 效果 | 建议场景 |
| --- | --- | --- |
| 0 | 无重叠，无冗余 | chunk_size 小 + 文档结构清晰 |
| 10%~20% | 最佳平衡点 | 大多数场景的默认选择 |
| >30% | 冗余严重，检索结果高度重复 | 仅当 chunk_size 极大时考虑 |

**经验公式**经验公式：`overlap ≈ chunk_size × 0.15 ~ 0.2`overlap ≈ chunk_size × 0.15 ~ 0.2

例如 chunk_size=512 → overlap=64~128

### 2.3 分块策略对比

| 策略 | 原理 | 优势 | 劣势 | 推荐场景 |
| --- | --- | --- | --- | --- |
| 递归切分 | 按分隔符层级递归切分 | 简单高效，保留语义边界 | 不考虑语义连贯性 | 默认首选 |
| 语义分块 | 基于句子 embedding 相似度断点 | 语义完整性最强 | 计算成本高 | 高精度问答 |
| 结构化分块 | 按文档标题/章节结构切分 | 保留文档层次结构 | 依赖文档格式清晰 | 技术文档/法律文本 |
| 句子窗口 | 单句建索引，命中后扩窗 | 检索精度 + 丰富上下文 | 实现复杂 | 长文档问答 |

---

## 三、检索参数调优

### 3.1 top_k 的选择逻辑

top_k 控制传给 LLM 的上下文数量，直接影响**召回率 vs. 噪声**召回率 vs. 噪声的平衡。

| top_k | 召回率 | 噪声量 | Token 消耗 | 适用场景 |
| --- | --- | --- | --- | --- |
| 1~2 | 低 | 极低 | 少 | 简单事实性问答 |
| 3~5 | 中高 | 可控 | 适中 | 大多数场景的最佳区间 |
| 5~8 | 高 | 较高 | 多 | 复杂分析、多方面问题 |
| 8~10 | 很高 | 高 | 很多 | 探索性搜索、研究调研 |

> [!note]
> 💡 **注意**注意：top_k 不是越大越好。传给 LLM 太多无关内容反而会降低生成质量（"Lost in the Middle" 效应——LLM 对中间位置的信息关注度最低）。

### 3.2 检索类型选择

| 检索类型 | 原理 | 擅长 | 不擅长 |
| --- | --- | --- | --- |
| 纯向量检索 | embedding 余弦相似度 | 语义相近的查询 | 精确专有名词、编号 |
| 纯 BM25 | 关键词词频匹配 | 精确术语、编号、代码 | 语义相近但表述不同 |
| 混合检索 + RRF | 向量 + BM25 + 倒数排名融合 | 兼顾语义和精确匹配 | — |

> [!note]
> 混合检索是生产环境的标配。纯向量检索在专有名词、编号、代码等精确匹配场景下效果差，加上 BM25 后 Hit@3 通常能提升 15%~25%。

### 3.3 Rerank 的价值

| 配置 | Precision@5 | Faithfulness | 说明 |
| --- | --- | --- | --- |
| 无 Rerank | 0.55 | 0.72 | 基线 |
| 加 Rerank | 0.78 | 0.91 | Precision 提升 42%，幻觉率大幅下降 |

**Rerank 的本质**Rerank 的本质：检索是"粗筛"（从万级文档中选百级），Rerank 是"精排"（从百级中选十级给 LLM）。跳过 Rerank = 把噪声直接喂给 LLM。

---

## 四、多阶段网格搜索实验框架

### 4.1 什么是网格搜索？

网格搜索（Grid Search）的核心思想：**穷举所有参数组合**穷举所有参数组合，每种组合跑一遍评估，选出最优。

> [!note]
> 📈 `[Mermaid 图表]
> flowchart LR
>     A[定义参数空间] --> B[生成所有组合]
>     B --> C[逐个跑评估]
>     C --> D[收集指标数据]
>     D --> E[排序选最优]`

### 4.2 为什么要分阶段？

**全部参数一起网格搜索 = 组合爆炸。**全部参数一起网格搜索 = 组合爆炸。

例子：5 个 chunk_size × 4 个 top_k × 3 种检索 × 2 种 rerank × 3 种 embedding = **360 种组合**360 种组合。每种跑 50 条测试用例 = 18000 次 LLM 调用。

**分阶段调优**分阶段调优 = 每阶段只调 1~2 个参数，大幅缩减搜索空间：

> [!note]
> 📈 `[Mermaid 图表]
> flowchart TD
>     S[开始] --> P1[第一阶段<br/>只调 chunk_size × overlap]
>     P1 --> P1E[固定其他参数<br/>跑网格搜索]
>     P1E --> P1R[锁定最优分块配置]
>     P1R --> P2[第二阶段<br/>只调 top_k × 检索类型]
>     P2 --> P2E[固定其他参数<br/>跑网格搜索]
>     P2E --> P2R[锁定最优检索配置]
>     P2R --> P3[第三阶段<br/>只调 rerank × similarity_threshold]
>     P3 --> P3E[固定其他参数<br/>跑网格搜索]
>     P3E --> P3R[锁定最优 Rerank 配置]
>     P3R --> P4[第四阶段<br/>微调生成参数 temperature]
>     P4 --> E[最终最优配置]`

| 阶段 | 调优参数 | 固定参数 | 组合数 |
| --- | --- | --- | --- |
| 第一阶段 | chunk_size × overlap | top_k=5, 向量检索, 无 Rerank | 5×3 = 15 |
| 第二阶段 | top_k × 检索类型 | 最优分块, 无 Rerank | 4×3 = 12 |
| 第三阶段 | rerank × threshold | 最优分块 + 检索 | 3×4 = 12 |
| 第四阶段 | temperature | 全部最优 | 3×1 = 3 |
| 总计 | — | — | 42 |

从 360 缩减到 42，效率提升 **8.5 倍**8.5 倍。

---

### 4.3 完整实验代码框架

```python
import itertools
import numpy as np
from dataclasses import dataclass
from typing import List, Dict, Any

@dataclass
class ExperimentConfig:
    """一组参数配置"""
    chunk_size: int
    chunk_overlap: int
    top_k: int
    retrieval_type: str  # "vector", "bm25", "hybrid"
    use_rerank: bool
    rerank_model: str = ""
    similarity_threshold: float = 0.5

@dataclass
class ExperimentResult:
    """一组实验结果"""
    config: ExperimentConfig
    hit_rate: float
    precision: float
    recall: float
    faithfulness: float
    answer_relevance: float
    latency_ms: float

class RAGGridSearch:
    def __init__(self, documents, test_suite, embed_model, llm):
        self.documents = documents
        self.test_suite = test_suite
        self.embed_model = embed_model
        self.llm = llm
        self.results: List[ExperimentResult] = []

    def build_index(self, config: ExperimentConfig):
        """根据配置构建索引"""
        # 1. 分块
        splitter = RecursiveCharacterTextSplitter(
            chunk_size=config.chunk_size,
            chunk_overlap=config.chunk_overlap
        )
        chunks = splitter.split_documents(self.documents)

        # 2. 建索引
        if config.retrieval_type == "vector":
            index = VectorStore.from_documents(chunks, self.embed_model)
        elif config.retrieval_type == "bm25":
            index = BM25Index.from_documents(chunks)
        else:  # hybrid
            index = HybridSearchIndex.from_documents(chunks, self.embed_model)

        return index

    def run_single_experiment(self, config: ExperimentConfig) -> ExperimentResult:
        """跑单组实验"""
        # 1. 建索引
        index = self.build_index(config)

        # 2. 逐条测试
        hits, precisions, recalls = [], [], []
        faith_scores, rel_scores, latencies = [], [], []

        for case in self.test_suite:
            import time
            t0 = time.time()

            # 检索
            retrieved = index.retrieve(case["question"], top_k=config.top_k)

            # 可选 Rerank
```

```python

            if config.use_rerank:
                retrieved = rerank(retrieved, case["question"], model=config.rerank_model)

            # 过滤低分
            retrieved = [r for r in retrieved if r.score >= config.similarity_threshold]

            latency = (time.time() - t0) * 1000

            # 检索指标
            retrieved_ids = {r.id for r in retrieved}
            relevant_ids = set(case["relevant_chunk_ids"])
            tp = len(retrieved_ids & relevant_ids)
            hits.append(1 if tp > 0 else 0)
            precisions.append(tp / max(len(retrieved_ids), 1))
            recalls.append(tp / max(len(relevant_ids), 1))

            # 生成 + 生成指标
            answer = self.llm.generate(case["question"], [r.text for r in retrieved])
            faith = evaluate_faithfulness(case["question"], [r.text for r in retrieved], answer)
            rel = evaluate_relevance(case["question"], answer)
            faith_scores.append(faith)
            rel_scores.append(rel)
            latencies.append(latency)

        return ExperimentResult(
            config=config,
            hit_rate=np.mean(hits),
            precision=np.mean(precisions),
            recall=np.mean(recalls),
            faithfulness=np.mean(faith_scores),
            answer_relevance=np.mean(rel_scores),
            latency_ms=np.mean(latencies)
        )

    def run_grid_search(self, param_grid: Dict[str, List]) -> List[ExperimentResult]:
        """网格搜索"""
        # 生成所有组合
        keys = list(param_grid.keys())
        values = list(param_grid.values())
        combinations = list(itertools.product(*values))

        print(f"共 {len(combinations)} 种组合待测试")

        for i, combo in enumerate(combinations):
            config = ExperimentConfig(**dict(zip(keys, combo)))
            print(f"\n[{i+1}/{len(combinations)}] chunk={config.chunk_size}, "
                  f"overlap={config.chunk_overlap}, top_k={config.top_k}, "
                  f"retrieval={config.retrieval_type}, 
```

```python
rerank={config.use_rerank}")

            result = self.run_single_experiment(config)
            self.results.append(result)
            print(f"  → Hit={result.hit_rate:.2f}, P={result.precision:.2f}, "
                  f"R={result.recall:.2f}, Faith={result.faithfulness:.2f}")

        return self.results

    def find_best(self, metric: str = "faithfulness") -> ExperimentResult:
        """找最优配置"""
        return max(self.results, key=lambda r: getattr(r, metric))

# ========== 使用示例 ==========
if __name__ == "__main__":
    # 第一阶段：只调分块参数
    grid_phase1 = {
        "chunk_size": [256, 512, 1024],
        "chunk_overlap": [0, 64, 128],
        "top_k": [5],
        "retrieval_type": ["vector"],
        "use_rerank": [False],
        "similarity_threshold": [0.5],
    }

    searcher = RAGGridSearch(documents, test_suite, embed_model, llm)
    results = searcher.run_grid_search(grid_phase1)
    best = searcher.find_best("recall")

    print(f"\n最优配置: chunk_size={best.config.chunk_size}, "
          f"overlap={best.config.chunk_overlap}")
    print(f"Recall={best.recall:.2f}, Precision={best.precision:.2f}")
```

---

## 五、LlamaIndex ParamTuner 实战

LlamaIndex 提供了内置的 `ParamTuner`ParamTuner，可以直接用于 RAG 参数网格搜索。

### 5.1 基本用法

```python
from llama_index.experimental.param_tuner import ParamTuner

# 定义参数网格
param_dict = {
    "chunk_size": [256, 512, 1024],
    "top_k": [1, 2, 5],
}

# 固定参数
fixed_param_dict = {
    "docs": docs,
    "eval_qs": eval_qs[:10],
    "ref_response_strs": ref_response_strs[:10],
}

# 创建调优器
param_tuner = ParamTuner(
    param_fn=objective_function,  # 评估函数
    param_dict=param_dict,
    fixed_param_dict=fixed_param_dict,
    show_progress=True,
)

# 运行网格搜索
results = param_tuner.tune()

# 输出最优结果
best = results.best_run_result
print(f"最优 Score: {best.score}")
print(f"最优 chunk_size: {best.params['chunk_size']}")
print(f"最优 top_k: {best.params['top_k']}")
```

### 5.2 Ray Tune 分布式调优

当参数组合特别多时，可以用 Ray Tune 做分布式并行搜索：

```python
from llama_index.experimental.param_tuner import RayTuneParamTuner

param_tuner = RayTuneParamTuner(
    param_fn=objective_function,
    param_dict=param_dict,
    fixed_param_dict=fixed_param_dict,
)
results = param_tuner.tune()
```

---

## 六、调优结果分析

### 6.1 结果可视化表格

假设我们完成了第一阶段（分块参数）的网格搜索：

| chunk_size | overlap | Hit@5 | Precision@5 | Recall@5 | F1 |
| --- | --- | --- | --- | --- | --- |
| 256 | 0 | 0.72 | 0.68 | 0.58 | 0.63 |
| 256 | 64 | 0.78 | 0.72 | 0.65 | 0.68 |
| 512 | 128 | 0.88 | 0.80 | 0.82 | 0.81 |
| 512 | 256 | 0.86 | 0.74 | 0.84 | 0.79 |
| 1024 | 128 | 0.82 | 0.60 | 0.88 | 0.71 |
| 1024 | 256 | 0.84 | 0.55 | 0.90 | 0.68 |

**分析**分析：

- chunk_size=256 时 Recall 明显不足（信息碎片化）
- chunk_size=1024 时 Precision 低（噪声太多）
- **chunk_size=512 + overlap=128 是最优平衡点**chunk_size=512 + overlap=128 是最优平衡点，F1 最高

### 6.2 多阶段调优结果汇总

> [!note]
> 📈 `[Mermaid 图表]
> flowchart LR
>     subgraph Phase1[第一阶段：分块]
>         A1[256+0 → F1=0.63]
>         A2[512+128 → F1=0.81 ✅]
>         A3[1024+256 → F1=0.68]
>     end
>     subgraph Phase2[第二阶段：检索]
>         B1[vector+k3 → Hit=0.82]
>         B2[hybrid+k5 → Hit=0.91 ✅]
>         B3[bm25+k5 → Hit=0.75]
>     end
>     subgraph Phase3[第三阶段：Rerank]
>         C1[无Rerank → Faith=0.78]
>         C2[bge-reranker → Faith=0.93 ✅]
>     end
>     Phase1 --> Phase2 --> Phase3`

---

## 七、生产环境调优 Checklist

### 7.1 调优前必做

- [ ] 准备**标准化测试集**标准化测试集（至少 50 条，覆盖正常/边缘/恶意 query）
- [ ] 确定评估指标优先级（Recall 优先还是 Precision 优先？）
- [ ] 建立基线配置（Baseline）
- [ ] 确定参数搜索空间（不要盲目扩大）

### 7.2 调优中注意

- [ ] 每阶段**只调 1~2 个参数**只调 1~2 个参数，控制变量
- [ ] 每组实验至少跑 **3 次取均值**3 次取均值（LLM 输出有随机性）
- [ ] 记录每次实验的**完整配置和指标**完整配置和指标（用 DataFrame 或 CSV）
- [ ] 注意 **Token 成本**Token 成本（评估 100 组 × 50 条 = 大量 API 调用）

### 7.3 调优后验证

- [ ] 最优配置在**测试集之外**测试集之外的数据上也验证通过
- [ ] 关注**延迟指标**延迟指标（调优后的配置不能太慢）
- [ ] 纳入 **CI/CD 回归测试**CI/CD 回归测试（防止后续改动劣化）
- [ ] 灰度上线，收集**线上真实反馈**线上真实反馈

---

---

---

## 八、参考资料

| 来源 | 链接 | 内容 |
| --- | --- | --- |
| CSDN - RAGs性能调优实战 | https://blog.csdn.net/gitblog_00658/article/details/151743762 | Chunk Size 与 Top-K 调优策略 |
| CSDN - ParamTuner 超参数优化 | https://blog.csdn.net/ppoojjj/article/details/140310019 | LlamaIndex ParamTuner 网格搜索 |
| CSDN - RAG评估与优化指南 | https://blog.csdn.net/zhangzhentiyes/article/details/148527841 | RAGAS/ARES 评估框架实战 |
| CSDN - RAG 文本切分方法 | https://juejin.cn/post/7545087762838126630 | 分块策略全面对比 |
| LlamaIndex 官方文档 | https://docs.llamaindex.io/ | ParamTuner / RayTuneParamTuner |
| RAGAS 官方文档 | https://docs.ragas.io/ | RAG 评估指标与框架 |

---

> [!note]
> 💡 **关联笔记**关联笔记：

> [!note]
> 💡 - [[10-RAG 评估：检索指标 + 生成指标]]

> [!note]
> 💡 - [[07-分块策略：chunk_size overlap 选择、语义分块]]

> [!note]
> 💡 - [[08-混合检索：向量 + BM25 + RRF 融合]]

> [!note]
> 💡 - [[09-Query 改写 + 多轮对话指代消解]]

## 相关链接

- 项目实践：[[笔记/技术学习/项目开发笔记/ai-resume笔记/01_技术研读/02_RAG流水线.md#第 10 层：参数化实验版|ai-resume: RAG流水线]]

---

→ [[技术学习清单#RAG]]

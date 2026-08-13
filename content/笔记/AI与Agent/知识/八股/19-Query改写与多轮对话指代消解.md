---

title: "Query改写与多轮对话指代消解"

created: "2026-07-21"

tags:

  - 八股文

  - ai

source: "AI学习备份迁移"

---

# Query改写与多轮对话指代消解

## Query 改写 + 多轮对话指代消解

> **适用场景**：RAG 系统查询预处理阶段 | **你的角色**：AI Agent 应用开发工程师（求职核心考点）

>

>

>

> **一句话总结**：用户说话像打哑谜——"它""那个""这个"满天飞，Query 改写就是那个帮你把哑谜翻译成标准题目的"翻译官"。

>

---

## 一、为什么需要 Query 改写？——"打哑谜" analogy

想象你是一个图书管理员，有人跑来问：

**第一轮**："你们会员等级怎么分的？"

**第二轮**："**它**具体有哪些权益？"

如果你只听到第二句，"它"是什么？会员？手机？套餐？完全不知道。

这就是多轮对话中 RAG 系统面临的**核心挑战**：用户的追问依赖于前文上下文，但检索系统只看到当前这一句话，像在听一个人突然冒出一句"它多少钱"——完全懵逼。

**三类典型问题：**

| 问题类型 | 举例 | 后果 |

| --- | --- | --- |

| **指代不明** | "它具体有哪些权益" | 检索系统不知道"它"指什么，召回完全不相关内容 |

| **口语化 vs 书面化** | "东西不想要了咋弄" vs 知识库写的是"7天内可申请无理由退货" | 语义相同但用词完全不同，向量相似度很低 |

| **复合意图** | "给公司采购办公用品有什么优惠？怎么付款和开发票？" | 一个问题包含多个子需求，需要拆解后分别检索 |

> 💡 **核心金句**："Query 改写解决的是'用户怎么问'和'知识库怎么写'之间的语义鸿沟。不改写，检索就是在猜。"

>

---

## 二、指代消解（Coreference Resolution）——"把'它'替换成真身"
### 2.1 核心思想

将最近几轮对话历史与当前问题一起交给 LLM，重构为一个**独立完整的、无需上下文也能理解**的问题。

### 2.2 实现原理

```text

用户第1轮: "你们会员等级怎么分的？"

AI: "分为普通、银卡、金卡、黑金四个等级"

用户第2轮: "它具体有哪些权益？"

        ↓ 指代消解

改写后: "金卡会员具体享有哪些权益？"

```

### 2.3 代码实现

```python

from langchain_openai import ChatOpenAI

from langchain_core.prompts import ChatPromptTemplate

# 改写用小模型，控制成本和延迟

rewrite_llm = ChatOpenAI(

    model="qwen-7b",    # 小模型即可

    temperature=0.0,     # 确定性输出

    max_tokens=512,

)

CONTEXT_RESOLVE_PROMPT = ChatPromptTemplate.from_messages([

    ("system",

     "你是一个查询改写助手。将用户在多轮对话中的最新问题，"

     "改写为一个完整、独立、无需上下文也能理解的问题。\n\n"

     "规则：\n"

     "1. 将「它」「这个」「那个」等指代词替换为具体对象\n"

     "2. 补充对话历史中隐含的背景信息\n"

     "3. 如果当前问题已经是独立的，直接原样输出\n"

     "4. 只输出改写后的问题，不要任何解释"),

    ("human",

     "对话历史：\n{chat_history}\n\n"

     "用户最新问题：{query}\n\n"

     "改写后的独立问题："),

])

async def resolve_context(query: str, chat_history: list[dict] | None = None) -> str:

    if not chat_history:

        return query

    # 只取最近 3 轮（6条消息），够用且省 token

    history_text = "\n".join(

        f"{'用户' if m['role'] == 'user' else '客服'}: {m['content']}"

        for m in chat_history[-6:]

    )

    chain = CONTEXT_RESOLVE_PROMPT | rewrite_llm

    result = await chain.ainvoke({"chat_history": history_text, "query": query})

    rewritten = result.content.strip()

    return rewritten if rewritten else query

```

### 2.4 话题切换检测——"别把新话题改坏了"

多轮对话中用户可能突然换话题，如果还强制拼历史上下文，会改写出错误的问题。

**方案**：用 embedding 相似度判断当前问题是否延续上一轮话题。

```python

import numpy as np

TOPIC_THRESHOLD = 0.65  # 低于此值认为是新话题

def is_topic_continued(query: str, chat_history: list[dict]) -> bool:

    last_user_msg = None

    for msg in reversed(chat_history):

        if msg["role"] == "user":

            last_user_msg = msg["content"]

            break

    if not last_user_msg:

        return False

    embeddings = embedding_model.embed_documents([query, last_user_msg])

    a, b = np.array(embeddings[0]), np.array(embeddings[1])

    similarity = float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))

    return similarity >= TOPIC_THRESHOLD

```

> 💡 **工程细节**：相似度 ≥ 0.65 → 同一话题，走指代消解；< 0.65 → 新话题，当首轮处理，直接检索。

>

---

## 三、Query 改写策略全家桶——"从入门到落地"
### 3.1 策略总览

| 策略 | 解决的问题 | 成本 | 适用场景 |

| --- | --- | --- | --- |

| **指代消解** | 多轮对话中的代词/省略 | 1次小模型调用 | 所有需要多轮对话的 RAG |

| **HyDE** | 口语化 vs 书面化的语义鸿沟 | 1次小模型调用 | 短且模糊的口语化查询 |

| **多查询扩展** | 单一查询遗漏多角度表述 | 1次小模型调用 | 需要更广召回面的场景 |

| **问题分解** | 复合意图的多子问题 | 1次小模型调用 | "对比A和B的价格和售后" |

### 3.2 HyDE（Hypothetical Document Embeddings）——"编个假答案去匹配"

**核心思想**：不直接用问题去检索，而是让 LLM 先生成一段"假想的理想答案"，用这段虚构文本的 embedding 去匹配真实文档。

**为什么有效**：假想答案的表述风格更接近知识库文档（正式、书面化），向量相似度更高。

```text

用户原始问题: "东西不想要了咋弄"

     ↓ LLM 生成假想答案

HyDE 文档: "用户可在签收7天内通过APP订单页面提交退货申请，

           商品需未经使用且包装完好，审核通过后48小时内寄回商品。"

     ↓ 用 HyDE 文档的 embedding 去检索

匹配到知识库: "自签收之日起7天内可申请无理由退货..." ✅

```

**代码实现**：

```python

HYDE_PROMPT = ChatPromptTemplate.from_messages([

    ("system",

     "你是一个知识库文档生成助手。根据用户的问题，生成一段假想的理想答案。\n\n"

     "要求：\n"

     "1. 风格接近正式的企业知识库文档\n"

     "2. 不需要100%准确，关键是语义覆盖可能的答案方向\n"

     "3. 长度控制在 3-5 句话\n"

     "4. 只输出假想答案，不要任何前缀"),

    ("human", "问题：{query}\n\n请生成假想的标准答案文档："),

])

async def generate_hyde(query: str) -> str:

    chain = HYDE_PROMPT | rewrite_llm

    result = await chain.ainvoke({"query": query})

    return result.content.strip()

```

**什么时候 HyDE 没用**：

- 用户提问本身已经很规范（如"Kafka 消费组 rebalance 机制"）

- 知识库是 FAQ 问答对形式，直接检索效果就够好

> 💡 **关键点**："HyDE 和原始 query 要同时检索，RRF 合并。即使 HyDE 偏了，原始 query 那路还在，不会翻车。"

>

### 3.3 多查询扩展（Multi-Query）——"从多个角度找"

让 LLM 从不同角度生成 2~3 个语义等价但表述不同的查询，分别检索后合并去重。

```python

MULTI_QUERY_PROMPT = ChatPromptTemplate.from_messages([

    ("system",

     "你是一个查询扩展助手。针对用户的问题，生成 2 个语义相同但表述不同的检索查询。\n\n"

     "要求：\n"

     "1. 每个查询从不同角度或使用不同措辞\n"

     "2. 每行输出一个查询，不要编号和解释"),

    ("human", "原始问题：{query}"),

])

```

**注意**：不要扩展太多（2~3 个就够），否则延迟和成本线性增长。

### 3.4 问题分解（Decomposition）——"大问题拆小问题"

将复杂问题拆解为多个独立子问题，分别检索后合并结果。

```text

原始: "对比一下 A 产品和 B 竞品的价格和售后政策"

     ↓ 分解

子问题1: "A 产品的价格是多少？"

子问题2: "B 产品的价格是多少？"

子问题3: "A 产品的售后政策是什么？"

子问题4: "B 产品的售后政策是什么？"

```

---

## 四、完整 Pipeline 架构——"意图路由驱动的智能调度"
### 4.1 不是所有 query 都需要全部策略

**意图路由器**根据 query 特征动态选择检索路径，避免无意义的 LLM 调用。

```text

用户提问

  │

  ▼

┌──────────────────────────────────────┐

│         意图路由器                      │

│  ├─ 话题检测 → 是否走指代消解         │

│  ├─ 精确标识符检测 → 走 BM25          │

│  ├─ 复杂度检测 → 走 Agentic RAG      │

│  └─ 长度判断 → 是否走 HyDE           │

└──────────┬───────────────────────────┘

           │

  ┌────────┼────────────────┐

  ▼        ▼                ▼

 BM25   向量检索        Agentic RAG

          ├─ HyDE 双路检索   (LangGraph)

          └─ RRF 合并

          │

          ▼

     Reranker 精排

          │

          ▼

     LLM 生成回答

```

### 4.2 路由逻辑代码

```python

import re

def route_query(query: str, chat_history: list[dict] | None) -> dict:

    # 第一步：话题检测 → 是否走指代消解

    need_resolve = False

    if chat_history and len(chat_history) >= 2:

        if is_topic_continued(query, chat_history):

            need_resolve = True

    # 第二步：复杂任务检测 → 走 Agentic RAG

    agent_keywords = ["对比", "比较", "趋势", "汇总", "报表", "图表", "分析"]

    if any(kw in query for kw in agent_keywords) and len(query) > 15:

        return {"need_resolve": need_resolve, "mode": "agentic"}

    # 第三步：精确标识符检测 → 走 BM25

    exact_patterns = [

        r'[A-Z]{2,}[-_]\d{4,}',   # ORD-12345

        r'ERR[_-]\w+',             # ERR_CONN_TIMEOUT

        r'1[3-9]\d{9}',             # 手机号

        r'\d{6,}',                 # 纯数字编号

    ]

    if any(re.search(p, query) for p in exact_patterns):

        return {"need_resolve": need_resolve, "mode": "keyword"}

    # 第四步：语义类 query → 按长度决定是否 HyDE

    if len(query) <= 8:

        return {"need_resolve": need_resolve, "mode": "vector_hyde"}

    else:

        return {"need_resolve": need_resolve, "mode": "vector"}

```

### 4.3 主流程调度

```python

async def rag_main(query: str, chat_history: list[dict] | None = None):

    route = route_query(query, chat_history)

    # Step 1: 指代消解（需要时）

    resolved = await resolve_context(query, chat_history) \

        if route["need_resolve"] else query

    # Step 2: 根据路由走不同检索路径

    if route["mode"] == "keyword":

        docs = bm25_search(resolved)

    elif route["mode"] == "vector_hyde":

        hyde_doc = await generate_hyde(resolved)

        candidates = vector_retrieve([resolved, hyde_doc])

        merged = rrf_merge(candidates)

        docs = rerank(resolved, merged, top_k=3)

    elif route["mode"] == "vector":

        candidates = vector_retrieve([resolved])

        docs = rerank(resolved, candidates, top_k=3)

    elif route["mode"] == "agentic":

        return await agent_rag.run(resolved)

    # Step 3: LLM 生成回答

    return await generate_answer(resolved, docs)

```

---

## 五、对话历史管理——"别把整段对话都塞给模型"
### 5.1 问题：对话越来越长，token 爆炸

10 轮对话后，历史可能有 5000+ tokens，全塞给 LLM 既贵又慢。

### 5.2 三种管理策略

| 策略 | 做法 | 优点 | 缺点 |

| --- | --- | --- | --- |

| **滑动窗口** | 只保留最近 N 轮（如 3~6 条消息） | 简单、省 token | 丢失早期重要上下文 |

| **对话摘要** | 定期用 LLM 对早期对话做摘要 | 大幅压缩上下文 | 摘要可能丢失细节 |

| **上下文裁剪** | 只保留与当前话题相关的历史 | 精准、信息无损 | 实现复杂 |

**工程推荐**：滑动窗口 + 摘要组合。最近 3 轮保留原文，更早的对话压缩为一段摘要。

```python

def build_context(chat_history: list[dict], max_recent_turns: int = 3) -> str:

    """构建上下文：最近 N 轮原文 + 早期摘要"""

    if len(chat_history) <= max_recent_turns * 2:

        # 不超过阈值，全用原文

        return format_history(chat_history)

    recent = chat_history[-max_recent_turns * 2:]

    older = chat_history[:-max_recent_turns * 2]

    # 早期对话压缩为摘要

    summary = summarize_history(older)

    return f"[早期对话摘要]\n{summary}\n\n[最近对话]\n{format_history(recent)}"

```

---

## 六、Pipeline 迭代演进路径——"不要一步到位"

| 阶段 | 做了什么 | Hit@5 提升 | 关键动因 |

| --- | --- | --- | --- |

| **阶段一** | 纯向量检索 | 70~75% | 基线 |

| **阶段二** |   • 指代消解 | +5% → ~80% | 解决多轮指代问题 |

| **阶段三** |   • HyDE | +8% → ~88% | 解决口语化召回 |

| **阶段四** |   • Reranker | +4% → ~92% | 解决排序不准 |

| **阶段五** |   • 意图路由 + BM25 | +3% → ~95% | 解决精确标识符 |

> 💡 **工程原则**："每个阶段都应该有数据支撑：先跑基准测试看当前指标，改完后对比效果，提升明显才合并。不要凭感觉加组件。"

>

---

## 七、LangChain / LlamaIndex 框架支持
### 7.1 LlamaIndex — CondenseQuestionChatEngine

```python

from llama_index.core import VectorStoreIndex, SimpleDirectoryReader, Settings

from llama_index.llms.openai import OpenAI

Settings.llm = OpenAI(model="gpt-3.5-turbo", temperature=0)

index = VectorStoreIndex.from_documents(documents)

chat_engine = index.as_chat_engine(

    chat_mode="condense_question",

    verbose=True  # 打印重写后的问题

)

# 多轮对话

response1 = chat_engine.chat("Paul Graham 创办 YC 后做了什么？")

response2 = chat_engine.chat("那之后呢？")

# 内部自动重写为: "Paul Graham 开始画画之后做了什么？"

```

### 7.2 LangChain — ConversationalRetrievalChain

```python

from langchain.chains import ConversationalRetrievalChain

from langchain.memory import ConversationBufferMemory

memory = ConversationBufferMemory(

    memory_key="chat_history",

    return_messages=True

)

qa_chain = ConversationalRetrievalChain.from_llm(

    llm=llm,

    retriever=vectorstore.as_retriever(),

    memory=memory

)

# 多轮对话 — 内部自动做 query 改写

result1 = qa_chain.invoke({"question": "会员等级怎么分？"})

result2 = qa_chain.invoke({"question": "它有哪些权益？"})

```

**注意**：LangChain 默认用 ConversationBufferMemory 存完整历史，对话长了会爆 token。生产环境建议换成 ConversationSummaryMemory 或自定义滑动窗口。

---

## 八、常见问题速答
### Q1：多轮对话中为什么不直接把历史拼到 query 后面检索？

**答**：因为会把噪声也带进检索。历史里可能有多轮不相关的话题、确认信息（"好的""知道了"），全拼进去会稀释真正的查询意图。而且历史越长，query 越模糊，检索精度越差。正确做法是用 LLM 做指代消解，只输出一句独立完整的问题去检索。

### Q2：指代消解和 HyDE 有什么区别？

**答**：

| 维度 | 指代消解 | HyDE |

| --- | --- | --- |

| 解决的问题 | 多轮对话中的代词/省略 | 口语化 vs 书面化的语义鸿沟 |

| 输入 | 当前问题 + 对话历史 | 当前问题（不需要历史） |

| 输出 | 一个独立完整的问题 | 一段假想的理想答案文档 |

| 何时触发 | 有对话历史且话题延续 | 短且模糊的口语化查询 |

两者可以串联使用：先消解指代，再对改写后的 query 生成 HyDE。

### Q3：话题切换怎么处理？

**答**：用 embedding 相似度判断当前问题和上一轮用户问题是否延续同一话题。如果相似度低于阈值（如 0.65），认为是新话题，跳过指代消解，当首轮处理。否则会错误地把新话题的指代词关联到旧话题的实体上。

### Q4：Query 改写会增加多少延迟？怎么优化？

**答**：每次改写多一次 LLM 调用（小模型），大约 300~500ms。优化措施：

1. **用小模型**（如 qwen-7b），不需要最强模型

2. **相同 query 短期缓存**（如 5 分钟内相同问题不重复改写）

3. **按需触发**：通过意图路由，不是每个 query 都走全部改写策略

4. **异步并行**：指代消解和 HyDE 如果都需要，可以并行执行

### Q5：你的项目中 Query 改写是怎么设计的？

**答（参考模板）**：

1. **意图路由**：先判断是否需要改写（精确标识符直接走 BM25，不改写）

2. **话题检测**：有历史时，embedding 相似度 ≥ 0.65 才走指代消解

3. **指代消解**：取最近 3 轮历史 + 当前问题，用小模型生成独立问题

4. **HyDE**：对 ≤ 8 字的短查询生成假想答案，双路检索 + RRF 合并

5. **效果**：指代消解提升 5 个点 Hit@5，HyDE 提升 8 个点

---

## 九、总结：Query 改写的"黄金法则"

> 🎯 **总结**：Query 改写不是锦上添花，而是多轮 RAG 的刚需。核心解决三类问题：指代消解（把"它"替换成真身）、HyDE（编个假答案匹配书面化文档）、问题分解（大问题拆小问题）。工程上用意图路由动态选择策略，不要每个 query 都跑全套——精确标识符走 BM25，短模糊查询走 HyDE，多轮对话走指代消解，复合问题走分解。每个阶段都要用数据验证，不要凭感觉加组件。

>

---

##

> ▶ 对应实操：[[AI与Agent/知识/八股/37-有副作用工具的权限控制|AI与Agent/知识/八股/37-有副作用工具的权限控制]]

## 十、项目落地与实践

> 本主题已**合并原理与落地实践**，不再维护八股/实战两份。

- 进阶路线：[[技术学习路线图#RAG|技术学习路线图 · RAG 模块]]

### 参考资料

- RAG Query 改写最佳实践 — 从入门到落地 — 掘金，2026-07（含完整 Pipeline 架构和迭代路径）

- 2025 最新 RAG 多轮会话优化技术图谱 — CSDN，2025-09

- LangChain ConversationalRetrievalChain 官方文档

- LlamaIndex CondenseQuestionChatEngine 官方文档

## 速记卡（面试闪卡）

**Q1：一句话讲清「Query 改写 + 多轮对话指代消解」到底是什么？**

A：Query 改写是 RAG 的翻译官，把用户打哑谜的口语补全成检索能懂的标准题。

**Q2：一、为什么需要改写 —— 怎么理解？**

A：用户追一句"它有哪些权益"，检索系统像突然听到"它多少钱"直接懵——指代/口语/复合三类鸿沟让召回跑偏（Semantic Gap 语义鸿沟）。

**Q3：二、指代消解与策略全家桶 —— 怎么理解？**

A：把最近几轮历史交给小模型，把"它"换成真身；再配 HyDE 编假答案、多查询扩展、问题分解四件套（Coreference Resolution 指代消解）。

**Q4：三、意图路由 Pipeline —— 怎么理解？**

A：不是每个 query 都跑全套：路由按话题/标识符/长度派活，精确编号走 BM25、短模糊走 HyDE、多轮走消解（Intent Router 意图路由）。

**Q5：四、对话历史与迭代 —— 怎么理解？**

A：历史太长 token 爆炸，用滑动窗口+摘要；每加一个组件都拿 Hit@5 数据说话，指代消解 +5、HyDE +8（RAG Pipeline 检索流水线）。

**Q6：核心速记主线有哪些？**

- 三类问题：指代不明、口语书面、复合意图

- 指代消解：最近 3 轮 + 小模型补成独立问题

- HyDE 与原始 query 双路检索、RRF 合并

- 迭代靠数据：每阶段对比 Hit@5 再合并

**口诀**

A：用户打谜你别懵，改写翻译把它通；

指代消来解真身，HyDE 假案来对空；

意图路由派对路，BM25 与向量同；

数据说话步步稳，召回攀高不落空。

相关链接

- [[13-RAG检索增强生成]]

- [[17-语义搜索与混合检索]]

- [[22-RAG评估指标]]

- [[八股文学习路线图]]

- [[00-全局导航|全局导航]]

## 相关链接

- [[笔记/AI与Agent/知识/八股/14-文档解析与分块策略|文档解析与分块策略]]

- [[笔记/AI与Agent/知识/八股/13-RAG检索增强生成|RAG检索增强生成]]

- [[笔记/AI与Agent/知识/八股/17-语义搜索与混合检索|语义搜索与混合检索]]

- [[笔记/AI与Agent/知识/八股/22-RAG评估指标|RAG评估指标]]

- [[笔记/AI与Agent/知识/八股/36-工具调用死循环检测|工具调用死循环检测]]


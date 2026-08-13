---

title: "Prompt权重变化"

tags:

  - agent方法论

  - 认知升级

  - 四层模型

created: "2026-07-21"

---

# Prompt 权重从 ~90% 降到 <30%

> 本条目是「认知升级：四层模型」的第 5 部分，对应学习清单条目 1.2.1。

>

> **前置依赖**：四层模型整体认知

> **为以下铺垫**：后续所有方法论学习

---

## 一、核心观点

> **Prompt 权重从约 90% 降到 <30%**：剩下 70% 是 Context + Harness + Loop。

> 这是本系列的中央论点——不是否定 Prompt，而是重新分配注意力。

---

## 二、为什么是 90% → 30%

| 时代 | Prompt 权重 | 其他权重 | 原因 |
|------|-------------|----------|------|
| 2022–2024 | ~90% | ~10% | 只有 Prompt Engineering，没有其他层 |
| 2025 | ~50% | ~50% | Context Engineering 兴起 |
| 2026 初 | ~30% | ~70% | Harness + Loop 加入 |

「权重」指三层含义：重要性、工作量、对最终效果的贡献。

---

## 三、含义
### 3.1 Prompt 仍重要，但不再是「全部」

- 每次 LLM 调用仍需要好的 Prompt

- 单靠 Prompt 无法解决系统性问题（信息缺失、环境限制、自动化）

- **定位**：`Prompt 是 Context 的子集`

### 3.2 高杠杆在 Context / Harness / Loop

| 层 | 解决的问题 | 杠杆效应 |
|----|-----------|----------|
| **Context** | 模型看到什么 | 信息缺失 → 回答不准确 |
| **Harness** | 在什么环境跑 | 无法生产部署 |
| **Loop** | 谁去 prompt Agent | 无法自动化 |

---

## 四、类比

| 角色 | 关注点 | 类比 |
|------|--------|------|
| **厨师** | 怎么炒菜 | Prompt Engineering |
| **餐厅经营者** | 菜单 + 采购 + 厨房 + 供应链 | Context + Harness + Loop |

> 厨师手艺仍重要，但餐厅成败取决于整个系统。

| 角色 | 关注点 | 类比 |
|------|--------|------|
| **程序员** | 怎么写代码 | Prompt Engineering |
| **架构师** | 系统设计 + 模块 + 部署运维 | Context + Harness + Loop |

---

## 五、实践中的应用
### 5.1 不要「只写 Prompt」

- ❌ 花 80% 时间优化 Prompt，忽略 Context / Harness / Loop

- ✅ 约 30% 写 Prompt + 30% 设计 Context + 20% 搭 Harness + 20% 设计 Loop

### 5.2 项目中的体现

**ai-resume-analyzer 项目**：

- Prompt：简历分析指令设计

- Context：RAG 检索 + 混合搜索 + Rerank

- Harness：MCP + 限流 + JWT

- Loop：LangGraph + Reflexion

**cr-agent 项目**：

- Prompt：代码审查指令设计

- Context：代码库 + 评审规则

- Harness：Git 操作 + 工具调用

- Loop：多 Agent 协作 + 质量检查

---

## 六、支撑该论点的行业数据（2024–2026）

- **Gartner（2026-05）**：40% 企业应用将在 2026 年底内置任务型 AI Agent（年初 <5%）；AI Agent 软件支出 2026 年预计 **\$206.5B**，2027 年 **\$376.3B**（+82%）。

- **Stanford HAI（2026-04）**：已部署 AI 企业在软件开发获 **26%** 生产率提升、客服 **14–15%**、营销产出 **+73%**——这些收益来自系统化而非单点 Prompt。

- **生产落差警示**：turion.ai（2026）称 **88%** 的 AI Agent 未能进入生产；进入生产的 12% 平均 ROI **171%**。说明「会写 Prompt」离「能交付」差距巨大，必须靠 Harness/Loop 补完。

---

## 七、学习资源

- Anthropic《Effective context engineering for AI agents》(2025-09)

- Gartner AI Agent 市场预测 (2026-05)

- Stanford HAI AI Index (2026-04)

- 进阶：[[06-设计Agent思考框架|设计 Agent 思考框架]] · [[07-四层嵌套关系|四层嵌套关系]]

---

## 参考来源（一手链接 · 可溯源深挖）

- **Gartner (2026-05)** AI Agent 支出 $206.5B→$376.3B、40% 企业应用内置 Agent：[AI Agents Statistics 2026](https://enterprisedna.co/resources/stats/ai-agents)（汇总 Gartner 原始发布）

- **Stanford HAI (2026-04)** AI Index：软件 +26%、客服 14–15%、营销 +73%：[Stanford HAI AI Index 2026](https://hai.stanford.edu/ai-index)

- **turion.ai (2026)** 88% Agent 未进生产、12% ROI 171%——原始发布待核实（未检索到一手来源，建议谨慎引用）

- **Anthropic (2025-09)**《Effective context engineering for AI agents》：[anthropic.com/engineering/effective-context-engineering-for-ai-agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

## 速记卡（面试闪卡）

**Q1：一句话讲清「Prompt 权重从 ~90% 降到 <30%」到底是什么？**

A：Prompt 权重从约 90% 降到 <30%，剩下交给 Context+Harness+Loop——不是否定 Prompt，是重新分配注意力。

**Q2：为什么 90% → 30% —— 怎么理解？**

A：2022–24 只有 Prompt Engineering，权重 ~90%；2025 Context Engineering 兴起降到 ~50%；2026 初 Harness+Loop 加入降到 ~30%。"权重"指重要性、工作量、对效果的贡献三重含义（weight shift over time）。

**Q3：Prompt 仍是子集，但非全部 —— 怎么理解？**

A：每次 LLM 调用仍要好 Prompt，但单靠它解决不了信息缺失、环境限制、自动化等系统问题。定位上 Prompt 是 Context 的子集。好比厨师手艺仍重要，但餐厅成败看整个系统（Prompt ⊂ Context）。

**Q4：高杠杆在 Context/Harness/Loop —— 怎么理解？**

A：Context 管"模型看到什么"（信息缺失→答不准），Harness 管"在什么环境跑"（无法生产部署），Loop 管"谁去 prompt Agent"（无法自动化）。三层才是 70% 杠杆所在（context/harness/loop leverage）。

**Q5：实践分配 + 项目体现 —— 怎么理解？**

A：别只写 Prompt——约 30% 写 Prompt + 30% 设计 Context + 20% 搭 Harness + 20% 设计 Loop。项目里 ai-resume 用 RAG+Context、MCP+Harness、LangGraph+Reflexion+Loop 印证（allocation, project mapping）。

**Q6：核心速记主线有哪些？**

- 论点：Prompt 权重 90%→<30%，余下归 Context+Harness+Loop

- Prompt 是 Context 子集，单靠它解不了系统问题

- 三层杠杆：Context(看什么)/Harness(在哪跑)/Loop(谁驱动)

- 实践配比 ~3:3:2:2；项目以 RAG+MCP+LangGraph 印证

**口诀**

A：Prompt 权重逐季降，

九成落到三成旁；

三层接力补短板，

系统取胜不单枪。

## 相关链接

- 系列清单：[[学习路线图/Agent方法论与产品思维学习路线图|Agent 方法论与产品思维学习路线图]]

- 上一层级：[[00-Agent方法论与产品思维|认知升级：四层模型 · 索引]]

- 同主题：[[01-2024PromptEngineering时代|Prompt Engineering 时代]] · [[04-2026中LoopEngineering爆发|Loop Engineering 爆发]]


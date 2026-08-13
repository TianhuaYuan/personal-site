---

title: "设计Agent思考框架"

tags:

  - agent方法论

  - 认知升级

  - 四层模型

created: "2026-07-21"

---

# 高手不再「写」Prompt，而是「设计 Agent 思考框架」

> 本条目是「认知升级：四层模型」的第 6 部分，对应学习清单条目 1.2.2。

>

> **前置依赖**：四层模型整体认知

> **为以下铺垫**：后续所有方法论学习

---

## 一、核心观点

> **高手不再「写」Prompt，而是「设计 Agent 思考框架」**：从雕琢一句话，升级为设计让系统持续产出的结构。

---

## 二、「写」vs「设计」

| 维度 | 写 Prompt | 设计思考框架 |

|------|-----------|--------------|

| **关注点** | 单次调用的措辞 | 整个系统的结构 |

| **产出** | 一段 Prompt | 一套规则 + 流程 + 约束 |

| **复用性** | 低，每次重写 | 高，一次设计多处复用 |

| **可维护性** | 差，散落各处 | 好，集中管理 |

| **类比** | 写操作手册 | 设计工作流程 |

**思考框架**是指导 Agent 如何思考、决策、行动的结构化规则：行为规则、决策准则、输出规范、错误处理、质量标准。

---

## 三、为什么要从「写」升级到「设计」

- ❌ 只写 Prompt：每次新任务重写、难一致、难维护、难扩展

- ✅ 设计框架：一次设计多处复用、行为一致、集中管理、可扩展

这一跃迁与 Loop Engineering 同构：人从「操作者」变为「设计者」。

---

## 四、思考框架的两套落地方法
### 4.1 Prompt 2.0 方法论（理论根基）

| 模块 | 内容 | 作用 |

|------|------|------|

| **四大黄金法则** | 给目标、写评估、分离规划执行、设计反馈闭环 | 理论根基 |

| **结构化六段式** | 角色→能力边界→决策准则→输出格式→安全规则→错误处理 | 具体怎么写 |

| **Maker-Checker** | Creator 写实现，Checker 独立审查 | 质量保证 |

| **框架选型** | ReAct / Plan-and-Execute / Reflexion | 场景适配 |

### 4.2 Skill Engineering 方法论（编码成可复用模块）

| 要素 | 内容 | 作用 |

|------|------|------|

| **适用场景** | 什么情况下自动激活 | 避免误触发 |

| **不适用场景** | 什么情况下不要激活 | 防止滥用 |

| **输入要求** | 预期接收什么格式 | 让 Skill 知道「我在处理什么」 |

| **操作步骤** | 核心执行流程 | 动作优先 |

| **工具使用** | 依赖哪些工具/API | 明确能力边界 |

| **输出格式** | 生成结果规范 | 让下游可消费 |

| **质量标准** | 什么算「做好」 | 可自检的终止条件 |

| **失败处理** | 常见失败模式 + 恢复策略 | **最高价值内容** |

| **安全边界** | 哪些禁止、哪些需人工确认 | 防生产事故 |

---

## 五、实践中的应用

**ai-resume-analyzer** 不是写「请分析简历」的 Prompt，而是设计完整分析框架：

- 输入规范：简历格式、字段要求

- 分析维度：工作经历、技能匹配、项目经验

- 输出格式：结构化评分 + 改进建议

- 错误处理：字段缺失、格式错误的降级策略

> 框架沉淀在 CLAUDE.md / Skills（Harness 层），由 Loop 反复调用——这正是「设计思考框架」的工程形态。

---

## 六、最新研究与企业数据（2024–2026）

- **可复用框架带来复利**：Stanford HAI（2026-04）显示已部署企业在软件开发获 26% 生产率提升，主因是流程/框架沉淀而非单次 Prompt。

- **Skills 即框架的工程化**：Anthropic / Paradime（2026）将 Skill 定义为「按需加载、可复用的指令集」，正是把「思考框架」编码进 harness。

---

## 七、学习资源

- Anthropic《Effective context engineering for AI agents》(2025-09)

- Paradime《Claude Code Skills & Harness Engineering》(2026-02)

- 进阶：[[02-2025ContextEngineering兴起|Context Engineering]] · [[03-2026初HarnessEngineering|Harness Engineering]]

---

## 参考来源（一手链接 · 可溯源深挖）

- **Stanford HAI (2026-04)** AI Index：流程/框架沉淀带来 +26% 生产率：[Stanford HAI AI Index 2026](https://hai.stanford.edu/ai-index)

- **Anthropic / Paradime (2026)** Skill = 按需加载可复用指令集：[Anthropic Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents)；Paradime《Claude Code Skills & Harness Engineering》(2026-02，链接待核实)

- **Anthropic (2025-09)**《Effective context engineering for AI agents》：[anthropic.com/engineering/effective-context-engineering-for-ai-agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

## 速记卡（面试闪卡）

**Q1：一句话讲清「高手不再「写」Prompt，而是「设计 Agent 思考框架」」到底是什么？**

A：设计 Agent 思考框架是让人从「雕琢一句 Prompt」升级为「设计让系统持续产出的结构」——一套规则+流程+约束，一次设计多处复用。

**Q2：「写」vs「设计」的本质差异 —— 怎么理解？**

A：写 Prompt 像写一份操作手册（单次措辞、难复用）；设计思考框架像设计工作流程（规则+流程+约束集中管理、可复用可维护）——前者每次重写，后者一次设计处处跑。

**Q3：为什么要从写升级到设计 —— 怎么理解？**

A：只写 Prompt 每新任务重写、难一致难维护；设计框架一次设计多处复用、行为一致、可扩展——人和 Loop Engineering 同构：从「操作者」变「设计者」。

**Q4：两套落地方法 —— 怎么理解？**

A：Prompt 2.0 方法论（四大黄金法则/六段式/Maker-Checker/框架选型）给理论根基；Skill Engineering 把框架编码成可复用模块（适用场景/步骤/工具/输出/质量标准/失败处理/安全边界）。

**Q5：实践与复利证据 —— 怎么理解？**

A：ai-resume 项目不是写「请分析简历」，而是设计输入规范+分析维度+输出格式+错误处理的完整框架，沉淀在 CLAUDE.md/Skills 由 Loop 调用；Stanford HAI 显示可复用框架带来 +26% 生产率。

**Q6：核心速记主线有哪些？**

- 「写」vs「设计」：操作手册 vs 工作流程

- 升级动机：复用/一致/可维护/可扩展

- 落地法一：Prompt 2.0（黄金法则+六段式+Maker-Checker）

- 落地法二：Skill Engineering 编码成可复用模块

- 复利证据：Stanford HAI +26% 生产率来自框架沉淀

**口诀**

A：高手不写单句 Prompt，设计框架系统跑；

一次设计多处用，操作手册比不了；

Prompt 2.0 打地基，Skill 装进模块包；

框架复利靠沉淀，一次设计胜百抄。

## 相关链接

- 系列清单：[[学习路线图/Agent方法论与产品思维学习路线图|Agent 方法论与产品思维学习路线图]]

- 上一层级：[[00-Agent方法论与产品思维|认知升级：四层模型 · 索引]]

- 同主题：[[05-Prompt权重变化|Prompt 权重变化]] · [[07-四层嵌套关系|四层嵌套关系]]


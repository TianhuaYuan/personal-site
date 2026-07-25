---
title: "AI Agent 方法论学习路线图"
created: "2026-07-14"
tags:
  - 路线图
  - agent方法论
  - 前沿认知
---

# AI Agent 方法论学习路线图

> **掌握优先级**：🔴 核心必学 · 🟡 进阶重点 · 🟢 了解即可。每条链接指向对应的学习笔记，可按主题顺序推进。


---

## 📑 目录

|  #  | 主题           | 包含内容                             |  条目数   |
| :-: | ------------ | -------------------------------- | :----: |
|  一  | #一、认知与框架 | 四层模型                             |   10   |
|  二  | #二、指令工程  | Prompt 2.0 + Context Engineering |   30   |
|  三  | #三、能力工程  | Skill Engineering + 多工具协同        |   24   |
|  四  | #四、系统工程  | Loop Engineering + Agent产品化      |   29   |
|     | **总计**       |                                  | **93** |

---

## 一、认知与框架

### 1.1 四层模型演进全貌（2022→2026）

- 1. 2024 Prompt Engineering 时代：ChatGPT 级提示词技巧 🔴 重点 →[[笔记/AI Agent 方法论笔记/01-认知升级/01-2024 Prompt Engineering 时代.md|2024 Prompt Engineering 时代]]
- 2. 2025 Context Engineering 兴起：RAG + 信息筛选 + 注意力管理 🔴 重点 →[[笔记/AI Agent 方法论笔记/01-认知升级/01-2024 Prompt Engineering 时代.md|2024 Prompt Engineering 时代]]
- 3. 2026 初 Harness Engineering：CLAUDE.md + Hooks + MCP 的工程外壳 🔴 重点 →[[笔记/AI Agent 方法论笔记/01-认知升级/01-2024 Prompt Engineering 时代.md|2024 Prompt Engineering 时代]]
- 4. 2026 中 Loop Engineering 爆发：从手动调 Prompt 到设计自主循环 🔴 重点 →[[笔记/AI Agent 方法论笔记/01-认知升级/01-2024 Prompt Engineering 时代.md|2024 Prompt Engineering 时代]]

### 1.2 四层模型核心认知（嵌套关系、权重变化）

- 5. Prompt 权重从 ~90% 降到 <30%——其余 70% 是 Context + Harness + Loop 🔴 重点 →[[笔记/AI Agent 方法论笔记/01-认知升级/01-2024 Prompt Engineering 时代.md|2024 Prompt Engineering 时代]]
- 6. 高手不再"写"Prompt，而是"设计 Agent 思考框架" 🔴 重点 →[[笔记/AI Agent 方法论笔记/01-认知升级/01-2024 Prompt Engineering 时代.md|2024 Prompt Engineering 时代]]
- 7. 四层是**嵌套关系**（外层包含内层），不是替代关系 🔴 重点

### 1.3 Boris Cherny语录与范式转移信号

- 8. Boris Cherny（Claude Code Lead）语录理解：*"I don't prompt Claude anymore. I have loops running that prompt Claude."* 🔴 重点 →[[笔记/AI Agent 方法论笔记/01-认知升级/08-Boris Cherny语录.md|Boris Cherny语录]]

### 1.4 输出要求

- 9. 能用一句话说清四层模型的核心区别 🔴 重点
- 10. 能画出四层模型演进图 🔴 重点

---

## 二、指令工程

### 2.1 Prompt 2.0 四大黄金法则

- 1. **给目标，不给步骤**：定义"完成状态"，不是执行路径 🔴 重点 →[[笔记/AI Agent 方法论笔记/02-Prompt 2.0/01-给目标不给步骤.md|给目标不给步骤]]
- 2. **把评估标准写进指令**：让 Agent 知道什么算"做完" 🔴 重点 →[[笔记/AI Agent 方法论笔记/00-Agent 方法论与产品思维.md|Agent 方法论与产品思维]]
- 3. **分离规划与执行**：Maker-Checker 模式 🔴 重点 →[[笔记/AI Agent 方法论笔记/02-Prompt 2.0/03-分离规划与执行.md|分离规划与执行]]
- 4. **设计反馈闭环**：从单次 Prompt 到 execute → evaluate → adjust 循环 🔴 重点 →[[笔记/AI Agent 方法论笔记/01-认知升级/01-2024 Prompt Engineering 时代.md|2024 Prompt Engineering 时代]]

### 2.2 结构化 Prompt 设计（六段式）

- 5. Prompt 分层的标准结构：**角色 → 能力边界 → 决策准则 → 输出格式 → 安全规则 → 错误处理** 🔴 重点 →[[笔记/AI Agent 方法论笔记/01-认知升级/01-2024 Prompt Engineering 时代.md|2024 Prompt Engineering 时代]]
- 6. 单个 section 约束数量控制在 **1-6 条以内**（超过 6 条可靠性下降） 🔴 重点

### 2.3 反例约束与禁止模糊词

- 7. **反例约束 Hard Gate**：不只说"要做什么"，更要说"不要做什么" 🔴 重点 →[[笔记/AI Agent 方法论笔记/02-Prompt 2.0/07-反例约束.md|反例约束]]
- 8. 禁止出现模糊词：`try` `ideally` `if possible`——都是漏洞 🔴 重点

### 2.4 Maker-Checker 模式

- 9. **概念**：Creator Agent 写实现 + Checker Agent 审查，双重校验 🔴 重点 →[[笔记/AI Agent 方法论笔记/00-Agent 方法论与产品思维.md|Agent 方法论与产品思维]]
- 10. **实战落地**：Claude Code post-edit hook 做 lint + typecheck 自动验证 🔴 重点 →[[笔记/AI Agent 方法论笔记/05-多工具协同/01-Claude Code特性.md|Claude Code特性]]
- 11. **成本考量**：Checker 用低成本模型（Haiku）就够了 🔴 重点 →[[笔记/AI Agent 方法论笔记/02-Prompt 2.0/09-Maker-Checker概念.md|Maker-Checker概念]]
- 12. 能讲清"为什么用 Maker-Checker 而不是让同一个 Agent 自检" 🔴 重点 →[[笔记/AI Agent 方法论笔记/02-Prompt 2.0/09-Maker-Checker概念.md|Maker-Checker概念]]

### 2.5 框架选型判断力

- 13. **ReAct**（推理+行动交替）：适合任务路径不确定的场景 🔴 重点 →[[笔记/AI Agent 方法论笔记/02-Prompt 2.0/13-ReAct框架.md|ReAct框架]]
- 14. **Plan-and-Execute**（先计划再执行）：适合步骤固定的确定性任务 🔴 重点 →[[笔记/AI Agent 方法论笔记/02-Prompt 2.0/14-Plan-and-Execute框架.md|Plan-and-Execute框架]]
- 15. **Reflexion**（带自我反思）：适合需要多轮迭代的场景 🟢 了解 →[[笔记/AI Agent 方法论笔记/02-Prompt 2.0/15-Reflexion框架.md|Reflexion框架]]
- 16. **选型判断标准**：不是所有任务都需复杂框架，从最简方案起步 🔴 重点 →[[笔记/AI Agent 方法论笔记/02-Prompt 2.0/16-选型判断标准.md|选型判断标准]]

### 2.6 Context Engineering 理论根基

- 17. **Lost in the Middle 现象**：长 Prompt 中间的指令遵循率仅 ~60% 🔴 重点 →[[笔记/AI Agent 方法论笔记/03-Context Engineering/01-Lost in the Middle.md|Lost in the Middle]]
- 18. **Attention Budget 概念**：Agent 每 token 注意力有限，超过阈值开始忽略 🔴 重点 →[[笔记/AI Agent 方法论笔记/02-Prompt 2.0/09-Maker-Checker概念.md|Maker-Checker概念]]
- 19. **150 条指令天花板**：超过 ~150 条独立规则，模型选择性忽略 🔴 重点 →[[笔记/AI Agent 方法论笔记/03-Context Engineering/03-150条指令天花板.md|150条指令天花板]]
- 20. **文件大小与注意力衰减**：1-100 行 (~95% 遵循) → 600+ 行 (~45%) 🔴 重点 →[[笔记/AI Agent 方法论笔记/03-Context Engineering/04-文件大小与注意力衰减.md|文件大小与注意力衰减]]

### 2.7 Context Engineering 实战技术

- 21. 关键规则放**首尾位置**（开头 + 末尾，避开中间区域） 🔴 重点
- 22. 渐进式加载：长资料按需加载，不要一股脑塞进 System Prompt 🔴 重点 →[[笔记/AI Agent 方法论笔记/03-Context Engineering/06-渐进式加载.md|渐进式加载]]
- 23. 工具返回结果做**摘要压缩**，不堆原始输出 🔴 重点
- 24. 长任务中间**重注入**一次关键约束 🔴 重点
- 25. **Recency Effect 利用**：Claude 对末尾指令遵循率更高，hard gate 放末尾 🔴 重点 →[[笔记/AI Agent 方法论笔记/03-Context Engineering/07-Recency Effect.md|Recency Effect]]

### 2.8 Context Engineering 诊断与修复

- 26. 识别上下文污染的征兆：回答跑偏、遗忘之前指令、自言自语 🔴 重点
- 27. 找到你自己的"临界点"：对话多长后开始跑偏？ 🟢 了解 →[[笔记/AI Agent 方法论笔记/03-Context Engineering/11-找到临界点.md|找到临界点]]
- 28. 用 `/compact` 或手动总结中间结果 🟢 了解 →[[笔记/AI Agent 方法论笔记/03-Context Engineering/12-compact总结.md|compact总结]]
- 29. 设计关键规则重注入的格式 🟢 了解
- 30. 路径作用域（Path-scoping）：CLAUDE.md 放子目录，减少全局上下文 🟢 了解 →[[笔记/AI Agent 方法论笔记/03-Context Engineering/14-路径作用域.md|路径作用域]]

---

## 三、能力工程

### 3.1 Skill Engineering 基础

- 1. **Skill ≠ 长 Prompt**：是可复用的能力模块 🔴 重点 →[[笔记/AI Agent 方法论笔记/01-认知升级/01-2024 Prompt Engineering 时代.md|2024 Prompt Engineering 时代]]
- 2. **读外部 Skills 建立认知**：打开 `.claude/skills/`，读所有已安装技能的 structure 🔴 重点 →[[笔记/AI Agent 方法论笔记/04-Skill Engineering/01-读外部Skills.md|读外部Skills]]
- 3. **分析优秀 Skill**：挑 3 个写得好的外部 skill，逐行分析为什么好 🔴 重点 →[[笔记/AI Agent 方法论笔记/04-Skill Engineering/01-读外部Skills.md|读外部Skills]]
- 4. **对比工具 Skill 系统**：Claude Code skills vs Codex rules vs Cursor rules 🟢 了解 →[[笔记/AI Agent 方法论笔记/04-Skill Engineering/01-读外部Skills.md|读外部Skills]]

### 3.2 Skill 结构设计（九要素）

- 5. **适用场景**：什么情况下自动激活 🔴 重点
- 6. **不适用场景**：什么情况下不要激活（"Do not activate"路由块） 🔴 重点
- 7. **输入要求**：预期接收什么格式/内容 🔴 重点
- 8. **操作步骤**：核心执行流程（动作优先，不是知识优先） 🔴 重点
- 9. **工具使用**：依赖哪些工具/API/MCP 🔴 重点
- 10. **输出格式**：生成结果的规范 🔴 重点
- 11. **质量标准**：什么算"做好"、"足够好"、"重做" 🔴 重点
- 12. **失败处理**：常见失败模式 + 恢复策略 🔴 重点（**最高价值内容**）
- 13. **安全边界**：哪些操作禁止、哪些需要人工确认 🔴 重点

### 3.3 Skill 书写与优化

- 14. 选一个日常重复任务，写第一个 `SKILL.md` 🔴 重点 →[[笔记/AI Agent 方法论笔记/04-Skill Engineering/06-写第一个SKILL.md|写第一个SKILL]]
- 15. 改一遍：加入失败处理 + 安全边界 🟢 了解
- 16. 再改一遍：压缩到 500 行以内，深度内容移到 `references/` 🟢 了解 →[[笔记/AI Agent 方法论笔记/04-Skill Engineering/07-压缩到500行.md|压缩到500行]]
- 17. 实际用一次，根据反馈迭代 🔴 重点

### 3.4 AGENTS.md / CLAUDE.md 编写哲学

- 18. 控制在 200 行以内：超过这个长度，人和 Agent 都会 skim-read 🔴 重点
- 19. 每条规则必须可证伪：「写干净代码」不可验证；「所有 async 函数必须设 timeout」可验证 🔴 重点
- 20. 只写 Agent 自己读不出来的事：项目用 TypeScript 不用写，因为 `tsconfig.json` 会告诉它 🔴 重点 →[[笔记/AI Agent 方法论笔记/00-Agent 方法论与产品思维.md|Agent 方法论与产品思维]]

### 3.5 Skill 生命周期管理

- 21. 理解延迟加载机制：description 匹配触发 vs 显式 `/skill-name` 调用 🔴 重点
- 22. 多文件 Skill 架构：`SKILL.md` + `references/` + `scripts/` 三层 🔴 重点 →[[笔记/AI Agent 方法论笔记/04-Skill Engineering/01-读外部Skills.md|读外部Skills]]
- 23. Gotchas 章节优先：经验证实的失败模式 > 理论指导 🔴 重点 →[[笔记/AI Agent 方法论笔记/04-Skill Engineering/05-Gotchas优先.md|Gotchas优先]]
- 24. Skill 落地 git，每月 review 一次 🟢 了解 →[[笔记/AI Agent 方法论笔记/02-Prompt 2.0/10-Maker-Checker落地.md|Maker-Checker落地]]

### 3.6 多工具协同基础

- 25. **三大工具路线对比**：Claude Code（终端原生）/ Codex（云端异步）/ Cursor（IDE 原生） 🔴 重点 →[[笔记/AI Agent 方法论笔记/05-多工具协同/01-Claude Code特性.md|Claude Code特性]]
- 26. **工具选型原则**：按协作姿势选型，不按名气 🔴 重点 →[[笔记/AI Agent 方法论笔记/05-多工具协同/04-工具选型原则.md|工具选型原则]]
- 27. **MCP 工具选型**：你的项目真正需要哪几个 MCP Server 🟡 进阶 →[[笔记/AI Agent 方法论笔记/05-多工具协同/04-工具选型原则.md|工具选型原则]]
- 28. **工具箱管理**：定制化配置 + 私有 skills + 环境偏好 🟡 进阶 →[[笔记/AI Agent 方法论笔记/05-多工具协同/06-工具箱管理.md|工具箱管理]]

### 3.7 协议层

- 29. **MCP**（Model Context Protocol）：Agent 的"USB 接口"，解决 Agent 调工具 🔴 重点 →[[笔记/AI Agent 方法论笔记/01-认知升级/02-2025 Context Engineering 兴起.md|2025 Context Engineering 兴起]]
- 30. **A2A**（Agent-to-Agent Protocol）：Agent 的"互联网协议"，解决跨平台 Agent 协作 🟡 进阶 →[[笔记/AI Agent 方法论笔记/00-Agent 方法论与产品思维.md|Agent 方法论与产品思维]]
- 31. **ACP**（Agent Client Protocol）：Agent 的"进程管理器"，解决 IDE 管理 Agent 🟡 进阶 →[[笔记/AI Agent 方法论笔记/00-Agent 方法论与产品思维.md|Agent 方法论与产品思维]]

### 3.8 跨工具协作

- 32. 一个项目里混用多种 Agent 的策略 🔴 重点 →[[笔记/AI Agent 方法论笔记/00-Agent 方法论与产品思维.md|Agent 方法论与产品思维]]
- 33. 工具切换成本管理：不要频繁换工具，但要知道什么时候该换 🟡 进阶

---

## 四、系统工程

### 4.1 Loop Engineering 六大构件

- 1. **Automations**（自动化心跳）：定时/事件触发 🟡 进阶 →[[笔记/AI Agent 方法论笔记/06-Loop Engineering/01-Automations.md|Automations]]
- 2. **Worktrees**（工作树隔离）：多 Agent 并行不冲突 🟢 了解 →[[笔记/AI Agent 方法论笔记/00-Agent 方法论与产品思维.md|Agent 方法论与产品思维]]
- 3. **Skills**（技能编码）：项目知识沉淀 🟢 了解 →[[笔记/AI Agent 方法论笔记/04-Skill Engineering/01-读外部Skills.md|读外部Skills]]
- 4. **Plugins & Connectors**（MCP 协议接入真实工具） 🟢 了解 →[[笔记/AI Agent 方法论笔记/05-多工具协同/05-MCP工具选型.md|MCP工具选型]]
- 5. **Sub-agents**（子 Agent 编排）：创造者与检查者分离 🟡 进阶 →[[笔记/AI Agent 方法论笔记/00-Agent 方法论与产品思维.md|Agent 方法论与产品思维]]
- 6. **State**（外部状态/记忆）：跨 session 持久化 🟢 了解 →[[笔记/AI Agent 方法论笔记/06-Loop Engineering/06-State.md|State]]

### 4.2 Loop Engineering 安全护栏

- 7. **终止条件必须机器可检查**：不要相信 Agent 的"我做好了" 🔴 重点 →[[笔记/AI Agent 方法论笔记/00-Agent 方法论与产品思维.md|Agent 方法论与产品思维]]
- 8. **预算护栏必设**：`MAX_STEPS` + token 上限 + 重试上限 🔴 重点 →[[笔记/AI Agent 方法论笔记/06-Loop Engineering/08-预算护栏必设.md|预算护栏必设]]
- 9. 子 Agent 超过 **15 个工具调用**触发上下文衰减——这时需要拆分任务 🔴 重点 →[[笔记/AI Agent 方法论笔记/00-Agent 方法论与产品思维.md|Agent 方法论与产品思维]]
- 10. **Warning**：Loop = 高风险——成本爆炸 + 验证绕过 + 无限循环 🔴 重点 →[[笔记/AI Agent 方法论笔记/01-认知升级/04-2026中 Loop Engineering 爆发.md|2026中 Loop Engineering 爆发]]

### 4.3 Loop Engineering 预算护栏细节

- 11. **爆炸半径（blast radius）**：一个 worktree、一个分支，工作目录外只读 🔴 重点 →[[笔记/AI Agent 方法论笔记/06-Loop Engineering/11-爆炸半径.md|爆炸半径]]
- 12. **断路器（circuit breaker）**：连续 3 次用相同参数调相同工具 = Agent 卡住了 🔴 重点 →[[笔记/AI Agent 方法论笔记/06-Loop Engineering/12-断路器.md|断路器]]
- 13. **存活检查（heartbeat）**：每次运行向状态文件写心跳，静默 = 已死 🔴 重点 →[[笔记/AI Agent 方法论笔记/06-Loop Engineering/13-存活检查.md|存活检查]]

### 4.4 Loop Engineering 最小实践

- 14. 写一个最简单的 Closed Loop 脚本：触发 → 执行 → 验证 → 停止 🟢 了解 →[[笔记/AI Agent 方法论笔记/01-认知升级/04-2026中 Loop Engineering 爆发.md|2026中 Loop Engineering 爆发]]
- 15. 给 Loop 加一个"看门狗"：检测重复 Action 后熔断 🟢 了解 →[[笔记/AI Agent 方法论笔记/01-认知升级/04-2026中 Loop Engineering 爆发.md|2026中 Loop Engineering 爆发]]

### 4.5 Agent 产品化思维

- 16. **Workflow vs Agent 选型**：路径固定→Workflow；路径不确定→Agent；混合模式更优 🔴 重点 →[[笔记/AI Agent 方法论笔记/00-Agent 方法论与产品思维.md|Agent 方法论与产品思维]]
- 17. 不要为翻译任务设计四步 Agent——简单任务最简单方案 🟡 进阶 →[[笔记/AI Agent 方法论笔记/00-Agent 方法论与产品思维.md|Agent 方法论与产品思维]]
- 18. **幻觉递归陷阱**：步骤越多误差越易累积，设置人工确认点 🟢 了解 →[[笔记/AI Agent 方法论笔记/07-Agent产品化/03-幻觉递归陷阱.md|幻觉递归陷阱]]

### 4.6 需求翻译与方案设计

- 19. 用户需求 → Agent 方案的四步翻译：**目标 → 约束 → 边界 → 评估** 🟡 进阶 →[[笔记/AI Agent 方法论笔记/00-Agent 方法论与产品思维.md|Agent 方法论与产品思维]]
- 20. 失败模式预判：提前想清楚 Agent 在什么场景下必挂 🟢 了解 →[[笔记/AI Agent 方法论笔记/00-Agent 方法论与产品思维.md|Agent 方法论与产品思维]]
- 21. 降级路径设计：Agent 不行时怎么回退到可用状态 🟡 进阶 →[[笔记/AI Agent 方法论笔记/00-Agent 方法论与产品思维.md|Agent 方法论与产品思维]]

### 4.7 效果度量

- 22. **三维评估框架**：**成功率** + **工具调用准确率** + **Token 成本** 🔴 重点 →[[笔记/AI Agent 方法论笔记/07-Agent产品化/07-三维评估框架.md|三维评估框架]]
- 23. **LLM-as-Judge**：什么时候该用 LLM 判分、局限性在哪 🟡 进阶 →[[笔记/AI Agent 方法论笔记/07-Agent产品化/08-LLM-as-Judge.md|LLM-as-Judge]]
- 24. A/B 测试对照：加载 Skill 与否、旧 Skill vs 新 Skill 🟢 了解 →[[笔记/AI Agent 方法论笔记/03-Context Engineering/06-渐进式加载.md|渐进式加载]]

### 4.8 交互与体验设计

- 25. **Human-in-the-Loop 时机判断标准**：哪些操作需要人介入 🔴 重点 →[[笔记/AI Agent 方法论笔记/01-认知升级/04-2026中 Loop Engineering 爆发.md|2026中 Loop Engineering 爆发]]
- 26. **权限分级**：read-only / write / destructive 三级防护 🔴 重点 →[[笔记/AI Agent 方法论笔记/07-Agent产品化/11-权限分级.md|权限分级]]
- 27. Agent 决策过程可追溯，不是黑盒 🟡 进阶 →[[笔记/AI Agent 方法论笔记/00-Agent 方法论与产品思维.md|Agent 方法论与产品思维]]
- 28. 错误恢复路径：用户要能看到 Agent 出错了然后自动恢复 🟢 了解 →[[笔记/AI Agent 方法论笔记/00-Agent 方法论与产品思维.md|Agent 方法论与产品思维]]

### 4.9 产品化四维模型

- 29. **产品价值 = 意图清晰度 × 控制可见性 × 交互摩擦最小化 × 执行可信度** 🟡 进阶
- 30. 四维乘积模型：任何一个维度为 0，整个系统价值崩塌 🟡 进阶 →[[笔记/AI Agent 方法论笔记/07-Agent产品化/14-四维乘积模型.md|四维乘积模型]]

---


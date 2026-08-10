---
title: "2026初 Harness Engineering"
tags:
  - agent方法论
  - 认知升级
  - 四层模型
  - harness
created: "2026-07-21"
---

# 2026 初 Harness Engineering：CLAUDE.md + Hooks + MCP 的工程外壳

> 本条目是「认知升级：四层模型」的第 3 部分，对应学习清单条目 1.1.3。
>
> **前置依赖**：Prompt Engineering、Context Engineering
> **为以下铺垫**：Loop Engineering

---

## 一、核心观点

> **Harness Engineering = 给 Agent 搭的「工作台 + 护栏」**：`coding agent = AI model(s) + harness`。
> 它解决「Agent 在什么环境里跑、能用什么工具、错了怎么拦」，让 Agent 在生产环境可靠运行。

---

## 二、定义与出处（可验证）

- **术语提出**：由 **HumanLayer** 团队在 2026 年提出并定义——Agent 不只是模型，而是「模型 + 围绕它的 harness」。
- **定位**：Harness 是 Context Engineering 的**结构化子集**（structural subset）。CLAUDE.md / Skills 优化的是单次推理的输入（Context 层）；Commands / Hooks / Permissions 约束的是系统的长期行为（Harness 层）。
- **核心判断**：「如果你的问题在 Harness 层，再大的 CLAUDE.md 也不够。」

---

## 三、为什么需要 Harness

没有 harness，Agent 呈现四类可预测的失败模式：

| 失败模式 | 表现 | Harness 解法 |
|----------|------|--------------|
| **Context Rot** | 长会话中 stale 信息填满窗口，性能下降 | 压缩 / 子 Agent 隔离 |
| **决策不一致** | 相同情境跨会话给出不同选择 | Rules 文件固化约定 |
| **工具混淆** | 工具过多、功能重叠导致误用 | MCP 只接必要项 |
| **知识缺口** | 不知团队规范、禁用操作 | Skills / CLAUDE.md 注入 |

---

## 四、核心组件

| 组件 | 角色 | 配置位置 |
|------|------|----------|
| **CLAUDE.md / .claude/rules** | 每次会话自动加载的全局约定 | 仓库根 |
| **Skills** | 按需加载的专项流程（部署、审查） | `.claude/skills/` |
| **MCP Servers** | 连接外部工具/数据（搜索、Jira、DB） | `settings.json` |
| **Hooks** | 生命周期事件点的**确定性**自动化 | `settings.json` |
| **Sub-agents** | 隔离上下文的专业化助手 | Claude Code |
| **Plugins / Permissions** | 可分发扩展、自动批准范围 | `settings.json` |

### 4.1 Hooks：确定性执行，不靠模型记忆

```json
{
  "hooks": {
    "PostToolUse": [{ "matcher": "Edit|Write",
      "hooks": [{ "type": "command", "command": "npx prettier --write" }] }],
    "PreToolUse": [{ "matcher": "Bash",
      "hooks": [{ "type": "command", "command": ".claude/hooks/pre-bash-firewall.sh" }] }]
  }
}
```

> **铁律**：需要每次都执行的事用 Hooks，**不要用 Prompt 提醒**。Hook 退出码 `0` = 允许，`2` = 阻断。Hooks 是确定性的，Skills 是概率性的。

---

## 五、核心原则

> **Mitchell Hashimoto（HashiCorp 创始人）**：「Every time the agent makes a mistake, engineer it so that mistake can never happen again.」
> 不是修复错误，而是建造一个让错误**不可能再发生**的结构。

实践要点：

- **Progressive Disclosure**：不要把所有指令塞进 System Prompt，用 Skills 按需加载，节省上下文
- **CLAUDE.md < 200 行**：超过此长度人与 Agent 都会 skim-read，其余移入 Skills
- **MCP 只接必要项**：连接器越多，token 浪费与判断错误概率越高；用不上的就断开

---

## 六、最新研究与企业数据（2024–2026）

- **OpenAI Codex 实验（representative case）**：5 个月、100 万行代码、**0 行由人直接写**、合并 1,500 个 PR，单工程师日均完成约 3.5 个任务——靠的是 harness 而非人写代码。
- **Terminal Bench 2.0 反差**：同一模型（Opus 4.6）用默认 harness 排名第 40，用优化 harness 排名**第 1**。"**The harness, not the model, determined the ranking.**"（GoCodeLab, 2026）
- **Hooks 实测收益**：Back-Pressure 模式（Hooks 静音成功、只显失败）让 4,000 行测试输出不再淹没上下文。

---

## 七、优势与局限

- ✅ 把「会犯的错误」变成「结构上不可能的错误」；跨会话一致性；可累积为团队知识
- ❌ 初始搭建有一次性成本；配置过多反而增加认知负担；需要持续迭代

---

## 八、学习资源

- **权威指南**
  - Paradime《Claude Code Skills & Harness Engineering: Complete Guide》(2026-02-26)
  - cuiliang.ai《Harness Engineering：Agent 工程的第三次范式跃迁》
  - Anthropic《Effective context engineering for AI agents》(2025-09)
- **进阶阅读**
  - [[04-2026中 Loop Engineering 爆发|2026 中 Loop Engineering 爆发]]——Harness 是 Loop 的一个执行单元

---

**下一篇**：[[04-2026中 Loop Engineering 爆发|2026 中 Loop Engineering 爆发]]——有了工作台，下一步让系统自己跑。

---

## 参考来源（一手链接 · 可溯源深挖）

- **HumanLayer (2026)** 提出并定义 Harness Engineering（「coding agent = AI model(s) + harness」）：[humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents](https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents)
- **OpenAI Codex「Symphony」实验**（5 个月 / ~100 万行 / 0 行人手写 / ~1500 PR）——来源待核实：OpenAI Frontier Product Exploration 二手报道，未检索到一手发布
- **LangChain (2026-02)** Terminal Bench 2.0 harness 实验（排名 30→前 5）：[langchain.com/blog/the-anatomy-of-an-agent-harness](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness)
- **Mitchell Hashimoto** Harness 原则（「让错误永远不再发生」）：引自 [HumanLayer 博文](https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents) 与 [martinfowler.com/articles/harness-engineering.html](https://martinfowler.com/articles/harness-engineering.html)
- **Paradime (2026-02-26)**《Claude Code Skills & Harness Engineering》——原始链接待核实
- **cuiliang.ai**《Harness Engineering：Agent 工程的第三次范式跃迁》——原始链接待核实
- **Anthropic (2025-09)**《Effective context engineering for AI agents》：[anthropic.com/engineering/effective-context-engineering-for-ai-agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

**相关链接**：
- 系列清单：[[学习路线图/Agent 方法论与产品思维学习清单|Agent 方法论与产品思维学习清单]]
- 上一层级：[[00-Agent 方法论与产品思维|认知升级：四层模型 · 索引]]
- 同主题：[[02-2025 Context Engineering 兴起|Context Engineering 兴起]] · [[05-Prompt权重变化|Prompt 权重变化]]

---
title: "AI Agent框架对比"
created: "2026-07-13"
tags:
  - 八股文
  - ai
---

# AI Agent 框架对比

> 2026 年 AI Agent 框架百花齐放：Claude Agent SDK、OpenAI Agents SDK、LangGraph 各有侧重。爱问"你用什么框架？为什么？"——本篇横向对比三大框架的核心差异。

## LangChain vs LlamaIndex

| 维度 | LangChain | LlamaIndex |
| ------ | ----------- | ------------ |
| 定位 | 通用 LLM 应用框架 | 数据索引和检索框架 |
| 核心能力 | Chain + Agent + Tool | Index + Query Engine |
| RAG 支持 | 通过 Retriever 集成 | 原生深度支持 |
| Agent 支持 | 强（LangGraph） | 有但不是重点 |
| 数据连接 | 通过 Loader 集成 | 丰富的 Data Connector |
| 索引类型 | 依赖向量数据库 | Vector/Keyword/Tree/KG |
| 社区生态 | 最大，集成最多 | 中等，专注数据领域 |
| 学习曲线 | 中等 | 较低（专注 RAG 场景） |
| 生产就绪 | LangSmith 提供监控 | 有评估工具 |

**选择建议：**
- **纯 RAG 场景**：LlamaIndex 更简单直接
- **需要 Agent / 复杂工作流**：LangChain + LangGraph
- **两者结合**：LlamaIndex 做检索，LangChain 做 Agent 编排

## Claude Agent SDK vs OpenAI Agents SDK

| 维度 | Claude Agent SDK | OpenAI Agents SDK |
| ------ | ----------------- | ------------------- |
| 开发商 | Anthropic | OpenAI |
| 核心理念 | 安全优先，工具调用标准化 | 简洁 API，快速上手 |
| 工具调用 | Tool Use API + MCP | Function Calling |
| 多 Agent | 内置 Handoff 机制 | 内置 Agent-to-Agent 转接 |
| 安全机制 | Constitutional AI + 工具权限控制 | Guardrails 内置 |
| 流式输出 | 原生支持 | 原生支持 |
| 生态 | 与 Claude 深度集成 | 与 GPT/Whisper/DALL-E 集成 |
| 适用场景 | 安全敏感、企业级应用 | 快速原型、通用 Agent 开发 |

## 框架选型原则

| 场景 | 推荐框架 | 理由 |
| ------ |----------| ------ |
| 快速原型验证 | OpenAI Agents SDK | API 简洁，上手快 |
| 安全敏感场景 | Claude Agent SDK | Constitutional AI + 权限控制 |
| 复杂多步工作流 | LangGraph | 有向图 + 状态机 + 循环 |
| 纯 RAG 应用 | LlamaIndex | 原生 RAG 支持，索引类型丰富 |
| 企业级全栈 | LangChain + LangSmith | 生态完善，可观测性强 |
| 多 Agent 协作 | LangGraph 或 CrewAI | 图编排 / 角色扮演 |

> **核心原则**：不要为了用框架而用——讲清楚"什么场景该用什么框架"比罗列框架名值钱。

---

## 相关链接
- [[八股文学习清单]]
- [[00-全局导航|全局导航]]

---

## 快速问答

| 问题 | 参考答案 |
| ------ | --------- |
| Claude Agent SDK 和 OpenAI Agents SDK 的核心区别？ | Claude SDK 安全优先（Constitutional AI + 权限控制），OpenAI SDK 简洁易上手。Claude 内置 Handoff，OpenAI 内置 Guardrails |
| 什么时候用 LangGraph 而不是 LangChain Agent？ | 需要循环、分支、有状态工作流时用 LangGraph。简单线性流程用 LangChain Agent 即可 |
| 框架选型的核心原则？ | 不要为了用框架而用。根据场景复杂度、团队经验、安全要求、可观测性需求选择。简单场景用轻量方案，复杂场景再上框架 |
| 为什么自研 Harness 而不是直接用框架？ | ① 框架抽象层过多导致调试困难 ② 性能损耗 ③ 定制化受限 ④ 版本迭代快。生产环境常见"框架做编排 + 自研 Harness 管安全" |

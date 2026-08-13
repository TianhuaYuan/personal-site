---

title: "Checker成本考量"

tags:

  - agent方法论

  - prompt-2.0

  - maker-checker

created: "2026-07-21"

---

# Checker 成本考量

> 本条目是「Prompt 2.0 方法论」的第 11 部分，对应学习清单条目 2.4.3。

>

> **前置依赖**：Maker-Checker概念、Maker-Checker落地

> **为以下铺垫**：Skill Engineering、Loop Engineering

---

## 一、核心观点

> **成本考量**：Checker 用低成本模型（Haiku）就够了，硬门禁交给确定性测试。

> 让米其林大厨来「尝咸淡」是浪费——尝一口不需要他亲自炒。Checker 干的活（核对标准、找不一致）比 Maker（写复杂实现）轻得多。

---

## 二、定义与原理（类比先行）
### 2.1 类比：大厨与试菜员

Maker 像主厨，要设计、要颠勺、要火候——贵得有理。Checker 像试菜员，只要舌头灵、对照清单打钩——没必要也是主厨。把 checker 换成 lighter 模型，质量不掉、账单大降。

### 2.2 为什么 Checker 可以用便宜模型？

- **Maker 的任务**：生成复杂代码、设计方案（重推理）

- **Checker 的任务**：核对是否符合标准、找不一致（轻判断）

- Checker 不需要最强推理，只需要「对清单 + 找茬」

---

## 三、模型选型（经验分层）

| 层级 | 模型 | 成本 | 适用 |

|------|------|------|------|

| **L1** | Haiku | 低 | 简单验证、规则检查 |

| **L2** | Sonnet | 中 | 复杂审查、跨模型互审 |

| **L3** | Opus | 高 | 最终决策、关键判断 |

**推荐组合**：Maker = Opus/Sonnet（生成），Checker = Haiku（验证），硬门禁 = 确定性测试（pytest/ruff/mypy）。

> 注：L1/L2/L3 为按能力-成本的经验分层，并非某篇论文的严格结论；「Checker 用 lighter 模型」的合理性来自下方一手指南。

---

## 四、示例（精简版）

```python

maker_model = "claude-opus"      # 生成复杂修复方案

checker_model = "claude-haiku"   # 核对标准 / 找不一致

run("pytest && ruff check . && mypy .")  # 硬门禁：确定性测试

```

---

## 五、优劣势

- ✅ 成本显著下降，质量靠「独立视角 + 确定性测试」兜底

- ✅ 跨模型互审（Claude 写、Codex 审）还能补盲区

- ❌ 太弱的 Checker 可能漏掉强 Maker 也忽略的错；关键决策仍用强模型

---

## 六、最新研究与企业数据（2024–2026）

- **Anthropic《Claude 4 prompt engineering best practices》(2025/2026)**：指出 Plan-and-Execute 等架构中，**子任务可以交给更小、更便宜、更聚焦的模型**，大模型只在（重新）规划与最终回答时调用——即「生成贵、核对便宜」的工程依据。

- **Anthropic《Building Effective Agents》(2024-12)**：强调从最简方案起步，并可在工作流的「简单步骤」路由到轻量模型以控成本。

- **（来源待核实：GitHub Copilot「小模型执行 + 大模型导师」的配对模式，出自 Anthropic 在 AI Engineer 大会的分享，二手报道转述，未定位到一手发布；引用时建议标注为「据报道」。）**

---

## 七、学习资源

- **权威指南**：Anthropic《Claude 4 prompt engineering best practices》

- **研究**：Anthropic《Building Effective Agents》

- **进阶**：[[12-MakervsChecker自检|Maker vs Checker 自检]] · [[16-选型判断标准|选型判断标准]]

---

## 核心要点

| 要点 | 速记 |

|------|------|

| 核心 | Checker 用便宜模型，硬门禁用确定性测试 |

| 类比 | 大厨炒菜、试菜员尝咸淡 |

| 组合 | Maker=Opus/Sonnet，Checker=Haiku，门禁=pytest/ruff/mypy |

| 依据 | Anthropic：子任务可交更小更便宜的模型 |

| 提醒 | 关键决策仍用强模型；弱 Checker 会漏 |

**下一篇**：[[12-MakervsChecker自检|Maker vs Checker 自检]]——为什么不用同一个 Agent 既当 Maker 又当 Checker？

---

## 参考来源（一手链接 · 可溯源深挖）

- **Anthropic (2025/2026)**《Claude 4 prompt engineering best practices》：[docs.anthropic.com/.../claude-4-best-practices](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/claude-4-best-practices)

- **Anthropic (2024-12)**《Building Effective Agents》：[anthropic.com/engineering/building-effective-agents](https://www.anthropic.com/engineering/building-effective-agents)

## 速记卡（面试闪卡）

**Q1：一句话讲清「Checker 成本考量」到底是什么？**

A：Checker 成本考量：核对活交给便宜模型 Haiku，硬门禁交给确定性测试，别让大厨尝咸淡。

**Q2：一、核心类比 —— 怎么理解？**

A：Maker 像主厨要颠勺火候，贵得有理；Checker 像试菜员只对照清单打钩，没必要也是主厨。把 checker 换成 lighter 模型，质量不掉、账单大降。让米其林尝咸淡是浪费（maker-checker）。

**Q3：二、为什么能便宜 —— 怎么理解？**

A：Maker 干重推理（写复杂实现），Checker 干轻判断（核对标准、找不一致），只需“对清单+找茬”。所以 Checker 用 Haiku 足够，关键决策才上 Opus。像质检员不必是工程师（task complexity）。

**Q4：三、模型分层 —— 怎么理解？**

A：L1 Haiku 低成本的简单验证、L2 Sonnet 中成本的复杂审查、L3 Opus 高成本的最终决策。推荐：Maker=Opus/Sonnet、Checker=Haiku、硬门禁=pytest/ruff/mypy。分层即省钱（tiered modeling）。

**Q5：四、优劣势 —— 怎么理解？**

A：✅ 成本大降，质量靠独立视角+确定性测试兜底，跨模型互审还能补盲区。❌ 太弱的 Checker 可能漏强 Maker 也忽略的错，关键决策仍用强模型。便宜要有底线（cost-quality tradeoff）。

**Q6：核心速记主线有哪些？**

- 类比：主厨生成、试菜员核对

- 分层：L1 Haiku / L2 Sonnet / L3 Opus

- 组合：Maker=Opus/Sonnet，Checker=Haiku，门禁=测试

- 依据：Anthropic 子任务可交更小更便宜模型

- 提醒：关键决策仍用强模型，弱 Checker 会漏

**口诀**

A：主厨炒菜贵得夸

试菜员尝不用怕

Haiku 核对把钱化

硬门禁守稳如塔

## 相关链接

- 系列清单：[[学习路线图/Agent方法论与产品思维学习路线图|Agent 方法论与产品思维学习路线图]]

- 上一层级：[[00-Agent方法论与产品思维|Prompt 2.0 方法论 · 索引]]

- 同主题：[[09-Maker-Checker概念|Maker-Checker 概念]] · [[12-MakervsChecker自检|Maker vs Checker 自检]]


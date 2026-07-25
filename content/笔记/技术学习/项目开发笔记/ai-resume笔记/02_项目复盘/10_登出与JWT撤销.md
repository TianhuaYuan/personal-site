---
title: "登出与简历智能分析"
tags:
  - 项目笔记
  - ai-resume
created: "2026-07-21"
---

# 登出与简历智能分析

> **电梯陈述**：「处理了两个前端缺口——登出不调后端导致 JWT 30分钟内仍有效、MCP 分析工具前端无入口。用  完成：后端 15/15、前端 34/34 全绿。核心决策：登出静默容错（后端挂了也不影响）、REST 端点包装  而非前端直调 JSON-RPC、Service 层抽取共享逻辑。」

## 一、背景

项目有两个缺口：

1. **登出安全**：前端 `logout()` 只清 localStorage，从不调后端撤销 JTI。已签发的  在过期前仍有效，登出形同虚设。
2. **分析入口**：后端  的 `analyze_resume` 工具很完备了，但前端没入口触发——等于工具有但用不起来。

## 二、逐个讲：问题 → 怎么想 → 怎么解

### 登出调后端撤销 JTI

**问题**：`logout` 只 `localStorage.removeItem("access_token")`，后端 `/auth/logout` + `revoke_token(jti)` 从未被触发。

**关键设计——静默容错**：登出时**先调后端，再清本地**。后端调用用 try/catch 包起来静默吞掉错误——不管后端成功还是失败，本地  一定要清掉。

```typescript
export async function logout() {
  const token = localStorage.getItem("access_token");
  if (token) {
    try { await api.post("/api/v1/auth/logout"); }
    catch { /* 静默吞掉：本地清理必须继续 */ }
  }
  localStorage.removeItem("access_token");
  localStorage.removeItem("refresh_token");
}
```

放弃方案 A（后端失败就报错不让登出）的原因：用户体验优先，用户主动登出的场景 token 泄露概率几乎为零。这是 fail-safe 设计。

### MCP 工具的前端入口

**问题一：MCP 是 JSON-RPC，前端怎么调？**

两条路：
- **方案 A：前端直调 MCP JSON-RPC** — 好处是省后端工时；坏处是前端要组装 JSON-RPC 请求、鉴权方式不同、跟项目  风格不一致
- **方案 B：后端加  端点包装 MCP** — 好处是前端调用一致、错误码标准（ 状态码）、鉴权统一、MCP 协议变化不影响前端

**选 B**。MCP 是给  用的协议，前端  就该用 REST。

**问题二：MCP 工具和  端点代码重复？**

抽取 `services/analyze_service.py` 做共享  层，MCP 工具和  端点都是薄包装调它：

```text
analyze_service.py（核心逻辑）
    ↑            ↑
 工具       端点
```

好处：DRY、可测试（ 可独立测）、可扩展（加第三个入口直接调 service）。

**问题三：6 个设计技能叠加的约束怎么整合？**

按"硬禁令"和"必须项"分类找交集：
- 禁 em-dash / emoji / 纯黑 `#000000` / `h-screen` / generic spinner
- 必须三态、Phosphor 图标、`active:scale-[0.98]` 触觉反馈、`prefers-reduced-motion` 降级

综合后  用 skeleton shimmer（非转圈）、Phosphor 图标、三态展示。

## 三、关键决策与取舍

| 决策 | 选择 | 取舍 |
|------|------|------|
| 登出容错 | 后端失败静默，本地必登出 |  分钟  窗口，但主动登出场景风险极低 |
|  调用方式 |  端点包装 | 多写一个薄端点，但前端调用一致 |
| 代码复用 |  层抽取 | 多一个文件，但 DRY+可测+可扩展 |
| 加载态 |  骨架屏 | 比转圈信息量大，纯  零  开销 |
| 图标库 | Phosphor | 零依赖变一个  包，但 tree-shaking 友好 |

## 四、踩坑

1. **em-dash 违规**：注释里写了 `——`（中文破折号），撞上设计技能「禁 em-dash」的硬禁令。教训：设计约束是全局的，代码注释也算。
2. **没有 `head`**：`npm test | head -80` 报错，PowerShell 里用 `Select-Object -First`。
3. **警告**：useEffect 触发异步加载后 setState，测试里用 `waitFor` 而非直接断言。
4. **全套测试太慢**：本地只跑相关测试文件快速验证，全套留 CI/CD。

## 五、Q&A

**Q：为什么登出要调后端？只清本地不行吗？** — 只清本地的话  在  分钟内仍有效，被窃取后登出形同虚设。调后端把  加黑名单，立即失效。

**Q：MCP 工具为什么不直接让前端调？** —  是 JSON-RPC 协议，给  用的。前端  就该用 REST，协议解耦、错误处理标准、鉴权统一。

**Q：为什么抽  层？** — DRY、可测试、可扩展。标准分层架构，核心逻辑在中间，入口都是适配器。

**Q：为什么用  不用转圈？** — 骨架屏比转圈信息量大——用户能看到"内容大概长这样"，心理预期更明确。纯  动画，零  开销。


> ▶ 关联技术研读：[[01_技术研读/01_架构概览|01_架构概览]]

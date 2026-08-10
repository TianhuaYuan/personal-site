---
title: "过期提示"
tags:
  - 项目笔记
  - ai-resume
created: "2026-07-21"
---

# 过期提示

## 一、需求回顾

**问题**：原来的代码里，token 过期后要么静默刷新，要么直接 `window.location.href = "/login"` 秒跳登录页，用户体验很差——正在打字呢页面突然跳走了，连个招呼都不打。

**目标**：
1.  过期时弹模态弹窗，告诉用户"登录已过期"，由用户主动点击"去登录"再跳转
2.  即将过期时（提前预警）弹温和提示，给个倒计时，用户可以选择"延长登录"（刷新 token）或"忽略"
3.  流式问答的  也要走同样的弹窗逻辑
4.  模式下弹窗不能通过 Esc、点击遮罩关闭——只能点按钮

---

## 二、架构设计思路

### 2.1 为什么用「全局事件总线」而不是直接调用 Context？

一开始想过直接在 `client.ts` 里 import `AuthContext` 然后 dispatch，但这样有个问题：

- **层依赖 React Context → 反向依赖**。API 层本来应该是纯逻辑层，不应该依赖 React。一旦哪天不用  了（比如迁移到 Vue），API 层代码就废了。
- 而且 `client.ts` 是普通  模块，不是  组件，没法直接 `useAuth()`。

所以最终方案是用 **CustomEvent** 做全局事件总线：

```text
┌─────────────┐   dispatch event    ┌─────────────────┐
│  client.ts   │ ──────────────────→ │  AuthProvider   │
│  qa.ts       │                     │  (React 组件)   │
│  (纯 TS)     │ ←────────────────── │                 │
└─────────────┘   notifySessionXxx   └─────────────────┘
```

-  层只负责 `window.dispatchEvent`，完全不感知 React
-  里 `window.addEventListener` 监听事件，然后更新  控制弹窗
- 解耦得很干净

### 2.2 为什么  模式"只弹一次"？

用了个 `warningShownRef = useRef(false)` 来标记。原因：

- 如果不做防抖，token 快过期时，每次重新渲染/每次接口调用都可能触发一次 warning，弹窗会反复弹，用户烦
- 但如果用户点了"延长登录"并且成功了，就得重置这个标记，下一次快过期时再弹
- 登出/重新登录也要重置

---

## 三、踩过的坑

### 坑 1：vi.mock 把  也  没了

**现象**：改完 `qa.ts` 后测试一直报 `handler 调用了  次`，明明代码里 `notifySessionExpired()` 写得好好的。

**排查**：突然想到，`qa.test.ts` 顶部有个 `vi.mock("./client", ...)`，里面只  了 `api` 和 `refreshToken`，没写 `notifySessionExpired`。那默认会变成 `undefined` 啊！

**验证**：在 `notifySessionExpired` 里打 console.log，测试里果然没打印——函数根本就没被调用到，因为它是 undefined。

**解决方案**：用 `vi.importActual` 把真实模块拿进来，只覆盖需要  的部分：

```ts
vi.mock("./client", async () => {
  const actual = await vi.importActual<typeof import("./client")>("./client");
  return {
    ...actual,           // 保留所有真实实现，包括 notifySessionExpired
    api: { post: vi.fn(), get: vi.fn(), delete: vi.fn() },
    refreshToken: vi.fn(),
  };
});
```

**教训**：`vi.mock` 不是"增量替换"，而是"整个模块替换"。如果你只写了几个属性，其他属性就都是 undefined。

### 坑 2：测试  删了但 vi.mocked 还在用

**现象**：把 `clearSessionAndRedirect` 的  删了之后，13 个测试全挂了，报 `ReferenceError: clearSessionAndRedirect is not defined`。

**原因**：`beforeEach` 里还有 `vi.mocked(clearSessionAndRedirect).mockReset()`，变量名没了当然报错。

**解决方案**：把 `beforeEach` 里的那行也删了，同时 `vi.mock` 里也删掉。

**教训**：改  的时候，一定要全局搜一下这个名字还在哪里被引用了——测试里、mock 里、注释里，都可能有。

### 坑 3：expired 模式不能用  控制关闭

一开始想偷懒，直接用 `onOpenChange` 来处理关闭逻辑，但很快发现：

-  模式下，用户按  或者点遮罩会触发 `onOpenChange(false)`
- 如果在 `onOpenChange` 里直接 `setOpen(false)`，那  模式就关不掉了——但  内部会尝试关闭吗？
- 不对，Radix  的 `onOpenChange` 是"状态变化时的回调"，不是"请求关闭时的回调"。它是受控的，只要 `open`  是 true，Dialog 就不会关。

所以正确做法是：
- 让  完全受控（`open` 由父组件控制）
-  模式下，`onOpenChange` 里什么都不做（或者只处理  的情况）
- 只有点击"去登录"按钮才真正跳转

---

## 四、关键设计决策

### 决策 1：提前  分钟预警

选  分钟（ 秒）是拍脑袋的，但有依据：
- 太短（比如  分钟）：用户可能还没反应过来就过期了
- 太长（比如  分钟）：频繁弹窗打扰用户
-  分钟是个比较常见的行业惯例（很多银行网站、企业系统都是  分钟预警）

这个值做成了常量 `WARNING_BEFORE_SECONDS`，以后产品说改就改。

### 决策 2：warning 模式倒计时用组件内部 state，不用全局

`SessionExpiredDialog` 内部维护 `countdown` state，每秒减 1。为什么不放在  里？

- 倒计时是纯  行为，跟业务逻辑无关
- 放在  里会导致每秒都触发  重渲染，所有消费  的组件都跟着重绘
- 组件内部自己维护就行，父组件只需要传一个初始 `remainingSeconds`

### 决策 3：倒计时到  不自动切换到  模式

倒计时结束后，没有自动切到  模式。原因：

- 用户可能点了"忽略"把弹窗关了，那  过期后应该由后续的  请求  来触发  弹窗
- 如果用户一直在看着弹窗倒计时，那到  了应该...其实也可以切，但增加复杂度
- 简单原则：倒计时只是个视觉提醒，真正的过期判定以  返回  为准

---

## 五、代码结构概览

```text
src/
├── components/
│   └── SessionExpiredDialog.tsx      # 弹窗组件（expired/warning 两模式）
│   └── SessionExpiredDialog.test.tsx #  个测试用例
├── context/
│   └── AuthContext.tsx               # 加了  状态 + 事件监听 + 定时器
├── api/
│   ├── client.ts                     # 加了 notifySessionExpired/Warning，401 改发事件
│   └── qa.ts                         # SSE  也改发事件
└── App.tsx                           #  里渲染弹窗
```

---

## 六、测试覆盖清单

### 组件（ 个）

| # | 测试用例 | 模式 | 验证点 |
|---|---------|------|-------|
| 1 | 渲染标题「登录已过期」 | expired | 文案正确 |
| 2 | 只显示「去登录」按钮 | expired | 没有忽略按钮 |
| 3 | 点「去登录」调用 onPrimary | expired | 回调触发 |
| 4 | 按  不关闭 | expired |  不被调用 |
| 5 | 点遮罩不关闭 | expired |  不被调用 |
| 6 |  时按钮禁用 + spinner | expired | 防重复点击 |
| 7 | 打开时  不可滚动 | expired | scroll lock |
| 8 | 渲染标题「登录即将过期」 | warning | 文案正确 |
| 9 | 显示两个按钮（延长/忽略） | warning | 都有 |
| 10 | 显示倒计时秒数 | warning | 初始值正确 |
| 11 | 倒计时每秒递减 | warning |  秒后减了 2 |
| 12 | 点「延长登录」调用 onPrimary | warning | 回调触发 |
| 13 | 点「忽略」调用 onIgnore | warning | 回调触发 |
| 14 | 按  可以关闭 | warning | 可关闭 |
| 15 | 点遮罩可以关闭 | warning | 可关闭 |

### 集成层面

- `qa.test.ts` 里的 SSE  刷新失败 → 验证触发了 `session:expired` 事件

---

## 七、可改进的地方

1. **倒计时精度**：用 `setInterval` 做倒计时有累积误差，长时间不准。可以改用 `Date.now()` 差值来算剩余时间。但该场景最多  分钟，误差可以忽略。

2. **多标签页同步**：现在只监听了当前页面的事件。如果用户开了多个标签页，一个标签页过期了，其他标签页不知道。可以用 `storage` 事件或者 `BroadcastChannel` 做跨标签页同步。

3. **即将过期的刷新时机**：现在是"倒计时到  了也不自动刷新"，必须用户点按钮。也可以加个"自动续期"选项，检测到用户有操作（鼠标移动、键盘输入）就静默刷新。

---

## 八、总结

这次任务的核心不是写个弹窗——弹窗谁都会写。真正的技术点在于：

1. **解耦**：用  让  层不依赖 React，保持架构干净
2. **用户体验**：从"秒跳登录页"变成"友好提示 + 用户主动操作"，减少用户困惑
3. **测试思维**：两模式分开测、关闭方式分别测、loading 状态测、倒计时测，15 个用例覆盖主要场景
4. **坑是最好的老师**：vi.mock 的"全量替换"特性、import 删除后的残留引用，这些都是踩过一次就不会忘的



> ▶ 关联技术研读：[[01_技术研读/01_架构概览|01_架构概览]]

---
title: "为什么需要Hooks"
created: "2025-07-12"
tags:
  - 八股文
  - react
  - react-ts-js
---

# 为什么需要Hooks
## 一、为什么需要 Hooks？



> 延伸提问"Hooks 是什么"，不是在要定义——是在问"**它解决了什么问题**"。两句话讲清楚即可，重点在后面几节的底层原理。



React 早期只有类组件——`class Foo extends React.Component`，写 `this.state`、`render()`、`.bind(this)`。你没写过，因为 2019 年 Hooks 出来后，函数组件一统天下。



Hooks 解决了三个麻烦：**① 没有 `this` 了**，函数组件根本不需要 `this`；**② 逻辑复用不用套娃了**，类组件要写 HOC 多层嵌套，Hooks 一个自定义 Hook 搞定；**③ 同一逻辑不用拆到三个方法了**，类组件"订阅→取消订阅"散落在三个生命周期里，Hooks 一个 `useEffect` 写完 setup + cleanup。



```tsx

// ✅ 你实际写的：函数组件 + Hooks —— 没有 class、没有 render()、没有 this

function ChatRoom({ roomId }: { roomId: string }) {

  useEffect(() => {                                  // ① 副作用集中在这个函数里

    const sub = chatAPI.subscribe(roomId);           // ② 订阅（setup）

    return () => sub.unsubscribe();                  // ③ 取消订阅（cleanup）——和 setup 写在一起

  }, [roomId]);                                      // ④ roomId 变了自动清理旧订阅 + 建新订阅

}

```



| | 类组件（你不写这个） | Hooks（你写的） |

| :--- | :--- | :--- |

| 状态 | `this.state = {}` + `this.setState()` | `const [x, setX] = useState(0)` |

| 副作用 | 散落在三个生命周期方法里 | 集中在 `useEffect`，setup 和 cleanup 写在一起 |

| 复用逻辑 | HOC 套了一层又一层 | 自定义 Hook，一行调用 |

| 有无 `this` | 有，要 `.bind(this)` | 没有 `this`，纯函数 |



> **一句话**：Hooks 让你用普通函数就能写带状态、带副作用的组件——不需要 `class`、`constructor`、`render()`、`this`。函数每次渲染重新执行，但 React 在底层用链表帮你把状态"记"住了——下面几节就是讲这个"记"是怎么实现的。



---





## 相关链接



- 📋 目录：[[00-React]]

- 📚 学习清单：[[八股文学习清单]]

- 🔗 [[八股文笔记/React-TS-JS/JavaScript/19-async与await|TS async与await]] — Hooks与async/await都是"用语法糖简化复杂模式"

- 🔗 [[笔记/八股文笔记/Python/并发/15-async-await本质|Python async与await]] — Python和JS的异步模型对比

- 🔗 [[八股文笔记/React-TS-JS/React/05-useState底层机制|useState底层机制]] — Hooks的核心：状态管理




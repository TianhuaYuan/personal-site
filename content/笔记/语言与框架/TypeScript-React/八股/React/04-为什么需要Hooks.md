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





## 速记卡（面试闪卡）



**Q1：一句话讲清「为什么需要Hooks」到底是什么？**

A：Hooks 让函数组件用普通函数就能写带状态、带副作用的组件，不用 class/this（React Hooks）。



**Q2：一、为什么需要 Hooks？ —— 怎么理解？**

A：React 早期只有类组件（this.state、render()、bind(this)），2019 年 Hooks 出来后函数组件一统天下。Hooks 解决三个麻烦：① 没 this；② 逻辑复用不套娃（免 HOC）；③ 同一逻辑不用拆三个生命周期。像把"带身份的门禁卡"换成"人人可用的工牌"（React Hooks）。



**Q3：Hooks 解决的三麻烦：无 this、复用不套娃、逻辑不拆散 —— 怎么理解？**

A：① 没有 this——函数组件根本不需要 this，不用 bind；② 逻辑复用不套娃——类组件要写 HOC 多层嵌套，Hooks 一个自定义 Hook 搞定；③ 同一逻辑不拆散——类组件"订阅→取消订阅"散落三个生命周期，Hooks 一个 useEffect 写完。像把三处零散活合并到一处（自定义 Hook，Custom Hook）。



**Q4：useEffect 如何把副作用收进一处（setup + cleanup） —— 怎么理解？**

A：类组件里"订阅→取消订阅"要散在 componentDidMount / componentWillUnmount 两个生命周期。Hooks 一个 useEffect：进去 subscribe，return 清理函数做 unsubscribe；roomId 变了 React 自动先清旧再建新。setup 和 cleanup 写在一起，一眼看懂配对（副作用，Side Effect）。



**Q5：没有 this 与自定义 Hook 如何替代 HOC 套娃 —— 怎么理解？**

A：类组件每个方法要 .bind(this)，Hooks 纯函数没有 this，状态靠 React 底层链表按调用顺序"记住"。复用逻辑：类组件写高阶组件 HOC 一层包一层，Hooks 抽成自定义 Hook，组件里一行 const x = useChatRoom() 就能用。从"包汉堡"变"插一行"（高阶组件，HOC）。



**Q6：核心速记主线有哪些？**

- Hooks 让函数组件带状态/副作用，告别 class 与 this

- 三麻烦：无 this、逻辑复用不套娃、同一逻辑不拆散

- useEffect 把订阅/清理收进一处（setup+cleanup）

- 自定义 Hook 一行调用替代 HOC 多层嵌套



**口诀**

A：类组件 this 来绑，Hooks 纯函数清爽；

状态副作用一处写，不散三生命周期；

复用免套娃 HOC，自定义 Hook 一行；

2019 函数组件，Hooks 一统天下。



## 相关链接





- 📋 目录：[[00-React]]



- 📚 学习清单：[[八股文学习路线图]]



- 🔗 [[语言与框架/TypeScript-React/八股/JavaScript/19-async与await|TS async与await]] — Hooks与async/await都是"用语法糖简化复杂模式"



- 🔗 [[语言与框架/Python/八股/并发/15-async-await本质|Python async与await]] — Python和JS的异步模型对比



- 🔗 [[语言与框架/TypeScript-React/八股/React/05-useState底层机制|useState底层机制]] — Hooks的核心：状态管理




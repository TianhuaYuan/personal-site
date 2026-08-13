---

title: "React Compiler新趋势"

created: "2025-07-12"

tags:

  - 八股文

  - react

  - react-ts-js

---



# React Compiler新趋势

## 一、React Compiler（2026 新趋势——进阶亮点）





> 这是今年最大的新变量。可能会试探性问一句："听说过 React Compiler 吗？"





### 1.1 它是什么？





React Compiler（原 React Forget）——**编译时自动优化**。简单说：你写的代码不变，编译器在构建时自动插入 memo / useMemo / useCallback，**你不用再手写了**。





```mermaid



graph LR



  subgraph 你写的代码



    Input["function Todoitems<br/>const filtered = items.filter<br/>return List items=filtered"]



  end



  subgraph SGjg9ag["编译器输出[编译器输出的代码 简化示意]"]



    Output["function Todoitems<br/>const $ = _c2  缓存槽<br/>const filtered = items.filter<br/>if $[0] !== items   自动比对依赖<br/>return List items=$[1]"]



  end



  Input -->|React Compiler| Output



```





### 1.2 三问





**问："有了 React Compiler，还需要手写 useMemo 吗？"**





> 答：大部分场景不需要——编译器自动处理。但编译器不是万能的——副作用在渲染中、可变状态更新、条件调用 Hooks 等反模式会导致编译器 **bailout（放弃优化）**，这些边界情况仍然需要手动优化。





**问："哪些代码会导致编译器 bailout？"**





> 答：三条红线。编译器看到这些 → 直接跳过该组件 → 回退到你的手写优化。





**红线 ①：渲染期间执行副作用**





```tsx



// ❌ 直接在函数体里发请求——编译器看到副作用，放弃优化



function TodoList() {



  fetch("/api/todos")                          // 渲染期间发请求！



    .then(res => res.json())



    .then(data => setTodos(data));             // 触发 state 更新 → 可能无限循环





  return <ul>{/* ... */}</ul>;



}





// ✅ 副作用放 useEffect——渲染之后再执行



function TodoList() {



  useEffect(() => {



    fetch("/api/todos").then(res => res.json()).then(setTodos);



  }, []);



  return <ul>{/* ... */}</ul>;



}



```





**红线 ②：可变状态更新**





```tsx



// ❌ 直接改原数组/对象——编译器不知道"变了"，无法追踪依赖



function TodoList() {



  const [todos, setTodos] = useState([]);





  function addTodo(text: string) {



    todos.push({ text, done: false });         // 直接 push——改了原数组！



    setTodos(todos);                           // 引用没变 → React 认为没更新 → 不渲染



  }



}





// ✅ 不可变更新——创建新引用，编译器能正确追踪



function addTodo(text: string) {



  setTodos(prev => [...prev, { text, done: false }]);  // 新数组，新引用 ✅



}



```





**红线 ③：条件调用 Hooks**





```tsx



// ❌ Hook 放在 if 里——调用顺序不固定，编译器无法建立 Hook 链表



function Profile({ isAdmin }: { isAdmin: boolean }) {



  if (isAdmin) {



    const [permissions, setPermissions] = useState([]);  // isAdmin=false 时不调用！



  }



  const [name, setName] = useState("");                  // Hook 索引偏移 → 全乱套



}





// ✅ Hooks 永远在顶层，无条件调用



function Profile({ isAdmin }: { isAdmin: boolean }) {



  const [permissions, setPermissions] = useState([]);     // 始终第 0 个 Hook



  const [name, setName] = useState("");                   // 始终第 1 个 Hook



  // 逻辑判断放在 Hook 之后



}



```





> 三条红线总结：**渲染期间别干副作用的事、更新状态别改原值、Hooks 别放条件里**。写"惯用 React"——编译器就能帮你自动优化。





**问："现在就开始用吗？"**





> 答：React 19 已内置 Compiler（Beta），React 20 预计正式稳定。2026 求职实践中提到它属于"进阶亮点"，不需要深入源码，但要能解释它做什么 + 什么情况下失效。





### 1.3 一句话





> React Compiler = 把"该不该加 memo"的判断从人脑转移到编译器。前提是写"惯用 React"——纯渲染、不可变更新、Hooks 规则。





---





## 速记卡（面试闪卡）



**Q1：一句话讲清「React Compiler新趋势」到底是什么？**

A：React Compiler 是编译时自动插 memo 的优化器，写"惯用 React"就免手写 useMemo。



**Q2：它是什么（compile-time auto-memo） —— 怎么理解？**

A：类比：以前为防止重复渲染要手写 useMemo/useCallback，像每次过河自己搭桥。React Compiler（原 React Forget）是编译器在构建时自动插入这些缓存，你写的代码一字不变——桥它替你搭好，过河直接走。（Auto memoization）



**Q3：还需手写吗（bailout） —— 怎么理解？**

A：类比：大部分场景编译器自动处理，但三条红线会让它"放弃优化"（bailout）：① 渲染期间干副作用（直接 fetch）、② 可变状态更新（直接 push 原数组）、③ 条件调用 Hooks（Hook 放 if 里）。踩中任一条就回退到你手写优化。（Bailout red lines）



**Q4：三条红线详解（side-effect / mutation / conditional hook） —— 怎么理解？**

A：类比：红线①副作用放 useEffect 而不是函数体；红线②状态更新用 [...prev, x] 建新引用而非改原数组（否则引用没变 React 以为没更新）；红线③ Hooks 永远顶层无条件调用，否则 Hook 链表索引乱套。写"惯用 React"编译器才帮得上忙。（Idiomatic React）



**Q5：现在要上吗（React 19/20） —— 怎么理解？**

A：类比：React 19 已内置 Compiler（Beta），React 20 预计正式稳定。求职里它属于"进阶亮点"——不要求读源码，但要能讲清它做什么（自动 memo）+ 什么情况失效（三条红线）。像自动挡车：会开即可，不要求会造。（Gradual adoption）



**Q6：核心速记主线有哪些？**

- React Compiler：构建时自动插 memo/useMemo/useCallback，代码不变

- 大部分场景免手写，但三条红线会触发 bailout 放弃优化

- 红线：渲染副作用 / 可变更新 / 条件调用 Hook

- 惯用写法：副作用进 useEffect、不可变更新、Hooks 顶层

- 现状：React 19 Beta 内置，20 稳定；属进阶亮点非必深究



**口诀**

A：Compiler 自动桥，memo 不用描；

三条红线碰，优化立刻消；

副作用进 effect，新引用才牢；

惯用 React 写，编译帮您捞。



## 相关链接





- 📋 目录：[[00-React]]



- 📚 学习清单：[[八股文学习路线图]]




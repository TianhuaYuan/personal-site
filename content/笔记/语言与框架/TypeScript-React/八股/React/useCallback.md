---

title: "useCallback缓存函数引用"

created: "2025-07-12"

tags:

  - 八股文

  - react

  - react-ts-js

---



# useCallback缓存函数引用

## 三、useCallback —— 缓存函数引用





> useMemo 冻值，useCallback 冻函数。原理一样——依赖不变就返回上次的那份，不创建新的。





### 3.0 为什么函数也要"冻"？





React 组件每次渲染都会重新执行函数体。在组件里写的函数，每次渲染都是新引用：





```tsx



function Parent() {



  // 第一次渲染 → 创建函数A



  // 第二次渲染 → 创建函数B（逻辑一样，但是新函数！Object.is(A, B) → false）



  // 第三次渲染 → 创建函数C（又是新的！）



  const handleClick = () => setCount(c => c + 1);





  return <Child onClick={handleClick} />;



  //     ↑ 每次传的都是新函数 → Child 的 memo 失效



}



```





**用 useCallback 冻住**：





```tsx



function Parent() {



  // 第一次渲染 → 创建函数A → 存入缓存



  // 第二次渲染 → 依赖[]没变 → 跳过创建 → 直接返回函数A



  // 第三次渲染 → 同上，还是函数A



  const handleClick = useCallback(() => setCount(c => c + 1), []);





  return <Child onClick={handleClick} />;



  //     ↑ 永远是同一个函数引用 → memo 生效 ✅



}



```





### 3.1 依赖数组怎么填





```tsx



// ① 空数组 []：函数只创建一次，永远不变



//    ⚠️ 里面不能读 state——会闭包陷阱（见 3.3）



const fn = useCallback(() => { /* 不依赖任何 state */ }, []);





// ② 有依赖 [a, b]：这些值变了才重建函数



const fn = useCallback(() => {



  console.log(count);   // 函数里读了 count → 必须写进依赖



}, [count]);             // count 变 → 函数重建 → 拿到最新 count





// ③ 用函数式更新：不需要闭包捕获 state，依赖可以空



const fn = useCallback(() => {



  setCount(c => c + 1);  // React 把最新值传给你，不靠闭包



}, []);                   // 依赖可以为空 ✅



```





### 3.2 useMemo vs useCallback —— 一张表记住





> **最高频对比题**。





```tsx



// 这两行等价：



useCallback(fn, [deps]);       // 缓存函数本身



useMemo(() => fn, [deps]);     // 缓存"返回 fn"的计算结果 → 还是 fn



```





| | useMemo | useCallback |



| :--- | :--- | :--- |



| 缓存什么 | 计算结果（**值**） | 函数**引用** |



| 返回值 | 缓存的值 | 缓存的函数（可直接调用） |



| 典型签名 | `useMemo(() => value, deps)` | `useCallback(() => { doSth(); }, deps)` |



| 使用场景 | 昂贵计算、传给子组件的对象/数组 | 传给子组件的回调（配合 React.memo） |



| 用错会怎样 | 没缓存到该缓存的值 → 性能浪费 | 函数引用还是每次变 → 子组件 memo 失效 |





**一句话讲清（20 秒版）**：





> "useMemo 缓存计算结果，useCallback 缓存函数引用。useCallback(fn, deps) 本质上就是 useMemo( => fn, deps) 的语法糖。关键区别：useMemo 返回的是值——用于避免重复计算；useCallback 返回的是函数——用于保持函数引用不变，配合 React.memo 阻止子组件无谓重渲染。"





### 3.3 三个高频对比





```tsx



// ① useCallback 配合 React.memo —— 标准搭档



const Parent = () => {



  const [count, setCount] = useState(0);



  const handleClick = useCallback(() => setCount(c => c + 1), []);



  //                ↑ 函数引用永远不变





  return <Child onClick={handleClick} />;



  //     ↑ Child 被 memo 包裹，props 不变 → 不重新渲染



};





// ② useMemo 缓存对象 —— 给 memo 子组件传对象



const Parent = () => {



  const [count, setCount] = useState(0);



  const config = useMemo(() => ({ pageSize: 20, theme: "dark" }), []);



  //                ↑ 对象引用永远不变





  return <Table config={config} />;  // Table = React.memo(Table)



};





// ③ 有人问"用 useCallback 包起来是不是性能更好？"——不是



// ❌ 滥用：不传给子组件，自己内部用的函数不需要 useCallback



function Bad() {



  const handleScroll = useCallback(() => {



    console.log("scrolling...");



  }, []);  // 没传给子组件——白缓存了，还多了 deps 比较开销





  useEffect(() => {



    window.addEventListener("scroll", handleScroll);



    return () => window.removeEventListener("scroll", handleScroll);



  }, [handleScroll]);  // 倒是这里依赖数组需要稳定引用



}



```





上面的第三个例子实际**需要用** useCallback——因为它在 useEffect 的依赖数组里。这是一个微妙的反例：不传子组件，但需要函数引用稳定来避免 effect 反复重建。总结：





| 需要 useCallback 吗？ | 判断条件 |



| :--- | :--- |



| ✅ 需要 | 传给 `React.memo` 包裹的子组件 |



| ✅ 需要 | 在 useEffect / useMemo / 其他 Hook 的依赖数组里 |



| ✅ 需要 | 作为自定义 Hook 的返回值（调用方可能加依赖） |



| ❌ 不需要 | 只在当前组件内部使用，且不在依赖数组里 |





### 3.4 useCallback 的闭包陷阱





> 和 useEffect 闭包陷阱是同一个根因——函数捕获了旧的 state 值。可能会进一步追问："useCallback 依赖数组为空，有问题吗？"





```tsx



function Counter() {



  const [count, setCount] = useState(0);





  // ❌ 依赖数组为空 → 闭包捕获了初始 count=0 → 永远 log 0



  const handleClick = useCallback(() => {



    console.log(count);  // 永远是 0！因为函数只在首次渲染时创建



  }, []);





  // ✅ 方案一：把 count 放进依赖数组——count 变了就重建函数



  const handleClick = useCallback(() => {



    console.log(count);  // 每次 count 变化后拿到最新值



  }, [count]);





  // ✅ 方案二：用函数式更新——不需要 count 出现在闭包里



  const handleClick = useCallback(() => {



    setCount(c => c + 1);  // React 把最新 state 传给你，不需要闭包捕获



  }, []);



}



```





**为什么会这样？** useCallback 缓存的是**创建那一刻的函数实例**。依赖数组为空 → 只在首次渲染时创建一次 → 函数里的 `count` 永远是首次渲染时的值（0）。每次 `setCount` 后组件重新渲染，但 handleClick 还是那个旧函数。





> **一句话讲清**："useCallback 和 useEffect 一样有闭包陷阱——依赖数组为空时，函数内部的 state 值定格在首次渲染。解法要么加依赖，要么用函数式更新绕开闭包。"





---





## 速记卡（面试闪卡）



**Q1：一句话讲清「useCallback缓存函数引用」到底是什么？**

A：useCallback 把函数"冻"住——依赖不变就返回同一份引用，让 React.memo 子组件不白重渲染；本质是 useMemo 的语法糖。



**Q2：为什么函数也要"冻" —— 怎么理解？**

A：组件每次渲染都重跑函数体，里面写的函数每次都是新引用（Object.is 为 false），传给 memo 子组件就让它白重渲染。useCallback 把函数存进缓存，依赖没变就吐同一份。好比给函数发了张长期工牌（stable reference, React.memo）。



**Q3：依赖数组怎么填 —— 怎么理解？**

A：空 [] 函数只建一次（但里面不能读会变 state，否则闭包陷阱）；有依赖 [a,b] 这些值变了才重建；用函数式更新 setC(c=>c+1) 则依赖可为空。填错要么拿旧值要么白缓存（dependency array）。



**Q4：useMemo vs useCallback —— 怎么理解？**

A：useMemo 缓存"计算结果"（值），useCallback 缓存"函数本身"（引用）；其实 useCallback(fn,deps) 就是 useMemo(()=>fn,deps) 的语法糖。区别：值用于避免重复算，引用用于稳住子组件（memoization）。



**Q5：闭包陷阱与何时真需要 —— 怎么理解？**

A：依赖为空时函数定格在首次渲染的 state（永远 log 0），解法加依赖或函数式更新。真需要 useCallback 的三种情况：传给 memo 子组件、在 hook 依赖数组里、作为自定义 hook 返回值；内部自用且不在依赖里则不须（closure trap）。



**Q6：核心速记主线有哪些？**

- useCallback 冻函数引用，配 React.memo 阻止子组件无谓重渲染

- 依赖空数组=只建一次；读变 state 会闭包陷阱，用函数式更新绕开

- 与 useMemo 区别：一个缓存值、一个缓存引用，后者是前者的糖

- 真需要的三处：memo 子组件、hook 依赖数组、自定义 hook 返回值



**口诀**

A：useCallback 把函数锁，

依赖不变引用牢；

memo 子件免重描，

闭包陷阱更新好。



## 相关链接





- 📋 目录：[[00-React]]



- 📚 学习清单：[[八股文学习路线图]]




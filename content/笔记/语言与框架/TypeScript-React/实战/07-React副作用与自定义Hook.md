---

title: "React副作用与自定义Hook"

created: "2025-07-12"

tags:

  - 技术学习

  - typescript

  - react

  - useref

  - 自定义hook

---

# React副作用与自定义Hook

> 写给初学者：每个概念都从"为什么需要它"开始讲，配合通俗比喻和完整可运行代码。

---

## 一、useEffect 进阶
### 1.1 闭包陷阱

闭包是 JS 的核心机制——**函数能"记住"它出生时周围的变量**，即使外部代码已经跑完了。

```tsx

// 闭包的最简例子

function createPrinter(name) {

  return function () {

    console.log(name);  // 这个 name 来自 createPrinter 的参数

  };

}

```

**闭包 + useEffect = 陷阱**：

每次组件渲染都创建一套新的变量和新的闭包。`useEffect(fn, [])` 把第一次渲染的 fn 钉死了。

```mermaid

flowchart LR

  R1["渲染 #1：count=0"] --> A["产生闭包 A（count=0）"]

  A --> E1["useEffect(fnA, [])<br/>执行 fnA，启动定时器"]

  R2["渲染 #2：count=1"] --> B["产生闭包 B（count=1）"]

  B --> E2["useEffect(fnB, [])<br/>依赖 [] 没变，跳过 fnB"]

  E1 -.-> Timer["定时器回调 → 仍使用闭包 A（count=0）"]

```

**三种解法**：

| 场景 | 推荐方案 |

|:---|:---|

| 只是用 state 算新 state | `setCount(prev => prev + 1)`（函数式更新，绕过闭包） |

| 需要读 state 做其他事 | 加依赖数组 |

| 需要读最新值但不方便重建 Effect | useRef |

```tsx

// 方案 1：加依赖数组

useEffect(() => {

  const timer = setInterval(() => {

    console.log("count:", count);

  }, 1000);

  return () => clearInterval(timer);

}, [count]);

// 方案 2：函数式更新（推荐）

useEffect(() => {

  const timer = setInterval(() => {

    setCount(prev => prev + 1);  // prev 来自 React，不依赖闭包

  }, 1000);

  return () => clearInterval(timer);

}, []);

// 方案 3：useRef 存最新值

const countRef = useRef(count);

countRef.current = count;

useEffect(() => {

  const timer = setInterval(() => {

    console.log("count:", countRef.current);  // 永远是最新值

  }, 1000);

  return () => clearInterval(timer);

}, []);

```

### 1.2 cleanup 函数的三种触发时机

| 时机 | 触发条件 | 清理的是哪次 Effect |

|:---|:---|:---|

| 依赖变化 | 依赖变了，准备执行新 Effect 之前 | 上一次 setup 返回的 cleanup |

| 组件卸载 | 组件从 DOM 移除 | 最后一次 setup 返回的 cleanup |

| 依赖不变 | 不触发 | 不做任何事 |

```tsx

// 典型 cleanup：事件监听

function MouseTracker() {

  const [position, setPosition] = useState({ x: 0, y: 0 });

  useEffect(() => {

    function handleMove(e: MouseEvent) {

      setPosition({ x: e.clientX, y: e.clientY });

    }

    window.addEventListener("mousemove", handleMove);

    return () => {

      window.removeEventListener("mousemove", handleMove);

    };

  }, []);

  return <div>鼠标位置: {position.x}, {position.y}</div>;

}

```

> **为什么 `handleMove` 必须定义在 useEffect 里面？** `removeEventListener` 需要和 `addEventListener` **完全相同的函数引用**。定义在 useEffect 里面 + `[]` 空依赖 → 函数只创建一次 → 装和拆用的是同一个引用。

---

## 二、useRef — 可变但不触发渲染

有两个需求是 useState 做不到的：

- 记一个值，但变了不要触发重新渲染

- 直接拿到 DOM 元素

> **通俗比喻**：useState = 教室里的黑板（写上去大家都看到，擦掉重写大家也看到）；useRef = 你兜里的便签（写给自己看的，别人不知道）。

`useRef(初始值)` 返回一个普通 JS 对象：`{ current: 初始值 }`。

```tsx

const countRef = useRef(0);

countRef.current = 5;         // ✅ 直接赋值

console.log(countRef.current); // ✅ 直接读取 → 5

```

### 2.1 核心用途 1：存储可变值（不触发渲染）

```tsx

function TimerWithRef() {

  const [seconds, setSeconds] = useState(0);

  const intervalRef = useRef(null);

  function start() {

    if (intervalRef.current !== null) return;  // 防重复启动

    intervalRef.current = setInterval(() => {

      setSeconds(prev => prev + 1);

    }, 1000);

  }

  function stop() {

    if (intervalRef.current !== null) {

      clearInterval(intervalRef.current);

      intervalRef.current = null;

    }

  }

  return (

    <div>

      <p>{seconds} 秒</p>

      <button onClick={start}>开始</button>

      <button onClick={stop}>停止</button>

    </div>

  );

}

```

**三句话总结**：

1. **定时器归浏览器管**——启动后独立运行，React 渲染不干涉它

2. **渲染归 state 管**——`setSeconds` 触发渲染，`intervalRef.current = 1` 不触发

3. **ref 贯穿所有渲染**——同一个 `{ current }` 对象，跨渲染不变

### 2.2 核心用途 2：DOM 引用

```tsx

function AutoFocus() {

  const inputRef = useRef(null);

  useEffect(() => {

    if (inputRef.current) {

      inputRef.current.focus();  // 直接操作 DOM

    }

  }, []);

  return <input ref={inputRef} type="text" placeholder="自动聚焦" />;

}

```

**分阶段**：

- 渲染阶段：`inputRef.current = null`（DOM 还没创建）

- 提交阶段：React 创建 `<input>` → `inputRef.current = 那个 <input>`

- useEffect 执行：`inputRef.current.focus()`（DOM 已存在）

### 2.3 state vs ref 速查

| | useState | useRef |

|:---|:---|:---|

| 修改触发渲染 | 会 | 不会 |

| 多次渲染间保留值 | 会 | 会 |

| 直接操作 DOM | 不行 | 可以 |

| 异步回调中取最新值 | 小心闭包 | 一直是最新的 |

---

## 三、自定义 Hook
### 3.1 命名规则：必须以 use 开头

```tsx

useWindowSize     → React 知道这是 Hook

windowSize        → React 不知道，不做检查

```

### 3.2 基本模式

```tsx

function useWindowSize() {

  const [size, setSize] = useState({

    width: window.innerWidth,

    height: window.innerHeight,

  });

  useEffect(() => {

    function handleResize() {

      setSize({ width: window.innerWidth, height: window.innerHeight });

    }

    window.addEventListener("resize", handleResize);

    return () => window.removeEventListener("resize", handleResize);

  }, []);

  return size;

}

// 任何组件只要一行就拿到窗口大小

function MyComponent() {

  const { width } = useWindowSize();

  return <div>窗口宽度: {width}</div>;

}

```

### 3.3 自定义 Hook 可以组合

```tsx

function useUserPreferences(userId: number) {

  const windowSize = useWindowSize();

  const theme = useLocalStorage("theme", "light");

  const fontSize = useLocalStorage("fontSize", "16");

  return { ...windowSize, theme, fontSize };

}

```

React 不关心你套了几层 Hook——它只按调用顺序把所有 `useState`、`useEffect` 登记在同一个链表上。

### 3.4 useLocalStorage 完整实现

```tsx

function useLocalStorage(key: string, initial: any) {

  // 1. 初始化，从 localStorage 读值

  const [stored, setStored] = useState(() => {

    try {

      const item = window.localStorage.getItem(key);

      return item ? JSON.parse(item) : initial;

    } catch {

      return initial;

    }

  });

  // 2. 值变化时自动写入 localStorage

  useEffect(() => {

    try {

      window.localStorage.setItem(key, JSON.stringify(stored));

    } catch {

      console.warn("写入 localStorage 失败");

    }

  }, [key, stored]);

  // 3. 返回和 useState 一样的 API

  return [stored, setStored];

}

// 使用

const [theme, setTheme] = useLocalStorage("theme", "light");

```

---

## 速查表

| 概念 | 一句话解释 | 关键代码 |

|:---|:---|:---|

| 闭包陷阱 | Effect 记住的是旧值 | 把用到的变量加进依赖数组 |

| `useRef` | 可变但不触发渲染的便签 | `const r = useRef(0)` → `r.current` |

| ref + DOM | 直接拿到 DOM 元素 | `<div ref={myRef}>` |

| 自定义 Hook | 抽取可复用的逻辑 | `function useXXX() { ... }` |

| cleanup | 组件卸载时擦屁股 | `return () => clearInterval(t)` |

---

## 速记卡（面试闪卡）

**Q1：一句话讲清「React副作用与自定义Hook」到底是什么？**

A：useEffect 负责渲染后的副作用，useRef 存可变值且不触发重渲染，自定义 Hook 是把这类逻辑抽成 use 开头的复用函数。

**Q2：useEffect 的闭包陷阱与 cleanup？ —— 怎么理解？**

A：生活比喻：useEffect 里的函数像被“钉”在第一次渲染的快照上，用的还是那一次的变量，依赖没写全就拿旧值——这就是闭包陷阱。解法是依赖数组写全，或用 useRef 存最新值。cleanup 函数（return 的函数）像退租时的打扫：三种时机触发——①组件卸载、②下次 effect 重跑前先清旧的、③严格模式（StrictMode，开发环境故意多跑一次以暴露漏清理）下多跑一趟。所以订阅要在 cleanup 里取消，否则内存泄漏。

**Q3：useRef 两大用途是什么？ —— 怎么理解？**

A：生活比喻：useRef 像一个 {current} 可变盒子，改它不触发重渲染——和 useState 最关键的区别。useState 是教室黑板（写了大家都看到），useRef 是你兜里便签（写给自己看，别人不知道）。用途一：存可变值（定时器 id、上一次的值），state 一变就重渲、ref 不变；用途二：DOM 引用（useRef 绑 ref={inputRef}，就能 inputRef.current.focus() 直接操作 DOM）。要“界面跟着变”用 state，要“悄悄记点东西”用 ref。

**Q4：自定义 Hook 怎么写？ —— 怎么理解？**

A：生活比喻：自定义 Hook 是命名必须以 use 开头的函数（React 靠这个前缀识别它是 Hook，才能用调用顺序定位、做规则检查）。基本模式：函数里正常调用其他 Hook，返回需要的数据/方法。它能组合——一个自定义 Hook 里调用另一个，比如 useLocalStorage(key) 内部用 useState + useEffect 实现“状态自动同步到 localStorage”。价值：把“有副作用的逻辑”从组件里抽走，组件只管渲染，干干净净。

**Q5：useLocalStorage 怎么实现？ —— 怎么理解？**

A：生活比喻：useLocalStorage 像带自动存档的游戏存档点。思路：①useState 初始化时从 localStorage 读（没值用默认值）；②useEffect 监听值变化，变化时 localStorage.setItem(key, JSON.stringify(val)) 写回；③返回 [value, setValue] 和普通 state 一样的接口。组件里 const [name,setName]=useLocalStorage("name","") 就能自动持久化。注意 JSON 序列化（对象要 stringify/parse），localStorage 是浏览器本地存储（Web Storage，键值对、存字符串）。

**Q6：核心速记主线有哪些？**

- useEffect 闭包陷阱：依赖写全或用 ref 存最新值

- cleanup 三时机：卸载、重跑前、严格模式多跑一趟

- useRef 两用途：可变值不重渲、DOM 引用

- 自定义 Hook：use 开头、可组合、逻辑抽离组件

**口诀**

A：useEffect 管副作用，cleanup 三时机莫忘；

闭包陷阱依赖全，严格模式多跑一趟；

useRef 可变不重渲，DOM 引用它最棒；

自定义 Hook use 起，逻辑抽走组件爽。

## 相关链接

- 目录：[[00-TypeScript]]

- 上一篇：[[06-ReactHooks深入]]

- 下一篇：[[05-React-Router路由系统|React Router路由系统]]

---

→ [[技术学习路线图#React 入门]]


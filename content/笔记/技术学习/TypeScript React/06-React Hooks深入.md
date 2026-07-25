---
title: "React Hooks深入"
created: "2025-07-12"
tags:
  - 技术学习
  - typescript
  - react
  - hooks
---

# React Hooks深入

> 写给初学者：每个概念都从"为什么需要它"开始讲，配合通俗比喻和完整可运行代码。

---

## 一、useState 进阶

### 1.1 为什么「连点三次 +1」可能只加了 1？

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);   // ❌ 三行都等价于 setCount(0 + 1)
    setCount(count + 1);   // 因为 handleClick 执行那瞬间 count 还是 0
    setCount(count + 1);
  }
  // 结果：count = 1，不是 3
}
```

> **通俗比喻**：你让三个人分别"把仓库里的苹果数量 +1 并存回去"。但三个人看的是同一张老清单（写着 0），都以为现在有 0 个，都存了 1 回去——结果仓库还是 1 个。

### 1.2 函数式更新

给 `setCount` 传一个**函数**，React 会用"它内部最真实的最新值"来调用：

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(prev => prev + 1);   // ✅ prev 是 React 给的最新值
    setCount(prev => prev + 1);   // ✅ 这次的 prev 已经是 1
    setCount(prev => prev + 1);   // ✅ 这次的 prev 已经是 2
  }
  // 点击一次 → count: 0 → 3 ✅
}
```

**什么时候用函数式更新**：
- 新值**无关**旧值：`setCount(0)`、`setName(input)`
- 新值**基于**旧值：`setCount(prev => prev + 1)`

### 1.3 对象状态——必须整个换掉

```tsx
// ❌ 直接改属性——React 不会重新渲染！
user.age = user.age + 1;

// ✅ 创建一个新对象
setUser({ ...user, age: user.age + 1 });

// 函数式更新版本（更稳）
setUser(prev => ({ ...prev, age: prev.age + 1 }));
// 注意：箭头函数返回对象要加括号 () => ({ ... })
```

> **通俗比喻**：React 是个只认收据的仓库管理员。你必须交一张**新的入库单**（`{...user, age: 26}`），管理员才肯重新盘点。

### 1.4 数组状态——同样要换新数组

| 想做的事 | 写法 | Python 对照 |
|:---|:---|:---|
| 末尾加一个 | `[...arr, item]` | `arr + [item]` |
| 开头加一个 | `[item, ...arr]` | `[item] + arr` |
| 删掉下标 i 的 | `arr.filter((_, idx) => idx !== i)` | 列表推导 |
| 合并两数组 | `[...a, ...b]` | `a + b` |

### 1.5 初值的惰性初始化

```tsx
// ⚠️ 每次渲染都会执行 computeInitial()，浪费
const [data, setData] = useState(computeInitial());

// ✅ 传函数本身，React 只在第一次渲染时调用
const [data, setData] = useState(() => computeInitial());

// 实际场景——从 localStorage 读初始值
const [name, setName] = useState(() => {
  return localStorage.getItem("myName") || "匿名";
});
```

---

## 二、useState 的三条铁律

**铁律 1：只在组件顶层调用，不能放进 if/循环/函数里**

```tsx
function Good({ isVip }) {
  // ✅ 所有 useState 都在最顶层
  const [a, setA] = useState(0);
  const [b, setB] = useState(isVip ? 1 : 0);
}
```

React 内部用数组按顺序存每个 useState 的值，靠"调用顺序"对应。放进 if 里会导致错位。

> **通俗比喻**：像在食堂打饭按顺序领餐盘。你今天插队少领一个，后面所有人拿到的餐盘就全是别人的。

**铁律 2：不可变更新——永远创建新值，不直接改老值**

```tsx
// ❌ 直接改
user.age = 26; setUser(user);
todos.push("x"); setTodos(todos);

// ✅ 生成新值
setUser({ ...user, age: 26 });
setTodos([...todos, "x"]);
```

**铁律 3：拆分状态，别把所有东西塞一个对象**

```tsx
// ❌ 把无关的状态塞一坨
const [form, setForm] = useState({ name: "", age: 0, loading: false, error: null });

// ✅ 无关的状态分开
const [name, setName] = useState("");
const [age, setAge] = useState(0);
const [loading, setLoading] = useState(false);

// 例外：总是一起变化的放一起
const [pos, setPos] = useState({ x: 0, y: 0 });
```

---

## 三、受控组件

**受控组件 = 输入框的值由 React 状态控制，用户输入立刻同步到状态。**

```tsx
// input
function NameInput() {
  const [name, setName] = useState("");
  return (
    <div>
      <input value={name} onChange={e => setName(e.target.value)} />
      <p>你好, {name}!</p>
    </div>
  );
}

// checkbox（注意是 checked 不是 value）
<input
  type="checkbox"
  checked={agreed}
  onChange={e => setAgreed(e.target.checked)}
/>

// select
<select value={color} onChange={e => setColor(e.target.value)}>
  <option value="red">红色</option>
  <option value="blue">蓝色</option>
</select>

// textarea（React 统一用 value 属性）
<textarea value={bio} onChange={e => setBio(e.target.value)} />
```

**受控组件统一定律**：不管哪种表单元素，都是「一个状态 × value/checked 属性 × onChange 同步」。

> 表单提交时务必 `e.preventDefault()` 阻止页面刷新。

---

## 四、useEffect — 处理副作用

### 4.1 基本结构

```tsx
useEffect(() => {
  // 副作用代码（渲染"完"之后执行）

  return () => {
    // 清理函数（组件消失时、或下次 effect 执行前调用）
  };
}, [依赖1, 依赖2]);
```

三要素：
- **副作用函数**：干正事（发请求、设定时器）
- **清理函数**：收拾烂摊子（清定时器、取消订阅）
- **依赖数组**：告诉 React"什么变了要重新执行"

### 4.2 四种使用模式

| 模式 | 写法 | 用途 |
|:---|:---|:---|
| 只跑一次 | `useEffect(fn, [])` | 初始加载、设全局定时器 |
| 依赖变化才跑 | `useEffect(fn, [id])` | 依赖某些值的副作用 |
| 每次渲染都跑 | `useEffect(fn)` | 几乎不用 |
| 带清理 | `return () => clearInterval(t)` | 定时器/订阅 |

### 4.3 完整的异步数据获取模式（实战必背）

```tsx
function UserList() {
  const [users, setUsers] = useState<User[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    async function load() {
      try {
        setLoading(true);
        const res = await fetch("/api/users");
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const data: User[] = await res.json();
        setUsers(data);
      } catch (e) {
        setError(e instanceof Error ? e.message : "未知错误");
      } finally {
        setLoading(false);
      }
    }
    load();
  }, []);

  if (loading) return <p>加载中...</p>;
  if (error) return <p>出错了: {error}</p>;
  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

> 这个"loading / error / data 三态 + try/catch/finally + 包 async 函数"模式，是 React 里最常见的数据获取写法。

---

## 速查表

| 概念 | 一句话 | 关键代码 |
|:---|:---|:---|
| 函数式更新 | 基于"最新值"计算新值 | `setCount(prev => prev + 1)` |
| 对象状态 | 必须整个换新对象 | `setUser({...prev, age: 26})` |
| 受控 input | value 受状态控制 | `<input value={t} onChange={...} />` |
| `useEffect` | 副作用（请求/定时器/订阅） | `useEffect(fn, deps)` |
| 数据获取三态 | loading/error/data | `useState(true)`/`useState<null>(null)` |

---

## 相关链接

- 目录：[[00-TypeScript]]
- 上一篇：[[03-JSX+函数组件+Props|JSX与组件]]
- 下一篇：[[07-React副作用与自定义Hook]]
- 项目实践：[[笔记/技术学习/项目开发笔记/ai-resume笔记/01_技术研读/01_架构概览|ai-resume: 架构概览]]

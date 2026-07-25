---
title: "React.memo浅比较盾牌"
created: "2025-07-12"
tags:
  - 八股文
  - react
  - react-ts-js
---

# React.memo浅比较盾牌
## 一、React.memo —— 给函数组件加一层"浅比较盾牌"



### 1.1 先看问题——为什么需要 React.memo？



React 的默认行为：**父组件重新渲染 → 所有子组件全部重新渲染**，即使子组件的 props 一个字都没变。



```tsx

// ❌ 没有 memo：父组件 state 变了，Child 即使 props 完全不变也会重新执行

function Parent() {

  const [count, setCount] = useState(0);

  const [name, setName] = useState("张三");  // 和 Child 毫无关系



  return (

    <div>

      <button onClick={() => setCount(count + 1)}>count: {count}</button>

      <Child name={name} />  {/* name 没变，但 Child 还是重新渲染了 */}

    </div>

  );

}



function Child({ name }: { name: string }) {

  console.log("Child 渲染了");  // 点一次按钮就打印一次——浪费

  return <div>{name}</div>;

}

```



**React 为什么这么设计？** 因为 React 不知道你的组件是不是"纯"的——它宁可多渲染一次，也不能漏掉更新。`React.memo` 就是你对 React 的承诺："我这个组件是纯的——props 不变，输出就不变，放心跳过。"



### 1.2 核心原理——浅比较 props



```tsx

// React.memo 包裹后：props 没变 → 跳过渲染

const Child = React.memo(function Child({ name }: { name: string }) {

  console.log("Child 渲染了");  // name 不变就不会打印

  return <div>{name}</div>;

});

```



**React.memo 做了什么**：



```mermaid

graph TD

  A[父组件渲染] --> B[React准备渲染 Child]

  B --> C{Child被memo包裹?}

  C -->|没有| D[直接渲染 默认行为]

  C -->|有| E{浅比较新旧props}

  E -->|每个prop都相同 Object.is| F[跳过渲染 复用上次结果 ✅]

  E -->|有任何一个prop不同| G[重新渲染]

```



**第二个参数 `areEqual`**：默认用浅比较。你也可以传自定义比较函数：



```tsx

const Child = React.memo(

  function Child({ user }: { user: { name: string; age: number } }) {

    return <div>{user.name}, {user.age}</div>;

  },

  (prevProps, nextProps) => {

    // 返回 true = "相等，跳过渲染"；返回 false = "不等，重新渲染"

    // ⚠️ 注意：这个返回值语义和 shouldComponentUpdate 相反！

    return prevProps.user.name === nextProps.user.name

        && prevProps.user.age === nextProps.user.age;

  }

);

```



> **记忆技巧**：`areEqual` 返回 `true` → "确实相等（are equal）" → 跳过渲染。别写反了。



### 1.3 React.memo 失效的四种场景



> **常见**："memo 包裹的组件为什么还是会重新渲染？"



| # | 失效场景 | 根因 | 解法 |

| :---: | :--- | :--- | :--- |

| 1 | 父组件传了**新对象/数组** | 每次渲染创建新引用，浅比较认为 props 变了 | `useMemo` 缓存对象/数组 |

| 2 | 父组件传了**新函数** | 每次渲染创建新函数，引用不同 | `useCallback` 缓存函数 |

| 3 | 子组件**消费了 Context** | memo 只比较 props，不管 context | 拆分组件——把 context 消费上移到 wrapper |

| 4 | 子组件内部有 **useState / useReducer** | 自身的 state 更新触发自身重渲染，memo 拦不住 | 正常的——自身状态当然要渲染 |



**场景 1 详解——对象引用陷阱**：



```tsx

// ❌ 父组件每次渲染都创建新的 style 对象 → memo 失效

function Parent() {

  const [count, setCount] = useState(0);



  return (

    <div>

      <button onClick={() => setCount(count + 1)}>+1</button>

      <Child style={{ color: "red" }} />  {/* ⚠️ 每次渲染都是新对象！*/}

    </div>

  );

}



const Child = React.memo(function Child({ style }: { style: object }) {

  console.log("还是会打印——因为 {} !== {}");  // 浅比较：两个不同对象 → 不等

  return <div style={style}>hello</div>;

});

```



```tsx

// ✅ 用 useMemo 固定对象引用

function Parent() {

  const [count, setCount] = useState(0);

  const style = useMemo(() => ({ color: "red" }), []);  // 引用永远不变



  return (

    <div>

      <button onClick={() => setCount(count + 1)}>+1</button>

      <Child style={style} />  {/* style 引用不变 → memo 生效 ✅ */}

    </div>

  );

}

```



> **一句话**：React.memo 是盾牌，但只挡"值没变的 props"。对象/数组/函数每次渲染都是新引用——盾牌认不出它们是"旧朋友"，照样放行。解法就是用 useMemo / useCallback 把这些引用固定住。



---





## 相关链接



- 📋 目录：[[00-React]]

- 📚 学习清单：[[八股文学习清单]]




---
title: "useMemo缓存计算结果"
created: "2025-07-12"
tags:
  - 八股文
  - react
  - react-ts-js
---

# useMemo缓存计算结果
## 二、useMemo —— 缓存计算结果



### 2.1 核心原理



```tsx

const memoizedValue = useMemo(() => expensiveComputation(a, b), [a, b]);

//                              ↑ 计算函数              ↑ 依赖：这些值没变就不重新计算

```



**React 内部做的事**：



```mermaid

graph TD

  A[渲染时执行到useMemo] --> B{依赖数组[a,b]<br/>和上次一样?}

  B -->|一样 Object.is| C[不执行计算函数<br/>直接返回上次缓存的值 ✅]

  B -->|不一样| D[执行计算函数]

  D --> E[存新值]

  E --> F[返回新值]

```



### 2.2 适用 vs 不适用场景



| 适用 ✅ | 不适用 ❌ |

| :--- | :--- |

| 大数组排序/过滤/映射（O(n) 以上） | 简单加减乘除——缓存开销 > 计算开销 |

| 传给子组件的对象/数组（配合 React.memo） | 副作用操作（那是 useEffect 的事） |

| 派生数据——一个状态衍生出的复杂计算 | 每次渲染本来就要用的值——缓存了也是白存 |



```tsx

// ✅ 适用：大数组过滤——不缓存的话每次渲染都 O(n)

function TodoList({ todos, filter }: Props) {

  const filteredTodos = useMemo(

    () => todos.filter(t => t.status === filter).sort(/* 复杂排序 */),

    [todos, filter]                    // 只有数据或筛选条件变了才重新计算

  );

  return <List items={filteredTodos} />;

}



// ❌ 不适用：简单运算——useMemo 本身也有开销（存依赖、比依赖）

function Bad({ a, b }: Props) {

  const sum = useMemo(() => a + b, [a, b]);  // 多余！a+b 比 useMemo 内部逻辑还快

  return <div>{sum}</div>;

}

```



> **经验法则**：如果你不确定要不要加 useMemo——先不加。等性能出问题了，用 React DevTools Profiler 定位瓶颈再加。**不要提前优化**。



---





## 相关链接



- 📋 目录：[[00-React]]

- 📚 学习清单：[[八股文学习清单]]




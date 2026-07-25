---
title: "useEffect底层机制"
created: "2025-07-12"
tags:
  - 八股文
  - react
  - react-ts-js
---

## 三、useEffect 底层机制



> TS 笔记讲透了 useEffect 的"怎么用"（setup/cleanup/三个阶段）。本节只讲延伸追问的三个底层问题。



### 3.1 Effect 存在哪——也在 Hook 链表上



useEffect 和 useState 共用同一条 Hook 链表。区别是 Hook 节点里存的字段不同：



```mermaid

graph LR

  Fiber[Fiber节点] --> MS[memoizedState]

  MS --> H1[Hook ① useState<br/>.memoizedState=0<br/>.queue=更新队列]

  H1 --> H2[Hook ② useEffect<br/>.memoizedState=effect对象<br/>.tag=标记位<br/>.create=fn setup函数<br/>.destroy=cleanup函数<br/>.deps=[0] 上次依赖数组<br/>.next=null]

```



**effect 对象字段解释**：



| 字段 | 存储内容 | 类比 |

| :--- | :--- | :--- |

| `tag` | 标记位（区分 useEffect / useLayoutEffect） | 快递标签：普通件 vs 当日达 |

| `create` | 你传的 setup 函数 `() => { ... }` | 待执行的指令 |

| `destroy` | setup 返回的 cleanup 函数 | 上一次的"擦屁股"函数 |

| `deps` | 上次渲染的依赖数组 | 参照物——这次拿新的来比 |



### 3.2 useEffect 的执行时机——提交后异步执行



> 先忘掉 React，理解一个常识：**浏览器更新画面是分步骤的**。你改了 DOM（比如改了一个 `<p>` 的文字），浏览器不是立刻画到屏幕上——它要等到合适的时机才"画"（Paint）。就像你在 Word 里打字，字先写入文档（DOM 变了），然后 Word 才刷新显示（Paint）。



用你写过的 `Counter` 组件，跟踪一次 `setCount(1)` 的完整时间线：



```tsx

function Counter() {

  const [count, setCount] = useState(0);



  useEffect(() => {

    document.title = `你点了 ${count} 次`;       // ⑥ 最后做这个，不挡用户

  }, [count]);



  return <button onClick={() => setCount(count + 1)}>   // ① 用户点按钮

    {count}                                            // ⑤ 用户看到新数字

  </button>;

}

```



**慢动作分解（每一步写清楚"用户看到了什么"）：**



```mermaid

sequenceDiagram

  participant User as 用户

  participant React as React

  participant DOM as DOM

  participant Paint as 浏览器Paint

  participant Effect as useEffect



  User->>React: ① 点击按钮

  React->>React: ② setCount(1) → 标记渲染

  React->>React: ③ Render阶段<br/>Counter()重新执行<br/>useState→count=1<br/>生成虚拟DOM

  Note over React: ⚠️ 屏幕无变化<br/>用户看到按钮仍是"0"

  React->>DOM: ④ Commit阶段<br/>写入真实DOM<br/>按钮文字"0"→"1"

  Note over DOM: ⚠️ DOM变了≠屏幕变了<br/>用户仍看到"0"



  DOM->>DOM: useLayoutEffect<br/>同步执行(阻塞Paint)<br/>可量DOM尺寸

  DOM->>Paint: ⑤ 浏览器Paint

  Paint->>User: 用户看到"1" ✅

  Paint->>Effect: ⑥ useEffect异步执行

  Effect->>Effect: document.title='你点了1次'<br/>发请求/记日志

```



**一分钟理解版（一个比喻）：**



```mermaid

graph LR

  A[你 React] --> B[把新海报贴到墙上<br/>Commit: DOM变了]

  C[现场导演] --> D[海报歪不歪? 量一下摆正<br/>useLayoutEffect: 量DOM调DOM]

  E[摄影师] --> F[咔嚓拍照<br/>Paint: 用户看到画面]

  G[后勤] --> H[拍完照后擦桌子扔垃圾<br/>useEffect: 发请求记日志]

```



| | useLayoutEffect | useEffect |

| :--- | :--- | :--- |

| 在 Paint 前还是后？ | Paint **前**，同步执行 | Paint **后**，异步执行 |

| 会阻塞用户看到新画面吗？ | 会（如果里面写耗时操作，页面卡） | 不会（用户先看到新画面） |

| 能读 DOM 尺寸吗？ | ✅ 能，DOM 已更新、还没画 | 读到的是旧画面还是新画面不一定 |

| 什么时候用它？ | 弹窗定位、滚动位置恢复、防止闪烁 | 网络请求、日志上报、写 localStorage |

| 你 99% 代码里用的是？ | ❌ 很少用 | ✅ 就用这个 |



> **一句话**：`useEffect` 让你在"用户已经看到新画面之后"再做额外的事——不挡着用户。`useLayoutEffect` 是"DOM 变了但用户还没看到"那一刻——让你有机会抢在 Paint 之前调整布局，代价是如果写太慢用户会觉得卡。



**一句话讲清（30 秒版）**：



> "useEffect 在浏览器绘制之后异步执行，不阻塞渲染，90% 的场景用它。useLayoutEffect 在 DOM 提交之后、浏览器绘制之前同步执行——只在需要测量 DOM 布局、或者改了 DOM 不想让用户看到闪烁时才用。"



### 3.3 依赖数组比较——为什么是浅比较 + 为什么不能改





#### 先搞懂：什么是浅比较、什么是深比较？（用 Python 讲，2 分钟）



你从 Python 来——这个概念你其实已经会了：



```python

# === 浅比较 = Python 的 is（只比身份证） ===

a = [1, 2, 3]

b = [1, 2, 3]

c = a



print(a is b)   # False ← 浅比较：两个不同的列表对象，内存地址不同 → "不一样"

print(a is c)   # True  ← 浅比较：同一个对象 → "一样"



# === 深比较 = Python 的 ==（递归比内容，逐层往里看） ===

print(a == b)   # True  ← 深比较：a 和 b 长得一模一样，每一个元素都相等



# JS 里对应的：

#   Object.is(a, b) ≈ Python 的 is（浅比较，只比引用）

#   深比较 ≈ 递归调用 == 比每个字段



# === 对象的情况（类比组件的 props 或依赖数组里的对象） ===

class User:

    def __init__(self, name):

        self.name = name



u1 = User("张三")

u2 = User("张三")

print(u1 is u2)  # False ← 浅比较：两个不同的对象

print(u1 == u2)  # False ← Python 默认 == 对自定义对象也是比引用（除非实现 __eq__）

#                           但如果是 dict/list，== 就是深比较了

```



**一张表总结**：



| | 浅比较（React 用的） | 深比较（React 不用的） |

| :--- | :--- | :--- |

| Python 等价 | `a is b` | 递归版 `a == b` |

| 比什么 | 内存地址（引用） | 递归比较每一层内容 |

| `{}` vs `{}` | ❌ 不等（两个不同对象） | ✅ 相等（内容一样） |

| `[1, 2]` vs `[1, 2]` | ❌ 不等 | ✅ 相等 |

| `"abc"` vs `"abc"` | ✅ 相等（基本类型 JS 也比值） | ✅ 相等 |

| 速度 | 极快（就比一个地址） | 慢（要递归遍历所有层级） |

| 为什么 JS 不默认用深比较 | — | 对象可能很深，遍历要时间；而且"内容相同但不同对象"这个边界情况让开发者自己处理更好 |



**核心洞见**：浅比较下，`{}` 和 `{}` 永远是"不等"的——因为它们是两个不同的对象。这就是为什么不能把对象直接写进依赖数组的原因。



**问：为什么 React 用 `Object.is`（浅比较）而不是深比较？**



```mermaid

graph TD

  Why[为什么React用浅比较?] --> P1[① 性能<br/>深比较递归遍历所有层级<br/>依赖可能有复杂对象<br/>每次深比较会变慢]

  Why --> P2[② 可预测性<br/>浅比较行为固定只比引用<br/>开发者能预测<br/>effect何时触发]

  Why --> P3[③ 倒逼好代码<br/>逼你把依赖拆成基本类型<br/>而不是丢一个对象<br/>框架设计引导最佳实践]

  style Why fill:#1a1a2e,color:#fff

```



**一句话**：深比较听起来好，实际会让 React 又慢又不确定。React 选浅比较 = 选"快且可预测"。



---





## 相关链接



- 📋 目录：[[00-React]]

- 📚 学习清单：[[八股文学习清单]]




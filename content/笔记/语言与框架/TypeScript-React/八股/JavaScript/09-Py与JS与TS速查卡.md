---

title: "Py与JS与TS速查卡"

created: "2025-07-12"

tags:

  - 八股文

  - javascript

  - react-ts-js

---



# Py与JS与TS速查卡

## 十、Py / JS / TS 速查卡（一页纸）





| 你要写什么 | Python | JavaScript | TypeScript |
| :--- | :--- | :--- | :--- |
| 声明变量 | `x = 1` | `let x = 1;` | `let x: number = 1;` |
| 常量 | `PI = 3.14`（约定） | `const PI = 3.14;` | `const PI: number = 3.14;` |
| 函数 | `def add(a, b):` | `function add(a, b) {}` | `function add(a: number, b: number): number {}` |
| 箭头/匿名 | `lambda x: x * 2` | `x => x * 2` | `(x: number) => x * 2` |
| 字典/对象 | `{"name": "张三"}` | `{ name: "张三" }` | `interface U { name: string }` |
| 访问属性 | `user["name"]` | `user.name` | `user.name` |
| 不存在属性 | `KeyError` | `undefined`（不报错） | `undefined`（TS 编译时报错） |
| 空值 | `None` | `null` / `undefined` | `null` / `undefined` |
| 列表/数组 | `[1, 2, 3]` | `[1, 2, 3]` | `number[]` |
| 取长度 | `len(arr)` | `arr.length` | `arr.length` |
| 追加 | `arr.append(x)` | `arr.push(x)` | `arr.push(x)` |
| 映射 | `[x*2 for x in arr]` | `arr.map(x => x * 2)` | `arr.map(x => x * 2)` |
| 过滤 | `[x for x in arr if x>1]` | `arr.filter(x => x > 1)` | `arr.filter(x => x > 1)` |
| 数组→字符串 | `"、".join(arr)` | `arr.join("、")` ⚠️反了 | 同 JS |
| 字符串模板 | `f"{a}{b}"` | `` `${a}${b}` `` | 同 JS |
| 条件 | `if x > 0:`（缩进） | `if (x > 0) {}`（大括号） | 同 JS |
| 三元 | `真 if 条件 else 假` | `条件 ? 真 : 假` ⚠️反了 | 同 JS |
| 逻辑运算 | `and` / `or` / `not` | `&&` / `||` / `!` | 同 JS |
| 判断类型 | `type(x) == int` | `typeof x === "number"` | 同 JS |
| 遍历数组 | `for x in arr:` | `for (let x of arr) {}` | 同 JS |
| `self` / `this` | `self`（显式参数） | `this`（隐式，运行时定） | 同 JS |
| 异步 | `async def` / `await` | `async function` / `await` | `async function(): Promise<T>` |





---





## 速记卡（面试闪卡）



**Q1：一句话讲清「Py与JS与TS速查卡」到底是什么？**

A：这份速查卡对照 Python、JavaScript、TypeScript 三种语言在变量、函数、类型等写法上的差异。



**Q2：十、Py / JS / TS 速查卡（一页纸） —— 怎么理解？**

A：像三兄弟说明书：同一件事三种写法并排看——声明变量 Py 用 x=1、JS/TS 用 let x=1（TS 加类型注解 let x:number）；函数 Py 用 def、JS 用 function、TS 加返回类型。一页纸搞定切换成本。Cheat Sheet（速查卡）。



**Q3：二、变量与函数写法对照 —— 怎么理解？**

A：像点菜暗号不同：常量 Py 靠约定 PI=3.14、JS/TS 用 const；箭头函数 Py 是 lambda x:x*2、JS/TS 是 x=>x*2；字典/对象 Py 用 {}、JS/TS 用 {}（TS 还能写 interface 定结构）。记住 TS 是 JS 的超集，加了类型。



**Q4：三、空值与类型判断差异 —— 怎么理解？**

A：像三种"没有"：Py 的 None、JS/TS 的 null/undefined；访问不存在属性 Py 抛 KeyError、JS 给 undefined 不报错、TS 编译期就拦。判断类型 Py 用 type(x)==int、JS 用 typeof x==="number"。TS 的静态类型让错在编译现形。



**Q5：四、异步与 self/this 对照 —— 怎么理解？**

A：像异步三胞胎：三者都用 async/await，但 TS 返回 Promise<T> 标类型。self/this 之别：Py 的 self 是显式参数写死在签名，JS 的 this 隐式、运行时才定（谁调用归谁）。这是从 Py 转 TS 最容易踩的坑。



**Q6：核心速记主线有哪些？**

- 变量：Py 动态赋值、JS let/const、TS 加类型注解

- 函数：Py def / JS function / TS 带参数与返回类型

- 空值：Py None、JS null+undefined、TS 编译期查 undefined

- 异步：三者 async/await，self 显式 vs this 运行时定



**口诀**

A：Py JS TS 三兄弟，变量函数写法异；

Py 动态 JS 用 let，TS 加类型更严谨；

None undefined 各不同，编译查错 TS 先；

self 显式 this 运行时，async await 都齐。



## 相关链接





- 📋 目录：[[00-JavaScript]]



- 📚 学习清单：[[八股文学习路线图]]




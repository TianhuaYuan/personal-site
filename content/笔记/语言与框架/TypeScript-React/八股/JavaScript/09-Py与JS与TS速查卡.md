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





## 相关链接



- 📋 目录：[[00-JavaScript]]

- 📚 学习清单：[[八股文学习清单]]




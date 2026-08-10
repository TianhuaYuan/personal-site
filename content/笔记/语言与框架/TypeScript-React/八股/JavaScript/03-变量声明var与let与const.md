---
title: "变量声明var与let与const"
created: "2025-07-12"
tags:
  - 八股文
  - javascript
  - react-ts-js
---

# 变量声明var与let与const
## 一、变量声明：`var` / `let` / `const`



### 1.1 基础语法



JS 声明变量**必须**在前面加关键字 `var`、`let` 或 `const`。这和 Python 不一样——Python 裸写 `x = 1` 就行，JS 不行。



```javascript

// ===== 三种声明方式 =====



var name = "张三";    // var：老语法，函数级作用域（容易出坑，尽量不用）

let age = 20;         // let：新语法，块级作用域（行为最像 Python 变量）

const PI = 3.14;      // const：常量，声明后不能重新赋值



// ===== Python 对比 =====

// Python:  x = 1        （裸写就行）

// JS:      let x = 1;   （必须加关键字，推荐加分号）

```



### 1.2 `let` 怎么用（最常用的声明方式）



```javascript

// let 声明一个变量，之后可以修改

let count = 0;          // 声明 count，初始值 0

count = 1;              // 修改 count 为 1

count = count + 2;      // count 变成 3



// let 有块级作用域——被 {} 拦住

if (true) {

    let inside = "我在里面";     // inside 只在这个 {} 里有效

    console.log(inside);         // ✅ "我在里面"

}

console.log(inside);             // ❌ ReferenceError: inside is not defined

// Python 里也是一样的，函数内的变量外面访问不到

```



**`let` 的行为 = Python 变量的作用域逻辑**。你在 Python 里怎么理解变量作用域，在 JS 里用 `let` 就怎么理解。



### 1.3 `const` 怎么用（声明常量）



```javascript

// const 声明的变量不能重新赋值

const PI = 3.14;

// PI = 3.15;           // ❌ TypeError: Assignment to constant variable



// ⚠️ 但是：const 只锁住变量本身，不锁住变量指向的对象内容

const user = { name: "张三" };

user.name = "李四";      // ✅ 可以修改对象的属性

user.age = 25;           // ✅ 可以添加属性

// user = {};            // ❌ 不能重新赋值整个变量



// 数组同理

const nums = [1, 2, 3];

nums.push(4);            // ✅ 可以修改数组内容

// nums = [4, 5, 6];    // ❌ 不能重新赋值

```



**理解关键**：`const` 锁的是"变量名指向谁"，不锁"指向的东西能不能改"。把变量想象成一根绳子——`const` 不让绳子绑到别的东西上，但被绑住的东西（对象、数组）自己可以变。



> **Python 对照**：Python 没有 `const`。Python 的变量都是 `let`（可以随时指向新对象）。如果你想让一个值不可改，在 Python 里用元组 `tuple` 或者 `@dataclass(frozen=True)`。



### 1.4 `var` 怎么用（以及为什么尽量不用）



```javascript

// var 和 let 的写法一样，但作用域行为完全不同

var count = 1;

count = 2;               // 可以修改



// ⚠️ var 的坑：穿透 {}

if (true) {

    var inside = "我在里面";    // var 无视 if 的 {}，这仍然是"外面的变量"

}

console.log(inside);            // "我在里面" —— 能访问到！



// 对比 let：

if (true) {

    let insideLet = "我在里面";  // let 被 {} 拦住

}

// console.log(insideLet);      // ❌ ReferenceError —— 访问不到

```



**Python 对照**：Python 里没有 `var` 这个概念，所有变量都是 `let` 的行为（被 `def`/`if`/`for` 的块拦住）。所以你看到 `var` 要高度警惕——它的行为和 Python 直觉完全相反。



**为什么总考 `var`**：因为 `for (var i = 0; ...)` + `setTimeout` 组合的闭包陷阱，根源就是 `var` 穿透了 `{}`。



### 1.5 语句结尾的分号



```javascript

let a = 1;    // 加分号——稳妥，推荐

let b = 2     // 不加分号——大多数情况也行，JS 会自动插入分号（ASI）



// 团队项目一般都加，推荐加

```



---





## 相关链接



- 📋 目录：[[00-JavaScript]]

- 📚 学习清单：[[八股文学习清单]]




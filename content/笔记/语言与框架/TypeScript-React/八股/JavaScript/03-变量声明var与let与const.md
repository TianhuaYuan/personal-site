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

## 速记卡（面试闪卡）

**Q1：一句话讲清「变量声明var与let与const」到底是什么？**

A：JS 用 var/let/const 三种关键字声明变量：var 穿透块级易踩坑，let 块级最像 Python，const 锁绑定不锁内容。

**Q2：var 的坑：穿透块级作用域 —— 怎么理解？ —— 怎么理解？**

A：var 像一栋没围墙的院子，变量一声明就归"整个函数"所有，if/for 的 {} 根本拦不住它。生活类比：你在屋里（块内）放了把伞，屋外（块外）居然还能拿到——这违背了直觉。let 则像带门禁的小区，{} 一关，外面彻底访问不到（ReferenceError）。英文术语：var 是 function-scoped（函数级作用域），let 是 block-scoped（块级作用域）。

**Q3：let 的块级作用域：行为同 Python —— 怎么理解？ —— 怎么理解？**

A：let 的作用域逻辑和 Python 一模一样：被 def / if / for 的块拦住，外面拿不到。生活类比：Python 里函数内定义的变量，出了函数就消失；JS 里 let 出 {} 也消失。这也是为什么从 Python 转 JS，let 最"亲切"。英文术语：block scope、temporal dead zone（暂时性死区，let 声明前访问会报错）。

**Q4：const 锁绑定不锁内容 —— 怎么理解？ —— 怎么理解？**

A：const 像一根绳子，把变量"绑"在某个值上，不让它改绑到别的东西（重新赋值报错），但被绑住的箱子（对象/数组）里面的东西随便改。生活类比：你租了 3 号柜（const box = {}），不能换成 5 号柜（box = {} 报错），但往 3 号柜里塞衣服、取衣服都行（box.name = 'x' 合法）。英文术语：const 锁的是 binding（绑定），不是 value（值）。

**Q5：为什么总考 var + 分号 ASI —— 怎么理解？ —— 怎么理解？**

A：var 是面试题常客，根源在 for(var i)+setTimeout 的闭包陷阱：var 穿透块级，所有定时器共享同一个 i，输出全是最大值。把 var 换成 let 立刻解决。另一个考点是分号：JS 有 ASI（自动插入分号）机制，不写分号大多也行，但团队项目建议显式写上，避免行首括号引发的歧义。英文术语：closure（闭包）、ASI（Automatic Semicolon Insertion）。

**Q6：核心速记主线有哪些？**

- var：函数级作用域，穿透 {}，是闭包陷阱根源

- let：块级作用域，行为同 Python，最推荐

- const：锁变量绑定，对象/数组内容仍可改

- 分号：推荐显式写，JS 用 ASI 自动补

**口诀**

A：var 穿透块级是隐患，函数作用域收不拦；

let 块级最像 Python，出括号就看不见；

const 只绑引用不绑值，对象内容仍可变；

显式分号加为好，ASI 自动补也别嫌。

## 相关链接

- 📋 目录：[[00-JavaScript]]

- 📚 学习清单：[[八股文学习路线图]]


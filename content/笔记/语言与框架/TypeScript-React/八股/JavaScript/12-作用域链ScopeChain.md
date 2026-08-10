---
title: "作用域链ScopeChain"
created: "2025-07-12"
tags:
  - 八股文
  - javascript
  - react-ts-js
---

# 作用域链ScopeChain
## 一、作用域链（Scope Chain）



### 1.1 作用域是什么？



> **作用域 = 变量的"活动范围"**。在哪个范围内可以访问这个变量，出了这个范围就找不到了。



**通俗比喻**：你在自己家里（函数作用域）可以随便拿冰箱里的东西；邻居家（另一个函数）的冰箱你进不去。全局作用域 = 小区公共区域，谁都能用。



### 1.2 三种作用域



> **前置**：`var`、`let`、`const` 都是 JS 声明变量的关键字。`var name = '小明'` 意思是"创建一个变量叫 name，里面放 '小明'"。三者区别看下表。



| 作用域类型 | 定义 | 示例 | 说明 |

| :--- | :--- | :--- | :--- |

| **全局作用域** | 在任何 `{}` 外面定义的变量 | `var a = 1` | 哪里都能访问 |

| **函数作用域** | 在 `function` 内部定义的变量 | `function fn() { var b = 2 }` | 只有函数内部能访问 |

| **块级作用域** | `{}` 内用 `let/const` 定义的变量 | `{ let c = 3 }` | ES6 新增，`var` 没有块级作用域 |



```javascript

// === 三种作用域演示 ===

var globalVar = '全局';           // 全局作用域



function outer() {

    var outerVar = '函数内';      // 函数作用域（var）



    if (true) {

        let blockVar = '块级';    // 块级作用域（let）

        var notBlockVar = '还是函数'; // var 无视 {}，仍然是函数作用域

    }



    console.log(globalVar);     // ✅ '全局' — 全局作用域可访问

    console.log(outerVar);      // ✅ '函数内' — 函数作用域可访问

    console.log(notBlockVar);   // ✅ '还是函数' — var 穿透了 {}

    console.log(blockVar);      // ❌ ReferenceError — let 被 {} 限住了

}

```



> **记忆口诀**：`var` 是函数级，`let/const` 是块级。`var` 能穿透 `if` 的 `{}`，`let/const` 不能。



---



### 1.3 作用域链的查找规则



> **作用域链 = 一层套一层的变量查找链。** JS 引擎找变量时，从当前作用域开始，一层层往外找，找到就停，找到最外层（全局）还没找到就报 `ReferenceError`。



```javascript

var a = '全局';



function outer() {

    var a = 'outer';



    function inner() {

        var a = 'inner';

        console.log(a);   // 'inner' → 先找自己，找到了，停止

    }



    inner();

}



outer();

```



**查找方向永远是由内向外，不会反过来。**



```text

inner 作用域 → outer 作用域 → 全局作用域 → 找不到 → ReferenceError

```



### 1.4 词法作用域（Lexical Scope）— 必问概念



> **JS 是词法作用域（静态作用域）**：函数的作用域在**定义时**就确定了，和**在哪里调用**无关。



```javascript

var name = '全局';



function fn() {

    console.log(name);   // 定义时 name 指向全局，所以输出 '全局'

}



function run() {

    var name = 'run内部';  // 这个 name 在 fn 的词法作用域之外

    fn();                  // fn 定义在全局，不是在 run 内部定义的

}



run();  // 输出 '全局'，不是 'run内部'

```



> **一句话**：函数记住的是自己出生的地方，不是被调用的地方。闭包正是利用了这个特性。



---





## 相关链接



- 📋 目录：[[00-JavaScript]]

- 📚 学习清单：[[八股文学习清单]]




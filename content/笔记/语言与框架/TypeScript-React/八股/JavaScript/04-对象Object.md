---
title: "对象Object数据容器"
created: "2025-07-12"
tags:
  - 八股文
  - javascript
  - react-ts-js
---

# 对象Object数据容器
## 三、对象（Object）—— JS 的数据容器



### 3.1 什么是对象？



JS 的"对象"用 `{}` 包裹，里面是 `key: value` 键值对。**它 ≈ Python 的字典 + class 实例的混合体。**



```javascript

// 创建一个对象（≈ Python 创建一个 dict）

const user = {

    name: "张三",         // 属性名: 值

    age: 20,              // key 可以用引号，也可以不用（推荐不用）

    "phone": "13800138000",  // 带特殊字符的 key 必须加引号

    isVip: false,

};



// 访问属性 —— 两种方式

console.log(user.name);         // "张三" —— 点语法（最常用）

console.log(user["name"]);      // "张三" —— 方括号语法（key 是变量或含特殊字符时用）



// 修改属性

user.age = 21;                  // 修改已有属性



// 添加属性

user.email = "zs@qq.com";       // 直接赋值 = 添加新属性



// 删除属性

delete user.phone;              // 删除属性



// ⚠️ 访问不存在的属性 → undefined，不报错！

console.log(user.height);       // undefined（Python 会 KeyError）

```



**Python 对照核心差异**：

- 访问不存在属性：Python → `AttributeError` 或 `KeyError`；JS → `undefined`（不报错）

- 添加属性：Python → `user["email"] = "zs@qq.com"`；JS → `user.email = "zs@qq.com"`

- JS 对象没有 `dict` 的 `.keys()`、`.values()`、`.items()` 方法（有 `Object.keys(obj)` 替代）



### 3.2 嵌套对象



```javascript

const person = {

    name: "张三",

    address: {                      // 嵌套对象

        city: "北京",

        street: "长安街 100 号",

    },

    hobbies: ["篮球", "编程"],       // 对象里可以放数组

};



console.log(person.address.city);   // "北京" —— 逐层用点访问

// Python 对照：person["address"]["city"]

```



---





## 相关链接



- 📋 目录：[[00-JavaScript]]

- 📚 学习清单：[[八股文学习清单]]




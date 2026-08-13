---

title: "对象Object数据容器"

created: "2025-07-12"

tags:

  - 八股文

  - javascript

  - react-ts-js

---

# 对象Object数据容器

## 一、对象（Object）—— JS 的数据容器
### 1.1 什么是对象？

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

### 1.2 嵌套对象

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

## 速记卡（面试闪卡）

**Q1：一句话讲清「对象Object数据容器」到底是什么？**

A：JS 对象用 {} 包裹键值对，≈Python 字典+class 实例的混合体，是组织数据的核心容器。

**Q2：一、对象是什么、怎么创建访问 —— 怎么理解？**

A：JS 对象用 {} 包裹 key:value 键值对，≈ Python 字典 + class 实例的混合体。访问用点语法 user.name（最常用）或方括号 user["name"]（key 是变量或含特殊字符时用）。比喻：对象像带标签的收纳格，每格贴个名、塞个值，随时按名取。英文：object（对象）、dot notation（点语法）、bracket notation（方括号）。

**Q3：二、访问不存在的属性：JS 不报错 —— 怎么理解？**

A：Python 访问不存在的 key 抛 KeyError，JS 访问不存在属性返回 undefined 不报错——这是新手最大坑。添加属性直接 user.email="..." 赋值即可（Python 用 user["email"]=...）。比喻：JS 的收纳格你问空位它答"没有"（undefined），Python 直接说"这格不存在"报错。英文：undefined、KeyError。

**Q4：三、嵌套对象与数组混装 —— 怎么理解？**

A：对象里能嵌套对象（address.city 逐层点访问）和数组（hobbies: ["篮球","编程"]）。比喻：收纳格里还能套小收纳格，小格里再放一串标签。Python 对照 person["address"]["city"]，JS 用 person.address.city 逐层点。英文：nested object（嵌套对象）。

**Q5：四、和 Python 字典的本质异同 —— 怎么理解？**

A：同：都是键值对容器。异：JS 对象没有 dict 的 .keys()/.values()/.items()，要用 Object.keys(obj) 替代；key 默认是字符串（数字 key 会被转字符串）；对象还能挂方法（≈class 实例）。所以它是"字典+实例"的混合体，不只是纯数据表。英文：Object.keys、mixed type（混合类型）。

**Q6：核心速记主线有哪些？**

- 对象用 {} 包裹键值对，≈字典+实例混合体

- 点语法/方括号访问，key 含特殊字符用方括号

- 访问不存在→undefined 不报错（区别于 Python KeyError）

- 可嵌套对象与数组

- 无 .keys()/.items()，用 Object.keys(obj)

**口诀**

A：JS 对象用花括号，键值成对像收纳；

点语法取最顺手，方括号容特殊名号。

问空位答 undefined，不报错的温柔刀；

嵌套格中套小格，字典实例混合造。

## 相关链接

- 📋 目录：[[00-JavaScript]]

- 📚 学习清单：[[八股文学习路线图]]


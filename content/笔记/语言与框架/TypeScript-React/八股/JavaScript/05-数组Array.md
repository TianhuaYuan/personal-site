---

title: "数组Array列表"

created: "2025-07-12"

tags:

  - 八股文

  - javascript

  - react-ts-js

---

# 数组Array列表

## 四、数组（Array）—— JS 的列表
### 4.1 创建和基本操作

```javascript

// 创建数组（和 Python 的列表几乎一样）

const nums = [1, 2, 3, 4, 5];

const names = ["张三", "李四", "王五"];

const mixed = [1, "hello", true, null];  // 可以混合类型（不推荐）

// 访问元素（下标从 0 开始，和 Python 一样）

console.log(nums[0]);        // 1

console.log(names[2]);       // "王五"

// 长度（注意：是 .length 不是 .length()）

console.log(nums.length);    // 5  ← Python 是 len(nums)

```

### 4.2 常用方法对照

| 操作 | JavaScript | Python |

| :--- | :--- | :--- |

| 取长度 | `arr.length` | `len(arr)` |

| 追加到最后 | `arr.push(x)` | `arr.append(x)` |

| 移除最后一项 | `arr.pop()` | `arr.pop()` |

| 取子数组 | `arr.slice(1, 3)` | `arr[1:3]` |

| 删除/插入 | `arr.splice(1, 2, "x")` | `arr[1:3] = ["x"]` |

| 查找索引 | `arr.indexOf(x)` | `arr.index(x)` |

| 判断包含 | `arr.includes(x)` | `x in arr` |

| 用分隔符拼接 | `arr.join("、")` | `"、".join(arr)` ⚠️ 反了！ |

| 映射 | `arr.map(fn)` | `[fn(x) for x in arr]` |

| 过滤 | `arr.filter(fn)` | `[x for x in arr if fn(x)]` |

| 查找 | `arr.find(fn)` | `next(x for x in arr if fn(x), None)` |

```javascript

// ===== .map() —— React 列表渲染的核心 =====

const nums = [1, 2, 3];

// .map()：对每个元素执行回调函数，返回一个新数组

const doubled = nums.map(function(n) {

    return n * 2;

});

console.log(doubled);  // [2, 4, 6]

// 箭头函数简写（React 里最常见的写法）

const doubled2 = nums.map(n => n * 2);  // 同上

// React 里的用法：

// {items.map(item => <li key={item.id}>{item.name}</li>)}

// ===== .filter() —— 过滤数组 =====

const big = nums.filter(function(n) {

    return n > 1;          // 保留 return true 的元素

});

console.log(big);  // [2, 3]

// ===== .find() —— 查找第一个匹配的元素 =====

const found = nums.find(function(n) {

    return n > 1;          // 找到第一个 >1 的就停

});

console.log(found);  // 2

// ===== .join() —— ⚠️ 调用者反了！ =====

// JS:  数组.join(分隔符)

// Python: 分隔符.join(数组)

const arr = ["A", "B", "C"];

console.log(arr.join("、"));   // "A、B、C"

// Python: "、".join(["A", "B", "C"]) → "A、B、C"

```

---

## 速记卡（面试闪卡）

**Q1：一句话讲清「数组Array列表」到底是什么？**

A：JS 数组（Array）是带方法的动态列表，下标从 0 开始。

**Q2：创建与基本操作 —— 怎么理解？ —— 怎么理解？**

A：像一排带编号的储物柜（下标从 0 开始），往里塞任意类型都不挑。取长度用 .length 不是 .length()。英语：Array / list。

**Q3：常用方法 map/filter —— 怎么理解？ —— 怎么理解？**

A：像流水线工人：.map() 把每个元素加工成新样子返回新数组；.filter() 只挑符合条件的留下。React 列表渲染就靠 .map()。英语：map / filter。

**Q4：find 与 join 的坑 —— 怎么理解？ —— 怎么理解？**

A：.find() 像在队伍里找第一个满足条件的就停；.join() 是"数组.join(分隔符)"——别和 Python 的"分隔符.join(数组)"搞反。英语：find / join。

**Q5：类型混装与边界 —— 怎么理解？ —— 怎么理解？**

A：像杂物箱能混装数字、字符串、布尔（不推荐），但真正清爽的项目会让数组只装一种类型。类型清晰才好维护。英语：typed array。

**Q6：核心速记主线有哪些？**

- 数组是带方法的动态列表，下标从 0（不是 1）开始

- .length 取长度，.push/.pop 增删末尾

- .map 加工、.filter 过滤、.find 取首个匹配

- .join 调用者是数组，别和 Python 写反

**口诀**

A：数组下标零开头，

map 变换 filter 留；

join 调用别写丢，

类型干净才好走。

## 相关链接

- 📋 目录：[[00-JavaScript]]

- 📚 学习清单：[[八股文学习路线图]]


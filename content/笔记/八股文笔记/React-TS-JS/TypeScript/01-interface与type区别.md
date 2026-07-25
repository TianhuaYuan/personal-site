---
title: "interface vs type 区别"
created: "2026-07-21"
tags:
  - 八股文
  - typescript
---

# interface vs type 区别

## 零、一句话定位

TypeScript 里声明"对象形状"有两把锤子：`interface` 和 `type`。**interface 偏"契约"**（可被重复声明合并、可被 implements/extends），**type 偏"别名"**（能表达联合 `|`、交叉 `&`、元组、基本类型别名等 interface 干不了的活）。

> 英文全称：**interface = 接口/结构契约**；**type alias = 类型别名**；**declaration merging = 声明合并**；**intersection = 交叉类型（`&`）**；**union = 联合类型（`|`）**。

## 一、生活化类比

- **interface 像"公司章程"**：可以反复补充条款——你今天写一条、明天再写一条，TS 自动把同名 interface 的条款**合并**进同一份章程。
- **type 像"便利贴别名"**：给一个复杂类型贴个短名，还能把几个类型用 `|` 粘成一坨（联合），或用 `&` 叠成一个超集（交叉）。但同名 type 不能重复声明（便利贴只能贴一张）。

## 二、核心区别速查

| 维度 | interface | type |
|---|---|---|
| 描述对象形状 | ✅ | ✅ |
| 声明合并（同名重复声明自动合并） | ✅ 独有能力 | ❌ 报错 |
| 联合 `A | B` / 交叉 `A & B` | ❌ | ✅ |
| 元组 / 基本类型别名 `type ID = string` | ❌ | ✅ |
| extends / implements | ✅ | ✅（交叉模拟 extends） |

```ts
// interface 声明合并
interface User { name: string; }
interface User { age: number; }   // 合并成 { name; age }，不报错

// type 联合/交叉
type Status = "ok" | "err";
type WithTime<T> = T & { time: number };
```

## 三、怎么选

- 描述"可被扩展、可被类实现"的对象契约 → **interface**（库/框架 API 推荐，方便下游扩展）。
- 需要联合、交叉、元组、工具类型、条件类型 → **type**。
- 两者描述对象时几乎等价，团队统一一种即可，别混着用制造心智负担。

---

> 写给初学者：每个概念都从"为什么需要它"开始讲，配合通俗比喻和完整可运行代码。

---

## 一、接口（interface）

### 1.1 为什么需要 interface？

每次写对象类型都要复制一大串 `{ name: string; age: number; ... }`。

**interface 就是给对象的"形状"起个名字，一次定义，到处复用。**

> **通俗比喻**：interface 就像快递单模板——规定了"收件人""电话""地址"这些字段，所有快递单都按这个模板填。模板只定义一次，但可以印无数张快递单。

### 1.2 基本用法

```typescript
interface User {
    name: string;              // 必填属性
    age: number;               // 必填属性
    email: string;             // 必填属性
}

const zhangsan: User = {
    name: "张三",
    age: 25,
    email: "zhangsan@qq.com",
};
```

如果少一个字段或多一个字段都会报错——interface 要求"形状"精确匹配。

### 1.3 可选属性（?）

```typescript
interface User {
    name: string;
    age: number;
    email?: string;            // 可选——加了 ? 号，可以不传
    phone?: string;
}

const zhangsan: User = {
    name: "张三",
    age: 25,                   // email 和 phone 都可以不传
};

// 可选属性的值类型是 "类型 | undefined"
if (zhangsan.email) {
    console.log(zhangsan.email.toUpperCase());
}
```

### 1.4 只读属性（readonly）

```typescript
interface User {
    readonly id: number;       // 创建后不能修改
    name: string;
    age: number;
}

const zhangsan: User = { id: 1, name: "张三", age: 25 };
zhangsan.name = "张三丰";      // ✅ 普通属性可以改
// zhangsan.id = 2;            // ❌ 报错
```

> **通俗比喻**：`readonly` 就像刻在石头上的字——创建时刻上去，之后改不了。适合用在 ID、创建时间这类不该被修改的字段上。

### 1.5 接口继承（extends）

```typescript
interface Animal {
    name: string;
    age: number;
}

interface Dog extends Animal {
    breed: string;
    bark(): void;
}

const myDog: Dog = {
    name: "旺财",
    age: 3,
    breed: "柴犬",
    bark() { console.log("汪汪！"); },
};

// 多重继承
interface Pet { owner: string; }
interface GuardDog extends Dog, Pet {
    trained: boolean;
}
```

> **通俗比喻**：接口继承就像"抄作业"——Animal 是基础模板，Dog 抄了 Animal 的所有字段，再加自己的品种和叫声。

### 1.6 接口定义函数类型

```typescript
interface MathFunc {
    (a: number, b: number): number;
}

const add: MathFunc = (a, b) => a + b;
const multiply: MathFunc = (a, b) => a * b;
```

---

## 二、类型别名（type）

### 2.1 为什么需要 type？

interface 只能描述对象的形状，但有时候你需要给**任意类型**起名字。`type` 就是做这个的。

> **通俗比喻**：`interface` 是"对象的模具"，`type` 是"类型的外号"——不管什么类型，都能给它起个简短的名字。

### 2.2 基本用法

```typescript
// 给基本类型起名
type ID = string | number;
type Name = string;

// 给对象类型起名
type UserType = {
    name: string;
    age: number;
    email?: string;
};

// 给联合类型起名
type Status = "pending" | "active" | "banned";
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";
```

### 2.3 联合类型详解

```typescript
type StringOrNumber = string | number;

// 可辨识联合（Discriminated Union）模式
type Circle = { kind: "circle"; radius: number };
type Rectangle = { kind: "rectangle"; width: number; height: number };
type Shape = Circle | Rectangle;

function getArea(shape: Shape): number {
    if (shape.kind === "circle") {
        return Math.PI * shape.radius ** 2;
    } else {
        return shape.width * shape.height;
    }
}
```

### 2.4 交叉类型（Intersection Types）

```typescript
type HasName = { name: string };
type HasAge = { age: number };
type UserWithAll = HasName & HasAge;  // 同时有 name 和 age

const user: UserWithAll = {
    name: "张三",
    age: 25,
};
```

**联合 vs 交叉**：
- `A | B`（联合）：是 A **或** B → 取并集，范围变大
- `A & B`（交叉）：是 A **且** B → 取交集，约束变强

---

## 三、interface vs type 怎么选？

| 特性 | `interface` | `type` |
|------|-----------|--------|
| 定义对象形状 | ✅ | ✅ |
| 定义联合类型 | ❌ | ✅ |
| 定义基本类型别名 | ❌ | ✅ |
| 继承（扩展） | `extends` | `&` 交叉类型 |
| 声明合并 | ✅ 同名自动合并 | ❌ 同名报错 |

```typescript
// interface 的声明合并
interface User { name: string; }
interface User { age: number; }
// 最终 User = { name: string; age: number }

// type 不允许同名
// type User2 = { name: string };
// type User2 = { age: number };   // ❌ 报错
```

**经验法则**：
- 定义对象的形状 → 优先用 `interface`（语义更清晰）
- 需要联合类型、基本类型别名 → 用 `type`
- 不确定 → 用 `interface`，大多数场景够用

---

## 速查表

| 概念 | 一句话解释 | 关键代码 |
|------|---------|---------|
| `interface` | 定义对象的形状 | `interface User { name: string }` |
| 可选属性 | 属性可以不传 | `email?: string` |
| 只读属性 | 属性不能修改 | `readonly id: number` |
| 接口继承 | 子接口继承父接口 | `interface Dog extends Animal {}` |
| `type` | 给任意类型起别名 | `type ID = string | number` |
| 交叉类型 | 同时满足多个类型 | `HasName & HasAge` |

---

## 相关链接

- 目录：[[00-TypeScript]]
- 上一篇：[[01-TS基础类型与注解]]
- 下一篇：[[03-泛型generics]]

## 相关链接

- 📋 目录：[[00-TypeScript]]
- 📚 学习清单：[[八股文学习清单#十一、React-TS-JS]]
- 🔗 [[02-泛型与约束与keyof|泛型]] — interface/type 都能写泛型

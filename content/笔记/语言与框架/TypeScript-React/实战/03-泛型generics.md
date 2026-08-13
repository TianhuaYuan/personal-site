---
title: "泛型generics"
created: "2025-07-12"
tags:
  - 技术学习
  - typescript
  - 泛型
---

# 泛型generics

> 写给初学者：每个概念都从"为什么需要它"开始讲，配合通俗比喻和完整可运行代码。

---

## 为什么需要泛型？

```typescript
// ❌ 用 any——丢失类型信息
function identity(value: any): any {
    return value;
}
const result = identity("hello");  // result 是 any，不是 string

// ❌ 为每种类型写一个函数——重复
function identityString(value: string): string { return value; }
function identityNumber(value: number): number { return value; }

// ✅ 泛型——一个函数搞定所有类型，且保留类型信息
function identity<T>(value: T): T {
    return value;
}
const str = identity("hello");  // T = string，返回值类型是 string
const num = identity(42);       // T = number
str.toUpperCase();              // ✅ TypeScript 知道 str 是 string
num.toFixed(2);                 // ✅ TypeScript 知道 num 是 number
```

**泛型就是"类型参数"——调用时才确定是什么类型，但确定后 TypeScript 就记住了。**

> **通俗比喻**：泛型就像快递柜的"万能格口"——不管放进去的是包裹、文件还是外卖，取出来的时候系统都记得你放的是什么，给你对应的处理方式。而 `any` 就是一个"失忆格口"——放进去是啥取出来就忘了。

---

## 泛型函数

```typescript
// 返回数组的第一个元素
function getFirst<T>(arr: T[]): T | undefined {
    return arr[0];
}

const firstNum = getFirst([1, 2, 3]);        // T = number
const firstStr = getFirst(["a", "b", "c"]);  // T = string

// 手动指定类型参数（一般不需要，TS 能自动推断）
const first = getFirst<number>([1, 2, 3]);

// 多个类型参数
function makePair<K, V>(key: K, value: V): [K, V] {
    return [key, value];
}

const pair1 = makePair("name", "张三");      // K=string, V=string
const pair2 = makePair("age", 25);           // K=string, V=number
```

---

## 泛型约束（extends）

泛型默认"什么类型都行"，但有时候你需要限制：

```typescript
interface HasLength { length: number; }

function getLength<T extends HasLength>(value: T): number {
    return value.length;       // ✅ TS 知道 T 一定有 length
}

getLength("hello");            // ✅ string 有 length
getLength([1, 2, 3]);          // ✅ 数组有 length
// getLength(42);              // ❌ number 没有 length

// keyof 约束
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
    return obj[key];
}

const user = { name: "张三", age: 25 };
getProperty(user, "name");     // ✅ 返回 string
getProperty(user, "age");      // ✅ 返回 number
// getProperty(user, "email"); // ❌ user 没有 email 属性
```

> **通俗比喻**：泛型约束就像餐厅的着装要求——"商务休闲装"泛型约束了 T 必须是某种风格，但具体穿衬衫还是西装由你决定。

---

## 泛型接口和泛型类型

```typescript
// 泛型接口
interface ApiResponse<T> {
    code: number;
    message: string;
    data: T;
}

interface User { name: string; age: number; }

// data 是 User
const userResponse: ApiResponse<User> = {
    code: 200,
    message: "成功",
    data: { name: "张三", age: 25 },
};

// data 是 string[]
const listResponse: ApiResponse<string[]> = {
    code: 200,
    message: "成功",
    data: ["张三", "李四"],
};

// 泛型类型别名
type ApiResult<T> = {
    success: boolean;
    data: T;
    error?: string;
};
```

---

## 泛型的实际应用场景

```typescript
// 场景 1：通用的 API 分页响应
interface PaginatedResponse<T> {
    total: number;
    page: number;
    pageSize: number;
    items: T[];
}

type UserListResponse = PaginatedResponse<User>;

// 场景 2：通用的状态管理
interface State<T> {
    value: T;
    loading: boolean;
    error: string | null;
}

// 场景 3：通用的过滤函数
function myFilter<T>(arr: T[], predicate: (item: T) => boolean): T[] {
    return arr.filter(predicate);
}

const numbers = [1, 2, 3, 4, 5, 6];
const evenNumbers = myFilter(numbers, (n) => n % 2 === 0);  // [2, 4, 6]
```

---

## 速查表

| 概念 | 一句话解释 | 关键代码 |
|------|---------|---------|
| 泛型 | 类型参数，调用时才确定类型 | `function f<T>(x: T): T` |
| 泛型约束 | 限制泛型的范围 | `<T extends { length: number }>` |
| `keyof` | 获取对象所有 key 的联合类型 | `keyof User` |
| 泛型接口 | 接口带类型参数 | `interface Box<T> { value: T }` |

---


## 速记卡（面试闪卡）

**Q1：一句话讲清「泛型generics」到底是什么？**
A：泛型是 TypeScript 的"类型参数"，调用时才定类型，确定后仍保留类型信息。

**Q2：为什么需要泛型？ —— 怎么理解？**
A：像快递柜的"万能格口"：放包裹、文件还是外卖，取出来系统都记得你放的是啥，给你对应处理方式；而 `any` 是"失忆格口"，取出来忘了类型。泛型一个函数搞定所有类型还保住类型信息。

**Q3：泛型函数 —— 怎么理解？**
A：像带占位符的公式：`identity<T>(value: T): T` 调用时才把 T 换成 string/number，返回值类型自动跟随。还能 `makePair<K,V>` 一次带多个类型参数，手动指定 `getFirst<number>` 也行。

**Q4：泛型约束（extends） —— 怎么理解？**
A：像餐厅着装要求：泛型默认啥类型都行，但 `<T extends HasLength>` 约束 T 必须有 length；`K extends keyof T` 把 K 锁成对象的合法 key。约束后 TS 才敢让你访问 `.length`、`.name` 这些属性。

**Q5：泛型接口和泛型类型 —— 怎么理解？**
A：像统一包装盒：`interface ApiResponse<T> { data: T }` 一套结构装 User 也装 string[]；`type ApiResult<T>` 同理。实战里分页响应、状态管理 State<T>、过滤函数都靠泛型一次写死、到处复用。

**Q6：核心速记主线有哪些？**
- 动机：any 丢类型、每类型写一遍重复，泛型一个函数保类型
- 函数：`<T>` 占位、自动推断、可多参数 `<K,V>`、可手动指定
- 约束：`extends` 限制范围，keyof 锁合法 key，避免乱访问
- 接口：ApiResponse<T> / State<T> 一套结构复用所有数据类型

**口诀**
A：泛型是占位，调用才定型；
any 失记忆，泛型记得清；
extends 加约束，keyof 锁 key；
接口包一层，复用处处行。

## 相关链接

- 目录：[[00-TypeScript]]
- 上一篇：[[01-interface与type区别]]
- 下一篇：[[04-函数类型与重载]]

---
→ [[技术学习路线图#TypeScript 基础]]

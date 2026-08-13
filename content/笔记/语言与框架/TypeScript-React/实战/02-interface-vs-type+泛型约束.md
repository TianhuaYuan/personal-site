---

title: "interface vs type + 泛型约束"

tags:

  - typescript

  - 技术学习

created: "2026-07-21"

---

# interface vs type + 泛型约束

> **一句话**：interface和type都用于定义类型，但interface支持声明合并，type支持更多类型操作。泛型约束用于限制泛型参数的范围。

## 1. interface vs type
### 1.1 基础对比

```typescript

// interface

interface User {

    name: string;

    age: number;

}

// type

type User = {

    name: string;

    age: number;

};

```

### 1.2 主要区别

| 特性 | interface | type |

|------|-----------|------|

| 声明合并 | ✅ 支持 | ❌ 不支持 |

| 继承 | ✅ 支持extends | ✅ 支持交叉类型 |

| 联合类型 | ❌ 不支持 | ✅ 支持 |

| 元组 | ❌ 不支持 | ✅ 支持 |

| 映射类型 | ❌ 不支持 | ✅ 支持 |

### 1.3 声明合并

```typescript

// interface支持声明合并

interface User {

    name: string;

}

interface User {

    age: number;

}

// 合并后

interface User {

    name: string;

    age: number;

}

// type不支持声明合并

type User = {

    name: string;

};

// type User = {

//     age: number; // 错误：重复标识符

// };

```

## 2. 泛型
### 2.1 基础泛型

```typescript

// 泛型函数

function identity<T>(arg: T): T {

    return arg;

}

// 泛型接口

interface Box<T> {

    value: T;

}

// 泛型类

class Container<T> {

    private items: T[] = [];

    add(item: T): void {

        this.items.push(item);

    }

    get(index: number): T {

        return this.items[index];

    }

}

```

### 2.2 泛型约束

```typescript

// 泛型约束

interface HasLength {

    length: number;

}

function logLength<T extends HasLength>(arg: T): void {

    console.log(arg.length);

}

// 使用

logLength("hello"); // 5

logLength([1, 2, 3]); // 3

// logLength(123); // 错误：number没有length属性

```

### 2.3 泛型默认值

```typescript

// 泛型默认值

interface ApiResponse<T = any> {

    data: T;

    status: number;

}

// 使用默认值

let response: ApiResponse = {

    data: "hello",

    status: 200

};

// 指定类型

let userResponse: ApiResponse<User> = {

    data: { name: "Alice", age: 30 },

    status: 200

};

```

## 3. 高级用法
### 3.1 条件类型

```typescript

// 条件类型

type IsString<T> = T extends string ? true : false;

type A = IsString<string>; // true

type B = IsString<number>; // false

```

### 3.2 映射类型

```typescript

// 映射类型

type Readonly<T> = {

    readonly [P in keyof T]: T[P];

};

interface User {

    name: string;

    age: number;

}

type ReadonlyUser = Readonly<User>;

// 等价于

// type ReadonlyUser = {

//     readonly name: string;

//     readonly age: number;

// };

```

### 3.3 工具类型

```typescript

// 内置工具类型

type Partial<T> = { [P in keyof T]?: T[P] };

type Required<T> = { [P in keyof T]-?: T[P] };

type Pick<T, K extends keyof T> = { [P in K]: T[P] };

type Omit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>;

// 使用

interface User {

    name: string;

    age: number;

    email: string;

}

type UserWithoutEmail = Omit<User, "email">;

// 等价于

// type UserWithoutEmail = {

//     name: string;

//     age: number;

// };

```

## 4. 实际案例
### 4.1 React组件类型

```typescript

// 组件Props类型

interface ButtonProps {

    label: string;

    onClick: () => void;

    variant?: "primary" | "secondary";

}

// 泛型组件

interface ListProps<T> {

    items: T[];

    renderItem: (item: T) => React.ReactNode;

}

function List<T>({ items, renderItem }: ListProps<T>) {

    return (

        <ul>

            {items.map((item, index) => (

                <li key={index}>{renderItem(item)}</li>

            ))}

        </ul>

    );

}

```

### 4.2 API响应类型

```typescript

// 泛型API响应

interface ApiResponse<T> {

    data: T;

    status: number;

    message: string;

}

// 具体响应类型

interface UserResponse extends ApiResponse<User> {}

interface PostResponse extends ApiResponse<Post> {}

// 使用

async function fetchUser(id: number): Promise<UserResponse> {

    const response = await fetch(`/api/users/${id}`);

    return response.json();

}

```

## 5. 常见坑点
### 1. 过度使用any

```typescript

// 问题：使用any类型

function process(data: any): any {

    return data;

}

// 解决：使用泛型

function process<T>(data: T): T {

    return data;

}

```

### 2. 泛型约束过严

```typescript

// 问题：泛型约束过严

function process<T extends string | number>(data: T): T {

    return data;

}

// 解决：放宽约束

function process<T>(data: T): T {

    return data;

}

```

## 核心要点

```typescript

// interface

interface User {

    name: string;

    age: number;

}

// type

type User = {

    name: string;

    age: number;

};

// 泛型

function identity<T>(arg: T): T {

    return arg;

}

// 泛型约束

function logLength<T extends HasLength>(arg: T): void {

    console.log(arg.length);

}

```

## 速记卡（面试闪卡）

**Q1：一句话讲清「interface vs type + 泛型约束」到底是什么？**

A：interface 和 type 都用来定义 TS 类型，但 interface 支持声明合并，type 更灵活。

**Q2：1. interface vs type —— 怎么理解？**

A：像两种户型图纸：interface 是"可扩建图纸"（同一名字多次声明自动合并成一套），type 是"一次性成品图"（别名、联合、映射都能画，但不能合并）。声明合并叫 Declaration Merging。

**Q3：2. 泛型 —— 怎么理解？**

A：像带尺寸参数的模具：函数或接口写一次 <T>，调用时塞进具体类型就得到专用版本，既保类型安全又省重复代码。泛型叫 Generics。

**Q4：3. 高级用法 —— 怎么理解？**

A：像给类型做"变形金刚"：条件类型按条件选类型、映射类型批量改写字段（如加 readonly）、内置工具类型 Partial/Pick/Omit 直接拿来用。条件类型叫 Conditional Types。

**Q5：4. 实际案例 —— 怎么理解？**

A：像给零件贴规格标签：React 组件 Props 用 interface 声明，泛型列表组件 <T> 让渲染逻辑一套通吃各种数据，API 响应用泛型接口统一结构。

**Q6：核心速记主线有哪些？**

- interface 支持声明合并，type 不支持

- 泛型用 <T> 约束参数范围，避免 any

- 条件/映射类型 + 工具类型做类型变换

- 实际用于 Props、泛型组件、API 响应

**口诀**

A：interface 可合并，type 更灵巧；

泛型 <T> 收范围，any 别乱跑；

条件映射工具型，类型变换一手包；

Props 响应泛型写，TS 类型错不了。

## 相关链接

- 📋 目录：[[00-TypeScript-React]]

- 📚 学习清单： React

- 🔗 [[01-TS基础类型与注解|TS基础类型]]

- 🔗 [[03-JSX+函数组件+Props|React组件]]


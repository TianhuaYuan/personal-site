---
title: "TS语法衔接"
created: "2025-07-12"
tags:
  - 八股文
  - javascript
  - react-ts-js
---

# TS语法衔接
## 九、TS 语法衔接 —— 你的 React 代码其实是 JS + 类型



拿出你的 TS/React 笔记里的代码，去掉类型标注，剩下的就是 JS：



```typescript

// ===== 完整示例：interface + Props + 组件 + 调用 =====



// ① 定义数据类型 interface：描述"用户"长什么样

interface User {

    name: string;          // 必填：用户名

    age: number;           // 必填：年龄

    vip?: boolean;         // 可选：是不是 VIP（不传就是 undefined）

}



// ② 定义 Props 类型（业界惯例：组件名 + Props）

interface UserCardProps {

    user: User;            // user 属性的类型是 User（上面定义的 interface）

}



// ③ 组件函数：{ user } 是 JS 解构拆出 props 里的 user

//              : UserCardProps 是 TS 类型标注

function UserCard({ user }: UserCardProps) {

    return (

        <div>

            <h3>{user.name}</h3>

            <p>{user.age}岁</p>

            {user.vip && <span>VIP</span>}

        </div>

    );

}



// ④ 父组件中调用 UserCard，传 user 数据

function App() {

    const zhangsan: User = {               // 创建一个 User 对象

        name: "张三",

        age: 25,

        vip: true,

    };



    return (

        <div>

            <UserCard user={zhangsan} />   {/* 把 user 对象当 props 传给组件 */}

            <UserCard user={{ name: "李四", age: 30 }} />

            {/*                         ↑ 也可以直接写对象字面量 */}

        </div>

    );

}

```



**逐层拆 `{ user }: UserCardProps`：**



```text

{  user  }  :  UserCardProps

    ↓              ↓

 JS 解构      TS 类型标注



解构：从 props 这个对象里，把"user"这个属性的值掏出来，直接当变量用

类型：告诉 TS，"props 对象的形状是 UserCardProps（里面有一个 user: User）"

```



```javascript

// ===== 去掉类型标注，剩下的 JS 如下 =====

// interface User { ... }          ← 类型全部被擦除

// : UserCardProps                 ← 类型全部被擦除

// const zhangsan: User = ...      ← 类型标注没了，变成普通对象



function UserCard({ user }) {

    //               ^^^^ JS 解构：从 props 里拆出 user 属性直接用

    return (

        <div>

            <h3>{user.name}</h3>

            {/*       ↑ 对象属性访问——JS 点语法 */}

            {user.vip && <span>VIP</span>}

            {/*         ↑ && 短路——条件渲染 */}

        </div>

    );

}



function App() {

    const zhangsan = {                    // 普通 JS 对象，没了 : User

        name: "张三",

        age: 25,

        vip: true,

    };



    return (

        <div>

            <UserCard user={zhangsan} />

            {/*         ↑ 把对象当 props 属性传进去 */}

        </div>

    );

}

```



**结论**：你在 React 里写的每一行，底层都在跑 JS 语法。TS 只负责编译阶段检查类型——运行时全是 JS。



---





## 相关链接



- 📋 目录：[[00-JavaScript]]

- 📚 学习清单：[[八股文学习清单]]




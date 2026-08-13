---

title: "TS语法衔接"

created: "2025-07-12"

tags:

  - 八股文

  - javascript

  - react-ts-js

---



# TS语法衔接

## 一、TS 语法衔接 —— 你的 React 代码其实是 JS + 类型





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





## 速记卡（面试闪卡）



**Q1：一句话讲清「TS语法衔接」到底是什么？**

A：TS 语法衔接点破：React 里的 TS 代码去掉类型标注后底层全是 JS；TS 只在编译期查类型，运行时跑的就是普通 JavaScript。



**Q2：一个组件看清 TS + JS —— 怎么理解？**

A：看 UserCard 组件：interface User 描述数据形状，{ user }: UserCardProps 里前半是 JS 解构、后半是 TS 类型标注。像快递单模板（类型）贴在包裹（值）上，运输时模板撕掉，只剩包裹跑全程。



**Q3：去掉类型标注剩 JS —— 怎么理解？**

A：把 interface、: UserCardProps、: User 全擦掉，剩下的 function UserCard({user}){...} 就是纯 JS。解构、点语法、&& 短路条件渲染都是 JS 原生能力。TS 像给 JS 加了层"体检"，体检报告不进产线。



**Q4：为什么理解这点重要 —— 怎么理解？**

A：看懂"TS 只是 JS 的编译期外衣"，就不会被类型语法吓到——调试时记住运行时根本没有类型。新人常以为 TS 改了语言，其实它只是编辑器里的红波浪线工厂。像给自行车贴赛车贴纸，骑起来还是自行车。



**Q5：类型标注写在哪 —— 怎么理解？**

A：TS 标注只出现在变量声明、函数参数/返回值、对象形状上，运行时全被擦除。所以同段逻辑，TS 版和 JS 版产物一致，区别只在开发期能否提前抓错。像两份菜谱，一份加了"别放盐"的提醒，做出来味道一样。



**Q6：核心速记主线有哪些？**

- React 的 TS 代码去标注即 JS，底层跑 JS

- { user }: Props 前半解构(JS)后半标注(TS)

- TS 编译期查类型，运行时类型擦除

- 类型只帮开发期抓错，不改变运行行为



**口诀**

A：TS 外衣裹 JS，编译期里查类型；

去掉标注剩原样，运行时刻全擦净。

解构是 JS 活，标注是 TS 影；

贴纸不动车，类型不进产线。



## 相关链接





- 📋 目录：[[00-JavaScript]]



- 📚 学习清单：[[八股文学习路线图]]




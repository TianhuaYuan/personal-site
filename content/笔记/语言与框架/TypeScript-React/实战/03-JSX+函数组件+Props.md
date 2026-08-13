---

title: "JSX + 函数组件 + Props"

tags:

  - react

  - 技术学习

created: "2026-07-21"

---

# JSX + 函数组件 + Props

> **一句话**：JSX是JavaScript的语法扩展，用于描述UI结构。函数组件是使用函数定义的React组件，Props是组件接收的参数。

## 1. JSX基础
### 1.1 什么是JSX？

```mermaid

graph LR

    A[JSX] --> B[JavaScript XML]

    A --> C[语法扩展]

    A --> D[描述UI]

    A --> E[编译为React元素]

    style A fill:#e1f5fe

```

**JSX**：JavaScript的语法扩展，允许在JavaScript中编写类似HTML的代码。

### 1.2 JSX语法

```jsx

// JSX表达式

const element = <h1>Hello, world!</h1>;

// 嵌入表达式

const name = "Alice";

const element = <h1>Hello, {name}!</h1>;

// 条件渲染

const isLoggedIn = true;

const element = (

    <div>

        {isLoggedIn ? <h1>Welcome!</h1> : <h1>Please log in.</h1>}

    </div>

);

// 列表渲染

const numbers = [1, 2, 3, 4, 5];

const listItems = numbers.map((number) =>

    <li key={number.toString()}>{number}</li>

);

```

## 2. 函数组件
### 2.1 基础函数组件

```jsx

// 基础函数组件

function Welcome() {

    return <h1>Hello, world!</h1>;

}

// 箭头函数组件

const Welcome = () => {

    return <h1>Hello, world!</h1>;

};

```

### 2.2 带Props的函数组件

```jsx

// 带Props的函数组件

function Welcome(props) {

    return <h1>Hello, {props.name}!</h1>;

}

// 使用

<Welcome name="Alice" />

```

### 2.3 解构Props

```jsx

// 解构Props

function Welcome({ name, age }) {

    return (

        <div>

            <h1>Hello, {name}!</h1>

            <p>Age: {age}</p>

        </div>

    );

}

// 使用

<Welcome name="Alice" age={30} />

```

## 3. Props
### 3.1 Props类型

```jsx

// Props可以是任何类型

function UserCard({ name, age, isActive, hobbies }) {

    return (

        <div>

            <h2>{name}</h2>

            <p>Age: {age}</p>

            <p>Active: {isActive ? "Yes" : "No"}</p>

            <ul>

                {hobbies.map((hobby, index) => (

                    <li key={index}>{hobby}</li>

                ))}

            </ul>

        </div>

    );

}

// 使用

<UserCard

    name="Alice"

    age={30}

    isActive={true}

    hobbies={["reading", "coding"]}

/>

```

### 3.2 默认Props

```jsx

// 默认Props

function Welcome({ name = "World" }) {

    return <h1>Hello, {name}!</h1>;

}

// 使用

<Welcome /> // Hello, World!

<Welcome name="Alice" /> // Hello, Alice!

```

### 3.3 children Props

```jsx

// children Props

function Card({ title, children }) {

    return (

        <div className="card">

            <h2>{title}</h2>

            <div className="card-content">

                {children}

            </div>

        </div>

    );

}

// 使用

<Card title="My Card">

    <p>This is the card content.</p>

    <button>Click me</button>

</Card>

```

## 4. 实际案例
### 4.1 用户卡片组件

```jsx

function UserCard({ user }) {

    return (

        <div className="user-card">

            <img src={user.avatar} alt={user.name} />

            <h3>{user.name}</h3>

            <p>{user.email}</p>

            <p>Joined: {user.joinDate}</p>

        </div>

    );

}

// 使用

<UserCard

    user={{

        name: "Alice",

        email: "alice@example.com",

        avatar: "/avatars/alice.jpg",

        joinDate: "2026-01-01"

    }}

/>

```

### 4.2 列表组件

```jsx

function UserList({ users }) {

    return (

        <div className="user-list">

            {users.map(user => (

                <UserCard key={user.id} user={user} />

            ))}

        </div>

    );

}

// 使用

<UserList users={usersArray} />

```

## 5. 常见坑点
### 1. 忘记key属性

```jsx

// 问题：列表渲染没有key

{items.map(item => <li>{item}</li>)}

// 解决：添加key属性

{items.map(item => <li key={item.id}>{item}</li>)}

```

### 2. Props修改

```jsx

// 问题：尝试修改Props

function Welcome({ name }) {

    name = "Bob"; // 错误：Props是只读的

    return <h1>Hello, {name}!</h1>;

}

// 解决：使用state

function Welcome({ name }) {

    const [displayName, setDisplayName] = useState(name);

    return <h1>Hello, {displayName}!</h1>;

}

```

## 核心要点

```jsx

// JSX

const element = <h1>Hello, {name}!</h1>;

// 函数组件

function Welcome({ name }) {

    return <h1>Hello, {name}!</h1>;

}

// 使用

<Welcome name="Alice" />

// children

<Card title="My Card">

    <p>Content</p>

</Card>

```

## 速记卡（面试闪卡）

**Q1：一句话讲清「JSX + 函数组件 + Props」到底是什么？**

A：JSX 是写类 HTML 的 JS 语法扩展，函数组件是用函数定义的 React 组件，Props 是传给组件的参数。

**Q2：1. JSX基础 —— 怎么理解？**

A：像在 JS 里写乐高说明书：JSX 允许在 JavaScript 里写 <h1> 这样的标签，{name} 嵌入表达式，用 ? : 做条件渲染、map 做列表渲染。它最终被编译成 React.createElement 调用。JSX（JavaScript XML，JS 的语法扩展）。

**Q3：2. 函数组件 —— 怎么理解？**

A：像一台小机器：function Welcome() { return <h1/> } 用函数定义，返回 UI。箭头函数也行。接收 props 参数，用解构 { name, age } 直接取，比一坨 props.name 清爽。函数组件是 React 现代写法主力。

**Q4：3. Props —— 怎么理解？**

A：像组件的进料口：Props 可以是任意类型（字符串、数字、数组、甚至 children 子节点）。<Welcome name="Alice"/> 把 name 传进去；默认 Props 用 { name="World" } 兜底；children 让组件能包住别的内容。Props（属性，组件入参）。

**Q5：4. 实际案例 —— 怎么理解？**

A：像拼用户卡片：UserCard 收 user 对象渲染头像名字邮箱；UserList 用 users.map 把每个 user 交给 <UserCard key={id}/>。列表渲染别忘了 key（稳定 id），它是 React 区分元素的身份证，少了会出 bug。

**Q6：核心速记主线有哪些？**

- JSX：类 HTML 语法，编译成 React 元素，支持 {} 表达式

- 函数组件：函数返回 UI，支持解构 Props

- Props：组件入参，支持默认值和 children

- 列表渲染必须给 key，Props 只读不可改

**口诀**

A：JSX 写类 HTML，编译成元素好渲染；

函数组件返回 UI，解构 Props 真方便；

进料口里传数据，默认 children 都能填；

列表 key 不能忘，Props 只读莫去改。

## 相关链接

- 📋 目录：[[00-TypeScript-React]]

- 📚 学习清单： React

- 🔗 [[04-useState+useEffect+自定义Hook|React Hooks]]

- 🔗 [[05-React-Router路由系统|React Router]]


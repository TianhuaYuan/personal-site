---

title: "useState + useEffect + 自定义 Hook"

tags:

  - react

  - 技术学习

created: "2026-07-21"

---

# useState + useEffect + 自定义 Hook

> **一句话**：useState用于管理组件状态，useEffect用于处理副作用，自定义Hook是复用状态逻辑的机制。

## 1. useState
### 1.1 什么是useState？

```mermaid

graph LR

    A[useState] --> B[状态管理]

    A --> C[返回值]

    A --> D[更新函数]

    style A fill:#e1f5fe

```

**useState**：React Hook，用于在函数组件中添加状态。

### 1.2 基础用法

```jsx

import { useState } from 'react';

function Counter() {

    const [count, setCount] = useState(0);

    return (

        <div>

            <p>Count: {count}</p>

            <button onClick={() => setCount(count + 1)}>

                Increment

            </button>

        </div>

    );

}

```

### 1.3 函数式更新

```jsx

// 函数式更新

function Counter() {

    const [count, setCount] = useState(0);

    // 好：使用函数式更新

    const increment = () => {

        setCount(prev => prev + 1);

    };

    // 不好：直接使用当前值

    const incrementBad = () => {

        setCount(count + 1); // 可能有问题

    };

    return (

        <div>

            <p>Count: {count}</p>

            <button onClick={increment}>Increment</button>

        </div>

    );

}

```

## 2. useEffect
### 2.1 什么是useEffect？

```mermaid

graph LR

    A[useEffect] --> B[副作用处理]

    A --> C[依赖数组]

    A --> D[清理函数]

    style A fill:#e8f5e8

```

**useEffect**：React Hook，用于处理组件的副作用（如数据获取、订阅、手动修改DOM等）。

### 2.2 基础用法

```jsx

import { useState, useEffect } from 'react';

function UserProfile({ userId }) {

    const [user, setUser] = useState(null);

    useEffect(() => {

        // 数据获取

        fetch(`/api/users/${userId}`)

            .then(response => response.json())

            .then(data => setUser(data));

    }, [userId]); // 依赖数组

    if (!user) return <div>Loading...</div>;

    return (

        <div>

            <h1>{user.name}</h1>

            <p>{user.email}</p>

        </div>

    );

}

```

### 2.3 清理函数

```jsx

import { useEffect } from 'react';

function Timer() {

    useEffect(() => {

        const interval = setInterval(() => {

            console.log('Timer tick');

        }, 1000);

        // 清理函数

        return () => {

            clearInterval(interval);

        };

    }, []); // 空依赖数组

    return <div>Timer running...</div>;

}

```

## 3. 自定义Hook
### 3.1 什么是自定义Hook？

```mermaid

graph LR

    A[自定义Hook] --> B[复用状态逻辑]

    A --> C[以use开头]

    A --> D[可组合]

    style A fill:#e1f5fe

```

**自定义Hook**：以"use"开头的函数，用于复用状态逻辑。

### 3.2 基础示例

```jsx

import { useState, useEffect } from 'react';

// 自定义Hook：获取用户信息

function useUser(userId) {

    const [user, setUser] = useState(null);

    const [loading, setLoading] = useState(true);

    const [error, setError] = useState(null);

    useEffect(() => {

        setLoading(true);

        setError(null);

        fetch(`/api/users/${userId}`)

            .then(response => {

                if (!response.ok) {

                    throw new Error('Failed to fetch user');

                }

                return response.json();

            })

            .then(data => {

                setUser(data);

                setLoading(false);

            })

            .catch(err => {

                setError(err.message);

                setLoading(false);

            });

    }, [userId]);

    return { user, loading, error };

}

// 使用自定义Hook

function UserProfile({ userId }) {

    const { user, loading, error } = useUser(userId);

    if (loading) return <div>Loading...</div>;

    if (error) return <div>Error: {error}</div>;

    return (

        <div>

            <h1>{user.name}</h1>

            <p>{user.email}</p>

        </div>

    );

}

```

### 3.3 多个自定义Hook组合

```jsx

// 自定义Hook：表单处理

function useForm(initialValues) {

    const [values, setValues] = useState(initialValues);

    const handleChange = (e) => {

        const { name, value } = e.target;

        setValues(prev => ({

            ...prev,

            [name]: value

        }));

    };

    const reset = () => {

        setValues(initialValues);

    };

    return { values, handleChange, reset };

}

// 自定义Hook：提交处理

function useSubmit(onSubmit) {

    const [isSubmitting, setIsSubmitting] = useState(false);

    const handleSubmit = async (values) => {

        setIsSubmitting(true);

        try {

            await onSubmit(values);

        } finally {

            setIsSubmitting(false);

        }

    };

    return { handleSubmit, isSubmitting };

}

// 组合使用

function LoginForm() {

    const { values, handleChange } = useForm({

        email: '',

        password: ''

    });

    const { handleSubmit, isSubmitting } = useSubmit(async (values) => {

        await login(values);

    });

    return (

        <form onSubmit={(e) => {

            e.preventDefault();

            handleSubmit(values);

        }}>

            <input

                name="email"

                value={values.email}

                onChange={handleChange}

            />

            <input

                name="password"

                type="password"

                value={values.password}

                onChange={handleChange}

            />

            <button type="submit" disabled={isSubmitting}>

                {isSubmitting ? 'Logging in...' : 'Login'}

            </button>

        </form>

    );

}

```

## 4. 实际案例
### 4.1 数据获取Hook

```jsx

function useFetch(url) {

    const [data, setData] = useState(null);

    const [loading, setLoading] = useState(true);

    const [error, setError] = useState(null);

    useEffect(() => {

        const abortController = new AbortController();

        setLoading(true);

        setError(null);

        fetch(url, { signal: abortController.signal })

            .then(response => {

                if (!response.ok) {

                    throw new Error('Network response was not ok');

                }

                return response.json();

            })

            .then(data => {

                setData(data);

                setLoading(false);

            })

            .catch(err => {

                if (err.name !== 'AbortError') {

                    setError(err.message);

                    setLoading(false);

                }

            });

        return () => {

            abortController.abort();

        };

    }, [url]);

    return { data, loading, error };

}

// 使用

function UserList() {

    const { data: users, loading, error } = useFetch('/api/users');

    if (loading) return <div>Loading...</div>;

    if (error) return <div>Error: {error}</div>;

    return (

        <ul>

            {users.map(user => (

                <li key={user.id}>{user.name}</li>

            ))}

        </ul>

    );

}

```

## 5. 常见坑点
### 1. 依赖数组问题

```jsx

// 问题：缺少依赖

useEffect(() => {

    fetchData(userId);

}, []); // 缺少userId

// 解决：添加所有依赖

useEffect(() => {

    fetchData(userId);

}, [userId]);

```

### 2. 无限循环

```jsx

// 问题：useEffect导致无限循环

useEffect(() => {

    setCount(count + 1); // 每次渲染都更新状态

});

// 解决：使用依赖数组

useEffect(() => {

    setCount(prev => prev + 1);

}, [dependency]);

```

## 核心要点

```jsx

// useState

const [count, setCount] = useState(0);

// useEffect

useEffect(() => {

    // 副作用

    return () => {

        // 清理

    };

}, [dependencies]);

// 自定义Hook

function useCustomHook() {

    const [state, setState] = useState(initial);

    // ...

    return { state, setState };

}

```

## 速记卡（面试闪卡）

**Q1：一句话讲清「useState + useEffect + 自定义 Hook」到底是什么？**

A：useState 管组件状态、useEffect 管副作用、自定义 Hook 是以 use 开头复用状态逻辑的函数。

**Q2：1. useState —— 怎么理解？**

A：useState 像"组件的小账本"：const [count,setCount]=useState(0) 拿到状态值和改状态的笔；函数式更新 setCount(prev=>prev+1) 才稳妥（state hook 状态钩子）。

**Q3：2. useEffect —— 怎么理解？**

A：useEffect 像"副作用管家"：数据获取、订阅、改 DOM 都放这；依赖数组控制何时跑，return 清理函数负责收尾（side effect 副作用）。

**Q4：3. 自定义Hook —— 怎么理解？**

A：自定义 Hook 像"可复用的逻辑积木"：以 use 开头，内部用 useState/useEffect，返回 {data,loading}，多个组合搭出复杂功能（custom hook 自定义钩子）。

**Q5：5. 常见坑点 —— 怎么理解？**

A：两坑要记：依赖数组漏写 userId 导致用旧值；useEffect 里直接 setCount(count+1) 会无限循环，要用依赖或函数式更新（infinite loop 无限循环）。

**Q6：核心速记主线有哪些？**

- useState：返回 [值, 改值的函数]，函数式更新避免闭包旧值

- useEffect：管副作用，依赖数组控时机，return 做清理

- 自定义 Hook：use 开头复用状态逻辑，可组合

- 坑点：依赖别漏写；effect 里 setState 要带依赖否则死循环

**口诀**

A：useState 小账本，值配改笔 useState；

useEffect 管家，副作用里收尾清；

自定义 Hook 积木，use 开头逻辑复；

依赖别漏写，effect 防死循环。

## 相关链接

- 📋 目录：[[00-TypeScript-React]]

- 📚 学习清单： React

- 🔗 [[03-JSX+函数组件+Props|JSX和组件]]

- 🔗 [[05-React-Router路由系统|React Router]]


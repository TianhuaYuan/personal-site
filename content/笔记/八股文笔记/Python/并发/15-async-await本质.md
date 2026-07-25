---
title: "async与await语法与本质"
created: "2026-07-20"
tags:
  - 八股文
  - python
---

## 三、async/await —— 语法与本质



### 3.1 async def 做了什么？



```python

# 普通函数：调用 → 执行 → 返回结果

def normal():

    return 42



result = normal()    # result = 42





# async 函数（协程函数）：调用 → 返回协程对象 → 不执行！

async def coro_func():

    return 42



coro = coro_func()   # ⚠️ 返回 <coroutine object>，函数体还没执行！

print(coro)          # <coroutine object coro_func at 0x...>



# 要让协程执行，必须交给事件循环：

result = asyncio.run(coro)   # result = 42

```



```text

调用 async 函数

  ↓

返回协程对象（只是"待执行的任务描述"）

  ↓

必须 await 或 asyncio.run() 才会真正执行

```



> **认知刷新**：`async def` 并不让你的函数"异步"——它只是标记这个函数**内部可以用 await**，并且调用它会返回一个**协程对象**而不是立即执行。



### 3.2 await 做了什么？



```python

async def fetch_data():

    print("开始下载...")

    data = await some_io_operation()   # ← await 在这里做什么？

    print(f"下载完成: {data}")

    return data

```



**await 三步**：



```text

执行到 await some_io_operation()

  ↓

① 暂停当前协程（保存所有局部变量、执行位置）

  ↓

② 把 some_io_operation() 产生的等待对象注册到事件循环

  ↓

③ 控制权交还给事件循环 → 事件循环去执行其他就绪协程

  ↓

  ...（其他协程在执行，或事件循环在等 I/O）...

  ↓

④ I/O 就绪 → some_io_operation() 完成 → 事件循环唤醒本协程

  ↓

⑤ 从暂停处继续——data 拿到结果，执行下一行

```



```python

# await 只能用在 async 函数里——因为普通函数无法暂停

def normal():

    await something   # ❌ SyntaxError: 'await' outside async function



# await 后面必须是"可等待对象"（awaitable）

async def demo():

    await 42          # ❌ TypeError: object int can't be used in 'await'

    await "hello"     # ❌ 同上——字符串不是 awaitable

```



### 3.3 三种可等待对象（Awaitable）



> `await` 后面能放什么？三种东西：



```python

# ① 协程对象：调用 async 函数得到

async def foo():

    return 1



await foo()                    # ✅ foo() 返回协程对象



# ② Task：把协程包装成 Task——交给事件循环后台执行

task = asyncio.create_task(foo())

await task                     # ✅ Task 也是 awaitable



# ③ Future：底层对象——协程和 Task 的基类

#    一般不直接创建，asyncio 内部用

loop = asyncio.get_running_loop()

future = loop.create_future()

# future.set_result(42)        # 某处设置结果

# await future                 # 某处等待结果

```



| 可等待对象 | 谁创建 | 什么时候用 |

| :--- | :--- | :--- |

| **协程（Coroutine）** | `async def` 函数调用 | 直接 await——串行等待 |

| **Task** | `asyncio.create_task()` | 后台并发——不阻塞当前协程 |

| **Future** | `loop.create_future()` | 底层——库作者才直接操作 |



### 3.4 async with / async for



```python

# async with：异步上下文管理器——enter 和 exit 都可以 await

async with aiohttp.ClientSession() as session:     # 连接池初始化可能异步

    async with session.get(url) as resp:            # 等待响应

        data = await resp.text()



# 等价于：

session = await aiohttp.ClientSession().__aenter__()

try:

    resp = await session.get(url).__aenter__()

    data = await resp.text()

finally:

    await resp.__aexit__()

    await session.__aexit__()





# async for：异步迭代器——每次迭代都可能 await

async for chunk in response.content.iter_chunked():  # 每次读取下一个数据块

    process(chunk)                                    # 可能在等网络



# 等价于：

it = response.content.iter_chunked().__aiter__()

while True:

    try:

        chunk = await it.__anext__()   # 每次取下一个值都可能异步

    except StopAsyncIteration:

        break

```



> **不需要死记**——知道 `async with` 和 `async for` 是"里面可以 await 的 with/for"就行。源码逻辑和普通的 with/for 一样，只是 `__enter__` 变成 `__aenter__`。



---



## 相关链接



- 📋 目录：[[00-Python]]

- 📚 学习清单：[[八股文学习清单]]

- 🔗 [[笔记/八股文笔记/Python/并发/14-asyncio事件循环原理|asyncio事件循环原理]]

- 🔗 [[笔记/八股文笔记/Python/并发/16-gather与create_task|gather与create_task]]

- 🔗 [[八股文笔记/操作系统/01-进程vs线程vs协程对比|进程vs线程vs协程]] — 操作系统视角的并发基础

- 🔗 [[八股文笔记/React-TS-JS/JavaScript/15-异步与事件循环|JS异步与事件循环]] — Python asyncio vs JS 事件循环

- 🔗 [[八股文笔记/React-TS-JS/JavaScript/19-async与await|JS async/await]] — Python 与 JS 的 async/await 对比


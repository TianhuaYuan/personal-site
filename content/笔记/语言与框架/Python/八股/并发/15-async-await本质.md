---

title: "async与await语法与本质"

created: "2026-07-20"

tags:

  - 八股文

  - python

---

# async与await语法与本质

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

#    一般不直接创建，asyncio 内部用

loop = asyncio.get_running_loop()

future = loop.create_future()

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

## 速记卡（面试闪卡）

**Q1：一句话讲清「async与await语法与本质」到底是什么？**

A：async def 调用返回协程对象不直接执行，必须 await 或事件循环驱动；await 暂停协程把控制权交还事件循环。

**Q2：3.1 async def 做了什么？ —— 怎么理解？**

A：async def 像"按下但不启动的开关"：调用只返回一个待执行的协程对象，必须 await 或 asyncio.run 才真正跑（coroutine 协程）。

**Q3：3.2 await 做了什么？ —— 怎么理解？**

A：await 像"暂时让出柜台"：暂停当前协程、把等待对象交给事件循环、去跑别人，I/O 好了再被唤醒继续（event loop 事件循环）。

**Q4：3.3 三种可等待对象（Awaitable） —— 怎么理解？**

A：三种可等对象像三层工具：协程（直接 await）、Task（create_task 后台并发）、Future（底层库作者用）（Task 任务 / Future 未来对象）。

**Q5：3.4 async with / async for —— 怎么理解？**

A：async with/for 像"能中途 await 的 with/for"：进出处可异步（如连接池），__enter__ 变 __aenter__，逻辑和普通一样（async context manager 异步上下文管理器）。

**Q6：核心速记主线有哪些？**

- async def 调用返回协程对象，不立即执行，需 await/run 驱动

- await 暂停协程、交还事件循环，I/O 就绪再唤醒

- 三种可等待：协程 / Task（并发）/ Future（底层）

- async with 异步上下文、async for 异步迭代，里面可 await

**口诀**

A：async def 返回协程，不跑等 await；

await 让出柜台，事件循环调度快；

协程 Task 加 Future，三层可等对象；

async with 和 for，里面 await 才自在。

## 相关链接

- 📋 目录：[[00-Python]]

- 📚 学习清单：[[八股文学习路线图]]

- 🔗 [[语言与框架/Python/八股/并发/14-asyncio事件循环原理|asyncio事件循环原理]]

- 🔗 [[语言与框架/Python/八股/并发/16-gather与create_task|gather与create_task]]

- 🔗 [[计算机基础/操作系统/01-进程vs线程vs协程对比|进程vs线程vs协程]] — 操作系统视角的并发基础

- 🔗 [[语言与框架/TypeScript-React/八股/JavaScript/15-异步与事件循环|JS异步与事件循环]] — Python asyncio vs JS 事件循环

- 🔗 [[语言与框架/TypeScript-React/八股/JavaScript/19-async与await|JS async/await]] — Python 与 JS 的 async/await 对比


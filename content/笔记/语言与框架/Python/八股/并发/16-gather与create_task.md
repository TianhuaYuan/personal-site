---

title: "asyncio核心API"

created: "2026-07-20"

tags:

  - 八股文

  - python

---

# asyncio核心API

## 一、asyncio 核心 API —— 常见
### 1.1 asyncio.run —— 程序的入口

```python

# ③ 协程完成后关闭事件循环

async def main():

    return "hello"

result = asyncio.run(main())     # "hello"

# ⚠️ 不要在已经有事件循环的地方再调 asyncio.run()——会报错

```

### 1.2 asyncio.create_task —— 后台并发

> **常见**：`create_task` vs 直接 `await` 的区别。

```python

# ❌ 直接 await：串行

async def main():

    await download("url1")       # 等 url1 下载完...

    await download("url2")       # 才开始下载 url2

    # 总时间 = t1 + t2

# ✅ create_task：并发

async def main():

    task1 = asyncio.create_task(download("url1"))   # ① 提交到事件循环——后台执行

    task2 = asyncio.create_task(download("url2"))   # ② 提交到事件循环——后台执行

    # 两个下载已经在跑了！

    result1 = await task1        # ③ 等 task1 完成（可能已经完成了）

    result2 = await task2        # ④ 等 task2 完成

    # 总时间 ≈ max(t1, t2)

```

```text

直接 await（串行）：

download1: [====运行====]

download2:                [====运行====]

create_task + await（并发）：

download1: [====运行====]

download2: [====运行====]

           ↑ 同时进行——总时间 ≈ 最慢的那个

```

**create_task 到底做了什么？**

```text

create_task(coro)

  ↓

① 把协程包装成 Task 对象

  ↓

② Task 被注册到当前事件循环

  ↓

③ 事件循环在"下次有机会"时开始执行这个 Task

  ↓

④ 返回 Task 对象——你可以拿着它 await 或 cancel

```

### 1.3 asyncio.gather —— 批量并发 + 收集结果

```python

# gather：等一批协程全部完成，返回结果列表

async def main():

    results = await asyncio.gather(

        download("url1"),

        download("url2"),

        download("url3"),

    )

    # results = [data1, data2, data3]  ← 顺序和传入顺序一致！

# 也可以用 * 展开列表

urls = ["url1", "url2", "url3"]

tasks = [download(url) for url in urls]

results = await asyncio.gather(*tasks)

```

**gather 的行为细节**：

```python

# ① 默认：一个协程抛异常 → gather 立即抛出，但其他协程不取消

async def main():

    try:

        await asyncio.gather(

            download("url1"),         # 正常

            download("bad_url"),      # 抛异常！

            download("url3"),         # 仍在后台跑

        )

    except Exception:

        pass  # url3 可能还没跑完——但没人等它了

# ② return_exceptions=True：异常不抛出，作为结果返回

results = await asyncio.gather(

    download("url1"),

    download("bad_url"),              # 抛异常

    return_exceptions=True

)

# results = [data1, HTTPError("404"), ...]   ← 异常变成了结果项

```

### 1.4 asyncio.wait —— 更细粒度的控制

```python

# 比 gather 更灵活——可以控制"等第一个完成就返回"

done, pending = await asyncio.wait(

    [download(url) for url in urls],

    timeout=5.0,                              # 最多等 5 秒

    return_when=asyncio.FIRST_COMPLETED,      # 第一个完成就返回

)

for task in done:

    result = task.result()      # 获取已完成任务的结果

for task in pending:

    task.cancel()               # 取消还没完成的任务

```

| | gather | wait |
| :--- | :--- | :--- |
| 返回值 | 结果列表（按传入顺序） | 两个集合：`(done, pending)` |
| 异常处理 | 默认抛异常；可设 `return_exceptions` | 异常存在 task 里，不自动抛 |
| 超时 | 不支持 | 支持 `timeout` |
| 完成条件 | 全部完成 | 支持 `FIRST_COMPLETED` / `FIRST_EXCEPTION` |
| 使用场景 | "全部完成，给我所有结果" | "等第一个完成就行" 或 "加超时" |

### 1.5 asyncio.wait_for —— 单个协程超时

```python

# 等一个协程，超时就抛 TimeoutError

try:

    result = await asyncio.wait_for(

        download("slow_url"),

        timeout=3.0                      # 3 秒没完成就抛异常

    )

except asyncio.TimeoutError:

    print("太慢了，不等了")

```

### 1.6 asyncio.as_completed —— 谁先完成先处理谁

```python

# 不关心顺序——谁先回来先处理谁

async def main():

    tasks = [download(url) for url in urls]

    for coro in asyncio.as_completed(tasks):      # 返回迭代器——按完成顺序

        result = await coro                         # 等最快的那个完成

        print(f"完成了一个: {len(result)} 字节")

```

---

## 速记卡（面试闪卡）

**Q1：一句话讲清「asyncio核心API」到底是什么？**

A：asyncio 里 create_task 把协程丢进事件循环后台并发跑，gather 等它们全部完成按序收结果——比串行 await 快到约等于最慢那个。

**Q2：create_task：把协程挂上事件循环 —— 怎么理解？**

A：像点了两杯奶茶同时做：直接 await 是一杯做完才点下一杯（串行，总时间 t1+t2）；create_task 是两杯一起下单一同做（并发，总时间≈max）。create_task 把协程包成 Task 注册到当前循环，返回句柄你再 await 或 cancel。

**Q3：gather：批量等全部完成，按序收结果 —— 怎么理解？**

A：gather 像发了一叠快递单，等所有包裹到齐才一次性把结果列表给你，且顺序和传入一致。细节：一个协程抛异常，默认 gather 立刻抛但其他协程不取消；设 return_exceptions=True 则把异常当结果项返回，不中断整体。

**Q4：wait / wait_for / as_completed：更细的控制 —— 怎么理解？**

A：gather 是"全都要"，这仨是"看情况"：wait 返回 (done, pending) 两个集合，支持 FIRST_COMPLETED 和 timeout；wait_for 给单个协程加超时，超了抛 TimeoutError；as_completed 谁先完成先处理谁，不关心顺序。

**Q5：何时用哪个 —— 怎么理解？**

A：经验：要"全部完成给我所有结果"→gather；要"等第一个完成或加超时"→wait/wait_for；要"边收边处理"→as_completed。共同前提：都基于单事件循环，create_task 是它们能并发的根。

**Q6：核心速记主线有哪些？**

- create_task：协程挂上循环后台跑，比串行 await 快

- gather：等全部完成，结果按传入顺序返回

- 异常：默认抛出不取消其他；return_exceptions 变结果项

- wait/wait_for/as_completed：超时、首完成、边到边处理

- 选型：全收集用 gather，细控用 wait 系

**口诀**

A：asyncio 并发靠循环，create_task 挂后台；

gather 收齐按序排，wait 超时首完成来；

异常默认抛不 cancel，return_exceptions 装回；

as_completed 谁先到，边收边处理不等待。

## 相关链接

- 📋 目录：[[00-Python]]

- 📚 学习清单：[[八股文学习路线图]]

- 🔗 [[语言与框架/Python/八股/并发/14-asyncio事件循环原理|asyncio事件循环原理]]

- 🔗 [[语言与框架/Python/八股/并发/17-协程阻塞陷阱|协程阻塞陷阱]]

- 🔗 [[计算机基础/操作系统/01-进程vs线程vs协程对比|进程vs线程vs协程]] — 操作系统视角的并发基础

- 🔗 [[语言与框架/TypeScript-React/八股/JavaScript/15-异步与事件循环|JS异步与事件循环]] — Python asyncio vs JS 事件循环

- 🔗 [[计算机基础/操作系统/13-IO模型四种|IO模型四种]] — gather 本质是并发等待 IO


---
title: "asyncio核心API"
created: "2026-07-20"
tags:
  - 八股文
  - python
---

## 四、asyncio 核心 API —— 常见



### 4.1 asyncio.run —— 程序的入口



```python

# asyncio.run() 做了三件事：

# ① 创建事件循环

# ② 把协程丢进去执行

# ③ 协程完成后关闭事件循环



async def main():

    return "hello"



result = asyncio.run(main())     # "hello"

#           ↑ 阻塞直到 main() 执行完毕



# ⚠️ 一个程序通常只调一次 asyncio.run()——它是"顶层入口"

# ⚠️ 不要在已经有事件循环的地方再调 asyncio.run()——会报错

```



### 4.2 asyncio.create_task —— 后台并发



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



### 4.3 asyncio.gather —— 批量并发 + 收集结果



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



### 4.4 asyncio.wait —— 更细粒度的控制



```python

# wait 返回两个集合：(done, pending)

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



### 4.5 asyncio.wait_for —— 单个协程超时



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



### 4.6 asyncio.as_completed —— 谁先完成先处理谁



```python

# 不关心顺序——谁先回来先处理谁

async def main():

    tasks = [download(url) for url in urls]

    for coro in asyncio.as_completed(tasks):      # 返回迭代器——按完成顺序

        result = await coro                         # 等最快的那个完成

        print(f"完成了一个: {len(result)} 字节")

```



---



## 相关链接



- 📋 目录：[[00-Python]]

- 📚 学习清单：[[八股文学习清单]]

- 🔗 [[笔记/八股文笔记/Python/并发/14-asyncio事件循环原理|asyncio事件循环原理]]

- 🔗 [[笔记/八股文笔记/Python/并发/17-协程阻塞陷阱|协程阻塞陷阱]]

- 🔗 [[八股文笔记/操作系统/01-进程vs线程vs协程对比|进程vs线程vs协程]] — 操作系统视角的并发基础

- 🔗 [[八股文笔记/React-TS-JS/JavaScript/15-异步与事件循环|JS异步与事件循环]] — Python asyncio vs JS 事件循环

- 🔗 [[八股文笔记/操作系统/13-IO模型四种|IO模型四种]] — gather 本质是并发等待 IO


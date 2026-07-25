---
title: "混合并发实际项目怎么组合"
created: "2026-07-20"
tags:
  - 八股文
  - python
---

## 第六章：混合并发——实际项目怎么组合使用



### 6.1 真实场景：Web API 服务器



```mermaid

graph TD

    REQ[请求进来] --> ASYNC[async def endpoint<br/>asyncio协程处理请求]

    ASYNC --> DB[user = await db.fetch_userid<br/>等数据库 = IO 协程切走]

    DB --> PDF[report = await generate_pdfuser<br/>生成PDF = CPU密集]

    PDF --> WARN{⚠️ 在协程里直接调CPU密集函数？}

    WARN -->|会堵死事件循环| BAD[❌ 错误]

    WARN -->|正确做法| GOOD[✅ 丢到线程池或进程池]

    GOOD --> LOOP[loop = asyncio.get_running_loop<br/>report = await loop.run_in_executor<br/>process_pool<br/>generate_pdf user]

    LOOP --> RET[return report<br/>回到协程 返回响应]

```



### 6.2 三种 run_in_executor 模式



```python

import asyncio, time

from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor



# 准备两个专用执行器（全局复用，不要每次创建！）

thread_pool = ThreadPoolExecutor(max_workers=10)    # IO 密集任务

process_pool = ProcessPoolExecutor(max_workers=4)   # CPU 密集任务



def cpu_intensive_task(n: int) -> int:

    """纯计算——应该在进程池跑"""

    total = 0

    for i in range(n * 10_000_000):

        total += i

    return total



def blocking_io_task(url: str) -> str:

    """阻塞 IO（用了同步库 requests）——应该在线程池跑"""

    import requests

    return requests.get(url).text[:100]



async def mixed_workload():

    """同时有 IO 密集和 CPU 密集任务"""

    

    # ① IO 密集：丢线程池（线程在等网络时释放 GIL）

    io_results = await asyncio.gather(

        asyncio.get_running_loop().run_in_executor(thread_pool, blocking_io_task, "https://httpbin.org/get"),

        asyncio.get_running_loop().run_in_executor(thread_pool, blocking_io_task, "https://httpbin.org/headers"),

    )

    

    # ② CPU 密集：丢进程池（真正并行计算）

    cpu_results = await asyncio.gather(

        asyncio.get_running_loop().run_in_executor(process_pool, cpu_intensive_task, 5),

        asyncio.get_running_loop().run_in_executor(process_pool, cpu_intensive_task, 8),

    )

    

    # ③ 在协程里等结果——事件循环不会堵塞

    return {"io": io_results, "cpu": cpu_results}



# asyncio.run(mixed_workload())

```



### 6.3 选型决策树——终极版



```mermaid

graph TD

    TASK[你的任务] --> WAIT{主要时间花在等什么}

    WAIT -->|等网络/数据库/磁盘/消息| IO[IO密集型]

    WAIT -->|等CPU算完| CPU[CPU密集型]

    IO --> CONCUR{并发量}

    CONUR -->|>100并发| ASYNC[asyncio 首选]

    CONUR -->|<10个任务| THREAD[threading.ThreadPoolExecutor]

    ASYNC --> SUPPORT{库支持asyncio}

    SUPPORT -->|支持| OK[用异步库 aiohttp/httpx]

    SUPPORT -->|不支持| CHANGE{能换库}

    CHANGE -->|能| OK

    CHANGE -->|不能| EXECUTOR[run_in_executor线程池 asyncio]

    CPU --> SPLIT{能拆成独立任务}

    SPLIT -->|能| MP[多进程ProcessPoolExecutor<br/>真正利用多核]

    SPLIT -->|不能| SMALL[单进程优化算法]

    CPU --> NUMPY{用了NumPy/Pandas}

    NUMPY -->|是| MT[多线程也行 C层释放GIL]

    TASK --> MIXED{同时需要IO+CPU密集}

    MIXED -->|是| COMBINED[asyncio主事件循环<br/>+ run_in_executor进程池]

    TASK --> ISOLATE{需要任务隔离}

    ISOLATE -->|是| PROCESS[多进程 崩了只影响自己]

```



---



## 相关链接



- 📋 目录：[[00-Python]]

- 📚 学习清单：[[八股文学习清单]]

- 🔗 [[笔记/八股文笔记/Python/并发/18-threading模块|threading模块]]

- 🔗 [[笔记/八股文笔记/Python/并发/19-multiprocessing模块|multiprocessing模块]]

- 🔗 [[八股文笔记/操作系统/01-进程vs线程vs协程对比|进程vs线程vs协程]] — 操作系统视角的并发基础

- 🔗 [[八股文笔记/React-TS-JS/JavaScript/15-异步与事件循环|JS异步与事件循环]] — Python asyncio vs JS 事件循环

- 🔗 [[八股文笔记/操作系统/01-进程vs线程vs协程对比|进程vs线程vs协程]] — Executor 封装了线程/进程池


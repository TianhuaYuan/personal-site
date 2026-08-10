---
title: "GIL对多线程的影响"
created: "2026-07-20"
tags:
  - 八股文
  - python
---

## 三、GIL 对多线程的影响（核心考点）



### 3.1 CPU 密集型 vs IO 密集型



> 这是理解 GIL 影响的关键区分。用厨房比喻继续——



**CPU 密集型 = 炒菜**：厨师必须一直在灶台前翻锅，不能离开，4 个厨师抢 1 个灶台 = 白等。



**IO 密集型 = 煲汤**：把汤放上去，厨师可以离开去切菜、洗碗，等汤好了再回来。4 个厨师可以**交替用灶台**，谁的汤好了谁去端。



| 类型 | 定义 | GIL 影响 | 例子 |

| :---: | :--- | :---: | :--- |

| **CPU 密集型** | 大量计算，几乎不等待 | 🔴 **严重** — 多线程几乎无提升 | 数学计算、图像处理、加密解密 |

| **IO 密集型** | 大量等待网络/磁盘/用户输入 | 🟢 **轻微** — 等待时释放 GIL | 网络请求、文件读写、数据库查询 |



### 3.2 代码对比：CPU 密集型的惨状



```python

import threading, time                         # 导入线程模块和时间模块



def cpu_work(n):

    """CPU密集：计算 1+2+...+n"""

    total = 0                                  # 初始化累加器

    for i in range(n):                         # 循环 n 次，纯计算，不涉及任何 IO

        total += i                             # 每次循环都在占用 CPU

    return total                               # 返回计算结果



# ---- 单线程：两个任务串行执行 ----

start = time.time()                            # 记录开始时间

cpu_work(100_000_000)                          # 第一个任务，算 1 亿次

cpu_work(100_000_000)                          # 第二个任务，必须等第一个完成才开始

print(f"单线程: {time.time() - start:.2f}s")   # ~12s



# ---- 双线程：两个任务"并发"执行 ----

start = time.time()                            # 记录开始时间

t1 = threading.Thread(target=cpu_work, args=(100_000_000,))  # 创建线程 1

t2 = threading.Thread(target=cpu_work, args=(100_000_000,))  # 创建线程 2

t1.start(); t2.start()                         # 两个线程同时启动，但 GIL 只让一个跑

t1.join(); t2.join()                           # 等两个线程都结束

print(f"双线程: {time.time() - start:.2f}s")   # ~12s（没有变快！因为 GIL）

```



> **结果几乎一样**，因为两个线程抢同一把 GIL，始终只有一个在执行。



### 3.3 IO 密集型没问题



```python

import threading, time, requests               # 导入线程、时间、HTTP 请求库



def fetch(url):

    """IO密集：网络请求（大部分时间在等服务器响应）"""

    requests.get(url)                          # 发送 GET 请求，等待响应期间不占 CPU



# ---- 串行：一个一个请求，每个等 1 秒 ----

start = time.time()                            # 记录开始时间

for _ in range(4):                             # 循环 4 次

    fetch("https://httpbin.org/delay/1")       # 每个请求等 1 秒，4 个 = 4 秒

print(f"串行: {time.time() - start:.2f}s")    # ~4s



# ---- 多线程：4 个请求同时发，等待时 GIL 被释放 ----

start = time.time()                            # 记录开始时间

threads = [threading.Thread(target=fetch, args=("https://httpbin.org/delay/1",)) for _ in range(4)]

                                               # 创建 4 个线程，每个发一个请求

for t in threads: t.start()                    # 4 个线程同时启动

for t in threads: t.join()                     # 等全部完成

print(f"多线程: {time.time() - start:.2f}s")  # ~1s（4 倍加速！网络等待时不占 GIL）

```



> 网络等待期间 GIL 被释放，其他线程可以继续发请求。



---



## 相关链接



- 📋 目录：[[00-Python]]

- 📚 学习清单：[[八股文学习清单]]

- 🔗 [[语言与框架/Python/八股/并发/05-GIL是什么|GIL是什么]]

- 🔗 [[语言与框架/Python/八股/并发/08-怎么绕过GIL|怎么绕过GIL]]

- 🔗 [[计算机基础/操作系统/01-进程vs线程vs协程对比|进程vs线程vs协程]] — 操作系统视角的并发基础

- 🔗 [[语言与框架/TypeScript-React/八股/JavaScript/15-异步与事件循环|JS异步与事件循环]] — Python asyncio vs JS 事件循环

- 🔗 [[计算机基础/操作系统/01-进程vs线程vs协程对比|进程vs线程vs协程]] — GIL 决定多线程能否真并行


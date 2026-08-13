---

title: "怎么绕过GIL"

created: "2026-07-20"

tags:

  - 八股文

  - python

---

# 怎么绕过GIL

## 四、怎么绕过 GIL？（必答）
### 方案一：multiprocessing 多进程（最常用）

> 每个进程有**独立的 Python 解释器和 GIL**，可以真正利用多核。

**通俗比喻**：4 个厨师抢 1 个灶台（多线程）太挤了？那就**开 4 间厨房**（多进程），每间各有自己的灶台，各做各的菜——真正的并行。代价是厨房之间传菜（进程间通信）比较麻烦，而且每间厨房都要占地方（内存）。

```python

from multiprocessing import Pool               # 导入进程池

import time                                    # 导入时间模块

def cpu_work(n):

    """CPU密集任务"""

    total = 0                                  # 初始化累加器

    for i in range(n):                         # 循环 n 次纯计算

        total += i                             # 每次循环占用 CPU

    return total                               # 返回结果

if __name__ == "__main__":                     # 多进程必须在 main 里写，否则会无限 fork

    # ---- 单进程：串行执行两次，共 ~12s ----

    start = time.time()                        # 记录开始时间

    cpu_work(100_000_000)                      # 第一个任务

    cpu_work(100_000_000)                      # 第二个任务，串行等完

    print(f"单进程: {time.time() - start:.2f}s")   # ~12s

    # ---- 多进程：两个进程各跑一个任务，各自有独立 GIL，真并行 ----

    start = time.time()                        # 记录开始时间

    with Pool(2) as p:                         # 开 2 个进程（= 2 间厨房，各有各的灶台）

        p.map(cpu_work, [100_000_000, 100_000_000])  # 把两个任务分给两个进程并行执行

    print(f"多进程: {time.time() - start:.2f}s")     # ~6s（真并行，时间减半）

```

| 优点 | 缺点 |

| :--- | :--- |

| 真正并行，能用多核 | 进程间通信开销大（IPC） |

| 代码改动小 | 内存占用高（每个进程独立内存空间） |

| 适合 CPU 密集型 | 序列化/反序列化有成本 |

### 方案二：C 扩展释放 GIL

> NumPy、Pandas 等库的底层 C 代码可以在**计算密集部分主动释放 GIL**，计算完再拿回来。

**通俗比喻**：厨师（Python 线程）炒菜时拿着灶台钥匙（GIL），但 NumPy 说"这块活我用专业设备（C 代码）干，不需要你盯着"，于是厨师**主动把钥匙挂门上**（释放 GIL），让别的厨师也能用灶台。等 NumPy 干完了，再把钥匙拿回来。

```python

import numpy as np                             # 导入 NumPy

a = np.random.rand(10000, 10000)              # 创建 10000x10000 的随机矩阵 A

b = np.random.rand(10000, 10000)              # 创建 10000x10000 的随机矩阵 B

c = np.dot(a, b)                              # 矩阵乘法：底层 C/Fortran 执行，自动释放 GIL

                                               # 所以多线程调用 np.dot 时，线程可以真正并行利用多核

```

> **一句话讲清**：像 NumPy 这样的库，底层用 C/Fortran 实现，计算时会调用 `Py_BEGIN_ALLOW_THREADS` 释放 GIL，计算完再 `Py_END_ALLOW_THREADS` 重新获取。所以用 NumPy 做矩阵运算时，多线程是可以利用多核的。

### 方案三：asyncio 异步 IO（IO 密集首选）

> 不是绕过 GIL，而是**在等待 IO 时让出控制权**，让其他协程执行。

**通俗比喻**：多线程像**4 个厨师各端一盘菜等出餐**，每人占一个位子（~8MB 栈内存）；asyncio 像**1 个厨师同时盯着 4 个锅**，哪个锅好了就去处理那个，不占额外位子（协程只要几 KB）。区别是：多线程是"多人干活"，asyncio 是"一个人眼疾手快轮着干"。

```python

import asyncio, aiohttp                       # 导入异步 IO 框架和异步 HTTP 库

async def fetch(session, url):

    """异步请求：遇到 await 就让出控制权，去处理别的协程"""

    async with session.get(url) as resp:       # 发起异步 GET 请求，等待时不阻塞线程

        return await resp.text()               # 等响应时，别的协程可以跑

async def main():

    async with aiohttp.ClientSession() as session:  # 创建异步 HTTP 会话

        # 创建 4 个协程任务，同时发请求

        tasks = [fetch(session, "https://httpbin.org/delay/1") for _ in range(4)]

        await asyncio.gather(*tasks)           # 并发执行所有任务，全部完成才返回

asyncio.run(main())                            # ~1s（和多线程一样快，但只用了 1 个线程 + 几 KB 内存）

```

| 对比 | 多线程 | asyncio |

| :---: | :--- | :--- |

| 并发模型 | 操作系统线程切换 | 单线程协程切换 |

| 内存开销 | 每个线程 ~8MB 栈 | 每个协程 ~几KB |

| 适合场景 | IO密集 + 需要用同步库 | IO密集 + 纯异步生态 |

| 编程难度 | 简单，但要注意线程安全 | 需要 async/await 全链路 |

### 方案四：换解释器（了解即可）

| 解释器 | GIL 情况 | 说明 |

| :---: | :--- | :--- |

| **PyPy** | 有 GIL，但 JIT 编译后快很多 | CPU 密集型场景用 PyPy 可能比 CPython 快 10 倍 |

| **Jython** | 无 GIL | 基于 JVM，但生态兼容性差 |

| **IronPython** | 无 GIL | 基于 .NET，用的人少 |

| **CPython 3.13+** | 可选关闭 (`-X nogil`) | 实验性，2024 年推出，未来可能成为默认 |

---

## 速记卡（面试闪卡）

**Q1：一句话讲清「怎么绕过GIL」到底是什么？**

A：绕过 GIL 只有两条真并行路：多进程与 C 扩展释放 GIL；asyncio 只是 IO 时让权不并行。

**Q2：一、multiprocessing 多进程（真并行） —— 怎么理解？**

A：每个进程有独立的 Python 解释器和 GIL，可真正利用多核。比喻：4 个厨师抢 1 个灶台太挤？开 4 间厨房，每间各有自己的灶台各做各的——真并行。代价是进程间通信麻烦、内存占用高。英文：multiprocessing（多进程）、IPC（进程间通信）。

**Q3：二、C 扩展释放 GIL（NumPy/Pandas） —— 怎么理解？**

A：NumPy/Pandas 底层 C 代码在计算密集部分主动释放 GIL，算完再拿回。比喻：厨师炒菜拿着灶台钥匙（GIL），NumPy 说"这活用专业设备干"——主动把钥匙挂门上，让别人也能用灶台。所以多线程调 np.dot 能真并行。英文：Py_BEGIN_ALLOW_THREADS、C extension（C 扩展）。

**Q4：三、asyncio 异步 IO（不是绕过 GIL） —— 怎么理解？**

A：asyncio 不是绕过 GIL，而是等待 IO 时让出控制权给别的协程。比喻：多线程像 4 个厨师各端一盘菜等出餐（每人占 ~8MB 栈）；asyncio 像 1 个厨师盯 4 个锅，哪个好了处理哪个（几 KB）。区别：多线程多人干活，asyncio 一人轮着干。英文：asyncio、coroutine（协程）、await。

**Q5：四、换解释器与选型总结 —— 怎么理解？**

A：PyPy 有 GIL 但 JIT 快很多；Jython/IronPython 无 GIL 但生态差；CPython 3.13+ 可 -X nogil 实验性关闭。选型：CPU 密集→多进程或 C 扩展；IO 密集→asyncio 或多线程；记住 asyncio 不并行只是让权。英文：PyPy、nogil、interpreter（解释器）。

**Q6：核心速记主线有哪些？**

- 真并行只有两条路：多进程、C 扩展释放 GIL

- 多进程每进程独立 GIL，代价是 IPC 与内存

- C 扩展（NumPy）计算时释放 GIL

- asyncio 只是 IO 时让权，不并行

- CPU 密集用多进程/C 扩展，IO 密集用 asyncio

**口诀**

A：GIL 锁住单线程，多进程各开一间房；

C 扩展算时松手，NumPy 并行不慌张。

asyncio 盯锅转，让权不并行为哪桩；

CPU 密集多进程，IO 密集它最强。

## 相关链接

- 📋 目录：[[00-Python]]

- 📚 学习清单：[[八股文学习路线图]]

- 🔗 [[语言与框架/Python/八股/并发/05-GIL是什么|GIL是什么]]

- 🔗 [[语言与框架/Python/八股/并发/06-GIL为什么存在|GIL为什么存在]]

- 🔗 [[计算机基础/操作系统/01-进程vs线程vs协程对比|进程vs线程vs协程]] — 操作系统视角的并发基础

- 🔗 [[语言与框架/TypeScript-React/八股/JavaScript/15-异步与事件循环|JS异步与事件循环]] — Python asyncio vs JS 事件循环

- 🔗 [[计算机基础/操作系统/01-进程vs线程vs协程对比|进程vs线程vs协程]] — 绕过 GIL 用多进程


---

title: "GIL对多线程的影响"

created: "2026-07-20"

tags:

  - 八股文

  - python

---

# GIL对多线程的影响

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

## 速记卡（面试闪卡）

**Q1：一句话讲清「GIL对多线程的影响」到底是什么？**

A：GIL 让同一时刻只有一个线程执行 Python 字节码，故 CPU 密集多线程几乎无提升，IO 密集因等待释放 GIL 仍能加速。

**Q2：三、CPU 密集 vs IO 密集的区分 —— 怎么理解？**

A：CPU 密集＝炒菜：厨师必须一直翻锅不能离，4 个厨师抢 1 个灶台＝白等。IO 密集＝煲汤：汤放上灶可离开切菜，谁汤好谁去端，4 人可交替用灶台。GIL 对 CPU 密集🔴严重（多线程几乎无提升），对 IO 密集🟢轻微（等待时释放 GIL）。

**Q3：3.2 CPU 密集型的惨状（代码对比） —— 怎么理解？**

A：两个 1 亿次累加任务：单线程串行 ~12s，开双线程"并发"还是 ~12s——因为两线程抢同一把 GIL，始终只有一个在跑。GIL 把"多核"锁成"单核"，多线程在纯计算上等于没并行，证明 GIL 是 CPU 密集的硬伤。

**Q4：3.3 IO 密集型没问题 —— 怎么理解？**

A：4 个网络请求串行 ~4s（每个等 1s），开 4 线程"同时"发 ~1s（4 倍加速）。因为网络等待期间 GIL 被释放，其他线程能继续发请求。IO 密集场景多线程／协程才真正发挥并发价值，GIL 几乎不挡。

**Q5：怎么绕过 GIL（延伸） —— 怎么理解？**

A：既然 GIL 只卡 CPU 密集，绕法有：① 用多进程（multiprocessing）真正并行，各进程有各自 GIL；② 用 C 扩展（numpy 等）在释放 GIL 的 C 代码里算；③ 用 asyncio 协程处理 IO 密集；④ Python 3.13 自由线程（PEP703）可编译无 GIL。选型：CPU 密用多进程、IO 密用多线程／协程。

**Q6：核心速记主线有哪些？**

- GIL：同一时刻仅一个线程跑 Python 字节码

- CPU 密集多线程≈无提升（抢一把锁）；IO 密集因等待释放 GIL 可加速

- 代码实证：CPU 双线程时长≈单线程；IO 四线程≈1/4 时长

- 绕 GIL：多进程／ C 扩展／ asyncio／ 3.13 自由线程

- 选型：CPU 密→多进程，IO 密→多线程或协程

**口诀**

A：GIL 一把锁，时刻只放一线程

CPU 密集抢破头，多线程也白费劲

IO 等待放了锁，协程线程都灵

要真并行多进程，或用 C 扩展顶

## 相关链接

- 📋 目录：[[00-Python]]

- 📚 学习清单：[[八股文学习路线图]]

- 🔗 [[语言与框架/Python/八股/并发/05-GIL是什么|GIL是什么]]

- 🔗 [[语言与框架/Python/八股/并发/08-怎么绕过GIL|怎么绕过GIL]]

- 🔗 [[计算机基础/操作系统/01-进程vs线程vs协程对比|进程vs线程vs协程]] — 操作系统视角的并发基础

- 🔗 [[语言与框架/TypeScript-React/八股/JavaScript/15-异步与事件循环|JS异步与事件循环]] — Python asyncio vs JS 事件循环

- 🔗 [[计算机基础/操作系统/01-进程vs线程vs协程对比|进程vs线程vs协程]] — GIL 决定多线程能否真并行


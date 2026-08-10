---
title: "IO密集型与CPU密集型"
created: "2026-07-20"
tags:
  - 八股文
  - python
---

## 第五章：IO 密集型 vs CPU 密集型——怎么判断、怎么量化

### 5.1 一句话判断法

```mermaid
graph TD
    CPU[查看 CPU 利用率] --> LOW[&lt; 20%]
    CPU --> HIGH[&gt; 90%]
    LOW --> IO[IO 密集型<br/>大部分时间在等硬盘/网络/数据库]
    IO --> IOOPT[优化：增加并发数<br/>asyncio > 多线程 > 多进程]
    HIGH --> CPU_BOUND[CPU 密集型<br/>CPU 一直在算]
    CPU_BOUND --> CPUOPT[优化：增加CPU核数<br/>多进程 > C扩展如NumPy > 换语言]
```

### 5.2 代码级别的判断——看时间花在哪

```python
import time

def measure_profile(func):
    """粗略测量一个函数是 IO 密集还是 CPU 密集"""
    start_wall = time.time()      # 墙上时间（真实流逝时间）
    start_cpu = time.process_time()  # CPU 占用时间（不含等待）
    
    result = func()
    
    wall_time = time.time() - start_wall
    cpu_time = time.process_time() - start_cpu
    
    print(f"墙上时间: {wall_time:.2f}s")    # 你等的总时间
    print(f"CPU 时间: {cpu_time:.2f}s")     # CPU 实际干活的时间
    print(f"CPU 占比: {cpu_time/wall_time*100:.0f}%")
    
    if cpu_time / wall_time > 0.8:
        print("→ CPU 密集型")
    else:
        print("→ IO 密集型")

# 测试 1：纯计算
def pure_compute():
    sum(i*i for i in range(50_000_000))
# measure_profile(pure_compute)
# 墙上时间: 3.5s, CPU 时间: 3.5s, CPU 占比: 100% → CPU 密集型

# 测试 2：网络请求
import requests
def network_io():
    requests.get("https://httpbin.org/delay/1")
# measure_profile(network_io)
# 墙上时间: 1.2s, CPU 时间: 0.05s, CPU 占比: 4% → IO 密集型
```

### 5.3 实际业务场景分类

| 场景 | 类型 | 推荐方案 | 原因 |
| :--- | :---: | :--- | :--- |
| Web API 服务器（CRUD） | IO 密集 | **asyncio** | 每个请求大部分时间在等数据库/其他服务 |
| 爬虫（大量 HTTP 请求） | IO 密集 | **asyncio + aiohttp** | 等网络响应的时间占比 >95% |
| 文件上传下载 | IO 密集 | **asyncio / 多线程** | 等磁盘 IO |
| 图像处理（Pillow） | CPU 密集 | **多进程** | 解码/缩放/滤镜都是纯计算 |
| 视频转码 | CPU 密集 | **多进程 + FFmpeg** | 编码计算量极大 |
| 科学计算（NumPy/Pandas） | CPU 密集 | **多线程也行** | NumPy 底层 C 释放 GIL |
| 机器学习推理 | CPU 密集 | **多进程 / GPU** | 矩阵运算量大 |
| WebSocket / 长连接 | IO 密集 | **asyncio** | 大量连接但每个数据量少，大部分时间在等 |
| 消息队列消费 | 混合 | **asyncio + 线程池** | 等消息是 IO，处理消息可能是 CPU |

### 5.4 性能对比——同一任务用三种方案跑

```python
"""
模拟任务：从 20 个 URL 下载数据，然后对每个结果做 JSON 解析
- 下载是 IO 密集（等网络）
- JSON 解析是 CPU 密集（纯计算，但很快——微秒级）
- 总体偏向 IO 密集

对比三种方案的实际速度：
"""

import time, json, threading, asyncio, aiohttp
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor
import requests

URLS = ["https://httpbin.org/delay/0.5"] * 10  # 10 个模拟请求

# ---- 方案一：单线程串行 ----
def serial():
    results = []
    for url in URLS:
        resp = requests.get(url, timeout=10)
        results.append(resp.json())
    return results

# ---- 方案二：多线程（ThreadPoolExecutor）----
def threaded():
    def fetch_one(url):
        return requests.get(url, timeout=10).json()
    with ThreadPoolExecutor(max_workers=10) as ex:
        return list(ex.map(fetch_one, URLS))

# ---- 方案三：asyncio + aiohttp ----
async def async_main():
    async def fetch_one(session, url):
        async with session.get(url, timeout=aiohttp.ClientTimeout(total=10)) as resp:
            return await resp.json()
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_one(session, url) for url in URLS]
        return await asyncio.gather(*tasks)

# ---- 方案四：多进程 ----
def process_worker(url):
    return requests.get(url, timeout=10).json()

def multiprocessed():
    with ProcessPoolExecutor(max_workers=4) as ex:
        return list(ex.map(process_worker, URLS))

# 实际跑：
# 串行:      ~5.5s  (10 × 0.5s = 5s 网络等待 + 0.5s 开销)
# 多线程:    ~0.7s  (并发等待，~0.5s 网络 + 线程切换开销)
# asyncio:   ~0.6s  (并发等待，~0.5s 网络 + 最小切换开销)
# 多进程:    ~2.0s  (进程启动开销 + 序列化开销，但只有 10 个请求不值得开多进程)

# 结论：IO 密集型 → asyncio ≈ 多线程 >> 多进程 >> 串行
#       多进程在 IO 密集场景反而是最慢的（启动进程的固定成本太高）
```

---

## 三个核心 API

```python
import asyncio

# ① 定义协程函数 — async def 不是普通函数，调用它返回 coroutine 对象，不会立即执行
async def say(name: str, delay: float) -> str:
    await asyncio.sleep(delay)    # 异步版 time.sleep()，不阻塞事件循环
    return f"{name} 说完了"

# ② 运行 — asyncio.run() 创建事件循环 + 跑协程 + 清理，整个程序只调一次
result = asyncio.run(say("小明", 1.0))

# ③ 创建 Task — 不等待，立即把协程"提交到后台"跑
task = asyncio.create_task(say("小红", 2.0))
```

```text
asyncio.run() vs asyncio.create_task():

asyncio.run(coro):
```

```mermaid
flowchart TB
    R1["① 创建事件循环"] --> R2["② 把 coro 注册进去"]
    R2 --> R3["③ 死循环跑，直到 coro 完成"]
    R3 --> R4["④ 关闭循环，返回 coro 的结果"]
```

```text
  一个程序只调一次，是整个异步程序的"入口"

asyncio.create_task(coro):
```

```mermaid
flowchart TB
    C1["把 coro 包装成 Task<br/>丢进当前事件循环的待执行队列"] --> C2["立即返回 Task 对象（不阻塞！）"]
```

```text
  拿到的 task 是一个"承诺"——"我会跑的，你先忙"
```

---

## 串行 vs 并发

```python
import asyncio

# ============================================================
# ❌ 串行 — await 一个一个等，白写了 async
# ============================================================
async def download_serial(urls: list[str]) -> list[str]:
    results = []
    for url in urls:
        result = await fetch(url)   # 等 url1 完成才发 url2，跟同步一样慢
        results.append(result)
    return results
    # 10 个 URL → ~10s（每个 1s）

# ============================================================
# ✅ 并发 — create_task 全部发射 + gather 统一收
# ============================================================
async def download_concurrent(urls: list[str]) -> list[str]:
    tasks = [asyncio.create_task(fetch(url)) for url in urls]
    results = await asyncio.gather(*tasks)
    return results
    # 10 个 URL → ~1s（最慢的那个决定总时间）

async def fetch(url: str) -> str:
    print(f"开始请求 {url}")
    await asyncio.sleep(1.0)
    print(f"完成请求 {url}")
    return f"{url} 的内容"
```

```text
时间线对比:

串行 (await 一个一个来):
URL1: [====1s====]
URL2:              [====1s====]
URL3:                           [====1s====]
总时间: 3s

并发 (create_task + gather):
URL1: [====1s====]
URL2: [====1s====]    ← 三个同时跑！
URL3: [====1s====]
总时间: 1s
```

> [!note] 核心原则
> `create_task()` 是"发射"——立即返回不阻塞；`await` 是"等待"——暂停当前协程等结果。**先 create_task 发射全部，再 gather 统一收**，这才是正确的并发姿势。

---

## gather vs Task 四种用法

```python
import asyncio

async def fetch(id: int) -> dict:
    await asyncio.sleep(0.5)
    return {"id": id, "data": f"数据{id}"}

async def main():
    # ============================================================
    # 用法1: gather — 等全部完成，按传入顺序返回结果列表
    # ============================================================
    results = await asyncio.gather(
        fetch(1),
        fetch(2),
        fetch(3),
    )
    # results = [{"id":1,"data":"数据1"}, {"id":2,...}, {"id":3,...}]
    # 返回顺序 = 传入顺序，不是完成顺序

    # ============================================================
    # 用法2: as_completed — 谁先完成先处理谁
    # ============================================================
    tasks = [asyncio.create_task(fetch(i)) for i in range(5)]
    for done_task in asyncio.as_completed(tasks):
        result = await done_task
        print(f"最快完成: {result}")

    # ============================================================
    # 用法3: return_exceptions — 某个任务挂了不影响其他
    # ============================================================
    async def risky_fetch(id: int) -> str:
        if id == 2:
            raise ValueError(f"{id} 炸了")
        await asyncio.sleep(0.3)
        return f"ok_{id}"

    results = await asyncio.gather(
        risky_fetch(1),
        risky_fetch(2),
        risky_fetch(3),
        return_exceptions=True,
    )
    # results = ["ok_1", ValueError("2 炸了"), "ok_3"]
    for i, r in enumerate(results):
        if isinstance(r, Exception):
            print(f"任务{i+1} 失败: {r}")
        else:
            print(f"任务{i+1} 成功: {r}")

    # ============================================================
    # 用法4: wait_for — 加超时控制
    # ============================================================
    async def slow_fetch():
        await asyncio.sleep(10)
        return "太慢了"

    try:
        result = await asyncio.wait_for(slow_fetch(), timeout=2.0)
    except asyncio.TimeoutError:
        print("超时！2 秒没返回，取消请求")
```

---

## 常见错误

```python
# ❌ 错误1：忘记 await — 协程没被执行
async def get_data():
    return "data"

async def main():
    result = get_data()          # 返回 coroutine 对象，不是 "data"！
    print(type(result))          # <class 'coroutine'>

    result = await get_data()    # ✅ await 才真正执行
    print(result)                # "data"

# ❌ 错误2：在协程里用 time.sleep() — 阻塞整个事件循环
import time
async def bad():
    time.sleep(5)                # 整个线程停 5 秒，所有协程都被卡住
    return "done"

async def good():
    await asyncio.sleep(5)       # ✅ 把控制权还给事件循环，别的协程可以跑
    return "done"

# ❌ 错误3：在同步函数里 await
def normal_func():
    await asyncio.sleep(1)       # SyntaxError! 普通函数不能 await

async def coroutine_func():
    await asyncio.sleep(1)       # ✅ 只有 async def 的函数才能用 await

# ❌ 错误4：create_task 之后没保存引用，Task 可能被 GC 回收
async def main():
    asyncio.create_task(fetch(1))   # 没保存引用，Task 可能还没跑就没了

    task = asyncio.create_task(fetch(1))  # ✅ 保存引用
    await task

# ❌ 错误5：在已有事件循环的环境用 asyncio.run()（如 Jupyter）
# asyncio.run(main())   # RuntimeError

# ✅ Jupyter 里直接用 await
# await main()
```

---

## 速查

```python
# 创建并发任务
tasks = [asyncio.create_task(coro) for coro in coros]
results = await asyncio.gather(*tasks)

# 超时控制
try:
    result = await asyncio.wait_for(slow_coro, timeout=5.0)
except asyncio.TimeoutError:
    print("超时")

# 按完成顺序处理
for done in asyncio.as_completed(tasks):
    result = await done
```

---

---

---

## 三个核心 API

```python
import asyncio

# ① 定义协程函数 — async def 不是普通函数，调用它返回 coroutine 对象，不会立即执行
async def say(name: str, delay: float) -> str:
    await asyncio.sleep(delay)    # 异步版 time.sleep()，不阻塞事件循环
    return f"{name} 说完了"

# ② 运行 — asyncio.run() 创建事件循环 + 跑协程 + 清理，整个程序只调一次
result = asyncio.run(say("小明", 1.0))

# ③ 创建 Task — 不等待，立即把协程"提交到后台"跑
task = asyncio.create_task(say("小红", 2.0))
```

```text
asyncio.run() vs asyncio.create_task():

asyncio.run(coro):
```

```mermaid
flowchart TB
    R1["① 创建事件循环"] --> R2["② 把 coro 注册进去"]
    R2 --> R3["③ 死循环跑，直到 coro 完成"]
    R3 --> R4["④ 关闭循环，返回 coro 的结果"]
```

```text
  一个程序只调一次，是整个异步程序的"入口"

asyncio.create_task(coro):
```

```mermaid
flowchart TB
    C1["把 coro 包装成 Task<br/>丢进当前事件循环的待执行队列"] --> C2["立即返回 Task 对象（不阻塞！）"]
```

```text
  拿到的 task 是一个"承诺"——"我会跑的，你先忙"
```

---

## 串行 vs 并发

```python
import asyncio

# ============================================================
# ❌ 串行 — await 一个一个等，白写了 async
# ============================================================
async def download_serial(urls: list[str]) -> list[str]:
    results = []
    for url in urls:
        result = await fetch(url)   # 等 url1 完成才发 url2，跟同步一样慢
        results.append(result)
    return results
    # 10 个 URL → ~10s（每个 1s）

# ============================================================
# ✅ 并发 — create_task 全部发射 + gather 统一收
# ============================================================
async def download_concurrent(urls: list[str]) -> list[str]:
    tasks = [asyncio.create_task(fetch(url)) for url in urls]
    results = await asyncio.gather(*tasks)
    return results
    # 10 个 URL → ~1s（最慢的那个决定总时间）

async def fetch(url: str) -> str:
    print(f"开始请求 {url}")
    await asyncio.sleep(1.0)
    print(f"完成请求 {url}")
    return f"{url} 的内容"
```

```text
时间线对比:

串行 (await 一个一个来):
URL1: [====1s====]
URL2:              [====1s====]
URL3:                           [====1s====]
总时间: 3s

并发 (create_task + gather):
URL1: [====1s====]
URL2: [====1s====]    ← 三个同时跑！
URL3: [====1s====]
总时间: 1s
```

> [!note] 核心原则
> `create_task()` 是"发射"——立即返回不阻塞；`await` 是"等待"——暂停当前协程等结果。**先 create_task 发射全部，再 gather 统一收**，这才是正确的并发姿势。

---

## gather vs Task 四种用法

```python
import asyncio

async def fetch(id: int) -> dict:
    await asyncio.sleep(0.5)
    return {"id": id, "data": f"数据{id}"}

async def main():
    # ============================================================
    # 用法1: gather — 等全部完成，按传入顺序返回结果列表
    # ============================================================
    results = await asyncio.gather(
        fetch(1),
        fetch(2),
        fetch(3),
    )
    # results = [{"id":1,"data":"数据1"}, {"id":2,...}, {"id":3,...}]
    # 返回顺序 = 传入顺序，不是完成顺序

    # ============================================================
    # 用法2: as_completed — 谁先完成先处理谁
    # ============================================================
    tasks = [asyncio.create_task(fetch(i)) for i in range(5)]
    for done_task in asyncio.as_completed(tasks):
        result = await done_task
        print(f"最快完成: {result}")

    # ============================================================
    # 用法3: return_exceptions — 某个任务挂了不影响其他
    # ============================================================
    async def risky_fetch(id: int) -> str:
        if id == 2:
            raise ValueError(f"{id} 炸了")
        await asyncio.sleep(0.3)
        return f"ok_{id}"

    results = await asyncio.gather(
        risky_fetch(1),
        risky_fetch(2),
        risky_fetch(3),
        return_exceptions=True,
    )
    # results = ["ok_1", ValueError("2 炸了"), "ok_3"]
    for i, r in enumerate(results):
        if isinstance(r, Exception):
            print(f"任务{i+1} 失败: {r}")
        else:
            print(f"任务{i+1} 成功: {r}")

    # ============================================================
    # 用法4: wait_for — 加超时控制
    # ============================================================
    async def slow_fetch():
        await asyncio.sleep(10)
        return "太慢了"

    try:
        result = await asyncio.wait_for(slow_fetch(), timeout=2.0)
    except asyncio.TimeoutError:
        print("超时！2 秒没返回，取消请求")
```

---

## 常见错误

```python
# ❌ 错误1：忘记 await — 协程没被执行
async def get_data():
    return "data"

async def main():
    result = get_data()          # 返回 coroutine 对象，不是 "data"！
    print(type(result))          # <class 'coroutine'>

    result = await get_data()    # ✅ await 才真正执行
    print(result)                # "data"

# ❌ 错误2：在协程里用 time.sleep() — 阻塞整个事件循环
import time
async def bad():
    time.sleep(5)                # 整个线程停 5 秒，所有协程都被卡住
    return "done"

async def good():
    await asyncio.sleep(5)       # ✅ 把控制权还给事件循环，别的协程可以跑
    return "done"

# ❌ 错误3：在同步函数里 await
def normal_func():
    await asyncio.sleep(1)       # SyntaxError! 普通函数不能 await

async def coroutine_func():
    await asyncio.sleep(1)       # ✅ 只有 async def 的函数才能用 await

# ❌ 错误4：create_task 之后没保存引用，Task 可能被 GC 回收
async def main():
    asyncio.create_task(fetch(1))   # 没保存引用，Task 可能还没跑就没了

    task = asyncio.create_task(fetch(1))  # ✅ 保存引用
    await task

# ❌ 错误5：在已有事件循环的环境用 asyncio.run()（如 Jupyter）
# asyncio.run(main())   # RuntimeError

# ✅ Jupyter 里直接用 await
# await main()
```

---

## 速查

```python
# 创建并发任务
tasks = [asyncio.create_task(coro) for coro in coros]
results = await asyncio.gather(*tasks)

# 超时控制
try:
    result = await asyncio.wait_for(slow_coro, timeout=5.0)
except asyncio.TimeoutError:
    print("超时")

# 按完成顺序处理
for done in asyncio.as_completed(tasks):
    result = await done
```

---

## 速记卡（面试闪卡）

**Q1：一句话讲清「测试 1：纯计算」到底是什么？**
A：| 场景 | 类型 | 推荐方案 | 原因 |
| :--- | :---: | :--- | :--- |
| Web API 服务器（CRUD） | IO 密集 | **asyncio** | 每个请求大部分时间在等数据库/其他服务 |
| 爬虫（大量 HTTP 请求） | IO 密集 | **asyncio + aiohttp** | 等网络响应的时间占比 >95% |

**Q2：第五章：IO 密集型 vs CPU 密集型——怎么判断、怎么量化 —— 怎么理解？**
A：| 场景 | 类型 | 推荐方案 | 原因 |
| :--- | :---: | :--- | :--- |
| Web API 服务器（CRUD） | IO 密集 | **asyncio** | 每个请求大部分时间在等数据库/其他服务 |
| 爬虫（大量 HTTP 请求） | IO 密集 | **asyncio + aiohttp** | 等网络响应的时间占比 >95% |

**Q3：三个核心 API —— 怎么理解？**
A：---

**Q4：串行 vs 并发 —— 怎么理解？**
A：[!note] 核心原则
 是"发射"——立即返回不阻塞； 是"等待"——暂停当前协程等结果。**先 create_task 发射全部，再 gather 统一收**，这才是正确的并发姿势。
---

**Q5：gather vs Task 四种用法 —— 怎么理解？**
A：---

**Q6：核心速记主线有哪些？**
A：抓住这几根：第五章：IO 密集型 vs CPU 密集型——怎么判断、怎么量化、三个核心 API、串行 vs 并发、gather vs Task 四种用法、常见错误、速查。


## 相关链接

- 下一篇：[[02-aiohttp异步HTTP客户端]]

---
→ [[技术学习清单#asyncio（AI 后端核心）]]

## 相关链接

- 下一篇：[[02-aiohttp异步HTTP客户端]]

---
→ [[技术学习清单#asyncio（AI 后端核心）]]


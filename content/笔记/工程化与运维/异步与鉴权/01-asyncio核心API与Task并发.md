---
title: "asyncio 核心 API 与 Task 并发"
created: "2025-07-12"
tags:
  - 技术学习
  - python
  - asyncio
---

# asyncio 核心 API 与 Task 并发


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
async def download_serial(urls: list[str]) -> list[str]:
    results = []
    for url in urls:
        result = await fetch(url)   # 等 url1 完成才发 url2，跟同步一样慢
        results.append(result)
    return results
    # 10 个 URL → ~10s（每个 1s）

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

**Q1：一句话讲清「asyncio 核心 API 与 Task 并发」到底是什么？**
A：asyncio 核心是用事件循环跑协程，create_task 发射、gather 收网才是真并发。

**Q2：三个核心 API —— 怎么理解？**
A：async def 定义协程（调用只返回 coroutine 不执行）；asyncio.run() 建循环跑协程，全程只调一次；create_task() 把协程丢进后台立刻返回。就像餐厅：菜谱是协程，开门营业是 run，把单子甩给后厨是 create_task（coroutine / event loop）。

**Q3：串行 vs 并发 —— 怎么理解？**
A：串行是一个一个 await，等于白写 async；并发是先 create_task 把全部任务发射，再 gather 统一收。时间线从 3s 变 1s，像三桌菜同时炒而不是一桌炒完再下一桌（concurrency）。

**Q4：gather 与 Task 四种用法 —— 怎么理解？**
A：gather 按传入顺序返回全部结果；as_completed 谁先好先处理；return_exceptions 让某个挂了不影响其他；wait_for 加超时。像四种收单方式：等齐、先到先上、漏单不停、超时就撤（gather / as_completed / wait_for）。

**Q5：五个常见错误 —— 怎么理解？**
A：忘 await 协程不执行；用 time.sleep 阻塞整个循环；普通函数里 await 报语法错；create_task 不保存引用会被 GC；已有循环里再 run 抛异常。像新手厨师：菜没下锅、占着灶台、在禁火区点火（common pitfalls）。

**Q6：核心速记主线有哪些？**
- 入口：asyncio.run() 建循环，全程只调一次
- 发射：create_task 立即返回不阻塞，要保存引用
- 收网：先发射全部再 gather，勿串行 await
- 避坑：别用 time.sleep，await 才真正执行

**口诀**
A：async 写菜谱，run 开门营业忙；
create_task 甩后厨，发射不堵当场畅；
gather 收网等齐回，as_completed 先到上；
忘 await 菜不熟，超时 wait_for 莫硬扛。

## 相关链接

- 下一篇：[[02-aiohttp异步HTTP客户端]]

---
→ [[技术学习路线图#asyncio（AI 后端核心）]]

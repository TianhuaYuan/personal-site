---
title: "aiohttp 异步 HTTP 客户端"
created: "2025-07-12"
tags:
  - 技术学习
  - python
  - aiohttp
  - http
---

# aiohttp 异步 HTTP 客户端


---

## 为什么不用 requests？

```text
requests（同步）:
  发请求 → 等网络返回 → 拿结果 → 发下一个
  I/O 等待期间，Python 线程被挂起，但 GIL 让别的代码也跑不了

aiohttp（异步）:
  发请求1 → 不等！→ 发请求2 → 不等！→ 发请求3 → ...
  所有请求同时在飞，I/O 等待期间事件循环去推进其他协程
```

**100 个 URL，每个延迟 1s：requests 串行 ~100s，aiohttp 并发 ~1s。差 100 倍。**

---

## 基础用法

```bash
pip install aiohttp
```

### ① 最简 GET

```python
import aiohttp

async def fetch_one(url: str) -> str:
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.text()
```

### ② 并发抓多个 URL — 复用 Session

```python
import asyncio
import aiohttp

async def fetch_many(urls: list[str]) -> list[dict]:
    async with aiohttp.ClientSession() as session:
        tasks = [asyncio.create_task(fetch_with_session(session, url)) for url in urls]
        return await asyncio.gather(*tasks)

async def fetch_with_session(session: aiohttp.ClientSession, url: str) -> dict:
    try:
        async with session.get(url, timeout=aiohttp.ClientTimeout(total=10)) as resp:
            return {
                "url": url,
                "status": resp.status,
                "body": await resp.text(),
            }
    except asyncio.TimeoutError:
        return {"url": url, "status": 0, "body": "TIMEOUT"}
    except Exception as e:
        return {"url": url, "status": 0, "body": str(e)}
```

### ③ POST 请求 — 带 JSON body

```python
async def post_json(url: str, data: dict) -> dict:
    async with aiohttp.ClientSession() as session:
        async with session.post(url, json=data) as resp:
            return await resp.json()
```

---

## 并发限流 — Semaphore

1000 个 URL 不能同时发，限 20 个并发。

```python
async def fetch_with_limit(urls: list[str], concurrency: int = 20) -> list:
    semaphore = asyncio.Semaphore(concurrency)

    async def bounded_fetch(session, url):
        async with semaphore:
            return await fetch_with_session(session, url)

    async with aiohttp.ClientSession() as session:
        tasks = [asyncio.create_task(bounded_fetch(session, url)) for url in urls]
        return await asyncio.gather(*tasks)
```

```text
Semaphore(3) 时的执行流：

  任务1: [======跑======]
  任务2: [=====跑=====]
  任务3: [========跑========]
  任务4: ..............等待..............[===跑===]
  任务5: ..............等待...................[==跑==]
         ↑ 最多3个同时在跑，其余排队
```

---

## FastAPI 集成

FastAPI 本身运行在 asyncio 事件循环上，路由里调外部 API 必须用异步 HTTP 客户端，否则阻塞整个事件循环。

```python
from fastapi import FastAPI
import aiohttp

app = FastAPI()

@app.get("/ai/chat")
async def ai_chat(message: str):
    """调外部 AI API — 必须用异步 HTTP 客户端"""
    async with aiohttp.ClientSession() as session:
        async with session.post(
            "https://api.deepseek.com/v1/chat/completions",
            headers={"Authorization": f"Bearer {DEEPSEEK_KEY}"},
            json={
                "model": "deepseek-chat",
                "messages": [{"role": "user", "content": message}],
            },
        ) as resp:
            data = await resp.json()
            return {"reply": data["choices"][0]["message"]["content"]}
```

---

## 速查 — 并发爬取模板

```python
import aiohttp, asyncio

async def fetch_all(urls: list[str], concurrency: int = 10) -> list[dict]:
    sem = asyncio.Semaphore(concurrency)
    async with aiohttp.ClientSession() as session:
        async def fetch_one(url):
            async with sem:
                async with session.get(url, timeout=aiohttp.ClientTimeout(total=15)) as resp:
                    return {"url": url, "status": resp.status, "body": await resp.text()}
        return await asyncio.gather(*[fetch_one(u) for u in urls])
```

---


## 速记卡（面试闪卡）

**Q1：一句话讲清「aiohttp 异步 HTTP 客户端」到底是什么？**
A：aiohttp 让一个线程同时"放出去"很多 HTTP 请求，谁先回来先处理，不再干等。

**Q2：一、为什么不用 requests —— 怎么理解？ —— 怎么理解？**
A：像点外卖：requests 是同步的，你下一个单就蹲门口等送到才下下一个，100 单等 100 秒；aiohttp 是异步的，100 单一起下单，哪个骑手先到拿哪个，约 1 秒搞定。差距来自 I/O 等待期间线程不再傻等。英文：synchronous vs asynchronous I/O。

**Q3：二、基础用法与并发抓取 —— 怎么理解？ —— 怎么理解？**
A：像开个外卖调度中心：建一个 ClientSession 当总台，async with session.get(url) 发单，asyncio.gather 把一堆任务一起 await。关键是一个 session 要复用，别每个请求都新建；超时要给 ClientTimeout 兜底。英文：ClientSession / asyncio.gather。

**Q4：三、并发限流 Semaphore —— 怎么理解？ —— 怎么理解？**
A：像只放 20 个取餐窗口：1000 个 URL 不能全涌进来，用 asyncio.Semaphore(20) 卡住并发数，其余排队。否则瞬间打爆对方服务器或被自己内存撑爆。英文：Semaphore（信号量）。

**Q5：四、FastAPI 集成与速查 —— 怎么理解？ —— 怎么理解？**
A：像在餐厅后厨调外卖平台：FastAPI 本身就跑在 asyncio 事件循环上，路由里调外部 AI API 必须用异步客户端，否则会阻塞整个循环、所有请求卡死。模板就是"建 session→限流→gather→收集结果"。英文：event loop / non-blocking。

**Q6：核心速记主线有哪些？**
- 本质：异步 I/O 让单线程并发传很多请求，I/O 等待期不空转
- 替代 requests：同样 100 个 URL，aiohttp 并发比同步快约百倍
- 关键组件：ClientSession 复用、asyncio.gather 并发、ClientTimeout 兜底
- 限流必备：Semaphore 控并发数，防打爆下游或撑爆自己
- 集成要点：FastAPI 路由内必须用异步客户端，否则阻塞事件循环

**口诀**
A：requests 同步苦等单，aiohttp 并发一锅端；
单线程管多连接，I/O 空档别空盼；
Semaphore 卡窗口，二十并发莫泛滥；
FastAPI 莫阻塞，事件循环稳如磐。

## 相关链接

- 上一篇：[[01-asyncio核心API与Task并发]]
- 下一篇：[[03-JWT原理与设计]]

---
→ [[技术学习路线图#asyncio（AI 后端核心）]]

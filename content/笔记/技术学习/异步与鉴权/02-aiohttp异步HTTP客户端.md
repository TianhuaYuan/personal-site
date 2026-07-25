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

## 相关链接

- 上一篇：[[01-asyncio核心API与Task并发]]
- 下一篇：[[03-JWT原理与设计]]
- 项目实践：[[笔记/技术学习/项目开发笔记/ai-resume笔记/01_技术研读/01_架构概览|ai-resume: 架构概览]]
- 项目实践：[[笔记/技术学习/项目开发笔记/cr-agent笔记/01-技术研读/04-三层容错与并发bug|cr-agent: 三层容错与并发bug]]

---
→ [[技术学习清单#asyncio（AI 后端核心）]]

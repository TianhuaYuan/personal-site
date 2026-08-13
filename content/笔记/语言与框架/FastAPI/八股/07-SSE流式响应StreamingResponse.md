---

title: "SSE 流式响应 StreamingResponse"

created: "2026-07-20"

tags:

  - 八股文

  - fastapi-web

---



# SSE 流式响应 StreamingResponse

## 一句话总结



> **StreamingResponse 让接口“一点一点吐数据”，而不是等全部算完再一次性返回。SSE（Server-Sent Events，服务器推送事件）是它最经典的玩法：用 `text/event-stream` 格式，服务端像发弹幕一样逐条推给浏览器，前端用 `EventSource` 接收——ChatGPT 那种逐字输出就是这么干的。**



---



## 生活类比：自来水 vs 桶装水



- **普通响应**：服务端烧好一整桶水（算完全部结果），一次性扛给你。慢任务时你要干等。

- **流式响应**：服务端接上水管（异步生成器），水一来就流一点，你边接边用。ChatGPT 边生成边显示，就是“水管模式”。



## StreamingResponse 基础



```python

from fastapi import FastAPI

from fastapi.responses import StreamingResponse

import asyncio



app = FastAPI()



async def gen():

    for i in range(10):

        yield f"chunk {i}\n"      # 每个 yield 立刻 flush 给客户端

        await asyncio.sleep(0.5)



@app.get("/stream")

def stream():

    return StreamingResponse(gen(), media_type="text/plain")

```



关键点：`gen` 是**异步生成器**，每次 `yield` 的内容立刻发往客户端（带 backpressure 感知）。



## SSE 格式（重点）



SSE 是 HTML5 标准，规定事件用 `data: ...\n\n` 双换行分隔：



```text

data: {"token": "你"}



data: {"token": "好"}



event: done

data: [DONE]

```



```python

@app.get("/chat/stream")

async def chat_stream(prompt: str):

    async def event_gen():

        for tok in generate_tokens(prompt):          # 伪代码: LLM 逐 token

            yield f"data: {json.dumps({'token': tok})}\n\n"

        yield "event: done\ndata: [DONE]\n\n"

    return StreamingResponse(

        event_gen(),

        media_type="text/event-stream",

        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"},

    )

```



> `text/event-stream` 是 SSE 的 MIME 类型；`X-Accel-Buffering: no` 防止 Nginx 等代理缓冲把流“攒成一批”才发。



## FastAPI 的 `EventSourceResponse`（更省心）



FastAPI 直接提供 `fastapi.sse.EventSourceResponse` + `ServerSentEvent`，自动处理格式、keepalive、断线续传：



```python

from fastapi.sse import EventSourceResponse, ServerSentEvent

from collections.abc import AsyncIterable



@app.get("/items/stream", response_class=EventSourceResponse)

async def stream_items() -> AsyncIterable[ServerSentEvent]:

    for i, item in enumerate(items):

        yield ServerSentEvent(data=item, event="item_update", id=str(i + 1), retry=5000)

```



它默认帮你：

- 加 `Cache-Control: no-cache` 和 `X-Accel-Buffering: no`

- 每 ~15 秒发 `: keepalive` 注释行，防止代理空闲断开

- `data` 自动 JSON 编码；需要原样文本用 `raw_data`



## 断线续传：Last-Event-ID



浏览器断开重连时会带上 `Last-Event-ID` 头。服务端读它，从上次 ID 之后接着推：



```python

from typing import Annotated

from fastapi import Header



@app.get("/stream", response_class=EventSourceResponse)

async def stream(last_event_id: Annotated[int | None, Header()] = None):

    start = last_event_id + 1 if last_event_id is not None else 0

    # 从 start 之后继续 yield ...

```



## StreamingResponse vs WebSocket vs SSE



| 维度 | SSE（StreamingResponse） | WebSocket |
| :--- | :--- | :--- |
| 方向 | 服务器→客户端（单向） | 双向全双工 |
| 协议 | 普通 HTTP（长连接） | 独立 WS 协议（握手升级） |
| 客户端 API | 浏览器原生 `EventSource` | 需 `WebSocket` 对象 |
| 适用 | 日志流、LLM 逐字、通知推送 | 聊天、实时游戏、协同编辑 |



> 和 [[计算机基础/计算机网络/00-计算机网络|计算机网络 八股文]] 互补：那边讲协议与浏览器端，这边讲 FastAPI 后端实现。



## 生产注意



- **断开检测**：`if await request.is_disconnected(): break`，避免给已走客户端白干活

- **keepalive**：空闲时发注释行 `: ping\n\n` 防代理超时

- **背压**：生成太快客户端处理不过来时，TCP 缓冲满会自然减速（backpressure）

- **NDJSON** 也是流式选项：`application/x-ndjson`，每行一个 JSON，适合程序化客户端



## 记忆口诀



> **StreamingResponse = 异步生成器，yield 一次发一次。**

> **SSE 格式 data: 结尾双换行，浏览器 EventSource 收。**

> **EventSourceResponse 更省心：自动 keepalive + 续传。**

> **单向推送用 SSE，双向聊天用 WebSocket。**



---



##

> ▶ 对应实操：[[07-中间件|07-中间件]]





> ▶ 对应实操：[[03-SSE流式响应与Backpressure控制|03-SSE流式响应与Backpressure控制]]





## 速记卡（面试闪卡）



**Q1：一句话讲清「SSE 流式响应 StreamingResponse」到底是什么？**

A：SSE（Server-Sent Events）是服务端向浏览器单向推送事件的 HTTP 技术，配合 StreamingResponse 实现逐字流式输出。



**Q2：生活类比：自来水 vs 桶装水 —— 怎么理解？**

A：普通响应像桶装水——服务端烧好一整桶（算完全部结果）一次性扛给你，慢任务只能干等；流式响应像自来水——服务端接上水管（异步生成器，英文 Async Generator），水一来就流一点，你边接边用。ChatGPT 边生成边显示就是"水管模式"。



**Q3：SSE 格式（重点） —— 怎么理解？**

A：SSE 是 HTML5 标准，`text/event-stream` 格式，事件用 `data: ...\n\n` 双换行分隔，结束发 `event: done\ndata: [DONE]`。关键响应头 `X-Accel-Buffering: no` 防止 Nginx 等代理把流攒成一批才发。前端用浏览器原生 `EventSource` 接收，不用额外库。



**Q4：FastAPI 的 EventSourceResponse（更省心） —— 怎么理解？**

A：FastAPI 直接提供 `EventSourceResponse` + `ServerSentEvent`，自动处理格式、keepalive（每约 15 秒发 `: keepalive` 注释防代理断开）、断线续传，比手写 StreamingResponse 省心。默认加 `Cache-Control: no-cache`，`data` 自动 JSON 编码，需要原样文本用 `raw_data`。



**Q5：StreamingResponse vs WebSocket vs SSE —— 怎么理解？**

A：三者方向不同：SSE 是服务器→客户端单向推送（普通 HTTP 长连接，前端用 EventSource），适合日志流、LLM 逐字、通知；WebSocket 是双向全双工（独立 WS 协议握手升级），适合聊天、实时游戏、协同编辑。要双向聊天用 WebSocket，单向推送用 SSE。



**Q6：核心速记主线有哪些？**

- 流式 vs 桶装：StreamingResponse 用异步生成器 yield 一次发一次

- SSE 格式：data: 结尾双换行，text/event-stream，X-Accel-Buffering: no

- EventSourceResponse 更省心：自动 keepalive + 续传

- SSE 单向推送、WebSocket 双向；选 SSE 做 LLM 逐字输出



**口诀**

A：桶装水慢等整桶，水管模式边流边用；

SSE 双换行 data，EventSource 前端收；

EventSourceResponse 省心，keepalive 防断流；

单向推送用 SSE，双向聊天 WebSocket。



相关链接



- [[04-中间件Middleware机制|中间件]]

- [[06-JWT鉴权全链路|JWT 鉴权]]

- [[05-def与async-def路由选择|def vs async 路由]]

- [[计算机基础/计算机网络/00-计算机网络|计算机网络 八股文]]

- [[00-FastAPI-Web|FastAPI-Web 索引]]

## 相关链接



- [[笔记/语言与框架/FastAPI/八股/12-BackgroundTasks与Celery对比|BackgroundTasks vs Celery 对比]]

- [[笔记/语言与框架/FastAPI/八股/10-SQLAlchemy2.0异步集成|SQLAlchemy 2.0 异步集成]]

- [[笔记/语言与框架/FastAPI/八股/03-Pydantic数据校验与v2新特性|Pydantic 数据校验与 v2 新特性]]

- [[笔记/语言与框架/FastAPI/八股/02-依赖注入Depends原理与生命周期|依赖注入 Depends 原理与生命周期]]

- [[笔记/语言与框架/FastAPI/八股/04-中间件Middleware机制|中间件 Middleware 机制]]


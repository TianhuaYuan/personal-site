---
title: "中间件 Middleware 机制"
created: "2026-07-20"
tags:
  - 八股文
  - fastapi-web
---

# 中间件 Middleware 机制

## 一句话总结

> **中间件是“每个请求进路由前、出路由后都会经过的一道门”：统一加日志、计时、CORS、压缩、HTTPS 重定向。它像洋葱——请求从外往里穿，响应从里往外穿；后注册的中间件包在最外层。**

---

## 生活类比：小区门禁 + 快递柜

把请求想成快递员进小区：
- **请求进入时**（前处理）：门禁登记、测体温、分配通行码
- **到达你家（路由）**：你收件
- **响应返回时**（后处理）：快递柜贴条、记送达时间

中间件就是那道**所有快递员都必须经过的门禁**，而不是你在家门口给每个快递员单独装锁。

## 中间件长什么样

```python
import time
from fastapi import FastAPI, Request

app = FastAPI()

@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start = time.perf_counter()        # 前处理
    response = await call_next(request) # 把请求交给下一层(路由/内层中间件)
    process_time = time.perf_counter() - start
    response.headers["X-Process-Time"] = str(process_time)  # 后处理
    return response
```

- `request`：进来的请求
- `call_next`：一个函数，把请求传下去，返回路由生成的响应
- 在 `call_next` **前**改请求，在 **后**改响应

> 自定义专有响应头可用 `X-` 前缀；要让浏览器 JS 能读到，得在 CORS 的 `expose_headers` 里声明。

## 执行顺序：洋葱模型（必考）

```mermaid
flowchart TD
    subgraph 洋葱模型 [请求方向: 外→里 | 响应方向: 里→外]
        B[MiddlewareB 前处理<br/>后注册=最外层] --> A[MiddlewareA 前处理<br/>先注册=最内层]
        A --> R[路由函数]
        R --> A2[MiddlewareA 后处理]
        A2 --> B2[MiddlewareB 后处理]
    end
```

代码顺序：`add_middleware(A)` 先，`add_middleware(B)` 后 →

| 阶段 | 执行顺序 |
| :--- | :--- |
| 请求 | B 前 → A 前 → 路由 |
| 响应 | 路由 → A 后 → B 后 |

> **记忆法**：后注册的中间件**先**跑前处理、**后**跑后处理（像栈：后进先出前置，先进后出后置）。

## 内置中间件（都来自 Starlette）

FastAPI 继承自 Starlette，直接 `app.add_middleware(...)` 即可：

| 中间件 | 作用 |
| :--- | :--- |
| `CORSMiddleware` | 跨域（见 [[08-CORS跨域配置]]） |
| `GZipMiddleware` | 响应压缩，省带宽 |
| `HTTPSRedirectMiddleware` | 强制 HTTP→HTTPS |
| `TrustedHostMiddleware` | 限制允许访问的 Host |
| `SessionMiddleware` | 基于 cookie 的会话 |

```python
from fastapi.middleware.gzip import GZipMiddleware
app.add_middleware(GZipMiddleware, minimum_size=1000)
```

## 中间件 vs 依赖注入：何时用谁？

| 场景 | 用中间件 | 用 Depends |
| :--- | :--- | :--- |
| 全局统一逻辑（日志/计时/CORS） | ✅ 每个请求都过 | ❌ |
| 特定路由需要的资源（db/当前用户） | ❌ | ✅ 精准注入 |
| 需要访问路由函数返回值做包装 | ✅ 后处理改响应 | ⚠️ 较绕 |
| 需要请求级缓存/复用 | 中 | ✅ `use_cache` |

简单说：**中间件管“横切全局”，依赖注入管“路由专属资源”**。

## ⚠️ 与 yield 依赖、BackgroundTasks 的顺序坑

- 带 `yield` 的依赖，其**收尾代码在中间件之后**执行
- `BackgroundTasks`（后台任务）在所有中间件**之后**执行

所以如果你在中间件里读了响应头、又指望后台任务改了点什么，顺序要对得上。

## 延伸追问

**Q：中间件能改请求体吗？**
A：能，但要小心——请求体是流式的，读一次就没了。需要多次读取要手动缓存（`request.body()` 会缓冲），注意内存。

**Q：中间件和 Starlette 的关系？**
A：FastAPI 几乎不实现中间件，`@app.middleware("http")` 和 `add_middleware` 都直接来自 Starlette。FastAPI 只是帮你从 `starlette.middleware.*` 重新导出，方便 import。

## 记忆口诀

> **中间件 = 每请求必过的门禁，前处理+后处理。**
> **call_next 把请求传下去，前后各插一刀。**
> **后注册包外层，请求先进后出，响应后进先出。**
> **全局横切用中间件，路由专属用 Depends。**

---

## 
> ▶ 对应实操：[[04-响应模型与状态码|04-响应模型与状态码]]


> ▶ 对应实操：[[07-中间件|07-中间件]]

相关链接

- [[02-依赖注入Depends原理与生命周期|依赖注入 Depends]]
- [[08-CORS跨域配置|CORS 跨域]]
- [[12-BackgroundTasks与Celery对比|BackgroundTasks]]
- [[09-全局异常处理|全局异常处理]]
- [[八股文笔记/计算机网络/00-计算机网络|计算机网络 八股文]]
- [[00-FastAPI-Web|FastAPI-Web 索引]]

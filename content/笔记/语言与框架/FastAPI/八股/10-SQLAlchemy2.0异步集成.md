---
title: "SQLAlchemy 2.0 异步集成"
created: "2026-07-20"
tags:
  - 八股文
  - fastapi-web
---

# SQLAlchemy 2.0 异步集成

## 一句话总结

> **SQLAlchemy 2.0 异步三件套：`create_engine`→`create_async_engine`、`Session`→`AsyncSession`、查询统一用 `select()` 并加 `await`。再配合 FastAPI 的 `Depends` + `yield` 管理会话生命周期，实现“每请求一会话、用完即关”。**

---

## 生活类比：图书馆借阅

`AsyncSession` 像一张借书卡。你（请求）借卡进馆（连接池取连接），挑书查资料（`await session.execute(select(...))`），看完还卡（`async with` 自动关，连接回池）。异步的意思是：你查字典等书送来的空档，馆员去服务别人，而不是干站着。

## 三件套

### 1. 异步引擎

```python
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/db",
    pool_size=20, max_overflow=10, pool_pre_ping=True,
)
```

> 注意 **`+asyncpg`** 驱动后缀——同步用 `psycopg2`，异步必须用 `asyncpg`（PostgreSQL）或 `aiomysql`（MySQL）等异步驱动。

### 2. 异步 Session 工厂

```python
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker

AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)
```

### 3. 查询（2.0 风格，统一 `select`）

```python
from sqlalchemy import select

# 2.0 同步
user = session.execute(select(User).where(User.name == "Alice")).scalar_one()

# 2.0 异步 (必须 await)
result = await session.execute(select(User).where(User.id == user_id))
user = result.scalar_one()
```

> 2.0 推荐用 **`select()` 构造器**取代老的 `session.query(Model)`（已弃用）。`add`/`scalar_one` 不需要 await，只有 `execute`/`scalar`/`commit` 等 IO 操作需要。

## FastAPI 集成完整流程

```python
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:   # 预处理: 取会话
        yield session                             # 交给路由
        # 后处理: async with 结束自动关连接回池

@app.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).where(User.id == user_id))
    user = result.scalar_one_or_none()
    if not user:
        raise HTTPException(status_code=404)
    return user
```

详见 [[02-依赖注入Depends原理与生命周期|依赖注入 Depends]] 的 yield 生命周期。

## 四大陷阱（常踩）

| 陷阱 | 解法 |
| :--- | :--- |
| `AsyncSession` 和 `Session` 混用 | 统一用 `AsyncSession` + `async_sessionmaker` |
| `expire_on_commit=True` 导致响应后访问属性崩 | 设 `expire_on_commit=False` |
| 关系懒加载在 async 下报错（无事件循环执行 lazy SQL） | 用 `selectinload`/`joinedload` 提前加载，见 ORM N+1 |
| session 没正常关闭导致连接泄露 | 始终用 `async with` 管理生命周期 |

## 2.0 vs 1.x 关键差异

| 维度 | 1.x | 2.0 |
| :--- | :--- | :--- |
| 查询 API | `session.query(Model)` | `select(Model)` + `execute` |
| 引擎 | `create_engine` | 异步用 `create_async_engine` |
| 映射声明 | `declarative_base()` | `DeclarativeBase`（`mapped_column` 风格） |
| 配置 | `class Config` | `model_config` / `mapped_column` |

## 原理：异步引擎怎么不阻塞事件循环

`create_async_engine` 底层用异步驱动（asyncpg）在网络 IO 时 `await`，把控制权交还事件循环，于是 FastAPI 在等 DB 期间能处理别的请求。这正好契合 [[01-FastAPI为什么快-Starlette与Pydantic与async|FastAPI 的异步内核]]。

## 记忆口诀

> **Engine 加 async，Session 变 Async。**
> **查询用 select，execute 加 await；add/scalar 不用 await。**
> **expire_on_commit 设 False，lazy load 换 eager。**
> **async with 管生命周期，连接不泄露。**

---

## 
> ▶ 对应实操：[[10-Rate-Limiting|10-Rate-Limiting]]


> ▶ 对应实操：[[11-异步SQLAlchemy与连接池|11-异步SQLAlchemy与连接池]]

相关链接

- [[11-ORM关系映射与N+1查询解决|ORM 关系与 N+1]]
- [[02-依赖注入Depends原理与生命周期|依赖注入 Depends]]
- [[05-def与async-def路由选择|def vs async 路由]]
- [[13-FastAPI性能优化-连接池与uvicorn-workers|连接池与性能]]
- [[语言与框架/MySQL/八股/00-MySQL|MySQL 八股文]]
- [[00-FastAPI-Web|FastAPI-Web 索引]]

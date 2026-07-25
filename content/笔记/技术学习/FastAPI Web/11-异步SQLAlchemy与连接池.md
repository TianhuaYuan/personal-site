---
title: "异步 SQLAlchemy + 连接池：async engine + pool_pre_ping + pool_recycle"
tags:
  - python
  - 技术学习
  - 学习笔记
created: "2026-07-21"
---

# 异步 SQLAlchemy + 连接池：async engine + pool_pre_ping + pool_recycle

> **一句话**：SQLAlchemy 是 Python 最流行的 ORM 库，支持同步和异步操作。异步 SQLAlchemy 避免阻塞事件循环，连接池管理数据库连接复用。在 AI 应用中，常用于异步操作数据库。

## 1. 异步 SQLAlchemy 基础

### 1.1 安装

```bash
pip install sqlalchemy[asyncio]
# 异步驱动
pip install asyncpg  # PostgreSQL
pip install aiosqlite  # SQLite
```

### 1.2 异步引擎

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

# 异步引擎
engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/db",
    echo=True,  # SQL日志
    pool_size=10,  # 连接池大小
    max_overflow=20,  # 最大溢出连接
    pool_pre_ping=True,  # 连接前ping检测
    pool_recycle=3600,  # 连接回收时间（秒）
)

# 异步会话工厂
async_session = sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)
```

## 2. 连接池配置

### 2.1 连接池参数

```mermaid
graph TD
    A[连接池配置] --> B[pool_size: 10]
    A --> C[max_overflow: 20]
    A --> D[pool_pre_ping: True]
    A --> E[pool_recycle: 3600]
    A --> F[pool_timeout: 30]
    
    B --> B1[核心连接数]
    C --> C1[超出核心的连接数]
    D --> D1[连接前检测存活]
    E --> E1[连接最大存活时间]
    F --> F1[获取连接超时]
```

### 2.2 详细配置

```python
engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/db",
    
    # 连接池配置
    pool_size=10,  # 核心连接数（默认5）
    max_overflow=20,  # 超出核心的连接数（默认10）
    pool_pre_ping=True,  # 连接前ping检测（推荐True）
    pool_recycle=3600,  # 连接回收时间（秒，默认-1不回收）
    pool_timeout=30,  # 获取连接超时（秒，默认30）
    pool_use_lifo=True,  # 后进先出（推荐True）
    
    # 连接参数
    connect_args={
        "server_settings": {
            "application_name": "myapp",
        }
    }
)
```

### 2.3 pool_pre_ping 的作用

```python
# 问题：数据库连接可能断开（网络问题、数据库重启等）
# 解决：pool_pre_ping 在每次获取连接前发送 SELECT 1 检测

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/db",
    pool_pre_ping=True,  # ✅ 推荐开启
)
```

### 2.4 pool_recycle 的作用

```python
# 问题：数据库可能有连接超时（如MySQL默认8小时）
# 解决：pool_recycle 定期回收连接

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/db",
    pool_recycle=3600,  # 每小时回收一次
)
```

## 3. 基础用法

### 3.1 模型定义

```python
from sqlalchemy import Column, Integer, String, DateTime
from sqlalchemy.orm import DeclarativeBase
from datetime import datetime

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(50), nullable=False)
    email = Column(String(100), unique=True, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
```

### 3.2 异步CRUD

```python
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

# 创建
async def create_user(session: AsyncSession, name: str, email: str):
    user = User(name=name, email=email)
    session.add(user)
    await session.commit()
    await session.refresh(user)
    return user

# 查询
async def get_user(session: AsyncSession, user_id: int):
    result = await session.execute(
        select(User).where(User.id == user_id)
    )
    return result.scalar_one_or_none()

# 更新
async def update_user(session: AsyncSession, user_id: int, name: str):
    user = await get_user(session, user_id)
    if user:
        user.name = name
        await session.commit()
    return user

# 删除
async def delete_user(session: AsyncSession, user_id: int):
    user = await get_user(session, user_id)
    if user:
        await session.delete(user)
        await session.commit()
```

## 4. FastAPI集成

### 4.1 依赖注入

```python
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import AsyncSession

app = FastAPI()

async def get_session():
    """获取数据库会话"""
    async with async_session() as session:
        try:
            yield session
        finally:
            await session.close()

@app.post("/users/")
async def create_user(
    name: str,
    email: str,
    session: AsyncSession = Depends(get_session)
):
    user = await create_user(session, name, email)
    return {"id": user.id, "name": user.name}
```

### 4.2 会话管理

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def get_session_context():
    """会话上下文管理器"""
    session = async_session()
    try:
        yield session
        await session.commit()
    except Exception:
        await session.rollback()
        raise
    finally:
        await session.close()

# 使用
async def example():
    async with get_session_context() as session:
        user = await create_user(session, "Alice", "alice@example.com")
        # 会话自动提交或回滚
```

## 5. 高级特性

### 5.1 批量操作

```python
from sqlalchemy import insert

async def bulk_create_users(session: AsyncSession, users_data: list):
    """批量创建用户"""
    await session.execute(
        insert(User),
        users_data
    )
    await session.commit()
```

### 5.2 关系查询

```python
from sqlalchemy.orm import selectinload

async def get_user_with_posts(session: AsyncSession, user_id: int):
    """查询用户及其文章"""
    result = await session.execute(
        select(User)
        .options(selectinload(User.posts))  # 预加载关系
        .where(User.id == user_id)
    )
    return result.scalar_one_or_none()
```

### 5.3 事务管理

```python
async def transfer_money(session: AsyncSession, from_id: int, to_id: int, amount: int):
    """转账事务"""
    async with session.begin():
        # 获取账户
        from_account = await session.get(Account, from_id)
        to_account = await session.get(Account, to_id)
        
        # 检查余额
        if from_account.balance < amount:
            raise ValueError("余额不足")
        
        # 执行转账
        from_account.balance -= amount
        to_account.balance += amount
        
        # 自动提交或回滚
```

## 6. 性能优化

### 6.1 连接池监控

```python
from sqlalchemy import event

@event.listens_for(engine.pool, "checkout")
def on_checkout(dbapi_conn, connection_rec, connection_proxy):
    """连接获取事件"""
    print(f"获取连接: {dbapi_conn}")

@event.listens_for(engine.pool, "checkin")
def on_checkin(dbapi_conn, connection_rec):
    """连接归还事件"""
    print(f"归还连接: {dbapi_conn}")
```

### 6.2 查询优化

```python
# 使用 loaded_options 预加载
from sqlalchemy.orm import selectinload

async def get_user_optimized(session: AsyncSession, user_id: int):
    """优化查询"""
    result = await session.execute(
        select(User)
        .options(
            selectinload(User.posts),
            selectinload(User.comments)
        )
        .where(User.id == user_id)
    )
    return result.scalar_one_or_none()
```

## 7. 常见坑点

### 1. 忘记关闭会话
```python
# 错误：未关闭会话导致连接泄漏
async def bad_example():
    session = async_session()
    user = await create_user(session, "Alice", "alice@example.com")
    # 忘记关闭会话

# 正确：使用async with或try/finally
async def good_example():
    async with async_session() as session:
        user = await create_user(session, "Alice", "alice@example.com")
```

### 2. 连接池耗尽
```python
# 问题：并发请求过多，连接池满
# 解决：调整连接池大小或使用异步锁
import asyncio

semaphore = asyncio.Semaphore(10)  # 限制并发数

async def limited_create_user(name: str, email: str):
    async with semaphore:
        async with async_session() as session:
            return await create_user(session, name, email)
```

### 3. 连接超时
```python
# 问题：数据库连接超时
# 解决：开启pool_pre_ping
engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/db",
    pool_pre_ping=True,  # ✅ 连接前检测
)
```

## 核心要点

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

# 异步引擎
engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/db",
    pool_size=10,
    pool_pre_ping=True,
    pool_recycle=3600,
)

# 异步会话
async_session = sessionmaker(engine, class_=AsyncSession)

# CRUD
async with async_session() as session:
    # 创建
    user = User(name="Alice", email="alice@example.com")
    session.add(user)
    await session.commit()
    
    # 查询
    result = await session.execute(select(User).where(User.id == 1))
    user = result.scalar_one_or_none()
```

## 
> ▶ 对应原理：10-SQLAlchemy2.0异步集成

相关链接

- 📋 目录：[[00-FastAPI]]
- 📚 学习清单： Web
- 🔗 [[06-依赖注入|依赖注入]]
- 🔗 [[MySQL 实战/00-MySQL|MySQL学习目录]]
- 🔗 [[08-后台任务与CORS|后台任务与CORS]]
- 项目实践：[[笔记/技术学习/项目开发笔记/ai-resume笔记/01_技术研读/01_架构概览|ai-resume: 架构概览]]
- 项目实践：[[笔记/技术学习/项目开发笔记/cr-agent笔记/01-技术研读/05-API与CLI接口契约|cr-agent: API与CLI接口契约]]

---
title: "请求体与Pydantic"
created: "2025-07-12"
tags:
  - 技术学习
  - fastapi
  - pydantic
---

# 请求体与Pydantic

> 写给初学者：每个概念都从"为什么需要它"开始讲，配合通俗比喻和完整可运行代码。

---

## 一、请求体（Request Body）

### 1.1 什么是请求体？

路径参数和查询参数都放在 URL 里，但有些数据太复杂或太敏感，不适合放在 URL 里。比如：

```json
{
  "username": "张三",
  "email": "zhangsan@qq.com",
  "age": 25
}
```

这种数据放在 HTTP 请求的 **body（请求体）** 里，格式是 JSON。

### 1.2 用 Pydantic 模型接收请求体

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

# 定义请求体的结构
# 继承 BaseModel，声明每个字段的类型
class UserCreate(BaseModel):
    username: str
    email: str
    age: int

# 参数类型是 BaseModel 的子类 → FastAPI 自动从请求体取数据
@app.post("/users")
async def create_user(user: UserCreate):
    return {
        "username": user.username,
        "email": user.email,
        "age": user.age,
        "msg": "用户创建成功"
    }
```

**用 curl 测试：**
```bash
curl -X POST "http://127.0.0.1:8000/users" \
  -H "Content-Type: application/json" \
  -d '{"username": "张三", "email": "zhangsan@qq.com", "age": 25}'
```

**返回：**
```json
{"username": "张三", "email": "zhangsan@qq.com", "age": 25, "msg": "用户创建成功"}
```

### 1.3 FastAPI 怎么判断参数从哪来？

**判断规则（从左到右依次检查）：**

| 条件 | 数据来源 | 例子 |
|------|---------|------|
| 参数名在路径里 `{xxx}` | **路径参数** | `@app.get("/users/{user_id}")` 的 `user_id` |
| 参数类型是 BaseModel 子类 | **请求体** | `user: UserCreate` |
| 其他情况 | **查询参数** | `skip: int = 0` |

```python
@app.put("/users/{user_id}")
async def update_user(
    user_id: int,           # 在路径里 → 路径参数
    user: UserCreate,       # BaseModel 子类 → 请求体
    notify: bool = False    # 其他 → 查询参数
):
    return {
        "user_id": user_id,      # 来自 URL 路径: /users/42
        "username": user.username, # 来自请求体 JSON
        "notify": notify           # 来自 URL 查询: ?notify=true
    }
```

**请求示例：**
```text
PUT /users/42?notify=true
Body: {"username": "张三", "email": "zhangsan@qq.com", "age": 25}
```

### 1.4 请求体 + 路径参数 + 查询参数混用

这是实际开发中最常见的场景：

```python
from fastapi import FastAPI, Path, Query
from pydantic import BaseModel

app = FastAPI()

class ItemCreate(BaseModel):
    name: str
    price: float
    description: str = ""  # 有默认值，可选

@app.post("/users/{user_id}/items")
async def create_item_for_user(
    user_id: int,                    # 路径参数
    item: ItemCreate,                # 请求体
    category: str = "未分类",         # 查询参数（可选）
    urgent: bool = False             # 查询参数（可选）
):
    return {
        "user_id": user_id,
        "item_name": item.name,
        "item_price": item.price,
        "description": item.description,
        "category": category,
        "urgent": urgent
    }
```

**请求示例：**
```text
POST /users/42/items?category=电子&urgent=true
Body: {"name": "键盘", "price": 299.0, "description": "机械键盘"}
```

**返回：**
```json
{
  "user_id": 42,
  "item_name": "键盘",
  "item_price": 299.0,
  "description": "机械键盘",
  "category": "电子",
  "urgent": true
}
```

---

## 二、Query / Path / Body / Field 显式声明（进阶）

### 2.1 为什么需要显式声明？

当 FastAPI 自动判断不够精确时，你可以用 `Query()`、`Path()`、`Body()`、`Field()` 显式声明参数来源和约束。

```python
from fastapi import FastAPI, Query, Path, Body
from pydantic import BaseModel, Field

app = FastAPI()
```

### 2.2 Query —— 给查询参数加约束

```python
@app.get("/items")
async def list_items(
    # Query() 可以加更多约束
    q: str = Query(
        ...,                    # 必填
        min_length=3,           # 最短 3 个字符
        max_length=50,          # 最长 50 个字符
        description="搜索关键词"  # 文档里的描述
    ),
    skip: int = Query(0, ge=0),          # 大于等于 0
    limit: int = Query(10, ge=1, le=100) # 1 到 100 之间
):
    return {"q": q, "skip": skip, "limit": limit}
```

| 请求 | 结果 |
|------|------|
| `/items?q=py` | ❌ 422：`q` 只有 2 个字符，不满足 `min_length=3` |
| `/items?q=python` | ✅ 正常 |
| `/items?q=python&skip=-1` | ❌ 422：`skip` 必须 >= 0 |
| `/items?q=python&limit=200` | ❌ 422：`limit` 必须 <= 100 |

### 2.3 Path —— 给路径参数加约束

```python
@app.get("/users/{user_id}")
async def get_user(
    user_id: int = Path(..., gt=0, description="用户 ID，必须大于 0")
):
    return {"user_id": user_id}
```

### 2.4 Body —— 给请求体加额外信息

```python
class Item(BaseModel):
    name: str
    price: float

class User(BaseModel):
    username: str
    email: str

@app.put("/items/{item_id}")
async def update_item(
    item_id: int,
    item: Item,                                    # 请求体（自动识别）
    user: User = Body(...),                         # 另一个请求体字段
    importance: int = Body(..., gt=0)               # 请求体里的单个字段
):
    return {
        "item_id": item_id,
        "item": item.model_dump(),
        "user": user.model_dump(),
        "importance": importance
    }
```

**请求示例：**
```json
{
  "item": {"name": "键盘", "price": 299},
  "user": {"username": "张三", "email": "zhangsan@qq.com"},
  "importance": 5
}
```

### 2.5 Field —— 给模型字段加约束

`Field()` 定义在 BaseModel 内部，用来约束字段本身的长度、范围、描述等。

```python
from pydantic import BaseModel, Field

class UserCreate(BaseModel):
    # ... 表示必填
    # min_length=3：最少 3 个字符
    # max_length=20：最多 20 个字符
    username: str = Field(..., min_length=3, max_length=20, description="用户名")

    # gt=0：greater than 0，大于 0
    # le=150：less than or equal 150，小于等于 150
    age: int = Field(..., gt=0, le=150, description="年龄")

    # 有默认值就变成可选字段
    email: str = Field(default="未填写", description="邮箱")
```

**Field 常用参数速查：**

| 参数 | 含义 | 例子 |
|------|------|------|
| `...` | 必填字段 | `Field(...)` |
| `default` | 默认值 | `Field(default="未知")` |
| `min_length` | 字符串最短长度 | `Field(min_length=3)` |
| `max_length` | 字符串最长长度 | `Field(max_length=20)` |
| `gt` | greater than（大于） | `Field(gt=0)` |
| `ge` | greater than or equal（大于等于） | `Field(ge=0)` |
| `lt` | less than（小于） | `Field(lt=100)` |
| `le` | less than or equal（小于等于） | `Field(le=100)` |
| `description` | 描述（显示在 API 文档里） | `Field(description="用户名")` |

**测试效果：**
```python
# ✅ 合法
user = UserCreate(username="张三", age=25)

# ❌ 报错：username 只有 2 个字符
user = UserCreate(username="ab", age=25)
# → ValidationError: String should have at least 3 characters

# ❌ 报错：age 不在范围内
user = UserCreate(username="张三", age=200)
# → ValidationError: Input should be less than or equal to 150
```

### 2.6 Field vs Body 的区别

这两个容易搞混，但用途完全不同：

| | `Field()` | `Body()` |
|---|---|---|
| **定义位置** | BaseModel 内部 | 函数参数上 |
| **作用** | 约束字段本身（长度、范围、描述） | 声明参数从请求体取 |
| **影响范围** | 模型在哪用都生效 | 只影响当前函数参数 |
| **典型场景** | 所有请求体都需要校验 | 多个 BaseModel 混用或单个字段从 body 取 |

```python
from fastapi import FastAPI, Body
from pydantic import BaseModel, Field

app = FastAPI()

# Field() —— 定义在模型里，约束字段本身
# 不管这个模型用在哪个接口，username 都必须 3-20 个字符
class UserCreate(BaseModel):
    username: str = Field(..., min_length=3, max_length=20)
    email: str
    age: int = Field(..., gt=0, le=150)

@app.post("/users")
async def create_user(user: UserCreate):
    return user

# Body() —— 用在函数参数上，解决歧义
# 当你需要从请求体里取单个字段（不是整个 BaseModel），用 Body() 告诉 FastAPI
@app.put("/users/{user_id}")
async def update_user(
    user_id: int,
    user: UserCreate,                          # BaseModel → 自动识别为请求体
    admin_note: str = Body(..., min_length=1),  # 单个字段 + Body() → 从请求体取
    priority: int = Body(default=0, ge=0, le=5) # 单个字段 + Body() → 从请求体取
):
    return {
        "user_id": user_id,
        "user": user.model_dump(),
        "admin_note": admin_note,
        "priority": priority
    }
```

**第二个接口的请求体长这样：**
```json
{
  "user": {"username": "张三", "email": "zhangsan@qq.com", "age": 25},
  "admin_note": "VIP 用户",
  "priority": 3
}
```

> **一句话总结：`Field()` 管"这个字段长什么样"，`Body()` 管"这个参数从哪来"。** 大多数时候你只需要 `Field()`，只有在函数参数层面需要解决歧义时才用 `Body()`。

---

## 三、完整示例：三种参数的综合运用

```python
from fastapi import FastAPI, Query, Path, HTTPException
from pydantic import BaseModel, Field
from typing import Optional

app = FastAPI()

# ========== 模拟数据 ==========
fake_items = {
    1: {"id": 1, "name": "键盘", "price": 299.0, "tags": ["电子", "外设"]},
    2: {"id": 2, "name": "鼠标", "price": 149.0, "tags": ["电子", "外设"]},
    3: {"id": 3, "name": "显示器", "price": 1999.0, "tags": ["电子"]},
}

# ========== 请求体模型 ==========
class ItemCreate(BaseModel):
    name: str = Field(..., min_length=1, max_length=50, description="商品名称")
    price: float = Field(..., gt=0, description="商品价格，必须大于 0")
    tags: list[str] = Field(default=[], description="商品标签")

# ========== 接口 ==========

# 查询商品列表（查询参数）
@app.get("/items")
async def list_items(
    keyword: Optional[str] = Query(None, min_length=1, description="搜索关键词"),
    min_price: float = Query(0, ge=0, description="最低价格"),
    max_price: Optional[float] = Query(None, gt=0, description="最高价格"),
    skip: int = Query(0, ge=0, description="跳过多少条"),
    limit: int = Query(10, ge=1, le=100, description="返回多少条"),
):
    """用查询参数实现搜索和分页"""
    results = list(fake_items.values())

    # 按关键词过滤
    if keyword:
        results = [item for item in results if keyword in item["name"]]

    # 按价格过滤
    results = [item for item in results if item["price"] >= min_price]
    if max_price is not None:
        results = [item for item in results if item["price"] <= max_price]

    # 分页
    total = len(results)
    results = results[skip : skip + limit]

    return {"total": total, "items": results}

# 查询单个商品（路径参数）
@app.get("/items/{item_id}")
async def get_item(
    item_id: int = Path(..., gt=0, description="商品 ID")
):
    """用路径参数指定要查哪个商品"""
    if item_id not in fake_items:
        raise HTTPException(status_code=404, detail=f"商品 {item_id} 不存在")
    return fake_items[item_id]

# 创建商品（请求体）
@app.post("/items", status_code=201)
async def create_item(item: ItemCreate):
    """用请求体接收商品数据"""
    new_id = max(fake_items.keys()) + 1
    new_item = {"id": new_id, **item.model_dump()}
    fake_items[new_id] = new_item
    return new_item

# 综合运用：为指定用户创建订单（路径 + 查询 + 请求体）
@app.post("/users/{user_id}/orders")
async def create_order(
    user_id: int = Path(..., gt=0, description="用户 ID"),
    item: ItemCreate = ...,                              # 请求体
    quantity: int = Query(1, ge=1, le=99, description="购买数量"),
    note: Optional[str] = Query(None, description="订单备注"),
):
    """
    三种参数同时使用：
    - user_id：路径参数（指定给谁创建订单）
    - item：请求体（买了什么）
    - quantity、note：查询参数（额外选项）
    """
    total_price = item.price * quantity
    return {
        "user_id": user_id,
        "item": item.name,
        "quantity": quantity,
        "total_price": total_price,
        "note": note,
        "msg": f"用户 {user_id} 购买了 {quantity} 个 {item.name}，总价 {total_price} 元"
    }
```

**测试这个综合接口：**
```text
POST /users/42/orders?quantity=3&note=尽快发货
Body: {"name": "键盘", "price": 299.0, "tags": ["电子"]}
```

**返回：**
```json
{
  "user_id": 42,
  "item": "键盘",
  "quantity": 3,
  "total_price": 897.0,
  "note": "尽快发货",
  "msg": "用户 42 购买了 3 个键盘，总价 897 元"
}
```

---

## 速查表

| 概念 | 一句话解释 | 关键代码 |
|------|---------|---------|
| 请求体 | POST/PUT 发送的 JSON 数据 | `def f(user: UserCreate)` |
| `BaseModel` | 定义请求体结构 | `class User(BaseModel): ...` |
| `Field()` | 给请求体字段加约束（长度、范围） | `name: str = Field(..., min_length=3)` |
| `Query()` | 给查询参数加约束 | `q: str = Query(..., min_length=3)` |
| `Path()` | 给路径参数加约束 | `user_id: int = Path(..., gt=0)` |
| `Body()` | 显式声明请求体字段 | `info: str = Body(...)` |

---

## 
> ▶ 对应原理：[[03-Pydantic数据校验与v2新特性|03-Pydantic数据校验与v2新特性]]

相关链接

- 目录：[[00-FastAPI]]
- 上一篇：[[02-路径参数与查询参数]]
- 下一篇：[[04-响应模型与状态码]]
- 项目实践：[[笔记/技术学习/项目开发笔记/ai-resume笔记/01_技术研读/01_架构概览|ai-resume: 架构概览]]
- 项目实践：[[笔记/技术学习/项目开发笔记/cr-agent笔记/01-技术研读/05-API与CLI接口契约|cr-agent: API与CLI接口契约]]

---
→ [[技术学习清单#基础篇]]

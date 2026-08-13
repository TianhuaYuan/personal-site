---

title: "SQLAlchemy与ORM实战"

created: "2025-07-12"

tags:

  - 技术学习

  - mysql

  - sqlalchemy

  - orm

  - fastapi

---

# SQLAlchemy与ORM实战

> 前四天我们都在 MySQL 命令行里手写 SQL——建表、插数据、JOIN、子查询，纯手工操作。但真实项目里，没人直接在命令行写 SQL 操作数据库。Python 程序通过 ORM"翻译"代码为 SQL，让你用 Python 对象操作数据库行。今天就是把前四天的 SQL 功力，接上 Python 的地气。

---

## 🔁 前置复习

| 前几天学的 | 今天怎么用 |
|-----------|-----------|
| `CREATE TABLE` + 数据类型 / 约束 | SQLAlchemy 用 Python 类定义表结构，自动生成 `CREATE TABLE` |
| `INSERT INTO ... VALUES` | `session.add(对象)` → ORM 帮你生成 INSERT |
| `SELECT ... WHERE` | `session.query().filter()` → ORM 帮你生成 SELECT + WHERE |
| `UPDATE ... SET ... WHERE` | 先查出来改属性，`session.commit()` → ORM 帮你生成 UPDATE |
| `DELETE FROM ... WHERE` | `session.delete(对象)` → ORM 帮你生成 DELETE |
| `JOIN / GROUP BY` | `session.query().join()` → ORM 也能做关联查询 |

---

## 一、为什么要用 ORM？
### 1.1 原始 SQL 的痛点

```python

import pymysql  # pymysql 是 Python 连接 MySQL 的库

# 第一步：建立连接

conn = pymysql.connect(

    host="localhost",

    user="root",

    password="123456",

    database="my_shop"

)

# 第二步：拿到"光标"

cursor = conn.cursor()

# 第三步：执行 SQL 查询

cursor.execute(

    "SELECT id, username, email FROM users WHERE id = %s",

    (1,)

)

# 第四步：取结果

row = cursor.fetchone()

# 你得记住第 0 列是 id，第 1 列是 username...

print(row[1])  # zhangsan — 这个 1 是谁？全靠记忆

conn.close()    # 第五步：用完必须关连接

```

痛点：

- **返回值是元组**——`row[0]`、`row[1]` 完全靠记忆，加一列全崩

- **SQL 拼接容易出错**——少一个引号、少一个逗号，运行时才炸

- **不同数据库方言不同**——MySQL 用 `%s`，SQLite 用 `?`，PostgreSQL 又不一样

- **连接管理繁琐**——每次都要 `connect` / `close`，忘了关就泄漏

### 1.2 ORM = 翻译官

> **ORM（Object-Relational Mapping，对象-关系映射）**：你说 Python（对象、属性），它帮你翻译成 SQL 交给数据库；数据库返回表格，它帮你翻译成 Python 对象还给你。你全程不碰 SQL，但底层跑的就是 SQL。

```python

# 用 ORM 查同一个用户

user = session.query(User).filter(User.id == 1).first()

print(user.username)  # zhangsan — 属性名即列名，不会搞混

```

优势：

- **`user.username`** 代替 `row[1]`——属性名即列名

- **不用手写 SQL**——ORM 帮你生成，语法错误几乎不可能

- **换数据库只改连接字符串**——代码不用动

- **自动管理连接**——Session 帮你管，再也不怕忘记关

### 1.3 为什么选 SQLAlchemy？

| 框架 | 特点 | 适合场景 |
|------|------|---------|
| **SQLAlchemy** | 功能最全，生态最广，FastAPI 官方推荐 | 正式项目、复杂查询 |
| Django ORM | 和 Django 深度绑定，简单好用 | Django 项目 |
| Tortoise ORM | 异步原生，轻量 | 异步项目（但不成熟） |

> **选 SQLAlchemy，因为 FastAPI 官方教程就用它，社区资源最多。**

---

## 二、环境搭建
### 2.1 安装依赖

```bash

pip install fastapi uvicorn sqlalchemy pymysql

```

| 包 | 作用 | 一句话理解 |
|----|------|-----------|
| `fastapi` | Web 框架，写 API 接口 | 你的"厨房"，处理点单和做菜 |
| `uvicorn` | ASGI 服务器，运行 FastAPI | 你的"服务员"，把顾客请求端进厨房 |
| `sqlalchemy` | ORM，Python 对象 ↔ 数据库表 | 你的"翻译器" |
| `pymysql` | Python 连 MySQL 的驱动 | 你的"通往仓库的路" |

### 2.2 项目结构

```mermaid

graph TB

    subgraph my_crud_app

        main["main.py<br/>FastAPI 入口 + 路由<br/>🍳 厨师：接单做菜"]

        db["database.py<br/>数据库连接 + Session<br/>🔑 仓库钥匙：怎么拿到食材"]

        models["models.py<br/>ORM 模型<br/>📋 食材清单：有什么原料"]

        schemas["schemas.py<br/>Pydantic 模型<br/>📝 菜单：顾客能看到什么菜"]

        req["requirements.txt<br/>依赖列表"]

    end

```

> **为什么分文件？** `models.py` 是"食材清单"，`schemas.py` 是"菜单"，`database.py` 是"仓库钥匙"，`main.py` 是"厨师"。各司其职，改菜单不用动食材清单。

---

## 三、数据库连接与引擎
### 3.1 连接字符串

连接字符串就是告诉 SQLAlchemy "去哪里找数据库"的地址：

```text

mysql+pymysql://用户名:密码@主机地址:端口/数据库名

```

逐段拆解：

- `mysql` —— 数据库类型（MySQL）

- `pymysql` —— 用哪个驱动连

- `://` —— 固定分隔符

- `用户名:密码` —— 登录凭证

- `@主机地址:端口` —— 数据库在哪台机器的哪个端口

- `/数据库名` —— 连哪个库

```python

# database.py

from sqlalchemy import create_engine

SQLALCHEMY_DATABASE_URL = "mysql+pymysql://root:123456@localhost:3306/my_shop_demo"

engine = create_engine(SQLALCHEMY_DATABASE_URL)

```

> [!CAUTION] 密码里有特殊字符（如 `@`、`#`）会出错！

> 密码 `my@pass#123` 中的 `@` 会被误认为主机地址开始。解决：用 `quote_plus()` 编码。

>

> ```python

> from urllib.parse import quote_plus

> password = quote_plus("my@pass#123")  # → my%40pass%23123

> URL = f"mysql+pymysql://root:{password}@localhost:3306/my_shop_demo"

> ```

### 3.2 引擎是什么？

> **引擎 = 数据库的"前台总机"。** 你不直接跟数据库通话，所有请求都经过引擎——它负责建立连接、管理连接池、翻译 SQL 方言。

```mermaid

graph LR

    A["Python 代码"] --> B["引擎(engine)"]

    B --> C["连接池(pool)"]

    C --> D["MySQL 数据库"]

```

`create_engine` 默认行为：

- **懒连接**——创建引擎时不会立刻连数据库，第一次查询时才连

- **连接池**——默认维护 5 个连接，用完归还而不是销毁，下次复用

- **自动重连**——连接断开后会自动重建

### 3.3 连接池参数

```python

engine = create_engine(

    URL,

    pool_size=5,       # 连接池常驻连接数（默认 5）

    max_overflow=10,   # 高峰期额外允许的连接数（默认 10）

    pool_recycle=3600  # 连接最长存活时间（秒），防 MySQL 8 小时断连

)

```

> **为什么 `pool_recycle=3600`？** MySQL 默认 8 小时（28800 秒）没活动的连接会自动断开。设 `pool_recycle=3600` 让连接每小时换新，永远不会有"过期连接"。

---

## 四、定义 ORM 模型
### 4.1 核心概念：Python 类 = 数据库表

> **ORM 模型就是数据库表的"设计图纸"。** 你在 Python 里画好图纸（定义类），SQLAlchemy 拿着图纸去建表。类的每个属性 = 表的一列，类的一个实例 = 表的一行。

```mermaid

graph LR

    subgraph python["Python 类（图纸）"]

        class_def["class User:"]

        f1["  id : int"]

        f2["  username : str"]

        f3["  email : str"]

    end

    subgraph sql["数据库表（实体）"]

        table["CREATE TABLE users ("]

        c1["  id INT PRIMARY KEY,"]

        c2["  username VARCHAR(50),"]

        c3["  email VARCHAR(100)"]

        c4[")"]

    end

    python -->|"ORM 映射"| sql

```

### 4.2 基本写法——逐行拆解

```python

# models.py

from sqlalchemy import Column, Integer, String, Float, DateTime, func

from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):

    pass  # 空类——所有模型的"族谱源头"

class User(Base):

    __tablename__ = "users"                # 🔴 必须写！对应数据库表名

    id       = Column(Integer, primary_key=True, autoincrement=True)

    #           ↑ 定义一列    ↑ 整数类型   ↑ 主键              ↑ 自增

    # 相当于 MySQL：id INT PRIMARY KEY AUTO_INCREMENT

    username = Column(String(50), nullable=False, comment="用户名")

    #                   ↑ 字符串，最长50   ↑ 不允许为空      ↑ 列注释

    # 相当于 MySQL：username VARCHAR(50) NOT NULL COMMENT '用户名'

    email    = Column(String(100), unique=True, comment="邮箱")

    #                      ↑ 唯一约束（不能重复）

    # 相当于 MySQL：email VARCHAR(100) UNIQUE COMMENT '邮箱'

    def __repr__(self):

        # __repr__ = "representation"，定义打印对象时显示什么

        return f"<User(id={self.id}, username='{self.username}')>"

```

> [!TIP]

> `__repr__` 是调试利器。不写的话 `print(user)` 会显示 `<User object at 0x7f...>`，写了之后显示 `<User(id=1, username='zhangsan')>`，一目了然。

### 4.3 字段映射对照表

| MySQL 写法 | SQLAlchemy 写法 | 说明 |
|-----------|----------------|------|
| `INT PRIMARY KEY AUTO_INCREMENT` | `Column(Integer, primary_key=True, autoincrement=True)` | 主键自增 |
| `VARCHAR(50) NOT NULL` | `Column(String(50), nullable=False)` | 非空字符串 |
| `VARCHAR(100) UNIQUE` | `Column(String(100), unique=True)` | 唯一约束 |
| `FLOAT DEFAULT 0` | `Column(Float, default=0)` | 默认值 |
| `DATETIME DEFAULT NOW()` | `Column(DateTime, server_default=func.now())` | 数据库层默认值 |
| `COMMENT '备注'` | `Column(..., comment="备注")` | 列注释 |

> [!WARNING] `default` vs `server_default`——新手必踩的坑

> - `default=0` → Python 层填默认值（ORM 生成 INSERT 语句时自动带上值）

> - `server_default="0"` → 数据库层填默认值（DDL 里的 `DEFAULT 0`）

>

> 大多数时候用 `default` 就行。但像 `NOW()` 这种"必须是数据库执行时的时间"的情况，必须用 `server_default=func.now()`。

### 4.4 完整模型示例

```python

# models.py

from sqlalchemy import Column, Integer, String, Float, DateTime, ForeignKey, func

from sqlalchemy.orm import DeclarativeBase, relationship

class Base(DeclarativeBase):

    pass

class User(Base):

    __tablename__ = "users"

    id       = Column(Integer, primary_key=True, autoincrement=True)

    username = Column(String(50), nullable=False)

    email    = Column(String(100), unique=True)

    # relationship：Python 层的"关联"，不是数据库列！

    orders = relationship("Order", back_populates="user")

    # 通过 user.orders 可以拿到这个用户的所有订单

class Product(Base):

    __tablename__ = "products"

    id       = Column(Integer, primary_key=True, autoincrement=True)

    name     = Column(String(100), nullable=False)

    price    = Column(Float, nullable=False)

    category = Column(String(50))

class Order(Base):

    __tablename__ = "orders"

    id         = Column(Integer, primary_key=True, autoincrement=True)

    user_id    = Column(Integer, ForeignKey("users.id"), nullable=False)

    #           外键！相当于 MySQL：FOREIGN KEY (user_id) REFERENCES users(id)

    product    = Column(String(100), nullable=False)

    product_id = Column(Integer, ForeignKey("products.id"))

    quantity   = Column(Integer, default=1)

    total      = Column(Float, nullable=False)

    order_date = Column(DateTime, server_default=func.now())

    user        = relationship("User", back_populates="orders")

    product_rel = relationship("Product")

```

### 4.5 `ForeignKey` 和 `relationship` 的区别

| | `ForeignKey` | `relationship` |
|---|---|---|
| **是什么** | 数据库层的约束 | Python 层的便捷属性 |
| **存不存在于数据库** | ✅ 生成 `FOREIGN KEY` 约束 | ❌ 数据库里没有这一列 |
| **作用** | 保证数据引用合法 | 让你用 `order.user` 直接拿到用户对象 |

```python

# relationship 做的事：

order = session.query(Order).first()

print(order.user.username)  # 不用 JOIN，直接 .user 就能访问关联用户

```

### 4.6 自动建表

```python

# 在 database.py 或 main.py 中执行：

Base.metadata.create_all(bind=engine)

# 这一行 = 把所有继承 Base 的类翻译成 CREATE TABLE IF NOT EXISTS ... 并执行

```

> [!CAUTION]

> `create_all` 只建新表，不修改已有表！表已存在时加了新列不会自动加。要改已有表的结构，需要用 Alembic 做数据库迁移。

---

## 五、Session：与数据库对话的"窗口"
### 5.1 为什么需要 Session？

> **引擎(engine)是公司的前台总机，Session 是你拿到的"通话话筒"。** 拿起话筒（开 Session）→ 说话（操作数据）→ 挂电话（关 Session）。没挂电话之前，你说的所有话都在"草稿"里——必须按"发送"（commit）才算数。

### 5.2 创建 Session——逐行拆解

```python

# database.py

from sqlalchemy.orm import sessionmaker, Session

from typing import Generator

SessionLocal = sessionmaker(

    autocommit=False,   # 不自动提交——必须手动 db.commit() 才真正写入

    autoflush=False,    # 不自动刷新——不自动把内存改动推到数据库

    bind=engine         # 绑定到哪个引擎

)

# FastAPI 依赖注入：获取 Session 的标准写法

def get_db() -> Generator[Session, None, None]:

    db = SessionLocal()

    try:

        yield db         # 把 db 交给路由函数使用

    finally:

        db.close()       # 无论出不出错，都关掉 Session

```

| 参数 | 含义 | 一句话理解 |
|------|------|-----------|
| `autocommit=False` | 不自动提交，必须手动 `db.commit()` | 别替我做主，我自己按发送 |
| `autoflush=False` | 不自动把内存中的改动刷到数据库 | 别替我发半成品 |
| `bind=engine` | 绑定到哪个引擎 | 告诉话筒连哪个总机 |

### 5.3 Session 的核心操作一览

| 操作 | SQL 等价 | ORM 写法 |
|------|---------|---------|
| 插入一行 | `INSERT INTO ...` | `db.add(对象)` + `db.commit()` |
| 查所有行 | `SELECT * FROM ...` | `db.query(Model).all()` |
| 按条件查 | `SELECT * WHERE ...` | `db.query(Model).filter(条件).all()` |
| 查一行 | `SELECT * WHERE id=X LIMIT 1` | `db.query(Model).filter(Model.id == X).first()` |
| 改一行 | `UPDATE ... SET ... WHERE ...` | 改对象属性 + `db.commit()` |
| 删一行 | `DELETE FROM ... WHERE ...` | `db.delete(对象)` + `db.commit()` |

### 5.4 增删改查速览（纯 ORM）

```python

from database import SessionLocal

from models import User

db = SessionLocal()

# ===== 增（INSERT）=====

new_user = User(username="wangwu", email="wang@qq.com")

db.add(new_user)

db.commit()            # 正式写入数据库

db.refresh(new_user)   # 刷新，拿到自动生成的 id

print(new_user.id)     # 4

# ===== 查（SELECT）=====

all_users = db.query(User).all()

# 相当于：SELECT * FROM users

one_user = db.query(User).filter(User.id == 1).first()

# 相当于：SELECT * FROM users WHERE id = 1 LIMIT 1

zhang = db.query(User).filter(User.username == "zhangsan").first()

# ===== 改（UPDATE）=====

one_user.email = "new_zhang@qq.com"

db.commit()  # UPDATE users SET email='new_zhang@qq.com' WHERE id=1

# ===== 删（DELETE）=====

db.delete(one_user)

db.commit()  # DELETE FROM users WHERE id=1

db.close()

```

> [!CAUTION] 忘记 `commit()` 是新手第一号报错原因！

> ```

> 常见新手代码：

>     db.add(new_user)    # 加了

>     print("添加成功！")  # 打印了——但数据库里根本没有！

>     # 💀 忘了 db.commit，数据丢了都不知道

> ```

---

## 六、完整 CRUD 接口（FastAPI）
### 6.1 Pydantic 模型：API 的"菜单"

> **Pydantic 模型 = 餐厅的"菜单 + 外卖包装"。** 客人（前端）点菜，必须照着菜单来（UserCreate，校验输入）。菜做好了，用统一的外卖盒装好端出去（UserResponse，格式化输出）。

```python

# schemas.py

from pydantic import BaseModel

from typing import Optional

# 创建用户时需要的字段——"入职登记表"

class UserCreate(BaseModel):

    username: str    # 必填

    email: str       # 必填

# 更新用户时，所有字段都可选——"修改信息表"

class UserUpdate(BaseModel):

    username: Optional[str] = None  # 选填

    email: Optional[str] = None     # 选填

# 返回给前端时的字段——"员工工牌"

class UserResponse(BaseModel):

    id: int

    username: str

    email: str

    class Config:

        from_attributes = True  # 🔴 关键配置！允许从 ORM 对象读取属性

```

为什么分三个模型？

| 模型 | 用途 | 为什么不能合并？ |
|------|------|---------------|
| `UserCreate` | 创建时校验——必填字段必须有 | 创建时需要 username，但创建时没有 id |
| `UserUpdate` | 更新时校验——传什么改什么 | 更新时所有字段选填，创建时必填 |
| `UserResponse` | 返回时格式——该暴露的暴露 | 返回时多了 id，且不包含密码等敏感信息 |

### 6.2 完整 CRUD 接口——逐行注释版

```python

# main.py

from fastapi import FastAPI, Depends, HTTPException

from sqlalchemy.orm import Session

from typing import List

from database import engine, get_db

from models import Base, User

from schemas import UserCreate, UserUpdate, UserResponse

# 启动时建表

Base.metadata.create_all(bind=engine)

app = FastAPI(title="用户管理 API", version="1.0")

# ========== 查所有用户 ==========

@app.get("/users", response_model=List[UserResponse])

def get_users(db: Session = Depends(get_db)):

    """相当于: SELECT * FROM users"""

    users = db.query(User).all()

    return users

# ========== 查单个用户 ==========

@app.get("/users/{user_id}", response_model=UserResponse)

def get_user(user_id: int, db: Session = Depends(get_db)):

    """相当于: SELECT * FROM users WHERE id = {user_id}"""

    user = db.query(User).filter(User.id == user_id).first()

    if not user:

        raise HTTPException(status_code=404, detail="用户不存在")

    return user

# ========== 创建用户 ==========

@app.post("/users", response_model=UserResponse, status_code=201)

def create_user(user_data: UserCreate, db: Session = Depends(get_db)):

    """相当于: INSERT INTO users (username, email) VALUES (...)"""

    # 先检查邮箱是否已存在

    existing = db.query(User).filter(User.email == user_data.email).first()

    if existing:

        raise HTTPException(status_code=400, detail="邮箱已被注册")

    new_user = User(username=user_data.username, email=user_data.email)

    db.add(new_user)

    db.commit()

    db.refresh(new_user)

    return new_user

# ========== 更新用户 ==========

@app.put("/users/{user_id}", response_model=UserResponse)

def update_user(user_id: int, user_data: UserUpdate, db: Session = Depends(get_db)):

    """相当于: UPDATE users SET ... WHERE id = {user_id}"""

    user = db.query(User).filter(User.id == user_id).first()

    if not user:

        raise HTTPException(status_code=404, detail="用户不存在")

    # 只更新传了值的字段

    if user_data.username is not None:

        user.username = user_data.username

    if user_data.email is not None:

        user.email = user_data.email

    db.commit()

    db.refresh(user)

    return user

# ========== 删除用户 ==========

@app.delete("/users/{user_id}")

def delete_user(user_id: int, db: Session = Depends(get_db)):

    """相当于: DELETE FROM users WHERE id = {user_id}"""

    user = db.query(User).filter(User.id == user_id).first()

    if not user:

        raise HTTPException(status_code=404, detail="用户不存在")

    db.delete(user)

    db.commit()

    return {"message": f"用户 {user.username} 已删除"}

```

### 6.3 `Depends(get_db)` 图解

```mermaid

sequenceDiagram

    participant Client as 请求

    participant FastAPI as FastAPI

    participant get_db

    participant DB as 数据库

    Client->>FastAPI: 请求进来

    FastAPI->>get_db: Depends(get_db)

    get_db->>get_db: db = SessionLocal()

    get_db->>FastAPI: yield db

    FastAPI->>DB: 路由函数执行

    DB-->>FastAPI: 返回结果

    get_db->>get_db: finally: db.close()

```

> `Depends(get_db)` 就像餐厅的"自动传菜窗口"。每个服务员（请求）进来，窗口自动递一个餐盘（Session），用完自动收回清洗。

### 6.4 运行项目

```bash

uvicorn main:app --reload

# --reload：文件改动后自动重启（开发时用，生产环境别加）

```

打开浏览器访问 `http://127.0.0.1:8000/docs`，可以看到 FastAPI 自动生成的交互式文档页面。

### 6.5 用 curl 测试接口

```bash

# 创建用户

curl -X POST http://127.0.0.1:8000/users \

  -H "Content-Type: application/json" \

  -d '{"username": "zhaoliu", "email": "zhao@qq.com"}'

# 查所有用户

curl http://127.0.0.1:8000/users

# 查单个用户

curl http://127.0.0.1:8000/users/1

# 更新用户

curl -X PUT http://127.0.0.1:8000/users/1 \

  -H "Content-Type: application/json" \

  -d '{"username": "zhangsan_new"}'

# 删除用户

curl -X DELETE http://127.0.0.1:8000/users/1

```

### 6.6 请求/响应全流程图

```mermaid

sequenceDiagram

    participant Client as 前端请求

    participant FastAPI as FastAPI

    participant DB as MySQL 数据库

    Client->>FastAPI: POST /users {username, email}

    FastAPI->>FastAPI: 1. Pydantic 校验 (UserCreate)

    FastAPI->>DB: 2. SELECT ... WHERE email=?

    DB-->>FastAPI: 查询结果

    FastAPI->>FastAPI: 3. 创建 ORM 对象 User(...)

    FastAPI->>DB: 4. INSERT INTO users ...

    DB-->>FastAPI: 插入成功

    FastAPI->>FastAPI: 5. Pydantic 格式化 (UserResponse)

    FastAPI-->>Client: 返回 {id, username, email}

```

---

## 七、常见报错与排错
### 7.1 连接失败

```text

sqlalchemy.exc.OperationalError: (pymysql.err.OperationalError) (2003, "Can't connect to MySQL server")

```

**原因：** MySQL 没启动，或地址/端口/密码写错。

**排查：** 先在命令行验证 MySQL 能连上：

```bash

mysql -u root -p -h localhost -P 3306

```

### 7.2 表不存在

```text

sqlalchemy.exc.ProgrammingError: (pymysql.err.ProgrammingError) (1146, "Table 'my_shop_demo.users' doesn't exist")

```

**原因：** 忘了执行 `Base.metadata.create_all(bind=engine)`。

**解决：** 确保 `create_all` 在所有模型定义之后执行。

### 7.3 忘记 commit

```text

# 代码没报错，但数据库里查不到新数据

```

**原因：** `db.add()` 之后没有 `db.commit()`——改动只在内存里。

**防翻车口诀：** 每次 `add` / 改属性 / `delete` 之后，养成立刻写 `commit()` 的习惯。

### 7.4 Pydantic 报错——from_attributes 漏写

```text

ValidationError: Input should be a valid dictionary

```

**原因：** `UserResponse` 的 `Config.from_attributes = True` 漏写了。

**解决：**

```python

class UserResponse(BaseModel):

    ...

    class Config:

        from_attributes = True  # 🔴 这一行必须有！

```

### 7.5 邮箱唯一约束冲突

```text

sqlalchemy.exc.IntegrityError: (pymysql.err.IntegrityError) (1062, "Duplicate entry 'zhang@qq.com' for key 'email'")

```

**原因：** 数据库 `email` 列有 `UNIQUE` 约束，插入重复值了。

**解决：** 代码里先查再插（`create_user` 中已做），或捕获 `IntegrityError`。

### 7.6 FastAPI 422 Unprocessable Entity

**原因：** 前端发的 JSON 字段名或类型和 `UserCreate` 定义的不一致。

```text

❌ 发了 {"name": "zhangsan"}         → username 字段缺失 → 422

❌ 发了 {"username": 123}             → str 类型不匹配   → 422

✅ 发了 {"username": "zhangsan", "email": "z@qq.com"}  → 通过

```

---

## 速查表
### SQLAlchemy ORM 核心写法

| 操作 | 写法 | SQL 等价 |
|------|------|---------|
| 查所有 | `db.query(Model).all()` | `SELECT * FROM table` |
| 按条件查 | `db.query(Model).filter(Model.col == val).all()` | `SELECT * WHERE col = val` |
| 查一个 | `db.query(Model).filter(...).first()` | `SELECT * WHERE ... LIMIT 1` |
| 按主键查 | `db.get(Model, id)` | `SELECT * WHERE id = X` |
| 插入 | `db.add(对象); db.commit()` | `INSERT INTO ...` |
| 更新 | `对象.属性 = 新值; db.commit()` | `UPDATE SET ... WHERE id=X` |
| 删除 | `db.delete(对象); db.commit()` | `DELETE FROM ... WHERE id=X` |
| 刷新 | `db.refresh(对象)` | 重新 SELECT 拿最新数据 |
| 回滚 | `db.rollback()` | 撤销未 commit 的改动 |

### FastAPI 路由模板

| 操作 | 装饰器 | 状态码 |
|------|--------|--------|
| 查所有 | `@app.get("/items")` | 200 |
| 查一个 | `@app.get("/items/{id}")` | 200 |
| 创建 | `@app.post("/items")` | 201 |
| 更新 | `@app.put("/items/{id}")` | 200 |
| 删除 | `@app.delete("/items/{id}")` | 200 |

### CRUD 对应 HTTP 方法

| HTTP 方法 | 语义 | 幂等？ | SQL 对应 |
|----------|------|--------|---------|
| GET | 读取 | ✅ 幂等 | SELECT |
| POST | 创建 | ❌ 非幂等 | INSERT |
| PUT | 全量更新 | ✅ 幂等 | UPDATE |
| DELETE | 删除 | ✅ 幂等 | DELETE |

---

## 八、综合实战：完整项目代码
### 写代码前：思考顺序

```mermaid

graph LR

    A["🏗️ 地基<br/>database.py"] --> B["🏢 楼体结构<br/>models.py"]

    B --> C["🎨 装修标准<br/>schemas.py"]

    C --> D["🚪 施工入口<br/>main.py"]

```

**为什么这个顺序？** 每一步都依赖前一步的结果：

- 没有 `database.py`，`models.py` 不知道连哪个库

- 没有 `models.py`，`schemas.py` 没法参考 ORM 字段

- 没有前三者，`main.py` 没东西可导入

### database.py

```python

# database.py —— 数据库连接和 Session 配置

from sqlalchemy import create_engine

from sqlalchemy.orm import sessionmaker, Session

from typing import Generator

SQLALCHEMY_DATABASE_URL = "mysql+pymysql://root:123456@localhost:3306/my_shop_demo"

engine = create_engine(

    SQLALCHEMY_DATABASE_URL,

    pool_size=5,

    pool_recycle=3600,

)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

def get_db() -> Generator[Session, None, None]:

    db = SessionLocal()

    try:

        yield db

    finally:

        db.close()

```

### models.py

```python

# models.py —— ORM 模型定义

from sqlalchemy import Column, Integer, String, Float, DateTime, ForeignKey, func

from sqlalchemy.orm import DeclarativeBase, relationship

class Base(DeclarativeBase):

    pass

class User(Base):

    __tablename__ = "users"

    id       = Column(Integer, primary_key=True, autoincrement=True)

    username = Column(String(50), nullable=False)

    email    = Column(String(100), unique=True)

    orders = relationship("Order", back_populates="user")

    def __repr__(self):

        return f"<User(id={self.id}, username='{self.username}')>"

class Product(Base):

    __tablename__ = "products"

    id       = Column(Integer, primary_key=True, autoincrement=True)

    name     = Column(String(100), nullable=False)

    price    = Column(Float, nullable=False)

    category = Column(String(50))

class Order(Base):

    __tablename__ = "orders"

    id         = Column(Integer, primary_key=True, autoincrement=True)

    user_id    = Column(Integer, ForeignKey("users.id"), nullable=False)

    product    = Column(String(100), nullable=False)

    product_id = Column(Integer, ForeignKey("products.id"))

    quantity   = Column(Integer, default=1)

    total      = Column(Float, nullable=False)

    order_date = Column(DateTime, server_default=func.now())

    user        = relationship("User", back_populates="orders")

    product_rel = relationship("Product")

```

### schemas.py

```python

# schemas.py —— Pydantic 模型定义

from pydantic import BaseModel

from typing import Optional

class UserCreate(BaseModel):

    username: str

    email: str

class UserUpdate(BaseModel):

    username: Optional[str] = None

    email: Optional[str] = None

class UserResponse(BaseModel):

    id: int

    username: str

    email: str

    class Config:

        from_attributes = True

```

### main.py

```python

# main.py —— FastAPI 应用入口 + 所有路由

from fastapi import FastAPI, Depends, HTTPException

from sqlalchemy.orm import Session

from typing import List

from database import engine, get_db

from models import Base, User

from schemas import UserCreate, UserUpdate, UserResponse

Base.metadata.create_all(bind=engine)

app = FastAPI(title="用户管理 API")

@app.get("/users", response_model=List[UserResponse])

def get_users(db: Session = Depends(get_db)):

    return db.query(User).all()

@app.get("/users/{user_id}", response_model=UserResponse)

def get_user(user_id: int, db: Session = Depends(get_db)):

    user = db.query(User).filter(User.id == user_id).first()

    if not user:

        raise HTTPException(status_code=404, detail="用户不存在")

    return user

@app.post("/users", response_model=UserResponse, status_code=201)

def create_user(user_data: UserCreate, db: Session = Depends(get_db)):

    if db.query(User).filter(User.email == user_data.email).first():

        raise HTTPException(status_code=400, detail="邮箱已被注册")

    new_user = User(username=user_data.username, email=user_data.email)

    db.add(new_user)

    db.commit()

    db.refresh(new_user)

    return new_user

@app.put("/users/{user_id}", response_model=UserResponse)

def update_user(user_id: int, user_data: UserUpdate, db: Session = Depends(get_db)):

    user = db.query(User).filter(User.id == user_id).first()

    if not user:

        raise HTTPException(status_code=404, detail="用户不存在")

    if user_data.username is not None:

        user.username = user_data.username

    if user_data.email is not None:

        user.email = user_data.email

    db.commit()

    db.refresh(user)

    return user

@app.delete("/users/{user_id}")

def delete_user(user_id: int, db: Session = Depends(get_db)):

    user = db.query(User).filter(User.id == user_id).first()

    if not user:

        raise HTTPException(status_code=404, detail="用户不存在")

    db.delete(user)

    db.commit()

    return {"message": f"用户 {user.username} 已删除"}

```

### 运行与测试

```bash

# 启动服务

uvicorn main:app --reload

# 测试完整 CRUD 流程

curl -X POST http://127.0.0.1:8000/users \

  -H "Content-Type: application/json" \

  -d '{"username":"test_user","email":"test@qq.com"}'

curl http://127.0.0.1:8000/users

curl -X PUT http://127.0.0.1:8000/users/1 \

  -H "Content-Type: application/json" \

  -d '{"username":"updated_user"}'

curl -X DELETE http://127.0.0.1:8000/users/1

```

---

## 速记卡（面试闪卡）

**Q1：一句话讲清「SQLAlchemy与ORM实战」到底是什么？**

A：SQLAlchemy 是 Python 的 ORM 框架，用类定义表、用对象操作数据，自动翻译 SQL 并管理连接。

**Q2：一、为什么要用 ORM？ —— 怎么理解？**

A：像"请翻译官"：原始 SQL（pymysql）返回元组 row[1] 全靠记、SQL 拼接易错、方言不同（%s/?）、连接忘关就漏。ORM 让你写 user.username，属性名即列名，换库只改连接串，Session 自动管连接——全程不碰 SQL 但底层跑 SQL。

**Q3：二、环境搭建 —— 怎么理解？**

A：像"配厨房四件套"：pip 装 fastapi/uvicorn/sqlalchemy/pymysql；分文件——models.py 是食材清单（表结构）、schemas.py 是菜单（Pydantic 出入参）、database.py 是仓库钥匙（连接+Session）、main.py 是厨师（路由）。各司其职，改菜单不动清单。

**Q4：三、数据库连接与引擎 —— 怎么理解？**

A：引擎（Engine）像"前台总机"：create_engine 默认懒连接、维护连接池（默认 5）、自动重连；连接串 mysql+pymysql://用户:密码@主机:端口/库。pool_recycle=3600 防 MySQL 8 小时断连；密码含 @ 用 quote_plus 编码。

**Q5：四、定义 ORM 模型 —— 怎么理解？**

A：模型是"表的设计图纸"：类属性=列（Column(Integer, primary_key) 等），实例=一行。ForeignKey 是数据库约束（保证引用合法），relationship 是 Python 层便捷属性（order.user 直接拿用户）。create_all 只建新表不修改已有表，改结构要用 Alembic 迁移。

**Q6：核心速记主线有哪些？**

- ORM 当翻译官：Python 对象 ↔ SQL，属性名即列名

- Engine 管连接池，Session 是工作单元（commit 才生效）

- Model 管表结构，Pydantic Schema 管接口出入参

- ForeignKey 是库约束，relationship 是 Python 便捷属性

**口诀**

A：ORM 翻译官，Python 对象变 SQL；

Engine 管连接，Session 管事务。

Model 是表，Schema 是菜单，

各司其职不串味。

## 相关链接

- 目录：[[00-MySQL]]

- 上一篇：[[05-聚合函数与分组GROUP_BY]]

- 下一篇：[[00-MySQL]]

---

→ [[技术学习路线图#五、MySQL 实战]]


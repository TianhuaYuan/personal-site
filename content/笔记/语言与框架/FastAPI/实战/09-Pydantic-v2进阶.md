---
title: "Pydantic v2 进阶：field_validator / EmailStr / model_config"
tags:
  - python
  - 技术学习
  - 学习笔记
created: "2026-07-21"
---

# Pydantic v2 进阶：field_validator / EmailStr / model_config

> **一句话**：Pydantic v2 是 Pydantic 的第二个主要版本，提供了更好的性能（Rust 核心）、更强大的验证功能、更灵活的配置。在 FastAPI 中，Pydantic 用于数据验证、序列化和文档生成。

## 1. Pydantic v2 新特性

### 1.1 性能提升

```mermaid
graph LR
    A[Pydantic v1] -->|Python核心| B[性能一般]
    C[Pydantic v2] -->|Rust核心| D[性能提升10-50倍]
    
    style C fill:#e8f5e8
    style D fill:#e1f5fe
```

- **Rust 核心**：验证逻辑用 Rust 实现，比纯 Python 快 10-50 倍
- **更好的错误信息**：更精确的错误定位
- **JSON Schema 支持**：更标准的 JSON Schema 生成

### 1.2 安装

```bash
pip install pydantic>=2.0
# 或
pip install "pydantic[email]"  # 包含 EmailStr 支持
```

## 2. field_validator

### 2.1 基础用法

```python
from pydantic import BaseModel, field_validator

class User(BaseModel):
    name: str
    age: int
    email: str

    @field_validator('name')
    @classmethod
    def validate_name(cls, v):
        if len(v) < 2:
            raise ValueError('姓名至少2个字符')
        return v.title()  # 转换为首字母大写

    @field_validator('age')
    @classmethod
    def validate_age(cls, v):
        if v < 0 or v > 150:
            raise ValueError('年龄必须在0-150之间')
        return v

# 使用
user = User(name="alice", age=25, email="alice@example.com")
print(user.name)  # Alice（自动转换）
```

### 2.2 多字段验证

```python
from pydantic import BaseModel, field_validator, model_validator

class PasswordChange(BaseModel):
    old_password: str
    new_password: str
    confirm_password: str

    @field_validator('new_password')
    @classmethod
    def validate_new_password(cls, v):
        if len(v) < 8:
            raise ValueError('新密码至少8位')
        if not any(c.isupper() for c in v):
            raise ValueError('新密码必须包含大写字母')
        return v

    @model_validator(mode='after')
    def validate_passwords_match(self):
        if self.new_password != self.confirm_password:
            raise ValueError('两次密码不一致')
        return self
```

## 3. EmailStr

### 3.1 基础用法

```python
from pydantic import BaseModel, EmailStr

class User(BaseModel):
    name: str
    email: EmailStr  # 自动验证邮箱格式

# 使用
user = User(name="Alice", email="alice@example.com")  # ✅
# user = User(name="Alice", email="invalid")  # ❌ ValidationError
```

### 3.2 自定义邮箱验证

```python
from pydantic import BaseModel, EmailStr, field_validator

class User(BaseModel):
    name: str
    email: EmailStr

    @field_validator('email')
    @classmethod
    def validate_email(cls, v):
        # 只允许特定域名
        allowed_domains = ['example.com', 'company.org']
        domain = v.split('@')[1]
        if domain not in allowed_domains:
            raise ValueError(f'只允许{allowed_domains}域名')
        return v.lower()  # 转换为小写
```

## 4. model_config

### 4.1 基础配置

```python
from pydantic import BaseModel, ConfigDict

class User(BaseModel):
    model_config = ConfigDict(
        str_strip_whitespace=True,  # 自动去除字符串前后空格
        str_lower=True,             # 字符串自动转小写
        validate_default=True,      # 验证默认值
        extra='forbid',             # 禁止额外字段
        frozen=True,                # 实例不可变
    )
    
    name: str
    age: int = 0
    email: str

# 使用
user = User(name="  Alice  ", age=25, email="ALICE@EXAMPLE.COM")
print(user.name)   # alice（自动去空格+转小写）
print(user.email)  # alice@example.com
```

### 4.2 从环境变量加载

```python
from pydantic import BaseModel, ConfigDict
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    model_config = ConfigDict(
        env_file='.env',
        env_prefix='APP_',  # 环境变量前缀
        env_file_encoding='utf-8',
    )
    
    database_url: str
    api_key: str
    debug: bool = False

# .env 文件：
# APP_DATABASE_URL=postgresql://user:pass@localhost/db
# APP_API_KEY=secret123
# APP_DEBUG=true

settings = Settings()
print(settings.database_url)
```

## 5. SettingsConfigDict（FastAPI配置）

### 5.1 基础用法

```python
from fastapi import FastAPI
from pydantic_settings import BaseSettings
from pydantic import ConfigDict

class Settings(BaseSettings):
    model_config = ConfigDict(
        env_file='.env',
        env_prefix='FASTAPI_',
    )
    
    app_name: str = "My App"
    debug: bool = False
    database_url: str
    secret_key: str

app = FastAPI()
settings = Settings()

@app.get("/")
async def root():
    return {"app_name": settings.app_name}
```

### 5.2 嵌套配置

```python
from pydantic import BaseModel
from pydantic_settings import BaseSettings

class DatabaseConfig(BaseModel):
    url: str
    pool_size: int = 10
    max_overflow: int = 20

class RedisConfig(BaseModel):
    url: str
    max_connections: int = 50

class Settings(BaseSettings):
    model_config = ConfigDict(
        env_file='.env',
        env_nested_delimiter='__',  # 嵌套分隔符
    )
    
    app_name: str = "My App"
    database: DatabaseConfig
    redis: RedisConfig

# .env 文件：
# APP_NAME=My App
# DATABASE__URL=postgresql://user:pass@localhost/db
# DATABASE__POOL_SIZE=10
# REDIS__URL=redis://localhost:6379
# REDIS__MAX_CONNECTIONS=50

settings = Settings()
print(settings.database.url)
```

## 6. 与 FastAPI 集成

### 6.1 请求体验证

```python
from fastapi import FastAPI
from pydantic import BaseModel, field_validator, EmailStr

app = FastAPI()

class UserCreate(BaseModel):
    name: str
    email: EmailStr
    password: str

    @field_validator('name')
    @classmethod
    def validate_name(cls, v):
        if len(v) < 2:
            raise ValueError('姓名至少2个字符')
        return v

    @field_validator('password')
    @classmethod
    def validate_password(cls, v):
        if len(v) < 8:
            raise ValueError('密码至少8位')
        return v

@app.post("/users/")
async def create_user(user: UserCreate):
    return {"user": user.model_dump()}
```

### 6.2 响应模型

```python
from fastapi import FastAPI
from pydantic import BaseModel, EmailStr

app = FastAPI()

class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)  # ORM模式
    
    id: int
    name: str
    email: EmailStr
    is_active: bool

@app.get("/users/{user_id}", response_model=UserResponse)
async def get_user(user_id: int):
    # 从数据库获取用户
    user = get_user_from_db(user_id)
    return user  # 自动转换为UserResponse
```

## 7. 高级特性

### 7.1 自定义类型

```python
from pydantic import GetCoreSchemaHandler, GetJsonSchemaHandler
from pydantic.json_schema import JsonSchemaValue
from pydantic_core import core_schema

class PositiveInt:
    """正整数类型"""
    
    @classmethod
    def __get_pydantic_core_schema__(
        cls, source_type, handler
    ) -> core_schema.CoreSchema:
        return core_schema.no_info_plain_validator_function(
            cls.validate,
            serialization=core_schema.plain_serializer_function_ser_schema(
                lambda v: v
            ),
        )
    
    @classmethod
    def __get_pydantic_json_schema__(
        cls, schema, handler
    ) -> JsonSchemaValue:
        return {"type": "integer", "minimum": 1}
    
    @classmethod
    def validate(cls, v):
        if not isinstance(v, int):
            raise TypeError('必须是整数')
        if v <= 0:
            raise ValueError('必须是正整数')
        return v

class Product(BaseModel):
    name: str
    price: PositiveInt  # 使用自定义类型
```

### 7.2 模型继承

```python
from pydantic import BaseModel

class BaseUser(BaseModel):
    name: str
    email: str

class UserCreate(BaseUser):
    password: str

class UserResponse(BaseUser):
    id: int
    is_active: bool

# UserCreate 包含 name, email, password
# UserResponse 包含 name, email, id, is_active
```

## 8. 常见坑点

### 1. 字段顺序
```python
class Bad(BaseModel):
    b: int
    a: str = "default"

# 字段顺序：b → a（必须按此顺序实例化）
# Bad(1)  # ❌ TypeError
Bad(b=1)  # ✅

# 解决：使用默认值或Optional
class Good(BaseModel):
    a: str = "default"
    b: int
```

### 2. 可变默认值
```python
class Bad(BaseModel):
    items: list = []  # ❌ 所有实例共享同一个list

class Good(BaseModel):
    items: list = []  # ✅ Pydantic会自动处理
```

### 3. 性能考虑
```python
# Pydantic v2性能很好，但频繁创建模型有开销
# 建议：复用模型定义，不要在循环中动态创建
```

## 核心要点

```python
from pydantic import BaseModel, field_validator, EmailStr
from pydantic import ConfigDict

# 基础模型
class User(BaseModel):
    name: str
    email: EmailStr

# 字段验证
@field_validator('name')
@classmethod
def validate_name(cls, v):
    if len(v) < 2:
        raise ValueError('姓名至少2个字符')
    return v

# 模型配置
model_config = ConfigDict(
    str_strip_whitespace=True,
    extra='forbid',
    frozen=True,
)

# FastAPI集成
@app.post("/users/")
async def create_user(user: UserCreate):
    return user.model_dump()
```

## 
> ▶ 对应原理：[[03-Pydantic数据校验与v2新特性|03-Pydantic数据校验与v2新特性]]

相关链接

- 📋 目录：[[00-FastAPI]]
- 📚 学习清单： Web
- 🔗 [[03-请求体与Pydantic|请求体与Pydantic基础]]
- 🔗 [[05-错误处理与数据校验|错误处理与数据校验]]
- 🔗 [[06-依赖注入|依赖注入]]
- 项目实践：[[项目实战/ai-resume笔记/01_技术研读/01_架构概览|ai-resume: 架构概览]]
- 项目实践：[[项目实战/cr-agent笔记/01-技术研读/05-API与CLI接口契约|cr-agent: API与CLI接口契约]]

## 速记卡（面试闪卡）

**Q1：一句话讲清「Pydantic v2 进阶：field_validator / EmailStr / model_config」到底是什么？**
A：**Rust 核心**：验证逻辑用 Rust 实现，比纯 Python 快 10-50 倍
**更好的错误信息**：更精确的错误定位
**JSON Schema 支持**：更标准的 JSON Schema 生成

**Q2：1. Pydantic v2 新特性 —— 怎么理解？**
A：**Rust 核心**：验证逻辑用 Rust 实现，比纯 Python 快 10-50 倍
**更好的错误信息**：更精确的错误定位
**JSON Schema 支持**：更标准的 JSON Schema 生成

**Q3：核心速记主线有哪些？**
A：抓住这几根：1. Pydantic v2 新特性、2. field_validator、3. EmailStr、4. model_config、5. SettingsConfigDict（FastAPI配置）、6. 与 FastAPI 集成。


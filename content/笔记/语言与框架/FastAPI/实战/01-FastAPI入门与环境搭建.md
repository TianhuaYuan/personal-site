---
title: "FastAPI入门与环境搭建"
created: "2025-07-12"
tags:
  - 技术学习
  - fastapi
  - web框架
---

# FastAPI入门与环境搭建

> 写给初学者：每个概念都从"为什么需要它"开始讲，配合通俗比喻和完整可运行代码。

---

## 一、FastAPI 是什么？

**一句话：用 Python 写 API 接口的框架，自动生成文档，速度快。**

对比其他方案：

| 方案 | 特点 |
|------|------|
| Flask | 轻量但功能靠插件拼，没有自动文档 |
| Django REST | 功能全但笨重，学习成本高 |
| **FastAPI** | 轻量 + 自动文档 + 类型校验 + 异步支持 |

**通俗比喻：** FastAPI 就像一个自动化的前台接待员——你定义好规则（接口），它自动帮你接待来访者（处理请求）、检查证件（校验数据）、返回结果（响应）。

---

## 二、环境安装与第一个接口

### 2.1 安装

```bash
pip install fastapi uvicorn
```

- `fastapi`：框架本身
- `uvicorn`：ASGI 服务器，用来运行你的 FastAPI 应用（相当于把你的代码跑起来的"发动机"）

### 2.2 最简代码

创建文件 `main.py`：

```python
from fastapi import FastAPI

# 创建 FastAPI 应用实例
# 这个 app 就是你的整个应用，所有接口都注册在它身上
app = FastAPI()

# 定义一个接口
# @app.get("/") 的意思是：当有人用 GET 方法访问 "/" 这个路径时
# 执行下面这个函数
@app.get("/")
async def hello():
    return {"message": "Hello World"}
```

### 2.3 运行

```bash
uvicorn main:app --reload
```

- `main`：文件名（`main.py` 去掉 `.py`）
- `app`：变量名（你代码里 `app = FastAPI()` 这个变量）
- `--reload`：代码改了自动重启（开发时用，上线时去掉）

**运行后你会看到：**
```text
Uvicorn running on http://127.0.0.1:8000
```

### 2.4 访问接口

| 地址 | 说明 |
|------|------|
| `http://127.0.0.1:8000/` | 你的接口，返回 `{"message": "Hello World"}` |
| `http://127.0.0.1:8000/docs` | **自动生成的 API 文档**（Swagger UI） |
| `http://127.0.0.1:8000/redoc` | 另一种风格的文档（ReDoc） |

> **FastAPI 最爽的一点：你不用写文档，它自动根据你的代码生成。** 打开 `/docs` 就能看到所有接口，还能直接在里面测试。

---

## 速查表

| 概念 | 一句话解释 | 关键代码 |
|------|---------|---------|
| `FastAPI()` | 创建应用实例 | `app = FastAPI()` |
| `uvicorn` | 运行应用的服务器 | `uvicorn main:app --reload` |
| `/docs` | 自动生成的 API 文档 | 浏览器打开即可 |

---

## 
> ▶ 对应原理：[[01-FastAPI为什么快-Starlette与Pydantic与async|01-FastAPI为什么快-Starlette与Pydantic与async]]


> ▶ 对应原理：[[05-def与async-def路由选择|05-def与async-def路由选择]]


> ▶ 对应原理：[[13-FastAPI性能优化-连接池与uvicorn-workers|13-FastAPI性能优化-连接池与uvicorn-workers]]

相关链接

- 目录：[[00-FastAPI]]
- 下一篇：[[02-路径参数与查询参数]]
- 项目实践：[[项目实战/ai-resume笔记/01_技术研读/01_架构概览|ai-resume: 架构概览]]
- 项目实践：[[项目实战/cr-agent笔记/01-技术研读/05-API与CLI接口契约|cr-agent: API与CLI接口契约]]

---
→ [[技术学习清单#基础篇]]

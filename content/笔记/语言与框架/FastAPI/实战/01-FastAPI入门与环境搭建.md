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

# 这个 app 就是你的整个应用，所有接口都注册在它身上

app = FastAPI()

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

## 速记卡（面试闪卡）

**Q1：一句话讲清「FastAPI入门与环境搭建」到底是什么？**

A：FastAPI 是用 Python 写 API 的框架，自带自动文档、类型校验和异步支持，等于一个自动接待访客的前台。

**Q2：它和 Flask、Django 比强在哪？ —— 怎么理解？**

A：像开餐厅——Flask 是空店面自己买家具（轻量但全靠插件），Django 是整栋大酒店（功能全但笨重），FastAPI 是精装智能前台：轻量 + 自动生成 Swagger 文档 + 类型校验 + 异步，开箱即用。

**Q3：怎么装和跑起来第一个接口？ —— 怎么理解？**

A：像买发动机装车——pip install fastapi uvicorn，uvicorn 是 ASGI 服务器（把代码跑起来的发动机）；几行写个 app 实例加 @app.get 路由，uvicorn main:app --reload 启动，访问 /docs 就能看自动文档。

**Q4：最爽的点为什么是自动文档？ —— 怎么理解？**

A：像菜做好了自动摆盘还附说明书——你不用写文档，FastAPI 根据你的代码和类型注解自动生成 Swagger UI（/docs）和 ReDoc（/redoc），还能直接在页面里点按钮测试接口。

**Q5：@ app.get 和 uvicorn 命令怎么理解？ —— 怎么理解？**

A：@app.get("/") 像在门上贴条"有人 GET 这个路径就执行这个函数"；uvicorn main:app 里 main 是文件名、app 是你代码里 FastAPI() 的变量名，--reload 是开发时改代码自动重启。

**Q6：核心速记主线有哪些？**

- FastAPI = Python 写 API 的框架，特点：轻量 + 自动文档 + 类型校验 + 异步

- 对比：Flask 轻量无文档、Django 笨重，FastAPI 取中间甜点

- 安装用 fastapi + uvicorn（ASGI 服务器），启动 uv uvicorn main:app --reload

- 打开 /docs 看自动生成的 Swagger 文档，还能在线测试接口

**口诀**

A：FastAPI写接口，前台自动接待你。

pip装fastapi加uvicorn，发动机点火跑起。

几行代码挂路由，get路径就响应。

/docs自动出文档，在线点测真省心。

相关链接

- 目录：[[00-FastAPI]]

- 下一篇：[[02-路径参数与查询参数]]

---

→ [[技术学习路线图#基础篇]]

## 相关链接

- [[笔记/语言与框架/FastAPI/实战/07-中间件|中间件]]

- [[笔记/语言与框架/FastAPI/实战/02-路径参数与查询参数|路径参数与查询参数]]

- [[笔记/语言与框架/FastAPI/实战/06-依赖注入|依赖注入与路由分组]]

- [[笔记/语言与框架/FastAPI/实战/08-后台任务与CORS|后台任务与CORS]]

- [[笔记/语言与框架/FastAPI/实战/03-请求体与Pydantic|请求体与Pydantic]]


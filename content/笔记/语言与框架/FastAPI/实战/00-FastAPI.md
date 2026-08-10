---
title: "FastAPI学习目录索引"
created: "2025-07-12"
tags:
  - 技术学习
  - fastapi
  - 索引
---

# FastAPI学习目录索引

> 📚 **八股文版** → [[语言与框架/FastAPI/八股/00-FastAPI-Web|FastAPI 八股文]]

---

## 学习路线（按技术学习清单顺序）

### 基础篇

| 编号 | 主题 | 核心内容 |
|------|------|---------|
| 01 | [[01-FastAPI入门与环境搭建]] | FastAPI是什么、环境安装、第一个接口、自动文档 |
| 02 | [[02-路径参数与查询参数]] | 路径参数、查询参数、类型转换、可选参数 |
| 03 | [[03-请求体与Pydantic]] | 请求体、BaseModel、Field约束、Query/Path/Body显式声明 |

### 进阶篇

| 编号 | 主题 | 核心内容 |
|------|------|---------|
| 04 | [[04-响应模型与状态码]] | 响应模型、响应类型、状态码、JSONResponse/HTMLResponse/FileResponse |
| 05 | [[05-错误处理与数据校验]] | Pydantic校验、HTTPException、自定义异常、全局异常处理 |

### 工程化篇

| 编号 | 主题 | 核心内容 |
|------|------|---------|
| 06 | [[06-依赖注入]] | Depends依赖注入、类依赖、yield依赖、APIRouter路由分组 |
| 07 | [[07-中间件]] | HTTP中间件、洋葱模型、请求日志、异常兜底 |
| 08 | [[08-后台任务与CORS]] | BackgroundTasks、CORSMiddleware、SSE流式响应 |
| 09 | [[09-Pydantic-v2进阶]] | field_validator、EmailStr、model_config、SettingsConfigDict |
| 10 | [[10-Rate-Limiting]] | slowapi四档限流、存储后端、错误处理 |
| 11 | [[11-异步SQLAlchemy与连接池]] | async engine、pool_pre_ping、pool_recycle、FastAPI集成 |
| 12 | [[12-安全响应头中间件]] | X-Content-Type-Options、X-Frame-Options、X-XSS-Protection |

---

## 快速查阅

- **想看参数处理？** → [[02-路径参数与查询参数]]、[[03-请求体与Pydantic]]
- **想看数据校验？** → [[03-请求体与Pydantic]]、[[05-错误处理与数据校验]]
- **想看响应控制？** → [[04-响应模型与状态码]]
- **想看错误处理？** → [[05-错误处理与数据校验]]
- **想看项目架构？** → [[06-依赖注入]]、[[08-后台任务与CORS]]

---

## 相关链接

- 🔗 [[语言与框架/FastAPI/八股/00-FastAPI-Web|FastAPI 八股文笔记]] — 对应FastAPI原理篇

- 总目录：[[00-技术学习总目录]]
- 八股文笔记：[[八股文学习清单]]

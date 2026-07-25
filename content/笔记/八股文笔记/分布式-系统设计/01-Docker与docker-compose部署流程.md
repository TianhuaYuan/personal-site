---
title: "Docker 与 docker-compose 部署流程"
created: "2026-07-20"
tags:
  - 八股文
  - 分布式-系统设计
---

# Docker 与 docker-compose 部署流程

> 📌 本篇归属「十、系统设计-分布式 / 架构与容器」，讲 Docker 镜像与容器、docker-compose 多服务编排与生产部署流程。

## 一句话总结

> **Docker = 标准化集装箱（一次构建到处跑），docker-compose = 工头（一键起全部服务）。FastAPI 服务打包成镜像，配好 uvicorn workers，再和 PostgreSQL/Redis 用 compose 编排，本地 `up` 一键起，生产 `push` 到仓库再 `pull` 跑。**

---

## Image vs Container

| 概念 | 类比 | 说人话 |
| :--- | :--- | :--- |
| **Image（镜像）** | 菜谱 | 只读模板，含代码+依赖+环境 |
| **Container（容器）** | 做好的菜 | 镜像的运行实例，可启停删 |
| **Dockerfile** | 菜谱写法 | 告诉 Docker 怎么构建镜像 |

## Dockerfile（FastAPI 生产版）

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
# 生产用 gunicorn 拉起多个 uvicorn worker, 见 13-性能优化
CMD ["gunicorn", "main:app", "-k", "uvicorn.workers.UvicornWorker", "--workers", "4", "--bind", "0.0.0.0:8000"]
```

## docker-compose.yml

```yaml
services:
  backend:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/myapp
    depends_on:
      - db
  db:
    image: postgres:15
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata:
```

## 8 步部署流程

| 步骤 | 命令 |
| :--- | :--- |
| ① 本地开发 | `docker-compose up` |
| ② 构建镜像 | `docker-compose build` |
| ③ 打标签 | `docker tag` |
| ④ 推仓库 | `docker push` |
| ⑤ 登录服务器 | `ssh user@server` |
| ⑥ 拉镜像 | `docker-compose pull` |
| ⑦ 重启 | `docker-compose up -d` |
| ⑧ 收工 | `docker-compose down` |

## 常见

| Q | A |
| :--- | :--- |
| 镜像和容器的区别？ | 镜像=模板（只读），容器=实例（可写） |
| `depends_on` 能保证可用吗？ | 不能——只保证启动顺序，要可用得加 `healthcheck` |
| 生产用 compose 够吗？ | 单机够，集群上 K8s |

## 与性能优化的关系

compose 里的 backend 服务用 `gunicorn + UvicornWorker --workers 4` 起多进程，正是 [[13-FastAPI性能优化-连接池与uvicorn-workers|13-性能优化]] 的生产落地。

## 记忆口诀

> **镜像一次构建到处跑，容器是跑起来的实例。**
> **Dockerfile 写菜谱，compose 当工头。**
> **depends_on 只管顺序不管健康，生产加 healthcheck。**

---

## 相关链接

- [[13-FastAPI性能优化-连接池与uvicorn-workers|性能优化]]
- [[00-分布式-系统设计|分布式-系统设计 索引]]

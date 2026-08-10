---
title: "CI-CD加固"
tags:
  - 项目笔记
  - ai-resume
created: "2026-07-21"
---

# CI-CD加固

> 电梯陈述：「把一条充满隐雷的 CI/CD 流水线，从 Docker Hub + 稀疏检出 + 旧版 SDK + 错误回滚逻辑中捞出来，换成了阿里云 ACR + 原生 SSH + 版本锁定的稳定链路。」

## 一、背景（为什么要做这个）

项目已经到 Phase  工程收尾阶段，CI/CD 是上线前最后一道关卡。之前跑通了  个测试，但 CI/CD 流水线本身有多处隐性问题：

1. **`mcp>=1.28.0` 不锁定版本** — `pip install` 随时可能拉到 2.0  版导致  被移除，11 个测试同时崩溃
2. **回滚逻辑打反 tag** — 部署失败的回滚其实是把新镜像  成旧镜像名，等于没回滚
3. **ECS 到 Docker  网络不通** — 国内服务器拉不到 Docker  镜像
4. **sparse-checkout 导致后端源码消失** — 工作目录只剩下  目录
5. **`JSONFormatter` 遇到  就崩溃** —  注册  时 logger.debug 带 AnyUrl，序列化失败
6. **`appleboy/ssh-action` drone-ssh 密钥解析 bug** — 新版  私钥格式不被支持

不做的话，上线当天大概率部署失败，或者出现"回滚了但没完全回滚"的生产事故。

## 三、逐个讲：问题 → 怎么想 → 怎么解

### 问题 1 & 2：AnyUrl 序列化崩溃

**问题：** 测试在  上全崩，错误是 `TypeError: Object of type AnyUrl is not JSON serializable`。本地却跑得通。

**一开始怎么想的：** 以为是测试代码的问题，用了 `str()` 转换 dict key。跑完发现本地  全过。于是怀疑版本不一致。

**实际分析：** 看到完整堆栈后才发现 — 不是测试的问题。`_register_handlers()` → 导入资源模块 → `@mcp.resource("resume://list")` → `self.add_resource(resource)` → `logger.debug(resource)` → 日志记录器把  放进了 log_entry → `JSONFormatter` 的 `json.dumps()` 没有兜底 → 崩溃。

```mermaid
sequenceDiagram
    participant Test  测试
    participant Register as _register_handlers
    participant MCP as FastMCP
    participant Logger as JSONFormatter
    Register->>MCP: 导入 resumes.py
    MCP->>MCP: @mcp.resource("resume://list")
    MCP->>Logger: logger.debug(resource)
    Logger->>Logger: json.dumps(log_entry)
    Note over Logger:  不可序列化 ❌
    Logger-->>Test: TypeError
```

**修复：** 给 `json.dumps()` 加 `default=_json_default`，遇到不可序列化类型降级为 `str(obj)`。

同时顺手加上了 `mcp>=1.28.0,<2.0.0` 的版本上限。

### 问题 4：CD 回滚逻辑打反 tag

**问题：** 部署失败后的回滚代码跑的其实是新版本。

```bash
# 当时的代码（错的）
docker tag "app:${IMAGE_TAG}" "${OLD_BACKEND}"
docker compose up -d
```

这行的本意是"把旧镜像拿回来重启"，但实际效果是把新镜像  成旧镜像的名字 —— 重启的还是新版本。

**修复：** 保存旧的 IMAGE_TAG，失败后 `export IMAGE_TAG=${OLD_TAG}` 再 `docker compose up -d`。

### 问题 5 & 6：Secrets 作用域 + 阿里云  国内网络

**问题：** 部署时 `missing server host`，但实际上已经在  配了 Secrets。

**排查：** 发现 `prepare`  没有 `environment:` 声明。GitHub  的环境级  只在有 `environment:` 的  中可以访问。`prepare` 能读到  级 Secrets，但读不到环境级的 `STAGING_DEPLOY_HOST`。

**修复：** 把主机地址解析从 `prepare`  移到 `deploy` job（有 `environment:`）。

紧接着遇到第二堵墙：阿里云  无法连接 Docker Hub。先是 `curl: (6) Could not resolve host`，修好  换阿里云 DNS `223.5.5.5` 后能解析了，但 Docker  国际出口还是超时。

```mermaid
flowchart LR
    subgraph 最初[最初]
        GH[GitHub Actions] -->|push| DH[Docker Hub]
        DH -->|pull| ECS[阿里云 ECS]
        ECS -.->|超时| DH
    end
    subgraph 最终[最终]
        GH2[GitHub Actions] -->|push| ACR[阿里云  广州]
        ACR -->|内网拉取| ECS2[阿里云  广州]
    end
```

**最终方案：** 切换到阿里云容器镜像服务 ACR。GitHub  构建后推到广州节点 ACR，同区域  从  内网拉取，速度从超时降到秒级。

### 问题 8：git sparse-checkout 导致源码消失

**问题：** 工作目录只剩 `deploy/` 目录，`backend/`、`frontend/`、`.github/` 全部消失。测试跑不了，文件读不到。

**一开始以为是误删：** 想从 git restore，但  说文件不存在。

**排查：** `git config core.sparseCheckout` 返回 `true`，`git sparse-checkout list` 返回 `deploy`。

**根因：** 之前的某个操作（可能是 pre-commit 或手动命令）执行了 `git sparse-checkout set deploy`，git 从此只显示  目录，其他文件虽然在 git  中但不出现在工作目录。

**修复：** `git sparse-checkout disable`。需要注意的是  后文件恢复到工作目录，但**未提交的修改不会被覆盖**。

### 问题 12：appleboy/ssh-action 密钥解析

**问题：** 无论怎么填 `DEPLOY_KEY`，都报 `ssh.ParsePrivateKey: ssh: no key found`。试了 RSA PEM、OpenSSH 格式，甚至重新生成了密钥对，全部失败。

**根源：** `appleboy/ssh-action@v1` 使用的是 **drone-ssh v1.8.2**（非常老），其 Go  库对某些密钥格式解析有 bug。

**修复：** 换用 `webfactory/ssh-agent` + 原生 `ssh` 命令。`ssh-add` 用  标准库或  解析密钥，不会有 drone-ssh 的兼容性问题。

```mermaid
flowchart LR
    subgraph 之前[之前]
        A[appleboy/ssh-action v1] --> B[drone-ssh v1.8.2]
        B --> C[ssh.ParsePrivateKey]
        C -.->|bug| D[no key found]
    end
    subgraph 之后[之后]
        E[webfactory/ssh-agent] --> F[ssh-add]
        F --> G[标准  库解析]
        G --> H[连接成功]
    end
```

## 四、关键决策与取舍

| 决策 | 选项 | 选了这个 | 放弃了 |
|------|------|----------|--------|
| Docker Registry | Docker Hub / Aliyun ACR | ACR（国内网络原因） | Docker  的全球分发 |
|  方式 | appleboy/ssh-action / 原生 SSH | 原生 SSH + webfactory/ssh-agent |  的第三方维护 |
|  版本策略 | 锁定 minor / 锁定 exact / 设上限 | `>=1.28.0,<2.0.0` | 精确锁定可能错过安全更新 |
|  网络方案 | 镜像加速器 / ACR / VPN | 镜像加速器（公开镜像）+ ACR（私有） | 纯  方案需要迁移全部依赖 |
|  密钥 | 复用本机 key /  生成 |  生成  格式专用 key | 本机  可移植性 |
| 前端测试 | 单独 tsconfig.test / 构建排除 | 构建时排除 `*.test.*` | 测试文件的类型检查独立配置 |

## 五、踩坑（值得讲的故事）

### 1. git sparse-checkout：没删文件，只是看不见

发现 `tests/` 目录从工作目录消失了，第一反应是误删了。`git status` 没显示删除，`git ls-files --deleted` 返回 0。折腾了半天才发现 `git sparse-checkout list` 返回 `deploy`。这个功能设计上就不让用户察觉 — 文件在  里完好，只是工作目录不显示。

### 2. drone-ssh +  密钥：不兼容就是不给过

`ssh: no key found` 这个错误排查了  分钟。试了  格式、OpenSSH 格式、ECS 重新生成、本机转换。最终还是不行。查了 appleboy/drone-ssh 的源码才发现 v1.8.2 的密钥解析路径在某些边界条件下有问题。最终把整个  换掉了才解决 — 不是密钥的内容不对，是解析器有 bug。

### 3. 回滚打反 tag：看上去对，实际上大错特错

`docker tag new:tag old_tag` — 这行代码看起来像是在恢复旧镜像，实际效果是把新镜像的名称改成旧镜像的名字。就像把新手机贴上旧手机的标签，以为在用旧手机。`docker compose up -d` 用的还是 `IMAGE_TAG` 这个环境变量 — 而它的值根本没有变。这个  在代码审查阶段应该被发现的。

### 4. 阿里云  到 Docker  的"最后一公里"

即使配了镜像加速器，私有仓库的拉取请求仍然走 Docker  直连，加速器不管用。这是  的设计：只有公开镜像（`library/nginx`）走 mirror，私有仓库（`yourname/app`）不经过 mirror。所以即使加速器配好了，私有镜像还是拉不了。

## 六、常见疑问

### 为什么不用 Docker  而用 ACR？
**答：** GitHub  的  在海外，推镜像到 Docker  没问题。但部署的  在阿里云广州，国内服务器连 Docker  经常超时。试了阿里云镜像加速器解决了公开镜像的问题，但自定义镜像（私有仓库）不经过加速器。最终选了 ACR：同一区域的  从  拉取走阿里内网，稳定且快。

### appleboy/ssh-action 的密钥问题的根因是？
**答：** 旧版 appleboy/ssh-action（v1.x）内置了一个叫做 drone-ssh 的  二进制（v1.8.2）来处理  连接和脚本执行。这个版本的密钥解析代码对  格式（`-----BEGIN OPENSSH PRIVATE KEY-----`）支持不完整。换成了 webfactory/ssh-agent + 原生  命令，使用系统本身的  客户端解析密钥，兼容性比内置二进制好得多。

### git sparse-checkout 是怎么被触发的？
**答：** 这是误操作。`git sparse-checkout set deploy` 这个命令会在本地开启稀疏检出模式，工作目录只显示 `deploy/` 目录。触发原因不确定，可能是某个 pre-commit  或手动命令。解决方案很简单：`git sparse-checkout disable`。

### 为什么 `${{ secrets.X }}` 在注释中也会报错？
**答：** GitHub  的表达式求值是"编译时"行为，在  被解析时就会求值 `${{ }}`，不管它在注释里还是实际代码里。所以被注释掉的 `${{ secrets.DINGTALK_WEBHOOK }}` 也会被求值，如果  不存在就报错。解决方案是删掉注释掉的代码，或者用 `'${{'` 逃逸。

### 整个过程中你最大的收获是什么？
**答：** 心态上不要急着修代码，先追根因。AnyUrl 错误一开始以为是测试或版本问题，最终堆栈显示是  的 json.dumps 没有  兜底。回滚打反  的问题，单看那行 `docker tag` 觉得写得没问题，但对照完整流程（tag → compose up）才发现新版本根本没变。教训是：CI/CD 的问题，60% 是对工具链的理解深度不够。



> ▶ 关联技术研读：[[01_技术研读/01_架构概览|01_架构概览]]

## 速记卡（面试闪卡）

**Q1：一句话讲清「CI-CD加固」到底是什么？**
A：项目已经到 Phase  工程收尾阶段，CI/CD 是上线前最后一道关卡。之前跑通了  个测试，但 CI/CD 流水线本身有多处隐性问题：
** 不锁定版本** —  随时可能拉到 2.0  版导致  被移除，11 个测试同时崩溃
**回滚逻辑打反 tag** — 部署失败的回滚其实是把新镜像  成旧镜像名，等于没回滚

**Q2：一、背景（为什么要做这个） —— 怎么理解？**
A：项目已经到 Phase  工程收尾阶段，CI/CD 是上线前最后一道关卡。之前跑通了  个测试，但 CI/CD 流水线本身有多处隐性问题：
** 不锁定版本** —  随时可能拉到 2.0  版导致  被移除，11 个测试同时崩溃
**回滚逻辑打反 tag** — 部署失败的回滚其实是把新镜像  成旧镜像名，等于没回滚
**ECS 到 Docker  网络不通** — 国内服务器拉不到 Docker  镜像

**Q3：三、逐个讲：问题 → 怎么想 → 怎么解 —— 怎么理解？**
A：**问题：** 测试在  上全崩，错误是 。本地却跑得通。
**一开始怎么想的：** 以为是测试代码的问题，用了  转换 dict key。跑完发现本地  全过。于是怀疑版本不一致。
**实际分析：** 看到完整堆栈后才发现 — 不是测试的问题。 → 导入资源模块 →  →  →  → 日志记录器把  放进了 log_entry →  的  没有兜底 → 崩溃。
**修复：** 给  加 ，遇到不可序列化类型降级为 。

**Q4：四、关键决策与取舍 —— 怎么理解？**
A：| 决策 | 选项 | 选了这个 | 放弃了 |
|------|------|----------|--------|
| Docker Registry | Docker Hub / Aliyun ACR | ACR（国内网络原因） | Docker  的全球分发 |
|  方式 | appleboy/ssh-action / 原生 SSH | 原生 SSH + webfactory/ssh-agent |  的第三方维护 |

**Q5：五、踩坑（值得讲的故事） —— 怎么理解？**
A：发现  目录从工作目录消失了，第一反应是误删了。 没显示删除， 返回 0。折腾了半天才发现  返回 。这个功能设计上就不让用户察觉 — 文件在  里完好，只是工作目录不显示。
 这个错误排查了  分钟。试了  格式、OpenSSH 格式、ECS 重新生成、本机转换。最终还是不行。查了 appleboy/drone-ssh 的源码才发现 v1.8.2 的密钥解析路径在某些边界条件下有问题。

**Q6：核心速记主线有哪些？**
A：抓住这几根：一、背景（为什么要做这个）、三、逐个讲：问题 → 怎么想 → 怎么解、四、关键决策与取舍、五、踩坑（值得讲的故事）、六、常见疑问。


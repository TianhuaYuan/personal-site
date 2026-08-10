---
title: "GitHub PR 集成"
tags:
  - 项目笔记
  - cr-agent
created: "2026-07-21"
---

# GitHub PR 集成

> .patch 格式 vs REST API、Webhook HMAC-SHA256 签名验证、patch 解析上下文行丢失的隐蔽坑，以及 request body consume-once 陷阱。

## 一、背景（为什么要做这个）

Playground 粘贴代码审查很方便，但真实场景下开发者的代码在 GitHub 上，审查入口应该是 PR URL。做 Phase 7 时，目标是让用户输入 PR 链接就能触发完整审查——自动拉取 diff、检测语言、并行审查、生成报告。核心价值是消除"复制粘贴"这一步骤。

## 二、逐个讲：问题 → 怎么想 → 怎么解

### 1. .patch 格式 vs REST API

**问题**：从 GitHub 拉取 PR diff 有两种方式——REST API（有 rate limit，需 token）或 `.patch` 后缀（纯 HTTP GET，无 token，不限速）。选哪个？

**怎么想**：REST API 适合需要结构化数据（文件列表、commit 信息、review 评论）的场景。但我只需要 diff 内容——`owner/repo/pull/123.patch` 直接返回 unified diff 纯文本，0 鉴权，0 限速。缺点：私有仓库需要 token，公开仓库够用。

**怎么解**：选 `.patch` 格式。`GitHubClient.get_pr_patch()` 在 PR URL 后加 `.patch`，HTTP GET 拉取。`parse_pr_url()` 用正则解析 `github.com/{owner}/{repo}/pull/{number}`。错误处理分两层：URL 格式不对返回 400，网络/权限失败返回 502。

### 2. patch 解析上下文行丢失

**问题**：patch 格式只包含改动行附近的行。`API_KEY = "sk-xxx"` 这类关键上下文如果不在改动范围内，就不会出现在 diff 里——LLM 审查时看不到它，等于盲审。

**怎么想**：patch 的核心问题是**只保留了 `+` 新增行和上下文行，丢弃了 `-` 删除行**。但上下文的"边界"由 git 决定（默认 3 行），不是由"重要性"决定。要解决这个问题，要么用完整文件替换 patch（但 cost 高），要么接受这个局限。

**怎么解**：在 `parse_patch_to_code()` 里做有限优化——保留 `+` 新增行和上下文行，丢弃 `-` 删除行，但每段 diff 前加 `# File: path/to/file.py` 标记。这样 LLM 至少知道哪个文件出了问题。后续优化方向：用 GitHub Contents API 拿完整文件。

### 3. Webhook HMAC-SHA256 签名验证

**问题**：GitHub Webhook 用 secret 签名 payload，验证时 HMAC 计算方式和 GitHub 的实现必须完全一致——一个字符不对就验签失败。

**怎么想**：关键是 `hmac.compare_digest` 防时序攻击 + payload 必须是原始字节（不能 decode/re-encode，字符编码一致性问题）。

**怎么解**：`_verify_signature(payload: bytes, signature: str, secret: str) -> bool`——用 `hmac.new(secret.encode(), payload, hashlib.sha256).hexdigest()` 计算签名，前缀 `sha256=` 后与 `signature` 比较。关键细节：payload 必须是 `await request.body()` 拿到的原始字节，不能 `json.loads` 再 `json.dumps`（会改变 key 顺序）。同时加 `WEBHOOK_SECRET_REQUIRED` 开关——开发态默认关闭（没配 secret 也能本地测），生产部署必须设 True。

### 4. request body consume-once 陷阱

**问题**：Webhook handler 里先 `await request.body()` 验签，再 `await request.json()` 读 payload——第二次读返回空。

**怎么想**：Starlette 的 Request.body 是一次性消费的——读完就被消费了，不能读第二次。但验签需要原始字节，处理逻辑需要 JSON 对象。

**怎么解**：`body = await request.body()` 拿到原始字节，验签用 body，`data = json.loads(body)` 解析 JSON。避免两次 await。或者把解包后的 data 放到 `request.state` 里供后续 handler 用。

## 三、关键决策与取舍

| 决策 | 选择 | 取舍 |
|------|------|------|
| PR diff 获取 | `.patch` 格式 | 0 token 0 限速，但私有仓库要 token |
| 多文件 patch 合并 | 文件标记 + 合并字符串 | 比逐个文件审查简单，但大 PR 上下文可能丢失 |
| API 端点 | 独立 `from-pr` 端点，不复用 `/reviews` | 路由多了但语义清晰，schema 简单 |
| 前端模式切换 | 分段控件，同页切换 | 不复用报告/进度/历史的迁移成本 |
| Webhook 安全开关 | `WEBHOOK_SECRET_REQUIRED` 默认 False | 开发友好，但生产部署必须手动开启 |

## 四、踩坑（值得讲的故事）

**1. patch 解析后上下文行消失**

现象 → 根因 → 解法 → 教训

LLM 审查 PR 时漏报了 `API_KEY = "sk-xxx"`——这个变量在上下文行里，但不在 diff 的改动范围内。patch 只包含改动行 ±3 行，超出的"关键上下文"不在 diff 里。解法：接受这个局限，加文件标记，后续用 Contents API 补完。教训：**patch 是"增量"，不是"全量"**——用 patch 审查要注意"看不到的部分"可能藏问题。

**2. Webhook payload body 消费两次**

现象 → 根因 → 解法 → 教训

验签时 `body = await request.body()`，处理时 `data = await request.json()`——返回空 dict。查了一小时发现 Starlette 的 Request.body 消费后 body 流就关闭了，第二次 await 拿到的是空。解法：`json.loads(body)` 在验签后直接用 body 解析。教训：**ASGI 的 request body 是一次性流**，要先算出要什么（验签需要原始字节）再设计消费顺序。

**3. PR URL 正则不够宽松**

现象 → 根因 → 解法 → 教训

`https://github.com/owner/repo/pull/123` 解析正常，但 `https://www.github.com/owner/repo/pull/123/files` 解析失败——正则没考虑 `www.` 前缀和尾部 `/files`。解法：正则用 `search()` 而非 `match()`，且 tail 部分接受任意字符。教训：**URL 解析正则要比"只覆盖自己见过的格式"更宽松**——用户粘贴的 URL 格式千奇百怪。

## 五、常见疑问

**Q1：为什么不用 GitHub REST API，非要用 .patch？**

A：三个原因。第一，零鉴权——公开仓库直接 HTTP GET 就行，不需要 GitHub App 或 Personal Access Token。第二，不限速——REST API 有 5000 req/h 的 rate limit，.patch 没有。第三，实现简单——就是文本拉取 + 解析，不需要处理分页、文件列表。缺点是不能访问私有仓库，Playground 场景公开仓库够用。

**Q2：patch 解析后多文件怎么处理？**

A：合并成一个带文件标记的代码字符串：每个文件前加 `# File: path/to/file.py`，diff 内容保留 `+` 新增行和上下文行，丢弃 `-` 删除行。然后交给 supervisor，它的 decompose_node 会按逻辑再拆解成任务分给 Worker。相当于两层拆解——先按文件拆，再按审查职责拆。

**Q3：Webhook 验签为什么要用原始字节，不能 JSON 再序列化？**

---

## 相关笔记

- [[01-技术研读/00-17条架构决策总览|00 17条架构决策总览]]
- [[01-技术研读/08-GitHub集成|08 GitHub集成]]
- [[10_LLM评测体系搭建|10 LLM评测体系搭建]]

## 技术学习笔记

- [[15-Agent架构与工具调用|Agent架构与工具调用]]
- [[26-MCP协议核心概念|MCP协议基础]]
- [[工程化与运维/工程化部署/04-GitHub-Actions-CI流水线|GitHub-Actions-CI]]
- [[工程化与运维/工程化部署/05-GitHub-Actions-CD流水线|GitHub-Actions-CD]]
- [[工程化与运维/工程化部署/10-bcrypt密码哈希|bcrypt密码哈希]]

A：HMAC 签名是基于精确字节计算的。JSON 的 key 顺序没有保证——Python dict 的 key 顺序和 GitHub 发送的 payload 的 key 顺序不一定一致。`json.loads` 再 `json.dumps` 后字节流变了，签名就不匹配。所以验签必须在 `request.body()` 的原始字节上做，不能 parse 后再 serialize。

---
title: "08：GitHub 集成"
tags:
  - 项目笔记
  - cr-agent
created: "2026-07-21"
---

# 08：GitHub 集成

## 概述

拉取 GitHub PR diff 并解析为可审查代码，支持 Webhook 自动触发审查。

**源码**：`integrations/github.py`（139 行）🟡 + `api/webhooks.py`（104 行）🟡

---

## 第一级：GitHubClient

📍 `integrations/github.py`：`GitHubClient`、`_build_patch_url`、`fetch_patch`、`parse_patch_to_code`、`detect_language`

### 为什么用 .patch 而非 REST API

GitHub 提供 `https://patch-diff.githubusercontent.com/raw/{owner}/{repo}/pull/{number}.patch` 直接返回 unified diff 纯文本。

| 对比 | .patch 方式 | REST API `/pulls/{n}/files` |
|------|-----------|----------------------------|
| 认证 | 不需要（可选加 token 提限速） | 必须 token |
| 限速 | 同 IP 限速，宽松 | 基于 token，严格 |
| 分页 | 一整个 body，无分页 | 文件多时分页 |
| 场景 | 轻量代码审查 | CI/CD 完整管道 |

### 核心 API

| 方法 | 输入 | 输出 | 用途 |
|------|------|------|------|
| `parse_pr_url(url)` | GitHub PR URL（正则匹配） | `(owner, repo, number)` | URL 解析 |
| `get_pr_patch(owner, repo, number)` | 仓库标识 | patch 原始文本 | 拉取 diff |
| `parse_patch_to_code(patch)` | patch 文本 | 带 `# File:` 标记的单字符串 | 交 supervisor 审查 |
| `parse_patch_to_files(patch)` | patch 文本 | `[(filename, code), ...]` | 逐文件审查 |
| `detect_language(patch)` | patch 文本 | 语言名（多数票） | 供 CLI/Webhook 复用 |

### 依赖注入模式

```python
class GitHubClient:
    def __init__(self, token=None, http_client=None):
        # http_client 可注入 fake，测试不触网
```

同 W1 LLM 注入模式——测试可 mock，不改业务逻辑。

---

## 第二级：.patch 解析策略

### parse_patch_to_code——逐行规则

```python
for line in patch.splitlines():
    if line.startswith("diff --git"):     # → "# File: <path>" 标记文件分界
    elif line.startswith("@@"):          # → "# @@ ..." 保留行号信息作为注释
    elif line.startswith("+++"):         # 跳过元数据
    elif line.startswith("---"):         # 跳过元数据
    elif line.startswith(" "):           # 上下文行：保留，去掉前导空格
    elif line.startswith("+"):           # 新增行：保留，去掉前导 +
    # - 删除行：直接丢弃
```

**关键选择**：保留 `+`（新增行）和 ` `（上下文行），丢掉 `-`（删除行）——审查只关注"加进来的代码"，删除行不产生新问题。

### parse_patch_to_files——按文件分组

作用和 `parse_patch_to_code` 类似但按文件返回 `[(filename, code)]`，适合独立逐文件审查，每个文件交给一个 Worker。

### detect_language——从扩展名统计

遍历 patch 中 `diff --git` 行的文件名后缀，用 `EXT_TO_LANGUAGE` 映射取多数票。无已知扩展名默认 `python`。

---

## 第三级：Webhook

📍 `api/webhooks.py`：Webhook 处理、HMAC 验证

### 端点

```text
POST /api/v1/webhooks/github
```

### 处理流程

```text
收到请求
  → _read_body_limited() 限流读取 body（上限 1MB，防 DoS）
  → _verify_signature() 验证 X-Hub-Signature-256
  → 检查 event == "pull_request" 且 action == "opened"
  → 提取 pr.html_url
  → fetch_pr_code(url) 复用 GitHubClient
  → build_supervisor_graph().ainvoke()
  → 返回报告
```

### 签名验证：HMAC-SHA256

```python
def _verify_signature(secret, body, signature):
    expected = hmac.new(secret.encode(), body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(signature[len("sha256="):], expected)
```

- `hmac.compare_digest`：**防时序攻击**（timing attack），恒定时间比较
- secret 为空 + `WEBHOOK_SECRET_REQUIRED=False`：开发态跳过验签
- secret 为空 + `WEBHOOK_SECRET_REQUIRED=True`：拒绝请求——生产环境必须配 secret

### 限流读取 request body

```python
_MAX_WEBHOOK_BODY = 1_000_000  # 1MB

async def _read_body_limited(request, max_bytes):
    # 先检查 Content-Length，超限直接 413
    # 再流式累加，超过上限终止
```

**为什么**（可讲）：未授权攻击者可能发送超大 payload，在验签前即拒绝超大 body，将攻击面最小化。GitHub PR diff 通常远小于此值。

### 事件过滤

只处理 `pull_request opened`（PR 新建/重开），其他事件返回 `{"status": "ignored"}`。避免 `synchronize`（push 到分支）产生重复审查噪音。

---

## Q&A

**Q1：为什么用 .patch 不用 GitHub REST API？**  
**A**：.patch 是纯文本 diff，无需 GitHub App/OAuth/token，不限速。REST 拉 files 要 token 且分页复杂。对于公开仓库，.patch 足够。

**Q2：Webhook 签名验证怎么保证安全？**  
**A**：先 await request.body 取原始字节，HMAC-SHA256 计算签名，hmac.compare_digest 防时序攻击。WEBHOOK_SECRET_REQUIRED 默认 False（开发友好），生产部署设 True。

**Q3：patch 解析时上下文行为什么重要？**  

---

## 相关笔记

- [[00-17条架构决策总览|00 17条架构决策总览]]
- [[05-API与CLI接口契约|05 API与CLI接口契约]]
- [[10-安全加固四道防线|10 安全加固四道防线]]

## 技术学习笔记

- [[15-Agent架构与工具调用|Agent架构与工具调用]]
- [[26-MCP协议核心概念|MCP协议基础]]
- [[工程化与运维/工程化部署/04-GitHub-Actions-CI流水线|GitHub-Actions-CI]]
- [[工程化与运维/工程化部署/05-GitHub-Actions-CD流水线|GitHub-Actions-CD]]
- [[工程化与运维/工程化部署/10-bcrypt密码哈希|bcrypt密码哈希]]
**A**：如果只保留 + 新增行，API_KEY = "sk-xxx" 这类未改动但关键的上下文行会消失，审查会变盲。必须保留 + 行和上下文行。

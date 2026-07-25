---
title: "JWT 原理与设计"
created: "2025-07-12"
tags:
  - 技术学习
  - python
  - jwt
  - 鉴权
---

# JWT 原理与设计


---

## 问题：HTTP 记不住你

你写过登录接口，但下一个请求进来时，服务器怎么知道是谁发的？

```text
请求1: POST /login   →  服务器确认"你是赵某，密码对"
请求2: GET /me       →  服务器收到的是另一个独立请求
                        HTTP 协议本身不记录任何"之前发生过什么"
```

> [!note] HTTP 无状态
> 每个请求互相独立，服务器不保留上一个请求的记忆。

---

## 方案一：Session — 服务器帮你记

登录后服务器生成一个编号 `abc123`，记在本子上，同时把编号给你。

```mermaid
sequenceDiagram
    participant 你
    participant 服务器

    Note over 你,服务器: 登录
    你->>服务器: POST /login {username, password}
    服务器->>服务器: 查密码 → 对 → 随机生成"abc123"
    服务器->>服务器: 在本子上记: abc123 → {"user_id": 1}
    服务器-->>你: 返回 abc123

    Note over 你,服务器: 后续请求
    你->>服务器: GET /me, 带着 abc123
    服务器->>服务器: 拿 abc123 查本子 → 找到当前用户
```

术语对应：
| 说法 | 术语 |
|------|------|
| 服务器记的"本子" | Session（会话数据） |
| 给你的编号"abc123" | session_id |
| 你每次请求带着编号 | Cookie |

### Session 的问题

```text
问题1: 记不住
  用户越多 → 本子越厚 → 内存不够 → 加 Redis → 加成本

问题2: 多台服务器不共享
  服务器A 存了 abc123 → 赵某
  下次请求打到服务器B → B 不认识 abc123 → 用户重新登录

  解决办法: 加公共 Redis

```mermaid
flowchart BT
    A[服务器A] --> Redis[Redis]
    B[服务器B] --> Redis
    Redis --> RISK[⚠️ 单点故障风险]
```text

问题3: 服务器重启 → 所有 Session 全丢 → 全部重新登录
```

Session 方式叫"有状态"——服务器的行为依赖它自己存的那本"本子"。

---

## 方案二：JWT — 防伪身份证

服务器**不记本子了**，改成给你发一张防伪身份证，票上写清楚你的信息。以后你每次来只验票。

```mermaid
sequenceDiagram
    participant 你
    participant 服务器

    Note over 你,服务器: 登录
    你->>服务器: POST /login
    服务器->>服务器: 验证密码 → 对
    服务器->>服务器: 生成防伪身份证<br/>（用只有自己知道的密码做的）
    服务器->>服务器: 发给你后服务器就忘了这事（不存！）
    服务器-->>你: 收到防伪身份证

    Note over 你,服务器: 后续请求
    你->>服务器: GET /me, 带着防伪身份证
    服务器->>服务器: 检查防伪标记 → ✓ 是老子自己签的
    服务器->>服务器: 读票上内容 → 知道你是谁
    服务器->>服务器: 全程没查任何数据库/Redis
```

```mermaid
flowchart LR
    subgraph Session["Session"]
        direction TB
        S1["登录 → 服务器生成编号 → 存 Redis → 给你编号"]
        S2["请求 → 你带编号 → 服务器查 Redis → 知道你是谁"]
    end

    subgraph JWT["JWT"]
        direction TB
        J1["登录 → 服务器做防伪身份证 → 给你 → 服务器忘掉"]
        J2["请求 → 你带票 → 检查防伪 → 读票内容 → 知道你是谁"]
    end
```

> [!danger] 核心认知
> JWT **没有加密你的信息**，它只保证"信息没被改过"。就像身份证——卡面信息明晃晃的，但防伪水印让你改不了。

---

## 签名机制（HS256）

服务器用一个只有自己知道的密码（密钥），对票的内容算一个"指纹"。

```mermaid
flowchart TB
    subgraph SIGN["① 做票时"]
        direction LR
        A1["内容: user_id=1, username=赵某"] -->|"+ 密码 → HMAC-SHA256"| A2["指纹: a8f3b9"]
        A2 --> A3["票 = 内容 + 指纹"]
    end

    subgraph VERIFY["② 验票时"]
        B1["服务器重新算: 内容 + 密码 → 指纹 a8f3b9"]
        B2{"跟票上的指纹比对"}
        B1 --> B2
        B2 -->|一致| B3["✅ 没被改过"]
    end

    subgraph TAMPER["③ 如果有人篡改"]
        C1["票被改成 user_id=1, username=赵某, vip=999"]
        C2["服务器重新算: 新内容 + 密码 → 指纹 c4d2e1"]
        C3{"跟票上的指纹 a8f3b9 比对"}
        C1 --> C2 --> C3
        C3 -->|不匹配| C4["❌ 拒收"]
    end
```

术语对应：
| 说法 | 术语 |
|------|------|
| 密码 | 密钥 (secret key) |
| 数学运算 | HMAC-SHA256 (HS256) |
| 算出的指纹 | 签名 (signature) |
| 内容+指纹一起发给用户 | 签发 Token |
| 服务器检查指纹 | 验证 Token |

> 把 HS256 理解成一个黑盒：`f(内容, 密码) → 指纹`。同一个输入永远输出同一个指纹，换一点内容指纹就全变了。

---

## JWT 长什么样

```mermaid
flowchart LR
    H["第一段 (Header)<br/>算法说明"] --> P["第二段 (Payload)<br/>用户信息 + 时间戳"] --> S["第三段 (Signature)<br/>防伪指纹"]
```

> [!note] Base64 不是加密
> 每一段都是 Base64 编码的文本，任何人都能解码，不要放密码等敏感信息。

### 亲手拆一个 JWT

```python
import base64, json

token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxIiwidXNlcm5hbWUiOiLotcTmn7wiLCJpYXQiOjE3MTk1MDAwMDAsImV4cCI6MTcxOTUwMTgwMH0.HrWQ3KxLmN8pY2zF5vB6dC7sA1tG4jU9wE0oR3nL5xY"

header_b64, payload_b64, signature = token.split(".")

def decode(s):
    s += "=" * (4 - len(s) % 4)
    return json.loads(base64.urlsafe_b64decode(s))

# 第一段：算法说明
print(decode(header_b64))
# → {"alg": "HS256", "typ": "JWT"}

# 第二段：票上的信息（明文！）
print(decode(payload_b64))
# → {"sub": "1", "username": "赵某", "iat": 1719500000, "exp": 1719501800}

# 第三段：指纹（乱码，无法还原为原始内容）
print(signature)
# → "HrWQ3KxLmN8pY2zF5vB6dC7sA1tG4jU9wE0oR3nL5xY"
```

```mermaid
flowchart LR
    H["🏷️ Header<br/><br/>算法说明:<br/>'我用 HS256 算的'<br/><br/>⚠️ 明文，随便看"]
    P["📋 Payload<br/><br/>用户信息:<br/>user_id=1, username=赵某<br/>签发时间, 过期时间<br/><br/>⚠️ 明文，随便看！<br/>绝不放密码！"]
    S["🔒 Signature<br/><br/>防伪指纹:<br/>f(Header+Payload, 我的密码)<br/><br/>← 这个没人能伪造<br/>只要他不知道密码"]
    H --> P --> S
```

---

## Payload 标准字段

| 字段 | 全称 | 含义 | 示例 |
|------|------|------|------|
| `sub` | Subject | 这个 Token 是谁的——通常写 user_id | `"1"` |
| `iat` | Issued At | 什么时候签发的 | `1719500000` |
| `exp` | Expiration | 什么时候过期 | `1719501800`（30分钟后） |
| `iss` | Issuer | 谁签发的——写你的应用名 | `"my-fastapi-app"` |
| `aud` | Audience | 谁能用这个 Token | `"my-frontend"` |
| `jti` | JWT ID | Token 唯一编号 | `随机uuid` |

> 实际只要记住三个：`sub`（用户）、`exp`（过期时间）、`iat`（签发时间）。其他项目阶段再加。

---

## Access + Refresh Token 设计

```text
只有一个 Token 的困境:
  有效期短（5 分钟）→ 用户每 5 分钟要重新登录
  有效期长（30 天）→ 一旦被偷，30 天内小偷都能冒充你

解决：发两张票，分工不同:

```mermaid
flowchart LR
    subgraph AT["Access Token（门禁卡）"]
        direction TB
        AT1["天天带、每次请求都用"]
        AT2["短命（15-30 分钟）"]
        AT3["丢了 → 短时间风险"]
        AT4["每次 API 请求带"]
    end

    subgraph RT["Refresh Token（保险柜密码）"]
        direction TB
        RT1["很少用、只存在客户端"]
        RT2["长命（7-30 天）"]
        RT3["丢了 → 账号被盗"]
        RT4["只在换卡时拿出来"]
    end
```text
```

工作流程：
1. 登录 → 服务器发 [Access Token, Refresh Token]
2. 正常用 → 每次请求带 Access Token
3. Access 过期 → 服务端返回 401
4. 客户端拿 Refresh Token 调 `/auth/refresh` → 拿到新 Access Token
5. Refresh 也过期 → 重新登录

> Access Token 是门禁卡——天天刷，丢了挂失就好。Refresh Token 是保险柜密码——存好了别动，只在换卡时才拿出来。

---

## 扩展话题

### HS256 vs RS256

HS256 笔记里用的叫"对称算法"——同一个密码既做签名又做验证。单体应用够了。

但如果以后拆微服务，认证服务签发的 Token 要给 API 服务验证——不能把密码发给 API 服务。这时候用 RS256——"非对称算法"：私钥签名 + 公钥验证。

```mermaid
flowchart LR
    subgraph HS["HS256（单体应用）"]
        direction TB
        HS_A["🔑 一把钥匙，自己用"]
        HS_B["SIGN_KEY = 'my-secret'"]
        HS_C["sign('内容', SIGN_KEY)"]
        HS_D["verify('签名', SIGN_KEY)"]
    end

    subgraph RS["RS256（微服务）"]
        direction TB
        RS_A["🔑🔑 两把钥匙"]
        RS_B["PRIVATE_KEY = '绝密，不出服务器'"]
        RS_C["PUBLIC_KEY = '谁要都给'"]
        RS_D["sign('内容', PRIVATE_KEY)"]
        RS_E["verify('签名', PUBLIC_KEY)"]
    end
```

### HTTPS 保证外层安全

JWT 在网络传输时可能被中间人截获——这靠 HTTPS 解决，不是 JWT 的职责。HTTPS 在传输层加密，JWT 在应用层做身份验证。各管各的。

---

## 相关链接

- 上一篇：[[02-aiohttp异步HTTP客户端]]
- 下一篇：[[04-FastAPI+JWT全链路实现]]
- 项目实践：[[笔记/技术学习/项目开发笔记/ai-resume笔记/01_技术研读/01_架构概览|ai-resume: 架构概览]]
- 项目实践：[[笔记/技术学习/项目开发笔记/cr-agent笔记/01-技术研读/10-安全加固四道防线|cr-agent: 安全加固四道防线]]

---
→ [[技术学习清单#JWT 鉴权]]

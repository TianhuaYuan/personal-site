---

title: "JWT 鉴权全链路"

created: "2026-07-20"

tags:

  - 八股文

  - fastapi-web

---

# JWT 鉴权全链路

## 一句话总结

> **JWT（JSON Web Token）鉴权全链路 = 登录验密码 → 用密钥签一张带过期时间的令牌 → 客户端每次请求在 `Authorization: Bearer` 头里带上 → 服务端验签名+过期，解码出用户，全程无状态、不查库。本质是“把身份信息装进一张防伪造的门票里”。**

---

## 生活类比：酒店房卡

你到前台（登录接口）出示身份证（用户名+密码），前台验证后给你一张**房卡（JWT）**：卡里写着房号（用户ID）、有效期（exp），并用前台私章（密钥签名）防伪。之后你刷每张门（每个接口），门禁自己验卡，不用每次跑前台。卡过期或被篡改，门禁直接拒。

## 什么是 JWT

JWT 是一个用 `.` 分隔的三段 Base64 字符串：`Header.Payload.Signature`

```mermaid

flowchart LR

    H["Header<br/>alg=HS256, typ=JWT"] --> P["Payload<br/>sub=用户ID, exp=过期时间"]

    H --> S["Signature<br/>HMACSHA256(base64(header)+'.'+base64(payload), 密钥)"]

    P --> S

```

- **Header**：算法（如 HS256）和令牌类型

- **Payload（载荷）**：声明（claims），如 `sub`（subject，主体=用户ID）、`exp`（expiration 过期时间）、`iat`（签发时间）

- **Signature**：用密钥对前两段签名，服务端据此验证“谁签的、有没有被改”

> JWT **不加密**（任何人都能 base64 解码看 payload），它只是**签名**防伪造。别在 payload 里放密码等机密。

## 全链路四步

```mermaid

sequenceDiagram

    participant C as 客户端

    participant S as 服务端

    C->>S: 1. POST /login (用户名+密码)

    S->>S: 2. 验密码(hash 比对)

    S-->>C: 3. 签发 JWT (sub+exp, 签名)

    C->>C: 4. 存 token (localStorage/cookie)

    C->>S: 5. 后续请求 Authorization: Bearer <token>

    S->>S: 6. 验签名+exp, 解码出用户

    S-->>C: 7. 返回受保护数据

```

## 手写最小可用全链路（⭐必考）

```python

import jwt

from datetime import datetime, timedelta, timezone

from fastapi import FastAPI, Depends, HTTPException, status

from fastapi.security import OAuth2PasswordBearer

from pydantic import BaseModel

SECRET_KEY = "换成 openssl rand -hex 32 生成的随机串"  # 真实项目放环境变量!

ALGORITHM = "HS256"

ACCESS_TOKEN_EXPIRE_MINUTES = 30

app = FastAPI()

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="login")  # 告诉 /docs 去哪拿 token

# ---- 1. 签发 ----

def create_access_token(data: dict, expires: timedelta | None = None) -> str:

    to_encode = data.copy()

    expire = datetime.now(timezone.utc) + (expires or timedelta(minutes=15))

    to_encode.update({"exp": expire})

    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

# ---- 2. 登录端点 (真实项目要验密码 hash) ----

@app.post("/login")

def login(username: str, password: str):

    # 伪代码: user = db 查用户; verify_password(password, user.hashed)

    if username != "alice" or password != "secret":

        raise HTTPException(401, "用户名或密码错误")

    token = create_access_token({"sub": username})

    return {"access_token": token, "token_type": "bearer"}

# ---- 3. 解码依赖 (被每个受保护路由复用) ----

def get_current_user(token: str = Depends(oauth2_scheme)) -> str:

    try:

        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])

        username = payload.get("sub")

        if not username:

            raise HTTPException(401, "无效令牌")

    except jwt.ExpiredSignatureError:

        raise HTTPException(401, "令牌已过期")

    except jwt.InvalidTokenError:

        raise HTTPException(401, "伪造/无效令牌")

    return username

# ---- 4. 受保护路由 ----

@app.get("/me")

def read_me(current_user: str = Depends(get_current_user)):

    return {"user": current_user}

```

> **OAuth2PasswordBearer** 是 FastAPI 提供的帮手，声明“令牌从哪获取”，让 `/docs` 自带 Authorize 按钮；**PyJWT**（`jwt` 库）负责编解码。**HS256** = HMAC-SHA256，对称签名算法。

## 密码千万别明文存

```python

from pwdlib import PasswordHash

pwd = PasswordHash.recommended()          # 默认 Argon2id

hashed = pwd.hash("明文密码")               # 存库的是 hash

pwd.verify("明文密码", hashed)              # 登录时比对, 返回 bool

```

> 官方现在推荐 **pwdlib + Argon2id**（内存硬哈希，抗 GPU 爆破），替代老的 `passlib+bcrypt`。**绝不存明文密码**。

## 刷新令牌（Refresh Token）

短命 access token（15–30 分）+ 长命 refresh token（天级）是常见搭配：access 过期后用 refresh 换新的，避免用户频繁登录，也缩小被盗窗口。

## JWT vs Session（必比）

| 维度 | JWT | Session（服务端会话） |
| :--- | :--- | :--- |
| 状态 | 无状态（令牌自包含） | 有状态（服务端存 session） |
| 注销/吊销 | 难（令牌到期前一直有效，需黑名单） | 易（删服务端记录即可） |
| 跨服务 | 友好（微服务/移动端通用） | 需共享 session 存储 |
| 性能 | 每次验签名（快，免查库） | 每次查 session 存储 |

## 安全红线

1. **SECRET_KEY 绝不硬编码**，用环境变量 + `openssl rand -hex 32` 生成

2. **access token 短命**，必要时配 refresh + 黑名单

3. **HTTPS 传输**，否则令牌被中间人截获

4. payload **不放机密**（它只是签名不是加密）

5. `alg=none` 攻击：服务端必须**固定算法**（不信任令牌里的 alg 字段）

## 记忆口诀

> **登录验密 → 签名发牌 → 请求带 Bearer → 验签解码出用户。**

> **JWT 三段：头.载荷.签名；签名防伪造，不加密。**

> **无状态靠自包含，吊销难靠短命+黑名单。**

> **密钥进环境变量，密码存 Argon2 hash。**

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

```

问题3: 服务器重启 → 所有 Session 全丢 → 全部重新登录

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

## 第一段：算法说明

print(decode(header_b64))

## 第二段：票上的信息（明文！）

print(decode(payload_b64))

## 第三段：指纹（乱码，无法还原为原始内容）

print(signature)

## → "HrWQ3KxLmN8pY2zF5vB6dC7sA1tG4jU9wE0oR3nL5xY"

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

---

---

## 依赖安装

```bash

pip install "python-jose[cryptography]" passlib[bcrypt] python-multipart

```

| 包 | 用途 |

|---|------|

| python-jose | JWT 签发 (encode) + 验证 (decode) |

| passlib | 密码哈希（bcrypt 算法） |

| python-multipart | FastAPI 解析 form-data（登录接口用） |

---

## 第一步：密码哈希 — 永远不存明文

```python

from passlib.context import CryptContext

## 创建密码上下文 — bcrypt 是业界标准

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

## 哈希密码 — 注册时用

plain_password = "my_secret_123"

hashed = pwd_context.hash(plain_password)

## 验证密码 — 登录时用

is_valid = pwd_context.verify("my_secret_123", hashed)   # → True

is_valid = pwd_context.verify("wrong_password", hashed)  # → False

```

```mermaid

flowchart LR

    subgraph BCRYPT["bcrypt 哈希结构"]

        A["$2b$<br/>算法标识"] --> B["12<br/>轮数 (2¹² 次迭代)<br/>越大越慢越安全"]

        B --> C["KIXxNcLxH6qQ5YqJqz6ZqeT8zYOzLf5G<br/>盐 (前 22 字符，每次随机生成)<br/>+ 哈希值 (后 31 字符)"]

    end

    style A fill:#1a1a2e,stroke:#e94560,color:#eee

    style B fill:#16213e,stroke:#e94560,color:#eee

    style C fill:#0f3460,stroke:#e94560,color:#eee

```

```text

为什么用 bcrypt 而不用 SHA256？

SHA256("password123") → 永远是同一个结果

  → 可以建彩虹表（大量常见密码的 SHA256 结果）

  → 数据库泄露后，查表秒破

bcrypt("password123") → 每次结果不同（随机盐）

  → 无法建彩虹表

  → 验证时 bcrypt 内部把盐取出来，用同样的盐哈希输入的密码再比较

  → 2^12 轮迭代，故意慢（~0.3s），暴力破解成本极高

```

---

## 第二步：JWT 签发 + 验证

```python

from datetime import datetime, timedelta, timezone

from jose import JWTError, jwt

from typing import Optional

## ============================================================

SECRET_KEY = "a3f8b2c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6"

ALGORITHM = "HS256"

ACCESS_TOKEN_EXPIRE_MINUTES = 30

REFRESH_TOKEN_EXPIRE_DAYS = 7

## ============================================================

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None) -> str:

    to_encode = data.copy()

    if expires_delta:

        expire = datetime.now(timezone.utc) + expires_delta

    else:

        expire = datetime.now(timezone.utc) + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)

    to_encode.update({"exp": expire, "iat": datetime.now(timezone.utc)})

    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

    return encoded_jwt

## ============================================================

def verify_access_token(token: str) -> dict:

    try:

        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])

        return payload

    except JWTError:

        raise

## ============================================================

def create_refresh_token(data: dict) -> str:

    to_encode = data.copy()

    expire = datetime.now(timezone.utc) + timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS)

    to_encode.update({

        "exp": expire,

        "iat": datetime.now(timezone.utc),

        "type": "refresh",

    })

    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

```

```mermaid

flowchart TB

    subgraph ENC["签发 (encode)"]

        direction LR

        E1["payload + secret_key"] --> E2["signature"]

        E2 --> E3["拼成 header.payload.signature<br/>→ 返回 token 字符串"]

    end

    subgraph DEC["验证 (decode)"]

        DEC_T["token 字符串"] --> DEC_A["解码 header → 拿到 alg"]

        DEC_T --> DEC_B["解码 payload → 检查 exp 是否过期"]

        DEC_T --> DEC_C["用同样的 secret_key 重新算签名 → 比对"]

        DEC_C -->|匹配| DEC_OK["✅ 返回 payload"]

        DEC_C -->|不匹配| DEC_FAIL["❌ 抛 JWTError（被篡改过！）"]

    end

```

---

## 第三步：FastAPI 依赖注入

```python

from fastapi import Depends, HTTPException, status

from fastapi.security import OAuth2PasswordBearer

## 定义 Token 提取器 — 从请求头 Authorization: Bearer <token> 拔 token

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")

async def get_current_user(token: str = Depends(oauth2_scheme)) -> dict:

    credentials_exception = HTTPException(

        status_code=status.HTTP_401_UNAUTHORIZED,

        detail="无法验证凭据",

        headers={"WWW-Authenticate": "Bearer"},

    )

    try:

        payload = verify_access_token(token)

        user_id: str = payload.get("sub")

        if user_id is None:

            raise credentials_exception

    except JWTError:

        raise credentials_exception

    return {"user_id": int(user_id)}

## 在路由中使用

@app.get("/me")

async def read_self(current_user: dict = Depends(get_current_user)):

    return {"user_id": current_user["user_id"]}

```

```mermaid

flowchart TB

    A["请求: GET /me<br/>Header: Authorization: Bearer ey..."]

    A --> B["OAuth2PasswordBearer<br/>从请求头拔 token 字符串"]

    B --> C["get_current_user(token=token)"]

    C --> D["verify_access_token(token) → payload"]

    D --> E{"签名通过?"}

    E -->|"是"| F["继续"]

    E -->|"过期/伪造"| G["raise 401"]

    C --> H["return {'user_id': 37}"]

    F --> H

    H --> I["read_self(current_user={'user_id': 37})<br/>→ 返回 {'user_id': 37}"]

```

> [!note] OAuth2PasswordRequestForm

> 登录接口接收的是 form-data 格式（x-www-form-urlencoded），不是 JSON。这是 FastAPI OAuth2 的默认规范。

---

## 第四步：注册 + 登录 + 刷新 — 完整路由

```python

from fastapi import FastAPI, Depends, HTTPException, status

from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm

from pydantic import BaseModel

from datetime import timedelta

app = FastAPI(title="JWT 鉴权 Demo")

## ============================================================

fake_users_db: dict[str, dict] = {}

## ============================================================

class UserRegister(BaseModel):

    username: str

    password: str

class UserOut(BaseModel):

    id: int

    username: str

class TokenResponse(BaseModel):

    access_token: str

    refresh_token: str

    token_type: str = "bearer"

class RefreshRequest(BaseModel):

    refresh_token: str

## ============================================================

@app.post("/auth/register", response_model=UserOut)

async def register(data: UserRegister):

    if data.username in fake_users_db:

        raise HTTPException(status_code=400, detail="用户名已存在")

    hashed_password = pwd_context.hash(data.password)

    user_id = len(fake_users_db) + 1

    fake_users_db[data.username] = {

        "id": user_id,

        "username": data.username,

        "hashed_password": hashed_password,

    }

    return {"id": user_id, "username": data.username}

## ============================================================

@app.post("/auth/login", response_model=TokenResponse)

async def login(form_data: OAuth2PasswordRequestForm = Depends()):

    user = fake_users_db.get(form_data.username)

    if not user:

        raise HTTPException(status_code=401, detail="用户名或密码错误")

    if not pwd_context.verify(form_data.password, user["hashed_password"]):

        raise HTTPException(status_code=401, detail="用户名或密码错误")

    access_token = create_access_token(data={"sub": str(user["id"])})

    refresh_token = create_refresh_token(data={"sub": str(user["id"])})

    return {

        "access_token": access_token,

        "refresh_token": refresh_token,

        "token_type": "bearer",

    }

## ============================================================

@app.post("/auth/refresh", response_model=TokenResponse)

async def refresh_access_token(data: RefreshRequest):

    try:

        payload = verify_access_token(data.refresh_token)

    except JWTError:

        raise HTTPException(status_code=401, detail="Refresh Token 无效或过期")

    if payload.get("type") != "refresh":

        raise HTTPException(status_code=401, detail="非法 Token 类型")

    user_id = payload.get("sub")

    new_access = create_access_token(data={"sub": user_id})

    new_refresh = create_refresh_token(data={"sub": user_id})

    return {

        "access_token": new_access,

        "refresh_token": new_refresh,

        "token_type": "bearer",

    }

## ============================================================

@app.get("/me", response_model=UserOut)

async def read_self(current_user: dict = Depends(get_current_user)):

    user_id = current_user["user_id"]

    for username, user in fake_users_db.items():

        if user["id"] == user_id:

            return {"id": user["id"], "username": user["username"]}

    raise HTTPException(status_code=404, detail="用户不存在")

```

---

## 完整鉴权链路

```mermaid

sequenceDiagram

    participant 浏览器

    participant FastAPI

    Note over 浏览器,FastAPI: 注册

    浏览器->>FastAPI: POST /auth/register {username, password}

    FastAPI->>FastAPI: 检查用户名唯一性

    FastAPI->>FastAPI: passlib.hash(password) -> 存哈希

    FastAPI-->>浏览器: 返回 {"id":1, "username":"alice"}

    Note over 浏览器,FastAPI: 登录

    浏览器->>FastAPI: POST /auth/login (form-data)

    FastAPI->>FastAPI: 查用户 -> passlib.verify(明文, 哈希) -> 匹配

    FastAPI->>FastAPI: 签发 Access Token (30min) + Refresh Token (7天)

    FastAPI-->>浏览器: 返回 Tokens

    Note over 浏览器,FastAPI: 请求受保护接口

    浏览器->>FastAPI: GET /me, Authorization: Bearer 〈access_token〉

    FastAPI->>FastAPI: OAuth2PasswordBearer 拔 token

    FastAPI->>FastAPI: get_current_user -> verify_access_token

    FastAPI->>FastAPI: jwt.decode -> 检查 exp -> 检查签名

    FastAPI-->>浏览器: 返回用户数据

    Note over 浏览器,FastAPI: Access Token 过期

    浏览器->>FastAPI: POST /auth/refresh (带 Refresh Token)

    FastAPI->>FastAPI: 验证签名 + 类型

    FastAPI->>FastAPI: 签发新 Access + Refresh Token（滚动刷新）

    FastAPI-->>浏览器: 新 Tokens

    浏览器->>浏览器: 更新存储，重试原请求

```

---

## 速查
### JWT 最小骨架

```python

## 配置

SECRET_KEY = os.environ["JWT_SECRET"]

ALGORITHM = "HS256"

## 签发

def create_token(user_id: int, minutes: int = 30) -> str:

    expire = datetime.now(timezone.utc) + timedelta(minutes=minutes)

    return jwt.encode(

        {"sub": str(user_id), "exp": expire, "iat": datetime.now(timezone.utc)},

        SECRET_KEY, algorithm=ALGORITHM

    )

## 验证

def verify_token(token: str) -> dict:

    return jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])

## FastAPI 依赖注入

oauth2 = OAuth2PasswordBearer(tokenUrl="/auth/login")

async def get_user(token: str = Depends(oauth2)) -> dict:

    try:

        return verify_token(token)

    except JWTError:

        raise HTTPException(401, "无效凭据")

@app.get("/me")

async def me(user: dict = Depends(get_user)):

    return user

```

### 密码处理

```python

from passlib.context import CryptContext

pwd = CryptContext(schemes=["bcrypt"])

hashed = pwd.hash("mypassword")            # 注册

valid = pwd.verify("mypassword", hashed)   # 登录

```

---

---

---

单文件完整实现 JWT 鉴权系统 + aiohttp 并发压测。刻意不用 `Depends(get_current_user)` 的简化方式，手写每个环节看清 token 如何在 HTTP 里传递。

---

## 完整代码

```python

"""

JWT 全链路 + asyncio 并发测试。

第一部分：JWT 鉴权全链路

  注册 → 登录(签发双Token) → 访问保护接口(携带Access Token)

  → Access过期 → 调Refresh换新Token

第二部分：asyncio 并发测试

  用 aiohttp 模拟多用户同时注册→登录→访问

"""

import os

import time as _time

from datetime import datetime, timedelta, timezone

from typing import Optional

from dotenv import load_dotenv

from fastapi import FastAPI, HTTPException, Depends, status, Request

from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm

from fastapi.responses import JSONResponse

from passlib.context import CryptContext

from jose import JWTError, jwt

from pydantic import BaseModel

import uvicorn

load_dotenv()

## ============================================================

SECRET_KEY = os.environ.get("JWT_SECRET", "dev-secret-change-in-production-!!!")

ALGORITHM = "HS256"

ACCESS_TOKEN_EXPIRE_MINUTES = 30

REFRESH_TOKEN_EXPIRE_DAYS = 7

## ============================================================

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

## ============================================================

class FakeDB:

    def __init__(self):

        self.users: dict[str, dict] = {}

        self._next_id = 1

    def create(self, username: str, hashed_password: str) -> dict:

        if username in self.users:

            return None

        user = {"id": self._next_id, "username": username, "hashed_password": hashed_password}

        self.users[username] = user

        self._next_id += 1

        return user

    def get_by_username(self, username: str) -> Optional[dict]:

        return self.users.get(username)

    def get_by_id(self, user_id: int) -> Optional[dict]:

        for user in self.users.values():

            if user["id"] == user_id:

                return user

        return None

db = FakeDB()

## ============================================================

class RegisterRequest(BaseModel):

    username: str

    password: str

class UserResponse(BaseModel):

    id: int

    username: str

class TokenResponse(BaseModel):

    access_token: str

    refresh_token: str

    token_type: str = "bearer"

class RefreshRequest(BaseModel):

    refresh_token: str

## ============================================================

def _make_token(data: dict, expires_delta: timedelta, token_type: str = "access") -> str:

    to_encode = data.copy()

    now = datetime.now(timezone.utc)

    to_encode.update({

        "iat": now,

        "exp": now + expires_delta,

        "type": token_type,

    })

    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

def create_access_token(user_id: int) -> str:

    return _make_token(

        {"sub": str(user_id)},

        timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES),

        token_type="access",

    )

def create_refresh_token(user_id: int) -> str:

    return _make_token(

        {"sub": str(user_id)},

        timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS),

        token_type="refresh",

    )

def verify_token(token: str, expected_type: Optional[str] = None) -> dict:

    payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])

    if expected_type and payload.get("type") != expected_type:

        raise JWTError(f"Token 类型不匹配，期望 {expected_type}")

    return payload

## ============================================================

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login", auto_error=False)

async def get_current_user(token: str = Depends(oauth2_scheme)) -> dict:

    if not token:

        raise HTTPException(status_code=401, detail="未提供 Token")

    try:

        payload = verify_token(token, expected_type="access")

    except JWTError:

        raise HTTPException(status_code=401, detail="Token 无效或过期")

    user_id = int(payload["sub"])

    user = db.get_by_id(user_id)

    if not user:

        raise HTTPException(status_code=404, detail="用户不存在")

    return {"user_id": user["id"], "username": user["username"]}

## ============================================================

app = FastAPI(title="JWT + Asyncio 实战")

@app.post("/auth/register", response_model=UserResponse)

async def register(data: RegisterRequest):

    if db.get_by_username(data.username):

        raise HTTPException(status_code=400, detail="用户名已存在")

    hashed = pwd_context.hash(data.password)

    user = db.create(data.username, hashed)

    if not user:

        raise HTTPException(status_code=500, detail="注册失败")

    return {"id": user["id"], "username": user["username"]}

@app.post("/auth/login", response_model=TokenResponse)

async def login(form_data: OAuth2PasswordRequestForm = Depends()):

    user = db.get_by_username(form_data.username)

    if not user:

        raise HTTPException(status_code=401, detail="用户名或密码错误")

    if not pwd_context.verify(form_data.password, user["hashed_password"]):

        raise HTTPException(status_code=401, detail="用户名或密码错误")

    access = create_access_token(user["id"])

    refresh = create_refresh_token(user["id"])

    return {

        "access_token": access,

        "refresh_token": refresh,

        "token_type": "bearer",

    }

@app.post("/auth/refresh", response_model=TokenResponse)

async def refresh(data: RefreshRequest):

    try:

        payload = verify_token(data.refresh_token)

        if payload.get("type") != "refresh":

            raise HTTPException(status_code=401, detail="非法 Token 类型")

        user_id = int(payload["sub"])

        new_access = create_access_token(user_id)

        new_refresh = create_refresh_token(user_id)

        return {

            "access_token": new_access,

            "refresh_token": new_refresh,

            "token_type": "bearer",

        }

    except JWTError:

        raise HTTPException(status_code=401, detail="Refresh Token 无效或过期")

@app.get("/me", response_model=UserResponse)

async def read_me(current_user: dict = Depends(get_current_user)):

    return current_user

@app.get("/public")

async def public():

    return {"message": "Hello, 这是公开接口"}

## ============================================================

@app.exception_handler(HTTPException)

async def http_exception_handler(request: Request, exc: HTTPException):

    return JSONResponse(

        status_code=exc.status_code,

        content={"detail": exc.detail},

    )

```

---

## 测试 1：JWT 全链路（5 用户）

```python

async def test_jwt_full_chain():

    """

    模拟 5 个用户的完整生命周期：

    注册 → 登录(拿到双Token) → 访问/me(带Access Token)

    → 用 Refresh Token 换新 Access → 再次访问 /me

    """

    import aiohttp

    BASE = "http://127.0.0.1:8000"

    async with aiohttp.ClientSession() as session:

        print("=" * 60)

        print("JWT 全链路并发测试 (5 个用户)")

        print("=" * 60)

        test_users = [

            {"username": f"user_{i}", "password": f"pass_{i}"}

            for i in range(1, 6)

        ]

        for user in test_users:

            print(f"\n── {user['username']} ──")

            # ① 注册

            async with session.post(f"{BASE}/auth/register", json=user) as resp:

                if resp.status == 200:

                    reg = await resp.json()

                    print(f"  ① 注册成功: id={reg['id']}, username={reg['username']}")

                elif resp.status == 400:

                    print(f"  ① 已注册（跳过）")

                else:

                    print(f"  ① 注册失败: {resp.status} {await resp.text()}")

            # ② 登录（OAuth2PasswordRequestForm 需要 form-data，不是 JSON）

            form = aiohttp.FormData()

            form.add_field("username", user["username"])

            form.add_field("password", user["password"])

            async with session.post(f"{BASE}/auth/login", data=form) as resp:

                if resp.status != 200:

                    print(f"  ② 登录失败: {resp.status} {await resp.text()}")

                    continue

                tokens = await resp.json()

                access = tokens["access_token"]

                refresh = tokens["refresh_token"]

                print(f"  ② 登录成功: access=...{access[-20:]}, refresh=...{refresh[-20:]}")

            # ③ 访问 /me（带 Access Token）

            headers = {"Authorization": f"Bearer {access}"}

            async with session.get(f"{BASE}/me", headers=headers) as resp:

                me = await resp.json()

                print(f"  ③ /me 返回: {me}")

            # ④ 刷新 Token

            async with session.post(f"{BASE}/auth/refresh", json={"refresh_token": refresh}) as resp:

                new_tokens = await resp.json()

                new_access = new_tokens["access_token"]

                print(f"  ④ 刷新成功: new_access=...{new_access[-20:]}")

            # ⑤ 用新 Access Token 再访问 /me

            new_headers = {"Authorization": f"Bearer {new_access}"}

            async with session.get(f"{BASE}/me", headers=new_headers) as resp:

                me2 = await resp.json()

                print(f"  ⑤ 新 Token 访问 /me: {me2}")

        # ⑥ 测试不带 Token → 应返回 401

        print(f"\n── 异常测试 ──")

        async with session.get(f"{BASE}/me") as resp:

            print(f"  无 Token 访问 /me: {resp.status} {await resp.json()}")

        # ⑦ 测试错误的 Token

        bad_headers = {"Authorization": "Bearer this.is.fake"}

        async with session.get(f"{BASE}/me", headers=bad_headers) as resp:

            print(f"  假 Token 访问 /me: {resp.status} {await resp.json()}")

```

---

## 测试 2：20 用户并发压测

```python

async def test_concurrent_users():

    """

    模拟 20 个用户同时注册→登录→访问 /me。

    验证 asyncio 并发能力 + JWT 系统在并发下无竞态。

    """

    import aiohttp

    BASE = "http://127.0.0.1:8000"

    N = 20

    async with aiohttp.ClientSession() as session:

        async def one_user_lifecycle(i: int) -> dict:

            username = f"cuser_{i}"

            password = f"cpass_{i}"

            log = {"user": username}

            # 注册

            async with session.post(f"{BASE}/auth/register", json={"username": username, "password": password}) as resp:

                log["register"] = resp.status

            # 登录

            form = aiohttp.FormData()

            form.add_field("username", username)

            form.add_field("password", password)

            async with session.post(f"{BASE}/auth/login", data=form) as resp:

                if resp.status != 200:

                    log["login"] = f"FAIL({resp.status})"

                    return log

                tokens = await resp.json()

                log["login"] = "OK"

            # 访问 /me

            headers = {"Authorization": f"Bearer {tokens['access_token']}"}

            async with session.get(f"{BASE}/me", headers=headers) as resp:

                log["me"] = resp.status

            return log

        print("\n" + "=" * 60)

        print(f"并发测试: {N} 个用户同时注册→登录→访问")

        print("=" * 60)

        t0 = _time.time()

        results = await asyncio.gather(*[

            one_user_lifecycle(i) for i in range(1, N + 1)

        ])

        elapsed = _time.time() - t0

        ok_count = sum(1 for r in results if r.get("login") == "OK")

        print(f"\n结果: {ok_count}/{N} 成功, 耗时 {elapsed:.2f}s")

        print(f"平均每个用户: {elapsed/N*1000:.0f}ms")

```

---

## 启动与运行

```bash

## 终端1：启动服务

python main.py

## 终端2：跑测试

python main.py --test

```

### 测试输出示例

```text

============================================================

JWT 全链路并发测试 (5 个用户)

============================================================

── user_1 ──

  ① 注册成功: id=1, username=user_1

  ② 登录成功: access=...SflKxwRJSMeKKF2QT4f, refresh=...ZyJIUzI1NiIsInR5c

  ③ /me 返回: {'id': 1, 'username': 'user_1'}

  ④ 刷新成功: new_access=...8d9e0f1a2b3c4d5e6f7

  ⑤ 新 Token 访问 /me: {'id': 1, 'username': 'user_1'}

...

── 异常测试 ──

  无 Token 访问 /me: 401 {'detail': '未提供 Token'}

  假 Token 访问 /me: 401 {'detail': 'Token 无效或过期'}

============================================================

并发测试: 20 个用户同时注册→登录→访问

============================================================

结果: 20/20 成功, 耗时 2.34s

平均每个用户: 117ms

```

### 入口代码

```python

if __name__ == "__main__":

    import sys

    if len(sys.argv) > 1 and sys.argv[1] == "--test":

        async def run_tests():

            await test_jwt_full_chain()

            await test_concurrent_users()

        asyncio.run(run_tests())

    else:

        uvicorn.run(app, host="127.0.0.1", port=8000)

```

---

---

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

```

问题3: 服务器重启 → 所有 Session 全丢 → 全部重新登录

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

# 第二段：票上的信息（明文！）

print(decode(payload_b64))

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

---

---

## 依赖安装

```bash

pip install "python-jose[cryptography]" passlib[bcrypt] python-multipart

```

| 包 | 用途 |
|---|------|
| python-jose | JWT 签发 (encode) + 验证 (decode) |
| passlib | 密码哈希（bcrypt 算法） |
| python-multipart | FastAPI 解析 form-data（登录接口用） |

---

## 第一步：密码哈希 — 永远不存明文

```python

from passlib.context import CryptContext

# 创建密码上下文 — bcrypt 是业界标准

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# 哈希密码 — 注册时用

plain_password = "my_secret_123"

hashed = pwd_context.hash(plain_password)

# 验证密码 — 登录时用

is_valid = pwd_context.verify("my_secret_123", hashed)   # → True

is_valid = pwd_context.verify("wrong_password", hashed)  # → False

```

```mermaid

flowchart LR

    subgraph BCRYPT["bcrypt 哈希结构"]

        A["$2b$<br/>算法标识"] --> B["12<br/>轮数 (2¹² 次迭代)<br/>越大越慢越安全"]

        B --> C["KIXxNcLxH6qQ5YqJqz6ZqeT8zYOzLf5G<br/>盐 (前 22 字符，每次随机生成)<br/>+ 哈希值 (后 31 字符)"]

    end

    style A fill:#1a1a2e,stroke:#e94560,color:#eee

    style B fill:#16213e,stroke:#e94560,color:#eee

    style C fill:#0f3460,stroke:#e94560,color:#eee

```

```text

为什么用 bcrypt 而不用 SHA256？

SHA256("password123") → 永远是同一个结果

  → 可以建彩虹表（大量常见密码的 SHA256 结果）

  → 数据库泄露后，查表秒破

bcrypt("password123") → 每次结果不同（随机盐）

  → 无法建彩虹表

  → 验证时 bcrypt 内部把盐取出来，用同样的盐哈希输入的密码再比较

  → 2^12 轮迭代，故意慢（~0.3s），暴力破解成本极高

```

---

## 第二步：JWT 签发 + 验证

```python

from datetime import datetime, timedelta, timezone

from jose import JWTError, jwt

from typing import Optional

# ============================================================

SECRET_KEY = "a3f8b2c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6"

ALGORITHM = "HS256"

ACCESS_TOKEN_EXPIRE_MINUTES = 30

REFRESH_TOKEN_EXPIRE_DAYS = 7

# ============================================================

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None) -> str:

    to_encode = data.copy()

    if expires_delta:

        expire = datetime.now(timezone.utc) + expires_delta

    else:

        expire = datetime.now(timezone.utc) + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)

    to_encode.update({"exp": expire, "iat": datetime.now(timezone.utc)})

    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

    return encoded_jwt

# ============================================================

def verify_access_token(token: str) -> dict:

    try:

        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])

        return payload

    except JWTError:

        raise

# ============================================================

def create_refresh_token(data: dict) -> str:

    to_encode = data.copy()

    expire = datetime.now(timezone.utc) + timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS)

    to_encode.update({

        "exp": expire,

        "iat": datetime.now(timezone.utc),

        "type": "refresh",

    })

    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

```

```mermaid

flowchart TB

    subgraph ENC["签发 (encode)"]

        direction LR

        E1["payload + secret_key"] --> E2["signature"]

        E2 --> E3["拼成 header.payload.signature<br/>→ 返回 token 字符串"]

    end

    subgraph DEC["验证 (decode)"]

        DEC_T["token 字符串"] --> DEC_A["解码 header → 拿到 alg"]

        DEC_T --> DEC_B["解码 payload → 检查 exp 是否过期"]

        DEC_T --> DEC_C["用同样的 secret_key 重新算签名 → 比对"]

        DEC_C -->|匹配| DEC_OK["✅ 返回 payload"]

        DEC_C -->|不匹配| DEC_FAIL["❌ 抛 JWTError（被篡改过！）"]

    end

```

---

## 第三步：FastAPI 依赖注入

```python

from fastapi import Depends, HTTPException, status

from fastapi.security import OAuth2PasswordBearer

# 定义 Token 提取器 — 从请求头 Authorization: Bearer <token> 拔 token

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")

async def get_current_user(token: str = Depends(oauth2_scheme)) -> dict:

    credentials_exception = HTTPException(

        status_code=status.HTTP_401_UNAUTHORIZED,

        detail="无法验证凭据",

        headers={"WWW-Authenticate": "Bearer"},

    )

    try:

        payload = verify_access_token(token)

        user_id: str = payload.get("sub")

        if user_id is None:

            raise credentials_exception

    except JWTError:

        raise credentials_exception

    return {"user_id": int(user_id)}

# 在路由中使用

@app.get("/me")

async def read_self(current_user: dict = Depends(get_current_user)):

    return {"user_id": current_user["user_id"]}

```

```mermaid

flowchart TB

    A["请求: GET /me<br/>Header: Authorization: Bearer ey..."]

    A --> B["OAuth2PasswordBearer<br/>从请求头拔 token 字符串"]

    B --> C["get_current_user(token=token)"]

    C --> D["verify_access_token(token) → payload"]

    D --> E{"签名通过?"}

    E -->|"是"| F["继续"]

    E -->|"过期/伪造"| G["raise 401"]

    C --> H["return {'user_id': 37}"]

    F --> H

    H --> I["read_self(current_user={'user_id': 37})<br/>→ 返回 {'user_id': 37}"]

```

> [!note] OAuth2PasswordRequestForm

> 登录接口接收的是 form-data 格式（x-www-form-urlencoded），不是 JSON。这是 FastAPI OAuth2 的默认规范。

---

## 第四步：注册 + 登录 + 刷新 — 完整路由

```python

from fastapi import FastAPI, Depends, HTTPException, status

from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm

from pydantic import BaseModel

from datetime import timedelta

app = FastAPI(title="JWT 鉴权 Demo")

# ============================================================

fake_users_db: dict[str, dict] = {}

# ============================================================

class UserRegister(BaseModel):

    username: str

    password: str

class UserOut(BaseModel):

    id: int

    username: str

class TokenResponse(BaseModel):

    access_token: str

    refresh_token: str

    token_type: str = "bearer"

class RefreshRequest(BaseModel):

    refresh_token: str

# ============================================================

@app.post("/auth/register", response_model=UserOut)

async def register(data: UserRegister):

    if data.username in fake_users_db:

        raise HTTPException(status_code=400, detail="用户名已存在")

    hashed_password = pwd_context.hash(data.password)

    user_id = len(fake_users_db) + 1

    fake_users_db[data.username] = {

        "id": user_id,

        "username": data.username,

        "hashed_password": hashed_password,

    }

    return {"id": user_id, "username": data.username}

# ============================================================

@app.post("/auth/login", response_model=TokenResponse)

async def login(form_data: OAuth2PasswordRequestForm = Depends()):

    user = fake_users_db.get(form_data.username)

    if not user:

        raise HTTPException(status_code=401, detail="用户名或密码错误")

    if not pwd_context.verify(form_data.password, user["hashed_password"]):

        raise HTTPException(status_code=401, detail="用户名或密码错误")

    access_token = create_access_token(data={"sub": str(user["id"])})

    refresh_token = create_refresh_token(data={"sub": str(user["id"])})

    return {

        "access_token": access_token,

        "refresh_token": refresh_token,

        "token_type": "bearer",

    }

# ============================================================

@app.post("/auth/refresh", response_model=TokenResponse)

async def refresh_access_token(data: RefreshRequest):

    try:

        payload = verify_access_token(data.refresh_token)

    except JWTError:

        raise HTTPException(status_code=401, detail="Refresh Token 无效或过期")

    if payload.get("type") != "refresh":

        raise HTTPException(status_code=401, detail="非法 Token 类型")

    user_id = payload.get("sub")

    new_access = create_access_token(data={"sub": user_id})

    new_refresh = create_refresh_token(data={"sub": user_id})

    return {

        "access_token": new_access,

        "refresh_token": new_refresh,

        "token_type": "bearer",

    }

# ============================================================

@app.get("/me", response_model=UserOut)

async def read_self(current_user: dict = Depends(get_current_user)):

    user_id = current_user["user_id"]

    for username, user in fake_users_db.items():

        if user["id"] == user_id:

            return {"id": user["id"], "username": user["username"]}

    raise HTTPException(status_code=404, detail="用户不存在")

```

---

## 完整鉴权链路

```mermaid

sequenceDiagram

    participant 浏览器

    participant FastAPI

    Note over 浏览器,FastAPI: 注册

    浏览器->>FastAPI: POST /auth/register {username, password}

    FastAPI->>FastAPI: 检查用户名唯一性

    FastAPI->>FastAPI: passlib.hash(password) -> 存哈希

    FastAPI-->>浏览器: 返回 {"id":1, "username":"alice"}

    Note over 浏览器,FastAPI: 登录

    浏览器->>FastAPI: POST /auth/login (form-data)

    FastAPI->>FastAPI: 查用户 -> passlib.verify(明文, 哈希) -> 匹配

    FastAPI->>FastAPI: 签发 Access Token (30min) + Refresh Token (7天)

    FastAPI-->>浏览器: 返回 Tokens

    Note over 浏览器,FastAPI: 请求受保护接口

    浏览器->>FastAPI: GET /me, Authorization: Bearer 〈access_token〉

    FastAPI->>FastAPI: OAuth2PasswordBearer 拔 token

    FastAPI->>FastAPI: get_current_user -> verify_access_token

    FastAPI->>FastAPI: jwt.decode -> 检查 exp -> 检查签名

    FastAPI-->>浏览器: 返回用户数据

    Note over 浏览器,FastAPI: Access Token 过期

    浏览器->>FastAPI: POST /auth/refresh (带 Refresh Token)

    FastAPI->>FastAPI: 验证签名 + 类型

    FastAPI->>FastAPI: 签发新 Access + Refresh Token（滚动刷新）

    FastAPI-->>浏览器: 新 Tokens

    浏览器->>浏览器: 更新存储，重试原请求

```

---

### JWT 最小骨架

```python

# 配置

SECRET_KEY = os.environ["JWT_SECRET"]

ALGORITHM = "HS256"

# 签发

def create_token(user_id: int, minutes: int = 30) -> str:

    expire = datetime.now(timezone.utc) + timedelta(minutes=minutes)

    return jwt.encode(

        {"sub": str(user_id), "exp": expire, "iat": datetime.now(timezone.utc)},

        SECRET_KEY, algorithm=ALGORITHM

    )

# 验证

def verify_token(token: str) -> dict:

    return jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])

# FastAPI 依赖注入

oauth2 = OAuth2PasswordBearer(tokenUrl="/auth/login")

async def get_user(token: str = Depends(oauth2)) -> dict:

    try:

        return verify_token(token)

    except JWTError:

        raise HTTPException(401, "无效凭据")

@app.get("/me")

async def me(user: dict = Depends(get_user)):

    return user

```

### 密码处理

```python

from passlib.context import CryptContext

pwd = CryptContext(schemes=["bcrypt"])

hashed = pwd.hash("mypassword")            # 注册

valid = pwd.verify("mypassword", hashed)   # 登录

```

---

---

---

单文件完整实现 JWT 鉴权系统 + aiohttp 并发压测。刻意不用 `Depends(get_current_user)` 的简化方式，手写每个环节看清 token 如何在 HTTP 里传递。

---

## 完整代码

```python

"""

JWT 全链路 + asyncio 并发测试。

第一部分：JWT 鉴权全链路

  注册 → 登录(签发双Token) → 访问保护接口(携带Access Token)

  → Access过期 → 调Refresh换新Token

第二部分：asyncio 并发测试

  用 aiohttp 模拟多用户同时注册→登录→访问

"""

import os

import time as _time

from datetime import datetime, timedelta, timezone

from typing import Optional

from dotenv import load_dotenv

from fastapi import FastAPI, HTTPException, Depends, status, Request

from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm

from fastapi.responses import JSONResponse

from passlib.context import CryptContext

from jose import JWTError, jwt

from pydantic import BaseModel

import uvicorn

load_dotenv()

# ============================================================

SECRET_KEY = os.environ.get("JWT_SECRET", "dev-secret-change-in-production-!!!")

ALGORITHM = "HS256"

ACCESS_TOKEN_EXPIRE_MINUTES = 30

REFRESH_TOKEN_EXPIRE_DAYS = 7

# ============================================================

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# ============================================================

class FakeDB:

    def __init__(self):

        self.users: dict[str, dict] = {}

        self._next_id = 1

    def create(self, username: str, hashed_password: str) -> dict:

        if username in self.users:

            return None

        user = {"id": self._next_id, "username": username, "hashed_password": hashed_password}

        self.users[username] = user

        self._next_id += 1

        return user

    def get_by_username(self, username: str) -> Optional[dict]:

        return self.users.get(username)

    def get_by_id(self, user_id: int) -> Optional[dict]:

        for user in self.users.values():

            if user["id"] == user_id:

                return user

        return None

db = FakeDB()

# ============================================================

class RegisterRequest(BaseModel):

    username: str

    password: str

class UserResponse(BaseModel):

    id: int

    username: str

class TokenResponse(BaseModel):

    access_token: str

    refresh_token: str

    token_type: str = "bearer"

class RefreshRequest(BaseModel):

    refresh_token: str

# ============================================================

def _make_token(data: dict, expires_delta: timedelta, token_type: str = "access") -> str:

    to_encode = data.copy()

    now = datetime.now(timezone.utc)

    to_encode.update({

        "iat": now,

        "exp": now + expires_delta,

        "type": token_type,

    })

    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

def create_access_token(user_id: int) -> str:

    return _make_token(

        {"sub": str(user_id)},

        timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES),

        token_type="access",

    )

def create_refresh_token(user_id: int) -> str:

    return _make_token(

        {"sub": str(user_id)},

        timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS),

        token_type="refresh",

    )

def verify_token(token: str, expected_type: Optional[str] = None) -> dict:

    payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])

    if expected_type and payload.get("type") != expected_type:

        raise JWTError(f"Token 类型不匹配，期望 {expected_type}")

    return payload

# ============================================================

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login", auto_error=False)

async def get_current_user(token: str = Depends(oauth2_scheme)) -> dict:

    if not token:

        raise HTTPException(status_code=401, detail="未提供 Token")

    try:

        payload = verify_token(token, expected_type="access")

    except JWTError:

        raise HTTPException(status_code=401, detail="Token 无效或过期")

    user_id = int(payload["sub"])

    user = db.get_by_id(user_id)

    if not user:

        raise HTTPException(status_code=404, detail="用户不存在")

    return {"user_id": user["id"], "username": user["username"]}

# ============================================================

app = FastAPI(title="JWT + Asyncio 实战")

@app.post("/auth/register", response_model=UserResponse)

async def register(data: RegisterRequest):

    if db.get_by_username(data.username):

        raise HTTPException(status_code=400, detail="用户名已存在")

    hashed = pwd_context.hash(data.password)

    user = db.create(data.username, hashed)

    if not user:

        raise HTTPException(status_code=500, detail="注册失败")

    return {"id": user["id"], "username": user["username"]}

@app.post("/auth/login", response_model=TokenResponse)

async def login(form_data: OAuth2PasswordRequestForm = Depends()):

    user = db.get_by_username(form_data.username)

    if not user:

        raise HTTPException(status_code=401, detail="用户名或密码错误")

    if not pwd_context.verify(form_data.password, user["hashed_password"]):

        raise HTTPException(status_code=401, detail="用户名或密码错误")

    access = create_access_token(user["id"])

    refresh = create_refresh_token(user["id"])

    return {

        "access_token": access,

        "refresh_token": refresh,

        "token_type": "bearer",

    }

@app.post("/auth/refresh", response_model=TokenResponse)

async def refresh(data: RefreshRequest):

    try:

        payload = verify_token(data.refresh_token)

        if payload.get("type") != "refresh":

            raise HTTPException(status_code=401, detail="非法 Token 类型")

        user_id = int(payload["sub"])

        new_access = create_access_token(user_id)

        new_refresh = create_refresh_token(user_id)

        return {

            "access_token": new_access,

            "refresh_token": new_refresh,

            "token_type": "bearer",

        }

    except JWTError:

        raise HTTPException(status_code=401, detail="Refresh Token 无效或过期")

@app.get("/me", response_model=UserResponse)

async def read_me(current_user: dict = Depends(get_current_user)):

    return current_user

@app.get("/public")

async def public():

    return {"message": "Hello, 这是公开接口"}

# ============================================================

@app.exception_handler(HTTPException)

async def http_exception_handler(request: Request, exc: HTTPException):

    return JSONResponse(

        status_code=exc.status_code,

        content={"detail": exc.detail},

    )

```

---

## 测试 1：JWT 全链路（5 用户）

```python

async def test_jwt_full_chain():

    """

    模拟 5 个用户的完整生命周期：

    注册 → 登录(拿到双Token) → 访问/me(带Access Token)

    → 用 Refresh Token 换新 Access → 再次访问 /me

    """

    import aiohttp

    BASE = "http://127.0.0.1:8000"

    async with aiohttp.ClientSession() as session:

        print("=" * 60)

        print("JWT 全链路并发测试 (5 个用户)")

        print("=" * 60)

        test_users = [

            {"username": f"user_{i}", "password": f"pass_{i}"}

            for i in range(1, 6)

        ]

        for user in test_users:

            print(f"\n── {user['username']} ──")

            # ① 注册

            async with session.post(f"{BASE}/auth/register", json=user) as resp:

                if resp.status == 200:

                    reg = await resp.json()

                    print(f"  ① 注册成功: id={reg['id']}, username={reg['username']}")

                elif resp.status == 400:

                    print(f"  ① 已注册（跳过）")

                else:

                    print(f"  ① 注册失败: {resp.status} {await resp.text()}")

            # ② 登录（OAuth2PasswordRequestForm 需要 form-data，不是 JSON）

            form = aiohttp.FormData()

            form.add_field("username", user["username"])

            form.add_field("password", user["password"])

            async with session.post(f"{BASE}/auth/login", data=form) as resp:

                if resp.status != 200:

                    print(f"  ② 登录失败: {resp.status} {await resp.text()}")

                    continue

                tokens = await resp.json()

                access = tokens["access_token"]

                refresh = tokens["refresh_token"]

                print(f"  ② 登录成功: access=...{access[-20:]}, refresh=...{refresh[-20:]}")

            # ③ 访问 /me（带 Access Token）

            headers = {"Authorization": f"Bearer {access}"}

            async with session.get(f"{BASE}/me", headers=headers) as resp:

                me = await resp.json()

                print(f"  ③ /me 返回: {me}")

            # ④ 刷新 Token

            async with session.post(f"{BASE}/auth/refresh", json={"refresh_token": refresh}) as resp:

                new_tokens = await resp.json()

                new_access = new_tokens["access_token"]

                print(f"  ④ 刷新成功: new_access=...{new_access[-20:]}")

            # ⑤ 用新 Access Token 再访问 /me

            new_headers = {"Authorization": f"Bearer {new_access}"}

            async with session.get(f"{BASE}/me", headers=new_headers) as resp:

                me2 = await resp.json()

                print(f"  ⑤ 新 Token 访问 /me: {me2}")

        # ⑥ 测试不带 Token → 应返回 401

        print(f"\n── 异常测试 ──")

        async with session.get(f"{BASE}/me") as resp:

            print(f"  无 Token 访问 /me: {resp.status} {await resp.json()}")

        # ⑦ 测试错误的 Token

        bad_headers = {"Authorization": "Bearer this.is.fake"}

        async with session.get(f"{BASE}/me", headers=bad_headers) as resp:

            print(f"  假 Token 访问 /me: {resp.status} {await resp.json()}")

```

---

## 测试 2：20 用户并发压测

```python

async def test_concurrent_users():

    """

    模拟 20 个用户同时注册→登录→访问 /me。

    验证 asyncio 并发能力 + JWT 系统在并发下无竞态。

    """

    import aiohttp

    BASE = "http://127.0.0.1:8000"

    N = 20

    async with aiohttp.ClientSession() as session:

        async def one_user_lifecycle(i: int) -> dict:

            username = f"cuser_{i}"

            password = f"cpass_{i}"

            log = {"user": username}

            # 注册

            async with session.post(f"{BASE}/auth/register", json={"username": username, "password": password}) as resp:

                log["register"] = resp.status

            # 登录

            form = aiohttp.FormData()

            form.add_field("username", username)

            form.add_field("password", password)

            async with session.post(f"{BASE}/auth/login", data=form) as resp:

                if resp.status != 200:

                    log["login"] = f"FAIL({resp.status})"

                    return log

                tokens = await resp.json()

                log["login"] = "OK"

            # 访问 /me

            headers = {"Authorization": f"Bearer {tokens['access_token']}"}

            async with session.get(f"{BASE}/me", headers=headers) as resp:

                log["me"] = resp.status

            return log

        print("\n" + "=" * 60)

        print(f"并发测试: {N} 个用户同时注册→登录→访问")

        print("=" * 60)

        t0 = _time.time()

        results = await asyncio.gather(*[

            one_user_lifecycle(i) for i in range(1, N + 1)

        ])

        elapsed = _time.time() - t0

        ok_count = sum(1 for r in results if r.get("login") == "OK")

        print(f"\n结果: {ok_count}/{N} 成功, 耗时 {elapsed:.2f}s")

        print(f"平均每个用户: {elapsed/N*1000:.0f}ms")

```

---

## 启动与运行

```bash

# 终端1：启动服务

python main.py

# 终端2：跑测试

python main.py --test

```

### 测试输出示例

```text

============================================================

JWT 全链路并发测试 (5 个用户)

============================================================

── user_1 ──

  ① 注册成功: id=1, username=user_1

  ② 登录成功: access=...SflKxwRJSMeKKF2QT4f, refresh=...ZyJIUzI1NiIsInR5c

  ③ /me 返回: {'id': 1, 'username': 'user_1'}

  ④ 刷新成功: new_access=...8d9e0f1a2b3c4d5e6f7

  ⑤ 新 Token 访问 /me: {'id': 1, 'username': 'user_1'}

...

── 异常测试 ──

  无 Token 访问 /me: 401 {'detail': '未提供 Token'}

  假 Token 访问 /me: 401 {'detail': 'Token 无效或过期'}

============================================================

并发测试: 20 个用户同时注册→登录→访问

============================================================

结果: 20/20 成功, 耗时 2.34s

平均每个用户: 117ms

```

### 入口代码

```python

if __name__ == "__main__":

    import sys

    if len(sys.argv) > 1 and sys.argv[1] == "--test":

        async def run_tests():

            await test_jwt_full_chain()

            await test_concurrent_users()

        asyncio.run(run_tests())

    else:

        uvicorn.run(app, host="127.0.0.1", port=8000)

```

---

##

> ▶ 对应实操：[[06-依赖注入|06-依赖注入]]

> ▶ 对应实操：[[03-JWT原理与设计|03-JWT原理与设计]]

相关链接

- 上一篇：[[04-FastAPI+JWT全链路实现]]

---

→ [[技术学习路线图#JWT 鉴权]]

## 速记卡（面试闪卡）

**Q1：一句话讲清「JWT 鉴权全链路」到底是什么？**

A：JWT 是把用户身份装进一张签名防伪造的令牌，实现无状态鉴权，请求带 Bearer 即验即过。

**Q2：生活类比：酒店房卡 —— 怎么理解？**

A：像酒店房卡：登录验身份后发你一张写房号+有效期的卡，用私钥签名防伪；之后刷每道门禁自己验卡，不用每次跑前台(JWT as Hotel Keycard)。

**Q3：什么是 JWT —— 怎么理解？**

A：像三段式防伪票：Header.Payload.Signature，payload 明文存 sub/exp 但不加密，Signature 用密钥算指纹防篡改(JSON Web Token)。

**Q4：全链路四步 —— 怎么理解？**

A：像四步通关：登录验密→签 token(带 sub+exp)→客户端存 Bearer→每次请求服务端验签名过期解码出用户(Auth Flow)。

**Q5：手写最小可用全链路（⭐必考） —— 怎么理解？**

A：像照方抓药：create_access_token 用 HS256 签名，get_current_user 依赖解码验 exp，OAuth2PasswordBearer 从请求头拔 token(PyJWT)。

**Q6：核心速记主线有哪些？**

- JWT 三段：Header.Payload.Signature，签名防伪造不加密

- 全链路：登录签 token→请求带 Bearer→验签解码出用户

- 无状态：令牌自包含，免查库；吊销靠短命+黑名单

- 安全红线：密钥进环境变量、HTTPS 传输、payload 不放机密

**口诀**

A：JWT 三段票，头载签名载信息

登录签发出，请求 Bearer 带回去

验签解出来，无状态里不查库

密钥进变量，HTTPS 护传输

## 相关链接

- 上一篇：[[03-JWT原理与设计]]

- 下一篇：[[05-JWT综合实战与并发压测]]

---

→ [[技术学习路线图#JWT 鉴权]]


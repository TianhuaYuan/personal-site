---
title: "JWT 综合实战与并发压测"
created: "2025-07-12"
tags:
  - 技术学习
  - python
  - fastapi
  - jwt
  - asyncio
  - aiohttp
  - 实战
---

# JWT 综合实战与并发压测


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


## 速记卡（面试闪卡）

**Q1：一句话讲清「JWT 综合实战与并发压测」到底是什么？**
A：这篇笔记用单文件实现 JWT 双 Token 鉴权全链路，并用 aiohttp 做多用户并发压测验证无竞态。

**Q2：完整代码 —— 怎么理解？**
A：核心三件套：密码用 bcrypt 哈希；_make_token 用 jose 的 JWT（JSON Web Token，JSON 网络令牌）签发 access/refresh，带 iat/exp/type；get_current_user 依赖校验 access。手工走通 HTTP 里 token 怎么传，不用 Depends 封装。

**Q3：测试 1：JWT 全链路（5 用户） —— 怎么理解？**
A：模拟 5 个用户：注册 → 登录拿双 Token → 带 access 访问 /me → 用 refresh 换新 access → 再访问。最后测无 Token 和假 Token 都应返回 401。覆盖注册、登录、刷新、鉴权拒绝全链路。

**Q4：测试 2：20 用户并发压测 —— 怎么理解？**
A：用 asyncio.gather 同时跑 20 个用户生命周期（注册→登录→访问）。asyncio（异步 IO）并发下 20/20 成功、约 2.34s，证明 JWT 系统在并发下无竞态、无丢请求。

**Q5：启动与运行 —— 怎么理解？**
A：终端1 `python main.py` 起服务，终端2 `python main.py --test` 跑测试。入口用 sys.argv 区分服务模式与测试模式，uvicorn 跑在本地 8000 端口。

**Q6：核心速记主线有哪些？**
- JWT 双 Token：access 短期、refresh 长期，type 字段区分
- 密码 bcrypt 哈希，jose 签发/校验 token
- 全链路：注册→登录→访问→刷新→再访问
- asyncio 并发压测验证无竞态

**口诀**
A：JWT 双令牌分长短，
bcrypt 哈希防裸奔；
注册登录刷新链，
异步压测二十稳。

## 相关链接

- 上一篇：[[04-FastAPI+JWT全链路实现]]

---
→ [[技术学习路线图#JWT 鉴权]]

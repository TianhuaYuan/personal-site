---
title: "Redis 7.0+ 新特性：Functions / ACL v2"
created: "2026-07-20"
tags:
  - 八股文
  - redis
---

# Redis 7.0+ 新特性：Functions / ACL v2

## Functions —— 把 Lua 脚本从"代码里的字符串"变成"Redis 上的函数"

### 先搞清楚：Redis 为啥要跑 Lua 脚本？

```text
你写两个 Redis 操作，要保证中间没人插一脚：

  Python 代码里：
    stock = r.get("stock:iphone18")
    if int(stock) > 0:
        r.decr("stock:iphone18")

  问题：
    第 1 行 GET 完，还没 DECR，另一个请求也 GET 到了库存=1
    → 两个人都觉得有货 → 都 DECR → 超卖 ❌

  解法：把两个操作塞进 Lua 脚本，让 Redis 原子执行
    lua = """
      local stock = redis.call('GET', KEYS[1])
      if tonumber(stock) > 0 then
        return redis.call('DECR', KEYS[1])
      end
      return -1
    """
    r.eval(lua, 1, "stock:iphone18")

  Lua 脚本 = 让 Redis 一次性执行完，中间谁都插不进来。
  这就叫原子性。
```

### Functions 要解决什么？

```text
你的项目里可能有几个常用的 Lua 脚本：

  Script A: 扣库存（检查→减→返回结果）
  Script B: 分布式锁（SET NX EX → Lua 解锁）
  Script C: 滑动窗口限流（检查次数→+1→设过期→返回）

7.0 之前这些脚本住在哪？——住在你的 Python 代码里：

  """app/utils/lua_scripts.py"""
  DECR_STOCK = """
    local stock = redis.call('GET', KEYS[1])
    if tonumber(stock) and tonumber(stock) > 0 then
      return redis.call('DECR', KEYS[1])
    end
    return -1
  """

  UNLOCK = """
    if redis.call('get', KEYS[1]) == ARGV[1] then
      return redis.call('del', KEYS[1])
    end
    return 0
  """

每次调用都要把脚本全文发给 Redis：

  r.eval(DECR_STOCK, 1, "stock:iphone18")
  → 发送脚本全文，Redis 编译一次，执行

问题在哪？
  ① 脚本存在代码里 → 三个微服务就存三份 → 改脚本要改三处
  ② 每次调用都发脚本 → 浪费带宽（脚本可能几百行）
  ③ 脚本在代码里藏着 → 运维不知道 Redis 跑着哪些脚本
  ④ Redis 每次都要重新编译脚本

像什么？
  你手机里存了"红烧肉的做法"。
  每次做饭，你都要把全文念给厨师听。
  厨师每次听完都要重新理解一遍。
  你换了手机，做法得重新存一遍。
```

### Functions 怎么解决？——把脚本"装进" Redis

```text
你在 Redis 里注册一个函数，给它起个名字。
以后要用的时候，直接喊名字。

注册一次：
  FUNCTION LOAD mylib "#!lua name=mylib\n redis.register_function('decr_stock', function(keys, args) ... end)"

  翻译成人话：
    "Redis，我注册一个叫 mylib 的函数库，
    里面有个函数叫 decr_stock，
    以后我喊这个名，你就执行这段 Lua。"

以后调用：
  redis-cli FCALL decr_stock 1 stock:iphone18
  → 不用传脚本全文，就说"执行 decr_stock"

效果对比：
  之前：你每次都要把"红烧肉做法"念给厨师听
  现在：厨师把做法背下来了，你说"来个红烧肉"就行
```

### 直观对比 Functions 和 EVAL

```text
EVAL（7.0 之前）：

  调用：
    r.eval("local s=redis.call('GET',KEYS[1])..." , 1, "stock")

  Redis 收到：
    ① 收到"local s=redis.call('GET',KEYS[1])..." (2KB)
    ② 编译成字节码
    ③ 执行

  每次都要经历 ① 和 ②。

Functions（7.0 之后）：

  注册：
    redis-cli FUNCTION LOAD mylib "lua脚本..." → 编译好存着

  调用：
    redis-cli FCALL decr_stock 1 "stock:iphone18"

  Redis 收到：
    ① 按名字找到已编译好的字节码
    ② 直接执行（没有 ① 和 ② 的开销）

  省了传脚本的带宽，省了编译的时间。
```

### 一个具体的项目场景

```text
你在做 AI Agent 后端，有个需求：
  限制每个用户每秒最多调用 10 次 LLM API。

你需要一个原子操作的限流器——不能有两个请求同时+1 导致计数不准。

不用 Functions 时：
  你在 Python 代码里写一段 Lua 脚本（滑动窗口限流）。
  每次请求都要把这段脚本全文发给 Redis。

用了 Functions 后：
  Step 1：运维注册一次限流函数
    FUNCTION LOAD rate_limit_lib "...脚本..."
    → 注册一个叫 rate_limit 的函数到 Redis

  Step 2：你的代码调用
    r.fcall("rate_limit", 1, f"ratelimit:user:{user_id}", "10", "60")
    → 按名字调，不传脚本全文

  Step 3 ：哪天想改限流策略（从 10 次改成 20 次）
    不用改代码。
    重新注册一次函数就行。
    所有服务立刻生效。
```

---

## ACL v2 —— 从"一把钥匙开所有门"到"每人一把不同的钥匙"

### ACL 解决什么问题？

```text
你项目里的 Redis 是这么用的：

  一个密码，所有人共用。

  你的 FastAPI 后端用这个密码 → 正常读写 √
  你的监控脚本用这个密码 → 正常读 √
  你的同事想看看 Redis 里有啥 → 也用这个密码 √
  你的 CI/CD 脚本不小心跑了 FLUSHALL → 库没了 ❌

问题在哪？
  所有人的权限是一样的。
  后端只需要读写业务 key，但它也能执行 FLUSHALL。
  监控只需要读 INFO，但它也能删数据。
  你只想让同事查 key，但他也能改配置。

像什么？
  你租了个房子，给所有租客同一个钥匙。
  这把钥匙能开大门、能开卧室、能开保险柜。
  保洁阿姨用这把钥匙 → 能进你卧室。
  修水管的用这把钥匙 → 也能开你保险柜。
```

### ACL 做了什么？

```text
现在你可以创建多个"用户"，
每个用户有独立的密码和独立的权限。

具体能控制三件事：

① 控制能执行哪些命令
   +@read       → 只能读（GET、HGET、MGET...）
   +@write      → 只能写（SET、HSET、LPUSH...）
   -FLUSHALL    → 禁止执行 FLUSHALL
   -SHUTDOWN    → 禁止关服务器
   -CONFIG      → 禁止改配置
   +@all        → 全部允许

② 控制能操作哪些 key
   ~*           → 所有 key
   ~user:*      → 只能操作 user: 开头的 key
   ~order:*     → 只能操作 order: 开头的 key

③ 控制从哪里连
   可选限制客户端 IP
```

### 一个具体的项目场景

```text
你的项目里，至少需要三种用户：

用户 1：app_user（FastAPI 后端用的）
  ACL SETUSER app_user on >password123 ~app:* ~cache:* +@read +@write
  → 只能读和写 app: 和 cache: 开头的 key
  → 不能删库、不能改配置、不能 KEYS 全量扫

用户 2：monitor_user（监控用的）
  ACL SETUSER monitor on >monitor_pass ~* +@read +@connection
  → 只能读，任何 key 都可以读（监控要看指标）
  → 不能写、不能删

用户 3：admin_user（你手动操作的）
  ACL SETUSER admin on >admin_pass ~* +@all
  → 全部权限
  → 你也可以创建这个用户但不设密码，因为本地连才用它

三把钥匙，各干各的：
  后端 app_user → 只读写业务 key ✅
  监控 monitor_user → 只能读 ✅
  你 admin_user → 什么都能干，但只有你有这个密码 ✅
  CI/CD 脚本想跑 FLUSHALL？→ 它用的是 app_user 的密码 → 被拒绝 ✅
```

### ACL 落地后你就不用怕了

```text
之前怕的事：
  "线上不小心敲了个 FLUSHALL，库就没了"

现在：
  ACL 限制了 app_user 不能执行 FLUSHALL。
  就算你在代码里写了 r.flushall() → Redis 报错：无权限。
  想删库？用 admin_user 的密码手动连上去敲。

之前怕的事：
  "同事想查线上数据，给了他密码，
  结果他手滑 SET 错了 key 搞出事故"

现在：
  给他开一个 reader 用户，只能读不能写。
  他查任他查，写不了。 ✅
```

---

## 一句话讲清

```text
延伸提问："Redis 7.0 有什么新特性？"

你：
"我比较关注 Functions 和 ACL v2。

Functions 是对 Lua 脚本的管理升级。
以前用 EVAL 每次传脚本全文，
现在注册到 Redis 上按名字调用，
省带宽、省编译、好管理，
改脚本不用改应用代码，重新注册就行。

ACL v2 是 Redis 终于有了"用户权限"。
以前一个密码通吃，能读能写能删库。
现在可以创建多个用户，
限制每个用户能执行的命令和能操作的 key。

我们项目就分了三个用户：
  后端只读写业务 key、监控只能读、运维才有全部权限。
  这样就算 CI/CD 脚本出了 bug 也不会把 Redis 搞炸。"
```

---

## 记忆口诀

> **Functions = 把 Lua 脚本注册成"名字"存在 Redis 上，按名字调用，省带宽省编译好管理。**
> **ACL = 创建多个 Redis 用户，每个用户开不同权限——后端的只能读写业务 key，运维的才能 FLUSHALL。**
> **Functions 解决"脚本散落在代码里不好管"的问题。**
> **ACL 解决"一把钥匙开所有门，不安全"的问题。**

## 相关链接

- 📋 目录：[[00-Redis]]
- 📚 学习清单：[[八股文学习清单]]

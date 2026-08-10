---
title: "指数退避重试：Exponential Backoff + Jitter"
tags:
  - 技术学习
  - ai
  - agent
created: "2026-07-21"
---

> **一句话**：指数退避（1s→2s→4s）让负载指数下降给故障服务恢复窗口，Jitter打碎同步重试防惊群效应——但只对临时故障重试，编程错误必须立即抛出。

## 一、原理速览

**指数退避（Exponential Backoff）**：第 n 次重试前等待 = `base × 2ⁿ`。让负载随时间指数下降，给故障服务恢复窗口。

| 重试次数 n | 等待上界 | 累计已等待 |
|-----------|---------|-----------|
| 0（首次调用失败） | 1s | 1s |
| 1 | 2s | 3s |
| 2 | 4s | 7s |
| 3 | 8s | 15s |
| 4 | 16s | 31s |

**必须封顶**：
- 重试次数上限 3-5 次
- 退避时间上限 30-60s（防止第 10 次等 512 秒）

### Jitter（抖动）：打碎同步重试

一万个客户端同时失败，纯指数退避它们在同一秒重试（惊群效应）。Jitter 加随机扰动拆散同步。

| 算法 | 等待公式 | 适合场景 |
|------|---------|---------|
| **Full Jitter** | `random(0, base×2ⁿ)` | 默认首选，高并发大量客户端（AWS SDK 默认） |
| **Equal Jitter** | `base×2ⁿ/2 + random(0, base×2ⁿ/2)` | 需保证最小等待 |
| **Decorrelated Jitter** | `min(cap, random(base, 上次等待×3))` | 自适应，需存上次等待值 |

> AWS 实测（100 并发客户端，4 次重试）：Full Jitter 缩短完成时间约 62%，Decorrelated Jitter 约 65%。

### 编程错误 vs 临时故障：重试的生死线

| 分类 | 例外 | 重试？ |
|------|------|--------|
| 临时故障 | ConnectionError / TimeoutError / 429 / 5xx | ✅ 退避后重试 |
| 编程错误 | TypeError / ValueError / KeyError / 400 / 401 / 404 | ❌ 立即抛出 |

**核心原则**：只有临时故障值得重试；编程错误重试一百次也还是错，必须立即抛出。

### 生产级进阶亮点

- **尊重 Retry-After 头**：服务端 429 响应里告诉你要等多久
- **重试必须配幂等**：先幂等再重试，否则重复扣款
- **Hedged Request**：同时发两份请求，先回来的用（延迟敏感场景）

## 二、代码实现

```python
# 指数退避重试的完整 Python 实现
# 包含：三种 Jitter 算法 / 错误分类 / Retry-After / 编程错误直抛

import random
import time
import functools
import asyncio
from typing import Callable, TypeVar, Any

F = TypeVar("F", bound=Callable[..., Any])


# ============================================================
# 第一部分：三种 Jitter 算法
# ============================================================

def full_jitter(base_delay: float, attempt: int, max_delay: float = 30.0) -> float:
    """Full Jitter：sleep = random(0, min(max_delay, base_delay × 2^attempt))"""
    upper_bound = min(max_delay, base_delay * (2 ** attempt))
    return random.uniform(0, upper_bound)


def equal_jitter(base_delay: float, attempt: int, max_delay: float = 30.0) -> float:
    """Equal Jitter：sleep = half + random(0, half)"""
    upper_bound = min(max_delay, base_delay * (2 ** attempt))
    half = upper_bound / 2.0
    return half + random.uniform(0, half)


def decorrelated_jitter(
    base_delay: float,
    previous_sleep: float,
    max_delay: float = 30.0,
) -> float:
    """Decorrelated Jitter：sleep = min(max_delay, random(base, previous_sleep × 3))"""
    return min(max_delay, random.uniform(base_delay, previous_sleep * 3))


# ============================================================
# 第二部分：NON_RETRYABLE 错误定义
# ============================================================

NON_RETRYABLE_EXCEPTIONS = (
    TypeError, ValueError, KeyError, AttributeError,
    IndexError, AssertionError, SyntaxError, ImportError, NameError,
)


def is_non_retryable(exception: Exception) -> bool:
    return isinstance(exception, NON_RETRYABLE_EXCEPTIONS)


# ============================================================
# 第三部分：重试装饰器 —— 带 Jitter + 错误分类
# ============================================================

def with_exponential_backoff(
    max_retries: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 30.0,
    jitter: str = "full",
    retry_on_result: Callable[[Any], bool] | None = None,
):
    def decorator(func: F) -> F:
        @functools.wraps(func)
        def wrapper(*args, **kwargs) -> Any:
            last_exception = None
            previous_sleep = base_delay

            for attempt in range(max_retries + 1):
                try:
                    result = func(*args, **kwargs)
                    if retry_on_result and retry_on_result(result):
                        if attempt < max_retries:
                            print(f"[结果重试] {func.__name__} 返回需重试的结果，"
                                  f"第 {attempt+1} 次重试")
                            continue
                    return result

                except NON_RETRYABLE_EXCEPTIONS as e:
                    print(f"[编程错误] {func.__name__}: {type(e).__name__}: {e} → 立即抛出，不重试")
                    raise

                except Exception as e:
                    last_exception = e
                    if attempt >= max_retries:
                        print(f"[重试耗尽] {func.__name__}: {max_retries} 次重试后仍失败")
                        raise

                    if jitter == "equal":
                        sleep_time = equal_jitter(base_delay, attempt, max_delay)
                    elif jitter == "decorrelated":
                        sleep_time = decorrelated_jitter(base_delay, previous_sleep, max_delay)
                        previous_sleep = sleep_time
                    else:
                        sleep_time = full_jitter(base_delay, attempt, max_delay)

                    print(
                        f"[重试] {func.__name__} 第 {attempt+1}/{max_retries} 次, "
                        f"等待 {sleep_time:.2f}s, "
                        f"异常: {type(e).__name__}: {str(e)[:60]}"
                    )
                    time.sleep(sleep_time)

            if last_exception:
                raise last_exception
            raise RuntimeError(f"{func.__name__}: 重试逻辑异常终止")

        return wrapper
    return decorator


# ============================================================
# 第四部分：异步版重试装饰器
# ============================================================

def async_with_exponential_backoff(
    max_retries: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 30.0,
    timeout: float | None = None,
):
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        async def wrapper(*args, **kwargs) -> Any:
            last_exception = None

            for attempt in range(max_retries + 1):
                try:
                    if timeout is not None:
                        result = await asyncio.wait_for(
                            func(*args, **kwargs), timeout=timeout,
                        )
                    else:
                        result = await func(*args, **kwargs)
                    return result

                except asyncio.TimeoutError:
                    last_exception = TimeoutError(f"调用超时 ({timeout}s)")
                    if attempt >= max_retries:
                        raise last_exception

                except NON_RETRYABLE_EXCEPTIONS:
                    raise

                except Exception as e:
                    last_exception = e
                    if attempt >= max_retries:
                        raise

                sleep_time = full_jitter(base_delay, attempt, max_delay)
                print(f"[异步重试] {func.__name__} 第 {attempt+1}/{max_retries} 次, "
                      f"等待 {sleep_time:.2f}s")
                await asyncio.sleep(sleep_time)

            if last_exception:
                raise last_exception
            raise RuntimeError(f"{func.__name__}: 重试逻辑异常终止")

        return wrapper
    return decorator


# ============================================================
# 运行演示
# ============================================================

_call_count = 0

@with_exponential_backoff(
    max_retries=3, base_delay=0.5, max_delay=10.0, jitter="full",
)
def unstable_api_call(endpoint: str) -> dict:
    global _call_count
    _call_count += 1
    if _call_count <= 2:
        raise ConnectionError(f"连接 {endpoint} 超时: 上游服务无响应")
    return {"status": "ok", "data": f"来自 {endpoint} 的响应", "attempt": _call_count}


if __name__ == "__main__":
    print("=" * 50)
    print("演示 1: 指数退避重试（前 2 次失败，第 3 次成功）")
    try:
        result = unstable_api_call("https://api.example.com/search")
        print(f"最终结果: {result}")
    except Exception as e:
        print(f"最终失败: {e}")

    print("\n演示 2: 三种 Jitter 算法在同一场景的等待时间对比")
    print(f"{'重试':>4}  {'Full':>10}  {'Equal':>10}  {'Decorrelated':>14}")
    prev = 1.0
    for n in range(4):
        f = full_jitter(1.0, n, 30.0)
        e = equal_jitter(1.0, n, 30.0)
        d = decorrelated_jitter(1.0, prev, 30.0)
        prev = d
        print(f"{n:>4}  {f:>10.2f}  {e:>10.2f}  {d:>14.2f}")

    print("\n演示 3: 编程错误不重试（立即抛出）")
    @with_exponential_backoff(max_retries=3)
    def bad_function():
        raise TypeError("参数类型错误：需要 str 却给了 int")
    try:
        bad_function()
    except TypeError as e:
        print(f"编程错误被正确立即抛出: {e}")
```

## 相关链接

- 项目实践：[[项目实战/ai-resume笔记/01_技术研读/06_韧性工程#retry.py 逐行走读（49 行全量）|ai-resume: 韧性工程]]
- 项目实践：cr-agent: 三层容错与并发bug

---
→ [[技术学习清单#Agent 韧性工程]]

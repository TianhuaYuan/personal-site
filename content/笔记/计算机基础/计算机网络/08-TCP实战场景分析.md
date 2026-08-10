---
title: "TCP实战场景分析"
created: "2025-07-12"
tags:
  - 八股文
  - 计算机网络
---

## TCP 实战场景分析

### 场景 1：浏览器访问网页

> 从输入 URL 到页面展示，TCP 连接的完整生命周期。

```mermaid
graph TD
    A["DNS 解析域名 → IP 地址"] --> B["TCP 三次握手建立连接"]
    B --> C["发送 HTTP 请求"]
    C --> D["服务器返回 HTML/CSS/JS"]
    D --> E["浏览器解析渲染"]
    E --> F{"HTTP/1.1 长连接?"}
    F -->|是| G["保持连接，复用"]
    F -->|否| H["TCP 四次挥手关闭连接"]
    G --> I["关闭页面或超时后"]
    I --> H
```

> 详见 [[14-浏览器输入URL到页面展示|浏览器输入URL到页面展示]]。

### 场景 2：高并发服务器 TIME_WAIT 优化

> **问题**：大量短连接导致 TIME_WAIT 过多，端口耗尽，新连接 bind 失败。

**解决方案**：

```python
# 使用连接池复用连接，避免频繁建立/关闭 TCP 连接
import aiohttp
import asyncio

async def fetch(session, url):
    async with session.get(url) as response:
        return await response.text()

async def main():
    # 创建连接池，复用 TCP 连接
    connector = aiohttp.TCPConnector(limit=100, ttl_dns_cache=300)
    async with aiohttp.ClientSession(connector=connector) as session:
        tasks = [fetch(session, f"http://api.example.com/{i}") for i in range(100)]
        await asyncio.gather(*tasks)

asyncio.run(main())
```

> 其他方案：开启 `tcp_tw_reuse`、使用长连接（HTTP/1.1 Keep-Alive）、负载均衡分散连接。详见 [[05-TIME_WAIT与粘包拆包|TIME_WAIT 与粘包拆包]]。

### 场景 3：网络调试常用命令

```bash
# 查看 TCP 连接状态分布
ss -tan | awk 'NR>1 {states[$1]++} END { for (s in states) printf "%-15s %d\n", s, states[s] }' | sort -k2 -rn

# 查看特定状态连接
ss -tan state time-wait    # TIME_WAIT 连接
ss -tan state close-wait   # CLOSE_WAIT 连接（Bug 探测器）
ss -tan state established  # 活跃连接

# 查看内核参数
sysctl net.ipv4.tcp_tw_reuse
sysctl net.ipv4.tcp_max_syn_backlog
sysctl net.ipv4.tcp_max_tw_buckets

# 抓包分析三次握手
tcpdump -i eth0 port 80 -w handshake.pcap
```

### 场景 4：SYN Flood 攻击排查与防御

> **现象**：服务器 SYN_RCVD 状态连接暴增，CPU 和内存正常但新连接建立失败。

```bash
# 查看半连接队列大小
ss -tan state syn-recv | wc -l

# 查看当前 SYN backlog 配置
sysctl net.ipv4.tcp_max_syn_backlog

# 查看是否开启 SYN Cookie
sysctl net.ipv4.tcp_syncookies
```

**防御措施**：
1. 开启 SYN Cookie（`tcp_syncookies=1`）：不分配资源，把信息编码在序列号里
2. 增加半连接队列大小（`tcp_max_syn_backlog`）
3. 限制 SYN 速率（防火墙规则）

> 详见 [[07-TCP高频追问汇总|TCP高频追问汇总]] Q8。

---

## 速记卡（面试闪卡）

**Q1：一句话讲清「使用连接池复用连接，避免频繁建立/关闭 TCP 连接」到底是什么？**
A：从输入 URL 到页面展示，TCP 连接的完整生命周期。
详见 。
**问题**：大量短连接导致 TIME_WAIT 过多，端口耗尽，新连接 bind 失败。
**解决方案**：
其他方案：开启 、使用长连接（HTTP/1.1 Keep-Alive）、负载均衡分散连接。详见 。
**现象**：服务器 SYN_RCVD 状态连接暴增，CPU 和内存正常但新连接建立失败。

**Q2：TCP 实战场景分析 —— 怎么理解？**
A：从输入 URL 到页面展示，TCP 连接的完整生命周期。
详见 。
**问题**：大量短连接导致 TIME_WAIT 过多，端口耗尽，新连接 bind 失败。
**解决方案**：
其他方案：开启 、使用长连接（HTTP/1.1 Keep-Alive）、负载均衡分散连接。详见 。
**现象**：服务器 SYN_RCVD 状态连接暴增，CPU 和内存正常但新连接建立失败。

**Q3：核心速记主线有哪些？**
A：抓住这几根：TCP 实战场景分析。


## 相关链接

- 📋 目录：[[00-计算机网络]]
- 📚 学习清单：[[八股文学习清单]]
- 🔗 [[02-TCP三次握手|TCP三次握手与四次挥手]] — 握手与挥手流程
- 🔗 [[06-TCP状态机|TCP状态机]] — 状态转换
- 🔗 [[05-TIME_WAIT与粘包拆包|TIME_WAIT与粘包拆包]] — TIME_WAIT 优化
- 🔗 [[14-浏览器输入URL到页面展示|浏览器输入URL到页面展示]] — 完整网络流程

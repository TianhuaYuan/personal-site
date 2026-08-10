---
title: "事件循环EventLoop"
created: "2025-07-12"
tags:
  - 八股文
  - javascript
  - react-ts-js
---

# 事件循环EventLoop
## 二、同步任务与异步任务



```javascript

console.log('1: 同步');            // 同步任务



setTimeout(function() {

    console.log('2: 异步');        // 异步任务（setTimeout）

}, 0);



console.log('3: 同步');            // 同步任务

// 输出：1: 同步 → 3: 同步 → 2: 异步

```



**为什么 `setTimeout(fn, 0)` 也排在同步代码后面？**



```text

同步代码在调用栈里立刻执行完（'1'、'3'）

setTimeout 的回调被扔进任务队列，等调用栈空了才被取出来执行

所以哪怕是 0 毫秒，也要等同步代码全部跑完

```



> **一句话**：同步任务在调用栈里\"插队优先\"执行，异步任务乖乖排队，等同步全跑完才轮到。



### 异步任务的来源



```javascript

setTimeout(fn, 1000);                     // ① 定时器

setInterval(fn, 1000);                    // ② 周期定时器（每隔 1 秒跑一次）

fetch('https://api.xx.com').then(...)     // ③ 网络请求

fs.readFile('a.txt', callback)            // ④ 文件 IO（Node 环境）

btn.addEventListener('click', fn)         // ⑤ 用户事件（点击）

```



> 这些\"慢操作\"都不会卡住主线程，它们的回调都会进队列。今天重点是它们的回调**怎么排队、什么时候被执行**。



---





## 相关链接



- 📋 目录：[[00-JavaScript]]

- 📚 学习清单：[[八股文学习清单]]

- 🔗 [[语言与框架/Python/八股/并发/14-asyncio事件循环原理|Python asyncio事件循环]] — 两种语言的事件循环实现对比

- 🔗 [[计算机基础/操作系统/14-IO多路复用select_poll_epoll|IO多路复用]] — 事件循环的底层是IO多路复用

- 🔗 [[语言与框架/TypeScript-React/八股/React/11-ReactFiber架构与Scheduler|React Fiber架构]] — React的调度器运行在事件循环之上




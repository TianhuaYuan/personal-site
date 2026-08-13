---

title: "事件循环EventLoop"

created: "2025-07-12"

tags:

  - 八股文

  - javascript

  - react-ts-js

---



# 事件循环EventLoop

## 一、同步任务与异步任务





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





## 速记卡（面试闪卡）



**Q1：一句话讲清「事件循环EventLoop」到底是什么？**

A：事件循环是 JS 的单线程调度器，同步代码先跑完、异步回调排队等栈空再执行。



**Q2：同步 vs 异步（call stack / task queue） —— 怎么理解？**

A：类比：主线程是唯一的收银台（调用栈），同步代码是当场结账的顾客立刻办完；setTimeout/fetch 的回调是"取号排队"的顾客，等收银台空了才被叫号。所以哪怕 setTimeout(fn,0) 也要排在同步代码后面——0 毫秒只是"尽快"，不是"插队"。（Call stack）



**Q3：异步任务从哪来（timers / IO / events） —— 怎么理解？**

A：类比：会进队列的"慢操作"有几种来源：setTimeout/setInterval 定时器、fetch 网络请求、文件 IO、addEventListener 用户点击。它们都不会卡主线程，回调统一进任务队列等栈空——重点就是这些回调怎么排队、何时被执行。（Async sources）



**Q4：为什么单线程不卡（non-blocking） —— 怎么理解？**

A：类比：单线程听起来会卡，但"等"的活（等网络/等磁盘）全交给底层（libuv/内核）去熬，JS 主线程只管派活和收结果。事件循环像餐厅一个服务员：点完单后他去别桌，菜好了再回来上——全程不干等，所以单线程也能高并发。（Non-blocking IO）



**Q5：和 Python asyncio 的关系（cross-language） —— 怎么理解？**

A：类比：JS 事件循环和 Python asyncio 是同一个思想的两种方言：都是一个主循环 + 任务队列 + 非阻塞等待。区别在于 JS 内建在语言运行时、Python 靠 asyncio 库显式 await。底层都靠 IO 多路复用（epoll）撑起高并发。（Shared idea）



**Q6：核心速记主线有哪些？**

- 同步任务在调用栈立刻执行，异步回调进任务队列等栈空

- setTimeout(fn,0) 也排在同步之后，0 毫秒≠插队

- 异步来源：定时器 / 网络 / 文件 IO / 用户事件

- 单线程不卡：等待交给底层，主线程非阻塞派活收结果

- 与 Python asyncio 同源：主循环+队列+非阻塞，底层靠 epoll



**口诀**

A：同步先结账，异步取号排；

栈空才叫号，零秒也不掰；

单线程不卡，底层替你挨；

JS 与 asyncio，同曲异调来。



## 相关链接





- 📋 目录：[[00-JavaScript]]



- 📚 学习清单：[[八股文学习路线图]]



- 🔗 [[语言与框架/Python/八股/并发/14-asyncio事件循环原理|Python asyncio事件循环]] — 两种语言的事件循环实现对比



- 🔗 [[计算机基础/操作系统/14-IO多路复用select_poll_epoll|IO多路复用]] — 事件循环的底层是IO多路复用



- 🔗 [[语言与框架/TypeScript-React/八股/React/11-ReactFiber架构与Scheduler|React Fiber架构]] — React的调度器运行在事件循环之上




---

title: "Python 并发 八股文笔记"

created: "2026-07-20"

tags:

  - 八股文

  - python

  - python并发

  - 索引

---



# Python 并发 八股文笔记

## 📑 目录（01-21）



- [[语言与框架/Python/八股/并发/01-CPU密集与IO密集的并发选型|CPU 密集与 IO 密集的并发选型]]

- [[语言与框架/Python/八股/并发/02-多线程多进程协程三者对比|多线程/多进程/协程三者对比]]

- [[语言与框架/Python/八股/并发/03-为什么需要异步|为什么需要异步]]

- [[语言与框架/Python/八股/并发/04-协程vs线程的调度本质区别|协程 vs 线程的调度本质区别]]

- [[语言与框架/Python/八股/并发/05-GIL是什么|GIL 是什么]]

- [[语言与框架/Python/八股/并发/06-GIL为什么存在|GIL 为什么存在]]

- [[语言与框架/Python/八股/并发/07-GIL对多线程的影响|GIL 对多线程的影响]]

- [[语言与框架/Python/八股/并发/08-怎么绕过GIL|怎么绕过 GIL]]

- Python 3.13 自由线程与 PEP 703

- [[语言与框架/Python/八股/并发/10-生成器与yield原理|生成器与 yield 原理]]

- [[语言与框架/Python/八股/并发/11-send双向通信|send 双向通信]]

- [[语言与框架/Python/八股/并发/12-yieldfrom委托生成器|yield from 委托生成器]]

- [[语言与框架/Python/八股/并发/13-上下文管理器with原理|上下文管理器 with 原理]]

- [[语言与框架/Python/八股/并发/14-asyncio事件循环原理|asyncio 事件循环原理]]

- [[语言与框架/Python/八股/并发/15-async-await本质|async/await 本质]]

- [[语言与框架/Python/八股/并发/16-gather与create_task|gather 与 create_task]]

- [[语言与框架/Python/八股/并发/17-协程阻塞陷阱|协程阻塞陷阱]]

- [[语言与框架/Python/八股/并发/18-threading模块|threading 模块]]

- [[语言与框架/Python/八股/并发/19-multiprocessing模块|multiprocessing 模块]]

- [[语言与框架/Python/八股/并发/20-concurrent-futures线程池进程池|concurrent.futures 线程池/进程池]]

- [[语言与框架/Python/八股/并发/21-生产者消费者模型|生产者-消费者模型]]



## 相关链接



- 📋 上级索引：[[语言与框架/Python/八股/00-Python|Python 总索引]]

- 📚 学习清单：[[学习路线图/八股文学习路线图|八股文学习路线图]]

- 🔗 [[语言与框架/Python/八股/基础/00-Python-基础|Python 基础八股文]] — 装饰器、作用域、OOP 底层

- 🔗 [[计算机基础/操作系统/00-操作系统|操作系统八股文]] — 进程/线程/协程的底层视角

- 🔗 [[计算机基础/计算机网络/00-计算机网络|计算机网络八股文]] — IO 模型与并发的网络基础

- 🔗 [[计算机基础/分布式-系统设计/00-分布式-系统设计|分布式系统设计八股文]] — 消息队列是生产者-消费者的分布式版


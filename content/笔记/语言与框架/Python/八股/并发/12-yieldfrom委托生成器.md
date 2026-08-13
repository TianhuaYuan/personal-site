---

title: "yield from委托生成器"

created: "2026-07-20"

tags:

  - 八股文

  - python

---

# yield from委托生成器

### 4.1 基本语法：替代内层 for 循环





```python



# 没有 yield from：嵌套 for



def flatten(nested):



    for sublist in nested:



        for item in sublist:       # 内层循环



            yield item





# 有 yield from：一行搞定



def flatten(nested):



    for sublist in nested:



        yield from sublist         # 等价于上面的内层 for + yield





list(flatten([[1, 2], [3, 4], [5]]))   # [1, 2, 3, 4, 5]



```





### 4.2 yield from 的三重作用





```python



# ① 简化嵌套——替代 for + yield



def main():



    yield from [1, 2, 3]           # 等价于 for x in [1,2,3]: yield x



    yield from (4, 5)              # 也支持元组、集合等可迭代对象





list(main())   # [1, 2, 3, 4, 5]





# ② 双向通道——send/throw/close 直接透传给子生成器



def sub_gen():



    while True:



        x = yield



        print(f"子生成器收到: {x}")





def main_gen():



    yield from sub_gen()           # send 的值会穿透到 sub_gen





g = main_gen()



next(g)



g.send("hello")                    # "子生成器收到: hello" ← 直接传进去了





# ③ 获取子生成器的 return 值



def sub():



    yield 1



    yield 2



    return "子生成器的返回值"       # return 的值会被 yield from 捕获





def main():



    result = yield from sub()      # result = "子生成器的返回值"



    print(f"拿到: {result}")





g = main()



next(g)     # 1



next(g)     # 2



next(g)     # 拿到: 子生成器的返回值 → StopIteration



```





> **一句话讲清**：`yield from` 不只是语法糖——它在调用者和子生成器之间建了一条**双向通道**，`send()`、`throw()`、`close()` 都能穿透。这是 Python 协程（asyncio 的前身）的底层机制。





---





## 速记卡（面试闪卡）



**Q1：一句话讲清「yield from委托生成器」到底是什么？**

A：yield from 是 Python 语法，让一个生成器把迭代工作委托给另一个可迭代对象并建起双向通道。



**Q2：4.1 基本语法：替代内层 for 循环 —— 怎么理解？**

A：yield from 最简单用法是替代嵌套 for 循环的 yield：`for item in sublist: yield item` 等价于 `yield from sublist`。也支持元组、集合等可迭代对象，如 `yield from [1,2,3]` 等于逐个 yield 1,2,3。展平嵌套列表 `list(flatten([[1,2],[3,4]]))` 一行搞定。



**Q3：4.2 yield from 的三重作用 —— 怎么理解？**

A：yield from 有三重作用：① 简化嵌套（替代 for+yield）；② 双向通道（英文 Two-way Channel）——`send()`/`throw()`/`close()` 直接透传给子生成器（主生成器 send 的值会穿透到 sub_gen）；③ 获取子生成器的 return 值——`result = yield from sub()` 捕获 sub 里 return 的内容。



**Q4：send/throw/close 穿透（双向通道） —— 怎么理解？**

A：双向通道是协程底层机制：主生成器 `yield from sub_gen()` 后，调用者 `g.send("hello")` 的值会直接传进子生成器里 `x = yield` 收到，不用你手动转发。throw/close 同理穿透。这让"调用者↔子生成器"像直连管道，是 asyncio 协程（英文 Coroutine）的前身。



**Q5：捕获子生成器的 return 值 —— 怎么理解？**

A：普通 yield 拿不到子生成器 return 的值，但 yield from 可以：`def sub(): yield 1; yield 2; return "结果"`，主生成器 `result = yield from sub()` 就能拿到 "结果"（底层是捕获 StopIteration 的 value）。这是把子任务的计算结果汇总的干净写法。



**Q6：核心速记主线有哪些？**

- yield from 替代嵌套 for+yield，展平列表一行搞定

- 三重作用：简化嵌套、双向通道透传 send/throw/close、捕获 return 值

- 双向通道是 Python 协程（asyncio 前身）底层机制

- result = yield from sub() 捕获子生成器 return 值



**口诀**

A：yield from 替 for 循环，展平嵌套一行完；

三重用处要记全，简化嵌套是首端；

send throw 穿透子生成器，双向通道协程源；

return 值被它捕获，汇总结果不费难。



## 相关链接





- 📋 目录：[[00-Python]]



- 📚 学习清单：[[八股文学习路线图]]



- 🔗 [[语言与框架/Python/八股/并发/10-生成器与yield原理|生成器与yield原理]]



- 🔗 [[语言与框架/Python/八股/并发/11-send双向通信|send双向通信]]



- 🔗 [[计算机基础/操作系统/01-进程vs线程vs协程对比|进程vs线程vs协程]] — 操作系统视角的并发基础



- 🔗 [[语言与框架/TypeScript-React/八股/JavaScript/15-异步与事件循环|JS异步与事件循环]] — Python asyncio vs JS 事件循环



- 🔗 [[语言与框架/TypeScript-React/八股/React/04-为什么需要Hooks|React Hooks]] — 生成器/协程与 Hooks 的设计理念对比




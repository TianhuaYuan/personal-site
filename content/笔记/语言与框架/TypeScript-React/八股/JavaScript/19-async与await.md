---

title: "async与await语法糖"

created: "2025-07-12"

tags:

  - 八股文

  - javascript

  - react-ts-js

---

# async与await语法糖

## 一、经典输出题逐题拆解
### 题1：setTimeout + 同步

```javascript

console.log('a');

setTimeout(function() { console.log('b'); }, 0);

console.log('c');

// 输出：a c b

// 拆：a、c 同步先跑完，b 是宏任务最后才轮到

```

### 题2：引入微任务（最经典）

```javascript

console.log('1');                        // ① 同步

setTimeout(function() {

    console.log('2');                    // ⑥ 宏任务

}, 0);

Promise.resolve().then(function() {

    console.log('3');                    // ⑤ 微任务

});

console.log('4');                        // ② 同步

// 输出顺序：1 4 3 2

```

**六步法拆：**

```mermaid

sequenceDiagram

  participant Screen as 屏幕

  participant Sync as 同步执行

  participant MicroQ as 微任务队列

  participant MacroQ as 宏任务队列

  Note over Screen,MacroQ: 步骤①② 执行同步 登记异步

  Sync->>Screen: '1' 同步输出

  Sync->>MacroQ: setTimeout回调进队列<br/>队列: [宏:'2'回调]

  Sync->>MicroQ: Promise.then回调进队列<br/>队列: [微:'3'回调]

  Sync->>Screen: '4' 同步输出

  Note over Screen: 此时屏幕: 1 4

  Note over Screen,MacroQ: 步骤③④ 清空微任务

  MicroQ->>Screen: '3'回调出列 → 输出3

  Note over Screen: 此时屏幕: 1 4 3

  Note over Screen,MacroQ: 步骤⑤ 取下一个宏任务

  MacroQ->>Screen: '2'回调出列 → 输出2

  Note over Screen: 最终: 1 4 3 2

```

> **结论**：微任务 3 排在宏任务 2 前面执行。这就是\"微任务优先于宏任务\"的直观体现。

### 题3：多个微任务 + 多个宏任务

```javascript

console.log('start');

setTimeout(function() {                               // 宏A

    console.log('timeout1');

    Promise.resolve().then(function() {              // 宏A中产生的微任务

        console.log('microtask1');

    });

}, 0);

Promise.resolve().then(function() {                  // 微B

    console.log('then1');

}).then(function() {                                 // 微C（链式，要等微B完成才入队）

    console.log('then2');

});

console.log('end');

// 输出：start end then1 then2 timeout1 microtask1

```

**拆解（关键在于微任务一轮全清空）：**

```mermaid

sequenceDiagram

  participant Sync as 同步

  participant Screen as 屏幕

  participant Micro as 微任务队列

  participant Macro as 宏任务队列

  Note over Sync,Macro: 同步阶段

  Sync->>Screen: start, end 输出

  Sync->>Macro: 宏A入队 [宏A]

  Sync->>Micro: 微B入队 [微B]<br/>微C等微B完才入队

  Note over Sync,Macro: 清空微任务

  Micro->>Screen: 微B执行 → then1

  Micro->>Micro: 微B的.then → 微C入队

  Micro->>Screen: 微C执行 → then2

  Note over Screen: 屏幕: start end then1 then2

  Note over Sync,Macro: 取宏任务

  Macro->>Screen: 宏A执行 → timeout1

  Macro->>Micro: 产生新微任务 microtask1入队

  Note over Sync,Macro: 当前宏任务结束 → 清空微任务

  Micro->>Screen: microtask1执行 → microtask1

  Note over Screen: 最终: start end then1 then2 timeout1 microtask1

```

> **核心规律**：每一轮宏任务结束后，立刻清空所有当时积累的微任务，再取下一个宏任务。`then2` 虽然是第二层 `.then` 产生的，但它仍属于\"清空微任务\"这一轮，会赶在下一个宏任务 `timeout1` 前面执行。

### 题4：async/await 综合题

```javascript

async function async1() {

    console.log('async1 start');           // 同步（async函数体到await前都是同步）

    await async2();                        // await会暂停，后续代码变微任务

    console.log('async1 end');             // 微任务（await之后）

}

async function async2() {

    console.log('async2');                 // 同步

}

console.log('script start');               // 同步

setTimeout(function() {

    console.log('setTimeout');             // 宏任务

}, 0);

async1();                                   // 调用，见下拆解

new Promise(function(resolve) {

    console.log('promise');                // 同步（构造函数同步）

    resolve();

}).then(function() {

    console.log('then');                   // 微任务

});

console.log('script end');                 // 同步

// 输出顺序：

// script start → async1 start → async2 → promise → script end

// → async1 end → then → setTimeout

```

**完整拆解（这道题必会）：**

```mermaid

sequenceDiagram

  participant S as 同步

  participant Screen as 屏幕

  participant Micro as 微任务队列

  participant Macro as 宏任务队列

  Note over S,Macro: 同步阶段

  S->>Screen: 'script start'

  S->>Macro: setTimeout回调入队

  S->>Screen: async1中 'async1 start'

  S->>Screen: async2中 'async2'

  S->>Micro: await → 'async1 end'入队

  S->>Screen: new Promise → 'promise'

  S->>Micro: .then回调入队

  S->>Screen: 'script end'

  Note over Screen: 屏幕: script start async1 start async2 promise script end

  Note over Micro: 微任务: [async1 end回调, then回调]

  Note over Macro: 宏任务: [setTimeout回调]

  Note over S,Macro: 清空微任务

  Micro->>Screen: async1 end → async1 end

  Micro->>Screen: then → then

  Note over Screen: 屏幕: ... async1 end then

  Note over S,Macro: 取宏任务

  Macro->>Screen: setTimeout → setTimeout

  Note over Screen: 最终: script start async1 start async2 promise script end async1 end then setTimeout

```

> **`await` 的本质**：`await x` 后面的代码，等价于放到 `x.then(...)` 里，所以是微任务。上面题里 `async1 end` 是微任务，`then` 也是微任务，两者按入队先后顺序执行（`await` 那句在 `new Promise` 之前，所以 `async1 end` 先于 `then`）。

### 题5：嵌套 setTimeout（区分轮次）

```javascript

setTimeout(function() {                    // 宏A

    console.log('outer');

    setTimeout(function() {                // 宏B（宏A中产生的）

        console.log('inner');

    }, 0);

}, 0);

// 输出：outer inner

// 但要明白：inner 属于\"下一轮\"宏任务

```

```mermaid

graph TD

  R1[第1轮宏任务<br/>宏A → 输出outer → 注册宏B] -->|清微任务: 无| R2[第2轮宏任务<br/>宏B → 输出inner]

```

> 内层 `setTimeout` 的回调是新的一轮宏任务，不会和它所在的 `outer` 回调同轮。这点在判断\"几次渲染\"\"几次轮询\"时很关键。

---

## 速记卡（面试闪卡）

**Q1：一句话讲清「async与await语法糖」到底是什么？**

A：async/await 是 Promise 的语法糖，把回调地狱变成看起来同步的顺序代码。

**Q2：一、六步拆解法 —— 怎么理解？**

A：输出题用六步：① 标同步先跑；② 标宏任务（setTimeout）和微任务（Promise.then）；③ 每轮先同步、再清空微任务、再取一个宏任务；④ await 后代码进微任务；⑤ 嵌套 setTimeout 分轮次；⑥ 画时间线就不乱。

**Q3：二、同步→微→宏的总顺序 —— 怎么理解？**

A：题1（setTimeout+同步）：同步先打，setTimeout 回调最后。题2（引入微任务最经典）：同步跑完清微任务，所以 .then 里的内容排在 setTimeout 之前。铁律：同步 → 微任务 → 宏任务。

**Q4：三、await 等价于 .then —— 怎么理解？**

A：题4 综合：await xxx 后面的代码等价于 xxx.then(...) 里的回调，所以被切成微任务，排在同步之后、setTimeout 之前。async 函数本身返回 Promise，不影响主时间线。

**Q5：四、嵌套 setTimeout 分轮次 —— 怎么理解？**

A：题5：里层 setTimeout 在外层回调执行时才注册，属于"下一轮"宏任务。外层 0ms、里层 0ms 实际是：第1轮跑外层 → 清微任务 → 第2轮跑里层。别以为都是 0ms 就一起跑。

**Q6：核心速记主线有哪些？**

- async/await 是 Promise 语法糖

- 总顺序：同步 → 微任务 → 宏任务

- await 后面的代码是微任务

- 每轮宏任务结束清空全部微任务

- 嵌套 setTimeout 要区分轮次

**口诀**

A：输出题用六步拆，

同步先跑莫发呆；

一轮宏完清微任，

await 微任务排在前来。

## 相关链接

- 📋 目录：[[00-JavaScript]]

- 📚 学习清单：[[八股文学习路线图]]

- 🔗 [[语言与框架/Python/八股/并发/15-async-await本质|Python async与await]] — Python和JS的async/await对比

- 🔗 [[语言与框架/TypeScript-React/八股/React/04-为什么需要Hooks|React Hooks]] — Hooks和async/await都是简化异步模式的语法糖


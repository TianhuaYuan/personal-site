---

title: "this指向"

created: "2025-07-12"

tags:

  - 八股文

  - javascript

  - react-ts-js

---

# this指向

## 七、`this` 指向
### 7.1 this 是什么？

**`this` 是函数执行时的"上下文对象"，在调用时决定，不在定义时决定。**

这和 Python 的 `self` 完全不同——`self` 是方法的显式第一个参数，永远指向实例本身。JS 的 `this` 是隐式的，取决于"谁调用了这个函数"。

```python

# Python：self 永远是实例自己（显式参数）

class Person:

    def say(self):

        print(self.name)    # self = 调用的实例，永远是它，不会变

```

```javascript

// JS：this 取决于调用方式（隐式绑定）

function say() {

    console.log(this.name);  // this = ? 看你怎么调

}

// this 不是固定的，是根据调用方式动态决定的

```

### 7.2 五种 this 绑定规则
#### 规则一：默认绑定 —— 裸调 `fn()`

```javascript

function fn() {

    console.log(this);

}

fn();   // 浏览器: window（全局对象）  |  严格模式: undefined

//      ↑ 前面什么都不写直接调 → this = window（浏览器） 或 undefined（严格模式）

```

**记忆**：`fn()` —— 函数前面什么都没有 → this = window / undefined。

#### 规则二：隐式绑定 —— `obj.fn()` 谁调用指向谁

```javascript

const obj = {

    name: "张三",

    sayName: function() {

        console.log(this.name);   // this 看"谁调用"

    },

};

obj.sayName();  // "张三" —— obj 调的，this = obj ✅

```

**⚠️ 隐式丢失（最爱考）**：

```javascript

const obj = {

    name: "张三",

    sayName: function() {

        console.log(this.name);

    },

};

// 正常调用：点前面有对象

obj.sayName();               // "张三" —— this = obj ✅

// 赋给变量再调：点前面什么都没了

const fn = obj.sayName;      // 只把函数本身拿出来，不绑定 obj

fn();                        // undefined —— 裸调，this = window ❌

// 作为回调传进去：回调就是裸调

setTimeout(obj.sayName, 0);  // undefined —— setTimeout 里就是 fn()，裸调 ❌

```

**速记口诀**：方法一旦脱离对象单独调用，`this` 就丢了（回退到默认绑定）。就像手机脱离主人——谁都联系不上。

#### 规则三：显式绑定 —— `call` / `apply` / `bind`

```javascript

const person = { name: "张三" };

function greet(greeting, punctuation) {

    console.log(`${greeting}, ${this.name}${punctuation}`);

}

// call：改 this + 立即执行，参数逐个传

greet.call(person, "你好", "！");       // "你好, 张三！"

// apply：改 this + 立即执行，参数用数组传

greet.apply(person, ["早上好", "~"]);   // "早上好, 张三~"

// bind：改 this + 返回新函数，不立即执行（适合做回调）

const boundFn = greet.bind(person, "晚上好");

boundFn("。");                          // "晚上好, 张三。"

```

| 方法 | 是否立即执行 | 参数形式 | 常用场景 |
| :--- | :--- | :--- | :--- |
| `call` | ✅ 立即 | 逐个逗号传：`fn.call(ctx, a, b, c)` | 临时改 this 调用 |
| `apply` | ✅ 立即 | 数组传：`fn.apply(ctx, [a, b, c])` | 参数已经是数组时 |
| `bind` | ❌ 不立即 | 逐个逗号传：`fn.bind(ctx, a, b)` | 做回调、事件处理 |

#### 规则四：`new` 绑定 —— 构造函数

```javascript

// 构造函数（首字母大写是约定）

function Student(name, age) {

    this.name = name;       // new 调用时，this = 新创建的空对象

    this.age = age;

}

const s = new Student("张三", 20);

console.log(s);  // { name: "张三", age: 20 }

// new 做了四件事：

// ① 创建一个空对象 {}

// ② 把这个空对象和构造函数关联（原型链）

// ③ 把构造函数的 this 指向这个空对象，执行函数体

// ④ 返回这个对象

```

#### 规则五：箭头函数 —— 没有自己的 `this`

```javascript

const obj = {

    name: "张三",

    normal: function() {

        console.log(this.name);        // this = obj ✅

    },

    arrow: () => {

        console.log(this.name);        // 箭头函数没有自己的 this

    },                                 // 去外层找 → window → undefined ❌

};

obj.normal();  // "张三"

obj.arrow();   // undefined

```

**箭头函数的 this 怎么看**：找到箭头函数**定义的位置**，往外看第一层非箭头函数，它的 this 就是箭头函数的 this。

```javascript

// React 里箭头函数的好处：自动捕获外层的 this

const obj = {

    name: "张三",

    init: function() {                   // 普通函数，this = obj

        setTimeout(() => {               // 箭头函数，没有自己的 this

            console.log(this.name);      // 往外找 → init 的 this = obj → "张三" ✅

        }, 1000);

    },

};

obj.init();  // "张三" —— 箭头函数救了 this

```

### 7.3 优先级：`new > 显式 > 隐式 > 默认`

```javascript

function Foo() { this.name = "foo"; }

const obj = {};

const BoundFoo = Foo.bind(obj);   // bind 强绑 this = obj

const instance = new BoundFoo();  // new 覆盖 bind

console.log(instance.name);  // "foo" —— new 赢了 ✅

console.log(obj.name);       // undefined —— bind 被 new 覆盖

```

---

## 速记卡（面试闪卡）

**Q1：一句话讲清「this指向」到底是什么？**

A：JS的this在调用时决定（不像Python的self定义时定），绑定规则有五种：默认、隐式、显式call/apply/bind、new、箭头函数——谁调用、怎么调用决定它指向谁。

**Q2：this 不是 self：调用时才定 —— 怎么理解？**

A：Python 的 self 是定义方法时显式写死的第一个参数，永远指向实例；JS 的 this（Context Object）是隐式的，取决于"谁调用了函数"。同一个函数不同调法 this 不同。

**Q3：默认与隐式：裸调 vs 谁调用 —— 怎么理解？**

A：默认绑定（Default Binding）：fn() 前面啥都没有，this = window（非严格）或 undefined（严格）。隐式绑定（Implicit）：obj.fn() 谁调用指向谁——但把方法单独取出再调，隐式就丢了。

**Q4：显式绑定：call/apply/bind 强行指定 —— 怎么理解？**

A：显式绑定（Explicit Binding）用 fn.call(thisArg)、fn.apply(thisArg, args) 立刻执行并指定 this；fn.bind(thisArg) 返回一个永久绑死 this 的新函数。三者是"强行指定 context"的三板斧。

**Q5：new 与箭头：特殊两兄弟 —— 怎么理解？**

A：new 绑定：new Fn() 时 this 指向新创建的对象。箭头函数（Arrow Function）没有自己的 this，它捕获外层词法作用域的 this，且 call/apply 也改不了——这是它和常规函数最大区别。

**Q6：核心速记主线有哪些？**

- this 在调用时决定，非定义时（区别于 self）

- 默认绑定 fn()→window/undefined；隐式 obj.fn()→obj

- 显式 call/apply/bind 强行指定 this

- new 指向新对象；箭头函数捕获外层 this

**口诀**

A：this 调用时才定

裸调 window 隐式主

显式三板强指定

箭头捕获外层域

## 相关链接

- 📋 目录：[[00-JavaScript]]

- 📚 学习清单：[[八股文学习路线图]]


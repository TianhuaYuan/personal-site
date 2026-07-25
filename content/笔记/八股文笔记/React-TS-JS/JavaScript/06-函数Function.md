---
title: "函数Function"
created: "2025-07-12"
tags:
  - 八股文
  - javascript
  - react-ts-js
---

## 五、函数（Function）



### 5.1 三种定义方式



```javascript

// ===== 方式一：function 声明（最传统）=====

function add(a, b) {           // function 关键字 + 函数名 + (参数) + { 函数体 }

    return a + b;              // return 返回值，和 Python 完全一样

}



// ===== 方式二：函数表达式（匿名函数赋给变量）=====

const multiply = function(a, b) {  // 没有函数名，赋给变量 multiply

    return a * b;

};



// ===== 方式三：箭头函数（ES6，最简洁）=====

const divide = (a, b) => {         // 箭头 = 大于号，读作 "goes to" → "a b goes to..."

    return a / b;

};



// 三种方式调用完全一样：

add(1, 2);       // 3

multiply(1, 2);  // 2

divide(1, 2);    // 0.5

```



```python

# Python 对照

def add(a, b):       # JS function add(a, b) {}

    return a + b     # JS return a + b;



multiply = lambda a, b: a * b   # JS const multiply = (a, b) => a * b;

# lambda ≈ 箭头函数（但 JS 箭头函数还有 this 的差异，后面讲）

```



### 5.2 函数参数怎么写



```javascript

// ===== ① 普通参数（和 Python 一样）=====

function add(a, b) {            // a, b 是形参

    return a + b;

}

add(1, 2);                      // 1, 2 是实参 → 3



// ===== ② 默认参数（Python 也有的功能）=====

function greet(name, greeting = "你好") {   // greeting 有默认值

    return `${greeting}, ${name}！`;

}

greet("张三");                  // "你好, 张三！" —— 不传就是用默认值

greet("张三", "早上好");         // "早上好, 张三！" —— 传了就用传的



// ===== ③ 剩余参数 ...args（= Python 的 *args）=====

function sum(...nums) {         // ...nums 把传进来的所有参数装进数组

    return nums.reduce((a, b) => a + b, 0);

    // reduce：把数组归约成一个值，0 是初始值

}

sum(1, 2);          // 3

sum(1, 2, 3, 4);    // 10

sum();              // 0 —— 空数组，返回初始值



// ===== ④ JS 的参数很"宽容"——少传多传都不报错 =====

function show(a, b) {

    console.log("a:", a, "b:", b);

}

show(1);            // a: 1  b: undefined —— 少传的参数是 undefined

show(1, 2, 3);      // a: 1  b: 2 —— 多传的直接忽略



// Python 对照：

// def show(a, b):

//     print(f"a: {a} b: {b}")

// show(1)       # ❌ TypeError: missing required argument

// show(1, 2, 3) # ❌ TypeError: too many arguments

// ↑ JS 比 Python 宽容得多，少传是 undefined，多传被忽略

```



### 5.3 方法（Method）—— 对象里的函数



**方法 = 写在对象里的函数。** 写法差不多，但多了 `this` 指向的坑。



你每天写的 `console.log("hello")` 就是方法调用——`console` 是内置对象，`.log` 是它上面的方法。和 `user.sayHi()` 完全一样的形式。



```python

# Python 对照：

# print("hello")        # Python 的函数，直接调，不需要对象.方法()

# console.log("hello")  # JS 的 console 对象 + log 方法，必须是 对象.方法()

```



```javascript

// ===== 定义方法：在对象里写函数 =====

const user = {

    name: "张三",

    age: 20,



    // 方法就是对象里的函数——属性名后面跟 function

    sayHi: function() {              // 完整写法

        console.log("你好, " + this.name);

    },



    // ES6 方法简写：丢掉 : function

    getAge() {                       // 等价于 getAge: function()

        return this.age;

    },

};



user.sayHi();     // "你好, 张三" —— 调用方式和 Python 一样：对象.方法()

user.getAge();    // 20

```



```python

# Python 对照（写在 class 里）

class User:

    def __init__(self, name, age):

        self.name = name

        self.age = age



    def say_hi(self):              # JS: sayHi: function() { ... }

        print(f"你好, {self.name}")

```



**关键差异**：

- Python 必须写在 `class` 里 —— JS 方法直接写在 `{}` 对象里就行

- Python 的 `self` 是显式参数 —— JS 的 `this` 是隐式的

- JS 的对象 + 方法 ≈ Python 的实例（没有 class 也行）



**"函数"和"方法"在调用上有区别吗？没有。** 都是 `名字()`，语法完全一样。区别只在于 `this`：

- 独立调用 `greet()` → this = window / undefined

- 对象调用 `user.sayHi()` → JS 自动把 this 指向 user

- 把方法抠出来再调 `const fn = user.sayHi; fn()` → this 又丢了（隐式丢失，见 5.8）



### 5.4 对象的简写语法（ES6）



React 里非常常见，看到缩写要知道在干什么。



```javascript

// ===== ① 属性值简写 → 同名变量自动填 =====

const name = "张三";

const age = 20;



// 老写法：

const user1 = { name: name, age: age };  // 啰嗦



// ES6 简写：属性名和变量名一样时，写一个就行

const user2 = { name, age };             // 等价于 { name: name, age: age }



// ===== ② 方法简写 → 丢掉 : function =====

// 老写法：

const obj1 = {

    sayHi: function() { console.log("hi"); },

};



// ES6 简写：

const obj2 = {

    sayHi() { console.log("hi"); },        // 丢掉 : function，一样的效果

};



// ===== ③ 计算属性名 → key 可以是变量 =====

const key = "email";

const person = {

    [key]: "zs@qq.com",       // [key] 不是真写 "key"，是用变量 key 的值

};

console.log(person.email);    // "zs@qq.com"



// React 里常见：多个 state 变量直接塞进对象

// const [name, setName] = useState("");

// const [age, setAge] = useState(0);

// const user = { name, age };  ← 就是简写

```



### 5.5 箭头函数的简写规则



```javascript

// ===== 完整写法 =====

const add = (a, b) => {

    return a + b;

};



// ===== 简写一：函数体只有一行 return，可以省略 {} 和 return =====

const add = (a, b) => a + b;



// ===== 简写二：只有一个参数，可以省略 () =====

const square = n => n * n;       // 等价于 (n) => n * n



// ===== 简写三：返回对象字面量，要加 () =====

const getUser = () => ({ name: "张三", age: 20 });

//                    ↑ 不加这个 ()，{} 会被当成函数体而不是对象



// ===== 无参数必须写 () =====

const sayHi = () => console.log("hi");

```



### 5.6 函数是一等公民（值）



```javascript

// 函数也是一种"值"——可以赋给变量、当参数传、当返回值返回



// ① 赋给变量

function greet() { return "hello"; }

const sayHello = greet;        // 把函数本身赋给 sayHello（不是调用结果）

console.log(sayHello());       // "hello" —— sayHello 和 greet 是同一个函数



// ② 当参数传（回调函数）

function run(fn) {             // fn 是一个函数参数

    fn();                      // 调用传进来的函数

}

run(function() { console.log("执行了"); });  // 传匿名函数进去



// ③ 当返回值返回（闭包的基础）

function outer() {

    return function() {        // 返回一个函数

        return "inner";

    };

}

const inner = outer();         // inner 是 outer 返回的函数

console.log(inner());          // "inner"

```



**Python 对照**：Python 里函数也是一等公民，`def greet(): ...` 也可以 `say_hello = greet`。完全一样。



### 5.7 回调函数（Callback）—— React 里天天写



```javascript

// 回调函数 = 把一个函数当作参数传给另一个函数，让它在合适的时机"回头调"你



// 例1：setTimeout —— 过一段时间执行

setTimeout(function() {                   // 这个匿名函数就是"回调"

    console.log("1 秒后执行");

}, 1000);                                 // 1000 毫秒 = 1 秒



// 例2：数组方法 .map / .filter / .find —— 每个都接收回调函数

[1, 2, 3].map(function(n) {              // 这个匿名函数就是"回调"

    return n * 2;

});



// 例3：React 事件处理 —— 你每天都在写的

<button onClick={function() { console.log("点击了"); }}>

//    ↑ onClick 属性值是一个回调函数，React 在按钮被点击时调用它



// 箭头函数写法更简洁：

<button onClick={() => console.log("点击了")}>

```



### 5.8 箭头函数和普通函数的 this 差异



**这是 JS 最重要的坑之一：箭头函数没有自己的 `this`。**



```javascript

// ===== 普通函数：this 看"谁调用" =====

const obj1 = {

    name: "张三",

    say: function() {                    // 普通函数

        console.log(this.name);          // this = 调用者

    },

};

obj1.say();  // "张三" —— obj1 调的，this = obj1 ✅



// ===== 箭头函数：没有自己的 this，去外层找 =====

const obj2 = {

    name: "李四",

    say: () => {                         // 箭头函数

        console.log(this.name);          // this 不是 obj2！去外层找 → window

    },

};

obj2.say();  // undefined —— 箭头函数的 this 往外层找，外层是 window ❌

```



**速记**：普通函数 `function() {}` → this 看调用者；箭头函数 `() => {}` → this 看外层。



---





## 相关链接



- 📋 目录：[[00-JavaScript]]

- 📚 学习清单：[[八股文学习清单]]




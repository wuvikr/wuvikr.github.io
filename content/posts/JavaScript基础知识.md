+++
date = '2026-10-20T17:13:23+08:00'
draft = false
title = 'JavaScript基础知识'
categories = ['Coding']
tags = ['JavaScript', '基础知识']
+++

# 1. JavaScript 是什么

JavaScript（简称 JS）是一门**动态类型、多范式**的编程语言。它最初为浏览器而生，如今已覆盖浏览器、服务器（Node.js）、桌面应用、移动端等环境。

几个关键特征：

- **解释执行**：不需要编译，引擎直接逐行运行。
- **动态类型**：变量类型在运行时确定，无需声明。
- **一等函数**：函数可以像值一样传递、赋值、作为参数。
- **基于原型**：对象继承通过原型链实现，而非传统类继承（ES6 提供了 `class` 语法糖）。

> **注意**：JavaScript 与 Java 是两门完全不同的语言，仅名字有历史渊源关系。

---

# 2. 运行环境

写 JS 代码前，需要一个运行环境。

## 2.1 浏览器环境

打开任意浏览器（Chrome / Edge / Firefox），按 `F12` 打开**开发者工具**，切换到 **Console**（控制台）标签，即可直接输入并运行 JS 代码：

```javascript
a="zhangsan";
console.log("Hello ",a);
```

也可以在 HTML 文件中通过 `<script>` 标签引入：

```html
<!DOCTYPE html>
<html>
<body>
  <script>
    console.log("页面加载时执行");
  </script>
</body>
</html>
```

## 2.2 Node.js 环境

[Node.js](https://nodejs.org/) 是一个免费、开源、跨平台的 JavaScript 运行时环境，它让开发人员能够创建服务器、 Web 应用、命令行工具和脚本。

安装后，在终端执行：

```bash
node -v          # 查看版本
node hello.js    # 运行文件
```

```javascript
// hello.js
console.log("Hello from Node.js");
```

> **注意**：浏览器和 Node.js 共享 ECMAScript 核心语法，但宿主 API 不同。浏览器有 `window`、`document`；Node.js 有 `fs`、`path`、`process`。

---

# 3. 变量与常量

## 3.1 三种声明方式

```javascript
var oldWay = 1;      // 旧写法，函数作用域，存在变量提升，现代代码不推荐
let way = 2;         // 块级作用域，可重新赋值
const PI = 3.14;     // 块级作用域，不可重新赋值（推荐默认用 const）
```

| 关键字 | 作用域 | 重新赋值 | 变量提升 | 推荐度 |
|--------|--------|---------|--------|--------|
| `var`  | 函数级 | 是        | 是      | 不推荐 |
| `let`  | 块级   | 是        | 否（有暂时性死区） | 需要重新赋值时使用 |
| `const`| 块级   | 否        | 否      | 默认首选 |

## 3.2 暂时性死区（TDZ）

暂时性死区（TDZ）指 `let`、`const`、`class` 声明的变量从进入作用域开始到执行到声明语句之前不可访问的区间，访问会直接抛出 `ReferenceError`。

声明的变量在声明前不可访问：

```javascript
console.log(a);  // ReferenceError: Cannot access 'a' before initialization
let a = 1;
```
> [!TIP] 提示：
> 默认用 `const`，需要重新赋值时用 `let`，避免使用 `var`；变量声明尽量靠近首次使用的位置，减少 TDZ 带来的困惑。

`const` 声明的对象，其内部属性仍可修改，不可变的是绑定，不是值本身：
```javascript
const user = { name: "Tom" };
user.name = "Jerry";   // 合法，修改了属性
user = {};             // TypeError: 不能重新赋值
```

## 3.3 命名规则
- 只能包含字母、数字、下划线、`$`，不能以数字开头。
- 区分大小写（`age` ≠ `Age`）。
- 建议使用 **camelCase**（小驼峰）：`userName`。
- 避免使用保留字（如 `class`、`function`、`return`）。

---

# 4. 基础数据类型

JS 共有 **7 种原始类型** + **1 种对象类型**：

## 4.1 原始类型（Primitive）

| 类型 | 示例 | 说明 |
|------|------|------|
| `Number` | `42`, `3.14`, `-0.5` | 所有数字都是 64 位浮点数，不分整数和浮点 |
| `String` | `"hello"`, `'world'`, `` `hi` `` | 字符串不可变 |
| `Boolean` | `true`, `false` | 布尔值 |
| `null` | `null` | 表示"空值"，需手动赋值 |
| `undefined` | `undefined` | 未赋值的变量默认值 |
| `Symbol` | `Symbol("id")` | 唯一标识符，ES6 新增 |
| `BigInt` | `9007199254740991n` | 任意精度整数，ES2020 新增 |

## 4.2 对象类型（Object）

```javascript
const obj = { name: "Tom", age: 25 };
const arr = [1, 2, 3];
const fn = function () {};
```

数组、函数本质上也都是对象。

## 4.3 typeof 运算符

```javascript
typeof 42;           // "number"
typeof "hello";      // "string"
typeof true;         // "boolean"
typeof undefined;    // "undefined"
typeof null;         // "object"  ← 历史遗留 bug，不要依赖它判断 null
typeof Symbol();     // "symbol"
typeof 10n;          // "bigint"
typeof {};           // "object"
typeof [];           // "object"  ← 数组也是 object
typeof function (){}; // "function"
```

> [!NOTE] 备注：
> - `typeof null === "object"` 是 JavaScript 的历史遗留问题，判断 null 请用 `x === null`。
> - 判断数组用 `Array.isArray(arr)`，不要用 `typeof`。

## 4.4 null 与 undefined 的区别

- `undefined`："此处应有值，但还没赋"——系统默认值。
- `null`："此处明确为空"——开发者手动设置的值。

```javascript
let x;
console.log(x);        // undefined：声明了但没赋值

const y = null;        // 明确表示"空"
console.log(y);        // null
```

---

# 5. 类型转换
JS 在运算时会自动转换类型（隐式转换），也可以手动转换（显式转换）。

## 5.1 显式转换

```javascript
// 转字符串
String(123);        // "123"
(123).toString();   // "123"

// 转数字
Number("123");      // 123
Number("12abc");    // NaN
parseInt("12px");   // 12
parseFloat("3.14"); // 3.14

// 转布尔
Boolean(1);         // true
Boolean(0);         // false
```

## 5.2 Falsy 值（转为 false 的值）

只有以下 6 个值在布尔上下文中为 `false`：

```javascript
if (false)
if (0)
if ("")       // 空字符串
if (null)
if (undefined)
if (NaN)
```
其余一切都是 `truthy`，包括 `[]`、`{}`、`"0"`、`"false"`，这是常见的坑。

## 5.3 == 与 === 的区别

- `==`：会做类型转换后比较。
- `===`：不做类型转换，类型不同直接返回 `false`。

```javascript
0 == "";        // true  （隐式转换）
0 === "";       // false
null == undefined;  // true
null === undefined; // false
```

> [!TIP] 提示：
> 建议始终使用 `===` 和 `!==`，避免 `==` 带来的意外。

---

# 6. 运算符

## 6.1 算术运算符

```javascript
let a = 10 + 5;    // 15
let b = 10 - 5;    // 5
let c = 10 * 5;    // 50
let d = 10 / 3;    // 3.3333333333333335
let e = 10 % 3;    // 1（取余）
let f = 2 ** 3;    // 8（幂运算，ES2016）

a++;  // 后置自增：先用后加
++a;  // 前置自增：先加后用
```

## 6.2 比较运算符

```javascript
5 > 3;        // true
5 >= 5;       // true
5 < 3;        // false
5 === "5";    // false
5 !== "5";    // true
```

## 6.3 逻辑运算符

```javascript
true && false;   // false（与：两边都真才真）
true || false;   // true（或：一边真就真）
!true;           // false（非）
```

## 6.4 空值合并运算符（ES2020）

```javascript
const name = null ?? "匿名";   // "匿名"：左侧为 null/undefined 时取右侧
const age = 0 ?? 18;           // 0：左侧不是 null/undefined，直接取左侧
```

> [!NOTE] 备注：
> `??` 与 `||` 的区别：`||` 会把所有 falsy 值（`0`、`""`、`false`）都当作空，而 `??` 只认 `null` 和 `undefined`。

## 6.5 可选链运算符（ES2020）
可选链运算符 `?.` 是 ES2020 引入的语法，用于安全访问可能为 `null` 或 `undefined` 的对象属性、数组元素或函数调用。

核心机制是：如果 `?.` 左侧的引用是 `null` 或 `undefined`，表达式会短路并直接返回 `undefined`，而不是抛出 `TypeError`。

```javascript
// 访问对象属性
const user = { profile: { name: "Tom" } };
console.log(user?.profile?.name);    // "Tom"
console.log(user?.address?.city);    // undefined（不报错）

// 访问数组或动态属性
const first = arr?.[0];
const val = obj?.[key];

// 调用函数
obj.method?.();  // 如果 obj.method 不存在，返回 undefined 而不是报错
```

对比传统写法：
```javascript
// 以前要这样层层判断
const name = user && user.profile && user.profile.name;

// 现在一行搞定
const name = user?.profile?.name;
```
> [!NOTE] 备注：
> 和 `??` 一样，`?.` 也只对 `null` 和 `undefined` 短路，对其他 falsy 值（`0`、`""`、`false`）不会短路，会继续正常访问。

---

# 7. 字符串

## 7.1 创建方式

```javascript
const s1 = '单引号';
const s2 = "双引号";
const s3 = `模板字符串`;   // 反引号，支持换行和插值
```

## 7.2 模板字符串

```javascript
const name = "Tom";
const age = 25;

// 插值
console.log(`我叫${name}，今年${age}岁`);

// 多行字符串
const multi = `第一行
第二行
第三行`;
```

## 7.3 常用方法

```javascript
const str = "Hello, World";

str.length;              // 12
str.toUpperCase();       // "HELLO, WORLD"
str.toLowerCase();       // "hello, world"
str.indexOf("World");    // 7（找不到返回 -1）
str.slice(0, 5);         // "Hello"
str.substring(0, 5);     // "Hello"
str.replace("World", "JS"); // "Hello, JS"
str.split(", ");         // ["Hello", "World"]
str.trim();              // 去除首尾空格
str.includes("Hello");   // true
str.startsWith("Hello"); // true
str.endsWith("World");   // true
str.charAt(0);           // "H"
str[0];                  // "H"
```

**注意**：字符串是不可变的，所有方法都返回新字符串，原字符串不变。

```javascript
const s = "abc";
s.toUpperCase();   // "ABC"
console.log(s);   // "abc"（原字符串没变）
```

---

# 8. 条件判断

## 8.1 if / else

```javascript
const score = 85;

if (score >= 90) {
  console.log("优秀");
} else if (score >= 60) {
  console.log("及格");
} else {
  console.log("不及格");
}
```

## 8.2 三元运算符

```javascript
const age = 20;
const status = age >= 18 ? "成年" : "未成年";
```

## 8.3 switch

```javascript
const day = 3;

switch (day) {
  case 1:
    console.log("周一");
    break;          // 必须写 break，否则会贯穿执行
  case 2:
    console.log("周二");
    break;
  case 3:
  case 4:
  case 5:
    console.log("工作日");
    break;
  default:
    console.log("周末");
}
```

> [!CAUTION] 注意:
> `switch` 使用严格相等（`===`）比较。忘记写 `break` 会导致"穿透"（fall through），后续 case 也会执行。

---

# 9. 循环

## 9.1 for 循环

```javascript
for (let i = 0; i < 5; i++) {
  console.log(i);   // 0 1 2 3 4
}
```

## 9.2 while 循环

```javascript
let i = 0;
while (i < 3) {
  console.log(i);
  i++;
}
```

## 9.3 do...while 循环

```javascript
let i = 0;
do {
  console.log(i);   // 至少执行一次
  i++;
} while (i < 3);
```

## 9.4 for...of（遍历可迭代对象的值）

```javascript
const arr = ["a", "b", "c"];
for (const item of arr) {
  console.log(item);   // "a" "b" "c"
}

// 也可遍历字符串
for (const ch of "abc") {
  console.log(ch);     // "a" "b" "c"
}
```

## 9.5 for...in（遍历对象的键）

```javascript
const user = { name: "Tom", age: 25 };
for (const key in user) {
  console.log(key, user[key]);   // "name" "Tom"  "age" 25
}
```

## 9.6 break 与 continue

```javascript
for (let i = 0; i < 10; i++) {
  if (i === 3) continue;   // 跳过本次，进入下一轮
  if (i === 6) break;      // 直接跳出循环
  console.log(i);          // 0 1 2 4 5
}
```

> [!TIP] 提示：
> - `for...of` 遍历值，`for...in` 遍历键。
> - 遍历数组用 `for...of`，不要用 `for...in`，`for...in` 遍历的是键名（索引），还可能遍历到原型上的属性。


---

# 10. 函数

## 10.1 函数声明

```javascript
function greet(name) {
  return "你好，" + name;
}

greet("Tom");   // "你好，Tom"
```

## 10.2 函数表达式

```javascript
const greet = function (name) {
  return "你好，" + name;
};
```

## 10.3 箭头函数（ES6）

```javascript
// 完整写法
const add = (a, b) => {
  return a + b;
};

// 单行可省略 return 和花括号
const add = (a, b) => a + b;

// 单个参数可省略括号
const square = x => x * x;

// 无参数
const hello = () => "Hello";
```

> **注意**：箭头函数没有自己的 `this`，它继承外层作用域的 `this`。这在回调函数中很常见，但也意味着不能用箭头函数作为对象方法或构造函数。

## 10.4 参数默认值

```javascript
function greet(name = "朋友") {
  return "你好，" + name;
}

greet();       // "你好，朋友"
greet("Tom");  // "你好，Tom"
```

## 10.5 剩余参数

```javascript
function sum(...numbers) {
  return numbers.reduce((total, n) => total + n, 0);
}

sum(1, 2, 3, 4);   // 10
```

### 立即执行函数（IIFE）
IIFE 是“定义后立即执行的函数表达式”，本质是借用函数作用域制造一个一次性私有作用域：
```javascript
(function () {
  const private = "外部访问不到";
})();
```
为什么要加一个括号呢？因为 function 写在语句位置会被解析成函数声明，声明不能直接调用。加括号后，它变成函数表达式，后面再跟 `()` 就能立即执行。

现代代码用块级作用域（`{ let ... }`）或模块替代 IIFE。

---

# 11. 作用域与闭包

## 11.1 三种作用域

```
┌─────────────────────────────────┐
│  全局作用域（Global Scope）       │
│  ┌───────────────────────┐      │
│  │  函数作用域（Function） │      │
│  │  ┌─────────────┐     │      │
│  │  │ 块作用域 {}   │     │      │
│  │  └─────────────┘     │      │
│  └───────────────────────┘      │
└─────────────────────────────────┘
```

- **全局作用域**：最外层，任何地方都能访问。
- **函数作用域**：函数内部声明的变量，外部不可见。
- **块级作用域**：`{}` 内部，仅 `let` / `const` 有效。

```javascript
// 全局
const globalVar = "我在全局";

function outer() {
  // 函数作用域
  const outerVar = "我在 outer";

  if (true) {
    // 块级作用域
    let blockVar = "我在块里";
    var notBlockVar = "var 不受块级限制";
  }

  console.log(blockVar);      // ReferenceError
  console.log(notBlockVar);   // "var 不受块级限制"
}
```

## 11.2 闭包（Closure）

闭包是指**函数能够记住并访问其词法作用域**，即使该函数在其词法作用域之外执行。

```javascript
function createCounter() {
  let count = 0;          // 外部无法直接访问

  return function () {
    count++;
    return count;
  };
}

const counter = createCounter();
counter();   // 1
counter();   // 2
counter();   // 3
```
`createCounter` 执行完后，局部变量 `count` 本应被销毁，但返回的内部函数引用着它，所以 `count` 一直存活。这就是闭包：**函数 + 其外部引用的变量环境**。

闭包的常见用途：
- 数据私有化（模拟私有变量）
- 函数工厂
- 事件处理中保持状态

---

# 12. 对象

## 12.1 创建对象

```javascript
// 对象字面量
const user = {
  name: "Tom",
  age: 25,
  sayHi: function () {
    console.log("你好，我是" + this.name);
  }
};

// 简写（ES6）
const user = {
  name: "Tom",
  age: 25,
  sayHi() {           // 方法简写
    console.log(`你好，我是${this.name}`);
  }
};
```

## 12.2 访问属性

```javascript
user.name;        // "Tom"（点语法）
user["name"];     // "Tom"（方括号语法，属性名含空格或动态时用）

const key = "age";
user[key];        // 25（动态属性名）
```

## 12.3 增删属性

```javascript
user.email = "tom@example.com";   // 新增
delete user.age;                  // 删除
```

### this 关键字

`this` 指向**调用函数的对象**，不是函数本身定义的位置。

```javascript
const user = {
  name: "Tom",
  greet: function () {
    console.log(this.name);
  }
};

user.greet();   // "Tom"（this 是 user）

const fn = user.greet;
fn();           // undefined 或报错（this 变成全局对象/undefined）
```

> [!CAUTION] 注意：
> 箭头函数没有自己的 `this`，它捕获定义时外层的 `this`。因此箭头函数不适合作为对象方法。

## 12.4 Object 常用方法

```javascript
Object.keys(user);      // ["name", "age"]（所有键）
Object.values(user);    // ["Tom", 25]（所有值）
Object.entries(user);   // [["name","Tom"], ["age",25]]
Object.assign({}, user, { age: 26 });  // 合并对象
```

## 12.5 解构赋值
解构赋值就是按“结构”从数组或对象里一次性把值取出来，直接赋给变量。
```javascript
const user = { name: "Tom", age: 25 };
const { name, age } = user;

// { name, age } 会按属性名从右边对象里取值。
console.log(name); // "Tom"
console.log(age);  // 25


// 数组解构
const [a, b, c] = [1, 2, 3];
console.log(a, b, c); // 1 2 3

// 跳过元素
const [, , c] = [1, 2, 3];
console.log(c); // 3
```
**常见用途**：
- 交换变量
  ```javascript
  let a = 1, b = 2;
  [a, b] = [b, a];
  ```
- 函数参数解构
  ```javascript
  function greet({ name, age }) {
    console.log(`${name} 今年 ${age} 岁`);
  }

  greet({ name: "Tom", age: 25 });
  ```
- 取函数返回的多个值
  ```javascript
  function getPoint() {
    return [10, 20];
  }

  const [x, y] = getPoint();
  ```

> [!CAUTION] 注意：
> 已声明的变量再解构赋值时，要加括号。
> ```javascript
> let name, age;
> ({ name, age } = { name: "Tom", age: > 25 }); // ✅
> ```
> 如果不加括号，`{ name, age  = ...}` 会被当成代码块，报错。

---

# 13. 数组

## 13.1 创建与访问

```javascript
const arr = [1, 2, 3, 4, 5];
arr[0];          // 1
arr.length;      // 5
```

## 13.2 常用方法

```javascript
const arr = [1, 2, 3];

// 修改原数组（变异方法）
arr.push(4);          // 末尾添加元素 4 → [1,2,3,4]
arr.pop();            // 末尾删除元素 4 → [1,2,3]
arr.unshift(0);       // 开头添加元素 0 → [0,1,2,3]
arr.shift();          // 开头删除元素 0 → [1,2,3]
arr.splice(1, 1);     // 从索引1删除1个元素 → [1,3]

// 返回新数组（非变异方法）
[1,2,3].concat([4,5]);      // [1,2,3,4,5]
[1,2,3].slice(0, 2);        // [1,2]（切片）
[1,2,3].join("-");          // "1-2-3"
[3,1,2].sort();              // [1,2,3]（注意默认按字符串 Unicode 排序）
[1,2,3].reverse();           // [3,2,1]
```
> [!TIP] 提示：
> **变异方法** = 原地改数组；**非变异方法** = 返回新数组、原数组不动。日常开发里，除非明确需要改原数组，否则优先用非变异写法更安全。


## 13.3 遍历与变换（高频）

```javascript
const nums = [1, 2, 3, 4, 5];

// forEach：遍历
nums.forEach(n => console.log(n));

// map：映射为新数组
const doubled = nums.map(n => n * 2);     // [2,4,6,8,10]

// filter：过滤
const evens = nums.filter(n => n % 2 === 0);  // [2,4]

// find：找第一个匹配
const found = nums.find(n => n > 2);      // 3

// reduce：归约为单个值
const sum = nums.reduce((acc, n) => acc + n, 0);  // 15

// some / every
nums.some(n => n > 3);     // true（有一个满足即可）
nums.every(n => n > 0);    // true（全部满足才 true）
```
---

# 14. 错误处理

## 14.1 try...catch...finally

```javascript
try {
  // 可能出错的代码
  const data = JSON.parse("无效 JSON");
} catch (err) {
  // 出错后执行
  console.error("出错了：", err.message);
} finally {
  // 无论是否出错都执行
  console.log("清理工作");
}
```

## 14.2 throw 抛出错误

```javascript
function divide(a, b) {
  if (b === 0) {
    throw new Error("除数不能为零");
  }
  return a / b;
}

try {
  divide(10, 0);
} catch (err) {
  console.log(err.message);   // "除数不能为零"
}
```

## 14.3 常见内置错误类型

| 错误类型 | 触发场景 |
|---------|---------|
| `Error` | 通用错误基类 |
| `SyntaxError` | 语法错误（解析阶段） |
| `ReferenceError` | 引用了不存在的变量 |
| `TypeError` | 值的类型不对（如对非函数调用） |
| `RangeError` | 数值超出范围 |
| `URIError` | `encodeURI`/`decodeURI` 参数非法 |

```javascript
// ReferenceError
console.log(undefinedVar);

// TypeError
null.foo;

// RangeError
new Array(-1);
```

> [!NOTE] 备注：
> `catch` 块中的错误对象 `err` 有三个常用属性：
> - `err.name`：错误类型名
> - `err.message`：错误描述
> - `err.stack`：完整调用栈（调试用）

---

# 15. 异步编程：Promise 与 async/await

## 15.1 为什么需要异步？

JavaScript 是**单线程**的，通常通过一个主线程执行代码。如果某个任务需要长时间占用线程，例如进行大量计算、死循环或同步读取文件，那么线程在任务结束前无法继续执行后续代码，这称为**线程阻塞**。

```javascript
console.log("开始");

// 模拟一个耗时的同步任务
for (let i = 0; i < 1000000000; i++) {}

console.log("结束");
```

在上面的代码中，“开始”执行后，JavaScript 必须等待循环结束才能输出“结束”。如果循环耗费的时间较长，整个页面就可能无法响应用户操作。

网络请求、读取文件、定时器等操作通常不需要一直占用 JavaScript 主线程。它们可以交给浏览器或 Node.js 底层的其他机制处理，并在完成时通知 JavaScript继续执行。这种无需等待任务完成就能让程序继续运行的机制称为**异步**。

例如：

```javascript
console.log("开始请求");

fetch("https://api.example.com/user")
  .then(res => res.json())
  .then(user => console.log(user));

console.log("请求已发起");
```

`fetch` 发起网络请求后会立即返回，JavaScript 可以继续执行后面的代码。等请求真正完成时，再通过 `.then()` 处理返回的数据。

虽然 JavaScript 的主执行线程通常是单线程的，但它并不是等待异步操作时什么都不做。浏览器或 Node.js 会通过**事件循环**协调任务：

1. JavaScript 先执行当前调用栈中的代码。
2. 异步操作在等待期间，JavaScript 可以继续处理其他任务。
3. 操作完成后，其回调进入任务队列。
4. 事件循环会在合适的时机取出这些回调并执行。

因此，异步并不会让耗时计算自动变快，而是可以把等待过程与 JavaScript 的执行分开，避免主线程被长时间阻塞。

---

## 15.2 回调函数时代的问题

异步操作出现之初，JavaScript 通常通过回调函数获取结果：异步 API 接收一个函数，任务完成后再调用这个函数。

```javascript
getUser(function (user) {
  console.log(user);
});
```

对于少量、互不依赖的异步操作，回调方式简单直接。但是，当多个异步任务存在先后关系时，回调函数往往需要不断嵌套。

```javascript
// 回调地狱（Callback Hell）
getUser(function (user) {
  getOrders(user.id, function (orders) {
    getDetail(orders[0].id, function (detail) {
      getComments(detail.id, function (comments) {
        console.log(comments);
      });
    });
  });
});
```

这段代码的问题并不在于回调本身，而在于多个有依赖关系的异步操作被迫嵌套。随着任务数量增加，代码会越来越难阅读和维护。

回调方式还存在以下问题：

- **代码层级过深**：需要不断缩进，逻辑结构不清晰。
- **错误处理分散**：每个回调都可能产生错误，逐层处理非常繁琐。
- **流程难以组合**：串行、并行和条件执行都需要编写复杂逻辑。
- **容易遗漏返回或抛出错误**：异常可能无法正确传递给上层。

为了解决这些问题，JavaScript 引入了 **Promise**。

---

## 15.3 Promise

Promise 是一个对象，用来表示某个异步操作将来完成或失败后的结果。它可以把回调式的异步流程改成更容易组合的链式结构。

Promise 有三种状态：
1. `pending`（进行中）：操作尚未结束。
2. `fulfilled`（成功）：操作成功完成，并产生一个结果值。
3. `rejected`（失败）：操作失败，并产生一个失败原因。

状态转换关系如下：
```text
pending（进行中）
    │
    ├──→ fulfilled（成功）──→ .then() 的成功回调
    │
    └──→ rejected（失败） ──→ .catch() 的失败回调
```

Promise 的状态一旦从 `pending` 变为 `fulfilled` 或 `rejected`，就不能再次改变。第一次转换的结果就是最终结果。

### 15.3.1 创建 Promise

```javascript
const promise = new Promise((resolve, reject) => {
  setTimeout(() => {
    const success = true;

    if (success) {
      resolve("操作成功");
    } else {
      reject("操作失败");
    }
  }, 1000);
});
```

创建 Promise 时传入的函数会立即执行：

- `resolve(value)`：将 Promise 切换为成功状态。
- `reject(reason)`：将 Promise 切换为失败状态。

### 15.3.2 处理 Promise

```javascript
promise
  .then(result => console.log(result))   // 操作成功时执行
  .catch(err => console.error(err))      // 操作失败时执行
  .finally(() => console.log("结束"));   // 无论成功还是失败都会执行
```

各方法的作用如下：

- `.then()`：处理成功结果，也可以通过抛出一个新错误使后续步骤失败。
- `.catch()`：统一处理 Promise 链中的错误。
- `.finally()`：无论前面的操作成功还是失败都会执行，常用于隐藏加载提示或释放资源。

需要注意的是，`.then()` 和 `.catch()` 本身也会返回新的 Promise，因此它们可以继续连接后续操作。

---

## 15.4 Promise 链式调用

Promise 的主要价值不只是处理单个异步操作，而是可以通过返回值将多个异步任务连接起来。

```javascript
fetch("https://api.example.com/user")
  .then(res => res.json())
  .then(user => fetch(`https://api.example.com/orders/${user.id}`))
  .then(res => res.json())
  .then(orders => console.log(orders))
  .catch(err => console.error(err));
```

执行过程如下：

1. 请求用户数据。
2. 把用户响应转换为 JavaScript 对象。
3. 使用用户 ID 请求订单。
4. 把订单响应转换为 JavaScript 对象。
5. 输出订单。
6. 如果任意步骤失败，则由末尾的 `.catch()` 统一处理。

这组任务存在依赖关系，因此必须按照顺序执行。Promise 链通过回调的返回值自动等待前一个异步操作结束，避免了传统回调中的层层嵌套。

---

## 15.5 为什么还需要 async/await？

Promise 解决了回调嵌套问题，但长时间使用 `.then()` 仍然需要不断拆分和缩进。为了让异步代码更接近日常同步代码的阅读习惯，ES2017 引入了 `async/await`。

`async/await` 是基于 Promise 的语法糖：

- `async` 用来声明异步函数。
- `await` 用来等待一个 Promise 完成并取得结果。
- `async` 函数总是返回一个 Promise。
- `await` 只会暂停当前异步函数，不会阻塞整个 JavaScript 线程。

### 15.5.1 使用 async/await

```javascript
async function getUserOrders() {
  try {
    const res = await fetch("https://api.example.com/user");
    const user = await res.json();

    const orderRes = await fetch(`https://api.example.com/orders/${user.id}`);
    const orders = await orderRes.json();

    return orders;
  } catch (err) {
    console.error("请求失败：", err);
  }
}
```

这段代码可以按照顺序理解为：

1. 等待用户接口返回响应。
2. 等待用户响应转换为对象。
3. 使用用户 ID 请求订单。
4. 等待订单响应转换为数组。
5. 返回订单数据。

它完成的操作与前面的 Promise 链基本相同，但代码结构更加扁平。

### 15.5.2 Promise 与 async/await 的关系

下面的两种写法作用相近：

```javascript
function getUserOrders() {
  return fetch("https://api.example.com/user")
    .then(res => res.json())
    .then(user =>
      fetch(`https://api.example.com/orders/${user.id}`)
    )
    .then(res => res.json());
}
```

```javascript
async function getUserOrders() {
  const userRes = await fetch("https://api.example.com/user");
  const user = await userRes.json();

  const orderRes = await fetch(`https://api.example.com/orders/${user.id}`);
  return orderRes.json();
}
```

`async/await` 并没有创造新的异步机制，它只是提供了一种更直观地使用 Promise 的语法，更便于阅读和理解。

### 15.5.3 注意事项

- `async` 函数始终返回 Promise：

  ```javascript
  async function getNumber() {
    return 42;
  }

  getNumber().then(console.log);
  ```

  `getNumber()` 实际返回的是 `Promise.resolve(42)`。

- 在 `async` 函数中，`throw` 会使返回的 Promise 进入失败状态：

  ```javascript
  async function getNumber() {
    throw new Error("发生错误");
  }
  ```

- 顶层 `await` 只能在支持它的 ES Module 环境中使用，传统脚本不能直接使用顶层 `await`。

- 连续使用 `await` 表示串行执行。后一个请求只有在前一个请求完成后才会开始。

---

## 15.6 并行执行异步请求

如果多个请求互不依赖，就不应该逐个等待。使用多个 `await` 会让它们串行执行，从而增加总耗时。

```javascript
// 串行执行：总耗时约为三个请求耗时之和
const users = await fetch("/api/users").then(res => res.json());
const products = await fetch("/api/products").then(res => res.json());
const orders = await fetch("/api/orders").then(res => res.json());
```

如果这些请求彼此独立，应使用 `Promise.all` 并行启动：

```javascript
const [users, products, orders] = await Promise.all([
  fetch("/api/users").then(res => res.json()),
  fetch("/api/products").then(res => res.json()),
  fetch("/api/orders").then(res => res.json())
]);
```

这些请求会在数组创建时依次发起，并行等待结果。`Promise.all` 最终返回的结果顺序与传入数组的顺序一致，而不是按照请求完成的先后顺序排列。

判断执行方式时，可以遵循以下原则：

- **任务存在依赖关系**：后一个操作需要前一个结果，使用串行执行。
- **任务互不依赖**：多个任务可以同时等待，使用 `Promise.all` 等组合方法。

---

## 15.7 Promise 的组合方法

除了 `Promise.all`，JavaScript 还提供了 `Promise.allSettled`、`Promise.race` 和 `Promise.any`。

| 方法 | 行为 | 常见场景 |
|------|------|----------|
| `Promise.all` | 所有 Promise 都成功后才成功；任意一个失败则整体失败 | 并行获取必须全部成功的数据 |
| `Promise.allSettled` | 等待所有 Promise 完成，不区分成功或失败 | 需要查看每项操作最终结果的场景 |
| `Promise.race` | 采用第一个完成的 Promise，无论成功还是失败 | 超时控制或获取最先返回的结果 |
| `Promise.any` | 采用第一个成功的 Promise；全部失败后才失败 | 从多个请求中选择最先成功的响应 |

---

# 16. JSON

JSON（JavaScript Object Notation）是一种通用数据格式。

```javascript
// JS 对象 → JSON 字符串
const obj = { name: "Tom", age: 25 };
const json = JSON.stringify(obj);
// '{"name":"Tom","age":25}'

// JSON 字符串 → JS 对象
const parsed = JSON.parse('{"name":"Tom","age":25}');
// { name: "Tom", age: 25 }
```

> [!CAUTION] 注意：
> - JSON 的键必须用双引号，不能用单引号。
> `JSON.parse` 解析非法 JSON 会抛异常，应放在 `try...catch` 中。
> - `JSON.stringify` 会自动忽略 `undefined`、函数和 Symbol。

---

# 17. 模块化

早期 JS 并没有原生模块系统；Node.js 时代使用 CommonJS 的 require 做模块化；直到 ES6 才正式加入了标准模块系统（ESM），是现代 JS 官方标准。

模块化是指将一个大型程序拆分成多个功能相对独立的模块。每个模块只负责完成一类功能，并通过导出和导入与其他模块协作。

使用模块好有下面这些好处：

- 减少全局变量污染
- 隐藏模块内部的实现细节
- 便于复用和维护代码
- 可以明确管理模块之间的依赖关系

ESM 主要包含三种操作：**导出**、**导入**和**重新导出**。

## 17.1 导出模块内容

使用 `export` 可以将模块中的变量或函数导出，供其他模块使用。

```javascript
// utils.js

// 命名导出：一个模块中可以导出多个命名成员
export const PI = 3.14;
export function sum(a, b) {
  return a + b;
}

// 默认导出：一个模块中最多只能有一个默认导出
export default function greet(name) {
  return `你好，${name}`;
}
```

**命名导出**：

`PI` 和 `sum` 都是命名导出。导入时，需要使用与导出时相同的名称：

```javascript
import { PI, sum } from "./utils.js";
```

一个模块可以有任意多个命名导出。对于包含多个功能的工具模块，命名导出通常更加清晰，也有利于代码维护和静态分析。

**默认导出**：

`greet` 是默认导出。一个模块最多只能包含一个默认导出，导入时可以使用任意本地名称：

```javascript
import sayHello from "./utils.js";
```

这里的 `sayHello` 是导入方自己取的名称，因此默认导出不要求导入方使用原函数名。

> [!TIP] 提示：
> 命名导出与默认导出可以同时存在于同一个模块中，但一个模块只能有一个默认导出。

## 17.2 导入模块内容

在 `main.js` 中，可以同时导入默认导出和命名导出：

```javascript
// main.js

import greet, { PI, sum } from "./utils.js";

console.log(greet("Tom")); // 你好，Tom
console.log(PI);           // 3.14
console.log(sum(2, 3));    // 5
```

其中：

- `greet` 对应 `utils.js` 中的默认导出
- `PI` 和 `sum` 对应模块中的命名导出

如果希望本地名称更清晰，可以使用 `as` 重命名：

```javascript
import greet, {
  PI as circlePi,
  sum as add,
} from "./utils.js";

console.log(greet("Tom"));
console.log(circlePi);
console.log(add(2, 3));
```

另外还可以**导入整个模块**，使用命名空间导入，将模块中的所有导出都放在一个对象中：

```javascript
import * as utils from "./utils.js";

console.log(utils.PI);
console.log(utils.sum(2, 3));
console.log(utils.default("Tom"));
```

命名空间导入适用于需要集中访问多个导出，或需要在遍历等场景中动态处理导出成员的情况。一般情况下，直接使用命名导入会更直观。

## 17.3 重新导出

模块不仅可以导入其他模块，还可以将导入的内容继续提供给其他模块，这称为“**重新导出**”。

```javascript
// index.js

export { PI, sum } from "./utils.js";
export { default as greet } from "./utils.js";
```

随后，其他模块只需从 `index.js` 导入：

```javascript
import greet, { PI, sum } from "./index.js";
```

这种写法常用于为项目提供统一的入口文件，避免外部模块直接依赖项目内部的文件结构。

还可以使用 `export *` 导出目标模块中的所有命名成员：

```javascript
export * from "./utils.js";
```

需要注意，`export *` **不会重新导出默认导出**，默认导出仍需明确指定：

```javascript
export { default as greet } from "./utils.js";
```

## 17.4 在浏览器中使用模块

浏览器需要通过 `type="module"` 告诉 JavaScript，该脚本是一个 ES 模块：

```html
<!-- index.html -->
<script type="module" src="./main.js"></script>
```

浏览器会从 `main.js` 开始加载，并继续处理其中导入的 `utils.js`。

模块脚本默认具有延迟执行特性。此外，浏览器还会对模块脚本执行同源策略，因此通过 HTTP 或 HTTPS 加载的模块通常需要满足相应的跨域规则。直接使用 `file://` 打开本地页面时，模块导入可能会受到浏览器安全限制，必要时应启动一个本地 HTTP 服务器。

## 17.5 在 Node.js 中使用模块

Node.js 支持两种常见的 ESM 使用方式。

- 方式一：使用 `.mjs` 后缀：将文件命名为 `main.mjs`，Node.js 会将它作为 ES 模块处理，使用`.mjs` 后缀能够明确表示该文件使用 ESM。
  ```javascript
  // main.mjs
  import greet, { PI, sum } from "./utils.mjs";

  console.log(greet("Tom"));
  ```



- 方式二：在 `package.json` 中设置模块类型
  ```json
  {
    "type": "module"
  }
  ```
  设置后，该项目目录中的 `.js` 文件也会按照 ES 模块处理。

## 17.6 动态导入

除了静态导入，ESM 还支持通过 `import()` 动态导入模块。动态导入会返回一个 `Promise`，适合按需加载模块：

```javascript
async function loadUtils() {
  const utils = await import("./utils.js");

  console.log(utils.PI);
  console.log(utils.sum(2, 3));
  console.log(utils.default("Tom"));
}

loadUtils();
```

在浏览器中，动态导入常用于实现代码分割和延迟加载，从而减少页面首次加载时需要执行的代码量。

---

# 18. DOM 基础

DOM（Document Object Model）是浏览器将 HTML 文档抽象成的树形结构，JS 通过 DOM API 操作页面。

## 18.1 选取元素

```javascript
// 通过 ID
const header = document.getElementById("header");

// 通过 CSS 选择器（推荐）
const btn = document.querySelector(".btn-primary");
const items = document.querySelectorAll("li");
```

## 18.2 修改内容与样式

```javascript
el.textContent = "新文本";           // 修改纯文本
el.innerHTML = "<strong>加粗</strong>"; // 修改 HTML

el.style.color = "red";              // 修改样式
el.classList.add("active");          // 添加 CSS 类
el.classList.remove("active");       // 移除 CSS 类
el.classList.toggle("active");       // 切换
```

## 18.3 事件监听

```javascript
const btn = document.querySelector("#myBtn");

btn.addEventListener("click", function (event) {
  console.log("按钮被点击了");
  console.log(event.target);   // 触发事件的元素
});
```
**常见事件**：

| 事件 | 触发时机 |
|------|---------|
| `click` | 鼠标点击 |
| `input` | 输入框内容变化 |
| `submit` | 表单提交 |
| `keydown` | 键盘按下 |
| `mouseover` | 鼠标悬停 |
| `load` | 页面/资源加载完成 |

---

# 19. 参考链接

**官方文档**：

- **MDN Web Docs**（最权威的 JS 参考）：<https://developer.mozilla.org/zh-CN/docs/Web/JavaScript>
- **ECMAScript 规范**：<https://tc39.es/ecma262/>
- **Microsoft JavaScript 文档**：<https://learn.microsoft.com/zh-cn/javascript/>

**入门教程**：

- **现代 JavaScript 教程（javascript.info）**：<https://zh.javascript.info/>
- **JavaScript 教程（阮一峰）**：<https://wangdoc.com/javascript/>
- **ES6 入门教程（阮一峰）**：<https://es6.ruanyifeng.com/>

**练习平台**：

- **Codewars**：<https://www.codewars.com/>
- **LeetCode**：<https://leetcode.cn/>
- **JavaScript30**：<https://javascript30.com/>


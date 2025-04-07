---
title: JavaScript 基础语法
tags:
  - fleet-note
  - front-end/javascript
date: 2025-04-06
time: 14:52
aliases: 
is_archie: false
---

`==` 和 `===` 判断，`isNaN`，`NaN` 判断都为 `false`

大整数

表示比 $2^{53}$ 更大的就用 BigInt

```javascript
// 使用BigInt:
var bi1 = 9223372036854775807n;
var bi2 = BigInt(12345);
var bi3 = BigInt("0x7fffffffffffffff");
console.log(bi1 === bi2); // false
console.log(bi1 === bi3); // true
console.log(bi1 + bi2);
```


模板变量

```js
let name = '小明';
let age = 20;
let message = `你好, ${name}, 你今年${age}岁了!`;
alert(message);
```




# Reference
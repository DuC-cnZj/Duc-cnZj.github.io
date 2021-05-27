---
title: 读 JavaScript 教程 后感
date: '2021-05-12 12:54:14'
sidebar: true
categories:
 - JavaScript
tags:
 - JavaScript
publish: true
---




> 全文内容摘自 https://wangdoc.com/javascript

JavaScript 内部，所有数字都是以64位浮点数形式储存，即使整数也是如此。所以，`1`与`1.0`是相同的，是同一个数。

> 这就是说，JavaScript 语言的底层根本没有整数，所有数字都是小数（64位浮点数）。容易造成混淆的是，某些运算只有整数才能完成，此时 JavaScript 会自动把64位浮点数，转成32位整数，然后再进行运算，参见《运算符》一章的”位运算“部分。
>
> 由于浮点数不是精确的值，所以涉及小数的比较和运算要特别小心。

```js
0.1 + 0.2 === 0.3
// false

0.3 / 0.1
// 2.9999999999999996

(0.3 - 0.2) === (0.2 - 0.1)
// false
```

`+0`或`-0`当作分母，返回的值是不相等的。

```js
(1 / +0) === (1 / -0) // false

// 上面的代码之所以出现这样结果，是因为除以正零得到+Infinity，除以负零得到-Infinity，这两者是不相等的（关于Infinity详见下文）。
```



注意，删除一个不存在的属性，`delete`不报错，而且返回`true`。

```js
var obj = {};
delete obj.p // true
```

另外，需要注意的是，`delete`命令只能删除对象本身的属性，无法删除继承的属性（关于继承参见《面向对象编程》章节）。

```js
var obj = {};
delete obj.toString // true
obj.toString // function toString() { [native code] }
```

上面代码中，`toString`是对象`obj`继承的属性，虽然`delete`命令返回`true`，但该属性并没有被删除，依然存在。这个例子还说明，即使`delete`返回`true`，该属性依然可能读取到值。

## with 语句

`with`语句的格式如下：

```js
with (对象) {
  语句;
}
```

它的作用是操作同一个对象的多个属性时，提供一些书写的方便。

```js
// 例一
var obj = {
  p1: 1,
  p2: 2,
};
with (obj) {
  p1 = 4;
  p2 = 5;
}
// 等同于
obj.p1 = 4;
obj.p2 = 5;

// 例二
with (document.links[0]){
  console.log(href);
  console.log(title);
  console.log(style);
}
// 等同于
console.log(document.links[0].href);
console.log(document.links[0].title);
console.log(document.links[0].style);
```

**注意，如果`with`区块内部有变量的赋值操作，必须是当前对象已经存在的属性，否则会创造一个当前作用域的全局变量。**

```js
var obj = {};
with (obj) {
  p1 = 4;
  p2 = 5;
}

obj.p1 // undefined
p1 // 4
```

### 函数的重复声明

如果同一个函数被多次声明，后面的声明就会覆盖前面的声明。

```js
function f() {
  console.log(1);
}
f() // 2

function f() {
  console.log(2);
}
f() // 2
```

上面代码中，后一次的函数声明覆盖了前面一次。而且，由于函数名的提升（参见下文），前一次声明在任何时候都是无效的，这一点要特别注意。

函数参数是复合类型的值（数组、对象、其他函数），传递方式是传址传递（pass by reference）。函数参数如果是原始类型的值（数值、字符串、布尔值），传递方式是传值传递（passes by value）。

## 同名的参数

如果有同名的参数，则取最后出现的那个值。

```js
function f(a, a) {
  console.log(a);
}

f(1, 2) // 2
```

上面代码中，函数`f`有两个参数，且参数名都是`a`。取值的时候，以后面的`a`为准，即使后面的`a`没有值或被省略，也是以其为准。

```js
function f(a, a) {
  console.log(a);
}

f(1) // undefined
```

调用函数`f`的时候，没有提供第二个参数，`a`的取值就变成了`undefined`。这时，如果要获得第一个`a`的值，可以使用`arguments`对象。

```js
function f(a, a) {
  console.log(arguments[0]);
}

f(1) // 1
```



> 严格模式下，`arguments`对象是一个只读对象，修改它是无效的，但不会报错。

如果出于种种原因，需要得到函数内的局部变量。正常情况下，这是办不到的，只有通过变通方法才能实现。那就是在函数的内部，再定义一个函数。

```js
function f1() {
  var n = 999;
  function f2() {
　　console.log(n); // 999
  }
}
```



### 立即调用的函数表达式

```js
(function(){ /* code */ }());
// 或者
(function(){ /* code */ })();
```

> 注意，上面两种写法最后的分号都是必须的。如果省略分号，遇到连着两个 IIFE，可能就会报错。

推而广之，任何让解释器以表达式来处理函数定义的方法，都能产生同样的效果，比如下面三种写法。

```js
var i = function(){ return 10; }();
true && function(){ /* code */ }();
0, function(){ /* code */ }();
```

甚至像下面这样写，也是可以的。

```js
!function () { /* code */ }();
~function () { /* code */ }();
-function () { /* code */ }();
+function () { /* code */ }();
```



本质上，数组属于一种特殊的对象。`typeof`运算符会返回数组的类型是`object`。

```js
typeof [1, 2, 3] // "object"
```





## 运算符`in`

检查某个键名是否存在的运算符`in`，适用于对象，也适用于数组。

```js
var arr = [ 'a', 'b', 'c' ];
2 in arr  // true
'2' in arr // true
4 in arr // false
```



## 且运算符（&&）

***与php里面不同！***

> 且运算符（`&&`）往往用于多个表达式的求值。
>
> **它的运算规则是：如果第一个运算子的布尔值为`true`，则返回第二个运算子的值（注意是值，不是布尔值）；如果第一个运算子的布尔值为`false`，则直接返回第一个运算子的值，且不再对第二个运算子求值。**

```js
't' && '' // ""
't' && 'f' // "f"
't' && (1 + 2) // 3
'' && 'f' // ""
'' && '' // ""

// “短路”
var x = 1;
(1 - 1) && ( x += 1) // 0
x // 1
// 上面代码的最后一个例子，由于且运算符的第一个运算子的布尔值为false，则直接返回它的值0，而不再对第二个运算子求值，所以变量x的值没变。

if (i) {
  doSomething();
}
// 等价于
i && doSomething();

// 运算符可以多个连用
true && 'foo' && '' && 4 && 'foo' && true
// ''

1 && 2 && 3
// 3
```

## 或运算符（||）

> 或运算符（`||`）也用于多个表达式的求值。它的运算规则是：如果第一个运算子的布尔值为`true`，则返回第一个运算子的值，且不再对第二个运算子求值；如果第一个运算子的布尔值为`false`，则返回第二个运算子的值。

```js
't' || '' // "t"
't' || 'f' // "t"
'' || 'f' // "f"
'' || '' // ""
```

短路规则对这个运算符也适用。

```js
var x = 1;
true || (x = 2) // true
x // 1

// 或运算符可以多个连用，这时返回第一个布尔值为true的表达式的值。如果所有表达式都为false，则返回最后一个表达式的值。
false || 0 || '' || 4 || 'foo' || true
// 4

false || 0 || ''
// ''
```

## 语句 vs 表达式

> 在 javascript 中所有表达式都有返回值（如果没有返回值就是`undefined`）
>
> 语句没有返回值。



## 逗号运算符

逗号运算符用于对两个表达式求值，并返回后一个表达式的值。

```js
'a', 'b' // "b"

var x = 0;
var y = (x++, 10);
x // 1
y // 10
```

上面代码中，逗号运算符返回后一个表达式的值。

逗号运算符的一个用途是，在返回一个值之前，进行一些辅助操作。

```js
var value = (console.log('Hi!'), true);
// Hi!

value // true
```

上面代码中，先执行逗号之前的操作，然后返回逗号后面的值。





### 圆括号的作用 

> 圆括号（`()`）可以用来提高运算的优先级，因为它的优先级是最高的，即圆括号中的表达式会第一个运算。



### 左结合与右结合

[here](https://wangdoc.com/javascript/operators/priority.html)

对于优先级别相同的运算符，大多数情况，计算顺序总是从左到右，这叫做运算符的“左结合”（left-to-right associativity），即从左边开始计算。

```
x + y + z
```

上面代码先计算最左边的`x`与`y`的和，然后再计算与`z`的和。

但是少数运算符的计算顺序是从右到左，即从右边开始计算，这叫做运算符的“右结合”（right-to-left associativity）。其中，最主要的是赋值运算符（`=`）和三元条件运算符（`?:`）。

```js
w = x = y = z;
q = a ? b : c ? d : e ? f : g;
```

上面代码的运算结果，相当于下面的样子。

```js
w = (x = (y = z));
q = a ? b : (c ? d : (e ? f : g));
```

上面的两行代码，各有三个等号运算符和三个三元运算符，都是先计算最右边的那个运算符。

指数运算符（**）也是右结合的。

```js
// 相当于 2 ** (3 ** 2)
2 ** 3 ** 2
// 512

```



### Boolean()

`Boolean`函数可以将任意类型的值转为布尔值。

它的转换规则相对简单：除了以下五个值的转换结果为`false`，其他的值全部为`true`。

- `undefined`
- `null`
- `-0`或`+0`
- `NaN`
- `''`（空字符串）

```js
Boolean(undefined) // false
Boolean(null) // false
Boolean(0) // false
Boolean(NaN) // false
Boolean('') // false
```

注意，所有对象（包括空对象）的转换结果都是`true`，**甚至连`false`对应的布尔对象`new Boolean(false)`也是`true`**（详见《原始类型值的包装对象》一章）。

```js
Boolean({}) // true
Boolean([]) // true
Boolean(new Boolean(false)) // true
```

所有对象的布尔值都是`true`，这是因为 JavaScript 语言设计的时候，出于性能的考虑，如果对象需要计算才能得到布尔值，对于`obj1 && obj2`这样的场景，可能会需要较多的计算。为了保证性能，就统一规定，对象的布尔值为`true`。

## throw 语句

实际上，`throw`可以抛出任何类型的值。也就是说，它的参数可以是任何值。

```js
// 抛出一个字符串
throw 'Error！';
// Uncaught Error！

// 抛出一个数值
throw 42;
// Uncaught 42

// 抛出一个布尔值
throw true;
// Uncaught true

// 抛出一个对象
throw {
  toString: function () {
    return 'Error!';
  }
};
// Uncaught {toString: ƒ}
```

对于 JavaScript 引擎来说，遇到`throw`语句，程序就中止了。引擎会接收到`throw`抛出的信息，可能是一个错误实例，也可能是其他类型的值。



```js
// good
function doAction(action) {
  var actions = {
    'hack': function () {
      return 'hack';
    },
    'slash': function () {
      return 'slash';
    },
    'run': function () {
      return 'run';
    }
  };

  if (typeof actions[action] !== 'function') {
    throw new Error('Invalid action.');
  }

  return actions[action]();
}

// bad
function doAction(action) {
  switch (action) {
    case 'hack':
      return 'hack';
      break;
    case 'slash':
      return 'slash';
      break;
    case 'run':
      return 'run';
      break;
    default:
      throw new Error('Invalid action.');
  }
}
```



## console 家族全部命令

https://wangdoc.com/javascript/features/console.html

控制台api

$_ 返回上一个表达式的值。

```js
2 + 2
// 4
$_
// 4
```

> 注意，通过`var obj = new Object()`的写法生成新对象，与字面量的写法`var obj = {}`是等价的。或者说，后者只是前者的一种简便写法。


具体来说，如果一个属性的`enumerable`为`false`，下面三个操作不会取到该属性。

- `for..in`循环
- `Object.keys`方法
- `JSON.stringify`方法

https://wangdoc.com/javascript/stdlib/attributes.html#enumerable



`configurable`(可配置性）返回一个布尔值，决定了是否可以修改属性描述对象。也就是说，`configurable`为`false`时，`value`、`writable`、`enumerable`和`configurable`都不能被修改了。



所谓“浅拷贝”，指的是新数组拷贝的是对象的引用。



`slice`方法的一个重要应用，是将类似数组的对象转为真正的数组，和 `Array.from`效果一样

```js
Array.prototype.slice.call({ 0: 'a', 1: 'b', length: 2 })
// ['a', 'b']

Array.prototype.slice.call(document.querySelectorAll("div"));
Array.prototype.slice.call(arguments);
```

```js
arr.splice(start, count, addElement1, addElement2, ...);
```



## js自然排序

如果想让`sort`方法按照自定义方式排序，可以传入一个函数作为参数。

```js
[10111, 1101, 111].sort(function (a, b) {
  return a - b;
})
// [111, 1101, 10111]

[
  { name: "张三", age: 30 },
  { name: "李四", age: 24 },
  { name: "王五", age: 28  }
].sort(function (o1, o2) {
  return o1.age - o2.age;
})
// [
//   { name: "李四", age: 24 },
//   { name: "王五", age: 28  },
//   { name: "张三", age: 30 }
// ]
```

上面代码中，`sort`的参数函数本身接受两个参数，表示进行比较的两个数组成员。如果该函数的返回值大于`0`，表示第一个成员排在第二个成员后面；其他情况下，都是第一个元素排在第二个元素前面。



`map`方法还可以接受第二个参数，用来绑定回调函数内部的`this`变量（详见《this 变量》一章）。

```js
var arr = ['a', 'b', 'c'];

[1, 2].map(function (e) {
  return this[e];
}, arr)
// ['b', 'c']
```

如果数组有空位，`map`方法的回调函数在这个位置不会执行，会跳过数组的空位。

```js
var f = function (n) { return 'a' };

[1, undefined, 2].map(f) // ["a", "a", "a"]
[1, null, 2].map(f) // ["a", "a", "a"]
[1, , 2].map(f) // ["a", , "a"]
```

上面代码中，`map`方法不会跳过`undefined`和`null`，但是会跳过空位。

https://wangdoc.com/javascript/stdlib/array.html

注意，`forEach`方法无法中断执行，总是会将所有成员遍历完。如果希望符合某种条件时，就中断遍历，要使用`for`循环。

`forEach`方法也会跳过数组的空位。

```js
var log = function (n) {
  console.log(n + 1);
};

[1, undefined, 2].forEach(log)
// 2
// NaN
// 3

[1, null, 2].forEach(log)
// 2
// 1
// 3

[1, , 2].forEach(log)
// 2
// 3
```

上面代码中，`forEach`方法不会跳过`undefined`和`null`，但会跳过空位。

### 还有这操作？！

```js
var arr = [0, 1, 'a', false];

arr.filter(Boolean)
// [1, "a"]
```

上面代码中，`filter`方法返回数组`arr`里面所有布尔值为`true`的成员。

注意，这两个方法不能用来搜索`NaN`的位置，即它们无法确定数组成员是否包含`NaN`。

```js
[NaN].indexOf(NaN) // -1
[NaN].lastIndexOf(NaN) // -1
```

这是因为这两个方法内部，使用严格相等运算符（`===`）进行比较，而`NaN`是唯一一个不等于自身的值。

注意，`false`对应的包装对象实例，布尔运算结果也是`true`。

```js
if (new Boolean(false)) {
  console.log('true');
} // true

if (new Boolean(false).valueOf()) {
  console.log('true');
} // 无输出
```

上面代码的第一个例子之所以得到`true`，是因为`false`对应的包装对象实例是一个对象，进行逻辑运算时，被自动转化成布尔值`true`（因为所有对象对应的布尔值都是`true`）。而实例的`valueOf`方法，则返回实例对应的原始值，本例为`false`。

```js
Boolean(undefined) // false
Boolean(null) // false
Boolean(0) // false
Boolean('') // false
Boolean(NaN) // false

Boolean(1) // true
Boolean('false') // true
Boolean([]) // true
Boolean({}) // true
Boolean(function () {}) // true
Boolean(/foo/) // true
```

最后，对于一些特殊值，`Boolean`对象前面加不加`new`，会得到完全相反的结果，必须小心。

```js
if (Boolean(false)) {
  console.log('true');
} // 无输出

if (new Boolean(false)) {
  console.log('true');
} // true

if (Boolean(null)) {
  console.log('true');
} // 无输出

if (new Boolean(null)) {
  console.log('true');
} // true
```

https://wangdoc.com/javascript/stdlib/boolean.html



```js
10..toString(2)
// "1010"

// 其他方法还包括
10 .toString(2) // "1010"
10.0.toString(2) // "1010"
```

https://wangdoc.com/javascript/stdlib/number.html



`toFixed`方法先将一个数转为指定位数的小数，然后返回这个小数对应的字符串。

```js
(10).toFixed(2) // "10.00"
10.005.toFixed(2) // "10.01"
```

正则表达式（regular expression）是一种表达文本模式（即字符串结构）的方法，有点像字符串的模板，常常用来按照“给定模式”匹配文本。比如，正则表达式给出一个 Email 地址的模式，然后用它来确定一个字符串是否为 Email 地址。JavaScript 的正则表达式体系是参照 Perl 5 建立的。

新建正则表达式有两种方法。一种是使用字面量，以斜杠表示开始和结束。

```js
var regex = /xyz/;
```

另一种是使用`RegExp`构造函数。

```js
var regex = new RegExp('xyz');
```

上面两种写法是等价的，都新建了一个内容为`xyz`的正则表达式对象。**它们的主要区别是，第一种方法在引擎编译代码时，就会新建正则表达式，第二种方法在运行时新建正则表达式，所以前者的效率较高。**而且，前者比较便利和直观，所以实际应用中，基本上都采用字面量定义正则表达式。

`replace`方法的一个应用，就是消除字符串首尾两端的空格。

```
var str = '  #id div.class  ';

str.replace(/^\s+|\s+$/g, '')
// "#id div.class"
```

`replace`方法的第二个参数可以使用美元符号`$`，用来指代所替换的内容。

- `$&`：匹配的子字符串。
- `$``：匹配结果前面的文本。
- `$'`：匹配结果后面的文本。
- `$n`：匹配成功的第`n`组内容，`n`是从1开始的自然数。
- `$$`：指代美元符号`$`。

```js
'hello world'.replace(/(\w+)\s(\w+)/, '$2 $1')
// "world hello"

'abc'.replace('b', '[$`-$&-$\']')
// "a[a-b-c]c"
```

https://wangdoc.com/javascript/stdlib/regexp.html

`replace`方法的第**二个参数还可以是一个函数**，将每一个匹配内容替换为函数返回值。

```js
'3 and 5'.replace(/[0-9]+/g, function (match) {
  return 2 * match;
})
// "6 and 10"

var a = 'The quick brown fox jumped over the lazy dog.';
var pattern = /quick|brown|lazy/ig;

a.replace(pattern, function replacer(match) {
  return match.toUpperCase();
});
// The QUICK BROWN fox jumped over the LAZY dog.
```

上面代码中，第一个正则表达式之所以不匹配，因为加号是元字符，不代表自身。第二个正则表达式使用反斜杠对加号转义，就能匹配成功。

正则表达式中，需要反斜杠转义的，一共有12个字符：`^`、`.`、`[`、`$`、`(`、`)`、`|`、`*`、`+`、`?`、`{`和`\`。需要特别注意的是，如果使用`RegExp`方法生成正则对象，转义需要使用两个斜杠，因为字符串内部会先转义一次。

```js
(new RegExp('1\+1')).test('1+1')
// false

(new RegExp('1\\+1')).test('1+1')
// true
```

上面代码中，`RegExp`作为构造函数，参数是一个字符串。但是，在字符串内部，反斜杠也是转义字符，所以它会先被反斜杠转义一次，然后再被正则表达式转义一次，因此需要两个反斜杠转义。

如果方括号内没有其他字符，即只有`[^]`，就表示匹配一切字符，其中包括换行符。相比之下，点号作为元字符（`.`）是不包括换行符的。

```js
var s = 'Please yes\nmake my day!';

s.match(/yes.*day/) // null
s.match(/yes[^]*day/) // [ 'yes\nmake my day']
```

预定义模式指的是某些常见模式的简写方式。

- `\d` 匹配0-9之间的任一数字，相当于`[0-9]`。
- `\D` 匹配所有0-9以外的字符，相当于`[^0-9]`。
- `\w` 匹配任意的字母、数字和下划线，相当于`[A-Za-z0-9_]`。
- `\W` 除所有字母、数字和下划线以外的字符，相当于`[^A-Za-z0-9_]`。
- `\s` 匹配空格（包括换行符、制表符、空格符等），相等于`[ \t\r\n\v\f]`。
- `\S` 匹配非空格的字符，相当于`[^ \t\r\n\v\f]`。
- `\b` 匹配词的边界。
- `\B` 匹配非词边界，即在词的内部。

通常，正则表达式遇到换行符（`\n`）就会停止匹配。

```
var html = "<b>Hello</b>\n<i>world!</i>";

/.*/.exec(html)[0]
// "<b>Hello</b>"
```

上面代码中，字符串`html`包含一个换行符，结果点字符（`.`）不匹配换行符，导致匹配结果可能不符合原意。这时使用`\s`字符类，就能包括换行符。

```
var html = "<b>Hello</b>\n<i>world!</i>";

/[\S\s]*/.exec(html)[0]
// "<b>Hello</b>\n<i>world!</i>"
```

上面代码中，`[\S\s]`指代一切字符。

### 贪婪模式

上一小节的三个量词符，默认情况下都是最大可能匹配，即匹配直到下一个字符不满足匹配规则为止。这被称为贪婪模式。

```js
var s = 'aaa';
s.match(/a+/) // ["aaa"]
```

上面代码中，模式是`/a+/`，表示匹配1个`a`或多个`a`，那么到底会匹配几个`a`呢？因为默认是贪婪模式，会一直匹配到字符`a`不出现为止，所以匹配结果是3个`a`。

如果想将贪婪模式改为非贪婪模式，可以在量词符后面加一个问号。

```js
var s = 'aaa';
s.match(/a+?/) // ["a"]
```

上面代码中，模式结尾添加了一个问号`/a+?/`，这时就改为非贪婪模式，一旦条件满足，就不再往下匹配。

除了非贪婪模式的加号，还有非贪婪模式的星号（`*`）和非贪婪模式的问号（`?`）。

- `+?`：表示某个模式出现1次或多次，匹配时采用非贪婪模式。
- `*?`：表示某个模式出现0次或多次，匹配时采用非贪婪模式。
- `??`：表格某个模式出现0次或1次，匹配时采用非贪婪模式。

```js
'abb'.match(/ab*b/) // ["abb"]
'abb'.match(/ab*?b/) // ["ab"]

'abb'.match(/ab?b/) // ["abb"]
'abb'.match(/ab??b/) // ["ab"]
```

**（1）g 修饰符**

默认情况下，第一次匹配成功后，正则对象就停止向下匹配了。`g`修饰符表示全局匹配（global），加上它以后，正则对象将匹配全部符合条件的结果，主要用于搜索和替换。

```js
var regex = /b/;
var str = 'abba';

regex.test(str); // true
regex.test(str); // true
regex.test(str); // true
```

上面代码中，正则模式不含`g`修饰符，每次都是从字符串头部开始匹配。所以，连续做了三次匹配，都返回`true`。

```js
var regex = /b/g;
var str = 'abba';

regex.test(str); // true
regex.test(str); // true
regex.test(str); // false
```

上面代码中，正则模式含有`g`修饰符，每次都是从上一次匹配成功处，开始向后匹配。因为字符串`abba`只有两个`b`，所以前两次匹配结果为`true`，第三次匹配结果为`false`。

**（2）i 修饰符**

默认情况下，正则对象区分字母的大小写，加上`i`修饰符以后表示忽略大小写（ignorecase）。

```
/abc/.test('ABC') // false
/abc/i.test('ABC') // true
```

上面代码表示，加了`i`修饰符以后，不考虑大小写，所以模式`abc`匹配字符串`ABC`。

**（3）m 修饰符**

`m`修饰符表示多行模式（multiline），会修改`^`和`$`的行为。默认情况下（即不加`m`修饰符时），`^`和`$`匹配字符串的开始处和结尾处，加上`m`修饰符以后，`^`和`$`还会匹配行首和行尾，即`^`和`$`会识别换行符（`\n`）。

```js
/world$/.test('hello world\n') // false
/world$/m.test('hello world\n') // true
```

上面的代码中，字符串结尾处有一个换行符。如果不加`m`修饰符，匹配不成功，因为字符串的结尾不是`world`；加上以后，`$`可以匹配行尾。

```js
/^b/m.test('a\nb') // true
```

上面代码要求匹配行首的`b`，如果不加`m`修饰符，就相当于`b`只能处在字符串的开始处。加上`b`修饰符以后，换行符`\n`也会被认为是一行的开始。

正则表达式内部，还可以用`\n`引用括号匹配的内容，`n`是从1开始的自然数，表示对应顺序的括号。

```
/(.)b(.)\1b\2/.test("abcabc")
// true
```

上面的代码中，`\1`表示第一个括号匹配的内容（即`a`），`\2`表示第二个括号匹配的内容（即`c`）。

下面是另外一个例子。

```js
/y(..)(.)\2\1/.test('yabccab') // true
```

括号还可以嵌套。

```js
/y((..)\2)\1/.test('yabababab') // true
```

上面代码中，`\1`指向外层括号，`\2`指向内层括号。

**（2）非捕获组** 不输出括号匹配

`(?:x)`称为非捕获组（Non-capturing group），表示不返回该组匹配的内容，即匹配的结果中不计入这个括号。

非捕获组的作用请考虑这样一个场景，假定需要匹配`foo`或者`foofoo`，正则表达式就应该写成`/(foo){1, 2}/`，但是这样会占用一个组匹配。这时，就可以使用非捕获组，将正则表达式改为`/(?:foo){1, 2}/`，它的作用与前一个正则是一样的，但是不会单独输出括号内部的内容。

请看下面的例子。

```js
var m = 'abc'.match(/(?:.)b(.)/);
m // ["abc", "c"]
```

上面代码中的模式，一共使用了两个括号。其中第一个括号是非捕获组，所以最后返回的结果中没有第一个括号，只有第二个括号匹配的内容。

```js
// 正常匹配
var url = /(http|ftp):\/\/([^/\r\n]+)(\/[^\r\n]*)?/;

url.exec('http://google.com/');
// ["http://google.com/", "http", "google.com", "/"]

// 非捕获组匹配
var url = /(?:http|ftp):\/\/([^/\r\n]+)(\/[^\r\n]*)?/;

url.exec('http://google.com/');
// ["http://google.com/", "google.com", "/"]
```

**（3）先行断言**

`x(?=y)`称为先行断言（Positive look-ahead），`x`只有在`y`前面才匹配，`y`不会被计入返回结果。比如，要匹配后面跟着百分号的数字，可以写成`/\d+(?=%)/`。

“先行断言”中，括号里的部分是不会返回的。

```js
var m = 'abc'.match(/b(?=c)/);
m // ["b"]
```

上面的代码使用了先行断言，`b`在`c`前面所以被匹配，但是括号对应的`c`不会被返回。

如果对象的属性是`undefined`、函数或 XML 对象，该属性会被`JSON.stringify`过滤。

```js
var obj = {
  a: undefined,
  b: function () {}
};

JSON.stringify(obj) // "{}"
```

如果数组的成员是`undefined`、函数或 XML 对象，则这些值被转成`null`。

```js
var arr = [undefined, function () {}];
JSON.stringify(arr) // "[null,null]"
```

上面代码中，数组`arr`的成员是`undefined`和函数，它们都被转成了`null`。

正则对象会被转成空对象。

```js
JSON.stringify(/foo/) // "{}"
```

```js
var obj = {
  'prop1': 'value1',
  'prop2': 'value2',
  'prop3': 'value3'
};

var selectedProperties = ['prop1', 'prop2'];

JSON.stringify(obj, selectedProperties)
// "{"prop1":"value1","prop2":"value2"}"
```

第二个参数还可以是一个函数，用来更改`JSON.stringify`的返回值。

```js
function f(key, value) {
  if (typeof value === "number") {
    value = 2 * value;
  }
  return value;
}

JSON.stringify({ a: 1, b: 2 }, f)
// '{"a": 2,"b": 4}'
```

另一方面，如果对普通函数（内部没有`this`关键字的函数）使用`new`命令，则会返回一个空对象。

`this`都有一个共同点：它总是返回一个对象。

简单说，`this`就是属性或方法“当前”所在的对象。

```
this.property
```

上面代码中，`this`就代表`property`属性当前所在的对象。

### 立即调用的函数表达式（IIFE）立即执行函数

在 Javascript 中，圆括号`()`是一种运算符，跟在函数名之后，表示调用该函数。比如，`print()`就表示调用`print`函数。

有时，我们需要在定义函数之后，立即调用该函数。这时，你不能在函数的定义之后加上圆括号，这会产生语法错误。

```js
function(){ /* code */ }();
// SyntaxError: Unexpected token (
```

产生这个错误的原因是，`function`这个关键字即可以当作语句，也可以当作表达式。

```js
// 语句
function f() {}

// 表达式
var f = function f() {}
```

为了避免解析上的歧义，JavaScript 引擎规定，如果`function`关键字出现在行首，一律解释成语句。因此，JavaScript 引擎看到行首是`function`关键字之后，认为这一段都是函数的定义，不应该以圆括号结尾，所以就报错了。

解决方法就是不要让`function`出现在行首，让引擎将其理解成一个表达式。最简单的处理，就是将其放在一个圆括号里面。

```js
(function(){ /* code */ }());
// 或者
(function(){ /* code */ })();
```

上面两种写法都是以圆括号开头，引擎就会认为后面跟的是一个表示式，而不是函数定义语句，所以就避免了错误。这就叫做“立即调用的函数表达式”（Immediately-Invoked Function Expression），简称 IIFE。

注意，上面两种写法最后的分号都是必须的。如果省略分号，遇到连着两个 IIFE，可能就会报错。

```js
// 报错
(function(){ /* code */ }())
(function(){ /* code */ }())
```

上面代码的两行之间没有分号，JavaScript 会将它们连在一起解释，将第二行解释为第一行的参数。

推而广之，任何让解释器以表达式来处理函数定义的方法，都能产生同样的效果，比如下面三种写法。

```js
var i = function(){ return 10; }();
true && function(){ /* code */ }();
0, function(){ /* code */ }();
```

甚至像下面这样写，也是可以的。

```js
!function () { /* code */ }();
~function () { /* code */ }();
-function () { /* code */ }();
+function () { /* code */ }();
```

通常情况下，只对匿名函数使用这种“立即执行的函数表达式”。它的目的有两个：一是不必为函数命名，避免了污染全局变量；二是 IIFE 内部形成了一个单独的作用域，可以封装一些外部无法读取的私有变量。

```js
// 写法一
var tmp = newData;
processData(tmp);
storeData(tmp);

// 写法二
(function () {
  var tmp = newData;
  processData(tmp);
  storeData(tmp);
}());
```

上面代码中，写法二比写法一更好，因为完全避免了污染全局变量。

函数实例的`call`方法，可以指定函数内部`this`的指向（即函数执行时所在的作用域），然后在所指定的作用域中，调用该函数。

```js
var obj = {};

var f = function () {
  return this;
};

f() === window // true
f.call(obj) === obj // true
```

```js
var obj = {};
obj.hasOwnProperty('toString') // false

// 覆盖掉继承的 hasOwnProperty 方法
obj.hasOwnProperty = function () {
  return true;
};
obj.hasOwnProperty('toString') // true

Object.prototype.hasOwnProperty.call(obj, 'toString') // false
```

上面代码中，`hasOwnProperty`是`obj`对象继承的方法，如果这个方法一旦被覆盖，就不会得到正确结果。`call`方法可以解决这个问题，它将`hasOwnProperty`方法的原始定义放到`obj`对象上执行，这样无论`obj`上有没有同名方法，都不会影响结果。

`bind`方法有一些使用注意点。

**（1）每一次返回一个新函数**

`bind`方法每运行一次，就返回一个新函数，这会产生一些问题。比如，监听事件的时候，不能写成下面这样。

```js
element.addEventListener('click', o.m.bind(o));
```

上面代码中，`click`事件绑定`bind`方法生成的一个匿名函数。这样会导致无法取消绑定，所以，下面的代码是无效的。

```js
element.removeEventListener('click', o.m.bind(o));
```

正确的方法是写成下面这样：

```js
var listener = o.m.bind(o);
element.addEventListener('click', listener);
//  ...
element.removeEventListener('click', listener);
```

除了前两个参数，`setTimeout`还允许更多的参数。它们将依次传入推迟执行的函数（回调函数）。

```json
setTimeout(function (a,b) {
  console.log(a + b);
}, 1000, 1, 1);
```

`setInterval`指定的是“开始执行”之间的间隔，并不考虑每次任务执行本身所消耗的时间。因此实际上，两次执行之间的间隔会小于指定的时间。比如，`setInterval`指定每 100ms 执行一次，每次执行需要 5ms，那么第一次执行结束后95毫秒，第二次执行就会开始。如果某次执行耗时特别长，比如需要105毫秒，那么它结束后，下一次执行就会立即开始。

为了确保两次执行之间有固定的间隔，可以不用`setInterval`，而是每次执行结束后，使用`setTimeout`指定下一次执行的具体时间。

```js
var i = 1;
var timer = setTimeout(function f() {
  // ...
  timer = setTimeout(f, 2000);
}, 2000);
```

上面代码可以确保，下一次执行总是在本次执行结束之后的2000毫秒开始。

## debounce 函数

有时，我们不希望回调函数被频繁调用。比如，用户填入网页输入框的内容，希望通过 Ajax 方法传回服务器，jQuery 的写法如下。

```
$('textarea').on('keydown', ajaxAction);
```

这样写有一个很大的缺点，就是如果用户连续击键，就会连续触发`keydown`事件，造成大量的 Ajax 通信。这是不必要的，而且很可能产生性能问题。正确的做法应该是，设置一个门槛值，表示两次 Ajax 通信的最小间隔时间。如果在间隔时间内，发生新的`keydown`事件，则不触发 Ajax 通信，并且重新开始计时。如果过了指定时间，没有发生新的`keydown`事件，再将数据发送出去。

这种做法叫做 debounce（防抖动）。假定两次 Ajax 通信的间隔不得小于2500毫秒，上面的代码可以改写成下面这样。

```js
$('textarea').on('keydown', debounce(ajaxAction, 2500));

function debounce(fn, delay){
  var timer = null; // 声明计时器
  return function() {
    var context = this;
    var args = arguments;
    clearTimeout(timer);
    timer = setTimeout(function () {
      fn.apply(context, args);
    }, delay);
  };
}
```

上面代码中，只要在2500毫秒之内，用户再次击键，就会取消上一次的定时器，然后再新建一个定时器。这样就保证了回调函数之间的调用间隔，至少是2500毫秒。

`setTimeout`和`setInterval`的运行机制，是将指定的代码移出本轮事件循环，等到下一轮事件循环，再检查是否到了指定时间。如果到了，就执行对应的代码；如果不到，就继续等待。

这意味着，`setTimeout`和`setInterval`指定的回调函数，**必须等到本轮事件循环的所有同步任务都执行完，才会开始执行。**由于前面的任务到底需要多少时间执行完，是不确定的，所以没有办法保证，`setTimeout`和`setInterval`指定的任务，一定会按照预定时间执行。

`setTimeout`的作用是将代码推迟到指定时间执行，如果指定时间为`0`，即`setTimeout(f, 0)`，那么会立刻执行吗？

答案是不会。因为上一节说过，必须要等到当前脚本的同步任务，全部处理完以后，才会执行`setTimeout`指定的回调函数`f`。也就是说，`setTimeout(f, 0)`会在下一轮事件循环一开始就执行

```js
setTimeout(function () {
  console.log(1);
}, 0);
console.log(2);
// 2
// 1
```

`setTimeout(f, 0)`有几个非常重要的用途。它的一大应用是，可以调整事件的发生顺序。比如，网页开发中，某个事件先发生在子元素，然后冒泡到父元素，即子元素的事件回调函数，会早于父元素的事件回调函数触发。如果，想让父元素的事件回调函数先发生，就要用到`setTimeout(f, 0)`。

```
// HTML 代码如下
// <input type="button" id="myButton" value="click">

var input = document.getElementById('myButton');

input.onclick = function A() {
  setTimeout(function B() {
    input.value +=' input';
  }, 0)
};

document.body.onclick = function C() {
  input.value += ' body'
};
```

上面代码在点击按钮后，先触发回调函数`A`，然后触发函数`C`。函数`A`中，`setTimeout`将函数`B`推迟到下一轮事件循环执行，这样就起到了，先触发父元素的回调函数`C`的目的了。

另一个应用是，用户自定义的回调函数，通常在浏览器的默认动作之前触发。比如，用户在输入框输入文本，`keypress`事件会在浏览器接收文本之前触发。因此，下面的回调函数是达不到目的的。

```
// HTML 代码如下
// <input type="text" id="input-box">

document.getElementById('input-box').onkeypress = function (event) {
  this.value = this.value.toUpperCase();
}
```

上面代码想在用户每次输入文本后，立即将字符转为大写。但是实际上，它只能将本次输入前的字符转为大写，因为浏览器此时还没接收到新的文本，所以`this.value`取不到最新输入的那个字符。只有用`setTimeout`改写，上面的代码才能发挥作用。

```
document.getElementById('input-box').onkeypress = function() {
  var self = this;
  setTimeout(function() {
    self.value = self.value.toUpperCase();
  }, 0);
}
```

上面代码将代码放入`setTimeout`之中，就能使得它在浏览器接收到文本之后触发。

由于`setTimeout(f, 0)`实际上意味着，将任务放到浏览器最早可得的空闲时段执行，所以那些计算量大、耗时长的任务，常常会被放到几个小部分，分别放到`setTimeout(f, 0)`里面执行。

```
var div = document.getElementsByTagName('div')[0];

// 写法一
for (var i = 0xA00000; i < 0xFFFFFF; i++) {
  div.style.backgroundColor = '#' + i.toString(16);
}

// 写法二
var timer;
var i=0x100000;

function func() {
  timer = setTimeout(func, 0);
  div.style.backgroundColor = '#' + i.toString(16);
  if (i++ == 0xFFFFFF) clearTimeout(timer);
}

timer = setTimeout(func, 0);
```

上面代码有两种写法，都是改变一个网页元素的背景色。写法一会造成浏览器“堵塞”，因为 JavaScript 执行速度远高于 DOM，会造成大量 DOM 操作“堆积”，而写法二就不会，这就是`setTimeout(f, 0)`的好处。

另一个使用这种技巧的例子是代码高亮的处理。如果代码块很大，一次性处理，可能会对性能造成很大的压力，那么将其分成一个个小块，一次处理一块，比如写成`setTimeout(highlightNext, 50)`的样子，性能压力就会减轻。

https://wangdoc.com/javascript/async/timer.html



Promise 对象是 JavaScript 的**异步**操作解决方案，为异步操作提供统一接口。它起到代理作用（proxy），充当异步操作与回调函数之间的中介，使得异步操作具备同步操作的接口。Promise 可以让异步操作写起来，就像在写同步操作的流程，而不必一层层地嵌套回调函数。

Promise 的回调函数属于异步任务，会在同步任务之后执行。

```js
new Promise(function (resolve, reject) {
  resolve(1);
}).then(console.log);

console.log(2);
// 2
// 1
```

`addEventListener`方法可以为针对当前对象的同一个事件，添加多个不同的监听函数。这些函数按照添加顺序触发，即先添加先触发。如果为同一个事件多次添加同一个监听函数，该函数只会执行一次，多余的添加将自动被去除（不必使用`removeEventListener`方法手动去除）

注意，`removeEventListener`方法移除的监听函数，必须是`addEventListener`方法添加的那个监听函数，而且必须在同一个元素节点，否则无效。

```js
div.addEventListener('click', function (e) {}, false);
div.removeEventListener('click', function (e) {}, false);
```

上面代码中，`removeEventListener`方法无效，因为监听函数不是同一个匿名函数。

监听函数内部的`this`指向触发事件的那个元素节点。

```js
<button id="btn" onclick="console.log(this.id)">点击</button>
```

一个事件发生后，会在子元素和父元素之间传播（propagation）。这种传播分成三个阶段。

- **第一阶段**：从`window`对象传导到目标节点（上层传到底层），称为“捕获阶段”（capture phase）。
- **第二阶段**：在目标节点上触发，称为“目标阶段”（target phase）。
- **第三阶段**：从目标节点传导回`window`对象（从底层传回上层），称为“冒泡阶段”（bubbling phase）。



如果希望事件到某个节点为止，不再传播，可以使用事件对象的`stopPropagation`方法。

```js
// 事件传播到 p 元素后，就不再向下传播了
p.addEventListener('click', function (event) {
  event.stopPropagation();
}, true);

// 事件冒泡到 p 元素后，就不再向上冒泡了
p.addEventListener('click', function (event) {
  event.stopPropagation();
}, false);
```

上面代码中，`stopPropagation`方法分别在捕获阶段和冒泡阶段，阻止了事件的传播。

但是，`stopPropagation`方法只会阻止事件的传播，不会阻止该事件触发`<p>`节点的其他`click`事件的监听函数。也就是说，不是彻底取消`click`事件。

```
p.addEventListener('click', function (event) {
  event.stopPropagation();
  console.log(1);
});

p.addEventListener('click', function(event) {
  // 会触发
  console.log(2);
});
```

上面代码中，`p`元素绑定了两个`click`事件的监听函数。`stopPropagation`方法只能阻止这个事件的传播，不能取消这个事件，因此，第二个监听函数会触发。输出结果会先是1，然后是2。

如果想要彻底取消该事件，不再触发后面所有`click`的监听函数，可以使用`stopImmediatePropagation`方法。

```js
p.addEventListener('click', function (event) {
  event.stopImmediatePropagation();
  console.log(1);
});

p.addEventListener('click', function(event) {
  // 不会被触发
  console.log(2);
});
```

上面代码中，`stopImmediatePropagation`方法可以彻底取消这个事件，使得后面绑定的所有`click`监听函数都不再触发。所以，只会输出1，不会输出2。

https://wangdoc.com/javascript/events/model.html

## 鼠标事件的种类 [#](https://wangdoc.com/javascript/events/mouse.html#%E9%BC%A0%E6%A0%87%E4%BA%8B%E4%BB%B6%E7%9A%84%E7%A7%8D%E7%B1%BB) 

鼠标事件指与鼠标相关的事件，继承了`MouseEvent`接口。具体的事件主要有以下一些。

- `click`：按下鼠标（通常是按下主按钮）时触发。
- `dblclick`：在同一个元素上双击鼠标时触发。
- `mousedown`：按下鼠标键时触发。
- `mouseup`：释放按下的鼠标键时触发。
- `mousemove`：当鼠标在一个节点内部移动时触发。当鼠标持续移动时，该事件会连续触发。为了避免性能问题，建议对该事件的监听函数做一些限定，比如限定一段时间内只能运行一次。
- `mouseenter`：鼠标进入一个节点时触发，进入子节点不会触发这个事件（详见后文）。
- `mouseover`：鼠标进入一个节点时触发，进入子节点会再一次触发这个事件（详见后文）。
- `mouseout`：鼠标离开一个节点时触发，离开父节点也会触发这个事件（详见后文）。
- `mouseleave`：鼠标离开一个节点时触发，离开父节点不会触发这个事件（详见后文）。
- `contextmenu`：按下鼠标右键时（上下文菜单出现前）触发，或者按下“上下文菜单键”时触发。
- `wheel`：滚动鼠标的滚轮时触发，该事件继承的是`WheelEvent`接口。

`click`事件指的是，用户在同一个位置先完成`mousedown`动作，再完成`mouseup`动作。因此，触发顺序是，`mousedown`首先触发，`mouseup`接着触发，`click`最后触发。

`dblclick`事件则会在`mousedown`、`mouseup`、`click`之后触发。

`mouseover`事件和`mouseenter`事件，都是鼠标进入一个节点时触发。两者的区别是，`mouseenter`事件只触发一次，而只要鼠标在节点内部移动，`mouseover`事件会在子节点上触发多次。



键盘事件由用户击打键盘触发，主要有`keydown`、`keypress`、`keyup`三个事件，它们都继承了`KeyboardEvent`接口。

- `keydown`：按下键盘时触发。
- `keypress`：按下有值的键时触发，即按下 Ctrl、Alt、Shift、Meta 这样无值的键，这个事件不会触发。对于有值的键，按下时先触发`keydown`事件，再触发这个事件。
- `keyup`：松开键盘时触发该事件。





如果用户一直按键不松开，就会连续触发键盘事件，触发的顺序如下。

1. keydown
2. keypress
3. keydown
4. keypress
5. ...（重复以上过程）
6. keyup



### KeyboardEvent.location

`KeyboardEvent.location`属性返回一个整数，表示按下的键处在键盘的哪一个区域。它可能取以下值。

- 0：处在键盘的主区域，或者无法判断处于哪一个区域。
- 1：处在键盘的左侧，只适用那些有两个位置的键（比如 Ctrl 和 Shift 键）。
- 2：处在键盘的右侧，只适用那些有两个位置的键（比如 Ctrl 和 Shift 键）。
- 3：处在数字小键盘。



### KeyboardEvent.repeat

`KeyboardEvent.repeat`返回一个布尔值，代表该键是否被按着不放，以便判断是否重复这个键，即浏览器会持续触发`keydown`和`keypress`事件，直到用户松开手为止。

`javascript:`协议的常见用途是书签脚本 Bookmarklet。由于浏览器的书签保存的是一个网址，所以`javascript:`网址也可以保存在里面，用户选择这个书签的时候，就会在当前页面执行这个脚本。为了防止书签替换掉当前文档，可以在脚本前加上`void`，或者在脚本最后加上`void 0`。

```js
<a href="javascript: void new Date().toLocaleTimeString();">点击</a>
<a href="javascript: new Date().toLocaleTimeString();void 0;">点击</a>
```

上面这两种写法，点击链接后，执行代码都不会网页跳转。

### defer 属性 [#](https://wangdoc.com/javascript/bom/engine.html#defer-%E5%B1%9E%E6%80%A7) 

为了解决脚本文件下载阻塞网页渲染的问题，一个方法是对`<script>`元素加入`defer`属性。它的作用是延迟脚本的执行，等到 DOM 加载生成后，再执行脚本。

```js 
<script src="a.js" defer></script>
<script src="b.js" defer></script>
```

上面代码中，只有等到 DOM 加载完成后，才会执行`a.js`和`b.js`。

### async 属性

解决“阻塞效应”的另一个方法是对`<script>`元素加入`async`属性。

```js
<script src="a.js" async></script>
<script src="b.js" async></script>
```

### 脚本的动态加载 

元素还可以动态生成，生成后再插入页面，从而实现脚本的动态加载。


```
['a.js', 'b.js'].forEach(function(src) {
  var script = document.createElement('script');
  script.src = src;
  document.head.appendChild(script);
});
```

这种方法的好处是，动态生成的`script`标签不会阻塞页面渲染，也就不会造成浏览器假死。但是问题在于，这种方法无法保证脚本的执行顺序，哪个脚本文件先下载完成，就先执行哪个。

如果想避免这个问题，可以设置async属性为`false`。

```js
['a.js', 'b.js'].forEach(function(src) {
  var script = document.createElement('script');
  script.src = src;
  script.async = false;
  document.head.appendChild(script);
});
```

### 加载使用的协议

如果不指定协议，浏览器默认采用 HTTP 协议下载。

```js
<script src="example.js"></script>
```

上面的`example.js`默认就是采用 HTTP 协议下载，如果要采用 HTTPS 协议下载，必需写明。

```js
<script src="https://example.js"></script>
```

但是有时我们会希望，根据页面本身的协议来决定加载协议，这时可以采用下面的写法。

```js
<script src="//example.js"></script>
```

需要注意的是，如果脚本网址与网页网址不在同一个域（比如使用了 CDN），浏览器根本不会提供详细的出错信息，只会提示出错，错误类型是“Script error.”，行号为0，其他信息都没有。这是浏览器防止向外部脚本泄漏信息。一个解决方法是在脚本所在的服务器，设置`Access-Control-Allow-Origin`的 HTTP 头信息。

```
Access-Control-Allow-Origin: *
```

然后，在网页的`<script>`标签中设置`crossorigin`属性。

```
<script crossorigin="anonymous" src="//example.com/file.js"></script>
```

上面代码的`crossorigin="anonymous"`表示，读取文件不需要身份信息，即不需要 cookie 和 HTTP 认证信息。如果设为`crossorigin="use-credentials"`，就表示浏览器会上传 cookie 和 HTTP 认证信息，同时还需要服务器端打开 HTTP 头信息`Access-Control-Allow-Credentials`。

## Cookie

有些开发者使用 Cookie 作为客户端储存。这样做虽然可行，但是并不推荐，因为 Cookie 的设计目标并不是这个，它的容量很小（4KB），缺乏数据操作接口，而且会影响性能。客户端储存应该使用 Web storage API 和 IndexedDB。

### HTTP 回应：Cookie 的生成 [#](https://wangdoc.com/javascript/bom/cookie.html#http-%E5%9B%9E%E5%BA%94%EF%BC%9Acookie-%E7%9A%84%E7%94%9F%E6%88%90) 

服务器如果希望在浏览器保存 Cookie，就要在 HTTP 回应的头信息里面，放置一个`Set-Cookie`字段。

```
Set-Cookie:foo=bar
```

AJAX 这个词第一次正式提出，它是 Asynchronous JavaScript and XML 的缩写

1995年，同源政策由 Netscape 公司引入浏览器。目前，所有浏览器都实行这个政策。

最初，它的含义是指，A 网页设置的 Cookie，B 网页不能打开，除非这两个网页“同源”。所谓“同源”指的是”三个相同“。

> - 协议相同
> - 域名相同
> - 端口相同


举例来说，`http://www.example.com/dir/page.html`这个网址，协议是`http://`，域名是`www.example.com`，端口是`80`（默认端口可以省略），它的同源情况如下。

- `http://www.example.com/dir2/other.html`：同源
- `http://example.com/dir/other.html`：不同源（域名不同）
- `http://v2.www.example.com/dir/other.html`：不同源（域名不同）
- `http://www.example.com:81/dir/other.html`：不同源（端口不同）
- `https://www.example.com/dir/page.html`：不同源（协议不同）

随着互联网的发展，同源政策越来越严格。目前，如果非同源，共有三种行为受到限制。

> （1） 无法读取非同源网页的 Cookie、LocalStorage 和 IndexedDB。
>
> （2） 无法接触非同源网页的 DOM。
>
> （3） 无法向非同源地址发送 AJAX 请求（可以发送，但浏览器会拒绝接受响应）。

同源政策规定，AJAX 请求只能发给同源的网址，否则就报错。

除了架设服务器代理（浏览器请求同源服务器，再由后者请求外部服务），有三种方法规避这个限制。

> - JSONP
> - WebSocket
> - CORS

### JSONP [#](https://wangdoc.com/javascript/bom/same-origin.html#jsonp) 

JSONP 是服务器与客户端跨源通信的常用方法。最大特点就是简单适用，老式浏览器全部支持，服务端改造非常小。

它的基本思想是，网页通过添加一个`<script>`元素，向服务器请求 JSON 数据，这种做法不受同源政策限制；服务器收到请求后，将数据放在一个指定名字的回调函数里传回来。

首先，网页动态插入`<script>`元素，由它向跨源网址发出请求。

```js
function addScriptTag(src) {
  var script = document.createElement('script');
  script.setAttribute("type","text/javascript");
  script.src = src;
  document.body.appendChild(script);
}

window.onload = function () {
  addScriptTag('http://example.com/ip?callback=foo');
}

function foo(data) {
  console.log('Your public IP address is: ' + data.ip);
};
```

上面代码通过动态添加`<script>`元素，向服务器`example.com`发出请求**。注意，该请求的查询字符串有一个`callback`参数，用来指定回调函数的名字，这对于 JSONP 是必需的。**

**服务器收到这个请求以后，会将数据放在回调函数的参数位置返回**。

```js
foo({
  "ip": "8.8.8.8"
});
```

由于`<script>`元素请求的脚本，直接作为代码运行。这时，只要浏览器定义了`foo`函数，该函数就会立即调用。作为参数的 JSON 数据被视为 JavaScript 对象，而不是字符串，因此避免了使用`JSON.parse`的步骤。

### WebSocket

WebSocket 是一种通信协议，使用`ws://`（非加密）和`wss://`（加密）作为协议前缀。该协议不实行同源政策，只要服务器支持，就可以通过它进行跨源通信。

下面是一个例子，浏览器发出的 WebSocket 请求的头信息（摘自[维基百科](https://en.wikipedia.org/wiki/WebSocket)）。

```http
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
Origin: http://example.com
```

上面代码中，有一个字段是`Origin`，表示该请求的请求源（origin），即发自哪个域名。

正是因为有了`Origin`这个字段，所以 WebSocket 才没有实行同源政策。因为服务器可以根据这个字段，判断是否许可本次通信。如果该域名在白名单内，服务器就会做出如下回应。

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=
Sec-WebSocket-Protocol: chat
```

### CORS

CORS 是跨源资源分享（Cross-Origin Resource Sharing）的缩写。它是 W3C 标准，属于跨源 AJAX 请求的根本解决方法。相比 JSONP 只能发`GET`请求，CORS 允许任何类型的请求。

CORS 请求分成两类：简单请求（simple request）和非简单请求（not-so-simple request）。

只要同时满足以下两大条件，就属于简单请求。

（1）请求方法是以下三种方法之一。

> - HEAD
> - GET
> - POST

（2）HTTP 的头信息不超出以下几种字段。

> - Accept
> - Accept-Language
> - Content-Language
> - Last-Event-ID
> - Content-Type：只限于三个值`application/x-www-form-urlencoded`、`multipart/form-data`、`text/plain`

https://wangdoc.com/javascript/bom/cors.html

如果`Origin`指定的域名在许可范围内，服务器返回的响应，会多出几个头信息字段。

```
Access-Control-Allow-Origin: http://api.bob.com
Access-Control-Allow-Credentials: true
Access-Control-Expose-Headers: FooBar
Content-Type: text/html; charset=utf-8
```

上面的头信息之中，有三个与 CORS 请求相关的字段，都以`Access-Control-`开头。

**（1）Access-Control-Allow-Origin**

该字段是必须的。它的值要么是请求时`Origin`字段的值，要么是一个`*`，表示接受任意域名的请求。

**（2）Access-Control-Allow-Credentials**

该字段可选。它的值是一个布尔值，表示是否允许发送 Cookie。默认情况下，Cookie 不包括在 CORS 请求之中。设为`true`，即表示服务器明确许可，浏览器可以把 Cookie 包含在请求中，一起发给服务器。这个值也只能设为`true`，如果服务器不要浏览器发送 Cookie，不发送该字段即可。

**（3）Access-Control-Expose-Headers**

该字段可选。CORS 请求时，`XMLHttpRequest`对象的`getResponseHeader()`方法只能拿到6个服务器返回的基本字段：`Cache-Control`、`Content-Language`、`Content-Type`、`Expires`、`Last-Modified`、`Pragma`。如果想拿到其他字段，就必须在`Access-Control-Expose-Headers`里面指定。上面的例子指定，`getResponseHeader('FooBar')`可以返回`FooBar`字段的值。

需要注意的是，如果服务器要求浏览器发送 Cookie，`Access-Control-Allow-Origin`就不能设为星号，必须指定明确的、与请求网页一致的域名。同时，Cookie 依然遵循同源政策，只有用服务器域名设置的 Cookie 才会上传，其他域名的 Cookie 并不会上传，且（跨域）原网页代码中的`document.cookie`也无法读取服务器域名下的 Cookie。



## 非简单请求

### 预检请求

非简单请求是那种对服务器提出特殊要求的请求，比如请求方法是`PUT`或`DELETE`，或者`Content-Type`字段的类型是`application/json`。

非简单请求的 CORS 请求，会在正式通信之前，增加一次 HTTP 查询请求，称为“预检”请求（preflight）。浏览器先询问服务器，当前网页所在的域名是否在服务器的许可名单之中，以及可以使用哪些 HTTP 动词和头信息字段。只有得到肯定答复，浏览器才会发出正式的`XMLHttpRequest`请求，否则就报错。这是为了防止这些新增的请求，对传统的没有 CORS 支持的服务器形成压力，给服务器一个提前拒绝的机会，这样可以防止服务器大量收到`DELETE`和`PUT`请求，这些传统的表单不可能跨域发出的请求。

下面是一段浏览器的 JavaScript 脚本。

```js
var url = 'http://api.alice.com/cors';
var xhr = new XMLHttpRequest();
xhr.open('PUT', url, true);
xhr.setRequestHeader('X-Custom-Header', 'value');
xhr.send();
```

上面代码中，HTTP 请求的方法是`PUT`，并且发送一个自定义头信息`X-Custom-Header`。

浏览器发现，这是一个非简单请求，就自动发出一个“预检”请求，要求服务器确认可以这样请求。下面是这个“预检”请求的 HTTP 头信息。

```http
OPTIONS /cors HTTP/1.1
Origin: http://api.bob.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: X-Custom-Header
Host: api.alice.com
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0...
```

“预检”请求用的请求方法是`OPTIONS`，表示这个请求是用来询问的。头信息里面，关键字段是`Origin`，表示请求来自哪个源。

除了`Origin`字段，“预检”请求的头信息包括两个特殊字段。

**（1）Access-Control-Request-Method**

该字段是必须的，用来列出浏览器的 CORS 请求会用到哪些 HTTP 方法，上例是`PUT`。

**（2）Access-Control-Request-Headers**

该字段是一个逗号分隔的字符串，指定浏览器 CORS 请求会额外发送的头信息字段，上例是`X-Custom-Header`。

## storage 事件 [#](https://wangdoc.com/javascript/bom/storage.html#storage-%E4%BA%8B%E4%BB%B6) 

Storage 接口储存的数据发生变化时，会触发 storage 事件，可以指定这个事件的监听函数。

```
window.addEventListener('storage', onStorageChange);
```

监听函数接受一个`event`实例对象作为参数。这个实例对象继承了 StorageEvent 接口，有几个特有的属性，都是只读属性。

- `StorageEvent.key`：字符串，表示发生变动的键名。如果 storage 事件是由`clear()`方法引起，该属性返回`null`。
- `StorageEvent.newValue`：字符串，表示新的键值。如果 storage 事件是由`clear()`方法或删除该键值对引发的，该属性返回`null`。
- `StorageEvent.oldValue`：字符串，表示旧的键值。如果该键值对是新增的，该属性返回`null`。
- `StorageEvent.storageArea`：对象，返回键值对所在的整个对象。也说是说，可以从这个属性上面拿到当前域名储存的所有键值对。
- `StorageEvent.url`：字符串，表示原始触发 storage 事件的那个网页的网址。

下面是`StorageEvent.key`属性的例子。

```js
function onStorageChange(e) {
  console.log(e.key);
}

window.addEventListener('storage', onStorageChange);
```

注意，该事件有一个很特别的地方，就是它不在导致数据变化的当前页面触发，而是在同一个域名的其他窗口触发。也就是说，如果浏览器只打开一个窗口，可能观察不到这个事件。比如同时打开多个窗口，当其中的一个窗口导致储存的数据发生改变时，只有在其他窗口才能观察到监听函数的执行。可以通过这种机制，实现多个窗口之间的通信。

`window.history`属性指向 History 对象，它表示当前窗口的浏览历史。

History 对象保存了当前窗口访问过的所有页面网址。下面代码表示当前窗口一共访问过3个网址。

```js
window.history.length // 3
```

由于安全原因，浏览器不允许脚本读取这些地址，但是允许在地址之间导航。

```js
// 后退到前一个网址
history.back()

// 等同于
history.go(-1)
```

### History.back()、History.forward()、History.go() [#](https://wangdoc.com/javascript/bom/history.html#historyback%E3%80%81historyforward%E3%80%81historygo) 

这三个方法用于在历史之中移动。

- `History.back()`：移动到上一个网址，等同于点击浏览器的后退键。对于第一个访问的网址，该方法无效果。
- `History.forward()`：移动到下一个网址，等同于点击浏览器的前进键。对于最后一个访问的网址，该方法无效果。
- `History.go()`：接受一个整数作为参数，以当前网址为基准，移动到参数指定的网址，比如`go(1)`相当于`forward()`，`go(-1)`相当于`back()`。如果参数超过实际存在的网址范围，该方法无效果；如果不指定参数，默认参数为`0`，相当于刷新当前页面。

```js
history.back();
history.forward();
history.go(-2);
```

`history.go(0)`相当于刷新当前页面。

```js
history.go(0); // 刷新当前页面
```

注意，移动到以前访问过的页面时，页面通常是从浏览器缓存之中加载，而不是重新要求服务器发送新的网页。

### History.pushState()，

`History.pushState()`方法用于在历史中添加一条记录。

```js
window.history.pushState(state, title, url)
```

该方法接受三个参数，依次为：

- `state`：一个与添加的记录相关联的状态对象，主要用于`popstate`事件。该事件触发时，该对象会传入回调函数。也就是说，浏览器会将这个对象序列化以后保留在本地，重新载入这个页面的时候，可以拿到这个对象。如果不需要这个对象，此处可以填`null`。
- `title`：新页面的标题。但是，现在所有浏览器都忽视这个参数，所以这里可以填空字符串。
- `url`：新的网址，必须与当前页面处在同一个域。浏览器的地址栏将显示这个网址。

假定当前网址是`example.com/1.html`，使用`pushState()`方法在浏览记录（History 对象）中添加一个新记录。

```js
var stateObj = { foo: 'bar' };
history.pushState(stateObj, 'page 2', '2.html');
```

添加新记录后，浏览器地址栏立刻显示`example.com/2.html`，但并不会跳转到`2.html`，甚至也不会检查`2.html`是否存在，它只是成为浏览历史中的最新记录。这时，在地址栏输入一个新的地址(比如访问`google.com`)，然后点击了倒退按钮，页面的 URL 将显示`2.html`；你再点击一次倒退按钮，URL 将显示`1.html`。

总之，`pushState()`方法不会触发页面刷新，只是导致 History 对象发生变化，地址栏会有反应。

使用该方法之后，就可以用`History.state`属性读出状态对象。

```js
var stateObj = { foo: 'bar' };
history.pushState(stateObj, 'page 2', '2.html');
history.state // {foo: "bar"}
```

如果`pushState`的 URL 参数设置了一个新的锚点值（即`hash`），并不会触发`hashchange`事件。反过来，如果 URL 的锚点值变了，则会在 History 对象创建一条浏览记录。

如果`pushState()`方法设置了一个跨域网址，则会报错。

```js
// 报错
// 当前网址为 http://example.com
history.pushState(null, '', 'https://twitter.com/hello');
```

上面代码中，`pushState`想要插入一个跨域的网址，导致报错。这样设计的目的是，防止恶意代码让用户以为他们是在另一个网站上，因为这个方法不会导致页面跳转。

## Location 对象 [#](https://wangdoc.com/javascript/bom/location.html#location-%E5%AF%B9%E8%B1%A1) 

`Location`对象是浏览器提供的原生对象，提供 URL 相关的信息和操作方法。通过`window.location`和`document.location`属性，可以拿到这个对象。

### 属性

`Location`对象提供以下属性。

- `Location.href`：整个 URL。
- `Location.protocol`：当前 URL 的协议，包括冒号（`:`）。
- `Location.host`：主机，包括冒号（`:`）和端口（默认的80端口和443端口会省略）。
- `Location.hostname`：主机名，不包括端口。
- `Location.port`：端口号。
- `Location.pathname`：URL 的路径部分，从根路径`/`开始。
- `Location.search`：查询字符串部分，从问号`?`开始。
- `Location.hash`：片段字符串部分，从`#`开始。
- `Location.username`：域名前面的用户名。
- `Location.password`：域名前面的密码。
- `Location.origin`：URL 的协议、主机名和端口。





这些属性里面，只有`origin`属性是只读的，其他属性都可写。

注意，如果对`Location.href`写入新的 URL 地址，浏览器会立刻跳转到这个新地址。

```
// 跳转到新网址
document.location.href = 'http://www.example.com';
```

这个特性常常用于让网页自动滚动到新的锚点。

```js
document.location.href = '#top';
// 等同于
document.location.hash = '#top';
```

直接改写`location`，相当于写入`href`属性。

```js
document.location = 'http://www.example.com';
// 等同于
document.location.href = 'http://www.example.com';
```

另外，`Location.href`属性是浏览器唯一允许跨域写入的属性，即非同源的窗口可以改写另一个窗口的`Location.href`属性，导致后者的网址跳转。`Location`的其他属性都不允许跨域写入。

**（3）Location.reload()**

`reload`方法使得浏览器重新加载当前网址，相当于按下浏览器的刷新按钮。

它接受一个布尔值作为参数。如果参数为`true`，浏览器将向服务器重新请求这个网页，并且重新加载后，网页将滚动到头部（即`scrollTop === 0`）。如果参数是`false`或为空，浏览器将从本地缓存重新加载该网页，并且重新加载后，网页的视口位置是重新加载前的位置。

```js
// 向服务器重新请求当前网址
window.location.reload(true);
```

**（4）Location.toString()**

`toString`方法返回整个 URL 字符串，相当于读取`Location.href`属性。

https://wangdoc.com/javascript/bom/location.html

```js
var url1 = new URL('index.html', 'http://example.com');
url1.href
// "http://example.com/index.html"

var url2 = new URL('page2.html', 'http://example.com/page1.html');
url2.href
// "http://example.com/page2.html"

var url3 = new URL('..', 'http://example.com/a/b.html')
url3.href
// "http://example.com/"
```

上面代码中，返回的 URL 实例的路径都是在第二个参数的基础上，切换到第一个参数得到的。最后一个例子里面，第一个参数是`..`，表示上层路径。

URL 实例的属性与`Location`对象的属性基本一致，返回当前 URL 的信息。

- URL.href：返回整个 URL
- URL.protocol：返回协议，以冒号`:`结尾
- URL.hostname：返回域名
- URL.host：返回域名与端口，包含`:`号，默认的80和443端口会省略
- URL.port：返回端口
- URL.origin：返回协议、域名和端口
- URL.pathname：返回路径，以斜杠`/`开头
- URL.search：返回查询字符串，以问号`?`开头
- URL.searchParams：返回一个`URLSearchParams`实例，该属性是`Location`对象没有的
- URL.hash：返回片段识别符，以井号`#`开头
- URL.password：返回域名前面的密码
- URL.username：返回域名前面的用户名

浏览器允许使用`URL.createObjectURL()`方法，针对 Blob 对象生成一个临时 URL，以便于某些 API 使用。这个 URL 以`blob://`开头，表明对应一个 Blob 对象，协议头后面是一个识别符，用来唯一对应内存里面的 Blob 对象。这一点与`data://URL`（URL 包含实际数据）和`file://URL`（本地文件系统里面的文件）都不一样。

## File 对象 [#](https://wangdoc.com/javascript/bom/file.html#file-%E5%AF%B9%E8%B1%A1) 

File 对象代表一个文件，用来读写文件信息。它继承了 Blob 对象，或者说是一种特殊的 Blob 对象，所有可以使用 Blob 对象的场合都可以使用它。

最常见的使用场合是表单的文件上传控件（`<input type="file">`），用户选中文件以后，浏览器就会生成一个数组，里面是每一个用户选中的文件，它们都是 File 实例对象。

```js
// HTML 代码如下
// <input id="fileItem" type="file">
var file = document.getElementById('fileItem').files[0];
file instanceof File // true
```

上面代码中，`file`是用户选中的第一个文件，它是 File 的实例。

### 构造函数

浏览器原生提供一个`File()`构造函数，用来生成 File 实例对象。

```js
new File(array, name [, options])
```

- array：一个数组，成员可以是二进制对象或字符串，表示文件的内容。
- name：字符串，表示文件名或文件路径。
- options：配置对象，设置实例的属性。该参数可选。

第三个参数配置对象，可以设置两个属性。

- type：字符串，表示实例对象的 MIME 类型，默认值为空字符串。
- lastModified：时间戳，表示上次修改的时间，默认为`Date.now()`。

下面是一个例子。

```js
var file = new File(
  ['foo'],
  'foo.txt',
  {
    type: 'text/plain',
  }
);
```

```js
<form>
  <input type="submit" value="提交">
</form>
```

上面表单就包含一个`submit`控件，点击这个控件，浏览器就会把表单数据向服务器提交。

注意，表单里面的`<button>`元素如果没有用`type`属性指定类型，那么默认就是`submit`控件。

除了点击`submit`控件提交表单，还可以用表单元素的`submit()`方法，通过脚本提交表单。

```js
formElement.submit();
```

表单元素的`reset()`方法可以重置所有控件的值（重置为默认值）。

```js
formElement.reset()
```

## 表单的内置验证

> https://wangdoc.com/javascript/bom/form.html

### 自动校验

表单提交的时候，浏览器允许开发者指定一些条件，它会自动验证各个表单控件的值是否符合条件。

```html
<!-- 必填 -->
<input required>

<!-- 必须符合正则表达式 -->
<input pattern="banana|cherry">

<!-- 字符串长度必须为6个字符 -->
<input minlength="6" maxlength="6">

<!-- 数值必须在1到10之间 -->
<input type="number" min="1" max="10">

<!-- 必须填入 Email 地址 -->
<input type="email">

<!-- 必须填入 URL -->
<input type="URL">
```

如果一个控件通过验证，它就会匹配`:valid`的 CSS 伪类，浏览器会继续进行表单提交的流程。如果没有通过验证，该控件就会匹配`:invalid`的 CSS 伪类，浏览器会终止表单提交，并显示一个错误信息。

### willValidate 属性

控件元素的`willValidate`属性是一个布尔值，表示该控件是否会在提交时进行校验。

```js
// HTML 代码如下
// <form novalidate>
//   <input id="name" name="name" required />
// </form>

var input = document.querySelector('#name');
input.willValidate // true
```

### 表单的 novalidate 属性

表单元素的 HTML 属性`novalidate`，可以关闭浏览器的自动校验。

```html
<form novalidate>
</form>
```

这个属性也可以在脚本里设置。

```
form.noValidate = true;
```

如果表单元素没有设置`novalidate`属性，那么提交按钮（`<button>`或`<input>`元素）的`formnovalidate`属性也有同样的作用。

```html
<form>
  <input type="submit" value="submit" formnovalidate>
</form>
```

## enctype 属性

表单能够用四种编码，向服务器发送数据。编码格式由表单的`enctype`属性决定。

假定表单有两个字段，分别是`foo`和`baz`，其中`foo`字段的值等于`bar`，`baz`字段的值是一个分为两行的字符串。

```
The first line.
The second line.
```

下面四种格式，都可以将这个表单发送到服务器。

**（1）GET 方法**

如果表单使用`GET`方法发送数据，`enctype`属性无效。

```
<form
  action="register.php"
  method="get"
  onsubmit="AJAXSubmit(this); return false;"
>
</form>
```

数据将以 URL 的查询字符串发出。

```
?foo=bar&baz=The%20first%20line.%0AThe%20second%20line.
```

**（2）application/x-www-form-urlencoded**

如果表单用`POST`方法发送数据，并省略`enctype`属性，那么数据以`application/x-www-form-urlencoded`格式发送（因为这是默认值）。

```
<form
  action="register.php"
  method="post"
  onsubmit="AJAXSubmit(this); return false;"
>
</form>
```

发送的 HTTP 请求如下。

```
Content-Type: application/x-www-form-urlencoded

foo=bar&baz=The+first+line.%0D%0AThe+second+line.%0D%0A
```

上面代码中，数据体里面的`%0D%0A`代表换行符（`\r\n`）。

**（3）text/plain**

如果表单使用`POST`方法发送数据，`enctype`属性为`text/plain`，那么数据将以纯文本格式发送。

```
<form
  action="register.php"
  method="post"
  enctype="text/plain"
  onsubmit="AJAXSubmit(this); return false;"
>
</form>
```

发送的 HTTP 请求如下。

```
Content-Type: text/plain

foo=bar
baz=The first line.
The second line.
```

**（4）multipart/form-data**

如果表单使用`POST`方法，`enctype`属性为`multipart/form-data`，那么数据将以混合的格式发送。

```
<form
  action="register.php"
  method="post"
  enctype="multipart/form-data"
  onsubmit="AJAXSubmit(this); return false;"
>
</form>
```

发送的 HTTP 请求如下。

```
Content-Type: multipart/form-data; boundary=---------------------------314911788813839

-----------------------------314911788813839
Content-Disposition: form-data; name="foo"

bar
-----------------------------314911788813839
Content-Disposition: form-data; name="baz"

The first line.
The second line.

-----------------------------314911788813839--
```

这种格式也是文件上传的格式。

## 文件上传

用户上传文件，也是通过表单。具体来说，就是通过文件输入框选择本地文件，提交表单的时候，浏览器就会把这个文件发送到服务器。

```html
<input type="file" id="file" name="myFile">
```

此外，还需要将表单`<form>`元素的`method`属性设为`POST`，`enctype`属性设为`multipart/form-data`。其中，`enctype`属性决定了 HTTP 头信息的`Content-Type`字段的值，默认情况下这个字段的值是`application/x-www-form-urlencoded`，但是文件上传的时候要改成`multipart/form-data`。

```html
<form method="post" enctype="multipart/form-data">
  <div>
    <label for="file">选择一个文件</label>
    <input type="file" id="file" name="myFile" multiple>
  </div>
  <div>
    <input type="submit" id="submit" name="submit_button" value="上传" />
  </div>
</form>
```

上面的 HTML 代码中，file 控件的`multiple`属性，指定可以一次选择多个文件；如果没有这个属性，则一次只能选择一个文件。

```js
var fileSelect = document.getElementById('file');
var files = fileSelect.files;
```

然后，新建一个 FormData 实例对象，模拟发送到服务器的表单数据，把选中的文件添加到这个对象上面。

```js
var formData = new FormData();

for (var i = 0; i < files.length; i++) {
  var file = files[i];

  // 只上传图片文件
  if (!file.type.match('image.*')) {
    continue;
  }

  formData.append('photos[]', file, file.name);
}
```

最后，使用 Ajax 向服务器上传文件。

```js
var xhr = new XMLHttpRequest();

xhr.open('POST', 'handler.php', true);

xhr.onload = function () {
  if (xhr.status !== 200) {
    console.log('An error occurred!');
  }
};

xhr.send(formData);
```

除了发送 FormData 实例，也可以直接 AJAX 发送文件。

```js
var file = document.getElementById('test-input').files[0];
var xhr = new XMLHttpRequest();

xhr.open('POST', 'myserver/uploads');
xhr.setRequestHeader('Content-Type', file.type);
xhr.send(file);
```

## IndexedDB API

> 现有的浏览器数据储存方案，都不适合储存大量数据：Cookie 的大小不超过4KB，且每次请求都会发送回服务器；LocalStorage 在 2.5MB 到 10MB 之间（各家浏览器不同），而且不提供搜索功能，不能建立自定义的索引。所以，需要一种新的解决方案，这就是 IndexedDB 诞生的背景。

## Web Worker

## 数据通信

前面说过，主线程与 Worker 之间的通信内容，可以是文本，也可以是对象。需要注意的是，这种通信是拷贝关系，即是传值而不是传址，Worker 对通信内容的修改，不会影响到主线程。事实上，浏览器内部的运行机制是，先将通信内容串行化，然后把串行化后的字符串发给 Worker，后者再将它还原。

主线程与 Worker 之间也可以交换二进制数据，比如 File、Blob、ArrayBuffer 等类型，也可以在线程之间发送。下面是一个例子。

```js
// 主线程
var uInt8Array = new Uint8Array(new ArrayBuffer(10));
for (var i = 0; i < uInt8Array.length; ++i) {
  uInt8Array[i] = i * 2; // [0, 2, 4, 6, 8,...]
}
worker.postMessage(uInt8Array);

// Worker 线程
self.onmessage = function (e) {
  var uInt8Array = e.data;
  postMessage('Inside worker.js: uInt8Array.toString() = ' + uInt8Array.toString());
  postMessage('Inside worker.js: uInt8Array.byteLength = ' + uInt8Array.byteLength);
};
```

但是，拷贝方式发送二进制数据，会造成性能问题。比如，主线程向 Worker 发送一个 500MB 文件，默认情况下浏览器会生成一个原文件的拷贝。为了解决这个问题，JavaScript 允许主线程把二进制数据直接转移给子线程，但是一旦转移，主线程就无法再使用这些二进制数据了，这是为了防止出现多个线程同时修改数据的麻烦局面。这种转移数据的方法，叫做[Transferable Objects](http://www.w3.org/html/wg/drafts/html/master/infrastructure.html#transferable-objects)。这使得主线程可以快速把数据交给 Worker，对于影像处理、声音处理、3D 运算等就非常方便了，不会产生性能负担。

如果要直接转移数据的控制权，就要使用下面的写法。

```js
// Transferable Objects 格式
worker.postMessage(arrayBuffer, [arrayBuffer]);

// 例子
var ab = new ArrayBuffer(1);
worker.postMessage(ab, [ab]);
```

## HTMLInputElement 的实例属性

> https://wangdoc.com/javascript/elements/input.html

### 特征属性

- `name`：字符串，表示`<input>`节点的名称。该属性可读写。
- `type`：字符串，表示`<input>`节点的类型。该属性可读写。
- `disabled`：布尔值，表示`<input>`节点是否禁止使用。一旦被禁止使用，表单提交时不会包含该`<input>`节点。该属性可读写。
- `autofocus`：布尔值，表示页面加载时，该元素是否会自动获得焦点。该属性可读写。
- `required`：布尔值，表示表单提交时，该`<input>`元素是否必填。该属性可读写。
- `value`：字符串，表示该`<input>`节点的值。该属性可读写。
- `validity`：返回一个`ValidityState`对象，表示`<input>`节点的校验状态。该属性只读。
- `validationMessage`：字符串，表示该`<input>`节点的校验失败时，用户看到的报错信息。如果该节点不需要校验，或者通过校验，该属性为空字符串。该属性只读。
- `willValidate`：布尔值，表示表单提交时，该`<input>`元素是否会被校验。该属性只读。
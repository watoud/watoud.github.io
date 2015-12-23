---
layout: post
title: JavaScript之变量与类型小记
date: 2015-12-16
comments: true
archive: true
tag: [JavaScript]
---

### 变量声明 <br/>
变量的声明使用关键字```var```,以字母、下划线或者$符号开头，后接数字。使用```var```关键字既可以声明局部变量，也可以声明全局变量。

```
/* 全局变量 */
var addr = "shenzhen";

function add(addition)
{
    /* 布局变量 */
    var base = 0;
    return base + (typeof addition === "number") ? addition : 0;
}
```

也可以不是用```var```关键字而直接给一个变量赋值，那么这个变量将是一个全局变量。

```
function add2(addition)
{
	/* 全局变量 */
	base = 0;
	return base + (typeof addition === "number") ? addition : 0;
}
```

### 变量计算
使用```var```关键字定义一个未初始化的变量，这个变量将会使用默认值```undefined```；访问一个未定义的变量将会抛出ReferenceError异常。

```
var a;
console.log("The value of a is " + a); // logs "The value of a is undefined"
console.log("The value of b is " + b); // throws ReferenceError exception
```

在数值计算中```undefined```计算值将会是NaN，而```null```将会被当做```0```。

```
var a;
a + 2;  // Evaluates to NaN

var n = null;
console.log(n * 32); // Will log 0 to the console
```

### 变量提升
在JavaScript中我们可以访问一个在当前语句后面定义的变量，这种特性称为变量提升（```variable hoisting```）。变量的声明将会提升到函数或者语句的最前面，且变量提升时拥有的值为```undefined```。

```
/**
 * Example 1
 */
console.log(x === undefined); // logs "true"
var x = 3;

/**
 * Example 2
 */
// will return a value of undefined
var myvar = "my value";

(function()
{
  console.log(myvar); // undefined
  var myvar = "local value";
})();
```

上面的代码跟下面代码的效果是一样。

```
/**
 * Example 1
 */
var x;
console.log(x === undefined); // logs "true"
x = 3;

/**
 * Example 2
 */
var myvar = "my value";

(function() {
  var myvar;
  console.log(myvar); // undefined
  myvar = "local value";
})();
```

### 数据类型及转换
JavaScript有6中基本类型以及对象类型。

| 数据类型        	| 说明     |
| :-----------  | :----------:  |
| Boolean	        |true或者false |
| null       | 特殊关键字，代表null       |
| undefined			| 顶级属性，它的值为undefined         |
| Number	        | 整数或者小数		|
| String     | 字符串        |
| Symbol   | ECMAScript6中的新类型  |
| Object   | 对象  |

JavaScript是一门动态语言，变量的类型可以在运行的过程中动态地发生变化。

```
var answer = 42;
......

answer = "Thanks for all the fish...";
```

数值与字符串进行```+```操作时，数值将会转换为String类型，但是数值与字符串进行其它操作的时候，数值并一定转换为String类型。

```
x = "The answer is " + 42 // "The answer is 42"
y = 42 + " is the answer" // "42 is the answer"

"37" - 7 // 30
```

有两种方式可以把字符串转换为数值：一种方式使用```parseInt()```或者```parseFloat()```方法；另一种使用```+```一元操作符。

```
"1.1" + "1.1" = "1.11.1"
(+"1.1") + (+"1.1") = 2.2
// Note: the parentheses are added for clarity, not required.
```

### 字面量
**数组字面量** <br/>

```
var coffees = ["French Roast", "Colombian", "Kona"];
```

如果在数组初始化中连续使用两个逗号，那么数组将会有一个没有undefined的元素，它的值为undefined，需要注意的是，在数组初始化中最后一个逗号将会被忽略。

```
var fish = ["Lion", , "Angel"];

var myList1 = ['home', , 'school', ];

var myList2 = ['home', , 'school', , ];
```

fish数组有3个元素，第一个元素为Lion，第二个元素为undefined，第三个元素为Angel。数组myList1的长度为3，myList2的长度为4。


**对象字面量** <br/>
对象字面量就是一系列包含在大括号中的key-value对，对象还可以进行嵌套。

```
var sales = "Toyota";

function carTypes(name)
{
  if (name == "Honda")
  {
    return name;
  }
  else
  {
    return "Sorry, we don't sell " + name + ".";
  }
}

var car = { myCar: "Saturn", getCar: carTypes("Honda"), special: sales };

console.log(car.myCar);   // Saturn
console.log(car.getCar);  // Honda
console.log(car.special); // Toyota

// 对象嵌套
var car = { manyCars: {a: "Saab", "b": "Jeep"}, 7: "Mazda" };

console.log(car.manyCars.b); // Jeep
console.log(car[7]); // Mazda
```

对象的属性可以是任意的字符串，包括空字符串。如果对象的属性不是一个有效的标识符或者数字，那么它们必须包含在引号中。访问属性名为非有效的标识符时，不能使用**```.```**进行访问，而应该使用类似类似数组的方式来访问（"[]"）。

```
var unusualPropertyNames =
{
  "": "An empty string",
  "!": "Bang!"
}
console.log(unusualPropertyNames."");   // SyntaxError: Unexpected string
console.log(unusualPropertyNames[""]);  // An empty string
console.log(unusualPropertyNames.!);    // SyntaxError: Unexpected token !
console.log(unusualPropertyNames["!"]); // Bang!

// ------------------------------------------

var foo = {a: "alpha", 2: "two"};
console.log(foo.a);    // alpha
console.log(foo[2]);   // two
//console.log(foo.2);  // Error: missing ) after argument list
//console.log(foo[a]); // Error: a is not defined
console.log(foo["a"]); // alpha
console.log(foo["2"]); // two
```

### 阅读资料
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Grammar_and_types












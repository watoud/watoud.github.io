---
layout: post
title: JavaScript之函数小记
date: 2015-12-23
comments: true
archive: true
tag: [JavaScript]
---

### 函数的定义
- function关键字
- 函数名称
- 用小括号括起来的逗号分隔的参数列表
- 用大括号括起来的语句

```
function square(number)
{
  return number * number;
}
```

除了这种方式方式之外，还可以用使用函数表达式以及Function构造函数来定义函数，通过函数表达式来定义的函数是一种匿名函数。

```
// 匿名函数
var square = function(number) { return number * number };
var x = square(4) // x gets the value 16
```

使用Function构造函数将创建一个函数对象，事实上在JavaScript中，任何一个函数都是一个函数对象。

```
//语法
new Function ([arg1[, arg2[, ...argN]],] functionBody)

// 示例
var adder = new Function('a', 'b', 'return a + b');
```

Function构造函数创建函数的效率比函数表达式以及使用function关键字的效率都要低，因为这类函数咋解析的时候需要带上函数体。<br/>

在JavaScript中可以基于判断条件来决定是否定义某个函数。

```
var myFunc;
if (num == 0)
{
  myFunc = function(theObject)
  {
    theObject.make = "Toyota"
  }
}
```

### 函数的作用域
在函数内部定义的变量是不能再函数外部访问的，但是函数可以访问改函数所在作用域内的所有变量以及函数。

```
// 全局作用域变量
var num1 = 20,
    num2 = 3,
    name = "Chamahk";

// 定义在全局作用域内的函数
function multiply()
{
  return num1 * num2;
}

multiply(); // 返回60

// 嵌套函数示例
function getScore ()
{
  var num1 = 2,
      num2 = 3;
  function add()
  {
    return name + " scored " + (num1 + num2);
  }
  return add();
}

getScore(); // 返回"Chamahk scored 5"
```

有三种方式可以让一个函数访问自己 <br/>

- 函数名称
- arguments.callee
- 函数变量

```
var foo = function bar()
{
   // statements go here
};
```

在上面的函数体重，bar()、arguments.callee()、foo的效果是一样的。

### 函数嵌套及闭包
在JavaScript中函数可以嵌套，内部函数形成一个闭包：内部函数可以访问外部函数的参数和变量，但是外层函数不能使用内部函数的参数和变量。

```
function outside(x)
{
  function inside(y)
  {
    return x + y;
  }
  return inside;
}
fn_inside = outside(3);

result = fn_inside(5); // returns 8

result1 = outside(3)(5); // returns 8
```

闭包必须要保存所有作用域中它所使用到的参数和变量，每次使用不同的参数对外层函数进行调用，都会产生一个不同的闭包。

### 函数的参数
函数的参数被保存在一个类似于数组的对象中，在函数体中可以使用```arguments[i]```对某个参数进行访问，参数的长度可以通过```arguments.length```获得。使用```arguments```对象，可以访问未在参数列表中声明的变量。

```
function myConcat(separator)
{
   var result = "", // initialize list
       i;
   // iterate through arguments
   for (i = 1; i < arguments.length; i++)
   {
      result += arguments[i] + separator;
   }
   return result;
}

// returns "red, orange, blue, "
myConcat(", ", "red", "orange", "blue");

// returns "elephant; giraffe; lion; cheetah; "
myConcat("; ", "elephant", "giraffe", "lion", "cheetah");

// returns "sage. basil. oregano. pepper. parsley. "
myConcat(". ", "sage", "basil", "oregano", "pepper", "parsley");
```

在ECMAScript 6之前函数没有默认参数，但是可以在函数体中对参数进行判断，如果是undefined则赋予某个值。

```
function multiply(a, b)
{
  b = typeof b !== 'undefined' ?  b : 1;

  return a*b;
}

multiply(5); // 5
```

### 箭号函数

```
// Basic
(param1, param2, …, paramN) => { statements }
(param1, param2, …, paramN) => expression
         // equivalent to:  => { return expression; }

// Parentheses are optional when there's only one parameter:
(singleParam) => { statements }
singleParam => { statements }

// Advanced
// A function with no parameters requires parentheses:
() => { statements }

// Parenthesize the body to return an object literal expression:
params => ({foo: bar})

// Rest parameters and default parameters are supported
(param1, param2, ...rest) => { statements }
(param1 = defaultValue1, param2, …, paramN = defaultValueN) => { statements }

// Destructuring within the parameter list is also supported
var f = ([a, b] = [1, 2], {x: c} = {x: a + b}) => a + b + c;
f();  // 6
```

### 阅读资料
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Functions
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions


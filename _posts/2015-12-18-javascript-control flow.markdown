---
layout: post
title: JavaScript之控制流程小记
date: 2015-12-18
comments: true
archive: true
tag: [JavaScript]
---
### 在布尔运算中转换为false的值
- false
- undefined
- null
- 0
- NaN
- the empty string ("")

所有其它的值包括对象，在布尔运算中都将被转换成true，需要注意值为false的Boolean对象在布尔表达式中依然是转换为true。

```
var b = new Boolean(false);
if (b) // this condition evaluates to true
```

### 异常处理
JavaScript可以抛出任何表达式的异常，如下所示。

```
throw "Error2";   // String type
throw 42;         // Number type
throw true;       // Boolean type
throw {toString: function() { return "I'm an object!"; } };
```

### for...in语句

```
    var book =
    {
        name:"Math",
        author:"foo",
        price:100
    };

    var result = "";
    for (tmp in book)
    {
        result += tmp+":"+book[tmp]+"\n";
    }
    // result = name:Math\nauthor:foo\nprice:100\n
```

在上面的代码中，tmp中每次迭代的值是对象中的properties的名称。

### 阅读资料
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Control_flow_and_error_handling
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for...in








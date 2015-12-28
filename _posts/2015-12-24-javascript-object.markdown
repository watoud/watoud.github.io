---
layout: post
title: JavaScript之对象小记
date: 2015-12-24
comments: true
archive: true
tag: [JavaScript]
---

### 列举对象的属性
在ECMAScript5中列举一个对象的属性有三种方法：

- for ... in
- Object.keys(o)
- Object.getOwnPropertyNames(o)


|  使用方法| 说明     |
| :-----------  | :----------:  |
|  for ... in  | 遍历一个对象及其原型对象上的所有可枚举属性 |
|Object.keys(o) | 返回一个对象的不包括原型对象的可枚举属性的数组|
|Object.getOwnPropertyNames(o)|返回一个对象的不包括原型对象的所有属性的数组|

```
function listAllProperties(o)
{
	var objectToInspect;
	var result = [];

	for(objectToInspect = o; objectToInspect !== null; objectToInspect = Object.getPrototypeOf(objectToInspect))
	{
		result = result.concat(Object.getOwnPropertyNames(objectToInspect));
	}
	return result;
}
```

### 创建对象
- 使用对象初始化<br/>

```
var object =
{
  foo: "bar",
  age: 42,
  baz: { myProp: 12 }
}
```

- 使用构造函数<br/>
通过一个构造函数来定义一个类型，一般构造函数的首字母大写；然后使用```new```关键字来创建对象。

```
// 定义构造函数
function Car(make, model, year)
{
  this.make = make;
  this.model = model;
  this.year = year;
}

// 通过构造函数创建对象
var mycar = new Car("Eagle", "Talon TSi", 1993);
```

- 使用```Object.create```方法<br/>
使用该方法创建对象时，我们可以选择创建对象的对象原型，而无需去定义构造函数。

```
// Animal properties and method encapsulation
var Animal =
{
  type: "Invertebrates", // Default value of properties
  displayType : function()
  {  // Method which will display type of Animal
    console.log(this.type);
  }
}

// Create new animal type called animal1
var animal1 = Object.create(Animal);
animal1.displayType(); // Output:Invertebrates

// Create new animal type called Fishes
var fish = Object.create(Animal);
fish.type = "Fishes";
fish.displayType(); // Output:Fishes
```

### 定义```getters```和```setters```方法<br/>

```
var o =
{
  a: 7,
  get b()
  {
    return this.a + 1;
  },
  set c(x)
  {
    this.a = x / 2
  }
};

console.log(o.a); // 7
console.log(o.b); // 8
o.c = 50;
console.log(o.a); // 25
```

o.a为对象o的一个属性；o.b为对象o的一个getter方法，返回a的值与1的和；o.c为对象o的一个setter方法，把a的值设为参数的一般。对于已经定义好的对象，可以使用```Object.defineProperty```函数为它们定义```getters```和```setters```方法。

```
var d = Date.prototype;
Object.defineProperty
(d, "year",
{
  get: function() {return this.getFullYear() },
  set: function(y) { this.setFullYear(y) }
});

var now = new Date;
console.log(now.year); // 2000
now.year = 2001; // 987617605170
console.log(now);
```

### 属性删除及对象比较
可以使用```delete```删除一个对象中非继承的属性，也可以用来删除不是使用```var```关键字定义的全局变量。

```
// Creates a new object, myobj, with two properties, a and b.
var myobj = new Object;
myobj.a = 5;
myobj.b = 12;

// Removes the a property, leaving myobj with only the b property.
delete myobj.a;
console.log ("a" in myobj) // yields "false"

g = 17;
delete g;
```

在JavaScript中对象是引用类型，两个不同的对象总是不相等的，就算他们拥有相同的属性。只有比较的对象是同一个对象的引用值的时候，它们才是相等的。

```
// Two variables, two distinct objects with the same properties
var fruit = {name: "apple"};
var fruitbear = {name: "apple"};

fruit == fruitbear // return false
fruit === fruitbear // return false

// Two variables, a single object
var fruit = {name: "apple"};
var fruitbear = fruit;  // assign fruit object reference to fruitbear

// here fruit and fruitbear are pointing to same object
fruit == fruitbear // return true
fruit === fruitbear // return true
```

### 对象的继承
在JavaScript中，对象的继承使用原型对象链。

```
function Employee()
{
  this.name = "";
  this.dept = "general";
}

function WorkerBee()
{
  Employee.call(this);
  this.projects = [];
}
WorkerBee.prototype = Object.create(Employee.prototype)

function Engineer()
{
   WorkerBee.call(this);
   this.dept = "engineering";
   this.machine = "";
}
Engineer.prototype = Object.create(WorkerBee.prototype);
```

![对象的继承关系](/assets/images/JavaScript/object/object-inherit.png)













### 阅读资料
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Working_with_Objects
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Details_of_the_Object_Model


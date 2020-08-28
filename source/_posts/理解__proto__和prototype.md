---
title: 理解__proto__和prototype
date: 2019-03-22 11:32:31
tags: JavaScript
categories:
  - JavaScript
---
\_\_proto\_\_和prototype是我们理解javascript最容易混淆的两个东西，但是理解了这两个对我们学习对象的继承和原型链有非常大的帮助。

下面先说说这两个东西的概念：

\_\_proto\_\_ ：指向一个对象的原型

prototype ：函数特有的一个对象属性，并不是一个空对象，该对象里有一个constructor属性，指向当前的构造函数

当我们使用new来执行一个函数时，实例会继承prototype对象上的属性，同时函数里的this将会指向调用这个构造函数的实例对象

下面将使用代码说明

```
let father = function () {
    this.a = 1
}

father.prototype.b = 2

let fatherIns = new father()

console.log(fatherIns) // {a: 1}
console.log(fatherIns.b) // 2
```

声明一个fatherIns的变量使用new关键词调用father，此时father函数里的this将会指向fatherIns，当我们打印这个对象时，fatherIns上并没有b这个属性，但是当我们尝试打印fatherIns.b时，之前我们在father.prototype上增加了一个<code>b = 2</code>的值，这时候js会尝试在这个对象原型链上寻找这个值，直到原型链最后为null，如果不存在这个属性则将会报出undefined的错误。

为什么fatherIns.b会在原型链上被找到，这个时候就轮到\_\_proto\_\_出场

\_\_proto\_\_是告诉js某个对象的原型对象指向谁，js就知道在原型链上下一步往哪去找

```
console.log(fatherIns.__proto__) // {b: 2}
```

所以我们可以得出以下结果

```
fatherIns.__proto__ === father.prototype
```

由此可知原型链就是这么来的，经典结论如下：

JS里一切皆对象，对象的原型为null

```
fatherIns.__proto__.__proto__ === Object.prototype

fatherIns.__proto__.__proto__.__proto__ === null
```

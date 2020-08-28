---
layout: /archives
title: 理解JS的eventloop
date: 2019-04-06 22:47:26
tags: JavaScript
categories:
  - JavaScript
---
异步，已经是js里变成必不可少的，而说到异步我们就不得不来说说js的event loop机制。

首先，一定要记住的一点：**js是单线程的。**

在event loop机制里，还涉及到宏任务（task）和微任务（Microtasks）概念，具体概念本文不作解释，说下属于这两个类型的分类：

1. 宏任务：script、setTimeout、setInterval
2. 微任务：Promise.then()、process.nextTick()

event loop在执行的时候先执行当前宏任务中的同步代码，碰到属于异步的代码，则先判断是属于宏任务还是微任务，然后注册新的任务到任务队列中。当前同步代码执行完成后将执行当前宏任务的微任务队列，直到微任务队列为空时我们将执行一个新的宏任务队列。

![](https://user-gold-cdn.xitu.io/2019/4/6/169f0f290787f313?w=800&h=658&f=jpeg&s=42740)

接下来就让我们用代码来理解event loop到底是怎么回事。


```
console.log(1)

setTimeout(function () {
    console.log(2)
})

console.log(3)
```

看到这段代码我们很容易知道答案

```
// 1 3 2
```

在这段代码里，setTimeout设置了一个定时任务（宏任务），将延迟多少时间后执行。

接下来再把promise加进来看看

```
console.log(1)

setTimeout(function () {
    console.log(2)
})

console.log(3)

new Promise(function (resolve, reject) {
    console.log(4)
    resolve()
}).then(function () {
    console.log(5)
})

console.log(6)
```

这里打印的结果是:

```
// 1 3 4 6 5 2
```

接下来我们对结果进行分析

首先执行同步的代码，new Promise()中的代码属于同步任务，所以先打印出1 3 4 6；当前代码有2个异步代码，setTimeout和Promise.then()，前面我们已经说过，setTimeout属于宏任务，Promise.then属于微任务，所以当前宏任务队列执行完后将执行当前队列的微任务。

所以最后部分打印结果 5 2

接下来再看一个例子：

```
console.log(1)

setTimeout(function () {
    console.log(2)
})

console.log(3)

new Promise(function (resolve, reject) {
    console.log(4)
    setTimeout(function () {
        console.log(5)
    })
    resolve()
}).then(function () {
    console.log(6)
    setTimeout(function () {
        console.log(7)
    })
})

console.log(8)
```

打印结果为：

```
// 1 3 4 8 6 2 5 7
```

分析的结果和上一部分代码是一样的，setTimeout属于宏任务将在当前宏任务执行完成后执行，执行顺序和注册任务的先后顺序有关。

最后再两个例子说明script和setTimeout属于宏任务类型

第一个例子

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<script>
    console.log(1)

    setTimeout(function () {
        console.log(2)
    })

    console.log(3)

    new Promise(function (resolve, reject) {
        console.log(4)
        resolve()
    }).then(function () {
        console.log(5)
        setTimeout(function () {
            console.log(7)
        })
        new Promise(function (resolve, reject) {
            console.log(8)
            resolve()
        }).then(function () {
            console.log(9)
        })
    })

    if (true) console.log(6)
</script>
<script src="index2.js"></script>
</body>
</html>
```

第二个例子

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<script>
console.log(1)

setTimeout(function () {
    console.log(2)
    let node = document.createElement('script')
    node.src = './index2.js'
    document.body.append(node)
})

console.log(3)

new Promise(function (resolve, reject) {
    console.log(4)
    resolve()
}).then(function () {
    console.log(5)
    setTimeout(function () {
        console.log(7)
    })
    new Promise(function (resolve, reject) {
        console.log(8)
        resolve()
    }).then(function () {
        console.log(9)
    })
})

if(true) console.log(6)
</script>
<script src="index2.js"></script>
</body>
</html>
```

index2.js的代码如下

```
console.log(1)

setTimeout(function () {
    console.log(2)
})

console.log(3)

new Promise(function (resolve, reject) {
    console.log(4)
    resolve()
}).then(function () {
    console.log(5)
    setTimeout(function () {
        console.log(7)
    })
    new Promise(function (resolve, reject) {
        console.log(8)
        resolve()
    }).then(function () {
        console.log(9)
    })
})

if(true) console.log(6)
```

在这两个例子中setTimeout的执行顺序和我们script创建时的顺序有关，根据event loop的执行机制分析即可。


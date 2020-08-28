---
layout: /archives
title: JS数据类型判断
date: 2020-08-15 22:52:01
tags: JavaScript
categories:
  - JavaScript
---
# **前端数据类型判断**

> 前端数据类型判断在开发过程中经常会遇到，那么我们又该怎么判断各种不同的数据类型呢？通过阅读文章，给你一套完整的判断方法以及对前端JS里数据类型的基本了解。

## 数据类型

javascript中的基础数据类型共有8种

1. string
2. number（特殊值NaN）
3. boolean
4. undefined
5. object（正则表达式、array、map、set、function属于对象类型）
6. null
7. symbol
8. bigInt



## 判断方法

### typeof方法判断

下面代码列出我们开发中经常用到的数据类型进行比较，代码如下👇

```javascript
typeof 1 // 'number'
typeof NaN // 'number'
typeof '1' // 'string'
typeof true // 'boolean'
typeof undefined // 'undefined'
typeof null // 'object'
typeof [] // 'object'
typeof {} // 'object'
typeof /\d/ // 'object'
typeof new Date('2020') // 'object'
typeof new Map() // 'object'
typeof new Set() // 'object'
typeof function() {} // 'function'
typeof Symbol() // 'symbol'
typeof BigInt(1) // 'bigInt'
```

把结果整理成表格如下方表格展示，通过表格中的结果对比来看，使用<code>typeof</code>方法判有以下2个缺陷：

1. 无法区分**数字**和**NaN**
2. 无法区分**null**、**{}**、**[]**、**/\d/**、**new Map()**、**new Set()**这些对象类型

| 参数                                     | 结果     |
| ---------------------------------------- | -------- |
| 1、NaN                                   | number   |
| '1'                                      | string   |
| true                                     | boolean  |
| null、{}、[]、/\d/、new Map()、new Set() | object   |
| function() {}                            | function |
| Symbol()                                 | symbol   |
| BigInt(1)                                | bigInt   |

### toString方法判断 

<code>toString</code>方法是判断数据类型最精确的，下面代码列出我们开发中经常用到的数据类型进行比较，代码如下👇

```javascript
let func = (val) => {
	return Object.prototype.toString.call(val)
}
func(1) // '[object Number]'
func(NaN) // '[object Number]'
func('1') // '[object String]'
func(true) // '[object Boolean]'
func(undefined) // '[object Undefined]'
func(null) // '[object Null]'
func([]) // '[object Array]'
func({}) // '[object Object]'
func(/\d/) // '[object RegExp]'
func(new Date('2020')) // '[object Date]'
func(new Map()) // '[object Map]'
func(new Set()) // '[object Set]'
func(function() {}) // '[object Function]'
func(Symbol()) // '[object Symbol]'
func(BigInt(1)) // '[object BigInt]'
```

把结果整理成表格如下方表格展示，通过表格中的结果对比来看，使用<code>toString</code>方法判断数据类型已经接近完美，但是特殊的值<code>NaN</code>还是无法区分，不过这个数据在开发过程中还是比较少出现的，判断方法在下文补充。

| 参数             | 结果               |
| ---------------- | ------------------ |
| 1、NaN           | [object Number]    |
| '1'              | [object String]    |
| true             | [object Boolean]   |
| undefined        | [object Undefined] |
| null             | [object Null]      |
| []               | [object Array]     |
| {}               | [object Object]    |
| /\d/             | [object RegExp]    |
| new Date('2020') | [object Date]      |
| new Map()        | [object Map]       |
| new Set()        | [object Set]       |
| function() {}    | [object Function]  |
| Symbol()         | [object Symbol]    |
| BigInt(1)        | [object BigInt]    |

## 特殊点

- 通过前文中整理可得，<code>NaN</code>这个特殊值无论是用<code>typeof</code>还是<code>toString</code>都无法区分，判断这个值最好使用<code>Number.isNaN</code>，该方法返还一个布尔值。

  ```javascript
  Number.isNaN(NaN) // true
  ```

- 还有一类数据是使用构造函数生成，这种数据从使用<code>typeof</code>和<code>toString</code>得到的结果来看，<code>typeof</code>方法得到的结果会更准确。

  ```javascript
  let func = (val) => {
  	return Object.prototype.toString.call(val)
  }
  
  let newStr = new String('1')
  let newNum = new Number(1)
  let newBool = new Boolean(true)
  
  typeof newStr // 'object'
  typeof newNum // 'object'
  typeof newBool // 'object'
  
  func(newStr) // '[object String]'
  func(newNum) // '[object Number]'
  func(newBool) // '[object Boolean]'
  ```

  从上面代码可以得到，在这个时候虽然使用<code>toString</code>方法得到的数据构造方法是正确的，但是真正的变量返回结果却是对象类型的，对于这种使用构造函数得到的返回值，结合<code>typeof</code>和<code>toString</code>一起判断是最精确的。

- 对于判断数组类型，我们还可以使用新的语法<code>Array.isArray</code>判断，返回的结果是一个布尔值。

---
title: 不懂就问系列-JavaScript到底怎么判断数据类型
date: 2020-05-01 15:11:18
categories: 
- javascript
tags:
- javascript
- 不懂就问
---


# typeof
返回一个值的基本数据类型，但对于引用类型的数据都返回 object
typeof 可以判断除了 null 和 数组 以外的数据类型

```javascript
typeof null // object
typeof [] // object
```

# instanceof 

说到 instanceof 需要先提一下 Object.constructor 属性，对象的 constructor 属性 指向 改对象的构造函数。

```javascript
function Person() {}
const p1 =  new Person(); // p1.constructor === Person  true
```

但是 constructor 的指向也是可以认为变更的，它不一定始终指向它的构造函数


# Object.prototype.toString.call()

通过这个方法能判断所有类型, 注意下返回 字符串的大小写就可以了

```javascript
Object.prototype.toString.call([]); // [object Array]
Object.prototype.toString.call(null); // [object Null]
Object.prototype.toString.call(123); // [object Number]

```




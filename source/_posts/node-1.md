---
title: Node模块机制
date: 2019-08-31 23:37:17
categories: 
- NODE
tags:
- NODE
---

## CommonJS 模块规范
主要分为模块引用、模块定义和模块标识3个部分。
### 模块引用
```JavaScript
const foo = require('foo')
```
### 模块定义
使用require进行引入，上下文提供了exports对象用于导出当前模块的方法或者变量，并且它是唯一的出口。在模块中还存在一个module对象，代表模块自身。在Node中一个文件就是一个模块，将方法挂载到exports对象上作为属性即可定义导出的方式。
```JavaScript
// foo.js
exports.sum = (a,b) => a+b;
```
### 模块标识
模块标记就是传递给require()方法的参数，其实就是文件模块的路径。

## Node的模块实现
在Node中，模块分为两类:一类是Node提供的模块，称为核心模块;另一类是用户编写的，称为文件模块。
- 核心模块在node源代码的编译过程中，编译进了二进制执行文件中。在NOde进程启动时，核心模块就直接被加载到内存中。
- 文件模块则是在 `运行时动态加载`，需要完整的路径分析、文件定位、编译执行过程，速度比核心模块慢。
注：Node对引用过的模块多会进行缓存，缓存的是编译和执行后的对象。

在Node中引入模块，需要经历下面3个步骤
1. 路径分析
2. 文件定位
3. 编译执行
  
### 路径分析
require接受一个模块标识作为参数，基于这个参数进行模块查找。
核心模块和路径形式的引用不细作说明。
这里说下 `自定义模块` 分析过程,自定义模块指的是既非核心模块，也不是路径形式的标识符。
模块路径是Node在定位未见时的查找策略，具体表现为一个路径组成的数组。路径的生成规则大概如下：
- 当前文件目录下的node_modules目录。
- 父目录下的node_modules目录。
- 父目录的父目录下的node_modules目录。 
- 沿路径向上逐级递归，直到根目录下的node_modules目录。

### 文件定位
- 扩展名
require()在分析标识符的过程中，会出现标识符中不包含文件扩展名的情况。CommonJS模块规范也允许在标识符中不包含文件扩展名，这种情况下，Node会按.js、.json、.node的次序补 足扩展名，依次尝试。
- 目录分析和包
在分析标识符的过程中，require()通过分析文件扩展名之后，可能没有查找到对应文件，但
却得到一个目录，这在引入自定义模块和逐个模块路径进行查找时经常会出现，此时Node会将目 录当做一个包来处理。
在这个过程中，Node对CommonJS包规范进行了一定程度的支持。首先，Node在当前目录下 查找package.json(CommonJS包规范定义的包描述文件)，通过JSON.parse()解析出包描述对象， 从中取出main属性指定的文件名进行定位。如果文件名缺少扩展名，将会进入扩展名分析的步骤。
而如果main属性指定的文件名错误，或者压根没有package.json文件，Node会将index当做默 认文件名，然后依次查找index.js、index.json、index.node。

### 模块编译
在Node中每个文件模块都是一个对象
```JavaScript
function Module(id, parent) { 
  this.id = id;
  this.exports = {}; 
  this.parent = parent;
  if (parent && parent.children) { 
    parent.children.push(this);
  }
  this.filename = null; 
  this.loaded = false; 
  this.children = [];
}
```
编译和执行是引入文件模块的最后一个阶段。定位到具体的文件后，Node会新建一个模块对 象，然后根据路径载入并编译。对于不同的文件扩展名，其载入方法也有所不同，具体如下所示。
- .js文件。通过fs模块同步读取文件后编译执行。
- .node文件。这是用C/C++编写的扩展文件，通过dlopen()方法加载最后编译生成的文件。  .json文件。通过fs模块同步读取文件后，用JSON.parse()解析返回结果。
- 其余扩展名文件。它们都被当做.js文件载入
每一个编译成功的模块都会将其文件路径作为索引缓存在Module._cache对象上，以提高二次引入的性能。

#### JavaScript模块的编译
回到CommonJS模块规范，我们知道每个模块文件中存在着require、exports、module这3个
变量，但是它们在模块文件中并没有定义，那么从何而来呢?甚至在Node的API文档中，我们知 道每个模块中还有__filename、__dirname这两个变量的存在，它们又是从何而来的呢?如果我们 把直接定义模块的过程放诸在浏览器端，会存在污染全局变量的情况。
事实上，在编译的过程中，Node对获取的JavaScript文件内容进行了头尾包装。在头部添加 了(function (exports, require, module, __filename, __dirname) {\n，在尾部添加了\n});。 一个正常的JavaScript文件会被包装成如下的样子:
```JavaScript
(function (exports, require, module, __filename, __dirname) { 
  var math = require('math');
  exports.area = function (radius) {
    return Math.PI * radius * radius; 
  };
 });
```
这样每个模块文件之间都进行了作用域隔离。包装之后的代码会通过vm原生模块的
runInThisContext()方法执行(类似eval，只是具有明确上下文，不污染全局)，返回一个具体的 function对象。最后，将当前模块对象的exports属性、require()方法、module(模块对象自身)， 以及在文件定位中得到的完整文件路径和文件目录作为参数传递给这个function()执行。
这就是这些变量并没有定义在每个模块文件中却存在的原因。在执行之后，模块的exports 属性被返回给了调用方。exports属性上的任何方法和属性都可以被外部调用到，但是模块中的 其余变量或属性则不可直接被调用。

此外，许多初学者都曾经纠结过为何存在exports的情况下，还存在module.exports。理想情
况下，只要赋值给exports即可:
```JavaScript
exports = function () { // My Class
```
}; 但是通常都会得到一个失败的结果。其原因在于，exports对象是通过形参的方式传入的，
直接赋值形参会改变形参的引用，但并不能改变作用域外的值。测试代码如下:
```JavaScript
var change = function (a) { a = 100;
console.log(a); // => 100
};
var a = 10;
change(a); console.log(a); // => 10
```
如果要达到require引入一个类的效果，请赋值给module.exports对象。这个迂回的方案不改 变形参的引用。
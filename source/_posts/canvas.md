---
title: 不懂就问系列-Canvas和WebGL是啥关系？
date: 2019-10-15 10:52:14
categories: 
- WebGL
tags:
- Canvas
- WebGL
- 不懂就问
---

> 一直以来我都有一个很大的弱点，就是很多东西不会，但是不敢问，怕别人觉得问题太可笑，结果就是不会的的东西大部分都还不会。如何解决呢，我认为作为一个程序员，不懂就问，厚脸皮的提出来，这是一项很基本的能力，故开启这个“不懂就问系列”，希望对自己甚至对别人有所帮助。

最近学习制作 [《一镜到底》](https://github.com/huguobo/One-Take) 的h5页面，用到了PIXI.JS，发现它对于自己的描述是这样的：
> PixiJS is a rendering library that will allow you to create rich, interactive graphics, cross platform applications, and games without having to dive into the WebGL API or deal with browser and device compatibility.PixiJS has full WebGL support and seamlessly falls back to HTML5's canvas if needed. 

PIXI全面支持 WebGL 但是作为兜底会使用 Canvas，就是说他们两是一个层面的东西，有点晕了，查查资料，问问大佬搞清楚一点吧LOL

## Canvas
先说下Canvas，Canvas（翻译为画布）是HTML5的一个标签，Canvas提供了给JavaScript在浏览器内绘制的能力。

```HTML
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>Canvas</title>
</head>
<body onload="main()">
    <canvas id="container" width="1280px" height="720px"></canvas>
</body>
</html>
<script type="text/javascript" src="main.js"></script>
```

```JavaScript
function main() {
    var canvas = document.getElementById("container");
    var context = canvas.getContext("2d");
    context.fillStyle = "rgba(0, 0, 255, 1.0)";
    context.fillRect(120, 10, 150, 150);
}
```

Canvas只支持一些简单的2D绘制，不支持3D，更重要的是性能有限，WebGL弥补了这两方便的不足。

## WebGL
说 `WEBGL` 之前需要先了解下 [OpenGL](https://zh.wikipedia.org/wiki/OpenGL), 既然涉及到绘图能力，底层方面来说实际上是与显卡的交互，OpenGL是 底层的驱动级的图形接口（是显卡有直接关系的，类似于DirectX（玩PC游戏应该都接触过这个）。
但是我们想在浏览器里用JS使用这方面的图形渲染能力呢？这时候出现了WebGL，WebGL（全写Web Graphics Library）是一种3D绘图标准，WebGL允许工程师使用JS去调用部分封装过的 OpenGL ES2.0 标准接口去提供硬件级别的3D图形加速功能

最简单的webgl使用方法，
```JavaScript
function main() {
    var canvas = document.getElementById("container");
    var gl = canvas.getContext("webgl") || canvas.getContext("experimental-webgl");
    gl.clearColor(0.0, 0.0, 0.0, 1.0);// 指定清空canvas的颜色
    gl.clear(gl.COLOR_BUFFER_BIT);// 清空canvas
}
```
## 关系总结
Canvas就是画布，只要浏览器支持，可以在canvas上获取2D上下文和3D上下文，其中3D上下文一般就是WebGL，当然WebGL也能用于2D绘制，并且WebGL提供硬件渲染加速，性能更好。
但是 WEBGL 的支持性[caniuse](https://caniuse.com/#search=webgl)还不是特别好，所以在不支持 WebGL 的情况下，只能使用 Canvas 2D api，注意这里的降级不是降到 Canvas，它只是一个画布元素，而是降级使用 浏览器提供的 **Canvas 2D Api**，这就是很多库的兜底策略，如 Three.js, PIXI 等








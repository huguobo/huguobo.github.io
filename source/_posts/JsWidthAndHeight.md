---
title: WEB开发中涉及的各种宽高
date: 2019-09-09 13:56:05
categories: 
- JavaScript
tags:
- scorllTop
- height
- width
---

> web开发中经常涉及各种宽高和坐标的概念，相关的属性和概念较多，每次都得查一遍，故作此整理。

JS中Document对象的宽高有3个种类：`client`, `offset` , `scroll` 

## Client

### clientHeight 和 clientWidth
该属性指的是元素可视部分的宽度和高度，即`padding+content`， 不包括滚动轴等。
![](/../images/clientHeight.png)

### clientTop 和 clientLeft
该属性是读取元素 `border` 的宽度和高度的。

- clientTop = border-top 的 broder-width
- clientLeft = border-left 的 border-width


## Offset

### offsetHeight 和 offsetWidth
这一对属性指的是元素的 `border+padding+content` 的宽度和高度。包括padding、border、水平滚动条，但不包括margin的元素的高度。
该属性和其内部的内容是否超出元素大小无关，只和本来设置的border以及width和height有关。
![](/../images/offsetHeight.png)

### offsetTop 和 offsetLeft
这个需要先了解下 `offsetParent`：第一个 position 不为  static 的 父元素，一层层向上，知道 body。
当前元素顶部距离最近父元素`offsetParent`顶部的距离,和有没有滚动条没有关系。单位px，只读元素。 
同理left

## Scroll

### scollHeight 和 scrollWidth
因为子元素比父元素高，父元素不想被子元素撑的一样高就显示出了滚动条，在滚动的过程中本元素有部分被隐藏了，scrollHeight 代表包括当前不可见部分的元素的高度。而可见部分的高度其实就是clientHeight，也就是scrollHeight>=clientHeight恒成立。在有滚动条时讨论scrollHeight才有意义，在没有滚动条时scrollHeight==clientHeight恒成立。单位px，只读元素。 
![](/../images/scollHeight.png)

### scrollTop 和 scrollLeft
代表在有滚动条时，滚动条向下滚动的距离也就是元素顶部被遮住部分的高度。在没有滚动条时scrollTop==0恒成立，单位px。
![](/../images/scrollTop.png)


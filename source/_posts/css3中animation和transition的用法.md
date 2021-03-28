---
title: css3中animation和transition的用法
date: 2016-09-21 23:31:39
author: Shin
tags:
  - CSS
categories: CSS
---
### 1.animation 

用法：

```css
animation: rotate 10s linear infinite;
```

或是：

```css
animation-name: rotate;
animation-duration: 10s;
animation-timing-function: linear;
animation-interation-count: infinite;
```

* ainmation-name：设置动画名称，而动画名称一般又是@-keyframes设置的，类似于一个函数名。

```css
@-webkit-keyframes rotate {
  from { transform: rotate(0deg); }
  50% { transform: rotate(180deg); }
  to { transform: rotate(360deg); }
}s
```

> from(0%)是动画的起始点，to(100%)是动画的终点，而from和to的中间用50%来取中间关键帧的位置。

* animation-timing-function：是控制动画时间的属性

> 属性默认取值ease，取值linear,ease-in,ease-out,ease-in-out,cubic-beziern,n,n),cubic-bezier三次贝塞尔曲线可以自定义动画的时间函数，可以通过[工具网站](http://cubic-bezier.com/)定制，还有困惑的steps() 函数。要实现帧动画效果而不是线性的变化就需要引入steps这个值了，换句话就是没有过渡的效果，而是一帧帧的变化

* animation-duration：表示动画运行一个周期的时间。

* animation-delay：表示动画的运行前的延迟时间。

* animation-iteration-count：设置动画运行次数。

>取值可以是 n|infinite。infinite表示无限循环。

* animation-direction: 设置动画的方向

>取值normal | reverse | alternate | alternate-reverse

>+ reverse 从最终状态往初始状态变化
>+ alternate 往返变化，开始为初始状态
>+ alternate-reverse 往返变化，开始为最终状态

* animation-fill-mode: 设置动画的方向

>取值none | forwards | backwards | both
>- forwards 元素的最终状态为动画执行的终点
>- backwards 元素的最终状态为元素动画之行前
>- both 根据animation-direction轮流应用forwards和backwards规则,根据animation-direction轮流应用forwards和backwards规则

* animation-play-state：设置动画的状态。

>取值可以是：paused|playing。

>现代浏览器一般内置了监听animation动画执行的事件，animationstart，animaionend容易理解，而animationiteration是发生在动画每个迭代的结束，跟animation-direction配合比较好理解。

### 2.transition过渡

用法：

```css
transition: property duration timing-function delay;
```

写法：

```css
transition: width 3s ease 1s;
```

或是：

```css
transition-property: width;
transition-duration: 3s;
transition-timing-function: ease;
transition-delay: 1s;
```

>对象属性改变中间的缓冲。过渡有4个属性值。

* transition-property：设置对象参与过渡的css属性。

* transition-duration：设置过渡的持续时间。

* transition-timing-function：设置对象过渡的动画类型。

>跟animation-timing-function类似。

* transition-delay：设置对象延迟的过渡时间。


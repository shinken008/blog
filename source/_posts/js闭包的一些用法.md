---
title: js闭包的一些用法
date: 2016-09-22 23:18:43
author: Shin
tags:
  - JS 
categories: JS
---
### 1.封装变量

> 闭包可以帮助把一些不暴露在全局变量封装成“私有变量”。假如有个计算乘机的简单函数：

```javascript
var mult = function() {
  var a = 1;
  for (var i = arguments.length - 1; i >= 0; i--) {
    a = arguments[i];
  }
  return a;
}
```

> mult函数接受一些number类的参数，并返回这些参数的乘机。对于已经进行过计算的参数乘机，再次额外的计算是一种浪费，我们可以运用到缓存的知识提高函数的性能：

```javascript
var cache = {};
var mult = function() {
  var argument = Array.prototype.sort.apply(arguments);
  var args = Array.prototype.join.call(argument, ',');
  if(cache[args]) {
    return cache[args];
  }
  var a = 1;
  for (var i = arguments.length - 1; i >= 0; i--) {
    a = a * arguments[i];
  }
  return cache[args] = a;
}

console.log(mult(1,2,3))
```

>我们又看到cache只在函数内使用，但是又是跟函数暴露在同一个作用域，不如把它封装在函数内部，减少页面的全局变量，以免变量在其他地方修改而引发的错误。

```javascript
var mult = function() {
  var cache = {};
  return function() {
    var argument = Array.prototype.sort.apply(arguments);
    var args = Array.prototype.join.call(argument, ',');
    if(cache[args]) {
      return cache[args];
    }
    var a = 1;
    for (var i = arguments.length - 1; i >= 0; i--) {
      a = a * arguments[i];
    }
    return cache[args] = a;
  }
}

console.log(mult()(2,3,4));  //24
```

>提炼函数是一种技巧。如果能将大函数代码独立成来，有助于代码的复用，而且对于独立出来的小函数如果有个好的命名，也能起到注释的作用。如果这些小函数不需要在程序的其他地方使用，最好把它用闭包封装起来。

```javascript
var mult = function() {
  var cache = {};
  var calc = function() {
    var a = 1;
    for (var i = arguments.length - 1; i >= 0; i--) {
      a = a * arguments[i];
    }
    return a;
  };
  return function() {
    var argument = Array.prototype.sort.apply(arguments);
    var args = Array.prototype.join.call(argument, ',');
    if(args in cache) {
      return cache[args];
    }
    return cache[args] = calc.apply(null, arguments);
  }
}

console.log(mult()(1,3,4));     //12
```

### 2.延续局部变量的寿命

> img对象经常用于数据上报，如下所示：

```javascript
var report = function(src) {
  var img = new Image();
  img.src = src;
};

report('http://htmljs.b0.upaiyun.com/uploads/1398932756598-angularjs.jpg');
```

>有些浏览器使用report函数进行数据上报会丢失30%左右的数据，也就是说report函数并不是每一次都成功发起http请求。丢失的原因是img是report函数中的局部变量，当report函数调用结束后，img局部变量随即被销毁，而此时或许还没来得及发出http请求，所以此次请求就会丢失掉。我们用闭包将img变量封装起来，就能解决请求丢失的问题：

```javascript
var report = function() {
  var imgs = [];
  return function(src) {
    var img = new Image();
    imgs.push(img);
    img.src = src;
  }
}

report()('http://htmljs.b0.upaiyun.com/uploads/1398932756598-angularjs.jpg');
```
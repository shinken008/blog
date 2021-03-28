---
title: js多维数组处理
date: 2017-11-25 22:30:58
author: Shin
tags:
  - JS
categories: JS
---

处理数据结构时，数组运用在不同场景，下面罗列多维数组处理成一维数组的方法
### 1.运用字符串对数组进行处理
```javascript
// 处理[[1],[2, [3, [4]]]]
function flatten(arr) {
  const symbol = ','
  const str = arr.join(symbol)
  return str.split(symbol)
}
```
这种做法在有很大的局限性，对于处理数字而言，split后数组number变成string，处理string数组收到symbol的影响等

### 2.使用数组concat方法
```javascript
// 处理[[1],[2, {}]]
function flatten(arr) {
  return Reflect.apply([].concat, [], arr)
  // return Array.prototype.concat.apply([], arr)
}
```
apply只能处理二维数数组一维数组，但是保持了数据的原始性，推荐使用Reflect代替对象原型方法，使代码更加简洁易懂。

### 3.递归函数
```javascript
// 处理[[1],[2, [3, 4, [5, 6]]]]
var newArr = []
function flatten(arr) {
  arr.map(l => Array.isArray(l) ? flatten(l) : newArr.push(l))
  return newArr
}
```
需要预定义额外的变量用于保存递归的数据

### 4.reduce 递归
```javascript
// 处理[[1],[2, [3, 4, [5, 6]]]]
function flatten(arr) {
  return arr.reduce((a, b) => a.concat(Array.isArray(b) ? flatten(b): b), [])
}
```
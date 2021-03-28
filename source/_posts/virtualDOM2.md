---
title: '实现自己的虚拟DOM: Props & Events'
date: 2017-08-09 14:51:28
author: Shin
tags: React
categories: JS
---

很开心可以继续分享这个话题，下一个要讨论的就是事件驱动，将使我们实现的婴儿虚拟dom运用到真实的项目开发。

今天我们主要讨论的的是setting／diffing属性（props）和事件处理。

### babel处理

在这之前我么处理一个普通的标签

```html
<div></div>
```

通过babel处理会转化成props为null，因为没有属性，结果如下：

```javascript
{ type: 'div', props: null, children: [] }
```
我们最好将它（props）处理成一个空的对象，因为我们将要对props进行迭代取值

```javascript
function h(type, props, ...children) {
    return { type, props: props || {}, children };
}
```
设置简单的属性，比如下面的，会在我们的dom中怎么展现呢？我们将它存在一个对象里。

```html
<ul className="list" style="list-style: none"></ul>
```
我们在内存中将在如下展示
```javascript
{
    type: 'ul',
    props: { className: 'list', style: 'list-style: none' },
    children: []
}
```
我们取对象的属性值给我们真实的节点设置属性。

```javascript
fucntion setProp($target, name, value) {
    $target.setAttribute(name, value);
}
```
我们知道了设置一个属性，设置所有属性只不过是遍历一下props对象。
```javascript
function setprops($target, props) {
    Object.keys(props).forEach((name) => {
        setProp($target, name, props[name])
    })
}
```
现在还记得createElement()函数吗？我们仅仅在创建珍视的dom的时候给节点设置属性。

```javascript
function createElement(node) {
    if (typeof node === 'string') {
        return document.createTextNode(node)
    }
    const $el = document.createElement(node.type)
    setProps($el, node.props)
    node.children
        .map(createElement)
        .forEach($el.appendChild.bind($el));
    return $el
}
```
但是这没有结束，我们遗漏了一些细节，首先class在javascript是个关键字，所有我们不能用class，我们用className代替。
```html
<nav className="navbar light">
  <ul></ul>
</nav>
```
但是我们在真实的dom里面没有className属性这个值，所有我们要通过setProps（）函数处理。还有就是属性值设置成布尔值代表属性存在与否。
```html
<input type="checkbox" checked={true} />
```
在上面我不希望上面的属性值运用在真实的dom中，
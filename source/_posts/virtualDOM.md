---
title: 实现自己的虚拟DOM
date: 2017-07-15 21:18:43
author: Shin
tags: React
categories: JS
---

你需要知道两件事如果你创建自己的虚拟DOM，你甚至不需要深入React的源码或者深入其他的虚拟DOM实现的源码。他们太大也太复杂，但是我们可以根据核心代码实现一个简要的虚拟DOM仅仅50行的代码。

### 以下是两个概念

>* 虚拟DOM是代替真实DOM的一种呈现
>* 当我们改变我们的虚拟DOM树时，我们会得到一个新的DOM树。通过比较这两个DOM树（新的和旧的），发现不同的地方，我们仅仅需要将那些不同的地方在真实DOM树呈现，所以定义成虚拟的DOM。

### 下面就让我们深入这两个概念

## 我们的DOM

首先我们需要我们的DOM树存在缓存里，我们可以用一个普通的js对象才存。比如我们有这么个DOM树：

```html
<ul class="list">
  <li>item 1</li>
  <li>item 2</li>
</ul>
```
是不是看起来非常简单。我们该怎么用js的对象来表达这个DOM树呢？

```javascript
{ type: 'ul', props: { 'class': 'list' }, children: [
  { type: 'li', props: {}, children: ['item 1'] },
  { type: 'li', props: {}, children: ['item 2'] }
] }
```

这是可以注意到两件事:

>* 我们用对象来表述我们的DOM
```javascript
{ type: '…', props: { … }, children: [ … ] }
```
>* 我们用string来存DOM文本节点

写一颗很大的树在一定的程度上是很难的，所以我们用函数来帮助我们，所以我们能非常简单去理解这种结构

```javascript
function h(type, props, …children) {
  return { type, props, children };
}
```

现在我们能写出我们的DOM树：

```javascript
h('ul', { 'class': 'list' },
  h('li', {}, 'item 1'),
  h('li', {}, 'item 2'),
);
```

这样它看起来就非常清爽。但是我们还能做的更好些。在这之前你应该听过JSX。这里需要用到它，那它是怎么工作的呢？

如果你读了官方的babel [JSX](https://babeljs.io/docs/plugins/transform-react-jsx/)文档，babel将把代码编译成：

```html
<ul className="list">
  <li>item 1</li>
  <li>item 2</li>
</ul>
```
顺畅的转成：

```javascript
React.createElement('ul', { className: 'list' },
  React.createElement('li', {}, 'item 1'),
  React.createElement('li', {}, 'item 2'),
);
```

可以注意到一些相似点。React.createElement()和我们的h函数调用，这结果就是我们能平滑的使用JSX。我们仅仅需要在文件顶部行加入一行注释：

```html
/** @jsx h */
<ul className="list">
  <li>item 1</li>
  <li>item 2</li>
</ul>
```

所以我们将可以编译下面代码通过babel转义：

```javascript
const a = (
  h('ul', { className: 'list' },
    h('li', {}, 'item 1'),
    h('li', {}, 'item 2'),
  );
);
```

当我们使用f函数执行时，他将变成普通的js对象--我们的虚拟DOM:

```javascript
const a = (
  { type: 'ul', props: { className: 'list' }, children: [
    { type: 'li', props: {}, children: ['item 1'] },
    { type: 'li', props: {}, children: ['item 2'] }
  ] }
);
```

尝试使用jsfiddle来跑下：

<iframe
  style="width: 100%; height: 300px"
  src="https://jsfiddle.net/shinken/zadmbrgf/">
</iframe>

## 运用我的的DOM来展示

现在我们自己用普通的对象来展示DOM树。这是非常好的事情，但是我们还需要知道怎么通过它来创建真实的DOM。我们可不能就把我们的DOM插入真实的DOM中。

首先我们足一些假设和约定术语:

>* 我将写些可用真实的DOM节点(element, text nodes)，开头用"$"--$parent 将成为真实的DOM元素
>* 虚拟DOM的展示将用可用的节点来展示
>* 像React一样，你有且只有一个根节点--其他的所有节点都包含在这里面

好，就像上面说的那样，让我们写一个createElement()函数将虚拟DOM转化为真实的DOM。从现在开始忘掉'props'和'children'，我们在后面讨论：

```javascript
function createElement(node) {
  if (typeof node === 'string') {
    return document.createTextNode(node);
  }
  return document.createElement(node.type);
}
```

所以，因为我们文本和节点--用普通的js对象就是字符串和元素--这种js对象就是像这种：

```javascript
{ type: '…', props: { … }, children: [ … ] }
```

结果是我们可以通过虚拟的文本节点和DOM节点，它将可以工作。

现在我们需要考虑的是子节点--每个可能是文本节点或者是元素节点。所以他们同意可以用createElement()函数来创建。你可能想到了，所以我们可以调用createElement()遍历所以的子元素然后将它们appendChild()到我们的元素像这样：

```javascript
function createElement(node) {
  if (typeof node === 'string') {
    return document.createTextNode(node);
  }
  const $el = document.createElement(node.type);
  node.children
    .map(createElement)
    .forEach($el.appendChild.bind($el));
  return $el;
}
```

这样写看起来很好，现在我们在将'props'加上。我们等下再来进行讨论，我们不需要理解虚拟DOM的基本概念，但是它们将添加些更复杂的东西。

现在我们用jsfiddle试下：

<iframe
  style="width: 100%; height: 300px"
  src="https://jsfiddle.net/shinken/zdb00kvg/"
>
</iframe>

## 处理变化

现在我们能把虚拟DOM转化成真实的DOM了，是时候考虑虚拟DOM树的differ算法了。所以基本上我们需要写个算法，他将对边两个虚拟DOM树（新的和旧的），而我们需要将这些改变写入真实的DOM中。

怎样去比对两颗虚拟DOM树呢？我们需要处理下面那些问题：

>* 有新的节点增加了，所以node增加了，所以我们需要appendChild()

![](images/1.png)

>* 没有新的节点然而节点被删了所以我们需要removeChild()
![](images/2.png)

>* 节点比对结果节点全被不一样，所以我们需要replaceChild()
![](images/3.png)

>* 有些节点是一样的，所以我们需要进行深入的比较节点的不一样
![](images/4.png)

好，让我们写一个updateElement()的函数有三个参数$parent, newNode, oldNode, 而$parent是一个真实的DOM，作为我们虚拟DOM的父元素。现在我们来考虑怎么去解决上面描述的问题。

## 没有旧节点的情况下

下面这是非常直接了当，简直是不想解释：

```javascript
function updateElement($parent, newNode, oldNode) {
  if (!oldNode) {
    $parent.appendChild(
      createElement(newNode)
    );
  }
}
```

## 没有新节点的情况下

这是有一个问题，假如没有节点在新的，我们需要把它从真实的DOM中移除，但是我们应该怎么做呢？我们知道父元素(通过一个函数)结果我们目的是调用$parent.removeChild()通过真实的DOM元素引用。但是我们没有这些，然而，假如我们知道父节点的位置，我们就可以得到他的引用通过$parent.childNodes[index]，而下标index就是子节点相对父节点的位置。

我们目的是通过我们的函数找到我们的下标。我们的代码可能会是这个样子：

```javascript
function updateElement($parent, newNode, oldNode, index = 0) {
  if (!oldNode) {
    $parent.appendChild(
      createElement(newNode)
    );
  } else if (!newNode) {
    $parent.removeChild(
      $parent.childNodes[index]
    );
  }
}
```

## 节点改变了

首先我们需要去写一个函数去对比这两个新的和旧的节点，如果改变了就通知我们。我们应该考虑它可能是元素或者是文本节点：

```javascript
function changed(node1, node2) {
  return typeof node1 !== typeof node2 ||
         typeof node1 === 'string' && node1 !== node2 ||
         node1.type !== node2.type
}
```
现在我们有当前节点在父节点的下标我们可以很轻易的替换它通过创建一个新的节点：

```javascript
function updateElement($parent, newNode, oldNode, index = 0) {
  if (!oldNode) {
    $parent.appendChild(
      createElement(newNode)
    );
  } else if (!newNode) {
    $parent.removeChild(
      $parent.childNodes[index]
    );
  } else if (changed(newNode, oldNode)) {
    $parent.replaceChild(
      createElement(newNode),
      $parent.childNodes[index]
    );
  }
}
```

## 对比子元素

最后，我们应该通过对比新旧的两个节点去--事实上就是他们调用updateElement()去遍历它们。是的再次递归。

但是我们有些事情要考虑清楚在我们开始写代码：

>* 我们应该对比子节点仅且当节点是一个元素的时候(文本节点是没有子节点的)
>* 现在我们通过引用当前的节点作为父节点
>* 我们应该一个一个的对边所以的子元素--甚至有些情况我们会有'undefined'--没事--我们的函数可以处理
>* 再就是找下标--它仅仅在是在chilren数字里的子节点的下标

```javascript
function updateElement($parent, newNode, oldNode, index = 0) {
  if (!oldNode) {
    $parent.appendChild(
      createElement(newNode)
    );
  } else if (!newNode) {
    $parent.removeChild(
      $parent.childNodes[index]
    );
  } else if (changed(newNode, oldNode)) {
    $parent.replaceChild(
      createElement(newNode),
      $parent.childNodes[index]
    );
  } else if (newNode.type) {
    const newLength = newNode.children.length;
    const oldLength = oldNode.children.length;
    for (let i = 0; i < newLength || i < oldLength; i++) {
      updateElement(
        $parent.childNodes[index],
        newNode.children[i],
        oldNode.children[i],
        i
      );
    }
  }
}
```

## 将所有的结果放在一块

是的我们把所有的代码放在jsfiddle里面，真的花了50行代码

<iframe
  style="width: 100%; height: 300px;"
  src="https://jsfiddle.net/deathmood/0htedLra"
>
</iframe>

如果用开发者工具的话可以看到变化当你点击'reload'按钮时

![](images/5.gif)

## 总结

我们已经做到了，我们完整地写了个我们自己的虚拟DOM。而且他能工作。我希望正在读这篇文章的你能理解这些基本的概念--虚拟DOM什么工作的和react在这引擎下是怎么工作的。

然而有些事情在这里我们没有延伸到（我将在后续的文章对此进行讨论）

>* 设置元素属性和对比和更新他们
>* 处理事件--增加事件监听我们的元素
>* 使我们的虚拟DOM在组件里工作，像React
>* 获取真实DOM的节点引用
>* 使用虚拟DOM的第三方库和突变真实的DOM--像jQuery和它的插件
>* 更多...

原文地址：![https://medium.com/@deathmood/how-to-write-your-own-virtual-dom-ee74acc13060](https://medium.com/@deathmood/how-to-write-your-own-virtual-dom-ee74acc13060)
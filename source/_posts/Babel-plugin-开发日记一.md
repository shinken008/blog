---
title: Babel/plugin 开发日记一
date: 2020-10-21 00:12:22
author: Shin
top: true
tags: Babel
categories: Babel
---
## traverse 开始遍历

 https://github.com/babel/babel/blob/86f535b8635e780d0707c47ae03658de27ae08bd/packages/babel-core/src/transformation/index.js#L112

```js
// merge all plugin visitors into a single visitor
const visitor = traverse.visitors.merge(
  visitors,
  passes,
  file.opts.wrapPluginVisitorMethod,
);
traverse(file.ast, visitor, file.scope); 
```



## plugin 执行顺序

按钩子顺序执行，比如 plugin 里面有 Program.enter，则执行完Program.enter在执行下一个的Program.enter，等程序退出在再顺序触发Program.exit，一次按 plugin 顺序执行钩子。traverse visitor 之前会进行 plugins 里的 visitor 合并，钩子对象都具有 enter 和 exit 方法，钩子函数默认为 enter 函数，最后都合并到一个大的 visitor里面，比如 `{ ImportDeclaration: { enter: [...], exit: [...] } }`

## 一个简单的插件格式

```js
function babelPlug(babel) {
  return {
    name: 'babelPlug',
    visitor: {
      Program(path, state) {
        console.log(path, state);
      }
    }
  }
}
```



## visitor.Program

```js
visitor: {
  // visitor contents
  // Visitor 中的每个函数接收2个参数：path 和 state
  Program: {
    enter(path, state) {
      
    },
    exit(path, state) {
      
    },
  },
  Program(path, state) { // 等同于 enter 进入这个钩子
    
  }
}
```

## path.get('body')

获取 Program 的 node.body 里面的 Nodes

```js
for (const stmt of astPath.get('body')) {
  if (t.isImportDeclaration(stmt)) {
    // program import 语法
  }
}
```

## state && state.file 的 Map 管理

Program 状态管理，state Program 共享，整个 plugin 链式调用的状态共享，所以要注意状态冲突。

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gjvo3a3fqkj30ad03odgh.jpg)

File --> https://github.com/babel/babel/blob/86f535b8635e780d0707c47ae03658de27ae08bd/packages/babel-core/src/transformation/file/file.js#L34
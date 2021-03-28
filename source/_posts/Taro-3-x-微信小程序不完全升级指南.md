---
title: Taro@3.x 微信小程序不完全升级指南
date: 2020-09-13 00:17:42
author: Shin
top: true
tags:
  - 工程化
  - 小程序
  - Taro
categories: 小程序
---



## 前言

`Taro@3.0` 发版以来，基本上保持在一周的时间发版，主要是修复 `bug`。通过社区可以了解到，`Taro@3.x` 发版带来了重大变革，其中对 `H5` 和小程序场景进行重构，提供了新的特性和新的架构。

`Taro@3.0` 发版之后，社区一直很活跃，不少用户开始使用一键式（命令）升级。

```sh
$ taro update project [version]
```

敲完之后是这样：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghlwweyel1j30di0w0n03.jpg" width="150" />

还有这样的：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghlx3dg9dyj30nr02wq3a.jpg" />

看到这些我差点笑出声，这是我在 [issue](https://github.com/NervJS/taro/pull/7240) 在讨论 `3.x` 时看到的，这位同学版本应该是敲错了。不得不说逛社区是件很有意思的事情，有的时候看 `issue` 能发现一些同道中人，你踩的坑别人也在前赴后继[手动尴尬]。

`Taro` 目前还有很多的 [坑](https://github.com/NervJS/taro/issues) ，现在有 600+ 的 `issue` 数，平摊到各个平台其实不会很多，咋看一下 [V-3](https://github.com/NervJS/taro/issues?q=is%3Aissue+is%3Aopen+label%3AV-3) `issue` 也就几十个。也有可能是用户基数的关系，毕竟刚发版不久，幸存者偏差。这里我不是要劝退想要升级的同学，就像官方说的，”没有枪，没有炮，没有轮子自己造“，不要怂，就是*！首先我们先看看 `Taro@3.x` 特性。

<b>以下是在小程序场景下升级 `Taro@3.x` ，其他端升级仅供参考和学习。</b>

## Taro@3.x 特性

- 开放式框架
- 渲染 HTML 字符串
- 充满争议的 CSS-In-JS
- 虚拟列表 VirtualList 
- 预渲染 Prerender
- source-map 支持
- 更快的构建速度
- 更快的运行速度
- Babel@7
- ...

看到它那么多特性，似乎很诱人，详细参考 [官方文档](https://taro-docs.jd.com/)，文档有的东西在这里就不过多的赘述了。

下面是如何用上它呢？如果是新项目那很好办，只需要实现新功能，对于旧项目呢？”买不了吃亏，买不了上当“，以其忍受旧的技术栈，不如升级玩出新花样。

#### Taro@3.x 是个啥

首先先简单了解下框架原理和解决的问题。

##### 框架

下面是引用官方的一张架构图：

<img width="800" src="https://storage.jd.com/taro-source/taro-docs/WechatIMG1393.png" />

##### 解决了什么问题

- 提供了上面新的特性，比如解决开发体验上的问题；
- 得益于新的编译运行机制，我们不用去遵守以前的最佳实践（约束）了。[官方最佳实践](https://nervjs.github.io/taro/docs/best-practice/)；
  - JSX 变量在模板语法使用 `this.state.x`，编译的时候生成未赋值的 `data` 值，需要定义一个变量赋值 `this.state.x` ；
  - 不必遵守 `render*` 渲染 `JSX` 的约定；
  - 不必遵守 `on*` 渲染传递函数名的约定；
  - ...

从架构层面上讲，Taro 从一个编译型框架变成了一个运行时框架，基本上曾经的 `Taro` 无法运行的代码在 `Taro Next` 中完全没有压力。`Taro@3.x` 在运行时维护了一个 `DOM` 模型，使得编译的时候不去做 `data xml` 转化，从而规避掉了编译带来的 `bug`，同时降低学习成本，`PC` 端开发的同学也能轻松接入。

看起来挺香的。了解了一些知识后，下面我们先从依赖开始升级。

## Taro@3.x 依赖

以下是基于 `Taro@2.2.5` 升级到基于 `React@16.10.0` 框架 `Taro@lastest(@3.0.7)` 的历程。

#### 框架选型

对用户来说，开放式框架给了用户更多地选择，原来的版本只支持的类 `React` 语法，现在完全可以使用社区的其他框架，比如 `Vue 2`、`Vue 3`、`jQuery`。在这里因为我是一个 `React` 的重度用户，比起官方的 `Nerv`，倾向于 `React`。而且 `React` 是一个非常优秀的开源项目，有庞大的团队在维护。这里安利一下 [React](https://github.com/facebook/react)，[React@17.0](https://reactjs.org/blog/2020/08/10/react-v17-rc.html) 发布了，只是一个过渡版本，消除了设计上的隐患，帮助用户更安全的升级过渡，新的功能特性放在 `React@18` 发布，有没有一种服务到家的感觉（这里有点要黑 `Taro@3.x break change` 式升级的嫌疑）。

跑题了，下面继续介绍 `Taro`升级，下面是选择 `React` 框架需要改造的地方：

- 安装 `React` 技术栈所使用的 `npm` 包；
- 框架  `API`从 `Taro` 移到第三方框架，比如：`import React, { Component } from 'react'`；
- 配置 `config.framework: 'react'`；
- `babel.config.js` 配置 `React` 相关 `babel`，这里可以使用官方的 `babel-preset-taro`，下面【配置更改】会讲到；

#### Babel@7

在 `Taro@2.x` 使用的 `babel@6`，了解 `babel` 同学应该知道 6.x 版本和 7.x 版本的差异，比如 `Presets` ，新的 `Proposals` 等命名，`babel@7` 使用 `@babel/x` ` scope` 代替 `babel/x`，防止被占用。

#### 相关依赖包升级

- 使用 `npm` 挨个安装升级的需要 `babel` 包了；
- 安装官方 `babel-preset-taro`；
- 使用 `cli`。将 `taro-cli` 升级到3.x，执行 `taro update project[version]` 命令。建议使用 `cli` 升级命令最好是已经升级到 3.x 版本，对于2到3的升级，建议手动升级各个依赖包，因为只能`cli` 检测 `dependencies` [某一些包 ](https://github.com/NervJS/taro/blob/c171ddaedb413bac1f93a92f038f3a09a3e01e3b/packages/taro-helper/src/constants.ts#L129)进行升级。

> 这里我使用的是手动升级，先删除对应老版本的 `npm` 包，再安装新版本

```sh
npm i @babel/runtime @tarojs/components @tarojs/runtime @tarojs/taro @tarojs/react react-dom@16.10.0 react@16.10.0
```

```sh
npm i -D @tarojs/cli @types/webpack-env @types/react @tarojs/mini-runner @babel/core @tarojs/webpack-runner babel-preset-taro eslint-config-taro eslint eslint-plugin-react eslint-plugin-import eslint-plugin-react-hooks stylelint
```

> 各个端的相关包。比如原生小程序转过来会用 `with-weapp` 包一层

```sh
npm i @tarojs/with-weapp // withWeapp 接受一个小程序规范的 Page/App 构造器参数，转换为对应框架规范的组件实例
npm i @tarojs/taro-weapp // 微信小程序
npm i @tarojs/taro-alipay //解决方案支付宝小程序
```

> 新增的包。这里使用安装 `babel` `presets` 或者插件，因为我偷懒，安装 `babel-preset-taro`，然后在 `babel-config.js` 配置上老项目使用的插件和预设，参考文档配置或者 [代码 ](https://github.com/NervJS/taro/blob/c171ddaedb413bac1f93a92f038f3a09a3e01e3b/packages/babel-preset-taro/index.js#L4 )。

```sh
npm i @babel/runtime @tarojs/react
npm i @tarojs/runtime // Taro 运行时。在小程序端连接框架（DSL）渲染机制到小程序渲染机制，连接小程序路由和生命周期到框架对应的生命周期。在 H5/RN 端连接小程序生命周期规范到框架生命周期
npm i -D @babel/core babel-preset-taro // babel 相关
```

> 删除 `babel@7` 以下的包。安装 `babel-preset-taro` 可配置相关的 `babel`，如果有些 `babel presets plugins` 在`babel-preset-taro` 没有集成的话，请手动安装

```sh
npm remove babel-plugin-transform-class-properties babel-plugin-transform-decorators-legacy babel-plugin-transform-jsx-stylesheet babel-plugin-transform-object-rest-spread babel-plugin-transform-runtime babel-preset-env babel-runtime @tarojs/plugin-babel
```

> 删除老的 `nerv` 框架相关的包

```sh
npm remove nervjs nerv-devtools
```

#### 配置更改

- `babel-config.js`。在 `Taro@3.x` 之前是 `babel` 配置在 `config/index` 配置文件里面，现在把它放在根目录的 `babel` 配置文件里面

  ```js
  module.exports = {
    presets: [
      ['taro', {
        framework: 'react',
        ts: true // 是否启用ts
      }]
    ],
  };
  ```

  

- `.eslintrc`。在 `Taro@3.x` `eslint-plugin-taro` 废弃，因为之前的配置以及编码最佳实践将不适用。如果开启了 `ts` 编译，还需要安装`@typescript-eslint/parser` 和 `@typescript-eslint/eslint-plugin`。

  ```js
  {
    "extends": ["taro/react"]
  }
  ```

## Taro@3.x 业务代码改造

#### 页面 config 独立文件

在 `Taro` 1.x，2.x 时，配置是在页面实例里面去配置 `config`，编译后输出 `x.json` 配置文件。在 `Taro@3.x` 里，需要在同级目录里新增一个独立的 `JS` 文件去配置：`x.config.js`。

在这里需要注意的一点是，跟页面代码一起的 `config` 会通过 `webpack` 编译，而独立成 `x.config.js` 只会被 `babel-register` 进行编译，在项目里刚好 `config` 配置的 `pages` 常量是根据不同的场景定义的 `webpack` 常量，所以 `config.js` 取不到 `webpack` 常量配置报错。解决方案是，使用 [babel-plugin-transform-define](https://www.npmjs.com/package/babel-plugin-transform-define)，主要不能与 `webpack` 常量冲突，需要额外定义 `config.js` 里面的常量。这种解决方式毕竟不优雅，官方考虑在后续添加这个特性，[相关issue](https://github.com/NervJS/taro/issues/7011)。

#### config/index 配置文件

- `Config.framework` 配置当前小程序使用的框架 `React | Vue | Nerv`；
- `Config.defineContants`。`Taro@2.x` 常量的 `value` 可以使用 `key: '我是字符串'` 。在Taro@3.x 编译会报错，要用 `String` 再包一层，比如 `key: JSON.stringify('我是字符串')`，这里建议定义的常量值都使用 `JSON.stringify` 处理一下。
- `Config.terser `压缩配置项，terser 配置只在生产模式下生效。如果你正在使用 `watch` 模式，又希望启用 `terser`，那么则需要设置，`process.env.NODE_ENV` 为 `production`，将不会生成 `source-map` 文件。

#### 页面实例的改动

- `Taro.Component` 替换为 `React.Component`；

- 路由信息 `this.$router is undefined`；

  > `this.$router` 用 `Taro.getCurrentInstance().router`（或者 `Taro.Current.router`）代替。在项目入口文件 `app.js` 实例里面 `componentWillMount`, `componentDidMount` 取到 `Taro.Current.router is null`，路由信息只有在`componentDidShow`生命周期才能读取到，因为 `componentWillMount`, `componentDidMount` 生命周期 `App.onLaunc`h 方法里面触发，生命周期触发之后赋值 `Current.router`，[代码实现](https://github.com/NervJS/taro/blob/794c29055d393b6f33abb1e4054b023902c6f49c/packages/taro-runtime/src/dsl/react.ts#L222)。

- `Taro.getApp() is undefined`；

  > 在 实例化（`createReactApp`）之前调用 `Taro.getApp()` 将返回 `undefined`。我们的业务代码比如在 `app.js` 最顶层 `require(./common.js)`，如果 `common.js` 包含 `Taro.getApp() [x]` 使用将会报错。

- 页面实例方法的改变，比如`onShareMessage` 改成` static onShareMessage`；

  > 参考[代码实现]()。

- `e.currentTarget.dataset` 空对象;

  > `e.currentTarget.dataset` 不兼容。解决方案：使用 `func.bind(this, ...args)` 绑定参数的形式，参考 [issue](https://github.com/NervJS/taro/issues/7313)。这个解决方案改动也很多，绑定了 `dataset` 都要进行改造。我还是支持 `dataset` 的实现，这样框架更接近原生。

- `withWeapp` 问题；

  > 官方并没有在发布 3.0 的时候对 `withWeapp` 改造，导致 `withWeapp` 几个报错，`this.$router`，`this.state` 取值都是 `undefined`；解决方案：1、不用这个[装饰器](https://taro-docs.jd.com/taro/docs/taroize/#二次开发)。但是通常我们有不得以的苦衷，比如业务代码里面大量使用，更改代码的成本太高。2、官方[PR](https://github.com/NervJS/taro/pull/7240)，预计在 `Taro@3.0.8 `发布，并不会等太久；

- 没有了 `this.$scope` 和 `this.$componentType` 的概念；

  > 只是实现改变了，`runtime` 维护一个 `DOM` 内存结构，所以能直接取了。

  ``` js
  // Taro@2.x
  const ctx = Taro.createCanvasContext("image-cropper", this.$scope);
  const selectQuery = Taro.createSelectorQuery().in(this.$scope);
  
  // Taro@3.x
  const ctx = Taro.createCanvasContext("image-cropper");
  const selectQuery = Taro.createSelectorQuery();
  ```

  

- ...

## Taro@3.x 多场景下框架的兼容

在 `Taro@2.x` ，小程序分包我们可以使用宿主小程序的框架，但在升级 `Taro@3.x `过程中，我们需要分析 2.x 和 3.x 框架版本能不能做到共用。这个章节我们针对主包、分包和插件场景下对小程序的运行、打包和包大小进行分析，演示多场景框架升级的可行性。

#### 独立运行

在58小程序体系里面，经常会有同一份源码编译到不同端的场景，以及借助内部工程化管理工具 `MPS` 更为便捷的管理分包，组件。在分包和插件的场景下，是一份这样的目录结构：

<img width="200" src="https://tva1.sinaimg.cn/large/007S8ZIlgy1ghpj0nbtjfj30go10udiv.jpg" /><img width="200" src="https://tva1.sinaimg.cn/large/007S8ZIlgy1ghpj573w1wj30gc0l0tai.jpg" />

##### 分包

在分包场景中，为了减少包大小，分包尽量减少了依赖第三方 `vendors` 的注入，比如 `Taro`，共用主包的 `vendor`，删掉了无效的 `app.x` 和 `project.config.json`。在升级 `Taro@3.x` 的过程中，由于框架机制的改变，而无法做到与低于 3.0 的版本做到框架的共用。因为在3.x，各个 `Page` 依赖入口文件 `app.js`  `createReactApp(App: React.ComponentClass, react: typeof React, reactdom, config: AppConfig)`  创建的 `Current`（包含 `app` ,  `router` ,  `page` ）和 `Reconciler` 实例，就是 `page` 依赖执行 `app.js` 文件（参考[代码实现](https://github.com/NervJS/taro/blob/next/packages/taro-runtime/src/dsl/react.ts)）。但是分包的 `app.js` 不是作为入口文件执行的，咋办呢？

我想到的是在各个文件 `require('./**/app.js')`，在加载完整个分包后执行 `app.js`，实现 `App` 的实例化。

```js
// app.js
export default class App extends React.Component {
  componentDidMount() {
    console.log('app.onLaunch Taro.Current:', Taro.Current);
  }
  render() {
    return this.props.children;
  }
}
// home/list.js
export default class Home extends Component {
  componentDidMount() {
    console.log('page.onLoad Taro.Current:', Taro.Current);
  }
  componentDidShow() {
    console.log('page.onShow Taro.Current:', Taro.Current);
  }
  render() {
    return (
      <View>hello world</View>
    )
  }
}
```

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1ghrcpc6i2gj318w044754.jpg)

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1ghpl6odjprj30tg046my1.jpg)

可以看到，在入口文件我们取到实例，且在 `Page` 页面也能取到实例。但是有个问题发现了没，我们是跟主包共用一个 `App` 实例，这会导致我们改掉 `App` 实例的内容影响到主包。

注入 `app.js` 方案在分包场景是没有问题的，但是怎么解决 `App` 实例影响问题呢？这个问题等下再回答，我们再来以插件场景进行尝试这个方案。

##### 插件

还是跟分包一样的入口文件和 `page` 页面。我们先看下编译后的 `app.js` 代码长啥样：

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1ghplq8fs2yj319g050ac2.jpg)

这就是 `Taro@3.x` 编译后的入口文件，下面我们引入这个插件：

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1ghplws5w9nj31g606kadp.jpg)

发现在插件模式下，微信小程序没有提供 `App` 实例。

##### 分析分包和插件页面加载方案

从上面可以看出，`require('./**/app.js')方案` 从运行机制上来看可行，但是存在一些问题：

- 分包引入 `app.js` 执行会引入主包 `App` 实例被改问题；

- 插件模式下，微信小程序没有提供 `App` 实例报错；
- 怎么给每个 `Page` 注入 `app.js`；

怎么解决上面的问题呢？所以我们想到，我们能不能在分包和插件自己实现一个 `App` 函数，并且提供一个 `onLaunch` 空方法，让`taro-runtime` 能够执行`createReactApp` 呢？下面是我们使用插件 `@mps/mps-taro-plugin/dist/MpsRuntimeTaroPlugin` 实现了一个 `App` 函数：

```js
export function App(config) {
  config.onLaunch({})
}
```

并且通过 `webpack.ProvidePlugin` 注入到 `app.js` 中，打包后的文件是这样的：

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1ghpma9wki3j31ew09mjul.jpg)

我们再来手动引入 `app.js` 试试，没有报错，程序能正常运行。

我们不能每个 `Page` 都手动注入 `app.js` 吧？接下来如何给每个 `Page` 引入 `app.js` 呢？这时候我们可以通过 `webpack` 插件做这件事情，刚好我们现有 `@mps/mps-taro-plugin/dist/MpsBusinessTaroPlugin` 已经提供了这个能力，安装 `@mps/mps-taro-plugin@3.0.2` 版本。下面是提供 `App` 函数和给每个 `Page ` 注入 `app.js` 的插件配置：

```js
plugins: [
	'@mps/mps-taro-plugin/dist/MpsRuntimeTaroPlugin',
   ['@mps/mps-taro-plugin/dist/MpsBusinessTaroPlugin', {
      commonChunks: ['app'], 
   }],
]
```

撒花！通过以上的方式解决了多场景下框架版本的兼容性问题。

#### 打包文件

##### 入口文件处理

多场景如何定义入口文件，这是一个问题，我们经常在入口文件主包需要引用一一些 `sdk` 之外，然而在分包和插件场景并不需要使用到。比如在 `Taro2.x` 的是按 `WEBPACK_CONST` 条件将 `import` 进来模块赋值给已经赋值过的变量，引入 `target` 模块定义的上下文在没有使用的时候会被 `tree-shaking` 掉，其实这应该是一个 `bug`。或者说是现在的 `webpack + uglyfy(terser)` 目前没有实现的一个特性（[类消除](https://github.com/mishoo/UglifyJS/issues/1261)也是后面实现的），理论上编译器只有在静态分析100%确定没问题的情况下才会删，不会去分析程序流，意味着你的分包和插件在不使用的一些代码都会被打包进来。听起来可能不好理解，我们直接看代码，感兴趣的同学可以尝试下，分别放在 `Taro@3.x` 和 `Taro@2.x` 编译：

```js
// App.js 入口文件
import { cube } from './util.js';
let value = null;
if (WEBPACK_CONST) { // 场景常量，WEBPACK_CONST = false
	value = cube(2);
}

// utils.js
console.log('before utils');
export function square(x) {
  console.log('square'); 
  return x * x;
}

export function cube(x) {
  console.log('cube');
  return x * x * x;
}
```

- `Taro@3.x` 下，不会被删掉；

  > Development，`console.log('before utils');` 没被删掉，且 `unused` 模块也没有被删掉。

  <img width="400" src="https://tva1.sinaimg.cn/large/007S8ZIlly1ghqbvnwuy3j30p3092gme.jpg" />

  > Production，`console.log('before utils');` 没被删掉， `unused` 模块被删掉。

  ![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghqbh3qdtnj30be02tglw.jpg)

  

- `Taro@2.x` 下

  > Development，Production，`console.log('before utils');` 被删掉，`unused` 模块被删掉

  ![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghqc5m3a07j307302qglj.jpg)

如果入口文件沿用 `Taro@2.x` 的写法会带来几个问题:

- `import`， `require`，`requirePlugin` 等引入上下文会执行；

- 影响包体的大小；
- `Taro@3.x` `dev` 环境不会 `tree-shaking`，参考上面的配置说明 【`config/index` 配置文件】；

解决上面问题很简单，我们用 `webpak + require + defineConstants` 的方式对各个场景按需引入，比如：`if (WEBPACK_CONST) { require('..') }`，当 `WEBPACK_CONST` （编译常量）为 `false` 的时候，`Webpack`  编译分析不会把条件外的 `require` 内容引入进来。有了这个结论我们就对入口文件进行了处理：

```js
// app.js
let App = () => null;
if (WEBPACK_CONST === 'a') {
  App = require('./app.a').default;
}
if (WEBPACK_CONST === 'b') {
  App = require('./app.b').default;
}
if (WEBPACK_CONST === 'c') {
  App = require('./app.c').default;
}

export default App;

// app.a.js
import React from 'react'
import Taro from "@tarojs/taro"

export default class App extends React.Component {
  componentDidMount() {
    console.log('app.onLaunch Taro.Current:', Taro.Current);
  }
  render() {
    return this.props.children;
  }
}
```

咋一看，上面的文件是解决了，但是不太优雅，如果支持配置指定入口文件是不是更好。可惜的是， `Taro@3.x` 在小程序场景中的将入口文件写死了 `app.x`，所以不得不使用这种方式去做到分入口加载。感兴趣的同学可以看看这个 [PR](https://github.com/NervJS/taro/pull/7192)，`entryFileName` 其实已经支持在 `H5` 和 `RN` 里面配置。

#### 包大小

玩游戏最怕的是”一顿操作猛如虎，一看战绩零杠五“。要是包大小不通过，所有的解决方案都是白忙活。下面是我通过开发者工具统计的包大小信息：

|      | Taro@2.2.5 | Taro@3.0.7 |
| ---- | ---------- | ---------- |
| 主包 | 1469.0kb   | 1369.0kb   |
| 分包 | 1246.1kb   | 1235.0kb   |
| 插件 | 1493.0kb   | 1456.0kb   |

根据上面统计的信息，在主包，分包和插件场景下，现有的包大小没有超过原来的包大小。

## 总结

在这里，我们可以认为升级 `Taro@3x` 在现有的58体系里面是可行的，改动的范围也是可以接受的。

我是从发布不久开始接入，可以说从 `Taro@3.0.2` 一路踩坑过来的，开始的时候因为 `api` 改动大有点打击信心，但随着了解了一些运行机制之后和简单的开发体验后，越发觉得 `Taro@3.x` 颠覆式的重构反而能让 `Taro` 走的更远。有兴趣的同学可以按照上面的步骤进行升级，基本上没啥问题，有啥问题可以给提 `issue` 提 `PR`，官方人手不多，一起帮忙加特性（改 `bug` ）。还是那句话，”没有枪，没有炮，没有轮子自己造“。

## 参考资料

[1] Taro 3 正式版发布：开放式跨端跨框架解决方案: [https://aotu.io/notes/2020/06/30/taro-3-0-0/index.html](https://aotu.io/notes/2020/06/30/taro-3-0-0/index.html)

[2] 从旧版本迁移到 Taro Next: [https://nervjs.github.io/taro/docs/migration](https://nervjs.github.io/taro/docs/migration)

[3] Taro Next 发布预览版: [https://juejin.im/post/6844904063675400199](https://juejin.im/post/6844904063675400199)

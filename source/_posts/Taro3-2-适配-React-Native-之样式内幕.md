---
title: Taro3.2 适配 React Native 之样式内幕
date: 2021-03-28 11:13:23
author: Shin
top: true
tags:
  - Taro
  - React Native
  - 开源
categories: React Native
---


## 背景
从 Taro 3 开始，58同城成为 Taro 的战略合作伙伴，负责 Taro 3 React Native 部分的研发和推广。我们总结以往 JD Taro 同学们适配上的经验，以及内部对 React Native 使用的技术沉淀，为了能更好的提升开发体验，因此我们提出新的架构方案[0]。

在新的架构设计下，以往对样式处理的方案需要设计接入。在接入的过程中，需要考虑到 React-Native 的样式管理与样式的差异，框架提供对用户友好的兼容方案，以便在工程开发上更好的组织代码。

React-Native 的样式支持基本上是实现了 CSS 的一个子集，但属性名不完全一致。更大的不同是没有对层叠样式表支持，不能使用 class 读取静态样式，所以在跟 Web 的适配上有很大的困难。

## 适配上的问题
开始接入样式系统之前，先看一段代码

常见的 React Native 代码设置样式：

```jsx
import React from "react";
import { StyleSheet, Text, View } from "react-native";

const App = () => (
  <View style={styles.container}>
    <Text style={styles.title}>React Native</Text>
  </View>
);

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: "#eaeaea"
  },
  title: {
    fontSize: 30,
    fontWeight: "bold"
  }
});

export default App;
```

常见的 Taro 其他端代码设置样式：

```jsx
// app.js
import React from "react";
import { Text, View } from "@tarojs/components";
import "./app.css";

const App = () => (
  <View className="container">
    <Text className="title">React Native</Text>
  </View>
);

export default App;

// app.css
.container {
  flex: 1;
  background-color: "#eaeaea";
}
.title {
  font-size: 30;
  font-weight: "bold";
}
```
通过分析上面的代码，Taro 要适配 React Native 代码有以下及衍生的问题：
1. React Native 代码（样式代码） 与其他平台样式管理的差异性

    1. React Native 样式只支持声明式 style 的写法
    2. 其他端比如小程序端既可以使用声明式 style ，又可以通过 class 读取导入的样式进行布局
2. 通用的样式文件怎么转换为对象
3. 组件样式怎么传递

    React Native 是用 style 进行传递，其他端使用 class。理论上从写法上约束，使三端更好的适配，CSS In JS 样式解决方案更适合。
4. React Native 支持的样式有限，不支持的样式怎么处理
5. 平台型的特殊逻辑以及特殊样式怎么灵活处理

    Taro 在处理跨端文件上有做过处理，但是只是针对脚本文件，我们希望能够对样式文件也能够进行跨端，并且希望在 ios 和 android 两个端的差异文件匹配。比如 Shadow 类样式，只能在 IOS 上生效，Android 则需使用 elevation 替代。当然可以通过重写，但有的时候工程化需求能更明确的区分这两端。

## 总体设计

#### 流程图
样式适配设计流程图：

<img src='https://tva1.sinaimg.cn/large/008eGmZEgy1gn5ncsrc1yj30u011agz4.jpg' style="width:400px;margin:0 auto 20px;display: block" />

下面是流程释义，对应流程图标注。

核心流程：
1. 样式代码处理成对象
    1. CSS 样式解析成对象导出
    2. 引入样式文件处理成 JS 模块
    3. 引入多个样式文件处理
2. 样式校验
3. 标签属性 className 处理成 style

拓展补充流程：
4. 多种预编译语言的适配
5. 更灵活的跨平台工程适配

## 设计实现
#### 样式代码处理成对象
- CSS 样式解析成对象导出

    如果我们需要是样式文件在 React Native 能使用的话，首先需要将样式 CSS 处理成样式对象。在这里第一步将 CSS 解析成 AST 树[1]:
    
    ```css
    body {
        background: #eee;
        color: #888;
    }
    ```
    
    <img src='https://tva1.sinaimg.cn/large/008eGmZEly1gn4k6lt2huj30kw0saq6o.jpg' style="width:200px;margin:0 auto;display: block" />
    
    将解析出来的 AST rules（selector）进行遍历并且再对里面的 declarations（样式属性）遍历，使用 css-to-react-native[2] 将 CSS 属性转换成 React Native 的样式属性，在 selector 命名的对象设置转换后的属性值和属性名。最后用一个大的样式对象存储 selector 命名的对象。
    
    将样式处理成 JS 对象[3]：

    <img src="https://tva1.sinaimg.cn/large/008eGmZEgy1gn5ng51rooj316m0a0q42.jpg" alt="image-20210106105551032" style="width:500px;margin:0 auto;display: block" />
    
    这一步在源码实现上将 css-to-react-native 和 CSS parser 封装成 taro-css-to-react-native NPM 包。
    
    最后将转换后的样式对象导出：
    ```js
    import { StyleSheet } from 'react-native'
    const cssObject = {/**/}
    export default StyleSheet.create(cssObject)
    ```

- 引入样式文件处理成 JS 模块

    Metro 是 Facebook 用来支持 React Native 的打包工具，它 打包有三个阶段，Resolution（模块解析器），Transformation（模块转换），Serialization（模块序列化），直译过来比较容易理解，大概能猜到要用它们做什么事。接下我们需将样式文件进行模块化解析和模块转换，对应的 Metro 配置里面的 resolver 和 transformer 配置项。
    
    resolver 配置处理模块解析，`sourceExts` 用来配置需要处理文件后缀，通过 js 引入进来的模块将会当作 JS 模块处理，这个机制让我们可以将样式文件依赖引入。
    
    接下来把导入进来样式文件，在 transformer 这层将样式内容转成 JS 语法，导出 JS 模块，这样就可以在文件中引入样式对象了。
    
    ```js
    // metro.config.js
    const { getDefaultConfig } = require("metro-config")

    module.exports = (async () => {
      const {
        resolver: { sourceExts }
      } = await getDefaultConfig()
      return {
        resolver: {
          sourceExts: [...sourceExts, "scss", "sass"]
        },
        transformer: {
          // 样式 transformer， 配置忽略了对其他文件的处理，实际上根据不同后缀使用不同的 transformer
          babelTransformerPath: require.resolve("@tarojs/rn-style-transformer")
        },
      }
    })()
    ```
    
    将原来的样式代码字符串转成 JS 的代码字符串：
    ```js
    import { StyleSheet } from 'react-native'
    export default StyleSheet.create(cssObject)
    ```
    然后可以从页面上引入样式模块：
    ```js
    import a from 'a.css'
    const _stylesheet = a
    ```

- 引入多个样式文件处理

    引入一个样式文件时，我们不需要处理，只需将导出的对象赋值 `_stylesheet` 对象，然后在 `style` 属性进行引用。当引入多个样式文件时，要实现整个页面引入使用的样式生效，需要把多个样式合并成一个样式对象：
    ```js
    import a from 'a.css'
    import b from 'b.css'

    const _stylesheet = mergeStyles(a, b)
    ```
    要实现上面的功能，需要对 JS 代码处理，我们在 Babel 编译的时候定义 visitor 访问 importDeclaration 修改 AST 就可以了。
    下面是一段代码演示：
    ```js
    traverse(codeAst, { // visitor
      Program: {
        exit(ast) {
          const lastImportAst = findLastImport(ast.get('body'))
          if (styleList.length) {
            // 插入到最后一个 import 后面
            lastImportAst.insertAfter(template.ast(`
              function mergeStyle() {
                // 省略代码，实现 arguments 对象的合并，返回合并的对象
              }
              const _styleSheet = mergeStyle(${styleList.join(',')})
            `))
          }
        }
      },
      // 访问 importDeclaration ast
      ImportDeclaration(ast) {
        const { source } = path.node
        if (!isStyle(source)) return

        ast.node.specifiers.forEach(node => {
          // 导出的 default
          if (t.isImportDefaultSpecifier(node)) {
            styleList.push(node.local.name)
          }
        })
      },
    })
    ```

#### 标签属性 className 处理成 style
上面讲到的文件导入处理，以及多文件合并都是用 Babel 插件去实现的，将 className 处理成 style 也是通过 AST 修改 jsxElement 的属性。下面是插件实现属性转换的核心逻辑：

- className 是一个普通字符串

    这一步比较好办，直接把 `<View className='red' />` 转化成 `<View style={_styleSheet['red']} />`。

- className 是一个表达式

    className 如果是复杂表达式（比如函数调用等非 JS 基本类型）的话，在 Babel 编译时是无法处理这种结果的，所以得借助运行时方法去处理。这里 我们实现了一个 getStyle 函数，参数传入 className 表达式，在运行时得到表达式的值再去 _styleSheet 里面取出样式对象。代码层将 `<View className={'red'} />` 转化成 `<View className={getStyle('red')} />`。

- className 和 style 属性同时存在 

    同时存在这两种属性时，将两者的值进行合并。React Native 标签 style 属性支持数组对象，所以可以把 `<View className='red' style={{ width: 100 }} /> ` 处理成 `<View style=[{color: 'red'}, { width: 100 }] />` 。 
    
    实现下面核心逻辑的代码演示：
    
    ```js
    JSXOpeningElement(jsxPath) {
      const { attributes } = jsxPath.node
      const styleNode = attributes.find(node => node.name.name === 'style')
      const classNode = attributes.find(node => node.name.name === 'className')
      // 存在 className 属性
      if (classNode) {
        // 删除 className 属性
        attributes.splice(attributes.indexOf(classNode), 1)
        let class2StyleExpression = []
        // className 表达式
        if (t.isJSXExpressionContainer(classNode.value)) {
          const { value } = classNode.value.expression
          if (typeof value.expression.value === 'string') {
            // className={'red black'} => style={[_stylesheet['red'], _stylesheet['black']]}
            class2StyleExpression = value.expression.value === '' ? [] : getMap(value.expression.value) // getMap() 返回数组表达式，自行实现
          }
          // 标记需要在运行时动态计算 className 值
          else {
            // className={expression} => style={getStyle(expression)}
            class2StyleExpression = [t.callExpression(t.identifier('getStyle'), [value.value])]
            // TODO: 标记最后一个 import 后面插入 getStyle 函数，在 Program exit 时处理
          }
        }
        // className 是字符串
        else if (t.isStringLiteral(classNode.value)) {
          class2StyleExpression = classNode.value === '' ? [] : getMap(classNode.value.value) // getMap() 返回数组表达式
        }

        // 存在 style
        if (styleNode && t.isJSXExpressionContainer(styleNode.value)) {
          const { expression } = styleNode.value
          // style={[{ }]} 表达式
          if (expression.type === 'ArrayExpression') {
            expression.elements = class2StyleExpression.concat(expression.elements)
          }
          // style={{ }} 表达式
          else if (expression.type === 'ObjectExpression') {
            styleNode.value = t.jSXExpressionContainer(t.arrayExpression(class2StyleExpression.concat(expression)))
          }
        }

        // 不存在则新增 style 属性
        else {
          attributes.push(t.jSXAttribute(t.jSXIdentifier('style'), t.jSXExpressionContainer(t.arrayExpression(class2StyleExpression))))
        }
      }
    }
    ```
    
    还有其他场景感兴趣的同学请查看 babel-plugin-transform-react-jsx-to-rn-stylesheet 测试用例[4]。
    
    即使实现这种方式的转换，但仍然可能跟 Web 的样式呈现不一样：
    ```css
    /* app.css */
    .black {
        color: red;
    }
    .red {
        color: red;
    }
    ```
    
    ```jsx
    <View className='red black' />
    ```
    
    在 Web 端：
    
    ```jsx
    <div class='red black' />
    ```
    
    而 React Native 端处理成了：

    ```jsx
    <View style={[stylesheet.red, stylesheet.black]} />
    ```
    
    细心的同学可能发现问题了，在 Web 端的，class 跟样式文件的定义样式的顺序有关，跟标签的 class 属性书写顺序无关，而在 React Native，却跟 class（处理成 style）的顺序有关。这个问题，现阶段只能在代码层去规避，还没有想到很好的方法去解决。

以上讲的是如何将一般的 CSS 语法的文件处理成 React Native 标签的 style 属性能接收到的样式对象，而在实际开发生产时，样式预处理器更为推荐使用，毕竟优秀的语法糖能提高劳动人民的生产力。

#### 多种预编译语言的适配
- Sass

    这个版本的做法是基于 node-sass 解析 Sass 语法，内部调用 node-sass 提供的 API render 函数，并且提供一些 options 配置。跟 sass-loader[5] 类似，但只暴露了 options 和 additionalData 两个配置项。
    
    additionalData是一个全局配置，插入一段代码到引入的 Sass 文件中。而 config.sass 也是一个全局配置，他们之间是有关联的。addtionalData 在编译时注入的每个 Sass 文件头部，在 config.sass 配置的全局样式之前，就是说 config.sass 可以把 addtionalData 样式重写掉，或者可以复用 addtionalData 的变量。
    
    sass-loader 配置里面的有一个 implementation 配置，该配置能选择使用 dart-sass 实现 Sass 的解析，但是目前这个版本还没有支持到，所以目前还不能启用 dart-sass 解析。node-sass[6] 依赖 node-gyp 和 python2，官方不建议使用，不会有新特性的更新，取而代之的是使用 dart-sass[7]。这个库完全兼容 node-sass，官方介绍的更易安装，更容易集成到现代 Web 开发工作流程中。所以 Taro React Native 后面版本会支持到 implementation 配置。
    
- Less，Stylus

    像 less-loader 一样，Less 处理需要封装 less.js  提供的 api render 函数，以及封装自定义 options 和 addtionalData 处理。
    
    ```js
    less.render(src, {
      ...options,
      filename,
      plugins: plugins.concat(options.plugins || []),
      paths: paths.concat(options.paths || [])
    }, (err, output) => {
      if (err) {
        return reject(err.message)
      }
      resolve(output.css)
    })
    ```
    
     Stylus 支持也跟 Sass Less 类似，提供 options 和 additionalData 两个配置项，将 Stylus 语法解析成标准的 CSS。

- PostCSS
    
    PostCSS 不是类似上述预处理器，而是一种允许用 JS 插件来转变样式的工具。在源码里面我们对内置默认插件进行了封装，最后使用 postcss 库进行解析。

#### 校验样式
- 校验样式写法

    在设计的过程中，所有的样式文件都经过预处理语言 PostCSS 处理，这样可以把一些公共事务集中处理，比如 stylelint，条件编译，单位处理等等。样式 stylelint 检测是用来校验样式写法，使用 stylelint 插件，通过 stylelint-config-taro-rn[9] 配置 规则来约束不支持的样式写法，比如校验组合选择器。
- 校验样式对象

    React Native 支持的样式有限，写入一些不支持的样式将会导致应用报错或者闪退，所以我们需要一个功能去校验代码，在控制台打印样式错误日志。
    
    当写了一个不支持的样式属性 `o` 时：
    
    <img src='https://tva1.sinaimg.cn/large/008eGmZEly1gn4higfqvej30k60vswhp.jpg' style='width:300px;margin: 0 auto;display: block' />
    
    在控制台给用户提示报错信息：
    
    <img src='https://tva1.sinaimg.cn/large/008eGmZEly1gn4hmacxpej318o054gnh.jpg' style='width:600px;margin: 0 auto;display: block' />

#### 更灵活的跨平台工程适配
- 跨平台文件
    
    在文章开始的是提到过，我们需要更灵活的去匹配不同平台的文件（包括样式文件），并且希望区分 IOS 和 Android 两端的文件。

    实现上面最先想到的是用 Babel 插件去实现，因为 Babel 去修改 import 声非常简单。但是在实现的一两个版本里面，发现他有很严重的副作用，就是缓存问题。因为 Metro 对所有源文件进行了编译缓存，如果没有改动的话将读缓存的内容而不会默认重新编译，而 Babel 本质上是对代码的转化。当新增一个 跨平台优先级高的文件时，文件没有改动，并且没有打破文件依赖关系，所以不会重新编译的问题。但正是这个缓存机制，Metro 才有快速启动的优势。
    
    后面我们决定在 Resolution（模块解析器）在一层 去实现不同跨平台文件引入，将对样式文件和脚本文件都做跨平台处理。
    
    ```js
    function handleFile (context, realModuleName, platform, moduleName) {
        // 返回新的模块名，根据平台，文件路径和引入的模块
        moduleName = resolveExtFile(context, moduleName, platform)
    }
    ```
- 跨平台样式

    在一段样式代码内，希望区分小程序，H5 和 其他平台的场景时， 则需使用条件编译。条件编译是根据编译时的环境变量，对 CSS AST 内容的进行解析，决定对条件内的代码采用或者忽略，使用的是已有 PostCSS 插件 postcss-pxtransform[10] 实现。
    ```css
    /*  #ifndef  %PLATFORM%  */
    /* 平台特有样式 */
    /*  #endif  */
    ```

## 总结

通过使用上面的一些技术手段，在 Taro 开发 React Native 过程中相对而言可以有一个比较好的体验。但在 React Native style 属性作为样式管理限制下，其他端样式的适配上仍有限制。

1. 未来优化的方向和目前仍存在的样式限制 - 选择器的约束
    
    1. 在 JSX 这一层，只对了 className 进行 style 的转换，这个限制造成了在选择器上只能使用类选择器。
    
    2. 在 CSS 转成对象这一层，直接将类选择器转化成对象，没有对组合选择器支持，限制了组合选择器的使用。

    3. 在预处理语言嵌套写法中，也不能使用会编译成组合选择器的写法，但是可以使用能编译成 BEM[11] 的嵌套写法。
    
2. 关于组合选择器的一点思考

    1. Atomic CSS
    
        Atomic CSS[12] 是今年5月 Facebook 提出来的对样式管理的方案，官方号称使用了 Atomic CSS 将主页 CSS 代码量减少了80％，感兴趣的同学可以访问 Facebook 主页查看。
        
        <img src='https://tva1.sinaimg.cn/large/008eGmZEly1gn4mor24dcj30tf0dd78w.jpg' width='600' />
        
        Atomic CSS 是指给每一个样式属性设置都对应一个类选择器去控制，避免重复的样式代码。比如使用了一个公共样式，只想使用其部分样式，则将增加一个层级（后代选择器）去重写不想使用的样式，在一定程度上增加了代码量。如果项目的样式都用 Atomic CSS ，那么组合选择器的支持就显得没那么必要了。
        
    2. CSS in JS
    
        CSS in JS[13] 是使用语法糖，在 JS 里面定义样式，本质上是用内联的方式写样式，这跟 React Native 样式管理理念一致，不失为 CSS in JS 爱好者推崇。但在小程序官方有提到， style 接收动态的样式，在运行时会进行解析，会影响渲染速度。
 
## 参考
[0] https://github.com/NervJS/taro-rfcs/pull/8

[1] https://www.npmjs.com/package/css

[2] https://github.com/styled-components/css-to-react-native

[3] https://csstox.surge.sh/

[4] [https://github.com/NervJS/taro/blob/feat%2Freact-native/packages/babel-plugin-transform-react-jsx-to-rn-stylesheet/__tests__/index.spec.js](https://github.com/NervJS/taro/blob/feat%2Freact-native/packages/babel-plugin-transform-react-jsx-to-rn-stylesheet/__tests__/index.spec.js)

[5] https://github.com/webpack-contrib/sass-loader

[6] https://github.com/sass/node-sass

[7] https://github.com/sass/dart-sass

[8] https://www.npmjs.com/package/postcss-pxtransform

[9] https://www.npmjs.com/package/stylelint-config-taro-rn

[10] https://www.npmjs.com/package/postcss-pxtransform

[11] http://getbem.com/naming/

[12] https://engineering.fb.com/2020/05/08/web/facebook-redesign

[13] https://en.wikipedia.org/wiki/CSS-in-JS
---
tags:
  - eslint
  - 工程化
  - vscode
categories:
  - 前端
title: 记项目中的一次eslint + tsconfig更新
date: 2020-04-12
updated: 2020-04-12
---

项目近期目录刚好有一波调整，顺便优化了一波eslint的配置，让项目在开发的过程中lint提示更加友好（注：编辑器使用vscode 1.44版本）。

下面记录本次优化过程中的调整以及遇到的一些问题。

<!-- more -->
## 目录调整
优化之前的目录
```
- server
- client
  - tsconfig.json
- .eslintrc.js
- app.js
```
app.js是node的启动目录，server是node的一些功能聚合目录，client即前端文件的目录，项目基于react + ts开发的。

所以 .eslintrc.js 放在最外层可以检查整个项目，tsconfig.json放在client目录，期望可以检查client中的ts、tsx文件，但是实际中并没有生效。下文会说。

优化之后的目录
```
- server
  - app.js
  - .eslintrc.js
- client
  - eslintrc.js
  - tsconfig.json
- .eslintrc.js
```
简单的说就是把node端相关的内容全部移到server中去，并且针对不同的端做了一些不同的配置。下面讲讲本次优化过程中的改动。

## 调整项
### 继承
最外层 .eslintrc.js 是一些全局通用的配置。
```js
module.exports = {
  "parserOptions": {
    "ecmaFeatures": {
      // ...
    }
  }

  "rules": {
    // ...
  },

  // 全局变量如果想不报 "no-undef" 错误，就在这里定义。
  "globals": {
    // ...
  }
}
```
client 和 server 中.eslintrc.js 添加一些环境相关的配置。它们继承最外层通用的配置文件，并添加一些单独的配置。

如下server中的配置：
```javascript
module.exports = {
  extends: [
    'alloy',
    '../.eslintrc.js'
  ],

  "env": {
    "node": true
  },
  "parser": "babel-eslint"
}
```
server端node代码仍然是使用js编写的，所以eslint解析器使用babel-eslint。

如下client中的配置：
```javascript
module.exports = {
  "extends": [
    'alloy',
    'alloy/typescript',
    '../.eslintrc.js'
  ],
  "parser": "@typescript-eslint/parser",
  "rules": {
    // ...
    "react/jsx-no-bind": 0,
    // ...
    "@typescript-eslint/interface-name-prefix": ["warn", { "prefixWithI": "always" } ],
  },
  "plugins": [
    "@typescript-eslint",
    "react",
    "react-hooks"
  ],
  "parserOptions": {
    "ecmaVersion": 6,
    "sourceType": "module",
    "ecmaFeatures": {
      "jsx": true
    },
    "project": "./tsconfig.json"
  }
}
```
client中使用react + ts开发，所以使用 @typescript-eslint/parser 作为eslint的解析器，关于两种parser的区别，下文会讲到。
alloy的eslint规则是腾讯开源的，可以参考配置。如果发现alloy的一些配置和自己的项目不符合，可以在rules中禁用相关配置。  
rules添加一些react以及ts相关的rule。plugins是必须要指定的，否则eslint将会报错。sourceType: module + ecmaVersion: 6 表示解析es6除了模块化相关的所有语法。

相关的npm包：
```
eslint-config-alloy
eslint-plugin-react
eslint-plugin-react-hooks
@typescript-eslint/eslint-plugin
```

extend 提供的是 eslint 现有规则的一系列预设，而 plugin 则提供了除预设之外的自定义规则，当在 eslint 的规则里找不到合适的的时候，就可以借用插件来实现了。

### .eslintrc.js 的parser修改
在早期，ts的代码是通过tslint配置的，不过自从2019年1月ts开发团队意识到使用tslint会带来一些麻烦，并且业界已经有了成熟的eslint，就决定不再维护tslint，同时配合eslint来整合tslint那一套。相关文章见[ts在eslint上的未来](https://eslint.org/blog/2019/01/future-typescript-eslint)。

从上文可以看到，server端和client端的eslint的parser是不同的。原因在于server使用js，而client使用ts。  

parser | 优点 | 缺点
:-: | :-: | :-: 
|babel-eslint| 使用babel解析，通过插件可以支持各种语法 | 不支持检查ts文件
|@typescript-eslint/parser| 使用ts编译器解析，完美支持ts | babel通过插件定义的一些语法，ts编译器不能识别

**babel通过插件定义的一些语法，ts编译器不能识别** 这一部分在开发中是不常见的，比如可选链（?.），控制合并运算符（??）这些，在babel 7.0的时候就可以通过插件支持，而ts 3.7版本才内置。那么在babel 7.0 到 ts3.7这个时间端内，ts不能检测上面两种语法。后来在babel7.7的时候，babel-preset-env内置这两种语法。ts3.7同样支持。相关文章见[typescript-eslint
](https://github.com/typescript-eslint/typescript-eslint)。

需要注意的是：即使使用了@typescript-eslint/parser，eslint也不能检查出ts文件中的类型错误，只能检测出语法错误。如下图：
![](https://store-g1.seewo.com/36eec6aa2383417bbfe8e975b0efc372)
控制台中，通过运行eslint命令，检查出了aa这个变量未使用，这是eslint的rule的报错，它并没有报类型错误。将鼠标移到aa上面，之所以能在代码中看到错误提示，是vscode配置typescript检查出来的。如果要检查类型错误，可以通过tsc来检查，--noEmit表示不生成编译后的文件。

## 确定eslint的检查范围

### [包括哪些文件](https://cn.eslint.org/docs/user-guide/command-line-interface#--ext)
有两种方式来指定eslint的检查范围
1. 一种方式是通过 --ext 来确定eslint的检查范围，如：
```
eslint ./lib --ext .js,.jsx,.ts,.tsx
```
表示将检查lib文件夹下的所有js、jsx、ts、tsx文件。

2. 另一种方式通过配置glob语法
```
eslint {js,router,store}/**/*
```
表示将检查js，router，store文件夹下的所有文件。

需要注意的是，当--ext和glob语法一起使用时，将忽略--ext指定的文件。

### [排除哪些文件](https://cn.eslint.org/docs/user-guide/configuring#ignoring-files-and-directories)
通过建立一个 .eslintignore 文件来进一步筛选上述包括的文件。.eslintignore 使用glob语法，定义了哪些文件不需要被检查。比如上面的文件夹中有以 .test.js 为后缀的测试文件，它不需要被检查，那么可以配置：
```
**/*.test.js
```

## [vscode集成](https://github.com/microsoft/vscode-eslint)
vscode中可以针对eslint做一些单独的配置。配置方式是：首选项 -> 设置 -> 输入eslint。它分为对用户或者对当前工作区配置。
![](https://store-g1.seewo.com/42b3df34e26d47b394101b4451c18029)

当前工作区配置相当于在项目根目录下的 .vscode/settings.json 文件配置。
```
{
  "eslint.lintTask.enable": true,
  "typescript.tsdk": "node_modules/typescript/lib",
  "eslint.workingDirectories": [ "./client", "./server" ]
}
```

## 一些问题记录
### 1. vscode中配置了eslint但是却没有生效？
1. 首先，要在vscode的扩展商店下载 EsLint 扩展，这一步应该是没什么问题的
2. 更有可能是vscode运行eslint的时候出错了，这时候可以看下调试控制台
首选项 -> 设置 -> 搜索eslint。确保eslint是开启的
![](https://store-g1.seewo.com/bd98b57db6644ec1bc8b9b4bedaf05f8)

然后如下图查看调试控制台是否有报错
![](https://store-g1.seewo.com/fc1ddc57176a423fae810d85d1531340)

### 2. @types/react-redux 
```
'/node_modules/hoist-non-react-statics' has no exported member 'NonReactStatics'.

47 import { NonReactStatics } from 'hoist-non-react-statics';
```
这个[issue](https://github.com/DefinitelyTyped/DefinitelyTyped/issues/33690)提供了多种解决方案：
1. 降级react-redux版本
2. 添加hoist-non-react-statics依赖
3. tsconfig添加 `skipLibCheck: true`，表示忽略.d.ts文件的类型检查

### 3. ts的lib.dom.d.ts报错，xxx was also declared here
SO上有这个[问题的解决方案](https://stackoverflow.com/questions/57331779/typescript-duplicate-identifier-iteratorresult)，同样，使用 `skipLibCheck: true` 可以解决这个问题。


## 总结
本次eslint的配置过程中，对eslint整个结构和体系有了更深的了解。本文主要记录的是eslint相关的，tsconfig相关的内容较少，当然内容也可能不是很全面，比如eslint的overrides语法，eslint自带的解析器estree等。

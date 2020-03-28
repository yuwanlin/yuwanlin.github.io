---
tags:
  - webpack
  - loader
categories:
  - 前端
title: webpack loader
date:
updated:
---

![](https://store-g1.seewo.com/4b0b2f9cd91748779bf785b581cb17d7)

在 webapck 中，一切皆[模块](https://webpack.docschina.org/concepts/modules)。而 webpack 本质上是一个现代 JavaScript 应用程序的静态模块打包工具。因此，无论什么格式的文件，都需要转换成js可以识别的模块。比如图片文件、字体文件、css文件等，但是这些文件类型直接当做js模块使用是不被js模块系统理解的，所以需要一种方式将它们转换为js模块再使用--这个模块的转换过程由 webpack 的 loader 处理。

<!-- more -->

## loader的用法
loader有三种使用方式，分别是：
- 配置：在webpack配置文件中针对不同类型的文件制定loader，这也是推荐的方式
- 内联：在import语句中指定loader
- CLI：在shell命令中使用

### 配置
在webpack配置文件中可以方便得配置多个loader，看起来简单明了，对业务代码无倾入、统一配置便于维护。

loader从右到左（从下到上）顺序执行，如下部分：先执行sass-load，然后执行css-loader，再执行style-loader。

use 是一个数组，其中的loader可以是一个字符串，也可以是一个对象。
```typescript
module.exports = {
  module: {
    rules: [
      {
        test: /\.scss$/,
        use: [
          'style-loader'
          {
            loader: 'css-loader',
            options: {
              modules: true
            }
          },
          { loader: 'sass-loader' }
        ]
      }
    ]
  }
};
```
由于这种方式可读性好、易修改、易维护、对业务代码无倾入，所以推荐使用这种方式。

在开发自己的loader时，为了便于获取这些参数，webpack提供了 [loader-utils](https://github.com/webpack/loader-utils) 这个包。
```typescript
const loaderUtils = require("loader-utils");
const options = loaderUtils.getOptions(this);
```

### 内联
```typescript
import 'style-loader!css-loader?module!sass-loader./style.scss'
```
使用内联方式，以`!`分隔loader。内联loader的配置可以覆盖webpack中loader的配置。

每个loader可以使用参数，例如`?key=value&key2=value2`，或者一个json对象，如`?{key: value}`。

### CLI
这种方式用的比较少了，我在开发中没有用过。
```typescript
webpack --module-bind jade-loader --module-bind 'css=style-loader!css-loader'
```

## loader是什么
实际上，loader就是一个导出为函数的js模块，不过这个函数需要遵循一定的输入输出。[loader-runner](https://github.com/webpack/loader-runner) 会执行这个函数。如果有多个loader的时候，它们从右至左（从下至上）被执行。最先执行的loader接受资源文件，将处理后的返回结果传递给下一个loader，直至最后一个loader处理。loader接受的的资源类型可能是一个String或者一个Buffer（通过设置raw），最终输出一个字符串或者不输出（pitching loader）。另外还可以传递一个可选的 [SourceMap](https://github.com/mozilla/source-map) 结果（格式为 JSON 对象）。

需要说明的是所有的loader的导出都需要符合 commonjs 规范，因为webpack（loader） 运行在 Node.js 中。书写loader的过程中可以使用es模块编写，最后通过 babel 转换成 commonjs 模块规范即可。

话不多说，先进入代码吧~~~

## 最简单的loader -- raw-loader
raw-loader源码非常简单，它的作用就是将引入的文件当做一个字符串处理。如果引入的文件不是文本格式的，会得到一堆乱码。

```typescript
// 关键代码如下
import { getOptions } from 'loader-utils';
import validateOptions from 'schema-utils';

import schema from './options.json';

export default function rawLoader(source) {
  // 获取使用loader时的option
  const options = getOptions(this) || {};

  // 验证option中字段有效性
  validateOptions(schema, options, {
    name: 'Raw Loader',
    baseDataPath: 'options',
  });

  // source 是命中的资源文件
  const json = JSON.stringify(source)
    .replace(/\u2028/g, '\\u2028')
    .replace(/\u2029/g, '\\u2029');

  const esModule =
    typeof options.esModule !== 'undefined' ? options.esModule : true;

  return `${esModule ? 'export default' : 'module.exports ='} ${json};`;
}
```
[loader-utils](https://github.com/webpack/loader-utils) 是webpack提供的一个针对loader的工具库，其中提供了很多方便的方法用于开发者在编写loader时候使用。getOptions 用于获取webpack中配置loader时候的options。parseQuery 用于解析内联loader的参数。

[schema-utils](https://github.com/webpack/schema-utils) 是对loader-utils的封装，用于验证传参的合法性。

\u2028 是行分隔符，\u2029 是段落分隔符。由于历史原因，这两个unicode码点是可以在json中存在的，但是在js中通过JSON.stringify这两个码点会出现问题。这一点在最近有所改变，这两个码点既可以在json中存在，也可以在js中通过JSON.stringify来处理。所以为了兼容旧的浏览器，需要如上处理。[MDN上有对于此的介绍](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify#Issue_with_plain_JSON.stringify_for_use_as_JavaScript)。

最后将JSON.stringify后的内容导出。默认以es6模块规范导出，因为webpack2开始原生支持es6模块规范。

raw-loader 相当简单，非常适合入门理解。

## loader特性

loader有它的一些特性。
1. loader支持链式传递  
在之前已经介绍过。这类似于洋葱模型。所以多个loader的时候，是有一个进出的过程。这在下面的pitching loader中会提到。

2. loader可以是同步的，也可以是异步的  
如果一个loader是单个处理结果，可以通过同步操作直接返回。如果是多个处理结果，可以通过 `this.callback`（同步模式） 返回，也可以通过 `this.async`（异步模式） 返回。`this.async()` 执行后的返回值是`this.callback`
```typescript
// 同步模式
module.exports = function(content, map, meta) {
  return someSyncOperation(content);
};

module.exports = function(content, map, meta) {
  this.callback(null, someSyncOperation(content), map, meta);
  return; // 当调用 callback() 时总是返回 undefined
};

// 异步模式
module.exports = function(content, map, meta) {
  var callback = this.async();
  someAsyncOperation(content, function(err, result) {
    if (err) return callback(err);
    callback(null, result, map, meta);
  });
};
```
对于计算比较量比较大的过程，推荐使用异步loader。如果返回的内容不是es6或者commonjs导出（如上raw-loader），那么要确保下一个loader可以处理返回的内容。

3. 支持插件  
插件可以为loader提供更多的能力。比如postcss-loader。可以对css文件进行进一步的处理。

下面的配置中，将px布局转为rem布局，然后针对目标浏览器添加兼容性代码。
```typescript
{
  loader: 'postcss-loader',
  options: {
    plugins: [
      postcssAutoprefixer({
        browsers: targetBrowsers
      }),
      px2rem({
        remUnit: 75
      }),
    ],
    sourceMap: false,
  }
}
```

4. loader能够产生额外的任意文件  
比如使用 [file-loader](https://github.com/webpack-contrib/file-loader/blob/master/src/index.js) 处理图片，能够将源图片输出一份到目标文件夹下。

## “Raw” loader 与 url-loader
默认情况下，资源文件会被转换为utf-8字符串，然后传给loader。通过设置raw，loader可以接受原始内容的Buffer。每一个loader都可以用字符串或Buffer传递它的处理结果。
```
module.exports = function(content) {
  
}

module.exports.raw = true;
```
只有一个loader的时候，如果该loader设置raw为true，那么获得的content内容就是一个Buffer。

前面说到 [loader-runner](https://github.com/webpack/loader-runner/blob/master/lib/LoaderRunner.js) 会执行loader，它的源码会对content进行转换。

如果设置了raw并且接受到的content是字符串，会转换为Buffer。如果没有设置raw并且是Buffer，会首先被转换成字符串。
```typescript
function utf8BufferToString(buf) {
  var str = buf.toString("utf-8");
  if(str.charCodeAt(0) === 0xFEFF) {
    return str.substr(1);
  } else {
    return str;
  }
}

function convertArgs(args, raw) {
  if(!raw && Buffer.isBuffer(args[0]))
    args[0] = utf8BufferToString(args[0]);
  else if(raw && typeof args[0] === "string")
    args[0] = Buffer.from(args[0], "utf-8");
}
```
Buffer.from 是 nodejs 的API，可以将一个字符串转换为Buffer。

### url-loader
[url-loader](https://github.com/webpack-contrib/url-loader) 是对file-loader的封装，它的options有一个属性 limit，当输入的内容的字节小于 limit 的大小时，会以base64的形式输出内容。否则，让file-loader处理。

```typescript
if (shouldTransform(options.limit, src.length)) {
 const file = this.resourcePath;
 const mimetype = options.mimetype || mime.contentType(path.extname(file));

 if (typeof src === 'string') {
   // eslint-disable-next-line no-param-reassign
   src = Buffer.from(src);
 }

 const esModule =
   typeof options.esModule !== 'undefined' ? options.esModule : true;

 return `${
   esModule ? 'export default' : 'module.exports ='
 } ${JSON.stringify(
   `data:${mimetype || ''};base64,${src.toString('base64')}`
 )}`;
}
```
url-loader一般用于对资源文件（图片文件/字体文件等）进行处理。源码中的src就是一个Buffer。如果符合条件，就将Buffer转换为base64。


## pitching 
loader按照类型可分为前置(pre)、行内(inline)、普通(normal)、后置(post)。
多个loader执行时类似洋葱模型，符合新进后出的过程。所以就分为两个阶段。进的过程称为pitching阶段，出的过程称为normal阶段，也就是上面提到的从右往左执行。

pitching 阶段：loader 上的 pitch 方法，按照 后置(post)、行内(inline)、普通(normal)、前置(pre) 的顺序调用。  
normal阶段：loader 上的 常规方法，按照 前置(pre)、普通(normal)、行内(inline)、后置(post) 的顺序调用。模块源码的转换，发生在这个阶段。

所有普通 loader 可以通过在请求中加上 `!` 前缀来忽略（覆盖）。  
所有普通和前置 loader 可以通过在请求中加上 `-!` 前缀来忽略（覆盖）。  
所有普通，后置和前置 loader 可以通过在请求中加上 `!!` 前缀来忽略（覆盖）。  
不推荐在业务代码中使用行内 loader 和 `!` 前缀，因为它们是非标准的。不过它们可在由 loader 生成的代码中使用。

这里的覆盖是指覆盖同名的loader。比如对同一个文件的相同的loader配置了多种类型：
```typescript
import lazyDom from 'bundle-loader?name=rd1!react-dom'

{
  test: /react-dom/,
  use: [
    { 
      loader: 'bundle-loader',
      options: {
        name: 'rd2'
      }
    }
  ]
}
```
这时候，内联loader的配置通过 ! 前缀可以覆盖webpack中相同loader的配置。

前置loader
```typescript
{
  test: /\.jsx?$/,
  enforce: 'pre',
  include: [SOURCE_PATH],
  use: [{
    loader: 'eslint-loader',
    options: {
      plugins: [
        'eslint-plugin-react'
      ]
    }
  }]
}
```

对资源文件使用了多个loader情况下，如：

```typescript
use: [
  'a-loader',
  'b-loader',
  'c-loader'
]
```

那么实际上它的执行过程如下：
```typescript
a-loader.pitch
  b-loader.pitch
    c-loader.pitch
      资源文件（在normal阶段被处理）
    c-loader.normal
  b-loader.normal
a-loader.normal
```
这就符合先进后出的原则。如果某个loader在pitching阶段返回了一个不为undefined的值，那么这个过程就会提前终止，跳过该loader后面的loader。比如上述的b-loader在pitching阶段返回了一个值，那么整个执行过程将会变为：
```typescript
a-loader.pitch
  b-loader.pitch return something
a-loader.normal
```

在b-loader.pitch中返回的值将会作为a-loader.normal正常的输入。

所谓的pitching阶段其实就是loader导出的pitch函数。如：
```
module.exports = function loaderb(content, map, meta) {
  console.log('b-loader.normal', this.data); // { value: 'b-loader.pitch' }
  return `export default "b-loader"`
}

module.exports.pitch = function(remainingRequest, precedingRequest, data) {
  console.log('b-loader.pitch', remainingRequest,'---', precedingRequest, '---', data)
  data.value = 'b-loader.pitch';

  // 返回不为undefined的值
  return ''
}
```

所以同一个loader，如果有了pitch函数，那么它的default导出的函数是不会执行的。

上面可以看出，pitch函数的签名了。
- remainingRequest：字符串。剩余loader路径 + 待处理的文件路径
- precedingRequest：字符串。已经处理过的loader的路径
- data：可以在pitch以及normal阶段传递数据

以上述的a-loader、b-loader、c-loader为例。对同一个资源文件进行处理，打印出的结果如下：
```typescript
a-loader.pitch /absolutePath/loader-b.js!/absolutePath/loader-c.js!/absolutePath/src/test.txt ---  --- {}
b-loader.pitch /absolutePath/loader-c.js!/absolutePath/src/test.txt --- /absolutePath/loader-a.js --- {}
c-loader.pitch /absolutePath/test.txt --- /absolutePath/loader-a.js!/absolutePath/loader-b.js --- {}

c-loader.normal { value: 'c-loader.pitch' }
b-loader.normal { value: 'b-loader.pitch' }
a-loader.normal { value: 'a-loader.pitch' }
```


### bundle-loader
bundle-loader 是使用pitch的一个例子，bundle-loader可以按需加载代码和分隔代码块。该loader是 webpack1 时期的产物，它是使用webpack提供的 require.ensure 函数实现的。在 webpack2 及之后的版本，webpack推荐使用符合规范的 dynamic import 来实现按需加载和提取chunk。它的按需加载部分主要代码如下：
```typescript
if(query.lazy) {
  module.exports = function(cb) {
    require.ensure([], function(require) {
      cb(require(loaderUtils.stringifyRequest(this, '!!' + remainingRequest)))
    }, chunkName)
  }
}
```
使用方式如下：

```typescript
import React from 'react';
import LazyReactDom from 'react-dom';

const Counter = () => {
  return (
    <div>counter</div>
  )
}

LazyReactDom(ReactDOM => {
  ReactDOM.render(<Counter />, document.getElementById('root'))
})
```
![](https://store-g1.seewo.com/135b05a2117f4ecea60e44633504b22a)

可以看到，react-dom确实被分割成了一个chunk。在这里的webpack配置中，对react-dom使用了多个配置：
```typescipt
{
  test: /react-dom/,
  use: [
    // 'raw-loader',
    {
      loader: 'bundle-loader',
      options: {
        lazy: true,
        name: 'ReactDoms'
      }
    },
    path.resolve(__dirname, './loader/loader-a.js'),
    path.resolve(__dirname, './loader/loader-b.js'),
    path.resolve(__dirname, './loader/loader-c.js'),
  ]
},
```

打印出 `loaderUtils.stringifyRequest(this, '!!' + remainingRequest` 的值是
`"!!../../loader/loader-a.js!../../loader/loader-b.js!../../loader/loader-c.js!./index.js"`。
所以这就很好的解释了一个问题，尽管由于bundle-loader使用了pitch，导致跳过了abc loader的pitch方法，但是当bundle-loader执行的时候，它的remainingRequest仍然是有abc这三个loader的，它们同样也会被执行。

### style-loader
同样，style-loader也是pitch的一个应用场景。它的作用是插入css到dom中，一般在开发环境使用。可能会这样配置style-loader:
```typescript
{
  test: /\.css$/,
  use: [
    'style-loader',
    'css-loader'
  ]
}
```
style-loader 同样使用了pitch方法。它的执行的时候，调用css-loader，处理好css中的@import和url()后，得到css文件。再根据配置项决定如何将css插入到dom中。

## loader上下文
loader上下文表示在loader内部使用 this 可以访问的一些方法和属性。在上文中，已经使用过 `this.callback`、 `this.async()`、`this.data`、`this.resourcePath`等。

loader上下文包含非常多的信息以便我们操作：

loader相关：this.loaders、this.loaderIndex、 this.query、 this.data、 this.callback、 this.async  
资源文件相关：this.request、this.resource、this.resourcePath、this.resourceQuery  
输出到控制台：this.emitError、this.emitWarning  
操作其他文件：this.loadModule、 this.resolve、 this.addDependency、 this.addContextDependency、 this.clearDependencies、 this.emitFile  
上下文信息：this.rootContext、 this.context  
其他：this.fs、 this.sourceMap、 this.webpack、 this.target、 this.cacheable、 this.version

### this.loadModule
假设项目中有一些公共函数（globalUtil），期望每个js文件中都可以使用这些函数，我们可以在每个js文件中去引入这个util，也可以通过loader来注入。

loader配置如下：
```typescript
{
  test: /\.jsx?$/,
  exclude: /node_modules/,
  use: [
    {
      loader: 'babel-loader',
      options: {
        'cacheDirectory': true
      }
    },
    {
      loader: path.join(__dirname, './src/util-loader')
    }
  ]
}
```

对每个js(x)类型的文件应用util-loader。util-loader.js的内容如下：
```typescript
module.exports = function (content, map, meta) {
  // 需要异步加载文件，所以使用this.async
  const callback = this.async();
  // source是./util的内容
  this.loadModule('./util', function(err, source, sourceMap, module) {
    const utilContent = `const globalUtil = ${source.split('=')[1]}; ${content}`
    callback(null, utilContent, sourceMap, meta)
  })
}
```

再看下./util的内容，注意：这里util没有后缀，因为如果使用util.js，那么这个js文件还会被这个loader处理，从而陷入了循环。util的内容就是一个工具对象：
```typescript
module.exports = {
  add(a, b) {
    return a + b;
  },
  minus(a, b) {
    return a - b;
  }
}
```
自定义的loader中给globalUtil赋值的是`${source.split('=')[1]}`，就是module.exports的内容。

这样的任意的js文件中就可以使用globalUtil了，不需要任何引入。
```typescript
// index.js
const result = globalUtil.add(1, 2);
console.log('result', result) // 3
```

### this.emitFile
这个方法的作用是输出一个文件。`emitFile(name: string, content: Buffer|string, sourceMap: {...})`

file-loader 就使用了这个函数。file-loader的作用是解析 `import/resolve()` 引入的文件路径，根据webpack配置，输出一个相同的文件到输出目录。具体代码就不放了。

## 总结
文章先举例说明了loader的三种用法，然后具体介绍的loader以及它的特性，并通过举出实际的例子来阐述这些特性。最后列出了loader的上下文具有的属性和方法，这些属性和方法可以满足各种各样的loader场景。

loader就是一个约定格式的函数，它通过解析（为所欲为）各种各样格式的文件，最终输出一个任意的string到源代码。loader是webpack这个工具链中不可或缺的一环。

## 参考
1. [看完这篇webpack-loader，不再怕面试官问了](https://juejin.im/post/5e3389436fb9a02fef3a707a#heading-5)  
2. [webpack module](https://webpack.docschina.org/configuration/module/#rule)  
3. [编写一个loader](https://webpack.docschina.org/contribute/writing-a-loader/)  
4. [loader API](https://webpack.docschina.org/api/loaders)
5. [各种webpack loader](https://github.com/webpack-contrib)
6. [postcss loader](https://github.com/postcss/postcss-loader)

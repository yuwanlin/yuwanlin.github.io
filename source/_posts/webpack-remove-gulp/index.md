---
tags:
  - null
categories:
  - 前端
title: sourcemap引起的项目打包方案修改（移除gulp）
date:
updated:
---
## 背景
近期项目中要接入[elastic apm](https://www.elastic.co/guide/en/apm/agent/rum-js/current/intro.html)，kibana作为数据展示平台。
kibana上可以看到请求链路以及耗时，以及上报的错误信息。
<!-- more -->

关于错误信息这块，为了更方便定位生产环境的错误位置，需要上传sourcemap到kibana。

没有sourcemap：    
<img src='https://store-g1.seewo.com/7c0734b1c3f44827a65d5a0bd97cd760' width='500' />

有sourcemap：  
<img src='https://store-g1.seewo.com/6a3a4540ba144dfa8c74d2d0b363c5d5' width='500' />

OK，sourcemap不是本文的重点。这个项目是个老项目了，该项目打包机制用了gulp + webpack。webpack中output如下配置：
```
output: {
  path: path.join(OutputPath, 'statics'),
  filename: '[name].js',
  chunkFilename: 'appchunks/[name].[chunkhash:8].chunk.js',
  publicPath: '/statics/',
  sourceMapFilename : 'sourcemap/app/[file].map'
}
```
简单地说，对于入口chunk，输出的文件并没有加hash值，而对于动态chunk（import()/react-loadable）等，输出的chunk是被hash处理过的。

为什么要这样处理呢？这个项目用的是ejs模板引擎，node中render某个模板文件，模板文件中引入webpack输出的文件。如下：
```
// node controller
res.render('master.html');

// master.html
<!doctype>
<html>

<script src='/statics/main.js'></script>
</html>
```
main.js 是webpack入口chunk打包输出的文件。js打包完成后，gulp将模板文件由source目录移动到modules/views目录。打包完成后目录如下：
```
modules
- views
 - master.html
statics
- main.js
- appchunks
    - [name].xxxxx.chunk.js
    
src
- main.js
- views
    - master.html
```

**开发环境：**  
不对main.js进行hash，master.html中代码不用修改。node中render master.html模板文件不会报错，项目可以正常使用。

**生产环境：**  

1. 对于main.js，gulp进行 [glup-rev](https://github.com/sindresorhus/gulp-rev#readme) 处理，对main.js加上hash。`main.js -> main.xxxhash.js`。

2. 对html文件进行 [gulp-rev-collector](https://github.com/shonny-ua/gulp-rev-collector) 处理，替换html中的静态资源路径。
```
// master.html
<!doctype>
<html>

<script src='/statics/main.xxxhash.js'></script>
</html>
```

由于是老项目，以前都是这么干的，没有什么问题。

因为现在要接入apm，要给文件加上sourcemap。对于动态chunk文件，即appchunks下的文件，生成的chunk.js和chunk.js.map的hash值是对应的。但是对于入口chunk，webpack打包生成的bundle是没有hash的。这时候会生成两个文件，`main.js`和`main.js.map`。然后gulp-rev对这两文件加上hash，这时候问题就来了，它们两文件的hash是不对应的，就会产生这样一个结果：
![](https://store-g1.seewo.com/18bdab39d25f49fc91a1911a75907d89)
```
statics
- main.hash1.js
- sourcemap
    - app
        - appchunks
            - [name].xxxx.chunk.js.map
        - main.hash2.js.map

// main.hash1.js 文件尾部：
//# sourceMappingURL=main.hash2.js.map
```
在本地，如果main.hash1.js出错，通过sourcemap可以找到对应sourcemap文件。到这都是没问题的。

然而sourcemap文件以及对应的js文件需要通过脚本上传到kibana。由于statics下面除了main.js之外还有其他很多的js文件，所以不能将所有js都上传到kibana。

上传脚本执行时，会遍历sourcemap/app中的sourcemap文件，然后在statics中找到同名的js文件。这个时候，由于main.hash1.js的sourcemap文件名是main.hash2.js.map，所以在statics中找不到对应的js文件。

背景就是这样，就是为了解决js文件和对应的sourcemap文件不同名的问题。不同名的原因是gulp对文件hash的，所以如果对入口chunk也通过webpack hash就可以了，这样生成的sourcemap文件的hash和chunk的hash是一样的。

关键就是这个过程，涉及的修改点挺多的。

## 统一hash
由于gulp-rev对js文件及其对应sourcemap文件处理的hash不同，所以使用webpack来对入口chunk进行hash，所以webpack配置修改为：
```js
output: {
  path: path.join(OutputPath, 'statics'),
  filename: '[name].[contenthash:8].js',
  chunkFilename: 'appchunks/app/[name].[contenthash:8].chunk.js',
  publicPath: '/statics/',
  sourceMapFilename : 'sourcemap/app/[file].map'
}
```
修改了filename的值，并且移除gulp中对入口chunk进行hash的逻辑。这样的话，webpack打包后的文件输出：
```
statics
- main.hash1.js
- sourcemap
  - app
    - main.hash1.js
```
入口chunk及其sourcemap的hash值是一致的。

在这种情况下，输出的bundle（即main.hash1.js）被hash了，这时候master.html引用的js资源仍然是main.js（因为没有gulp-rev-collector进行静态资源替换了）。所以需要使用[html-webpack-plugin](https://github.com/jantimon/html-webpack-plugin)来动态将输出的bundle插入到html文件中。

### html-webpack-plugin处理

```js
new HtmlWebpackPlugin({
  template: '!!raw-loader!' + path.join(SourcePath, 'views/master.html'),
  filename: path.join(RootPath, 'modules/views/master.html')
}),
```

master.html使用 [raw-loader](https://github.com/webpack-contrib/raw-loader) 进行处理，因为master.html中有一些ejs变量，这些变量的值是从node中传过来的，所以不希望html-webpack-plugin对ejs语法进行处理。

**在生产环境下**，html-webpack-plugin会在modules/views下生成master.html文件，node在render模板文件的时候，可以找到它，是可以正常工作的。  
**唯一需要注意的一点是：** 因为master.html中本来已经引用了main.js，所以打包后master.html文件会引用两个main.js文件，一个未hash的，一个已经hash的。
```js
// master.html
<!doctype>
<html>

<script src='/statics/main.js'></script>
<script src='/statics/main.xxxhash.js'></script>
</html>
```
由于html文件使用了ejs模板，所以可以设置只在开发环境下渲染未hash的main.js文件。
```
<% if(开发环境) { %>
    <script type="text/javascript" src="/statics/main.js"></script>
  <% } %>
```

**开发环境下（开发环境配置是不对资源进行hash的）**，使用了webpack-dev-server，打包后输出的资源不会输出到文件夹。所以node中render master.html模板文件的时候会找不到，所以需要在开发环境打包的时候将html文件写入到文件夹。通过[html-webpack-harddisk-plugin](https://github.com/jantimon/html-webpack-harddisk-plugin)实现。
```js
plugins: [
  new HtmlWebpackPlugin({
    template: '!!raw-loader!' + path.join(SourcePath, 'views/master.html'),
    filename: path.join(RootPath, 'modules/views/master.html'),
    alwaysWriteToDisk: true
  }),
  new HtmlWebpackHarddiskPlugin()
]
```

针对单个入口的webpack打包，这样处理是OK的。

## 多入口打包
实际上，我们的项目是多入口的。由于历史原因，生产环境针对每个入口都使用一个webpack配置文件打包。而开发环境下，是会对多入口都进行打包的。所以生产环境如上配置没问题。

开发环境下，上文描述的只有main.js一个入口，实际上还有site.js这个入口。webpack入口配置：
```js
entry: {
  main: path.join(__dirname, 'xxx-main.js'),
  site: path.join(__dirname, 'xxx-site.js'),
}
```
如果继续按照上述配置，html-webpack-plugin输出的master.html中会有三个js引用：
```
<script src='/statics/main.js'></script>（master.html自带的）
<script src='/statics/main.js'></script>（html-webpack-plugin追加的）
<script src='/statics/site.js'></script>（html-webpack-plugin追加的）
```
这样当打开master.html对应的页面的时候，出错倒是不会出错，但是会重复加载main.js以及一个多余的site.js文件（其实开发环境也影响不大），只是看起来不是很舒服。所以开发环境如果是多入口打包，使用html-webpack-plugin会在master.html注入多个js文件引用，这样不太合适，自然就不能使用html-webpack-plugin插件了。需要找其他方法。

由于master.html中原本已经引用了main.js，所以想到，在开发环境下，master.html实际上不需要额外处理，只要保证node中render能找到就行。所以，简单的，直接将master.html复制一份到modules/views下即可。同时，由于开发环境下去除了html-webpack-plugin，自然html-webpack-harddisk-plugin也不会生效，需要使用 [copy-webpack-plugin](https://github.com/webpack-contrib/copy-webpack-plugin) 配合 [write-file-webpack-plugin](https://github.com/gajus/write-file-webpack-plugin) 将资源输出到文件夹。
```js
new WriteFilePlugin({
  test: /\.html$/,
  useHashIndex: true
}),
new CopyPlugin([
  {
    from: path.join(SourcePath, 'views/master.html'),
    to: path.join(RootPath, 'modules/views/master.html')
  }
])
```

## 总结
由于历史原因，项目使用gulp + webpack打包。在webpack输出sourcemap的情况下，使用gulp添加hash会导致chunk和chunk对应的sourcemap文件的hash值不一致。所以hash值只能统一使用webpack处理。html-webpack-plugin在生产环境下可以正常工作，但是由于我们项目的开发环境是多入口打包，它会带来其他一些问题。所以针对开发环境和生产环境不同配置，给出不同的html文件处理方案，让最终的处理结果在开发环境和生产环境都可以正常工作。

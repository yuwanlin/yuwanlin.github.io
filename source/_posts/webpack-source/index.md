---
tags:
  - webpack
  - 源码解析
  - 工程化
categories:
  - 前端
title: webpack4源码分析之二 -- 初始化配置
date: 2020-04-10
updated: 2020-04-10
---

![](https://store-g1.seewo.com/4b0b2f9cd91748779bf785b581cb17d7)

## 入口 -- cli.js
webpack-cli/bin/cli.js 是一个自执行函数（IIFE）。
```
// node_modules/webpack-cli/bin/cli.js
(function() {
  const importLocal = require("import-local");
	// Prefer the local installation of webpack-cli
	if (importLocal(__filename)) {
		return;
	}

	require("v8-compile-cache");

  const yargs = require("yargs").usage(`webpack-cli ${
		require("../package.json").version
	}

Usage: webpack-cli [options]
       webpack-cli [options] --entry <entry> --output <output>
       webpack-cli [options] <entries...> --output <output>
       webpack-cli <command> [options]

For more information, see https://webpack.js.org/api/cli/.`);

	require("./config-yargs")(yargs);

  
  yarns.options({
    // ...
  })

  yargs.parse(process.argv.slice(2), (err, argv, output) {
    // ...
  })
})()
```
<!-- more -->

import-local 表示优先使用本地的webpack-cli，v8-compile-cache [用于对编译中间结果持久化，加快整体执行时间](https://github.com/flyyang/blog/issues/13)。

yargs 可以让开发者定义控制台提示内容，以及可能的交互。比如输入 webpack-cli --help 会出现以下内容。
![](https://store-g1.seewo.com/9dcc7eb3b619403b859eec270c42ae98)

所以源码中的 `require("./config-yargs")(yargs)` 以及 `yarns.options` 都是配置控制台的提示信息。直到 `yargs.parse`，它解析命令行的传参，得到一个 argv 对象。

在 yargs.parse 中执行
```
options = require("./convert-argv")(argv);
```
得到webpack.config.js以及传参配置后的options对象。

## 合并cli参数以及webpack配置文件
node_modules/webpack-cli/bin/convert-argv.js 这个文件对webpack的配置文件以及命令行的传参进行一系列处理，得到一个最终的options。

下面是详细内容：
先对 argv 的简写进行处理，比如命令行传入-p，那么代表生产模式，则设置 `argv.mode = "production"`。进而对项目下的webpack配置文件进行遍历，得到配置文件的地址。

![](https://store-g1.seewo.com/aa4d1042bf9d4889be49b905d1dc5c07)

```
// 由于项目中存在webpack.config.js，所以 此处configFileLoaded 为 true
if (!configFileLoaded) {
  return processConfiguredOptions({});
} else if (options.length === 1) {
  return processConfiguredOptions(options[0]);
} else {
  return processConfiguredOptions(options);
}
```

`processprocessConfiguredOptions` 中先对options进行校验，
```
const webpackConfigurationValidationErrors = validateSchema(
	webpackConfigurationSchema,
	options
);
```
如果验证出错的话，会在控制台打印错误并退出执行。否则会对options进行一系列的判断（是否是promise、数组，对象），最终得到一个对象类型的options，执行：

```
processOptions(options);
```
`processOptions`中对options中的参数和 argv（命令行传参得到的对象）进行合并处理，相同配置argv中的优先级更高。并且根据args配置entry，output等，根据参数配置插件（--hot, --debug）等等。

## 配置Stats
在 node_modules/webpack-cli/bin/convert-argv.js 文件中对argv和配置文件处理得到一个合并后的options，然后返回 node_modules/webpack-cli/bin/cli.js 文件中执行同名的`processOptions`函数：
```
// 接上文
options = require("./convert-argv")(argv);

processOptions(options)
```
`processOptions`中首先根据命令行配置以及用户配置文件中的stats字段设置 [Stats](https://webpack.js.org/configuration/stats/) 信息，它是webpack编译过程中的输出到控制台统计信息。

然后进入webpack的编译环节，回调函数 `compilerCallback` 中使用了Stats信息。
```
compiler = webpack(options);

if (firstOptions.watch || options.watch) {
	// ...
  compiler.watch(watchOptions, compilerCallback);
  if (outputOptions.infoVerbosity !== "none")
	console.log("\nwebpack is watching the files…\n");
} else compiler.run(compilerCallback);
```

## 总结
本节先写到这里，总结一下：
1. 从 webpack-cli/bin/cli.js 文件开始，通过yargs这个包来配置命令行的一些提示性内容。如webpack-cli --help；
2. 合并命令行参数以及用户配置文件，得到一个全新的options；
3. 根据命令行命令行配置以及用户配置文件中的stats字段配置Stats信息，用于控制后续的编译过程中将哪些信息打印到控制台；

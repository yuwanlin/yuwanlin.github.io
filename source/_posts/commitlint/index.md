---
tags:
  - commitlint
  - 代码规范
categories:
  - 前端
title: 使用commitlint规范代码提交
date:
updated:
---


## 前言
在多人协作的项目开发过程中，如果commit message没有统一规范，会让他人不容易理解，而且自己在以后回顾的时候可能也会忘掉当时做了什么。

[commitlint](https://github.com/conventional-changelog/commitlint) 配合 [husky](https://github.com/typicode/husky) 可以在`git commit`的时候检查commit message，不符合规范的提交会被拒绝。
<!-- more -->

husky本质上是使用git的hooks，可以在git commit、push、update等[前后阶段](https://github.com/typicode/husky/blob/master/src/upgrader/index.ts)运行一些命令。

commitlint是在git `commit-msg` hook阶段对commit message进行检查。
```
// package.json
"husky": {
  "hooks": {
    "commit-msg": "commitlint -E HUSKY_GIT_PARAMS"
  }
}
```

## 代码提交
在github很多开源项目上，可以看到commit message都是类似如下形式：
```
type(scope?): subject
```
有三个字段，type表示本次提交的类型，scope表示本次提交的作用范围（它是可选的，也可以有多个），subject就是具体的提交信息，如：
```
chore: run tests on travis ci
fix(server): send cors headers
feat(blog): add comment section
```

这些格式都是统一的，为了防止其他的误提交，可以通过commitlint强制进行检查。根据项目的不同，可以自定义配置我们的代码提交信息规范。

### 安装依赖
```
yarn add @commitlint/cli -D
或者
npm install --save-dev @commitlint/cli
```

### 配置commitlint
1. 在项目根目录husky.rc或者package.json中添加下列代码：
```
{
  "husky": {
    "hooks": {
      "commit-msg": "commitlint -E HUSKY_GIT_PARAMS"
    }
  }
}
```
`-E` 参数在commitlint内部表示环境值。


2. 项目根目录建立`commitlint.config.js`文件，内容如下：
```
module.exports = {
  extends: ['@commitlint/config-conventional']
};
```
`@commitlint/config-conventional`（使用前先安装） 是官方的一个配置，配置文件中使用extends继承，这和eslint的extends是一样的。

### 检测配置
在项目根目录，运行:
```
echo 'fix(server): send cors headers' | npx commitlint
```
会发现没有任何内容输出，表示没有报错。将`fix`修改为`fi`，出现报错:
![](https://store-g1.seewo.com/b749c0c2cde74c2a8395ce40afb4ed51)
说明配置是生效的。

`npx` 表示只在本次命令中安装commitlint，如果不想每次都安装，可以全局安装commitlint，然后：
```
echo 'fix(server): send cors headers' | commitlint
```

## 配置自己的commitlint
github上有一些开源的包用于配置commitlint。但是我们自己的项目会有自己的规范，这时候需要定制commitlint。
```
// commitlint.config.js
module.exports = {
  extends: ['./commitlint/index.js']
};
```

`./commitlint/index.js`内容如下：
```
module.exports = {
  parserPreset: './parser-preset.js',
  rules: {
    'header-max-length': [2, 'always', 144],
    'body-leading-blank': [1, 'always'],
    'footer-leading-blank': [1, 'always'],

    'type-case': [2, 'always', 'lower-case'],
    'type-empty': [2, 'never'],
    'type-enum': [
      2,
      'always',
      [
        'add',
        'delete',
        'modify',
        'bugfix',
        'optimize'
      ]
    ],
    'scope-empty': [2, 'never'],
    'scope-case': [2, 'always', 'lower-case'],

    'subject-case': [
      2,
      'never',
      ['sentence-case', 'start-case', 'pascal-case', 'upper-case']
    ],
    'subject-empty': [2, 'never'],
    'subject-full-stop': [2, 'never', '.'],
  }
};
```
这里对type限定了只能是`add`、`delete`、`modify`、`bugfix`、`optimize`。以及配置了一些其他的规则，所有规则列表见[Rules](https://github.com/conventional-changelog/commitlint/blob/master/docs/reference-rules.md)。

`parser-preset`内容如下：
```
module.exports = {
  parserOpts: {
    headerPattern: /^\[(v\d*\.\d*\.\d*)\]\[(\w*)\](.*)$/,
    headerCorrespondence: ['scope', 'type', 'subject']
  }
};
```
这里我们限定了我们自己的scope、type和subject。正则部分有那个括号，分别对应scope、type、subject。所以我们的commit message是类似于这样的：
```
[v2.0.12][modify]这里是subject

// 测试一下
echo '[v2.0.12][modify]hello' | commitlint
```
scope比如以v开头，后面是数字版本号。

如果这个commit message规范需要在多个项目中用到，则可以封装成一个npm包，方便在其他项目中使用。

![](https://store-g1.seewo.com/ea995b70a3bd4ab197aa0008357dbfe2)

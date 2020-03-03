---
title: ts 条件类型，协变与逆变
date: 2020-03-02
tags:
- typescript
categories:
- 前端
---

## 前言
[Typescript 2.8](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-8.html) 引入了条件类型。条件类型类似于js语法中的条件表达式，使用 `condition ? A : B` 语法，condition成立返回A类型，否则返回B类型。

## 条件类型
```typescript
T extends U ? X : Y
```

上面的类型表示：若T能够赋值给U<sup>[1]<sup>，那么类型是X，否则为Y。




## 备注：
[1] [类型兼容性]()

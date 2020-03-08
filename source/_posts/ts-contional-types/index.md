---
title: ts 条件类型、infer、协变与逆变
date: 2020-03-08
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

上面的类型表示：若T能够赋值给U<sup>[1]</sup>，那么类型是X，否则为Y。如：

```typescript
type A<T, U> = T extends U ? number : string;

type Fun1 = () => void;
type Fun2 = (a: string) => void;

type A1 = A<Fun1, Fun2> // number
type A2 = A<Fun2, Fun1> // string


type Obj1 = {};
type Obj2 = { name: string }

type A3 = A<Obj1, Obj2>; // string;
type A4 = A<Obj2, Obj1>; // number
```

<!-- more -->

在 [类型兼容性](https://yuwanlin.github.io/2020/03/02/ts-type-compatibility-and-guard/index/) 文中说到函数的兼容性，所以这里 Fun1 是可以赋值给 Fun2 的，反之就不行。但是对象这里可能有点疑惑，可以这么理解，Obj1 是父类，Obj2 是子类，Obj2继承Obj1，并添加了自己的属性。

可以看到，条件类型和js中的条件表达式也是很相似的，它也支持条件类型的嵌套。

```typescript
type Box<T> = 
  T extends number ? number :
  T extends string ? string :
  T extends boolean ? boolean :
  undefined;

type A = Box<number>;
type B = Box<string>;
type C = Box<[]>
```

## 分布式有条件类型
条件类型中，如果待检查的类型是裸类型参数(naked type parameter)，那么它也被称为分布式有条件类型。

所谓裸类型参数就是指待检查的类型没有被包装在其他类型中，如数组，元祖，函数，Promise等。

上述例子中，待检查的类型是 T，T 就是裸类型。如下，不是裸类型参数：

```typescript
type A<T, U> = [T] extends [U] ? number : string;
```

那么为什么要弄个分布式有条件类型出来呢？根据官方文档解释，分布式有条件类型在实例化时会自动分发成联合类型。 例如，实例化 `T extends U ? X : Y`，T 的类型为`A | B | C`，会被解析为`(A extends U ? X : Y) | (B extends U ? X : Y) | (C extends U ? X : Y)`。

根据这个特性，可以设计很多类型，如下的 Diff 和 Filter：

```typescript
type Diff<T, U> = T extends U ? never : T;
type Filter<T, U> = T extends U ? T : never;
type NonNullabled<T> = Diff<T, null | undefined>;

type animal = 'chicken' | 'duck' | 'lion' | 'elephant' | 'tiger';
type food = 'chicken' | 'duck';

type DiffResult = Diff<animal, food>; // "lion" | "elephant" | "tiger"
type FilterResult = Filter<animal, food>; // "chicken" | "duck"
type Valued = NonNullabled<number | string | undefined> // string | number
```

上面类似中，animal 和 food 都是字面量联合类型，并且 Diff 和 Filter 都是分布式有条件类型，在实例化（使用）的时候会自动分发成联合类型，就好像对 animal 中每一个字面量都进行了遍历一样，

## infer
ts 中有一个关键字infer，它表示待推断的类型。比如要获得一个函数的返回值的类型，但是在条件类型实例化之前我们不知道开发者传入的的函数的返回类型是什么，借助 infer 关键字可以很好的解决这个问题。

```typescript
type ReturnType2<T> = T extends (...args: any[]) => infer U ? U : any;

type Result = ReturnType2<() => number> // number
type Result2 = ReturnType2<() => string> // string
```

在 ts 的 lib.es5.d.ts 声明文件中，声明了很多类型。使用 infer 的有：
```typescript
// 获得函数的各个参数类型，以元祖表示
type Parameters<T extends (...args: any) => any> = T extends (...args: infer P) => any ? P : never;

// 获得类的构造函数的参数类型，以元祖表示
type ConstructorParameters<T extends new (...args: any) => any> = T extends new (...args: infer P) => any ? P : never;

// 获得函数的返回值的类型
type ReturnType<T extends (...args: any) => any> = T extends (...args: any) => infer R ? R : any;

// 获得构造函数的返回值的类型，也就是类的实例的类型
type InstanceType<T extends new (...args: any) => any> = T extends new (...args: any) => infer R ? R : any;
```

可以看到，上述这些使用 infer 的场景都使用到了条件类型，条件类型结合 infer 是ts中最强大的武器。

## 协变与逆变
在谈论协变与逆变之前，先看如下例子：

```typescript
type Foo<T> = T extends { a: () => infer U, b: () => infer U } ? U : undefined;
type R1 = Foo<{a: () => string, b: () => number}> // string | number


type Bar<T> = T extends { a: (x: infer U) => any, b: (x: infer U) => any } ? U : undefined;
type R2 = Bar<{a: (x: string) => void, b: (x: number) => any}> // never 
```

`Foo<T>` 的例子从直观看起来就是这样，而 `Bar<T>` 的例子可能会让人疑惑，这里的 never 实际上是 string & number。也就是说，第一种情况下，U 的多个候选类型在实例化的时候被推算为联合类型；第二种情况下，U 的多个候选类型被推断为交叉类型。

根据官方的解释：
> 在协变位置上，同一个类型变量的多个候选类型会被推断为联合类型
> 
> 在逆变位置上，同一个类型变量的多个候选类型会被推断为交叉类型

所以从上面的例子，至少我们可以得出结论，函数返回值是协变的，函数参数是逆变的。实际上，函数的返回值，对象的值都是协变的。

那么到底什么是协变和逆变？凭我的水平很难解释清楚，[这里有一篇非常好的文章](https://jkchao.github.io/typescript-book-chinese/tips/covarianceAndContravariance.html)。

## 条件类型与infer的应用
有个经典的案例，将联合类型转为交叉类型，可以概括本文内容。如：number | string -> number & string

```typescript
type UnionToIntersection<U> = (U extends any ? (k: U) => void : never) extends ((k: infer I) => void) ? I : undefined;

type Result = UnionToIntersection<string | number>;
```

Result 是 string & number，即 never 类型，讲讲为什么会这样。

上面的条件类型要分为两部分来看，第一部分是`(U extends any ? (k: U) => void : never)`。这里的 U 是裸类型参数。前面说过，对于裸类型参数，分布式有条件类型会在实例化的时候被分发为条件类型。所以传入`string | number`，在这一部分会被分发为 `((k: string) => void) | ((k: number) => void)`。

然后将第一部分作为一个整体。作为整体来看，它的待推断类型是`(U extends any ? (k: U) => void : never)`, 不是裸类型参数。待推断的类型 I 处于函数参数的位置，它是逆变的。根据上述定义，在逆变位置上，同一个类型变量的多个候选类型会被推断为交叉类型。这里的候选类型是string和number，推断为交叉类型后就是never。

## 备注：
[1] [类型兼容性](https://yuwanlin.github.io/2020/03/02/ts-type-compatibility-and-guard/index/)

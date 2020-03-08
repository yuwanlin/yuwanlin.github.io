---
tags:
  - typescript
categories:
  - 前端
title: ts 类型兼容性 与 类型守卫
date: 2020-03-02
updated: 2020-03-03
---

## 前言
typescript 基于[结构类型（鸭子类型）](https://zh.wikipedia.org/wiki/%E9%B8%AD%E5%AD%90%E7%B1%BB%E5%9E%8B)，所以它的类型兼容性和java、c++这种基于继承的类型兼容性是不一样的。其基本规则是，如果 x 要兼容 y，那么 y 至少具有与 x 相同的属性。举个例子：

```typescript
interface Animal {
  name: string;
  age: number;
}

class Person  {
  constructor(public name: string, public age: number, public address: string;) {

  }
}

let x: Animal;

x = new Person('real', 20, 'somewhere');
```
上面的代码可以正常运行，因为 Animal 兼容 Person。反过来，Person 不兼容 Animal，因为 Animal 缺少 address 属性。顺便说一下，class的构造函数的参数如果有public声明，那么这些参数相当于class的属性声明，在使用的时候ts会自动提示。

<!-- more -->

## 函数参数兼容性
比较两个函数能否兼容，主要看两点。

1. 比较它们参数的数量以及类型。在参数的位置和类型上（和参数名无关），如果 y 函数能够完全对应 x 函数，那么可以将 x 赋值给 y，反之不行
2. 在满足 1 的条件下，函数的返回值要相同

看下面例子：

```typescript
let x = (a: number) => `${a}`;
let y = (aa: number, bb: string) => `${aa}${bb}`;

y = x; // ok
x = y; // error
```

可能对于上述赋值有点疑惑，为什么可以省略参数，像 `y = x`，就省略了bb参数。原因是忽略额外的参数值在js中是很常见的。比如数组的很多方法，第一个函数都是callback，callback可以接受三个参数。
```typescript
Array<T>.forEach(callbackfn: (value: T, index: number, array: T[]) => void, thisArg?: any): void

[1, 2, 3].forEach((value: number, index: number, array: number[]) => { // do something })
// 省略参数
[1, 2, 3].forEach((value: number) => { // do something }) 
```

如果是 rest 参数又会怎样呢？
```typescript
let foo = (x: number, y: number) => {};
let bar = (x?: number, y?: number) => {};
let baz = (...args: number[]) => {};

foo = bar = baz;
baz = bar = foo;
```
默认情况下，这是允许的。但是如果在 tsconfig.json 中配置了 `strictNullChecks: true`，就会报如下错误。因为 bar 函数中 x 和 y 都可能是 undefined，开启 strictNullChecks 规则后，undefined 和 null 只能赋值给它们自己和any，一个例外是 undefined 可以赋值给 void。

![](https://store-g1.seewo.com/afcd4605c88a470593369f8835701894)

## 枚举兼容性
数字和枚举类型之间相互兼容，而不同枚举类型之间不兼容。
```typescript
enum Status { Ready, Waiting };
enum Color { Red, Blue, Green };

let status = Status.Ready;
status = Color.Green;  // error
```
因为第4行中，status的类型被ts推断为 Status。

## 泛型兼容性
因为ts是结构性的类型系统，类型参数只影响使用其做为类型一部分的结果类型。如下例子：

```typescript
interface Person<T> {
  
}

let x: Person<string>
let y: Person<number>
y = x;
x = y;
```

上面相互赋值是允许的。因为从定义上来说，Person泛型的结果类型是一个{}，并没有使用到泛型参数。所以可以相互赋值。所以下面的例子的例子很容易理解：

```typescript
interface Person<T> {
  name: T
}

let x: Person<string>
let y: Person<number>
y = x; // error 不能将类型“Person<string>”分配给类型“Person<number>”
```

## 对象类型兼容性
这里的对象类型是泛指，包括类，对象，interface，type这些定义后的类型可以使用 `.` 操作符的，形如对象。如：
```typescript
interface Person {
  name: string
}

type Person2 = {
  name: string
}

class Person3 {
  constructor(public name: string) {
    this.name = name;
  }
}

class Person4 {
  constructor(public name: string) {
    this.name = name;
  }
}

const person5 = {
  name: ''
}

let p: Person = person5;
let p2: Person2 = person5;
let p3: Person3 = person5;
let p4: Person4 = person5;
let p5: typeof person5 = person5;

p = p2 = p3 = p4 = p5;
```
上面的赋值操作也是允许的，归根到底，还是因为ts是基于结构类型的。只要结构相同，就可以相互赋值。

## 类型守卫
根据上述所讲，如果结构一样可以任意赋值，那怎么区分相同结构的类型呢？所以需要使用到ts中的类型守卫。

ts中有3种类型守卫，分别是 `instanceof`、`in` 、字面量类型守卫。它们都是通过js的语法。如：

```typescript
class Person {
  name = 'aaa';
  age: number = 20;
}

class Animal {
  name = 'petty';
  color: string = 'pink';
}

function getSometing(arg: Person | Animal) {
  // instanceof
  if (arg instanceof Person) {
    console.log(arg.color); // error
    console.log(arg.name); // ok
  }
  if (arg instanceof Animal) {
    console.log(arg.age); // error
    console.log(arg.name); // ok
  }

  // in
  if ('age' in arg) {
    console.log(arg.color); // error
    console.log(arg.age); // ok
  }
  if ('color' in arg) {
    console.log(arg.age); // error
    console.log(arg.color); // ok
  }
}
```

通过 `instanceof` 和 `in` 操作符，就可以确定arg的类型。

还有一种就是 字面量类型守卫。什么是字面量类型？如 `'abc'`，`123`，`false` 等。你可以认为它们是值，但它们也可以表示为类型。

```typescript
let a: 'abc' = 'abc';
a = 'd'; // error
```
上述代码表示 a 的类型是字面量类型，它的类型是'abc'，所以它的值只能是'abc'。

对于字面量类型守卫，redux 中的 action 是一个极好的例子。

```typescript
interface BaseAction {
  payload: object;
}

interface AddAction extends BaseAction {
  type: 'add',
  payload: {
    id: string;
    value: number
  }
}

interface DeleteAction extends BaseAction {
  type: 'delete',
  payload: {
    id: string;
  }
}

const reducer = (state = {}, action: AddAction | DeleteAction) => {
  switch(action.type) {
    case 'add':
        action.payload.id // ok
        action.payload.value // ok
    case 'delete':
        action.payload.id // ok
        action.payload.value // error
  }
}
```

## 总结
本文阐述了ts的类型兼容性以及类型守卫，在开发过程中可以通过这些特征来避免一些问题。

---
title: OSCamp D1-Rustlings
date: 2024-04-09 04:57:27
tags: [Rust, OSCamp]
category: [trace]
---

今天的课程因为官网炸了导致无法进行，所以我就按照自己的进度来了。

# rustlings
仓库地址是：[https://github.com/LearningOS/rust-rustlings-2024-spring-Qiu233](https://github.com/LearningOS/rust-rustlings-2024-spring-Qiu233)

因为以前没怎么写过Rust，而且算法是我的薄弱项，所以多花了点时间。关于题目没什么好说的，下面说说我对Rust的整体理解。

## 类型系统
Rust的类型系统大致可以分成几个方面：
* System F，也就是parametric polymorphism
* trait，一种ad-hoc polymorphism，另外Rust没有函数重载
* Associated Type
* Const Generics
* Ownership，类似于Linear Types
* Borrow和Lifetime Parameter

Rust的trait可以看作具有强制类型参数`Self`的typeclass，例如：

```Rust
trait A {
    ...
}
```

本质上和Haskell的写法

```Haskell
class A Self where
    ...
```

差不太多，不过Rust的`Self`应该是用Associated Type实现的，所以从这个角度来说Rust的trait其实更像C#的abstract static interface，并且因为没有高阶类型，所以类型参数更像是泛型系统的一部分。

Rust的Associated Type和Haskell的Associated Type虽然同名，但Rust的Associated Type并不是Type Family，作用上来说只是一类特殊的类型参数，防止一个trait因为类型参数不同被多次实现。

Const Generics允许将编译期常量当作类型参数，不过只有少数几个类型(primitive types)可以，如果非要在类型论中找个解释，大概是把这些类型promote到trait，然后把它们的编译期确定的值promote到type，类似Haskell的DataKinds。

Rust没有kind，没有高阶类型，因此没有Monad。可预见的未来大概也不会有高阶类型了，因为就算有也很难和现在默认具有Self类型参数的trait兼容。

但是令人惊讶的是Rust的问号?有专属语法和还有Try和Error的trait支持，还不算太难用。

## 所有权

Rust最有趣的部分莫过于Ownership和Borrow，目前我的实践还不够，只能大概谈一下。

首先是所有权，即

* 每个值都有所有权
* 一个值的所有权最多被一个变量持有
* 所有权可以在变量之间移动，或通过函数的参数和返回值移动
* 如果所有权没有被移动，那么当前的scope必须把它drop掉

类比到Linear Types，那么scope所有值都被使用刚好一次，即
* 要不然被转移
* 要不然被销毁

一个特殊情况是实现Copy trait的类型，这种类型的值(即stack-only的值)的所有权不会被转移而是被复制，它们不需要drop。

所有权可以被暂时借走(borrow)，这种机制叫做引用，引用是类型，分为不可变和可变两种。

不可变引用可以有多个，且可以不限次数地复制，例如：

```Rust
let a = 2;
let b: &i32 = &a;
let c: &i32 = b;
test(b);
test(c);
```

其中`test`函数定义为`fn test(v: &i32) {}`。第三行只是把变量b复制给c，因此b并不会失去所有权，第四行就能把b传递给函数。

但是可变引用最多只有一个，下面这段代码会报错：

```Rust
let mut a = 2;
let b: &mut i32 = &mut a;
let c: &mut i32 = b;
test2(b);
test2(c);
```
其中`test2`函数定义为`fn test2(v: &mut i32) {}`。第四行会报错，因为变量b持有的所有权已经被转移给c。

下面这段代码能通过编译：

```Rust
let mut a = 2;
test2(&mut a);
test2(&mut a);
```

因为第一个可变引用已经使用完并被销毁，第二个可变引用才得以创建。

下面这段代码又会报错：

```Rust
fn test2(v: &mut i32) {}

fn wrap<'a>(v: &'a mut i32) -> &'a mut i32 {
    v
}

fn main() {
    let mut a = 2;
    let b = wrap(&mut a);
    let c = wrap(&mut a);
    test2(b);
    test2(c);
}
```

因为main第二行b延续了a的可变引用的生命周期，导致第三行无法再借出可变引用。

之后其他思考在其他文章里补充吧。
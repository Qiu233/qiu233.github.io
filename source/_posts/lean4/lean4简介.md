---
title: lean4简介
date: 2024-02-17 01:47:43
tags: [Lean4, 编程语言]
---
lean4是一个proof assistant兼函数式语言，我不打算大段大段地照搬官方文档，这篇文章只作为简介，我会挑几个个人认为很重要的点来说。

# Lean4的类型层次
语法`a : A`表示a的类型是A，例如`0 : Nat`表示0的类型是Nat。

Lean4的类型的层次结构是`Sort 0 : Sort 1 : Sort 2 : ...`，其中`Sort 0`的别称是`Prop`，`Sort 1`的别称是`Type`或者`Type 0`，而之后的`Sort 2`是`Type 1`，以此类推。

上面`Sort n`中的自然数n表示不同的universe，"普通的"类型比如Nat就落在`Sort 1`也就是`Type`当中，而`Prop`作为最低的Sort，有两个重要性质：  
1. `Prop : Type`意味着Prop和Nat等"普通"类型一样可以在同一个universe中讨论
2. 相比于"普通"类型，Prop仍然是一个Sort，意味着在其universe下可以构造新类型，例如下面的代码
```Haskell
inductive Or (a b : Prop) : Prop where
| inl (h : a) : Or a b
| inr (h : b) : Or a b
```
是lean4标准库中对析取连词的定义。**注意这里是Or : Prop : Type，Or并不是一个Type，千万不要混淆！**

# 自然推演系统
自然推演系统(Natural Deduction)是一个证明演算(proof calculus)系统，是lean4的基础，同样采用自然推演系统的还有Coq。简单来说有三种形式化的推理规则：

## Formation Rules
Formation Rules用于构造有效的term，在proof assistant中复杂一些，由语法结构和类型、运算等定义直接规定。

以$\neg$为例，在lean4中由两部分组成：
1. 符号定义：`notation:max "¬" p:40 => Not p`
2. 实际类型定义，此处是Not，代码如下`def Not (a : Prop) : Prop := a → False`

再以加法运算为例，符号定义是``macro_rules | `($x + $y)   => `(binop% HAdd.hAdd $x $y)``，hAdd的定义则是
```Haskell
class HAdd (α : Type u) (β : Type v) (γ : outParam (Type w)) where
  hAdd : α → β → γ
```

*lean4对于一般的term的formation rules没有统一的语法来描述。*

## Introduction Rules
Introduction Rules用于构造类型，除了∀类型(Π type)的之外全都可认为是constructor，例如∃类型(Σ类型)的定义如下
```Haskell
inductive Exists {α : Sort u} (p : α → Prop) : Prop where
| intro (w : α) (h : p w) : Exists p
```
*Σ类型也叫dependent pair，具体来说是这里的w和h构成的pair，因为h的类型依赖于w的值故得名dependent pair，具体请查阅dependent type相关资料。*

## Elimination Rules
与Introduction Rules刚好相反，Elimination Rules用于"拆解"类型，或者说指出了类型要如何被"消费"，在lean4中通常是定理的形式，还是以∃类型为例
```Haskell
theorem Exists.elim {α : Sort u} {p : α → Prop} {b : Prop}
   (h₁ : Exists (fun x => p x)) (h₂ : ∀ (a : α), p a → b) : b :=
  match h₁ with
  | intro a h => h₂ a h
```

## Propositions as Types
Curry–Howard correspondence是机器证明领域最重要的内容之一，具体到lean4则是其中的直觉主义自然推演系统(Intuitionistic Natural deduction)和类型化λ演算(typed lambda calculus)之间的对应关系。

总结起来就是一句话：**每一个逻辑命题都对应到程序中的一个类型**。基于此，请务必理解下面几个点：
* 对命题的证明=对应类型的值的构造。构造完成即证明结束。
* 命题的可证明与否=对应类型是否有值。
* 引入公理=假设对应类型的值存在。公理用于证明=假设存在的值参与构造目标值。
* 前文所述的Introduction Rules和Elimination Rules也是从逻辑规则到程序的对应关系。

*这一小节的内容有点抽象，因此我安排了下一小节作为重要例子。*

## 蕴含连词和Π Type
lean4中箭头符号`→`有三种重要的解释：
1. 从程序的角度来说，形如`X → Y`表示从类型`X`映射到类型`Y`的函数类型。  
2. 从逻辑的角度来说，假如`X`和`Y`是`Prop`，`X → Y`中的符号`→`可以解释为蕴含连词。  
3. 从类型的角度来说，假设`P`是一个`X → Prop`，那么类型`∀ x : X, P x`是函数类型`(x : X) → P x`的别名。*这里出现的类型在dependent type理论中叫做Π type或者pi-type或dependent function。*

实际上3就是∀量词的本质——即Π类型，这也是∀类型没有语言中定义的Introduction/Elimination Rules的原因——符号`→`的解释是语言内置的。  
既已清楚3可以归结到1，重要的是1和2要如何统一起来。答案十分简单，根据上一小节的内容，若`X`和`Y`都是`Prop`：  
假设类型为`f : (X → Y)`的函数存在；假设命题`X`可证，即值`x : X`存在，那么`f x : Y`即是命题`Y`的证明，因此命题`X → Y`得证。也就是说在1的基础上2的解释成立。

至此，∀量词和蕴含连词都归结到函数类型`X → Y`上。函数类型是lean4中最基础最原始的类型，它的Introduction rules和Elimination Rules分别是

$${\displaystyle {\cfrac {\begin{matrix}{\cfrac {}{x : X}}\\\vdots \\ y : Y\end{matrix}}{f : X\rightarrow Y}}\ \qquad {\cfrac {f : X\rightarrow Y\quad x : X}{y : Y}}}$$

*施工中*
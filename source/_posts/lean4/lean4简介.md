---
title: lean4简介
date: 2024-02-17 01:47:43
tags: [lean4, 语言]
---
lean4是一个proof assistant兼函数式语言，我不打算大段大段地照搬官方文档，这篇文章只作为简介，我会挑几个个人认为很重要的点来说。

# Lean4的类型层次
语法`a : A`表示a的类型是A，例如`0 : Nat`表示0的类型是Nat。

Lean4的类型的层次结构是`Sort 0 : Sort 1 : Sort 2 : ...`，其中`Sort 0`的别称是`Prop`，`Sort 1`的别称是`Type`或者`Type 0`，而之后的`Sort 2`是`Type 1`，以此类推。

上面`Sort n`中的自然数n表示不同的universe，"普通的"类型比如Nat就落在`Sort 1`也就是`Type`当中，而`Prop`作为最低的Sort，有两个重要性质：  
1. `Prop : Type`意味着Prop和Nat等"普通"类型一样可以在同一个universe中讨论
2. 相比于"普通"类型，Prop仍然是一个Sort，意味着在其universe下可以构造新类型，例如下面的代码
```lean
inductive Or (a b : Prop) : Prop where
| inl (h : a) : Or a b
| inr (h : b) : Or a b
```
是lean4标准库中对析取连词的定义。**注意这里是Or : Prop : Type，Or并不是一个Type，千万不要混淆！**

# 自然推演系统
自然推演系统(Natural Deduction)是一个证明演算(proof calculus)系统，是lean4的基础，同样采用自然推演系统的还有Coq。简单来说有三种形式化的推理规则：

## 1.Formation Rules
Formation Rules用于构造有效的term，在proof assistant中复杂一些，由语法结构和类型、运算等定义直接规定。

以$\neg$为例，在lean4中由两部分组成：
1. 符号定义：`notation:max "¬" p:40 => Not p`
2. 实际类型定义，此处是Not，代码如下`def Not (a : Prop) : Prop := a → False`

再以加法运算为例，符号定义是``macro_rules | `($x + $y)   => `(binop% HAdd.hAdd $x $y)``，hAdd的定义则是
```lean
class HAdd (α : Type u) (β : Type v) (γ : outParam (Type w)) where
  hAdd : α → β → γ
```

*lean4对于一般的term的formation rules没有统一的语法来描述。*

## 2.Introduction Rules
Introduction Rules用于构造类型，除了∀类型的之外全都可认为是constructor，例如∃类型的定义如下
```lean
inductive Exists {α : Sort u} (p : α → Prop) : Prop where
| intro (w : α) (h : p w) : Exists p
```

## 3.Elimination Rules
与Introduction Rules刚好相反，Elimination Rules用于"拆解"类型，或者说指出了类型要如何被"使用"，在lean4中通常是定理的形式，还是以∃类型为例
```lean
theorem Exists.elim {α : Sort u} {p : α → Prop} {b : Prop}
   (h₁ : Exists (fun x => p x)) (h₂ : ∀ (a : α), p a → b) : b :=
  match h₁ with
  | intro a h => h₂ a h
```

## ∀类型和函数类型
∀类型和函数类型在某种程度上是等价的

# Propositions as Types
待补充
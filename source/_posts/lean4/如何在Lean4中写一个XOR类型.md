---
title: 如何在Lean4中写一个XOR类型
date: 2024-02-23 03:46:51
tags: [Lean4]
---
逻辑连词XOR(异或)(exclusive or)，写作

$$A\oplus B$$

真值表是
| $A$    | $B$    | $A\oplus B$ |
| :----: | :----: | :---------: |
| $\top$ | $\top$ | $\bot$      |
| $\top$ | $\bot$ | $\top$      |
| $\bot$ | $\top$ | $\top$      |
| $\bot$ | $\bot$ | $\bot$      |

XOR的性质：
* 输入不同则为真，相同则为假
* $A\oplus B$为真则意味着$A$和$B$刚好一真一假
* 交换律：$A\oplus B = B\oplus A$
* 结合律：$(A\oplus B)\oplus C = A\oplus (B\oplus C)$

# 定义
根据真值表，$A\oplus B$为真的情况一共有两种，仿照连词$\lor$在Lean4标准库中的定义(例1)：

```Haskell
inductive Or (a b : Prop) : Prop where
| inl (h : a) : Or a b
| inr (h : b) : Or a b
```
可以很轻松写出XOR的定义(例2)：

```Haskell
inductive XOR (A B : Prop) : Prop where
| inl (a : A) (nb : ¬ B) : XOR A B
| inr (b : B) (na : ¬ A) : XOR A B
```
*命名XOR是因为Xor已经被标准库用来表示`^`运算了。*

此外为了方便使用，定义运算符$\oplus$：
```Haskell
notation:max lhs:max " ⊕ " rhs:max => XOR lhs rhs
```

*这里的`max`是优先级。不用`infixr`来定义运算符是因为$\oplus$已经被标准库的`Sum`用了，只能用`notation`来覆盖。*

## intro&elim
这里再扯C-H同构会浪费篇幅(前置知识请查资料或阅读我的其他文章)，直接给出XOR的introduction rules：

$$
\cfrac{A \quad \neg B}{A\oplus B} {\ \oplus _{I1}} \qquad \cfrac{B \quad \neg A}{A\oplus B} {\ \oplus _{I2}}
$$

然后是elimination rules：

$$
\cfrac{A\oplus B \quad A}{\neg B} {\ \oplus _{E1}} \qquad \cfrac{A\oplus B \quad B}{\neg A} {\ \oplus _{E2}}
$$

例2的两个constructor就分别对应了这里的两条intro rules，而elim rules则由下面两条定理来表达(例3)：

```Haskell
protected theorem XOR.elim_l {A B : Prop} : A ⊕ B → A → ¬ B := by
  rintro (⟨_, nb⟩ | ⟨_, na⟩)
  . intro _; exact nb
  . intro a _; apply na; exact a

protected theorem XOR.elim_r {A B : Prop} : A ⊕ B → B → ¬ A := by
  rintro (⟨_, nb⟩ | ⟨_, na⟩)
  . intro b _; apply nb; exact b
  . intro _; exact na
```

*formation rules实际上就是例2第一行的`XOR (A B : Prop) : Prop`。或者也可以等价地写成`XOR : (A : Prop) → (B : Prop) → Prop`。这在数学上的表达是：*

$$
\cfrac{A:\text{Prop} \quad B:\text{Prop}}{A\oplus B:\text{Prop}} {\ \oplus _{F}}
$$


# 性质
这一节的代码需要`import Mathlib`

## 非自反性
```Haskell
theorem XOR.irrefl {A : Prop} : (A ⊕ A) → False := by
  intro h
  cases h <;> contradiction
```

## 否定
```Haskell
protected lemma XOR.not_xor_of_not_not {A B : Prop} : ¬ A → ¬ B → ¬ (A ⊕ B) := by
  intro hnA hnB h
  cases h <;> contradiction

protected lemma XOR.not_xor_of_pos_pos {A B : Prop} : A → B → ¬ (A ⊕ B) := by
  intro hnA hnB h
  cases h <;> contradiction

theorem XOR.not {A B : Prop} [Decidable A] [Decidable B] :
    (¬ (A ⊕ B)) = (A ↔ B) := by
  apply propext
  constructor
  . intro h
    constructor
    . intro a; rw [← @Decidable.not_not B]; intro nb
      apply h
      exact XOR.inl a nb
    . intro b; rw [← @Decidable.not_not A]; intro na
      apply h
      exact XOR.inr b na
  . intro h
    rw [propext h]
    apply XOR.irrefl
```

## 交换律
```Haskell
theorem XOR.comm {A B : Prop} : (A ⊕ B) = (B ⊕ A) := by
  apply propext
  constructor
  . rintro (⟨a, nb⟩ | ⟨b, na⟩)
    . exact XOR.inr a nb
    . exact XOR.inl b na
  . rintro (⟨b, na⟩ | ⟨a, nb⟩)
    . exact XOR.inr b na
    . exact XOR.inl a nb
```

## 结合律
```Haskell
theorem XOR.assoc {A B C : Prop} [Decidable A] [Decidable B] [Decidable C] :
    ((A ⊕ B) ⊕ C) = (A ⊕ (B ⊕ C)) := by
  apply propext
  constructor
  . rintro (⟨⟨a, nb⟩ | ⟨b, na⟩, nc⟩ | ⟨c, nab⟩)
    . exact XOR.inl a (by apply XOR.not_xor_of_not_not nb nc)
    . apply XOR.inr
      . exact XOR.inl b nc
      . assumption
    . have := XOR.not.mp nab
      if a : A then
        exact XOR.inl (a) (by
          apply XOR.not.mpr; constructor
          . intro _; exact c
          . intro _; exact this.mp a)
      else
        apply XOR.inr
        . exact XOR.inr c (this.not.mp a)
        . exact a
  . rintro (⟨a, nbc⟩ | ⟨⟨b, nc⟩ | ⟨c, nb⟩, na⟩)
    . have := XOR.not.mp nbc
      if b : B then
        apply XOR.inr (this.mp b)
        rw [XOR.not]
        constructor <;> (intro _ ; assumption)
      else
        apply XOR.inl (XOR.inl a b) (this.not.mp b)
    . exact XOR.inl (XOR.inr b na) nc
    . apply XOR.inr c
      rw [XOR.not]
      constructor <;> (intro _; contradiction)
```

## Decidable
```Haskell
instance {A B : Prop} [Decidable A] [Decidable B] : Decidable (A ⊕ B) :=
  if a : A then
    if b : B then
      isFalse (by rw [XOR.not]; constructor <;> (intro _; assumption))
    else
      isTrue (XOR.inl a b)
  else
    if b : B then
      isTrue (XOR.inr b a)
    else
      isFalse (by rw [XOR.not]; constructor <;> (intro _; contradiction))
```
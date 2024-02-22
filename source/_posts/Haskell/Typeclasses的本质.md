---
title: Typeclasses的本质
date: 2024-02-21 21:23:57
tags: [Haskell, Typeclasses, FP, Lean4]
---
# Typeclass
如果读者熟悉Coq或Lean4，就应该非常清楚所谓typeclass本质上就是一个普通的高阶类型，除了语言内置了一套规则来简化class的使用以外没有什么特别的。  
关于什么是高阶类型，请自行查阅资料或者翻一下我的其他文档(~~如果有记得写的话~~)。

下面以Functor为例来说明：

例1.
```Haskell
class FunctorClass (f :: * -> *) where
    fmapC :: (a -> b) -> f a -> f b
instance FunctorClass Maybe where
  fmapC = \m fa -> case fa of Nothing -> Nothing
                              Just a -> Just (m a)
```
和下面的代码是几乎等价的：

例2.
```Haskell
data FunctorData (f :: * -> *) = FunctorData
  { fmapD :: forall a b. (a -> b) -> f a -> f b
  }
instFunctorMaybe :: FunctorData Maybe
instFunctorMaybe = FunctorData
  { fmapD = \m fa -> case fa of Nothing -> Nothing
                                Just a -> Just (m a)
  }
```

例2中`fmapD`是一个field selector，类型签名是：
```Haskell
fmapD :: forall a b f. FunctorData f -> (a -> b) -> f a -> f b
```
而例1的`fmapC`类型签名是
```Haskell
fmapC :: forall a b f. FunctorClass f => (a -> b) -> f a -> f b
```
看到区别了吗？两者的区别仅仅是`->`和`=>`，例1的`FunctorClass f =>`实际上叫做类型约束(type constraint)，在使用fmapC时我们不用自己传入FunctorClass的实例，因为这是编译器自动完成的。

给定一个类型，确定它是某个typeclass的实例的过程，叫做instance inference，这个过程的本质就是寻找类似例2中`instFunctorMaybe`的定义。  
Haskell中程序员不能直接干涉这个机制，在Lean4中所谓类型约束`Class t =>`只是加上中括号的参数`[Class t] ->`，这种参数默认由编译器填充，但程序员可以用前缀`@`来手动填充参数。

# 和Java的class相比
Haskell的class存在继承关系，例如说Monad就是Applicative的子类型，你可能很自然想到class的继承可以直接由data的继承关系得到，但很可惜Haskell的data并不支持继承。

另外，Java中也有class和继承的概念，且Haskell的class的值和它们一样叫**实例(instance)**，你可能会想把它们和Haskell统一到一个框架内。这里给出回答：两者完全不是一回事。
* Java的class是OOP的概念，含义接近于"表示一类具体事物或概念"，可以实例化并且在代码中直接操作，继承是对这一"事物"或"概念"的进一步具体化。
* 而Haskell的class描述的是性质，顾名思义是对类型的分类(type class)，它的实例化不由可执行代码完成，class的实例也不能由程序员直接操作，仅在类型层面发挥作用。

Lean4相比Haskell则更进一步，不仅structure支持继承，而且class的实例有对程序员可见的符号(inst开头的符号)。实际上Lean4的class其实仅是一种特殊的`inductive`和`structure`罢了，分别由下面两个例子说明：

例3.归纳定义的class
```Haskell
class inductive Nonempty (α : Sort u) : Prop where
  | intro (val : α) : Nonempty α
```

例4.structure class
```Haskell
class HAdd (α : Type u) (β : Type v) (γ : outParam (Type w)) where
  hAdd : α → β → γ
```

**例4可以被继承，而例3这种归纳定义的class无法被继承，报错'XX' is not a structure**。这一事实直接说明了继承在Lean4中是structure独有的机制，例4只是一种特殊的structure。
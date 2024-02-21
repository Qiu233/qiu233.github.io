---
title: Monad是什么
date: 2024-02-18 01:58:40
tags: [math, Haskell, Monad, FP]
---
~~Monad是自函子范畴上的幺半群~~。

这篇文章不扯太高大上的名词，力求通俗易懂地解释Monad在编程中是什么，由于语言组织能力欠佳，我将会从几个不同方面来分别解释。本文使用Haskell作示例，同时假设读者有C#的基础。

## Unit类型
这一小节是前置知识，因为十分简单所以放在开头。

`Unit`类型十分类似C#的`void`，区别只在于C#并没有`void`这个类型，~~`System.Void`存在只是因为反射需要~~。`Unit`类型在Haskell中写作一对括号`()`，它仅有唯一的取值也写作`()`。**注意不要混淆！记号`()`既可以表示`Unit`类型也可以表示它的唯一取值，但这是两个不同的事物，至于表示哪个是编译器根据上下文来决定的。**

在Haskell中`()`类型也用来表达"无返回值"的概念，例如标准库函数
```Haskell
putStrLn :: String -> IO ()
```
的类型表示它接受一个`String`类型的参数，并且"无返回值"。

## Statement和Expression
这一小节会需要一点点编译相关的知识。以C#为例，Stmt是完整的可执行的语句，或以分号`;`结尾，或是类似`if`、`while`等复杂的语句，下面是一些例子：
```C#
t = 1; // 赋值
Console.Write("Hello World"); // 函数调用
while(true) Console.Write(" "); // while循环
```
Expr指的是具有类型的表达式，例如`1`，类型是int；例如`a + b`，类型和`a`与`b`的类型有关；例如函数调用`f(a,b)`，类型是函数`f`的返回值类型。  
Expr可以粗略地认为是一个值，但Expr通常不是完整的语句。*例外：函数调用也是一个Expr，但加上分号时成为一个Stmt。*

如果你理解了以上内容，这里引入Monad的第一个作用：**统一Stmt和Expr**。也就是对任意`Monad M`，让Stmt成为类型为`M ()`的Expr。于是我们就只需要Expr这一个概念来描述可执行的代码。至于这么做的原因，请往后看。

*note: 说Monad是统一Stmt和Expr其实因果关系反了。Stmt本质上就是一类Expr，只是很多语言用特殊的设计——例如上面说的C#的`void`——简化了程序在相关逻辑上的表达。*

## 代码的执行顺序
如果读者对Haskell等函数式语言不了解，那么可能会认为每行代码按照出现的先后顺序来执行是天经地义的，例如说如下的C#代码(例3.1)：
```C#
int a = f(1);
int b = g(2);
return a + b;
```
显然函数f先被调用并将返回值存入a，然后是g被调用并将返回值存入b，最后计算a+b并返回。

但考虑如下Haskell代码(例3.2)：
```Haskell
f :: Int -> Int --省略定义
g :: Int -> Int --省略定义
func :: Int
func = let a = f 1
           b = g 2
       in a + b
```
当中f和g实际情况下哪个先被执行是不确定的，*实际上也是无关紧要的，但要完全说明原因还必须考虑纯函数的不变性，暂且defer到第5节*。

实际上"有序性"不是天经地义的，是一个第二性的概念，例如说在数学中我们是先有公理化的(无序的)集合，然后到定义出tuple为止才真正出现有序性。

Haskell的let-bindings和where-clauses本身都是无序的，假如要在Haskell中编写类似上面C#那种顺序确定的代码，这里引入Monad的第二个作用：**定义代码的执行顺序**。例如如下代码(例3.3)：
```Haskell
f :: Int -> IO Int --省略定义
g :: Int -> IO Int --省略定义
func :: IO Int
func =
    (f 1) >>= \a ->
    (g 2) >>= \b ->
    pure (a + b)
```
就会先求值`f 1`，代入运算符`>>=`右侧的变量`a`，再求值`g 2`，代入变量`b`，最后返回`a + b`的值。

也可以重写成更简洁的形式(例3.4)：
```Haskell
f :: Int -> IO Int --省略定义
g :: Int -> IO Int --省略定义
func :: IO Int
func = do
    a <- f 1
    b <- g 2
    pure (a + b)
```
实际上这里利用do-notation简化了例3.3的代码，可以看作利用了语法糖。

*note: 如果你对这里的`do`、`<-`、`IO`、`>>=`和`pure`感到一头雾水，别担心，后文会详细地说明。*

*note: `\x y -> f x y`是Haskell中lambda表达式的语法，这里的例子是具有两个参数x和y的函数。*

## 纯性
函数的纯性(purity)指的是同时满足以下两个条件
1. 函数不会产生任何外部的(全局的)状态变化，也就是没有任何副作用。
2. 引用透明性，简单来说是对相同输入的返回值相同，更准确的说法是**不可分者同一性原理**：值相等的表达式总是可以相互替换。

以C#为例，不纯的函数的一个例子是`Console.WriteLine`，不纯的原因是该函数会导致标准输出缓冲区发生变化，进而可能导致输出设备(例如说屏幕)发生变化，也就是会产生副作用。

请理解并牢记这句话：**Haskell的所有函数都是纯函数**。  
你可能会质疑：啊？你不是刚说完`Console.WriteLine`是不纯的吗？那Haskell的`putStrLn`怎么会是纯函数呢？  
别急，`putStrLn`还真是纯函数，它的类型是`String -> IO ()`，上文说它"无返回值"其实是错误的，它会把相同的`String`值映射到同一个`IO ()`的值上(因此引用透明)，只有当这个`IO ()`真正被执行时才会产生副作用，所以该函数是纯的。  
举例说明，`putStrLn "a"`和`putStrLn "b"`分别会得到一个不同的`IO ()`，但你无论怎样调用`putStrLn "a"`得到的`IO ()`都是同一个值。

你可能又会质疑：你说的"真正被执行"是什么意思？要怎样执行一个`IO ()`？  
答案十分简单：Haskell的入口点`main`函数的类型就是`IO ()`，程序中所有副作用的实际执行都归结到`main`函数的执行，而`main`的执行是编译器和运行时完成的，你只需要把一个`IO ()`交给`main`即可。例如下面的代码(例4.1)：
```Haskell
main :: IO ()
main = putStrLn "Hello World"
```
执行后会输出"Hello World"，注意这里左右侧的类型是一致的。

到这里仍然有一个关键问题没解决：要怎么按顺序执行多个`IO ()`？这个问题直指Monad的核心，第5节回答这个问题。

## IO
`IO`也是一个Monad，表示一个IO操作，前文已经提过`()`是unit type，这里`IO ()`表示一个"无返回值"的IO操作，例如`putStrLn`就是`String -> IO ()`，表示接受一个String参数将它输出，且无返回值。

要把多个`IO ()`连接起来，例如说先输出"Hello"然后输出"World"，可以用如下代码(例5.1)：
```Haskell
main :: IO ()
main = putStrLn "Hello" >>= \x -> putStrLn "World"
```
注意这里`putStrLn "Hello"`和`putStrLn "World"`的类型都是`IO ()`，且`main`的类型也是`IO ()`。

实际上运算符`>>=`的类型是`IO () -> (() -> IO ()) -> IO ()`，总共可以分成三部分：
1. 第一个参数类型`IO ()`，也就是上文的`putStrLn "Hello"`，我们已经知道`()`表示它没有返回值，但是`()`仍然是一个类型，叫做unit type。
2. 第二个参数类型`() -> IO ()`，这是一个函数类型，表示接受一个类型为`()`的参数(也就是1的返回值)，映射到一个`IO ()`。也就是上文的`\x -> putStrLn "World"`。
3. 返回一个`IO ()`，表示以上两个`putStrLn`前后相连得到的新的`IO ()`。

可以看到因为`putStrLn`"无返回值"，所以这里直接将变量`x`忽略了，这种情况有一个更简洁的写法，利用`>>`运算符(例5.2)：
```Haskell
main :: IO ()
main = putStrLn "Hello" >> putStrLn "World"
```
更进一步，如果我们利用do-notation，还有更简洁的写法(例5.3)：
```Haskell
main :: IO ()
main = do
    putStrLn "Hello"
    putStrLn "World"
```
以上三段代码是**完全等价**的。

---

以上是前一个IO没有返回值的情况，如果有返回值呢？例如说`IO Int`就表示返回值是`Int`，下面以例3.3来说明，首先我们先把f和g的定义补充完整(例5.4)：
```Haskell
f :: Int -> IO Int
f x = pure (x * 2)

g :: Int -> IO Int
g x = pure (x + 1)

func :: IO Int
func =
    (f 1) >>= \a ->
    (g 2) >>= \b ->
    pure (a + b)
```
首先解释类型，`f`和`g`的类型都是`Int -> IO Int`，表示接受一个`Int`类型的参数，然后返回一个`Int`。`pure`函数在这里的作用是将`Int`转换为`IO Int`。

然后看`func`函数，类型是`IO Int`，表示返回一个`Int`。最后一行的`pure (a + b)`，前面说了`pure`是用来转换的，所以显然这里的`a + b`是一个`Int`，不难推知`a`和`b`都是`Int`。

那么一个问题来了，变量`a`和`b`都是什么，从哪来的？回顾例5.1的解释，假设`>>=`左侧的类型是`IO X`，那么右侧函数的参数的类型应当是对应的`X`。上面`(f 1)`和`(g 2)`的类型都是`IO Int`，所以`a`和`b`的类型都是`Int`，分别表示这个过程中得到的值。

最后为了严谨起见，将例5.4的代码中的`func`函数加上表示优先级的括号(例5.5)：
```Haskell
func :: IO Int
func =
    (f 1) >>= (\a ->
    (g 2) >>= (\b ->
    pure (a + b)))
```
我们一层一层地解释。首先`func`是`IO Int`，因此右边整体是`IO Int`，`f 1`的类型是`IO Int`，不难推知第一个`>>=`的右边整块`(\a -> (g 2) >>= (\b -> pure (a + b)))`的类型是`Int -> IO Int`。  
然后去掉参数`a`，得到`(g 2) >>= (\b -> pure (a + b))`的类型是`IO Int`，由于`g 2`的类型也是`IO Int`，可得同理`(\b -> pure (a + b)`的类型也是`Int -> IO Int`。  
再去掉参数`b`，得到`pure (a + b)`，类型是`IO Int`。

---

再谈一下例3.4的do-notation，以下是改写后的代码(例5.6)：
```Haskell
func :: IO Int
func = do
    a <- f 1
    b <- g 2
    pure (a + b)
```
符号`do`表示进入一个do-block，在do-block中可以用符号`<-`在语义上表示执行右侧的表达式，例如上面的`f 1 :: IO Int`执行后的值"赋值"到`a : Int`中。上面的代码只是利用do-notation的代码简化，实际上编译时就会被转化为和例5.5完全一致的代码。

## Monad
上一节详细介绍了IO这个Monad，但除了IO之外还有很多Monad，例如Maybe，例如List。这一节我们先不管每个不同Monad的细节，而暂时关注抽象的Monad。

注意！当我们提到*Monad*一词时，实际上是在讨论一系列具有类似性质的(高阶)类型，而不是某个具体的类型。当然Monad本身在Haskell里是一个class。

### Monad都是高阶类型
Haskell中的`Int`、`String`等等完整的类型叫做普通类型(proper types)，写作`Type`或`*`，高阶类型指的是类似`Maybe`、`IO`这种需要类型参数来构成完整普通类型的类型。  
例如上文我们提到了`IO Int`是一个类型，实际上说的是它是一个proper type，但是单独一个`IO`却不是，它的阶签名是`Type -> Type`，高阶类型可以更复杂，例如`Either`是`Type -> Type -> Type`。

**所有Monad都是一个参数的高阶类型`Type -> Type`。**

*施工中*
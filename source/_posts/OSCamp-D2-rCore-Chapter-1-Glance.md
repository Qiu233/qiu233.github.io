---
title: OSCamp D2-rCore Chapter 1-Glance
date: 2024-04-10 02:54:10
tags: [Rust, OSCamp, RISC-V]
category: [trace]
---

今天开始正式学习OS，主要是rCore第一章的内容。

_注，由于我一开始搞错了，这篇和练习篇都是对应V3的第一章内容，V3是2022年及之前的材料，从第二章开始我才回到2024S的材料。_

# RISCV 的启动
RISC-V的启动分成三个阶段：
* 第一阶段：机器加电，CPU状态初始化，pc寄存器设置为0x1000，执行很短的固定指令序列后，跳转至`0x80000000`，进入第二阶段
* 第二阶段：即bootloader，起始地址为固定的`0x80000000`，第二阶段决定第三阶段如何被执行，教程里的sbi-rt默认会跳转至`0x80200000`，进入第三阶段
* 第三阶段：即我们实现的内核(kernel)

第一阶段完全固定，是写死在硬件里的，如果用qemu之类的模拟器则是写死在代码里的，无法更改。

第二阶段的bootloader在qemu启动命令中用`-bios`参数传递，实际上它的功能也和x86语境下的bios极其相似，即提供一系列最基础的功能，例如屏幕输出和关机等。

根据查到的资料，bootloader一般保存在ROM或者Flash这种非易变(non-volatile)介质中，但通过特殊手段(烧写)也能更改。rCore教程中的sbi-rt是预编译好的.bin文件，教程的关注点也不是bootloader，所以更深入的知识得等以后慢慢学了。

一般来说，bootloader要自己决定如何加载内核，可以类比到x86的固定读取结尾为`0x55AA`的第一个扇区到`0x7C00`，但教程的sbi-rt并没有从储存介质加载而是直接跳转到`0x80200000`，这是因为启动命令的参数已经让qemu**启动前**就把内核加载到固定地址了，即
```Makefile
-device loader,file=$(KERNEL_BIN),addr=0x80200000
```

我猜测这样做只是为了方便。

## SBI

第三阶段开始正式执行内核代码，但是还要用到第二阶段bootloader提供的功能，可以类比到x86实模式下用`int`指令触发中断来调用bios功能，RISC-V用`ecall`指令来调用，关于这条指令，spec中是这么说的：

> The ECALL instruction is used to make a request to the supporting execution environment, which is
 usually an operating system.

有两种情况：
* U-Mode的ECALL实际上是syscall
* S-Mode的ECALL是对SEE的调用

注意到上面原文中的execution environment一词，SEE(Supervisor Execution Environment)指的就是bootloader提供给(S-Mode)内核的一套基础功能，与这套功能交互的接口叫做SBI(Supervisor Binary Interface)，教程中的sbi-rt就是一个SBI的实现。

要注意的是x86进入保护模式后所有bios中断程序都不能再使用，但是RISC-V仍然能从S-Mode用ecall调用M-Mode的功能。

## 二进制格式
section的作用分别如下：
* .text：可执行的代码
* .data：编译期初始化的数据
* .bss：未初始化的数据
* .rodata：只读数据

其中三个数据段在RISC-V的ABI规范中都有对应的small版本，即：
* .sdata
* .sbss
* .srodata

用来存放小数据，提高效率。

值得注意的是linker script中把`.eh_frame`段丢弃了，根据查到的信息，这个段主要是有异常的语言用来保存有关stack unwinding的信息，stack unwinding指的是在值在goes out of scope的时候被销毁(Rust中的drop)。

所有元数据，包括段的名称、访问权限等信息，被objcopy --strip-all -O binary指令全部裁去，仅剩下纯二进制数据以供qemu加载。

## M、S、U模式
RISC-V有三种最基本的模式：
* Machine Mode，可类比x86的实模式，RISC-V语境下的bootloader或SEE可以类比到x86的bios，使用的地址都是物理地址
* Supervisor Mode，可类比x86的protected mode，使用虚拟地址，操作系统(内核)就在这个模式下运行
* User Mode，用户态进程的模式

第一章似乎还没提进入S-Mode的内容，所以暂时就不写了。

## 小结
进展比我预想的顺利，相比于x86，RISC-V的架构清晰得多，而且教程已经把sbi实现好了，只用寥寥数行代码就能进入Rust world，甚至到目前为止都没有接触过多少RISC-V的实际指令。

下一步就是动手实践和做练习题了。

# 关于Rust
补充上一篇文章。

Rust的Algebraic Data Types：

* Empty，即感叹号`!`
* Unit，即`()`
* 积类型，即tuple和`struct`
* 和类型，即discriminated union，也就是Rust的`enum`

这里不得不提一嘴，Rust的Emtpy是编译器开的后门，程序员不能自己定义。这样的写法
```Rust
struct Unit;
```

居然是Rust的Unit，而大部分函数式编程语言或者proof assistant中这种没有constructor的定义都是Empty，就你搞特殊是吧？

至于GADT，因为Rust完全没有高阶类型和阶(kind)的概念，所以GADT也基本不用想了。


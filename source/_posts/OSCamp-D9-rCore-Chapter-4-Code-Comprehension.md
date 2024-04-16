---
title: OSCamp D9-rCore-Chapter-4-Code Comprehension
date: 2024-04-16 12:14:07
tags: [Rust, RISC-V]
category: [OSCamp]
---
今天花了大半天读代码，然后整理出下面这张图，作为rCore第4章地址空间一览。图中包括内核的地址空间和两个进程的地址空间。**虚线表示地址相等，没有虚线的部分的宽度比例没有实际意义。**

地址转换机制请见上一篇文章。

{% asset_img rcore-ch4-mm-spaces.png %}

## 地址空间切换
虚拟地址空间到物理地址的映射由`MemorySet`决定，内核只有一个全局的实例`KERNEL_SPACE`，而每一个进程都有独特的`MemorySet`。用户程序切换到内核态时，在执行复杂代码前必须先转换地址空间。在回到用户态之前必须把地址空间转换回去。

如图，包括内核和所有进程的虚拟地址空间的最后一个VP(虚拟页)被映射到**同一个PP(物理页)**：**蹦床(Trampoline)**，即`.text.trampoline`段中的`__alltraps`。

图中内核的最后一个VP涂色和进程的不一样，原因是内核虽然这一段映射与进程相同，但内核态的trap处理程序会被切换成另一个函数。

下面是第四章引入的新的trap跳转机制：

1. 用户态`stvec`保存了`TRAMPOLINE`，用户态发生trap时会跳转至该处。
2. 切换`sp`为`TRAP_CONTEXT_BASE`，在此处(**右方**)保存当前的上下文，不同进程的`TRAP_CONTEXT_BASE`映射到的物理页不同，所以上下文保存在各自的进程空间中。
3. 切换`satp`，**进入内核空间**。切换`sp`为内核栈。
4. 跳转执行处理trap的逻辑代码，该代码开始时会切换`stvec`为内核空间的trap处理程序。
5. trap处理完成，`stvec`切回`TRAMPOLINE`，并跳转至`__restore`。
6. `satp`切回进程空间。`sp`切换为`TRAP_CONTEXT_BASE`，然后从该处恢复之前保存的信息。
7. `sp`切回保存的：trap前的栈顶。
8. `sret`返回用户态。

## 内核的Identity Mappings
Identity Mappings指的是直接把VP映射到相同的PP。注意到图中左上角的`Kernel Range`和`Physical Space Range`，它们是第四章唯二的Identity Mappings：
* Kernel Range：包含linker script中的所有段，也就是所有的内核代码、数据，请见`MemorySet::new_kernel`。
* Physical Space Range：从`ekernel`开始直到`0x88000000`的空间，可以被分配和管理的空间。


## 内存分配
注意图中天蓝色和灰色区域。

* 灰色区域：没有映射到物理地址的虚拟地址空间，如果被访问将会触发著名的Segmentation Fault。
* 标记为guard的灰色page的目的是防止栈越界访问到其他区域。
* 天蓝色区域是Identity Mapping，整块由全局的`FRAME_ALLOCATOR`分配和回收，它分配的PP目前只有两种用途：**内核栈和进程内存**。

**这里没有循环定义，因为Identity Mappings本身不会被`FRAME_ALLOCATOR`管理——它们的PP是直接由VP确定的。**

进程空间的User Stack左侧有三块特殊的Area，对应ELF文件的类型为PT_LOAD的Program Headers，分别是：
1. 具有读取、执行权限的代码区域。
2. 具有读取权限的只读数据区域。
3. 具有读取、写入权限的数据区域，包括未初始化的bss段。

这三块区域连同右侧的User Stack、倒数第二页的TRAP_CONTEXT，**总计五块区域**在程序正式执行前由操作系统分配固定大小的空间。

另外两个则是：

* 最后的TRAP_HANDLER：固定映射到同一个PP，前面已经说过，不再赘述。
* Heap：初始时`BRK`和`Heap Bottom`重合，因为大小为0。但是程序可以通过系统调用让OS分配更多内存，同时会将`BRK`右移，Heap扩大。


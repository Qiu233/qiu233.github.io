---
title: OSCamp D9-rCore-Chapter 4 Code Comprehension
date: 2024-04-16 12:14:07
tags: [Rust, RISC-V]
category: [OSCamp]
---

我花了一天多时间做了下面两张图，分别是*地址空间一览*和*第四章内核态内存管理的层次结构一览*。

* [第一节-地址空间](#地址空间)关注第一张图，解释地址空间的分配。
* [第二节-内存管理](#内存管理)利用第二张图解释第四章是如何实现内存管理的。


{% asset_img rcore-ch4-mm-spaces.png %}
{% asset_img rcore-ch4-mm-hierarchy.png %}

# 地址空间
图中包括内核的地址空间和两个进程的地址空间。**虚线表示地址相等，没有虚线的部分的宽度比例没有实际意义。**

地址转换机制请见上一篇文章。


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


## 特殊的地址空间
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

# 内存管理
从这里开始请关注第二张图。

开始之前请读者确认这一点：
> 因为内核的Heap实际上是一段分配在.bss上的全局数组`HEAP_SPACE`和对应的`HEAP_ALLOCATOR`，所以也包含在Kernel Range中了。

**内存管理完全在内核空间中进行。**

第四章的`TaskManager`有如下变化：
* 将固定大小的`TaskControlBlock`(之后简称TCB)数组改为一个分配在Heap上的`Vec`。
* `TCB`新增一些字段，其中最重要的是`memory_set: MemorySet`，决定进程的地址空间。

`MemorySet`表示一个完整的地址空间，每个进程都有独立的`MemorySet`，内核有唯一一个全局的`KERNEL_SPACE`。

注意：进程的`MemorySet`对进程本身不可见，它分配在内核的Heap上，只对内核可见。内核管理进程的内存必须切换到内核态和内核空间中。

* `MemorySet`持有一个`PageTable`，保存页表树节点的Frame都在此处。
* 除此之外的，即最"普通"的Frame，在更下一层的结构`MapArea`中。
* **Identity Mappings没有Frame会被持有，因此也没有回收。**

`MemorySet`持有一个`MapArea`列表，一个`MapArea`表示地址空间中一段连续的、**具有相同性质**的多个VP。例如代码和数据就应当分属不同的`MapArea`，因为代码可执行，而数据一般不可执行。

`MapArea`的细节如图右下角所示，最重要的是Data Frames，它用一个分配在内核Heap上的`BTreeMap`保存了VPN到Frame的映射。

## 内存分配
内核启动后仅清理.bss和输出log信息后，立刻将`KERNEL_SPACE`初始化，并设置`satp`寄存器启用之。

启动程序时(见`MemorySet::from_elf`和`TaskControlBlock::new`)：
1. 映射全局固定的`TRAMPOLINE`
2. 解析ELF文件，为各个类型为`PT_LOAD`的Program Header分配内存，并加载到内存中
3. 分配栈空间(长度3页，分配2页)
4. 保留堆区域，初始时长度为0
5. 分配`TRAP_CONTEXT`空间，并映射于地址`TRAP_CONTEXT_BASE`
6. 为进程分配内核栈，**映射于内核空间中**

过程还会解析程序入口点地址，程序启动跳转与前两章没多少区别。

## 内存回收
当进程结束时，内核空间中对应进程的整个`MemorySet`应当被销毁，此时它所有下级结构持有的Frame全部都会被`FRAME_ALLOCATOR`回收。

此外，由于启动进程时还在内核空间中分配了进程的内核栈，进程结束时该空间也应当一并回收。

但是我在原仓库没有找到任何有关程序结束内存回收的代码，可能是因为目前程序的执行数量和顺序都在编译期决定，不需要回收也不影响结果。
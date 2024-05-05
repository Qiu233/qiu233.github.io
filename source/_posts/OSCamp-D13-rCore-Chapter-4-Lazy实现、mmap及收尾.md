---
title: OSCamp D13-rCore Chapter 4-Lazy实现、mmap及收尾
date: 2024-04-20 17:50:33
tags: [Rust, RISC-V, Memory]
category: [OSCamp]
---

第四章的难度略高，这作为本章学习记录的最后一篇吧，不打算花更多时间在这上面了。

<!--more-->

# Lazy实现思路

把原有`MemorySet::push`函数分成两种：

* `push_strict`：将area压入并立刻为其分配所有需要的Frame，对`TRAP_CONTEXT_BASE`和ELF文件的LOAD段应当采用这种方式。
* `push_lazy`：将area压入，参数`data`不为`None`时分配需要的空间，如果为`None`则完全不分配空间。

程序运行时若触发page fault，trap handler会对`stval`中地址对应的目标虚拟页调用`MemorySet::translate`，该函数返回`MMResult<PageTableEntry>`，函数的作用是：

1. 若所有area的范围都不包含目标页(`vpn`)，那么返回错误。
2. 若有area的范围包含目标页，调用该area的`ensure_range`函数，传入`[vpn, vpn + 1)`。
3. `MapArea::ensure_range`函数会**保证这一段区域被分配物理内存且映射**，否则返回错误，如果已经映射那么直接返回Ok。
4. 以上流程后，调用`page_table`的`translate`，计算出PTE的位置并返回PTE的值。


如上述得到的结果`MMResult<PageTableEntry>`有如下用途：

* 如果是错误码，那么一定出现了无法恢复的错误，例如尝试分配内存但是内存耗尽，或请求的地址没有在任何area中，OS立刻结束当前程序。
* 如果是Ok，那么得到的PTE将用于检查对应位是否具有请求的访问权限。例如触发page fault的是写入指令，那么PTE的W和R必须为1，否则访问权限出错，程序结束。
* 本章课程的进程空间所有映射的U位都一定为1，所以不必检查U位。

Lazy的一个重要性质是：
> 不必考虑"分配失败"时的回收，如果某个page fault发生时分配也失败，那么直接结束程序释放所有内存。这样做的原因是：
> 
> 用`push_lazy`分配地址空间时不会因为物理内存用尽而失败，程序当下用到多少就分配多少，而page fault时一次只分配一个Frame，如果映射失败，获得的FrameTracker会被drop掉。
>
> 目前还没做页面置换，所以物理内存耗尽时没有什么更好的办法了。

# mmap和munmap

这两个东西实现起来有些繁琐，虽然课程内容似乎没有要求，但我实现了如下性质：

> `munmap`的参数可以横跨多个area，也可以在某个area的中间或两端。总而言之，唯一的要求是参数传入的范围必须**全部都在已有的area中**。

* 如果传入的范围(本段内简称range)在某个area的中间，那么调用后该area会被裂成两块。
* 上述的两块中若有空area，即range在原area的两端之一，那么调用后仅有不为空的块保留。
* 更特殊的情况是range覆盖了某个area，那么该area会被移除，即上述两块都为空的情况。
* 该函数会对所有area重复以上过程，因此对横跨多个area的情况也适用。

综上，只有刚刚好在指定范围中的虚拟页会被unmap，而原有的area中除此之外的虚拟页会被保留。该算法在`mmap`和`munmap`导致的area数量较多时效率会降低，因为我没有实现相同权限的相邻area合并。

此外，`munmap`要求：
* 参数地址必须按4K对齐。
* 参数指定的范围不能含有特殊页，例如`TRAP_CONTEXT_BASE`和`TRAMPOLINE`。如果这两块被unmap那么进程空间将永久失去trap能力，系统崩溃。
* 参数指定的范围不能含有未被任何area包含的部分。


`mmap`需要检查：
* 参数地址必须按4K对齐。
* 参数指定的范围不能含有特殊地址，例如`TRAP_CONTEXT_BASE`和`TRAMPOLINE`。原因同上
* 权限参数必须有效，即XWR三项不得都为0，且除这三项外必须全为0。
* 参数指定范围没有任何已经map的部分。

而`mmap`实现十分简单，只需要在检查之后调用`push_lazy`将area记录即可，具体的分配要等到page fault触发时。

# 内存顺序
内存顺序(Memory Order)真的是一个大坑，本节相关的只有`sfence.vma`指令，作用是在设置satp寄存器之后刷新TLB。

LR/SC全称是(Load Reserved/Store Conditional)：
* LR.aq：从内存某个位置加载，并且**获得**(acquire)该位置。
* SC.rl：写入内存位置，只有之前用LR.aq获得了该位置且并未失效，这条指令才会成功。

失效的原因是该位置被其他hart或IO设备写入了，代表前面读取的值已经不再有效，此时`SC.rl`的第一个寄存器operand中的值不为0表示失败，一般在失败时要跳回`LR.aq`重新再来。这样一个完整的流程叫做CAS。

aq和rl是RISC-V所有原子指令都有的字段：
* `aq`表示acquire，为1表示该指令后的所有存储指令必须等它完成才开始执行，`LR.aq`保证读取发生在后面的指令之前。
* `rl`表示release，为1表示该指令前的所有存储指令必须为该指令观察到，`SC.rl`保证写入发生时前面的读取已完成。


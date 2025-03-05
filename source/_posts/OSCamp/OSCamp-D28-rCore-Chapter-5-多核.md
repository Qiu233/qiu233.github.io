---
title: OSCamp D28-rCore Chapter 5-多核
date: 2024-05-05 20:44:51
tags: [Rust, RISC-V, ASync]
category: [OSCamp]
---

之前花了很多时间赶排行榜，没时间写博客，好在以第一名出线了，从这里开始补前面的坑。

这篇文章的内容是第五章的选做题——多核调度。QEMU启动参数加上`-smp n`即可以`n`个hart启动。

<!--more-->

# DTB和hartid的寻找

DTB全称Device Tree Blob，是一段保存在固件中的二进制序列，在RISC-V启动时被加载到内存中，由a1寄存器传递到SBI，并同样由a1寄存器传递到内核。QEMU会在启动模拟时根据模拟的设备生成DTB。

DTB保存了设备的信息，比如CPU的频率、核心数量及ID、寄存器映射等等，SBI和内核可以解析这些信息，具体的格式不必关心(请见spec)，我们可以直接用第三方库hermit-dtb来解析。

CPU核心在叫做`cpus`节点下的子节点中，每个CPU的格式是`cpu@n`，例如`cpu@0`就是0号CPU。通过遍历这些子节点并且用字符'@'分割名称即可得到可用的hartid。

# hart的启动
需要注意，设备(或模拟器)启动时运行的hart并不一定是0号，这个hart称为boot hart，内核入口点处a0寄存器保存的即是boot hart的hartid。启动其他hart的时候必须绕开boot hart。

启动hart可以直接用sbi函数`sbi_hart_start(hartid: usize, start_addr: usize, opaque: usize)`完成，参数分别是：
* hartid：即前面解析DTB找到的id，要跳过boot hart。
* start_addr：启动后的入口点，此时`a0`保存了hartid，`a1`保存了opaque。
* opaque：启动后的参数，如上所说被保存在`a1`中。

opaque可以传入一些参数来初始化hart，例如satp和hart的boot stack。因为我们知道原本的`run_tasks`运行在boot stack上，而为了让其他hart也有自己的idle task，自己的栈是必须有的。

# Hart Local Data
类比线程的Thread Local Storage，这里我给每个hart自己独有的数据起了个名Hart Local Data(不知道正式叫法)，在每个hart启动时生成，分配在kernel heap上，地址保存在tp寄存器中。

做lab时我设计的HLD只有两个数据：
* hartid：hart初始化时保存在此处再也不动，可以用来丰富输出信息方便调试。
* Processor实例：每个hart都有独有的Processor实例来运行Task。

然后写一个函数`get_processor`，实现是从tp寄存器读取到指针，然后从指针的位置获取Processor引用。这样每个不同hart调用该函数都能获取自己的Processor引用，无需任何参数。

rCore之前的trap代码并没有保存和恢复tp寄存器，所以需要改动一下：
* 在`TrapContext`中加入新字段`kernel_tp: usize`，保存内核态的tp寄存器。
* 在`trap.S`中加入保存和恢复用户程序`tp`寄存器的代码。
* 在`trap.S`中加入恢复和保存内核`tp`寄存器的代码。

当hart准备执行一个task的用户态时，会将当前的`tp`寄存器保存到`kernel_tp`的位置，然后恢复用户态的`tp`。当遇到中断进入内核态时，会保存用户态的`tp`并恢复内核态的`tp`。

# 调度同步
以上实现还有个问题：
> 一个hart切换到下一个任务时，会：
> 1. 先将`Arc<TaskControlBlock>`加入调度队列，
> 2. 然后才调用`schedule`，`schedule`会调用`__switch`保存当前任务的`TaskContext`，并且切换到idle task。
>
> 问题是其他hart可能在**从调度队列获取到任务时**，该任务的`TaskContext`还没有被前一个hart保存完毕，此时其中还是无效的信息，无法被`__switch`正确利用。

为了解决这个问题，需要额外的同步手段，我的做法是：
* 在`TaskContext`中加入新字段`lock: AtomicUsize`。
* 在`__switch`中间将前一个`TaskContext`的lock释放(置0)，并且获取下一个`TaskContext`的lock(读0置1).

我的实现用的是RISC-V的`LR/SC`指令，需要在文件头部加入`.option arch, +a`开启atomic extension。

# 死锁问题
第五章、第六章(不包括第八章)有两个明显的死锁问题必须解决，分别在`exit_current_and_run_next`和`sys_waitpid`处。

具体如何解决就不说了，读者自己想，多核支持反正是选做题，而且死锁问题知道发生在哪其实已经解决80%了。


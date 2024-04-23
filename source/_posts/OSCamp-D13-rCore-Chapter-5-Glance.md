---
title: OSCamp D14-rCore Chapter 5-Glance
date: 2024-04-20 21:04:38
tags: [Rust, RISC-V, Memory]
category: [OSCamp]
---

下面是第五章的进程切换示意图：

{% asset_img rcore-ch5-task-scheduling.png %}

<!--more-->

# 进程调度

上图展示的是第五章的进程切换方式，和前几章不同的是第五章用一个`idle task`来切换，它有如下性质：

* 没有用户代码，整个idle task只执行内核代码。
* 不在进程队列中，不会被"管理"，只有在切换进程时才会被调度。

idle task的代码只有一个简单的循环(函数`run_tasks`中)：

0. 循环头。
1. 通过函数`fetch_task`从进程队列取出下一个要执行的进程。
2. 用`__switch`切换至该进程，并且把当前状态保存至`idle_task_cx`。
3. 当该进程返回，会通过`__switch`切回到此处，跳转至循环头，开始下一轮调度。

`run_tasks`函数的唯一一次调用在`rust_main`中，内核在初始化完毕后直接进入此处开始执行进程，在进程队列为空时变成空耗的循环。

有一个特殊情况：当进程结束并切换至idle task时，TaskContext不会被保存，因为该进程绝对不会再次被调度，见函数`exit_current_and_run_next`。


# TCB和TaskManager

TaskManager的变化：
* 不再保存Task数量和当前Task的id。
* 原本的`Vec<TaskControlBlock>`改为一个计数引用(~~我用计数引用表示引用计数的名词~~)的双向链表`Deque<Arc<TaskControlBlock>>`。

当前执行状态被转写到另一个新类型`Processor`中：

```Rust
pub struct Processor {
    ///The task currently executing on the current processor
    current: Option<Arc<TaskControlBlock>>,

    ///The basic control flow of each core, helping to select and switch process
    idle_task_cx: TaskContext,
}
```

Processor有一个全局唯一的静态对象，因为教程目前完全是单核的，所以这个设计也没什么问题。

其中`idle_task_cx`是idle task的TaskContext，它的值初始时全为0，直到`run_tasks`第一次调度进程才会被填充。

current是当前进程`TCB`的引用，至于它为什么被`Option`包着，请见`take_current_task`函数的所有调用处，原因是：
> current只会在`run_tasks`的循环里，在切换之前，被设置为非`None`值。  
> 当进程被suspend或exit时，`current`中原值被拿出，`None`被写入，从此之后直到进入`run_tasks`的循环之前都为None。  
> 特殊情况是当前进程队列为空，`run_tasks`空转，`current`恒为`None`，直到有新的进程入队。

## TCB
我看到源码里面有一些注释提到`PCB`，但是`TaskControlBlock`还是原来的名字，我暂且还是用TCB称呼它。

`TCB`相比前几章最重要的区别是以下四个字段：

* `pid: PidHandle`，该类型实现了`Drop`，会在被销毁时将pid回收以便之后的进程创建使用。
* `parent: Option<Weak<TaskControlBlock>>`：持有父进程的弱引用。
* `children: Vec<Arc<TaskControlBlock>>`：持有一列子进程的计数引用。
* `exit_code: i32`：进程结束时设置的值，说明进程的结果，父进程可用于判断子进程是否按预期结束。

需要注意parent和children用了不同的智能指针：
* 弱引用`Weak<TCB>`不会阻止值被drop，因此子进程可以在**父进程已结束且被销毁**的情况下继续运行，此时`parent.upgrade`会返回`None`，由此判断父进程已被销毁。
* 若父进程存活，那么始终持有子进程的`Arc<TCB>`，这种情况下子进程即便已结束也**不会被销毁**，如此父进程便有机会检查子进程的`exit_code`。

# 进程的销毁

进程的创建，除了多了几个syscall，和前几章没有太大变化，defer到最后。这里先关注进程的销毁和相关数据的生命周期。

下面这个树状图根据类型的嵌套关系绘制：

{% asset_img rcore-ch5-lifetime.png %}

其中矩形的颜色表示是否实现Copy和Drop：

* 绿色表示不实现Copy，但实现Drop。
* 蓝色表示实现Copy。

其中最重要的是以下三个**实现Drop**并且有对应**全局Allocator**的类型：
* PidHandle：进程创建时分配。
* KernelStack：进程创建时分配。
* FrameTracker：进程运行过程中不断分配和回收。

当进程的`Arc<TCB>`的最后一个引用被销毁，TCB被drop，树的下层所有结构全部被回收，包括以上三者。

# 相关syscall

## sys_fork
完全复制当前进程包括当前执行状态等，生成一个子进程。唯一区别是`sys_fork`的返回不同：

* 新的子进程的`sys_fork`返回0。
* 父进程的`sys_fork`返回子进程的PID。

如果觉得抽象请见user库的`ch5b_initproc`程序：该程序用子进程执行`ch5b_user_shell`并用当前进程等待shell结束。

## sys_exec
用程序名执行目标程序，注意，该函数只保留PID和内核栈，会**完全替换**当前进程的地址空间，相当于在原进程中腾笼换鸟。

和`sys_fork`结合起来就能实现创建一个全新的进程，请见`ch5b_initproc`。

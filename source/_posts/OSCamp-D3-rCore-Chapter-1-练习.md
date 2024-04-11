---
title: OSCamp D3-rCore-Chapter-1-练习
date: 2024-04-11 03:48:00
tags: [Rust, OSCamp, Quiz]
category: [trace]
---

## 练习3
> 题目实现一个基于rcore/ucore tutorial的应用程序C，用sleep系统调用睡眠5秒（in rcore/ucore tutorial v3: Branch ch1）

我个人认为这是第一章最有趣的练习，所以就放在第一位了。

### 思路

最"朴素"的思路是首先用`sbi_rt::set_timer(u64) -> SbiRet`函数设置要等待的时间，然后用`wfi`指令等到下一个中断即可。虽然这个思路在qemu中模拟没问题，但是实际环境下"下一个中断"有可能不是timer中断，所以完整的思路是：

1. 用`sbi_rt::set_timer`设置要等待的时间
2. 用`csrrs`指令读取sie寄存器并把STIE置1，即开启timer中断
3. 用`wfi`指令等待下一个中断
4. 用`csrr`指令读取`sip`，并判断STIP是否为1，如是则说明是timer中断，继续执行，否则说明被其他中断唤醒，跳转到3.重新等待
5. 到这里说明timer中断已触发，用`csrrw`指令将2.读出的值恢复到sie中，并返回

其中1.在Rust中调用会方便很多，但是2.开始的所有步骤在汇编里实现更简洁，下面是代码实现，增加在src/entry.asm文件的最后：

```assembly
# src/entry.asm
    .text
    .global wait_STIP
wait_STIP:
    li t1, (1<<5)
	csrrs t0, sie, t1
.loop:
    wfi
    csrr t2, sip
    and t2, t1, t2
    beqz t2, .loop
    csrrw a0, sie, t0
    ret
```

在Rust中即可写出这样的函数：

```Rust
// src/main.rs
fn sleep(time: u64) {
    extern "C" {
        fn wait_STIP();
    }
    sbi_rt::set_timer(time);
    unsafe {
        wait_STIP();
    }
}
```

其中参数time是要等待的时钟周期，QEMU的时钟周期是10 MHz，题目要求的5秒延迟即是：

```Rust
sleep(50_000_000);
```

添加到`rust_main`函数中就能看见效果。

### 解释
SIE和SIP是两个S模式下的CSR寄存器，分别用来控制中断是否可用和中断是否等待处理，它们的完整结构请参考RISC-V的文档。从0开始算的第5位即STIE和STIP，分别控制timer中断的可用和等待处理。

*SIE和SIP是S模式下的寄存器，在M模式下有对应的MIE和MIP。bootloader(即教程的sbi_rt)在转移控制权到kernel之前已经进入了S模式，所以我们无法用M模式的CSR寄存器。*

我们先把STIE置1，即开启timer中断，然后通过循环等待直到STIP为1为止。需要注意，STIP的置1不是程序来完成的，而是通过内部的stime和stimecmp寄存器的比较自动完成的。这个过程不受STIE的取值影响，那么一个问题是：能不能不设置STIE而只检测STIP的值？

逻辑上来说确实可行，但`wfi`指令的定义是直到下一个中断才被唤醒，所以如果STIE为0(即timer中断不会触发)那么有可能要等待的时间超过设定值。因此这种情况不应该用`wfi`而是写一个循环检测，但这就无法利用RISC-V的节省能源的优势了。

_未完_
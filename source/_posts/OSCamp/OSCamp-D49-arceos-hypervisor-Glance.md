---
title: OSCamp D49-arceos-hypervisor-Glance
date: 2024-05-26 15:39:23
tags: [Rust, RISC-V]
category: [OSCamp]
---


今天废了很大劲才发现riscv只支持linux，不能用nimbos，按照(https://github.com/arceos-hypervisor/hypercraft)的README即可配好环境。

启动指令：

```bash
make ARCH=riscv64 A=apps/hv HV=y LOG=info run
```

启动之后会进入shell，没有`sudo`，执行`poweroff`即可关闭。

接下来就是在这个基础上继续开发hypervisor了。

<!--more-->

# 理论知识

## RISC-V的虚拟化原理

* 在S模式和U模式之间加入VS模式，原来的S模式变为HS模式。
* 加入一系列`h`开头的寄存器，只能在HS模式中访问，用于控制VM，例如地址转换等等。
* 大部分S模式的CSR都有对应的VS模式的镜像，以`vs`开头，可在VS模式下当成S模式寄存器访问。
* 加入读写指令：
  * `HLV`：在M、HS模式中读取guest的内存。在U模式中读取需要`hstatus.HU=1`。
  * `HSV`：同上，但是时写入而非读取。
  * `HLVX`：与`HLV`相同，但是要求页面的权限是可执行而**非**可读。

## 地址转换

地址转换分为两个阶段：

* `VS-Stage`：guest virtual address到guest physical address，也叫第一阶段。
* `G-Stage`：guest physical address到supervisor physical address，也叫第二阶段。

第一阶段和非虚拟化的地址转换毫无区别，第二阶段的root table则占用16K而非4K。

# ArceOS

## HV的启动
在`rust_main`中有下列代码：
```Rust
#[cfg(feature = "hv")]
unsafe {
    main(cpu_id)
};

#[cfg(not(feature = "hv"))]
unsafe {
    main()
};
```

很显然在hv开关有效时，内核启动后会直接调用库`apps/hv`的`main`函数，进入hypervisor。

所以第一个确定性的结论就是：`hv`和`hypercraft`的代码都是完全运行在内核态的。

## VM::run

函数`VM::run`是最重要的函数，其中包含一个死循环，类似于rCore的`run_tasks`：

* `get_vcpu`找到执行的cpu。
* 调用`vcpu.run()`，开始执行虚拟机并维护状态，后面再展开。该函数返回时代表从VS陷入HS。
* `vcpu.run()`返回trap信息，根据这些信息决定如何行动，会更新寄存器状态。
* 设置新的寄存器状态，然后进入下一次循环。

正常退出循环的唯一机会是guest请求类型为reset的sbi call，此时整个系统都会被shutdown，否则就只有panic才能退出了。


## VCpu::run
该函数的原理类似于rCore的`schedule`：

* 首先调用`_run_guest`，类似rCore的`__switch`，作用是保存当前上下文，运行，并在trap发生时返回该处。
* 根据返回的trap信息生成VmExit信息，并且返回上一层。

## _run_guest
类似rCore的`__switch`：

* 传入的指针(a0)同时包括了hypervisor和guest的寄存器。
* 首先保存当前寄存器到传入的指针中的hyp寄存器，但是不保存callee-saved的临时寄存器t*。
* 交换CSR寄存器，这里只挑重要的说：
  * `sepc`并没有交换，只是单纯地把`guest_sepc`写入，最后的sret指令会跳转到该处。`guest_sepc`最初的值在`VCpu::new`中设置。
  * `stvec`设置为`_guest_exit`的地址，原来的值被保存。目的是在guest发生trap时跳转到`_guest_exit`。
  * 将`a0`保存到`sscratch`中，原来的值保存，会在`_guest_exit`中被换回来。
* 最后从指针依次恢复guest的所有通用寄存器，`a0`在最后。
* `sret`，开始执行guest。

## _guest_exit

* 第一条指令就是`csrrw a0, sscratch, a0`，将之前保存的指针a0换出来。
* 基本是`_run_guest`倒过来，先保存guest寄存器，然后恢复hyp寄存器。


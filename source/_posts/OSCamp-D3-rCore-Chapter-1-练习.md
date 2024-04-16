---
title: OSCamp D3-rCore-Chapter-1-练习
date: 2024-04-11 03:48:00
tags: [Rust, RISC-V]
category: [OSCamp]
---


# 编程题

## 练习1
> 实现一个linux应用程序A，显示当前目录下的文件名。（用C或Rust编程）

思路：用`std::fs::read_dir`函数读取当前文件夹，唯一要注意的是参数类型是`std::path::Path`，但是可以从`&str`自动coerce，所以没必要写得太麻烦。

```Rust
fn main() -> Result<(), std::io::Error> {
    let d = std::fs::read_dir(".")?;
    for f in d.into_iter() {
        let f = f?;
        println!("{}", f.file_name().to_str().unwrap());
    }
    Ok(())
}
```

## 练习2
> 实现一个linux应用程序B，能打印出调用栈链信息。（用C或Rust编程）

这道题不难，但是实现起来稍微有点麻烦，我手上没有可供调试的RISC-V设备，而且题目并没有规定非得在RISC-V上实现，所以我这里就写一个x86版本吧(经测试在wsl2上运行无误)。

### 思路
利用GNU函数`backtrace_symbols_fd`，传入return_address即可获得calling function的symbol。

不用`backtrace_symbols`是因为这个函数会返回一个`*mut *mut c_char`数组并且需要我们自己来释放。而`backtrace_symbols_fd`会输出到传入的fd(即file descriptor)中，利用`memfile`在内存中保存它的输出即可。

### 准备工作
x86下Rust生成的程序默认没有rbp，也就是默认启用frame pointer omission机制，导致rbp永远是0，要关闭这个机制，需要在cargo.toml第一行启用`profile-rustflags`并在profile中加入参数：

```toml
# cargo.toml
cargo-features = ["profile-rustflags"]
# ...
rustflags = ["-C", "force-frame-pointers=yes"]
```

而rustflags这个项当下需要nightly，所以添加rust-toolchain.toml文件，并加入下面的内容：
```toml
# rust-toolchain.toml
[toolchain]
channel = "nightly-2024-01-18"
```

### 代码
代码要点如下：

1. `get_trace`函数根据传入的return address返回符号信息，用`memfile`暂存输出
2. `traverse`函数从当前stack frame开始往前一直走，直到`rbp=0`为止，说明已经到底了，函数返回一个(rbp, return_address)的iterator
3. 在64位的x86中，`[rbp]`是frame pointer，`[rbp+8]`是return address

```Rust
extern "C" {
    fn backtrace_symbols_fd
        (buffer: *const *mut std::ffi::c_void, size: i32, fd: i32);
}

fn get_trace(ra: usize) -> Option<String> {
    use std::ffi::*;
    use std::os::fd::AsRawFd;
    use memfile::{MemFile, CreateOptions};
    use std::io::{Read, Seek};
    unsafe {
        let mut mem_file = MemFile::create("trace_mem", CreateOptions::new()).ok()?;
        let fd = mem_file.as_fd().as_raw_fd();
        backtrace_symbols_fd(&(ra as *mut c_void), 1, fd);
        mem_file.seek(std::io::SeekFrom::Start(0)).ok()?;
        let mut symbol: String= "".into();
        mem_file.read_to_string(&mut symbol).ok()?;
        Some(symbol)
    }
}

fn traverse() -> impl Iterator<Item = (usize, usize)> {
    unsafe {
        let mut ra : usize;
        let mut bot: usize;
        core::arch::asm!("mov {0}, [rbp]", out(reg) bot);
        core::arch::asm!("mov {0}, [rbp+8]", out(reg) ra);
        let mut result: Vec<(usize, usize)> = Vec::new();
        loop {
            result.push((bot, ra));
            core::arch::asm!("mov rax, {0}", in(reg) bot);
            core::arch::asm!("mov {0}, [rax]", out(reg) bot);
            if bot == 0 {
                break;
            }
            core::arch::asm!("mov {0}, [rax+8]", out(reg) ra);
        }
        result.into_iter()
    }
}

fn main() {
    for (rbp, ra) in traverse() {
        println!("rbp: {:#x}", rbp);
        println!("ra: {:#x}", ra);
        if let Some(s) = get_trace(ra) {
            println!("{}", s);
        }
    }
}
```


## 练习3
> 实现一个基于rcore/ucore tutorial的应用程序C，用sleep系统调用睡眠5秒（in rcore/ucore tutorial v3: Branch ch1）

### 思路和实现

为确保思路清晰明确，详细解释放在本节的最后，如需要则请跳转该处。

最"朴素"的思路是首先用`sbi_rt::set_timer(u64) -> SbiRet`函数设置要等待的时间，然后用`wfi`指令等到下一个中断。虽然这个思路在qemu中模拟没问题，但是实际环境下"下一个中断"有可能不是timer中断，所以完整的思路是：

1. 用`sbi_rt::set_timer`设置要等待的时间
2. 用`csrrs`指令读取sie寄存器并把STIE置1，即开启timer中断
3. 用`wfi`指令等待下一个中断
4. 用`csrr`指令读取`sip`，并判断STIP是否为1，如是则说明是timer中断，继续执行，否则说明被其他中断唤醒，跳转到3.重新等待
5. 到这里说明timer中断已触发，用`csrrw`指令将2.读出的值恢复到sie中，并返回

其中1.在Rust中调用会方便一些，但是2.开始的所有步骤在汇编里实现更简洁，下面是代码实现，增加在src/entry.asm文件的最后：

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

其中参数time是要等待的时钟周期，QEMU的频率是10 MHz，题目要求的5秒延迟即是：

```Rust
sleep(50_000_000);
```

添加到`rust_main`函数中即可看见效果。

### 解释
SIE和SIP是两个S模式下的CSR寄存器，分别用来控制中断是否可用和中断是否等待处理，它们的完整结构请参考RISC-V文档。从0开始的第5位即STIE和STIP，分别控制timer中断的可用和等待处理。

*SIE和SIP在M模式下有对应的MIE和MIP。bootloader(即教程的sbi_rt)在转移控制权到kernel之前已经进入了S模式，所以我们无法用M模式的CSR寄存器。*

我们先把STIE置1，即开启timer中断，然后通过循环等待直到STIP为1为止。需要注意，STIP的置1不是程序来完成的，而是通过内部的stime和stimecmp寄存器的比较自动完成的。这个过程不受STIE影响，那么一个问题是：能否不设置STIE而只检测STIP的值？

逻辑上来说确实可行，但`wfi`指令定义为直到下一个中断才被唤醒，因此如果STIE为0(即timer中断不会触发)那么有可能要等待的时间超过设定值。所以这种情况不应该用`wfi`而是写一个循环检测，但这就无法利用RISC-V的节省能源的优势了。

# 问答题
## 1.应用程序在执行过程中，会占用哪些计算机资源？
1. CPU时间，包括线程，过多程序运行会导致程序执行变慢，因为程序被分配的时间片更少了
2. 内存，或主存，或RAM
3. IO设备，如硬盘上的文件，屏幕等
   
## 2.请用相关工具软件分析并给出应用程序A的代码段/数据段/堆/栈的地址空间范围。
在回答之前必须说明，应用程序的堆栈段(segment)是在程序被运行时才确定的，但是代码、数据等段(section)的信息在ELF文件的section headers中就有，segment和section并不是一一对应的。

首先是sections：

1. 代码段：`rust-objdump --section-headers \<file\> | grep ".text"`
2. 数据段：`rust-objdump --section-headers \<file\> | grep ".data"`，输出包括了.data和.rodata段
3. .bss段：`rust-objdump --section-headers \<file\> | grep ".bss"`

以上三者的输出，以代码段为例：
```
target/debug/test:      file format elf64-x86-64

Sections:
Idx Name          Size     VMA              Type
 15 .text         00042514 0000000000006060 TEXT
```

其中VMA是Virtual Memory Address的缩写，说明代码段.text从地址0x0000000000006060开始，长度为0x00042514。

而堆和栈的地址空间要在运行时才确定，用gdb断点并运行指令

```$ (gdb) info proc mappings```

输出中有这样两行：

```
0x5555555b2000     0x5555555d3000    0x21000        0x0  rw-p   [heap]
...
0x7ffffffdd000     0x7ffffffff000    0x22000        0x0  rw-p   [stack]
```
说明
* 堆起始地址为0x5555555b2000，目前大小为0x21000。
* 栈底为0x7ffffffff000，栈空间总大小为0x22000。

该输出也能用指令`cat /proc/pid/maps`得到，这里用gdb是为了下断点。另外要注意程序刚启动时可能没有`[heap]`，所以断点要下在分配过堆上数据的位置。

## 3.请简要说明应用程序与操作系统的异同之处。
相同点：
1. 都是软件
2. 具有.text、.data等逻辑段


区别在于操作系统：
1. 权限比应用程序更高，RV架构中运行在S模式，而应用程序运行在U模式。
2. 内核程序可以是纯二进制文件，意味着sections等元数据被剪除。
3. 内存分配由程序员手动完成，而应用程序的内存由操作系统分配。
4. 内核最开始一小段时间内并没有分页机制(satp)，且启用分页后内核一般有identity mapping，即将虚拟地址映射到相同的物理地址。而应用程序的虚拟地址几乎不可能映射到相同的物理地址。

## 4.请基于QEMU模拟RISC—V的执行过程和QEMU源代码，说明RISC-V硬件加电后的几条指令在哪里？完成了哪些功能？
QEMU源码：[qemu/hw/riscv/boot.c#L382](https://github.com/qemu/qemu/blob/02e16ab9f4f19c4bdd17c51952d70e2ded74c6bf/hw/riscv/boot.c#L382)

```C
uint32_t reset_vec[10] = {
    0x00000297,                  /* 1:  auipc  t0, %pcrel_hi(fw_dyn) */
    0x02828613,                  /*     addi   a2, t0, %pcrel_lo(1b) */
    0xf1402573,                  /*     csrr   a0, mhartid  */
    0,
    0,
    0x00028067,                  /*     jr     t0 */
    start_addr,                  /* start: .dword */
    start_addr_hi32,
    fdt_load_addr,               /* fdt_laddr: .dword */
    fdt_load_addr_hi32,
                                    /* fw_dyn: */
};
```

我的qemu上模拟的结果：

```
0x1000:      auipc   t0,0x0
0x1004:      addi    a2,t0,40
0x1008:      csrr    a0,mhartid
0x100c:      ld      a1,32(t0)
0x1010:      ld      t0,24(t0)
0x1014:      jr      t0
0x1018:      unimp
0x101a:      .2byte  0x8000
0x101c:      unimp
0x101e:      unimp
0x1020:      unimp
0x1022:      .2byte  0x8700
0x1024:      unimp
0x1026:      unimp
```

其中填充的一些数据(virt,rv64)：
* `start_addr=0x80000000,start_addr_hi32=0`，请见[qemu/hw/riscv/virt.c#L90](https://github.com/qemu/qemu/blob/02e16ab9f4f19c4bdd17c51952d70e2ded74c6bf/hw/riscv/virt.c#L90)
* `fdt_load_addr=0x87000000,start_addr_hi32=0`，请见[qemu/hw/riscv/boot.c#L294](https://github.com/qemu/qemu/blob/02e16ab9f4f19c4bdd17c51952d70e2ded74c6bf/hw/riscv/boot.c#L294)
* `reset_vec[3] = 0x0202b583;   /*     ld     a1, 32(t0) */`
* `reset_vec[4] = 0x0182b283;   /*     ld     t0, 24(t0) */`，这两个数据在上文源码附近即可找到

首先是跳转的流程：第一条指令`auipc`之后，t0的值变为`0x1000`，然后在第五条指令加载`0x1000+24=0x1018`处的数据到`t0`，即`t0=0x80000000`，然后`jr`跳转至此处。

然后是第三、四条指令的作用，分别是：
* 将当前的hartid加载到a0，hart是hardware thread的缩写，每个hart都有独立的id。hart和core的区别在于一个core可以有多个hart，但是一个core只有一个instruction fetch unit。hart可以类比理解成x86的hyperthread。
* 将`0x1020`开始的8个字节加载到a1，也就是小端地址`0x87000000`，即上面的`fdt_load_addr`的值。该地址是device tree在内存中的地址，在qemu源码中缩写为FDT，即Flattened Device Tree。device tree包含了CPU、内存、总线和主板的信息，也称作DTB(Device Tree Blob)，DTB的内容储存在非易变(non-volatile)的介质中。

在跳转到`0x80000000`之前a0和a1必须依上述加载，请见[docs.kernel.org](https://docs.kernel.org/arch/riscv/boot.html)。

第二行代码把`0x1028`加载到a2，刚好就是上面这段之后的地址，起初我以为这是为了跳转回来而保存的地址，但GDB输出完全不像是可执行的指令：

```
0x1028: 0x4942534f      0x00000000      0x00000002      0x00000000
0x1038: 0x00000000      0x00000000      0x00000001      0x00000000
0x1048: 0x00000000      0x00000000      0x00000000      0x00000000
0x1058: 0x00000000      0x00000000      0x00000000      0x00000000
```

然后我就一头雾水了，暂时没有查到什么资料，既然是传入的参数，随便找一个SBI实现应该就能搞清楚了。或者QEMU源码也可能有答案。


## 5.RISC-V中的SBI的含义和功能是啥？
SBI全称为Supervisor Binary Interface，规定了一套和SEE交互的接口。SBI的目的是在OS和硬件之间引入一层抽象，以便OS实现能在不改动或少量改动的情况下迁移到其他硬件环境，提供的功能主要与硬件相关，例如关机、屏幕输出、设置timer等。

## 6.为了让应用程序能在计算机上执行，操作系统与编译器之间需要达成哪些协议？

### 可执行文件格式
例如Windows的exe格式，Linux的elf格式。操作系统规定了合法的可执行程序的格式，编译器，或者准确来说是链接器，必须生成这种格式的文件。

### ABI和Syscall
ABI即Application Binary Interface，例如calling convention，Windows的64位程序的calling convention就和Linux的不同，编译器必须按照ABI生成代码。

Syscall则是编程语言的库函数(或者说API)的基础，不同的操作系统Syscall不同，编译器的库函数(例如标准库)实现必须对应到操作系统。一个具体的例子是CRT。

## 7.请简要说明从QEMU模拟的RISC-V计算机加电开始运行到执行应用程序的第一条指令这个阶段的执行过程。

1. 加电，CPU初始化，pc设置为0x1000，执行少量指令并跳转至0x80000000，开始执行bootloader。bootloader一般在特殊的存储器如ROM或Flash中，被固定加载至0x80000000，教程中是sbi_rt的预编译版本。这一阶段处于M模式，拥有最高的权限。
2. bootloader执行一些初始化，进入S模式，并跳转至kernel或下一级bootloader，教程中用qemu参数在启动前就把kernel加载至0x80200000，sbi_rt会固定跳转至0x80200000。更一般的情况是bootloader来加载kernel。bootloader另一个例子是OpenSBI，会跳转至0x84000000。
3. (Optional)第二级bootloader，例如U-Boot，加载kernel并跳转。和上一级不同的是这一级进入时就已经是S模式了。
4. kernel开始执行，运行在S模式下，启动虚拟地址(satp)，设置中断向量表(stvec)，中断向量表包括了系统调用(即scause=8)的情况。
5. kernel加载应用程序到内存，分析程序的格式，重定位(relocation)，为堆、栈、可执行代码和静态数据分配空间，然后跳转至程序入口，程序执行开始。

## 8.为何应用程序员编写应用时不需要建立栈空间和指定地址空间？
不需要建立栈空间是因为由操作系统来完成，实际上栈空间也不应该由程序自己建立，因为同一个进程下的不同线程也有独立的栈，而线程的数量和创建都是程序编译时无法确定的。

不需要指定地址空间是因为
* 开启satp后，每个进程都有完全独立的虚拟地址空间，程序不必担心和其他进程的地址空间重叠导致冲突。
* 实际上程序在(动态或静态)链接时会发生重定位。重定位是为了解决程序和其他依赖之间可能存在的地址冲突，因此自己指定地址空间毫无意义。

## 9.现代的很多编译器生成的代码，默认情况下不再严格保存/恢复栈帧指针。在这个情况下，我们只要编译器提供足够的信息，也可以完成对调用栈的恢复。

对于题目的情况，假如一般的函数有如下形式：

```assembly
addi    sp, sp, -size
...
addi    sp, sp, size
ret
```

* 当调试器断点在函数内部时，也就是在**pc位于第一条addi之后、最后一条addi之前**的情况下，`sp + size`即是calling function的栈帧顶部。
* 假如pc正好在第一条指令前，或在最后一条addi指令后，那么当前`sp`即是calling function的栈顶部。

如果能确保所有函数都有如上形式，那么调试器能从第一条指令的操作数中读出栈帧的总长度。但如果不能确保这一点，那么编译器必须提供每个函数的栈帧总长度，调试器仍然可以根据pc确认函数执行的位置来计算栈帧。

还有一种特殊情况，例如C语言的variable-length array可以在栈上分配运行时确定长度的数组，我通过反编译发现即便编译时带参数`-fomit-frame-pointer`，x86下生成的汇编代码也会使用`rbp`保存frame pointer。而网站[https://godbolt.org/](https://godbolt.org/)反汇编得到的RISC-V结果似乎把fp(即s0)当成通用寄存器使用，所以这个特殊情况似乎在不同平台上表现不同，我暂时得不出确定性的结论。

# 实验练习
题目太长就不在此转述了，请见原文。

因为课程的语言是Rust，log crate已经解决了彩色输出问题，所以这道题我只关注这个要求，同时这也是我觉得第一章最有趣的部分：

> challenge: 支持多核，实现多个核的 boot。

思路如下：

* QEMU的`-SMP`参数指定核心总数，默认为一，所以要在Makefile里面把-SMP设置为2或者更大的数。
* 机器(QEMU)启动时只有一个hart，称为booting hart，我们要在代码里自己启动其他hart，另外注意booting hart的hartid不一定为0，我被这个问题折磨了好久才发现。
* 在`rust_main`函数入口处保存hartid(在a0中)到局部变量，然后在`clearbss();`之后保存到全局变量，我用的是`BOOT_HART`。要注意顺序很重要，因为`clearbss();`会把初值为0和未初始化的全局变量全部清除。
* 然后从0开始遍历一小段hartid，跳过和`BOOT_HART`相等的值，分别调用`sbi_rt::hart_start(hartid, thread_entry, opaque)`并检查返回值，在第一次成功时跳出循环。我的实验只额外创建了一个hart，如果你需要更多hart的话可以写自己的逻辑。

## 实现

首先要找到一个可用的hartid，我找了很多资料也没有找到能在运行时探测可用hartid的方法，或者应该说是implementation-specific的，所以目前我是直接遍历一小段hartid，在QEMU下没有遇到问题。其中用到的全局变量`BOOT_HART`在前文描述思路时已经说过了，比较trivial所以代码就不放了。

```Rust
fn start_one_hart() {
    extern "C" {
        fn thread_1_entry();
    }
    unsafe {
        for i in 0..10 { // find the first available hartid
            if i == BOOT_HART {
                continue;
            }
            let result: sbi_rt::SbiRet = sbi_rt::hart_start(i as usize, thread_1_entry as usize, 0);
            if result.is_ok() {
                info!("[kernel] Successfully started hart #{:x}H.", i);
                break;
            } else {
                error!("[kernel] Failed to started hart #{:x}H: err= {}", i, result.error as isize)
            }
        }
    }
}
```


其他hart和booting hart一样，都需要栈空间，而且每个hart的栈空间相互独立。栈的初始化和kernel的入口点一样`la sp, thread_stack_top`即可，栈空间可以分配在`entry.asm`的`boot_stack_top`附近，即在原来的boot_stack前或后加入这段代码：

```
    .globl thread_1_stack_lower_bound
thread_1_stack_lower_bound:
    .space 4096 * 16
    .globl thread_1_stack_top
thread_1_stack_top:
```

`hart_start`函数的第二个参数指定启动的hart跳转到的地址，**不要跳转到Rust的函数**，因为`#[naked]`特性目前还是用不了。我建议跳转到`entry.asm`的新加入的.text段中的函数，设置好`sp`，然后立即用`call`跳转到Rust函数：

```
    .text
    ...
    # thread 1 entry point
    .global thread_1_entry
thread_1_entry:
    la sp, thread_1_stack_top
    call thread_1_main
```

如果按照这个步骤来做，那么跳回的Rust函数的第一、二个参数分别就是hartid和前面传入的opaque参数，不过这里因为没有用到所以注释掉了：

```Rust
// jump from function `thread_1_entry` in entry.asm
#[no_mangle]
fn thread_1_main(hartid: usize/*, opaque: usize*/) -> ! {
    info!("[kernel] Running from another thread, hartid = {}", hartid);
    loop {}
}
```

可以看到用`info!`输出了被启动的hart的hartid，但如果不加锁的话输出会乱序，所以将`console.rs`文件的`print`函数改为：

```Rust
// src/console.rs
use core::sync::atomic::{AtomicU32, Ordering};
...
static STDOUT_LOCK: AtomicU32 = AtomicU32::new(0);
...
pub fn print(args: fmt::Arguments) {
    while let Err(_) = STDOUT_LOCK.compare_exchange(0, 1, Ordering::AcqRel, Ordering::Acquire) {}
    Stdout.write_fmt(args).unwrap();
    STDOUT_LOCK.store(0, Ordering::Release);
}
```

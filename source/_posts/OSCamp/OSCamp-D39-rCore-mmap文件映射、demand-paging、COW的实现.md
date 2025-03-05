---
title: OSCamp D39-rCore mmap文件映射、demand paging、COW的实现
date: 2024-05-16 16:49:02
tags: [Rust, RISC-V, Memory]
category: [OSCamp]
---

大概是二阶段的最后一篇文章，内容是选做的：
* mmap支持文件的Shared和Private映射
* fork的COW，之前的文章里做过，这里把底层重新设计了一遍
* demand paging和lazy allocation

都是在ch6上实现的，并和之前已经实现的multi-hart兼容。

不在ch8上实现因为ch8我始终没有解决multi-hart的问题，且代码合并过于困难，而且最近忙别的去了没时间。

这篇文章的实现涉及上千行的代码修改，即便尽量关注实现思路，但必要的代码也不会少。因为不是必做题，所以除非感兴趣否则建议忽略。

<!--more-->

# 页面
对内存管理的代码小修小改无法系统化地支持这么多不同特性，所以我对ch6原仓库的代码进行了重构，其中最重要的类型是：

```Rust
enum Page {
    Identity(MIdentityHandle),
    Framed(MFrameHandle),
    FileShared(MFileHandle),
    FilePriv(MFileHandle)
}
```

从上到下分别是：
* Identity Mapping，仅为保持一致性而存在，实际上为了节省内存，内核空间的Identity Mapping都被特殊处理了
* 分配的物理内存页帧，包括了lazy和fork时COW的情况
* 以Shared模式映射的文件页，所有修改都会反应到文件和其他相同的映射中
* 以Private模式映射的文件页，如只读或未被修改则和Shared模式无异，当进程尝试修改时会转换为`Framed`

这些以`Handle`结尾的类型全部都实现了`Drop`，保存了一个指针`*mut PageTableEntry`，并且由外部的不同Manager来分别管理。

* 当被销毁时，它们会将对应的PTE中的valid位置0，由此实现unmap。
* 当page fault发生时，MapArea会根据page fault的类型来决定如何让这些Handle完成实际的映射。

Handle的具体实现细节就不一起说了，后面会多少提到一点。

# MapArea重构：
主要改动如下：
1. 将`data_frames`字段改写为`data_frames: BTreeMap<VirtPageNum, Page>`。
    * 对于内核地址空间一大片Identity Mapping的情况，我在`new_kernel`中特殊处理了，并不会被`MapArea`保存，所以不会因此浪费大量内存。
2. 增加字段记录映射的文件和偏移。
3. 删除`map`、`map_one`、`unmap`、`unmap_one`四个函数，`data_frames`的初始化在`new`中完成，并且在之后的生命周期中几乎不再改动：
   * 当MapArea被销毁时，这些Page都会被销毁，也就是unmap。
   * 当MapArea被创建时，实际上只是创建了对应了PTE和handle，内存的实际分配或映射都被推迟到page fault发生时。
   * 唯一的例外是Private模式映射的文件被写入时，会从FilePriv转换为Framed。但是此时MapArea的元数据仍然记录了文件和偏移，所以可以判断是Private映射的文件。
4. 增加函数`page_fault`用来处理page fault，逻辑有些复杂，大致是：
   * 首先检查page fault的原因，如果是访问权限有问题，则检查MapArea本身保存的权限位，如果不匹配则表示用户程序出错。
   * 如果逻辑上的权限没错，那么根据Page的不同variant和当前PTE的状态来判断应该怎样恢复。
     * 例如Framed的情况只有两种合法的page fault，分别是fork的COW和lazy allocation。

其他改动：
1. 实现`MapArea`的分割和合并。
2. 重写shrink和append，分别利用分割和合并的操作。
3. munmap用到了分割。
4. 分割发生后，未被MemorySet持有的部分会被drop，相关映射被无效化，映射拥有的页帧会被释放。

# lazy allocation和fork的COW

lazy allocation和cow都有一些需要额外注意的东西，例如特殊地址TRAP_CONTEXT_BASE那一页就必须立刻分配，fork时必须立刻复制。其他细节请见我之前的文章，这篇文章主要关注如何系统化地实现这些内存管理的特性。

在`MFrameHandle`和相关的Manager中有一个类型：

```Rust
enum MFrame {
    /// not yet allocated
    Lazy,
    /// fully owned, with all possible permissions
    Owned(FrameTracker),
    /// shared until write, without write permission
    COW(Arc<FrameTracker>),
}
```

根据注释，分别是：
* 尚未分配，当进程尝试访问该虚拟页时会分配并转变为`Owned`
* 已分配，并且被唯一一个进程的唯一一个虚拟页映射
* COW，由多个进程共享，当有一个进程尝试写入时会复制并转换为`Owned`

`MFrameHandle`有如下数个函数值得说明：
* `map_lazy(pte: *mut PageTableEntry) -> Self`，创建尚未分配的Handle
* `map_strict(pte: *mut PageTableEntry, flags: PTEFlags) -> Self`，立刻分配并映射
* `map_owned(pte: *mut PageTableEntry, frame: FrameTracker, flags: PTEFlags) -> Self`，映射到已分配的`FrameTracker`
* `load(&self, flags: PTEFlags)`，加载`Lazy`的情况，变为`Owned`
* `cown(&self)`，将COW转换为`Owned`
* `share_cow(&self, other: *mut PageTableEntry) -> MFrameHandle`，从`Owned`转换为`COW`并且将参数other映射到该页帧上
* `strict_dup(&self, other: *mut PageTableEntry) -> MFrameHandle`，从`Owned`复制到另一个分配的页帧上，并以`Owned`将other映射到该页帧

# 文件系统的修改

文件映射需要文件系统支持：
* 文件映射的页帧与block cache的IO syscall写入同步。
* Shared模式映射区域的写入不能改变文件的原始大小，但是`Inode`原本的写入函数是默认扩张文件的。

第一点不需要修改文件系统，只需要在`OSInode::write`的最后加上通知MFile的调用就行，MFile接到通知应该立刻从文件读取被修改的内容。实际上是从BlockCache读取，因为MFile的读写都经过BlockCache。并且因为这个原因我们也不需要BlockCache从MFile的页帧上同步内容。

第二点只需要仿照`Inode`原本的`write_at`，去掉`increase_size`即可。

# 文件映射

和上面的`MFrame`类似，文件映射也有类似的类型：

```Rust
#[derive(Clone)]
pub struct FilePos {
    inode: Arc<Inode>,
    offset: usize
}
struct MFile {
    pos: FilePos,
    frame: Mutex<Option<FrameTracker>>
}
```

`FilePos`表示一个文件的位置：
* 按约定offset必须以4K对齐，这是为了保证一个文件并不会被错位映射。
* 保存的是`Arc<Inode>`而不是`Arc<OSInode>`，因此文件映射即使在fd被close之后仍然有效，直到被unmap为止。

`MFile`表示一个驻留在内存中的文件页面：
* 对每一个独特的`FilePos`只有全局唯一的`MFile`实例，无论是Shared模式的映射还是Private模式但还未写入的映射。
* 多个不同的进程，甚至同一个进程的不同虚拟页，可以映射到同一个MFile，如此便可以实现共享内存。
* 初始时`frame`为`None`，当某个映射的虚拟页访问它时会被初始化，即分配页帧并从文件读入内容。
  * 注意`frame`被读入并不意味着所有映射的PTE都被设置好，实际上只有发生page fault时才会设置PTE，只是如果本来就已读入的话就无需再读入了。
* 当最后一个映射被unmap后，MFile会被drop，它持有的FrameTracker也会被释放。
* MFile的Manager保存一个反向映射，记录映射到一个MFile的所有PTE，如果页面被强行换出那么这些所有PTE都会被无效化，直到它们发生page fault时才会重新读入。


`MFileHandle`的一些函数：
* `map(pte: *mut PageTableEntry, inode: Arc<Inode>, offset: usize) -> Self`
* `load(&self, flags: PTEFlags)`，为文件页面分配页帧并读入
* `share_fully(&self, other: *mut PageTableEntry) -> Self`，复制文件映射，不提供flags是因为被调用时MFile可能还没有读入，所以不会立刻将other实际映射到页帧。主要用于进程的fork。
* `strict_dup(&self, frame: &FrameTracker)`，将已经读入的页帧完全复制到另一个页帧上，用于Private映射的写入发生时。

# Demand Paging

有了文件映射Demand Paging就好办多了：

* 加载ELF文件时，只需要载入64 bytes的ELF header(64位的情况)即可，从中解析出各个LOAD program header的文件偏移和长度。
* 以Private模式将LOAD program header的文件及偏移映射到进程的地址空间中，等到page fault发生时自然就会加载到内存中。
* LOAD program header的虚拟地址保证是4K的倍数，因此不用担心上面offset的问题。文件中大小并不是4K的倍数，我没有考虑多余区域的情况，不过不会影响程序的运行。
* 因为是文件映射实现的，所以同一个ELF文件的多个进程可以始终共享只读区域，节省内存。

不足之处：我没有实现文件的锁定，所以ELF文件被Private模式映射后，在可写区域被写入前，仍然会反映磁盘上(BlockCache中)的更改。只读区域更是会始终反映这些更改。这可能导致正在执行的代码或者数据区域发生变化，进而导致正在执行的进程崩溃。



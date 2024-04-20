---
title: OSCamp D8-rCore-Chapter 4 Address Translation
date: 2024-04-15 14:23:31
tags: [Rust, RISC-V]
category: [OSCamp]
---

# 地址转换
今天花了不少时间读spec，然后做了下面这张图(Sv39)。其中去掉了权限等等细节，只保留了地址转换的大框架。本节内容围绕这张图展开。

{% asset_img sv39.png Sv39 %}

## 物理地址、虚拟地址

* 物理地址简称PA，虚拟地址简称VA。
* PPN是Physical Page Number的缩写，Sv39中**所有PPN的长度都为44位**。
* VPN是Virtual Page Number的缩写，Sv39中**所有VPN的长度都为27位**。
* PAGESIZE是一个Page的大小，RV32和RV64的PAGESIZE都是4096(即4K)，长度为12位。

给定Page Number计算对应地址的算法是：

$$xA=xPN \times 4K\tag{1}$$

注意到乘以4K相当于左移12位，即：
* PA的长度为44+12=56位。
* VA的长度为27+12=39位，Sv39因此得名。

例如图的最上方是寄存器`satp`的结构，其中root的PPN是低44位，长度正好和描述的一致。

## 页表
页表是一个树状结构，在Sv39中除root外**最多只有三层**，从高到低记作level 2、level 1、level 0。root是页表的根节点，保存在`satp`的低44位，将它乘以4K即得到root下一层的物理地址。

从虚拟地址到物理地址的映射由MMU通过算法完成，该算法输入一个VA，从页表的root出发找到一条到叶子节点的路径，并根据该路径转换到PA。VA在这个过程中决定了该路径，如果不存在这样的路径，那么会出现page fault。

页表的项叫做PTE(Page Table Entry)。每一个PTE都有PPN字段，图中由并排的(26+9+9)三个矩形表示，长度刚好是44位，而完整的PTE长度是64位，没有标出的20位包含了页的访问权限等信息，我们暂时不考虑这些。

PTE有两种情况：
* 叶子节点：图中所有PPN带有涂色的PTE。包括一块到三块涂色，都是叶子节点。
* 内部节点：PPN三个块均无涂色的PTE。

## 转换算法

* 0层的叶子节点的2、1、0块PPN(共44位)都被涂成绿色，表示转换后的PA的，**高44位**被这44位填充，低12位直接从输入的VA的page offset复制。
* 1层的叶子节点只有2、1块PPN(共35位)被涂成蓝色，0块为0，表示转换后PA只有，**高35位**被涂色的部分填充，低12+9=21位直接从VA复制。
* 2层的叶子节点只有编号为2的块(共26位)被涂成橙色，表示PA只有**高26位**被涂色部分填充，低12+9+9=30位直接从VA复制。

注意！后两种情况被称为superpage，没有被涂色的部分必须为0，否则将会page fault。

内部节点没有涂色，此时整块PPN(44位)根据公式$(1)$得到的PA即是下一层页表的起始地址，MMU会根据VA.VPN计算出下一层的具体位置。

总结出来即是以下三条公式：

$$
\mathrm{pte}(\mathrm{ppn}, \mathrm{vpn}) = *(\mathrm{ppn}\times 4096+\mathrm{vpn}\times 8)\tag{2}
$$

$$
\begin{matrix}
    \mathrm{trans}(\mathrm{va}, \mathrm{ppn},i) =\left\{
    \begin{aligned}
        &\mathrm{let}\ \mathrm{pte'} = \mathrm{pte}(\mathrm{ppn}, \mathrm{va.vpn[\mathnormal{i}]})\ \mathrm{in}\\
        &\mathrm{pte'}.\mathrm{ppn[\mathnormal{i}:2]}\oplus\mathrm{va.vpn[0:\mathnormal{i}-1]\oplus\mathrm{va.pgoff}}&&,\mathrm{if\ pte'\ is\ leaf} \\
        &\mathrm{trans(va,pte'.ppn, \mathnormal{i}-1)}&&,\mathrm{otherwise}
    \end{aligned}\right.
\end{matrix}\tag{3}
$$

$$
\mathrm{pa(va)} = \mathrm{trans(va,SATP.ppn,2)}\tag{4}
$$

运算符$\oplus$表示将bit从高位到地位相连。

* 公式$(4)$是初始条件，传入页表根节点的PPN。
* 公式$(2)$根据页表的PPN和VA的某块VPN，计算PTE的位置，并解引用返回PTE。对于初始条件，函数`pte`会返回level 2的节点。
* 公式$(3)$是递归定义，由于递归不是很深，读者可自行验证正确性。

此外，公式没有覆盖出错的情况，例如如果在`i=0`时仍然不是叶子节点，依spec所述会发生page fault。如果需要完整的细节，请参考spec原文。

该算法可以直接迁移到其他分页机制上，其中有几个常量可能需要改变：
* 4096(bytes)，即4K，RV32和RV64的页大小都是4K，无需更改。
* 8(bytes)，即PTE的长度，Sv32是4，本文的Sv39及其他RV64的分页机制都是8。
* 2，即层数减去1，Sv39的层数是3所以这里是2。如果是Sv32那么就是1。如果是Sv48那么就是3。具体请参见文档。

# ASID和TLB
ASID是寄存器`satp`的一个字段，在RV64中长度为16位，全称为Address Space Identifier。

TLB全称为Translation Lookaside Buffer，作用是缓存最近的地址转换结果来加速后续转换。TLB是**不在RAM上的**一块储存空间，每个hart都有自己独特的TLB。TLB中缓存的记录也包括对应的ASID。

更新页表之后需要用指令`SFENCE.VMA`刷新**当前**hart的TLB，该指令也可以通过两个寄存器operand指定某个ASID和某个虚拟地址只让TLB的部分记录无效化，避免刷新整个TLB。

注意！据spec所述，并不是所有RISC-V实现都支持ASID；即便支持，也不一定16个位都可用。在不支持ASID的实现上只能使用编号为0的ASID。一般一个ASID对应一个进程，但是ASID的总数有限，要运行更多的进程必须复用ASID，这个机制叫做ASID Recycling。在只有0号ASID的实现上每次切换进程都要把整个TLB刷新，相当于不断复用这一个ASID。

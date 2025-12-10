---
title: RISCV-Svvptc扩展
date: 2025-12-08 13:38:32
modify: 2025-12-08 13:38:32
author: days
category: RISCV
published: 2025-12-08
draft: false
---

# RISCV-Svvptc 扩展

![image.png](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/20251208220958.png)

## 1. 解析
- 全称：Supervisor virtual-address translation prefetch/tlb coherence
- 翻译
```
第 17 章 “Svvptc” 扩展：用于在标记页表项（PTE）为有效后省去内存管理指令，版本 1.0
当实现了 Svvptc 扩展时，如果一个 Hart 通过显式存储指令将叶子或非叶子页表项（PTE）的有效位（Valid bit）从 0 更新为 1，并且该修改对该 Hart 可见，那么该修改终将在一个有界的时间范围内，被该 Hart 后续对这些 PTE 的隐式访问所感知。
ℹ️ 
Svvptc 免除了操作系统执行某些内存管理指令（如 SFENCE.VMA 或 SINVAL.VMA）的义务，这些指令通常用于在驻留内存的 PTE 从无效（Invalid）变为有效（Valid）时，同步 Hart 的地址转换缓存。
对于驻留内存的 PTE 的其他形式的更新（包括当 PTE 从有效变为无效时）来同步 Hart 的地址转换缓存，仍然需要使用适当的内存管理指令。
Svvptc 保证 PTE 从无效到有效的更改在有界时间内可见，从而使这些内存管理指令的执行变得多余（Redundant）。省略这些指令带来的性能收益，超过了可能发生的偶尔的、无谓的额外缺页异常的成本。
根据微架构的不同，促进 Svvptc 实现的一些可能方法包括：
    没有任何地址转换缓存；
    不在地址转换缓存中存储无效（Invalid）的 PTE；
    使用有界计时器自动驱逐无效的 PTE；
    或者使地址转换缓存与修改 PTE 的存储指令保持一致性（Coherent）。
```
- 简要解释：
硬件给操作系统的保证。硬件保证：当软件将页表项（PTE）从无效（Invalid, V=0）修改为有效（Valid, V=1）时，硬件会在有界的时间内自动感知到这一变化，而无需软件显式执行刷新指令（`SFENCE.VMA`）。

- 举个例子：
也就是说，当 hartA 和 hartB 的 TLB 中同时存在某一个 Invalid 的页表项，当 hartA 给这个页表项分配内存，将页表项从 Invalid 修改为 Valid 的时候，hartB 在有界的时间内可以自动感知到这个页表项变化。

*   适用场景：仅限 内存分配 (Map) 场景（$V=0 \to 1$）。
*   不适用场景：内存回收 (Unmap) 或修改映射（$V=1 \to 0$ 或修改 PPN）。这些场景仍必须使用 `SFENCE.VMA`。

## 2. 解决的问题

在标准 RISC-V 规范（无 Svvptc）中，硬件允许 TLB 缓存“无效条目”

*   旧现状：OS 分配了新内存，但 TLB 里可能还记着“这里是空的”。为了安全，OS 必须执行 `SFENCE.VMA`（预防性刷新）。
*   代价：`SFENCE.VMA` 是一条串行化指令，会清空流水线和 TLB，开销巨大。在系统启动或高频内存分配（如 `vmalloc`）时，这会导致严重的性能损耗。

总结：省去昂贵的 `SFENCE.VMA` 这个刷 TLB 的操作。

假设场景：

- Core A：修改了内核页表（分配了一个新页面，Invalid -> Valid）。
- Core B：需要同步这个变化。
### 没有 svvptc

流程：

1. Core A：修改页表。
2. Core A：发送 IPI（核间中断） 给 Core B。
3. Core B：收到中断，暂停手头工作，进入中断处理函数。
4. Core B (关键点)：<u>执行 sfence.vma 指令</u>。
    - 代价： sfence.vma 会清空流水线，清空 TLB，导致后续指令重新取指、重新查表。这是一条很“重”的指令。
5. Core B：回复 Core A "做完了"，然后返回。
### 有 svvptc

流程：

1. Core A：修改页表。
2. Core A：发送 IPI（核间中断） 给 Core B。/ 或者有的实现下面直接省略掉这部分
    - (注：在 https://lore.kernel.org/all/20240717060125.139416-1-alexghiti@rivosinc.com/ 实现的中，这一步甚至变成了被动的缺页异常，连 IPI 都省了)
3. Core B：收到中断（或异常），进入处理函数。
4. Core B (关键点)：什么都不做 (执行 NOP)。 (Core B: 知道了，但反正硬件会自动更新，我不用执行sfence.vma刷。)
    - 代价： 0。Core B 不需要执行那条昂贵的刷新指令。
5. Core B：直接返回。
## 3. 硬件原理

规范里面列出了 svvptc 的几种实现方式：

1. 根本不使用 TLB
2. TLB 中不存无效条目：
    *   TLB 只存 Valid=1 的记录。
    *   当 OS 修改内存为 Valid 后，TLB 因为没存旧的 Invalid，下次访问直接 Miss，去查物理内存，自然读到最新的 Valid。
3.  使用有界计时器自动驱逐 Invalid 条项 ：
    *   允许存 Invalid，但给它设极短的寿命（如 100ns）。过期自动删除。
4.  硬件一致性/监听 ：
    *   TLB 控制器时刻监听（Snoop）数据总线或 Store Buffer，发现页表变了立刻更新。

硬件不保证修改马上生效，但保证在“有界时间”内生效。这个时间通常远小于操作系统处理一次异常的时间。

## 4. 一些问题

关于 https://lore.kernel.org/all/20240717060125.139416-1-alexghiti@rivosinc.com/ 的问题

Q1: 既然硬件承诺自动更新，为什么还会触发异常？

A: 因为硬件承诺的是“有界时间”更新，而不是“原子级瞬间”更新。

如果在软件改完页表的瞬间（纳秒级窗口内），或者因为 CPU 乱序/预取，CPU 抢在硬件同步完成前访问了地址，就会触发“无谓的缺页异常 (Gratuitous Page Fault)”。

*   处理策略：内核识别出这是假异常，直接让 CPU 重试。第二次访问时，硬件肯定同步好了。

 Q2: 重试一次一定能成功吗？会不会死循环？

A: 一次就够，不会死循环。

因为操作系统进入异常处理、保存寄存器、查位图这套流程的耗时（微秒级），远远大于硬件同步所需的“有界时间”（纳秒级）。当执行 `sret` 返回重试时，硬件早就准备好了。

***

 Q3: Svvptc 会自动把更新广播给其他核心吗？

A: 不会主动推送。

Svvptc 是“被动”生效的。

*   它不负责跨核通知（这是数据 FENCE 和 软件位图/IPI 的事）。
*   它只保证：当 Core B 决定去内存里看一眼时（比如重试访问），它一定能看到 Core A 写进去的新值，而不会被它自己 TLB 里的旧垃圾挡住视线。

Q4: 既然有了 Svvptc，还需要 `FENCE` 指令吗？

A: 需要，且不能省。

*   `SFENCE.VMA` / Svvptc：管的是 TLB（地址转换） 的一致性。
*   `FENCE`：管的是 Store Buffer（数据写入） 的顺序和可见性。
*   Core A 修改页表后，必须执行 `FENCE` 确保数据真正落盘（Global Visible）。Svvptc 只能保证数据落盘后，TLB 能读到它。

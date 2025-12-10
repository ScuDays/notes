---
title: SFENCE.VMA
date: 2025-12-08 13:30:26
modify: 2025-12-08 13:30:26
author: days
category: RISCV
published: 2025-12-08
draft: false
---

# SFENCE.VMA
## 参考手册

riscv-privileged-Chapter 12 Supervisor-Level

![riscv-privileged-Chapter 12 Supervisor-Level ISA Version 113-123-149_dual_智谱4Flash-part0-page-1.png](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/riscv-privileged-Chapter%2012%20Supervisor-Level%20ISA%20Version%20113-123-149_dual_%E6%99%BA%E8%B0%B14Flash-part0-page-1.png)

![riscv-privileged-Chapter 12 Supervisor-Level ISA Version 113-123-149_dual_智谱4Flash-part0-page-2.png](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/riscv-privileged-Chapter%2012%20Supervisor-Level%20ISA%20Version%20113-123-149_dual_%E6%99%BA%E8%B0%B14Flash-part0-page-2.png)

![riscv-privileged-Chapter 12 Supervisor-Level ISA Version 113-123-149_dual_智谱4Flash-part0-page-2.png](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/riscv-privileged-Chapter%2012%20Supervisor-Level%20ISA%20Version%20113-123-149_dual_%E6%99%BA%E8%B0%B14Flash-part0-page-3.png)

![riscv-privileged-Chapter 12 Supervisor-Level ISA Version 113-123-149_dual_智谱4Flash-part0-page-4.png](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/riscv-privileged-Chapter%2012%20Supervisor-Level%20ISA%20Version%20113-123-149_dual_%E6%99%BA%E8%B0%B14Flash-part0-page-4.png)

## 解析 (根据手册翻译解读)

### 指令定义与目的

- 功能：用于同步内存中页表数据结构的更新（软件写）与硬件当前的执行（MMU 读）。
    
- 解决的矛盾：解决“显式存储”（Explicit Stores，即 CPU 写页表）与“隐式访问”（Implicit Accesses，即 MMU 查页表）之间的顺序问题。
    
- 主要有两个作用：
    
    1. 排序 (Ordering)：保证先执行的写操作在后执行的查表操作之前生效，避免执行乱序。
        
    2. 失效 (Invalidation)：使本地 Hart 的地址转换缓存（TLB）失效。
        
- 命名哲学：为什么叫 Fence 而不叫 Flush？因为它强调的是指令执行的顺序语义，而不仅仅是清空缓存的动作。
 ![image.png](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/20251209161433.png)

***
> [!NOTE]
> 

解析图片第一段的内容

1. 核心矛盾：显式访问 vs 隐式访问

> 原文： “指令执行会导致对这些数据结构的隐式读取和写入；然而，这些隐式引用通常与显式加载和存储的顺序无关.....指令中对
内存管理数据结构的隐式引用之前”  

- 显式访问 ：
    - 软件（操作系统）做的。
    - 比如 OS 执行了一条指令，修改了内存里的页表项（PTE）。
- 隐式访问 (Implicit Access)：
    - 硬件（MMU ）做的。
    - 当 CPU 执行任何一条涉及内存的指令（如 ld, sw, 取指）时，硬件把虚拟地址翻译成物理地址，自动去读取内存里的页表。

问题：在现代高性能 CPU 中，为了速度，硬件的“显式执行单元”和“隐式翻译单元”往往是独立工作的，甚至是乱序的。硬件默认不保证：当你用显式指令改了页表后，隐式单元能立马看到这个修改。

例子：TLB 中存在一条 invalid 的条目，软件给这个条目分配内存，然后马上访问这个内存，但MMU 可能因为乱序或者缓存，查到的还是旧页表，报错说没分配，导致一些奇怪的问题。

2. 作用
- 两个作用：
    1. 排序 (Ordering)：保证前的写操作在后的查表操作之前生效。
    2. 失效 (Invalidation)：使本地 Hart 的地址转换缓存（TLB）失效。

***
> [!NOTE]
> 解析图片感叹号的内容  
- 命名哲学：为什么叫 Fence 而不叫 Flush？因为它强调的是指令执行的顺序语义，而不仅仅是清空缓存的动作。
### 作用域和多核一致性

![image.png](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/20251209184100.png)

- 本地性：明确指出 SFENCE.VMA 仅对当前执行它的 Hart（核心）有效。
    
- TLB Shootdown：解释了在多核系统中，如果修改了共享页表，软件必须负责通知其他核心（通常通过 IPI 核间中断），让它们也执行 SFENCE.VMA。（可以通过 svvptc 来避免这个的开销）
### 指令的四种模式

第 130 页和第 131 页上部分，省略手册具体内容。

- 全域刷新 (rs1=x0, rs2=x0)：刷所有地址、所有 ASID。
- 指定 ASID 刷新 (rs1=x0, rs2≠x0)：刷指定进程的所有页。
- 指定地址刷新 (rs1≠x0, rs2=x0)：刷指定地址（忽略 ASID，通常用于全局/内核映射）。
- 精确刷新 (rs1≠x0, rs2≠x0)：刷指定进程的指定地址（最高效）。
- 过度刷新 (Over-fencing)：允许硬件“偷懒”。如果硬件不支持细粒度刷新，可以将所有指令都视为“全域刷新”来执行，这是合法的。
### 硬件行为规范

第 131 页和第 132 页部分，不是我们所关心的内容，省略。

### 指令使用指南

手册指导了什么时候需要使用 SFENCE. VMA 什么时候不需要。

1. 免刷
	- 权限位：sstatus.SUM (内核读用户), sstatus.MXR (可执行即刻读)。
	- 模式切换：satp.MODE (开启/关闭分页)。
	- 进程切换：satp.ASID (修改当前 ASID)。
2. 需要使用 SFENCE. VMA 刷新
*   回收 ASID：把旧 ASID 分给新进程用时 $\to$ `sfence.vma x0, asid`。
*   无 ASID 硬件：每次切页表 $\to$ `sfence.vma x0, x0` (全刷)。
    *   注：若保留内核映射，可设 rs2=0 (非 x0) 以保护 Global 项。
*   改目录表 (Non-leaf)：影响大范围映射 $\to$ `sfence.vma x0, asid`。
*   改页表项 (Leaf)：影响单页映射 $\to$ `sfence.vma vaddr, asid`。
*   涉及全局 (Global)：如果 PTE 的 G=1 $\to$ `rs2` 必须设为 `x0`。

![image.png](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/20251209192359.png)

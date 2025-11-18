---
title: RISC-V--Svadu-Svade
date: 2025-11-18 19:27:23
modify: 2025-11-18 19:27:23
author: days
category: ICSAS
published: 2025-11-18
draft: false
---
# RISC-V--Svade-Svadu

阅读材料：

- https://docs.riscv.org/reference/isa/priv/supervisor.html#sec:svadu

- riscv 特权指令集 Chapter 16. "Svadu" Extension for Hardware Updating of A/D Bits, Version 1.0

## 硬件手册阅读

![image.png](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/20251118193033.png)

### Svade

riscv-privileged 手册 12.3.1. Addressing and Memory Protection 

![image.png](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/20251118195842.png)

![image.png](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/20251118195908.png)

### Svadu

![image.png](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/20251118200050.png)

![image.png](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/20251118200133.png)

![image.png](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/20251118200145.png)

![image.png](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/20251118200154.png)

## 核心目标

与ARM DBM完全相同：解决因软件管理页表A/D位而产生的页错误异常所带来的巨大性能开销。

Svadu 可以认为是 Svade 扩展的一个超集，同时可以向下兼容成 Svade。

## 工作原理 

RISC-V 的哲学是“提供选择权”，它定义了两种互斥的A/D位管理方案，并让软件（特别是Hypervisor）来决定在特定场景下使用哪一种。

*   方案一: `Svade` (基于异常)
    *   当CPU尝试访问一个 `A` 位为 `0` 或写入一个 `D` 位为 `0` 的页面时，硬件必须产生一个页错误异常。
    *   硬件只产生页面错误，软件负责解决实际的问题（在异常处理程序中手动修改PTE）。
    *   优点: 软件可及时知道异常
    *   缺点:每次必须产生异常退出进行处理，性能开销大。

*   方案二:  `Svadu`  (基于硬件自动更新) 
    *   当CPU访问 `A=0` 或写入 `D=0` 的页面时，硬件不产生异常。
    *   相反，硬件会自动、原子地更新内存中的PTE，将对应的A/D位置 `1`。
    *   优点: 性能极高，对程序透明。
    *   缺点: 软件无法立即得知访问事件，需要后续主动扫描。

*   `Svadu` 扩展:
    *   `Svadu` 是一个更高级的扩展，它同时包含了“硬件自动更新”的能力，并提供了一个开关来决定是启用此能力，还是退回到并模拟 `Svade` 的行为。

## 3. 权限控制

RISC-V 的控制体系是分层的，从最高权限的M-Mode到底层的PTE，一层一层授权。

A. M-Mode 控制寄存器 (最高授权)

*   `menvcfg.ADUE` (Machine Environment Configuration Register):
    *   这是由最高权限的M-Mode（固件）控制的，设置。
    *   `ADUE=1`: 授权整个系统可以使用硬件自动更新功能。
    *   `ADUE=0`: 禁止硬件自动更新功能。此时，`henvcfg.ADUE` 会被强制为0，整个系统只能工作在 `Svade` 模式。

B. Hypervisor 控制寄存器 (策略配置)

*   `henvcfg.ADUE` (Hypervisor Environment Configuration Register):
    *   这是由Hypervisor控制的、针对单个 guest os 是否为它使用  `Svadu` 的对应控制。
    *   `ADUE=1`: 为该虚拟机启用硬件自动更新模式。
    *   `ADUE=0`: 为该虚拟机强制启用基于异常的模式（即模拟`Svade`行为）。

C. 页表项 (PTE) 标志位 (状态载体)

*   `A` (Accessed) 位 和 `D` (Dirty) 位:
    *   RISC-V 在PTE中有专用的A/D位。
    *   这些位是状态的载体。无论是软件（在`Svade`模式的异常处理中）还是硬件（在自动更新模式中），最终都是为了将这两个位从 `0` 修改为 `1`。

## 4. 软件职责

1.  M-Mode 软件 (固件): 负责进行最高级别的授权，通过设置 `menvcfg.ADUE` 来决定是否允许Hypervisor使用此高级功能。
2.  Hypervisor 软件 (KVM):
    *   决策: 
	    * 根据场景决策。在虚拟机正常运行时，设置 `henvcfg.ADUE = 1` 以追求高性能。
	    * 在需要进行热迁移、精确追踪脏页时，设置 `henvcfg.ADUE = 0` 来强制虚拟机回退到通过产生页面错误可及时跟踪只读页表写入。
    *   执行: 根据所选的模式，执行不同的操作：
        *   在 `Svade` 模式下，需要捕获注入到Guest的页错误。
        *   在 `Svadu` 在硬件更新模式下，需要主动扫描Guest的页表以检查A/D位的变化。

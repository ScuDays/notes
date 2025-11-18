---
title: DBM--arm64硬件脏页控制
date: 2025-11-11 14:47:27
modify: 2025-11-11 14:47:27
author: days
category: ICSAS
published: 2025-11-11
draft: false
---
# DBM--arm64硬件脏页控制

## DBM-硬件手册阅读

阅读ARMv8-Reference-Manual，可以看到D4-2068 页有对于 DBM 的介绍

![image.png](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/20251111151231.png)

Armv8.1引入了处理器管理一个块页的dirty状态的能力。dirty状态记录着该块或页是否已经被写入。这是有用的，因为如果一个块页被踢出，dirty状态会通知管理软件，RAM的内容是否需要被写回到storage（根存储位置）。例如，一个文件从磁盘load到RAM。当它晚些时候从memory移除时，OS需要知道RAM里的内容是否比磁盘里的内容更新。如果是，那么磁盘的内容就需要被更新。如果不是，就直接丢弃。当管理dirty状态被打开时，软件创建转换列表时会把访问权限设置为只读并且DBM(dirty bitmodifier)置位。如果该页被写入，硬件自动将访问权限更新为可读写，并且不会导致权限异常

![页表控制位(D4-2066)](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/20251111151028.png)

![DBM-具体信息 -(D4-2067)](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/20251118190321.png)

![image.png](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/20251118192409.png)

![image.png](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/20251118192518.png)

![image.png](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/20251118192526.png)

![image.png](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/20251118192547.png)

![image.png](https://days-notes.oss-cn-shenzhen.aliyuncs.com/img/20251118192617.png)

## 核心目标

ARM DBM 机制解决因追踪内存“脏页”（被写入的页面）而产生的性能瓶颈。以往，操作系统或 Hypervisor 通过将页面设置为“只读”，通过访问时候产生的页面错误来捕获首次写入的操作，但这会触发代价高昂的页错误异常。DBM 的核心目标就是通过硬件自动化来规避这些异常，从而在不牺牲功能的前提下，大幅提升写密集型应用（尤其是在虚拟化热迁移场景下）的性能。

## 原理 

DBM 机制的规避异常指的是：软件不再被动地等待页面错误的发生，而是主动地“授权”硬件在特定条件下自动处理本来应该发生的页面错误。

工作流程简述：

- 软件设置: Hypervisor 在追踪脏页时，会修改 Stage 2 页表项（PTE），将其权限设置为只读，同时将 DBM 设为 1
- 虚拟机写入 : Guest VM 执行一条写入指令，访问该页面。
- 硬件自动化处理 :
	- 硬件MMU在地址翻译时，发现是写入只读页。
	- 此时DBM=1 
	- 于是，不会产生页错误异常。
	- 取而代之，它会执行一次原子的“读-改-写”操作，直接修改内存中的那个PTE，将页面的权限从“只读”更新为“可写”。
	- 原始的写入指令得以顺利完成。

- 软件收集脏页: 稍后，Hypervisor主动扫描页表。通过检查页面权限是否已从它最初设置的“只读”变为了“可写”，就能得知这个页面已经被写入，即为“脏页”。

## 关键寄存器

A. 系统寄存器 (全局开关)

这些寄存器负责在整个地址翻译阶段（Stage 1 或 Stage 2）全局启用硬件脏状态管理。

- TCR_EL1.HD, TCR_EL2.HD, TCR_EL3.HD:
	- 用于Stage 1 翻译（由OS管理）。HD (Hardware Dirty management) 位为 1 时启用。
 - VTCR_EL2.HD:
	 - 用于Stage 2 翻译（由Hypervisor管理）。这是KVM等虚拟化场景中最关键的开关。
- 前置条件:
	- 架构规定，要启用 HD 位，必须首先启用对应的 HA (Hardware Access flag management) 位。即，硬件脏状态管理依赖于硬件访问标志管理。

B. 页表项 (PTE) 标志位 (单页开关)

这些标志位位于最终级别的页表描述符中，对单个页面进行精细控制。

- DBM (Dirty Bit Modifier) 位:
	- 这是ARM机制的核心标志。当它为 1 时，表示该页面适用硬件脏状态管理。它就是软件给硬件的“授权信号”。
- 权限位 (作为状态指示器):
	- AP[2] (Access Permission): 在 Stage 1 PTE 中，AP[2]=1 表示只读。硬件会自动将其修改为 0。
	- S2AP[1] (Stage 2 Access Permission): 在 Stage 2 PTE 中，S2AP[1]=0 表示只读。硬件会自动将其修改为 1（在Stage 2中，1代表可写）。

## 软件使用 DBM 跟踪流程

- 初始化: 在系统启动时，通过修改 VTCR_EL2 等系统寄存器，启用硬件管理功能。

- 脏页跟踪: 在开始脏页追踪时，遍历相关页表，将目标页表设置为 DBM=1 并设为只读权限。

- 查询: 在需要获取脏页列表时，再次遍历页表，检查只读位是否被硬件修改为可写。

-  重置 : 收集完脏页信息后，将 PTE 的权限再次设回只读，为下一轮追踪做准备。
- 内存排序: 必须注意，硬件对 PTE 的更新与程序流中的其他内存访问没有严格的顺序保证。如果需要确保 PTE 的更新对后续操作可见，软件必须使用 DSB (数据同步屏障) 指令。


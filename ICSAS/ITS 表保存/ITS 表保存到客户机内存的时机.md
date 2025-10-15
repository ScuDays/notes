---
title: ITS 表保存到客户机内存的时机
date: 2025-09-01 16:41:13
modify: 2025-09-01 16:41:13
author: days
category: ICSAS
published: 2025-09-01
draft: false
---

ITS 表保存到客户机内存的时机非常明确。

## ITS 表保存到客户机内存的触发时机

### 1. 核心触发函数

Readarm_gicv 3_its_kvm. c L 75-90

### 2. 具体触发条件

关键判断条件：if (running) - 只有当 VM 停止运行时才会保存 ITS 表。

具体的触发时机包括：

#### A. 热迁移开始时

c

Apply to spice-core. c

// 迁移流程中 VM 状态变化：

// RUNNING -> STOP (for migration)

#### B. VM 暂停时

c

Apply to spice-core. c

// 用户手动暂停 VM：

// (qmp) stop

// RUNNING -> PAUSED

#### C. VM 关机时

c

Apply to spice-core. c

// VM 正常关机：

// RUNNING -> SHUTDOWN

#### D. 其他停止状态

- RUN_STATE_SAVE_VM - 保存 VM 状态时
- RUN_STATE_FINISH_MIGRATE - 完成迁移时
- RUN_STATE_POSTMIGRATE - 迁移后状态

### 3. 注册时机和生命周期

Readarm_gicv 3_its_kvm. c L 115-130

注册条件：只有当内核支持 ITS 迁移时（即支持 KVM_DEV_ARM_VGIC_GRP_CTRL），才会注册这个状态变化处理函数。

### 4. 为什么选择 VM 停止时保存

让我查看注释中的重要说明：

Readarm_gicv 3_its_kvm. c L 130-140

关键原因：

> "ITS tables are flushed into guest RAM separately and earlier, through the VM change state handler, since at the moment pre_save () is called, the guest RAM has already been saved."

时序问题：

1. VM 运行阶段：ITS 表数据在内核中动态更新
2. VM 停止：触发 vm_change_state_handler () 保存表数据到客户机 RAM
3. 迁移阶段：客户机 RAM 被保存到迁移流
4. 设备保存：pre_save () 只保存寄存器（此时 RAM 已保存完毕）

### 5. 具体的时序图

### 6. 总结

ITS 表保存到客户机内存的时机：

1. 触发条件：VM 从运行状态变为任何停止状态
2. 具体时机：

- 热迁移开始时 VM 暂停
- 用户手动暂停 VM
- VM 关机
- 任何导致 VM 停止运行的事件

1. 保存内容：

- Device Table（设备表）
- Collection Table（集合表）
- Interrupt Translation Tables（中断转换表）

1. 保存位置：客户机 RAM 中预先分配的表区域
2. 时序重要性：必须在迁移保存 RAM 之前完成，这样表数据就会被包含在迁移的 RAM 副本中

这种设计确保了 ITS 的所有状态（表数据在 RAM 中，寄存器在迁移流中）都能正确保存和恢复。
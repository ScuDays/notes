---
title: 调用mark_page_dirty_in_slot的函数路径
date: 2025-09-09 10:48:06
modify: 2025-09-09 10:48:06
author: days
category: ICSAS
published: 2025-09-09
draft: false
---

## 所有最终调用`mark_page_dirty_in_slot`的函数路径

### 1. 直接调用路径
#### 1.1 通用内存写入函数
```c
// 基础页写入
__kvm_write_guest_page() → mark_page_dirty_in_slot()

// 缓存写入
kvm_write_guest_offset_cached() → mark_page_dirty_in_slot()

// 高层写入函数调用链：
kvm_write_guest() → kvm_write_guest_page() → __kvm_write_guest_page() → mark_page_dirty_in_slot()

kvm_vcpu_write_guest() → kvm_vcpu_write_guest_page() → __kvm_write_guest_page() → mark_page_dirty_in_slot()

kvm_write_guest_cached() → kvm_write_guest_offset_cached() → mark_page_dirty_in_slot()

kvm_clear_guest() → kvm_write_guest_page() → __kvm_write_guest_page() → mark_page_dirty_in_slot()
```

#### 1.2 通过中间函数调用
```c
// 通用脏页标记
mark_page_dirty() → mark_page_dirty_in_slot()

kvm_vcpu_mark_page_dirty() → mark_page_dirty_in_slot()

// 页面映射和取消映射
kvm_vcpu_unmap() → kvm_vcpu_mark_page_dirty() → mark_page_dirty_in_slot()

// Guest物理地址缓存
kvm_gpc_mark_dirty_in_slot() → mark_page_dirty_in_slot()
```
### 2. 架构特定调用路径
#### 2.1 RISC-V架构
```c
// PMU (性能监控单元)
vcpu_pmu.c:
  kvm_riscv_vcpu_pmu_snapshot_set_shmem() → kvm_vcpu_write_guest() → ... → mark_page_dirty_in_slot()
  kvm_riscv_vcpu_pmu_snapshot_read() → kvm_vcpu_write_guest() → ... → mark_page_dirty_in_slot()

// SBI STA (SBI Steal Time Accounting)  
vcpu_sbi_sta.c:
  kvm_sbi_sta_steal_time_set_shmem() → kvm_vcpu_write_guest() → ... → mark_page_dirty_in_slot()
  kvm_sbi_sta_steal_time_update() → kvm_vcpu_mark_page_dirty() → mark_page_dirty_in_slot()

// 内存管理
mmu.c:
  gstage_page_fault() → mark_page_dirty() → mark_page_dirty_in_slot()
```

### 3. 宏定义和内联函数路径
```c
// 通用宏
include/linux/kvm_host.h:
  kvm_put_guest() → mark_page_dirty() → mark_page_dirty_in_slot()
```
---
title: qemu_add_vm_change_state_handler(vm_change_state_handler, s)
date: 2025-09-01 16:35:06
modify: 2025-09-01 16:35:06
author: days
category: ICSAS
published: 2025-09-01
draft: false
---

一旦调用了 `qemu_add_vm_change_state_handler(vm_change_state_handler, s)`，每次 VM 状态发生变化时，都会调用注册的 `vm_change_state_handler` 函数。

## 详细的触发机制

### 1. 注册过程（第 122 行）

```
// hw/intc/arm_gicv3_its_kvm.c:122
qemu_add_vm_change_state_handler(vm_change_state_handler, s);
```

这行代码将 ITS 设备的 `vm_change_state_handler` 函数注册到全局的 VM 状态变化监听列表中。

### 2. 每次状态变化都会触发

无论 VM 状态如何变化，都会调用这个处理器：

#### A. VM 启动时（`running = true`）

```
// VM状态：RUN_STATE_PAUSED -> RUN_STATE_RUNNING
vm_change_state_handler(s, true, RUN_STATE_RUNNING)
{
    if (running) {
        return;  // 直接返回，不做任何操作
    }
}
```

#### B. VM 暂停时（`running = false`）

```
// VM状态：RUN_STATE_RUNNING -> RUN_STATE_PAUSED
vm_change_state_handler(s, false, RUN_STATE_PAUSED)
{
    if (running) {
        return;
    }
    // 执行ITS表保存
    kvm_device_access(..., KVM_DEV_ARM_ITS_SAVE_TABLES, ...);
}
```

#### C. 热迁移停机时（`running = false`）

```
// VM状态：RUN_STATE_RUNNING -> RUN_STATE_FINISH_MIGRATE
vm_change_state_handler(s, false, RUN_STATE_FINISH_MIGRATE)
{
    if (running) {
        return;
    }
    // 执行ITS表保存
    kvm_device_access(..., KVM_DEV_ARM_ITS_SAVE_TABLES, ...);
}
```

#### D. VM 关机时（`running = false`）

```
// VM状态：RUN_STATE_RUNNING -> RUN_STATE_SHUTDOWN
vm_change_state_handler(s, false, RUN_STATE_SHUTDOWN)
{
    // 同样会执行保存操作
}
```

### 3. 智能过滤机制

虽然每次状态变化都会调用，但函数内部有智能过滤：

```
// hw/intc/arm_gicv3_its_kvm.c:75-90
static void vm_change_state_handler(void *opaque, bool running, RunState state)
{
    GICv3ITSState *s = (GICv3ITSState *)opaque;
    Error *err = NULL;

    // 关键过滤：只在VM停止时才执行保存操作
    if (running) {
        return;  // VM启动/运行时，直接返回
    }

    // 只有在VM停止时才保存ITS表
    kvm_device_access(s->dev_fd, KVM_DEV_ARM_VGIC_GRP_CTRL,
                      KVM_DEV_ARM_ITS_SAVE_TABLES, NULL, true, &err);
    if (err) {
        error_report_err(err);
    }
}
```

### 4. 实际的调用频率

![](https://cdn.nlark.com/yuque/__mermaid_v3/fba0d553771ef40d85efdd824d044948.svg)

### 5. 总结

- 触发频率：每次 VM 状态变化都会触发
- 执行频率：只有在 VM 从运行状态变为停止状态时才真正执行保存操作
- 设计优势：

- 确保不遗漏任何需要保存状态的时机
- 通过 `running` 参数高效过滤不需要处理的情况
- 自动适应所有可能的 VM 停止场景（暂停、迁移、关机等）

这种设计既保证了全覆盖（不会遗漏任何停机场景），又保证了高效率（运行时快速返回，不影响性能）。
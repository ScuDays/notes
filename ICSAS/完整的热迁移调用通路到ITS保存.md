---
title: 完整的热迁移调用通路到ITS保存
date: 2025-09-01 16:36:05
modify: 2025-09-01 16:36:05
author: days
category: ICSAS
published: 2025-09-01
draft: false
---

我来为你详细梳理从热迁移触发到最终调用 `vm_change_state_handler` 的完整函数调用通路：

## 完整的热迁移调用通路详解

现在我为你梳理从热迁移触发到最终调用 `vm_change_state_handler` 的完整函数调用通路：

### 1. 用户空间入口层 (QEMU)

```c
// 用户执行命令
(qmp) migrate "tcp:target_host:port"
        ↓
// QMP命令处理 - migration/migration.c:2040-2127
qmp_migrate(const char *uri, ...)
{
    // 解析URI和参数
    // 建立迁移连接
    socket_start_outgoing_migration(s, saddr, &local_err);  // 第2099行
}
```

### 2. 迁移连接建立层

```c
// socket连接建立后回调 - migration/migration.c:3672
migrate_fd_connect(MigrationState *s, Error *error_in)
{
    // 迁移准备工作
    // ...
    
    // 关键：停止VM - 第3750行
    if (migrate_mode_is_cpr(s)) {
        ret = migration_stop_vm(s, RUN_STATE_FINISH_MIGRATE);
    }
    
    // 启动迁移线程 - 第3761行
    qemu_thread_create(&s->thread, "live_migration", migration_thread, s, ...);
}
```

### 3. VM 停止处理层

```c
// VM停止函数 - migration/migration.c:193
static int migration_stop_vm(MigrationState *s, RunState state)
{
    migration_downtime_start(s);
    s->vm_old_state = runstate_get();
    global_state_store();
    
    // 关键调用：强制停止VM - 第202行
    ret = vm_stop_force_state(state);  // state = RUN_STATE_FINISH_MIGRATE
    
    return ret;
}
```

### 4. 系统运行状态控制层

```c
// 强制停止VM - system/cpus.c:763
int vm_stop_force_state(RunState state)
{
    if (runstate_is_live(runstate_get())) {
        return vm_stop(state);  // 第766行
    } else {
        // VM已经停止的情况
        runstate_set(state);
        // ...
    }
}
        ↓
// VM停止函数 - system/cpus.c:684
int vm_stop(RunState state)
{
    if (qemu_in_vcpu_thread()) {
        // 如果在vCPU线程中，异步处理
        qemu_system_vmstop_request(state);
        return 0;
    }
    
    // 直接调用停止函数 - 第697行
    return do_vm_stop(state, true);
}
```

### 5. 实际 VM 停止执行层

```c
// 实际停止VM - system/cpus.c:278
static int do_vm_stop(RunState state, bool send_stop)
{
    int ret = 0;
    RunState oldstate = runstate_get();
    
    if (runstate_is_live(oldstate)) {
        runstate_set(state);           // 设置新状态
        cpu_disable_ticks();           // 禁用CPU时钟
        
        if (oldstate == RUN_STATE_RUNNING) {
            pause_all_vcpus();         // 暂停所有vCPU
        }
        
        // 🔥 关键调用：通知状态变化 - 第290行
        vm_state_notify(0, state);     // running=false, state=RUN_STATE_FINISH_MIGRATE
        
        if (send_stop) {
            qapi_event_send_stop();    // 发送停止事件
        }
    }
    
    return ret;
}
```

### 6. 状态通知分发层

```c
// 状态变化通知 - system/runstate.c:357
void vm_state_notify(bool running, RunState state)
{
    VMChangeStateEntry *e, *next;
    
    trace_vm_state_notify(running, state, RunState_str(state));
    
    if (running) {
        // VM启动时的处理（正向遍历）
        QTAILQ_FOREACH_SAFE(e, &vm_change_state_head, entries, next) {
            e->cb(e->opaque, running, state);
        }
    } else {
        // 🔥 VM停止时的处理（反向遍历）- 第374-382行
        QTAILQ_FOREACH_REVERSE_SAFE(e, &vm_change_state_head, entries, next) {
            if (e->prepare_cb) {
                e->prepare_cb(e->opaque, running, state);  // 预处理回调
            }
        }
        
        QTAILQ_FOREACH_REVERSE_SAFE(e, &vm_change_state_head, entries, next) {
            // 🎯 这里调用各个设备的vm_change_state_handler
            e->cb(e->opaque, running, state);  // 主回调函数
        }
    }
}
```

### 7. ITS 设备处理层

```c
// ITS设备的状态变化处理器 - hw/intc/arm_gicv3_its_kvm.c:75
static void vm_change_state_handler(void *opaque, bool running, RunState state)
{
    GICv3ITSState *s = (GICv3ITSState *)opaque;
    Error *err = NULL;
    
    // 关键判断：只在VM停止时处理
    if (running) {
        return;  // running=false时才继续
    }
    
    // 🎯 最终目标：调用KVM保存ITS表
    kvm_device_access(s->dev_fd, KVM_DEV_ARM_VGIC_GRP_CTRL,
                      KVM_DEV_ARM_ITS_SAVE_TABLES, NULL, true, &err);
}
```

### 8. 调用链时序图

![](https://cdn.nlark.com/yuque/__mermaid_v3/6edac7dc466d6cf93544f612eb5b2d14.svg)

### 9. 关键要点

1. 触发条件：`running = false` 且 `state = RUN_STATE_FINISH_MIGRATE`
2. 执行上下文：QEMU 主线程，非 vCPU 上下文
3. 遍历顺序：VM 停止时采用反向遍历（第 374 行），确保清理顺序正确
4. 注册时机：ITS 设备在初始化时通过 `qemu_add_vm_change_state_handler` 注册回调
5. 执行时机：在 CPU 暂停、状态设置之后，确保 VM 已完全停止

这就是从用户发起热迁移命令到最终调用 ITS 设备的 `vm_change_state_handler` 的完整调用通路！
---
title: migration_thread三个阶段
date: 2025-09-01 19:46:53
modify: 2025-09-01 19:46:53
author: days
category: ICSAS
published: 2025-09-01
draft: false
---

三个迁移阶段对应的具体代码实现：

## 🎯 Setup 阶段 - 初始化和设置

### 主要函数: `qemu_savevm_state_setup()` 

📍 文件: `/migration/savevm.c:1322-1374`

```c
void qemu_savevm_state_setup(QEMUFile *f)
{
    // 遍历所有注册的状态处理器
    QTAILQ_FOREACH(se, &savevm_state.handlers, entry) {
        // 🔸 方式1: VMStateDescription保存 - CPU寄存器、设备配置
        if (se->vmsd && se->vmsd->early_setup) {
            ret = vmstate_save(f, se, ms->vmdesc);  // 保存CPU状态等
        }
        
        // 🔸 方式2: SaveVMHandlers设置 - RAM、块设备准备
        if (se->ops && se->ops->save_setup) {
            ret = se->ops->save_setup(f, se->opaque);  // RAM初始化等
        }
    }
}
```

### 调用位置: 

📍 文件: `/migration/migration.c:3518` (在 `migration_thread` 中)

### 具体实现例子:
- RAM 设置: `ram_save_setup()` - `/migration/ram.c:3070`
- 块设备设置: `block_save_setup()` - `/migration/block.c:714`
- CPU 状态: 通过 `vmstate_save()` 保存各架构 CPU 寄存器

---

## 🔄 Iterate 阶段 - 迭代传输脏页

### 主要函数: `qemu_savevm_state_iterate()`

📍 文件: `/migration/savevm.c:1410-1460`

```c
int qemu_savevm_state_iterate(QEMUFile *f, bool postcopy)
{
    QTAILQ_FOREACH(se, &savevm_state.handlers, entry) {
        if (se->ops && se->ops->save_live_iterate) {
            ret = se->ops->save_live_iterate(f, se->opaque);  // 迭代保存脏页
        }
    }
}
```

### 调用位置: 

📍 文件: `/migration/migration.c:3279` (在 `migration_iteration_run` 中)

### 迭代控制逻辑:
```c
static MigIterateState migration_iteration_run(MigrationState *s)
{
    // 检查待传输数据量
    if (pending_size < s->threshold_size && can_switchover) {
        migration_completion(s);  // 触发完成阶段
        return MIG_ITERATE_BREAK;
    }
    
    // 继续迭代传输
    qemu_savevm_state_iterate(s->to_dst_file, in_postcopy);
    return MIG_ITERATE_RESUME;
}
```

### 具体实现例子:
- RAM 迭代: `ram_save_iterate()` - 传输脏页数据
- 块设备迭代: `block_save_iterate()` - 传输块设备数据

---

## ✅ Complete 阶段 - 最终完成传输

### 主要函数: `qemu_savevm_state_complete_precopy()`

📍 文件: `/migration/savevm.c:1510-1546`

```c
int qemu_savevm_state_complete_precopy_iterable(QEMUFile *f, bool in_postcopy)
{
    QTAILQ_FOREACH(se, &savevm_state.handlers, entry) {
        if (se->ops && se->ops->save_live_complete_precopy) {
            ret = se->ops->save_live_complete_precopy(f, se->opaque);  // 最终完成传输
        }
    }
}
```

### 调用位置: 

📍 文件: `/migration/migration.c:2776` (在 `migration_completion` 中)

### 触发条件:
- 脏页数量降至阈值以下 (`pending_size < s->threshold_size`)
- 可以进行切换 (`can_switchover`)

### 具体实现例子:
- RAM 完成: `ram_save_complete()` - 传输剩余 RAM 数据
- 块设备完成: `block_save_complete()` - 完成块设备数据传输

---

## 📋 阶段调用链总结

```
migration_thread()  // 主迁移线程
├─ qemu_savevm_state_setup()           // Setup阶段
├─ while(migration_is_active()) {
│   └─ migration_iteration_run()
│       └─ qemu_savevm_state_iterate()  // Iterate阶段
│   }
└─ migration_completion()
    └─ qemu_savevm_state_complete_precopy()  // Complete阶段
```

每个阶段都会遍历所有注册的状态处理器 (SaveStateEntry)，分别调用对应的处理函数来完成 VM 状态的分阶段传输！

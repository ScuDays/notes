---
title: 未命名
date: 2025-09-01 16:24:55
modify: 2025-09-01 16:24:55
author: days
category: ICSAS
published: 2025-09-01
draft: false
---

让我帮您追踪 `s->threshold_size = expected_bw_per_ms * migrate_downtime_limit();` 这行代码的调用链。

[6 tools called]

让我为您详细追踪 `s->threshold_size = expected_bw_per_ms * migrate_downtime_limit();` 这行代码的完整调用链。

## 🔗 完整调用链追踪

### 📍 代码位置

```c

// 在migration_update_counters()函数中，第3190行

s->threshold_size = expected_bw_per_ms * migrate_downtime_limit();

```

### 🎯 调用链分析

#### 1️⃣ 所在函数: `migration_update_counters()`

```c

static void migration_update_counters(MigrationState *s, int64_t current_time)

{

uint64_t transferred, transferred_pages, time_spent;

uint64_t current_bytes;

uint64_t switchover_bw;

double expected_bw_per_ms;

double bandwidth;

// ... 计算实际传输带宽 ...

if (switchover_bw) {

// 如果用户指定了切换带宽，使用用户指定值

expected_bw_per_ms = switchover_bw / 1000;

} else {

// 如果用户没有指定，使用估算的带宽

expected_bw_per_ms = bandwidth;

}

// 🎯 关键代码：动态计算阈值

s->threshold_size = expected_bw_per_ms * migrate_downtime_limit();

// ... 更新其他统计信息 ...

}

```

#### 2️⃣ 调用者 1: `migration_rate_limit()`

```c

bool migration_rate_limit(void)

{

int64_t now = qemu_clock_get_ms(QEMU_CLOCK_REALTIME);

MigrationState *s = migrate_get_current();

bool urgent = false;

migration_update_counters(s, now); // ← 调用位置

// ... 速率限制逻辑 ...

}

```

#### 3️⃣ 调用者 2: `bg_migration_thread()`

```c

static void *bg_migration_thread(void *opaque)

{

// ... 初始化 ...

while (migration_is_active()) {

MigIterateState iter_state = bg_migration_iteration_run(s);

// ... 迭代处理 ...

// 直接调用更新计数器

migration_update_counters(s, qemu_clock_get_ms(QEMU_CLOCK_REALTIME)); // ← 调用位置

}

// ... 清理 ...

}

```

#### 4️⃣ 最终调用者: `migration_thread()`

```c

static void *migration_thread(void *opaque)

{

// ... 设置阶段 ...

// 主迁移循环：持续迁移直到完成或失败

while (migration_is_active()) {

if (urgent || !migration_rate_exceeded(s->to_dst_file)) {

MigIterateState iter_state = migration_iteration_run(s);

// ... 处理迭代结果 ...

}

// ... 错误检测 ...

urgent = migration_rate_limit(); // ← 这里调用migration_rate_limit()

}

// ... 清理 ...

}

```

## 🔄 完整调用链路图

```

QMP命令: migrate

↓

qmp_migrate()

↓

[传输方式选择]

↓

migration_channel_connect()

↓

migrate_fd_connect()

↓

qemu_thread_create(&s->thread, "live_migration", migration_thread, s, ...)

↓

migration_thread() # 主迁移线程

↓

while (migration_is_active()) { # 主循环

↓

migration_iteration_run() # 执行迁移迭代

↓

migration_rate_limit() # 速率控制检查

↓

migration_update_counters() # 更新统计信息

↓

s->threshold_size = expected_bw_per_ms * migrate_downtime_limit(); # 🎯 目标代码

}

```

## ⚡ 调用时机和频率

### 🕐 何时调用

1. 每个迁移迭代后: 在 `migration_thread` 的主循环中

2. 速率限制检查时: 通过 `migration_rate_limit()` 间接调用

3. 背景迁移时: 在 `bg_migration_thread` 中直接调用

### 📊 调用频率

- 高频调用: 每次迁移循环迭代都会调用

- 实时更新: 根据当前网络状况动态调整阈值

- 自适应机制: 带宽变化时立即反映到阈值计算中

### 🎯 调用目的

1. 动态阈值调整: 根据实际传输性能更新停机阈值

2. 带宽估算: 基于实际传输速度或用户配置计算期望带宽

3. 性能统计: 更新迁移性能指标（mbps、pages_per_second 等）

## 💡 关键理解

这行代码的核心作用是：

- 实时调整: 在迁移过程中动态调整停机阈值

- 自适应性: 根据网络状况和传输性能自动优化

- 精确控制: 确保停机时间在用户设置的限制范围内

这就是为什么 QEMU 热迁移能够在不同网络环境下都能保持较好的停机时间控制！
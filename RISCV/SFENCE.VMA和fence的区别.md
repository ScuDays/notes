---
title: 未命名
date: 2025-12-06 15:33:16
modify: 2025-12-06 15:33:16
author: days
category: ICSAS
published: 2025-12-06
draft: false
---
## 5. 总结

| 场景   | 动作                    | 无 Svvptc (旧)            | 有 Svvptc (新)           |
| :--- | :-------------------- | :---------------------- | :--------------------- |
| 内存分配 | Invalid $\to$ Valid   | 必须 `SFENCE.VMA`         | 直接跳过 (NOP)             |
| 内存回收 | Valid $\to$ Invalid   | 必须 `SFENCE.VMA`         | 必须 `SFENCE.VMA` (安全底线) |
| 权限提升 | Read $\to$ Read/Write | 必须 `SFENCE.VMA`         | 直接跳过 (NOP) (通常可行)      |
| 异常处理 | 处理 vmalloc 懒惰更新       | 清标志 + `SFENCE.VMA` + 重试 | 清标志 + NOP + 重试         |

一句话总结 Svvptc：

它通过硬件上的改进（不再死记硬背“无效”状态），允许操作系统在分配内存时做一个“甩手掌柜”，省去了昂贵的通知指令，极大地提升了内存密集型任务和系统启动的性能。

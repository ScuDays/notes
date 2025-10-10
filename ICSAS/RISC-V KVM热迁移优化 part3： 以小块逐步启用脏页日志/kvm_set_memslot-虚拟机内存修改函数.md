---
title: kvm_set_memslot-虚拟机内存修改函数
date: 2025-10-09 17:02:24
modify: 2025-10-09 17:02:24
author: days
category: ICSAS
published: 2025-10-09
draft: false
---
# kvm_set_memslot-虚拟机内存修改函数
## 简介

简单来说，`kvm_set_memslot` 是 KVM 中用于执行所有与客户机（Guest）内存布局相关的原子性修改的核心函数。它就像一个总指挥，负责安全、有序地创建、删除、移动或修改虚拟机内存区域（memslot）的属性。

每当用户空间（如 QEMU）通过 `KVM_SET_USER_MEMORY_REGION` ioctl 请求改变虚拟机的内存布局时，最终都会调用到这个函数。

> virt/kvm/kvm_main.c  

```c
static int kvm_set_memslot(struct kvm *kvm,
			   struct kvm_memory_slot *old,
			   struct kvm_memory_slot *new,
			   enum kvm_mr_change change)
{
	struct kvm_memory_slot *invalid_slot;
	int r;

	/*
	 * Released in kvm_swap_active_memslots().
	 *
	 * Must be held from before the current memslots are copied until after
	 * the new memslots are installed with rcu_assign_pointer, then
	 * released before the synchronize srcu in kvm_swap_active_memslots().
	 *
	 * When modifying memslots outside of the slots_lock, must be held
	 * before reading the pointer to the current memslots until after all
	 * changes to those memslots are complete.
	 *
	 * These rules ensure that installing new memslots does not lose
	 * changes made to the previous memslots.
	 * 
	 * 这个锁在 kvm_swap_active_memslots() 中释放。
     *
     * 必须从复制当前 memslots 之前一直持有，直到用 rcu_assign_pointer 安装
     * 新的 memslots 之后。然后在 kvm_swap_active_memslots() 中的
     * synchronize_srcu 之前释放。
     *
     * 当在 slots_lock 之外修改 memslots 时，必须在读取当前 memslots 指针之前
     * 持有该锁，直到对这些 memslots 的所有更改完成之后。
     *
     * 这些规则确保安装新的 memslots 不会丢失对先前 memslots 的更改。
	 */
	mutex_lock(&kvm->slots_arch_lock);

	/*
	 * Invalidate the old slot if it's being deleted or moved.  This is
	 * done prior to actually deleting/moving the memslot to allow vCPUs to
	 * continue running by ensuring there are no mappings or shadow pages
	 * for the memslot when it is deleted/moved.  Without pre-invalidation
	 * (and without a lock), a window would exist between effecting the
	 * delete/move and committing the changes in arch code where KVM or a
	 * guest could access a non-existent memslot.
	 *
	 * Modifications are done on a temporary, unreachable slot.  The old
	 * slot needs to be preserved in case a later step fails and the
	 * invalidation needs to be reverted.
	 * 
	 * 如果旧的内存槽正在被删除或移动，则先将其无效化。这样做是为了让 vCPU
     * 可以继续运行，通过确保在删除/移动内存槽时，不存在该槽的映射或影子页。
     * 如果没有预先无效化（并且没有锁），在执行删除/移动和在架构代码中提交
     * 变更之间会存在一个时间窗口，KVM 或客户机可能会访问一个不存在的内存槽。
     *
     * 修改是在一个临时的、不可达的槽上进行的。旧的槽需要被保留，以防后续
     * 步骤失败时需要回滚无效化操作。
	 */
	if (change == KVM_MR_DELETE || change == KVM_MR_MOVE) {
		invalid_slot = kzalloc(sizeof(*invalid_slot), GFP_KERNEL_ACCOUNT);
		if (!invalid_slot) {
			mutex_unlock(&kvm->slots_arch_lock);
			return -ENOMEM;
		}
		kvm_invalidate_memslot(kvm, old, invalid_slot);
	}

	// 调用准备函数，进行检查和资源分配（如 dirty_bitmap）
	r = kvm_prepare_memory_region(kvm, old, new, change);
	if (r) {
		/*
		 * For DELETE/MOVE, revert the above INVALID change.  No
		 * modifications required since the original slot was preserved
		 * in the inactive slots.  Changing the active memslots also
		 * release slots_arch_lock.
		 * 
		 * 对于 DELETE/MOVE 操作，回滚上面的 INVALID 变更。
         * 由于原始的槽被保留在非活动槽中，所以不需要修改。
         * 更改活动内存槽也会释放 slots_arch_lock。
		 */
		if (change == KVM_MR_DELETE || change == KVM_MR_MOVE) {
			kvm_activate_memslot(kvm, invalid_slot, old);
			kfree(invalid_slot);
		} else {
			mutex_unlock(&kvm->slots_arch_lock);
		}
		return r;
	}

	/*
	 * For DELETE and MOVE, the working slot is now active as the INVALID
	 * version of the old slot.  MOVE is particularly special as it reuses
	 * the old slot and returns a copy of the old slot (in working_slot).
	 * For CREATE, there is no old slot.  For DELETE and FLAGS_ONLY, the
	 * old slot is detached but otherwise preserved.
	 * 
	 * 对于 DELETE 和 MOVE，工作槽现在作为旧槽的 INVALID 版本处于活动状态。
     * MOVE 特别之处在于它重用了旧槽，并返回旧槽的一个副本（在 working_slot 中）。
     * 对于 CREATE，没有旧槽。对于 DELETE 和 FLAGS_ONLY，旧槽被分离但被保留。
     */
	// 根据不同的变更类型，执行相应的内存槽操作
	if (change == KVM_MR_CREATE)
		// 如果是创建新的内存槽
		kvm_create_memslot(kvm, new);
	else if (change == KVM_MR_DELETE)
		// 如果是删除内存槽
		kvm_delete_memslot(kvm, old, invalid_slot);
	else if (change == KVM_MR_MOVE)
		// 如果是移动内存槽（修改其客户机物理地址 GPA）
		kvm_move_memslot(kvm, old, new, invalid_slot);
	else if (change == KVM_MR_FLAGS_ONLY)
		// 如果只是修改内存槽的标志位（例如，开启/关闭脏页日志）
		kvm_update_flags_memslot(kvm, old, new);
	else
		BUG();

	/* Free the temporary INVALID slot used for DELETE and MOVE. */
	// 释放用于 DELETE 和 MOVE 的临时 INVALID 槽。
	
	if (change == KVM_MR_DELETE || change == KVM_MR_MOVE)
		kfree(invalid_slot);

	/*
	 * No need to refresh new->arch, changes after dropping slots_arch_lock
	 * will directly hit the final, active memslot.  Architectures are
	 * responsible for knowing that new->arch may be stale.
	 * 
	 * 无需刷新 new->arch，因为在释放 slots_arch_lock 之后的更改将直接作用于
     * 最终的活动内存槽。各个体系结构需要自己负责处理 new->arch 可能是陈旧数据的情况。
	 */
	kvm_commit_memory_region(kvm, old, new, change);

	return 0;
}
```

---

### 这个函数具体做了什么？

`kvm_set_memslot` 的工作可以根据传入的 `change` 类型分为几个主要场景：

*   `KVM_MR_CREATE`: 创建一个新的内存区域（比如给虚拟机添加内存）。
*   `KVM_MR_DELETE`: 删除一个现有的内存区域。
*   `KVM_MR_MOVE`: 移动一个内存区域，即改变它在客户机物理地址空间（GPA）中的位置，但其在宿主机上的用户空间地址（HVA）不变。
*   `KVM_MR_FLAGS_ONLY`: 只修改一个内存区域的标志位，比如开启或关闭脏页日志（`KVM_MEM_LOG_DIRTY_PAGES`），或者将其设置为只读。

### 核心工作流程是怎样的？

为了保证在多 VCPU 环境下安全地修改内存布局，`kvm_set_memslot` 遵循一个非常严谨和复杂的流程，可以概括为以下几个关键步骤：

1.  加锁 (`mutex_lock(&kvm->slots_arch_lock)`):
    首先，它会获取一个重要的锁，以确保在整个多步骤的修改过程中，不会有其他线程来干扰内存槽的配置，防止竞态条件。

2.  预先失效 (仅针对 `DELETE` 和 `MOVE`):
    *   这是一个非常关键的安全步骤。在真正删除或移动一个内存槽之前，它会先创建一个临时的、标记为“无效”（INVALID）的副本。
    *   然后，它会激活这个无效副本，并强制所有 VCPU 刷新它们的 TLB 和影子页表，确保没有任何 VCPU 还持有对旧内存区域的有效映射。
    *   这样就杜绝了在后续操作中，有 VCPU 访问到一个已经被删除或正在被移动的、处于不一致状态的内存区域的风险。

3.  准备阶段 (`kvm_prepare_memory_region`):
    *   这是您之前关注的函数。在此阶段，KVM 会为新的内存槽配置做准备工作。
    *   最重要的任务就是分配和初始化脏页位图（dirty bitmap）。如果开启了“初始全脏”优化，位图会在这里被全部设置为 1；否则，它会被创建并保持为 0。
    *   如果准备阶段失败（例如内存不足），函数会安全地回滚之前的操作并返回错误。

4.  执行变更:
    *   根据 `change` 的类型，调用相应的辅助函数（`kvm_create_memslot`, `kvm_delete_memslot`, `kvm_move_memslot`, `kvm_update_flags_memslot`）。
    *   这些函数会更新 KVM 内部的内存槽数据结构（比如红黑树），将新的配置应用到“非活动”的内存槽集合中，然后通过一个原子操作（`rcu_assign_pointer`）将其切换为“活动”状态。

5.  提交阶段 (`kvm_commit_memory_region`):
    *   这是最后一步，负责调用特定于体系结构的代码（`kvm_arch_commit_memory_region`）来最终完成变更。
    *   正是在这个阶段的深层调用链中，如果未使用优化，KVM 会执行传统的、耗时的写保护操作（即调用 `kvm_mmu_slot_remove_write_access`）。
    *   同时，它还会清理旧的、不再需要的资源，比如释放旧的 `memslot` 结构体或旧的脏页位图。

### 总结

`kvm_set_memslot` 是一个高度封装的、健壮的接口。它通过锁、RCU（读-拷贝-更新）、原子操作和分阶段提交等多种机制，确保了对虚拟机内存布局的修改是原子且安全的，即使在有多个 VCPU 同时运行的情况下也不会出错。它是连接 KVM 通用层内存管理逻辑和具体体系结构 MMU 实现的关键桥梁。
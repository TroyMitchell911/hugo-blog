+++
date = '2026-04-24T23:00:00+08:00'
draft = false
title = 'linux-mm[1]: folio、HVO 与内存生命周期'
tags = ["linux", "linux-subsystem-mm", "riscv"]
summary = '深入解析 Linux 内核中的 struct folio、HugeTLB Vmemmap 优化、反向映射、按需分页与脏页管理机制。'
+++

## 概述

本文接续 linux-mm[0]（物理内存、struct page、引用计数），继续深入：folio 抽象、HugeTLB Vmemmap 优化（HVO）、反向映射（rmap）、按需分页（demand paging）以及脏页管理。

## 1. RISC-V 的内存模型

Linux 支持三种物理内存模型，RISC-V 三种都支持：

- **FLATMEM**：假设物理地址空间是单一连续区间。简单快速，但不灵活。
- **SPARSEMEM**：将物理内存划分为固定大小的 section，支持内存空洞和热插拔。
- **VMEMMAP**：SPARSEMEM 的优化版本。将所有 `struct page` 映射到一段连续的虚拟数组（`vmemmap`），使得 `pfn_to_page(pfn)` 变成简单的数组下标访问：`vmemmap + pfn`。

64 位 RISC-V 默认使用 VMEMMAP。vmemmap 数组在开机时一次性分配，覆盖所有可能的 PFN，不存在动态递归。

## 2. struct folio 与 HVO 的区别

两者解决的是不同问题：

- **struct folio** 是类型安全抽象。它包装复合页（compound page）的 head page，给内核提供一个有类型的句柄，防止把 tail page 误传给需要 head 的函数。运行时零开销——它和 `struct page` 共用同一块内存。
- **HVO（HugeTLB Vmemmap Optimization）** 是节省内存的优化。一个 2MB HugeTLB 大页对应 512 个 `struct page`（512 × 64 字节 = 32KB，占 8 个物理页）。HVO 将 511 个 tail page 的 vmemmap 条目重映射到与 head 相同的物理页，把物理开销从 8 页压缩到 1 页，释放 7 页还给 buddy allocator。

## 3. struct folio 的内存布局

`struct folio` 与 `struct page` 共用内存，头部字段完全重叠：

```c
struct folio {
    union {
        struct page page;   /* head page，同一块内存 */
        struct {
            unsigned long flags;
            struct list_head lru;
            /* ... */
        };
    };
    /* folio 独有字段借用 tail page 的槽位存放： */
    /* __page_1 -> _deferred_list、_entire_mapcount 等 */
    /* __page_2 -> _nr_pages_mapped、_pincount        */
};
```

folio 独有的字段放不进 head page 时，借用 tail page 的 `struct page` 槽位（`__page_1`、`__page_2`、`__page_3`）。这是可行的，因为 tail page 不会被独立使用，它们的槽位可以被复合页重新利用。

## 4. HVO：vmemmap 重映射与 fake head 识别

一个 2MB HugeTLB 大页对应 512 × 4KB 物理页，vmemmap 正常情况下有 512 个独立的 `struct page` 条目。HVO 将条目 1–511 重映射到与条目 0（head）相同的物理页。

HVO 之后，512 个 vmemmap 条目的虚拟地址各不相同，但物理上指向同一页。head `struct page` 实际上变成只读共享。

**Fake head 识别**：内核判断一个 page 是真正的 compound head 还是 HVO fake head，需要同时满足三个条件：

1. 地址 4KB 对齐
2. `PG_head` 标志位为 1
3. `page[1].compound_head` 的 bit 0 为 1（指向 head，LSB=1）

真正的 head 满足这三个条件，HVO fake head 也满足——这是故意设计的，内核在大多数场景下对两者一视同仁。

## 5. 释放 HugeTLB 大页：先恢复 vmemmap

HugeTLB 大页归还给 buddy allocator 之前，必须先撤销 HVO：

1. 分配 7 个新物理页，用于 tail `struct page`
2. 将 head `struct page` 的内容复制到每个新页
3. 将 vmemmap 条目 1–511 重新映射回各自独立的物理页
4. TLB flush（vmemmap 虚拟地址现在指向新的物理页）
5. 将 512 × 4KB 数据页归还给 buddy

为什么必须恢复？buddy allocator 需要向每个 `struct page` 写入 LRU 链表指针。HVO 激活时，所有 tail page 共享一个只读物理页，写操作会 fault 或破坏 head。每个 `struct page` 必须是独立可写的。

## 6. rmap：反向映射

给定一个 folio，内核如何找到所有映射它的虚拟地址？这在内存回收（释放前先解除映射）和 COW 时都需要。

匿名页：

```
folio->mapping  ->  anon_vma
anon_vma        ->  VMA 链表
对每个 VMA：
    vaddr = vma->vm_start + (folio->index - vma->vm_pgoff) * PAGE_SIZE
    ptep  = 遍历页表找到 vaddr
    pte_clear(ptep)   /* 解除映射 */
```

文件映射页：

```
folio->mapping  ->  address_space（inode）
address_space->i_mmap  ->  VMA 区间树
/* 虚拟地址计算方式相同 */
```

`folio->index` 存储该页在文件或匿名映射中的偏移量，使得每个 VMA 的虚拟地址计算是 O(1)。

## 7. VMA 与按需分页

进程调用 `malloc()` 或 `mmap()` 时，内核只创建一个 VMA（`vm_area_struct`）——描述虚拟地址范围的元数据。此时不分配任何物理页。

第一次访问该范围内的地址时：

1. CPU 触发 page fault（没有对应的 PTE）
2. 内核查找覆盖该地址的 VMA
3. 没有 VMA：发送 `SIGSEGV`
4. 有 VMA：分配物理页，填充内容（匿名页清零，文件映射页从文件读取），安装 PTE
5. 返回用户态——访问透明地成功

这就是按需分页（demand paging）。进程可以 `malloc(1GB)` 但只访问 10MB——实际只会分配 10MB 的物理页。

## 8. 脏页管理：两层机制

页面被写入后变为"脏页"。有两个独立的脏标记：

**硬件层——PTE dirty bit**：CPU 在任何写操作时自动设置 PTE 中的 dirty bit，这是 MMU 的硬件特性。内核可以清除它来追踪上次检查后哪些页被写过。

**软件层——PG_dirty 标志**：内核在 `struct page`（或 folio）的 flags 字段中维护 `PG_dirty`。这是内核的权威脏状态记录，回写线程依赖它来决定哪些页需要刷盘。

两层同步：内核定期扫描 PTE dirty bit，将其"收割"到 `PG_dirty`，然后清除 PTE dirty bit 以便下次检测。

## 9. xarray tag 机制：高效查找脏页

文件的 page cache 用 xarray（基数树）存储所有 folio。每个节点有一个 tag 位图，其中包含 `DIRTY` tag。

当一个 folio 被标记为脏时，`DIRTY` tag 沿树向上传播到根节点。回写线程查找脏页时：

```c
/* 从 xarray 中找出所有带 DIRTY tag 的 folio */
xas_for_each_marked(&xas, folio, end_index, PAGECACHE_TAG_DIRTY) {
    /* 只访问脏页，跳过干净页 */
}
```

这是 O(log n) 的操作，不需要遍历所有页。一个有 100 万个页但只有 10 个脏页的文件，回写线程只需访问 10 个节点（加上路径上的内部节点），而不是扫描全部 100 万个。

## 10. struct page 的固定开销与开机分配

每个 `struct page` 占 64 字节，每个物理页 4KB。开销比例：

```
64 / 4096 = 1/64 ≈ 1.5625%
```

8GB 内存的机器，vmemmap 占用约 128MB。这是固定开销，无论内存如何使用。

**为什么不会无限递归？** 存放 `struct page` 本身的物理页，在 vmemmap 数组里也有对应的 `struct page` 条目。但这不是动态递归——内核在开机时按照机器的物理页总数，一次性分配好所有 `struct page` 的物理空间。所有层级都在这一次分配中完成，之后不再动态增长。

## 小结

本文从 RISC-V 内存模型出发，梳理了 folio 的类型安全设计、HVO 的 vmemmap 重映射优化、大页释放时的 vmemmap 恢复流程、rmap 反向映射的实现路径、demand paging 的懒分配机制，以及脏页管理的两层架构和 xarray tag 的高效查找。这些机制共同构成了 Linux 内存子系统中页面生命周期管理的核心。

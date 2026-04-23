+++
date = '2026-04-23T18:15:51+08:00'
draft = false
title = 'linux-mm[0]: 物理内存与 struct page'
tags = ["linux", "linux-subsystem-mm"]
summary = 'Linux 内核如何将 RAM 划分为页，通过 struct page 跟踪每页元数据，并通过引用计数和脏页回写机制管理页的生命周期。'
+++

## 物理内存是什么

学习 Linux 内核内存管理，第一个问题看起来很简单：从内核的角度来看，物理内存到底是什么？

你的内存条是一大片连续的字节。内核不会把它当成一整块来用——它把内存切成固定大小的块，每一块叫做一个**页（page）**。在 x86_64 上，默认页大小是 **4KB**，一台 8GB 内存的机器大约有 200 万个页。

### 4KB 是谁定的？为什么是 4KB？

这是**硬件决定的，不是内核决定的**。每颗现代 CPU 里都有一个专门负责虚拟地址到物理地址翻译的硬件单元，叫做 **MMU（内存管理单元）**。MMU 在芯片设计阶段就固定了它支持的页大小，内核只能用 MMU 支持的那几种，没有别的选择。

x86_64 的 MMU 支持三种页大小：

| 大小 | 名称 | 典型用途 |
|------|------|----------|
| 4KB  | 普通页 | 通用 |
| 2MB  | 大页（Huge Page） | 数据库、大内存映射 |
| 1GB  | 巨页（Gigantic Page） | 特殊高性能场景 |

4KB 成为默认值是因为它在两个极端之间取得了平衡：足够小，不会在稀疏分配时浪费太多内存；足够大，不会让页表变得过于庞大。内核可以通过 **THP（透明大页）** 或显式的 `mmap` 标志使用 2MB 或 1GB 的大页，但 4KB 是一切的基础。

## struct page — 每个页的"档案卡"

内核需要追踪每一个页的状态：它在被使用吗？被谁用？能不能被回收？有没有被写脏？

这些信息存在 `struct page` 里，定义在 `include/linux/mm_types.h`。可以把它想象成停车场里每个停车位上贴的一张档案卡——停车场是你的内存，每个停车位是一个页，档案卡记录了谁停在这里、现在是什么状态。

```c
struct page {
    memdesc_flags_t flags;      /* 状态标志位 — 脏页、锁定、回写中等 */
    union {
        struct list_head lru;   /* LRU 链表指针，用于内存回收 */
        ...
    };
    atomic_t _mapcount;         /* 有多少个页表项映射了这个页 */
    atomic_t _refcount;         /* 引用计数 — 降到 0 时页才能被释放 */
    ...
};
```

逐个字段来看。

### flags

一个位掩码，记录页的各种状态。常见的标志位：

- `PG_dirty` — 页已被写入但尚未刷回磁盘
- `PG_locked` — 有人持有页锁（比如正在做 I/O）
- `PG_writeback` — 页正在被写回存储
- `PG_uptodate` — 页的数据是有效且最新的
- `PG_lru` — 页在某个 LRU 链表上

这些标志位的操作都是原子的，因为多个 CPU 可能同时访问同一个页。

### _mapcount

记录有多少个进程的页表项（PTE）指向这个页。当一个页被多个进程共享时（比如共享库），`_mapcount` 就会大于 0。它的初始值是 -1，表示"没有任何映射"，每增加一个映射就加一。

### lru

一个 `list_head`，把这个页链入内核的某个 LRU（最近最少使用）链表。内存回收子系统（`kswapd`）通过这些链表找到可以驱逐的页。这个字段是一个 union 的一部分——根据页当前的用途，它会被复用来存储其他信息。

## 现代抽象：folio

在较新的内核版本（5.16+）中，内核引入了 `struct folio` 作为 `struct page` 的上层抽象。一个 folio 代表一组物理连续、大小为 2 的幂次方的页，作为一个整体来操作。引入它的动机是减少处理大页时的开销——与其对一个 2MB 大页的 512 个 `struct page` 逐一操作，不如用一个 folio 来统一处理。

大多数新内核代码都操作 folio 而不是原始的 `struct page`。底层的 `struct page` 还在，但 folio API（`folio_get()`、`folio_put()`、`folio_ref_count()`）是推荐的接口。

## 引用计数：_refcount

`_refcount` 是页的引用计数——一个 `atomic_t`，记录当前有多少个内核子系统持有对这个页的引用。规则很简单：**`_refcount` 降到零时，页才能被释放**。

```c
// include/linux/mm.h
static inline void folio_get(struct folio *folio) {
    atomic_inc(&folio->_refcount);
}

void __folio_put(struct folio *folio) {
    if (folio_is_zone_device(folio)) {
        free_zone_device_folio(folio);
        return;
    }
    /* ... 最终把页还给伙伴分配器 */
}
```

谁会持有引用？

- **进程的页表**映射了这个页 → +1
- **页缓存（page cache）** 持有一个文件页 → +1
- **驱动程序**正在用这个页做 DMA → +1
- **内核**临时 pin 住一个页做 I/O → +1

当所有这些引用都释放后，`_refcount` 归零，`__folio_put()` 被调用，页被归还给**伙伴分配器**（后续文章会详细讲）。

`_refcount` 和 `_mapcount` 容易混淆。`_mapcount` 只统计页表映射。`_refcount` 统计**所有**引用，包括非映射类型的引用（比如页缓存的 pin）。一个页可以 `_refcount > 0` 同时 `_mapcount == -1`——它被内核持有，但没有映射到任何进程的地址空间。

## 脏页与写回

当进程写入一个页，而这个修改还没有刷回后端存储（磁盘或网络文件系统）时，这个页就变成了"脏页"。内核不会立即写回每一个脏页——那样会慢得无法接受。它用三道机制来保证脏页最终会被写回：

**第一道：定时器，每 5 秒触发一次**
```c
// mm/page-writeback.c
unsigned int dirty_writeback_interval = 5 * 100; /* 单位：百分之一秒 */
```
内核线程每 5 秒唤醒一次，把脏了太久的页刷回去（默认超过 30 秒的脏页会被强制写回）。

**第二道：脏页比例阈值**
```c
int dirty_ratio = 20;           /* 进程触发同步写回的阈值 */
int dirty_background_ratio = 10; /* 后台写回线程启动的阈值 */
```
当脏页占总内存的比例超过 `dirty_background_ratio`（默认 10%），后台写回线程就会启动。超过 `dirty_ratio`（默认 20%），写入进程本身会被阻塞，强制同步写回。

**第三道：内存回收触发写回**
当系统内存紧张，`kswapd` 开始回收页时，遇到脏页会先触发写回，写完再回收。

这三道防线保证了即使没有显式的 `fsync()`，脏页也不会永远留在内存里。

## DMA 一致性：内存管理的边界

学到这里自然会想到 DMA。DMA 操作的内存必须是物理连续的，所以驱动会调用 mm/ 里的接口来分配页（比如 `alloc_pages(GFP_DMA)`）。但 `dma_alloc_coherent()` 这类 API 管的是另一件事——**CPU cache 和内存之间的一致性**。

这一层是纯软件概念，跟具体设备无关。它的实现在：
- `kernel/dma/coherent.c` — `dma_alloc_coherent()` 的核心逻辑
- `arch/x86/mm/`、`arch/arm64/mm/` — 各架构的 cache flush 实现

所以 DMA coherency 不属于 mm/ 子系统，而是 `kernel/dma/` + 各架构 mm/ 的交叉地带。mm/ 负责分配物理内存，DMA 层负责保证访问这块内存时 cache 是一致的。

## 总结

本文建立了 Linux 物理内存管理的基础认知：内核如何将 RAM 切分为固定大小的页、通过 `struct page` 追踪每个页的元数据、以及 `_refcount` 引用计数如何管理页的生命周期。我们还分析了脏页通过三道独立机制保证最终写回的原理，以及 DMA coherency 与 mm/ 子系统的边界关系。`struct folio` 作为现代抽象层叠加在这一切之上，为内核其他部分提供了更简洁的操作接口。

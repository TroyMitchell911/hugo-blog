+++
date = '2026-04-23T18:15:51+08:00'
draft = false
title = 'linux-mm[0]: Physical Memory and struct page'
tags = ["linux", "linux-subsystem-mm"]
+++

## What Is Physical Memory?

When we talk about memory management in the Linux kernel, the first question to answer is deceptively simple: what *is* physical memory, from the kernel's perspective?

Your RAM is a flat array of bytes. The kernel doesn't treat it as one giant blob — it slices it into fixed-size chunks called **pages**. On x86_64, the default page size is **4KB**, and a machine with 8GB of RAM has roughly 2 million of them.

### Why 4KB? Who Decided That?

This is a hardware decision, not a software one. Every modern CPU has a dedicated hardware unit called the **MMU (Memory Management Unit)** that handles virtual-to-physical address translation. The MMU is designed at the silicon level to work with specific page sizes — the kernel has no choice but to use what the MMU supports.

On x86_64, the MMU supports three page sizes:

| Size | Name | Typical Use |
|------|------|-------------|
| 4KB  | Normal page | General purpose |
| 2MB  | Huge page | Large mappings, databases |
| 1GB  | Gigantic page | Specialized workloads |

4KB became the standard because it strikes a balance: small enough to avoid wasting memory on sparse allocations, large enough to keep the page table from becoming enormous. The kernel can use 2MB or 1GB huge pages via **THP (Transparent Huge Pages)** or explicit `mmap` flags, but 4KB is the baseline everything else is built on.

## struct page — The Kernel's Metadata for Every Page

The kernel needs to track the state of every single page: Is it in use? By whom? Can it be reclaimed? Is it dirty?

It does this with `struct page`, defined in `include/linux/mm_types.h`. Think of it as a filing card attached to every parking spot in a massive parking garage — the garage is your RAM, each spot is a page, and the card records who parked there and what state it's in.

```c
struct page {
    memdesc_flags_t flags;      /* status flags — dirty, locked, writeback, etc. */
    union {
        struct list_head lru;   /* LRU list linkage for reclaim */
        ...
    };
    atomic_t _mapcount;         /* how many page table entries map this page */
    atomic_t _refcount;         /* reference count — page is freed when this hits 0 */
    ...
};
```

Let's walk through the important fields.

### flags

This is a bitmask of status flags. Some notable ones:

- `PG_dirty` — the page has been written to but not yet flushed to disk
- `PG_locked` — someone holds the page lock (e.g., during I/O)
- `PG_writeback` — the page is currently being written back to storage
- `PG_uptodate` — the page's data is valid and up to date
- `PG_lru` — the page is on an LRU list

These flags are manipulated atomically because multiple CPUs can touch the same page concurrently.

### _mapcount

Tracks how many page table entries (PTEs) across all processes point to this page. When a page is shared between processes (e.g., a shared library), `_mapcount` reflects that. It starts at -1 (meaning "not mapped"), and each mapping increments it.

### lru

A `list_head` that links this page into one of the kernel's LRU (Least Recently Used) lists. The memory reclaim subsystem (`kswapd`) uses these lists to find pages to evict when memory is tight. This field is part of a union — it gets reused for other purposes depending on the page's current role.

## The Modern Abstraction: folio

In recent kernel versions (5.16+), the kernel introduced `struct folio` as a higher-level abstraction over `struct page`. A folio represents a contiguous, power-of-two-aligned group of pages as a single unit. The motivation is to reduce overhead when dealing with large pages — instead of iterating over 512 individual `struct page` entries for a 2MB huge page, you work with one folio.

Most new kernel code operates on folios rather than raw pages. The `struct page` is still there underneath, but the folio API (`folio_get()`, `folio_put()`, `folio_ref_count()`) is the preferred interface going forward.

## Reference Counting: _refcount

`_refcount` is the page's reference count — an `atomic_t` that tracks how many kernel subsystems currently hold a reference to this page. The rule is simple: **when `_refcount` drops to zero, the page can be freed**.

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
    /* ... eventually returns the page to the buddy allocator */
}
```

Who holds references?

- A **process's page table** mapping the page → +1
- The **page cache** holding a file-backed page → +1
- A **driver** doing DMA with the page → +1
- The **kernel** temporarily pinning a page for I/O → +1

When all of these release their references, `_refcount` hits zero, `__folio_put()` is called, and the page is returned to the **buddy allocator** (more on that in a later post).

The distinction between `_refcount` and `_mapcount` trips people up at first. `_mapcount` counts page table mappings specifically. `_refcount` counts *all* references, including non-mapping ones like page cache pins. A page can have `_refcount > 0` with `_mapcount == -1` — it's held by the kernel but not mapped into any process's address space.

## Dirty Pages and Write-Back

A page becomes "dirty" when a process writes to it and the change hasn't been flushed to the backing storage (disk or network filesystem) yet. The kernel doesn't write every dirty page immediately — that would be catastrophically slow. Instead, it uses three mechanisms to ensure dirty pages eventually get written back:

**1. Periodic writeback (every 5 seconds)**
```c
// mm/page-writeback.c
unsigned int dirty_writeback_interval = 5 * 100; /* centiseconds */
```
A kernel thread wakes up every 5 seconds and flushes pages that have been dirty for too long.

**2. Dirty ratio threshold**
When the ratio of dirty pages to total memory exceeds a threshold (`/proc/sys/vm/dirty_ratio`), the kernel starts throttling writes and forcing writeback synchronously.

**3. Memory pressure**
When the system is low on memory and needs to reclaim pages, dirty pages must be written back before they can be freed.

## A Note on DMA Coherency

One question that naturally comes up when studying memory: where does `dma_alloc_coherent()` fit? Is DMA coherency part of the memory subsystem?

The short answer: it lives at the intersection. `dma_alloc_coherent()` is implemented in `kernel/dma/coherent.c` and ultimately calls into `mm/` to allocate physically contiguous pages (using `alloc_pages(GFP_DMA)`). But the *coherency* part — ensuring CPU caches and device-visible memory stay in sync — is handled per-architecture in `arch/x86/mm/`, `arch/arm64/mm/`, etc.

So DMA coherency is not purely a memory subsystem concern, but it's deeply intertwined with it. The memory subsystem provides the pages; the architecture layer handles the cache semantics.

## Summary

This post covered the foundation of Linux physical memory management: how the kernel slices RAM into fixed-size pages, what metadata it tracks per page via `struct page`, and how reference counting through `_refcount` governs a page's lifetime. We also looked at how dirty pages are guaranteed to be written back through three independent mechanisms, and where DMA coherency fits relative to the mm/ subsystem. The modern `struct folio` abstraction sits on top of all of this, providing a cleaner API for the rest of the kernel to work with.

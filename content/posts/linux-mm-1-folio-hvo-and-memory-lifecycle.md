+++
date = '2026-04-24T23:00:00+08:00'
draft = false
title = 'linux-mm[1]: folio, HVO, and the Memory Lifecycle'
tags = ["linux", "linux-subsystem-mm", "riscv"]
summary = 'A deep dive into struct folio, HugeTLB Vmemmap Optimization, reverse mapping, demand paging, and dirty page management in the Linux kernel.'
+++

## Overview

This post continues from linux-mm[0], which covered physical memory, struct page, and reference counting. Here we go deeper: the folio abstraction, HugeTLB Vmemmap Optimization (HVO), reverse mapping, demand paging, and dirty page management.

## 1. RISC-V Memory Models

Linux supports three physical memory models, and RISC-V supports all three:

- **FLATMEM**: assumes a single contiguous physical address range. Simple, fast, but inflexible.
- **SPARSEMEM**: divides physical memory into fixed-size sections. Handles holes and hotplug.
- **VMEMMAP**: an optimization on top of SPARSEMEM. Maps all struct pages into a contiguous virtual array (`vmemmap`), so `pfn_to_page(pfn)` is just an array lookup: `vmemmap + pfn`.

RISC-V uses VMEMMAP by default on 64-bit. The vmemmap array is allocated at boot time covering all possible PFNs — a one-time static allocation with no dynamic recursion.

## 2. struct folio vs HVO

These solve different problems:

- **struct folio** is a type-safety abstraction. It wraps the head page of a compound page and gives the kernel a typed handle so you cannot accidentally pass a tail page where a head is expected. Zero runtime cost — it is literally the same memory as `struct page`.
- **HVO (HugeTLB Vmemmap Optimization)** is a memory-saving optimization. For a 2MB HugeTLB page, the kernel normally needs 512 struct pages (512 × 64 bytes = 32KB, spanning 8 physical pages). HVO remaps the vmemmap entries for the 511 tail pages to all point at the same physical page as the head, reducing the physical cost from 8 pages to 1 page and freeing 7 pages back to the buddy allocator.

## 3. struct folio Memory Layout

`struct folio` shares its memory with `struct page`. The first fields overlap exactly:

```c
struct folio {
    union {
        struct page page;   /* head page — same memory */
        struct {
            unsigned long flags;
            struct list_head lru;
            /* ... */
        };
    };
    /* folio-specific fields stored in tail page slots: */
    /* __page_1 -> _deferred_list, _entire_mapcount, etc. */
    /* __page_2 -> _nr_pages_mapped, _pincount            */
};
```

The folio-specific fields that do not fit in the head page are stored in the slots of tail pages (`__page_1`, `__page_2`, `__page_3`). This works because tail pages are not used independently — their `struct page` slots are available for the compound page to repurpose.

## 4. HVO: vmemmap Remap and Fake Head Detection

For a 2MB HugeTLB page backed by 512 × 4KB physical pages, the vmemmap normally has 512 separate `struct page` entries. HVO remaps entries 1–511 to point at the same physical page as entry 0 (the head).

After HVO, all 512 vmemmap entries are virtually distinct addresses but physically the same page. The head `struct page` becomes effectively read-only and shared.

**Fake head detection**: when the kernel needs to identify whether a page is a real compound head or a HVO fake head, it checks three conditions simultaneously:

1. Address is 4KB aligned
2. `PG_head` flag is set
3. `page[1].compound_head` has bit 0 set (points back to head with LSB=1)

A real head satisfies all three. A fake HVO head also satisfies all three — which is intentional, because the kernel treats them the same for most purposes.

## 5. Freeing a HugeTLB Page: Restore vmemmap First

Before a HugeTLB page can be returned to the buddy allocator, HVO must be undone:

1. Allocate 7 new physical pages for the tail `struct page`s
2. Copy the head `struct page` content into each
3. Remap vmemmap entries 1–511 back to their own physical pages
4. TLB flush (the vmemmap virtual addresses now point to new physical pages)
5. Return the 512 × 4KB data pages to buddy

Why must vmemmap be restored? The buddy allocator writes LRU list pointers into each `struct page`. With HVO active, all tail pages share one read-only physical page — writes would fault or corrupt the head. Each `struct page` must be independently writable.

## 6. rmap: Reverse Mapping

Given a folio, how does the kernel find all the virtual addresses that map it? This is needed for reclaim (unmap before freeing) and for COW.

For anonymous pages:

```
folio->mapping  ->  anon_vma
anon_vma        ->  list of VMAs
for each VMA:
    vaddr = vma->vm_start + (folio->index - vma->vm_pgoff) * PAGE_SIZE
    ptep  = walk page tables to vaddr
    pte_clear(ptep)   /* unmap */
```

For file-backed pages:

```
folio->mapping  ->  address_space (inode)
address_space->i_mmap  ->  interval tree of VMAs
/* same vaddr calculation */
```

`folio->index` stores the page offset within the file or anonymous mapping, making the virtual address calculation O(1) per VMA.

## 7. VMA and Demand Paging

When a process calls `malloc()` or `mmap()`, the kernel only creates a VMA (`vm_area_struct`) — a descriptor of the virtual address range. No physical pages are allocated yet.

On first access to any address in that range:

1. CPU raises a page fault (no PTE exists)
2. Kernel finds the VMA covering the faulting address
3. If no VMA: `SIGSEGV`
4. If VMA exists: allocate a physical page, fill it (zero for anonymous, read from file for file-backed), install PTE
5. Return to userspace — the access succeeds transparently

This is demand paging. A process can `malloc(1GB)` and only touch 10MB — only 10MB of physical pages are ever allocated.

## 8. Dirty Page Management: Two Layers

A page becomes dirty when written. There are two independent dirty signals:

**Hardware layer — PTE dirty bit**: The CPU sets the dirty bit in the PTE automatically on any write. This is a hardware feature of the MMU. The kernel can clear it to track which pages have been written since the last check.

**Software layer — PG_dirty flag**: The kernel sets `PG_dirty` on the `struct page` (or folio) when it acknowledges the page needs writeback. The writeback subsystem uses this flag, not the PTE dirty bit directly.

The kernel periodically harvests the hardware dirty bits and promotes them to `PG_dirty`. On reclaim, a page with `PG_dirty` must be written back to its backing store before it can be freed.

## 9. xarray Tags: Efficient Dirty Page Discovery

The page cache uses an xarray (a radix-tree-like structure) to index pages by file offset. Each node in the tree carries tag bits, including a `DIRTY` tag.

When a page is marked dirty, the `DIRTY` tag is set on its xarray leaf node and propagated up to all ancestor nodes. This means:

- A writeback thread can skip entire subtrees that have no dirty pages
- Finding all dirty pages in a file is O(dirty pages × log n), not O(total pages)
- The same mechanism works for `WRITEBACK` tags to track in-flight I/O

```c
/* mark a folio dirty in the page cache */
filemap_dirty_folio(mapping, folio);
/* internally calls xas_set_mark(&xas, PAGECACHE_TAG_DIRTY) */
```

## 10. struct page Fixed Overhead and Boot-Time Allocation

Every physical page needs a `struct page`. At 64 bytes per struct and 4KB per page, the overhead is:

```
64 / 4096 = 1/64 ≈ 1.5% of RAM
```

A machine with 8GB of RAM spends about 128MB on `struct page` metadata.

All `struct page`s are allocated at boot time in the vmemmap array. This includes the `struct page`s for the physical pages that *hold* the vmemmap array itself — there is no infinite recursion because the allocation is done in one pass over the total PFN range, not lazily on demand.

```
boot: allocate vmemmap[0 .. max_pfn]
      this covers PFNs for RAM, for vmemmap itself, for everything
      no further allocation needed at runtime
```

## Summary

`struct folio` and HVO are complementary improvements to the same underlying `struct page` machinery. folio adds type safety at zero cost; HVO reduces the physical memory overhead of HugeTLB pages by sharing vmemmap entries. Together with rmap, demand paging, and the xarray dirty-tag mechanism, they form the core of how Linux manages the full lifecycle of a physical page — from allocation through mapping, dirtying, writeback, and eventual reclaim.

# Chapter 11 — Memory Allocation APIs

> **Goal:** choose the right allocator every time, understand GFP flags at the level where you
> can debug an OOM, and know exactly what happens inside `kmalloc()`.

---

## Theory & First Principles

*Read this section before the API tables. Kernel allocators are not arbitrary — every design
choice is a response to a theorem or a measured pathology. If you understand the theory you
can derive the API.*

### T.0 — Start here: why the kernel cannot just use `malloc`

In userspace you call `malloc` and it works. Ask what has to be true for that, and every
assumption breaks in the kernel:

| `malloc` assumes | In the kernel |
|---|---|
| It can call `mmap`/`brk` to get more memory | **It *is* what backs `mmap`.** There is nothing underneath |
| It can block while the OS finds a page | It may be in an interrupt handler, where blocking is illegal |
| Failure means the process dies | Failure must be *handled*, on every path |
| Any address is as good as any other | A DMA buffer may need to be **physically contiguous** and below a device's address limit |
| Virtual memory is effectively unlimited | Physical memory is the actual resource |
| One allocator is fine | Allocation happens in wildly different contexts with different rules |

**So the kernel needs several allocators, and it needs the caller to say which situation they
are in.** That is what the `GFP_` flags are — not options, but a statement about your context:

```c
kmalloc(size, GFP_KERNEL);   /* "I can sleep. Reclaim memory if you must." */
kmalloc(size, GFP_ATOMIC);   /* "I CANNOT sleep. Use emergency reserves.
                                 I accept a much higher failure rate." */
kmalloc(size, GFP_NOIO);     /* "I am IN the I/O path. Do not recurse into I/O
                                 to free memory, or you will deadlock on me." */
```

That third one is worth pausing on, because it is the kind of constraint that does not exist
in userspace at all. A filesystem writing back a page calls the block layer, which allocates
a request. If that allocation triggers reclaim, and reclaim decides to write back a page, it
calls the filesystem — **which is waiting for the allocation.** Deadlock. `GFP_NOIO` is the
caller saying "you may not recurse into me," and forgetting it is a real and very hard-to-
debug class of hang.

**Now the deeper problem: physical contiguity.** A device doing DMA often needs *N*
consecutive physical pages. After the machine has been up for a week, physical memory looks
like this:

```
 page frames:  [U][ ][U][U][ ][U][ ][ ][U][ ][U][U][ ][U][ ][U]
                                  ↑
   40% free, but the largest contiguous run is TWO PAGES.
   A driver asking for 64 contiguous pages FAILS, on a machine with
   gigabytes free. This is EXTERNAL FRAGMENTATION and it is not a
   theoretical concern -- it is why camera and network drivers fail
   after uptime.
```

Userspace never sees this because the MMU hides it: any set of physical pages can be made
virtually contiguous. **A device has no MMU** (unless there is an IOMMU — Ch. 36), so for
DMA the fragmentation is real and unfixable by indirection.

That single requirement forces the whole design:

```
  buddy allocator      power-of-two PAGE runs. Coalescing is O(1) and
  (mm/page_alloc.c)    ALWAYS POSSIBLE because of the power-of-two invariant.
         │             This is what fights fragmentation.          §T.2
         │
         ├─► slab/SLUB   caches of same-sized OBJECTS carved from buddy pages.
         │  (mm/slub.c)  Buddy's granularity is a page; most allocations are
         │               32-256 bytes. Without this you waste 90%+.     §T.4
         │                  └─► kmalloc()  -- generic size-class caches
         │
         └─► vmalloc     stitches NON-contiguous pages into a contiguous
            (mm/vmalloc.c) VIRTUAL range. Solves fragmentation for CPU access;
                           useless for DMA. Costs page tables + TLB entries.
```

**Three questions answer "which allocator", and they are the whole practical content of this
chapter:**

1. **Can I sleep here?** No → `GFP_ATOMIC`, and handle failure. (Ch. 00 §T.9)
2. **Do I need *physical* contiguity?** Only for DMA without an IOMMU, or for page-table-like
   structures. Otherwise `vmalloc`/`kvmalloc` is fine.
3. **How big, and how often?** Small and frequent → slab. Page-multiples → buddy directly.
   Large and rare → `vmalloc`. Same-type and very frequent → your own
   `kmem_cache_create`.

```bash
cd ~/src/linux
cat /proc/buddyinfo       # free runs by order -- watch high orders vanish over uptime
sudo slabtop -o -s c      # where the slab memory actually is
cat /proc/meminfo | grep -E 'Slab|Vmalloc|MemFree'
```

---

### T.1 The dynamic storage allocation problem

Formally: maintain a set of free intervals over an address space; service a request stream
`alloc(size)` / `free(ptr)` online, minimizing (a) **fragmentation**, (b) **time per
operation**, (c) **metadata overhead**, and (d) **cache/TLB footprint**. These four objectives
are mutually antagonistic. Every allocator is a point on that Pareto surface.

Two kinds of waste:

- **External fragmentation** — total free memory is sufficient but no single *contiguous*
  run is. This is what kills the kernel: a driver needs order-4 (64 KiB contiguous) for DMA
  and the machine has 8 GiB free, all in order-0 scraps.
- **Internal fragmentation** — you asked for 100 bytes, the allocator gave you a 128-byte
  bucket. 28 bytes are unreachable but charged to you.

**Robson's bounds (1971/1977)** are the depressing theoretical backdrop: for any online
allocator handling requests in the size range `[1, n]`, the worst-case memory needed is
`Ω(M · log n)` where `M` is the live high-water mark. You *cannot* build an online allocator
with a constant-factor worst-case guarantee. Therefore all real allocators optimize for
*observed* request distributions, not worst case. This is why the kernel has **five**
different allocators instead of one: each one specializes on a different request distribution.

### T.2 Why a buddy system? (the contiguity theorem)

Knuth (*TAOCP* Vol. 1, §2.5) analyses the classic families:

| Policy | Search cost | Fragmentation | Coalescing cost |
|---|---|---|---|
| First-fit | O(n) | good in practice ("50% rule") | O(n) boundary search |
| Best-fit | O(n) or O(log n) | leaves slivers | O(n) |
| Segregated fit | O(1) | bucket rounding | O(1) but no merging |
| **Buddy** | **O(log n)** | ≤ 2× internal | **O(1) merge test** |

The buddy system's unique property: the buddy of a block at PFN `p` with order `k` is
`p XOR (1 << k)`. Coalescing is therefore a *single XOR and a bit test* — no search, no free
list scan, no boundary tags. That O(1) merge is why Linux uses it for the physical page
allocator, where merging must happen constantly to keep higher orders available.

The price is **internal fragmentation bounded by 2×** (a 17-page request becomes order-5 =
32 pages) — which is why nothing calls the buddy allocator directly for small objects.

**Knuth's "50% rule"**: in steady state with random alloc/free, the number of free blocks
tends to half the number of allocated blocks. This is why free-list scanning costs grow and
why the kernel keeps *per-order, per-migratetype, per-CPU* lists rather than one global list.

### T.3 Fragmentation as a control problem — migratetypes and anti-fragmentation

Fragmentation is irreversible unless pages can **move**. This is the deep insight behind
Mel Gorman's anti-fragmentation work (2006–2007) and it reframed the problem from
"allocation policy" to "mobility classification":

- A page holding a user-space anonymous mapping is **movable**: copy it elsewhere, update the
  PTE, done.
- A page holding a slab object is **unmovable**: kernel pointers to it exist everywhere and
  cannot be found or rewritten.
- A page holding clean page-cache data is **reclaimable**: drop it and re-read from disk.

The kernel therefore *segregates by mobility* (`MIGRATE_UNMOVABLE`, `MIGRATE_MOVABLE`,
`MIGRATE_RECLAIMABLE`, `MIGRATE_CMA`, `MIGRATE_ISOLATE`). The invariant being maintained is:
**keep unmovable allocations clustered into as few pageblocks as possible**, so the remaining
pageblocks stay defragmentable by compaction.

This is why `GFP_KERNEL` (unmovable slab) and `GFP_HIGHUSER_MOVABLE` (user pages) draw from
different free lists even though they're the same physical RAM. When you pick a GFP flag you
are making a *mobility declaration*, and getting it wrong degrades the whole machine's
long-term ability to allocate huge pages.

**Fallback and stealing:** when a migratetype's lists are empty, the allocator steals a whole
pageblock from another type rather than a single page — stealing one page would permanently
pollute a clean block. This "steal whole blocks" rule is pure fragmentation-theory.

### T.4 Why slab? Bonwick's object-caching argument

Jeff Bonwick's 1994 USENIX paper *"The Slab Allocator: An Object-Caching Kernel Memory
Allocator"* made three claims that the Linux kernel still runs on:

1. **Initialization is often more expensive than allocation.** A `struct inode` has locks,
   lists, and atomics to initialize. If the object is *freed in an initialized state* and
   handed back initialized, you amortize construction across the object's whole lifetime.
   This is the `ctor` argument to `kmem_cache_create()`.
   *(Linux has since narrowed constructor use, because a partially-initialized freed object
   is a security liability — but the theory is why the hook exists.)*

2. **Segregating by type eliminates rounding waste and improves locality.** All objects in a
   cache are the same size, so the free list is trivially O(1) and there is *zero* internal
   fragmentation. Objects of the same type are used together temporally, so keeping them on
   the same pages improves TLB behaviour.

3. **The free list can live inside the free objects.** A free object's memory is by definition
   unused — store the "next free" pointer there. Metadata overhead for free objects is
   therefore **zero**. (This is also why heap-spray exploits target the freepointer, and why
   `CONFIG_SLAB_FREELIST_HARDENED` XORs it with a per-cache secret and its own address.)

Bonwick also introduced **cache colouring**: offsetting the start of each slab by a different
multiple of the cache-line size so that the same field of objects in different slabs does not
map to the same L1 set. On modern highly-associative caches with hashed indexing this matters
much less, and SLUB dropped it — an example of a theoretical optimization invalidated by
hardware evolution. Knowing *why* it was dropped is senior-level.

### T.5 Why SLUB won: the magazine/queue critique

SLAB (the original Linux implementation) followed Bonwick's later *magazine* design: per-CPU
arrays of object pointers, plus per-node shared arrays, plus alien caches for NUMA. This gives
great throughput but has a pathology: **the queues themselves consume memory proportional to
`nr_caches × nr_cpus × queue_len`**, and on a 256-core machine with thousands of caches that
is gigabytes of metadata that is invisible to reclaim. It also makes the free path's worst
case unbounded (flushing queues).

SLUB (Christoph Lameter, 2007) discards queues entirely. Its theory:

- Keep a **single per-CPU active slab** plus a short per-CPU partial list.
- Represent the free list as an intrusive singly linked chain *within the slab*.
- Make the fast path a single `this_cpu_cmpxchg_double()` on (freelist, tid) — lock-free,
  no interrupt disabling, no queue memory.
- Push the hard cases (slab exhausted, NUMA remote, debugging) into a slow path guarded by
  a per-node list lock.

The result: O(1) alloc/free, metadata proportional to *live slabs* rather than to
*caches × CPUs*, and merged caches (`slab_merge`) that further shrink the cache count.
In 6.8 SLAB was deleted outright — a rare event in the kernel and a case study in how
*metadata scalability* eventually beats *raw fast-path throughput*.

### T.6 The virtual-memory escape hatch, and its cost model

`vmalloc()` exists because the MMU lets you *manufacture* contiguity: gather arbitrary
physical pages and map them to consecutive virtual addresses. This trades away:

- **TLB entries.** A 2 MiB `kmalloc` region on the direct map may be covered by a single
  1 GiB or 2 MiB huge TLB entry. The same region in vmalloc space needs 512 4 KiB entries.
  On a TLB-pressured workload this is measurable.
- **Page-table walk and shootdown cost.** Setting up and tearing down PTEs is expensive, and
  unmapping requires TLB invalidation across all CPUs (IPIs). Linux therefore uses **lazy
  TLB flushing** in vmalloc: freed regions accumulate until a batch threshold, then one
  global flush. So `vfree()` does not immediately return address space — an important fact
  when debugging vmalloc-space exhaustion on 32-bit.
- **DMA usability.** Devices see *physical* addresses (or IOVAs). A vmalloc buffer is not
  physically contiguous, so `virt_to_phys()` on it is meaningless and `dma_map_single()` on
  it is a bug. You must walk it with `vmalloc_to_page()` and build a scatterlist.

The principled statement: **`kmalloc` allocates *memory*; `vmalloc` allocates *address
space* and backs it with memory.** They fail for different reasons and at different scales.

### T.7 GFP flags are a re-entrancy and deadlock algebra, not a "hint"

This is the part most people get wrong. Consider: a filesystem is writing back a page. It
needs a small buffer, so it calls `kmalloc(GFP_KERNEL)`. Memory is tight, so the allocator
enters **direct reclaim**, which decides to write back dirty pages, which calls *into the same
filesystem*, which tries to take a lock **the original caller already holds**. Deadlock.

So GFP flags encode a **capability lattice** describing what the allocator is permitted to
re-enter:

```
                       GFP_KERNEL          (may sleep, may do I/O, may re-enter FS)
                           │  drop __GFP_FS
                       GFP_NOFS            (may sleep, may do I/O, must NOT re-enter FS)
                           │  drop __GFP_IO
                       GFP_NOIO            (may sleep, must NOT start I/O)
                           │  drop __GFP_DIRECT_RECLAIM
                       GFP_NOWAIT          (must not sleep at all)
                           │  add  __GFP_HIGH
                       GFP_ATOMIC          (must not sleep; may use emergency reserves)
```

Moving *down* this lattice makes allocation more likely to fail but safer against deadlock.
Two independent axes are being expressed:

1. **Blocking capability** — determined by your *execution context* (can you call
   `schedule()`?). Hard constraint; violating it is an immediate bug (`might_sleep()` fires).
2. **Re-entrancy capability** — determined by *which locks you hold*. Violating it is a
   latent deadlock that appears under memory pressure at a customer site at 3 a.m.

Axis 2 is a property of the **call stack**, not of the call site — which is precisely why
threading `GFP_NOFS` through twenty functions as a parameter was always the wrong design.
The modern fix is the **scoped** API (`memalloc_nofs_save()/restore()`), which stores the
constraint in `current->flags` so it applies to the entire dynamic extent, including code
that doesn't know it's being called from a filesystem. That is a shift from a *lexical* to a
*dynamic* scoping model, and it is the correct one for a re-entrancy constraint.

`__GFP_NOFAIL` deserves special mention: it converts allocation failure into an infinite
loop. Theoretically it makes the system **unable to make forward progress** if the request
can never be satisfied, converting a recoverable error into a livelock. It exists only
because some legacy call sites have no error path. Adding a new one requires a maintainer
argument.

### T.8 Watermarks as a feedback control system

Reclaim is a control loop, and the zone watermarks (`min`, `low`, `high`) are its setpoints:

```
free pages
   ▲
   │ ─────────────── high  ── kswapd stops
   │      hysteresis band (kswapd runs here, asynchronously, cheap)
   │ ─────────────── low   ── kswapd wakes
   │      danger band (allocators enter DIRECT reclaim: latency spike)
   │ ─────────────── min   ── only __GFP_HIGH/ATOMIC may go below
   │      reserve (PF_MEMALLOC, network RX during swap-over-network)
   0
```

The gap between `low` and `high` is deliberate **hysteresis** — without it, kswapd would
oscillate (thrash) around a single threshold. The reserve below `min` exists to break a
**circular dependency**: reclaiming memory (writing to swap over the network) itself requires
memory. Without a reserve you get a *memory deadlock*, which is why `PF_MEMALLOC` exists: it
marks a task as "already reclaiming, let it dip into reserves, do not recurse".

`min_free_kbytes` sets `min`; scaled per zone by size. Tuning it up trades usable memory for
reduced direct-reclaim latency — a classic latency/throughput dial you will actually turn in
production.

**Direct reclaim is the single most important latency source in Linux memory management.**
When you see p99 spikes with no obvious cause, look at `allocstall_*` and
`pgscan_direct` in `/proc/vmstat`. This is the number one thing a senior engineer checks.

### T.9 The NUMA dimension

On a multi-socket machine, "memory" is not uniform: remote-node access costs 1.5–2.5×.
Allocators therefore become *placement* algorithms. Linux's policy layer (`mm/mempolicy.c`)
offers `MPOL_DEFAULT` (local-first), `MPOL_BIND`, `MPOL_INTERLEAVE`,
`MPOL_PREFERRED[_MANY]`, plus weighted interleave (6.9+) for tiered memory/CXL.

The theoretical tension: **locality vs. balance**. Strict local allocation gives best latency
but causes one node to hit its watermarks while another is empty, triggering reclaim on a
machine that is 50% free. `zone_reclaim_mode` and, more recently, NUMA balancing
(automatic page migration based on fault sampling) are attempts to close this loop
dynamically. With CXL memory tiers this has become a *tiering* problem — hot/cold page
classification (`kernel/sched/fair.c` NUMA hinting faults, `mm/memory-tiers.c`).

### T.10 Deriving the API from the theory

You can now re-derive the decision table instead of memorizing it:

| Question | Theory that answers it |
|---|---|
| Do I need physical contiguity? | Only DMA and page-table hardware do. → `kmalloc`/`alloc_pages` vs `vmalloc` |
| Is my size > 2 pages? | Buddy internal fragmentation bound is 2× → prefer `vmalloc`/`kvmalloc` |
| Is my size caller-controlled? | Robson: unbounded size range → segregate → `kvmalloc` + overflow-safe helpers |
| Are these objects identical and hot? | Bonwick: type segregation removes rounding and improves locality → own `kmem_cache` |
| Am I on a reclaim path? | Re-entrancy lattice → `GFP_NOFS`/`NOIO`, or better, `memalloc_*_save()` |
| Must this never fail? | Forward-progress requirement → `mempool` (pre-reserved), **not** `__GFP_NOFAIL` |
| Is the refcount/alloc a cacheline hotspot? | Amdahl on the shared line → per-CPU (`alloc_percpu`, `percpu_ref`) |

---

## 1. Concept — the allocator hierarchy

Everything in kernel memory management is built in layers. You must know which layer you're at.

```
                     ┌───────────────────────────────────────┐
   Physical RAM  →   │  memblock (early boot only)           │
                     └──────────────┬────────────────────────┘
                                    │ handed over at mm_core_init()
                     ┌──────────────▼────────────────────────┐
   Layer 1           │  Buddy allocator / page allocator     │  granularity: 2^order pages
                     │  alloc_pages(), __free_pages()        │  mm/page_alloc.c
                     └──────┬─────────────────┬──────────────┘
                            │                 │
              ┌─────────────▼──────┐   ┌──────▼─────────────────┐
   Layer 2    │ SLUB (slab alloc)  │   │ vmalloc                │
              │ kmem_cache_alloc() │   │ virtually contiguous   │
              │ kmalloc()          │   │ mm/vmalloc.c           │
              │ mm/slub.c          │   └────────────────────────┘
              └─────────┬──────────┘
                        │
              ┌─────────▼────────────────────────────────────┐
   Layer 3    │ mempool, percpu, dmapool, genpool, kvmalloc  │
              └──────────────────────────────────────────────┘
```

### 1.1 The decision table (tape this to your monitor)

| Need | Use | Notes |
|---|---|---|
| < ~8 KiB, physically contiguous | `kmalloc()` | the default; power-of-2 buckets |
| Many identical objects, hot path | `kmem_cache_create()` + `kmem_cache_alloc()` | own cache, constructor, `/proc/slabinfo` |
| Large (> 2 pages), contiguity irrelevant | `vmalloc()` / `kvmalloc()` | slow, TLB pressure, can't DMA |
| Size from userspace / variable, maybe large | `kvmalloc()` | tries kmalloc, falls back to vmalloc |
| Whole pages | `alloc_pages()`, `__get_free_pages()` | returns `struct page *` / virtual addr |
| One page, zeroed | `get_zeroed_page(GFP_KERNEL)` | |
| DMA-able buffer | `dma_alloc_coherent()` / `dma_map_*` | **never** `kmalloc` + `virt_to_phys` |
| Small DMA descriptors, many | `dma_pool_create()` | sub-page DMA allocations with alignment |
| Must not fail on reclaim path | `mempool_create()` | pre-reserved elements |
| Per-CPU counters/state | `alloc_percpu()` / `DEFINE_PER_CPU` | Ch. 16 |
| Tied to a device lifetime | `devm_kmalloc()` | auto-freed on driver detach (Ch. 28) |
| Userspace-controlled size | `kmalloc_array()` / `kcalloc()` | **overflow-safe** — see §4 |

> **Senior instinct:** if you see `kmalloc(n * sizeof(x), ...)` in review, flag it.
> Overflow → tiny allocation → heap overflow → CVE. Use `kmalloc_array()`.

---

## 2. Internals

### 2.1 GFP flags — the real semantics

`gfp_t` is a bitmask defined in `include/linux/gfp_types.h`. It answers three questions:

1. **Where** can memory come from? (zone modifiers)
2. **What** is the allocator allowed to do to get it? (action modifiers)
3. **How hard** should it try? (reclaim/retry policy)

```c
/* The composite flags you actually use: */
GFP_KERNEL      /* __GFP_RECLAIM | __GFP_IO | __GFP_FS  — MAY SLEEP */
GFP_NOWAIT      /* no reclaim, no sleep, may fail. Atomic-ish but no emergency reserves */
GFP_ATOMIC      /* __GFP_HIGH — may dip into reserves, will not sleep */
GFP_NOIO        /* __GFP_RECLAIM only — may reclaim but NOT start I/O */
GFP_NOFS        /* reclaim + I/O but not filesystem re-entry */
GFP_USER        /* user-space allocations, subject to cpuset */
GFP_DMA32       /* addressable below 4 GiB */
GFP_HIGHUSER_MOVABLE  /* page cache / anon pages; movable for compaction */
```

Modifier bits worth knowing:

| Flag | Meaning |
|---|---|
| `__GFP_ZERO` | zero the memory (`kzalloc` = `kmalloc(.., GFP_KERNEL\|__GFP_ZERO)`) |
| `__GFP_NOWARN` | suppress the allocation-failure splat (use when you handle failure) |
| `__GFP_NORETRY` | try once-ish, then fail. Pair with a fallback |
| `__GFP_RETRY_MAYFAIL` | try hard, but *can* still fail — the modern "big allocation" flag |
| `__GFP_NOFAIL` | loop forever. **Almost always wrong.** Requires maintainer justification |
| `__GFP_HIGHMEM` | 32-bit legacy; allow highmem zone |
| `__GFP_ACCOUNT` | charge to the memory cgroup (`kmalloc`-based objects from user requests) |
| `__GFP_COMP` | compound page (needed for >order-0 pages you'll refcount as a unit) |

**Context → flag mapping:**

```c
if (in_atomic_or_irq_context())      use GFP_ATOMIC or GFP_NOWAIT;
else if (on_the_writeback_path())    use GFP_NOFS or GFP_NOIO;
else                                 use GFP_KERNEL;
```

The modern replacement for sprinkling `GFP_NOFS` everywhere is the **scoped API**:

```c
unsigned int flags = memalloc_nofs_save();
/* everything here implicitly drops __GFP_FS */
memalloc_nofs_restore(flags);
```
Also `memalloc_noio_save/restore()` and `memalloc_noreclaim_save/restore()`.
See `Documentation/core-api/gfp_mask-from-fs-io.rst`. This is what filesystem and
block maintainers expect in 2024+ code.

### 2.2 The buddy allocator

`mm/page_alloc.c`. Free memory is kept in per-zone, per-order, per-migratetype free lists:

```c
struct free_area {
    struct list_head free_list[MIGRATE_TYPES];
    unsigned long    nr_free;
};
struct zone {
    ...
    struct free_area free_area[NR_PAGE_ORDERS];   /* orders 0..10 (MAX_PAGE_ORDER) */
};
```

Allocation of order *N*: pop from `free_area[N]`; if empty, split a block from order *N+1*
(recursively), putting the buddy half back. Free: if the buddy of the block is also free and
same order, coalesce upward. Buddy address is `page_pfn ^ (1 << order)` — one XOR, which is
why it's fast.

**Migratetypes** (`MIGRATE_UNMOVABLE`, `MIGRATE_MOVABLE`, `MIGRATE_RECLAIMABLE`,
`MIGRATE_CMA`, `MIGRATE_ISOLATE`) exist to fight fragmentation: keep unmovable kernel
allocations clustered so that movable user pages can be compacted into huge pages.
Fragmentation is *the* long-term enemy; `CONFIG_COMPACTION` and `kcompactd` fight it.

Per-CPU page lists (`struct per_cpu_pages`) cache order-0 pages to avoid the zone lock —
this is why order-0 alloc is ~50 ns and order-4 is ~1 µs.

Watermarks per zone: `min`, `low`, `high`.
- below `low` → wake `kswapd` (background reclaim)
- below `min` → direct reclaim in the allocating task (**this is your latency spike**)
- `__GFP_HIGH`/`GFP_ATOMIC` may go below `min` into reserves

```bash
cat /proc/zoneinfo | grep -A4 'Node 0, zone'
cat /proc/buddyinfo     # free blocks per order — fragmentation at a glance
cat /proc/pagetypeinfo  # per migratetype
```

### 2.3 SLUB — what `kmalloc()` actually does

Linux has had SLAB, SLOB and SLUB. **As of 6.8, SLAB is removed; SLUB is the only allocator.**
Don't learn SLAB internals; learn SLUB.

Core idea: a `kmem_cache` owns **slabs** (one or more contiguous pages). Each slab is carved
into equal-size objects. Free objects form a singly linked list *inside the free objects
themselves* (the "freepointer"), so zero metadata overhead per free object.

```c
struct kmem_cache {
    struct kmem_cache_cpu __percpu *cpu_slab;  /* fast path */
    slab_flags_t flags;
    unsigned int  size;        /* object size incl. metadata */
    unsigned int  object_size; /* what the caller asked for */
    unsigned int  offset;      /* freepointer offset inside object */
    struct kmem_cache_order_objects oo;
    struct kmem_cache_node *node[MAX_NUMNODES];
    const char *name;
    ...
};

struct kmem_cache_cpu {
    void **freelist;      /* lockless per-CPU freelist */
    unsigned long tid;    /* for cmpxchg_double race detection */
    struct slab *slab;
    struct slab *partial;
};
```

**Fast path** (`slab_alloc_node()` in `mm/slub.c`): read per-CPU `freelist`, take head,
update with `this_cpu_cmpxchg_double(freelist, tid)`. No locks, no interrupts disabled.
~15–25 ns.

**Slow path:** per-CPU slab exhausted → grab from the per-CPU partial list → per-node partial
list (`n->list_lock`) → allocate a new slab from the buddy allocator.

`kmalloc()` is a thin wrapper over an array of generic caches:

```c
/* mm/slab_common.c */
struct kmem_cache *kmalloc_caches[NR_KMALLOC_TYPES][KMALLOC_SHIFT_HIGH + 1];
/* types: KMALLOC_NORMAL, KMALLOC_DMA, KMALLOC_RECLAIM, KMALLOC_CGROUP, KMALLOC_RANDOM_* */
```

Buckets are powers of two (8, 16, 32, …, 8192) **plus** two extras: 96 and 192.
`kmalloc(100)` therefore consumes 128 bytes — 28% waste. For hot structures, either pad to
a power of two deliberately or create your own `kmem_cache`.

Above `KMALLOC_MAX_CACHE_SIZE` (typically 8 KiB = 2 pages), `kmalloc()` falls through to the
page allocator directly (`__kmalloc_large_node()`).

**Hardening features you should know exist:**
- `CONFIG_SLAB_FREELIST_HARDENED` — freepointer XOR-obfuscated with a per-cache secret
- `CONFIG_SLAB_FREELIST_RANDOM` — randomize initial freelist order
- `CONFIG_RANDOM_KMALLOC_CACHES` (6.6+) — multiple random `kmalloc` cache sets, breaking
  deterministic heap grooming
- `CONFIG_SLAB_BUCKETS` / `kmalloc_nolock()` (6.1x) — separate buckets for user-controlled sizes
- `kfree_sensitive()` — zeroes before freeing

### 2.4 `vmalloc()`

Allocates **virtually** contiguous, **physically** scattered memory from the vmalloc arena.

```c
void *p = vmalloc(1 << 20);   /* 1 MiB */
vfree(p);
```

Cost: page-table setup per page, TLB pressure, and a global lock historically (now a
per-CPU KVA allocator, `mm/vmalloc.c`, much better since 5.2 and rewritten again in 6.9).
Lazy TLB flushing means freed vmalloc areas linger — `vmalloc` is *not* for hot paths.

**Never** DMA from a `vmalloc()` buffer via `dma_map_single()` — it is not physically
contiguous and `virt_to_phys()` is invalid on vmalloc addresses. Use `vmalloc_to_page()`
+ scatter-gather, or `dma_alloc_coherent()`.

`kvmalloc(size, GFP_KERNEL)` is the pragmatic default for "big and size-is-not-constant":

```c
void *buf = kvmalloc_array(n, sizeof(*buf), GFP_KERNEL);
...
kvfree(buf);
```
It tries `kmalloc(__GFP_NORETRY | __GFP_NOWARN)` first, falls back to `vmalloc`.

### 2.5 Specialized allocators

**mempool** — guarantees progress on reclaim paths (block I/O, writeback):

```c
mempool_t *pool = mempool_create_slab_pool(MIN_NR, my_cache);
void *e = mempool_alloc(pool, GFP_NOIO);  /* dips into reserve rather than failing */
mempool_free(e, pool);
```

**dmapool** — sub-page, alignment-constrained, DMA-coherent chunks:

```c
struct dma_pool *p = dma_pool_create("desc", dev, 64, 64 /*align*/, 0 /*boundary*/);
void *v = dma_pool_alloc(p, GFP_ATOMIC, &dma_handle);
```

**genpool** (`lib/genalloc.c`) — manage an arbitrary on-chip SRAM region.

**CMA** — Contiguous Memory Allocator: reserves a region at boot that is used for movable
pages normally, but can be evacuated to satisfy large contiguous DMA requests
(`dma_alloc_contiguous()`). Essential on ARM SoCs with cameras/codecs.

---

## 3. Practice

### 3.1 A module that exercises every allocator

```c
// SPDX-License-Identifier: GPL-2.0
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/slab.h>
#include <linux/vmalloc.h>
#include <linux/gfp.h>
#include <linux/mm.h>

struct widget {
	u64 id;
	char name[24];
	struct list_head node;
};

static struct kmem_cache *widget_cache;

static void widget_ctor(void *obj)
{
	struct widget *w = obj;

	INIT_LIST_HEAD(&w->node);   /* runs once per object when slab is created */
}

static int __init alloc_demo_init(void)
{
	void *k, *v, *kv;
	struct page *pg;
	struct widget *w;

	/* 1. kmalloc: physically contiguous, small */
	k = kmalloc(100, GFP_KERNEL);
	if (!k)
		return -ENOMEM;
	pr_info("kmalloc(100)      -> %p, actual size %zu\n", k, ksize(k));

	/* 2. page allocator: order-2 = 4 pages = 16 KiB */
	pg = alloc_pages(GFP_KERNEL | __GFP_ZERO, 2);
	if (!pg)
		goto err_k;
	pr_info("alloc_pages(2)    -> pfn %lu, virt %p\n",
		page_to_pfn(pg), page_address(pg));

	/* 3. vmalloc: virtually contiguous, 1 MiB */
	v = vmalloc(1 << 20);
	if (!v)
		goto err_pg;
	pr_info("vmalloc(1MiB)     -> %p (is_vmalloc_addr=%d)\n", v, is_vmalloc_addr(v));

	/* 4. kvmalloc: the pragmatic default */
	kv = kvmalloc(64 * 1024, GFP_KERNEL);
	if (!kv)
		goto err_v;
	pr_info("kvmalloc(64KiB)   -> %p (vmalloc? %d)\n", kv, is_vmalloc_addr(kv));

	/* 5. dedicated slab cache with a constructor */
	widget_cache = kmem_cache_create("widget", sizeof(struct widget),
					 0, SLAB_HWCACHE_ALIGN | SLAB_PANIC,
					 widget_ctor);
	w = kmem_cache_alloc(widget_cache, GFP_KERNEL);
	if (!w)
		goto err_cache;
	w->id = 42;
	pr_info("kmem_cache_alloc  -> %p, list already inited: %d\n",
		w, list_empty(&w->node));

	kmem_cache_free(widget_cache, w);
	kmem_cache_destroy(widget_cache);
	kvfree(kv);
	vfree(v);
	__free_pages(pg, 2);
	kfree(k);
	return 0;

err_cache:
	kmem_cache_destroy(widget_cache);
	kvfree(kv);
err_v:
	vfree(v);
err_pg:
	__free_pages(pg, 2);
err_k:
	kfree(k);
	return -ENOMEM;
}

static void __exit alloc_demo_exit(void) { pr_info("bye\n"); }

module_init(alloc_demo_init);
module_exit(alloc_demo_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Allocator tour");
```

Note the **goto error ladder** — this is *the* kernel error-handling idiom. Labels are named
after what they undo, and unwind in reverse order. Learn it now; you'll write it forever.

### 3.2 Observe the allocators

```bash
sudo cat /proc/slabinfo | head -20
sudo slabtop -s c                      # sorted by cache size
cat /proc/vmallocinfo | sort -k2 -n | tail
cat /proc/meminfo | egrep 'Slab|SReclaimable|SUnreclaim|VmallocUsed'
cat /proc/buddyinfo

# who is allocating? (needs CONFIG_PAGE_OWNER=y and page_owner=on)
sudo cat /sys/kernel/debug/page_owner | head -40

# slab allocation tracing
sudo bpftrace -e 'kprobe:kmem_cache_alloc { @[kstack] = count(); }'
sudo perf trace -e 'kmem:*' -a sleep 2
```

### 3.3 Force an OOM (in a VM only!)

```bash
# In your QEMU guest from Ch. 04:
stress-ng --vm 4 --vm-bytes 90% --timeout 30s
dmesg | grep -A40 'Out of memory'
```
Read the OOM report: it shows per-zone free, watermarks, slab usage, and the chosen victim's
`oom_score`. Being able to read this report fluently is a senior-level skill.

---

## 3.5 Extended practice

### Lab 11.A — Measure every allocator on the same axis

```c
// SPDX-License-Identifier: GPL-2.0
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/slab.h>
#include <linux/vmalloc.h>
#include <linux/ktime.h>

#define ITERS 200000

static struct kmem_cache *cache;

#define BENCH(name, alloc_expr, free_stmt)                       \
do {                                                             \
	u64 t0 = ktime_get_ns();                                 \
	int i;                                                   \
	for (i = 0; i < ITERS; i++) {                            \
		void *p = (alloc_expr);                          \
		if (!p) { pr_warn(name ": failed\n"); break; }    \
		free_stmt;                                       \
	}                                                        \
	pr_info("%-22s %6llu ns/op\n", name,                     \
		(ktime_get_ns() - t0) / ITERS);                  \
} while (0)

static int __init alloc_bench_init(void)
{
	cache = kmem_cache_create("bench64", 64, 0, SLAB_HWCACHE_ALIGN, NULL);
	if (!cache)
		return -ENOMEM;

	BENCH("kmalloc(64)",        kmalloc(64, GFP_KERNEL),        kfree(p));
	BENCH("kzalloc(64)",        kzalloc(64, GFP_KERNEL),        kfree(p));
	BENCH("kmem_cache_alloc",   kmem_cache_alloc(cache, GFP_KERNEL),
								    kmem_cache_free(cache, p));
	BENCH("kmalloc(4096)",      kmalloc(4096, GFP_KERNEL),      kfree(p));
	BENCH("get_zeroed_page",    (void *)get_zeroed_page(GFP_KERNEL),
								    free_page((unsigned long)p));
	BENCH("alloc_pages(order=4)", page_address(alloc_pages(GFP_KERNEL, 4)),
								    free_pages((unsigned long)p, 4));
	BENCH("vmalloc(64K)",       vmalloc(65536),                 vfree(p));
	BENCH("kvmalloc(64K)",      kvmalloc(65536, GFP_KERNEL),    kvfree(p));
	BENCH("kmalloc(100) waste", kmalloc(100, GFP_KERNEL),       kfree(p));

	/* How much did kmalloc(100) actually cost us? */
	{
		void *p = kmalloc(100, GFP_KERNEL);

		pr_info("kmalloc(100): ksize=%zu  (%zu bytes wasted, %zu%%)\n",
			ksize(p), ksize(p) - 100, (ksize(p) - 100) * 100 / ksize(p));
		kfree(p);
	}
	kmem_cache_destroy(cache);
	return 0;
}
static void __exit alloc_bench_exit(void) { }
module_init(alloc_bench_init); module_exit(alloc_bench_exit);
MODULE_LICENSE("GPL");
```

Expected ordering (ns/op, typical x86-64): `kmem_cache_alloc` ≈ `kmalloc(64)` ≈ 20–40 ≪
`alloc_pages(4)` ≈ 300–1500 ≪ `vmalloc(64K)` ≈ 5000–20000. **The vmalloc number is the one
that should change your behaviour** — it is 100–500× a `kmalloc`, and that is T.6's TLB and
page-table cost made visible.

### Lab 11.B — Watch fragmentation happen, then fix it

```bash
# Baseline
cat /proc/buddyinfo
cat /proc/pagetypeinfo | head -30
grep -E 'compact_|pgsteal|allocstall' /proc/vmstat
```

Now fragment memory deliberately (in a VM), then try a high-order allocation:

```c
/* Allocate many order-0 pages, free every other one. */
static struct page **pages;
static int nr = 200000;

static void fragment(void)
{
	int i;

	pages = vzalloc(nr * sizeof(*pages));
	for (i = 0; i < nr; i++)
		pages[i] = alloc_page(GFP_KERNEL);
	for (i = 0; i < nr; i += 2) {       /* free every other → checkerboard */
		__free_page(pages[i]);
		pages[i] = NULL;
	}
}

static void try_high_order(void)
{
	struct page *p = alloc_pages(GFP_KERNEL | __GFP_NOWARN | __GFP_NORETRY, 8);

	pr_info("order-8 alloc: %s\n", p ? "OK" : "FAILED");
	if (p)
		__free_pages(p, 8);
}
```
```bash
cat /proc/buddyinfo          # high orders now empty
# Ask the kernel to compact and retry:
echo 1 | sudo tee /proc/sys/vm/compact_memory
cat /proc/buddyinfo
grep compact_ /proc/vmstat   # compact_stall, compact_success, compact_fail
```
This is Ch. 11 T.2/T.3 in one experiment. Note that compaction can only move **movable**
pages — repeat the experiment allocating with `GFP_HIGHUSER_MOVABLE` and observe that
compaction now succeeds.

### Lab 11.C — Trigger and read an OOM report

```bash
# In a VM with a small amount of RAM (2 GB) and no swap:
sudo swapoff -a
stress-ng --vm 4 --vm-bytes 90% --vm-keep --timeout 60s
# or, to be precise about it, use a cgroup:
sudo mkdir /sys/fs/cgroup/oomtest
echo 200M | sudo tee /sys/fs/cgroup/oomtest/memory.max
echo $$   | sudo tee /sys/fs/cgroup/oomtest/cgroup.procs
stress-ng --vm 1 --vm-bytes 500M --timeout 30s

dmesg | grep -A60 'Out of memory\|invoked oom-killer'
```
Annotate the report line by line: the `gfp_mask` (decode it against T.7's lattice!), the
`order`, the per-zone free/min/low/high, `Node 0 Normal: NxM (U|E|M...)` (that's
`/proc/buddyinfo` at the moment of death), the slab totals, and the victim-selection table
with `oom_score_adj`. **Being able to read this fluently is a hiring-level skill.**

### Lab 11.D — Prove the GFP re-entrancy lattice matters (T.7)

```c
/* Simulate the filesystem deadlock shape: take a lock, then allocate with
 * GFP_KERNEL, and have the reclaim path try to take the same lock. */
static DEFINE_MUTEX(fs_lock);

static int my_shrinker_scan(struct shrinker *s, struct shrink_control *sc)
{
	mutex_lock(&fs_lock);          /* ← reclaim re-enters our lock */
	/* ...pretend to free something... */
	mutex_unlock(&fs_lock);
	return 0;
}

static void bad_path(void)
{
	mutex_lock(&fs_lock);
	kmalloc(1 << 20, GFP_KERNEL);   /* may enter reclaim → my_shrinker_scan → DEADLOCK */
	mutex_unlock(&fs_lock);
}

static void good_path(void)
{
	unsigned int flags;

	mutex_lock(&fs_lock);
	flags = memalloc_nofs_save();   /* drop __GFP_FS for this whole dynamic extent */
	kmalloc(1 << 20, GFP_KERNEL);   /* now cannot re-enter the FS shrinker */
	memalloc_nofs_restore(flags);
	mutex_unlock(&fs_lock);
}
```
Build with `CONFIG_PROVE_LOCKING=y` and apply memory pressure. Lockdep will report the
inversion via the `fs_reclaim` pseudo-lock — **that's how the kernel makes this checkable**:

```bash
grep -rn 'fs_reclaim_acquire\|__fs_reclaim_map' mm/page_alloc.c include/linux/sched/mm.h
```
Explain in writing why `fs_reclaim` is modelled as a lock at all. This is one of the
cleverest pieces of lockdep integration in the kernel.

### Lab 11.E — Find who is allocating, with `page_owner` and BPF

```bash
# boot with:  page_owner=on
sudo cat /sys/kernel/debug/page_owner | head -60
# Aggregate it:
sudo ./tools/mm/page_owner_sort /sys/kernel/debug/page_owner /tmp/sorted.txt
head -40 /tmp/sorted.txt

# Live, no reboot:
sudo bpftrace -e 'kprobe:__kmalloc_noprof,kprobe:__kmalloc { @[kstack(6)] = count(); }'
sudo bpftrace -e 'tracepoint:kmem:kmalloc { @bytes = hist(args->bytes_alloc); }'
sudo bpftrace -e 'tracepoint:kmem:mm_page_alloc { @order = lhist(args->order, 0, 11, 1); }'

# Slab allocation accounting by cache:
sudo slabtop -s c
sudo cat /proc/slabinfo | awk 'NR>2 {print $1, $3*$4/1024" KiB"}' | sort -k2 -rn | head

# Memory allocation profiling (6.10+, CONFIG_MEM_ALLOC_PROFILING):
sudo cat /proc/allocinfo | sort -rn | head -20
```
`/proc/allocinfo` is new and excellent — it attributes every live allocation to a source
line with near-zero overhead. Learn it; it makes leak-hunting dramatically easier.

### Lab 11.F — Build a `mempool` and prove it guarantees progress

```c
#include <linux/mempool.h>

static mempool_t *pool;
static struct kmem_cache *elem_cache;

static int __init mp_init(void)
{
	elem_cache = kmem_cache_create("mp_elem", 256, 0, 0, NULL);
	if (!elem_cache)
		return -ENOMEM;

	/* 16 elements pre-reserved at creation time, when memory was available */
	pool = mempool_create_slab_pool(16, elem_cache);
	if (!pool) {
		kmem_cache_destroy(elem_cache);
		return -ENOMEM;
	}

	/* Under extreme pressure this still succeeds, because of the reserve: */
	{
		void *e = mempool_alloc(pool, GFP_NOIO);

		pr_info("mempool_alloc -> %px (reserve=%d)\n", e, pool->curr_nr);
		mempool_free(e, pool);
	}
	return 0;
}
```
Now run it under `FAILSLAB` fault injection (Ch. 06 Lab 6.8) with a 100% failure rate and
show that plain `kmem_cache_alloc` fails while `mempool_alloc` still succeeds. That is the
forward-progress guarantee from T.7, demonstrated.

### Lab 11.G — DMA-safe allocation: what breaks and why

```c
/* WRONG — vmalloc memory is not physically contiguous */
void *buf = vmalloc(4096);
dma_addr_t dma = dma_map_single(dev, buf, 4096, DMA_TO_DEVICE);  /* BUG */

/* WRONG — a kmalloc'd buffer sharing a cache line with other data,
 * on a non-coherent architecture */
struct foo { int flag; char dma_buf[64]; };   /* dma_buf may share a line with flag */

/* RIGHT */
void *cpu; dma_addr_t handle;
cpu = dma_alloc_coherent(dev, 4096, &handle, GFP_KERNEL);
/* or, for streaming: */
void *b = kmalloc(4096, GFP_KERNEL);          /* ARCH_KMALLOC_MINALIGN guarantees
                                               * cache-line alignment for DMA safety */
handle = dma_map_single(dev, b, 4096, DMA_TO_DEVICE);
```
```bash
# Catch the first two automatically:
./scripts/config -e DMA_API_DEBUG -e DMA_API_DEBUG_SG -e DEBUG_VIRTUAL
grep -rn 'ARCH_KMALLOC_MINALIGN\|ARCH_DMA_MINALIGN' arch/arm64/include/asm/cache.h \
	include/linux/slab.h | head
dmesg | grep -i 'DMA-API'
```
Then answer drill 6 below with the evidence you just gathered.

---

## 4. Mastery drills

1. **Overflow hunt.** Find three call sites in `drivers/` using `kmalloc(a * b, ...)`.
   Determine whether `a` is attacker-controlled. Write a patch converting to
   `kmalloc_array()` (this is a classic first-patch — see Ch. 86).

2. **Measure the fast path.** Write a module that times 1e6 `kmalloc(64)/kfree` pairs using
   `ktime_get_ns()`, then repeat with a dedicated `kmem_cache`. Then repeat with
   `SLAB_HWCACHE_ALIGN` off. Explain the deltas.

3. **GFP audit.** Open `fs/ext4/`. Find every `GFP_NOFS` and determine whether it could be
   replaced by `memalloc_nofs_save()`. (Several upstream patches did exactly this — find them
   with `git log --grep memalloc_nofs`.)

4. **Fragmentation experiment.** In a VM, allocate and free order-0 pages in a pattern that
   fragments memory, then try `alloc_pages(GFP_KERNEL, 8)`. Watch `/proc/buddyinfo` and
   `compact_stall` in `/proc/vmstat`.

5. **Read `__alloc_pages()`** in `mm/page_alloc.c` end to end. Diagram the fast path,
   slow path, direct reclaim, compaction, and OOM as a flowchart. This function is the heart
   of Linux memory management.

6. **Answer:** why does `kmalloc()` guarantee alignment to `ARCH_KMALLOC_MINALIGN`, and why
   does that matter for DMA? (Hint: non-coherent DMA and cache line sharing.)

---

## 5. Further reading

- `Documentation/core-api/memory-allocation.rst` — **read first, it is excellent**
- `Documentation/core-api/gfp_mask-from-fs-io.rst`
- `Documentation/mm/` — especially `physical_memory.rst`, `page_owner.rst`
- Christoph Lameter, "Slab allocators in the Linux kernel" (LinuxCon)
- LWN: "The SLUB allocator" (Corbet), "Toward a reliable kmalloc()" ,
  "The end of the SLAB allocator" (6.8), "Randomized kmalloc caches" (6.6)
- Mel Gorman, *Understanding the Linux Virtual Memory Manager* (old but the buddy chapter
  is still the clearest explanation in print)

→ Next: [12-refcounting.md](12-refcounting.md)

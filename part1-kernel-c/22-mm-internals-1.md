# Chapter 22 — Memory Management Internals I: Page Tables, Folios, VMAs, Page Faults

> **Goal:** be able to translate a virtual address to a physical one by hand, explain why
> `struct page` became `struct folio`, describe every reason a page fault can occur, and read
> `mm/memory.c` without getting lost.

---

## Theory & First Principles

### T.0 — Start here: allocate a terabyte on a laptop

```c
#include <stdio.h>
#include <sys/mman.h>
int main(void) {
	void *p = mmap(NULL, 1UL << 40, PROT_READ | PROT_WRITE,
	               MAP_PRIVATE | MAP_ANONYMOUS | MAP_NORESERVE, -1, 0);
	printf("1 TiB at %p\n", p);
	((char *)p)[0]       = 'a';         /* touch ONE byte */
	((char *)p)[1 << 30] = 'b';         /* and one 1 GiB away */
	getchar();
	return 0;
}
```

It succeeds on a machine with 8 GiB of RAM. Check what it actually consumed:

```bash
grep -E 'VmSize|VmRSS' /proc/$(pgrep -n tb)/status
#   VmSize:  1073741824 kB     <- 1 TiB of ADDRESS SPACE
#   VmRSS:         1500 kB     <- ~1.5 MB of actual MEMORY
```

**Address space and memory are different resources.** You allocated an enormous amount of
the first and almost none of the second. Everything in this chapter follows from the
indirection that makes this possible.

**What the kernel actually did:**

1. `mmap` created a **VMA** — a `struct vm_area_struct` recording "addresses X to X+1 TiB are
   valid, anonymous, read-write." That is a ~200-byte bookkeeping object. **No page tables
   were built and no memory was allocated.**
2. Touching byte 0 caused a **page fault**. The kernel looked up the VMA, saw the access was
   legal, allocated one 4 KiB page, installed a PTE, and restarted the instruction.
3. Touching byte 2^30 did it again. The 262,143 pages in between still do not exist.

**The page fault is the key object here**, and the general form unifies most of the memory
manager:

> A page fault is a **programmable interception point**: the hardware hands control to the
> kernel at the exact instant a program touches an address, *before* the access completes.

Once you have that, an enormous amount becomes one mechanism:

| Feature | What the fault handler does |
|---|---|
| **Demand paging** | Allocate a zero page now, on first touch |
| **Copy-on-write** | Page is shared read-only; copy it, remap writable (`fork`, Ch. 20 §T.3) |
| **mmap'd files** | Fetch from the page cache, or issue I/O (Ch. 52) |
| **Swapping** | Read the page back from disk |
| **NUMA balancing** | Deliberately unmap, to *detect* which node touches it (Ch. 105 §T.5) |
| **userfaultfd** | Deliver the fault to a *userspace* handler — live migration, CRIU |
| **KSM, THP promotion, memory tiering** | All the same hook |

**Seven major features, one mechanism.** That is why `handle_mm_fault()` is among the most
important functions in the kernel and why it is worth reading in full.

**Now the cost, because the abstraction is not free.** Every access needs a virtual-to-
physical translation, which means walking a multi-level page table:

```
 x86-64, 4-level paging, TLB miss:
   PML4 -> PDPT -> PD -> PT -> the data itself
     \___________ ___________/
                 V
     FOUR dependent memory accesses BEFORE your load completes.
     ~400 ns if they all miss cache -- for ONE access.
```

The TLB caches translations to make this bearable, and it is **the only cache in the machine
with no hardware coherence** (§T.3). When one CPU changes a page table, every other CPU that
might have cached that translation must be told *by an inter-processor interrupt* — 1–10 µs,
scaling with CPU count, and a source of latency on cores doing entirely unrelated work. That
one asymmetry drives `mmu_gather` batching, PCID/ASID tagging, and a real chunk of Ch. 105.

---

### T.1 Virtual memory: one indirection, five problems solved

Virtual memory is the canonical instance of Butler Lampson's observation that *"all problems
in computer science can be solved by another level of indirection."* Inserting a translation
between the address a program uses and the address the DRAM controller sees solves **five
distinct problems at once**, and confusing them is the root of most MM misunderstanding:

| Problem | How translation solves it |
|---|---|
| **Relocation** | a program can be linked at a fixed address and loaded anywhere |
| **Protection** | a process cannot even *name* memory not in its page tables |
| **Sharing** | two page tables can point at one physical frame (libraries, COW, shmem) |
| **Sparse address spaces** | a 128 TiB address space backed by 4 MiB of RAM; holes cost nothing |
| **Overcommit / lazy work** | a mapping can exist with no backing store until touched |

The fifth is the deepest and most underappreciated. Once translation exists, **the page fault
becomes a programmable hook**: any access to a not-present page traps to software, which may
do *anything* before resuming. Demand paging, copy-on-write, memory-mapped files, swap,
`userfaultfd`, NUMA balancing, KSM, and live migration are all the same trick — *use the MMU
to intercept an access and do work lazily*. When you meet a new MM feature, ask "what work is
it deferring, and what does it do in the fault handler?" and you will usually have it.

### T.2 Page tables are a radix trie (you already know this structure)

Address translation is a **map from VPN → PFN over a huge, extremely sparse key space**.
That is exactly the problem Ch. 10 T.1–T.2 solved with the XArray. The same answer applies,
in hardware:

```
x86-64, 4-level, 4 KiB pages:

  63          48 47      39 38      30 29      21 20      12 11         0
 ┌──────────────┬──────────┬──────────┬──────────┬──────────┬────────────┐
 │ sign extend  │  PGD idx │  PUD idx │  PMD idx │  PTE idx │   offset   │
 └──────────────┴──────────┴──────────┴──────────┴──────────┴────────────┘
       16 bits      9 bits     9 bits     9 bits     9 bits     12 bits

  CR3 ──▶ PGD ──▶ PUD ──▶ PMD ──▶ PTE ──▶ physical frame
```

Each level is a **512-entry table occupying exactly one 4 KiB page** (512 × 8 bytes). That is
not a coincidence: making a node exactly one page means the allocator, the TLB, and the
hardware walker all operate in the same unit. The branching factor 512 = 2⁹ falls out of it.

Why a radix trie rather than a hash table or a balanced tree?

- **Hardware must walk it.** A page walk happens on every TLB miss, so it must be
  branch-free, allocation-free, and bounded: exactly *N* dependent loads for an *N*-level
  table. A tree with variable depth or a hash with collision chains is unacceptable.
- **Sparsity is free.** An unmapped 512 GiB region costs one zero PGD entry.
- **Prefix sharing.** Contiguous mappings share upper levels, so the table for a 1 GiB
  contiguous mapping is ~2 MiB, not 2 GiB.

The cost: **a TLB miss costs up to 4 dependent memory accesses** (and up to 24 under
virtualization with EPT — Ch. 04 T.3, since each guest level walk requires a full host walk).
This is why page-table *walk caches* exist in hardware and why huge pages matter so much.

Linux abstracts this into a **fixed five-level API** regardless of what the hardware has,
using "folding" for missing levels:

```c
pgd_t *pgd = pgd_offset(mm, addr);
p4d_t *p4d = p4d_offset(pgd, addr);      /* folded to a no-op on 4-level */
pud_t *pud = pud_offset(p4d, addr);
pmd_t *pmd = pmd_offset(pud, addr);
pte_t *pte = pte_offset_map(pmd, addr);
```
This is the `asm-generic` fallback pattern (Ch. 02 T.3) applied to page tables: generic MM
code is written once for five levels; a 2-level 32-bit arch defines the middle levels away at
compile time. Adding 5-level paging (LA57, 57-bit addresses) to Linux was therefore mostly a
matter of *un*folding a level.

### T.3 The TLB: a cache with no hardware coherence

A page walk is far too slow to do on every access, so the CPU caches translations in the
**TLB**. Typical modern part: ~64 L1 dTLB entries, ~1500–3000 L2 STLB entries.

**TLB coverage** is the number that matters:

$$ \text{coverage} = \text{entries} \times \text{page size} $$

| Page size | 1536 L2 entries | Coverage |
|---|---|---|
| 4 KiB | 1536 | **6 MiB** |
| 2 MiB | 1536 | **3 GiB** |
| 1 GiB | 16 | 16 GiB |

A workload with a 1 GiB working set using 4 KiB pages will miss the TLB constantly — its
*page tables alone* (2 MiB) barely fit in L2 cache. Measured TLB-miss overhead of 10–30% on
large-footprint workloads is routine, and occasionally 2×. **This single table is the entire
justification for huge pages**, and it is the first thing to check on any memory-heavy
performance problem.

**The critical fact:** unlike data caches, **TLBs are not coherent in hardware.** If CPU 0
changes a PTE, CPU 1's TLB may hold the stale translation indefinitely. Maintaining
coherence is *software's job*, and it is expensive:

```
CPU 0: modify PTE
CPU 0: local TLB invalidate (INVLPG / TLBI)
CPU 0: send IPI to every CPU that might have this mm loaded  ← "TLB shootdown"
CPU 1..N: receive IPI, invalidate, acknowledge
CPU 0: wait for all acknowledgements, THEN free the page
```

A TLB shootdown costs **microseconds** and scales badly with CPU count. Consequences that
shape the whole MM subsystem:

- `munmap()` of a large region is expensive, and `mmap_lock` is held throughout.
- Linux **batches** shootdowns (`struct mmu_gather`, `tlb_gather_mmu()` /
  `tlb_flush_mmu()` / `tlb_finish_mmu()`): collect many PTE changes, flush once, free the
  pages after.
- `mm_cpumask` tracks which CPUs have ever run this mm, so IPIs go only where needed.
- **Lazy TLB**: a kernel thread doesn't switch page tables at all (Ch. 20 T.8); it borrows
  `active_mm`. That CPU must then be told to drop the borrow before the mm is freed.
- **PCID/ASID** (x86 PCID, ARM ASID) tag TLB entries with an address-space id so a context
  switch does not require flushing the whole TLB. Without it every context switch costs a
  full TLB reload; KPTI made this critical because it adds a CR3 write on every syscall.
- Newer hardware offers **broadcast invalidation** (ARM `TLBI` is broadcast by design;
  x86 `INVLPGB` on AMD Zen 3+) which removes the IPI — a genuine architectural improvement.

> **Interview-grade statement:** *"The TLB is the only cache in the machine whose coherence
> is software's responsibility, and the cost of that responsibility (shootdown IPIs) is why
> `mmu_gather` batching, lazy TLB, and PCIDs all exist."*

### T.4 Page size: a three-way trade

| | Small pages (4 KiB) | Huge pages (2 MiB / 1 GiB) |
|---|---|---|
| **Internal fragmentation** | low | **high** (a 2 MiB page for a 4 KiB object) |
| **TLB coverage** | **poor** | excellent |
| **Page table size** | large | small (a PMD entry *is* the mapping) |
| **Fault cost** | cheap (~1 µs) | expensive (zeroing 2 MiB ≈ 50–100 µs) |
| **COW cost** | 4 KiB copy | **2 MiB copy** |
| **Reclaim granularity** | fine | coarse (must split or evict 2 MiB) |
| **Allocation** | always available | needs contiguity → **fragmentation** (Ch. 11 T.3) |

Linux offers three mechanisms at different points on this curve:

1. **`hugetlbfs`** — pages reserved at boot, explicitly requested, never swapped, never
   split. Deterministic and used by databases (Oracle, PostgreSQL `huge_pages=on`) and DPDK.
   The cost is rigidity: reserved memory is unavailable to anyone else.
2. **THP (Transparent Huge Pages)** — the kernel opportunistically allocates 2 MiB pages on
   fault (`khugepaged` also promotes existing 4 KiB pages by collapsing them). Transparent,
   but introduces latency spikes (direct compaction to find contiguity) and memory bloat
   (a sparsely-touched 2 MiB page wastes up to 2 MiB). `defrag=madvise` +
   `enabled=madvise` is the standard production compromise — **`defrag=always` is a
   well-known cause of multi-millisecond stalls.**
3. **mTHP / multi-size THP** (6.8+) — intermediate orders (16 KiB, 64 KiB…) tuned per-size.
   This is the current direction: the 4 KiB/2 MiB binary choice was always too coarse, and
   arm64's 64 KiB "contiguous PTE" hint gives TLB benefits without full 2 MiB granularity.

### T.5 `struct page` → `struct folio`: the memory-descriptor problem

The kernel needs a descriptor per physical frame. With 4 KiB pages, **there is one
`struct page` for every 4 KiB of RAM**, so its size is multiplied by RAM/4096:

```
sizeof(struct page) = 64 bytes  →  64/4096 = 1.5625% of ALL physical memory
   1 TiB of RAM  →  16 GiB of struct page
```

That constraint dominates its design and explains everything odd about it: aggressive use of
unions (the same bytes mean different things for slab pages, page-cache pages, anon pages,
and page-table pages), bit-packing flags into `page->flags` alongside the node/zone/section
id, and a decade-long refusal to add fields.

**The compound-page confusion.** A huge page is represented as a *compound page*: a head
`struct page` plus N−1 tail pages. Tail pages have almost no valid fields — their `->mapping`,
`->index`, `->lru` are meaningless, and you must call `compound_head()` before using them.
The problem is that **`struct page *` does not tell you whether you hold a head or a tail**,
so thousands of call sites had to defensively call `compound_head()`, and the ones that
forgot were subtle bugs. Worse, "is this page a tail page?" could change *concurrently* under
THP split.

**The folio is a type-system fix** (Matthew Wilcox, 5.16+):

```c
struct folio {
	unsigned long flags;
	struct list_head lru;
	struct address_space *mapping;
	pgoff_t index;
	void *private;
	atomic_t _mapcount, _refcount;
	...
};
```
A `struct folio *` is, by construction, **never a tail page**. The compiler enforces it. So:

- Functions taking a `folio` need no `compound_head()` call — a real performance win
  (measured ~7% on some page-cache paths) *and* a class of bugs eliminated.
- A folio is "one or more pages, of a size the caller must query" (`folio_nr_pages()`),
  which makes code naturally size-agnostic — the prerequisite for mTHP.
- It is the same move as Ch. 08 T.2's `__bitwise`: **encode an invariant in the type so
  violations become compile errors.** Recognize this as a recurring kernel technique.

The long-term plan is to shrink `struct page` to a single "memory descriptor" word pointing
at a type-specific descriptor (`struct slab`, `struct folio`, `struct ptdesc`…), reclaiming
most of that 1.5%. That conversion has been underway for years and is a good example of how
tree-wide refactoring happens (Ch. 00 T.5: no stable internal API is what makes it possible).

### T.6 The address space as a set of intervals: VMAs

A process's address space is described by **VMAs** (`struct vm_area_struct`), each a
half-open interval `[vm_start, vm_end)` with uniform properties:

```c
struct vm_area_struct {
	unsigned long vm_start, vm_end;
	struct mm_struct *vm_mm;
	pgprot_t vm_page_prot;
	vm_flags_t vm_flags;               /* VM_READ|WRITE|EXEC|SHARED|GROWSDOWN|... */
	struct file *vm_file;              /* NULL => anonymous */
	unsigned long vm_pgoff;
	const struct vm_operations_struct *vm_ops;   /* ★ ->fault(), ->mmap(), ->close() */
	void *vm_private_data;
	struct anon_vma *anon_vma;
	struct vma_lock *vm_lock;          /* per-VMA lock (6.4+) */
	...
};
```

Two design points worth understanding:

**(a) Why the maple tree replaced the rbtree.** The operations are: point lookup
("which VMA contains this address?" — on *every* page fault), range query, and
insert/split/merge on `mmap`/`munmap`/`mprotect`. Ch. 10 T.4's argument applies directly:
a B-tree touches far fewer cache lines than an rbtree for the same lookup, and — crucially —
the maple tree is **RCU-safe**, which the old rbtree + `mmap_lock` design was not.

**(b) That RCU-safety enabled per-VMA locking**, which is the real prize. Historically
`mmap_lock` (a single `rw_semaphore` per `mm`) protected the whole address space. Every page
fault took it for read; every `mmap`/`munmap` took it for write. On a many-threaded process
this was *the* scalability bottleneck — a single cache line and a single writer serializing
all faults.

Per-VMA locking (Suren Baghdasaryan, 6.4+) lets a page fault take **only the lock of the VMA
it faults in**, via an RCU lookup in the maple tree, with a sequence-counter fallback to
`mmap_lock` if the VMA is being modified. Measured: 30–70% reduction in fault latency on
threaded workloads. **This is the clearest example in the kernel of a data-structure change
being the prerequisite for a locking change** — study the ordering of those two patch series
as a model for how to land a big scalability project.

### T.7 The page fault: one exception, a dozen meanings

```
  CPU: access to a virtual address
   │
   ├─ TLB hit ─────────────────────────────▶ done (no software involved)
   │
   └─ TLB miss → hardware page walk
        ├─ translation found, permissions OK ─▶ fill TLB, done
        └─ not present OR permission violation
              │
              ▼
        #PF exception → arch handler → handle_mm_fault()
```

`handle_mm_fault()` then answers: *why did this fault, and what work was being deferred?*

| Fault cause | What the handler does | The deferred work |
|---|---|---|
| Anonymous, first touch | allocate a zeroed page (or map the shared zero page for reads) | lazy allocation |
| File-backed, not in page cache | `->fault()` → read from disk | demand paging |
| File-backed, in page cache | just install the PTE (**minor fault**) | — |
| Write to a COW page | `do_wp_page()`: copy or reuse | fork's copy |
| Page is in swap | `do_swap_page()`: read it back | swapping |
| NUMA hinting fault (`PROT_NONE`) | `do_numa_page()`: migrate page or task | placement profiling |
| Write to a clean file page | mark dirty, notify the filesystem | writeback accounting |
| `userfaultfd`-registered region | hand the fault to a *userspace* handler | anything |
| Genuine violation | `SIGSEGV` (or `__do_kernel_fault` → Oops) | — |

**Minor vs major faults** is the distinction to have on hand: a *minor* fault resolves from
memory (page cache hit, COW, first touch) and costs ~0.5–2 µs; a *major* fault requires I/O
and costs ~100 µs (NVMe) to ~10 ms (spinning disk). `/proc/<pid>/stat` reports both; a rising
major-fault rate means you are swapping or thrashing the page cache.

**`userfaultfd` is the generalization**: it exports the fault hook to userspace. A process
registers a range, and faults become messages on an fd that a handler thread resolves with
`UFFDIO_COPY`/`UFFDIO_ZEROPAGE`/`UFFDIO_CONTINUE`. This is how post-copy live migration,
userspace garbage collectors, distributed shared memory, and CRIU work. It is also a
security-sensitive primitive — it lets an attacker *stop the kernel mid-operation* at a
chosen address, which is why `vm.unprivileged_userfaultfd` defaults to 0.

### T.8 Reverse mapping: "who maps this physical page?"

Forward mapping (virtual → physical) is what page tables do. Reclaim, migration, and
`try_to_unmap()` need the **inverse**: given a `folio`, find and clear every PTE that maps
it. Without it you cannot evict a page.

Linux 2.4 kept a linked list of PTEs per page — correct but a huge memory overhead and a
long-standing scalability problem. Linux 2.6 introduced **object-based reverse mapping**,
which stores no per-page data at all; instead it *derives* the mappings:

**File-backed pages:** the `address_space` has an interval tree (`i_mmap`) of every VMA
mapping that file. Given `folio->mapping` and `folio->index`, query the interval tree for
VMAs covering that page offset, and compute the virtual address arithmetically:

```c
addr = vma->vm_start + ((folio->index - vma->vm_pgoff) << PAGE_SHIFT);
```
**Zero per-page overhead.** This is a beautiful application of "store the *relation*, not the
*instances*".

**Anonymous pages** have no file, so `struct anon_vma` plays that role: a VMA's anon pages
point (via `folio->mapping` with the `PAGE_MAPPING_ANON` low bit set) to an `anon_vma`, which
owns an interval tree of the VMAs that may map those pages. `fork()` makes this genuinely
hard: the child's VMA shares pages with the parent, so `anon_vma_chain` builds a *forest* of
parent/child relationships so a page can be found from any descendant. `mm/rmap.c`'s header
comment explains the lock ordering, and it is one of the trickiest areas of the kernel —
read it slowly.

`folio->_mapcount` counts how many PTEs map the folio; `folio->_refcount` counts all
references (mappings + page cache + transient pins). **Confusing these two is the source of a
long line of bugs**, including Dirty COW's family. `folio_mapcount()` vs `folio_ref_count()`
vs `folio_maybe_dma_pinned()` each answer a different question.

### T.9 Overcommit: statistical multiplexing of memory

`malloc()` succeeding does not mean the memory exists. Linux allows the sum of all mappings
to exceed RAM + swap, betting that not everyone will touch everything — the same statistical
argument as airline overbooking or network link oversubscription.

```bash
cat /proc/sys/vm/overcommit_memory   # 0 = heuristic, 1 = always, 2 = strict
cat /proc/sys/vm/overcommit_ratio    # for mode 2: (RAM * ratio/100) + swap
cat /proc/meminfo | grep -E 'Commit'
```

- **Mode 0 (default, heuristic):** refuse only "obviously insane" allocations.
- **Mode 1 (always):** never refuse. Required by workloads that map huge sparse regions
  (Redis `BGSAVE` fork, some JVM/Go runtimes, DPDK).
- **Mode 2 (strict):** never overcommit. `malloc()` fails honestly instead of the OOM killer
  shooting something later. The right choice for systems where predictability beats
  utilization.

The price of overcommit is the **OOM killer**: when the bet fails, someone must die, and the
kernel picks using `oom_score` (roughly proportional to RSS, adjusted by
`oom_score_adj`). This is an *architectural* consequence, not a bug: once you allow
overcommit, a mechanism to resolve the shortfall is mandatory. Systems that dislike the OOM
killer should set `overcommit_memory=2`, not complain about the killer.

Modern refinements: **PSI-driven userspace OOM** (`systemd-oomd`, Meta's `oomd`) act on
*pressure* before the kernel's last-resort killer, and cgroup v2's `memory.high`/`memory.max`
give per-workload throttling and killing (Ch. 23).

---

## 1. Internals

### 1.1 The physical memory model

```
        physical RAM
             │
   ┌─────────▼─────────┐
   │  node (pg_data_t) │   one per NUMA node
   └─────────┬─────────┘
             │
   ┌─────────▼─────────┐   ZONE_DMA (≤16 MiB, legacy ISA)
   │  zone (struct zone)│  ZONE_DMA32 (≤4 GiB)
   └─────────┬─────────┘   ZONE_NORMAL (directly mapped)
             │             ZONE_MOVABLE (only movable allocations)
   ┌─────────▼─────────┐   ZONE_DEVICE (persistent memory / device memory)
   │ free_area[order]  │   the buddy allocator (Ch. 11 T.2)
   └───────────────────┘
```

Zones exist because **not all memory is equally usable**: old DMA controllers can only
address low memory, 32-bit devices only 4 GiB, and `ZONE_MOVABLE` exists purely to give
compaction and hotplug a region it can guarantee to evacuate.

`SPARSEMEM_VMEMMAP` (the modern default) maps the `struct page` array into a virtual region
(`vmemmap`), so `pfn_to_page()` is pointer arithmetic even with memory holes:

```c
#define pfn_to_page(pfn)  (vmemmap + (pfn))
#define page_to_pfn(page) ((page) - vmemmap)
```

### 1.2 The address-space structures

```c
struct mm_struct {
	struct maple_tree     mm_mt;          /* ★ the VMA tree (was an rbtree) */
	pgd_t                *pgd;            /* root of the page tables */
	atomic_t              mm_users;       /* threads using this mm */
	atomic_t              mm_count;       /* +1 for lazy-tlb borrowers */
	struct rw_semaphore   mmap_lock;      /* ★ the historic bottleneck */
	spinlock_t            page_table_lock;
	unsigned long         start_code, end_code, start_data, end_data;
	unsigned long         start_brk, brk, start_stack, mmap_base;
	unsigned long         total_vm, locked_vm, pinned_vm, data_vm, exec_vm, stack_vm;
	struct percpu_counter rss_stat[NR_MM_COUNTERS];
	cpumask_t             cpu_bitmap;     /* for TLB shootdown targeting */
	unsigned long         flags;
	struct mmu_notifier_subscriptions *notifier_subscriptions;
	...
};
```

**`mm_users` vs `mm_count`** is a classic two-level refcount (Ch. 12): `mm_users` counts
*users of the address space* (threads); when it hits zero the VMAs and pages are torn down.
`mm_count` counts *references to the `mm_struct` itself* (including lazy-TLB borrowers and
kernel threads); only when *that* hits zero is the structure freed. Two counts because the
teardown of the *contents* and the freeing of the *descriptor* must happen at different
times.

### 1.3 The fault path

```
arch fault entry (arch/x86/mm/fault.c: exc_page_fault / arm64: do_mem_abort)
  → do_user_addr_fault()
      lock_vma_under_rcu()               ★ per-VMA lock fast path (6.4+)
        └─ fallback: mmap_read_lock(mm); vma = find_vma(mm, addr)
      access checks (vm_flags vs error_code)
      → handle_mm_fault(vma, addr, flags, regs)
          → __handle_mm_fault()
              pgd/p4d/pud/pmd walk, allocating tables as needed
              → handle_pte_fault(vmf)
                  !pte_present:
                      vma->vm_ops ? do_fault()        /* file-backed */
                                  : do_anonymous_page()
                      pte_none==false → do_swap_page()
                  pte_present && write && !writable → do_wp_page()   /* COW */
                  pte_protnone → do_numa_page()
              → returns VM_FAULT_* flags
      mmap_read_unlock() / vma_end_read()
```

`do_fault()` splits three ways — `do_read_fault()`, `do_cow_fault()`, `do_shared_fault()` —
and all call `vma->vm_ops->fault()`, which for a filesystem is `filemap_fault()`
(Ch. 52). `filemap_map_pages()` performs **fault-around**: on a fault, map up to 16 nearby
already-cached pages at once, amortizing the fault cost. That is a pure
latency-vs-wasted-work heuristic, tunable via `fault_around_bytes`.

### 1.4 Source map

```
mm/memory.c            ★★ handle_mm_fault, do_wp_page, do_anonymous_page, mmu_gather
mm/mmap.c              ★ mmap/munmap/mprotect, VMA split/merge, per-VMA locking
mm/vma.c               (6.10+) the VMA manipulation core, split out of mmap.c
mm/rmap.c              ★ reverse mapping, anon_vma, try_to_unmap
mm/huge_memory.c       THP, mTHP, split_huge_page
mm/hugetlb.c           hugetlbfs
mm/gup.c               get_user_pages / pin_user_pages
mm/mmu_gather.c        TLB shootdown batching
mm/userfaultfd.c
mm/page_table_check.c  a sanity checker worth enabling in dev kernels
include/linux/mm.h, mm_types.h, pgtable.h, rmap.h
arch/x86/mm/fault.c, arch/x86/mm/tlb.c     ★ read tlb.c for T.3
arch/arm64/mm/fault.c, arch/arm64/mm/context.c  (ASIDs)
Documentation/mm/       ★ physical_memory.rst, page_tables.rst, vmemmap_dedup.rst,
                          transhuge.rst, hugetlbfs_reserv.rst, process_addrs.rst
Documentation/admin-guide/mm/  ★ transhuge.rst, hugetlbpage.rst, userfaultfd.rst,
                                 concepts.rst, pagemap.rst
```

---

## 2. Practice

### Lab 22.1 — Walk the page tables by hand, from a module

```c
// SPDX-License-Identifier: GPL-2.0
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/mm.h>
#include <linux/sched/mm.h>
#include <linux/pgtable.h>
#include <linux/debugfs.h>
#include <linux/uaccess.h>

static pid_t target_pid;
static unsigned long target_addr;
module_param(target_pid, int, 0644);
module_param(target_addr, ulong, 0644);

static void walk_one(struct mm_struct *mm, unsigned long addr)
{
	pgd_t *pgd; p4d_t *p4d; pud_t *pud; pmd_t *pmd; pte_t *pte;
	spinlock_t *ptl;

	pr_info("=== walking 0x%lx (PAGE_SIZE=%lu, %d levels)\n",
		addr, PAGE_SIZE, CONFIG_PGTABLE_LEVELS);

	pgd = pgd_offset(mm, addr);
	pr_info("PGD [%3lu] @ %px = 0x%016lx %s\n",
		pgd_index(addr), pgd, pgd_val(*pgd),
		pgd_none(*pgd) ? "NONE" : "present");
	if (pgd_none(*pgd) || pgd_bad(*pgd))
		return;

	p4d = p4d_offset(pgd, addr);
	if (p4d_none(*p4d) || p4d_bad(*p4d))
		return;

	pud = pud_offset(p4d, addr);
	pr_info("PUD [%3lu] @ %px = 0x%016lx %s\n",
		pud_index(addr), pud, pud_val(*pud),
		pud_leaf(*pud) ? "LEAF (1GiB huge page!)" : "table");
	if (pud_none(*pud) || pud_leaf(*pud))
		return;

	pmd = pmd_offset(pud, addr);
	pr_info("PMD [%3lu] @ %px = 0x%016lx %s\n",
		pmd_index(addr), pmd, pmd_val(*pmd),
		pmd_leaf(*pmd) ? "LEAF (2MiB huge page!)" : "table");
	if (pmd_none(*pmd) || pmd_leaf(*pmd))
		return;

	pte = pte_offset_map_lock(mm, pmd, addr, &ptl);
	if (!pte)
		return;
	pr_info("PTE [%3lu] @ %px = 0x%016lx\n", pte_index(addr), pte, pte_val(*pte));
	if (pte_present(*pte)) {
		struct page *p = pte_page(*pte);

		pr_info("  -> PFN %lu  phys 0x%llx\n",
			pte_pfn(*pte), (u64)pte_pfn(*pte) << PAGE_SHIFT);
		pr_info("  -> flags: %s%s%s%s%s%s\n",
			pte_write(*pte)   ? "WRITE "   : "ro ",
			pte_dirty(*pte)   ? "DIRTY "   : "",
			pte_young(*pte)   ? "ACCESSED ": "",
			pte_exec(*pte)    ? "EXEC "    : "nx ",
			pte_special(*pte) ? "SPECIAL " : "",
			pte_soft_dirty(*pte) ? "SOFTDIRTY " : "");
		pr_info("  -> struct page %px refcount=%d mapcount=%d\n",
			p, page_ref_count(p), atomic_read(&p->_mapcount));
	} else {
		pr_info("  -> NOT PRESENT (swap entry or none)\n");
	}
	pte_unmap_unlock(pte, ptl);
}

static int walk_show(struct seq_file *m, void *v)
{
	struct task_struct *t;
	struct mm_struct *mm;

	rcu_read_lock();
	t = pid_task(find_vpid(target_pid), PIDTYPE_PID);
	if (t)
		get_task_struct(t);
	rcu_read_unlock();
	if (!t)
		return -ESRCH;

	mm = get_task_mm(t);
	if (mm) {
		mmap_read_lock(mm);
		walk_one(mm, target_addr);
		mmap_read_unlock(mm);
		mmput(mm);
	}
	put_task_struct(t);
	seq_puts(m, "see dmesg\n");
	return 0;
}
DEFINE_SHOW_ATTRIBUTE(walk);

static struct dentry *d;
static int __init pw_init(void)
{
	d = debugfs_create_file("pagewalk", 0444, NULL, NULL, &walk_fops);
	return 0;
}
static void __exit pw_exit(void) { debugfs_remove(d); }
module_init(pw_init); module_exit(pw_exit);
MODULE_LICENSE("GPL");
```
```bash
# Find an address to walk
cat /proc/self/maps | head
sudo insmod pagewalk.ko target_pid=$$ target_addr=0x$(grep heap /proc/self/maps | cut -d- -f1)
sudo cat /sys/kernel/debug/pagewalk && dmesg | tail -20
```
**Then verify by hand:** decode the address's 9-bit indices and confirm they match what the
module printed. Do this once and page tables stop being abstract forever.

### Lab 22.2 — Same thing from userspace with `/proc/<pid>/pagemap`

```c
/* pagemap.c — gcc -O2 -o pagemap pagemap.c */
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdint.h>
#include <sys/mman.h>

static void show(void *addr)
{
	uint64_t off = ((uintptr_t)addr / getpagesize()) * 8, e;
	int fd = open("/proc/self/pagemap", O_RDONLY);

	pread(fd, &e, 8, off);
	close(fd);

	printf("va=%p present=%llu swapped=%llu excl=%llu softdirty=%llu pfn=0x%llx phys=0x%llx\n",
	       addr,
	       (unsigned long long)(e >> 63) & 1,
	       (unsigned long long)(e >> 62) & 1,
	       (unsigned long long)(e >> 56) & 1,
	       (unsigned long long)(e >> 55) & 1,
	       (unsigned long long)(e & ((1ULL << 55) - 1)),
	       (unsigned long long)((e & ((1ULL << 55) - 1)) * getpagesize()));
}

int main(void)
{
	char *p = mmap(NULL, 1 << 20, PROT_READ | PROT_WRITE,
		       MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);

	printf("before touch: "); show(p);       /* not present */
	p[0] = 1;
	printf("after  touch: "); show(p);       /* present, has a PFN */
	printf("page 100    : "); show(p + 100 * 4096);   /* still not present */

	/* Watch COW: */
	if (fork() == 0) { printf("child before write: "); show(p);
	                   p[0] = 2;
	                   printf("child after  write: "); show(p); _exit(0); }
	sleep(1);
	printf("parent      : "); show(p);
	return 0;
}
```
```bash
sudo ./pagemap      # needs CAP_SYS_ADMIN to see real PFNs
# Then correlate with kpageflags/kpagecount:
sudo ./tools/vm/page-types -p $$ | head -20
sudo ./tools/mm/page-types -r -l 0x<pfn>
```
**The COW demonstration is the key result:** parent and child show the *same* PFN before the
write and *different* PFNs after.

### Lab 22.3 — Measure TLB coverage and prove T.3/T.4

```c
/* tlb.c — random-stride pointer chase over a variable working set.
   gcc -O2 -o tlb tlb.c  */
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <sys/mman.h>

static double now(void) {
	struct timespec t; clock_gettime(CLOCK_MONOTONIC, &t);
	return t.tv_sec + t.tv_nsec / 1e9;
}

int main(int argc, char **argv)
{
	size_t mb   = argc > 1 ? atoi(argv[1]) : 64;
	int    huge = argc > 2 ? atoi(argv[2]) : 0;
	size_t sz   = mb << 20, n = sz / 4096, i;
	size_t *idx;
	char *p;
	double t0;
	volatile long sink = 0;

	p = mmap(NULL, sz, PROT_READ | PROT_WRITE,
		 MAP_PRIVATE | MAP_ANONYMOUS | (huge ? MAP_HUGETLB : 0), -1, 0);
	if (p == MAP_FAILED) { perror("mmap"); return 1; }
	memset(p, 1, sz);

	/* random permutation of page indices: defeats prefetch, maximizes TLB pressure */
	idx = malloc(n * sizeof(*idx));
	for (i = 0; i < n; i++) idx[i] = i;
	for (i = n - 1; i > 0; i--) { size_t j = rand() % (i + 1), t = idx[i]; idx[i] = idx[j]; idx[j] = t; }

	t0 = now();
	for (int rep = 0; rep < 20; rep++)
		for (i = 0; i < n; i++)
			sink += p[idx[i] * 4096];
	printf("%5zu MiB %-5s : %6.1f ns/access\n", mb, huge ? "2MiB" : "4KiB",
	       (now() - t0) / (20.0 * n) * 1e9);
	return sink == 0;
}
```
```bash
# 4 KiB pages: watch the cliff when the working set exceeds TLB coverage
for mb in 1 2 4 8 16 32 64 128 256 512 1024; do ./tlb $mb 0; done

# Now with 2 MiB pages
echo 1024 | sudo tee /proc/sys/vm/nr_hugepages
for mb in 64 256 1024; do ./tlb $mb 1; done

# Prove it with hardware counters
sudo perf stat -e dTLB-loads,dTLB-load-misses,dtlb_load_misses.walk_active,\
cycle_activity.stalls_mem_any,page-faults -- ./tlb 512 0
sudo perf stat -e dTLB-loads,dTLB-load-misses -- ./tlb 512 1

# Your machine's actual TLB geometry:
cpuid -1 2>/dev/null | grep -i tlb | head -20
lscpu | grep -i tlb
```
**The result to record:** the ns/access curve for 4 KiB pages has a knee right around
`L2_TLB_entries × 4 KiB` (≈ 6 MiB). With 2 MiB pages that knee moves out by ~500×. That knee
*is* T.4's table.

### Lab 22.4 — Observe TLB shootdowns (T.3)

```bash
# Shootdown IPIs are counted per CPU:
grep -E 'TLB|CAL|RES' /proc/interrupts

# Generate them: many threads, one address space, lots of mmap/munmap
cat > /tmp/shoot.c <<'EOF'
#define _GNU_SOURCE
#include <pthread.h>
#include <sys/mman.h>
#include <stdio.h>
#include <unistd.h>
static void *w(void *a) {
	for (int i = 0; i < 20000; i++) {
		void *p = mmap(NULL, 1<<20, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
		*(char *)p = 1;
		munmap(p, 1<<20);
	}
	return NULL;
}
int main(int argc, char **argv) {
	int n = argc > 1 ? atoi(argv[1]) : 8;
	pthread_t t[64];
	for (int i = 0; i < n; i++) pthread_create(&t[i], NULL, w, NULL);
	for (int i = 0; i < n; i++) pthread_join(t[i], NULL);
}
EOF
gcc -O2 -o /tmp/shoot /tmp/shoot.c -lpthread

B=$(grep TLB /proc/interrupts | awk '{s=0; for(i=2;i<=NF-2;i++) s+=$i; print s}')
time /tmp/shoot 8
A=$(grep TLB /proc/interrupts | awk '{s=0; for(i=2;i<=NF-2;i++) s+=$i; print s}')
echo "TLB shootdown IPIs: $((A-B))"

# Scale the thread count and watch it superlinearly worsen:
for n in 1 2 4 8 16; do echo -n "$n threads: "; ( time /tmp/shoot $n ) 2>&1 | grep real; done

# Trace them:
sudo bpftrace -e 'kprobe:flush_tlb_mm_range,kprobe:native_flush_tlb_others { @[probe] = count(); }'
sudo perf stat -e tlb:tlb_flush -a -- /tmp/shoot 8
sudo bpftrace -e 'tracepoint:tlb:tlb_flush { @[args->reason] = count(); }'

# And see the batching machinery:
$EDITOR mm/mmu_gather.c arch/x86/mm/tlb.c
grep -n 'PCID\|INVLPGB' arch/x86/mm/tlb.c | head
```

### Lab 22.5 — Every kind of page fault, observed

```bash
# Per-process counters
ps -o pid,min_flt,maj_flt,cmd -p $$
awk '{print "minflt="$10, "majflt="$12}' /proc/self/stat

# Live, by type and by process
sudo bpftrace -e '
software:page-faults:1 { @[comm] = count(); }'

sudo bpftrace -e '
tracepoint:exceptions:page_fault_user  { @user[comm] = count(); }
tracepoint:exceptions:page_fault_kernel{ @kern[comm] = count(); }'

# Which VMA / file is faulting?
sudo bpftrace -e '
kprobe:handle_mm_fault {
	$vma = (struct vm_area_struct *)arg0;
	@[comm, $vma->vm_file ? str($vma->vm_file->f_path.dentry->d_name.name) : "anon"] = count(); }'

# Fault LATENCY distribution — minor vs major is visible as a bimodal histogram
sudo bpftrace -e '
kprobe:handle_mm_fault   { @s[tid] = nsecs; }
kretprobe:handle_mm_fault /@s[tid]/ { @us = hist((nsecs - @s[tid])/1000); delete(@s[tid]); }'

# COW faults specifically
sudo bpftrace -e 'kprobe:do_wp_page { @[comm] = count(); }'
sudo bpftrace -e 'kprobe:do_anonymous_page { @[comm] = count(); }'
sudo bpftrace -e 'kprobe:do_swap_page { @[comm] = count(); }'   # = major faults from swap

# Fault-around in action
cat /sys/kernel/debug/fault_around_bytes
echo 4096 | sudo tee /sys/kernel/debug/fault_around_bytes   # disable it
# re-run a file-mmap workload and compare minor fault counts
```

### Lab 22.6 — Transparent huge pages: benefit and cost (T.4)

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled     # [always] madvise never
cat /sys/kernel/mm/transparent_hugepage/defrag
ls /sys/kernel/mm/transparent_hugepage/hugepages-2048kB/    # mTHP controls (6.8+)

# Per-process THP usage
grep -E 'AnonHugePages|ShmemPmdMapped|FilePmdMapped' /proc/$$/smaps_rollup
grep AnonHugePages /proc/$$/smaps | awk '{s+=$2} END {print s" kB"}'

# System-wide THP statistics
grep -E 'thp_|compact_' /proc/vmstat
cat /sys/kernel/mm/transparent_hugepage/khugepaged/pages_collapsed

# Benchmark with and without
for mode in always never madvise; do
  echo $mode | sudo tee /sys/kernel/mm/transparent_hugepage/enabled >/dev/null
  echo -n "THP=$mode: "; ./tlb 512 0
done

# THE PATHOLOGY: defrag=always causes direct-compaction stalls
echo always | sudo tee /sys/kernel/mm/transparent_hugepage/defrag
# fragment memory first (Ch. 11 Lab 11.B), then:
sudo bpftrace -e '
kprobe:try_to_compact_pages   { @s[tid] = nsecs; }
kretprobe:try_to_compact_pages /@s[tid]/ { @stall_us = hist((nsecs-@s[tid])/1000); delete(@s[tid]); }'
grep -E 'compact_stall|compact_fail|thp_fault_fallback' /proc/vmstat
echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/defrag
```
**Record the stall histogram.** Multi-millisecond `try_to_compact_pages` stalls under
`defrag=always` are the single most common THP production incident.

### Lab 22.7 — `userfaultfd`: implement demand paging in userspace (T.7)

```c
/* uffd.c — gcc -O2 -o uffd uffd.c -lpthread */
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <linux/userfaultfd.h>
#include <poll.h>
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <sys/syscall.h>
#include <unistd.h>

static int uffd;
static char *region;
static size_t len = 16 * 4096;

static void *handler(void *arg)
{
	static char page[4096];
	struct uffd_msg msg;
	struct uffdio_copy c;
	struct pollfd p = { .fd = uffd, .events = POLLIN };

	for (;;) {
		poll(&p, 1, -1);
		read(uffd, &msg, sizeof(msg));
		if (msg.event != UFFD_EVENT_PAGEFAULT)
			continue;

		unsigned long addr = msg.arg.pagefault.address & ~4095UL;
		unsigned long idx  = (addr - (unsigned long)region) / 4096;

		printf("  [handler] fault at %p (page %lu, %s) -> filling\n",
		       (void *)addr, idx,
		       (msg.arg.pagefault.flags & UFFD_PAGEFAULT_FLAG_WRITE) ? "write" : "read");

		memset(page, 'A' + (idx % 26), sizeof(page));
		c.src = (unsigned long)page;
		c.dst = addr;
		c.len = 4096;
		c.mode = 0;
		c.copy = 0;
		ioctl(uffd, UFFDIO_COPY, &c);
	}
	return NULL;
}

int main(void)
{
	struct uffdio_api api = { .api = UFFD_API };
	struct uffdio_register reg;
	pthread_t t;
	size_t i;

	uffd = syscall(SYS_userfaultfd, O_CLOEXEC | O_NONBLOCK);
	if (uffd < 0) { perror("userfaultfd (try sysctl vm.unprivileged_userfaultfd=1)"); return 1; }
	ioctl(uffd, UFFDIO_API, &api);

	region = mmap(NULL, len, PROT_READ | PROT_WRITE,
		      MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);

	reg.range.start = (unsigned long)region;
	reg.range.len   = len;
	reg.mode        = UFFDIO_REGISTER_MODE_MISSING;
	ioctl(uffd, UFFDIO_REGISTER, &reg);

	pthread_create(&t, NULL, handler, NULL);

	for (i = 0; i < len; i += 4096) {
		printf("main: reading page %zu...\n", i / 4096);
		printf("main: got '%c'\n\n", region[i]);
	}
	return 0;
}
```
```bash
sudo sysctl -w vm.unprivileged_userfaultfd=1   # or run as root
./uffd
```
**You have just implemented demand paging in 60 lines of userspace code.** Now consider what
else this enables (post-copy migration, CRIU, userspace GC) and why it is a security-relevant
primitive.

### Lab 22.8 — Reverse mapping in action (T.8)

```bash
# Map the same file into many processes, then find all the mappings
for i in $(seq 1 5); do
  python3 -c "
import mmap,time,sys
f=open('/usr/lib/x86_64-linux-gnu/libc.so.6','rb')
m=mmap.mmap(f.fileno(),0,prot=mmap.PROT_READ)
m[0]
time.sleep(300)" &
done

# Who maps this file?
sudo fuser -v /usr/lib/x86_64-linux-gnu/libc.so.6
sudo lsof /usr/lib/x86_64-linux-gnu/libc.so.6 | head

# How many PTEs map a given physical page?
sudo ./tools/mm/page-types -f /usr/lib/x86_64-linux-gnu/libc.so.6 | head -20
sudo ./tools/mm/page-types -r -b mmap

# See mapcount vs refcount for real pages:
sudo dd if=/proc/kpagecount bs=8 skip=$PFN count=1 2>/dev/null | xxd
sudo dd if=/proc/kpageflags bs=8 skip=$PFN count=1 2>/dev/null | xxd

# Trace reclaim using rmap:
sudo bpftrace -e 'kprobe:try_to_unmap { @ = count(); }'
sudo bpftrace -e 'kprobe:rmap_walk_file,kprobe:rmap_walk_anon { @[probe] = count(); }'

$EDITOR mm/rmap.c        # read the ~150-line header comment on lock ordering
```

### Lab 22.9 — Per-VMA locking: measure the scalability win (T.6)

```bash
grep -E 'PER_VMA_LOCK' /boot/config-$(uname -r) .config

# Fault-heavy multithreaded benchmark
cat > /tmp/faults.c <<'EOF'
#define _GNU_SOURCE
#include <pthread.h>
#include <sys/mman.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#define SZ (256UL<<20)
static int nthr;
static void *w(void *a) {
	char *p = mmap(NULL, SZ/nthr, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
	for (size_t i = 0; i < SZ/nthr; i += 4096) p[i] = 1;   /* one fault per page */
	return NULL;
}
int main(int argc, char **argv) {
	struct timespec a, b; pthread_t t[64];
	nthr = argc > 1 ? atoi(argv[1]) : 1;
	clock_gettime(CLOCK_MONOTONIC, &a);
	for (int i = 0; i < nthr; i++) pthread_create(&t[i], NULL, w, NULL);
	for (int i = 0; i < nthr; i++) pthread_join(t[i], NULL);
	clock_gettime(CLOCK_MONOTONIC, &b);
	printf("%2d threads: %.3f s  (%.0f faults/s)\n", nthr,
	  (b.tv_sec-a.tv_sec)+(b.tv_nsec-a.tv_nsec)/1e9,
	  (SZ/4096.0)/((b.tv_sec-a.tv_sec)+(b.tv_nsec-a.tv_nsec)/1e9));
}
EOF
gcc -O2 -o /tmp/faults /tmp/faults.c -lpthread
for n in 1 2 4 8 16; do /tmp/faults $n; done

# Are we taking the fast path?
grep -E 'pgfault|vma_lock' /proc/vmstat
sudo bpftrace -e '
kprobe:lock_vma_under_rcu { @attempt = count(); }
kretprobe:lock_vma_under_rcu /retval == 0/ { @fallback_to_mmap_lock = count(); }'

# Contention on mmap_lock (if you built with LOCK_STAT):
sudo grep -A3 mmap_lock /proc/lock_stat
```
Read the two patch series in order and note the dependency:
```bash
git log --oneline --grep='maple tree' -- mm/ | head
git log --oneline --grep='per-VMA' -- mm/ | head
```

### Lab 22.10 — Overcommit and the OOM killer (T.9)

```bash
cat /proc/meminfo | grep -E 'Commit|MemAvailable|MemFree'
cat /proc/sys/vm/overcommit_memory /proc/sys/vm/overcommit_ratio

# Prove overcommit: allocate far more than RAM without touching it
cat > /tmp/over.c <<'EOF'
#include <stdio.h>
#include <sys/mman.h>
int main(void) {
	size_t gb = 0;
	while (1) {
		void *p = mmap(NULL, 1UL<<30, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS|MAP_NORESERVE, -1, 0);
		if (p == MAP_FAILED) break;
		printf("mapped %zu GiB\n", ++gb);
	}
	getchar();
}
EOF
gcc -O2 -o /tmp/over /tmp/over.c && /tmp/over &
grep -E 'Committed_AS|VmallocTotal' /proc/meminfo
grep -E 'VmSize|VmRSS' /proc/$!/status     # huge VmSize, tiny VmRSS

# Now strict mode:
sudo sysctl -w vm.overcommit_memory=2 vm.overcommit_ratio=50
/tmp/over      # fails much earlier, honestly
sudo sysctl -w vm.overcommit_memory=0

# OOM victim selection
for p in $(pgrep -n bash) 1; do
  echo "pid $p oom_score=$(cat /proc/$p/oom_score) adj=$(cat /proc/$p/oom_score_adj)"
done
echo -1000 | sudo tee /proc/$$/oom_score_adj    # protect this shell
sudo bpftrace -e 'kprobe:oom_kill_process { printf("OOM killing\n"); }'
```

---

## 3. Mastery drills

1. **Translate by hand.** Pick a userspace address. Decode its five index fields. Use the
   Lab 22.1 module (or `/proc/<pid>/pagemap`) to get the real PFN. Then verify by reading
   each page-table level yourself. Repeat for an address inside a THP and show the PMD is a
   leaf.

2. **Read `handle_mm_fault()` and `handle_pte_fault()`** end to end (`mm/memory.c`). Draw a
   decision tree covering *every* return path. Cross-check against T.7's table — did you find
   causes it omits?

3. **`do_wp_page()` in detail.** Under exactly what conditions does it reuse the page instead
   of copying? Why is that condition so hard to get right? Read the Dirty COW fix and at
   least one `folio_maybe_dma_pinned()` related fix, and explain each race.

4. **The folio conversion.** `git log --oneline --grep='folio' -- mm/ | head -50`. Read
   Matthew Wilcox's original cover letter on lore. Summarize: what bugs did it eliminate,
   what performance did it gain, and what is the endgame for `struct page`?

5. **Compute the descriptor overhead.** For a machine with 1 TiB of RAM, compute the memory
   consumed by `struct page`. Then compute it under the proposed "memory descriptor" scheme.
   Then explain why `vmemmap_dedup` exists for hugetlb (hint: `Documentation/mm/vmemmap_dedup.rst`).

6. **TLB shootdown cost model.** Using your Lab 22.4 numbers, estimate the cost of a single
   shootdown IPI on your machine as a function of CPU count. At what thread count does
   `munmap` become the bottleneck? Then read `arch/x86/mm/tlb.c` and explain
   `tlb_remove_table` / `flush_tlb_batched_pending` / `INVLPGB`.

7. **Design the reverse map.** Before reading `mm/rmap.c`, design a scheme to answer "which
   PTEs map this physical page?" for both file and anonymous pages, with zero per-page
   overhead. Then compare with what Linux does. What does `fork()` break in your design?

8. **`anon_vma` forests.** Draw the `anon_vma` / `anon_vma_chain` structure after:
   `P` allocates anon memory → `P` forks `C1` → `C1` forks `C2` → `P` writes (COW). Show how
   `rmap_walk_anon` finds all mappings of the original page from any of the three.

9. **Huge page economics.** For a workload with a 100 GiB working set: compute the page-table
   memory, TLB coverage, and expected TLB-miss rate under 4 KiB, 2 MiB, and 1 GiB pages.
   Then argue for a specific configuration, including whether to use hugetlbfs or THP, and
   state what you would measure to validate it.

10. **`mmap_lock` archaeology.** Find three separate scalability series that attacked
    `mmap_lock` (`git log --grep='mmap_lock'`). For each, state the approach and why it was
    or wasn't sufficient. Then explain why the maple tree had to land first.

11. **mTHP.** Read `Documentation/admin-guide/mm/transhuge.rst` on multi-size THP. Explain
    why intermediate orders help and how arm64's contiguous-PTE hint differs from a true
    PMD-mapped huge page.

12. **GUP.** Read `mm/gup.c`'s header comment. Explain the difference between
    `get_user_pages()` and `pin_user_pages()`, why the split was necessary, and what breaks
    if a driver DMAs into a `get_user_pages()` page that then gets COW'd. (This is the
    `FOLL_PIN` saga — a genuinely instructive multi-year design story.)

13. **Design question.** You are adding a device that needs 1 GiB of physically contiguous,
    DMA-coherent memory, available at any time after boot, on a system that also runs
    general-purpose workloads. Evaluate: boot-time reservation (`memmap=`), CMA, hugetlbfs
    with 1 GiB pages, and `ZONE_MOVABLE`. State the trade-offs in terms of T.4 and Ch. 11 T.3.

---

## 4. Further reading

**Kernel documentation:**
- `Documentation/mm/` ★★ — `physical_memory.rst`, `page_tables.rst`, `process_addrs.rst`,
  `transhuge.rst`, `vmemmap_dedup.rst`, `hugetlbfs_reserv.rst`, `page_owner.rst`,
  `page_migration.rst`, `mmu_notifier.rst`, `active_mm.rst`
- `Documentation/admin-guide/mm/` ★ — `concepts.rst`, `transhuge.rst`, `hugetlbpage.rst`,
  `userfaultfd.rst`, `pagemap.rst`, `ksm.rst`, `numa_memory_policy.rst`
- `Documentation/core-api/pin_user_pages.rst` ★ — the GUP/PUP distinction
- `Documentation/filesystems/proc.rst` — `smaps`, `pagemap`, `maps` field-by-field

**Source, in reading order:**
1. `include/linux/mm_types.h` — `struct page`, `struct folio`, `struct vm_area_struct`,
   `struct mm_struct`
2. `mm/memory.c` — `handle_mm_fault()` downward
3. `arch/x86/mm/fault.c` — the entry path
4. `mm/mmap.c` / `mm/vma.c` — VMA split/merge, the trickiest arithmetic in the kernel
5. `mm/rmap.c` — read the header comment twice
6. `arch/x86/mm/tlb.c` — T.3 in code

**Books:**
- Gorman, *Understanding the Linux Virtual Memory Manager* (2004) — ancient (2.4/2.6) but
  still the only book-length treatment; the *concepts* chapters remain the best available
- Bovet & Cesati, *Understanding the Linux Kernel*, Ch. 2, 8, 9, 15–17
- Bryant & O'Hallaron, *Computer Systems: A Programmer's Perspective*, Ch. 9 — the clearest
  introduction to VM and page tables anywhere
- Arpaci-Dusseau, *OSTEP*, Part II (Virtualization) — **free**, excellent on the theory in
  T.1–T.4
- Hennessy & Patterson, *Computer Architecture*, Appendix B & Ch. 2 — TLB/cache theory

**Papers:**
- Denning, "The Working Set Model for Program Behavior" (CACM 1968) — the foundation of
  demand paging and reclaim
- Denning, "Virtual Memory" (Computing Surveys 1970) — the canonical survey
- Bhattacharjee & Lustig, *Architectural and Operating System Support for Virtual Memory*
  (Morgan & Claypool, 2017) — **the modern treatment of TLBs, huge pages, and shootdowns**
- Basu et al., "Efficient Virtual Memory for Big Memory Servers" (ISCA 2013) — direct
  segments; the quantitative case for huge pages
- Panwar, Bansal & Gopinath, "HawkEye: Efficient Fine-grained OS Support for Huge Pages"
  (ASPLOS 2019) — THP's real-world pathologies, measured
- Amit, "Optimizing the TLB Shootdown Algorithm with Page Access Tracking" (ATC 2017)
- Clements, Kaashoek & Zeldovich, "RadixVM: Scalable Address Spaces for Multithreaded
  Applications" (EuroSys 2013) — the academic ancestor of per-VMA locking

**LWN (mm is the most intensively covered subsystem):**
- "Memory folios" / "Clarifying memory management with folios"
- "The maple tree" and "Per-VMA locking" — read both, in that order
- "Transparent huge pages" series; "Multi-size THP"
- "Userfaultfd" and "Post-copy live migration"
- "Reverse mapping" / "The object-based reverse-mapping VM"
- "Pinning pages, and the FOLL_PIN saga"
- "Five-level page tables"
- The annual LSFMM summit coverage — **the single best way to know where mm is going**

→ Next: [23-mm-internals-2.md](23-mm-internals-2.md)

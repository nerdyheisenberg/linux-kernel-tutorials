# Chapter 23 — Memory Management Internals II: Reclaim, Writeback, Compaction, THP, memcg

> **Goal:** explain what the kernel does when memory runs low, at the level of the algorithms
> — not the sysctls. Diagnose thrashing, swapping, writeback stalls, compaction stalls, and
> cgroup OOMs from first principles.

---

## Theory & First Principles

### T.0 — Start here: your "free" memory is not free, and that is correct

```bash
free -h
#               total        used        free      shared  buff/cache   available
# Mem:           31Gi       4.2Gi       412Mi       1.1Gi        27Gi        26Gi
#                                       ^^^^^                   ^^^^^
#                                 "only 412 MB free!"      27 GB of CACHE
```

Every few months someone files a bug: *"Linux is using all my RAM."* It is not a bug, and
understanding why is the entry point to the whole reclaim subsystem.

**Unused memory is wasted memory.** A page of DRAM sitting empty does nothing. The same page
holding a copy of a file you might read again is a free 1000x speedup if you do (80 ns versus
80 µs, Ch. 00 §T.3). So the kernel fills memory with cache and frees it instantly when
something needs it. **`free` is the wrong column; `available` is the right one** — and
`available` exists precisely because so many people read the wrong one.

**Which reframes the entire problem:**

> Physical memory is a **cache** for a much larger backing store (files on disk, and swap).
> Everything in this chapter is caching theory applied to that cache.

And caching theory has an unfortunate result waiting. The optimal policy is Belady's MIN —
evict the page whose next use is furthest in the future — which **requires knowing the
future** and is unimplementable. The best implementable approximation is **LRU**. So: just
implement LRU.

**You cannot.** Work out what exact LRU requires:

```c
/* On EVERY memory access -- every load, every store, billions per second: */
list_move(&page->lru, &active_list);   /* a lock, two stores, a cacheline bounce */
```

Exact LRU means updating a global ordered structure **on every access**, in hardware, for
every one of the machine's millions of pages. It would cost orders of magnitude more than the
memory it saves. The hardware helps with exactly one bit — the PTE *accessed* bit, set by the
MMU on reference — and that bit is all the information you get.

**So the real question of this chapter is:**

> Given one bit per page, set by hardware, with no ability to observe access *order*, how
> well can you approximate LRU?

Linux's answer has three layers, each recovering information the previous one discarded:

1. **Two lists (active/inactive), not one.** New pages enter *inactive*; a second reference
   promotes to *active*. This is a 2Q approximation and its purpose is **scan resistance** —
   `cat bigfile` must not evict your working set. Single-list LRU fails catastrophically
   here, and it is the single most important thing the design gets right. §T.2
2. **Refault distance.** When a page is evicted, remember *when*, in a shadow entry. If it
   comes back, you can compute how many other pages were evicted in between — which
   **reconstructs the reuse distance you threw away**, after the fact. The kernel is
   measuring whether its own previous decision was wrong, and rebalancing anon versus file
   from the answer. §T.3
3. **MGLRU.** Multiple generations plus page-table-walk-based aging, sampling many accessed
   bits per walk instead of one per fault. Much cheaper at scale. §T.4

**The framing to carry forward:** this is not "the reclaim code." It is an *online algorithm*
operating with radically incomplete information, under adversarial access patterns, where
being wrong costs a 1000x latency penalty. Judged that way the design is genuinely
impressive rather than merely intricate.

```bash
cat /proc/meminfo | grep -E 'Active|Inactive|Dirty|Writeback'
cat /proc/vmstat  | grep -E 'workingset|pgscan|pgsteal|allocstall'
cat /proc/pressure/memory     # PSI: stall time, the metric that actually matters
```

---

### T.1 Memory is a cache, and caching theory applies

The single reframing that makes this chapter tractable:

> **Physical RAM is a cache for a much larger backing store** (files on disk, swap, and
> regenerable kernel state). Reclaim is *eviction*. Everything follows.

The optimal policy is known: **Belady's MIN** (1966) — evict the page whose next reference is
furthest in the future. It is provably optimal and **provably unimplementable**, because it
requires knowledge of the future. Every real policy is an *approximation of MIN using the
past as a predictor of the future*, which works only because programs exhibit **locality**.

Denning's **working set model** (1968) formalizes locality: the working set $W(t, \tau)$ is
the set of pages referenced in the last $\tau$ time units. The claim — validated empirically
for sixty years — is that $W$ is small, slowly-varying, and predictive. If a process's
working set fits in RAM it runs at memory speed; if it does not, it **thrashes**: nearly
every access faults, the system spends all its time on I/O, and throughput collapses
non-linearly. Thrashing is not gradual degradation; it is a phase transition, and that is why
"the machine got a bit slow" turns into "the machine is unusable" so abruptly.

### T.2 Why exact LRU is impossible, and what Linux does instead

LRU (evict the least-recently-used page) approximates MIN well. But *exact* LRU requires
updating a global ordering **on every memory access** — impossible, because accesses happen
in hardware with no software involvement (Ch. 22 T.7: only a *fault* traps).

All the hardware gives you is one bit per PTE: the **accessed/referenced bit**, set by the
MMU on access, cleared by software. So the practical problem becomes:

> *Approximate a total recency ordering using one bit per page, sampled periodically.*

The classical answers:

| Algorithm | Idea | Cost |
|---|---|---|
| **CLOCK / second-chance** | circular scan; if accessed bit set, clear it and skip; else evict | O(1) amortized, one bit |
| **CLOCK-Pro / LIRS** | track *reuse distance*, not just recency | more state |
| **ARC** (Megiddo & Modha) | adaptively balance recency vs frequency lists | patented, not in Linux |
| **2Q / LRU-K** | two queues: probationary and protected | simple, effective |

**Linux uses a two-list variant of 2Q**, and the reason is a specific pathology:

> **The use-once / scan-resistance problem.** A `cat huge_file > /dev/null` reads gigabytes,
> each page used exactly once. Under plain LRU those pages are "most recently used" and evict
> your entire working set. A single backup job would destroy the database's cache.

So Linux maintains, per-node and per-memcg, **two LRU lists per type**:

```
        ┌──────────── active list ─────────────┐   "referenced more than once"
        │  protected; not scanned first        │
        └───────────────┬──────────────────────┘
                        │ demote when active list too large
        ┌───────────────▼──────────────────────┐
        │        inactive list                 │   "probationary — used once"
        │  scanned first; evicted from the tail│
        └──────────────────────────────────────┘
              ▲ promote on SECOND reference
```

A new page enters **inactive**. It is promoted to **active** only on a *second* reference.
Therefore a streaming scan cycles through the inactive list and never displaces the active
set. This is frequency-awareness bolted onto recency — a 2Q/LRU-2 approximation.

There are four such list pairs: `LRU_INACTIVE_ANON`, `LRU_ACTIVE_ANON`, `LRU_INACTIVE_FILE`,
`LRU_ACTIVE_FILE`, plus `LRU_UNEVICTABLE` (mlocked, ramfs).

```bash
grep -E 'nr_active_anon|nr_inactive_anon|nr_active_file|nr_inactive_file|nr_unevictable' /proc/vmstat
cat /proc/meminfo | grep -E 'Active|Inactive|Unevictable|Mlocked'
```

### T.3 Refault distance: reconstructing the information you threw away

This is the most elegant idea in Linux reclaim, and most kernel developers cannot explain it.
It is worth real study.

**The problem with the two-list scheme:** once a page is evicted, the kernel forgets it ever
existed. If it is read back immediately, the kernel cannot tell whether that was
(a) a genuinely hot page that should never have been evicted, or (b) a cold page being
streamed. Under memory pressure this causes **cache thrashing that is invisible to the
policy** — the kernel keeps evicting exactly the pages it is about to need.

**Johannes Weiner's solution (3.15, "workingset detection"):** when evicting a page, leave a
**shadow entry** in the page cache (an XArray *value entry* — Ch. 10 T.3, costing no
allocation) recording the value of a monotonically increasing **eviction counter** at the
moment of eviction.

When the page is read back, compute:

$$ \text{refault distance} = \text{evictions since} = E_{\text{now}} - E_{\text{stored}} $$

This is an estimate of the page's **reuse distance** — how many *distinct other pages* were
touched between the two accesses. And now the decision is principled:

> If `refault_distance <= size_of_active_list`, then had the page been allowed to stay, it
> **would have been re-referenced before reaching the eviction end of the list**. Therefore
> it deserved to be in the active list. **Promote it directly to active on refault.**

Otherwise, the page genuinely does not fit and should stay in the inactive list.

What makes this remarkable: it **reconstructs reuse distance — the quantity LIRS/ARC need —
from a single integer per evicted page, with zero memory cost for resident pages.** It is
effectively an online estimator for "would a larger cache have helped?", which is the
question any cache-sizing decision needs.

The same counters feed **`/proc/pressure/memory` (PSI)**: refault rate is a direct measure of
*time lost to insufficient memory*, which is far more actionable than "free memory".

```bash
grep -E 'workingset_' /proc/vmstat
#   workingset_refault_file / workingset_activate_file / workingset_restore_file
#   workingset_nodereclaim
cat /proc/pressure/memory
```
**A high `workingset_refault` rate with low `workingset_activate` means genuine thrashing.
A high activate rate means the detector is doing its job.**

### T.4 MGLRU: generations instead of lists

The two-list scheme has real limits: promotion/demotion decisions are coarse, the scan uses
`rmap` (expensive — Ch. 22 T.8), and the active/inactive ratio is a crude knob.

**MGLRU (Multi-Gen LRU, Yu Zhao, 6.1+)** replaces it with a **generational aging scheme**:

- Pages live in one of `MAX_NR_GENS` (≤4) generations per type.
- **Aging**: periodically increment the "max generation"; pages in old generations age out.
  Crucially, aging scans **page tables directly** rather than walking `rmap` backwards —
  far cheaper for processes with large mappings, because you read PTE accessed-bits
  sequentially instead of doing a reverse lookup per page.
- **Eviction**: reclaim from the oldest generation.
- Bloom filters track which page-table subtrees are worth scanning, so sparse address spaces
  are skipped.
- Per-generation refault feedback tunes the aging rate (the T.3 idea, generalized).

The theoretical framing: generations are a **coarser but much cheaper approximation of the
same aging signal**, and the cost saving buys you the ability to sample *more often*, which
makes the approximation *better* overall. That "cheaper approximation, sampled more, wins"
argument recurs throughout systems design.

```bash
cat /sys/kernel/mm/lru_gen/enabled           # 0x0007 = enabled
echo y | sudo tee /sys/kernel/mm/lru_gen/enabled
cat /sys/kernel/debug/lru_gen                # per-memcg generation state
cat /sys/kernel/debug/lru_gen_full
```

### T.5 The reclaim machinery

```
allocation fails watermark check (Ch. 11 T.8)
   │
   ├─ background: wake kswapd (per NUMA node kthread)  ── cheap, async
   │     balance_pgdat() → shrink_node() until the high watermark
   │
   └─ foreground: DIRECT RECLAIM in the allocating task ── ★ your latency spike
         __alloc_pages_slowpath() → __alloc_pages_direct_reclaim()
              → shrink_zones() → shrink_node()
                   ├─ shrink_lruvec()          page cache + anon (LRU lists)
                   │     get_scan_count()      ★ how much anon vs file to scan?
                   │     shrink_list() → shrink_folio_list()
                   │          folio_referenced()   (rmap: check accessed bits)
                   │          try_to_unmap()       (rmap: clear all PTEs)
                   │          pageout()/writeback  (dirty pages)
                   │          → free, or activate, or keep
                   └─ shrink_slab()            ★ SHRINKERS: everything else
```

**`get_scan_count()` decides the anon:file split**, and this is where `swappiness` lives:

```bash
cat /proc/sys/vm/swappiness     # 0..200 (>100 allowed since 5.8)
```
`swappiness` is **not** "how much to swap". It is the **relative cost ratio** the kernel
assumes between reclaiming anonymous memory (requires a swap write) and file memory
(may be free if clean, or requires writeback if dirty). Combined with measured refault rates
for each type, it produces a scan ratio. Hence:

- `swappiness=0` — only swap when file reclaim cannot keep up (near-OOM). Common for
  databases that manage their own caching.
- `swappiness=60` (default) — balanced.
- `swappiness=100` — treat anon and file as equally expensive.
- `swappiness=200` — favour swapping out anon (sensible with zram/zswap, where "swap" is a
  fast in-memory compression rather than a disk write).

**The key asymmetry:** a *clean* file page can be reclaimed for **free** (just drop it); a
*dirty* file page needs writeback; an *anonymous* page **always** needs a swap write (unless
it is a zero page or in a swap cache already). With no swap configured, anon memory is
**unreclaimable**, which is why swapless systems OOM abruptly — the kernel has only file
pages to work with and the anon footprint is an immovable floor.

### T.6 Writeback as a control system

Dirty pages must eventually reach storage. If applications dirty faster than storage can
absorb, memory fills with dirty pages that cannot be reclaimed → the system stalls. So
writeback is a **feedback control problem**, and Linux implements an explicit controller
(largely Fengguang Wu's work, 3.1+).

Three mechanisms:

1. **Background writeback.** Above `dirty_background_ratio` (default 10% of available
   memory), per-BDI (backing device info) flusher threads start writing, asynchronously.
2. **Throttling.** A task dirtying pages is made to *sleep* in
   `balance_dirty_pages()`, with the sleep duration computed from how far above the setpoint
   the system is and how fast the device is draining.
3. **Hard limit.** Above `dirty_ratio` (default 20%), dirtiers block hard.

The interesting part is the controller design:

- The kernel **estimates each device's writeback bandwidth** at runtime
  (`bdi->write_bandwidth`, an EWMA — the same technique as PELT, Ch. 21 T.5).
- It computes a **per-task dirty rate limit** so the *aggregate* dirty rate matches the
  device's drain rate, giving each dirtier a proportional share.
- It uses a **proportional controller with a setpoint** rather than a hard threshold, so
  throttling is smooth (microsecond sleeps) instead of the old "everything stops at 20%"
  cliff. This is genuine control theory in the kernel: `pos_ratio` is the proportional term,
  and the derivative of the dirty count damps oscillation.

```bash
cat /proc/sys/vm/dirty_background_ratio /proc/sys/vm/dirty_ratio
cat /proc/sys/vm/dirty_background_bytes /proc/sys/vm/dirty_bytes   # absolute alternative
cat /proc/sys/vm/dirty_expire_centisecs /proc/sys/vm/dirty_writeback_centisecs
grep -E 'Dirty|Writeback' /proc/meminfo
cat /sys/kernel/debug/bdi/*/stats        # per-device: BdiWriteBandwidth, DirtyThresh
```

**Why ratios are dangerous on big machines:** 20% of 1 TiB is 200 GiB of dirty data. If the
device drains at 500 MB/s, flushing that takes 400 seconds, and any `fsync()` or reclaim
attempt in the meantime stalls. **Use `dirty_bytes`/`dirty_background_bytes` on large-memory
systems**, sized to a few seconds of device bandwidth. This is one of the highest-value
production tunings in Linux and is still widely missed.

### T.7 Shrinkers: reclaiming everything that isn't a page

Page cache and anon memory are only part of the story. Dentries, inodes, XFS buffers, GPU
buffer objects, NFS caches, and dozens of other subsystems hold reclaimable memory. The
**shrinker** interface is the callback protocol for them:

```c
struct shrinker *s = shrinker_alloc(SHRINKER_NUMA_AWARE | SHRINKER_MEMCG_AWARE, "my-cache");
s->count_objects = my_count;   /* how many objects COULD I free? */
s->scan_objects  = my_scan;    /* free up to sc->nr_to_scan; return how many you freed */
s->seeks         = DEFAULT_SEEKS;   /* cost to RECREATE an object, in "seeks" */
shrinker_register(s);
...
shrinker_free(s);
```

The protocol is a **two-phase negotiation**: `shrink_slab()` asks every shrinker how much it
holds, computes a proportional share of the total reclaim target, and then asks each to free
that share. `seeks` is the cost model — an object that is expensive to recreate is reclaimed
less aggressively. This is proportional-share allocation (Ch. 21 T.2) applied to *reclaim
pressure* instead of CPU.

Rules that reviewers enforce:
- `count_objects` must be **cheap** and must not sleep long; it is called constantly.
- `scan_objects` runs with `__GFP_FS`/`__GFP_IO` possibly cleared — check `sc->gfp_mask` and
  **do not re-enter your own filesystem** (Ch. 11 T.7). This is the classic shrinker deadlock.
- Return `SHRINK_STOP` if you cannot make progress; returning 0 forever causes a livelock.
- Must be `SHRINKER_MEMCG_AWARE` if the objects are charged to a cgroup, or accounting lies.

```bash
grep -E 'slab|SReclaimable|SUnreclaim' /proc/meminfo
sudo cat /sys/kernel/debug/shrinker/*         # per-shrinker counts (6.7+)
echo 2 | sudo tee /proc/sys/vm/drop_caches     # 1=pagecache 2=slab 3=both — TESTING ONLY
sudo bpftrace -e 'tracepoint:vmscan:mm_shrink_slab_start { @[str(args->shr)] = count(); }'
```

### T.8 Compaction: defragmentation by migration

Ch. 11 T.3 established that fragmentation is the long-term enemy and that the cure requires
pages to be **movable**. Compaction is the mechanism:

```
   Fragmented pageblock:
   [U][ ][M][ ][M][U][ ][M]        U=unmovable  M=movable  [ ]=free

   Two scanners walk toward each other:
     - migration scanner: from the low end, finds MOVABLE pages
     - free scanner:      from the high end, finds FREE pages
   Move each movable page into a free slot, update every PTE via rmap.

   Result:
   [U][M][M][M][U][ ][ ][ ]        → contiguous free space at the top
```

Three things make this possible, and each is a non-trivial prerequisite:

1. **Migratetype segregation** (Ch. 11 T.3) keeps unmovable allocations clustered.
2. **Reverse mapping** (Ch. 22 T.8) lets you find and update every PTE pointing at a page.
3. **The `->migratepage`/`->migrate_folio` address_space op** lets a subsystem participate —
   a filesystem or driver can say how *its* pages are moved, or refuse.

Compaction runs in three modes: **`kcompactd`** (background, per-node), **proactive**
(`vm.compaction_proactiveness`, keeps fragmentation low pre-emptively), and **direct
compaction** (synchronous, inside an allocation — **the source of THP allocation stalls**,
Ch. 22 T.4).

```bash
cat /proc/buddyinfo /proc/pagetypeinfo
grep -E 'compact_' /proc/vmstat
#   compact_stall, compact_fail, compact_success, compact_daemon_wake,
#   compact_migrate_scanned, compact_free_scanned
cat /sys/kernel/debug/extfrag/extfrag_index     # 0..1000; ~1000 = badly fragmented
cat /proc/sys/vm/compaction_proactiveness
echo 1 | sudo tee /proc/sys/vm/compact_memory   # force a full compaction
```

**`extfrag_index` is the number to watch**: it measures *external fragmentation* per order —
"is allocation failing because memory is low, or because it is fragmented?" Those two
failures look identical from the outside and need opposite fixes.

### T.9 Memory cgroups: accounting, limits, and pressure

memcg answers "who is using this memory, and how do I bound them?" The mechanism is
**charging**: every chargeable allocation (user pages, page cache, and — with
`__GFP_ACCOUNT` — kernel slab objects) is attributed to a cgroup at allocation and
uncharged at free.

```c
struct page_counter {
	atomic_long_t usage;          /* per-CPU cached via memcg_stock */
	unsigned long min, low, high, max;
	struct page_counter *parent;  /* hierarchical: charging propagates up */
	...
};
```

cgroup v2's four knobs form a **protection/throttling ladder**, and understanding the
difference is essential:

| Knob | Semantics |
|---|---|
| `memory.min` | **hard protection** — memory below this is never reclaimed, even at OOM risk |
| `memory.low` | **best-effort protection** — reclaimed only if no unprotected memory remains |
| `memory.high` | **throttle** — over this, the task is aggressively reclaimed and *slowed down* (`mem_cgroup_handle_over_high()` sleeps proportionally). **No OOM kill.** |
| `memory.max` | **hard limit** — allocation fails or the cgroup OOM killer fires |

The design insight: `memory.high` provides **graceful degradation instead of a cliff**. A
workload exceeding `high` gets slower and slower (back-pressure) rather than being killed,
giving a userspace controller time to react. Combined with `memory.pressure` (PSI), this
enables **proactive, pressure-driven** management — the basis of `systemd-oomd` and Meta's
`oomd`/senpai. That is the modern answer to "the OOM killer is too late and too blunt".

```bash
cd /sys/fs/cgroup/myapp
cat memory.current memory.peak memory.min memory.low memory.high memory.max
cat memory.stat            # ★ the full breakdown: anon, file, slab, shmem, workingset_*
cat memory.events          # low, high, max, oom, oom_kill counts
cat memory.pressure        # ★ PSI: some/full avg10 avg60 avg300 total
cat memory.swap.current memory.swap.max
cat memory.reclaim         # write a byte count to force proactive reclaim (5.19+)
echo "1G" > memory.reclaim
```

**`memory.events` and `memory.pressure` are the two files to alert on in production.**
`memory.current` alone tells you nothing — a cgroup at 99% of its limit with zero pressure is
perfectly healthy.

### T.10 Swap, compression, and tiering

Classic swap trades a disk write for a freed page. Two developments changed the calculus:

**zram / zswap — trading CPU for I/O.** Compressing a page (lz4/zstd, ~1–3 µs) and keeping it
in RAM at a 2–4× ratio is far cheaper than a disk write. `zram` is a compressed *block
device* used as swap; `zswap` is a compressed *cache in front of* real swap. With these,
`swappiness=100..200` becomes reasonable — "swapping" no longer means "disk". This is why
ChromeOS, Android, and Fedora enable zram by default.

```bash
zramctl; cat /sys/block/zram0/mm_stat
cat /sys/module/zswap/parameters/{enabled,compressor,zpool,max_pool_percent}
cat /sys/kernel/debug/zswap/*
swapon --show
```

**Memory tiering / CXL — the new frontier.** With CXL-attached memory and persistent memory,
"memory" is no longer uniform: there are fast-local, slow-remote, and very-slow tiers. This
turns reclaim into **promotion/demotion between tiers** rather than eviction to disk:

- `mm/memory-tiers.c` builds a tier hierarchy from HMAT/ACPI distance data.
- Demotion: instead of swapping, migrate cold pages to a slower tier
  (`vm.demotion_enabled`).
- Promotion: NUMA hinting faults (Ch. 21 T.7) detect hot pages on slow tiers and promote.

The theory is a straightforward generalization — it is still Belady, with a multi-level cache
hierarchy and a cost per level — but the engineering is active and unfinished. Watch
`DAMON` (Data Access MONitor), which provides low-overhead access-frequency profiling and
`DAMOS` schemes to act on it ("if a region is cold for 10s, demote it"). DAMON is the most
interesting new MM subsystem and is worth learning now.

```bash
ls /sys/kernel/mm/damon/admin/
cat /sys/devices/virtual/memory_tiering/*/nodelist 2>/dev/null
cat /proc/sys/kernel/numa_balancing /proc/sys/vm/demote_scale_factor 2>/dev/null
```

### T.11 OOM: the last resort, and why it exists

Once you allow overcommit (Ch. 22 T.9), a shortfall is possible and something must resolve
it. The OOM killer picks a victim to maximize freed memory while minimizing damage:

```c
points = oom_badness(task)   /* ≈ RSS + swap + pagetables, in pages */
       + oom_score_adj * totalpages / 1000;
```

Policy choices encoded here: pick the *biggest* consumer (maximize benefit), prefer the
process that *triggered* the shortfall's cgroup, never kill kernel threads or init, and let
userspace bias with `oom_score_adj` (−1000 = immune, +1000 = kill first).

Three refinements you should know:
- **`memory.oom.group = 1`** — kill the whole cgroup atomically. Essential for containers,
  where killing one process of a multi-process workload leaves a broken half-alive service.
- **`oom_reaper`** — a kthread that asynchronously frees the victim's anonymous memory
  *without waiting for it to exit*, preventing the classic livelock where the victim itself
  needs memory to die.
- **`panic_on_oom`** — for systems where a reboot is better than an arbitrary kill
  (appliances, RT systems).

```bash
cat /proc/sys/vm/panic_on_oom /proc/sys/vm/oom_kill_allocating_task
cat /proc/<pid>/oom_score /proc/<pid>/oom_score_adj
cat /sys/fs/cgroup/myapp/memory.oom.group
dmesg | grep -A60 'invoked oom-killer'
```

---

## 1. Internals

### 1.1 Source map

```
mm/vmscan.c            ★★ shrink_node, shrink_lruvec, get_scan_count, shrink_folio_list,
                          kswapd, MGLRU (lru_gen_*)
mm/workingset.c        ★ refault distance / shadow entries (T.3)
mm/page-writeback.c    ★ balance_dirty_pages, the controller (T.6)
fs/fs-writeback.c      the flusher threads, wb_workfn
mm/compaction.c        ★ the two scanners, kcompactd
mm/migrate.c           migrate_pages, folio migration ops
mm/memcontrol.c        ★★ memcg: charging, page_counter, memory.high throttling
mm/oom_kill.c          oom_badness, oom_reaper
mm/swapfile.c, mm/swap_state.c, mm/page_io.c
mm/zswap.c, drivers/block/zram/
mm/shrinker.c          (6.7+) the shrinker registry
mm/memory-tiers.c, mm/damon/
mm/khugepaged.c        THP collapse
include/linux/mmzone.h ★ struct lruvec, struct pglist_data
Documentation/mm/      ★ multigen_lru.rst, damon/, balance.rst, page_reclaim.rst
Documentation/admin-guide/mm/  ★ concepts.rst, damon/, zswap.rst, multigen_lru.rst
Documentation/admin-guide/cgroup-v2.rst   ★★ the memory controller section
```

### 1.2 The `lruvec` — where the lists actually live

```c
struct lruvec {
	struct list_head        lists[NR_LRU_LISTS];
	spinlock_t              lru_lock;
	unsigned long           anon_cost, file_cost;     /* ★ measured reclaim cost */
	atomic_long_t           nonresident_age;          /* ★ the refault counter (T.3) */
	unsigned long           refaults[ANON_AND_FILE];
	unsigned long           flags;
#ifdef CONFIG_LRU_GEN
	struct lru_gen_folio    lrugen;                   /* MGLRU generations */
	struct lru_gen_mm_state mm_state;
#endif
	struct pglist_data      *pgdat;
};
```
There is one `lruvec` **per memcg per NUMA node** — the cross-product. That is what makes
per-cgroup reclaim possible: pressure in one cgroup scans only that cgroup's lists.

`anon_cost` / `file_cost` are measured, not assumed: the kernel tracks how much I/O each type
actually cost and feeds it back into `get_scan_count()`. `swappiness` biases this measured
ratio rather than overriding it.

---

## 2. Practice

### Lab 23.1 — Demonstrate scan resistance (T.2)

```bash
# 1. Build a "hot" working set and get it into the ACTIVE list
dd if=/dev/urandom of=/tmp/hot.dat bs=1M count=512
sync; echo 3 | sudo tee /proc/sys/vm/drop_caches
cat /tmp/hot.dat > /dev/null      # first read  -> inactive
cat /tmp/hot.dat > /dev/null      # second read -> promoted to ACTIVE
grep -E 'Active\(file\)|Inactive\(file\)' /proc/meminfo

# 2. Now stream a huge file through the cache (the "use-once" attack)
dd if=/dev/urandom of=/tmp/cold.dat bs=1M count=8192
cat /tmp/cold.dat > /dev/null
grep -E 'Active\(file\)|Inactive\(file\)' /proc/meminfo

# 3. Is the hot file still cached?
sudo apt install -y vmtouch
vmtouch /tmp/hot.dat       # should still be largely resident — THAT is scan resistance
vmtouch /tmp/cold.dat      # mostly evicted

# Compare with what plain LRU would have done (hot.dat would be gone).
```

### Lab 23.2 — Watch refault distance work (T.3)

```bash
grep -E 'workingset' /proc/vmstat > /tmp/ws.before

# Create pressure with a working set slightly larger than free RAM
free -h
stress-ng --vm 1 --vm-bytes 70% --vm-keep --timeout 60s &
# ...while repeatedly touching a file that should be hot:
for i in $(seq 1 30); do cat /tmp/hot.dat > /dev/null; sleep 1; done
wait

grep -E 'workingset' /proc/vmstat > /tmp/ws.after
diff /tmp/ws.before /tmp/ws.after
```
Read the deltas:
- `workingset_refault_file` — pages read back that had been evicted
- `workingset_activate_file` — of those, how many were **promoted straight to active**
  because their refault distance said they deserved to stay

**`activate / refault` is your "the cache is too small" ratio.** Near 1.0 means the kernel is
evicting pages it immediately needs — buy RAM or shrink the working set. Near 0 means
eviction decisions were correct.

Then see the shadow entries themselves:
```bash
grep -E 'nr_shadow|workingset_nodes' /proc/vmstat
sudo bpftrace -e 'kprobe:workingset_refault { @ = count(); }'
$EDITOR mm/workingset.c    # read the 150-line header comment — it IS the T.3 derivation
```

### Lab 23.3 — MGLRU vs the classic LRU (T.4)

```bash
cat /sys/kernel/mm/lru_gen/enabled

run_bench() {
  echo 3 | sudo tee /proc/sys/vm/drop_caches >/dev/null
  /usr/bin/time -v your-memory-heavy-workload 2>&1 | grep -E 'Elapsed|Maximum resident|Major|Minor'
  grep -E 'pgscan|pgsteal|workingset_refault' /proc/vmstat
}

echo 0 | sudo tee /sys/kernel/mm/lru_gen/enabled; echo "=== classic LRU"; run_bench
echo 7 | sudo tee /sys/kernel/mm/lru_gen/enabled; echo "=== MGLRU";       run_bench

# MGLRU internals
sudo cat /sys/kernel/debug/lru_gen | head -40
#   memcg ... node ...
#     <gen> <anon> <file>   per-generation page counts
sudo cat /sys/kernel/debug/lru_gen_full | head -40   # includes refault stats per gen

# Force aging / eviction manually (the debugfs command interface):
echo "+ 0 0 1 0" | sudo tee /sys/kernel/debug/lru_gen   # age memcg 0 node 0
```
**Record `pgscan_*` for both.** MGLRU's headline claim is far fewer pages scanned per page
reclaimed — i.e. a cheaper approximation that can be applied more often (T.4).

### Lab 23.4 — Direct reclaim: find your latency spikes (T.5)

```bash
# The counters that matter
grep -E 'allocstall|pgscan_direct|pgscan_kswapd|pgsteal_direct|pgsteal_kswapd|pageoutrun' /proc/vmstat

# Direct reclaim LATENCY histogram — the single most useful MM measurement
sudo bpftrace -e '
kprobe:try_to_free_pages   { @s[tid] = nsecs; }
kretprobe:try_to_free_pages /@s[tid]/ {
	@direct_reclaim_us = hist((nsecs - @s[tid]) / 1000);
	@by_comm[comm] = hist((nsecs - @s[tid]) / 1000);
	delete(@s[tid]); }'

# Who is causing it?
sudo bpftrace -e 'kprobe:try_to_free_pages { @[comm, kstack(8)] = count(); }'

# kswapd vs direct: the ratio tells you whether background reclaim is keeping up
sudo bpftrace -e '
tracepoint:vmscan:mm_vmscan_direct_reclaim_begin { @direct = count(); }
tracepoint:vmscan:mm_vmscan_wakeup_kswapd        { @kswapd_wakes = count(); }'

# Tune the watermark distance and re-measure (Ch. 11 T.8)
cat /proc/sys/vm/min_free_kbytes
cat /proc/sys/vm/watermark_scale_factor          # raise this to widen the kswapd band
sudo sysctl -w vm.watermark_scale_factor=200     # default 10 (=0.1%)
```
**The production rule:** if `pgscan_direct` is a significant fraction of `pgscan_kswapd`,
kswapd is not waking early enough — raise `watermark_scale_factor` or `min_free_kbytes` and
re-measure. This trades a little usable memory for a large p99 improvement.

### Lab 23.5 — Writeback throttling, measured (T.6)

```bash
# Baseline
grep -E 'Dirty|Writeback' /proc/meminfo
cat /proc/sys/vm/dirty_ratio /proc/sys/vm/dirty_background_ratio
sudo cat /sys/kernel/debug/bdi/$(mountpoint -d /)/stats

# Generate dirty pages faster than the device can drain
fio --name=dirty --rw=write --bs=1M --size=8G --ioengine=sync --numjobs=4 \
    --directory=/tmp --group_reporting &

watch -n1 'grep -E "Dirty:|Writeback:|NFS_Unstable" /proc/meminfo'

# Watch the controller throttle tasks
sudo bpftrace -e '
kprobe:balance_dirty_pages   { @s[tid] = nsecs; }
kretprobe:balance_dirty_pages /@s[tid]/ {
	@throttle_us = hist((nsecs - @s[tid])/1000); delete(@s[tid]); }'

sudo trace-cmd record -e writeback -- sleep 10
trace-cmd report | grep -E 'balance_dirty_pages|writeback_single_inode' | head -30

# Now do the production fix: absolute bytes instead of ratios
sudo sysctl -w vm.dirty_bytes=$((512*1024*1024)) \
                vm.dirty_background_bytes=$((128*1024*1024))
# re-run and compare the throttle histogram and fsync latency
```
**Measure `fsync()` latency before and after.** On a large-memory machine the improvement is
often an order of magnitude at p99, because you are no longer allowed to accumulate a
multi-hundred-second backlog.

### Lab 23.6 — Write a shrinker (T.7)

```c
// SPDX-License-Identifier: GPL-2.0
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/shrinker.h>
#include <linux/slab.h>
#include <linux/list.h>
#include <linux/spinlock.h>

struct cached_obj {
	struct list_head node;
	char             data[4096];
};

static LIST_HEAD(cache);
static DEFINE_SPINLOCK(cache_lock);
static unsigned long nr_cached;
static struct shrinker *my_shrinker;

static unsigned long my_count(struct shrinker *s, struct shrink_control *sc)
{
	/* Must be CHEAP. Called very frequently. */
	return READ_ONCE(nr_cached) ?: SHRINK_EMPTY;
}

static unsigned long my_scan(struct shrinker *s, struct shrink_control *sc)
{
	unsigned long freed = 0;
	struct cached_obj *o, *tmp;

	/* ★ Respect the caller's re-entrancy constraints (Ch. 11 T.7) */
	if (!(sc->gfp_mask & __GFP_FS))
		return SHRINK_STOP;     /* we'd need FS access to rebuild; don't recurse */

	spin_lock(&cache_lock);
	list_for_each_entry_safe(o, tmp, &cache, node) {
		if (freed >= sc->nr_to_scan)
			break;
		list_del(&o->node);
		kfree(o);
		nr_cached--;
		freed++;
	}
	spin_unlock(&cache_lock);

	pr_info("shrinker: asked for %lu, freed %lu, %lu left\n",
		sc->nr_to_scan, freed, nr_cached);
	return freed;
}

static void populate(unsigned long n)
{
	while (n--) {
		struct cached_obj *o = kzalloc(sizeof(*o), GFP_KERNEL);

		if (!o)
			break;
		spin_lock(&cache_lock);
		list_add(&o->node, &cache);
		nr_cached++;
		spin_unlock(&cache_lock);
	}
}

static int __init shr_init(void)
{
	my_shrinker = shrinker_alloc(0, "demo-cache");
	if (!my_shrinker)
		return -ENOMEM;
	my_shrinker->count_objects = my_count;
	my_shrinker->scan_objects  = my_scan;
	my_shrinker->seeks         = DEFAULT_SEEKS;
	shrinker_register(my_shrinker);

	populate(50000);           /* ~200 MiB of reclaimable cache */
	pr_info("populated %lu objects\n", nr_cached);
	return 0;
}

static void __exit shr_exit(void)
{
	struct cached_obj *o, *tmp;

	shrinker_free(my_shrinker);      /* ★ unregister BEFORE freeing the objects */
	list_for_each_entry_safe(o, tmp, &cache, node) { list_del(&o->node); kfree(o); }
}
module_init(shr_init); module_exit(shr_exit);
MODULE_LICENSE("GPL");
```
```bash
sudo insmod shrinkerdemo.ko
grep SReclaimable /proc/meminfo
sudo cat /sys/kernel/debug/shrinker/demo-cache-* 2>/dev/null

# Apply pressure and watch it get called:
stress-ng --vm 2 --vm-bytes 80% --timeout 30s
dmesg | grep shrinker | tail -20

# Or force it:
echo 2 | sudo tee /proc/sys/vm/drop_caches
sudo bpftrace -e 'tracepoint:vmscan:mm_shrink_slab_start { @[str(args->shr)] = count(); }'
```
Then **deliberately break it**: return 0 always from `my_scan`, apply pressure, and observe
the livelock. Then remove the `__GFP_FS` check and reason about why that is a deadlock.

### Lab 23.7 — Compaction: fragment, measure, defragment (T.8)

```bash
# Measure fragmentation
cat /proc/buddyinfo
cat /sys/kernel/debug/extfrag/extfrag_index    # per-order, 0..1000
cat /sys/kernel/debug/extfrag/unusable_index

# Fragment (use the Ch. 11 Lab 11.B module, or a real workload)
# Then try a high-order allocation and watch compaction fire:
grep -E 'compact_' /proc/vmstat > /tmp/c.before
echo 200 | sudo tee /proc/sys/vm/nr_hugepages     # demands order-9 contiguity
grep -E 'compact_' /proc/vmstat > /tmp/c.after
diff /tmp/c.before /tmp/c.after
cat /proc/meminfo | grep HugePages

# Direct compaction stalls (the THP pathology, Ch. 22 T.4)
sudo bpftrace -e '
kprobe:try_to_compact_pages   { @s[tid] = nsecs; }
kretprobe:try_to_compact_pages /@s[tid]/ {
	@stall_us = hist((nsecs - @s[tid])/1000);
	@by_comm[comm] = count(); delete(@s[tid]); }'

# Proactive compaction: keep fragmentation low in the background
cat /proc/sys/vm/compaction_proactiveness       # 0..100, default 20
sudo sysctl -w vm.compaction_proactiveness=50
watch -n2 'cat /sys/kernel/debug/extfrag/extfrag_index | head -3'

# Force full compaction and see buddyinfo recover
echo 1 | sudo tee /proc/sys/vm/compact_memory
cat /proc/buddyinfo

# Trace migrations
sudo bpftrace -e 'tracepoint:migrate:mm_migrate_pages { @[args->mode] = sum(args->succeeded); }'
```

### Lab 23.8 — memcg: the full protection/throttling ladder (T.9)

```bash
sudo mkdir -p /sys/fs/cgroup/demo
cd /sys/fs/cgroup/demo
cat cgroup.controllers                  # ensure "memory" is there
echo "+memory" | sudo tee ../cgroup.subtree_control 2>/dev/null

# (a) memory.max — hard limit, OOM kill
echo 200M | sudo tee memory.max
echo 0    | sudo tee memory.swap.max
( echo $BASHPID | sudo tee cgroup.procs >/dev/null
  stress-ng --vm 1 --vm-bytes 500M --timeout 20s ) &
watch -n1 'cat memory.current memory.events; echo ---; cat memory.pressure'
dmesg | tail -30       # cgroup OOM report

# (b) memory.high — THROTTLE instead of kill
echo max  | sudo tee memory.max
echo 200M | sudo tee memory.high
( echo $BASHPID | sudo tee cgroup.procs >/dev/null
  time stress-ng --vm 1 --vm-bytes 500M --vm-keep --timeout 30s ) &
watch -n1 'cat memory.current; grep -E "high|max" memory.events; cat memory.pressure'
# Note: NO kill. The workload just gets slower. Measure HOW much slower.
sudo bpftrace -e 'kprobe:mem_cgroup_handle_over_high { @throttles = count(); }'

# (c) memory.min / memory.low — protection
echo 100M | sudo tee memory.min
# put a workload in a SIBLING cgroup and apply global pressure;
# the protected cgroup should keep its pages

# (d) memory.reclaim — proactive reclaim from userspace (5.19+)
echo "50M" | sudo tee memory.reclaim
cat memory.current

# (e) The full accounting breakdown
cat memory.stat | grep -E '^(anon|file|kernel|slab|sock|shmem|file_mapped|workingset)'
cat memory.numa_stat

# (f) OOM group kill
echo 1 | sudo tee memory.oom.group
```
**Deliverable:** a table of `memory.current`, `memory.events`, `memory.pressure` (some/full),
and wall-clock runtime for each of max/high/min. That table *is* T.9.

### Lab 23.9 — PSI-driven management (T.9)

```bash
cat /proc/pressure/memory /proc/pressure/io /proc/pressure/cpu
# some avg10=... avg60=... avg300=... total=...
# full avg10=... ...

# "some" = at least one task stalled; "full" = ALL tasks stalled (system-wide waste)

# Write a PSI trigger: wake me when memory pressure exceeds 50ms in any 1s window
cat > /tmp/psi.c <<'EOF'
#include <fcntl.h>
#include <poll.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
int main(void) {
	int fd = open("/proc/pressure/memory", O_RDWR | O_NONBLOCK);
	const char *trig = "some 50000 1000000";      /* 50ms out of 1s */
	if (write(fd, trig, strlen(trig) + 1) < 0) { perror("write"); return 1; }
	struct pollfd p = { .fd = fd, .events = POLLPRI };
	while (1) {
		if (poll(&p, 1, -1) > 0 && (p.revents & POLLPRI)) {
			char buf[256]; lseek(fd, 0, SEEK_SET);
			read(fd, buf, sizeof(buf));
			printf("PRESSURE EVENT:\n%s\n", buf);
		}
	}
}
EOF
gcc -O2 -o /tmp/psi /tmp/psi.c && /tmp/psi &
stress-ng --vm 4 --vm-bytes 80% --timeout 30s
```
This is exactly how `systemd-oomd` works. Compare its reaction time with the kernel OOM
killer's.

### Lab 23.10 — zram/zswap: trade CPU for I/O (T.10)

```bash
# zram as swap
sudo modprobe zram num_devices=1
echo zstd | sudo tee /sys/block/zram0/comp_algorithm
echo 4G   | sudo tee /sys/block/zram0/disksize
sudo mkswap /dev/zram0 && sudo swapon -p 100 /dev/zram0
swapon --show
sudo sysctl -w vm.swappiness=180        # swapping is now cheap

stress-ng --vm 2 --vm-bytes 70% --vm-keep --timeout 60s
cat /sys/block/zram0/mm_stat
#  orig_data_size compr_data_size mem_used_total ... → compute the compression ratio
echo "ratio: $(awk '{printf "%.2f", $1/$2}' /sys/block/zram0/mm_stat)"

# zswap (in front of a real swap device)
echo 1     | sudo tee /sys/module/zswap/parameters/enabled
echo zstd  | sudo tee /sys/module/zswap/parameters/compressor
echo zsmalloc | sudo tee /sys/module/zswap/parameters/zpool
echo 20    | sudo tee /sys/module/zswap/parameters/max_pool_percent
sudo cat /sys/kernel/debug/zswap/*

# Compare: same workload with disk swap, zram swap, zswap
sudo bpftrace -e 'kprobe:swap_writepage { @[comm] = count(); }'
grep -E 'pswpin|pswpout' /proc/vmstat
```

### Lab 23.11 — DAMON: profile access patterns cheaply (T.10)

```bash
./scripts/config -e DAMON -e DAMON_VADDR -e DAMON_PADDR -e DAMON_SYSFS -e DAMON_RECLAIM
ls /sys/kernel/mm/damon/admin/kdamonds/

# Minimal setup: monitor a PID's virtual address space
D=/sys/kernel/mm/damon/admin/kdamonds
echo 1 | sudo tee $D/nr_kdamonds
echo 1 | sudo tee $D/0/contexts/nr_contexts
echo vaddr | sudo tee $D/0/contexts/0/operations
echo 1 | sudo tee $D/0/contexts/0/targets/nr_targets
echo $PID | sudo tee $D/0/contexts/0/targets/0/pid_target
echo on | sudo tee $D/0/state

# Read the access-frequency heatmap
sudo cat /sys/kernel/debug/damon/monitor_on 2>/dev/null
# or use the userspace tool:
pip install --user damo
sudo damo record -o /tmp/damon.data $PID
sudo damo report heats --heatmap stdout
sudo damo report wss                     # working set size over time

$EDITOR Documentation/admin-guide/mm/damon/
```
**DAMON gives you Denning's working set, measured, with ~0.1% overhead.** That is a genuinely
new capability and the basis for tiering and proactive reclaim policies.

### Lab 23.12 — Read an OOM report properly

```bash
# Trigger one in a VM
sudo swapoff -a
stress-ng --vm 4 --vm-bytes 95% --vm-keep --timeout 60s
dmesg | grep -B5 -A80 'invoked oom-killer'
```
Annotate every section:
1. `<comm> invoked oom-killer: gfp_mask=0x..., order=N, oom_score_adj=M`
   — **decode the gfp_mask against Ch. 11 T.7's lattice**; the `order` tells you whether this
   was a fragmentation failure or a genuine shortage.
2. The call trace — who was allocating.
3. `Mem-Info:` — `active_anon`, `inactive_file`, `isolated`, `dirty`, `writeback`,
   `unstable`, `slab_reclaimable`, `slab_unreclaimable`, `mapped`, `shmem`, `pagetables`.
4. Per-node, per-zone free/min/low/high and the free-block histogram
   (**this is `/proc/buddyinfo` at the moment of death**).
5. The task table with `rss`, `pgtables_bytes`, `swapents`, `oom_score_adj`.
6. `Out of memory: Killed process N (name) total-vm:... anon-rss:... file-rss:... shmem-rss:...`

**Diagnostic questions to answer from the report alone:** Was memory actually exhausted, or
just fragmented (check `order` and the free-block histogram)? Was it reclaimable but dirty
(check `dirty`/`writeback`)? Was slab the problem (`slab_unreclaimable`)? Was it a cgroup
limit or global? Being able to answer these from a customer's `dmesg` with no access to the
machine is a senior-level skill.

---

## 3. Mastery drills

1. **Derive the refault rule.** Without looking, reconstruct the argument in T.3: why does
   `refault_distance <= nr_active` imply the page should be activated? Then read
   `mm/workingset.c`'s header comment and check yourself. Explain what changes when memcgs
   are involved.

2. **Read `shrink_folio_list()`** (`mm/vmscan.c`) line by line. Enumerate **every** reason a
   page is kept rather than reclaimed (dirty, writeback, mapped-and-referenced, pinned,
   locked, unevictable, under migration…). Which of these can a driver cause?

3. **`get_scan_count()`.** Read it and explain precisely how `swappiness`, `anon_cost`,
   `file_cost`, and the refault rates combine. Then predict the behaviour at
   `swappiness=0`, `60`, `200` for a workload with (a) 90% file / 10% anon and
   (b) 10% file / 90% anon. Verify by measurement.

4. **MGLRU design.** Read `Documentation/mm/multigen_lru.rst`. Explain (a) why scanning page
   tables can be cheaper than `rmap` walks, (b) what the Bloom filters are for, (c) how
   generations replace the active/inactive promotion decision. Then find a workload where
   MGLRU loses and explain why.

5. **Writeback control theory.** Read `balance_dirty_pages()` and the comments in
   `mm/page-writeback.c`. Identify the setpoint, the proportional term (`pos_ratio`), and the
   bandwidth estimator. Sketch the control loop as a block diagram. What damps oscillation?

6. **The dirty-ratio trap.** For a 512 GiB machine with a 1 GB/s device: compute the worst
   case dirty backlog at `dirty_ratio=20`, and the time to flush it. Then compute the
   `dirty_bytes` you would set for a 2-second flush target. Justify the choice to a DBA.

7. **Shrinker audit.** `git grep -l 'shrinker_alloc\|register_shrinker' | head -20`. Pick
   three from different subsystems. For each, state what it frees, what `seeks` it uses and
   why, and whether it correctly handles `!__GFP_FS`.

8. **Compaction vs. reclaim.** Both free memory. Explain when each is the right tool, and
   how `__alloc_pages_slowpath()` decides between them. Then find `should_compact_retry()`
   and explain the retry policy.

9. **memcg charging.** Trace a `malloc()`+touch in a cgroup from the page fault to
   `mem_cgroup_charge()`. Find the per-CPU stock (`memcg_stock`) and explain why it exists
   (Ch. 16!). What happens on `memory.max` overflow — exactly which function decides between
   reclaim, throttle, and OOM?

10. **`memory.high` vs `memory.max`.** Write 400 words for an SRE audience explaining when to
    use each, what "graceful degradation" buys, and how to alert on `memory.events`. Include
    the PSI signal.

11. **Thrashing, quantified.** Construct a workload whose working set is 1.1× available RAM.
    Measure: throughput, `pgmajfault` rate, `workingset_refault`, `/proc/pressure/memory`
    `full`. Then reduce the working set to 0.9× and re-measure. **Plot the phase transition**
    from T.1.

12. **Design question.** You run a 1 TiB machine with three workloads: a latency-critical
    in-memory database (400 GiB, manages its own cache), a batch analytics job (elastic), and
    a log shipper (small, must never die). Design the memcg configuration: `min`, `low`,
    `high`, `max`, `swap.max`, `oom.group`, `swappiness`, plus global `dirty_bytes`,
    `watermark_scale_factor`, and whether to enable THP/MGLRU/zswap. Justify every value and
    state the PSI alerts you would configure.

13. **Tiering.** Read `mm/memory-tiers.c` and the DAMON docs. Design a demotion policy for a
    machine with 256 GiB DRAM + 1 TiB CXL memory. What signal drives demotion? What prevents
    oscillation between tiers? Compare with the classic two-list LRU.

---

## 4. Further reading

**Kernel documentation:**
- `Documentation/mm/multigen_lru.rst` ★ — the MGLRU design, by its author
- `Documentation/mm/balance.rst`, `page_reclaim.rst`, `page_migration.rst`
- `Documentation/admin-guide/mm/concepts.rst` ★ — the best short overview of T.1–T.5
- `Documentation/admin-guide/mm/transhuge.rst`, `zswap.rst`, `damon/` ★
- `Documentation/admin-guide/cgroup-v2.rst` ★★ — the memory controller section is
  **the** reference for T.9 and is exceptionally well written
- `Documentation/accounting/psi.rst` ★
- `Documentation/admin-guide/sysctl/vm.rst` ★★ — every `vm.*` knob, explained
- `mm/workingset.c` and `mm/page-writeback.c` header comments — both are essays

**Papers:**
- Belady, "A Study of Replacement Algorithms for a Virtual-Storage Computer" (IBM Sys. J.
  1966) — MIN, and Belady's anomaly
- Denning, "The Working Set Model for Program Behavior" (CACM 1968) ★ — thrashing
- Denning, "Thrashing: Its Causes and Prevention" (AFIPS 1968)
- Mattson et al., "Evaluation Techniques for Storage Hierarchies" (IBM Sys. J. 1970) — stack
  algorithms and reuse distance, the theory underneath T.3
- Jiang & Zhang, "LIRS: An Efficient Low Inter-reference Recency Set Replacement Policy"
  (SIGMETRICS 2002) — reuse-distance-based replacement
- Megiddo & Modha, "ARC: A Self-Tuning, Low Overhead Replacement Cache" (FAST 2003)
- Johnson & Shasha, "2Q: A Low Overhead High Performance Buffer Management Replacement
  Algorithm" (VLDB 1994) — Linux's active/inactive scheme
- Corbató, "A Paging Experiment with the Multics System" (1968) — the origin of CLOCK
- Weiner et al., "TMO: Transparent Memory Offloading in Datacenters" (ASPLOS 2022) —
  PSI-driven proactive reclaim at Meta scale; **the modern production argument**
- Lagar-Cavilla et al., "Software-Defined Far Memory in Warehouse-Scale Computers"
  (ASPLOS 2019) — Google's zswap-based cold-page offloading

**LWN:**
- "Better active/inactive list balancing" and "Thrash detection-based file cache sizing"
  (the workingset/refault series) ★
- "The multi-generational LRU" series ★
- "Proactive compaction", "Fragmentation avoidance"
- "The pressure stall information interface"
- "Memory control group v2: the high limit"
- "DAMON: data access monitoring", "DAMOS: DAMON-based operation schemes"
- "Memory tiering and CXL" / LSFMM coverage each year

**Books & talks:**
- Gorman, *Understanding the Linux Virtual Memory Manager*, Ch. 10–14
- Gregg, *Systems Performance* 2e, Ch. 7 (Memory) — the measurement discipline
- Johannes Weiner's talks on PSI and memory pressure (LPC, Kernel Recipes)

→ Next: [24-syscalls-uapi.md](24-syscalls-uapi.md)

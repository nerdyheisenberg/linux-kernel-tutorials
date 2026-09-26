# Chapter 52 — The page cache, folios, and writeback

> **Goal:** Understand the page cache as the single structure that makes file I/O fast, why it is *unified* with the memory management of Ch. 22–23 rather than separate, why `struct page` had to become `struct folio` and what that fixed, how readahead is a prediction problem with an economic asymmetry, why writeback is a control system rather than a queue drain, and where the `dirty_ratio` cliff comes from. By the end you can explain every field in `/proc/meminfo`'s storage section, predict when a write will block, and tune a system's writeback behaviour from first principles rather than from a blog post.

---

## Theory & First Principles

### T.0 — Start here: read the same file twice

```bash
dd if=/dev/zero of=/tmp/f bs=1M count=512 && sync
echo 3 | sudo tee /proc/sys/vm/drop_caches    # empty the cache

time cat /tmp/f > /dev/null     # first read:  ~0.5 s   (from the DEVICE)
time cat /tmp/f > /dev/null     # second read: ~0.05 s  (from MEMORY)
```

A 10–100× difference, with no change to the program, the filesystem, or the device. **The
second read never touched the disk.**

That is the page cache, and the idea is one sentence:

> **Every page of every file that has been read or written recently is kept in memory, and
> file I/O is redirected to those pages.** Memory *is* the filesystem; the disk is where it
> is backed up.

**The numbers make it non-optional**, not an optimization (Ch. 00 §T.3):

```
  page cache hit:   ~80 ns     (a memory access)
  NVMe miss:        ~80 us     (1,000x)
  HDD miss:         ~10 ms     (125,000x)
```

At a 90% hit rate on NVMe, the cache is doing 99.9% of the work. Which is why Ch. 23 §T.0's
`free -h` shows nearly all your memory as "cache" and why that is correct.

**Now the hard half, which is where the design lives.** Reads are easy — cache a copy, throw
it away when memory is needed. **Writes are not**, because a dirty page is *the only copy* of
your data:

```
  write(fd, buf, 4096)
       |
       +-> copy into the page cache, mark the page DIRTY, return
           ^
           |   The data now exists ONLY in volatile memory.
           |   Power loss = data loss.
           |
           +-- so: when does it get written back?
```

**Three forces, in direct opposition**, and every writeback tunable is a point in this space:

| Write back *sooner* | Write back *later* |
|---|---|
| Less data at risk | Overwrites are absorbed: a file written 10 times is written *once* |
| Smoother, more predictable I/O | Small adjacent writes merge into large sequential ones |
| | Deletes before writeback cost *nothing* at all |

**Delay is not laziness — it is where most of the performance comes from.** A compiler writing
a 4 KiB object file that is deleted 200 ms later never touches the disk at all.

**And the failure mode this creates, which you will meet in production.** The thresholds are
*percentages of RAM* by default:

```bash
sysctl vm.dirty_background_ratio     # 10 -- start writeback in the background
sysctl vm.dirty_ratio                # 20 -- BLOCK the writer until it drains
```

On a 512 GiB server, 20% is **100 GiB of dirty pages**. When that threshold is hit, writers
stall while 100 GiB drains to a device that does maybe 2 GB/s — **a 50-second stall**, and a
classic "the database froze" incident. On large-memory machines always use the `_bytes`
variants instead (Ch. 94 §T.9).

**One more thing to notice before §T.1**, because it is the deepest idea here: `mmap()` and
`read()` are the *same mechanism*.

```
  read(fd, buf, 4096)   -> find the page-cache page, memcpy into your buffer
  mmap(...)             -> find the page-cache page, MAP IT INTO YOUR PAGE TABLES
```

Same pages, same cache. `mmap` just skips the copy. That unification — the page cache being
simultaneously the file cache and the backing store for memory mappings — is what makes the
virtual memory system and the filesystem one subsystem rather than two (Ch. 22 §T.0).

```bash
cat /proc/meminfo | grep -E 'Cached|Dirty|Writeback'
sudo /usr/share/bcc/tools/cachestat 1     # hits/misses live
sudo /usr/share/bcc/tools/writeback       # when writeback runs and why
```

---

### T.1 The problem: bytes, pages, and the 10,000× gap

Chapter 51 §T.1 named the first impedance mismatch: applications write bytes at arbitrary offsets, devices transfer blocks. But that framing understates it. The real numbers:

| | Latency | Granularity |
|---|---|---|
| DRAM access | 80 ns | 64 B (cache line) |
| NVMe 4 K read | 20–80 µs | 4096 B |
| HDD 4 K random read | 10 ms | 4096 B (plus a seek) |

The ratio between DRAM and NVMe is ~500×; between DRAM and HDD, ~125,000×. No amount of clever block-layer work closes a gap of that size. The only thing that closes it is **not going to the device at all**, which means caching.

So the page cache's problem statement is:

> **Given a file's contents on a device 10³–10⁵× slower than memory, keep the useful parts in memory, serve reads from memory, absorb writes into memory, and move data to and from the device at times and in sizes that suit the device rather than the application.**

Four sub-problems, each of which is a section below:

1. **Indexing** — given (file, offset), find the cached data in O(1)-ish (§T.3).
2. **Prediction** — fetch data before it is asked for (§T.5).
3. **Absorption** — let writes complete without waiting, and decide when to write back (§T.6, §T.7).
4. **Eviction** — decide what to drop when memory is short (Ch. 23, referenced here).

### T.2 Why the page cache is not a separate cache

A naive design gives the filesystem a fixed-size buffer cache, as early UNIX did (a tunable number of buffers, separate from the VM's page pool). Linux does not, and the reason is a genuine insight:

> **A page of file data in the cache and a page of anonymous memory are competing for the same resource, and the OS cannot know a priori which is more valuable.**

If the buffer cache is a fixed 10 % of RAM, then a machine running a database gets 10 % cache regardless of whether the other 90 % is idle, and a machine running a compute job wastes 10 % on a cache nothing uses. A *unified* page cache lets the reclaim algorithm (Ch. 23) arbitrate between file and anonymous pages using the same machinery — LRU lists, refault distances, `swappiness` as the cost ratio.

Three further consequences follow from unification, and each is structural:

**(a) `mmap()` and `read()` are the same mechanism.** A `read()` copies from a page-cache folio; an `mmap()` maps that same folio into the process's page tables. There is exactly one copy of the data. This is why `mmap` and `read` are coherent with each other, which is not true on all operating systems and is a genuine simplification.

**(b) The page cache participates in reclaim, so it is *free* memory in the accounting sense.** The `Cached:` line in `/proc/meminfo` is mostly reclaimable, which is why "Linux ate my RAM" is a perennial misunderstanding. `MemAvailable:` exists specifically to express "how much could you get if you asked."

**(c) Dirty pages are the hard case.** A clean file page can be dropped instantly — the data is on disk. A dirty page cannot be reclaimed until it is written. So dirty pages are *unreclaimable memory with an unbounded creation rate*, which is why writeback needs a control system rather than a queue (§T.7). This single fact is the origin of `dirty_ratio`, `balance_dirty_pages()`, and most writeback pathology.

### T.3 The address_space: one cache per file, indexed by a radix tree

Each cacheable object gets a `struct address_space`:

```c
struct address_space {
	struct inode		*host;
	struct xarray		i_pages;      /* offset -> folio */
	struct rw_semaphore	invalidate_lock;
	gfp_t			gfp_mask;
	atomic_t		i_mmap_writable;
	struct rb_root_cached	i_mmap;       /* VMAs mapping this (Ch. 22) */
	unsigned long		nrpages;
	pgoff_t			writeback_index;
	const struct address_space_operations *a_ops;
	unsigned long		flags;
	errseq_t		wb_err;       /* the fsyncgate fix (Ch. 51 T.5b) */
	spinlock_t		private_lock;
	struct list_head	private_list;
};
```

The index is an **XArray** (Ch. 10 §T.2): a radix tree keyed by page offset within the file, with 6-bit (64-way) fan-out. Why a radix tree and not a hash table?

| Property | Why it matters here |
|---|---|
| **Range queries are natural** | writeback, `fsync`, truncate, and readahead all operate on *ranges* of offsets. A hash table cannot enumerate a range. |
| **Sparse keys are cheap** | a 1 TiB file with 3 cached pages costs a few nodes, not a 1 TiB array. |
| **Tags** | the XArray stores per-entry tags (`DIRTY`, `WRITEBACK`, `TOWRITE`) *in the interior nodes*, so "find the next dirty page in this range" is a tree descent that skips entire clean subtrees — not a linear scan. |
| **RCU lookup** | readers walk it lock-free (Ch. 15), which matters because this is one of the hottest lookups in the kernel. |

The tag propagation is the elegant part and worth stating precisely: an interior node carries a tag bit if *any* descendant carries it. So `tag_pages_for_writeback()` over a 1 TiB file with 100 dirty pages visits O(100 · log n) nodes, not 2²⁸ slots. Without this, `fsync()` on a large sparse file would be unusable.

`a_ops` is the filesystem's side of the contract:

```c
struct address_space_operations {
	int  (*read_folio)(struct file *, struct folio *);
	void (*readahead)(struct readahead_control *);
	int  (*writepages)(struct address_space *, struct writeback_control *);
	bool (*dirty_folio)(struct address_space *, struct folio *);
	int  (*write_begin)(struct file *, struct address_space *, loff_t pos,
			    unsigned len, struct folio **, void **fsdata);
	int  (*write_end)(struct file *, struct address_space *, loff_t pos,
			  unsigned len, unsigned copied, struct folio *, void *);
	sector_t (*bmap)(struct address_space *, sector_t);
	void (*invalidate_folio)(struct folio *, size_t offset, size_t len);
	bool (*release_folio)(struct folio *, gfp_t);
	ssize_t (*direct_IO)(struct kiocb *, struct iov_iter *);
	int  (*migrate_folio)(struct address_space *, struct folio *dst,
			      struct folio *src, enum migrate_mode);
	int  (*launder_folio)(struct folio *);
	bool (*is_partially_uptodate)(struct folio *, size_t from, size_t count);
	int  (*error_remove_folio)(struct address_space *, struct folio *);
};
```

Note what is **not** there: no `read()` and no `write()`. The page cache does the copying; the filesystem only supplies "fill this folio from the device" and "map these file offsets to blocks." That division is what makes `generic_file_read_iter()` and `generic_perform_write()` shareable across every filesystem.

Note also `read_folio` (singular, mandatory) versus `readahead` (batched, optional). The old `->readpage`/`->readpages` pair was replaced in 5.18 because `->readpages` had an awkward contract (a list of pages the filesystem might or might not consume). `->readahead` takes a `readahead_control` and the filesystem pulls folios from it — a cleaner ownership model.

### T.4 Folios: fixing a type error that cost a decade

`struct page` describes 4 KiB. For twenty years, every subsystem that wanted to handle larger units — huge pages, compound pages, large file extents — did so by convention: "this is a head page, the next N are tails, and you must call `compound_head()` before doing almost anything." The compiler could not check it, and getting it wrong produced memory corruption.

The concrete costs of 4 KiB granularity:

| Cost | Detail |
|---|---|
| Per-page overhead | `struct page` is 64 B for 4096 B: **1.5625 % of all RAM** (Ch. 22 §T.5) |
| Per-page work | reading a 1 MiB file touches 256 page structs, takes 256 refcounts, walks 256 XArray slots |
| Lock traffic | 256 `folio_lock()`/unlock pairs |
| Bio construction | 256 `bio_add_page()` calls, each checking merge eligibility |
| TLB pressure | 256 mappings instead of one |

`struct folio` (5.16+, Matthew Wilcox) is the fix, and it is a **type-system fix, not a performance hack**:

```c
struct folio {
	unsigned long flags;
	union { struct list_head lru; ... };
	struct address_space *mapping;
	pgoff_t index;
	void *private;
	atomic_t _mapcount;
	atomic_t _refcount;
	/* ... */
	unsigned int _folio_nr_pages;   /* for large folios */
};
```

The declaration that matters is the *invariant*, not the fields:

> **A folio is never a tail page.** If you have a `struct folio *`, you have the head, and `folio_nr_pages()` tells you how big it is.

That single guarantee is what the compiler now enforces (via distinct types), and it eliminates the entire class of "forgot `compound_head()`" bugs while making large-unit handling natural rather than exceptional.

The API mirrors the old one with the ambiguity removed:

| Old | New | Difference |
|---|---|---|
| `lock_page(p)` | `folio_lock(f)` | operates on the whole folio |
| `get_page(p)` | `folio_get(f)` | one refcount for N pages |
| `set_page_dirty(p)` | `folio_mark_dirty(f)` | one dirty bit for N pages |
| `page_mapping(p)` | `folio_mapping(f)` | no `compound_head()` needed |
| `PageUptodate(p)` | `folio_test_uptodate(f)` | |
| — | `folio_size(f)`, `folio_nr_pages(f)` | **new: size is explicit** |

**Large folios in the page cache** (progressively enabled from 5.18 through 6.x) are the payoff: a filesystem can now cache a 64 KiB or 2 MiB range as one folio. Measured effects include 10–30 % improvements on large sequential I/O, large reductions in `struct page` cache-miss traffic, and — importantly — far fewer bio segments, because one folio is one contiguous physical range.

The constraint is fragmentation: allocating an order-4 folio requires 16 contiguous pages, which may not exist. So the cache asks for large folios and falls back (`mapping_set_large_folios()`, `filemap_alloc_folio()` with order hints), and readahead drives the sizing decision (§T.5).

### T.5 Readahead: prediction with an asymmetric payoff

Reading a file sequentially, page by page, one 80 µs NVMe round trip at a time, gives 50 MB/s from a device capable of 7 GB/s. The fix is to read ahead — but reading ahead is *speculation*, and the design question is how much to bet.

The economics are asymmetric, which is the key insight:

- **A correct prediction saves a full device round trip** (20 µs–10 ms).
- **A wrong prediction costs some bandwidth and some memory**, both of which are cheap and reclaimable.

So the optimal strategy is aggressive: predict eagerly, back off quickly when wrong. Linux implements this with a two-window scheme in `mm/readahead.c`:

```
   file offsets ──────────────────────────────────────────────►
   [ ....... already read ....... |  current window  | ahead window ]
                                          ▲                ▲
                                    app is reading    prefetched;
                                        here          async trigger
                                                      marker inside
```

The mechanism:

1. On a cache miss at offset `i`, if the access looks sequential (`ra->start + ra->size == i`), issue a synchronous read for the current window **and** an asynchronous read for the next window.
2. Mark one folio in the ahead window with `PG_readahead`. When the application's read *touches* that folio, that is the signal that it consumed the window, and the next async readahead is triggered — **before** the application stalls.
3. Double the window size on each successful sequential hit, up to `read_ahead_kb` (default 128 KiB, often raised to 1–4 MiB).
4. On a random access pattern, collapse the window to zero and read only what was asked.

The `PG_readahead` marker is the clever part. It converts "predict when to prefetch" into "observe when the application crosses a tripwire," which requires no timers and no rate estimation. It is a *demand-driven* pipeline: the application's own progress paces the prefetcher.

Explicit control is available and under-used:

| Interface | Effect |
|---|---|
| `posix_fadvise(fd, off, len, POSIX_FADV_WILLNEED)` | start readahead now |
| `POSIX_FADV_SEQUENTIAL` | double the readahead window |
| `POSIX_FADV_RANDOM` | disable readahead |
| `POSIX_FADV_DONTNEED` | **drop clean pages** for this range |
| `POSIX_FADV_NOREUSE` | hint: do not keep after reading |
| `readahead(fd, off, count)` | Linux-specific, explicit |
| `madvise(MADV_WILLNEED/SEQUENTIAL/RANDOM)` | the mmap equivalents |
| `/sys/block/*/queue/read_ahead_kb` | per-device default |
| `blockdev --setra` | same, older interface |

`POSIX_FADV_DONTNEED` is the one worth remembering: it is how a backup program avoids destroying the cache for everyone else. Reading 500 GB without it evicts everything useful; with it, the backup's pages are dropped as it goes. This is a deliberate, cooperative alternative to the kernel guessing (and relates to the scan-resistance problem of Ch. 23 §T.3 — `DONTNEED` lets the application tell the truth that the heuristic is trying to infer).

Readahead also drives large-folio sizing: `page_cache_ra_order()` uses the readahead window size to choose the folio order, so a workload reading sequentially naturally gets large folios and a random workload gets 4 KiB ones. Prediction and allocation granularity are the same decision.

### T.6 Writes: absorption, and why `write()` returns before anything happens

A buffered `write()` does five things and none of them is I/O:

```c
generic_perform_write():
  1. a_ops->write_begin()   /* find or create the folio; ask the fs to
                               allocate blocks (or defer -- see below) */
  2. copy_folio_from_iter_atomic()   /* the memcpy. This is the write. */
  3. a_ops->write_end()     /* update i_size, mark uptodate */
  4. folio_mark_dirty()     /* tag DIRTY in the XArray; attach to a wb */
  5. balance_dirty_pages_ratelimited()   /* the throttle -- see T.7 */
```

Step 2 is the entire user-visible cost in the common case: a `memcpy` into a folio that is already resident. That is why buffered writes measure ~1 µs (Ch. 51 Lab 3) and why applications that do not `fsync` see storage as approximately free.

Three things deserve emphasis:

**(a) `write_begin` may have to read first.** If you write 100 bytes into the middle of a 4 KiB page that is not cached, the page must be read from disk before the partial write, because the rest of it must be correct when written back. This is **read-modify-write**, and it is why misaligned small writes to uncached data are far more expensive than their size suggests. Filesystems avoid it when the write covers the whole folio (or the whole block), which is why `O_DIRECT` requires alignment and why databases use page sizes matching the filesystem's.

**(b) Delayed allocation moves the decision later.** ext4, XFS, and Btrfs do not allocate blocks in `write_begin`; they record a reservation and allocate at writeback time, when they know the full extent of the dirty range. The result is far better contiguity — one 100 MB extent instead of 25,600 individually allocated 4 KiB blocks. The trade: `ENOSPC` can surface at writeback time rather than at `write()` time, and a crash before writeback loses data that `write()` appeared to accept. (It does not violate POSIX, but it surprised many people during the ext4 transition; see the "delayed allocation and data loss" discussions of 2009.)

**(c) There are three page states, not two.** A folio is clean, **dirty**, or **under writeback**, and the last two are different:

| State | Tag | Meaning |
|---|---|---|
| clean | none | matches disk; reclaimable immediately |
| dirty | `PAGECACHE_TAG_DIRTY` | modified, not yet submitted |
| writeback | `PAGECACHE_TAG_WRITEBACK` | submitted to the block layer, not yet completed |

A folio under writeback is *neither* dirty nor clean: it cannot be reclaimed (I/O is in flight against it) and it must not be modified without care. Writing to a folio under writeback either waits (`folio_wait_writeback()`) or requires **stable pages** — a filesystem or device that computes checksums or does integrity protection needs the data to stop changing during the I/O, hence `mapping_stable_writes()` / `FOLIO_WAIT_STABLE`. This is a real source of latency spikes on such setups and a good example of an interface constraint (Ch. 51 §T.3) leaking upward.

### T.7 Writeback as a control system

Here is the crux of the chapter. Dirty pages are unreclaimable memory produced at an application-controlled rate and consumed at a device-controlled rate. If production exceeds consumption indefinitely, memory fills with dirty pages and the machine dies. Therefore the kernel must **throttle writers**, and the question is how.

The naive design — "block the writer when dirty memory exceeds a threshold" — produces exactly the pathology it is meant to avoid:

```
   dirty memory
       ▲
  20%  ┤                    ┌──────────  writers hard-blocked: huge
       │                   ╱              latency spikes, stalls, the
  10%  ┤        ┌─────────╱               "dirty_ratio cliff"
       │       ╱
       └───────┴────────────────────────► time
```

Everything runs at full speed until the threshold, then everything stops. Throughput oscillates, latency is bimodal, and the machine appears to hang for seconds. Anyone who has copied a large file to a slow USB stick on an older kernel has felt this.

Linux's answer (Wu Fengguang, 2011) is **proportional, per-task, feedback-controlled throttling**:

> Instead of blocking writers at a threshold, *continuously rate-limit each writer* so that the aggregate dirty-production rate converges to the aggregate writeback rate, and dirty memory converges to a setpoint.

The mechanism, in `mm/page-writeback.c`:

1. **Estimate the writeback bandwidth** per backing device (`wb->write_bandwidth`), as an EWMA of completed writeback over time.
2. **Compute a setpoint** between `dirty_background_ratio` and `dirty_ratio` — the target dirty level.
3. **Derive a position ratio** from how far current dirty is from the setpoint. Below the setpoint, allow more; above, allow less. The curve is cubic near the setpoint, so the response is gentle in the normal region and sharp near the limits.
4. **Compute a per-task rate limit** by dividing the device's bandwidth among the tasks dirtying it, scaled by the position ratio.
5. **Pause the task** in `balance_dirty_pages()` for exactly long enough to enforce that rate — typically 1–200 ms, in small increments rather than one long block.

The result is that a writer is slowed *smoothly and continuously* to the device's actual speed, and dirty memory sits near the setpoint rather than sawtoothing between 0 and the hard limit. The control law is a proportional controller with a derivative-ish term from the bandwidth estimate, and the kernel documentation for it (`Documentation/admin-guide/mm/`, plus Wu's design documents) reads like a controls paper — because it is one.

Three practical consequences:

**(a) `dirty_ratio` is a *limit*, not a target.** Reaching it means the controller failed to keep up; that is the emergency brake, not normal operation. If you are hitting it, the fix is elsewhere.

**(b) The percentage-based defaults are wrong on large machines.** `dirty_ratio = 20` on a 512 GiB machine permits 100 GiB of dirty data. On a device doing 200 MB/s, draining that takes 500 seconds, and a single `sync` or `fsync` can block for minutes. Hence `dirty_bytes` / `dirty_background_bytes`, which set absolute values and should be preferred on any machine with more than ~16 GiB:

```sh
# A reasonable starting point: a few seconds of device bandwidth
echo $((1024*1024*1024)) > /proc/sys/vm/dirty_bytes             # 1 GiB
echo $((256*1024*1024))  > /proc/sys/vm/dirty_background_bytes  # 256 MiB
```

Setting `dirty_bytes` zeroes `dirty_ratio` and vice versa — they are alternative expressions of one knob.

**(c) Per-device fairness required per-device state.** The original global accounting meant a slow USB stick's dirty pages counted against the same global limit as a fast NVMe's, so writing to the USB stick throttled writes to the NVMe. This is the historical "USB stick freezes my desktop" bug. The fix was **per-`bdi` (backing device info) dirty thresholds** with proportional allocation based on each device's observed bandwidth, plus per-cgroup writeback (§T.8). A slow device now gets a small share of the dirty budget and cannot monopolise it.

The knobs, with what each actually controls:

| Knob | Default | Meaning |
|---|---|---|
| `dirty_background_ratio` / `_bytes` | 10 % | start *background* writeback (no throttling) |
| `dirty_ratio` / `dirty_bytes` | 20 % | hard limit: writers are throttled hard |
| `dirty_expire_centisecs` | 3000 (30 s) | age at which a dirty page must be written |
| `dirty_writeback_centisecs` | 500 (5 s) | how often the flusher wakes to check |
| `dirtytime_expire_seconds` | 43200 (12 h) | lazytime: how long to defer pure timestamp updates |
| `vm.laptop_mode` | 0 | batch writeback to let disks spin down |

### T.8 Who does the writing, and for whom

Writeback runs in per-bdi (per-device) workqueues, not in the writing task:

```c
struct bdi_writeback {
	struct backing_dev_info	*bdi;
	unsigned long		state;
	struct list_head	b_dirty, b_io, b_more_io, b_dirty_time;
	unsigned long		write_bandwidth, avg_write_bandwidth;
	struct delayed_work	dwork;
	struct percpu_counter	stat[NR_WB_STAT_ITEMS];
	struct cgroup_subsys_state *memcg_css, *blkcg_css;   /* cgroup writeback */
	struct list_head	bdi_node;
};
```

The list structure (`b_dirty` → `b_io` → `b_more_io`) implements a simple queue discipline: inodes move from `b_dirty` (dirtied, waiting) to `b_io` (this writeback pass will handle them) to `b_more_io` (had more to write than the pass allowed; retry next round). It is a round-robin over inodes that prevents one enormous file from starving others.

Writeback is triggered by five distinct events, and knowing which one fired is most of writeback debugging:

| Trigger | Reason | `wb_reason` |
|---|---|---|
| Periodic timer | pages older than `dirty_expire_centisecs` | `WB_REASON_PERIODIC` |
| Background threshold | dirty exceeded `dirty_background_*` | `WB_REASON_BACKGROUND` |
| `balance_dirty_pages` | a writer hit the throttle | `WB_REASON_*` |
| `sync`/`fsync`/`syncfs` | explicit request | `WB_REASON_SYNC` |
| Memory reclaim | need this page | `WB_REASON_VMSCAN` |

The last one is a red flag: reclaim initiating writeback means the system is short of memory *and* the pages it wants are dirty, which is the worst combination. It shows up as `nr_vmscan_write` in `/proc/vmstat` and as latency spikes everywhere.

**Cgroup writeback** (4.2+) closed a long-standing hole. Before it, memory cgroups could limit a container's page cache, and blkio cgroups could limit its device I/O, but *buffered* writes escaped both: the dirty pages were charged to the cgroup, but the writeback I/O was issued by a kernel flusher thread belonging to nobody. So a container could exceed its I/O limit arbitrarily by writing buffered. The fix required plumbing the cgroup identity from the dirtying task, through the folio (`folio_memcg`), into the `bdi_writeback` (one per cgroup per bdi), and onto the bio (`bio_associate_blkg`). It is a good example of how a cross-cutting attribution requirement forces changes in five subsystems — and of why "who is responsible for this I/O" is a hard question when work is deferred.

### T.9 The honest limits of caching

Four cases where the page cache does not help, and knowing them prevents wasted effort:

**(a) Working set larger than memory.** If you randomly read a 10 TB dataset with 64 GB of RAM, the hit rate is 0.6 % and the cache is pure overhead. `O_DIRECT` plus application-managed caching is better, because the application knows its access pattern and the kernel does not. This is why every serious database has its own buffer pool and uses `O_DIRECT`.

**(b) Streaming data read once.** A backup, a video transcode, a `grep -r` over a huge tree. The cache fills with data that will never be read again and evicts data that would have been. Mitigations: `POSIX_FADV_DONTNEED` after reading, `POSIX_FADV_NOREUSE`, or `O_DIRECT`. Linux's active/inactive list separation (Ch. 23 §T.3) makes this less catastrophic than pure LRU would, because a once-read page never gets promoted to the active list — but it still costs.

**(c) Double caching.** A VM whose guest caches file data, running on a host whose page cache caches the same data, running on an SSD whose DRAM caches it again. Each level pays memory and none knows about the others. The fix is `cache=none` (O_DIRECT) on the host, so the guest's cache is the only one.

**(d) Data that must be durable immediately.** A write-ahead log is written, `fsync`ed, and (in the common case) never read. Caching it is pure overhead. Databases use `O_DIRECT|O_DSYNC` for exactly this.

The general principle:

> **The page cache is a heuristic that exploits locality. When the application knows its access pattern better than the heuristic can infer it, the application should say so (`fadvise`) or opt out (`O_DIRECT`).** Fighting the cache with tuning knobs is a distant third option.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `mm/filemap.c` | **the core**: `filemap_read`, `generic_perform_write`, `filemap_fault`, `filemap_get_folio`, `__filemap_add_folio` |
| `mm/readahead.c` | `page_cache_sync_ra`, `page_cache_async_ra`, `ondemand_readahead`, `page_cache_ra_order` |
| `mm/page-writeback.c` | `balance_dirty_pages`, `wb_update_bandwidth`, `domain_dirty_limits`, `folio_mark_dirty` |
| `fs/fs-writeback.c` | `wb_workfn`, `writeback_sb_inodes`, `__writeback_single_inode`, `sync_inodes_sb` |
| `mm/backing-dev.c` | `bdi_init`, per-cgroup wb creation |
| `mm/truncate.c` | `invalidate_mapping_pages`, `truncate_inode_pages_range` |
| `mm/folio-compat.c` | the shrinking compatibility shims from the folio conversion |
| `include/linux/pagemap.h` | the folio/page-cache API |
| `include/linux/mm_types.h` | `struct folio` |
| `include/linux/writeback.h` | `struct writeback_control` |
| `include/linux/backing-dev-defs.h` | `struct bdi_writeback` |
| `include/trace/events/writeback.h` | the tracepoints Lab 52.5 uses |

### 1.2 `struct writeback_control` — the request

```c
struct writeback_control {
	long nr_to_write;              /* budget: pages remaining this pass */
	long pages_skipped;
	loff_t range_start, range_end; /* for fsync of a range */

	enum writeback_sync_modes sync_mode;  /* WB_SYNC_NONE | WB_SYNC_ALL */

	unsigned for_kupdate:1;        /* periodic */
	unsigned for_background:1;     /* threshold-driven */
	unsigned tagged_writepages:1;  /* use TOWRITE tag (see below) */
	unsigned for_reclaim:1;        /* memory pressure */
	unsigned range_cyclic:1;       /* continue from writeback_index */
	unsigned for_sync:1;
	unsigned unpinned_netfs_wb:1;
	unsigned no_cgroup_owner:1;

	struct swap_iocb **swap_plug;
	struct list_head *list;
	struct folio_batch fbatch;
	pgoff_t index;
	int saved_err;
	struct bdi_writeback *wb;
	struct inode *inode;
};
```

`WB_SYNC_NONE` vs `WB_SYNC_ALL` is the key distinction: `NONE` means "write what you conveniently can, skip anything locked or under I/O"; `ALL` means "write everything in range and wait," which is what `fsync` needs.

`tagged_writepages` and the `PAGECACHE_TAG_TOWRITE` tag solve the **livelock** problem: if `fsync()` walks the dirty list while another thread keeps dirtying pages, it could never finish. The fix is to first *tag* the set of pages dirty at the start (`tag_pages_for_writeback()`), then write only the tagged set. New dirty pages get no tag and are not this `fsync`'s problem. That is a snapshot taken with a tree-tag pass — cheap, thanks to §T.3's tag propagation.

### 1.3 The read path

```
read(fd, buf, n)
 └─ vfs_read → file->f_op->read_iter  (usually generic_file_read_iter)
     └─ filemap_read()                        [mm/filemap.c]
         └─ filemap_get_pages()
             ├─ filemap_get_read_batch()      /* lock-free XArray lookup */
             ├─ MISS: page_cache_sync_readahead()
             │      └─ ondemand_readahead() → a_ops->readahead()
             │           └─ (fs) iomap_readahead / mpage_readahead
             │                └─ submit_bio()   ... wait ...
             ├─ folio_test_readahead() → page_cache_async_readahead()
             │      /* the PG_readahead tripwire of T.5 */
             └─ filemap_update_page() if !uptodate
         └─ copy_folio_to_iter()               /* the memcpy to userspace */
```

Note `filemap_get_read_batch()`: the fast path grabs a *batch* of folios in one RCU-protected XArray walk, rather than one lookup per page. On a large read with large folios this is one walk for the whole request.

### 1.4 The write and writeback paths

```
write(fd, buf, n)
 └─ generic_perform_write()
     ├─ a_ops->write_begin()            /* iomap_write_begin / block_write_begin */
     ├─ copy_folio_from_iter_atomic()
     ├─ a_ops->write_end()
     │    └─ filemap_dirty_folio()
     │         ├─ xas_set_mark(PAGECACHE_TAG_DIRTY)
     │         ├─ inc NR_FILE_DIRTY, wb_stat(WB_RECLAIMABLE)
     │         └─ inode_attach_wb(); wb_wakeup_delayed()
     └─ balance_dirty_pages_ratelimited()     /* T.7's controller */

--- asynchronously ---

wb_workfn(work)                                [fs/fs-writeback.c]
 └─ wb_do_writeback → wb_writeback(work)
     └─ writeback_sb_inodes()
         └─ __writeback_single_inode()
             └─ do_writepages() → a_ops->writepages()
                 └─ iomap_writepages()          /* modern fs */
                     └─ write_cache_pages()
                         ├─ tag_pages_for_writeback()   /* if tagged */
                         ├─ for each DIRTY folio in range:
                         │    folio_lock(); folio_clear_dirty_for_io();
                         │    folio_start_writeback();
                         │    map offsets->blocks; add to bio
                         └─ submit_bio()
                              ... completion ...
                              └─ folio_end_writeback()
                                   └─ clear TAG_WRITEBACK, wake waiters
```

`folio_clear_dirty_for_io()` before `folio_start_writeback()` is an ordering that matters: if a write lands between them, the folio is re-dirtied and will be written again, which is correct. The reverse order would lose the second write.

### 1.5 The observability surface

```sh
/proc/meminfo:
	Cached:          # page cache (excluding swap cache and buffers)
	Buffers:         # block-device page cache (metadata, mostly)
	Dirty:           # dirty, not yet submitted
	Writeback:       # submitted, in flight
	WritebackTmp:    # FUSE temporary
	NFS_Unstable:    # written to server, not committed
	Mapped:          # page cache mapped into page tables
	Shmem:           # tmpfs/shared, NOT reclaimable without swap
	MemAvailable:    # the honest "free memory" number

/proc/vmstat:
	nr_file_pages, nr_dirty, nr_writeback, nr_dirtied, nr_written
	nr_vmscan_write, nr_vmscan_immediate_reclaim   # reclaim doing writeback: bad
	pgpgin, pgpgout                                # block I/O in KB
	workingset_refault_file                        # cache thrash (Ch. 23)

/proc/sys/vm/dirty_*                               # the knobs of T.7
/sys/class/bdi/<major>:<minor>/                    # per-device: min_ratio,
                                                   # max_ratio, read_ahead_kb,
                                                   # stable_pages_required
/sys/block/*/queue/read_ahead_kb
/sys/kernel/debug/bdi/<dev>/stats                  # live wb state and bandwidth
/proc/pressure/io                                  # PSI: the metric to alert on
```

---

## 2. Practice

### Lab 52.1 — See the cache

```sh
# Baseline
free -h
grep -E '^(MemTotal|MemFree|MemAvailable|Cached|Buffers|Dirty|Writeback):' /proc/meminfo

# Create a file and drop caches
dd if=/dev/urandom of=/tmp/cachetest bs=1M count=512
sync
echo 3 | sudo tee /proc/sys/vm/drop_caches      # 1=pagecache 2=slab 3=both

# Cold read
grep ^Cached: /proc/meminfo
time cat /tmp/cachetest > /dev/null
grep ^Cached: /proc/meminfo                     # up by ~512 MB

# Warm read
time cat /tmp/cachetest > /dev/null             # 10-100x faster
```

Now see *which* pages of which file are cached:

```sh
sudo apt install vmtouch
vmtouch -v /tmp/cachetest
vmtouch /usr/bin/*            # how much of your binaries are resident
vmtouch -e /tmp/cachetest     # evict just this file
vmtouch -t /tmp/cachetest     # touch (load) it
vmtouch -l /tmp/cachetest     # lock it in memory (mlock)
```

Or do it yourself with `mincore()`:

```c
// SPDX-License-Identifier: GPL-2.0
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>

int main(int argc, char **argv)
{
	struct stat st;
	int fd = open(argv[1], O_RDONLY);
	size_t pages, i, resident = 0;
	unsigned char *vec;
	void *map;

	fstat(fd, &st);
	if (!st.st_size) return 0;

	map = mmap(NULL, st.st_size, PROT_READ, MAP_SHARED, fd, 0);
	pages = (st.st_size + 4095) / 4096;
	vec = malloc(pages);

	if (mincore(map, st.st_size, vec)) { perror("mincore"); return 1; }

	for (i = 0; i < pages; i++)
		if (vec[i] & 1) resident++;

	printf("%s: %zu/%zu pages resident (%.1f%%)\n",
	       argv[1], resident, pages, 100.0 * resident / pages);

	/* Visualise the first 256 pages */
	for (i = 0; i < pages && i < 256; i++)
		putchar((vec[i] & 1) ? '#' : '.');
	putchar('\n');

	munmap(map, st.st_size);
	free(vec);
	close(fd);
	return 0;
}
```

```sh
gcc -o cachemap cachemap.c
echo 3 | sudo tee /proc/sys/vm/drop_caches
./cachemap /tmp/cachetest            # 0% resident
head -c 4096 /tmp/cachetest > /dev/null
./cachemap /tmp/cachetest            # watch readahead: MORE than 1 page!
```

That last step is the important one: reading 4 KiB brings in ~128 KiB, and the visualisation shows exactly the readahead window of §T.5.

---

### Lab 52.2 — Measure readahead, then defeat it

```sh
IF=$(findmnt -no SOURCE / | sed 's|/dev/||;s|p\?[0-9]*$||')
cat /sys/block/$IF/queue/read_ahead_kb
```

Measure sequential read throughput as a function of readahead:

```sh
dd if=/dev/urandom of=/tmp/ratest bs=1M count=2048
sync

for ra in 0 16 128 512 2048 8192; do
  echo $ra | sudo tee /sys/block/$IF/queue/read_ahead_kb > /dev/null
  echo 3 | sudo tee /proc/sys/vm/drop_caches > /dev/null
  printf "ra=%-6s " $ra
  dd if=/tmp/ratest of=/dev/null bs=4k 2>&1 | tail -1
done
echo 128 | sudo tee /sys/block/$IF/queue/read_ahead_kb > /dev/null
```

`ra=0` gives you the un-prefetched device latency multiplied by the number of 4 K reads. The jump from 0 to 128 is the entire value of readahead, and on an HDD it is 50–100×.

Now show that readahead correctly *stops* for random access:

```c
// SPDX-License-Identifier: GPL-2.0
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <unistd.h>

static double now(void)
{
	struct timespec ts;
	clock_gettime(CLOCK_MONOTONIC, &ts);
	return ts.tv_sec + ts.tv_nsec/1e9;
}

int main(int argc, char **argv)
{
	int fd = open(argv[1], O_RDONLY);
	long size = lseek(fd, 0, SEEK_END);
	char buf[4096];
	double t0;
	int i, n = 2000;

	/* Sequential */
	posix_fadvise(fd, 0, 0, POSIX_FADV_DONTNEED);
	t0 = now();
	for (i = 0; i < n; i++)
		pread(fd, buf, 4096, (long)i * 4096);
	printf("sequential        : %7.1f us/read\n", (now()-t0)/n*1e6);

	/* Random */
	posix_fadvise(fd, 0, 0, POSIX_FADV_DONTNEED);
	t0 = now();
	for (i = 0; i < n; i++)
		pread(fd, buf, 4096, (random() % (size/4096)) * 4096L);
	printf("random            : %7.1f us/read\n", (now()-t0)/n*1e6);

	/* Random, with the hint */
	posix_fadvise(fd, 0, 0, POSIX_FADV_DONTNEED);
	posix_fadvise(fd, 0, 0, POSIX_FADV_RANDOM);
	t0 = now();
	for (i = 0; i < n; i++)
		pread(fd, buf, 4096, (random() % (size/4096)) * 4096L);
	printf("random +FADV_RANDOM: %6.1f us/read\n", (now()-t0)/n*1e6);

	/* Sequential with WILLNEED prefetch of the whole file */
	posix_fadvise(fd, 0, 0, POSIX_FADV_DONTNEED);
	posix_fadvise(fd, 0, 0, POSIX_FADV_WILLNEED);
	t0 = now();
	for (i = 0; i < n; i++)
		pread(fd, buf, 4096, (long)i * 4096);
	printf("seq +FADV_WILLNEED : %6.1f us/read\n", (now()-t0)/n*1e6);

	close(fd);
	return 0;
}
```

Then watch readahead decisions live:

```sh
sudo bpftrace -e '
kprobe:ondemand_readahead { @calls = count(); }
kprobe:page_cache_ra_order { @ra_order[arg2] = count(); }
interval:s:2 { print(@calls); print(@ra_order); clear(@calls); clear(@ra_order); }'
```

`@ra_order` shows the folio orders being requested — direct observation of §T.4's large-folio sizing driven by §T.5's window.

---

### Lab 52.3 — Be a good citizen: `POSIX_FADV_DONTNEED`

Demonstrate cache pollution and its fix.

```sh
# 1. Warm the cache with something you care about
dd if=/dev/urandom of=/tmp/important bs=1M count=1024
cat /tmp/important > /dev/null
vmtouch /tmp/important                    # ~100% resident

# 2. Simulate a backup that reads a lot
dd if=/dev/urandom of=/tmp/bulk bs=1M count=8192
cat /tmp/bulk > /dev/null                 # or: tar cf /dev/null /usr

# 3. Check the damage
vmtouch /tmp/important                    # evicted
```

Now the polite version:

```c
// SPDX-License-Identifier: GPL-2.0
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

#define CHUNK (1 << 20)

int main(int argc, char **argv)
{
	int fd = open(argv[1], O_RDONLY);
	char buf[CHUNK];
	off_t off = 0;
	ssize_t n;

	posix_fadvise(fd, 0, 0, POSIX_FADV_SEQUENTIAL);

	while ((n = read(fd, buf, CHUNK)) > 0) {
		/* ... process buf ... */
		off += n;
		/* Drop what we just read, keeping one chunk of slack so we
		 * do not fight readahead. */
		if (off > 2 * CHUNK)
			posix_fadvise(fd, 0, off - CHUNK, POSIX_FADV_DONTNEED);
	}
	posix_fadvise(fd, 0, 0, POSIX_FADV_DONTNEED);
	close(fd);
	return 0;
}
```

Re-run step 3 with this version and confirm `/tmp/important` survives. Then check that real tools do this:

```sh
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_fadvise64 {
	printf("%-16s advice=%d len=%llu\n", comm, args->advice, args->len); }'
# In another terminal:
rsync -a /usr/share/doc /tmp/rsynctest/     # rsync does NOT fadvise by default
nocache cp /tmp/bulk /tmp/bulk2            # `nocache` wraps with DONTNEED
dd if=/tmp/bulk of=/dev/null bs=1M iflag=nocache
```

`dd`'s `iflag=nocache`/`oflag=nocache` is the easiest way to get this behaviour from a shell, and knowing it exists saves real incidents.

---

### Lab 52.4 — Watch the writeback controller work

**(a) See the `dirty_ratio` behaviour.**

```sh
# Watch dirty memory while writing faster than the device can absorb
watch -n0.2 "grep -E '^(Dirty|Writeback):' /proc/meminfo; \
             echo; grep -E 'nr_dirtied|nr_written' /proc/vmstat"

# In another terminal, to a SLOW device (USB stick, or throttled loop dev):
dd if=/dev/zero of=/media/usb/big bs=1M count=4096
```

Observe: `Dirty` rises, plateaus near the setpoint (not at `dirty_ratio`), and `dd`'s reported throughput converges to the device's real speed. That plateau *is* the controller of §T.7 working.

**(b) Make the cliff appear.** Recreate the old behaviour by setting the background threshold equal to the hard limit:

```sh
# Save current values first!
sysctl vm.dirty_ratio vm.dirty_background_ratio

echo 90 | sudo tee /proc/sys/vm/dirty_ratio
echo 89 | sudo tee /proc/sys/vm/dirty_background_ratio
# Now write a lot; observe long stalls and bursty throughput
dd if=/dev/zero of=/tmp/big bs=1M count=8192 status=progress

# Restore
echo 20 | sudo tee /proc/sys/vm/dirty_ratio
echo 10 | sudo tee /proc/sys/vm/dirty_background_ratio
```

**(c) Use absolute limits on a big machine.**

```sh
free -g
echo $((512*1024*1024)) | sudo tee /proc/sys/vm/dirty_bytes
echo $((128*1024*1024)) | sudo tee /proc/sys/vm/dirty_background_bytes
cat /proc/sys/vm/dirty_ratio            # now 0: they are mutually exclusive
```

Measure `sync` latency before and after with a percentage-based limit on a machine with lots of RAM:

```sh
dd if=/dev/zero of=/tmp/big bs=1M count=4096
time sync
```

The difference is the bounded-drain-time argument of §T.7(b), measured.

**(d) Per-device isolation.** Write to a fast and a slow device simultaneously and show the fast one is not throttled by the slow one:

```sh
# Create a deliberately slow device with dm-delay
SZ=$(blockdev --getsz /dev/loop0)
sudo dmsetup create slow --table "0 $SZ delay /dev/loop0 0 50"   # 50ms delay
sudo mkfs.ext4 /dev/mapper/slow && sudo mount /dev/mapper/slow /mnt/slow

# Run both, measure each
( dd if=/dev/zero of=/mnt/slow/x bs=1M count=100 ; echo "slow done" ) &
( dd if=/dev/zero of=/tmp/y     bs=1M count=4096 ; echo "fast done" ) &
wait
cat /sys/kernel/debug/bdi/*/stats
```

---

### Lab 52.5 — Trace writeback end to end

```sh
sudo trace-cmd record -e writeback -e filemap -e block:block_rq_issue \
                      -e vmscan:mm_vmscan_writepage
# do some I/O
sudo trace-cmd report | head -60
```

The `writeback` tracepoint family is unusually good:

```sh
sudo bpftrace -e '
tracepoint:writeback:writeback_dirty_folio  { @dirtied = count(); }
tracepoint:writeback:writeback_queue        { @queued[str(args->name)] = count(); }
tracepoint:writeback:writeback_start        { @started[args->reason] = count(); }
tracepoint:writeback:writeback_written      { @written = sum(args->nr_pages); }
tracepoint:writeback:balance_dirty_pages    { @paused = hist(args->pause); }
interval:s:5 { print(@dirtied); print(@queued); print(@started);
               print(@written); print(@paused);
               clear(@dirtied); clear(@written); }'
```

`@started[reason]` maps directly to §T.8's trigger table — you can see whether writeback is periodic, background, sync, or (alarmingly) reclaim-driven.

`@paused` is the histogram of how long `balance_dirty_pages()` slept each time. In a healthy system these are short (1–20 ms) and frequent. Long pauses mean the controller is fighting, and the distribution's shape tells you whether it is a steady overload or a burst.

Also:

```sh
# Which files are being written back?
sudo bpftrace -e '
tracepoint:writeback:writeback_single_inode_start {
	printf("%s ino=%lu pages=%lu\n", str(args->name), args->ino,
	       args->nr_to_write); }'

# Is reclaim doing writeback?  (It should not be.)
grep -E 'nr_vmscan_write|nr_vmscan_immediate_reclaim' /proc/vmstat
sudo bpftrace -e 'tracepoint:vmscan:mm_vmscan_writepage { @ = count(); }'

# Live bdi state, including the bandwidth estimate
sudo cat /sys/kernel/debug/bdi/*/stats
```

The bdi stats file shows `BdiWriteback`, `BdiReclaimable`, `BdiDirtyThresh`, `DirtyThresh`, `BackgroundThresh`, `BdiWritten`, and `BdiWriteBandwidth`. `BdiWriteBandwidth` is the controller's estimate from §T.7(1) — compare it to `iostat`'s measured throughput and confirm the estimator is tracking.

---

### Lab 52.6 — Observe folios

```sh
# Is your kernel using large folios for the page cache?
grep -E 'CONFIG_TRANSPARENT_HUGEPAGE|CONFIG_READ_ONLY_THP' /boot/config-$(uname -r)
uname -r      # large folios in the page cache need 5.18+, more in 6.x

# Per-filesystem: which mappings allow large folios?
sudo bpftrace -e '
kretprobe:filemap_alloc_folio { @order = hist(retval != 0); }
kprobe:__filemap_add_folio { @add = count(); }'
```

Measure the effect directly. Compare a filesystem with large-folio support (XFS, and ext4 from 6.x) against one without, on a large sequential read:

```sh
# Set up two filesystems on loop devices
truncate -s 4G /tmp/xfs.img /tmp/ext4.img
sudo losetup -f --show /tmp/xfs.img       # -> /dev/loopN
sudo mkfs.xfs /dev/loopN && sudo mount /dev/loopN /mnt/xfs
# same for ext4

for m in /mnt/xfs /mnt/ext4; do
  sudo dd if=/dev/zero of=$m/f bs=1M count=2048
  sync; echo 3 | sudo tee /proc/sys/vm/drop_caches >/dev/null
  printf "%-10s " $m
  sudo dd if=$m/f of=/dev/null bs=1M 2>&1 | tail -1
done
```

Then count the bio segments, which is where the large-folio win shows up most clearly:

```sh
sudo bpftrace -e '
tracepoint:block:block_rq_issue { @sectors = hist(args->nr_sector); @n = count(); }'
# run the reads above; compare the histograms
```

Fewer, larger requests for the same bytes is the folio effect.

Also observe the page-struct overhead argument:

```sh
# struct page array size
grep -E 'MemTotal' /proc/meminfo
python3 -c "
total_kb = $(grep MemTotal /proc/meminfo | awk '{print $2}')
pages = total_kb * 1024 // 4096
print(f'{pages} pages x 64 B = {pages*64/1024/1024:.1f} MiB of struct page')
print(f'= {pages*64/(total_kb*1024)*100:.4f}% of RAM')"
```

---

### Lab 52.7 — Decide when the cache is wrong

Build the comparison that tells you whether to use the page cache at all.

```sh
# A file much larger than RAM, accessed randomly
free -g
SIZE=$(( $(free -g | awk '/^Mem:/{print $2}') * 2 ))
fallocate -l ${SIZE}G /tmp/huge

# Buffered random reads
fio --name=buffered --filename=/tmp/huge --rw=randread --bs=4k \
    --size=${SIZE}G --runtime=30 --time_based --iodepth=1 --numjobs=1 \
    --group_reporting

# Same, O_DIRECT
fio --name=direct --filename=/tmp/huge --rw=randread --bs=4k \
    --size=${SIZE}G --runtime=30 --time_based --iodepth=1 --numjobs=1 \
    --direct=1 --group_reporting

# Now with queue depth, where O_DIRECT + async really wins
fio --name=direct-qd --filename=/tmp/huge --rw=randread --bs=4k \
    --size=${SIZE}G --runtime=30 --time_based --iodepth=32 --numjobs=4 \
    --direct=1 --ioengine=io_uring --group_reporting
```

Record IOPS, latency percentiles, and CPU for each. Then monitor cache effectiveness during the buffered run:

```sh
sudo cachestat-bpfcc 1        # HITS, MISSES, hit ratio
grep workingset /proc/vmstat  # refaults: pages evicted then needed again
```

A high `workingset_refault_file` with a low hit ratio is the quantitative signature of §T.9(a): the cache is thrashing and costing you. Write down the hit-ratio threshold below which `O_DIRECT` wins on your hardware.

Then the streaming case:

```sh
# Measure cache damage from a streaming read
vmtouch -v /usr/lib > /tmp/before.txt
cat /tmp/huge > /dev/null
vmtouch -v /usr/lib > /tmp/after.txt
diff /tmp/before.txt /tmp/after.txt

# Repeat with nocache
vmtouch -v /usr/lib > /tmp/before.txt
dd if=/tmp/huge of=/dev/null bs=1M iflag=nocache
vmtouch -v /usr/lib > /tmp/after.txt
diff /tmp/before.txt /tmp/after.txt
```

---

## 3. Mastery drills

1. Prove that the XArray's tag-propagation invariant (an interior node is tagged iff any descendant is) makes `tag_pages_for_writeback()` over a range cost O(k log n) for k tagged pages, and construct the worst case.

2. `folio_clear_dirty_for_io()` runs before `folio_start_writeback()`. Show that reversing them loses a concurrent write, and identify the exact interleaving.

3. Derive the steady-state dirty-page level under the §T.7 controller, given a writer producing at rate P and a device draining at rate D, for the cases P < D, P = D, and P > D. What does the position-ratio curve's cubic shape buy over a linear one?

4. `dirty_ratio = 20` on a 512 GiB machine with a 200 MB/s device. Compute the worst-case `sync()` latency, then derive the `dirty_bytes` value that bounds it to 5 seconds.

5. Readahead's payoff is asymmetric. Formalise the expected-cost model (probability of use p, saved latency L, wasted bandwidth cost W) and derive the optimal window size. Compare to Linux's doubling heuristic.

6. The `PG_readahead` tripwire converts prefetch timing into a demand signal. Construct an access pattern that defeats it (triggers readahead that is never used) and explain what the kernel does about it.

7. Stable pages require a folio under writeback to be unmodifiable. Enumerate everything that needs this (device integrity, filesystem checksums, RAID parity, compression, encryption) and design an alternative that does not stall writers.

8. Cgroup writeback required attributing deferred work to its originator. List every place the cgroup identity had to be plumbed, and explain why the "charge at dirty time" and "issue at writeback time" split makes this hard.

9. Delayed allocation improves contiguity but moves `ENOSPC` later. Construct the sequence where an application's `write()` succeeds and its data is nonetheless lost, and state what the application must do to detect it.

10. Compute the total memory cost of a 1 TiB page cache in `struct page` overhead at order-0, and again with all order-4 folios. Then explain why the saving is not simply 16×.

11. Double caching (guest + host + device DRAM) wastes memory at three levels. Design a mechanism by which a guest could inform the host that a page is already cached, and explain why this has not been done.

12. `MemAvailable` estimates reclaimable memory. Read `si_mem_available()` and list every term; identify the one most likely to be wrong and construct a workload where it is.

13. Trace one 4 KiB buffered write through every lock: which are taken, in what order, and which could contend under a many-threaded workload writing to one file? Compare to the same write on separate files.

---

## 4. Further reading

**Kernel documentation**

- `Documentation/core-api/xarray.rst` ★★★ — the data structure of §T.3, including tags.
- `Documentation/filesystems/vfs.rst` ★★★ — the `address_space_operations` contract; every method documented.
- `Documentation/filesystems/porting.rst` ★★ — has the running record of the folio conversion, which doubles as a list of the sharp edges.
- `Documentation/admin-guide/sysctl/vm.rst` ★★★ — every `dirty_*` knob; the authoritative text for §T.7.
- `Documentation/admin-guide/mm/` — `concepts.rst`, `transhuge.rst`.
- `Documentation/core-api/mm-api.rst` — the folio API reference.
- `Documentation/filesystems/locking.rst` ★★★ — which lock is held when each `a_ops` method is called. Essential when writing a filesystem (Ch. 56).

**Papers**

- J. Ousterhout, "Why Aren't Operating Systems Getting Faster As Fast as Hardware?", USENIX 1990 — the original observation about memory/disk gaps.
- P. Cao, E. W. Felten, A. R. Karlin, K. Li, "A Study of Integrated Prefetching and Caching Strategies," SIGMETRICS 1995 ★★★ — the theory behind §T.5: prefetching and caching interact and should be decided together.
- A. Tomkins, R. H. Patterson, G. Gibson, "Informed Multi-Process Prefetching and Caching," SIGMETRICS 1997 — the case for `fadvise`-style hints.
- E. J. O'Neil, P. E. O'Neil, G. Weikum, "The LRU-K Page Replacement Algorithm," SIGMOD 1993 and T. Johnson, D. Shasha, "2Q," VLDB 1994 — the scan-resistance results behind Linux's active/inactive split (Ch. 23 §T.3).
- S. Jiang, X. Zhang, "LIRS," SIGMETRICS 2002 — reuse-distance-based replacement; the intellectual ancestor of the refault-distance work.
- M. Wilcox's folio design documents and the LSFMM 2021 proposal ★★★ — §T.4 in the author's words.
- Wu Fengguang, "IO-less Dirty Throttling," Linux Plumbers 2011, and the associated design documents ★★★ — the derivation of §T.7's controller, including the control-theory analysis. This is the primary source and it is excellent.

**LWN — essential for this chapter**

- "The page cache" series and "A folio for your thoughts" / "Clarifying memory management with page folios" (2021) ★★★
- "Large folios for the page cache" and the ongoing large-folio coverage (2021–2024) ★★★
- "Flushing out pdflush" (2009) and "Dirty page throttling" / "No-I/O dirty throttling" (2011) ★★★ — the history of §T.7
- "Writeback and control groups" (2015) — §T.8's cgroup work
- "Toward less-annoying background writeback" (2016) and "Writeback throttling" — WBT (Ch. 64)
- "The ext4 data-loss question" / "Delayed allocation and the zero-length file problem" (2009) — §T.6(b)'s controversy in full
- "Improving readahead" and "Readahead: the documentation I wanted to read" coverage
- "Fixing page-cache thrashing" / the workingset/refault series (2013–2014)

**Source reading order**

1. `mm/filemap.c`: `filemap_read()` → `filemap_get_pages()` → `filemap_create_folio()`. Then `generic_perform_write()`.
2. `mm/readahead.c`: all of it. It is ~800 lines and self-contained; `ondemand_readahead()` is the whole algorithm.
3. `mm/page-writeback.c`: `folio_mark_dirty()` and `balance_dirty_pages()`. The latter is dense; read it with Wu's design document open.
4. `fs/fs-writeback.c`: `wb_writeback()` → `writeback_sb_inodes()` → `__writeback_single_inode()`.
5. `mm/truncate.c`: short, and it shows the invalidation side of every invariant above.

**Tools**

- `vmtouch` ★★★ — the single most useful page-cache tool
- `mincore(2)`, `fincore(1)` (util-linux) — per-page residency
- `cachestat-bpfcc`, `cachetop-bpfcc`, `filetop-bpfcc`, `fileslower-bpfcc`
- `pcstat` — per-file cache statistics for a list of files
- `nocache` (the utility), `dd iflag=nocache`, `rsync --drop-cache` (some versions)
- `/proc/sys/vm/drop_caches` — for benchmarking only; never in production
- `sudo cat /sys/kernel/debug/bdi/*/stats` ★★★
- `trace-cmd record -e writeback` and the `writeback` tracepoint family ★★★
- `perf record -e major-faults,minor-faults` — page-cache misses as faults

---

→ Next: [53-vfs-1.md](53-vfs-1.md)

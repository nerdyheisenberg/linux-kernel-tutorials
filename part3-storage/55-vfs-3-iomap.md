# Chapter 55 — VFS III: `address_space` operations, iomap, and direct I/O

> **Goal:** Understand the layer where "a file" becomes "blocks on a device." Understand why `buffer_head` was the wrong abstraction and what replaced it, why `iomap` exists and what problem it solves that `->get_block()` could not, how a `read()` becomes a `bio`, what direct I/O actually bypasses and what it does *not*, why `O_DIRECT` has the alignment rules it has, and how extents, delayed allocation, unwritten extents, and copy-on-write all reduce to one question the filesystem must answer: *"what is at this file offset?"* By the end you can implement `->read_folio`, `->writepages`, and a full `iomap_ops`, and can explain every step from `read(2)` to a completed `bio`.

---

## Theory & First Principles

### T.0 — Start here: one question, asked a billion times

Strip every filesystem down to its essential job and this is what remains:

> **"Logical block N of this file lives at physical block M on the device — or nowhere yet."**

Everything else — directories, permissions, journals, extents — is bookkeeping in service of
answering that one question quickly and consistently.

**The old interface asked it one block at a time:**

```c
int (*get_block)(struct inode *inode, sector_t iblock,
                 struct buffer_head *bh, int create);
/*                ^ ONE block. 4 KiB. Per call. */
```

For a 1 GiB sequential read that is **262,144 calls**, each returning a `buffer_head` — a
~100-byte object allocated per block. Which means, per gigabyte:

```
  262,144 indirect calls
+ 262,144 buffer_head allocations  (~26 MB of metadata for 1 GB of data)
+ 262,144 opportunities to take a lock and touch a cacheline
```

The abstraction was designed when a "large" file was a few megabytes and a disk did 100 IOPS.
On a device doing a million IOPS it is the bottleneck itself. **This is Ch. 51 §T.0's theme
again: the layer is solving a problem the hardware no longer has, and the cost of the
abstraction now exceeds the cost of the I/O.**

**`iomap` asks the question in ranges instead:**

```c
int (*iomap_begin)(struct inode *inode, loff_t pos, loff_t length,
                   unsigned flags, struct iomap *iomap, struct iomap *srcmap);
/* "Where does the range [pos, pos+length) live?"
 *  Answer: one struct describing a CONTIGUOUS EXTENT, plus its type.  */

struct iomap {
	u64  addr;        /* device offset, or IOMAP_NULL_ADDR */
	loff_t offset;    /* file offset of this extent */
	u64  length;      /* how much this answer covers -- often MEGABYTES */
	u16  type;        /* MAPPED / HOLE / DELALLOC / UNWRITTEN / INLINE */
	...
};
```

One call can now describe a gigabyte. The 262,144 calls become **one**, and the
`buffer_head`s disappear entirely.

**The `type` field is the second, subtler improvement**, because it makes states explicit
that used to be implicit and bug-prone:

| Type | Meaning | Why it must be distinct |
|---|---|---|
| `MAPPED` | allocated, contains data | the normal case |
| `HOLE` | never written | a read returns zeros without touching the device |
| `DELALLOC` | dirty in memory, **not yet allocated on disk** | delayed allocation: decide *where* at writeback, when you know the size |
| `UNWRITTEN` | allocated but never written | preallocated (`fallocate`); reads must return zeros **without** exposing whatever was on disk before — a real security property |
| `INLINE` | the data is *in the inode* | tiny files cost no data block at all |

`UNWRITTEN` is worth dwelling on: `fallocate()` gives you the blocks immediately (so writes
never fail with `ENOSPC`) but the blocks may contain another user's deleted data. The
filesystem must therefore track "allocated but not yet written" and return zeros for reads of
that range. Conflate it with `MAPPED` and you have an information leak.

**The general lesson, which is the point of putting this chapter here:**

> When an interface's *per-call* cost dominates, the fix is not to make the call faster — it
> is to **change the granularity of the question** so that far fewer calls are needed.

That is the same move as `io_uring` batching syscalls (Ch. 76), NAPI polling instead of
interrupting (Ch. 46), virtio ringing a doorbell once per batch (Ch. 104), and `mmu_gather`
amortizing TLB shootdowns (Ch. 22). **Four unrelated subsystems, one idea.**

```bash
filefrag -v /path/to/file          # the extent map, as the filesystem sees it
xfs_bmap -vp /path/to/file         # XFS: with UNWRITTEN/DELALLOC states shown
cat /sys/kernel/debug/tracing/events/iomap/enable
sudo trace-cmd record -e 'iomap:*' -- dd if=/big of=/dev/null bs=1M count=100
```

---

### T.1 The one question

Everything in this chapter is a way of answering a single query:

> Given a file and a byte range, **where is that data, and what state is it in?**

The possible answers form a small alphabet:

| State | Meaning | On read | On write |
|---|---|---|---|
| **Mapped** | blocks are allocated and contain data | read them | write them (maybe CoW) |
| **Hole** | no blocks allocated; reads as zeros | return zeros, no I/O | allocate |
| **Unwritten** | blocks allocated but never written; reads as zeros | return zeros, no I/O | write, then convert to mapped |
| **Delalloc** | a write is buffered but blocks are not chosen yet | read from the page cache | already accounted |
| **Inline** | the data is inside the inode | copy from the inode | copy into the inode |

Five states. Every filesystem-specific complexity in this chapter — extent trees, delayed allocation, reflinks, preallocation — is a way of representing and transitioning between them.

The kernel's *first* answer to this question was `->get_block()`:

```c
typedef int (get_block_t)(struct inode *inode, sector_t iblock,
			  struct buffer_head *bh_result, int create);
```

"Tell me the device block for file block `iblock`." One block at a time. This is §T.2's problem.

### T.2 Why `buffer_head` was the wrong abstraction

`buffer_head` dates to the earliest Linux and encodes assumptions from 1991:

```c
struct buffer_head {
	unsigned long b_state;
	struct buffer_head *b_this_page;
	struct folio *b_folio;
	sector_t b_blocknr;             /* ONE device block */
	size_t b_size;
	char *b_data;
	struct block_device *b_bdev;
	bh_end_io_t *b_end_io;
	void *b_private;
	struct list_head b_assoc_buffers;
	struct address_space *b_assoc_map;
	atomic_t b_count;
	spinlock_t b_uptodate_lock;
};

/* On a 4 KiB page with 512-byte blocks: EIGHT of these per page. */
```

Four fatal problems:

**(a) One structure per block, not per range.** A 1 MiB contiguous write on a filesystem with 4 KiB blocks allocates 256 `buffer_head`s — 256 allocations, 256 `->get_block()` calls, 256 mapping lookups — to describe *one* contiguous extent. The structure cannot express "these 256 blocks are contiguous," even though that is the single most important fact about them.

**(b) It conflates three concerns.** A `buffer_head` is simultaneously: (i) a block-mapping cache entry, (ii) a per-block state bitmap within a page (uptodate/dirty per sub-page block), and (iii) an I/O submission unit. Three jobs, one structure, no way to use one without the others.

**(c) It is large and it is per-block.** ~104 bytes each. For a page with 512-byte blocks that is 832 bytes of metadata for 4096 bytes of data — 20 % overhead, all of it allocated and freed constantly.

**(d) It hardcodes `block size ≤ page size`.** The whole model assumes a page contains a whole number of blocks. Large folios (Ch. 52 §T.4) and block sizes larger than the page size break it.

The correct abstraction is the **extent**: `(file offset, length, device offset, state)`. One structure describing an arbitrarily large contiguous range. That is `struct iomap`.

### T.3 `iomap`: ask once, get a range

```c
struct iomap {
	u64			addr;      /* device offset, or IOMAP_NULL_ADDR */
	loff_t			offset;   /* file offset this mapping starts at */
	u64			length;   /* how much it covers */
	u16			type;     /* HOLE/DELALLOC/MAPPED/UNWRITTEN/INLINE */
	u16			flags;
	struct block_device	*bdev;
	struct dax_device	*dax_dev;
	void			*inline_data;
	void			*private;
	const struct iomap_folio_ops *folio_ops;
	u64			validity_cookie;
};
```

with types exactly matching §T.1's alphabet:

```c
#define IOMAP_HOLE      0
#define IOMAP_DELALLOC  1
#define IOMAP_MAPPED    2
#define IOMAP_UNWRITTEN 3
#define IOMAP_INLINE    4
```

and an operations vector with just two methods:

```c
struct iomap_ops {
	int (*iomap_begin)(struct inode *inode, loff_t pos, loff_t length,
			   unsigned flags, struct iomap *iomap,
			   struct iomap *srcmap);
	int (*iomap_end)(struct inode *inode, loff_t pos, loff_t length,
			 ssize_t written, unsigned flags, struct iomap *iomap);
};
```

Four structural improvements over `->get_block()`:

| | `->get_block()` | `iomap_begin()` |
|---|---|---|
| Granularity | one block | **arbitrary range** |
| Calls for 1 MiB contiguous | 256 | **1** |
| Can express "hole" | via a flag, awkwardly | a first-class type |
| Can express "unwritten" | poorly | a first-class type |
| Has a completion hook | no | `iomap_end()` |
| `srcmap` for CoW | impossible | yes |

The `srcmap` parameter is worth dwelling on: for a copy-on-write filesystem, a write needs *two* mappings — where to read the old data from (`srcmap`) and where to write the new data to (`iomap`). `->get_block()` had no way to express this, which is why btrfs and XFS reflink could not use the generic paths. `iomap` was designed with CoW as a first-class case.

`iomap_end()` closes the loop: "I asked for a mapping for 1 MiB, I actually wrote 700 KiB, here is what happened." That lets the filesystem punch back unused delayed-allocation reservations, convert unwritten extents, or roll back a transaction. `->get_block()` had no such hook, so filesystems had to reconstruct what happened from the page state — a persistent source of bugs.

### T.4 The `address_space_operations` contract

`address_space_operations` (Ch. 52 §1.2) is the interface between the page cache and the filesystem. The modern set:

```c
struct address_space_operations {
	int  (*writepage)(struct page *, struct writeback_control *);  /* gone */
	int  (*read_folio)(struct file *, struct folio *);
	int  (*writepages)(struct address_space *, struct writeback_control *);
	bool (*dirty_folio)(struct address_space *, struct folio *);
	void (*readahead)(struct readahead_control *);
	int  (*write_begin)(struct file *, struct address_space *, loff_t pos,
			    unsigned len, struct folio **, void **fsdata);
	int  (*write_end)(struct file *, struct address_space *, loff_t pos,
			  unsigned len, unsigned copied, struct folio *, void *);
	sector_t (*bmap)(struct address_space *, sector_t);
	void (*invalidate_folio)(struct folio *, size_t off, size_t len);
	bool (*release_folio)(struct folio *, gfp_t);
	void (*free_folio)(struct folio *);
	ssize_t (*direct_IO)(struct kiocb *, struct iov_iter *);
	int  (*migrate_folio)(struct address_space *, struct folio *dst,
			      struct folio *src, enum migrate_mode);
	int  (*launder_folio)(struct folio *);
	bool (*is_partially_uptodate)(struct folio *, size_t from, size_t count);
	int  (*error_remove_folio)(struct address_space *, struct folio *);
	int  (*swap_activate)(struct swap_info_struct *, struct file *, sector_t *);
	void (*swap_deactivate)(struct file *);
};
```

Three design rules govern this interface, and all three came from painful experience:

**Rule 1: `->writepage` is dead.** It wrote *one page*, which meant writeback could not batch and could not see contiguity. It was also called from reclaim context, which meant a filesystem could be asked to write a page while the caller held arbitrary locks and could not allocate — a recipe for deadlocks. It has been removed; **`->writepages` is the only write path.** Writing a new filesystem, never implement `->writepage`.

**Rule 2: `->readahead` replaced `->readpages`.** `->readpages` was handed a list of pages and had to handle each; `->readahead` is handed a `readahead_control` and *pulls* folios with `readahead_folio()`, which lets it stop early, batch arbitrarily, and handle large folios. The inversion of control matters: the filesystem now decides the batching.

**Rule 3: `folio`, not `page`.** Every method has been converted. A folio may be any order; the filesystem must not assume 4 KiB. This is Ch. 52 §T.4's type-system fix reaching the filesystem interface.

The `write_begin`/`write_end` pair is the buffered-write protocol:

```
write_begin(pos, len)  -> get a folio, ensure it is uptodate enough
                          to be partially overwritten, start a transaction
copy_from_user()       -> generic code copies the data
write_end(pos, len, copied) -> mark dirty, finish the transaction,
                          unlock and put the folio
```

The subtle part is "uptodate enough": if the write covers only part of the folio and the folio is not uptodate, the *rest* of the folio must be read from disk first, or the unwritten part would be exposed as garbage. This is the **read-modify-write** of Ch. 52 §T.6, and `write_begin` is where it happens. Getting it wrong is an information leak.

`iomap` provides `iomap_file_buffered_write()` which implements this whole protocol, so an iomap filesystem implements neither `write_begin` nor `write_end`.

### T.5 The buffered read path, end to end

```
read(fd, buf, n)
 └─ ksys_read -> vfs_read -> file->f_op->read_iter(kiocb, iter)
     └─ (typical) filemap_read(kiocb, iter, 0)
         ├─ filemap_get_pages()
         │   ├─ filemap_get_read_batch()      /* look in the XArray */
         │   ├─ MISS -> page_cache_sync_readahead()
         │   │            └─ ->readahead(rac)
         │   │                 └─ iomap_readahead()
         │   │                     ├─ iomap_begin()          /* ONE call */
         │   │                     └─ for each folio: build/extend a bio
         │   │                          └─ submit_bio()
         │   ├─ folio_wait_locked()           /* sleep until I/O completes */
         │   └─ if PG_readahead set -> page_cache_async_readahead()
         ├─ copy_folio_to_iter()              /* the actual copy */
         └─ repeat until n bytes or EOF
```

Two things deserve emphasis.

**The bio is built by extending.** `iomap_readahead` does not create one bio per folio; it tries to add each folio to the current bio with `bio_add_folio()`, and only submits and starts a new one when the current bio is full or the next folio is not device-contiguous. A 2 MiB readahead over a contiguous extent becomes **one bio** — which becomes one request, which becomes (given a capable device) one command. Ch. 63 picks this up.

**The mapping is fetched once per extent, not per folio.** This is §T.3's entire payoff, visible here.

The write path is the mirror image, with one crucial asymmetry: reads are synchronous with respect to the caller and writes are not. `write()` returns when the data is in the page cache. Everything after that is Ch. 52's writeback machinery, driven by `->writepages`:

```
wb_writeback -> __writeback_single_inode -> do_writepages
 └─ ->writepages(mapping, wbc)
     └─ iomap_writepages(mapping, wbc, &ctx, &ops)
         └─ write_cache_pages()          /* iterate DIRTY-tagged folios */
             └─ iomap_writepage_map()
                 ├─ ->map_blocks()        /* allocate if delalloc */
                 ├─ extend or submit the current ioend
                 └─ folio_start_writeback(); folio_unlock()
         └─ iomap_submit_ioend()
```

`struct iomap_ioend` is the batching unit: a list of contiguous folios sharing one bio and one completion. On completion, `iomap_finish_ioend()` runs, which is where unwritten extents get converted and the size is updated — work that must happen in process context, hence the workqueue.

### T.6 Direct I/O: what it actually bypasses

`O_DIRECT` is widely misunderstood. Precisely:

**What it does:** DMA directly between the device and the *user's* pages, with no copy through the page cache.

**What it does NOT do:**

| Misconception | Reality |
|---|---|
| "It makes writes durable" | **No.** The device may still have them in a volatile cache. You still need `fsync`/`fdatasync` or `O_DSYNC`. This is Ch. 51 §T.4's trap. |
| "It bypasses the filesystem" | No. Allocation, extent updates, and journalling all still happen. |
| "It is always faster" | No. It defeats readahead and write absorption. For anything but large aligned sequential I/O or an application with a better cache, it is usually slower. |
| "It bypasses the block layer" | No. It still builds bios and goes through the scheduler. |
| "It is coherent with buffered I/O" | **Only because the kernel works hard to make it so** — and the guarantees are weaker than you think (§T.7). |

The mechanism:

```
read(fd, buf, n) with O_DIRECT
 └─ ->read_iter -> (iocb->ki_flags & IOCB_DIRECT)
     └─ iomap_dio_rw(iocb, iter, ops, dops, flags, private, done_before)
         ├─ iomap_dio_iter() per extent:
         │   ├─ iomap_begin()
         │   ├─ HOLE/UNWRITTEN + read  -> iov_iter_zero(), NO I/O AT ALL
         │   └─ MAPPED -> iomap_dio_bio_iter()
         │        ├─ bio_iov_iter_get_pages()   /* PIN the user pages */
         │        └─ submit_bio()
         └─ wait (or return -EIOCBQUEUED for async)
```

`bio_iov_iter_get_pages()` is the heart: it pins the user's pages with `pin_user_pages_fast()` (FOLL_PIN, not the old `get_user_pages` — see the `struct page` refcount wars) so the pages cannot be migrated or freed while DMA is in flight, and builds a bio pointing at them directly.

This is why the alignment rules exist:

| Requirement | Why |
|---|---|
| File offset must be `logical_block_size`-aligned | the device addresses blocks, not bytes |
| Length must be a multiple of `logical_block_size` | same |
| **The user buffer address** must be `dma_alignment`-aligned | the DMA engine has alignment constraints (Ch. 35) |

Violating any of them gives `-EINVAL`, historically with no diagnostic whatsoever — one of the most frustrating errors in Linux. `STATX_DIOALIGN` (6.1+) finally lets you *query* the requirements:

```c
	struct statx stx;
	statx(AT_FDCWD, path, 0, STATX_DIOALIGN, &stx);
	/* stx.stx_dio_mem_align   -- buffer alignment */
	/* stx.stx_dio_offset_align -- offset/length alignment */
```

Before this, applications hardcoded 512 or 4096 and broke on devices with other geometries. This is a good example of Ch. 24's lesson: an interface that requires the caller to know something it cannot query is a broken interface, and the fix is to make it queryable.

Note the `HOLE`/`UNWRITTEN` case above: a direct read of a hole does **zero I/O**. It just zeroes the user buffer. This is correct, fast, and surprises people benchmarking on sparse files — a "10 GB/s O_DIRECT read" is usually a sparse file.

### T.7 The coherency problem

Buffered and direct I/O to the same file simultaneously is the hardest correctness problem in this chapter. Direct I/O writes to the device; buffered reads read from the page cache; without care they diverge.

The kernel's approach:

1. Before a direct write: `filemap_write_and_wait_range()` to flush overlapping dirty pages, then `invalidate_inode_pages2_range()` to drop them.
2. After a direct write: invalidate again, because a concurrent buffered read may have repopulated the cache during the DIO.
3. `IOMAP_DIO_UNWRITTEN`/`IOMAP_DIO_COW` handling on completion.

This is **best-effort**. The kernel documentation says so plainly:

> "Mixing `O_DIRECT` and normal I/O on the same file ... is likely to result in data corruption or undefined behaviour."

The reason it is best-effort rather than correct is that making it correct would require a lock held across the entire DIO, which would serialise all I/O to the file — defeating the purpose. The design chose performance and documented the hazard. Whether that was right is arguable; what matters is knowing it.

There is a second, related problem: **`mmap` and `O_DIRECT`.** A page mapped `PROT_WRITE` can be dirtied by the CPU at any time with no kernel involvement. There is no way for the kernel to know. So `mmap` + `O_DIRECT` on the same range is simply unsafe, and no amount of invalidation fixes it.

For serialisation, XFS and others take `i_rwsem` in shared mode for direct reads and — historically — exclusive for direct writes. The exclusive write lock serialises all DIO writes to a file, which is a real bottleneck for databases. Hence the long campaign for **concurrent DIO writes**: 6.8+ allows non-overlapping, non-extending, fully-aligned DIO writes to proceed in parallel under a shared lock, because such writes do not modify metadata and cannot overlap. Note how much had to be true for the optimisation to be safe — that list *is* the proof obligation.

### T.8 Delayed allocation, unwritten extents, and why both exist

Two mechanisms that look similar and solve opposite problems.

**Delayed allocation** (ext4, XFS, btrfs): on `write()`, reserve space but do not choose blocks. Choose at writeback time, when you know the total size.

Why: at `write()` time you have 4 KiB and no idea what comes next. At writeback time you have 40 MiB of contiguous dirty data and can allocate one extent. **Delalloc converts a sequence of small allocation decisions into one large one**, which is the single biggest anti-fragmentation mechanism in modern filesystems.

The cost: `ENOSPC` can be reported at `fsync`/`close`/writeback time rather than at `write()` time, because the actual allocation happens later. Applications that check the return of `write()` but not of `close()` lose data. (`man 2 close` warns about exactly this.) Reservation accounting keeps this rare but cannot eliminate it — the reservation is approximate because metadata costs are not known until allocation.

**Unwritten extents**: on `fallocate()`, allocate blocks and mark them "never written." Reads return zeros without I/O. The first write converts the extent to mapped.

Why: `fallocate()` must not expose stale data (that would be an information leak of whatever was previously in those blocks), but zeroing gigabytes at `fallocate()` time is unacceptable. The unwritten flag is a **promise recorded in metadata instead of work done on data** — the same trick as a hole, but with the space reserved.

The cost: converting an unwritten extent on write is a metadata update, so the first write to a preallocated region is more expensive than a subsequent one, and it can *split* an extent into three (written, unwritten, unwritten) causing metadata growth. Databases that `fallocate` then write randomly see this.

Contrast them:

| | Delayed allocation | Unwritten extents |
|---|---|---|
| Space reserved at | `write()` | `fallocate()` |
| Blocks chosen at | writeback | `fallocate()` |
| Purpose | contiguity | avoid zeroing, guarantee space |
| Visible in `iomap` as | `IOMAP_DELALLOC` | `IOMAP_UNWRITTEN` |
| `FIEMAP` flag | `DELALLOC` | `UNWRITTEN` |

Both exist because they answer different questions: delalloc optimises *where*, unwritten optimises *when*.

### T.9 `FIEMAP` and `SEEK_HOLE`: exposing the map to userspace

Two interfaces let userspace see §T.1's alphabet.

**`FIEMAP`** (an ioctl) returns the full extent list:

```c
	struct fiemap *fm = calloc(1, sizeof(*fm) + n * sizeof(struct fiemap_extent));
	fm->fm_start = 0;
	fm->fm_length = FIEMAP_MAX_OFFSET;
	fm->fm_extent_count = n;
	ioctl(fd, FS_IOC_FIEMAP, fm);
	/* fe_flags: LAST, UNWRITTEN, DELALLOC, ENCODED, SHARED, ... */
```

Note `FIEMAP_EXTENT_SHARED`: that is a reflink. `FIEMAP_EXTENT_DELALLOC` requires `FIEMAP_FLAG_SYNC` to be meaningful, because unflushed data has no location yet.

**`SEEK_HOLE`/`SEEK_DATA`** (`lseek` whences) are the portable, minimal version:

```c
	off_t data = lseek(fd, off, SEEK_DATA);   /* next data at or after off */
	off_t hole = lseek(fd, off, SEEK_HOLE);   /* next hole at or after off */
```

The specification has a deliberate weakness: a filesystem is **permitted** to report everything as data. That makes it always-correct-if-conservative, so `cp` can use it as an optimisation without a correctness dependency. Compare `FIEMAP`, which is informational and explicitly not a stable contract. The `SEEK_HOLE` design is the better one and the reason it is in POSIX while `FIEMAP` is not.

This matters practically: sparse-aware copying (`cp --sparse=auto`, `tar -S`, `rsync -S`) uses `SEEK_HOLE`. `copy_file_range(2)` goes further and lets the *kernel* do the copy, enabling server-side copy on NFS and reflink on btrfs/XFS — turning an O(size) copy into an O(1) metadata operation.

### T.10 DAX: when there is no page cache

If storage is byte-addressable persistent memory, the page cache is pure overhead: you would be copying from one region of memory to another region of memory. **DAX** removes it.

```c
	if (IS_DAX(inode))
		return dax_iomap_rw(iocb, to, &ops);
```

`mmap` with DAX maps the *storage* into the process. A store instruction writes to persistent memory. There is no page cache, no writeback, no `msync` flushing pages — but you do need `CLFLUSHOPT`/`CLWB` + `SFENCE` to get data out of the CPU cache, which is why `msync` still exists and why userspace libraries like PMDK exist.

DAX is a good closing argument for this chapter because it shows what all the machinery was *for*. Remove the latency gap between memory and storage and the page cache, readahead, writeback, and the elevator all become unnecessary. They exist because of a 10,000× gap (Ch. 52 §T.1), not because of any inherent property of files.

Note that `iomap` handles DAX through the same `iomap_ops`. That is a strong signal the abstraction is right: the same mapping query serves buffered I/O, direct I/O, and DAX.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `fs/iomap/buffered-io.c` | `iomap_read_folio`, `iomap_readahead`, `iomap_file_buffered_write`, `iomap_writepages` |
| `fs/iomap/direct-io.c` | `iomap_dio_rw` — the whole of §T.6 |
| `fs/iomap/iter.c` | `iomap_iter()` — the loop that calls `iomap_begin`/`iomap_end` |
| `fs/iomap/fiemap.c` | `iomap_fiemap`, `iomap_seek_hole`, `iomap_seek_data` |
| `fs/iomap/swapfile.c` | swapfile activation via iomap |
| `include/linux/iomap.h` | `struct iomap`, `iomap_ops`, `iomap_writeback_ops` |
| `mm/filemap.c` | `filemap_read`, `filemap_get_pages`, `generic_file_write_iter` |
| `fs/buffer.c` | the legacy `buffer_head` paths (`block_read_full_folio`, `__block_write_begin`) |
| `fs/direct-io.c` | the **old** DIO implementation; being removed as filesystems convert |
| `fs/xfs/xfs_iomap.c` ★★★ | the reference `iomap_ops` implementation |
| `fs/ext4/inode.c` | ext4's partial conversion — instructive for what conversion involves |
| `Documentation/filesystems/iomap/` ★★★ | design, operations, and porting guide |

### 1.2 The iomap iteration loop

Everything in iomap is built on one loop:

```c
struct iomap_iter {
	struct inode *inode;
	loff_t pos;
	u64 len;
	u64 iter_start_pos;
	int status;
	unsigned flags;
	struct iomap iomap;
	struct iomap srcmap;
	void *private;
};

int iomap_iter(struct iomap_iter *iter, const struct iomap_ops *ops);
```

used as:

```c
ssize_t iomap_file_buffered_write(struct kiocb *iocb, struct iov_iter *i,
				  const struct iomap_ops *ops, void *private)
{
	struct iomap_iter iter = {
		.inode	= iocb->ki_filp->f_mapping->host,
		.pos	= iocb->ki_pos,
		.len	= iov_iter_count(i),
		.flags	= IOMAP_WRITE,
		.private = private,
	};
	ssize_t ret;

	while ((ret = iomap_iter(&iter, ops)) > 0)
		iter.status = iomap_write_iter(&iter, i);
	...
}
```

Read that loop carefully — it is the shape of *every* iomap operation:

1. `iomap_iter()` calls `->iomap_begin()` for the remaining range, getting one extent.
2. The body processes as much of that extent as it can, setting `iter.status` to the bytes processed.
3. `iomap_iter()` calls `->iomap_end()` with what was processed, advances `pos`, and loops.

The filesystem answers §T.1's question once per *extent*. The generic code does everything else. This is why converting a filesystem to iomap deletes far more code than it adds.

### 1.3 A complete `iomap_ops`

```c
// SPDX-License-Identifier: GPL-2.0
static int myfs_iomap_begin(struct inode *inode, loff_t pos, loff_t length,
			    unsigned flags, struct iomap *iomap,
			    struct iomap *srcmap)
{
	struct myfs_inode_info *mi = MYFS_I(inode);
	struct myfs_extent ext;
	int ret;

	/* Find the extent covering `pos`, or the hole that does. */
	ret = myfs_extent_lookup(mi, pos, length, &ext);
	if (ret < 0)
		return ret;

	iomap->bdev   = inode->i_sb->s_bdev;
	iomap->offset = ext.file_off;
	iomap->length = ext.len;

	if (ext.is_hole) {
		if (!(flags & IOMAP_WRITE)) {
			iomap->type = IOMAP_HOLE;
			iomap->addr = IOMAP_NULL_ADDR;
			return 0;
		}
		/* Writing into a hole: allocate (or reserve, for delalloc). */
		ret = myfs_allocate(mi, pos, length, &ext);
		if (ret)
			return ret;
		iomap->flags |= IOMAP_F_NEW;   /* tells iomap to zero the tail */
	}

	iomap->addr = (u64)ext.dev_block << inode->i_blkbits;
	iomap->type = ext.unwritten ? IOMAP_UNWRITTEN : IOMAP_MAPPED;

	if (flags & IOMAP_WRITE && ext.shared) {
		/* Copy-on-write: srcmap says where to read the old data. */
		*srcmap = *iomap;
		ret = myfs_cow_allocate(mi, pos, length, &ext);
		if (ret)
			return ret;
		iomap->addr = (u64)ext.dev_block << inode->i_blkbits;
		iomap->type = IOMAP_MAPPED;
		iomap->flags |= IOMAP_F_SHARED;
	}
	return 0;
}

static int myfs_iomap_end(struct inode *inode, loff_t pos, loff_t length,
			  ssize_t written, unsigned flags, struct iomap *iomap)
{
	if (!(flags & IOMAP_WRITE))
		return 0;
	if (written < length) {
		/* We reserved or allocated more than was used. Give it back. */
		myfs_punch_unused(inode, pos + written, length - written);
	}
	return 0;
}

static const struct iomap_ops myfs_iomap_ops = {
	.iomap_begin = myfs_iomap_begin,
	.iomap_end   = myfs_iomap_end,
};
```

And the `address_space_operations` that uses it becomes nearly trivial:

```c
static int myfs_read_folio(struct file *f, struct folio *folio)
{
	return iomap_read_folio(folio, &myfs_iomap_ops);
}

static void myfs_readahead(struct readahead_control *rac)
{
	iomap_readahead(rac, &myfs_iomap_ops);
}

static int myfs_writepages(struct address_space *mapping,
			   struct writeback_control *wbc)
{
	struct iomap_writepage_ctx ctx = { .ops = &myfs_writeback_ops };

	return iomap_writepages(mapping, wbc, &ctx);
}

static const struct address_space_operations myfs_aops = {
	.read_folio	= myfs_read_folio,
	.readahead	= myfs_readahead,
	.writepages	= myfs_writepages,
	.dirty_folio	= iomap_dirty_folio,
	.release_folio	= iomap_release_folio,
	.invalidate_folio = iomap_invalidate_folio,
	.bmap		= myfs_bmap,
	.migrate_folio	= filemap_migrate_folio,
	.is_partially_uptodate = iomap_is_partially_uptodate,
	.error_remove_folio = generic_error_remove_folio,
};
```

Note the absence of `write_begin`, `write_end`, and `writepage`. `->write_iter` calls `iomap_file_buffered_write()` directly.

### 1.4 `iomap_folio_state`: per-block state without `buffer_head`

`iomap` still needs per-block uptodate/dirty state within a folio (for sub-folio-size blocks and partial writes). It stores it as **two bitmaps** in one small allocation:

```c
struct iomap_folio_state {
	spinlock_t	state_lock;
	unsigned int	read_bytes_pending;
	atomic_t	write_bytes_pending;
	unsigned long	state[];     /* uptodate bits, then dirty bits */
};
```

Compare to §T.2: for a 4 KiB folio with 512-byte blocks, `buffer_head` used 8 × ~104 = 832 bytes; `iomap_folio_state` uses ~40 bytes total. For a 2 MiB large folio with 4 KiB blocks, `buffer_head` would need 512 structures (~53 KiB); iomap needs 128 bits of bitmap.

**This is the same structure, done right**: separate the per-block state (a bitmap) from the mapping (an extent) from the I/O unit (a bio). Three jobs, three structures.

### 1.5 Direct I/O completion

```c
static void iomap_dio_bio_end_io(struct bio *bio)
{
	struct iomap_dio *dio = bio->bi_private;
	bool should_dirty = (dio->flags & IOMAP_DIO_DIRTY);

	if (bio->bi_status)
		iomap_dio_set_error(dio, blk_status_to_errno(bio->bi_status));

	if (!atomic_dec_and_test(&dio->ref))
		goto release_bio;

	if (dio->wait_for_completion) {
		struct task_struct *waiter = dio->submit.waiter;

		WRITE_ONCE(dio->submit.waiter, NULL);
		blk_wake_io_task(waiter);
	} else if (dio->flags & IOMAP_DIO_WRITE) {
		/* Completion work (unwritten conversion, i_size) needs a
		 * sleepable context, so defer to a workqueue. */
		struct inode *inode = file_inode(dio->iocb->ki_filp);

		WRITE_ONCE(dio->iocb->private, NULL);
		INIT_WORK(&dio->aio.work, iomap_dio_complete_work);
		queue_work(inode->i_sb->s_dio_done_wq, &dio->aio.work);
	} else {
		WRITE_ONCE(dio->iocb->private, NULL);
		iomap_dio_complete_work(&dio->aio.work);
	}
release_bio:
	...
}
```

The three-way branch is the whole story: synchronous DIO wakes the waiter; asynchronous *writes* need a workqueue because they must update metadata; asynchronous *reads* can complete in interrupt context because there is nothing to update. Matching completion context to required work is the recurring pattern (Ch. 17, Ch. 46).

### 1.6 Observability surface

| Where | What |
|---|---|
| `trace-cmd record -e iomap:\*` | `iomap_iter`, `iomap_readahead`, `iomap_writepage`, `iomap_dio_rw_begin`, `iomap_dio_complete` ★★★ |
| `trace-cmd record -e xfs:xfs_iomap\*` | XFS's mapping decisions with extent details |
| `trace-cmd record -e block:\*` | the bios that result |
| `filefrag -v FILE` | the extent map, human-readable |
| `xfs_bmap -vvp FILE` | XFS extents including unwritten/delalloc |
| `xfs_io -c 'fiemap -v' FILE` | portable `FIEMAP` dump |
| `xfs_io -c 'bmap -vp'` / `-c 'seek -a 0'` | `SEEK_HOLE`/`SEEK_DATA` walk |
| `statx` with `STATX_DIOALIGN` | the DIO alignment requirements |
| `/sys/block/*/queue/{logical,physical}_block_size` | device geometry |
| `/sys/block/*/queue/dma_alignment` | buffer alignment |

---

## 2. Practice

### Lab 55.1 — See the five states

```sh
sudo apt install -y xfsprogs e2fsprogs
mkdir -p /tmp/iolab && cd /tmp/iolab

# A file exercising every state
truncate -s 100M sparse.dat                       # HOLE
dd if=/dev/urandom of=sparse.dat bs=4k seek=100 count=100 conv=notrunc  # MAPPED
fallocate -o $((10*1024*1024)) -l 10M sparse.dat  # UNWRITTEN

filefrag -v sparse.dat
xfs_io -c 'fiemap -v' sparse.dat
```

Read the flags column carefully: `unwritten`, `last`, and (on a reflinked file) `shared`.

Now catch `DELALLOC` in the act:

```sh
# Write without syncing, then look immediately
dd if=/dev/zero of=delalloc.dat bs=1M count=50 2>/dev/null
xfs_io -c 'fiemap -v' delalloc.dat        # may show nothing / delalloc
sync
xfs_io -c 'fiemap -v' delalloc.dat        # now one big extent

# Contrast: allocate 4 KiB at a time with sync in between (fragmentation)
rm -f frag.dat
for i in $(seq 1 200); do
  dd if=/dev/zero of=frag.dat bs=4k count=1 seek=$i conv=notrunc 2>/dev/null
  sync
done
filefrag frag.dat                          # many extents

rm -f nofrag.dat
dd if=/dev/zero of=nofrag.dat bs=800k count=1 2>/dev/null
sync
filefrag nofrag.dat                        # one or two extents
```

That comparison **is** delayed allocation's value, measured.

And `SEEK_HOLE`/`SEEK_DATA`:

```c
// SPDX-License-Identifier: GPL-2.0
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int main(int argc, char **argv)
{
	int fd = open(argv[1], O_RDONLY);
	off_t pos = 0, end = lseek(fd, 0, SEEK_END), d, h;

	while (pos < end) {
		d = lseek(fd, pos, SEEK_DATA);
		if (d < 0) break;
		h = lseek(fd, d, SEEK_HOLE);
		if (h < 0) h = end;
		printf("DATA [%12ld, %12ld)  %ld bytes\n", d, h, h - d);
		pos = h;
	}
	close(fd);
	return 0;
}
```

```sh
gcc -o seekmap seekmap.c && ./seekmap sparse.dat
# Compare to filefrag: note that UNWRITTEN reads as a hole to SEEK_DATA
# on some filesystems and as data on others -- the spec permits both.
```

That last observation is §T.9's "conservative is allowed" rule made visible.

---

### Lab 55.2 — Trace a read from syscall to bio

```sh
sudo trace-cmd record -e iomap:iomap_iter -e iomap:iomap_readahead \
                      -e block:block_bio_queue -e block:block_rq_issue \
                      -- sh -c 'echo 3 > /proc/sys/vm/drop_caches;
                                dd if=/tmp/iolab/sparse.dat of=/dev/null bs=1M count=20'
sudo trace-cmd report | head -60
```

Count the ratio:

```sh
sudo trace-cmd report | grep -c iomap_iter
sudo trace-cmd report | grep -c block_bio_queue
```

**One `iomap_iter` per extent; many folios per bio.** That is §T.3 and §T.5 quantified.

Now the contrast with a fragmented file:

```sh
sudo trace-cmd record -e iomap:iomap_iter -e block:block_bio_queue \
  -- sh -c 'echo 3 > /proc/sys/vm/drop_caches; cat /tmp/iolab/frag.dat > /dev/null'
sudo trace-cmd report | grep -c iomap_iter     # many more
```

Fragmentation costs mapping calls *and* bios. Measure both:

```sh
for f in nofrag.dat frag.dat; do
  echo 3 | sudo tee /proc/sys/vm/drop_caches > /dev/null
  echo -n "$f ($(filefrag -k $f | grep -o '[0-9]* extent' )): "
  /usr/bin/time -f '%e s' cat $f > /dev/null
done
```

Per-bio detail:

```sh
sudo bpftrace -e '
tracepoint:block:block_bio_queue {
	@size = hist(args->nr_sector * 512);
	@count = count();
}
interval:s:5 { print(@size); print(@count); clear(@size); clear(@count); }'
```

Run `cat nofrag.dat` and `cat frag.dat` under this and compare the bio-size histograms. Large contiguous extents produce large bios; fragmented files produce many small ones.

---

### Lab 55.3 — Buffered vs direct I/O, honestly

```c
// SPDX-License-Identifier: GPL-2.0
/* iobench.c: buffered vs O_DIRECT, with correct alignment handling. */
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h>
#include <sys/syscall.h>
#include <time.h>
#include <unistd.h>
#include <linux/stat.h>

static double now(void)
{
	struct timespec ts;
	clock_gettime(CLOCK_MONOTONIC, &ts);
	return ts.tv_sec + ts.tv_nsec / 1e9;
}

static void dio_align(const char *path, unsigned *mem, unsigned *off)
{
#ifdef STATX_DIOALIGN
	struct statx stx;

	if (syscall(__NR_statx, AT_FDCWD, path, 0, STATX_DIOALIGN, &stx) == 0 &&
	    (stx.stx_mask & STATX_DIOALIGN) && stx.stx_dio_mem_align) {
		*mem = stx.stx_dio_mem_align;
		*off = stx.stx_dio_offset_align;
		printf("STATX_DIOALIGN: mem=%u offset=%u\n", *mem, *off);
		return;
	}
#endif
	*mem = *off = 4096;
	printf("STATX_DIOALIGN unavailable; assuming 4096\n");
}

static double run(const char *path, int direct, size_t bs, size_t total)
{
	unsigned memal, offal;
	int fd, flags = O_RDONLY;
	void *buf;
	double t0;
	size_t done = 0;

	dio_align(path, &memal, &offal);
	if (direct) {
		flags |= O_DIRECT;
		if (bs % offal) { fprintf(stderr, "bs unaligned\n"); exit(1); }
	}
	fd = open(path, flags);
	if (fd < 0) { perror("open"); exit(1); }

	if (posix_memalign(&buf, memal, bs)) { perror("memalign"); exit(1); }

	t0 = now();
	while (done < total) {
		ssize_t n = read(fd, buf, bs);
		if (n < 0) { perror("read"); exit(1); }
		if (n == 0) { lseek(fd, 0, SEEK_SET); continue; }
		done += n;
	}
	close(fd);
	free(buf);
	return total / (now() - t0) / (1 << 20);
}

int main(int argc, char **argv)
{
	const char *path = argv[1];
	size_t total = 512UL << 20;
	size_t sizes[] = { 4096, 65536, 1 << 20, 4 << 20 };
	int i;

	for (i = 0; i < 4; i++) {
		printf("bs=%-8zu buffered=%7.1f MB/s  direct=%7.1f MB/s\n",
		       sizes[i],
		       run(path, 0, sizes[i], total),
		       run(path, 1, sizes[i], total));
	}
	return 0;
}
```

```sh
gcc -O2 -o iobench iobench.c
dd if=/dev/urandom of=/tmp/iolab/bench.dat bs=1M count=1024
echo 3 | sudo tee /proc/sys/vm/drop_caches
./iobench /tmp/iolab/bench.dat
```

Expect: buffered wins at small block sizes (readahead), direct catches up at large ones. Then the crucial second run:

```sh
# WITHOUT dropping caches -- buffered now reads from RAM
./iobench /tmp/iolab/bench.dat
```

Buffered becomes 10–50× faster. **That is the page cache, and it is why `O_DIRECT` is not "the fast option."**

Now demonstrate the alignment errors:

```c
	/* All of these give EINVAL with O_DIRECT: */
	read(fd, buf, 511);              /* length not a multiple */
	pread(fd, buf, 4096, 100);       /* offset not aligned */
	read(fd, malloc(4096), 4096);    /* buffer not aligned */
```

```sh
sudo bpftrace -e '
kretprobe:iomap_dio_rw /retval == -22/ {
	printf("%s: O_DIRECT EINVAL\n", comm);
}'
```

And the sparse-file trap:

```sh
truncate -s 4G /tmp/iolab/allholes.dat
./iobench /tmp/iolab/allholes.dat    # absurd "throughput": zero I/O (T.6)
sudo trace-cmd record -e block:block_bio_queue -- ./iobench /tmp/iolab/allholes.dat
sudo trace-cmd report | wc -l        # essentially nothing
```

---

### Lab 55.4 — Prove `O_DIRECT` is not durable

This is Ch. 51 §T.4's trap, now with the mechanism visible.

```sh
sudo modprobe scsi_debug dev_size_mb=256 sector_size=512
DEV=$(lsblk -ndo NAME,MODEL | awk '/scsi_debug/{print "/dev/"$1}')
sudo mkfs.xfs -f $DEV
sudo mkdir -p /mnt/dtest && sudo mount $DEV /mnt/dtest

# Watch for flush requests
sudo bpftrace -e '
tracepoint:block:block_rq_issue {
	if (str(args->rwbs) == "FWS" || str(args->rwbs) == "WFS" ||
	    strcontains(str(args->rwbs), "F")) {
		printf("FLUSH/FUA: %s from %s\n", str(args->rwbs), args->comm);
	}
}' &

# O_DIRECT write alone: NO flush
sudo xfs_io -d -c 'pwrite -b 4k 0 4k' /mnt/dtest/f1

# O_DIRECT + fdatasync: flush appears
sudo xfs_io -d -c 'pwrite -b 4k 0 4k' -c fdatasync /mnt/dtest/f2

# O_DIRECT|O_DSYNC: FUA or flush per write
sudo xfs_io -d -s -c 'pwrite -b 4k 0 4k' /mnt/dtest/f3
```

The first case produces no flush. The data is on the device but possibly only in its volatile cache. **`O_DIRECT` bypassed the page cache, not the device cache.**

Compare the cost:

```sh
sudo xfs_io -d -c 'pwrite -b 4k -S 0x41 0 40m' /mnt/dtest/big1        # no sync
sudo xfs_io -d -s -c 'pwrite -b 4k -S 0x42 0 40m' /mnt/dtest/big2     # O_DSYNC
# The second is dramatically slower: a flush per 4 KiB.
```

---

### Lab 55.5 — Coherency: break it deliberately

```c
// SPDX-License-Identifier: GPL-2.0
/* incoherent.c: mixing buffered and direct I/O on one file. */
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main(void)
{
	const char *path = "/tmp/iolab/mixed.dat";
	void *dbuf;
	char bbuf[4096];
	int bfd, dfd;

	posix_memalign(&dbuf, 4096, 4096);

	/* Create and populate with 'A' via buffered I/O */
	bfd = open(path, O_RDWR | O_CREAT | O_TRUNC, 0644);
	memset(bbuf, 'A', sizeof(bbuf));
	pwrite(bfd, bbuf, 4096, 0);
	/* Deliberately do NOT sync. The page cache holds 'A'. */

	/* Now write 'B' via O_DIRECT */
	dfd = open(path, O_RDWR | O_DIRECT);
	memset(dbuf, 'B', 4096);
	pwrite(dfd, dbuf, 4096, 0);

	/* Read back both ways */
	pread(bfd, bbuf, 4096, 0);
	printf("buffered read sees: %c\n", bbuf[0]);
	pread(dfd, dbuf, 4096, 0);
	printf("direct   read sees: %c\n", ((char *)dbuf)[0]);

	close(bfd); close(dfd);
	return 0;
}
```

```sh
gcc -O2 -o incoherent incoherent.c && ./incoherent
```

The kernel flushes and invalidates around the direct write (§T.7), so this usually agrees. Now make it race:

```c
	/* Thread A: buffered writes in a loop
	 * Thread B: O_DIRECT writes to the same offset in a loop
	 * Thread C: buffered reads, checking for a value neither wrote */
```

```sh
sudo bpftrace -e '
kprobe:invalidate_inode_pages2_range { @invalidate = count(); }
kprobe:filemap_write_and_wait_range  { @flush = count(); }
kretprobe:invalidate_inode_pages2_range /retval != 0/ {
	@failed_invalidate = count();   /* someone held the page */
}
interval:s:2 { print(@invalidate); print(@flush); print(@failed_invalidate);
               clear(@invalidate); clear(@flush); clear(@failed_invalidate); }'
```

`failed_invalidate` is the interesting counter: it is the kernel being *unable* to drop a cached page because someone is using it. That is the window §T.7 documents. The point of the lab is not to produce corruption reliably — it is to see the machinery and understand why it is best-effort.

The `mmap` case has no such machinery at all:

```sh
xfs_io -c 'mmap -rw 0 4k' -c 'mwrite 0 4k' -c 'pwrite -b 4k 0 4k' /tmp/iolab/mixed.dat
# There is no mechanism that could make this coherent.
```

---

### Lab 55.6 — Reflinks, `copy_file_range`, and shared extents

```sh
# XFS or btrfs required
sudo mkfs.xfs -f -m reflink=1 $DEV
sudo mount $DEV /mnt/dtest

sudo dd if=/dev/urandom of=/mnt/dtest/orig bs=1M count=100
df -h /mnt/dtest

sudo cp --reflink=always /mnt/dtest/orig /mnt/dtest/clone
df -h /mnt/dtest                  # unchanged: O(1) copy
ls -l /mnt/dtest                  # both are 100M

sudo xfs_io -c 'fiemap -v' /mnt/dtest/orig  | head
sudo xfs_io -c 'fiemap -v' /mnt/dtest/clone | head
# Same physical extents, flagged 0x2000 (shared)

# Now write to the clone: copy-on-write kicks in
sudo dd if=/dev/urandom of=/mnt/dtest/clone bs=4k count=1 conv=notrunc
sync
sudo xfs_io -c 'fiemap -v' /mnt/dtest/clone | head
# The written extent is no longer shared; it was allocated elsewhere.
df -h /mnt/dtest                  # 4 KiB more used
```

Watch the CoW path through iomap:

```sh
sudo trace-cmd record -e xfs:xfs_reflink\* -e iomap:iomap_iter \
  -- sudo dd if=/dev/urandom of=/mnt/dtest/clone bs=4k count=1 seek=100 conv=notrunc
sudo trace-cmd report | head -30
```

You are watching `iomap_begin` fill in both `iomap` and `srcmap` — §T.3's CoW support, in action.

`copy_file_range`:

```c
// SPDX-License-Identifier: GPL-2.0
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <sys/sendfile.h>
#include <unistd.h>

int main(int argc, char **argv)
{
	int in = open(argv[1], O_RDONLY);
	int out = open(argv[2], O_WRONLY | O_CREAT | O_TRUNC, 0644);
	off_t off_in = 0, off_out = 0;
	ssize_t n;

	while ((n = copy_file_range(in, &off_in, out, &off_out,
				    1UL << 30, 0)) > 0)
		;
	if (n < 0) perror("copy_file_range");
	return 0;
}
```

```sh
gcc -o cfr cfr.c
sudo trace-cmd record -e block:block_bio_queue -- sudo ./cfr /mnt/dtest/orig /mnt/dtest/cfr-copy
sudo trace-cmd report | wc -l     # on a reflink fs: almost no data I/O
```

Compare with `cp --reflink=never`. The difference is O(1) metadata versus O(size) data movement.

---

### Lab 55.7 — Convert a toy filesystem to iomap

Take a minimal `buffer_head`-based filesystem and convert it. Start from the legacy form:

```c
// SPDX-License-Identifier: GPL-2.0
/* BEFORE: buffer_head based */
static int myfs_get_block(struct inode *inode, sector_t iblock,
			  struct buffer_head *bh, int create)
{
	sector_t phys = myfs_lookup_block(inode, iblock, create);

	if (!phys)
		return create ? -ENOSPC : 0;     /* hole */
	map_bh(bh, inode->i_sb, phys);
	return 0;
}

static int myfs_read_folio(struct file *f, struct folio *folio)
{
	return block_read_full_folio(folio, myfs_get_block);
}

static int myfs_write_begin(struct file *f, struct address_space *m,
			    loff_t pos, unsigned len,
			    struct folio **foliop, void **fsdata)
{
	return block_write_begin(m, pos, len, foliop, myfs_get_block);
}

static int myfs_writepages(struct address_space *m,
			   struct writeback_control *wbc)
{
	return mpage_writepages(m, wbc, myfs_get_block);
}
```

and convert to §1.3's form. The exercise:

1. Write `myfs_iomap_begin()` returning *ranges*. The key change: `myfs_lookup_block()` returns one block; you need `myfs_lookup_extent()` returning a run. Even a naive loop that probes forward for contiguity is a massive improvement.
2. Replace `block_read_full_folio` → `iomap_read_folio`, add `->readahead` → `iomap_readahead`.
3. Delete `write_begin`/`write_end`; make `->write_iter` call `iomap_file_buffered_write()`.
4. Implement `iomap_writeback_ops.map_blocks` and use `iomap_writepages`.
5. Add `iomap_fiemap` and `iomap_seek_hole`/`iomap_seek_data`.
6. Add `iomap_dio_rw` for `O_DIRECT`.

Measure before and after:

```sh
sudo trace-cmd record -e 'myfs:*' -e block:block_bio_queue -- \
  dd if=/mnt/myfs/big of=/dev/null bs=1M count=64
# BEFORE: one get_block per block (16384 calls for 64 MiB at 4 KiB)
# AFTER:  one iomap_begin per extent
```

That count ratio is §T.2 versus §T.3, measured on your own code.

---

### Lab 55.8 — Instrument the whole path

```sh
# Full-stack view of one read
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_read /comm == "dd"/ { @t[tid] = nsecs; }
kprobe:filemap_read            /@t[tid]/ { @stage["filemap_read"]  = count(); }
kprobe:page_cache_sync_readahead /@t[tid]/ { @stage["sync_ra"]     = count(); }
kprobe:iomap_readahead         /@t[tid]/ { @stage["iomap_readahead"] = count(); }
kprobe:submit_bio              /@t[tid]/ { @stage["submit_bio"]    = count(); }
kprobe:folio_wait_bit_common   /@t[tid]/ { @stage["wait_on_folio"] = count(); }
tracepoint:syscalls:sys_exit_read /@t[tid]/ {
	@latency = hist(nsecs - @t[tid]); delete(@t[tid]);
}
END { print(@stage); print(@latency); }'
```

```sh
echo 3 | sudo tee /proc/sys/vm/drop_caches
dd if=/tmp/iolab/bench.dat of=/dev/null bs=1M count=64
```

Then the same, warm — `submit_bio` should drop to zero.

Latency attribution across the whole stack:

```sh
sudo bpftrace -e '
kprobe:iomap_iter                { @iomap_t[tid] = nsecs; }
kretprobe:iomap_iter /@iomap_t[tid]/ {
	@iomap_ns = hist(nsecs - @iomap_t[tid]); delete(@iomap_t[tid]);
}
kprobe:submit_bio                { @bio_t[arg0] = nsecs; }
kprobe:bio_endio /@bio_t[arg0]/  {
	@bio_ns = hist(nsecs - @bio_t[arg0]); delete(@bio_t[arg0]);
}
END { printf("iomap_begin/end cost:\n"); print(@iomap_ns);
      printf("bio round trip:\n"); print(@bio_ns); }'
```

The two histograms should be orders of magnitude apart — mapping lookups in hundreds of nanoseconds, device I/O in tens or hundreds of microseconds. That ratio is *why* it was worth eliminating 255 of every 256 mapping calls, and also why it is not worth optimising further: mapping is no longer the bottleneck.

---

## 3. Mastery drills

1. Enumerate every field of `struct buffer_head` and assign each to exactly one of the three concerns of §T.2(b). Then show which structure holds it in the iomap design.

2. Compute the metadata overhead of `buffer_head` versus `iomap_folio_state` for: a 4 KiB folio with 4 KiB blocks, a 4 KiB folio with 512 B blocks, and a 2 MiB folio with 4 KiB blocks. Express as a percentage of the data.

3. `iomap_begin()` is given a range and may return a *shorter* extent. Under what circumstances must it, and what does the iteration loop do then? Construct a pathological file where this makes iomap no better than `->get_block()`.

4. `srcmap` exists for CoW. Write out, step by step, what a partial-folio write to a shared extent must do, and show why `->get_block()` could not express it.

5. `->writepage` was removed. Reconstruct the deadlock it enabled when called from reclaim context, identifying every lock involved.

6. `write_begin` must make a folio "uptodate enough." State the exact condition under which a read is required, and construct the information leak that occurs if you skip it.

7. Prove that `O_DIRECT` writes are not durable without a flush, using only the tracing from Lab 55.4 as evidence. Then state the minimal change that makes them durable and measure its cost.

8. The 6.8 concurrent-DIO-write optimisation requires four conditions. State each and explain what breaks if you drop it.

9. Delayed allocation can report `ENOSPC` at `close()`. Write the application code that loses data as a result, and the code that does not.

10. An unwritten extent can split into three on a partial write. Compute the worst-case metadata growth for a 1 GiB `fallocate` followed by N random 4 KiB writes, and name the workload that does exactly this.

11. `SEEK_HOLE` permits a filesystem to report everything as data; `FIEMAP` does not have an equivalent escape hatch. Argue which is the better interface design, citing Ch. 24's principles.

12. DAX removes the page cache. Enumerate every subsystem in Ch. 52 and this chapter that becomes unnecessary, and the two that do not. Explain the two.

13. You are handed a filesystem that gets 200 MB/s buffered and 1.8 GB/s with `O_DIRECT` on the same hardware. Give the ordered diagnostic procedure and the three most likely causes.

---

## 4. Further reading

**Kernel documentation**

- `Documentation/filesystems/iomap/design.rst` ★★★ — the rationale for §T.3, from the authors.
- `Documentation/filesystems/iomap/operations.rst` ★★★ — every operation and its contract.
- `Documentation/filesystems/iomap/porting.rst` ★★★ — the conversion guide; Lab 55.7 in normative form.
- `Documentation/filesystems/vfs.rst` — the `address_space_operations` contract.
- `Documentation/filesystems/locking.rst` ★★★ — which locks are held when each aop is called. **Read this before implementing any of them.**
- `Documentation/filesystems/dax.rst` — §T.10.
- `Documentation/filesystems/fiemap.rst` — the ioctl and its flags.
- `man 2 open` (the `O_DIRECT` section) ★★★ — including the coherency warning quoted in §T.7.
- `man 2 lseek` (`SEEK_HOLE`/`SEEK_DATA`), `man 2 copy_file_range`, `man 2 fallocate`, `man 2 statx`.

**Papers and primary sources**

- C. Mason, D. Chinner et al., the iomap patch series cover letters (2016–2019) ★★★ — the argument of §T.2 and §T.3 as originally made.
- M. Wilcox, the folio conversion series ★★★ — why the aops had to change.
- D. Chinner's talks on XFS and iomap (LCA, Vault, LSFMM) — the clearest available explanation of delayed allocation and unwritten extents.
- The original XFS paper: Sweeney et al., "Scalability in the XFS File System," USENIX 1996 — extents and delayed allocation, pre-Linux.

**LWN**

- "The iomap interface" / "Toward a better filesystem I/O interface" ★★★
- "The end of `->writepage()`" and the `writepages` conversion coverage
- "Folios and the page cache" series ★★★
- "O_DIRECT and dragons" and the recurring `O_DIRECT` threads (including Linus's famous assessment)
- "Concurrent direct I/O writes" (2023–2024) ★★★ — §T.7's optimisation
- "`STATX_DIOALIGN`" (2022)
- "Reflink and `copy_file_range()`" coverage
- "Supporting filesystems in persistent memory" ★★★ — DAX's origins
- "Large folios and filesystems" — the ongoing work

**Source reading order**

1. `Documentation/filesystems/iomap/design.rst` — start here.
2. `include/linux/iomap.h` — the structures; they are the design.
3. `fs/iomap/iter.c` — 80 lines, and it is the whole control flow.
4. `fs/iomap/buffered-io.c`: `iomap_readahead` → `iomap_readpage_iter` → the bio building.
5. `fs/xfs/xfs_iomap.c`: `xfs_read_iomap_begin` and `xfs_buffered_write_iomap_begin` ★★★ — the reference implementation; read these two functions carefully.
6. `fs/iomap/direct-io.c`: `iomap_dio_rw` → `iomap_dio_bio_iter` → `iomap_dio_bio_end_io`.
7. `fs/buffer.c`: `block_read_full_folio` — read this *after* the above, to appreciate the contrast.

**Tools**

- `filefrag -v`, `xfs_bmap -vvp`, `xfs_io -c fiemap` ★★★
- `trace-cmd record -e iomap:\*` ★★★ — the single most useful tracing for this chapter
- `fio` with `--direct=0/1`, `--ioengine=io_uring/libaio/psync`, `--bs`, `--rw` ★★★
- `bpftrace` on `iomap_iter`, `iomap_dio_rw`, `submit_bio`, `bio_endio`
- `xfs_io` ★★★ — `pwrite`, `fiemap`, `fsync`, `-d` for direct, `-s` for sync; the best filesystem exploration tool that exists
- `ioping`, `biolatency`, `biosnoop` (bcc)
- `statx` with `STATX_DIOALIGN` — write a two-line program; you will use it repeatedly

---

→ Next: [56-write-a-filesystem.md](56-write-a-filesystem.md)

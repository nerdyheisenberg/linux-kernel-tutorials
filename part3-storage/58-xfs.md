# Chapter 58 — XFS internals

> **Goal:** Understand the filesystem designed from the start for parallelism and scale, and see what changes when a format is not constrained by backwards compatibility with 1993. Understand allocation groups as a sharding strategy, why XFS uses B+trees for *everything*, the free-space-by-size and free-space-by-offset trees, dynamic inode allocation, the delayed logging revolution, the reverse-mapping and reference-count trees that enable reflink and online repair, and the "shut down rather than continue" error philosophy. By the end you can read `fs/xfs/`, use `xfs_db` and `xfs_bmap` fluently, and explain precisely why XFS behaves differently from ext4 under metadata-heavy parallel load.

---

## Theory & First Principles

### T.0 — Start here: what if you designed for 1000 CPUs from day one?

```bash
# ext4: one big filesystem, shared allocation metadata.
# XFS:  N nearly-independent filesystems that share a name space.
xfs_info /mnt | grep agcount     # agcount = 32  <- 32 parallel allocators
```

XFS was written at SGI in 1993 for machines with hundreds of CPUs and terabyte filesystems,
at a time when a PC had 8 MB of RAM. **It was designed for the problem everyone else would
have twenty years later**, which is why it aged better than anything from its era.

**Start with the contention question**, because it drives the whole design. Allocating a
block means updating shared free-space metadata. With one free-space structure, every
allocating thread on the machine serializes on it — Ch. 16 §T.0's counter that gets slower
with more CPUs, applied to a filesystem.

XFS's answer:

```
  +----------------+----------------+----------------+----------------+
  |  Allocation    |  Allocation    |  Allocation    |  Allocation    |
  |  Group 0       |  Group 1       |  Group 2       |  Group 3       |
  |                |                |                |                |
  | free-space     | free-space     | ...            | ...            |
  |   B+tree BY    |   B+tree BY    |                |                |
  |   OFFSET       |   OFFSET       |   independent locks,             |
  | free-space     | free-space     |   independent allocation,        |
  |   B+tree BY    |   B+tree BY    |   independent inode allocation   |
  |   SIZE         |   SIZE         |                |                |
  | inode B+tree   | inode B+tree   |                |                |
  +----------------+----------------+----------------+----------------+
```

**Allocation groups are not ext4's block groups.** Block groups share a global allocator and
exist to reduce seeks. AGs are *independently lockable allocation domains* that exist to
reduce **contention** — different motivation, and it scales where block groups do not.

**Second idea: B+trees for everything.** Free space, inodes, extent maps, directories,
attributes — all B+trees. Notice the free-space pair above: **two** trees over the same data,
one keyed by offset and one by size.

| Query | Tree used |
|---|---|
| "is the block next to this one free?" (extend an extent) | by **offset** |
| "give me a free run of exactly 512 blocks" | by **size** |

A bitmap answers the first in O(1) and the second in O(device size) by scanning. **Two
indices over one dataset, chosen per query — the same move a database makes**, and the reason
XFS allocates well on a nearly-full, heavily-aged filesystem where ext4 degrades.

**Third idea, and the one that made XFS famous: delayed logging.** XFS journals *metadata
changes*, and a busy workload modifies the same metadata repeatedly — the same AG free-space
block might be updated 10,000 times a second. Logging each change means writing that block
10,000 times.

Delayed logging (2010) accumulates changes **in memory**, aggregates repeated modifications to
the same object, and writes one combined log entry. Metadata-heavy workloads got roughly an
order of magnitude faster. **This is the writeback-absorption argument of Ch. 52 §T.0 applied
to the journal itself:** delay is what lets you collapse redundant work.

**What XFS bought and what it cost:**

| Buys | Costs |
|---|---|
| Excellent parallel and large-file performance | **Cannot shrink** — AGs are fixed at mkfs |
| Graceful behaviour on aged, full filesystems | No snapshots (use LVM/dm below it) |
| Online defragmentation, growth, quota | Historically weak on many-tiny-files workloads |
| Now: reflink, CoW data, metadata checksums | Repair needs lots of RAM on huge filesystems |

**The architectural lesson:** XFS is what you get when you take *scalability* as the primary
constraint and accept format complexity as the price — the mirror image of ext4's choice to
take *compatibility* as primary and accept structural limits. Neither is wrong; they optimized
different objectives, and Ch. 59 shows a third.

```bash
xfs_info /mnt                         # agcount, agsize, sector/block sizes
xfs_bmap -vp /mnt/file                # extents, with states
xfs_db -r -c 'freesp -s' /dev/sda1    # free-space distribution by size
cat /sys/fs/xfs/sda1/log/log_*        # log state
```

---

### T.1 A different starting point

XFS was written at SGI in 1993 for IRIX, for machines with dozens of CPUs, hundreds of disks, and files measured in terabytes — in 1993. The design constraints were the opposite of ext2's:

| | ext2 (1993) | XFS (1993) |
|---|---|---|
| Target machine | one disk, one CPU, 1 GB | disk arrays, 32 CPUs, TB |
| Target workload | a Linux workstation | video, scientific data, backup |
| Compatibility burden | ext, then ext3, then ext4 | none — a clean slate |
| Metadata structure | bitmaps and linear tables | **B+trees everywhere** |
| Parallelism | one lock per filesystem, largely | **allocation groups: shard everything** |
| Max file/fs size | 2 GB / 16 TB (later) | 8 EiB / 8 EiB, from day one |

Ported to Linux in 2001, it has been maintained continuously since — for the last decade largely by Dave Chinner and Darrick Wong, with an engineering culture noticeably different from ext4's: aggressive removal of old code, willingness to make incompatible format changes behind feature flags, and a long-term project (online repair) that has been under development for years.

The one-sentence summary: **XFS is what you design when you assume the hardware is big, the workload is parallel, and nothing needs to mount your filesystem except you.**

### T.2 Allocation groups: sharding as the primary design decision

An XFS filesystem is divided into **allocation groups** (AGs), typically 0.5–4 TiB each (capped at 1 TiB by default in `mkfs`). Each AG is **nearly an independent filesystem**:

```
┌─────────────────────────────────────────────────────────────┐
│ AG 0                                                        │
│  sector 0: XFS_SB    (superblock)                           │
│  sector 1: XFS_AGF   (free space header: bno/cnt/rmap roots)│
│  sector 2: XFS_AGI   (inode header: inobt/finobt roots)     │
│  sector 3: XFS_AGFL  (free list for B+tree block allocation)│
│  then: the B+trees and data                                 │
├─────────────────────────────────────────────────────────────┤
│ AG 1  (same structure, independent locks, independent trees)│
├─────────────────────────────────────────────────────────────┤
│ AG 2  ...                                                   │
└─────────────────────────────────────────────────────────────┘
```

The consequences, and this is the heart of XFS:

**(a) Parallelism.** Each AG has its own locks and its own free-space trees. N threads allocating in N different AGs do not contend at all. Ext4's block groups share a single superblock lock for the free-block counter; XFS shards the counters per-AG (with a per-CPU layer on top). **This is why XFS's advantage over ext4 grows with core count**, and why the gap is invisible on a single-threaded benchmark.

**(b) Bounded metadata addressing.** Within an AG, block numbers are 32 bits (`xfs_agblock_t`). A global block number is `(agno, agbno)` packed into 64 bits (`xfs_fsblock_t`). So the B+tree nodes inside an AG store 32-bit pointers, halving their size. The same trick applies to inode numbers: `xfs_agino_t` is AG-relative.

**(c) Damage containment.** Corrupting one AG's trees does not necessarily destroy the filesystem. This becomes important for online repair (§T.8).

**(d) A fixed AG count.** `mkfs.xfs` picks the AG count; `xfs_growfs` adds AGs. Too few AGs limits parallelism; too many wastes space on per-AG metadata and spreads files thinly. The default (4 for small filesystems, then one per TiB) is usually right, and the standard advice to "increase agcount for parallel workloads" is mostly cargo cult — the default already scales.

Compare with ext4's block groups (Ch. 57 §T.2): same word, different purpose. Ext4's groups are a *locality* mechanism with shared global counters. XFS's are a *sharding* mechanism with independent state. That difference is the single biggest architectural distinction between the two filesystems.

### T.3 B+trees for everything

XFS has one generic B+tree implementation (`fs/xfs/libxfs/xfs_btree.c`) and instantiates it for every structure:

| Tree | Key | Value | Purpose |
|---|---|---|---|
| **bnobt** | AG block number | length | free space, **ordered by location** |
| **cntbt** | (length, block) | — | free space, **ordered by size** |
| **inobt** | inode number | 64-inode chunk + alloc bitmap | inode allocation |
| **finobt** | inode number | same, **free inodes only** | fast free-inode lookup |
| **rmapbt** | AG block | (owner, offset, flags) | **reverse map**: who owns this block |
| **refcountbt** | AG block | refcount | **shared block tracking** (reflink) |
| **bmbt** | file offset | extent | per-inode block map |
| directory/attr | hash | entry | directories and xattrs |

**The two free-space trees are the design's signature.** The same free extents are indexed twice:

- `bnobt` answers "is there free space *near here*?" — for locality and for coalescing on free.
- `cntbt` answers "is there a free extent of *at least this size*?" — for large allocations.

An allocation request carries both a goal block and a minimum/maximum length, and XFS consults whichever tree fits the request better. A large allocation queries `cntbt` for a big-enough extent; a locality-sensitive one queries `bnobt` near the goal. Ext4's mballoc approximates this with in-memory buddy bitmaps (Ch. 57 §T.5) rebuilt on demand; XFS keeps it on disk, always current, always O(log n).

The cost: **every free and every allocate updates two trees**, both of which are journalled. XFS's metadata write amplification is higher than ext4's. That is a deliberate trade — pay on write, never pay a linear scan.

**`finobt`** deserves mention because it solves a real problem: finding a free inode without it means scanning `inobt` records looking for one with free slots, which is O(allocated inodes) on a nearly-full filesystem. `finobt` indexes only chunks with free inodes, making it O(log n). Added in 2014, and file-creation performance on large filesystems improved dramatically.

### T.4 Dynamic inode allocation

Ext4 fixes the inode count at `mkfs` time (Ch. 57 §T.7) and `df -i` can hit 100 % with free space remaining. XFS allocates inodes **in chunks of 64, on demand, anywhere in the AG**, tracked by `inobt`.

An XFS inode number encodes its location:

```
ino = (agno << agino_log) | (agblock << inopblog) | offset_in_block
```

So finding an inode is arithmetic, not a lookup — the same O(1) property as ext4's positional scheme, without the fixed table.

The costs, which are real:

**(a) Inode numbers can exceed 32 bits** on filesystems over ~1 TiB (or over 2 TiB with 512-byte inodes), because the AG number is in the high bits. 32-bit applications using `stat()` instead of `stat64()` get `EOVERFLOW`. The `inode32` mount option forces all inodes into the first AGs to keep numbers small — at the cost of concentrating metadata. `inode64` is the default and correct choice; `inode32` exists for legacy binaries and should be avoided.

**(b) Inode chunks need 64 × inode_size contiguous bytes** (16 KiB with 256-byte inodes). On a badly fragmented filesystem this can fail with `ENOSPC` even with free space. The `sparse_inodes` feature (4.2+) allows partial chunks and eliminates it — it is on by default in modern `mkfs.xfs`.

**(c) You cannot pre-reserve inodes.** `imaxpct` (default 25 %) caps the fraction of space inodes may consume, as a guard rather than a reservation.

The trade versus ext4 is clean: XFS never runs out of inodes with free space, ext4 never fragments its inode allocation. In practice XFS's approach is better — "out of inodes with 40 % free" is a much worse failure than "inode allocation is slightly scattered."

### T.5 The inode fork model

An XFS inode has up to three **forks**, and this is an unusually clean abstraction:

```c
struct xfs_inode {
	...
	struct xfs_ifork	i_df;	/* data fork: file data or dir entries */
	struct xfs_ifork	*i_af;	/* attr fork: extended attributes */
	struct xfs_ifork	*i_cowfp; /* CoW fork: in-flight reflink writes */
	...
};
```

Each fork independently has a **format**:

| Format | Meaning |
|---|---|
| `LOCAL` | the data is **inline in the inode** (short symlinks, small dirs, small xattrs) |
| `EXTENTS` | a list of extents inline in the inode's literal area |
| `BTREE` | a B+tree root inline, with the tree on disk |
| `DEV`/`UUID` | special files |

Each fork is promoted independently as it grows: `LOCAL` → `EXTENTS` → `BTREE`.

Two things make this elegant:

**The literal area is shared and dynamically split.** An inode is (say) 512 bytes: a fixed header plus a *literal area* divided between the data fork and the attr fork by `di_forkoff`. A file with no xattrs gives its whole literal area to extents; a file with many xattrs gives more to the attr fork. Ext4 has separate fixed spaces for extents (60 bytes) and inline xattrs, and cannot trade between them.

**The CoW fork** (added with reflink) holds the *new* extents for a copy-on-write in progress, keeping them entirely separate from the committed data fork until the write completes. Ch. 55 §T.3's `srcmap` at the format level. There is no way to express this in ext4's inode.

Practical consequence: **XFS inode size matters more than ext4's.** `mkfs.xfs -i size=512` or `1024` gives more literal area, which means more extents and xattrs stay inline. For SELinux systems or files with many extents this is a measurable win, and it is why 512 became the default.

### T.6 Delayed logging: the change that made XFS fast on Linux

For years XFS had a reputation for poor metadata performance on Linux — bad enough that "XFS is for big files, ext3 for small ones" became folklore. The cause was the journal.

**The problem.** The classic XFS log wrote a *physical* log item for every metadata change, immediately. A `rm -rf` of a directory tree modifies the same AGF, AGI, and B+tree blocks thousands of times, and each modification wrote a log entry. The log became the bottleneck, and log I/O dominated.

**The insight** (Dave Chinner, 2010): most metadata changes touch **the same small set of objects repeatedly**. Rather than logging every change, **accumulate changes in memory and log only the final state periodically.**

```
Classic:  change → log item → log I/O        (every change)
Delayed:  change → in-memory log item (CIL)  (accumulate)
          ... many changes to the same object coalesce ...
          periodically: CIL → checkpoint → log I/O
```

The **Committed Item List (CIL)** holds log items in memory. When an object is modified again while already in the CIL, the existing item is updated rather than a new one added. A checkpoint is pushed when the CIL exceeds a threshold (1/8 of the log) or on a sync.

The results were dramatic — **an order of magnitude** on metadata-heavy workloads, and it is why XFS's reputation reversed between 2010 and 2015.

The relationship to ext4's journal (Ch. 61): jbd2 also batches, but it batches *transactions* into a commit. XFS's CIL batches *changes to individual objects* across transactions, which is strictly more aggressive. `fast_commit` (Ch. 57 §T.10) is ext4 moving in this direction.

**What it costs:**

- Memory: the CIL holds dirty metadata in RAM.
- Recovery complexity: log recovery must replay checkpoints, which is more involved than replaying transactions.
- **Log size matters more.** A small log forces frequent checkpoints and defeats the coalescing. `mkfs.xfs -l size=` is one of the few tuning knobs that genuinely matters.

XFS's log is also **physical logging with a relogging discipline**: it logs the changed *regions* of metadata buffers, not logical operations. This makes recovery simple (replay overwrites) at the cost of log volume — which delayed logging then reduces. Note the shape: a simple, verifiable mechanism plus an optimisation layer, rather than a complex mechanism.

### T.7 Reverse mapping: the tree that enables everything modern

`rmapbt` maps **physical block → owner**:

```c
struct xfs_rmap_irec {
	xfs_agblock_t	rm_startblock;	/* physical */
	xfs_extlen_t	rm_blockcount;
	uint64_t	rm_owner;	/* inode number, or a special code */
	uint64_t	rm_offset;	/* offset within that file */
	unsigned int	rm_flags;	/* ATTR_FORK, BMBT_BLOCK, UNWRITTEN */
};
```

Special owners cover metadata: `XFS_RMAP_OWN_FS`, `OWN_LOG`, `OWN_AG`, `OWN_INOBT`, `OWN_INODES`, `OWN_REFC`. **Every single block in the filesystem has an rmap record.** Nothing is unaccounted.

Ext4 cannot answer "who owns block N?" without a full scan (Ch. 57 §T.9, demonstrated in Lab 57.6). XFS answers in O(log n). That capability unlocks four things:

| Capability | How rmap enables it |
|---|---|
| **Reflink / shared blocks** | you can find every owner of a block, so you know when it is truly free |
| **Online repair** | rebuild a damaged tree by scanning rmap, which is an independent record of the truth |
| **Meaningful error reports** | "media error at block N" becomes "file /foo/bar offset 4096 is damaged" |
| **Online defragmentation** | safely move a block knowing every reference |

The cost is substantial: another B+tree updated on every allocation and free, roughly 20–30 % more metadata I/O. XFS made the trade because the capabilities are worth it, and modern `mkfs.xfs` enables `rmapbt` by default.

**`refcountbt`** is the companion: a per-AG B+tree mapping block ranges to reference counts. A block with refcount > 1 is shared between files. Writing to a shared block triggers CoW: allocate new blocks (staged in the CoW fork, §T.5), write, then atomically remap.

Together these give XFS `reflink`, `copy_file_range` acceleration, `FIDEDUPERANGE` (deduplication), and snapshot-like workflows — without the whole-filesystem CoW that btrfs uses. **XFS is copy-on-write only where blocks are shared**, which is a genuinely interesting middle position: you get reflinks and dedup without paying CoW's fragmentation and write-amplification costs on ordinary files.

### T.8 The error philosophy: shut down

XFS's response to detected inconsistency is characteristic:

```c
	xfs_force_shutdown(mp, SHUTDOWN_CORRUPT_INCORE);
```

The filesystem stops. All operations return `EIO` (or `ESHUTDOWN`). You must unmount and repair. The messages are unmistakable:

```
XFS (sda1): Internal error xfs_trans_cancel at line 1021 of file fs/xfs/xfs_trans.c
XFS (sda1): Corruption detected. Unmount and run xfs_repair
XFS (sda1): xfs_do_force_shutdown(0x8) called from line 227
```

The reasoning: **continuing after detecting corruption makes things worse.** Every subsequent write based on bad metadata spreads the damage. Stopping preserves whatever is still correct and preserves the log for recovery.

Contrast ext4's `errors=` (Ch. 57 §1.2), which defaults to `remount-ro` and can be set to `continue`. XFS has no `continue` option, by design.

This is also why **XFS has no `fsck` at mount time**: `xfs_repair` is a separate, deliberate, offline operation. It does not run automatically, it requires the log to be clean (or `-L` to zero it, which loses data), and it is memory-hungry on large filesystems. The philosophy is that automatic repair of a filesystem you do not understand is dangerous.

**Online repair** (`xfs_scrub`, in development for years, increasingly usable) changes this. Because of rmapbt, XFS can verify every metadata structure against an independent source and rebuild damaged ones while mounted:

```sh
xfs_scrub -n /mnt    # check only
xfs_scrub /mnt       # check and repair online
```

This is the most ambitious filesystem project in the kernel and it is only possible because of §T.7. It is worth understanding as a demonstration that **redundant metadata is what makes repair possible** — you cannot rebuild a structure without an independent record of what it should contain.

Per-buffer **verifiers** support all of this: every metadata buffer type has read and write verify functions checking magic numbers, CRC32c, UUID, block number, and log sequence number. Corruption is caught at read time, at the buffer layer, before any code acts on it. The **self-describing metadata** format (v5, 2013) added the owner, block number, and UUID to every metadata block — so a block that has been relocated or belongs to a different filesystem fails verification even if its contents are internally consistent.

### T.9 Speculative preallocation and the "my file shrank" confusion

XFS aggressively preallocates beyond EOF on streaming writes, doubling up to `allocsize` (default: dynamic, up to 1 GiB). This produces excellent contiguity.

It also produces confused users: `du` shows more than the file's size, and the extra space disappears later. The reclaim rules:

- Freed on `close()` if the file is not being appended to.
- Freed by the background **`blockgc`** scanner after `speculative_prealloc_lifetime` (default 300 s).
- Freed on `ENOSPC` pressure, which is why XFS can return space to a filesystem that appears full.

```sh
xfs_io -c 'bmap -vp' file        # shows post-EOF preallocation
xfs_io -c 'fsync' -c 'bmap -vp' file
mount -o allocsize=64k ...       # bound it
```

The related confusion: **XFS `ENOSPC` reporting.** Because of per-CPU free-space counters, speculative preallocation, and reserved blocks, `df` on XFS is approximate and can disagree with reality in both directions. `xfs_info` and `xfs_db -c freesp` give the truth.

### T.10 What XFS cannot do

Honest counterpart to Ch. 57 §T.9:

| Missing | Why |
|---|---|
| **Data checksums** | same as ext4 — metadata only. Use dm-integrity or btrfs/ZFS. |
| **Shrink** | `xfs_growfs` grows only; shrinking requires relocating data across AGs with no mechanism for it. **This is XFS's most-cited limitation.** |
| **Snapshots** | reflink gives per-file CoW, not filesystem snapshots. Use LVM or dm-thin beneath. |
| **Compression** | no format support |
| **Built-in RAID/multi-device** | unlike btrfs/ZFS; use MD or dm |
| **Repair without unmounting (fully)** | `xfs_scrub` is getting there but is not complete |
| **Small-filesystem efficiency** | AG overhead and large metadata structures make XFS a poor choice below a few GiB |

The inability to shrink is a genuine operational problem and the most common reason people choose ext4 over XFS. It is not a small oversight: shrinking requires moving every extent in the removed AGs, updating every reference (which rmap now makes possible), and doing it atomically. It may eventually arrive; it has been "possible in principle" for a decade.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `fs/xfs/libxfs/` ★★★ | **shared with userspace `xfsprogs`** — the on-disk format code compiles into both the kernel and `xfs_repair`. An unusual and excellent arrangement. |
| `fs/xfs/libxfs/xfs_format.h` ★★★ | every on-disk structure |
| `fs/xfs/libxfs/xfs_btree.c` | the generic B+tree of §T.3 |
| `fs/xfs/libxfs/xfs_alloc.c`, `xfs_alloc_btree.c` | bnobt/cntbt |
| `fs/xfs/libxfs/xfs_ialloc.c`, `xfs_ialloc_btree.c` | inobt/finobt |
| `fs/xfs/libxfs/xfs_rmap.c`, `xfs_rmap_btree.c` | §T.7 |
| `fs/xfs/libxfs/xfs_refcount.c`, `xfs_refcount_btree.c` | reflink |
| `fs/xfs/libxfs/xfs_bmap.c` ★★★ | the per-inode block map; the largest and most important file |
| `fs/xfs/libxfs/xfs_dir2*.c` | directories, all four formats |
| `fs/xfs/xfs_log_cil.c` ★★★ | §T.6's delayed logging |
| `fs/xfs/xfs_log.c`, `xfs_log_recover.c` | the log and recovery |
| `fs/xfs/xfs_trans*.c` | the transaction layer |
| `fs/xfs/xfs_iomap.c` ★★★ | Ch. 55's reference `iomap_ops` implementation |
| `fs/xfs/xfs_buf.c` | the buffer cache and verifiers |
| `fs/xfs/scrub/` ★★★ | §T.8's online repair |
| `Documentation/filesystems/xfs/` | admin guide, self-describing metadata, online repair design |

The `libxfs` arrangement is worth a moment: the same source builds into the kernel module and into `xfs_repair`, `xfs_db`, and `mkfs.xfs`. So the repair tool's understanding of the format cannot drift from the kernel's. Ext4 has two independent implementations (kernel and `e2fsprogs`) and they have disagreed historically.

### 1.2 The superblock and AG headers

```c
struct xfs_sb {
	uint32_t	sb_magicnum;	/* XFS_SB_MAGIC = "XFSB" */
	uint32_t	sb_blocksize;
	xfs_rfsblock_t	sb_dblocks;	/* data blocks */
	xfs_rfsblock_t	sb_rblocks;	/* realtime blocks */
	xfs_rtblock_t	sb_rextents;
	uuid_t		sb_uuid;
	xfs_fsblock_t	sb_logstart;	/* internal log location */
	xfs_ino_t	sb_rootino;
	xfs_ino_t	sb_rbmino, sb_rsumino;
	xfs_agblock_t	sb_rextsize;
	xfs_agblock_t	sb_agblocks;	/* blocks per AG */
	xfs_agnumber_t	sb_agcount;	/* T.2 */
	xfs_extlen_t	sb_rbmblocks, sb_logblocks;
	uint16_t	sb_versionnum;
	uint16_t	sb_sectsize;
	uint16_t	sb_inodesize;	/* T.5 */
	uint16_t	sb_inopblock;
	char		sb_fname[XFSLABEL_MAX];
	uint8_t		sb_blocklog, sb_sectlog, sb_inodelog, sb_inopblog;
	uint8_t		sb_agblklog;	/* log2(agblocks), rounded up */
	uint8_t		sb_rextslog, sb_inprogress, sb_imax_pct;
	uint64_t	sb_icount, sb_ifree, sb_fdblocks, sb_frextents;
	...
	uint32_t	sb_features2, sb_bad_features2;
	uint32_t	sb_features_compat;	/* Ch. 56 T.3(d) */
	uint32_t	sb_features_ro_compat;
	uint32_t	sb_features_incompat;
	uint32_t	sb_features_log_incompat;
	uint32_t	sb_crc;			/* v5 */
	xfs_extlen_t	sb_spino_align;
	xfs_ino_t	sb_pquotino;
	xfs_lsn_t	sb_lsn;
	uuid_t		sb_meta_uuid;
};
```

`sb_agblklog` is the shift used to pack `(agno, agblock)` into a `xfs_fsblock_t` — §T.2(b) made concrete. Note `sb_features_log_incompat`: a fourth flag set, for features affecting *log format only*, so a filesystem with a dirty log can be recognised as unrecoverable by an old kernel while still being mountable after a clean unmount. Ext4's three sets became four here; the taxonomy keeps refining.

```c
struct xfs_agf {			/* free space header */
	__be32	agf_magicnum;		/* "XAGF" */
	__be32	agf_versionnum, agf_seqno, agf_length;
	__be32	agf_roots[XFS_BTNUM_AGF];   /* bnobt, cntbt, rmapbt roots */
	__be32	agf_levels[XFS_BTNUM_AGF];
	__be32	agf_flfirst, agf_fllast, agf_flcount;   /* the AGFL */
	__be32	agf_freeblks;
	__be32	agf_longest;		/* longest free extent: the key metric */
	__be32	agf_btreeblks;
	uuid_t	agf_uuid;
	__be32	agf_refcount_root, agf_refcount_level, agf_refcount_blocks;
	__be64	agf_rmap_blocks;
	...
};
```

`agf_longest` is the most operationally useful field in XFS: it is the largest contiguous free extent in this AG. When it drops, large allocations start fragmenting. `xfs_db -c 'agf 0'` shows it.

The **AGFL** (free list) solves a bootstrapping problem: allocating a block for a B+tree may require splitting the tree, which requires allocating a block. XFS keeps a small pre-allocated list of blocks specifically for B+tree growth, replenished outside of transactions. It is the same shape as Ch. 22's `mempool` or the emergency reserves in the page allocator: **a reserve that breaks the circular dependency.**

### 1.3 The inode

```c
struct xfs_dinode {
	__be16		di_magic;	/* "IN" */
	__be16		di_mode;
	__s8		di_version;
	__s8		di_format;	/* T.5: the DATA fork's format */
	__be16		di_onlink;
	__be32		di_uid, di_gid;
	__be32		di_nlink;
	__be16		di_projid_lo, di_projid_hi;
	__u8		di_pad[6];
	__be16		di_flushiter;
	xfs_timestamp_t	di_atime, di_mtime, di_ctime;
	__be64		di_size;
	__be64		di_nblocks;
	__be32		di_extsize;	/* extent size hint */
	__be32		di_nextents;	/* data fork extent count */
	__be16		di_anextents;	/* attr fork extent count */
	__u8		di_forkoff;	/* T.5: where the attr fork starts */
	__s8		di_aformat;	/* the ATTR fork's format */
	__be32		di_dmevmask;
	__be16		di_dmstate, di_flags;
	__be32		di_gen;		/* Ch. 56 T.8 */
	__be32		di_next_unlinked;
	/* v5 only: */
	__le32		di_crc;
	__be64		di_changecount, di_lsn;
	__be64		di_flags2;
	__be32		di_cowextsize;
	__u8		di_pad2[12];
	xfs_timestamp_t	di_crtime;
	__be64		di_ino;		/* self-identifying */
	uuid_t		di_uuid;	/* self-identifying */
	/* then the literal area: data fork, then attr fork at di_forkoff*8 */
};
```

`di_ino` and `di_uuid` in the inode itself are §T.8's self-describing metadata: an inode block read from the wrong location, or from a different filesystem, fails verification immediately.

`di_next_unlinked` implements the **unlinked inode list**: a file that is unlinked while still open is put on a per-AG singly-linked list rooted in the AGI. On recovery, XFS walks these lists and frees the inodes — the equivalent of ext4's orphan list, but embedded in the inode itself.

### 1.4 The generic B+tree

```c
struct xfs_btree_ops {
	size_t	key_len;
	size_t	rec_len;

	struct xfs_btree_cur *(*dup_cursor)(struct xfs_btree_cur *);
	void	(*set_root)(struct xfs_btree_cur *, const union xfs_btree_ptr *, int);
	int	(*alloc_block)(struct xfs_btree_cur *, const union xfs_btree_ptr *,
			       union xfs_btree_ptr *, int *);
	int	(*free_block)(struct xfs_btree_cur *, struct xfs_buf *);
	void	(*update_lastrec)(...);
	int	(*get_minrecs)(struct xfs_btree_cur *, int level);
	int	(*get_maxrecs)(struct xfs_btree_cur *, int level);
	void	(*init_key_from_rec)(union xfs_btree_key *, const union xfs_btree_rec *);
	void	(*init_rec_from_cur)(struct xfs_btree_cur *, union xfs_btree_rec *);
	int64_t	(*key_diff)(struct xfs_btree_cur *, const union xfs_btree_key *);
	const struct xfs_buf_ops *buf_ops;
	...
};
```

One implementation, seven instantiations. The `alloc_block`/`free_block` indirection is what lets the same code use the AGFL for AG-level trees and normal allocation for the per-inode bmbt.

`struct xfs_btree_cur` is a **cursor**: it holds the buffer and index at each level, so a lookup followed by an insert does not re-descend the tree. Range scans, increments, and decrements all work from the cursor. This is a database-style interface, and it is a large part of why XFS's tree code is reusable.

### 1.5 Delayed logging

```c
struct xfs_cil {
	struct xlog		*xc_log;
	struct list_head	xc_cil;		/* the items */
	spinlock_t		xc_cil_lock;
	struct rw_semaphore	xc_ctx_lock;
	struct xfs_cil_ctx	*xc_ctx;
	struct workqueue_struct	*xc_push_wq;
	wait_queue_head_t	xc_push_wait;
	struct list_head	xc_committing;
	...
};

/* The relogging core: if this item is already in the CIL, replace its
 * formatted content rather than adding a second copy. */
static void xlog_cil_insert_format_items(struct xlog *log,
					 struct xfs_trans *tp,
					 int *diff_len)
{
	struct xfs_log_item *lip;

	list_for_each_entry(lip, &tp->t_items, li_trans) {
		...
		if (!test_bit(XFS_LI_DIRTY, &lip->li_flags))
			continue;
		/* An existing lv is REPLACED -- this is the whole idea (T.6) */
		if (lip->li_lv && shadow->lv_size <= lip->li_lv->lv_size) {
			lv = lip->li_lv;
			...
		}
		lip->li_ops->iop_format(lip, lv);
	}
}
```

A push happens when the CIL exceeds `XLOG_CIL_SPACE_LIMIT` (1/8 of the log), on `sync`, or on a log force. The checkpoint is written as one large, ordered sequence.

Observe it:

```sh
trace-cmd record -e xfs:xfs_log_cil\* -e xfs:xfs_log_force
cat /sys/fs/xfs/*/log/*     # log statistics where exposed
```

### 1.6 Observability surface

| Where | What |
|---|---|
| `xfs_info MOUNTPOINT` ★★★ | geometry: AGs, block size, log, features |
| `xfs_db -r DEV` ★★★ | the full on-disk debugger |
| `xfs_bmap -vvp FILE` ★★★ | extents with AG numbers and flags |
| `xfs_spaceman -c 'freesp -s' MNT` ★★★ | free-space fragmentation histogram |
| `xfs_spaceman -c 'health' MNT` | per-AG health from scrub |
| `xfs_io` ★★★ | the Swiss army knife |
| `xfs_scrub -n MNT` | online consistency check |
| `xfs_repair -n DEV` | offline check |
| `xfs_quota`, `xfs_growfs`, `xfs_fsr` | quota, grow, defragment |
| `/sys/fs/xfs/DEV/` | error handling config, log stats |
| `/proc/fs/xfs/stat` ★★★ | detailed counters |
| `trace-cmd record -e xfs:\*` ★★★ | ~800 tracepoints — the most of any subsystem |

XFS has more tracepoints than any other filesystem by a wide margin. `xfs:xfs_alloc_*` alone covers every allocation decision path.

---

## 2. Practice

### Lab 58.1 — Geometry and allocation groups

```sh
sudo apt install -y xfsprogs
sudo modprobe scsi_debug dev_size_mb=8192
DEV=$(lsblk -ndo NAME,MODEL | awk '/scsi_debug/{print "/dev/"$1}')

sudo mkfs.xfs -f $DEV
sudo mkdir -p /mnt/xfs && sudo mount $DEV /mnt/xfs
xfs_info /mnt/xfs
```

Read every line of `xfs_info` against §T.2:

```
meta-data=/dev/sdb   isize=512    agcount=4, agsize=524288 blks
         =           sectsz=512   attr=2, projid32bit=1
         =           crc=1        finobt=1, sparse=1, rmapbt=1
         =           reflink=1    bigtime=1 inobtcount=1 nrext64=1
data     =           bsize=4096   blocks=2097152, imaxpct=25
naming   =version 2  bsize=4096   ascii-ci=0, ftype=1
log      =internal   bsize=4096   blocks=2560, version=2
realtime =none       extsz=4096   blocks=0, rtextents=0
```

- `agcount=4` × `agsize=524288 blks` × 4096 = 8 GiB. The AGs tile the device.
- `rmapbt=1` — §T.7 is on.
- `reflink=1` — refcountbt is on.
- `bigtime=1` — 64-bit timestamps (no 2038 problem, unlike a 128-byte-inode ext4).
- `nrext64=1` — 64-bit extent counters, lifting the old 2^31 extent limit.

Inspect an AG directly:

```sh
sudo umount /mnt/xfs
sudo xfs_db -r $DEV
```

```
xfs_db> sb 0
xfs_db> p
xfs_db> agf 0
xfs_db> p
      magicnum = 0x58414746          ("XAGF")
      versionnum = 1
      seqno = 0
      length = 524288
      bnoroot = 1
      cntroot = 2
      rmaproot = 5
      bnolevel = 1
      cntlevel = 1
      freeblks = 523229
      longest = 523229               <- T.1.2: the key metric
xfs_db> agi 0
xfs_db> p
      root = 3                       <- inobt
      free_root = 4                  <- finobt
      count = 64
      freecount = 61
xfs_db> agf 1
xfs_db> p                            <- INDEPENDENT trees (T.2)
xfs_db> quit
```

Vary the AG count and observe:

```sh
for n in 1 4 16 64; do
  sudo mkfs.xfs -f -d agcount=$n $DEV > /dev/null
  echo -n "agcount=$n: "
  sudo xfs_db -r -c 'sb 0' -c 'p agblocks agcount' $DEV | tr '\n' ' '
  echo
done
```

Now measure the parallelism claim:

```sh
sudo mkfs.xfs -f -d agcount=1 $DEV && sudo mount $DEV /mnt/xfs
echo "=== agcount=1 ==="
sudo fio --name=create --directory=/mnt/xfs --nrfiles=2000 --filesize=4k \
         --numjobs=8 --rw=write --create_only=1 --group_reporting 2>/dev/null | \
         grep -E 'iops|IOPS'
sudo umount /mnt/xfs

sudo mkfs.xfs -f -d agcount=32 $DEV && sudo mount $DEV /mnt/xfs
echo "=== agcount=32 ==="
sudo fio --name=create --directory=/mnt/xfs --nrfiles=2000 --filesize=4k \
         --numjobs=8 --rw=write --create_only=1 --group_reporting 2>/dev/null | \
         grep -E 'iops|IOPS'
```

Watch the AG spreading:

```sh
sudo trace-cmd record -e xfs:xfs_alloc_\* -- \
  sudo sh -c 'for i in $(seq 1 100); do dd if=/dev/zero of=/mnt/xfs/f$i bs=1M count=4 2>/dev/null; done'
sudo trace-cmd report | grep -oP 'agno \K\d+' | sort | uniq -c
# Allocations spread across AGs, not concentrated.
```

---

### Lab 58.2 — The two free-space trees

```sh
sudo mount $DEV /mnt/xfs

# Create a shredded free-space landscape
sudo sh -c 'for i in $(seq 1 3000); do
  dd if=/dev/zero of=/mnt/xfs/f$i bs=64k count=1 2>/dev/null; done'
sync
sudo sh -c 'for i in $(seq 2 2 3000); do rm /mnt/xfs/f$i; done'
sync

sudo xfs_spaceman -c 'freesp -s' /mnt/xfs
```

```
   from      to extents  blocks    pct
      1       1     742     742   0.03
      2       3    1498    3021   0.14
      4       7      12      63   0.00
   ...
```

That histogram comes straight from `cntbt` — the free-space-by-size tree, queried directly. Ext4 needs `e2freefrag` to build the equivalent by scanning bitmaps.

Now look at both trees:

```sh
sudo umount /mnt/xfs
sudo xfs_db -r $DEV
```

```
xfs_db> agf 0
xfs_db> addr bnoroot
xfs_db> p                    # free extents ordered by BLOCK NUMBER
xfs_db> agf 0
xfs_db> addr cntroot
xfs_db> p                    # the SAME extents ordered by LENGTH
xfs_db> quit
```

Seeing the same free extents indexed two ways is §T.3's core idea made concrete.

Demonstrate why both are needed:

```sh
sudo mount $DEV /mnt/xfs

# A large allocation: needs cntbt (find a big enough extent)
sudo trace-cmd record -e xfs:xfs_alloc_size\* -e xfs:xfs_alloc_near\* -- \
  sudo dd if=/dev/zero of=/mnt/xfs/large bs=1M count=200 2>/dev/null
sudo trace-cmd report | grep -c xfs_alloc_size
sudo trace-cmd report | grep -c xfs_alloc_near

# A small allocation near an existing file: needs bnobt
sudo trace-cmd record -e xfs:xfs_alloc_size\* -e xfs:xfs_alloc_near\* -- \
  sudo dd if=/dev/zero of=/mnt/xfs/f1 bs=4k count=1 seek=100 conv=notrunc 2>/dev/null
sudo trace-cmd report | grep -c xfs_alloc_near
```

`xfs_alloc_size_*` is the cntbt path; `xfs_alloc_near_*` is the bnobt path. Different requests, different trees.

---

### Lab 58.3 — Extents, forks, and the literal area

```sh
sudo mkfs.xfs -f $DEV && sudo mount $DEV /mnt/xfs
sudo dd if=/dev/urandom of=/mnt/xfs/big bs=1M count=500 2>/dev/null
sync

sudo xfs_bmap -vvp /mnt/xfs/big
```

```
 EXT: FILE-OFFSET      BLOCK-RANGE        AG AG-OFFSET        TOTAL FLAGS
   0: [0..1023999]:    1544..1545543       0 (1544..1545543) 1024000 00000
```

`xfs_bmap` shows the AG — ext4's `filefrag` cannot, because ext4 block numbers are global.

Fork format promotion:

```sh
# Short symlink: LOCAL (inline in the inode)
sudo ln -s short /mnt/xfs/slink
# Long symlink: EXTENTS
sudo ln -s "$(python3 -c 'print("/x"*200)')" /mnt/xfs/llink

for f in slink llink; do
  INO=$(stat -c %i /mnt/xfs/$f)
  echo "=== $f (inode $INO) ==="
  sudo xfs_db -r -c "inode $INO" -c 'p core.format core.nextents core.size' $DEV
done
```

Force the data fork through all three formats:

```sh
sudo mkdir /mnt/xfs/d
INO=$(stat -c %i /mnt/xfs/d)
for n in 2 20 200 5000; do
  sudo sh -c "cd /mnt/xfs/d && for i in \$(seq 1 $n); do : > g\$i; done"
  sync
  echo -n "n=$n: "
  sudo xfs_db -r -c "inode $INO" -c 'p core.format' $DEV
done
# local -> extents -> btree, as the directory grows (T.5)
```

The literal area split:

```sh
sudo touch /mnt/xfs/x
INO=$(stat -c %i /mnt/xfs/x)
sudo xfs_db -r -c "inode $INO" -c 'p core.forkoff core.aformat' $DEV

sudo setfattr -n user.a -v "$(python3 -c 'print("x"*100)')" /mnt/xfs/x
sudo xfs_db -r -c "inode $INO" -c 'p core.forkoff core.aformat core.anextents' $DEV
# forkoff moved: the attr fork took literal-area space from the data fork.
```

Inode size matters:

```sh
for isize in 256 512 1024; do
  sudo umount /mnt/xfs 2>/dev/null
  sudo mkfs.xfs -f -i size=$isize $DEV > /dev/null
  sudo mount $DEV /mnt/xfs
  # How many extents fit inline before the tree is needed?
  sudo python3 -c "
import os
fd=os.open('/mnt/xfs/frag', os.O_WRONLY|os.O_CREAT, 0o644)
for i in range(0, 200, 2): os.pwrite(fd, b'x'*4096, i*4096)
os.fsync(fd); os.close(fd)"
  INO=$(stat -c %i /mnt/xfs/frag)
  echo -n "isize=$isize: "
  sudo xfs_db -r -c "inode $INO" -c 'p core.format core.nextents' $DEV 2>/dev/null | tr '\n' ' '
  echo
  sudo rm -f /mnt/xfs/frag
done
```

Larger inodes keep more extents inline — §T.5's practical consequence.

---

### Lab 58.4 — Delayed logging, measured

```sh
sudo mkfs.xfs -f $DEV && sudo mount $DEV /mnt/xfs

# A metadata-heavy workload
cat > metabench.sh <<'EOF'
#!/bin/sh
D=$1; N=${2:-20000}
mkdir -p $D/bench
cd $D/bench
for i in $(seq 1 $N); do : > f$i; done
sync
for i in $(seq 1 $N); do rm f$i; done
sync
EOF
chmod +x metabench.sh

echo "=== default log ==="
sudo /usr/bin/time -f '%e s' ./metabench.sh /mnt/xfs

# Watch the CIL
sudo trace-cmd record -e xfs:xfs_log_cil\* -e xfs:xfs_log_force \
                      -e block:block_rq_issue \
  -- sudo ./metabench.sh /mnt/xfs 5000
sudo trace-cmd report | grep -c xlog_cil_push
sudo trace-cmd report | grep -c block_rq_issue
# Far more operations than log writes: that is relogging (T.6).
```

Log size is the tunable that matters:

```sh
for lsz in 2m 10m 64m 256m; do
  sudo umount /mnt/xfs 2>/dev/null
  sudo mkfs.xfs -f -l size=$lsz $DEV > /dev/null 2>&1 || continue
  sudo mount $DEV /mnt/xfs
  echo -n "log=$lsz: "
  sudo /usr/bin/time -f '%e s' ./metabench.sh /mnt/xfs 10000 2>&1 | tail -1
  sudo umount /mnt/xfs
done
```

A small log forces frequent checkpoints and defeats coalescing. Quantify it:

```sh
for lsz in 2m 256m; do
  sudo mkfs.xfs -f -l size=$lsz $DEV > /dev/null
  sudo mount $DEV /mnt/xfs
  sudo trace-cmd record -e xfs:xfs_log_cil_push -o /tmp/cil-$lsz.dat -- \
    sudo ./metabench.sh /mnt/xfs 10000 2>/dev/null
  echo -n "log=$lsz pushes: "
  sudo trace-cmd report -i /tmp/cil-$lsz.dat | grep -c cil_push
  sudo umount /mnt/xfs
done
```

Head-to-head with ext4 on metadata:

```sh
sudo mkfs.xfs -f $DEV  && sudo mount $DEV  /mnt/xfs
sudo mkfs.ext4 -qF $DEV2 && sudo mount $DEV2 /mnt/ext4

for M in /mnt/xfs /mnt/ext4; do
  echo "=== $M (single-threaded) ==="
  sudo /usr/bin/time -f '%e s' ./metabench.sh $M 20000 2>&1 | tail -1
done

for M in /mnt/xfs /mnt/ext4; do
  echo "=== $M (8 threads) ==="
  START=$(date +%s.%N)
  for t in 1 2 3 4 5 6 7 8; do
    sudo sh -c "mkdir -p $M/t$t; cd $M/t$t; for i in \$(seq 1 5000); do : > f\$i; done" &
  done
  wait; sync
  echo "$(echo "$(date +%s.%N) - $START" | bc) s"
  sudo rm -rf $M/t*
done
```

**The single-threaded numbers are close; the 8-thread numbers separate.** That is §T.2, measured.

---

### Lab 58.5 — Reverse mapping and reflink

```sh
sudo mkfs.xfs -f -m rmapbt=1,reflink=1 $DEV && sudo mount $DEV /mnt/xfs
xfs_info /mnt/xfs | grep -E 'rmapbt|reflink'

sudo dd if=/dev/urandom of=/mnt/xfs/orig bs=1M count=100 2>/dev/null
sync
```

The capability ext4 does not have:

```sh
BLK=$(sudo xfs_bmap -vvp /mnt/xfs/orig | awk 'NR==3{print $3}' | cut -d. -f1)
AG=$(sudo xfs_bmap -vvp /mnt/xfs/orig | awk 'NR==3{print $4}')
AGBLK=$(sudo xfs_bmap -vvp /mnt/xfs/orig | awk 'NR==3{print $5}' | tr -d '(' | cut -d. -f1)

sudo umount /mnt/xfs
time sudo xfs_db -r -c "agf $AG" -c "addr rmaproot" -c "p" $DEV | head -20
# O(log n). Compare Ch. 57 Lab 57.6's icheck, which is a full scan.
```

Reflink:

```sh
sudo mount $DEV /mnt/xfs
df -h /mnt/xfs

time sudo cp --reflink=always /mnt/xfs/orig /mnt/xfs/clone
df -h /mnt/xfs                    # unchanged

sudo xfs_bmap -vvp /mnt/xfs/orig  | head -4
sudo xfs_bmap -vvp /mnt/xfs/clone | head -4
# Identical physical extents.

# Compare with a real copy
time sudo cp --reflink=never /mnt/xfs/orig /mnt/xfs/realcopy
df -h /mnt/xfs
```

Watch CoW happen:

```sh
sudo trace-cmd record -e xfs:xfs_reflink\* -e xfs:xfs_cow\* -e xfs:xfs_bmap\* -- \
  sudo dd if=/dev/urandom of=/mnt/xfs/clone bs=4k count=1 seek=1000 conv=notrunc 2>/dev/null
sync
sudo trace-cmd report | head -30
# The CoW fork (T.5) is used to stage the new extent before remapping.

sudo xfs_bmap -vvp /mnt/xfs/clone | head -6
# The modified region has moved; the rest is still shared.
df -h /mnt/xfs                     # 4 KiB more used
```

Refcounts:

```sh
sudo umount /mnt/xfs
sudo xfs_db -r -c 'agf 0' -c 'addr refcntroot' -c 'p' $DEV | head -20
# Block ranges with refcount > 1.
```

Deduplication:

```sh
sudo mount $DEV /mnt/xfs
sudo cp --reflink=never /mnt/xfs/orig /mnt/xfs/dup
df -h /mnt/xfs
sudo apt install -y duperemove
sudo duperemove -dr /mnt/xfs
df -h /mnt/xfs                     # space reclaimed via FIDEDUPERANGE
```

Meaningful error reporting — the capability rmap enables:

```sh
# Inject a media error under the filesystem and see the report
sudo dmsetup create bad --table \
  "0 $(sudo blockdev --getsz $DEV) linear $DEV 0"
# (use dm-dust or dm-flakey to produce read errors at a specific block)
# XFS, with rmap, can say WHICH FILE is affected.
sudo xfs_scrub -n /mnt/xfs 2>&1 | head -20
```

---

### Lab 58.6 — `xfs_db`: read the filesystem by hand

```sh
sudo umount /mnt/xfs
sudo xfs_db -r $DEV
```

```
xfs_db> sb 0
xfs_db> p                            # the superblock
xfs_db> version                      # feature summary
xfs_db> frag                         # fragmentation factor
xfs_db> freesp -s                    # free-space histogram
xfs_db> agf 0
xfs_db> p
xfs_db> agi 0
xfs_db> p

xfs_db> sb 0
xfs_db> p rootino
xfs_db> inode 128                    # the root
xfs_db> p
xfs_db> p core.format core.nextents
xfs_db> bmap                         # the block map

xfs_db> addr u3.sfdir3               # short-form directory contents
xfs_db> p

xfs_db> blockget -n                  # build the block-ownership map
xfs_db> blockuse -n 1544             # who owns block 1544?
xfs_db> ncheck                       # inode -> path

xfs_db> agf 0
xfs_db> addr bnoroot
xfs_db> p
xfs_db> addr rmaproot
xfs_db> p                            # every block's owner

xfs_db> quit
```

`blockuse` versus ext4's `icheck` is the practical difference §T.7 makes.

Inspect the log:

```sh
sudo xfs_logprint -t $DEV | head -40
sudo xfs_logprint -c $DEV | head -40    # print the whole log
```

Self-describing metadata:

```sh
sudo xfs_db -r -c 'agf 0' -c 'p uuid' $DEV
sudo xfs_db -r -c 'sb 0' -c 'p uuid' $DEV
# Every metadata block carries the filesystem UUID (T.8).

# Prove a relocated block fails verification:
sudo xfs_db -x $DEV      # -x = writable, DANGEROUS on a real filesystem
# xfs_db> agf 0
# xfs_db> write uuid 00000000-0000-0000-0000-000000000000
# then mount: the verifier rejects it immediately.
```

---

### Lab 58.7 — Errors, shutdown, and repair

```sh
sudo mount $DEV /mnt/xfs
sudo cp -r /usr/share/doc /mnt/xfs/ 2>/dev/null
sync

# Error handling configuration
ls /sys/fs/xfs/$(basename $DEV)/error/
cat /sys/fs/xfs/$(basename $DEV)/error/metadata/EIO/max_retries
cat /sys/fs/xfs/$(basename $DEV)/error/metadata/EIO/retry_timeout_seconds
cat /sys/fs/xfs/$(basename $DEV)/error/fail_at_unmount
```

These control how long XFS retries a failing device before shutting down — important for transient SAN failures, where you want retries, versus permanently-dead devices, where you want fast failure.

Force a shutdown:

```sh
sudo xfs_io -x -c 'shutdown' /mnt/xfs
ls /mnt/xfs                        # EIO
dmesg | tail -5
sudo umount /mnt/xfs
sudo mount $DEV /mnt/xfs           # log recovery runs
dmesg | tail -5
```

Corrupt metadata and watch verification catch it:

```sh
sudo umount /mnt/xfs
# Damage an AGF
sudo xfs_db -x -c 'agf 1' -c 'write freeblks 999999999' $DEV
sudo mount $DEV /mnt/xfs
sudo dd if=/dev/zero of=/mnt/xfs/newfile bs=1M count=100 2>&1 | tail -2
dmesg | tail -10
# "Corruption detected. Unmount and run xfs_repair"
```

Repair:

```sh
sudo umount /mnt/xfs 2>/dev/null
sudo xfs_repair -n $DEV        # check only
sudo xfs_repair $DEV           # repair
sudo mount $DEV /mnt/xfs && ls /mnt/xfs
```

Online scrub:

```sh
sudo xfs_scrub -n -v /mnt/xfs
sudo xfs_spaceman -c 'health' /mnt/xfs
systemctl list-units 'xfs_scrub*'     # the periodic scrub timers
```

And the gap XFS shares with ext4:

```sh
# Corrupt FILE DATA
sudo sh -c 'echo "important" > /mnt/xfs/data'
sync
BLK=$(sudo xfs_bmap -vvp /mnt/xfs/data | awk 'NR==3{print $3}' | cut -d. -f1)
sudo umount /mnt/xfs
sudo dd if=/dev/urandom of=$DEV bs=4096 seek=$BLK count=1 conv=notrunc 2>/dev/null
sudo xfs_repair -n $DEV        # CLEAN
sudo mount $DEV /mnt/xfs
cat /mnt/xfs/data              # garbage, silently
```

**Same result as ext4 (Ch. 57 Lab 57.7).** Neither checksums data. Ch. 59 does.

---

### Lab 58.8 — Speculative preallocation and `ENOSPC` behaviour

```sh
sudo mkfs.xfs -f $DEV && sudo mount $DEV /mnt/xfs

# Watch preallocation grow
sudo sh -c 'exec 3> /mnt/xfs/stream
for i in $(seq 1 50); do
  dd if=/dev/zero bs=1M count=4 2>/dev/null >&3
  sync
done
exec 3>&-'

sudo xfs_bmap -vvp /mnt/xfs/stream | tail -5
ls -l /mnt/xfs/stream
du -h /mnt/xfs/stream              # MORE than ls shows (T.9)
```

Confirm it is post-EOF:

```sh
sudo xfs_io -c 'bmap -vp' /mnt/xfs/stream | grep -i 'flags\|hole'
# Look for an extent beyond i_size.
```

Bound it:

```sh
sudo mount -o remount,allocsize=64k /mnt/xfs
# repeat: preallocation is now capped
```

The background reclaimer:

```sh
cat /proc/sys/fs/xfs/speculative_prealloc_lifetime     # 300
echo 5 | sudo tee /proc/sys/fs/xfs/speculative_prealloc_lifetime
du -h /mnt/xfs/stream
sleep 10
sudo xfs_io -c fsync /mnt/xfs/stream
du -h /mnt/xfs/stream              # reclaimed
```

`ENOSPC` behaviour:

```sh
# Fill it
sudo dd if=/dev/zero of=/mnt/xfs/fill bs=1M 2>&1 | tail -2
df -h /mnt/xfs
sudo xfs_spaceman -c 'freesp -s' /mnt/xfs
# XFS returns preallocation on pressure -- df may recover space by itself.

sudo rm /mnt/xfs/fill
sync
df -h /mnt/xfs
```

Extent size hints, for workloads where you know the pattern:

```sh
sudo mkdir /mnt/xfs/hinted
sudo xfs_io -c 'extsize 1m' /mnt/xfs/hinted
sudo xfs_io -c 'extsize' /mnt/xfs/hinted

# Files created in this directory inherit the hint
sudo python3 -c "
import os
fd=os.open('/mnt/xfs/hinted/h', os.O_WRONLY|os.O_CREAT, 0o644)
for i in range(0, 200, 2): os.pwrite(fd, b'x'*4096, i*4096)
os.fsync(fd); os.close(fd)"
sudo xfs_bmap -vvp /mnt/xfs/hinted/h | head
# Allocations are 1 MiB-aligned: far less fragmentation than the default.
```

This is one of XFS's genuinely useful tuning knobs, and it is per-file/per-directory rather than global.

---

## 3. Mastery drills

1. State the precise difference between an ext4 block group and an XFS allocation group. Identify the single field in each filesystem's superblock that makes ext4's approach a scalability bottleneck.

2. XFS stores free space in two B+trees. Construct an allocation request best served by each, and compute the cost of serving it with only the other one.

3. Compute the metadata write amplification of one block allocation in XFS with `rmapbt=1,reflink=1` (list every tree updated) and compare to ext4. Then argue whether the trade is worth it.

4. `finobt` made file creation on large filesystems much faster. Derive the complexity before and after, and construct the filesystem state where the difference is greatest.

5. XFS inode numbers can exceed 32 bits. Compute the filesystem size at which this begins for 256-, 512-, and 1024-byte inodes, and explain exactly what `inode32` trades away.

6. The AGFL breaks a circular dependency. State the dependency precisely and name two other places in the kernel with the same structure and solution.

7. Delayed logging coalesces repeated changes to the same object. Construct the workload with maximum coalescing and the one with none, and predict the performance ratio.

8. Prove that the CIL size threshold (1/8 of the log) makes log size a first-order performance parameter. Measure it (Lab 58.4) and explain the shape of the curve.

9. `rmapbt` enables online repair. Explain precisely how a damaged `bnobt` can be rebuilt from rmap, and identify the one tree that cannot be rebuilt this way.

10. XFS's CoW fork keeps in-flight CoW extents separate from the data fork. Construct the failure that would occur without it, and state which of Ch. 25's patterns this is.

11. XFS shuts down on corruption; ext4 defaults to remount-ro and allows `continue`. Argue both positions, then state which you would choose for (a) a build server, (b) a database primary, (c) a laptop.

12. XFS cannot shrink. Enumerate everything that would have to happen to shrink by one AG, and identify which capabilities XFS now has that it lacked in 2010.

13. You are told XFS is slower than ext4 on a workload. Give the ordered diagnostic procedure and the five most likely causes, in order of probability.

---

## 4. Further reading

**Kernel and project documentation**

- `Documentation/filesystems/xfs/xfs-self-describing-metadata.rst` ★★★ — §T.8's design, by Dave Chinner. Excellent.
- `Documentation/filesystems/xfs/xfs-online-fsck-design.rst` ★★★ — Darrick Wong's design document for online repair. **Very long and one of the best pieces of technical writing in the kernel tree.** Read at least the first few sections.
- `Documentation/filesystems/xfs/xfs-delayed-logging-design.rst` ★★★ — §T.6 from the author.
- `Documentation/admin-guide/xfs.rst` — mount options and sysfs knobs.
- "XFS Algorithms & Data Structures" (the `xfsprogs` documentation, also at djwong.org) ★★★ — **the complete on-disk format specification.** The equivalent of ext4's `Documentation/filesystems/ext4/` and comparably good.
- `man 8 mkfs.xfs`, `man 8 xfs_db`, `man 8 xfs_repair`, `man 8 xfs_scrub`, `man 8 xfs_io`, `man 8 xfs_spaceman` ★★★

**Papers and primary sources**

- Sweeney, Doucette, Hu, Anderson, Nishimoto, Peck, "Scalability in the XFS File System," USENIX 1996 ★★★ — **the original design paper. Still the best explanation of why XFS is shaped this way.** Read it.
- Chinner, "XFS: the filesystem of the future?" (with Dave Chinner's LCA and Vault talks) ★★★
- Chinner, "Improving Metadata Performance By Reducing Journal Overhead" — delayed logging.
- Wong, "XFS Online Filesystem Repair" and the associated LSFMM talks ★★★
- Hellwig, "XFS: the big storage file system for Linux," USENIX ;login: 2009 — the Linux port's perspective.
- The `xfs` mailing list archives — unusually high signal, with Chinner and Wong explaining design decisions at length.

**LWN**

- "XFS: There and back ... and there again?" (Chinner) ★★★ — a superb retrospective on XFS's evolution and its future.
- "Delayed logging in XFS" (2010) ★★★
- "Reverse mapping for XFS" and "XFS reflink" coverage ★★★
- "Online filesystem checking for XFS" and the ongoing scrub coverage ★★★
- "The XFS filesystem and 2038" — `bigtime`
- "Btrfs and XFS: a comparison" threads
- LSFMM summaries mentioning XFS — annual, and consistently informative

**Source reading order**

1. The "XFS Algorithms & Data Structures" document, first. Do not start with the code.
2. `fs/xfs/libxfs/xfs_format.h` ★★★ — the structures.
3. `fs/xfs/libxfs/xfs_btree.c` and `xfs_btree.h` — the generic tree; understand the cursor.
4. `fs/xfs/libxfs/xfs_alloc.c`: `xfs_alloc_ag_vextent_near` and `_size` ★★★ — §T.3's two trees in use.
5. `fs/xfs/libxfs/xfs_bmap.c`: `xfs_bmapi_write`, `xfs_bmap_add_extent_hole_real` — the per-inode map.
6. `fs/xfs/xfs_iomap.c` ★★★ — the reference `iomap_ops` (Ch. 55).
7. `fs/xfs/xfs_log_cil.c` ★★★ — §T.6.
8. `fs/xfs/libxfs/xfs_rmap.c` and `xfs_refcount.c` — §T.7.
9. `fs/xfs/scrub/` — §T.8; start with `scrub/common.c` and `scrub/agheader.c`.

**Tools**

- `xfs_db` ★★★ — the on-disk debugger. Learn `sb`, `agf`, `agi`, `inode`, `bmap`, `addr`, `freesp`, `blockget`/`blockuse`, `frag`.
- `xfs_io` ★★★ — the single best filesystem exploration tool in existence: `pwrite`, `bmap`, `fiemap`, `fsync`, `falloc`, `fpunch`, `fcollapse`, `reflink`, `extsize`, `shutdown`, `chattr`, `chproj`.
- `xfs_bmap -vvp`, `xfs_spaceman -c 'freesp -s'`, `xfs_spaceman -c health` ★★★
- `xfs_info`, `xfs_growfs`, `xfs_fsr`, `xfs_quota`
- `xfs_scrub`, `xfs_repair -n`, `xfs_logprint`
- `trace-cmd record -e xfs:\*` ★★★ — ~800 tracepoints; nearly every question is answerable by tracing
- `/proc/fs/xfs/stat`, `/sys/fs/xfs/DEV/`
- `xfstests` — despite the name it tests all filesystems, but XFS is its home and coverage is deepest there

---

→ Next: [59-btrfs.md](59-btrfs.md)

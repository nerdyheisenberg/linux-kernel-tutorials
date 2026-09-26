# Chapter 60 — F2FS, ZFS, and log-structured design

> **Goal:** Understand the idea that "write everything sequentially, never overwrite" is a filesystem design in its own right — where it came from, why the original version failed, and how flash storage resurrected it. Understand F2FS as a filesystem designed for the FTL rather than against it, the segment/section/zone hierarchy, the NAT indirection that solves the wandering-tree problem, and cleaning as the fundamental cost. Then understand ZFS as the most complete storage design in existence — pooled storage, the ZIO pipeline, end-to-end checksums with Merkle trees, RAID-Z's solution to the write hole, and the ARC — and why it is not in the Linux kernel. By the end you can place any storage system in the design space Part 3 has mapped.

---

## Theory & First Principles

### T.0 — Start here: the device is lying about its geometry

Your SSD presents itself as an array of 4 KiB sectors you may overwrite freely. **Inside, it
cannot overwrite anything at all.**

```
  What the block layer sees:    a flat array of writable 4 KiB sectors

  What NAND actually is:        read  unit: 4 KiB page
                                write unit: 4 KiB page, ONCE after erase
                                ERASE unit: ~4 MiB BLOCK   <-- 1000x larger
```

To rewrite one 4 KiB sector, the device must: read the containing 4 MiB block, erase it
(slow, and wears it out), and write it back. **1000× write amplification** for a 4 KiB update.

So every SSD contains a **Flash Translation Layer** that quietly runs a log-structured
filesystem in firmware: writes go to fresh pages, a mapping table records where logical
sector X currently lives, and a garbage collector reclaims blocks whose pages are mostly
stale. **Your filesystem sits on top of another filesystem that it cannot see** (Ch. 70).

**F2FS's argument:** if the device is log-structured anyway, stop fighting it — make the
filesystem log-structured too, so the two layers agree instead of working against each other.

```
  +---------+--------+--------+--------+--------+--------+
  | metadata| segment| segment| segment| segment|  ....  |   2 MiB segments
  +---------+--------+--------+--------+--------+--------+
                ^
                | writes ALWAYS go to the head of an open segment
                | SIX active logs, separated by expected lifetime:
                |   hot/warm/cold  x  node/data
```

**The multi-log idea is the transferable one.** Mixing data with very different lifetimes in
one erase block guarantees garbage collection work later: the block cannot be erased until
*every* page in it is dead, so one long-lived page pins 4 MiB. Segregating by predicted
lifetime — directory metadata is hot, a downloaded video is cold — makes whole segments die
together. **Generational GC in a memory allocator is the same idea; so is `slab` cache
segregation (Ch. 11) and hot/cold page separation in the buddy allocator.**

**And the unavoidable cost of any log:**

> A log-structured design converts *random writes* into *sequential writes*, and in exchange
> acquires a **garbage collector**. The GC must run, it competes with foreground I/O, and it
> is the source of the latency tail. There is no log-structured design without this trade.

This is why F2FS has "foreground GC" (urgent, blocking) and "background GC" (idle-time), why
its worst-case latency is worse than ext4's, and why it still wins on flash for
small-random-write workloads — which is exactly the phone and embedded case it was built for.

**ZFS is the other half of this chapter, and it makes a different argument:** that *the
layering itself* is the bug.

| Traditional stack | ZFS |
|---|---|
| filesystem → LVM → MD RAID → block → device | one integrated pool |
| RAID knows blocks but not which are **used** | resilver copies only live data |
| RAID detects a *failed disk*, not a *wrong answer* | checksums detect corruption and **self-heal** from a mirror |
| `fsck` after a crash | CoW transaction groups — always consistent |

**The RAID-5 write hole (Ch. 67 §T.7) is the sharpest example.** A layered RAID cannot fix it,
because it does not know which stripes are in use or which copy is correct. ZFS's RAID-Z
makes every write a full-stripe write of variable width — **the hole cannot occur, by
construction**. That is an end-to-end argument (Ch. 62 §T.1) winning decisively.

**What integration costs:** ZFS is out-of-tree for licensing reasons, does not use the Linux
page cache (its own ARC — two caches, awkward interaction), has high memory requirements, and
historically could not shrink or easily reshape a vdev. **You cannot integrate and stay
modular; you pick one**, and that tension is the theme to carry into Ch. 66.

```bash
cat /sys/kernel/debug/f2fs/*/status      # segment usage, GC statistics
zpool status && zpool iostat -v 1
zfs get compression,checksum,recordsize tank/data
cat /proc/spl/kstat/zfs/arcstats | head  # ARC hit rate
```

---

### T.1 The log-structured idea

Rosenblum and Ousterhout's 1992 LFS paper began with an observation about hardware trends:

> Memory is getting cheaper, so caches are getting bigger, so **reads increasingly hit the cache**. Therefore disk traffic will become dominated by **writes**. Therefore the filesystem should be optimised for writing.

And the optimisation for writing on a seek-limited device is obvious: **write everything sequentially, in one place, never seeking.**

```
Traditional:  write inode (seek) → write bitmap (seek) → write data (seek) → write dir (seek)
Log-structured: append [inode | bitmap | data | dir entry] ────────► one sequential write
```

The consequences:

| Property | Effect |
|---|---|
| All writes are sequential appends | writes go at full streaming bandwidth |
| Nothing is overwritten in place | crash consistency is nearly free (like CoW, Ch. 59 §T.2) |
| Metadata and data are interleaved | locality is temporal, not spatial |
| **Free space must be reclaimed by cleaning** | this is the problem |

The last row is where LFS died. Once the log has wrapped, appending requires free segments, and free segments must be created by **cleaning**: read a segment, identify which blocks are still live, copy them to the head of the log, mark the segment free.

Cleaning costs read + write I/O proportional to the *liveness* of the segments cleaned. On a nearly-full filesystem with scattered live data, cleaning can consume most of the device's bandwidth. Seltzer's 1993 and 1995 measurements showed LFS performing *worse* than FFS on transaction-processing workloads for exactly this reason, and the resulting debate ("LFS vs FFS") was one of the sharper exchanges in systems research.

**The verdict in 1995: log-structured design is elegant but loses to cleaning costs.** LFS was not adopted.

### T.2 Flash resurrects it

Then NAND flash changed the hardware assumptions completely:

| Property | Rotating disk | NAND flash |
|---|---|---|
| Random read | slow (seek) | **same as sequential** |
| Random write | slow (seek) | **catastrophic** (read-modify-erase-write) |
| Erase granularity | n/a | **2–8 MiB erase block** |
| Write granularity | 512 B sector | 4–16 KiB page |
| **Overwrite in place** | possible | **impossible** — must erase first |
| Endurance | unlimited | 1,000–100,000 P/E cycles |

Flash cannot overwrite. To change a page you must erase its entire containing block (megabytes) and rewrite it. So **every flash device already runs a log-structured filesystem internally** — that is what the FTL (Flash Translation Layer) is:

```
Host LBA  --[FTL: a log-structured mapping]-->  physical NAND page
                     + garbage collection
                     + wear levelling
```

Which produces a perverse situation: **a conventional filesystem's careful in-place update discipline is translated by the FTL into a log-structured one anyway**, with the FTL guessing at what the filesystem meant. The result is:

- **Double garbage collection.** The filesystem thinks it is overwriting; the FTL copies live data to make room.
- **Write amplification.** One 4 KiB host write can become many KiB of NAND writes.
- **The FTL has no information.** It does not know which LBAs are free (hence `TRIM`/`discard`), which are hot, or which belong to the same file.

F2FS's premise: **if the device is going to be log-structured anyway, make the filesystem log-structured too, and align its log with the device's erase blocks.** Then the two layers cooperate instead of fighting.

### T.3 F2FS: log-structured, done for flash

Samsung's F2FS (Jaegeuk Kim, 2012) is LFS with four specific adaptations.

**(a) Flash-aware layout.** The address space is divided into a hierarchy matching flash geometry:

```
Segment  = 2 MiB (512 × 4 KiB blocks)   -- the cleaning unit
Section  = N segments                    -- aligned to the erase block
Zone     = M sections                    -- aligned to a flash channel/plane
```

`mkfs.f2fs -s <segs_per_sec> -z <secs_per_zone>` sets these. Aligning sections to the device's erase block is the single most important tuning decision: it means F2FS's cleaning unit is the device's erase unit, so cleaning a section lets the FTL erase a block with no copying.

**(b) The NAT: solving the wandering tree.** This is F2FS's key technical contribution.

In a pure log-structured filesystem, moving a data block requires updating its inode, which moves the inode, which requires updating the directory entry, which moves the directory block, which requires updating *its* inode… The update cascades to the root. This is the **wandering tree problem**, and it is the same amplification as btrfs's CoW-to-the-root (Ch. 59 §T.2) — but worse, because in LFS *every* write moves a block.

F2FS breaks the cascade with one level of indirection:

```
directory entry --> inode NUMBER (never changes)
                        |
                   NAT (Node Address Table): node id -> block address
                        |
                   the actual node block (moves freely)
```

**Node IDs are stable; node addresses are not.** Moving a node block updates one NAT entry. The NAT itself lives in a fixed area with two copies (updated alternately, checkpoint-selected), so it never wanders.

This is the same trick as a page table, a vnode layer, or any indirection: *add a level of naming to decouple identity from location.* Compare with btrfs, which does not do this and instead pays the CoW-to-root cost on every change.

Three node types share the NAT:

| Node | Contains |
|---|---|
| **inode node** | metadata + 923 direct pointers + 2 direct-node + 2 indirect-node + 1 double-indirect ids |
| **direct node** | 1018 data block addresses |
| **indirect node** | 1018 node ids |

Note that the inode node holds 923 inline pointers — a file up to ~3.6 MiB needs no separate node blocks at all.

**(c) Multi-head logging.** A single append point mixes hot metadata with cold data in the same segment, so cleaning a segment always finds a mixture. F2FS maintains **six open segments** simultaneously:

| Log | Contents |
|---|---|
| HOT_NODE | directory inodes |
| WARM_NODE | ordinary file inodes |
| COLD_NODE | indirect nodes |
| HOT_DATA | directory entries |
| WARM_DATA | ordinary file data |
| COLD_DATA | multimedia data, cold/GC'd data |

**Separating by expected lifetime means a segment tends to become entirely dead or entirely live**, so cleaning either frees it outright or has nothing to do. This is the classic answer to garbage-collection cost — the same insight as generational GC in language runtimes, and as the FTL's own hot/cold separation.

**(d) Adaptive logging.** Pure append-only writing fails badly when the filesystem is nearly full, because cleaning cannot keep up. F2FS switches modes:

| Mode | When | Behaviour |
|---|---|---|
| **LFS** | free space is plentiful | append to a clean segment; fully sequential |
| **SSR** (Slack Space Recycling) | free space is low | reuse invalid blocks *within* existing segments; random writes, but no cleaning |

SSR trades write sequentiality for avoiding cleaning. When the device is nearly full, random writes are cheaper than the read-copy-write of cleaning. The threshold is tunable (`min_ipu_util`, `min_fsync_blocks`).

**Checkpointing.** F2FS does not journal. It writes a **checkpoint** — a consistent snapshot of all the fixed-area metadata (NAT, SIT, SSA, orphan list) — to one of two checkpoint areas, alternately. A crash rolls back to the last checkpoint. `fsync` is handled by **roll-forward recovery**: data written after the checkpoint is found by following the "next block address" pointers in node blocks and replayed.

Two checkpoint areas updated alternately is the same idea as btrfs's multiple superblocks: **never destroy the only valid copy.**

### T.4 Cleaning: the cost that defines log-structured design

F2FS's cleaner (garbage collector) chooses a victim section, copies live blocks out, and frees it. Two policies:

| Policy | When | Victim selection |
|---|---|---|
| **Greedy** | foreground (on-demand, blocking) | fewest valid blocks — minimise copy cost now |
| **Cost-benefit** | background (idle) | `(1 - u) × age / (1 + u)` — balance cost against how long it will stay free |

The cost-benefit formula is Rosenblum's, unchanged from 1992: a segment that is old (its data has survived a long time, so it is probably cold and will stay valid) is a worse victim than a young one with the same utilisation, because cleaning the old one buys space that will be re-dirtied sooner.

**Foreground GC is the enemy.** It blocks the writer and produces latency spikes. Everything in F2FS's design — multi-head logging, background GC, adaptive logging, the `gc_urgent` modes — exists to avoid reaching the point where foreground GC is needed.

The general statement, which applies equally to the FTL, to ZFS, to btrfs, and to any language runtime's GC:

> **In any system that never overwrites, the fundamental cost is reclamation, and the fundamental optimisation is arranging for things that die together to be stored together.**

That single sentence is the main idea of this chapter.

### T.5 Zoned storage: the hardware admits it

The logical end point is **zoned block devices**: SMR (shingled magnetic recording) hard drives and ZNS (Zoned Namespace) NVMe SSDs that expose their write constraints directly.

```
Conventional device:  write anywhere, any order
Zoned device:         the LBA space is divided into ZONES
                      each zone must be written SEQUENTIALLY from its write pointer
                      to rewrite a zone, RESET it first
```

This is the FTL's internal reality, made visible. The trade is stark:

- **SMR:** shingled tracks overlap, so writing a track destroys the next one. Sequential-only writing gives ~25 % more capacity per platter.
- **ZNS:** the SSD exposes zones instead of hiding them. **The FTL's mapping table, over-provisioning, and garbage collector all disappear** — the host does that work with far better information. Result: lower cost (less DRAM on the drive), lower write amplification, and predictable latency.

The host must now do the work, which means the filesystem must be log-structured. Which filesystems support zoned devices?

| Filesystem | Zoned support |
|---|---|
| **F2FS** | yes — natural fit; `mkfs.f2fs -m` |
| **btrfs** | yes (5.12+) — CoW is already append-friendly |
| **ZFS** | in progress |
| **zonefs** | a trivial filesystem exposing one file per zone |
| ext4, XFS | **no** — in-place update is fundamental to both |

Plus `dm-zoned`, which presents a zoned device as conventional by doing the log-structuring in device-mapper — a compatibility shim with the usual costs.

Zoned storage is the clearest vindication of the log-structured idea: **the hardware has stopped pretending, and the filesystems that were designed for the pretence cannot follow.**

### T.6 ZFS: the other tradition

ZFS (Bonwick and Moore, Sun, 2001–2005) was designed with a completely different premise: **the entire storage stack is one system, and splitting it into filesystem / volume manager / RAID layers is the mistake.**

Compare:

```
Traditional:  filesystem  (knows about files, not redundancy)
              ────────────
              volume mgr  (knows about devices, not files)
              ────────────
              RAID        (knows about blocks, not what is live)
              ────────────
              disks

ZFS:          ZPL (POSIX layer) ─┐
              DMU (objects)      ├── all one system, all information available
              ZIO (pipeline)     │
              SPA (pooled storage)┘
              vdevs
```

What the layering costs, and what merging it buys:

| Problem with layering | ZFS's answer |
|---|---|
| RAID scrubs **every block**, including free space | ZFS knows what is live; scrubs only live data |
| RAID cannot tell which of two mismatched mirrors is correct | **checksums say which is correct** |
| The filesystem cannot ask for a different copy on error | ZIO retries the other mirror automatically |
| Volume sizes are fixed at creation | **pooled storage**: filesystems share free space |
| **The RAID5 write hole** | RAID-Z: variable-width stripes, every write a full stripe |

That third row is the crucial one. Consider a RAID1 mirror where the two copies differ. Block-level RAID has no way to know which is right — it usually just returns one. ZFS has a checksum stored *in the parent block*, so it knows which copy is correct, returns it, and repairs the other. **This capability is impossible without merging the layers.**

Six ZFS concepts worth knowing precisely:

**(a) Pooled storage.** A **zpool** is made of **vdevs** (mirror, raidz1/2/3, or single disks). Datasets (filesystems, volumes, snapshots) all draw from the pool's free space. No partitioning, no resizing, no "the /var volume is full while /home has 2 TB free."

**(b) Everything is an object, everything is a tree.** The DMU (Data Management Unit) provides transactional objects. Block pointers form a Merkle tree rooted at the **uberblock**:

```c
typedef struct blkptr {
	dva_t		blk_dva[3];	/* up to THREE copies (ditto blocks) */
	uint64_t	blk_prop;	/* size, compression, checksum type, type */
	uint64_t	blk_pad[2];
	uint64_t	blk_phys_birth;
	uint64_t	blk_birth;	/* the transaction group that wrote it */
	uint64_t	blk_fill;
	zio_cksum_t	blk_cksum;	/* 256-BIT CHECKSUM, stored in the PARENT */
} blkptr_t;
```

Two details carry enormous weight:

- **The checksum is in the parent, not with the data.** So verifying a block also verifies that you read the *right* block, and the chain of checksums from the uberblock down means the whole tree is authenticated. This is a Merkle tree, and it is strictly stronger than btrfs's self-identifying blocks (Ch. 59 §T.6) — an attacker or a failure cannot produce a consistent-looking subtree.
- **`blk_dva[3]`** — a block can have up to three independent copies at different addresses, *independent of RAID*. Metadata gets two by default, pool-wide metadata three. So critical metadata survives even on a single disk.

**(c) The ZIO pipeline.** Every I/O flows through a staged pipeline: compress → checksum → dedup → allocate → vdev (mirror/raidz) → issue → (on error) retry other copies → verify → decompress. Adding a feature means adding a stage. It is a genuinely well-factored design and a model worth studying independently of ZFS.

**(d) RAID-Z: the write hole, solved.** Traditional RAID5 has a fixed stripe width, so a partial-stripe write must read-modify-write parity — non-atomically. RAID-Z uses **variable-width stripes**: every write, of any size, becomes its own full stripe with its own parity.

```
RAID5:   [D][D][D][P]  fixed; partial writes need RMW; write hole
RAID-Z:  [D][D][P]     a 2-block write is a 3-wide stripe
         [D][D][D][D][P][P]  a 4-block write with raidz2 is a 6-wide stripe
```

Every write is a full-stripe write, so parity is always computed from data written in the same transaction. **There is no window.** The cost is that a random read may need to touch every disk in the vdev, so RAID-Z gives the IOPS of a single disk per vdev — which is why ZFS advice is "many small vdevs, not one wide one."

Btrfs's RAID5/6 has the write hole precisely because it kept fixed-width stripes (Ch. 59 §T.9). This is the single clearest example in Part 3 of one design solving a problem another did not.

**(e) The ARC.** ZFS does not use the page cache; it has its own **Adaptive Replacement Cache** (Megiddo & Modha, IBM, 2003), which maintains four lists — recently used, frequently used, and *ghost* lists of recently-evicted entries from each — and adapts the balance between recency and frequency based on which ghost list is being hit.

This is strictly better than LRU at resisting scan pollution (Ch. 52 §T.7's problem), and ZFS adopted it in 2005 — a decade before Linux's MGLRU addressed the same issue. The cost: ZFS's cache is separate from the page cache, so on Linux memory is managed by two independent policies that cannot see each other. `zfs_arc_max` exists because of this, and "ZFS uses all my RAM" complaints follow from it.

**(f) Transaction groups.** Writes accumulate in a **transaction group** (txg), committed every 5 seconds by writing the tree bottom-up and finally the uberblock. Identical in shape to btrfs's commit (Ch. 59 §1.3). `fsync` uses the **ZIL** (ZFS Intent Log), which can live on a separate fast device (a SLOG) — the same role as btrfs's log tree.

### T.7 Why ZFS is not in the Linux kernel

Purely a licensing question. ZFS is CDDL; Linux is GPLv2. Both are open source; both are "copyleft"; they are **mutually incompatible** because each imposes conditions the other forbids.

The positions:

- **Sun's intent** (per several Sun engineers) was reportedly to prevent exactly this — CDDL was chosen in part for GPL incompatibility.
- **The SFLC and Software Freedom Conservancy** hold that distributing a combined work (kernel + ZFS module) violates the GPL.
- **Canonical** ships ZFS as a kernel module, arguing that the module is a separate work.
- **Greg Kroah-Hartman** (2017): "my personal opinion is that any external kernel modules that do not have a GPLv2-compatible license are simply not allowed"; the kernel has since restricted some symbols (notably FPU save/restore) from non-GPL modules, which broke ZFS builds until OpenZFS worked around it.

The practical consequence: **OpenZFS is a genuinely excellent piece of engineering that most Linux users cannot easily use**, ships as DKMS, breaks on kernel updates, and cannot use GPL-only kernel interfaces.

This is worth thinking about as an engineering-management lesson, not just a legal one: **a licensing decision has kept the best-designed storage system of its generation out of the most widely-deployed kernel for twenty years.** Btrfs exists in large part as a response — Chris Mason started it at Oracle after working on ZFS-adjacent problems, explicitly to get ZFS-like capabilities under the GPL.

### T.8 The design space, mapped

Part 3 has now covered every major approach. Placing them:

| | Update | Crash consistency | Data integrity | Snapshots | Best for |
|---|---|---|---|---|---|
| **ext4** | in place | journal | metadata only | no | general, predictable, well-understood |
| **XFS** | in place | journal (CIL) | metadata only | reflink only | large, parallel, metadata-heavy |
| **btrfs** | CoW | CoW is the mechanism | **full** | **yes, O(1)** | workstation, backup, mismatched disks |
| **F2FS** | log | checkpoint + roll-forward | optional | no | flash, mobile, zoned |
| **ZFS** | CoW + log | CoW + ZIL | **full, Merkle** | **yes** | storage servers, if licensing permits |

And the two axes that actually matter:

**Axis 1: where does update-in-place happen?**

```
ext4/XFS ────────► btrfs/ZFS ────────► F2FS ────────► ZNS
in place           CoW per block       append-only    hardware enforces it
```

**Axis 2: where does the mapping/GC live?**

```
FTL hides it ──► filesystem cooperates ──► filesystem owns it
(ext4 on SSD)     (F2FS on SSD)            (F2FS/btrfs on ZNS)
```

Moving right on either axis means **more information available to the decision-maker** and **more responsibility**. That is the whole trade, and it recurs everywhere in systems design — the end-to-end argument (Ch. 00 §T.4) applied to storage.

### T.9 What to actually choose

An honest decision procedure:

| Situation | Choice | Why |
|---|---|---|
| Server, general purpose, predictable | **XFS** or **ext4** | boring, fast, extremely well-tested |
| Database | **XFS** | no fragmentation, best parallel metadata, predictable p99 |
| Very large filesystem (>50 TiB) | **XFS** | designed for it; ext4's `fsck` time becomes prohibitive |
| Laptop / workstation | **btrfs** | snapshots before updates, compression, checksums |
| Data you cannot afford to lose silently | **btrfs RAID1** or **ZFS** | only these detect bit rot |
| Backup target | **btrfs** or **ZFS** | `send`/`receive` incremental replication |
| Storage server, licensing permits | **ZFS** | the most complete design; ARC, RAID-Z, ditto blocks |
| Android / embedded flash | **F2FS** | designed for it; measurably better on eMMC/UFS |
| Zoned device (SMR/ZNS) | **F2FS** or **btrfs** | nothing else works |
| Small filesystem (<1 GiB) | **ext4** | XFS and btrfs have too much fixed overhead |
| Read-only image | **erofs** or **squashfs** | purpose-built; far smaller and faster |
| RAM | **tmpfs** | Ch. 56 §T.2 |

Note what is *not* on this list: "the newest one," "the one with the most features," or "whatever the distribution defaults to." **The right answer is determined by the workload and the failure modes you care about**, and Part 3's purpose has been to let you derive it rather than look it up.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `fs/f2fs/f2fs.h` ★★★ | every structure; start here |
| `include/linux/f2fs_fs.h` ★★★ | the on-disk format |
| `fs/f2fs/super.c` | mount, checkpoint validation |
| `fs/f2fs/node.c` ★★★ | §T.3(b)'s NAT |
| `fs/f2fs/segment.c` ★★★ | §T.3(c)'s multi-head logging, SSR, SIT |
| `fs/f2fs/gc.c` ★★★ | §T.4's cleaning |
| `fs/f2fs/checkpoint.c` | §T.3's checkpointing |
| `fs/f2fs/recovery.c` | roll-forward recovery |
| `fs/f2fs/data.c` | the I/O path |
| `fs/f2fs/compress.c` | per-file compression |
| `fs/zonefs/` | the trivial zoned filesystem |
| `block/blk-zoned.c` | §T.5's zone management |
| `drivers/md/dm-zoned-*.c` | the compatibility shim |
| `fs/btrfs/zoned.c` | btrfs's zoned support |
| `Documentation/filesystems/f2fs.rst` ★★★ | design and mount options |
| `Documentation/block/zoned.rst` ★★★ | §T.5 |
| *(ZFS is out of tree: `github.com/openzfs/zfs`)* | |

### 1.2 The F2FS layout

```
┌──────────┬──────┬──────┬──────┬──────┬──────────────────────────────┐
│ Super-   │ CP   │ SIT  │ NAT  │ SSA  │ Main Area                    │
│ block    │      │      │      │      │ (segments, 2 MiB each)       │
│ (2 copies│(2 cp)│(2 cp)│(2 cp)│      │                              │
└──────────┴──────┴──────┴──────┴──────┴──────────────────────────────┘
   fixed location metadata          the log
```

| Area | Contents |
|---|---|
| **CP** (Checkpoint) | two areas, alternately written: the consistency point |
| **SIT** (Segment Info Table) | per-segment: valid block count + validity bitmap |
| **NAT** (Node Address Table) | §T.3(b): node id → block address |
| **SSA** (Segment Summary Area) | per-block: which node/offset owns it — **the reverse map**, needed for cleaning |
| **Main Area** | the six logs of §T.3(c) |

The SSA is F2FS's equivalent of XFS's `rmapbt` (Ch. 58 §T.7), and it exists for the same reason: **you cannot move a block unless you can find its owner.** Cleaning moves blocks constantly, so the reverse map is mandatory, not optional.

Every fixed-area structure has two copies written alternately, selected by checkpoint version. Same principle as btrfs's superblocks and ZFS's uberblock array.

### 1.3 The node structure and the NAT

```c
struct f2fs_inode {
	__le16 i_mode;
	__u8   i_advise;
	__u8   i_inline;              /* inline data/dentry flags */
	__le32 i_uid, i_gid, i_links;
	__le64 i_size, i_blocks;
	__le64 i_atime, i_ctime, i_mtime;
	__le32 i_atime_nsec, i_ctime_nsec, i_mtime_nsec;
	__le32 i_generation;
	union {
		__le32 i_gc_failures;
		__le32 i_compr_blocks;
	};
	__le32 i_xattr_nid;
	__le32 i_flags;
	__le32 i_pino;                /* parent inode: for roll-forward recovery */
	__le32 i_namelen;
	__u8   i_name[F2FS_NAME_LEN];
	__u8   i_dir_level;
	struct f2fs_extent i_ext;     /* a cached extent, for read locality */
	union {
		struct { __le16 i_extra_isize, i_inline_xattr_size; ... };
		__le32 i_addr[DEF_ADDRS_PER_INODE];    /* 923 direct pointers */
	};
	__le32 i_nid[DEF_NIDS_PER_INODE];  /* 2 direct, 2 indirect, 1 double */
};

struct f2fs_nat_entry {
	__u8   version;
	__le32 ino;
	__le32 block_addr;      /* THE indirection of T.3(b) */
} __packed;
```

`i_name` and `i_pino` inside the inode look redundant — the directory already has them. They exist for **roll-forward recovery**: after a crash, F2FS walks node blocks written after the checkpoint, and a node block must be self-describing enough to reconstruct its directory entry without reading the directory.

The mapping chain:

```
file block  →  i_addr[] or a direct node  →  block address (may be NEW_ADDR/NULL_ADDR)
node id     →  NAT                        →  block address
```

Only NAT entries change when a node moves. **The wandering tree stops at the NAT.**

### 1.4 Multi-head logging

```c
enum {
	CURSEG_HOT_DATA = 0,	/* directory entries */
	CURSEG_WARM_DATA,	/* ordinary file data */
	CURSEG_COLD_DATA,	/* multimedia, GC'd data */
	CURSEG_HOT_NODE,	/* directory inodes */
	CURSEG_WARM_NODE,	/* file inodes */
	CURSEG_COLD_NODE,	/* indirect nodes */
	NR_PERSISTENT_LOG,
	CURSEG_COLD_DATA_PINNED = NR_PERSISTENT_LOG,
	CURSEG_ALL_DATA_ATGC,
	NO_CHECK_TYPE,
};

static int __get_segment_type_6(struct f2fs_io_info *fio)
{
	if (fio->type == DATA) {
		struct inode *inode = fio->page->mapping->host;

		if (is_inode_flag_set(inode, FI_ALIGNED_WRITE))
			return CURSEG_COLD_DATA_PINNED;
		if (page_private_gcing(fio->page))
			return CURSEG_COLD_DATA;         /* GC'd: survived once */
		if (file_is_cold(inode) || f2fs_need_compress_data(inode))
			return CURSEG_COLD_DATA;
		if (file_is_hot(inode) ||
		    is_inode_flag_set(inode, FI_HOT_DATA) ||
		    f2fs_is_cow_file(inode))
			return CURSEG_HOT_DATA;
		return f2fs_rw_hint_to_seg_type(F2FS_I_SB(inode),
						inode->i_write_hint);
	} else {
		if (IS_DNODE(fio->page))
			return is_cold_node(fio->page) ? CURSEG_WARM_NODE
						       : CURSEG_HOT_NODE;
		return CURSEG_COLD_NODE;
	}
}
```

Note `page_private_gcing` → `CURSEG_COLD_DATA`: **data that survived one cleaning is classified cold**, because survival predicts survival. This is exactly generational GC's hypothesis, applied to filesystem blocks.

Also note `f2fs_rw_hint_to_seg_type`: F2FS honours `RWH_WRITE_LIFE_*` hints from `fcntl(F_SET_RW_HINT)`. An application that knows its data's lifetime can tell the filesystem, which tells the device. Few applications do, but RocksDB and some databases do.

### 1.5 Cleaning

```c
static int get_victim_by_default(struct f2fs_sb_info *sbi,
				 unsigned int *result, int gc_type,
				 int type, char alloc_mode,
				 unsigned long long age)
{
	struct sit_info *sm = SIT_I(sbi);
	struct victim_sel_policy p;
	unsigned int secno, last_victim;
	...
	p.min_cost = get_max_cost(sbi, &p);

	while (1) {
		unsigned long cost, *dirty_bitmap = p.dirty_bitmap;
		unsigned int segno = find_next_bit(dirty_bitmap, ...);
		...
		cost = get_gc_cost(sbi, segno, &p);
		if (p.min_cost > cost) {
			p.min_segno = segno;
			p.min_cost = cost;
		}
		...
	}
	...
}

static inline unsigned int get_gc_cost(struct f2fs_sb_info *sbi,
				       unsigned int segno,
				       struct victim_sel_policy *p)
{
	if (p->alloc_mode == SSR)
		return get_seg_entry(sbi, segno)->ckpt_valid_blocks;

	if (p->gc_mode == GC_GREEDY)
		return get_valid_blocks(sbi, segno, true);   /* fewest live */
	else if (p->gc_mode == GC_CB)
		return get_cb_cost(sbi, segno);              /* cost-benefit */
	...
}

static unsigned int get_cb_cost(struct f2fs_sb_info *sbi, unsigned int segno)
{
	...
	u = (vblocks * 100) >> sbi->log_blocks_per_seg;

	/* Rosenblum's formula, unchanged since 1992 (T.4) */
	mtime = div_u64(mtime, usable_segs_per_sec);
	if (mtime < sit_i->min_mtime) sit_i->min_mtime = mtime;
	if (mtime > sit_i->max_mtime) sit_i->max_mtime = mtime;
	if (sit_i->max_mtime != sit_i->min_mtime)
		age = 100 - div64_u64(100 * (mtime - sit_i->min_mtime),
				      sit_i->max_mtime - sit_i->min_mtime);

	return UINT_MAX - ((100 * (100 - u) * age) / (100 + u));
}
```

Thirty-two years later, the formula is literally the same.

### 1.6 Observability

| Where | What |
|---|---|
| `/sys/fs/f2fs/DEV/` ★★★ | dozens of tunables: `gc_*`, `ipu_policy`, `min_*`, `discard_*` |
| `/proc/fs/f2fs/DEV/status` ★★★ | **live segment utilisation, GC stats, dirty counts** |
| `/proc/fs/f2fs/DEV/segment_info` | per-segment valid block counts |
| `/proc/fs/f2fs/DEV/victim_bits` | GC victim selection state |
| `dump.f2fs`, `fsck.f2fs`, `sload.f2fs` | offline tools |
| `trace-cmd record -e f2fs:\*` ★★★ | ~100 tracepoints |
| `blkzone report DEV` ★★★ | §T.5's zones |
| `zpool status/iostat`, `zdb`, `arc_summary` | ZFS |

`/proc/fs/f2fs/DEV/status` is exceptionally informative — it prints the utilisation of every log, GC counts, and the current mode, live. Lab 60.3 relies on it.

---

## 2. Practice

### Lab 60.1 — F2FS layout

```sh
sudo apt install -y f2fs-tools
sudo modprobe scsi_debug dev_size_mb=4096
DEV=$(lsblk -ndo NAME,MODEL | awk '/scsi_debug/{print "/dev/"$1}')

sudo mkfs.f2fs -f -l lab60 $DEV
```

Read the output against §1.2:

```
Info: Segments per section = 1
Info: Sections per zone = 1
Info: sector size = 512
Info: total sectors = 8388608 (4096 MB)
Info: zone aligned segment0 blkaddr: 512
Info: format successful
```

```sh
sudo dump.f2fs -d 1 $DEV 2>&1 | head -60
sudo mkdir -p /mnt/f2fs && sudo mount $DEV /mnt/f2fs
sudo cat /proc/fs/f2fs/$(basename $DEV)/status
```

```
=====[ partition info(sdb). #0, RW, CP: Good]=====
[SB: 1] [CP: 2] [SIT: 2] [NAT: 22] [SSA: 8] [MAIN: 2009(OverProv:34 Resv:26)]

Utilization: 0% (12 valid blocks, 1028064 discard blocks)
  - Node: 5 (Inode: 4, Other: 1)
  - Data: 7
  - Inline_xattr Inode: 0
  - Orphan/Append/Update Inode: 0, 0, 0

Main area: 2009 segs, 2009 secs 2009 zones
  - COLD  data: 2009, 2009, 2009
  - WARM  data: 1001, 1001, 1001
  - HOT   data: 1000, 1000, 1000
  - Dir   dnode: 999, 999, 999
  - File  dnode: 998, 998, 998
  - Indir nodes: 997, 997, 997
```

**The six logs of §T.3(c) are right there**, each with its own current segment. Watch them diverge:

```sh
sudo mkdir -p /mnt/f2fs/dir
sudo sh -c 'for i in $(seq 1 500); do echo data > /mnt/f2fs/dir/f$i; done'
sudo dd if=/dev/zero of=/mnt/f2fs/big bs=1M count=200 2>/dev/null
sync
sudo cat /proc/fs/f2fs/$(basename $DEV)/status | head -25
```

Directory data went to HOT_DATA, the big file to WARM_DATA, inodes to their own logs. Different segments advanced by different amounts.

Segment alignment:

```sh
for s in 1 2 4 8; do
  sudo umount /mnt/f2fs 2>/dev/null
  sudo mkfs.f2fs -f -s $s $DEV 2>&1 | grep -E 'Segments per section|Sections per zone'
done
# -s should match the device's erase block size (T.3(a)).
```

Inline data:

```sh
sudo mount $DEV /mnt/f2fs
echo "tiny" | sudo tee /mnt/f2fs/small > /dev/null
sync
stat -c '%s bytes, %b blocks' /mnt/f2fs/small    # 0 blocks: inline
sudo cat /proc/fs/f2fs/$(basename $DEV)/status | grep -i inline
```

---

### Lab 60.2 — Sequential writes, proven

```sh
sudo mkfs.f2fs -f $DEV && sudo mount $DEV /mnt/f2fs
sudo mkfs.ext4 -qF $DEV2 && sudo mount $DEV2 /mnt/ext4

# Random writes at the filesystem level
cat > randwrite.py <<'EOF'
import os, random, sys
path = sys.argv[1]
fd = os.open(path, os.O_RDWR | os.O_CREAT, 0o644)
os.ftruncate(fd, 200 * 1024 * 1024)
random.seed(42)
for _ in range(20000):
    os.pwrite(fd, b'x' * 4096, random.randrange(0, 200*1024*1024, 4096))
os.fsync(fd); os.close(fd)
EOF

for M in /mnt/f2fs /mnt/ext4; do
  echo "=== $M ==="
  sudo trace-cmd record -e block:block_rq_issue -o /tmp/$(basename $M).dat \
    -- sudo python3 randwrite.py $M/rw 2>/dev/null
  # Are the writes sequential on the device?
  sudo trace-cmd report -i /tmp/$(basename $M).dat 2>/dev/null | \
    grep block_rq_issue | grep -oP '\+ \K\d+' | head -200 > /tmp/$(basename $M).sectors
  python3 -c "
s = [int(x) for x in open('/tmp/$(basename $M).sectors')]
fwd = sum(1 for a, b in zip(s, s[1:]) if b > a)
print(f'  forward transitions: {fwd}/{len(s)-1} = {100*fwd/(len(s)-1):.0f}%')
"
done
```

F2FS turns random logical writes into (nearly) sequential physical writes. **That is §T.1's entire premise, measured.**

Visualise it:

```sh
sudo trace-cmd report -i /tmp/f2fs.dat | grep block_rq_issue | \
  awk '{print NR, $0}' | grep -oP '^\d+ .*\+ \K\d+' | head -500 > /tmp/f2fs.plot
sudo trace-cmd report -i /tmp/ext4.dat | grep block_rq_issue | \
  grep -oP '\+ \K\d+' | head -500 > /tmp/ext4.plot
# Plot sector vs time: f2fs is a staircase, ext4 is scatter.
gnuplot -e "set terminal dumb 100 25; plot '/tmp/f2fs.plot' with points title 'f2fs'"
gnuplot -e "set terminal dumb 100 25; plot '/tmp/ext4.plot' with points title 'ext4'"
```

Fragmentation, the cost of sequential writing:

```sh
sudo filefrag /mnt/f2fs/rw
sudo filefrag /mnt/ext4/rw
# F2FS's file is heavily fragmented LOGICALLY. On flash this does not
# matter for reads; on a rotating disk it would be catastrophic.
```

Read it back:

```sh
for M in /mnt/f2fs /mnt/ext4; do
  echo 3 | sudo tee /proc/sys/vm/drop_caches > /dev/null
  echo -n "$M sequential read: "
  sudo dd if=$M/rw of=/dev/null bs=1M 2>&1 | tail -1
done
# On a real SSD the difference is small; on scsi_debug (RAM) it is noise.
# The point is the WRITE pattern, not the read.
```

---

### Lab 60.3 — Watch cleaning happen

This is the lab that makes §T.4 real.

```sh
sudo umount /mnt/f2fs
sudo mkfs.f2fs -f -o 5 $DEV      # -o 5: only 5% overprovisioning, force GC sooner
sudo mount -o background_gc=on $DEV /mnt/f2fs

DN=$(basename $DEV)
watch_status() { sudo cat /proc/fs/f2fs/$DN/status | grep -E 'Utilization|GC calls|BG '; }

watch_status

# Fill it, then churn
sudo dd if=/dev/zero of=/mnt/f2fs/fill bs=1M count=3400 2>/dev/null
sync
watch_status

# Now create and delete repeatedly to fragment the free space
for round in $(seq 1 5); do
  sudo sh -c 'for i in $(seq 1 400); do dd if=/dev/zero of=/mnt/f2fs/t$i bs=1M count=2 2>/dev/null; done'
  sync
  sudo sh -c 'for i in $(seq 2 2 400); do rm /mnt/f2fs/t$i; done'
  sync
  echo "--- round $round ---"
  watch_status
done
```

```
GC calls: 47 (BG: 31)
  - data segments : 22 (18)
  - node segments : 25 (13)
Try to move 1204 blocks (BG: 891)
  - data blocks : 612 (455)
  - node blocks : 592 (436)
```

**Every "moved block" is I/O you did not ask for.** That is the cleaning cost.

Force foreground GC and measure the latency spike:

```sh
cat /sys/fs/f2fs/$DN/gc_urgent
echo 1 | sudo tee /sys/fs/f2fs/$DN/gc_urgent      # urgent mode

sudo trace-cmd record -e f2fs:f2fs_gc\* -e f2fs:f2fs_get_victim -- \
  sudo dd if=/dev/zero of=/mnt/f2fs/forcegc bs=1M count=200 2>/dev/null
sudo trace-cmd report | head -30
echo 0 | sudo tee /sys/fs/f2fs/$DN/gc_urgent
```

Latency distribution with and without GC pressure:

```sh
sudo bpftrace -e '
tracepoint:f2fs:f2fs_gc_begin { @gc_start[tid] = nsecs; }
tracepoint:f2fs:f2fs_gc_end /@gc_start[tid]/ {
	@gc_duration_us = hist((nsecs - @gc_start[tid]) / 1000);
	delete(@gc_start[tid]);
}
tracepoint:syscalls:sys_enter_write { @w[tid] = nsecs; }
tracepoint:syscalls:sys_exit_write /@w[tid]/ {
	@write_us = hist((nsecs - @w[tid]) / 1000); delete(@w[tid]);
}' &

sudo dd if=/dev/zero of=/mnt/f2fs/gcbench bs=4k count=100000 2>/dev/null
```

The write-latency histogram will have a long tail exactly where GC ran.

Compare GC policies:

```sh
cat /sys/fs/f2fs/$DN/gc_idle       # 0=cost-benefit, 1=greedy, 2=age-threshold
for policy in 0 1 2; do
  echo $policy | sudo tee /sys/fs/f2fs/$DN/gc_idle > /dev/null
  echo "=== gc_idle=$policy ==="
  # reset counters by remounting, then churn and compare "Try to move"
done
```

Adaptive logging (LFS vs SSR, §T.3(d)):

```sh
cat /sys/fs/f2fs/$DN/ipu_policy     # in-place-update policy bitmask
cat /sys/fs/f2fs/$DN/min_ipu_util   # utilisation % above which SSR kicks in

sudo trace-cmd record -e f2fs:f2fs_submit_page_bio -- \
  sudo python3 randwrite.py /mnt/f2fs/ssr 2>/dev/null
# At high utilisation, writes become in-place (SSR) rather than appended.
sudo cat /proc/fs/f2fs/$DN/status | grep -i 'SSR\|utilization'
```

---

### Lab 60.4 — Zoned block devices

```sh
# null_blk can emulate a zoned device
sudo modprobe null_blk nr_devices=0
cd /sys/kernel/config/nullb
sudo mkdir zoned0 && cd zoned0
echo 1 | sudo tee zoned > /dev/null
echo 128 | sudo tee zone_size > /dev/null        # 128 MiB zones
echo 0  | sudo tee zone_nr_conv > /dev/null      # no conventional zones
echo 4096 | sudo tee size > /dev/null            # 4 GiB
echo 1 | sudo tee memory_backed > /dev/null
echo 1 | sudo tee power > /dev/null
cd /

ZDEV=/dev/nullb0
sudo blkzone report $ZDEV | head -10
```

```
  start: 0x000000000, len 0x040000, cap 0x040000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(EM) [type: 2(SEQ_WRITE_REQUIRED)]
  start: 0x000040000, len 0x040000, cap 0x040000, wptr 0x000000 reset:0 non-seq:0, zcond: 1(EM) [type: 2(SEQ_WRITE_REQUIRED)]
```

Every zone has a **write pointer**. Prove the constraint:

```sh
cat /sys/block/nullb0/queue/zoned
cat /sys/block/nullb0/queue/nr_zones
cat /sys/block/nullb0/queue/chunk_sectors
cat /sys/block/nullb0/queue/max_open_zones

# Sequential write: fine
sudo dd if=/dev/zero of=$ZDEV bs=1M count=10 oflag=direct 2>&1 | tail -1
sudo blkzone report -o 0 -c 1 $ZDEV     # wptr has advanced

# Random write: REFUSED
sudo dd if=/dev/zero of=$ZDEV bs=1M count=1 seek=50 oflag=direct 2>&1 | tail -2
# "Invalid argument" or I/O error: you cannot write behind the pointer.

# Reset the zone to rewrite it
sudo blkzone reset -o 0 -c 1 $ZDEV
sudo blkzone report -o 0 -c 1 $ZDEV     # wptr back to 0
```

Now put filesystems on it:

```sh
# ext4: refuses
sudo mkfs.ext4 -F $ZDEV 2>&1 | tail -3

# XFS: refuses
sudo mkfs.xfs -f $ZDEV 2>&1 | tail -3

# F2FS: works
sudo mkfs.f2fs -f -m $ZDEV 2>&1 | tail -5
sudo mkdir -p /mnt/zoned && sudo mount $ZDEV /mnt/zoned
sudo dd if=/dev/urandom of=/mnt/zoned/data bs=1M count=500 2>/dev/null
sync
df -h /mnt/zoned
sudo blkzone report $ZDEV | head -8     # write pointers advancing in order
```

**ext4 and XFS cannot exist on this device.** That is §T.5's point, demonstrated in three commands.

btrfs on zoned:

```sh
sudo umount /mnt/zoned
sudo mkfs.btrfs -f -d single -m single $ZDEV
sudo mount $ZDEV /mnt/zoned
sudo btrfs filesystem usage /mnt/zoned
sudo dd if=/dev/urandom of=/mnt/zoned/data bs=1M count=500 2>/dev/null; sync
sudo blkzone report $ZDEV | head -8
```

`zonefs` — the minimal approach:

```sh
sudo umount /mnt/zoned
sudo mkzonefs -f $ZDEV
sudo mount -t zonefs $ZDEV /mnt/zoned
ls -l /mnt/zoned/
ls -l /mnt/zoned/seq/ | head
# One file per zone. Append-only. That is the entire filesystem.
sudo dd if=/dev/zero of=/mnt/zoned/seq/0 bs=1M count=10 conv=notrunc oflag=direct
ls -l /mnt/zoned/seq/0
```

`dm-zoned` — the compatibility shim:

```sh
sudo umount /mnt/zoned
sudo dmzadm --format $ZDEV
sudo dmsetup create dmz --table "0 $(sudo blockdev --getsz $ZDEV) zoned $ZDEV"
sudo mkfs.ext4 -F /dev/mapper/dmz       # now ext4 works
sudo mount /dev/mapper/dmz /mnt/zoned
# ...at the cost of dm-zoned doing the log-structuring, with less information.
```

---

### Lab 60.5 — F2FS versus ext4 on realistic flash workloads

```sh
sudo umount /mnt/f2fs /mnt/ext4 2>/dev/null
sudo mkfs.f2fs -f $DEV  && sudo mount $DEV  /mnt/f2fs
sudo mkfs.ext4 -qF $DEV2 && sudo mount $DEV2 /mnt/ext4

# SQLite-like: small random writes with frequent fsync (the mobile workload)
cat > sqlitebench.c <<'EOF'
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <unistd.h>
int main(int argc, char **argv) {
	int fd = open(argv[1], O_RDWR | O_CREAT | O_TRUNC, 0644), i;
	char buf[4096] = {0};
	struct timespec a, b;
	ftruncate(fd, 64 * 1024 * 1024);
	srandom(42);
	clock_gettime(CLOCK_MONOTONIC, &a);
	for (i = 0; i < 3000; i++) {
		pwrite(fd, buf, 4096, (random() % 16384) * 4096);
		fdatasync(fd);
	}
	clock_gettime(CLOCK_MONOTONIC, &b);
	printf("%.0f us per write+fdatasync\n",
	       ((b.tv_sec-a.tv_sec)*1e6 + (b.tv_nsec-a.tv_nsec)/1e3) / 3000);
	close(fd);
	return 0;
}
EOF
gcc -O2 -o sqlitebench sqlitebench.c

for M in /mnt/f2fs /mnt/ext4; do
  echo -n "$M: "; sudo ./sqlitebench $M/db
done
```

Measure the write amplification, which is the number that matters on flash:

```sh
for M in /mnt/f2fs /mnt/ext4; do
  D=$(findmnt -no SOURCE $M | xargs basename)
  BEFORE=$(awk "/$D /{print \$10}" /proc/diskstats)
  sudo ./sqlitebench $M/db > /dev/null
  sync
  AFTER=$(awk "/$D /{print \$10}" /proc/diskstats)
  WRITTEN=$(( (AFTER - BEFORE) * 512 ))
  echo "$M: app wrote $((3000*4096)) bytes, device wrote $WRITTEN bytes, WA = $(echo "scale=2; $WRITTEN / (3000*4096)" | bc)"
done
```

**Write amplification is the metric that determines flash lifetime.** A WA of 4 means your drive dies four times sooner.

Mobile-like mixed workload:

```sh
for M in /mnt/f2fs /mnt/ext4; do
  echo "=== $M ==="
  sudo fio --name=mobile --directory=$M --size=256m --numjobs=4 \
           --rw=randrw --rwmixread=70 --bs=4k --fsync=16 \
           --runtime=30 --time_based --group_reporting 2>/dev/null | \
    grep -E 'read:|write:|lat.*99'
done
```

F2FS's compression:

```sh
sudo umount /mnt/f2fs
sudo mkfs.f2fs -f -O extra_attr,compression $DEV
sudo mount -o compress_algorithm=zstd,compress_extension=txt,compress_extension=log $DEV /mnt/f2fs

sudo cp /usr/share/doc/bash/README /mnt/f2fs/test.txt 2>/dev/null || \
  sudo sh -c 'yes "compressible text line" | head -100000 > /mnt/f2fs/test.txt'
sync
ls -l /mnt/f2fs/test.txt
du -h /mnt/f2fs/test.txt              # less than ls shows
sudo cat /proc/fs/f2fs/$(basename $DEV)/status | grep -i compr
```

---

### Lab 60.6 — ZFS on Linux

```sh
sudo apt install -y zfsutils-linux
# (or on RHEL-family, the OpenZFS repo; this is DKMS, per T.7)
modinfo zfs | grep -E '^license|^version'
# license: CDDL     <- T.7
```

Pooled storage:

```sh
sudo modprobe -r scsi_debug; sudo modprobe scsi_debug dev_size_mb=1024 num_tgts=4
DEVS=$(lsblk -ndo NAME,MODEL | awk '/scsi_debug/{print "/dev/"$1}' | tr '\n' ' ')
echo $DEVS

sudo zpool create -f tank mirror $(echo $DEVS | cut -d' ' -f1-2) \
                          mirror $(echo $DEVS | cut -d' ' -f3-4)
sudo zpool status tank
sudo zpool list
sudo zfs list
```

Datasets share the pool — no partitioning:

```sh
sudo zfs create tank/home
sudo zfs create tank/var
sudo zfs create tank/backup
sudo zfs list
# All three show the SAME available space. T.6(a).

sudo zfs set quota=100M tank/var
sudo zfs set reservation=200M tank/home
sudo zfs list -o name,used,avail,quota,reservation
```

Checksums and self-healing — the same test as Ch. 59 Lab 59.4:

```sh
sudo dd if=/dev/urandom of=/tank/home/critical bs=1M count=50 2>/dev/null
sync
sudo md5sum /tank/home/critical | tee /tmp/orig.md5

sudo zpool export tank
# Corrupt one side of a mirror
sudo dd if=/dev/urandom of=$(echo $DEVS | cut -d' ' -f1) bs=1M seek=100 count=20 2>/dev/null
sudo zpool import tank

sudo md5sum -c /tmp/orig.md5        # PASSES: repaired from the mirror
sudo zpool status -v tank
#   read  write  cksum
#      0      0     47      <- detected and corrected
sudo zpool scrub tank
sudo zpool status tank
```

Ditto blocks (§T.6(b)) — redundancy *within* a single disk:

```sh
sudo zpool create -f single $(echo $DEVS | cut -d' ' -f1)
sudo zfs set copies=2 single
sudo zfs get copies single
sudo dd if=/dev/urandom of=/single/important bs=1M count=10 2>/dev/null
sync
sudo zdb -dddd single 2>/dev/null | grep -A3 'Object.*ZFS plain file' | head
# Each block has TWO DVAs: two physical copies on ONE disk.
```

RAID-Z and the write hole (§T.6(d)):

```sh
sudo zpool destroy single tank 2>/dev/null
sudo zpool create -f raidz raidz1 $DEVS
sudo zpool status raidz

# Variable-width stripes: write different sizes and inspect
for sz in 4k 16k 64k 1m; do
  sudo dd if=/dev/urandom of=/raidz/f$sz bs=$sz count=1 2>/dev/null
done
sync
sudo zdb -dddd raidz 2>/dev/null | grep -E 'DVA|ASIZE' | head -20
# Each record's stripe width depends on its SIZE. There is no
# partial-stripe write, so there is no write hole.
```

The ARC (§T.6(e)):

```sh
sudo apt install -y zfs-zed
arc_summary | head -40
cat /proc/spl/kstat/zfs/arcstats | grep -E '^(size|c_max|c_min|hits|misses|mfu_hits|mru_hits|mfu_ghost_hits|mru_ghost_hits)'

# Watch it adapt: a scan should NOT evict the frequently-used set
sudo dd if=/dev/urandom of=/raidz/hot bs=1M count=100 2>/dev/null
for i in $(seq 1 20); do sudo cat /raidz/hot > /dev/null; done   # make it frequent
grep -E '^(mfu_size|mru_size)' /proc/spl/kstat/zfs/arcstats

sudo dd if=/dev/urandom of=/raidz/scan bs=1M count=2000 2>/dev/null
sudo cat /raidz/scan > /dev/null                                  # one-time scan
grep -E '^(mfu_size|mru_size|mfu_ghost_hits|mru_ghost_hits)' /proc/spl/kstat/zfs/arcstats
# The MFU set survives. Compare Ch. 52 T.7's scan-resistance problem.
```

Snapshots, compression, send/receive:

```sh
sudo zfs set compression=lz4 raidz
sudo zfs snapshot raidz@snap1
sudo zfs list -t snapshot
sudo dd if=/dev/urandom of=/raidz/new bs=1M count=50 2>/dev/null
sudo zfs snapshot raidz@snap2
sudo zfs send -i raidz@snap1 raidz@snap2 | wc -c     # incremental
sudo zfs rollback raidz@snap1
ls /raidz/
```

`zdb` — the equivalent of `btrfs dump-tree`:

```sh
sudo zdb -C raidz              # pool configuration
sudo zdb -d raidz              # datasets
sudo zdb -dddd raidz 1         # object 1, in detail
sudo zdb -uuu raidz            # the UBERBLOCK -- T.6(b)'s root
sudo zdb -mmm raidz            # metaslabs (the allocator)
sudo zdb -bbb raidz            # block statistics: a full traversal
```

Clean up:

```sh
sudo zpool destroy raidz
```

---

### Lab 60.7 — Build the comparison table yourself

Run the same battery against all five filesystems and fill in §T.8 from your own data.

```sh
cat > compare.sh <<'SCRIPT'
#!/bin/bash
# usage: compare.sh <mountpoint> <label>
M=$1; L=$2
D=$(findmnt -no SOURCE $M | xargs basename)

metric() { printf "  %-28s %s\n" "$1" "$2"; }

echo "=== $L ==="

# 1. Sequential write
R=$(sudo dd if=/dev/zero of=$M/seq bs=1M count=500 conv=fsync 2>&1 | tail -1 | grep -oP '[\d.]+ [GM]B/s')
metric "sequential write" "$R"

# 2. Random write + fsync
sudo ./sqlitebench $M/db 2>/dev/null | sed 's/^/  random write+fsync: /'

# 3. Metadata: create 20000 files
sudo rm -rf $M/meta; sudo mkdir $M/meta
T=$( { /usr/bin/time -f '%e' sudo sh -c "cd $M/meta && for i in \$(seq 1 20000); do : > f\$i; done; sync"; } 2>&1 )
metric "create 20000 files" "${T}s"

# 4. Metadata: delete them
T=$( { /usr/bin/time -f '%e' sudo sh -c "rm -rf $M/meta; sync"; } 2>&1 )
metric "delete 20000 files" "${T}s"

# 5. Write amplification
B=$(awk "/$D /{print \$10}" /proc/diskstats)
sudo ./sqlitebench $M/db2 > /dev/null 2>&1; sync
A=$(awk "/$D /{print \$10}" /proc/diskstats)
metric "write amplification" "$(echo "scale=2; ($A-$B)*512 / (3000*4096)" | bc)"

# 6. Space overhead
metric "space for 500MB file" "$(du -sh $M/seq 2>/dev/null | cut -f1)"

# 7. Fragmentation
metric "extents after random writes" "$(sudo filefrag $M/db 2>/dev/null | grep -oP '\d+(?= extent)')"

# 8. Does it detect data corruption?
sudo sh -c "echo CANARY > $M/canary"; sync
metric "data checksums" "$(test -n "$(sudo lsattr $M/canary 2>/dev/null)" && echo '(test manually)' || echo '(test manually)')"

sudo rm -f $M/seq $M/db $M/db2 $M/canary
SCRIPT
chmod +x compare.sh

sudo umount /mnt/* 2>/dev/null
for fs in ext4 xfs btrfs f2fs; do
  sudo umount /mnt/test 2>/dev/null
  sudo mkfs.$fs -f $DEV > /dev/null 2>&1 || sudo mkfs.$fs -F $DEV > /dev/null 2>&1
  sudo mkdir -p /mnt/test && sudo mount $DEV /mnt/test
  ./compare.sh /mnt/test $fs
done
```

Then the corruption test, per filesystem:

```sh
for fs in ext4 xfs btrfs f2fs; do
  sudo umount /mnt/test 2>/dev/null
  sudo mkfs.$fs -f $DEV > /dev/null 2>&1 || sudo mkfs.$fs -F $DEV > /dev/null 2>&1
  sudo mount $DEV /mnt/test
  sudo sh -c 'echo "CANARY DATA" > /mnt/test/canary'; sync
  PHYS=$(sudo filefrag -v /mnt/test/canary 2>/dev/null | awk '/^ *0:/{print $4}' | tr -d '.')
  sudo umount /mnt/test
  [ -n "$PHYS" ] && sudo dd if=/dev/urandom of=$DEV bs=4096 seek=$PHYS count=1 conv=notrunc 2>/dev/null
  sudo mount $DEV /mnt/test 2>/dev/null
  echo -n "$fs: "
  sudo cat /mnt/test/canary 2>&1 | head -c 40; echo
done
```

**Only btrfs (and ZFS) return an error rather than garbage.** That single result is the most important thing in Part 3 so far.

---

## 3. Mastery drills

1. State Rosenblum and Ousterhout's hardware argument for LFS, then state exactly which premise flash invalidates and which it strengthens.

2. Seltzer's measurements showed LFS losing to FFS. Identify the workload property responsible and construct a workload where LFS wins decisively.

3. Explain the wandering-tree problem precisely, then show how F2FS's NAT eliminates it. Name three other places in the kernel that use the same indirection for the same reason.

4. F2FS uses six logs. For each, state the lifetime hypothesis it encodes, and construct a workload that defeats the separation.

5. `page_private_gcing` sends surviving data to `CURSEG_COLD_DATA`. State the hypothesis, name its equivalent in language-runtime GC, and construct the workload where it is wrong.

6. Derive Rosenblum's cost-benefit formula `(1-u)·age/(1+u)` from first principles. Explain each term and why `age` appears.

7. Compare cleaning in F2FS, garbage collection in an FTL, and the btrfs cleaner thread. Identify the shared structure and the one thing each knows that the others do not.

8. ZNS eliminates the FTL. Enumerate everything that moves from the device to the host, and state precisely what information the host has that the device did not.

9. ZFS stores the checksum in the *parent* block pointer. State the three failure classes this catches that a checksum stored with the data does not, and compare with btrfs's approach (Ch. 59 §T.6).

10. RAID-Z eliminates the write hole with variable-width stripes. Explain the mechanism, state what it costs in read IOPS, and explain why btrfs's RAID5/6 could not adopt it without a format change.

11. The ARC maintains ghost lists. Explain the adaptation mechanism and compare with Linux's active/inactive lists and with MGLRU (Ch. 52 §T.7).

12. ZFS's ARC is separate from the page cache. Enumerate every consequence, including two that are advantages.

13. For each of the eight scenarios in §T.9, justify the filesystem choice from first principles using only Part 3's material — no appeals to popularity or benchmarks.

---

## 4. Further reading

**Papers — this chapter is unusually paper-dense, and they are all worth reading**

- Rosenblum & Ousterhout, "The Design and Implementation of a Log-Structured File System," SOSP 1991 / ACM TOCS 1992 ★★★ — **the origin.** Short, clear, and the cost-benefit formula in §1.5 is Figure 5 of this paper.
- Seltzer et al., "An Implementation of a Log-Structured File System for UNIX," USENIX 1993, and "File System Logging Versus Clustering: A Performance Comparison," USENIX 1995 ★★★ — the counter-argument. Read both sides.
- Ousterhout, "A Critique of Seltzer's LFS Measurements" — the rebuttal. The exchange is a model of how systems arguments should be conducted.
- Lee, Shin, Kim et al., "F2FS: A New File System for Flash Storage," FAST 2015 ★★★ — **the F2FS design paper.** Explains the NAT, multi-head logging, and adaptive logging.
- Bonwick & Moore, "ZFS: The Last Word in File Systems" (slides) and Bonwick's blog posts on RAID-Z and the ZFS block pointer ★★★ — the primary sources.
- Zhang, Rajimwale, Arpaci-Dusseau, Arpaci-Dusseau, "End-to-end Data Integrity for File Systems: A ZFS Case Study," FAST 2010 ★★★ — **rigorously evaluates what ZFS's checksums actually catch.** Essential.
- Megiddo & Modha, "ARC: A Self-Tuning, Low Overhead Replacement Cache," FAST 2003 ★★★ — §T.6(e).
- Bjørling et al., "ZNS: Avoiding the Block Interface Tax for Flash-based SSDs," USENIX ATC 2021 ★★★ — §T.5's case, with measurements.
- Aghayev, Weil, Kuchnik, Nelson, Ganger, Amvrosiadis, "File Systems Unfit as Distributed Storage Backends: Lessons from 10 Years of Ceph Evolution," SOSP 2019 ★★★ — a superb, opinionated retrospective on why a POSIX filesystem is the wrong substrate for a storage system. **Read this after finishing Part 3; it will recontextualise everything.**
- Arpaci-Dusseau, *OSTEP*, Chapter 43 ("Log-structured File Systems") ★★★ — free, and the clearest introduction to §T.1 and §T.4.

**Documentation**

- `Documentation/filesystems/f2fs.rst` ★★★ — design and every mount option.
- `Documentation/block/zoned.rst` ★★★ and `Documentation/filesystems/zonefs.rst`
- `Documentation/admin-guide/f2fs.rst`, `/sys/fs/f2fs/` documentation
- The OpenZFS documentation at `openzfs.github.io/openzfs-docs/` ★★★ — particularly "Performance and Tuning" and the `zfsconcepts(7)` / `zpoolconcepts(7)` man pages.
- `man 8 zdb`, `man 8 zpool`, `man 8 zfs`, `man 7 zfsprops` ★★★
- `zonedstorage.io` ★★★ — the reference site for zoned storage; covers SMR, ZNS, the kernel interfaces, and every filesystem's support status.

**LWN**

- "An f2fs teardown" and the F2FS merge coverage ★★★
- "Zoned block devices" and the ZNS series ★★★
- "zonefs: a filesystem for zoned block devices"
- "Btrfs on zoned devices"
- "ZFS, Linux, and the GPL" / "Kernel modules and the GPL" ★★★ — §T.7's legal question, covered carefully
- "The Adaptive Replacement Cache" discussions in the MGLRU coverage
- "Barriers and journaling filesystems" and the recurring durability threads
- "Shingled magnetic recording and Linux"

**Source reading order**

1. The LFS paper, then the F2FS paper. Do not start with the code.
2. `include/linux/f2fs_fs.h` ★★★ — the on-disk format.
3. `fs/f2fs/node.c`: `f2fs_get_node_info`, `set_node_addr` — §T.3(b)'s NAT.
4. `fs/f2fs/segment.c`: `__get_segment_type_6`, `allocate_segment_by_default` — §T.3(c).
5. `fs/f2fs/gc.c`: `get_victim_by_default`, `get_cb_cost`, `do_garbage_collect` ★★★ — §T.4.
6. `fs/f2fs/checkpoint.c`: `f2fs_write_checkpoint`; `fs/f2fs/recovery.c` — roll-forward.
7. `block/blk-zoned.c` and `fs/zonefs/super.c` — §T.5.
8. OpenZFS: `module/zfs/arc.c` (the ARC), `module/zfs/vdev_raidz.c` (RAID-Z), `module/zfs/zio.c` (the pipeline) — out of tree, but excellent code.

**Tools**

- `/proc/fs/f2fs/DEV/status` ★★★ and `/sys/fs/f2fs/DEV/*`
- `dump.f2fs`, `fsck.f2fs`, `sload.f2fs`, `f2fs_io`
- `blkzone report/reset/open/close/finish` ★★★, `zonefs-tools`, `libzbd`
- `null_blk` with `zoned=1` ★★★ — the easiest way to experiment with zoned storage
- `nvme zns id-ns`, `nvme zns report-zones` — for real ZNS hardware
- `zdb` ★★★, `zpool status -v`, `zpool iostat -v 1`, `arc_summary`, `arcstat`
- `/proc/spl/kstat/zfs/arcstats`
- `fio` with `--zonemode=zbd` for zoned-device benchmarking
- `smartctl -a` — `Total_LBAs_Written` and `Percentage_Used`; the write-amplification reality check on real flash

---

→ Next: [61-journaling-consistency.md](61-journaling-consistency.md)

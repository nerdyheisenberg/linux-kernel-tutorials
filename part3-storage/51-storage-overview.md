# Chapter 51 — Storage stack overview: from `write()` to spinning rust

> **Goal:** Build the complete mental map of the Linux storage stack before diving into any one layer. Understand why there are *nine* layers between a `write()` and a platter, what each one is actually for, why the abstraction that unified them (the block device) was both the stack's greatest success and the source of its hardest problems, where the durability boundary sits and why almost every application gets it wrong, and how the arrival of flash invalidated assumptions baked into every layer. By the end you can trace a byte from userspace to media and back, name the transformation each layer performs, and predict where latency and data loss come from.

> **Part 3 begins here.** Chapters 52–70 each take one layer of this map and go to the bottom of it. This chapter is the map, and it is the one to return to whenever a later chapter's detail threatens to obscure the shape of the whole.

---

## Theory & First Principles

### T.0 — Start here: `write()` returned. Is your data safe?

```c
int fd = open("important.txt", O_WRONLY | O_CREAT, 0644);
write(fd, "money transferred", 17);   /* returns 17. Success! */
close(fd);
/* Pull the power cord HERE. */
```

**Your data is almost certainly gone.** `write()` returning does not mean the data reached
the disk. It means the bytes were copied into the page cache. Nothing has been written
anywhere persistent.

Count the layers that can be holding your 17 bytes at that moment:

```
  your process        ── libc may buffer (if you used fwrite)
  page cache          ── DIRTY. Written back in ~30 s, or under pressure.
  filesystem journal  ── metadata may be committed before or after data
  device mapper / MD  ── a RAID5 stripe may be half-written
  block layer         ── requests queued, merged, reordered
  device write cache  ── the SSD said "done" from its DRAM buffer
  ------------------------------------------------------------------
  NAND / platter      ── the only place that survives power loss
```

**Seven layers, six of which lie to you.** They lie because telling the truth costs 20–500 µs
per write (a device cache flush), and a system that told the truth on every `write()` would
be 1000× slower.

**The honest version:**

```c
write(fd, buf, len);
fsync(fd);          /* force writeback AND a device cache flush */
/* only NOW is the data durable... */
fsync(dirfd);       /* ...and only now does the FILE NAME exist durably */
```

That second `fsync` surprises nearly everyone. `fsync(fd)` makes the file's *contents*
durable; it says nothing about the **directory entry** that lets you find the file. Crash
between them and you have durable data in a file with no name. This is why the safe
write-a-file-atomically recipe is four steps, and why most programs get it wrong (Ch. 53,
`question-bank.md` E2).

**Now the second big idea, and it is the one that reorganizes the whole of Part 3.** All of
those layers exist because of a gap:

```
  A CPU sees:           bytes, individually addressable,      ~80 ns
  Storage provides:     SECTORS (512B/4KB), all-or-nothing,   ~80 us (NVMe)
                                                              ~10 ms (HDD)

            a 1,000x to 100,000x latency gap,
            and a granularity mismatch of 4,000x
```

**Every layer in the storage stack exists to bridge some part of that gap:**

| Layer | Bridges |
|---|---|
| Page cache | the *latency* gap — keep hot data in DRAM |
| Writeback | the *granularity* gap — batch many small writes into few large ones |
| Filesystem | the *naming* gap — sectors have numbers, humans want paths |
| Journal | the *atomicity* gap — sectors are atomic individually, operations are not |
| Block layer | the *scheduling* gap — many requesters, one device |
| Device mapper / MD | the *topology* gap — one logical device, many physical ones |

**And the thing that changed everything, which you should carry through Part 3:**

| | HDD | NVMe SSD | Ratio changed by |
|---|---|---|---|
| Sequential read | 150 MB/s | 7 GB/s | 45x |
| **Random IOPS** | **~100** | **~1,000,000** | **10,000x** |
| **random : sequential** | **~1000:1** | **~2:1** | the *shape* of the problem |

That last row is the important one. **Nearly every optimization in the storage stack was
designed to avoid seeks** — elevator scheduling, aggressive readahead, extent packing,
cylinder groups, defragmentation. On NVMe, seeks cost almost nothing, so most of that
machinery is now pure overhead. Recognizing which parts of the stack are solving a problem
that no longer exists is the analytical thread running through Ch. 63–69.

```bash
# See the layers for yourself:
lsblk -t                                  # topology, queue depths, scheduler
cat /sys/block/nvme0n1/queue/scheduler    # 'none' -- nothing left to reorder
cat /proc/meminfo | grep -E 'Dirty|Writeback'
sudo blktrace -d /dev/nvme0n1 -o - | blkparse -i -   # every request, live
```

---

### T.1 The problem: two irreconcilable models of storage

Applications want storage that is:

- **Byte-addressable** — write 7 bytes at offset 1,000,003.
- **Named hierarchically** — `/home/user/notes.txt`, not "sector 4,812,993".
- **Persistent** — survives power loss.
- **Concurrent** — many processes, correctly.
- **Fast** — memory speed, ideally.

Storage devices provide storage that is:

- **Block-addressable** — read/write in units of 512 B or 4 KiB, never less.
- **Flat** — an array of numbered blocks, no names.
- **Persistent with caveats** — the device has a volatile write cache it will happily lie about.
- **Single-threaded per command, with a queue** — and deep reordering.
- **Slow, and slow in structured ways** — seek time, erase blocks, garbage collection.

Every layer in the storage stack exists to close some part of that gap. The stack is therefore not arbitrary complexity; it is a sequence of *specific impedance mismatches*, each with a named solution:

| Mismatch | Layer that closes it | Chapter |
|---|---|---|
| bytes vs blocks | page cache | 52 |
| names vs numbers | filesystem | 53–60 |
| many filesystems vs one syscall API | VFS | 53–55 |
| crash-atomicity vs no atomicity | journalling / CoW | 61 |
| one logical device vs many physical | device mapper, MD | 66–67 |
| one queue vs many CPUs | blk-mq | 63–64 |
| fairness vs throughput | I/O schedulers | 64 |
| many transports vs one block API | SCSI/ATA/NVMe/MMC drivers | 68–70 |
| flash's erase-before-write vs rewrite-in-place | FTL (in the device) or F2FS/UBI (in the kernel) | 60, 70 |

### T.2 The full stack, once, in order

```
  ┌───────────────────────────────────────────────────────────────┐
  │ application:  write(fd, buf, len)  /  mmap store  /  io_uring  │
  └───────────────────────────────┬───────────────────────────────┘
                                  │  syscall boundary (Ch. 24)
  ┌───────────────────────────────▼───────────────────────────────┐
  │ VFS               generic_file_write_iter, path walk, dcache   │  Ch. 53–55
  │                   -> one API over every filesystem             │
  └───────────────────────────────┬───────────────────────────────┘
                                  │
  ┌───────────────────────────────▼───────────────────────────────┐
  │ page cache        folios in an XArray per address_space        │  Ch. 52
  │                   -> bytes become pages; writes become dirty   │
  └───────────────────────────────┬───────────────────────────────┘
                                  │  writeback (async) or O_DIRECT (bypass)
  ┌───────────────────────────────▼───────────────────────────────┐
  │ filesystem        ext4 / XFS / Btrfs / F2FS / overlayfs        │  Ch. 56–60, 62
  │                   -> file offsets become block numbers         │
  │                   -> journalling / CoW gives crash atomicity   │  Ch. 61
  └───────────────────────────────┬───────────────────────────────┘
                                  │  submit_bio(bio)
  ┌───────────────────────────────▼───────────────────────────────┐
  │ block layer       bio -> request; plugging, merging, splitting │  Ch. 63
  │                   blk-mq: per-CPU software queues              │
  │                   I/O scheduler: mq-deadline / bfq / kyber     │  Ch. 64
  │                   blk-cgroup: io.latency, io.max               │
  └───────────────────────────────┬───────────────────────────────┘
                                  │  (optional stacking, in either order)
  ┌───────────────────────────────▼───────────────────────────────┐
  │ device mapper / MD   dm-crypt, dm-thin, dm-cache, RAID 0/1/5/6 │  Ch. 66–67
  │                   -> one bio becomes N bios on other devices   │
  └───────────────────────────────┬───────────────────────────────┘
                                  │  request_queue -> driver
  ┌───────────────────────────────▼───────────────────────────────┐
  │ block driver      NVMe / SCSI midlayer + libata / MMC / MTD    │  Ch. 68–70
  │                   -> requests become protocol commands         │
  └───────────────────────────────┬───────────────────────────────┘
                                  │  DMA + doorbell (Ch. 34–38)
  ┌───────────────────────────────▼───────────────────────────────┐
  │ device            controller firmware, FTL, write cache, media │
  └───────────────────────────────────────────────────────────────┘
```

Read that diagram in both directions. Downward is the *submission* path, and each layer **adds information** (a name becomes an inode, an offset becomes a block, a block becomes a command). Upward is the *completion* path, and each layer **removes information** while propagating errors and timing. The asymmetry matters: the submission path is where policy lives; the completion path is where the hard concurrency lives (Ch. 63 §T.6).

### T.3 The block device: the stack's narrow waist

The single most consequential abstraction here is the **block device**: a linear array of fixed-size blocks supporting read and write at a block granularity. Everything above it is written against that interface; everything below implements it.

Its power is composability. Because `dm-crypt`, `md-raid`, `dm-thin`, and a partition all *consume* a block device and *produce* a block device, they stack arbitrarily:

```
   ext4
     └── dm-crypt
           └── LVM logical volume (dm-linear/dm-thin)
                 └── md RAID-5
                       ├── /dev/nvme0n1p2
                       ├── /dev/nvme1n1p2
                       └── /dev/nvme2n1p2
```

Nobody designed that stack; it composes because each element speaks the same interface. This is the hourglass/narrow-waist argument (Ch. 46 §T.1 for networking, Ch. 29 §T.1 for files) at its most successful.

Its costs are equally real, and Part 3 will keep returning to them:

| Cost | Consequence |
|---|---|
| **Information is lost going down** | the block layer does not know a write is metadata, or that two writes must be atomic together, or which file a block belongs to |
| **Semantics are lost going up** | the filesystem does not know the device is RAID-5 (read-modify-write costs), or has an FTL, or is compressing |
| **Ordering is not expressible** | "write A before B" has no representation; only "flush everything" does (§T.5) |
| **Atomicity is not expressible** | no "write these 3 blocks or none" — the entire reason journalling exists (Ch. 61) |
| **The linear address illusion is a lie on flash** | adjacent LBAs are not adjacent in media; the FTL indirection is invisible |

Each of those gaps produced a workaround somewhere in the stack, and a large fraction of Part 3's complexity is those workarounds. Recognising them as *consequences of one abstraction choice* makes the whole subject coherent rather than arbitrary.

Efforts to widen the waist are a recurring theme: `REQ_META` hints, `fallocate()` and discard/TRIM, write hints and stream IDs, **zoned block devices** (Ch. 69), and multi-actuator/multi-namespace devices. Each adds a channel for information the flat interface destroyed.

### T.4 Where the data actually is: the cache hierarchy

At any moment, a byte you wrote may exist in up to six places, each with different durability:

| Location | Survives process crash? | Survives kernel crash? | Survives power loss? |
|---|---|---|---|
| libc `FILE*` buffer | **no** | no | no |
| page cache (dirty folio) | yes | **no** | no |
| device write cache (DRAM on the drive) | yes | yes | **no** (unless PLP) |
| device write cache with power-loss protection | yes | yes | yes |
| media (platter / NAND) | yes | yes | yes |
| battery-backed / NVDIMM / CXL persistent memory | yes | yes | yes |

The critical row is the third, and it is the source of more data-loss bugs than any other fact in storage:

> **Consumer SSDs and HDDs acknowledge a write as soon as it reaches their volatile DRAM cache.** The device reports success; the data is not durable. Only an explicit **cache flush** (or a write tagged **FUA**) makes it durable.

Enterprise drives with **power-loss protection** (capacitors sufficient to drain the cache) can honestly report "write cache is durable" and the kernel will skip flushes. `sdparm`/`nvme id-ctrl` tell you which you have (Lab 51.4).

The three-level cache structure means there are three distinct "flush" operations, and conflating them is a classic bug:

```c
	fflush(f);   /* libc buffer -> kernel page cache.  NOT durable. */
	fsync(fd);   /* page cache -> device, AND flush device cache. Durable. */
	sync();      /* everything, everywhere. Durable, slow, rarely what you want. */
```

`fflush()` followed by `exit()` survives a process crash and nothing else. This is the single most common durability misunderstanding in application code.

### T.5 The durability contract, stated precisely

What does `write()` actually promise? The answer is narrower than people assume, and it is worth writing down formally.

**`write()` returning `n` promises:** any subsequent `read()` through the same filesystem, from any process, will see the new data. That is all. It is a *visibility* guarantee, not a *durability* guarantee. POSIX calls this the requirement that the write be atomic with respect to other reads (for regular files, within limits).

**`fsync(fd)` promises:** the file's data *and* the metadata needed to retrieve it are on durable media, including a device cache flush. On Linux, with a correctly implemented filesystem and an honest device.

**`fdatasync(fd)` promises:** the same, except metadata not needed for retrieval (mtime, atime) may be omitted. Cheaper when the file size did not change.

Four traps follow immediately, and each has caused production data loss:

**(a) `fsync()` on the file is not enough for a *new* file.** Creating `foo.tmp` and `fsync()`ing it does not guarantee the *directory entry* is durable. On crash, the file may exist with correct contents and no name, or not exist at all. The complete safe-rename idiom is:

```c
	fd = open("foo.tmp", O_WRONLY|O_CREAT|O_TRUNC, 0644);
	write(fd, data, len);
	fsync(fd);                 /* data durable */
	close(fd);
	rename("foo.tmp", "foo");  /* atomic name swap */
	dirfd = open(".", O_RDONLY|O_DIRECTORY);
	fsync(dirfd);              /* the RENAME is durable */
	close(dirfd);
```

Miss the directory `fsync()` and you have a race whose window is the writeback interval — typically 30 seconds.

**(b) A failed `fsync()` may not be retryable.** Before Linux 4.13, a writeback error could be consumed by *whichever* `fsync()` happened to see it, and later `fsync()`s on the same file would return success despite the data never reaching disk. The "fsyncgate" discussion (2018, prompted by PostgreSQL) led to per-file-description error tracking (`errseq_t`), so each open fd sees the error exactly once. But the deeper problem remains:

> **After `fsync()` returns an error, the state of your data is undefined and the dirty pages may have been discarded. There is no portable way to retry.** The only safe response is to treat the file as lost and fail over.

This is genuinely uncomfortable and it is worth sitting with, because it shapes how real databases are written.

**(c) Ordering between two files is not expressible** except by `fsync()`ing the first and waiting. There is no "write B after A is durable, but do not block me." This is the motivation behind proposals for `fbarrier()`/asynchronous durability and behind the way databases batch everything into one log file.

**(d) `O_DIRECT` bypasses the page cache but *not* the device cache.** `O_DIRECT|O_SYNC` or `O_DSYNC`, or `RWF_DSYNC` on `pwritev2()`, is needed for durability. `O_DIRECT` alone is a performance choice, not a durability one — a confusion that appears in a great deal of production code.

### T.6 The flush/FUA mechanism: how ordering is expressed at all

Given §T.3's "ordering is not expressible," how does journalling work? Through exactly two mechanisms the block layer provides:

| Flag | Meaning |
|---|---|
| `REQ_PREFLUSH` | "before this request, everything previously acknowledged must be on media" |
| `REQ_FUA` (Force Unit Access) | "this request is not complete until *it* is on media" |

A filesystem builds crash-consistency from these. The canonical journal commit is:

```
   write journal blocks ........................ (may be cached)
   PREFLUSH + write commit block + FUA ......... commit is durable only after
                                                  the journal blocks are durable
   ... later ...
   PREFLUSH + write the real metadata in place
   checkpoint: journal space can be reused
```

The `PREFLUSH` creates a **barrier in time**: nothing after it can be durable before everything before it. That is the only ordering primitive the block interface has, and every crash-consistency scheme in Part 3 is built from it.

Three things to internalise:

1. **A flush is expensive and its cost is not proportional to the data being flushed.** On a spinning disk it can cost a full rotation plus a seek; on an SSD it forces the FTL to commit its mapping tables. Costs of 0.1–10 ms are typical. This is why filesystems batch: one flush per transaction, not per write.
2. **The block layer optimises flushes aggressively** (`block/blk-flush.c`): concurrent flush requests are coalesced into one, since a single flush satisfies all of them. The state machine there is small and worth reading.
3. **`REQ_FUA` may be emulated.** If the device does not support FUA, the block layer turns it into write-then-flush. Drivers advertise their capability via `blk_queue_write_cache()`.

The `nobarrier`/`nobarriers` mount option, which some tuning guides still recommend, disables all of this. It makes benchmarks faster and makes crash consistency a fiction. Never use it on data you care about.

### T.7 Cost models: why the stack is shaped the way it is

Every design decision in Part 3 traces back to a number. Learn these orders of magnitude; they explain more than any amount of code reading.

| Operation | HDD (7200 rpm) | SATA SSD | NVMe SSD | Optane / SCM | DRAM |
|---|---|---|---|---|---|
| Random 4 K read latency | 8–15 ms | 80–150 µs | 20–80 µs | 8–10 µs | 80 ns |
| Sequential throughput | 150–250 MB/s | 500 MB/s | 3–14 GB/s | 2 GB/s | 20+ GB/s |
| Random 4 K IOPS (QD1) | ~100 | ~10 K | ~20 K | ~120 K | — |
| Random 4 K IOPS (QD32+) | ~150 | ~80 K | **1–3 M** | ~600 K | — |
| Random/sequential ratio | **~1000×** | ~5× | ~2× | ~1× | 1× |
| Cache flush cost | 5–15 ms | 0.5–2 ms | 20–500 µs | ~0 | — |
| Write amplification | 1× | 1.5–5× | 1.1–3× | ~1× | — |

Four consequences that shaped the entire stack:

**(a) HDD's 1000× random/sequential ratio made *everything* about avoiding seeks.** Elevator scheduling, extent-based allocation, block groups, readahead, delayed allocation, defragmentation — all of it is seek avoidance. Much of that machinery is now overhead on flash, and Part 3 will repeatedly note which optimisations aged badly.

**(b) The per-I/O CPU cost stopped being negligible.** At 100 IOPS, spending 10 µs of CPU per I/O is 0.1 % of a core. At 3 M IOPS it is impossible — you would need 30 cores just for the block layer. This is precisely why **blk-mq replaced the single-queue block layer** (Ch. 63 §T.3): the old design had one lock per device and could not exceed ~800 K IOPS on any hardware.

**(c) Queue depth became the primary performance variable.** HDDs get ~1.5× from deep queues (limited reordering benefit). NVMe gets 50–100×, because parallelism across NAND dies is the only way to reach the device's capability. This is why `io_uring` and asynchronous I/O matter far more now than in 2005 — the hardware cannot be saturated synchronously.

**(d) The scheduler's job inverted.** On HDD, the I/O scheduler's job was to *reorder* to minimise seeks — real, large wins. On NVMe, reordering buys nothing and the scheduler's remaining job is *fairness and latency isolation*. Hence `none` being the default scheduler for NVMe, and `kyber`/`bfq` being about QoS rather than throughput (Ch. 64).

### T.8 Flash changed the contract, and the stack is still adjusting

NAND flash has three properties that the block abstraction actively hides:

1. **You cannot overwrite.** A page (4–16 KiB) must be *erased* before rewriting, and erase operates on a *block* of 128–512 pages (several MiB).
2. **Cells wear out.** 500–100,000 program/erase cycles depending on the cell type (SLC/MLC/TLC/QLC).
3. **Reads disturb neighbours** and data retention degrades with time and temperature.

Reconciling that with "a linear array of rewritable blocks" requires a **Flash Translation Layer**: a log-structured, garbage-collecting, wear-levelling indirection from LBA to physical page. Consumer SSDs put the FTL in device firmware, and the result is that:

- **Write amplification** is real and invisible — writing 1 GiB may cause 3 GiB of NAND writes.
- **Latency is unpredictable** — a read can be stuck behind a garbage-collection erase, producing tail latencies 10–100× the median.
- **The device is doing a job the filesystem already does.** Both the FTL and a log-structured filesystem perform garbage collection, over the same data, without coordination.

That last point is the interesting one, and it motivates three responses that Part 3 covers:

| Response | Idea | Where |
|---|---|---|
| **TRIM/discard** | tell the device which LBAs are free so GC can skip them | Ch. 63, 69 |
| **Host-managed zones (ZNS)** | expose the erase-block structure; the host does the log management | Ch. 69 |
| **Raw flash in the kernel** (MTD/UBI/F2FS/JFFS2) | no FTL at all; the kernel *is* the FTL | Ch. 60, 70 |

The general principle, which recurs throughout systems work:

> **Two layers independently solving the same problem, without an interface to coordinate, is worse than either solving it alone. The fix is always to widen the interface, not to add a third layer.**

### T.9 The mental model to carry into Part 3

Four questions to ask of any storage layer you study:

1. **What transformation does it perform?** (bytes→pages, names→blocks, bios→requests, requests→commands)
2. **What information does it add, and what does it destroy?** The destroyed information is where the next layer's problems come from.
3. **Where is its durability boundary?** What does "done" mean here — visible, queued, or on media?
4. **What was it designed for, and does that assumption still hold?** Most of the stack was designed for one spinning disk with a 10 ms seek.

And three invariants that hold throughout:

- **Data is only durable after a flush that the device honoured.** Everything else is optimism.
- **Every layer can reorder, unless a flush forbids it.**
- **Errors propagate up but context does not.** An `-EIO` at the application says nothing about which layer failed or why — hence the debugging techniques of Lab 51.6.

---

## 1. Internals

### 1.1 Source map for the whole of Part 3

| Path | Layer | Chapter |
|---|---|---|
| `fs/read_write.c`, `fs/open.c`, `fs/namei.c` | syscall entry, path walking | 53–54 |
| `fs/inode.c`, `fs/dcache.c`, `fs/super.c`, `fs/namespace.c` | VFS objects, mounts | 53–54 |
| `mm/filemap.c`, `mm/page-writeback.c`, `mm/readahead.c` | page cache, writeback | 52 |
| `fs/iomap/` | the modern block-mapping and direct-I/O layer | 55 |
| `fs/ext4/`, `fs/xfs/`, `fs/btrfs/`, `fs/f2fs/` | filesystems | 57–60 |
| `fs/jbd2/` | ext4's journal | 61 |
| `fs/overlayfs/`, `fs/fuse/`, `fs/nfs/`, `fs/smb/` | stacking and network fs | 62 |
| `block/bio.c`, `blk-core.c`, `blk-mq*.c` | the block layer | 63 |
| `block/mq-deadline.c`, `bfq-*.c`, `kyber-iosched.c`, `blk-cgroup.c` | schedulers and QoS | 64 |
| `block/blk-flush.c` | the flush state machine of §T.6 | 61, 63 |
| `drivers/md/dm*.c`, `drivers/md/md*.c`, `raid*.c` | device mapper, MD RAID | 66–67 |
| `drivers/scsi/`, `drivers/ata/` | SCSI midlayer, libata | 68 |
| `drivers/nvme/host/` | NVMe | 69 |
| `drivers/mmc/`, `drivers/mtd/`, `drivers/mtd/ubi/` | eMMC/SD, raw flash | 70 |
| `include/linux/blk_types.h`, `blkdev.h`, `fs.h`, `pagemap.h` | the core types |
| `Documentation/filesystems/`, `Documentation/block/` | docs |

### 1.2 The objects, one sentence each

| Object | What it is |
|---|---|
| `struct file` | one *open*: position, flags, credentials, `f_op` |
| `struct dentry` | one *name→inode* link, cached; the path-walk unit |
| `struct inode` | one *file*: metadata, `i_op`, `i_mapping` |
| `struct address_space` | the *cached contents* of an inode: an XArray of folios + `a_ops` |
| `struct folio` | one or more contiguous pages of cached data (Ch. 22 §T.5) |
| `struct super_block` | one *mounted filesystem instance* |
| `struct vfsmount` / `struct mount` | one *attachment* of a superblock into a namespace |
| `struct bio` | one *I/O request in flight*: a device, a sector, a list of page segments, a direction, flags |
| `struct request` | one or more merged bios, as the block layer's schedulable unit |
| `struct request_queue` | a block device's I/O path: limits, scheduler, tag set |
| `struct gendisk` | a *block device*: name, partitions, `block_device_operations` |
| `struct block_device` | one *access path* to a gendisk or partition (`/dev/sda`, `/dev/sda1`) |

The `inode` / `address_space` split is worth noting now, because it recurs: metadata and cached contents are separate objects because a device can have cached contents without being a file, and because two inodes (a file and its block device) can map the same content.

### 1.3 `struct bio` — the unit of block I/O

```c
struct bio {
	struct bio		*bi_next;
	struct block_device	*bi_bdev;
	blk_opf_t		bi_opf;      /* op (READ/WRITE/FLUSH/DISCARD)
					        + flags (SYNC, META, FUA,
					          PREFLUSH, IDLE, RAHEAD, NOWAIT) */
	unsigned short		bi_flags;
	blk_status_t		bi_status;
	struct bvec_iter	bi_iter;     /* sector, size, current position */
	bio_end_io_t		*bi_end_io;  /* completion callback */
	void			*bi_private;
	struct bio_vec		*bi_io_vec;  /* the (page, offset, len) list */
	unsigned short		bi_vcnt, bi_max_vecs;
	atomic_t		__bi_remaining;
	/* ... cgroup, integrity, crypto context ... */
};
```

Four observations that will matter constantly:

- **A bio is a scatter-gather list**, not a buffer. `bi_io_vec` is an array of `(page, offset, len)` — the same shape as the DMA scatter-gather of Ch. 35 §T.5, and for the same reason: the pages backing a file's cached data are not contiguous.
- **`bi_iter` is a cursor.** Splitting a bio (because it exceeds a device limit, or crosses a RAID stripe) does not copy anything; it clones the bio and advances the iterator. `bio_split()`/`bio_chain()` make this cheap.
- **Completion is a callback**, not a wait. `bi_end_io` runs in whatever context the completion arrives in — often softirq. Every layer that transforms a bio must install its own `bi_end_io` and restore the caller's.
- **`bi_opf` carries the semantics** the block layer is allowed to know: is this synchronous (someone is waiting), metadata, read-ahead, a barrier. This is the narrow channel through which §T.3's destroyed information partially survives.

### 1.4 The path of one buffered write, in code

```
write(fd, buf, 4096)
 └─ ksys_write → vfs_write → new_sync_write
     └─ file->f_op->write_iter()          /* ext4_file_write_iter */
         └─ generic_perform_write()        [mm/filemap.c]
             ├─ a_ops->write_begin()       /* allocate blocks if needed,
             │                                get/create the folio */
             ├─ copy_folio_from_iter_atomic()   /* the actual memcpy */
             └─ a_ops->write_end()         /* mark folio dirty,
                                              update i_size */
                 └─ folio_mark_dirty()
                     └─ __folio_mark_dirty() → tag DIRTY in the XArray
                         └─ inode_attach_wb(), wb_wakeup_delayed()
   ... returns to userspace. NOTHING has reached the device. ...

   Later, one of: dirty_expire_centisecs elapsed (30 s)
                  dirty_ratio exceeded (balance_dirty_pages)
                  sync/fsync/syncfs
                  memory reclaim needs the page
 └─ wb_workfn → wb_do_writeback → writeback_sb_inodes
     └─ do_writepages → a_ops->writepages()     /* ext4_writepages */
         └─ iomap_writepages / mpage_writepages
             └─ submit_bio(bio)                  [block/blk-core.c]
                 └─ blk_mq_submit_bio()
                     ├─ blk_mq_sched_bio_merge()  /* merge into a queued rq? */
                     ├─ plugging (per-task list)
                     ├─ scheduler insert (mq-deadline/bfq/kyber/none)
                     └─ blk_mq_run_hw_queue → q->mq_ops->queue_rq()
                         └─ nvme_queue_rq()       [drivers/nvme/host/pci.c]
                             ├─ build NVMe command, map PRP/SGL (Ch. 35)
                             └─ write the submission-queue doorbell (Ch. 34)
   ... device DMAs the data, posts a completion entry, raises MSI-X (Ch. 38) ...
 └─ nvme_irq → nvme_poll_cq → blk_mq_complete_request
     └─ (possibly IPI to the submitting CPU) → blk_mq_end_request
         └─ bio_endio → folio_end_writeback → clear the WRITEBACK tag
```

Every chapter in Part 3 is an expansion of some contiguous slice of that trace. Copy it somewhere you can see it.

### 1.5 The observability surface

```sh
/proc/diskstats                       # per-device I/O counters
/sys/block/<dev>/stat                 # same, per device
/sys/block/<dev>/queue/               # THE tuning directory
	scheduler, nr_requests, read_ahead_kb, rotational,
	max_sectors_kb, logical/physical_block_size, nomerges,
	write_cache, fua, discard_granularity, zoned, rq_affinity
/sys/block/<dev>/integrity/           # T10 PI / DIF
/sys/kernel/debug/block/<dev>/        # blk-mq internals: per-hctx state
/proc/meminfo                         # Dirty:, Writeback:, Buffers:, Cached:
/proc/sys/vm/dirty_{ratio,background_ratio,expire_centisecs,writeback_centisecs}
/sys/fs/<fstype>/<dev>/               # per-filesystem knobs and stats
/proc/self/mountinfo                  # mounts with propagation state
```

---

## 2. Practice

### Lab 51.1 — Map your own storage stack

```sh
lsblk -o NAME,MAJ:MIN,RM,SIZE,RO,TYPE,MOUNTPOINTS,MODEL
lsblk -t                     # topology: alignment, min/opt io, scheduler, RA
sudo blkid
cat /proc/partitions
findmnt --real
cat /proc/self/mountinfo | head
```

For each layer you find, identify it in the §T.2 diagram:

```sh
# Device mapper stack, if any
sudo dmsetup ls --tree
sudo dmsetup table
sudo lvs -o +devices; sudo pvs; sudo vgs

# MD RAID
cat /proc/mdstat
sudo mdadm --detail /dev/md0 2>/dev/null

# The physical device
sudo nvme list 2>/dev/null
sudo smartctl -i /dev/nvme0n1 2>/dev/null || sudo smartctl -i /dev/sda
```

Now draw your machine's actual stack, top to bottom, and for each edge write down the transformation. Verify the queue limits propagate correctly through the stack:

```sh
for d in /sys/block/*/queue; do
  dev=$(basename $(dirname $d))
  printf "%-12s rot=%s sched=%-12s ra=%-6s max_sect=%-6s lbs=%-5s pbs=%-5s wc=%s\n" \
    "$dev" "$(cat $d/rotational)" "$(cat $d/scheduler | grep -o '\[.*\]')" \
    "$(cat $d/read_ahead_kb)" "$(cat $d/max_sectors_kb)" \
    "$(cat $d/logical_block_size)" "$(cat $d/physical_block_size)" \
    "$(cat $d/write_cache)"
done
```

Note where a stacked device's limits differ from its children's. A dm target must not exceed the *most restrictive* of its underlying devices, and when it does, you get bio splitting (Ch. 63) or errors. `lsblk -t` shows this as `MIN-IO`/`OPT-IO`/`ALIGNMENT`.

---

### Lab 51.2 — Watch a byte travel

Trace the full submission path for one buffered write.

```sh
# 1. Where does the write go?  (It does not go to the device.)
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_write /comm == "dd"/ { @writes = count(); }
tracepoint:block:block_rq_issue     { @block_io = count(); }
interval:s:1 { printf("writes=%d  block_io=%d\n", @writes, @block_io);
               clear(@writes); clear(@block_io); }'

# In another terminal:
dd if=/dev/zero of=/tmp/testfile bs=4k count=10000
# -> thousands of writes, near-zero block I/O, until writeback fires
sync
# -> now the block I/O appears
```

Now the full stack trace for a single I/O:

```sh
sudo bpftrace -e '
tracepoint:block:block_rq_issue /args->bytes > 0/ {
	printf("%-8s dev=%d,%d sect=%llu nsect=%u rwbs=%s\n",
	       comm, args->dev >> 20, args->dev & 0xfffff,
	       args->sector, args->nr_sector, args->rwbs);
	printf("%s\n", kstack);
	exit();
}'
dd if=/dev/zero of=/tmp/t bs=1M count=1 oflag=direct
```

The kernel stack in that output *is* §1.4's trace, for your kernel. Read it and map every frame to a layer.

Use the standard tracers too:

```sh
sudo biosnoop-bpfcc        # every block I/O: pid, comm, dev, sector, bytes, latency
sudo biolatency-bpfcc -D   # latency histogram per device
sudo biotop-bpfcc          # top by I/O
sudo ext4slower-bpfcc 1    # filesystem-level ops slower than 1 ms
sudo cachestat-bpfcc       # page cache hit/miss rates
sudo filetop-bpfcc         # per-file I/O
```

`biolatency -D` on your root device, while running a mixed workload, gives you the latency distribution that §T.7's table predicts. Compare the measured p50/p99 to the table and explain the gap.

---

### Lab 51.3 — Prove where the data is

Demonstrate each row of §T.4's durability table.

**(a) Page cache holds dirty data.**

```sh
grep -E '^(Dirty|Writeback|Cached|Buffers):' /proc/meminfo
dd if=/dev/zero of=/tmp/big bs=1M count=500
grep -E '^(Dirty|Writeback):' /proc/meminfo      # Dirty: ~500000 kB
sync
grep -E '^(Dirty|Writeback):' /proc/meminfo      # back to ~0
```

**(b) The writeback timer.**

```sh
cat /proc/sys/vm/dirty_expire_centisecs      # 3000 = 30 s
cat /proc/sys/vm/dirty_writeback_centisecs   # 500 = flusher wakes every 5 s
cat /proc/sys/vm/dirty_ratio                 # 20 (% of available memory)
cat /proc/sys/vm/dirty_background_ratio      # 10

dd if=/dev/zero of=/tmp/x bs=1M count=100
watch -n1 "grep -E '^(Dirty|Writeback):' /proc/meminfo"
# Watch Dirty decay to zero over ~30 s with no sync
```

**(c) `fsync()` costs what a flush costs.**

```c
// SPDX-License-Identifier: GPL-2.0
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>

static double now(void)
{
	struct timespec ts;

	clock_gettime(CLOCK_MONOTONIC, &ts);
	return ts.tv_sec + ts.tv_nsec / 1e9;
}

int main(int argc, char **argv)
{
	const char *path = argc > 1 ? argv[1] : "/tmp/durability-test";
	char buf[4096];
	int n = 1000, i;
	double t0;
	int fd;

	memset(buf, 'x', sizeof(buf));

	/* 1. buffered, no sync */
	fd = open(path, O_WRONLY | O_CREAT | O_TRUNC, 0644);
	t0 = now();
	for (i = 0; i < n; i++)
		if (write(fd, buf, sizeof(buf)) != sizeof(buf)) abort();
	printf("buffered          : %8.1f us/write\n", (now()-t0)/n*1e6);
	close(fd);

	/* 2. fsync every write */
	fd = open(path, O_WRONLY | O_CREAT | O_TRUNC, 0644);
	t0 = now();
	for (i = 0; i < n; i++) {
		if (write(fd, buf, sizeof(buf)) != sizeof(buf)) abort();
		fsync(fd);
	}
	printf("write+fsync       : %8.1f us/write\n", (now()-t0)/n*1e6);
	close(fd);

	/* 3. fdatasync: no metadata update needed after the first pass */
	fd = open(path, O_WRONLY, 0644);
	t0 = now();
	for (i = 0; i < n; i++) {
		if (pwrite(fd, buf, sizeof(buf), i * 4096L) != sizeof(buf)) abort();
		fdatasync(fd);
	}
	printf("pwrite+fdatasync  : %8.1f us/write\n", (now()-t0)/n*1e6);
	close(fd);

	/* 4. O_DSYNC: the kernel does it for us */
	fd = open(path, O_WRONLY | O_DSYNC);
	t0 = now();
	for (i = 0; i < n; i++)
		if (pwrite(fd, buf, sizeof(buf), i * 4096L) != sizeof(buf)) abort();
	printf("O_DSYNC pwrite    : %8.1f us/write\n", (now()-t0)/n*1e6);
	close(fd);

	/* 5. O_DIRECT without sync -- fast, and NOT durable (T.5 trap d) */
	fd = open(path, O_WRONLY | O_DIRECT);
	if (fd >= 0) {
		void *abuf;
		posix_memalign(&abuf, 4096, 4096);
		memset(abuf, 'x', 4096);
		t0 = now();
		for (i = 0; i < n; i++)
			if (pwrite(fd, abuf, 4096, i * 4096L) != 4096) abort();
		printf("O_DIRECT (no sync): %8.1f us/write  <-- NOT DURABLE\n",
		       (now()-t0)/n*1e6);
		close(fd);
		free(abuf);
	}
	unlink(path);
	return 0;
}
```

```sh
gcc -O2 -o durability durability.c && ./durability /tmp/durability-test
# Run again on a different filesystem/device and compare:
./durability /var/tmp/durability-test
```

Expect roughly: buffered ~1 µs, `fsync` 200 µs–10 ms depending on device, `fdatasync` somewhat less, `O_DIRECT` without sync fast. The ratio between rows 1 and 2 is **the price of durability**, and it is the number that explains why every database batches commits.

---

### Lab 51.4 — Interrogate the device's honesty

```sh
# Does the kernel think the device has a volatile write cache?
cat /sys/block/nvme0n1/queue/write_cache     # "write back" or "write through"
cat /sys/block/nvme0n1/queue/fua             # does it support FUA?

# SCSI/SATA
sudo sdparm --get=WCE /dev/sda               # Write Cache Enable
sudo hdparm -W /dev/sda

# NVMe: VWC bit = volatile write cache present
sudo nvme id-ctrl /dev/nvme0 | grep -E '^vwc|^awun|^awupf|^nn'
sudo nvme get-feature /dev/nvme0 -f 6 -H     # current volatile write cache state

# Power loss protection? (Enterprise drives)
sudo smartctl -a /dev/nvme0 | grep -i -E 'power.?loss|plp|capacitor'
sudo nvme smart-log /dev/nvme0
```

Two values worth understanding:

- **`awun` / `awupf`** (Atomic Write Unit Normal / Power Fail): how many logical blocks the NVMe device guarantees to write atomically across a power failure. Usually 1 (i.e. one sector). This is the *only* atomicity the hardware gives you, and it is why MySQL's doublewrite buffer exists.
- **`write_cache`** controls whether the kernel emits `REQ_PREFLUSH` at all. Setting it to `write through` on a device without PLP is a correctness change disguised as a tuning knob.

Measure the flush cost directly:

```sh
sudo fio --name=flushtest --filename=/dev/nvme0n1 --rw=write --bs=4k \
         --iodepth=1 --numjobs=1 --runtime=10 --time_based \
         --direct=1 --fdatasync=1 --group_reporting
# Compare with --fdatasync=0
```

The difference in mean latency is your device's flush cost. Then:

```sh
echo "write through" | sudo tee /sys/block/nvme0n1/queue/write_cache
# re-run; the kernel now skips flushes entirely
echo "write back" | sudo tee /sys/block/nvme0n1/queue/write_cache
```

Explain in one sentence why the second run is faster and why you must never ship that setting.

---

### Lab 51.5 — Demonstrate the four durability traps

**(a) The missing directory `fsync`.** Write the unsafe and safe versions, then crash the machine between them.

```c
/* UNSAFE */
fd = open("data.tmp", O_WRONLY|O_CREAT|O_TRUNC, 0644);
write(fd, payload, len);
fsync(fd);
close(fd);
rename("data.tmp", "data");
/* crash here -> "data" may not exist */

/* SAFE: add */
int dfd = open(".", O_RDONLY|O_DIRECTORY);
fsync(dfd);
close(dfd);
```

To actually crash reproducibly, use a VM:

```sh
# In a QEMU VM (Ch. 04), from the host:
# run the unsafe program in a loop, then:
virsh destroy vmname            # or: kill -9 the qemu process
# Reboot and check whether data exists, with what contents.
# Repeat 50 times and count.
```

Better: use **`dm-flakey`** or the `dm-log-writes` target to simulate power loss *deterministically* without killing anything:

```sh
# dm-log-writes records every write and flush; replay to any point.
sudo dmsetup create logwrites --table \
  "0 $(blockdev --getsz /dev/sdb1) log-writes /dev/sdb1 /dev/sdc1"
mkfs.ext4 /dev/mapper/logwrites
mount /dev/mapper/logwrites /mnt
# run your program
sudo ./replay-log --log /dev/sdc1 --replay /dev/sdb1 --end-mark <N>
# then fsck and inspect the filesystem state at that exact point
```

`tools/testing/selftests/` and `xfstests`' `dm-log-writes` support (used by `generic/482` and friends) do exactly this. This is how filesystem crash consistency is actually tested, and building the harness once is worth far more than reasoning about it.

**(b) `fsync()` errors are one-shot.** Use `dm-flakey` or `dm-error` to inject a write failure:

```sh
SZ=$(blockdev --getsz /dev/sdb1)
sudo dmsetup create flaky --table "0 $SZ flakey /dev/sdb1 0 5 5"
# up 5s, down 5s: writes fail during the down window
```

Write a program that: writes, `fsync()`s (gets `-EIO`), then `fsync()`s again. Record what the second call returns on your kernel. Then read the page's contents — are they still what you wrote, or were the dirty pages dropped?

**(c) `O_DIRECT` is not durable.** From Lab 51.3's row 5, use `dm-log-writes` to confirm no flush was issued.

**(d) Nothing orders two files.** Write file A, write file B, crash. Demonstrate that B can be durable while A is not, even though A was written first.

---

### Lab 51.6 — Attribute latency to a layer

The core skill: given "storage is slow," determine *which layer*.

```sh
# Layer-by-layer latency, top to bottom:

# 1. Application-visible syscall latency
sudo funclatency-bpfcc -u vfs_write
sudo funclatency-bpfcc -u vfs_fsync

# 2. Filesystem layer
sudo ext4slower-bpfcc 0        # or xfsslower, btrfsslower
sudo ext4dist-bpfcc 10

# 3. Page cache effectiveness
sudo cachestat-bpfcc 1
sudo cachetop-bpfcc

# 4. Block layer queueing vs device service time
sudo biolatency-bpfcc -Q 10    # -Q INCLUDES OS queue time
sudo biolatency-bpfcc 10       #    excludes it -> device service time
# The difference is queueing. If it dominates, the problem is above the device.

# 5. Device
iostat -xdz 1
# key columns: r_await/w_await (total), aqu-sz (queue depth), %util, rareq-sz
```

The `biolatency -Q` vs plain comparison is the single most useful diagnostic here, and few people know it:

| Observation | Conclusion |
|---|---|
| `-Q` ≫ plain | requests are queueing in the kernel: too much concurrency, or the scheduler is throttling. Look at `nr_requests`, cgroup limits, the scheduler. |
| `-Q` ≈ plain, both high | the **device** is slow. Look at the device: GC, thermal throttling, RAID rebuild, bad sectors. |
| Both low but app is slow | the problem is above the block layer: fsync frequency, metadata, page cache misses, lock contention in the fs. |

Also:

```sh
# Is anything throttled by cgroups?
cat /sys/fs/cgroup/io.stat
cat /sys/fs/cgroup/*/io.max /sys/fs/cgroup/*/io.latency 2>/dev/null

# Is writeback throttling engaging? (Ch. 64)
cat /sys/block/nvme0n1/queue/wbt_lat_usec
sudo cat /sys/kernel/debug/block/nvme0n1/rqos/wbt/* 2>/dev/null

# Pressure stall information: the best single indicator
cat /proc/pressure/io
# "some avg10=..." -> fraction of time at least one task was stalled on I/O
```

`/proc/pressure/io` (Ch. 23 §T.9's PSI) is the metric to alert on: it directly measures "was work delayed by I/O," which is the thing you actually care about, unlike `%util` which is nearly meaningless on devices with internal parallelism.

---

### Lab 51.7 — Build a throwaway stack to experiment on

You need a storage stack you can destroy. Build one with loop devices, so no lab in Part 3 can hurt real data.

```sh
# 1. Backing files
mkdir -p ~/storage-lab && cd ~/storage-lab
for i in 0 1 2 3; do truncate -s 1G disk$i.img; done

# 2. Loop devices
for i in 0 1 2 3; do sudo losetup -f --show disk$i.img; done
losetup -a

# 3. An MD RAID-5 on three of them
sudo mdadm --create /dev/md0 --level=5 --raid-devices=3 \
           /dev/loop0 /dev/loop1 /dev/loop2
cat /proc/mdstat

# 4. LVM on top
sudo pvcreate /dev/md0
sudo vgcreate labvg /dev/md0
sudo lvcreate -L 1G -n labdata labvg

# 5. dm-crypt on top of that
sudo cryptsetup luksFormat /dev/labvg/labdata     # passphrase at the prompt
sudo cryptsetup open /dev/labvg/labdata labcrypt

# 6. A filesystem on top
sudo mkfs.ext4 /dev/mapper/labcrypt
sudo mkdir -p /mnt/lab && sudo mount /dev/mapper/labcrypt /mnt/lab

# Now look at what you built:
lsblk -t
sudo dmsetup ls --tree
findmnt /mnt/lab
```

Trace one write all the way down through five layers:

```sh
sudo bpftrace -e '
tracepoint:block:block_bio_queue {
	printf("%-12s dev=%d,%d sect=%-10llu %s\n",
	       comm, args->dev>>20, args->dev&0xfffff, args->sector, args->rwbs);
}' &
sudo dd if=/dev/zero of=/mnt/lab/x bs=4k count=1 oflag=direct conv=fsync
```

You should see one write become several bios on several devices — the fan-out of RAID-5's read-modify-write plus the crypt layer's re-issue. Count them and explain the write amplification.

Add deliberate faults:

```sh
# Fail one RAID member
sudo mdadm /dev/md0 --fail /dev/loop2
cat /proc/mdstat                      # degraded, still working
sudo mdadm /dev/md0 --remove /dev/loop2
sudo mdadm /dev/md0 --add /dev/loop3  # rebuild
watch cat /proc/mdstat

# Inject I/O errors at the bottom
sudo dmsetup create bad --table "0 2097152 error"
```

Teardown script (write it now; you will use it often):

```sh
#!/bin/bash
sudo umount /mnt/lab 2>/dev/null
sudo cryptsetup close labcrypt 2>/dev/null
sudo lvremove -f labvg 2>/dev/null
sudo vgremove -f labvg 2>/dev/null
sudo pvremove -f /dev/md0 2>/dev/null
sudo mdadm --stop /dev/md0 2>/dev/null
sudo mdadm --zero-superblock /dev/loop[0-3] 2>/dev/null
for d in $(losetup -a | grep storage-lab | cut -d: -f1); do
	sudo losetup -d $d
done
```

**Faster alternative for most Part 3 labs:** `null_blk` and `scsi_debug` give you configurable virtual block devices with no backing store:

```sh
sudo modprobe null_blk nr_devices=1 queue_mode=2 irqmode=0 \
              submit_queues=4 gb=16 bs=4096
sudo modprobe scsi_debug dev_size_mb=1024 sector_size=512 \
              every_nth=100 opts=2       # opts=2 injects medium errors
```

`null_blk` in particular is how the block layer itself is benchmarked, because it removes the device from the measurement entirely — Ch. 63 and 65 use it heavily.

---

## 3. Mastery drills

1. For each of the nine layers in §T.2, state in one sentence: the transformation it performs, the information it adds, and the information it destroys. Then identify which later chapter's complexity is caused by each destruction.

2. Prove that `write()` followed by a successful `read()` in another process is guaranteed to return the new data, and identify exactly which lock or invariant provides the guarantee. Then find the case where it does *not* hold (hint: `O_DIRECT` mixed with buffered I/O).

3. Derive the complete set of `fsync()` calls required to make "create a file, write it, rename it over an existing file, and delete a third file" crash-atomic as a group. Then argue whether it is possible at all with POSIX.

4. `REQ_PREFLUSH` is the only ordering primitive. Show that journalling's "write journal, flush, write commit, flush, write in place" cannot be reduced to fewer flushes without losing crash consistency — or find the reduction and state the assumption it needs.

5. A flush's cost is independent of the data flushed. Derive the optimal batching interval for a database committing transactions, given arrival rate λ, flush cost F, and a per-transaction latency budget L.

6. Compute the maximum IOPS achievable by a block layer that takes one spinlock per I/O, given a 100 ns uncontended acquire and realistic contention scaling (use Ch. 14's USL). Compare to the pre-blk-mq measured ceiling of ~800 K and explain the discrepancy.

7. The FTL and a log-structured filesystem both do garbage collection over the same data, uncoordinated. Quantify the resulting write amplification, and show how ZNS eliminates it.

8. `awupf` is typically 1 sector. Explain why MySQL's doublewrite buffer exists, and compute its write amplification. Then explain how a filesystem with data journalling or CoW makes it unnecessary.

9. Design an interface extension that would let a filesystem tell the block layer "these N writes must be atomic together." What must each layer below do, and which of them can actually implement it?

10. `%util` from `iostat` is nearly meaningless for NVMe. Prove it, using the definition of the metric and the device's internal parallelism, and state what to use instead.

11. A buffered write returns in ~1 µs; an `fsync` costs 200 µs–10 ms. Design an application-level durability scheme that achieves group-commit semantics across independent client requests, and state the latency/throughput trade it makes.

12. Trace the complete lifetime of the folio holding your written data: allocation, dirtying, writeback tagging, bio construction, DMA mapping, completion, and reclaim eligibility. Name the lock held at each transition.

13. You are handed a production machine where p99 write latency jumped from 2 ms to 400 ms with no configuration change. Write the ordered diagnostic procedure, with the specific command at each step and the branch each result implies. Use only tools from Lab 51.6.

---

## 4. Further reading

**Start here**

- `Documentation/filesystems/vfs.rst` ★★★ — the VFS contract; you will re-read this through Ch. 53–55.
- `Documentation/block/` ★★★ — especially `biodoc.rst` (historical but foundational), `blk-mq.rst`, `queue-sysfs.rst`, `writeback_cache_control.rst`. The last one is the authoritative text on §T.6 and is short.
- `Documentation/admin-guide/sysctl/vm.rst` ★★★ — every `dirty_*` knob explained.

**Books**

- Remzi and Andrea Arpaci-Dusseau, *Operating Systems: Three Easy Pieces*, Part 3 (Persistence) ★★★ — **free online**, and the single best introduction to every concept in this chapter: I/O devices, disks, RAID, files, FFS, journalling, LFS, flash, and crash consistency. Chapters 36–44. If you read one thing alongside Part 3, read this.
- Bovet & Cesati, *Understanding the Linux Kernel*, 3rd ed., Ch. 12–18 — dated (2.6) but the structural explanations of VFS, page cache, and the block layer remain the clearest long-form treatment.
- Robert Love, *Linux Kernel Development*, 3rd ed., Ch. 13–16 — VFS, block I/O, address spaces. Concise and correct.
- Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman, *Linux Device Drivers*, 3rd ed., Ch. 16 — block drivers; the API is obsolete but the model is not.
- Bruce Jacob, Spencer Ng, David Wang, *Memory Systems: Cache, DRAM, Disk* — the hardware-level treatment behind §T.7's numbers.

**Papers — the foundations Part 3 builds on**

- M. K. McKusick, W. N. Joy, S. J. Leffler, R. S. Fabry, "A Fast File System for UNIX," *ACM TOCS* 2(3), 1984 ★★★ — cylinder groups, block/fragment sizes; the origin of locality-based allocation and thus of ext2/3/4's block groups.
- M. Rosenblum and J. K. Ousterhout, "The Design and Implementation of a Log-Structured File System," SOSP 1991 ★★★ — LFS; the intellectual ancestor of F2FS, of every SSD FTL, and of much of Btrfs.
- D. A. Patterson, G. Gibson, R. H. Katz, "A Case for Redundant Arrays of Inexpensive Disks (RAID)," SIGMOD 1988 ★★★ — Ch. 67's foundation.
- G. R. Ganger, Y. N. Patt, "Metadata Update Performance in File Systems," OSDI 1994 — soft updates vs journalling.
- V. Prabhakaran et al., "IRON File Systems," SOSP 2005 ★★★ — a taxonomy of how filesystems respond to disk failures; sobering and still relevant.
- T. S. Pillai et al., "All File Systems Are Not Created Equal: On the Complexity of Crafting Crash-Consistent Applications," OSDI 2014 ★★★ — **read this one**. It systematically documents §T.5's traps and shows that most real applications get them wrong. The ALICE tool is in it.
- R. Alagappan et al., "Protocol-Aware Recovery for Consensus-Based Storage," FAST 2018 — what to do when `fsync` fails.
- N. Agrawal et al., "Design Tradeoffs for SSD Performance," USENIX ATC 2008 — the FTL design space of §T.8.
- M. Bjørling et al., "ZNS: Avoiding the Block Interface Tax for Flash-based SSDs," USENIX ATC 2021 ★★★ — the quantified argument for widening the waist.
- J. Axboe, "Linux Block IO — present and future," OLS 2004, and M. Bjørling et al., "Linux Block IO: Introducing Multi-queue SSD Access on Multi-core Systems," SYSTOR 2013 ★★★ — the before and after of blk-mq (§T.7(b)).

**LWN — the historical record**

- "Ensuring data reaches disk" (2011) ★★★ — the clearest short explanation of §T.5 anywhere.
- "PostgreSQL's fsync() surprise" (2018) ★★★ — fsyncgate; read with the follow-up "Improved block-layer error handling."
- "The best way to throw away data" / barrier removal series (2010) — how `REQ_FLUSH`/`REQ_FUA` replaced barriers, and why.
- "The multiqueue block layer" (2013) and "Block layer introduction" parts 1–2 (2017) ★★★
- "Toward better performance on large I/O" / "Large folios in the page cache" (2021–2024)
- "Zoned namespaces" and "Zone append" coverage (2020–2021)
- "Write-behind and writeback throttling" (2016)
- The annual LSFMM (Linux Storage, Filesystem, MM and BPF) summit coverage ★★★ — the best single source for what the storage community is currently arguing about.

**Tools — install these now; Part 3 uses them throughout**

```sh
# Tracing
bcc-tools / bpfcc-tools      # biosnoop, biolatency, biotop, cachestat,
                             # ext4slower, xfsslower, filetop, funclatency
bpftrace
trace-cmd, kernelshark
blktrace / blkparse / btt / iowatcher

# Benchmarking
fio                          # the standard; learn it properly
                             # `fio --showcmd`, `fio --parse-only`
ioping                       # latency probe

# Inspection
util-linux: lsblk, blkid, findmnt, blkdiscard, blkzone, fallocate
nvme-cli, smartmontools, sdparm, hdparm, sg3-utils
e2fsprogs: dumpe2fs, debugfs, e2image
xfsprogs: xfs_db, xfs_info, xfs_logprint, xfs_spaceman
btrfs-progs: btrfs inspect-internal, btrfs-debug-tree
lvm2, mdadm, cryptsetup, dmsetup

# Testing
xfstests (fstests)           # the filesystem conformance suite
blktests                     # the block layer test suite
dm-flakey, dm-log-writes, dm-dust, dm-error   # fault injection targets
```

**Kernel source reading order for Part 3**

Do not read top-down. Read one complete path first, then branch:

1. `block/bio.c` — `bio_alloc`, `bio_add_page`, `bio_endio`, `bio_split`
2. `block/blk-core.c` — `submit_bio`, `submit_bio_noacct`
3. `block/blk-mq.c` — `blk_mq_submit_bio`, `blk_mq_dispatch_rq_list`, `blk_mq_end_request`
4. `mm/filemap.c` — `generic_perform_write`, `filemap_read`, `filemap_fault`
5. `mm/page-writeback.c` — `balance_dirty_pages`, `wb_writeback`
6. `fs/iomap/buffered-io.c` and `direct-io.c` — the modern mapping layer
7. Then a filesystem of your choice, using the above as the skeleton it hangs on.

---

→ Next: [52-page-cache-writeback.md](52-page-cache-writeback.md)

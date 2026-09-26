# Chapter 67 — MD RAID and software RAID internals

> **Goal:** Understand redundancy at the block layer: what each RAID level actually buys, why RAID 5's write hole exists and what closes it, how MD's personality architecture separates the common machinery from the per-level algorithms, the stripe cache that makes RAID 5 work at all, resync versus recovery versus check as three distinct operations, bitmaps as the mechanism that turns hours of resync into seconds, why URE rates make RAID 5 a bad bet on large modern drives, and how to diagnose a degraded array without making it worse. By the end you can read `drivers/md/`, interpret `/proc/mdstat` and `mdadm --examine` fluently, and reason correctly about what a given array will survive.

---

## Theory & First Principles

### T.0 — Start here: RAID is not a backup, and RAID-5 can lose data with no disk failure

Start with what RAID actually promises, because it is narrower than people think:

> **RAID converts *whole-device failure* into *degraded but correct operation*.** That is
> all. It does not protect against deletion, corruption by software, ransomware, controller
> bugs, or a fire.

```
  RAID 0   stripe      N x capacity, N x speed, N x failure probability. No redundancy.
  RAID 1   mirror      1 x capacity, survives N-1 failures, reads scale.
  RAID 5   N-1 data + 1 parity      survives 1 failure.
  RAID 6   N-2 data + 2 parity      survives 2 failures.
  RAID 10  mirrored stripes         the boring answer that is usually right.
```

**Parity is just XOR, and that is worth doing by hand once:**

```
  D1 = 1011      P = D1 ^ D2 ^ D3 = 1011 ^ 0110 ^ 1101 = 0000
  D2 = 0110
  D3 = 1101      lose D2?  D2 = D1 ^ D3 ^ P = 1011 ^ 1101 ^ 0000 = 0110  ✓
```

N disks of protection for the cost of one. Which sounds like a free lunch, so **now the
bill.**

**Bill #1 — the small-write penalty.** Change 4 KiB in a RAID-5 stripe:

```
  read old data  +  read old parity        (2 reads)
  P_new = P_old ^ D_old ^ D_new
  write new data +  write new parity       (2 writes)

  -> ONE 4 KiB application write = 4 device I/Os.
```

This is why RAID-5 is poor for databases and why RAID-10 wins on random-write workloads
despite "wasting" half the capacity.

**Bill #2 — the write hole, and this is the one to understand properly.** Those two writes
are not atomic:

```
   write D_new  -> lands
   write P_new  -> POWER LOSS. Never lands.

   Now: the stripe's parity does not match its data. Nothing reports an error;
        every disk is healthy and every read returns fine.

   Later: a disk fails. You reconstruct from parity.
          The parity is STALE, so you reconstruct WRONG DATA
          -- for a block that was never even being written.

   Silent corruption of UNRELATED data, from a power loss with ZERO disk failures.
```

**That is the sharpest illustration in this book of the end-to-end argument** (Ch. 62 §T.1).
The RAID layer cannot fix it alone: it does not know which stripes hold live data, and it has
no checksum to tell it which of "data" and "parity" is the correct one. The fixes each add
information from somewhere:

| Fix | Mechanism | Cost |
|---|---|---|
| `md` **PPL** / journal device | log the intent before writing (Ch. 61 §T.0 again) | a write to the log per stripe write |
| **NVRAM-backed** controller | make the pair of writes effectively atomic | hardware, and a new single point of failure |
| **ZFS RAID-Z** | variable-width **full-stripe** writes only | the hole cannot exist by construction (Ch. 60 §T.0) |
| **RAID-10** | no parity at all | 50% of capacity |

**Bill #3 — rebuild is the dangerous window, and the arithmetic has gotten worse.** Rebuilding
a failed 20 TB disk means reading *every sector of every surviving disk*, for many hours,
under full load — the single most stressful thing you can do to a set of aged drives, at
exactly the moment you have no redundancy left. With a nominal unrecoverable-read-error rate
of 1 in 10^14 bits, reading ~10^14 bits per rebuild makes a second failure during the rebuild
an *expected* event, not a freak one. **This is why RAID-6 displaced RAID-5 for large drives,
and why declustered/distributed parity is what modern large-scale systems use instead.**

**The habit to build:** for any redundancy scheme, ask (1) what failure does it actually
cover, (2) what is the update cost, (3) what happens on a crash mid-update, and (4) how bad
is the recovery window. Those four questions cover RAID, erasure coding, and database
replication alike.

```bash
cat /proc/mdstat                          # arrays, state, rebuild progress
sudo mdadm --detail /dev/md0
sudo mdadm --examine /dev/sd[b-e]1        # per-device superblocks
echo check | sudo tee /sys/block/md0/md/sync_action   # scrub for mismatches
cat /sys/block/md0/md/mismatch_cnt        # should be 0
```

---

### T.1 What RAID actually provides

RAID was named for "Inexpensive Disks" in Patterson, Gibson, and Katz's 1988 paper, arguing that arrays of cheap disks could outperform and outlast single expensive ones. The "I" is now usually read as "Independent," which is less honest about the original economics but more accurate about the current use.

What it gives you, precisely:

| Property | Mechanism |
|---|---|
| **Availability** | survive N device failures without downtime |
| **Capacity aggregation** | one address space from many devices |
| **Throughput** | parallel access across devices |

What it does **not** give you, and this list matters:

| Not provided | Why |
|---|---|
| **Backup** | `rm -rf` is replicated instantly to every mirror |
| **Data integrity** | RAID detects *device* failure, not *data* corruption (§T.8) |
| **Protection from correlated failure** | same batch, same rack, same power supply, same firmware bug |
| **Protection from operator error** | the most common cause of data loss |
| **Consistency across a crash** | see §T.5's write hole |

"RAID is not a backup" is repeated so often that it has lost force. State it precisely instead: **RAID protects against one specific failure mode — a device ceasing to respond — and nothing else.**

### T.2 The levels, and what each is for

| Level | Devices | Capacity | Survives | Read | Write | Use |
|---|---|---|---|---|---|---|
| **0** | N | N | **nothing** | N× | N× | scratch, cache, speed only |
| **1** | N | 1 | N−1 | N× | 1× | boot, small critical |
| **4** | N | N−1 | 1 | (N−1)× | **parity disk bottleneck** | obsolete |
| **5** | N ≥ 3 | N−1 | 1 | (N−1)× | RMW penalty | **see §T.9** |
| **6** | N ≥ 4 | N−2 | 2 | (N−2)× | worse RMW | large arrays |
| **10** | N (even) | N/2 | ≥1 | N× | N/2× | **databases, general** |
| **linear** | N | N | nothing | 1× | 1× | concatenation only |

Three things that are commonly misunderstood:

**(a) RAID 1 read scaling is real but limited.** MD can serve different reads from different mirrors, so random read throughput scales with N. Sequential read does not scale as well, because MD's read balancing prefers the disk whose head is nearest — good for rotating disks, irrelevant for SSDs. `/sys/block/mdX/md/rdN/` has no round-robin knob; the heuristic is in `read_balance()`.

**(b) RAID 10 in Linux is not "RAID 1 + RAID 0".** MD's `raid10` is a distinct personality supporting odd device counts and three layouts:

| Layout | Meaning |
|---|---|
| `n2` (near) | two copies on adjacent devices — the classic layout |
| `f2` (far) | copies on opposite halves of each device — **best sequential read** |
| `o2` (offset) | copies offset by one stripe — a compromise |

`f2` is genuinely clever: the first copy of all data occupies the outer (fast) tracks of every disk, so sequential reads get RAID-0 throughput from the outer tracks. Writes pay for the long seek. On rotating media this is a substantial win and is underused.

**(c) RAID 6's second parity is not just "another XOR."** It uses Reed-Solomon over GF(2⁸): P = XOR of all data blocks, Q = XOR of data blocks each multiplied by a distinct generator power. Recovering two failed blocks requires solving a 2×2 linear system in the field, which is why RAID 6 recovery is materially more expensive than RAID 5 recovery. §T.4 covers the implementation.

### T.3 The personality architecture

MD separates common machinery from per-level algorithms:

```
        md.c  (common)
   - device discovery and assembly
   - superblock read/write
   - /proc/mdstat, sysfs
   - resync/recovery scheduling
   - bitmap management
   - hot-add, hot-remove, fail
   - sysfs and ioctl interface
             |
   +---------+---------+---------+---------+
   |         |         |         |         |
 raid0    raid1     raid10    raid456   linear
             \                    |
          (make_request, sync_request, error, spare_active, ...)
```

```c
struct md_personality {
	char *name;
	int level;
	struct list_head list;
	struct module *owner;

	bool __must_check (*make_request)(struct mddev *mddev, struct bio *bio);
	int (*run)(struct mddev *mddev);
	int (*start)(struct mddev *mddev);
	void (*free)(struct mddev *mddev, void *priv);
	void (*status)(struct seq_file *seq, struct mddev *mddev);
	void (*error_handler)(struct mddev *mddev, struct md_rdev *rdev);
	int (*hot_add_disk)(struct mddev *mddev, struct md_rdev *rdev);
	int (*hot_remove_disk)(struct mddev *mddev, struct md_rdev *rdev);
	int (*spare_active)(struct mddev *mddev);
	sector_t (*sync_request)(struct mddev *mddev, sector_t sector_nr,
				 sector_t max_sector, int *skipped);
	int (*resize)(struct mddev *mddev, sector_t sectors);
	sector_t (*size)(struct mddev *mddev, sector_t sectors, int raid_disks);
	int (*check_reshape)(struct mddev *mddev);
	int (*start_reshape)(struct mddev *mddev);
	void (*finish_reshape)(struct mddev *mddev);
	void (*update_reshape_pos)(struct mddev *mddev);
	void (*quiesce)(struct mddev *mddev, int quiesce);
	void *(*takeover)(struct mddev *mddev);
	int (*change_consistency_policy)(struct mddev *mddev, const char *buf);
};
```

MD is **bio-based** (Ch. 63 §T.8): `make_request` receives raw bios and issues new ones to members. It does not use blk-mq, because merging and scheduling should happen on the member devices, not on the array.

`takeover` is the mechanism behind online level conversion: a RAID 1 array can become a RAID 5 array with two devices, which can then be reshaped to more. The personality provides a function that reinterprets the existing layout as its own.

### T.4 RAID 5/6: the read-modify-write problem

Writing one block to a RAID 5 stripe requires updating parity. There are two ways:

**Read-modify-write (RMW)** — for small writes:

```
read old data block, read old parity
new_parity = old_parity XOR old_data XOR new_data
write new data, write new parity
```

Cost: **2 reads + 2 writes** for one logical block write.

**Reconstruct-write (RCW)** — for large writes:

```
read the OTHER data blocks in the stripe
new_parity = XOR of all data blocks (new and unchanged)
write new data, write new parity
```

Cost: N−2 reads + 2 writes, but if you are writing the *whole* stripe, **zero reads**.

MD chooses per stripe based on how much of it is being written. The implication is the single most important performance fact about RAID 5:

> **A full-stripe write costs nothing extra. A partial-stripe write costs 4 I/Os per block.**

Hence:

- Filesystem stripe alignment matters enormously. `mkfs.xfs` reads `optimal_io_size` and sets `sunit`/`swidth`; `mkfs.ext4` sets `stride`/`stripe-width`. If these are wrong, every write is partial.
- Small random writes are catastrophic on RAID 5. A database doing 8 KiB random writes on a 5-disk RAID 5 with 512 KiB chunks does 4 physical I/Os per logical one.
- Chunk size is a real trade: large chunks mean fewer full-stripe writes for a given I/O size; small chunks mean more parity computation per byte.

RAID 6's Q parity makes RMW worse: three reads and three writes, plus GF multiplication rather than XOR.

**The stripe cache** exists to mitigate this. MD caches stripes in memory (`struct stripe_head`) so that:

- Several small writes to the same stripe can be merged into one parity update.
- The old data and parity needed for RMW may already be resident.
- Reads can be served from cached stripes.

```sh
cat /sys/block/md0/md/stripe_cache_size     # default 256 (× 4 KiB × N devices)
cat /sys/block/md0/md/stripe_cache_active
```

Increasing `stripe_cache_size` from 256 to 4096 or 8192 often doubles write throughput on RAID 5/6 — it is the single most effective MD tunable. The memory cost is `stripe_cache_size × 4 KiB × nr_devices`, so 8192 on a 6-disk array is 192 MiB.

### T.5 The write hole

This is the fundamental correctness problem with parity RAID, and it is worth stating precisely.

A stripe write is **not atomic**. It involves writing data to one device and parity to another. If power fails between them:

```
Before:  D1 D2 D3 P     (consistent: P = D1^D2^D3)
Write D2 -> D2'
Crash after writing D2' but before writing P'

After:   D1 D2' D3 P    (INCONSISTENT: P != D1^D2'^D3)
```

The array is now inconsistent, and **nothing detects it on its own**. If a device then fails, reconstruction uses the stale parity and produces **wrong data for a block that was never written** — silent corruption of data the application believed was safe.

Note the shape of the problem: the corruption is not in the block you were writing (that one you know is uncertain), but in a *different* block in the same stripe, which you never touched.

Three mitigations:

**(a) Journal / write-intent log (`--write-journal`).** MD can log stripe writes to a fast device before applying them. On recovery, incomplete stripes are replayed. This closes the hole completely, at the cost of writing everything twice and needing a reliable (ideally mirrored) journal device.

**(b) PPL (Partial Parity Log).** A lighter scheme: store just enough partial parity in the member devices' metadata space to reconstruct consistency after a crash. No separate device needed, much less overhead than a full journal, and it closes the hole for RAID 5. This is the practical answer for most deployments and is what `mdadm --consistency-policy=ppl` selects.

**(c) A full resync after an unclean shutdown.** The historical default: recompute all parity. It closes the window eventually but takes hours, and **during the resync the array is still vulnerable** — if a device fails before the resync passes that region, you reconstruct from bad parity.

**(d) Battery-backed cache / PLP devices.** If the writes cannot be lost, the hole cannot open. This is how hardware RAID controllers addressed it.

Compare with the CoW filesystems: ZFS's RAID-Z eliminates the hole by construction with variable-width stripes (Ch. 60 §T.6(d)); btrfs's RAID 5/6 kept fixed-width stripes and therefore kept the hole (Ch. 59 §T.9). **This is the clearest example in Part 3 of a design decision with a direct correctness consequence.**

### T.6 Three sync operations, often confused

| Operation | Trigger | What it does | Array state |
|---|---|---|---|
| **resync** | unclean shutdown, or new array | make redundancy consistent (recompute parity, or copy to mirrors) | **optimal** |
| **recovery** | a device was replaced | rebuild the *new* device's contents from the others | **degraded** |
| **check** | manual / scheduled | read everything, compare, **count mismatches without fixing** | optimal |
| **repair** | manual | read everything, fix mismatches (by rewriting parity) | optimal |
| **reshape** | `--grow` | change geometry (devices, level, chunk size) | special |

```sh
echo check  > /sys/block/md0/md/sync_action
echo repair > /sys/block/md0/md/sync_action
echo idle   > /sys/block/md0/md/sync_action     # stop
cat /sys/block/md0/md/mismatch_cnt
```

**`mismatch_cnt` after a `check` is the number of inconsistent sectors found.** Interpreting it correctly matters:

- On **RAID 1/10**, a non-zero count means the mirrors genuinely differ, which indicates a real problem — except for swap and some `mmap` cases where the kernel may legitimately write different data to each mirror while a page is being modified concurrently. This is a known false positive.
- On **RAID 5/6**, a non-zero count after an unclean shutdown is expected (§T.5) and a `repair` fixes it. A non-zero count on a cleanly-shut-down array indicates hardware trouble.
- `repair` on RAID 5 **assumes the data is correct and rewrites the parity.** If the data is what is corrupt, repair writes bad parity over good. RAID cannot tell which is wrong (§T.8).

Scheduled checks are standard practice (Debian and RHEL both ship monthly cron jobs) because they exercise every sector, finding latent bad blocks before a rebuild needs them — which is §T.9's whole problem.

### T.7 Bitmaps: the difference between seconds and hours

Without a write-intent bitmap, any interruption requires a full resync: read every sector of every device, recompute. On a 10 TB array that is many hours.

A **write-intent bitmap** records which regions have in-flight writes:

```
before a write: set the bit for its region, flush the bitmap
after the write completes: (lazily) clear the bit
```

After an unclean shutdown, only regions with set bits need resyncing — typically seconds instead of hours.

The cost is real: every write to a clean region requires a bitmap update and a flush before the data write. MD mitigates this by:

- **Lazy clearing**: bits are cleared only after a delay (`/sys/block/md0/md/bitmap/time_base`), so repeated writes to the same region set the bit once.
- **Coarse granularity**: one bit covers a chunk (default sized so the bitmap has a few thousand bits). Larger chunks mean fewer updates and more resync work.

```sh
mdadm --create ... --bitmap=internal --bitmap-chunk=64M
cat /sys/block/md0/md/bitmap/chunksize
cat /sys/block/md0/md/bitmap/backlog
```

Bitmap types:

| Type | Stored | Note |
|---|---|---|
| `internal` | in the member devices' metadata area | the usual choice |
| `none` | — | fastest writes, full resync on any interruption |
| file | a file on another filesystem | **deprecated and dangerous** |
| `clustered` | shared, for cluster-md | multi-host |

The **same bitmap enables `--re-add`**: a device that was temporarily removed (a cable knock, a transient controller failure) can be re-added and only the regions written while it was absent are copied. Without a bitmap, re-adding means a full rebuild.

This is an instance of a general pattern: **a small amount of persistent metadata recording intent converts an O(size) recovery into an O(changes) one.** The same idea as a journal (Ch. 61), as btrfs's generation numbers (Ch. 59 §T.3), and as incremental backup.

### T.8 What RAID cannot detect

MD trusts its devices. If a device returns data without an error, MD believes it.

| Failure | Detected? |
|---|---|
| Device does not respond | **yes** — the device returns an error |
| Device returns a read error (URE) | **yes** — reconstruct and rewrite |
| Device returns **wrong data silently** | **no** |
| Misdirected write (written to the wrong sector) | **no** |
| Lost write (acknowledged, never happened) | **no** |
| Torn write | **no** |
| Firmware bug returning stale data | **no** |

For RAID 1 and 10, `check` will *notice* a mismatch, but MD cannot tell which copy is correct. For RAID 5, `check` notices a parity mismatch, and `repair` guesses that the data is right.

This is exactly why ZFS and btrfs checksum data (Ch. 59 §T.6, Ch. 60 §T.6): **with a checksum you know which copy is correct; without one, redundancy only tells you that something is wrong.**

Bairavasundaram et al.'s FAST 2008 study of 1.5 million drives over 41 months found latent sector errors in 3.45 % of drives and silent corruption (checksum mismatches detected by NetApp's block checksums) in 0.5 % of nearline drives per year. Those numbers are old and drives have changed, but the phenomenon has not.

The mitigation on Linux without changing filesystem: **`dm-integrity` beneath MD.** It adds per-sector checksums, so a device returning wrong data produces a read error, which MD can then handle by reconstructing from the others. The cost is an extra write per sector and reduced capacity.

### T.9 The URE argument, honestly

The widely-repeated claim is that RAID 5 is "dead" because rebuilding a large array is statistically certain to hit an unrecoverable read error.

The arithmetic: a consumer drive specification of `<1 URE per 10^14 bits read` means one error per ~12.5 TB read. Rebuilding a 6×4 TB RAID 5 requires reading 20 TB. Naively, that is a 1.6× expected URE count — so the rebuild "will" fail.

**This argument is directionally right and quantitatively wrong**, and it is worth being precise:

1. The 10^14 figure is a **specification floor**, not a measured rate. Field studies (Schroeder & Gibson; Elerath & Pecht) consistently measure rates 10–100× better.
2. A URE during rebuild on RAID 5 loses **one stripe**, not the array. MD marks the block bad and continues; older implementations and many hardware controllers aborted the whole rebuild, which is where the "array is dead" framing came from.
3. UREs are **not independent**; they cluster, and the correlation matters more than the average.
4. **Scrubbing changes everything.** A monthly `check` finds and rewrites latent bad sectors while the array is still redundant. An array that is scrubbed regularly has far fewer latent errors waiting for a rebuild.

The honest conclusion:

| Situation | Verdict |
|---|---|
| Small arrays (< 4 TB total), scrubbed, with backups | RAID 5 is fine |
| Large arrays of large drives | **RAID 6 or RAID 10** |
| Any array, unscrubbed | you are gambling |
| Any array, unmonitored | you will discover the failure during the second failure |
| Any array without backups | you have not understood §T.1 |

The second-order effect matters more than the URE one: **rebuild time.** A 20 TB drive at 150 MB/s takes 37 hours to rebuild, during which the array is degraded and every remaining drive is under sustained load — exactly the conditions that expose correlated failures. RAID 6 exists to survive that window; RAID 10 exists to make the window shorter (rebuilding one mirror is a straight copy, not a reconstruction from all devices).

### T.10 MD versus dm-raid versus hardware RAID

| | **MD** | **dm-raid** | **hardware RAID** |
|---|---|---|---|
| Implementation | kernel personalities | MD, wrapped as a dm target (Ch. 66 §T.7) | firmware on a card |
| Management | `mdadm` | `lvm2` | vendor tools |
| Metadata | MD superblock | LVM metadata + MD superblock | proprietary |
| Portability | any Linux machine | any Linux machine | **the same card model** |
| CPU cost | XOR/RS on the host | same | offloaded |
| Battery-backed cache | no | no | **yes** — closes §T.5 |
| Visibility | complete | complete | **whatever the vendor exposes** |
| Recovery from controller death | n/a | n/a | **need an identical card** |

MD's XOR and Reed-Solomon implementations are heavily optimised (AVX2, AVX-512, NEON) and benchmarked at boot:

```
raid6: avx2x4   gen() 15234 MB/s
raid6: using algorithm avx2x4 gen() 15234 MB/s
xor: using function: avx (22000.000 MB/sec)
```

At 15 GB/s of parity generation, the CPU is not the bottleneck for any realistic array. **The historical argument for hardware RAID — parity offload — is obsolete.** The remaining argument is the battery-backed write cache, which genuinely closes the write hole and improves small-write latency. Against that: proprietary metadata, a single point of failure that requires an identical replacement card, and much worse observability.

The modern consensus for Linux: **MD (or dm-raid under LVM) unless you specifically need a BBU**, and if you need a BBU, consider whether PLP NVMe devices plus MD gets you the same property with better properties everywhere else.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `drivers/md/md.c` ★★★ | §T.3's common layer; ~10000 lines |
| `drivers/md/md.h` ★★★ | `mddev`, `md_rdev`, `md_personality` |
| `drivers/md/md-bitmap.c` ★★★ | §T.7 |
| `drivers/md/raid0.c` | striping; the simplest personality |
| `drivers/md/raid1.c` ★★★ | mirroring, `read_balance`, error handling |
| `drivers/md/raid10.c` | §T.2's layouts |
| `drivers/md/raid5.c` ★★★ | **§T.4's stripe cache and state machine**; ~9000 lines |
| `drivers/md/raid5-ppl.c` | §T.5(b) |
| `drivers/md/raid5-cache.c` | §T.5(a), the write journal |
| `drivers/md/md-linear.c`, `md-multipath.c` | trivial / deprecated |
| `lib/raid6/` ★★★ | the Reed-Solomon implementations, per architecture |
| `crypto/xor.c`, `include/asm-generic/xor.h` | XOR, benchmarked at boot |
| `drivers/md/md-autodetect.c` | legacy partition-type autodetection |
| `Documentation/admin-guide/md.rst` ★★★ | the sysfs interface, exhaustively |

### 1.2 `mddev` and `md_rdev`

```c
struct mddev {
	void				*private;
	struct md_personality		*pers;
	dev_t				unit;
	int				md_minor;
	struct list_head		disks;
	unsigned long			flags;
	unsigned long			sb_flags;

	int				suspended;
	struct percpu_ref		active_io;
	int				ro;

	struct gendisk			*gendisk;
	struct kobject			kobj;

	/* Superblock content */
	int				major_version, minor_version, patch_version;
	int				persistent;
	int				external;
	char				metadata_type[17];
	int				chunk_sectors;
	time64_t			ctime, utime;
	int				level, layout;
	char				clevel[16];
	int				raid_disks;
	int				max_disks;
	sector_t			dev_sectors;
	sector_t			array_sectors;
	int				external_size;
	__u64				events;

	/* Reshape state */
	sector_t			reshape_position;
	int				delta_disks, new_level, new_layout;
	int				new_chunk_sectors;
	int				reshape_backwards;

	struct md_thread __rcu		*thread;      /* the per-array thread */
	struct md_thread __rcu		*sync_thread; /* resync/recovery */
	char				*last_sync_action;
	sector_t			curr_resync;
	sector_t			curr_resync_completed;
	unsigned long			resync_mark;
	sector_t			resync_mark_cnt;
	sector_t			resync_max_sectors;
	atomic64_t			resync_mismatches;   /* mismatch_cnt */
	...
	struct bitmap_operations	*bitmap_ops;
	void				*bitmap;
	...
	atomic_t			recovery_active;
	wait_queue_head_t		recovery_wait;
	sector_t			recovery_cp;
	sector_t			resync_min, resync_max;
	...
};

struct md_rdev {
	struct list_head	same_set;
	sector_t		sectors;
	struct mddev		*mddev;
	int			last_events;
	struct block_device	*meta_bdev;
	struct block_device	*bdev;
	struct file		*bdev_file;
	struct page		*sb_page, *bb_page;
	int			sb_loaded;
	__u64			sb_events;
	sector_t		data_offset;
	sector_t		new_data_offset;
	sector_t		sb_start;
	int			sb_size;
	int			preferred_minor;
	struct kobject		kobj;
	unsigned long		flags;    /* Faulty, In_sync, WriteMostly, ... */
	wait_queue_head_t	blocked_wait;
	int			desc_nr;
	int			raid_disk;
	int			new_raid_disk;
	int			saved_raid_disk;
	sector_t		recovery_offset;
	atomic_t		nr_pending;
	atomic_t		read_errors;
	atomic_t		corrected_errors;
	struct badblocks	badblocks;    /* T.9's per-device bad block list */
	...
};
```

`struct badblocks` is worth noting: MD maintains a per-device list of known-bad sectors. A URE during a rebuild records the sector as bad rather than failing the whole device, so the array survives (§T.9's point 2). Visible at `/sys/block/mdX/md/rdN/bad_blocks`.

The rdev flags encode the state machine:

```c
enum flag_bits {
	Faulty,			/* device is known to have a fault */
	In_sync,		/* device is in_sync with the rest */
	Bitmap_sync,		/* the bitmap must be resynced */
	WriteMostly,		/* avoid reading if possible */
	AutoDetected,
	Blocked,		/* an error occurred; do not write */
	WriteErrorSeen,
	FaultRecorded,
	BlockedBadBlocks,
	WantReplacement,	/* replace this device when a spare is free */
	Replacement,		/* this IS the replacement */
	Candidate,
	Journal,		/* T.5(a)'s journal device */
	ClusterRemove,
	ExternalBbl,
	FailFast,		/* fail quickly rather than retrying */
	LastDev,		/* the last working device; do not fail it */
	CollisionCheck,
	Nonrot,
};
```

`LastDev` is a safety property: MD refuses to fail the last working device of a mirror, because doing so would destroy the array rather than degrade it.

`WriteMostly` is a useful and underused feature: mark a slow mirror (a network device, a slow disk) so reads never go to it but writes still do. `mdadm --write-mostly --add`.

### 1.3 RAID 1's read balancing

```c
static int read_balance(struct r1conf *conf, struct r1bio *r1_bio, int *max_sectors)
{
	const sector_t this_sector = r1_bio->sector;
	int sectors;
	int best_good_sectors;
	int best_disk, best_dist_disk, best_pending_disk;
	int disk;
	sector_t best_dist;
	unsigned int min_pending;
	struct md_rdev *rdev;
	...
	for (disk = 0 ; disk < conf->raid_disks * 2 ; disk++) {
		sector_t dist;
		unsigned int pending;

		rdev = conf->mirrors[disk].rdev;
		if (r1_bio->bios[disk] == IO_BLOCKED || rdev == NULL ||
		    test_bit(Faulty, &rdev->flags))
			continue;
		if (!test_bit(In_sync, &rdev->flags) &&
		    rdev->recovery_offset < this_sector + sectors)
			continue;
		if (test_bit(WriteMostly, &rdev->flags)) {
			/* T.1.2: never read from these unless nothing else works */
			if (best_dist_disk < 0) {
				...
				best_dist_disk = disk;
			}
			continue;
		}
		/* Does it have a bad block here? */
		if (is_badblock(rdev, this_sector, sectors, &first_bad, &bad_sectors)) {
			...
		}

		pending = atomic_read(&rdev->nr_pending);
		dist = abs(this_sector - conf->mirrors[disk].head_position);

		/* Non-rotational: just pick the least busy. */
		if (!test_bit(Nonrot, &rdev->flags) && ...) {
			...
		}
		if (dist < best_dist) {
			best_dist = dist;
			best_dist_disk = disk;
		}
		if (min_pending > pending) {
			min_pending = pending;
			best_pending_disk = disk;
		}
	}
	...
	/* Sequential read: keep going on the same disk to preserve readahead. */
	if (conf->mirrors[best_disk].next_seq_sect == this_sector ||
	    dist == 0)
		...
}
```

Three heuristics, and each addresses something specific: head position (seek cost on rotating disks), pending count (load balance), and sequential continuation (preserve the device's own readahead, §T.2's deceptive-idleness cousin from Ch. 64 §T.3).

### 1.4 RAID 5's stripe state machine

```c
struct stripe_head {
	struct hlist_node	hash;
	struct list_head	lru;
	struct llist_node	release_list;
	struct r5conf		*raid_conf;
	short			generation;
	sector_t		sector;        /* the stripe's position */
	short			pd_idx;        /* parity disk index */
	short			qd_idx;        /* Q disk index (RAID 6) */
	short			ddf_layout;
	short			hash_lock_index;
	unsigned long		state;         /* STRIPE_* bits */
	atomic_t		count;
	int			bm_seq;
	int			disks;
	int			overwrite_disks;
	enum check_states	check_state;
	enum reconstruct_states reconstruct_state;
	spinlock_t		stripe_lock;
	int			cpu;
	struct r5worker_group	*group;
	struct stripe_head	*batch_head;
	spinlock_t		batch_lock;
	struct list_head	batch_list;
	...
	struct stripe_operations {
		int		target, target2;
		enum sum_check_flags zero_sum_result;
	} ops;
	struct r5dev {
		struct bio	req, rreq;
		struct bio_vec	vec, rvec;
		struct page	*page, *orig_page;
		unsigned int	offset;
		struct bio	*toread, *read, *towrite, *written;
		sector_t	sector;
		unsigned long	flags;
		u32		log_checksum;
		unsigned short	write_hint;
	} dev[];
};
```

The state bits:

```c
#define STRIPE_ACTIVE		0
#define STRIPE_HANDLE		1
#define STRIPE_SYNC_REQUESTED	2
#define STRIPE_SYNCING		3
#define STRIPE_INSYNC		4
#define STRIPE_REPLACED		5
#define STRIPE_PREREAD_ACTIVE	6
#define STRIPE_DELAYED		7
#define STRIPE_DEGRADED		8
#define STRIPE_BIT_DELAY	9
#define STRIPE_EXPANDING	10
#define STRIPE_EXPAND_SOURCE	11
#define STRIPE_EXPAND_READY	12
#define STRIPE_IO_STARTED	13
#define STRIPE_FULL_WRITE	14   /* T.4: no RMW needed */
#define STRIPE_BIOFILL_RUN	15
#define STRIPE_COMPUTE_RUN	16
#define STRIPE_ON_UNPLUG_LIST	17
#define STRIPE_DISCARD		18
#define STRIPE_ON_RELEASE_LIST	19
#define STRIPE_BATCH_READY	20
#define STRIPE_BATCH_ERR	21
```

And the RMW/RCW decision of §T.4:

```c
static int handle_stripe_dirtying(struct r5conf *conf,
				  struct stripe_head *sh,
				  struct stripe_head_state *s,
				  int disks)
{
	int rmw = 0, rcw = 0, i;
	sector_t recovery_cp = conf->mddev->recovery_cp;

	/* If the array is not in sync, RMW would use unreliable parity. */
	if (conf->rmw_level == PARITY_DISABLE_RMW ||
	    (recovery_cp < MaxSector && sh->sector >= recovery_cp &&
	     s->failed == 0)) {
		rcw = 1; rmw = 2;
	} else for (i = disks; i--; ) {
		struct r5dev *dev = &sh->dev[i];

		if (((dev->towrite && !delay_towrite(conf, dev, s)) ||
		     i == sh->pd_idx || i == sh->qd_idx ||
		     test_bit(R5_InJournal, &dev->flags)) &&
		    !test_bit(R5_LOCKED, &dev->flags) &&
		    !(uptodate_for_rmw(dev) || test_bit(R5_Wantcompute, &dev->flags))) {
			if (test_bit(R5_Insync, &dev->flags))
				rmw++;            /* we would need to READ this */
			else
				rmw += 2*disks;   /* effectively impossible */
		}
		/* Would we have to read it for reconstruct-write? */
		if (!test_bit(R5_OVERWRITE, &dev->flags) &&
		    i != sh->pd_idx && i != sh->qd_idx &&
		    !test_bit(R5_LOCKED, &dev->flags) &&
		    !(test_bit(R5_UPTODATE, &dev->flags) ||
		      test_bit(R5_Wantcompute, &dev->flags))) {
			if (test_bit(R5_Insync, &dev->flags))
				rcw++;
			else
				rcw += 2*disks;
		}
	}

	/* Whichever needs fewer reads wins. */
	if (rmw < rcw && rmw > 0) {
		/* prefer read-modify-write */
		...
		schedule_reconstruction(sh, s, rcw == 0, 0);
	} else if (rcw <= rmw && rcw > 0) {
		/* prefer reconstruct-write */
		...
	}
	...
}
```

`rmw` and `rcw` count **how many reads each approach requires**, and the cheaper wins. That is §T.4's entire algorithm, in one comparison.

### 1.5 The superblock

```c
struct mdp_superblock_1 {
	/* constant array information - 128 bytes */
	__le32	magic;		/* MD_SB_MAGIC: 0xa92b4efc */
	__le32	major_version;	/* 1 */
	__le32	feature_map;
	__le32	pad0;
	__u8	set_uuid[16];	/* the array's identity */
	char	set_name[32];
	__le64	ctime;
	__le32	level;
	__le32	layout;
	__le64	size;		/* used size of component devices, in sectors */
	__le32	chunksize;
	__le32	raid_disks;
	union {
		__le32	bitmap_offset;
		struct { __le16 offset; __le16 size; } ppl;  /* T.5(b) */
	};

	/* reshape in progress - 64 bytes */
	__le32	new_level;
	__le64	reshape_position;
	__le32	delta_disks;
	__le32	new_layout;
	__le32	new_chunk;
	__le32	new_offset;

	/* per-device descriptor - 64 bytes */
	__le64	data_offset;
	__le64	data_size;
	__le64	super_offset;
	union { __le64 recovery_offset; __le64 journal_tail; };
	__le32	dev_number;
	__le32	cnt_corrected_read;
	__u8	device_uuid[16];
	__u8	devflags;
	__u8	bblog_shift;
	__le16	bblog_size;
	__le32	bblog_offset;

	/* array state information - 64 bytes */
	__le64	utime;
	__le64	events;         /* THE consistency counter */
	__le64	resync_offset;
	__le32	sb_csum;
	__le32	max_dev;
	__u8	pad3[64-32];
	__le16	dev_roles[];    /* role in array, or 0xffff for spare */
};
```

**`events` is the key field.** Every metadata update increments it on every member. On assembly, devices with a lower `events` count than the majority were absent for some writes and are treated as stale — which is how MD avoids assembling an array from an out-of-date device and silently reverting data.

Superblock versions and where they live:

| Version | Location | Note |
|---|---|---|
| 0.90 | **end** of device | legacy; 28 device limit, 2 TB limit; auto-detectable |
| 1.0 | **end** of device | modern format, end placement |
| 1.1 | start | |
| 1.2 | **4 KiB from start** | **the default**; leaves room for a bootloader |

1.2 is the default because placing metadata at the start prevents a stale partition table or filesystem superblock from being mistaken for real data, while the 4 KiB offset leaves room for boot sectors.

The 0.90-at-the-end placement had a specific hazard: a RAID 1 member could be mounted directly as a filesystem (the data starts at offset 0), which lets someone accidentally mount half a mirror, write to it, and corrupt the array. 1.2 prevents this.

### 1.6 Observability

| Where | What |
|---|---|
| `/proc/mdstat` ★★★ | the array list, states, and sync progress |
| `/sys/block/mdX/md/` ★★★ | **everything**, writable |
| `/sys/block/mdX/md/array_state` | `clear/inactive/suspended/readonly/read-auto/clean/active` |
| `/sys/block/mdX/md/sync_action` ★★★ | `idle/resync/recover/check/repair/reshape` |
| `/sys/block/mdX/md/mismatch_cnt` ★★★ | §T.6 |
| `/sys/block/mdX/md/stripe_cache_size` ★★★ | §T.4's tunable |
| `/sys/block/mdX/md/sync_speed_{min,max}` | resync throttling |
| `/sys/block/mdX/md/rdN/{state,errors,bad_blocks,slot}` ★★★ | per-device |
| `/sys/block/mdX/md/bitmap/` | §T.7 |
| `mdadm --detail /dev/mdX` ★★★ | |
| `mdadm --examine /dev/sdX` ★★★ | **the on-disk superblock** |
| `mdadm --examine-badblocks` | |
| `mdadm --monitor` ★★★ | email/script on events |
| `trace-cmd record -e md:\* -e raid5:\*` ★★★ | |
| `dmesg` | MD is unusually verbose and informative |

`mdadm --examine` versus `--detail` is worth internalising: **`--detail` shows the running array's state; `--examine` shows what is written on a device.** When an array will not assemble, `--examine` on each member (and comparing `Events` counts) is the diagnostic.

---

## 2. Practice

### Lab 67.1 — Build and inspect every level

```sh
sudo apt install -y mdadm
sudo modprobe scsi_debug dev_size_mb=512 num_tgts=8
DEVS=($(lsblk -ndo NAME,MODEL | awk '/scsi_debug/{print "/dev/"$1}'))
echo "${DEVS[@]}"
```

```sh
for level in 0 1 5 6 10; do
  sudo mdadm --stop /dev/md0 2>/dev/null
  sudo mdadm --zero-superblock "${DEVS[@]}" 2>/dev/null
  case $level in
    0|1)   N=2 ;;
    5)     N=4 ;;
    6|10)  N=4 ;;
  esac
  sudo mdadm --create /dev/md0 --run --level=$level --raid-devices=$N \
       "${DEVS[@]:0:$N}" 2>&1 | tail -2
  echo "=== RAID $level ($N devices) ==="
  cat /proc/mdstat | head -5
  echo -n "  capacity: "; sudo blockdev --getsize64 /dev/md0 | numfmt --to=iec
  echo -n "  chunk: "; cat /sys/block/md0/md/chunk_size 2>/dev/null
  sudo mdadm --detail /dev/md0 | grep -E 'Array Size|Used Dev|Layout|Chunk'
  sleep 2
done
```

The superblock:

```sh
sudo mdadm --examine ${DEVS[0]}
sudo mdadm --examine ${DEVS[0]} | grep -E 'Magic|Version|Array UUID|Events|Role'
sudo dd if=${DEVS[0]} bs=1 skip=4096 count=8 2>/dev/null | hexdump -C
# fc 4e 2b a9 01 00 00 00  -> MD_SB_MAGIC, version 1
```

Compare superblock versions:

```sh
for v in 0.90 1.0 1.2; do
  sudo mdadm --stop /dev/md0 2>/dev/null
  sudo mdadm --zero-superblock "${DEVS[@]:0:2}" 2>/dev/null
  sudo mdadm --create /dev/md0 --run --metadata=$v --level=1 \
       --raid-devices=2 "${DEVS[@]:0:2}" 2>/dev/null
  echo -n "metadata=$v: data offset = "
  sudo mdadm --examine ${DEVS[0]} | grep -i 'data offset\|super offset' | tr '\n' ' '
  echo
done
```

sysfs, exhaustively:

```sh
sudo mdadm --stop /dev/md0
sudo mdadm --zero-superblock "${DEVS[@]}" 2>/dev/null
sudo mdadm --create /dev/md0 --run --level=5 --raid-devices=4 "${DEVS[@]:0:4}"
sleep 5

find /sys/block/md0/md -maxdepth 1 -type f | while read f; do
  printf "%-28s %s\n" "$(basename $f)" "$(cat $f 2>/dev/null | head -c 50)"
done
```

---

### Lab 67.2 — The write penalty, measured

```sh
sudo mdadm --stop /dev/md0 2>/dev/null
sudo mdadm --zero-superblock "${DEVS[@]}" 2>/dev/null
sudo mdadm --create /dev/md0 --run --level=5 --raid-devices=4 \
     --chunk=64 "${DEVS[@]:0:4}"
sleep 3
cat /sys/block/md0/md/chunk_size
STRIPE=$((64 * 3))      # chunk * (N-1) data disks = 192 KiB
echo "full stripe = ${STRIPE}K"
```

Partial versus full stripe writes — §T.4's central fact:

```sh
for bs in 4k 64k 192k 384k; do
  echo -n "bs=$bs: "
  sudo fio --name=t --filename=/dev/md0 --direct=1 --rw=write --bs=$bs \
           --iodepth=8 --ioengine=libaio --runtime=8 --time_based \
           --offset_increment=0 2>/dev/null | grep -oP 'BW=\K[^ ]+'
done
```

Count the physical I/Os:

```sh
sudo blktrace -d ${DEVS[0]} -d ${DEVS[1]} -d ${DEVS[2]} -d ${DEVS[3]} -o /tmp/r5 &
sudo dd if=/dev/zero of=/dev/md0 bs=4k count=500 oflag=direct 2>/dev/null
sudo pkill blktrace; sleep 1
TOTAL=0
for d in "${DEVS[@]:0:4}"; do
  N=$(blkparse -i /tmp/r5.$(basename $d) 2>/dev/null | grep -c ' D ')
  TOTAL=$((TOTAL + N))
  echo "  $(basename $d): $N"
done
echo "logical writes: 500, physical I/Os: $TOTAL, amplification: $(echo "scale=2; $TOTAL/500" | bc)"
```

Now full-stripe:

```sh
sudo rm -f /tmp/r5*
sudo blktrace -d ${DEVS[0]} -d ${DEVS[1]} -d ${DEVS[2]} -d ${DEVS[3]} -o /tmp/r5 &
sudo dd if=/dev/zero of=/dev/md0 bs=192k count=500 oflag=direct 2>/dev/null
sudo pkill blktrace; sleep 1
TOTAL=0
for d in "${DEVS[@]:0:4}"; do
  TOTAL=$((TOTAL + $(blkparse -i /tmp/r5.$(basename $d) 2>/dev/null | grep -c ' D ')))
done
echo "full-stripe amplification: $(echo "scale=2; $TOTAL/500" | bc)"
```

Watch the RMW/RCW decision:

```sh
sudo trace-cmd record -e raid5:\* -- \
  sudo dd if=/dev/zero of=/dev/md0 bs=4k count=200 oflag=direct 2>/dev/null
sudo trace-cmd report | head -30
sudo trace-cmd report | grep -oP 'rmw|rcw' | sort | uniq -c
```

The stripe cache — §T.4's tunable:

```sh
cat /sys/block/md0/md/stripe_cache_size
for size in 256 1024 4096 16384; do
  echo $size | sudo tee /sys/block/md0/md/stripe_cache_size > /dev/null
  echo -n "stripe_cache_size=$size: "
  sudo fio --name=t --filename=/dev/md0 --direct=1 --rw=randwrite --bs=64k \
           --iodepth=32 --ioengine=libaio --runtime=8 --time_based \
           --numjobs=4 --group_reporting 2>/dev/null | grep -oP 'BW=\K[^ ]+'
  echo "  active: $(cat /sys/block/md0/md/stripe_cache_active)"
done
```

Chunk size:

```sh
for chunk in 16 64 256 1024; do
  sudo mdadm --stop /dev/md0 2>/dev/null
  sudo mdadm --zero-superblock "${DEVS[@]:0:4}" 2>/dev/null
  sudo mdadm --create /dev/md0 --run --level=5 --raid-devices=4 \
       --chunk=$chunk "${DEVS[@]:0:4}" 2>/dev/null
  sleep 3
  echo 8192 | sudo tee /sys/block/md0/md/stripe_cache_size > /dev/null
  echo -n "chunk=${chunk}K: "
  sudo fio --name=t --filename=/dev/md0 --direct=1 --rw=randwrite --bs=64k \
           --iodepth=16 --ioengine=libaio --runtime=6 --time_based 2>/dev/null | \
    grep -oP 'BW=\K[^ ]+'
done
```

Filesystem alignment — the practical consequence:

```sh
sudo mdadm --stop /dev/md0 2>/dev/null
sudo mdadm --zero-superblock "${DEVS[@]:0:4}" 2>/dev/null
sudo mdadm --create /dev/md0 --run --level=5 --raid-devices=4 --chunk=64 "${DEVS[@]:0:4}"
sleep 3

cat /sys/block/md0/queue/{optimal_io_size,minimum_io_size}
# minimum_io_size = chunk; optimal_io_size = chunk * data disks

sudo mkfs.xfs -f /dev/md0 2>&1 | grep -E 'sunit|swidth'
sudo mkfs.ext4 -qF /dev/md0
sudo tune2fs -l /dev/md0 | grep -i 'stride\|stripe'
# mkfs reads the limits and aligns automatically. Verify it did.
```

---

### Lab 67.3 — Failure and recovery

```sh
sudo mdadm --stop /dev/md0 2>/dev/null
sudo mdadm --zero-superblock "${DEVS[@]}" 2>/dev/null
sudo mdadm --create /dev/md0 --run --level=5 --raid-devices=4 \
     --spare-devices=1 "${DEVS[@]:0:5}"
sleep 5
cat /proc/mdstat
sudo mkfs.ext4 -qF /dev/md0
sudo mkdir -p /mnt/md && sudo mount /dev/md0 /mnt/md
sudo cp -r /usr/include /mnt/md/ 2>/dev/null
sudo md5sum /mnt/md/include/stdio.h > /tmp/before.md5
sync
```

Fail a device while I/O is running:

```sh
sudo fio --name=t --directory=/mnt/md --size=100m --rw=randrw --bs=4k \
         --iodepth=8 --ioengine=libaio --runtime=60 --time_based > /dev/null 2>&1 &
FIO=$!
sleep 3

sudo mdadm --fail /dev/md0 ${DEVS[1]}
cat /proc/mdstat
dmesg | tail -5
sudo mdadm --detail /dev/md0 | head -25
```

The spare kicks in automatically:

```sh
for i in $(seq 1 15); do
  cat /proc/mdstat | grep -A1 md0 | tail -1
  sleep 2
done
kill $FIO 2>/dev/null; wait 2>/dev/null

sudo md5sum -c /tmp/before.md5     # data intact throughout
```

Watch the recovery:

```sh
cat /sys/block/md0/md/sync_action
cat /sys/block/md0/md/sync_speed
cat /sys/block/md0/md/sync_completed
cat /sys/block/md0/md/sync_speed_min
cat /sys/block/md0/md/sync_speed_max

# Throttle it
echo 1000 | sudo tee /sys/block/md0/md/sync_speed_max > /dev/null
cat /sys/block/md0/md/sync_speed
echo 200000 | sudo tee /sys/block/md0/md/sync_speed_max > /dev/null
```

Remove and re-add:

```sh
sudo mdadm --remove /dev/md0 ${DEVS[1]}
sudo mdadm --detail /dev/md0 | tail -8
sudo mdadm --add /dev/md0 ${DEVS[1]}
cat /proc/mdstat
```

**Double failure on RAID 5 — the unsurvivable case:**

```sh
sudo mdadm --wait /dev/md0
sudo mdadm --fail /dev/md0 ${DEVS[0]}
sudo mdadm --fail /dev/md0 ${DEVS[2]}
cat /proc/mdstat
sudo cat /mnt/md/include/stdio.h > /dev/null 2>&1 || echo "array failed"
dmesg | tail -10
sudo umount -f /mnt/md 2>/dev/null
```

RAID 6 survives it:

```sh
sudo mdadm --stop /dev/md0
sudo mdadm --zero-superblock "${DEVS[@]}" 2>/dev/null
sudo mdadm --create /dev/md0 --run --level=6 --raid-devices=5 "${DEVS[@]:0:5}"
sleep 5
sudo mkfs.ext4 -qF /dev/md0 && sudo mount /dev/md0 /mnt/md
sudo cp -r /usr/include /mnt/md/ 2>/dev/null; sync

sudo mdadm --fail /dev/md0 ${DEVS[0]}
sudo mdadm --fail /dev/md0 ${DEVS[1]}
cat /proc/mdstat
sudo md5sum /mnt/md/include/stdio.h     # still works
sudo mdadm --fail /dev/md0 ${DEVS[2]}   # third failure
cat /proc/mdstat
sudo md5sum /mnt/md/include/stdio.h 2>&1 | tail -1
sudo umount -f /mnt/md 2>/dev/null
```

---

### Lab 67.4 — Bitmaps

```sh
sudo mdadm --stop /dev/md0 2>/dev/null
sudo mdadm --zero-superblock "${DEVS[@]}" 2>/dev/null

# Without a bitmap
sudo mdadm --create /dev/md0 --run --level=1 --raid-devices=2 \
     --bitmap=none "${DEVS[@]:0:2}"
sudo mdadm --wait /dev/md0 2>/dev/null
sudo mdadm --detail /dev/md0 | grep -i bitmap

sudo dd if=/dev/zero of=/dev/md0 bs=1M count=200 oflag=direct 2>/dev/null
sudo mdadm --fail /dev/md0 ${DEVS[1]}
sudo mdadm --remove /dev/md0 ${DEVS[1]}
sudo dd if=/dev/urandom of=/dev/md0 bs=1M count=10 oflag=direct 2>/dev/null

echo "re-adding WITHOUT bitmap:"
sudo mdadm --re-add /dev/md0 ${DEVS[1]} 2>&1 | tail -1
sudo mdadm --add /dev/md0 ${DEVS[1]} 2>/dev/null
time sudo mdadm --wait /dev/md0
```

With a bitmap:

```sh
sudo mdadm --stop /dev/md0
sudo mdadm --zero-superblock "${DEVS[@]:0:2}"
sudo mdadm --create /dev/md0 --run --level=1 --raid-devices=2 \
     --bitmap=internal --bitmap-chunk=8M "${DEVS[@]:0:2}"
sudo mdadm --wait /dev/md0 2>/dev/null
sudo mdadm --detail /dev/md0 | grep -i bitmap
ls /sys/block/md0/md/bitmap/
cat /sys/block/md0/md/bitmap/chunksize

sudo dd if=/dev/zero of=/dev/md0 bs=1M count=200 oflag=direct 2>/dev/null
sudo mdadm --fail /dev/md0 ${DEVS[1]}
sudo mdadm --remove /dev/md0 ${DEVS[1]}
sudo dd if=/dev/urandom of=/dev/md0 bs=1M count=10 oflag=direct 2>/dev/null

echo "re-adding WITH bitmap:"
time sudo mdadm --re-add /dev/md0 ${DEVS[1]}
sudo mdadm --wait /dev/md0
cat /proc/mdstat
```

**Seconds versus the full array.** That is §T.7.

The write cost:

```sh
for bm in none internal; do
  sudo mdadm --stop /dev/md0 2>/dev/null
  sudo mdadm --zero-superblock "${DEVS[@]:0:2}" 2>/dev/null
  sudo mdadm --create /dev/md0 --run --level=1 --raid-devices=2 \
       --bitmap=$bm "${DEVS[@]:0:2}" 2>/dev/null
  sudo mdadm --wait /dev/md0 2>/dev/null
  echo -n "bitmap=$bm: "
  sudo fio --name=t --filename=/dev/md0 --direct=1 --rw=randwrite --bs=4k \
           --iodepth=1 --fsync=1 --runtime=8 --time_based 2>/dev/null | \
    grep -oP 'IOPS=\K[^,]+'
done
```

Bitmap chunk size:

```sh
for chunk in 1M 64M 512M; do
  sudo mdadm --stop /dev/md0 2>/dev/null
  sudo mdadm --zero-superblock "${DEVS[@]:0:2}" 2>/dev/null
  sudo mdadm --create /dev/md0 --run --level=1 --raid-devices=2 \
       --bitmap=internal --bitmap-chunk=$chunk "${DEVS[@]:0:2}" 2>/dev/null
  sudo mdadm --wait /dev/md0 2>/dev/null
  echo -n "chunk=$chunk: bits=$(cat /sys/block/md0/md/bitmap/chunksize 2>/dev/null), "
  sudo fio --name=t --filename=/dev/md0 --direct=1 --rw=randwrite --bs=4k \
           --iodepth=8 --runtime=5 --time_based 2>/dev/null | grep -oP 'IOPS=\K[^,]+'
done
```

---

### Lab 67.5 — The write hole

Demonstrate §T.5, then close it.

```sh
sudo mdadm --stop /dev/md0 2>/dev/null
sudo mdadm --zero-superblock "${DEVS[@]}" 2>/dev/null
sudo mdadm --create /dev/md0 --run --level=5 --raid-devices=4 \
     --consistency-policy=resync "${DEVS[@]:0:4}"
sudo mdadm --wait /dev/md0
sudo mdadm --detail /dev/md0 | grep -i consistency
```

Force an inconsistency by corrupting a member directly:

```sh
sudo dd if=/dev/zero of=/dev/md0 bs=1M count=100 oflag=direct 2>/dev/null
sync

# Corrupt one member behind MD's back -- simulating a partial stripe write
sudo mdadm --stop /dev/md0
sudo dd if=/dev/urandom of=${DEVS[1]} bs=4k seek=5000 count=10 conv=notrunc 2>/dev/null
sudo mdadm --assemble /dev/md0 "${DEVS[@]:0:4}"
sleep 2

# check finds it
echo check | sudo tee /sys/block/md0/md/sync_action
sudo mdadm --wait /dev/md0
cat /sys/block/md0/md/mismatch_cnt      # non-zero
dmesg | tail -5

# repair "fixes" it -- by rewriting parity, assuming the DATA is right
echo repair | sudo tee /sys/block/md0/md/sync_action
sudo mdadm --wait /dev/md0
echo check | sudo tee /sys/block/md0/md/sync_action
sudo mdadm --wait /dev/md0
cat /sys/block/md0/md/mismatch_cnt      # 0 -- but the DATA is still wrong (T.8)
```

**That last step is the point: MD "repaired" the array by making the parity match the corrupt data.** It could not know which was wrong.

PPL — §T.5(b):

```sh
sudo mdadm --stop /dev/md0
sudo mdadm --zero-superblock "${DEVS[@]:0:4}"
sudo mdadm --create /dev/md0 --run --level=5 --raid-devices=4 \
     --consistency-policy=ppl "${DEVS[@]:0:4}"
sudo mdadm --wait /dev/md0
sudo mdadm --detail /dev/md0 | grep -i consistency
sudo mdadm --examine ${DEVS[0]} | grep -i ppl

# Cost
echo -n "ppl: "
sudo fio --name=t --filename=/dev/md0 --direct=1 --rw=randwrite --bs=4k \
         --iodepth=16 --ioengine=libaio --runtime=8 --time_based 2>/dev/null | \
  grep -oP 'IOPS=\K[^,]+'

sudo mdadm --stop /dev/md0
sudo mdadm --zero-superblock "${DEVS[@]:0:4}"
sudo mdadm --create /dev/md0 --run --level=5 --raid-devices=4 \
     --consistency-policy=resync "${DEVS[@]:0:4}" 2>/dev/null
sudo mdadm --wait /dev/md0
echo -n "no ppl: "
sudo fio --name=t --filename=/dev/md0 --direct=1 --rw=randwrite --bs=4k \
         --iodepth=16 --ioengine=libaio --runtime=8 --time_based 2>/dev/null | \
  grep -oP 'IOPS=\K[^,]+'
```

A write journal — §T.5(a):

```sh
sudo mdadm --stop /dev/md0
sudo mdadm --zero-superblock "${DEVS[@]}" 2>/dev/null
sudo mdadm --create /dev/md0 --run --level=5 --raid-devices=4 \
     --write-journal ${DEVS[4]} "${DEVS[@]:0:4}" 2>&1 | tail -2
sudo mdadm --detail /dev/md0 | grep -iE 'journal|consistency'
cat /proc/mdstat
```

And the mitigation for §T.8 — `dm-integrity` underneath:

```sh
sudo mdadm --stop /dev/md0 2>/dev/null
for d in "${DEVS[@]:0:3}"; do
  sudo integritysetup format --batch-mode $d 2>/dev/null
  sudo integritysetup open $d int_$(basename $d)
done
ls /dev/mapper/int_*
sudo mdadm --create /dev/md0 --run --level=5 --raid-devices=3 /dev/mapper/int_* 2>&1 | tail -1
sleep 3
sudo dd if=/dev/zero of=/dev/md0 bs=1M count=50 oflag=direct 2>/dev/null

# Now corrupt a member: dm-integrity turns silent corruption into an EIO,
# which MD can handle by reconstructing.
sudo mdadm --stop /dev/md0
sudo integritysetup close int_$(basename ${DEVS[0]})
sudo dd if=/dev/urandom of=${DEVS[0]} bs=4k seek=3000 count=1 conv=notrunc 2>/dev/null
sudo integritysetup open ${DEVS[0]} int_$(basename ${DEVS[0]})
sudo mdadm --assemble /dev/md0 /dev/mapper/int_* 2>/dev/null
sudo dd if=/dev/md0 of=/dev/null bs=1M count=50 2>/dev/null
dmesg | grep -iE 'integrity|checksum' | tail -5
```

Cleanup:

```sh
sudo mdadm --stop /dev/md0 2>/dev/null
for m in /dev/mapper/int_*; do sudo integritysetup close $(basename $m) 2>/dev/null; done
```

---

### Lab 67.6 — Reshaping

```sh
sudo mdadm --zero-superblock "${DEVS[@]}" 2>/dev/null
sudo mdadm --create /dev/md0 --run --level=5 --raid-devices=3 "${DEVS[@]:0:3}"
sudo mdadm --wait /dev/md0
sudo mkfs.ext4 -qF /dev/md0 && sudo mount /dev/md0 /mnt/md
sudo cp -r /usr/include /mnt/md/ 2>/dev/null; sync
df -h /mnt/md
```

Grow by adding a device, **online**:

```sh
sudo mdadm --add /dev/md0 ${DEVS[3]}
sudo mdadm --grow /dev/md0 --raid-devices=4 \
     --backup-file=/tmp/md0.backup 2>&1 | tail -2

for i in $(seq 1 20); do
  cat /proc/mdstat | grep -A1 md0 | tail -1
  cat /sys/block/md0/md/reshape_position 2>/dev/null
  sleep 2
done
sudo mdadm --wait /dev/md0

df -h /mnt/md                        # filesystem is still the old size
sudo resize2fs /dev/md0
df -h /mnt/md                        # now larger
ls /mnt/md/include | head -3         # data intact
```

Level conversion:

```sh
sudo mdadm --detail /dev/md0 | grep -i 'raid level'
sudo mdadm --add /dev/md0 ${DEVS[4]}
sudo mdadm --grow /dev/md0 --level=6 --raid-devices=5 \
     --backup-file=/tmp/md0.backup2 2>&1 | tail -3
sudo mdadm --wait /dev/md0
sudo mdadm --detail /dev/md0 | grep -i 'raid level'
```

Chunk-size change:

```sh
cat /sys/block/md0/md/chunk_size
sudo mdadm --grow /dev/md0 --chunk=128 --backup-file=/tmp/md0.backup3 2>&1 | tail -2
sudo mdadm --wait /dev/md0
cat /sys/block/md0/md/chunk_size
```

Takeover — RAID 1 to RAID 5:

```sh
sudo umount /mnt/md
sudo mdadm --stop /dev/md0
sudo mdadm --zero-superblock "${DEVS[@]}" 2>/dev/null
sudo mdadm --create /dev/md0 --run --level=1 --raid-devices=2 "${DEVS[@]:0:2}"
sudo mdadm --wait /dev/md0
sudo mdadm --grow /dev/md0 --level=5 2>&1 | tail -2
sudo mdadm --detail /dev/md0 | grep -iE 'raid level|raid devices'
sudo mdadm --add /dev/md0 ${DEVS[2]}
sudo mdadm --grow /dev/md0 --raid-devices=3 --backup-file=/tmp/b 2>&1 | tail -1
sudo mdadm --wait /dev/md0
cat /proc/mdstat
```

Interrupt a reshape and recover:

```sh
sudo mdadm --stop /dev/md0
sudo mdadm --zero-superblock "${DEVS[@]}" 2>/dev/null
sudo mdadm --create /dev/md0 --run --level=5 --raid-devices=3 "${DEVS[@]:0:3}"
sudo mdadm --wait /dev/md0
sudo mdadm --add /dev/md0 ${DEVS[3]}
sudo mdadm --grow /dev/md0 --raid-devices=4 --backup-file=/tmp/md0.b 2>/dev/null &
sleep 3
sudo mdadm --stop /dev/md0             # "crash" mid-reshape
cat /proc/mdstat

sudo mdadm --assemble /dev/md0 --backup-file=/tmp/md0.b "${DEVS[@]:0:4}" 2>&1 | tail -2
cat /proc/mdstat                       # the reshape resumes
sudo mdadm --examine ${DEVS[0]} | grep -i reshape
```

**The `--backup-file` is what makes an interrupted reshape recoverable**, because the critical section (where new and old layouts overlap) cannot be done in place.

---

### Lab 67.7 — Monitoring and diagnosis

```sh
sudo mdadm --stop /dev/md0 2>/dev/null
sudo mdadm --zero-superblock "${DEVS[@]}" 2>/dev/null
sudo mdadm --create /dev/md0 --run --level=5 --raid-devices=4 \
     --spare-devices=1 "${DEVS[@]:0:5}"
sudo mdadm --wait /dev/md0
```

Monitoring:

```sh
cat > /tmp/mdalert.sh <<'EOF'
#!/bin/sh
echo "$(date) EVENT=$1 DEV=$2 EXTRA=$3" >> /tmp/md-events.log
EOF
chmod +x /tmp/mdalert.sh

sudo mdadm --monitor --scan --daemonise --program=/tmp/mdalert.sh \
     --delay=5 --pid-file=/tmp/mdmon.pid

sudo mdadm --fail /dev/md0 ${DEVS[1]}
sleep 10
cat /tmp/md-events.log
sudo mdadm --wait /dev/md0
sleep 10
cat /tmp/md-events.log
sudo kill $(cat /tmp/mdmon.pid) 2>/dev/null
```

Configuration persistence:

```sh
sudo mdadm --detail --scan
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
grep -v '^#' /etc/mdadm/mdadm.conf | grep -v '^$'
# Then: update-initramfs -u   (so the array assembles at boot)
```

Scheduled scrubbing — §T.6 and §T.9's mitigation:

```sh
ls /etc/cron.d/mdadm /usr/share/mdadm/checkarray 2>/dev/null
cat /etc/cron.d/mdadm 2>/dev/null

# Run one manually
echo check | sudo tee /sys/block/md0/md/sync_action
watch -n1 'cat /proc/mdstat | head -5' &
sleep 10; kill %2 2>/dev/null
sudo mdadm --wait /dev/md0
cat /sys/block/md0/md/mismatch_cnt
cat /sys/block/md0/md/last_sync_action
```

Diagnosing an array that will not assemble:

```sh
sudo mdadm --stop /dev/md0
# Make one device stale: assemble without it, write, then try with it
sudo mdadm --assemble /dev/md0 "${DEVS[@]:0:3}" --run 2>/dev/null
sudo dd if=/dev/urandom of=/dev/md0 bs=1M count=20 oflag=direct 2>/dev/null
sudo mdadm --stop /dev/md0

echo "=== Events counts ==="
for d in "${DEVS[@]:0:4}"; do
  printf "%-12s " $(basename $d)
  sudo mdadm --examine $d 2>/dev/null | grep -E 'Events|Array State' | tr '\n' ' '
  echo
done
```

**Comparing `Events` counts across members is the primary diagnostic** for assembly failures.

```sh
sudo mdadm --assemble /dev/md0 "${DEVS[@]:0:4}" 2>&1 | tail -3
# The stale device is excluded. Force it (DANGEROUS -- data loss):
# sudo mdadm --assemble --force /dev/md0 "${DEVS[@]:0:4}"

sudo mdadm --assemble --run /dev/md0 "${DEVS[@]:0:4}" 2>&1 | tail -2
cat /proc/mdstat
```

Bad-block handling:

```sh
sudo mdadm --detail /dev/md0 | grep -i 'bad block'
cat /sys/block/md0/md/rd0/bad_blocks 2>/dev/null
sudo mdadm --examine-badblocks ${DEVS[0]} 2>&1 | head

# Inject a bad block with dm-dust (Ch. 66 T.8)
sudo mdadm --stop /dev/md0
SZ=$(sudo blockdev --getsz ${DEVS[0]})
sudo dmsetup create dusty --table "0 $SZ dust ${DEVS[0]} 0 512"
sudo dmsetup message dusty 0 addbadblock 10000
sudo dmsetup message dusty 0 enable
sudo mdadm --create /dev/md0 --run --level=5 --raid-devices=3 \
     /dev/mapper/dusty "${DEVS[@]:1:2}" 2>/dev/null
sleep 3
sudo dd if=/dev/md0 of=/dev/null bs=1M count=50 2>/dev/null
dmesg | grep -iE 'md|raid' | tail -10
cat /sys/block/md0/md/rd0/errors 2>/dev/null
sudo mdadm --stop /dev/md0; sudo dmsetup remove dusty
```

---

### Lab 67.8 — Compare everything

Build the evidence base rather than trusting folklore.

```sh
sudo mdadm --stop /dev/md0 2>/dev/null
sudo mdadm --zero-superblock "${DEVS[@]}" 2>/dev/null

run_bench() {
  local name=$1
  echo "=== $name ==="
  for rw in read randread write randwrite; do
    printf "  %-10s " $rw
    sudo fio --name=t --filename=/dev/md0 --direct=1 --rw=$rw --bs=64k \
             --iodepth=16 --ioengine=libaio --runtime=6 --time_based \
             --numjobs=4 --group_reporting 2>/dev/null | grep -oP 'BW=\K[^ ]+'
  done
  printf "  %-10s " "4k sync w"
  sudo fio --name=t --filename=/dev/md0 --direct=1 --rw=randwrite --bs=4k \
           --iodepth=1 --fsync=1 --runtime=6 --time_based 2>/dev/null | \
    grep -oP 'IOPS=\K[^,]+'
  echo -n "  capacity: "; sudo blockdev --getsize64 /dev/md0 | numfmt --to=iec
}

for spec in "0 4" "1 2" "5 4" "6 5" "10 4"; do
  set -- $spec
  LEVEL=$1; N=$2
  sudo mdadm --stop /dev/md0 2>/dev/null
  sudo mdadm --zero-superblock "${DEVS[@]}" 2>/dev/null
  sudo mdadm --create /dev/md0 --run --level=$LEVEL --raid-devices=$N \
       "${DEVS[@]:0:$N}" 2>/dev/null
  sudo mdadm --wait /dev/md0 2>/dev/null
  [ -e /sys/block/md0/md/stripe_cache_size ] && \
    echo 8192 | sudo tee /sys/block/md0/md/stripe_cache_size > /dev/null
  run_bench "RAID $LEVEL ($N devices)"
done
```

RAID 10 layouts:

```sh
for layout in n2 f2 o2; do
  sudo mdadm --stop /dev/md0 2>/dev/null
  sudo mdadm --zero-superblock "${DEVS[@]:0:4}" 2>/dev/null
  sudo mdadm --create /dev/md0 --run --level=10 --layout=$layout \
       --raid-devices=4 "${DEVS[@]:0:4}" 2>/dev/null
  sudo mdadm --wait /dev/md0 2>/dev/null
  echo "=== raid10 $layout ==="
  for rw in read randread write; do
    printf "  %-10s " $rw
    sudo fio --name=t --filename=/dev/md0 --direct=1 --rw=$rw --bs=64k \
             --iodepth=16 --ioengine=libaio --runtime=6 --time_based 2>/dev/null | \
      grep -oP 'BW=\K[^ ]+'
  done
done
```

Rebuild time — §T.9's real risk:

```sh
for spec in "1 2" "5 4" "6 5" "10 4"; do
  set -- $spec
  sudo mdadm --stop /dev/md0 2>/dev/null
  sudo mdadm --zero-superblock "${DEVS[@]}" 2>/dev/null
  sudo mdadm --create /dev/md0 --run --level=$1 --raid-devices=$2 \
       "${DEVS[@]:0:$2}" 2>/dev/null
  sudo mdadm --wait /dev/md0 2>/dev/null
  sudo dd if=/dev/zero of=/dev/md0 bs=1M count=400 oflag=direct 2>/dev/null

  sudo mdadm --fail /dev/md0 ${DEVS[1]} 2>/dev/null
  sudo mdadm --remove /dev/md0 ${DEVS[1]} 2>/dev/null
  START=$(date +%s.%N)
  sudo mdadm --add /dev/md0 ${DEVS[1]} 2>/dev/null
  sudo mdadm --wait /dev/md0 2>/dev/null
  echo "RAID $1 rebuild: $(echo "$(date +%s.%N) - $START" | bc)s"
done
```

The parity engine benchmark the kernel runs at boot:

```sh
dmesg | grep -iE 'raid6:|xor:' | head -20
cat /sys/module/raid6_pq/parameters/disable_dq 2>/dev/null
```

Cleanup:

```sh
sudo mdadm --stop /dev/md0 2>/dev/null
sudo mdadm --zero-superblock "${DEVS[@]}" 2>/dev/null
```

---

## 3. Mastery drills

1. State precisely what RAID protects against. Then, for each of the five things it does not protect against (§T.1), name the mechanism that does.

2. Derive the capacity, fault tolerance, read scaling, and write scaling of each RAID level from first principles. Then compute each for a 12-device array.

3. Explain MD's `raid10` `f2` layout and why it gives better sequential read throughput. Compute the improvement for a rotating disk, and state why it is zero for an SSD.

4. Derive the RMW and RCW costs for a write of `k` blocks to an `N`-device RAID 5 stripe. Find the `k` at which RCW becomes cheaper, and verify it against §1.4's code.

5. Compute the write amplification for 4 KiB random writes on a 6-device RAID 5 with 512 KiB chunks. Then compute it for RAID 6 and for RAID 10.

6. Explain the write hole precisely, including which block is corrupted and why the application never knew it was at risk. Then explain why RAID-Z does not have it.

7. Compare the three write-hole mitigations (journal, PPL, full resync) on: correctness, write amplification, hardware required, and recovery time.

8. `check` versus `repair` on RAID 5: explain what each does, and construct the case where `repair` makes things worse.

9. A write-intent bitmap converts O(size) resync into O(changes). Derive the write-path cost as a function of bitmap chunk size, and find the optimum for a given write pattern.

10. Enumerate everything MD cannot detect (§T.8). For each, state what would be needed to detect it and where in the stack that could live.

11. Evaluate the URE argument rigorously: state the claim, the arithmetic, and the four reasons it is quantitatively wrong. Then give the correct decision rule.

12. Compute the rebuild window for a 6×20 TB RAID 5 and a 6×20 TB RAID 6, and estimate the probability of a second failure during each given a 2 % annual failure rate.

13. You are handed a 5-device RAID 6 showing `[UU_U_]` with one spare that is not rebuilding. Give the ordered diagnostic procedure and the four most likely causes.

---

## 4. Further reading

**Kernel documentation**

- `Documentation/admin-guide/md.rst` ★★★ — **the sysfs interface, exhaustively documented.** Every file under `/sys/block/mdX/md/` with its semantics. Read it.
- `Documentation/driver-api/md/md-cluster.rst`, `raid5-cache.rst` ★★★ (§T.5(a)), `raid5-ppl.rst` ★★★ (§T.5(b))
- `Documentation/admin-guide/device-mapper/dm-raid.rst` — §T.10's wrapper.
- `man 8 mdadm` ★★★ — **long and worth reading completely.** Especially the `--create`, `--assemble`, `--grow`, `--monitor`, and `--examine` sections.
- `man 5 mdadm.conf`, `man 4 md` ★★★ — the `md` man page is an excellent conceptual overview.
- `lib/raid6/` — the algorithm implementations, with the benchmark harness.

**Papers**

- Patterson, Gibson, Katz, "A Case for Redundant Arrays of Inexpensive Disks (RAID)," SIGMOD 1988 ★★★ — **the original.** Short, and the levels are defined here.
- Chen, Lee, Gibson, Katz, Patterson, "RAID: High-Performance, Reliable Secondary Storage," ACM Computing Surveys 1994 ★★★ — the comprehensive survey; covers the write penalty and the small-write problem thoroughly.
- Anvin, "The mathematics of RAID-6" ★★★ — **in the kernel tree** (`Documentation/driver-api/md/raid5-cache.rst` references it; the paper is at kernel.org). The Reed-Solomon construction explained clearly by its implementer.
- Schroeder & Gibson, "Disk failures in the real world: What does an MTTF of 1,000,000 hours mean to you?" FAST 2007 ★★★ — **the paper that demolished vendor MTBF claims.** Essential for §T.9.
- Bairavasundaram et al., "An Analysis of Latent Sector Errors in Disk Drives," SIGMETRICS 2007, and "An Analysis of Data Corruption in the Storage Stack," FAST 2008 ★★★ — §T.8 and §T.9, with field data from 1.5 million drives.
- Elerath & Pecht, "A Highly Accurate Method for Assessing Reliability of Redundant Arrays of Inexpensive Disks (RAID)," IEEE Transactions on Computers 2009 — a more careful model than the naive URE arithmetic.
- Greenan, Plank, Wylie, "Mean time to meaningless: MTTDL, Markov models, and storage system reliability," HotStorage 2010 ★★★ — why the standard reliability calculations are misleading.
- Corbett et al., "Row-Diagonal Parity for Double Disk Failure Correction," FAST 2004 — an alternative to Reed-Solomon for RAID 6.

**LWN**

- "A block layer introduction" series (Neil Brown) — Brown maintained MD for years; his explanations are authoritative.
- "RAID5 and the write hole" and the PPL coverage ★★★
- "Journalled RAID5" (raid5-cache)
- "Bad block logs for MD"
- "Clustered MD"
- "The md driver and bio splitting"
- Neil Brown's blog posts on MD internals ★★★ — particularly his explanations of reshape and the bitmap.

**Source reading order**

1. `Documentation/admin-guide/md.rst` and `man 4 md` first.
2. `drivers/md/md.h` ★★★ — `mddev`, `md_rdev`, the flags.
3. `drivers/md/raid0.c` — the simplest personality; ~800 lines.
4. `drivers/md/raid1.c`: `raid1_make_request`, `read_balance` ★★★, `raid1_error`, `fix_read_error`.
5. `drivers/md/md.c`: `md_do_sync`, `md_check_recovery`, `super_1_load`/`super_1_sync` ★★★
6. `drivers/md/md-bitmap.c`: `md_bitmap_startwrite`, `md_bitmap_endwrite` — §T.7.
7. `drivers/md/raid5.c` ★★★ — **the hard one.** Start with `handle_stripe`, then `handle_stripe_dirtying` (§1.4), then `schedule_reconstruction`. ~9000 lines; budget real time.
8. `lib/raid6/algos.c` and `lib/raid6/int.uc` — the Reed-Solomon implementation and the boot-time benchmark.

**Tools**

- `mdadm` ★★★ — `--detail`, `--examine` ★★★, `--monitor`, `--grow`, `--wait`, `--examine-badblocks`
- `/proc/mdstat` ★★★ and `/sys/block/mdX/md/*` ★★★
- `mdadm --monitor --scan --daemonise` ★★★ — **set this up on any real array**
- `checkarray` (Debian) / the distribution's scrub cron job ★★★ — **enable it**
- `smartctl -a`, `smartctl -t long` ★★★ — find failing drives before they fail
- `blktrace` on member devices — see the write amplification directly
- `trace-cmd record -e md:\* -e raid5:\*`
- `dm-dust`, `dm-flakey`, `dm-delay` (Ch. 66 §T.8) — build failure scenarios
- `integritysetup` — §T.8's mitigation
- `fio` with `--rw=randwrite --bs=4k` — the RAID 5 worst case; always test it

---

→ Next: [68-scsi-ata.md](68-scsi-ata.md)

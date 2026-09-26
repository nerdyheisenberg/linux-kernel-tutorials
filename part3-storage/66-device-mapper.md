# Chapter 66 — Device mapper: targets, dm-crypt, dm-thin, dm-raid, LVM

> **Goal:** Understand the kernel's block-level composition framework — how a device made of other devices is described, built, and changed atomically while in use. Understand the table/target model and why it is the right abstraction, suspend/resume as the mechanism for online reconfiguration, dm-crypt's per-sector IV design and its performance architecture, thin provisioning's metadata problem and the `ENOSPC` hazard it creates, dm-cache and dm-writecache's different bets, how LVM is a userspace policy engine over a kernel mechanism, and the fault-injection targets that made every crash-consistency lab in Part 3 possible. By the end you can write a dm target, build a multi-layer stack by hand, and reason about where a stacked I/O actually spends its time.

---

## Theory & First Principles

### T.0 — Start here: your laptop's disk is already five devices

```bash
lsblk
# nvme0n1
# +-nvme0n1p3                       <- a partition
#   +-nvme0n1p3_crypt   (crypt)     <- dm-crypt
#     +-vg0-root        (lvm)       <- dm-linear
#     +-vg0-swap        (lvm)
#     +-vg0-snap        (lvm)       <- dm-snapshot
sudo dmsetup table
```

Every `mount` you do goes through four or five devices. **And each of them is the same kind
of object: a thing that takes a `bio` and does something with it.** That is the entire idea:

```
            bio in
              |
              v
  +-------------------------+
  |  dm target              |    map(bio) may:
  |                         |      - remap  (change bdev + sector) -> dm-linear, dm-stripe
  |    int (*map)(struct    |      - transform contents           -> dm-crypt, dm-integrity
  |      dm_target *ti,     |      - duplicate                    -> dm-mirror
  |      struct bio *bio);  |      - redirect on write            -> dm-snapshot, dm-thin
  +-------------------------+      - delay / error / drop         -> dm-delay, dm-flakey
              |
              v
         bio(s) out, to the device(s) below
```

**One interface. Twenty-plus targets. Arbitrary composition.** This is a pipeline of filters —
the Unix pipe idea (Ch. 00 §T.3b), applied to block I/O. You can stack dm-crypt over dm-raid
over dm-cache over a partition, and nothing in the stack knows or cares.

**The second piece is the table, and this is the part worth studying as design:**

```bash
# "sectors 0..2097151 of this device live at sector 2048 of /dev/nvme0n1"
echo "0 2097152 linear /dev/nvme0n1 2048" | dmsetup create mydev
```

A device is defined **entirely by a text table** — a list of (start, length, target, args)
rows. And tables are swapped atomically, by a protocol you have now seen several times:

```
  dmsetup suspend  mydev     <- STOP: quiesce, flush in-flight I/O, hold new bios
  dmsetup reload   mydev     <- LOAD: validate the new table. Failure is harmless.
  dmsetup resume   mydev     <- APPLY: swap, release the held bios
```

**Suspend / reload / resume is validate-then-apply** (Ch. 43 §T.0, Ch. 47, Ch. 89), and it is
what makes online reconfiguration safe: you can grow a filesystem's underlying device,
insert a cache layer, or switch a mirror leg **while the filesystem is mounted and in use**,
with no window in which a partially-applied table is visible. The held `bio`s make the change
invisible to everything above.

**Three consequences worth noticing, because they are why dm exists rather than putting these
features in filesystems:**

| | |
|---|---|
| **Composability** | dm-crypt did not need to know about RAID; dm-cache did not need to know about LVM. N features, not N² integrations. |
| **Policy in userspace** | the kernel provides *mechanism* (targets); `lvm2` provides *policy* (which tables to build). Ch. 00 §T.1. |
| **Testability** | `dm-flakey`, `dm-delay`, `dm-dust`, and `dm-error` let you inject exactly the failures you cannot reproduce with real hardware — this is how filesystem crash-consistency is actually tested (Ch. 61). |

**And the argument against it, which you met in Ch. 59–60 and should now be able to state
fairly:** btrfs and ZFS say this layering is precisely the problem — dm-raid cannot know
which blocks are in use (so a rebuild copies free space), and cannot know which mirror leg is
*correct* (so it can detect a dead disk but not silent corruption). **Modularity costs
information. Integration costs flexibility.** dm-integrity is the layered stack's partial
answer; whether it is a good one is §T.8's argument.

```bash
sudo dmsetup table --showkeys=0 && sudo dmsetup info
sudo dmsetup create bad --table "0 2048 flakey /dev/loop0 0 8 4"   # fails periodically
sudo dmsetup status                          # per-target runtime state
cat /sys/block/dm-0/dm/name
```

---

### T.1 The composition problem

A block device is a linear array of sectors. Useful storage systems need:

| Need | Example |
|---|---|
| Concatenation | make three 1 TB disks look like 3 TB |
| Striping | spread I/O across devices |
| Mirroring | redundancy |
| Transformation | encrypt, compress, verify |
| Indirection | snapshots, thin provisioning, caching |
| Fault injection | testing |

Each could be a separate driver. Device mapper's insight is that they are all the same shape:

> **A function from (this device's sector range) to (operations on other devices).**

So there is one framework, and each behaviour is a **target** — a plugin implementing that function. A dm device's behaviour is entirely described by its **table**: a list of (sector range → target type → target parameters).

```
0       2097152  linear  /dev/sda 0
2097152 2097152  linear  /dev/sdb 0
```

That is a two-target table concatenating two devices. The whole model is visible in those two lines: **ranges are contiguous and non-overlapping, and each names a target and its private arguments.**

Why this is the right abstraction:

- **Composition is free.** A dm device is a block device, so it can be the target of another dm device. Stacks of six are routine.
- **Policy lives in userspace.** The kernel executes tables; `lvm2` decides what tables to build. Changing LVM's allocation policy requires no kernel change.
- **Reconfiguration is a table swap** (§T.3), which makes online changes tractable.
- **Testing is a target.** `dm-flakey`, `dm-error`, `dm-dust`, `dm-log-writes` exist because adding a target is cheap.

The cost: the table is a *text* interface, parsed in the kernel, from userspace-supplied strings. Every target's `ctr()` is a parser for untrusted input — Ch. 56 §T.5's problem again, though mitigated by requiring `CAP_SYS_ADMIN`.

### T.2 Targets: the interface

```c
struct target_type {
	uint64_t features;
	const char *name;
	struct module *module;
	unsigned int version[3];

	dm_ctr_fn ctr;              /* parse the table line, build state */
	dm_dtr_fn dtr;              /* tear down */
	dm_map_fn map;              /* THE function: remap a bio */
	dm_clone_and_map_request_fn clone_and_map_rq;
	dm_endio_fn end_io;         /* called on completion */
	dm_presuspend_fn presuspend;
	dm_presuspend_undo_fn presuspend_undo;
	dm_postsuspend_fn postsuspend;
	dm_preresume_fn preresume;
	dm_resume_fn resume;
	dm_status_fn status;        /* for `dmsetup status/table` */
	dm_message_fn message;      /* runtime commands */
	dm_prepare_ioctl_fn prepare_ioctl;
	dm_iterate_devices_fn iterate_devices;
	dm_io_hints_fn io_hints;    /* adjust queue_limits */
	dm_dax_direct_access_fn direct_access;
	...
};
```

The essential five are `ctr`, `dtr`, `map`, `status`, and `iterate_devices`. `map` is the heart:

```c
static int my_map(struct dm_target *ti, struct bio *bio)
{
	struct my_c *mc = ti->private;

	bio_set_dev(bio, mc->dev->bdev);
	bio->bi_iter.bi_sector = mc->start +
		dm_target_offset(ti, bio->bi_iter.bi_sector);

	return DM_MAPIO_REMAPPED;
}
```

Return values, and this small enum encodes the whole range of target behaviours:

| Return | Meaning |
|---|---|
| `DM_MAPIO_REMAPPED` | I modified the bio; submit it |
| `DM_MAPIO_SUBMITTED` | I took ownership; I will complete it |
| `DM_MAPIO_REQUEUE` | try again later |
| `DM_MAPIO_KILL` | fail this bio |

`REMAPPED` is the cheap path — the bio is redirected with no copy and no allocation, and dm-linear's entire I/O path is the five lines above. `SUBMITTED` is for targets that must do work (encrypt, allocate a thin block, copy for a snapshot) and complete asynchronously.

**`iterate_devices` matters more than it looks.** It is how the framework discovers which underlying devices a target uses, and therefore how it computes stacked `queue_limits` (Ch. 63 §T.9), whether the stack supports discard/flush/write-zeroes, and which devices to hold open. A target that does not implement it correctly produces a device with wrong limits, which produces corruption.

### T.3 Suspend and resume: atomic reconfiguration

This is device mapper's most important mechanism and the reason LVM can do things that partition tables cannot.

```
dmsetup suspend NAME     -> stop accepting new I/O; flush in-flight
dmsetup reload NAME      -> install a new table as "inactive"
dmsetup resume NAME      -> swap inactive -> active; resume I/O
```

The device has **two tables**: active and inactive. `resume` swaps them. From the perspective of everything above, the device's contents change atomically between two I/Os.

```c
static void __dm_suspend(struct mapped_device *md, struct dm_table *map,
			 unsigned int suspend_flags, unsigned int task_state,
			 int dmf_suspended_flag)
{
	...
	/* 1. Stop new I/O entering. */
	set_bit(DMF_BLOCK_IO_FOR_SUSPEND, &md->flags);

	/* 2. Optionally flush the filesystem above (freeze). */
	if (noflush)
		set_bit(DMF_NOFLUSH_SUSPENDING, &md->flags);
	...
	/* 3. Tell targets to prepare. */
	if (map)
		dm_table_presuspend_targets(map);

	/* 4. Quiesce: wait for in-flight I/O to complete. */
	...
	r = dm_wait_for_completion(md, task_state);
	...
	/* 5. Targets do their post-suspend work (checkpoint metadata, etc.) */
	if (map)
		dm_table_postsuspend_targets(map);
}
```

Two flavours:

| | **flush suspend** (default) | **noflush suspend** |
|---|---|---|
| In-flight I/O | completed | **requeued** |
| Use | normal reconfiguration | the underlying device is gone |
| Risk | may block indefinitely if the device is dead | requeued I/O must be re-mappable |

`noflush` exists for the case where the device under you has vanished: waiting for in-flight I/O would never complete, so instead you push it back and it will be re-issued against the new table. This is how multipath survives a path failure.

**Freezing the filesystem** (`dmsetup suspend` without `--nolockfs`) calls `freeze_bdev()`, which asks the filesystem to quiesce and flush — giving a *filesystem-consistent* point, not merely a block-consistent one. That is what makes LVM snapshots usable: `lvcreate --snapshot` freezes, snapshots, thaws, and the snapshot contains a clean filesystem.

This mechanism is what enables, online, with the filesystem mounted:

- `pvmove` — relocate data between physical devices
- `lvextend`/`lvreduce` — resize
- `lvconvert` — change from linear to mirror to RAID5
- snapshot creation
- adding or removing a cache layer

Every one is "build a new table, suspend, reload, resume." **The uniformity is the point.**

### T.4 dm-crypt: encryption as a target

The design problem: a block device is randomly addressable, so encryption must be **per-sector and position-dependent**. You cannot use a stream cipher across the whole device (rewriting sector 5 would require re-encrypting everything after it), and you cannot use the same IV for every sector (identical plaintext sectors would produce identical ciphertext, leaking structure).

So: each sector is encrypted independently, with an IV derived from its number.

**IV modes**, in historical order, and each fixes a flaw in the previous:

| Mode | IV | Problem it solves / has |
|---|---|---|
| `plain` | sector number (32-bit) | trivial; **watermarking attacks** |
| `plain64` | sector number (64-bit) | same, for >2 TiB |
| `essiv` | `E(hash(key), sector)` | IV is unpredictable without the key |
| `xts` | sector as tweak, built into the mode | **the current standard**; no separate IV needed |
| `random` | random, stored separately | needs integrity metadata; used with `--integrity` |

`aes-xts-plain64` is the modern default. XTS (XEX-based tweaked codebook with ciphertext stealing) is designed exactly for this: a tweakable block cipher where the tweak is the sector number, giving position-dependence as part of the mode rather than bolted on.

**What dm-crypt does not provide: integrity.** An attacker who can write to the device can flip ciphertext bits, which flips plaintext bits in a controlled way within a block. dm-crypt alone is **confidentiality only**. `dm-integrity` underneath (or `cryptsetup --integrity`) adds authenticated encryption (AEAD), at the cost of extra space and an extra write per sector.

This matters: "my disk is encrypted" does not mean "my disk cannot be tampered with." For a laptop that is usually fine; for a device an attacker has repeated physical access to, it is not.

**The performance architecture** is the interesting engineering:

```
bio arrives -> dm-crypt clones it, allocating pages for the ciphertext
            -> queues crypto work
            -> crypto completes (possibly async, on an engine)
            -> submits the cloned bio to the underlying device
            -> on completion, completes the original bio
```

So every write involves an allocation and a copy. Three optimisations exist and each is a tunable:

| Option | Effect |
|---|---|
| `same_cpu_crypt` | do crypto on the submitting CPU (cache locality) vs spreading |
| `submit_from_crypt_cpus` | submit the underlying bio from the crypto CPU, avoiding a handoff |
| `no_read_workqueue` / `no_write_workqueue` | do crypto inline rather than deferring to a workqueue |

`no_write_workqueue` in particular can dramatically improve latency on fast NVMe with AES-NI, because the workqueue handoff costs more than the encryption. This became the default for some configurations after measurements showed the workqueue was pure overhead on modern hardware — a good example of an optimisation becoming a pessimisation as hardware changes.

The keyring integration (`:32:logon:cryptsetup:...`) exists so the key is not visible in the dm table, which `dmsetup table` would otherwise expose.

### T.5 Thin provisioning: the metadata problem

dm-thin separates **allocation** from **addressing**:

```
thin device (virtual, e.g. 10 TB)
      |
      | block map (a B-tree in the metadata device)
      v
data pool (physical, e.g. 1 TB)
```

A thin device's blocks are allocated on first write. Many thin devices share one pool. Snapshots are metadata operations: copy the B-tree root, share the blocks, copy-on-write on divergence — exactly btrfs's mechanism (Ch. 59 §T.2) at the block layer.

Two devices are needed: a **data** device and a **metadata** device. The metadata is a persistent B-tree (`drivers/md/persistent-data/`) with its own transaction machinery, its own space map, and its own copy-on-write discipline. It is effectively a small filesystem, and it has the same problem every CoW system has: **freeing space may require allocating space** (Ch. 59 §T.9).

Hence the notorious failure mode:

| State | Meaning |
|---|---|
| **Data full** | writes to unallocated blocks fail; reads and overwrites still work |
| **Metadata full** | **the pool goes read-only and may be unrecoverable** |

Metadata exhaustion is much worse than data exhaustion, and the metadata device is small (typically 0.1 % of the data device), so it is easy to under-size. The rule of thumb — 48 bytes of metadata per data block — means a 1 TiB pool with 64 KiB blocks needs ~800 MiB of metadata. Under-provisioning it is a common and serious operational error.

`--errorwhenfull` controls behaviour on exhaustion:

| Setting | Behaviour |
|---|---|
| `queue_if_no_space` (default) | block writers indefinitely, hoping space appears |
| `error_if_no_space` | return `EIO` immediately |

The default blocks, which means processes hang in `D` state and the machine appears wedged. That is often *preferable* to returning errors (a database that gets `EIO` may corrupt itself; one that hangs recovers when space is added), but it must be a conscious choice. Monitoring the pool's fullness (`dmeventd`, or `dmsetup status`) is mandatory, not optional.

The block size is the other important choice:

- **Small blocks (64 KiB)**: less wasted space, better snapshot sharing, **more metadata**.
- **Large blocks (1 MiB+)**: less metadata, faster, **more waste and more CoW amplification** (writing 4 KiB to a shared 1 MiB block copies 1 MiB).

There is no universally right answer; it depends on the write pattern. 64 KiB–256 KiB is the usual range.

### T.6 Caching: two different bets

**dm-cache** places a fast device in front of a slow one, caching *blocks*:

```
     dm-cache
    /    |    \
origin  cache  metadata
(slow)  (fast) (small)
```

Policies (`smq` is the default and effectively the only one now):

| Policy | Idea |
|---|---|
| `mq` | multiqueue, hit-count based (superseded) |
| `smq` | **stochastic multiqueue**: less memory, better hit rates, self-tuning |
| `cleaner` | write all dirty blocks back, then you can remove the cache |

Modes:

| Mode | Writes |
|---|---|
| `writethrough` | to both; the cache holds no unique data; **safe to lose the cache device** |
| `writeback` | to the cache only, flushed later; **faster, but losing the cache loses data** |
| `passthrough` | reads bypass the cache; used while a cache's coherency is uncertain |

**dm-writecache** makes a different bet: it caches *only writes*, in persistent memory or a fast SSD, and does not attempt to cache reads at all.

The reasoning: the page cache already caches reads well (Ch. 52), so a block-level read cache duplicates it. What the page cache cannot do is make writes durable quickly — `fsync` must reach stable storage. So a small, fast, persistent write cache attacks the one thing the page cache cannot.

dm-writecache is simpler than dm-cache (no policy, no hit tracking, just a log of recent writes), and on `fsync`-heavy workloads with a PMEM or Optane device in front of spinning disks it is dramatically effective. With an ordinary SSD in front of another SSD it usually is not.

**The general lesson:** dm-cache bets that access locality exists and can be learned; dm-writecache bets that the bottleneck is write latency specifically. Knowing which bet matches your workload is the whole decision.

### T.7 dm-raid and the relationship to MD

`dm-raid` does not reimplement RAID. It is a **shim** that instantiates MD's RAID personalities (Ch. 67) as a dm target:

```
dm-raid target  ->  md personality (raid1/4/5/6/10)  ->  member devices
```

Why bother? Because it lets LVM manage RAID with the same table/suspend/resume machinery as everything else:

```sh
lvcreate --type raid5 -i 3 -L 100G -n lv vg
lvconvert --type raid6 vg/lv          # online conversion
lvconvert --repair vg/lv
```

You get RAID that participates in the dm stack — can be snapshotted, cached, thin-provisioned, moved with `pvmove` — without MD's separate `mdadm` management model.

The trade: two layers of configuration (LVM's and MD's), and diagnostics require understanding both. `dmsetup status` gives the dm view; `/sys/block/dm-N/md/` gives the MD view if you know to look.

### T.8 The testing targets

These exist because §T.1's plugin model makes them cheap, and Part 3's crash-consistency labs would be impossible without them.

| Target | What it does |
|---|---|
| **`dm-error`** | every I/O fails |
| **`dm-flakey`** | works for N seconds, then fails/corrupts for M seconds |
| **`dm-dust`** | fails specific sectors on demand (simulates bad blocks) |
| **`dm-delay`** | adds configurable latency, separately for reads and writes |
| **`dm-log-writes`** ★★★ | **records every write with its flags**; replay to any point |
| **`dm-zero`** | reads return zeros, writes are discarded |
| **`dm-verity`** | read-only integrity via a Merkle tree |
| **`dm-integrity`** | per-sector checksums or AEAD tags |
| **`dm-switch`**, **`dm-ebs`** | specialised remapping |

`dm-flakey`'s feature list is worth knowing because it models real failure modes:

```
drop_writes            writes silently discarded (reads see old data)
error_writes           writes return EIO
corrupt_bio_byte       flip a byte at an offset, in one direction
random_read_corrupt    corrupt a fraction of reads
random_write_corrupt   corrupt a fraction of writes
```

`drop_writes` is particularly good at finding bugs: it simulates a device that acknowledges writes without performing them, which is exactly the lying-device case of Ch. 61 §T.6.

`dm-log-writes` is the rigorous tool (Ch. 61 §T.10). It records the order and flags of every write to a log device, and `replay-log` can reconstruct the device state as it would have been at any flush point. **That converts crash-consistency testing from sampling to enumeration.**

`dm-verity` deserves separate mention: it is how Android and Chrome OS verify their system partitions. A Merkle tree over the device's blocks, with the root hash signed and verified at boot. Any modification to any block is detected on read. It is read-only by construction — the tree would have to be rebuilt otherwise — and is the block-level counterpart to fs-verity (Ch. 57 §T.10).

### T.9 LVM: policy over mechanism

LVM2 is entirely userspace. It:

1. Writes metadata to **physical volumes** (PVs) — an on-disk format describing the volume group.
2. Computes tables from that metadata.
3. Calls `libdevmapper` (the `/dev/mapper/control` ioctl interface) to instantiate them.

```
PV (physical volume)  = a disk or partition with LVM metadata
VG (volume group)     = a pool of PVs
LV (logical volume)   = a dm device built from extents of the VG's PVs
PE (physical extent)  = the allocation unit, default 4 MiB
```

The kernel knows nothing about VGs or PVs. It sees only tables. **All the policy — allocation, striping decisions, snapshot management, `pvmove` planning — is userspace.** That separation (Ch. 00 §T.2) is why LVM has evolved substantially without kernel changes.

Seeing through the abstraction is the useful skill:

```sh
lvcreate -L 10G -n data vg          # userspace decides extents
dmsetup table vg-data               # the kernel's view: one linear target

lvcreate -L 10G -i 3 -I 64 -n striped vg
dmsetup table vg-striped            # a striped target

lvcreate -s -L 1G -n snap vg/data
dmsetup table vg-snap               # snapshot + snapshot-origin targets
dmsetup ls --tree                   # the whole stack
```

`dmsetup ls --tree` is the single most useful LVM debugging command, because it shows what actually exists rather than what LVM's metadata claims.

`pvmove` is the showcase: it moves an LV's extents from one PV to another, online, by building a `mirror` table that copies in the background and then swapping to a `linear` table pointing at the new location. Suspend/resume (§T.3) makes each step atomic.

### T.10 The costs of stacking

Every layer adds:

| Cost | Magnitude |
|---|---|
| A `map()` call | ~100 ns |
| Possible bio cloning and allocation | ~1 µs if it happens |
| Possible splitting | depends on limits |
| Limits intersection | the stack's limits are the *minimum* of all layers |
| Loss of information | the bottom layer cannot see what the top intended |

The last is the most important and the least discussed. A filesystem that issues a `REQ_META` read with `REQ_PRIO` has told the block layer "this is important." After six dm layers, some of which clone bios, the flags may or may not survive. Similarly, cgroup attribution (Ch. 64 §T.6) can be lost by a layer that submits from a workqueue.

Measure rather than assume:

```sh
# The same workload at each level of the stack
for dev in /dev/sdb /dev/mapper/crypt /dev/mapper/vg-lv; do
  fio --filename=$dev --direct=1 --rw=randread --bs=4k --iodepth=32 ...
done
```

A rule of thumb that holds up: **dm-linear and dm-stripe are nearly free; dm-crypt costs CPU proportional to throughput; dm-thin costs a metadata lookup per new block; dm-snapshot costs a full CoW per first write.** Anything else, measure.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `drivers/md/dm.c` ★★★ | the core: bio splitting, cloning, the map path |
| `drivers/md/dm-table.c` ★★★ | §T.1's table parsing, limits stacking |
| `drivers/md/dm-ioctl.c` | the userspace interface |
| `drivers/md/dm-linear.c` ★★★ | **~200 lines; the minimal target.** Read it first. |
| `drivers/md/dm-stripe.c` ★★★ | striping; still small |
| `drivers/md/dm-crypt.c` ★★★ | §T.4; ~3800 lines |
| `drivers/md/dm-thin.c`, `dm-thin-metadata.c` ★★★ | §T.5 |
| `drivers/md/persistent-data/` ★★★ | the B-tree and space maps thin and cache use |
| `drivers/md/dm-cache-target.c`, `dm-cache-policy-smq.c` | §T.6 |
| `drivers/md/dm-writecache.c` | §T.6 |
| `drivers/md/dm-raid.c` | §T.7's MD shim |
| `drivers/md/dm-flakey.c`, `dm-dust.c`, `dm-delay.c`, `dm-log-writes.c` ★★★ | §T.8 |
| `drivers/md/dm-verity-target.c`, `dm-integrity.c` | §T.8 |
| `drivers/md/dm-snap.c`, `dm-snap-persistent.c` | the old snapshot target |
| `include/linux/device-mapper.h` ★★★ | `target_type`, `dm_target` |
| `Documentation/admin-guide/device-mapper/` ★★★ | **one document per target** |

### 1.2 `dm_target` and the table

```c
struct dm_target {
	struct dm_table *table;
	struct target_type *type;

	sector_t begin;                 /* this target's range */
	sector_t len;

	uint32_t max_io_len;            /* split bios larger than this */
	unsigned int num_flush_bios;
	unsigned int num_discard_bios;
	unsigned int num_secure_erase_bios;
	unsigned int num_write_zeroes_bios;
	unsigned int per_io_data_size;  /* like cmd_size (Ch. 65 T.4) */

	void *private;                  /* the target's state */
	char *error;

	bool flush_supported:1;
	bool discards_supported:1;
	bool limit_swap_bios:1;
	bool emulate_zone_append:1;
	bool accounts_remapped_io:1;
	bool needs_bio_set_dev:1;
	...
};

struct dm_table {
	struct mapped_device *md;
	enum dm_queue_mode type;
	unsigned int depth;
	unsigned int counts[MAX_DEPTH];
	sector_t *index[MAX_DEPTH];     /* a B-tree of targets by sector */
	unsigned int num_targets;
	unsigned int num_allocated;
	sector_t *highs;
	struct dm_target *targets;
	struct list_head devices;       /* every underlying device */
	...
};
```

The `index[MAX_DEPTH]` array is a small B-tree so that finding the target for a sector is O(log n) rather than linear. Tables with thousands of targets (a heavily fragmented LV) are routine, so this matters.

```c
struct dm_target *dm_table_find_target(struct dm_table *t, sector_t sector)
{
	unsigned int l, n = 0, k = 0;
	sector_t *node;

	if (unlikely(sector >= dm_table_get_size(t)))
		return &t->targets[t->num_targets];

	for (l = 0; l < t->depth; l++) {
		n = get_child(n, k);
		node = get_node(t, l, n);

		for (k = 0; k < KEYS_PER_NODE; k++)
			if (node[k] >= sector)
				break;
	}
	return &t->targets[(KEYS_PER_NODE * n) + k];
}
```

### 1.3 The map path

```c
static blk_status_t __map_bio(struct dm_target_io *tio)
{
	struct bio *clone = &tio->clone;
	struct dm_io *io = tio->io;
	struct dm_target *ti = tio->ti;
	int r;

	clone->bi_end_io = clone_endio;

	/* dm-stats and zone handling */
	...
	down_read(&md->io_lock);
	r = ti->type->map(ti, clone);
	up_read(&md->io_lock);

	switch (r) {
	case DM_MAPIO_SUBMITTED:
		/* The target took ownership. Nothing to do. */
		break;
	case DM_MAPIO_REMAPPED:
		/* The target rewrote bi_bdev/bi_sector. Submit it. */
		if (unlikely(swap_bios_limit(ti, clone)))
			dm_io_set_flag(io, DM_IO_WAS_SPLIT);
		trace_block_bio_remap(clone, disk_devt(md->disk), sector);
		submit_bio_noacct(clone);
		break;
	case DM_MAPIO_KILL:
		...
		dm_io_dec_pending(io, BLK_STS_IOERR);
		break;
	case DM_MAPIO_REQUEUE:
		...
		dm_io_dec_pending(io, BLK_STS_DM_REQUEUE);
		break;
	default:
		DMCRIT("unimplemented target map return value: %d", r);
		BUG();
	}
	return ret;
}
```

Splitting across targets:

```c
static void __split_and_process_bio(struct clone_info *ci)
{
	struct dm_target *ti;
	unsigned int len;

	ti = dm_table_find_target(ci->map, ci->sector);
	if (unlikely(!ti))
		return __send_empty_flush(ci);
	...
	/* How much of this bio does THIS target cover? */
	len = min_t(sector_t, max_io_len(ti, ci->sector), ci->sector_count);
	setup_split_accounting(ci, len);

	if (unlikely(ci->bio->bi_opf & REQ_NOWAIT)) {
		if (unlikely(!dm_target_supports_nowait(ti->type)))
			return BLK_STS_NOTSUPP;
		if (unlikely(!blk_queue_nowait(bdev_get_queue(...))))
			return BLK_STS_AGAIN;
	}

	if (__map_bio(...))
		return;

	ci->sector += len;
	ci->sector_count -= len;
}
```

A bio spanning two targets is split and each piece mapped separately, with `bio_chain` (Ch. 63 §T.2) ensuring the original completes only when both do.

### 1.4 dm-linear, in full

```c
/* drivers/md/dm-linear.c -- essentially the whole target */
struct linear_c {
	struct dm_dev *dev;
	sector_t start;
};

static int linear_ctr(struct dm_target *ti, unsigned int argc, char **argv)
{
	struct linear_c *lc;
	unsigned long long tmp;
	char dummy;
	int ret;

	if (argc != 2) {
		ti->error = "Invalid argument count";
		return -EINVAL;
	}

	lc = kmalloc(sizeof(*lc), GFP_KERNEL);
	if (lc == NULL) {
		ti->error = "Cannot allocate linear context";
		return -ENOMEM;
	}

	ret = -EINVAL;
	if (sscanf(argv[1], "%llu%c", &tmp, &dummy) != 1 || tmp != (sector_t)tmp) {
		ti->error = "Invalid device sector";
		goto bad;
	}
	lc->start = tmp;

	/* This is how a target acquires an underlying device. */
	ret = dm_get_device(ti, argv[0], dm_table_get_mode(ti->table), &lc->dev);
	if (ret) {
		ti->error = "Device lookup failed";
		goto bad;
	}

	ti->num_flush_bios = 1;
	ti->num_discard_bios = 1;
	ti->num_secure_erase_bios = 1;
	ti->num_write_zeroes_bios = 1;
	ti->private = lc;
	return 0;

bad:
	kfree(lc);
	return ret;
}

static sector_t linear_map_sector(struct dm_target *ti, sector_t bi_sector)
{
	struct linear_c *lc = ti->private;

	return lc->start + dm_target_offset(ti, bi_sector);
}

static int linear_map(struct dm_target *ti, struct bio *bio)
{
	struct linear_c *lc = ti->private;

	bio_set_dev(bio, lc->dev->bdev);
	if (bio_sectors(bio) || op_is_zone_mgmt(bio_op(bio)))
		bio->bi_iter.bi_sector = linear_map_sector(ti, bio->bi_iter.bi_sector);

	return DM_MAPIO_REMAPPED;
}

static void linear_status(struct dm_target *ti, status_type_t type,
			  unsigned int status_flags, char *result, unsigned int maxlen)
{
	struct linear_c *lc = ti->private;
	size_t sz = 0;

	switch (type) {
	case STATUSTYPE_INFO:
		result[0] = '\0';
		break;
	case STATUSTYPE_TABLE:
		DMEMIT("%s %llu", lc->dev->name, (unsigned long long)lc->start);
		break;
	case STATUSTYPE_IMA:
		DMEMIT_TARGET_NAME_VERSION(ti->type);
		DMEMIT(",device_name=%s,start=%llu;", lc->dev->name,
		       (unsigned long long)lc->start);
		break;
	}
}

static int linear_iterate_devices(struct dm_target *ti,
				  iterate_devices_callout_fn fn, void *data)
{
	struct linear_c *lc = ti->private;

	return fn(ti, lc->dev, lc->start, ti->len, data);
}

static struct target_type linear_target = {
	.name   = "linear",
	.version = {1, 4, 0},
	.features = DM_TARGET_PASSES_INTEGRITY | DM_TARGET_NOWAIT |
		    DM_TARGET_ZONED_HM | DM_TARGET_PASSES_CRYPTO |
		    DM_TARGET_ATOMIC_WRITES,
	.report_zones = linear_report_zones,
	.module = THIS_MODULE,
	.ctr    = linear_ctr,
	.dtr    = linear_dtr,
	.map    = linear_map,
	.status = linear_status,
	.prepare_ioctl = linear_prepare_ioctl,
	.iterate_devices = linear_iterate_devices,
};
```

**That is a complete, production, shipping target in about 200 lines.** Read it before writing anything.

### 1.5 dm-crypt's I/O path

```c
struct dm_crypt_io {
	struct crypt_config *cc;
	struct bio *base_bio;          /* the ORIGINAL bio */
	u8 *integrity_metadata;
	struct work_struct work;
	struct tasklet_struct tasklet;
	struct convert_context ctx;
	atomic_t io_pending;
	blk_status_t error;
	sector_t sector;
	struct rb_node rb_node;
} CRYPTO_MINALIGN_ATTR;

static int crypt_map(struct dm_target *ti, struct bio *bio)
{
	struct dm_crypt_io *io;
	struct crypt_config *cc = ti->private;

	/* Flushes and other zero-data bios pass straight through. */
	if (unlikely(bio->bi_opf & REQ_PREFLUSH || bio_op(bio) == REQ_OP_DISCARD)) {
		bio_set_dev(bio, cc->dev->bdev);
		if (bio_sectors(bio))
			bio->bi_iter.bi_sector = cc->start +
				dm_target_offset(ti, bio->bi_iter.bi_sector);
		return DM_MAPIO_REMAPPED;
	}

	/* Enforce the sector-size contract. */
	if (unlikely((bio->bi_iter.bi_size & (cc->sector_size - 1)) != 0))
		return DM_MAPIO_KILL;

	io = dm_per_bio_data(bio, cc->per_bio_data_size);
	crypt_io_init(io, cc, bio, dm_target_offset(ti, bio->bi_iter.bi_sector));
	...
	if (bio_data_dir(io->base_bio) == READ) {
		if (kcryptd_io_read(io, CRYPT_MAP_READ_GFP))
			kcryptd_queue_read(io);
	} else {
		kcryptd_queue_crypt(io);
	}

	return DM_MAPIO_SUBMITTED;     /* WE own it now (T.2) */
}
```

Note `dm_per_bio_data(bio, cc->per_bio_data_size)` — the same preallocation trick as `blk_mq_rq_to_pdu` (Ch. 65 §T.4). The target declares `ti->per_io_data_size` and dm allocates that much extra with each cloned bio, so there is **no allocation in the map path**.

The IV generation:

```c
static int crypt_iv_plain64_gen(struct crypt_config *cc, u8 *iv,
				struct dm_crypt_request *dmreq)
{
	memset(iv, 0, cc->iv_size);
	*(__le64 *)iv = cpu_to_le64(dmreq->iv_sector);
	return 0;
}

static int crypt_iv_essiv_gen(struct crypt_config *cc, u8 *iv,
			      struct dm_crypt_request *dmreq)
{
	/* IV = E(hash(key), sector) -- unpredictable without the key */
	memset(iv, 0, cc->iv_size);
	*(__le64 *)iv = cpu_to_le64(dmreq->iv_sector);
	crypto_cipher_encrypt_one(essiv->essiv_tfm, iv, iv);
	return 0;
}
```

### 1.6 Observability

| Where | What |
|---|---|
| `dmsetup ls --tree` ★★★ | **the actual stack** |
| `dmsetup table [--showkeys]` ★★★ | the table; `--showkeys` reveals dm-crypt keys |
| `dmsetup status` ★★★ | per-target runtime state |
| `dmsetup info -c` | state, open count, event number |
| `dmsetup deps` | underlying devices |
| `dmsetup targets` | available target types |
| `dmsetup message NAME 0 CMD` | runtime commands to a target |
| `/sys/block/dm-N/dm/{name,uuid,suspended}` | |
| `/sys/block/dm-N/queue/*` | the stacked limits (Ch. 63 §T.9) |
| `dmstats` ★★★ | per-region I/O statistics |
| `trace-cmd record -e block:block_bio_remap` ★★★ | watch remapping |
| `lvs -a -o+devices,segtype,stripes` ★★★ | LVM's view |
| `lvdisplay -m` | the segment map |
| `dmeventd` / `/etc/lvm/lvm.conf` monitoring | thin pool alerts |

`dmstats` is underused and excellent — it can attach counters to arbitrary sector ranges:

```sh
dmstats create --areasize 1G vg-lv
dmstats report --interval 1 --count 5
```

so you can see which *region* of a logical volume is hot, not just the device as a whole.

---

## 2. Practice

### Lab 66.1 — Build a stack by hand

```sh
sudo apt install -y dmsetup lvm2 cryptsetup thin-provisioning-tools
sudo modprobe scsi_debug dev_size_mb=1024 num_tgts=4
DEVS=($(lsblk -ndo NAME,MODEL | awk '/scsi_debug/{print "/dev/"$1}'))
echo "${DEVS[@]}"
SZ=$(sudo blockdev --getsz ${DEVS[0]})
```

Concatenation:

```sh
HALF=$((SZ / 2))
sudo dmsetup create concat --table "\
0 $HALF linear ${DEVS[0]} 0
$HALF $HALF linear ${DEVS[1]} 0"

sudo dmsetup table concat
sudo dmsetup info -c concat
lsblk /dev/mapper/concat
sudo blockdev --getsize64 /dev/mapper/concat
```

Prove the mapping:

```sh
# Write a marker at the start of each half
sudo dd if=/dev/zero of=/dev/mapper/concat bs=512 count=1 2>/dev/null
echo "FIRST HALF" | sudo dd of=/dev/mapper/concat bs=512 seek=0 conv=notrunc 2>/dev/null
echo "SECOND HALF" | sudo dd of=/dev/mapper/concat bs=512 seek=$HALF conv=notrunc 2>/dev/null

sudo dd if=${DEVS[0]} bs=512 count=1 2>/dev/null | head -c 20; echo
sudo dd if=${DEVS[1]} bs=512 count=1 2>/dev/null | head -c 20; echo
```

Striping:

```sh
sudo dmsetup create striped --table \
  "0 $((SZ*2)) striped 2 128 ${DEVS[0]} 0 ${DEVS[1]} 0"
sudo dmsetup table striped

# Watch I/O alternate between devices
sudo blktrace -d ${DEVS[0]} -d ${DEVS[1]} -o /tmp/st &
sudo dd if=/dev/zero of=/dev/mapper/striped bs=64k count=100 oflag=direct 2>/dev/null
sudo pkill blktrace; sleep 1
for d in ${DEVS[0]} ${DEVS[1]}; do
  echo -n "$(basename $d): "
  blkparse -i /tmp/st.$(basename $d) 2>/dev/null | grep -c ' D '
done
```

Nesting:

```sh
sudo dmsetup create nested --table \
  "0 $((SZ*2)) linear /dev/mapper/striped 0"
sudo dmsetup ls --tree
sudo dmsetup deps nested
```

Stacked limits (Ch. 63 §T.9):

```sh
for d in $(basename ${DEVS[0]}) dm-0 dm-1 dm-2; do
  echo "=== $d ==="
  for f in max_sectors_kb max_segments logical_block_size physical_block_size \
           optimal_io_size minimum_io_size; do
    printf "  %-20s %s\n" $f "$(cat /sys/block/$d/queue/$f 2>/dev/null)"
  done
done
# Note optimal_io_size on the striped device = stripe width.
```

Clean up:

```sh
sudo dmsetup remove nested striped concat
```

---

### Lab 66.2 — Suspend and resume: atomic reconfiguration

```sh
sudo dmsetup create live --table "0 $SZ linear ${DEVS[0]} 0"
sudo mkfs.ext4 -qF /dev/mapper/live
sudo mkdir -p /mnt/live && sudo mount /dev/mapper/live /mnt/live
sudo sh -c 'echo "before" > /mnt/live/marker'
sync
```

Swap the table **while mounted**:

```sh
# Copy the data to the second device
sudo dd if=${DEVS[0]} of=${DEVS[1]} bs=1M 2>/dev/null

sudo dmsetup suspend live
sudo dmsetup table live
sudo dmsetup reload live --table "0 $SZ linear ${DEVS[1]} 0"
sudo dmsetup resume live

sudo dmsetup table live                  # now points at DEVS[1]
cat /mnt/live/marker                     # still works; still mounted
sudo sh -c 'echo "after" >> /mnt/live/marker'
sync
sudo dd if=${DEVS[1]} bs=1M count=100 2>/dev/null | strings | grep -c after
```

**The filesystem never noticed.** That is §T.3.

Suspend with I/O in flight:

```sh
sudo fio --name=t --filename=/mnt/live/f --direct=1 --rw=randwrite --bs=4k \
         --size=64m --iodepth=16 --ioengine=libaio --runtime=30 \
         --time_based > /dev/null 2>&1 &
FIO=$!
sleep 2

echo "suspending with I/O in flight..."
time sudo dmsetup suspend live           # waits for in-flight I/O
sudo dmsetup info -c live | head -2      # State: SUSPENDED

# The fio process is now blocked
ps -o pid,stat,comm -p $(pgrep -f 'name=t') 2>/dev/null

sudo dmsetup resume live
wait $FIO 2>/dev/null
```

Filesystem freeze:

```sh
# With --nolockfs: block-level consistency only
sudo dmsetup suspend --nolockfs live
sudo dmsetup resume live

# Without: freeze_bdev() is called, giving filesystem consistency
sudo trace-cmd record -e ext4:\* -- sh -c '
  sudo dmsetup suspend live; sudo dmsetup resume live' 2>/dev/null
sudo trace-cmd report | grep -i 'freeze\|sync' | head

# Directly:
sudo fsfreeze --freeze /mnt/live
cat /proc/self/mountinfo | grep live
sudo touch /mnt/live/x &                 # blocks
sleep 1
sudo fsfreeze --unfreeze /mnt/live
wait
```

noflush suspend:

```sh
# Simulate the underlying device vanishing
sudo dmsetup create ontop --table "0 $SZ linear /dev/mapper/live 0"
sudo dmsetup suspend --noflush ontop
sudo dmsetup info -c ontop
sudo dmsetup resume ontop
sudo dmsetup remove ontop
```

Cleanup:

```sh
sudo umount /mnt/live
sudo dmsetup remove live
```

---

### Lab 66.3 — dm-crypt

```sh
# By hand, to see the table
KEY=$(head -c 32 /dev/urandom | xxd -p -c 64)
sudo dmsetup create crypt --table \
  "0 $SZ crypt aes-xts-plain64 $KEY 0 ${DEVS[0]} 0"

sudo dmsetup table crypt                 # key is HIDDEN
sudo dmsetup table --showkeys crypt      # key is visible: be careful
sudo dmsetup status crypt
```

Prove encryption:

```sh
sudo mkfs.ext4 -qF /dev/mapper/crypt
sudo mkdir -p /mnt/crypt && sudo mount /dev/mapper/crypt /mnt/crypt
sudo sh -c 'echo "PLAINTEXT SECRET STRING" > /mnt/crypt/secret'
sync

sudo grep -a -c 'PLAINTEXT SECRET' /dev/mapper/crypt   # 1
sudo grep -a -c 'PLAINTEXT SECRET' ${DEVS[0]}          # 0
sudo umount /mnt/crypt
```

Entropy comparison:

```sh
sudo dd if=${DEVS[0]} bs=1M count=10 2>/dev/null | ent 2>/dev/null || \
sudo dd if=${DEVS[0]} bs=1M count=10 2>/dev/null | \
  python3 -c "
import sys, collections, math
d = sys.stdin.buffer.read()
c = collections.Counter(d)
h = -sum((n/len(d)) * math.log2(n/len(d)) for n in c.values())
print(f'entropy: {h:.3f} bits/byte (8.0 = random)')"
```

IV modes:

```sh
sudo dmsetup remove crypt
for iv in plain64 essiv:sha256 ; do
  sudo dmsetup create c_$iv --table \
    "0 $SZ crypt aes-cbc-$iv $KEY 0 ${DEVS[0]} 0" 2>/dev/null || continue
  echo "=== aes-cbc-$iv ==="
  sudo dmsetup table c_$iv | awk '{print $4}'
  sudo dmsetup remove c_$iv
done
```

Performance and the tunables of §T.4:

```sh
for opts in "" "1 no_write_workqueue" "2 no_read_workqueue no_write_workqueue" \
            "1 same_cpu_crypt"; do
  sudo dmsetup remove perf 2>/dev/null
  sudo dmsetup create perf --table \
    "0 $SZ crypt aes-xts-plain64 $KEY 0 ${DEVS[0]} 0 $opts" 2>/dev/null || continue
  echo "=== opts: ${opts:-default} ==="
  sudo fio --name=t --filename=/dev/mapper/perf --direct=1 --rw=randwrite \
           --bs=4k --iodepth=32 --ioengine=libaio --runtime=8 --time_based \
           --numjobs=4 --group_reporting 2>/dev/null | grep -E 'IOPS|clat.*avg'
done
sudo dmsetup remove perf 2>/dev/null
```

Crypto engine usage:

```sh
grep -E 'aes|xts' /proc/crypto | head -20
cat /proc/crypto | grep -A5 'name.*xts(aes)' | head -12
grep -o 'aes' /proc/cpuinfo | head -1    # AES-NI present?

sudo bpftrace -e '
kprobe:crypt_convert { @convert = count(); }
kprobe:kcryptd_crypt  { @queued = count(); }
interval:s:5 { print(@convert); print(@queued);
               clear(@convert); clear(@queued); }' &
sudo dd if=/dev/zero of=/dev/mapper/crypt bs=1M count=500 oflag=direct 2>/dev/null
```

The real tool — `cryptsetup`, which manages keys properly:

```sh
sudo dmsetup remove crypt 2>/dev/null
echo -n "testpassphrase" | sudo cryptsetup luksFormat --batch-mode ${DEVS[0]}
sudo cryptsetup luksDump ${DEVS[0]}
echo -n "testpassphrase" | sudo cryptsetup open ${DEVS[0]} luksdev
sudo dmsetup table luksdev              # the same dm-crypt target underneath
sudo dmsetup ls --tree

sudo cryptsetup status luksdev
sudo cryptsetup benchmark 2>&1 | head -20
```

With integrity (§T.4's caveat):

```sh
sudo cryptsetup close luksdev
echo -n "pass" | sudo cryptsetup luksFormat --batch-mode \
  --integrity hmac-sha256 --sector-size 4096 ${DEVS[1]} 2>&1 | tail -3
echo -n "pass" | sudo cryptsetup open ${DEVS[1]} intdev 2>/dev/null && {
  sudo dmsetup ls --tree            # dm-crypt ON TOP of dm-integrity
  sudo cryptsetup close intdev
}
```

---

### Lab 66.4 — Thin provisioning

```sh
sudo dmsetup remove_all 2>/dev/null
META=${DEVS[0]}; DATA=${DEVS[1]}
DATA_SZ=$(sudo blockdev --getsz $DATA)

# Zero the metadata device -- required for a fresh pool
sudo dd if=/dev/zero of=$META bs=4k count=1 2>/dev/null

# 128-sector (64 KiB) blocks, low-water mark 1024 blocks
sudo dmsetup create pool --table \
  "0 $DATA_SZ thin-pool $META $DATA 128 1024"
sudo dmsetup status pool
```

```
0 2097152 thin-pool 0 12/32768 0/16384 - rw discard_passdown queue_if_no_space -
                      ^          ^
                      metadata   data used/total
```

Create thin volumes — **each larger than the pool**:

```sh
sudo dmsetup message pool 0 "create_thin 0"
sudo dmsetup message pool 0 "create_thin 1"

VIRT=$((DATA_SZ * 4))     # 4x overcommit
sudo dmsetup create thin0 --table "0 $VIRT thin /dev/mapper/pool 0"
sudo dmsetup create thin1 --table "0 $VIRT thin /dev/mapper/pool 1"

lsblk /dev/mapper/thin0 /dev/mapper/thin1
sudo dmsetup status pool
```

Allocation on write:

```sh
sudo dmsetup status pool | awk '{print "used:", $6}'
sudo dd if=/dev/zero of=/dev/mapper/thin0 bs=1M count=100 oflag=direct 2>/dev/null
sudo dmsetup status pool | awk '{print "used:", $6}'
sudo dd if=/dev/zero of=/dev/mapper/thin1 bs=1M count=100 oflag=direct 2>/dev/null
sudo dmsetup status pool | awk '{print "used:", $6}'
```

Snapshots:

```sh
sudo mkfs.ext4 -qF /dev/mapper/thin0
sudo mkdir -p /mnt/thin && sudo mount /dev/mapper/thin0 /mnt/thin
sudo sh -c 'echo "original" > /mnt/thin/data'
sync

# Snapshot requires the origin to be quiescent
sudo dmsetup suspend thin0
sudo dmsetup message pool 0 "create_snap 2 0"
sudo dmsetup resume thin0

sudo dmsetup create snap --table "0 $VIRT thin /dev/mapper/pool 2"
sudo dmsetup status pool | awk '{print "used after snapshot:", $6}'   # unchanged

sudo mkdir -p /mnt/snap && sudo mount -o ro /dev/mapper/snap /mnt/snap
cat /mnt/snap/data

sudo sh -c 'echo "modified" > /mnt/thin/data'; sync
cat /mnt/thin/data /mnt/snap/data
sudo dmsetup status pool | awk '{print "used after divergence:", $6}'
```

**Now the failure mode of §T.5.** Fill the pool:

```sh
sudo umount /mnt/snap /mnt/thin
sudo dmsetup status pool

sudo dd if=/dev/urandom of=/dev/mapper/thin0 bs=1M oflag=direct 2>&1 | tail -2 &
DD=$!
for i in $(seq 1 20); do
  sudo dmsetup status pool | awk '{print $6, $8}'
  sleep 1
done
```

With `queue_if_no_space` (the default) the `dd` **hangs**:

```sh
ps -o pid,stat,comm -p $DD
sudo cat /proc/$DD/stack 2>/dev/null | head
dmesg | tail -5      # "Data device ... is now full"
```

Recover by adding space:

```sh
sudo dmsetup suspend pool
# In practice: extend the data device, then reload with the new size.
sudo dmsetup resume pool
```

Or switch to erroring:

```sh
sudo dmsetup reload pool --table \
  "0 $DATA_SZ thin-pool $META $DATA 128 1024 1 error_if_no_space"
sudo dmsetup suspend pool && sudo dmsetup resume pool
sudo dd if=/dev/urandom of=/dev/mapper/thin1 bs=1M count=100 oflag=direct 2>&1 | tail -2
# Now it returns EIO immediately instead of hanging.
```

Metadata exhaustion — the dangerous one:

```sh
sudo dmsetup status pool | awk '{print "metadata:", $5}'
# Monitor this. If it fills, the pool goes read-only.
sudo thin_check $META 2>&1 | head
sudo thin_dump $META 2>/dev/null | head -20
```

Block size trade-off:

```sh
for bs in 128 512 2048; do
  sudo dmsetup remove thin0 pool 2>/dev/null
  sudo dd if=/dev/zero of=$META bs=4k count=1 2>/dev/null
  sudo dmsetup create pool --table "0 $DATA_SZ thin-pool $META $DATA $bs 1024"
  sudo dmsetup message pool 0 "create_thin 0"
  sudo dmsetup create thin0 --table "0 $VIRT thin /dev/mapper/pool 0"
  sudo dd if=/dev/zero of=/dev/mapper/thin0 bs=4k count=1000 oflag=direct 2>/dev/null
  echo -n "block size $((bs/2))K: "
  sudo dmsetup status pool | awk '{print "metadata", $5, "data", $6}'
done
```

Small writes with large blocks waste enormous space — §T.5's trade, measured.

---

### Lab 66.5 — Caching

```sh
sudo dmsetup remove_all 2>/dev/null
SLOW=${DEVS[0]}; FAST=${DEVS[1]}; CMETA=${DEVS[2]}

# Make SLOW actually slow
sudo dmsetup create slow --table \
  "0 $SZ delay $SLOW 0 10"        # 10 ms delay
sudo dmsetup table slow

echo -n "slow device: "
sudo dd if=/dev/mapper/slow of=/dev/null bs=4k count=100 iflag=direct 2>&1 | tail -1
```

dm-cache:

```sh
sudo dd if=/dev/zero of=$CMETA bs=4k count=1 2>/dev/null
CACHE_SZ=$(sudo blockdev --getsz $FAST)

sudo dmsetup create cached --table \
  "0 $SZ cache $CMETA $FAST /dev/mapper/slow 512 1 writeback smq 0"
sudo dmsetup status cached
```

```
0 2097152 cache 8 340/32768 512 0/4096 0 0 0 0 0 0 0 1 writeback 2 migration_threshold 2048 smq 0 rw -
                  ^metadata      ^cache blocks used/total  ^reads ^read hits ^writes ^write hits
```

Watch the cache warm:

```sh
for i in 1 2 3 4 5; do
  echo -n "pass $i: "
  sudo dd if=/dev/mapper/cached of=/dev/null bs=64k count=500 iflag=direct 2>&1 | \
    grep -oP '[0-9.]+ [kMG]B/s'
  sudo dmsetup status cached | awk '{printf "  hits: r=%s/%s w=%s/%s\n", $9, $8, $11, $10}'
done
```

Throughput should climb as the working set migrates into the cache.

Write modes:

```sh
for mode in writethrough writeback; do
  sudo dmsetup remove cached 2>/dev/null
  sudo dd if=/dev/zero of=$CMETA bs=4k count=1 2>/dev/null
  sudo dmsetup create cached --table \
    "0 $SZ cache $CMETA $FAST /dev/mapper/slow 512 1 $mode smq 0"
  echo -n "$mode: "
  sudo dd if=/dev/zero of=/dev/mapper/cached bs=64k count=500 oflag=direct 2>&1 | \
    grep -oP '[0-9.]+ [kMG]B/s'
done
```

Draining the cache before removal (essential for `writeback`):

```sh
sudo dmsetup suspend cached
sudo dmsetup reload cached --table \
  "0 $SZ cache $CMETA $FAST /dev/mapper/slow 512 0 cleaner 0"
sudo dmsetup resume cached
while sudo dmsetup status cached | grep -q 'dirty'; do
  sudo dmsetup status cached | awk '{print "dirty blocks:", $12}'
  sleep 1
done
```

dm-writecache — the different bet (§T.6):

```sh
sudo dmsetup remove cached 2>/dev/null
sudo dd if=/dev/zero of=$FAST bs=4k count=1 2>/dev/null
sudo dmsetup create wcache --table \
  "0 $SZ writecache s /dev/mapper/slow $FAST 4096 0"
sudo dmsetup status wcache

# fsync-heavy workload: where writecache shines
for dev in /dev/mapper/slow /dev/mapper/wcache; do
  echo -n "$(basename $dev): "
  sudo fio --name=t --filename=$dev --direct=1 --rw=randwrite --bs=4k \
           --iodepth=1 --fsync=1 --runtime=8 --time_based 2>/dev/null | \
    grep -oP 'IOPS=\K[^,]+'
done

# But it does NOT help reads
for dev in /dev/mapper/slow /dev/mapper/wcache; do
  echo -n "$(basename $dev) reads: "
  sudo fio --name=t --filename=$dev --direct=1 --rw=randread --bs=4k \
           --iodepth=8 --runtime=8 --time_based 2>/dev/null | grep -oP 'IOPS=\K[^,]+'
done
```

---

### Lab 66.6 — The testing targets

These are the targets that made Chapters 56, 61, and 63's labs possible.

`dm-delay` — model any device:

```sh
sudo dmsetup remove_all 2>/dev/null
for delay in 0 1 10 100; do
  sudo dmsetup remove d 2>/dev/null
  sudo dmsetup create d --table "0 $SZ delay ${DEVS[0]} 0 $delay"
  echo -n "${delay}ms delay: "
  sudo fio --name=t --filename=/dev/mapper/d --direct=1 --rw=randread \
           --bs=4k --iodepth=1 --runtime=5 --time_based 2>/dev/null | \
    grep -oP 'IOPS=\K[^,]+'
done

# Asymmetric read/write delay
sudo dmsetup remove d
sudo dmsetup create d --table "0 $SZ delay ${DEVS[0]} 0 1 ${DEVS[0]} 0 50"
sudo dmsetup table d
```

`dm-error` and `dm-flakey`:

```sh
sudo dmsetup create err --table "0 $SZ error"
sudo dd if=/dev/mapper/err of=/dev/null bs=4k count=1 2>&1 | tail -2

sudo dmsetup remove err
# Works 5s, fails 5s
sudo dmsetup create flaky --table "0 $SZ flakey ${DEVS[0]} 0 5 5"
for i in $(seq 1 12); do
  printf "%2ds: " $i
  sudo dd if=/dev/mapper/flaky of=/dev/null bs=4k count=1 iflag=direct 2>&1 | tail -1
  sleep 1
done
```

`drop_writes` — simulating a lying device (Ch. 61 §T.6):

```sh
sudo dmsetup remove flaky
sudo dmsetup create flaky --table \
  "0 $SZ flakey ${DEVS[0]} 0 0 1 1 drop_writes"
sudo mkfs.ext4 -qF /dev/mapper/flaky 2>&1 | tail -2
# Writes are ACKNOWLEDGED but discarded. Filesystems hate this, correctly.

sudo dmsetup remove flaky
sudo dmsetup create flaky --table \
  "0 $SZ flakey ${DEVS[0]} 0 10 5 2 corrupt_bio_byte 32 r 1 0"
sudo mkfs.ext4 -qF /dev/mapper/flaky
sudo mount /dev/mapper/flaky /mnt/live 2>&1 | tail -2
```

`dm-dust` — targeted bad blocks:

```sh
sudo dmsetup remove flaky 2>/dev/null
sudo dmsetup create dust --table "0 $SZ dust ${DEVS[0]} 0 512"
sudo dmsetup message dust 0 addbadblock 60
sudo dmsetup message dust 0 addbadblock 61
sudo dmsetup message dust 0 enable
sudo dmsetup message dust 0 countbadblocks

sudo dd if=/dev/mapper/dust of=/dev/null bs=512 skip=60 count=1 2>&1 | tail -2
dmesg | tail -3
sudo dmsetup message dust 0 removebadblock 60
sudo dd if=/dev/mapper/dust of=/dev/null bs=512 skip=60 count=1 2>&1 | tail -1
```

`dm-log-writes` — the rigorous tool:

```sh
sudo dmsetup remove dust
sudo dmsetup create lw --table "0 $SZ log-writes ${DEVS[0]} ${DEVS[1]}"

sudo mkfs.ext4 -qF /dev/mapper/lw
sudo dmsetup message lw 0 mark mkfs
sudo mount /dev/mapper/lw /mnt/live
sudo sh -c 'echo "v1" > /mnt/live/f; sync'
sudo dmsetup message lw 0 mark v1
sudo sh -c 'echo "v2" > /mnt/live/f.tmp; mv /mnt/live/f.tmp /mnt/live/f'
sudo dmsetup message lw 0 mark v2_nosync
sudo umount /mnt/live

# Replay to each mark and check (Ch. 61 T.10)
git clone -q https://git.kernel.org/pub/scm/fs/xfs/xfstests-dev.git 2>/dev/null
make -C xfstests-dev/src/log-writes 2>/dev/null
REPLAY=xfstests-dev/src/log-writes/replay-log

for m in mkfs v1 v2_nosync; do
  sudo $REPLAY --log ${DEVS[1]} --replay ${DEVS[0]} --end-mark $m 2>/dev/null
  echo -n "$m: "
  sudo e2fsck -fn ${DEVS[0]} > /dev/null 2>&1 && echo -n "clean, " || echo -n "DIRTY, "
  sudo mount ${DEVS[0]} /mnt/live 2>/dev/null && {
    cat /mnt/live/f 2>/dev/null || echo "(no file)"
    sudo umount /mnt/live
  } || echo "(unmountable)"
done
```

`dm-verity`:

```sh
sudo dmsetup remove lw 2>/dev/null
sudo mkfs.ext4 -qF ${DEVS[0]}
sudo mount ${DEVS[0]} /mnt/live
sudo cp -r /usr/include /mnt/live/ 2>/dev/null
sudo umount /mnt/live

ROOT=$(sudo veritysetup format ${DEVS[0]} ${DEVS[1]} 2>/dev/null | \
       grep 'Root hash' | awk '{print $3}')
echo "root hash: $ROOT"
sudo veritysetup create vroot ${DEVS[0]} ${DEVS[1]} $ROOT
sudo dmsetup table vroot
sudo mount -o ro /dev/mapper/vroot /mnt/live
ls /mnt/live | head
sudo umount /mnt/live

# Corrupt a block and watch verification fail
sudo veritysetup close vroot
sudo dd if=/dev/urandom of=${DEVS[0]} bs=4k seek=2000 count=1 conv=notrunc 2>/dev/null
sudo veritysetup create vroot ${DEVS[0]} ${DEVS[1]} $ROOT
sudo mount -o ro /dev/mapper/vroot /mnt/live 2>/dev/null
sudo find /mnt/live -type f -exec cat {} \; > /dev/null 2>&1
dmesg | grep -i verity | tail -5
sudo umount /mnt/live 2>/dev/null; sudo veritysetup close vroot
```

---

### Lab 66.7 — LVM, and seeing through it

```sh
sudo dmsetup remove_all 2>/dev/null
sudo pvcreate -ff -y ${DEVS[@]}
sudo vgcreate testvg ${DEVS[@]}
sudo vgdisplay testvg
sudo pvs -o+pv_used
```

Every LVM operation, shown as the dm table it produces:

```sh
sudo lvcreate -L 200M -n linear testvg
sudo dmsetup table testvg-linear
sudo lvs -o+devices,segtype testvg/linear

sudo lvcreate -L 200M -i 3 -I 64 -n striped testvg
sudo dmsetup table testvg-striped

sudo lvcreate --type raid1 -m1 -L 100M -n mirrored testvg
sudo dmsetup ls --tree | head -20
sudo lvs -a -o+devices,segtype testvg

sudo lvcreate -L 500M --thinpool tpool testvg
sudo dmsetup ls --tree
sudo lvcreate -V 2G -T testvg/tpool -n thinlv
sudo lvs -a -o+segtype testvg
```

Online operations — §T.3 in action:

```sh
sudo mkfs.ext4 -qF /dev/testvg/linear
sudo mount /dev/testvg/linear /mnt/live
sudo sh -c 'echo data > /mnt/live/f'; sync

# Extend while mounted
df -h /mnt/live
sudo lvextend -L +200M -r testvg/linear
df -h /mnt/live
cat /mnt/live/f

# Move to a different PV while mounted
sudo lvs -o+devices testvg/linear
sudo pvmove -n testvg/linear ${DEVS[0]} ${DEVS[3]} 2>&1 | tail -3
sudo lvs -o+devices testvg/linear
cat /mnt/live/f
```

Watch what `pvmove` does:

```sh
sudo lvcreate -L 200M -n moveme testvg
sudo pvmove -n testvg/moveme ${DEVS[1]} ${DEVS[2]} -b 2>/dev/null
sleep 1
sudo dmsetup ls --tree              # a temporary pvmove mirror device
sudo dmsetup table | grep -i mirror
sudo pvmove --abort 2>/dev/null
```

Snapshots:

```sh
sudo lvcreate -s -L 100M -n snap testvg/linear
sudo dmsetup ls --tree
sudo dmsetup table testvg-snap
sudo dmsetup table testvg-linear     # now a snapshot-origin target!
sudo lvs testvg

# Writing to the origin triggers CoW
sudo dd if=/dev/urandom of=/mnt/live/big bs=1M count=50 2>/dev/null; sync
sudo lvs -o+snap_percent testvg/snap

# Fill the snapshot and watch it become invalid
sudo dd if=/dev/urandom of=/mnt/live/big2 bs=1M count=150 2>/dev/null; sync
sudo lvs -o+snap_percent,lv_attr testvg/snap
dmesg | tail -3
```

Cleanup:

```sh
sudo umount /mnt/live
sudo vgremove -f testvg
sudo pvremove -ff -y ${DEVS[@]}
```

---

### Lab 66.8 — Write a dm target

```c
// SPDX-License-Identifier: GPL-2.0
/* dm-count.c -- a linear target that counts and optionally delays I/O.
 * Demonstrates ctr/dtr/map/status/message/iterate_devices. */
#include <linux/device-mapper.h>
#include <linux/module.h>
#include <linux/init.h>
#include <linux/blkdev.h>
#include <linux/bio.h>
#include <linux/delay.h>

#define DM_MSG_PREFIX "count"

struct count_c {
	struct dm_dev	*dev;
	sector_t	start;
	atomic64_t	reads;
	atomic64_t	writes;
	atomic64_t	read_sectors;
	atomic64_t	write_sectors;
	atomic64_t	flushes;
	atomic64_t	discards;
	unsigned int	delay_us;
};

/* Per-bio private data -- preallocated by dm (T.5's pattern) */
struct count_io {
	unsigned long start_jiffies;
};

static int count_ctr(struct dm_target *ti, unsigned int argc, char **argv)
{
	struct count_c *cc;
	unsigned long long tmp;
	char dummy;
	int ret;

	if (argc < 2 || argc > 3) {
		ti->error = "Usage: <dev> <offset> [delay_us]";
		return -EINVAL;
	}

	cc = kzalloc(sizeof(*cc), GFP_KERNEL);
	if (!cc) {
		ti->error = "Cannot allocate context";
		return -ENOMEM;
	}

	if (sscanf(argv[1], "%llu%c", &tmp, &dummy) != 1 || tmp != (sector_t)tmp) {
		ti->error = "Invalid device sector";
		ret = -EINVAL;
		goto bad;
	}
	cc->start = tmp;

	if (argc == 3) {
		unsigned int d;

		if (kstrtouint(argv[2], 10, &d) || d > 1000000) {
			ti->error = "Invalid delay";
			ret = -EINVAL;
			goto bad;
		}
		cc->delay_us = d;
	}

	ret = dm_get_device(ti, argv[0], dm_table_get_mode(ti->table), &cc->dev);
	if (ret) {
		ti->error = "Device lookup failed";
		goto bad;
	}

	ti->num_flush_bios		= 1;
	ti->num_discard_bios		= 1;
	ti->num_write_zeroes_bios	= 1;
	ti->per_io_data_size		= sizeof(struct count_io);
	ti->private			= cc;
	return 0;

bad:
	kfree(cc);
	return ret;
}

static void count_dtr(struct dm_target *ti)
{
	struct count_c *cc = ti->private;

	dm_put_device(ti, cc->dev);
	kfree(cc);
}

static int count_map(struct dm_target *ti, struct bio *bio)
{
	struct count_c *cc = ti->private;
	struct count_io *ci = dm_per_bio_data(bio, sizeof(struct count_io));

	ci->start_jiffies = jiffies;

	switch (bio_op(bio)) {
	case REQ_OP_READ:
		atomic64_inc(&cc->reads);
		atomic64_add(bio_sectors(bio), &cc->read_sectors);
		break;
	case REQ_OP_WRITE:
		if (bio->bi_opf & REQ_PREFLUSH)
			atomic64_inc(&cc->flushes);
		atomic64_inc(&cc->writes);
		atomic64_add(bio_sectors(bio), &cc->write_sectors);
		break;
	case REQ_OP_FLUSH:
		atomic64_inc(&cc->flushes);
		break;
	case REQ_OP_DISCARD:
	case REQ_OP_WRITE_ZEROES:
		atomic64_inc(&cc->discards);
		break;
	default:
		break;
	}

	/* We are in a context that may not sleep, so a busy delay only. */
	if (cc->delay_us)
		udelay(min(cc->delay_us, 1000U));

	bio_set_dev(bio, cc->dev->bdev);
	if (bio_sectors(bio) || op_is_zone_mgmt(bio_op(bio)))
		bio->bi_iter.bi_sector =
			cc->start + dm_target_offset(ti, bio->bi_iter.bi_sector);

	return DM_MAPIO_REMAPPED;
}

static void count_status(struct dm_target *ti, status_type_t type,
			 unsigned int status_flags, char *result,
			 unsigned int maxlen)
{
	struct count_c *cc = ti->private;
	unsigned int sz = 0;

	switch (type) {
	case STATUSTYPE_INFO:
		DMEMIT("%lld %lld %lld %lld %lld %lld",
		       atomic64_read(&cc->reads),
		       atomic64_read(&cc->writes),
		       atomic64_read(&cc->read_sectors),
		       atomic64_read(&cc->write_sectors),
		       atomic64_read(&cc->flushes),
		       atomic64_read(&cc->discards));
		break;
	case STATUSTYPE_TABLE:
		DMEMIT("%s %llu", cc->dev->name,
		       (unsigned long long)cc->start);
		if (cc->delay_us)
			DMEMIT(" %u", cc->delay_us);
		break;
	default:
		break;
	}
}

static int count_message(struct dm_target *ti, unsigned int argc, char **argv,
			 char *result, unsigned int maxlen)
{
	struct count_c *cc = ti->private;
	unsigned int d;

	if (argc == 1 && !strcasecmp(argv[0], "reset")) {
		atomic64_set(&cc->reads, 0);
		atomic64_set(&cc->writes, 0);
		atomic64_set(&cc->read_sectors, 0);
		atomic64_set(&cc->write_sectors, 0);
		atomic64_set(&cc->flushes, 0);
		atomic64_set(&cc->discards, 0);
		return 0;
	}
	if (argc == 2 && !strcasecmp(argv[0], "delay")) {
		if (kstrtouint(argv[1], 10, &d) || d > 1000000)
			return -EINVAL;
		WRITE_ONCE(cc->delay_us, d);
		return 0;
	}
	DMWARN("unrecognised message: %s", argv[0]);
	return -EINVAL;
}

static int count_iterate_devices(struct dm_target *ti,
				 iterate_devices_callout_fn fn, void *data)
{
	struct count_c *cc = ti->private;

	/* T.2: this is how limits stacking and device holding work. */
	return fn(ti, cc->dev, cc->start, ti->len, data);
}

static int count_prepare_ioctl(struct dm_target *ti, struct block_device **bdev)
{
	struct count_c *cc = ti->private;
	struct dm_dev *dev = cc->dev;

	*bdev = dev->bdev;

	/* Only pass the ioctl down if we map the whole device. */
	if (cc->start || ti->len != bdev_nr_sectors(dev->bdev))
		return 1;
	return 0;
}

static struct target_type count_target = {
	.name		= "count",
	.version	= {1, 0, 0},
	.features	= DM_TARGET_PASSES_INTEGRITY | DM_TARGET_NOWAIT,
	.module		= THIS_MODULE,
	.ctr		= count_ctr,
	.dtr		= count_dtr,
	.map		= count_map,
	.status		= count_status,
	.message	= count_message,
	.prepare_ioctl	= count_prepare_ioctl,
	.iterate_devices = count_iterate_devices,
};

static int __init dm_count_init(void)
{
	int r = dm_register_target(&count_target);

	if (r < 0)
		DMERR("register failed %d", r);
	return r;
}

static void __exit dm_count_exit(void)
{
	dm_unregister_target(&count_target);
}

module_init(dm_count_init);
module_exit(dm_count_exit);
MODULE_DESCRIPTION(DM_NAME " I/O counting target");
MODULE_LICENSE("GPL");
```

```sh
cat > Makefile <<'EOF'
obj-m += dm-count.o
all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
EOF
make && sudo insmod dm-count.ko

sudo dmsetup targets | grep count
sudo dmsetup create cnt --table "0 $SZ count ${DEVS[0]} 0"
sudo dmsetup status cnt
```

```sh
sudo dd if=/dev/zero of=/dev/mapper/cnt bs=4k count=1000 oflag=direct 2>/dev/null
sudo dmsetup status cnt
# reads writes read_sectors write_sectors flushes discards

sudo mkfs.ext4 -qF /dev/mapper/cnt
sudo dmsetup message cnt 0 reset
sudo mount /dev/mapper/cnt /mnt/live
sudo sh -c 'for i in $(seq 1 100); do echo x > /mnt/live/f$i; done; sync'
sudo dmsetup status cnt              # see the filesystem's I/O profile
sudo umount /mnt/live
```

Runtime messages:

```sh
sudo dmsetup message cnt 0 delay 100
sudo dmsetup table cnt
sudo dd if=/dev/mapper/cnt of=/dev/null bs=4k count=100 iflag=direct 2>&1 | tail -1
sudo dmsetup message cnt 0 delay 0
```

Stack it:

```sh
sudo dmsetup create top --table "0 $SZ count /dev/mapper/cnt 0"
sudo dmsetup ls --tree
sudo dd if=/dev/zero of=/dev/mapper/top bs=4k count=100 oflag=direct 2>/dev/null
echo "top:"; sudo dmsetup status top
echo "cnt:"; sudo dmsetup status cnt      # identical counts: pass-through
```

Now measure §T.10's stacking cost:

```sh
for depth in 0 1 2 4 8; do
  sudo dmsetup remove_all 2>/dev/null
  PREV=${DEVS[0]}
  for i in $(seq 1 $depth); do
    sudo dmsetup create L$i --table "0 $SZ count $PREV 0"
    PREV=/dev/mapper/L$i
  done
  [ $depth -eq 0 ] && DEV=${DEVS[0]} || DEV=$PREV
  echo -n "depth $depth: "
  sudo fio --name=t --filename=$DEV --direct=1 --rw=randread --bs=4k \
           --iodepth=32 --ioengine=libaio --runtime=5 --time_based \
           --numjobs=4 --group_reporting 2>/dev/null | grep -oP 'IOPS=\K[^,]+'
done
```

Cleanup:

```sh
sudo dmsetup remove_all
sudo rmmod dm_count
```

---

## 3. Mastery drills

1. State device mapper's core abstraction in one sentence, then show that concatenation, striping, encryption, and thin provisioning are all instances of it.

2. `DM_MAPIO_REMAPPED` versus `DM_MAPIO_SUBMITTED`: classify each of dm-linear, dm-crypt, dm-thin, dm-cache, dm-delay, and dm-flakey, and justify each.

3. Explain what `iterate_devices` is used for, then construct the corruption that results from implementing it incorrectly.

4. Trace `dmsetup suspend` / `reload` / `resume` and prove that the table swap is atomic with respect to I/O. Then construct the case where `--noflush` is required.

5. Explain why encrypting a block device requires a per-sector, position-dependent IV. Then show the attack enabled by using a constant IV, and the one enabled by `plain` versus `essiv`.

6. dm-crypt provides confidentiality but not integrity. Construct a concrete bit-flipping attack against an `aes-xts` volume, and state what `--integrity` changes.

7. Compute the metadata required for a 10 TiB thin pool at block sizes of 64 KiB, 256 KiB, and 1 MiB. Then state the trade-off each makes.

8. `queue_if_no_space` versus `error_if_no_space`: construct the scenario where each is the correct choice, and explain why the default is what it is.

9. dm-cache and dm-writecache make different bets. State each bet precisely, and construct the workload where each wins decisively over the other.

10. LVM is entirely userspace. Enumerate everything it does, and for each, state what the kernel provides and what LVM decides.

11. Trace exactly what `pvmove` does at the dm level, step by step, and prove that it is safe against a crash at any point.

12. Compute the per-I/O cost of a six-layer dm stack. Then identify what information is lost at each layer and what the consequences are.

13. You are given a stack: ext4 → dm-crypt → dm-thin → dm-raid5 → four NVMe devices, showing 10 % of the raw device throughput. Give the ordered diagnostic procedure and the five most likely causes.

---

## 4. Further reading

**Kernel documentation**

- `Documentation/admin-guide/device-mapper/` ★★★ — **one document per target, all worth reading.** Particularly: `dm-crypt.rst`, `thin-provisioning.rst` ★★★, `cache.rst`, `cache-policies.rst`, `dm-flakey.rst`, `dm-integrity.rst`, `verity.rst`, `dm-raid.rst`, `writecache.rst`, `dm-log-writes.rst`, `dm-dust.rst`, `statistics.rst`.
- `Documentation/admin-guide/device-mapper/thin-provisioning.rst` ★★★ — **read this before deploying dm-thin.** It documents §T.5's failure modes explicitly.
- `Documentation/admin-guide/device-mapper/persistent-data.rst` — the B-tree used by thin and cache.
- `Documentation/admin-guide/device-mapper/dm-init.rst` — creating dm devices from the kernel command line.
- `man 8 dmsetup` ★★★, `man 8 cryptsetup` ★★★, `man 8 veritysetup`, `man 8 integritysetup`
- `man 8 lvm`, `man 8 lvcreate`, `man 8 lvconvert`, `man 8 pvmove`, `man 5 lvm.conf`
- `man 8 thin_check`, `thin_dump`, `thin_repair`, `cache_check` — the offline tools

**Papers and primary sources**

- Thornber, the dm-thin and dm-cache design discussions on dm-devel ★★★ — the reasoning behind §T.5 and §T.6.
- Broz, "dm-crypt: Linux kernel device-mapper crypto target" and the cryptsetup FAQ ★★★ — **the FAQ is unusually good**; it covers IV modes, integrity, performance, and the common mistakes.
- Rogaway, "Efficient Instantiations of Tweakable Blockciphers and Refinements to Modes OCB and PMAC" — the XEX construction underlying XTS.
- IEEE 1619-2007 — the XTS-AES standard.
- Hitz, Lau, Malcolm, "File System Design for an NFS File Server Appliance," USENIX 1994 — WAFL's snapshots; the ancestry of dm-thin's approach.
- The `dm-devel` mailing list archives — where all of this is designed.

**LWN**

- "Device mapper" introductory coverage and the target-by-target articles
- "Thin provisioning in the kernel" ★★★ and the follow-ups on the ENOSPC problem
- "dm-cache and bcache" comparisons ★★★
- "A block-layer write cache" (dm-writecache) ★★★
- "dm-crypt performance" and the workqueue-removal discussions ★★★ — §T.4's optimisation story
- "Authenticated encryption for block devices" (dm-integrity)
- "dm-verity and Android verified boot"
- "Testing filesystems with dm-log-writes" ★★★
- "The LVM2 and device-mapper relationship" discussions

**Source reading order**

1. `Documentation/admin-guide/device-mapper/` for the targets you care about.
2. `drivers/md/dm-linear.c` ★★★ — **~200 lines; read all of it.** It is the complete shape of a target.
3. `drivers/md/dm-stripe.c` — the next step up; shows multi-device mapping.
4. `drivers/md/dm-table.c`: `dm_table_add_target`, `dm_calculate_queue_limits`, `dm_table_find_target` ★★★
5. `drivers/md/dm.c`: `dm_submit_bio`, `__split_and_process_bio`, `__map_bio`, `clone_endio` ★★★
6. `drivers/md/dm-delay.c` and `dm-flakey.c` — small targets that do more than remap.
7. `drivers/md/dm-crypt.c`: `crypt_ctr`, `crypt_map`, `kcryptd_crypt`, the IV generators ★★★
8. `drivers/md/dm-thin.c` and `persistent-data/dm-btree.c` — last; the most complex.

**Tools**

- `dmsetup ls --tree` ★★★ — **the first command to run on any dm problem.**
- `dmsetup table`, `status`, `info -c`, `deps`, `targets`, `message` ★★★
- `dmstats` ★★★ — per-region statistics; underused
- `cryptsetup benchmark`, `luksDump`, `status` ★★★
- `thin_check`, `thin_dump`, `thin_delta`, `cache_check` — offline metadata tools
- `lvs -a -o+devices,segtype,stripes`, `lvdisplay -m`, `pvs -o+pv_used` ★★★
- `dmeventd` and `lvm.conf`'s monitoring settings — **mandatory for thin pools**
- `dm-log-writes` + `replay-log` ★★★ (Ch. 61 §T.10)
- `dm-delay`, `dm-flakey`, `dm-dust`, `dm-error` ★★★ — build every test harness from these
- `blktrace` with `block_bio_remap` events — watch I/O descend the stack
- `veritysetup`, `integritysetup`

---

→ Next: [67-md-raid.md](67-md-raid.md)

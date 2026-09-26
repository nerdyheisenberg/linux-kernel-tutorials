# Chapter 65 — Writing a block driver (`null_blk`, ramdisk, blk-mq driver)

> **Goal:** Build block drivers of increasing seriousness — a bio-based ramdisk, a full blk-mq driver with tags and timeouts, and a network-backed driver that must handle asynchronous completion and errors — and in doing so discover why every field of `blk_mq_ops` exists. You will register a `gendisk`, declare queue limits, implement `queue_rq`, handle completion from an interrupt-equivalent context, implement timeout and requeue, add `ioctl`s and partitions, and then deliberately break each one to see the failure mode. By the end, reading any real block driver is a matter of recognising choices you have already had to make.

---

## Theory & First Principles

### T.0 — Start here: a whole block device in 60 lines

```c
static blk_status_t my_queue_rq(struct blk_mq_hw_ctx *hctx,
                                const struct blk_mq_queue_data *bd)
{
	struct request *rq = bd->rq;
	struct my_dev  *dev = hctx->queue->queuedata;
	struct bio_vec  bvec;
	struct req_iterator iter;
	loff_t pos = blk_rq_pos(rq) << SECTOR_SHIFT;   /* sectors -> bytes */

	blk_mq_start_request(rq);                       /* ① start the timeout clock */

	rq_for_each_segment(bvec, rq, iter) {           /* ② walk the scatter list */
		void *buf = page_address(bvec.bv_page) + bvec.bv_offset;

		if (pos + bvec.bv_len > dev->size)      /* ③ VALIDATE. Always. */
			return BLK_STS_IOERR;

		if (req_op(rq) == REQ_OP_READ)
			memcpy(buf, dev->data + pos, bvec.bv_len);
		else
			memcpy(dev->data + pos, buf, bvec.bv_len);
		pos += bvec.bv_len;
	}

	blk_mq_end_request(rq, BLK_STS_OK);             /* ④ EXACTLY once */
	return BLK_STS_OK;
}
```

Register that with a `blk_mq_tag_set` and an `add_disk()`, and you have a device that `mkfs`,
`mount`, `fdisk`, `dd`, and `fio` will all use without knowing anything about it. **That is
Ch. 53's hourglass again, one layer down.**

**Four things in that function carry the entire weight of the interface:**

① **`blk_mq_start_request` starts a watchdog.** The block layer will call your `timeout`
handler if the request is not completed in time. Forget this and the timeout machinery never
arms; a lost request hangs the filesystem forever, in uninterruptible `D` state.

② **You get a scatter list, not a buffer.** A 1 MiB request is dozens of physically
discontiguous pages (Ch. 63 §T.0). A real driver hands this list to `dma_map_sg()` and the
device's DMA engine (Ch. 35) instead of `memcpy`-ing it.

③ **`blk_rq_pos()` is attacker-influenced input.** Sector numbers come from a filesystem that
may be mounted from a USB stick or a container. **This is a trust boundary** — Ch. 24 §T.0's
rule applies unchanged: validate at the boundary, every time.

④ **Completion must happen exactly once, on every path.** Complete twice and you free a
`request` that is still in the tag map — a use-after-free with attacker-controlled timing.
Never complete and the filesystem hangs. **There is no benign failure mode for getting this
wrong**, which is why it is the single hardest invariant in driver writing (Ch. 12 §T.0 is
the same lesson about refcounts).

**Now the thing that makes a block driver harder than a character driver.** A char driver can
sleep in `read()`; the caller is a process with a context. A block driver cannot:

```
   Who calls queue_rq?  Anyone. Including:
     - a kswapd reclaim path trying to free memory
     - a writeback worker
     - a process that is ALREADY holding a filesystem lock

   So: your driver must not allocate memory with GFP_KERNEL on the I/O path.
       Doing so can recurse into reclaim, which needs to write out a page,
       which calls YOUR DRIVER. Deadlock.  (Ch. 11 §T.0, GFP_NOIO.)

   Rule: preallocate everything the I/O path needs, at probe time.
         Use mempools for what you cannot preallocate.
```

**This is the general structure of any "I am in the memory-reclaim path" subsystem** — swap,
network block devices, and the loop device all have it, and it is why `GFP_NOIO`,
`memalloc_noio_save()`, and mempools exist at all.

**And the request lifecycle is worth memorizing**, because `blktrace` prints exactly these
letters:

```
  Q  queued        bio submitted to the block layer
  G  get request   a tag allocated
  M  merged        folded into an adjacent request (or not)
  I  inserted      placed in the scheduler / software queue
  D  dispatched    handed to YOUR driver  <-- your latency starts here
  C  completed     blk_mq_end_request     <-- and ends here
```

`D`-to-`C` is *your* driver's latency. `Q`-to-`D` is queueing and scheduling. When someone
says "the disk is slow", the first question is which of those two intervals grew.

```bash
sudo blktrace -d /dev/nullb0 -o - | blkparse -i -
sudo /usr/share/bcc/tools/biolatency -D 1
modprobe null_blk queue_mode=2 irqmode=0 nr_devices=1   # a reference driver
ls /sys/kernel/debug/block/nullb0/
```

---

### T.1 What a block driver must actually do

Stripped to essentials, a block driver is:

```
A function that takes "read/write N sectors at offset S into/from these pages"
and eventually calls a completion callback.
```

Everything else is infrastructure to make that safe, fast, and observable. The obligations, in order of how easy they are to get wrong:

| Obligation | Consequence of failure |
|---|---|
| Complete **every** request, exactly once | hang, or double-free |
| Never sleep in `queue_rq` (by default) | deadlock under memory pressure |
| Report limits accurately | corruption, or `EIO` on valid I/O |
| Handle `REQ_OP_FLUSH` if you claim a write cache | silent data loss (Ch. 61 §T.6) |
| Handle errors without leaking requests | resource exhaustion |
| Handle device removal while I/O is in flight | use-after-free |
| Handle timeouts | permanent hang |
| Be correct under concurrent submission from every CPU | corruption |

The first is the one that matters most and is the one beginners break: **a request that is never completed hangs the submitting process forever, in uninterruptible sleep.** There is no timeout above you; you are the timeout.

### T.2 The two driver models, chosen deliberately

Chapter 63 §T.8 introduced the distinction. Choosing between them is the first design decision:

| | **bio-based** | **request-based (blk-mq)** |
|---|---|---|
| Entry | `disk->fops->submit_bio(bio)` | `ops->queue_rq(hctx, bd)` |
| You receive | raw, unmerged bios | merged, scheduled requests |
| Merging | none | yes |
| I/O scheduler | bypassed | available |
| Tags | none | yes |
| Timeout handling | you build it | provided |
| Best for | **stacking** (dm, md), trivial memory-backed devices | **real hardware**, anything with a queue |

The decision rule:

- If you **pass I/O down to another block device**, be bio-based. Merging and scheduling will happen at the bottom; doing it twice is waste, and two schedulers reordering independently is worse than one.
- If you **talk to hardware or a remote endpoint with a finite command queue**, be request-based. You want tags (they map to command slots), merging (fewer commands), and the timeout machinery.
- If you are a **memory-backed device with no queue**, either works; `brd` is bio-based, `null_blk` supports both so you can measure the difference.

A subtlety: a bio-based driver still gets `queue_limits` and still gets bios split to them (Ch. 63 §T.9). What it does not get is merging, scheduling, tags, or timeouts.

### T.3 `gendisk`: the object userspace sees

```c
struct gendisk {
	int			major;
	int			first_minor;
	int			minors;
	char			disk_name[DISK_NAME_LEN];
	unsigned short		events;
	unsigned short		event_flags;
	struct xarray		part_tbl;
	struct block_device	*part0;
	const struct block_device_operations *fops;
	struct request_queue	*queue;
	void			*private_data;
	struct bio_set		bio_split;
	int			flags;
	unsigned long		state;
	struct mutex		open_mutex;
	unsigned		open_partitions;
	struct kobject		queue_kobj;
	...
};
```

Creating one is a three-step protocol, and the order matters:

```c
	/* 1. Allocate the disk AND its queue, with limits declared up front. */
	disk = blk_mq_alloc_disk(&tag_set, &lim, driver_data);

	/* 2. Fill in identity. The disk is NOT yet visible. */
	disk->major = major;
	disk->first_minor = minor;
	disk->minors = 16;
	disk->fops = &my_fops;
	disk->private_data = dev;
	snprintf(disk->disk_name, DISK_NAME_LEN, "mydev%d", idx);
	set_capacity(disk, sectors);

	/* 3. NOW publish it. From this instant, I/O can arrive. */
	ret = add_disk(disk);
```

**`add_disk()` is the publication point** (Ch. 25's P1). Everything the driver needs to serve I/O must be ready *before* it. A driver that calls `add_disk()` and then finishes initialising has a race that appears as a boot-time crash on fast machines, because udev opens the device immediately to probe it.

Teardown is the reverse, and it must be exactly the reverse:

```c
	del_gendisk(disk);        /* stop new I/O; wait for open()s to close */
	blk_mq_free_tag_set(&tag_set);
	put_disk(disk);           /* drop the last reference */
```

`del_gendisk()` waits for in-flight I/O and for the last `close()`. A module that unloads without it will be unloaded while a filesystem is mounted on its device — an immediate oops.

**Passing limits to `blk_mq_alloc_disk`** is the modern API (6.9+). Previously limits were set after allocation with `blk_queue_*` calls, which created a window where a queue existed with default limits. The current API eliminates it; older drivers still use the old form and you will see both.

### T.4 `blk_mq_ops`: what each method is for

```c
struct blk_mq_ops {
	/* THE function. Everything else is support. */
	blk_status_t (*queue_rq)(struct blk_mq_hw_ctx *,
				 const struct blk_mq_queue_data *);

	/* Batching hooks: the caller will submit several, then commit. */
	void (*commit_rqs)(struct blk_mq_hw_ctx *);
	void (*queue_rqs)(struct rq_list *);

	/* Polled completion (Ch. 63 T.5) */
	int  (*poll)(struct blk_mq_hw_ctx *, struct io_comp_batch *);

	/* Called on the completion CPU after steering */
	void (*complete)(struct request *);

	/* Per-hctx lifecycle */
	int  (*init_hctx)(struct blk_mq_hw_ctx *, void *, unsigned int);
	void (*exit_hctx)(struct blk_mq_hw_ctx *, unsigned int);

	/* Per-request lifecycle: called ONCE at tag-set setup, not per I/O */
	int  (*init_request)(struct blk_mq_tag_set *, struct request *,
			     unsigned int, unsigned int);
	void (*exit_request)(struct blk_mq_tag_set *, struct request *, unsigned int);

	/* THE safety net */
	enum blk_eh_timer_return (*timeout)(struct request *);

	void (*initialize_rq_fn)(struct request *);
	void (*cleanup_rq)(struct request *);
	bool (*busy)(struct request_queue *);
	void (*map_queues)(struct blk_mq_tag_set *);
	void (*show_rq)(struct seq_file *, struct request *);
};
```

The three that carry real design weight:

**`queue_rq` return values** are a small protocol with large consequences:

| Return | Meaning | Block layer's response |
|---|---|---|
| `BLK_STS_OK` | accepted; I will complete it | continue dispatching |
| `BLK_STS_RESOURCE` | out of a **shared** resource | stop this hctx briefly, retry |
| `BLK_STS_DEV_RESOURCE` | **this device** is full | stop until a request completes |
| any error | permanent failure | complete the request with that status |

`RESOURCE` versus `DEV_RESOURCE` (Ch. 63 §T.7) is the distinction drivers get wrong. Returning `RESOURCE` when you mean `DEV_RESOURCE` produces a busy-loop: the block layer retries immediately, you return busy again, forever. Returning `DEV_RESOURCE` when a *shared* resource freed elsewhere would have let you proceed produces a stall, because nothing on this device will complete to restart you. The rule: **`DEV_RESOURCE` is only correct if one of your own in-flight requests completing will unblock you.**

Also critical: **once `queue_rq` returns `BLK_STS_OK`, you own the request and must complete it.** You may not touch it after calling `blk_mq_complete_request()`.

**`init_request` is called once per tag**, at tag-set setup time — not per I/O. This is how per-request driver data is preallocated:

```c
	tag_set->cmd_size = sizeof(struct my_cmd);
	/* Then blk_mq_rq_to_pdu(rq) gives you a struct my_cmd * */
```

`cmd_size` bytes are allocated immediately after each `struct request`, so `blk_mq_rq_to_pdu()` is pointer arithmetic. **No allocation in the I/O path** — which is what makes the path safe under memory pressure (§T.7).

**`timeout` is the safety net.** If a request has been in flight longer than `rq->timeout`, this is called. It returns:

| Return | Meaning |
|---|---|
| `BLK_EH_DONE` | I have handled it (usually: I completed it) |
| `BLK_EH_RESET_TIMER` | it is still making progress; give it more time |

The handler races with completion (Ch. 63 §1.4), and the block layer's `cmpxchg` on `rq->state` resolves it — but **your handler must be prepared to find the request already completed.** The usual shape is: try to claim it, abort the hardware command, complete it with `BLK_STS_TIMEOUT`.

A driver with no `timeout` method gets no timeout at all, and a lost completion hangs forever.

### T.5 Queue limits: your contract with everything above

```c
	struct queue_limits lim = {
		.logical_block_size	= 512,
		.physical_block_size	= 4096,
		.io_min			= 4096,
		.io_opt			= 65536,
		.max_hw_sectors		= 2048,        /* 1 MiB */
		.max_segments		= 128,
		.max_segment_size	= 65536,
		.dma_alignment		= 511,
		.max_discard_sectors	= 2048,
		.discard_granularity	= 4096,
		.max_write_zeroes_sectors = 2048,
		.features		= BLK_FEAT_WRITE_CACHE | BLK_FEAT_FUA,
	};
```

Each of these is a promise, and breaking it has a specific consequence:

| Limit | If you understate | If you overstate |
|---|---|---|
| `max_hw_sectors` | more, smaller requests; slower | **you receive I/O you cannot handle** |
| `max_segments` | more splitting | DMA setup fails or corrupts |
| `logical_block_size` | — | filesystems issue unalignable I/O |
| `physical_block_size` | filesystems do RMW unnecessarily | filesystems assume atomicity you lack |
| `dma_alignment` | bounce buffering | DMA to a misaligned address |

**Declaring `BLK_FEAT_WRITE_CACHE` obliges you to implement `REQ_OP_FLUSH`.** If you claim a write cache and ignore flushes, every filesystem above you loses data on power failure and `fsync` lies (Ch. 61 §T.6). If you have no cache, do not set the flag and the block layer will never send you a flush.

`BLK_FEAT_FUA` similarly: if set, you must honour `REQ_FUA` on writes by making them durable before completion. If unset, the block layer emulates FUA as write-then-flush.

### T.6 The completion path

There are three places completion can happen and they have different rules:

| Context | Can sleep | Used by |
|---|---|---|
| **Interrupt handler** | no | most hardware |
| **Softirq / IPI** | no | `blk_mq_complete_request` steering (Ch. 63 §T.7) |
| **Workqueue / thread** | **yes** | network-backed, or anything needing to sleep |

```c
	/* From an interrupt or a non-sleeping context: */
	blk_mq_complete_request(rq);
	/* -> may IPI to the submitting CPU -> calls ops->complete(rq)
	 *    -> your complete() calls blk_mq_end_request(rq, status) */

	/* If you can complete inline and cheaply: */
	blk_mq_end_request(rq, BLK_STS_OK);
```

The difference matters: `blk_mq_complete_request` respects `rq_affinity` and may defer to the submitting CPU for cache locality. `blk_mq_end_request` completes immediately wherever you are. A ramdisk should use the latter (there is nothing to gain by an IPI); a real device should use the former.

**Batched completion** (`blk_mq_add_to_batch` + `blk_mq_end_request_batch`) lets a driver complete many requests with one pass, amortising the per-completion cost. NVMe uses it heavily, and it is a real win above ~100K IOPS.

A hard rule: **after `blk_mq_end_request` (or `blk_mq_complete_request`), the request is gone.** Reading `rq->anything` afterwards is use-after-free. This is the single most common block-driver bug, and KASAN catches it immediately — which is why §2's labs insist on running under KASAN.

### T.7 Memory allocation in the I/O path

Block drivers sit under swap. A driver that allocates memory to service a write may be asked to write out the very page being allocated for. Hence:

**Rule 1: `queue_rq` is called with interrupts enabled but in atomic context by default** (`BLK_MQ_F_BLOCKING` unset). You may not sleep. So `GFP_KERNEL` is forbidden.

**Rule 2: preallocate everything.** `cmd_size` for per-request data; a `bio_set` with a mempool for any bios you allocate; DMA descriptors allocated at probe time.

**Rule 3: if you genuinely must sleep**, set `BLK_MQ_F_BLOCKING` in the tag set. The block layer then calls `queue_rq` from a context where sleeping is allowed — at a cost, because it can no longer dispatch from softirq context. `nbd`, `loop`, and `rbd` do this.

**Rule 4: use `GFP_NOIO` (or `memalloc_noio_save()`) anywhere you must allocate.** This prevents reclaim from recursing into I/O, which is exactly the deadlock (Ch. 22). The pattern:

```c
	unsigned int noio_flag = memalloc_noio_save();
	/* ... allocations here cannot recurse into I/O ... */
	memalloc_noio_restore(noio_flag);
```

This is the same reserve-and-avoid-recursion discipline as XFS's AGFL (Ch. 58 §1.2), FUSE's `PR_SET_IO_FLUSHER` (Ch. 62 §T.7), and bio_set mempools (Ch. 63 §T.8). **The pattern recurs because the problem recurs: any component on the reclaim path must be able to make progress without allocating.**

### T.8 Hot-unplug and reference counting

A block device can be removed while a filesystem is mounted on it. The driver must survive this.

```c
static void my_remove(struct device *dev)
{
	struct my_device *d = dev_get_drvdata(dev);

	/* 1. Stop accepting new I/O and mark the disk dead. */
	del_gendisk(d->disk);

	/* 2. Any in-flight requests are completed with an error by the
	 *    block layer during del_gendisk -> blk_mark_disk_dead. */

	/* 3. Now tear down the queue. */
	blk_mq_free_tag_set(&d->tag_set);

	/* 4. Drop the disk reference. Userspace may still hold opens;
	 *    put_disk just drops OUR reference. */
	put_disk(d->disk);

	/* 5. Free driver state. Safe: nothing can reach us now. */
	kfree(d);
}
```

`blk_mark_disk_dead()` is the mechanism for surprise removal: it sets `GD_DEAD`, fails all future submissions with `BLK_STS_IOERR`, and wakes anything waiting. A driver for a hot-pluggable device (USB, NVMe) should call it from its removal path *before* the slow teardown, so waiters are released immediately rather than after a 30-second timeout.

The reference-counting invariant: **`put_disk` does not free the disk if userspace still has it open.** The `gendisk` outlives the driver's knowledge of it, which is why `disk->private_data` must remain valid or be checked. Drivers that free their private data in `remove()` and then get an `ioctl` from a still-open fd crash. The usual solution is a per-disk refcount plus a `dead` flag checked in every entry point.

### T.9 Testing a block driver

A block driver is infrastructure; "it seems to work" is not a standard. The minimum:

| Test | Catches |
|---|---|
| `mkfs` + `mount` + `fsstress` + `umount` | basic correctness |
| `fio --verify=crc32c` | data corruption |
| `blktests` ★★★ | the actual conformance suite |
| KASAN + lockdep + `DEBUG_ATOMIC_SLEEP` | memory bugs, locking bugs, sleeping in atomic |
| Hot-unplug during I/O | §T.8 |
| Fault injection (`should_fail_bio`) | error paths |
| Timeout injection | §T.4's handler |
| `dm-flakey`/`dm-log-writes` above it | crash consistency (Ch. 61) |
| Concurrent I/O from every CPU | scalability bugs |
| `xfstests` with a filesystem on it | everything at once |

`blktests` is the one to know: it is to block drivers what `xfstests` is to filesystems, and it has specific groups for `loop`, `nbd`, `null_blk`, `scsi`, `nvme`, and `zbd`. Running `./check block` against your driver finds real bugs.

The debug configuration to build with:

```
CONFIG_KASAN=y
CONFIG_KASAN_VMALLOC=y
CONFIG_UBSAN=y
CONFIG_PROVE_LOCKING=y
CONFIG_DEBUG_ATOMIC_SLEEP=y
CONFIG_DEBUG_KOBJECT_RELEASE=y
CONFIG_FAIL_MAKE_REQUEST=y
CONFIG_FAIL_IO_TIMEOUT=y
CONFIG_BLK_DEV_IO_TRACE=y
CONFIG_DEBUG_BLOCK_EXT_DEVT=y
```

`CONFIG_DEBUG_ATOMIC_SLEEP` in particular catches §T.7's rule-1 violations immediately, and they are otherwise invisible until a production deadlock.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `drivers/block/brd.c` ★★★ | ~400 lines; the minimal bio-based driver |
| `drivers/block/null_blk/` ★★★ | **the reference blk-mq driver**; configurable everything |
| `drivers/block/loop.c` ★★★ | `BLK_MQ_F_BLOCKING`, workqueue completion, a realistic mid-size driver |
| `drivers/block/nbd.c` ★★★ | network-backed; timeouts, reconnection, the hard cases |
| `drivers/block/virtio_blk.c` ★★★ | a real, simple, production driver — the best model |
| `drivers/block/zram/` | compressed ramdisk; more complex bio-based |
| `drivers/nvme/host/pci.c` | a high-performance driver; batching, polling, multiple queue maps |
| `drivers/block/rnbd/` | RDMA network block device |
| `block/blk-mq.c` ★★★ | the machinery your driver plugs into |
| `include/linux/blk-mq.h` ★★★ | `blk_mq_ops`, `blk_mq_tag_set`, the helpers |
| `include/linux/blkdev.h` ★★★ | `gendisk`, `block_device_operations`, `queue_limits` |
| `Documentation/block/` | as in Ch. 63 |

Read `brd.c` and `virtio_blk.c` in that order; between them they cover ninety percent of what a driver needs.

### 1.2 `block_device_operations`

```c
struct block_device_operations {
	void (*submit_bio)(struct bio *bio);          /* bio-based ONLY */
	int  (*poll_bio)(struct bio *, struct io_comp_batch *, unsigned int);
	int  (*open)(struct gendisk *disk, blk_mode_t mode);
	void (*release)(struct gendisk *disk);
	int  (*ioctl)(struct block_device *, blk_mode_t, unsigned, unsigned long);
	int  (*compat_ioctl)(struct block_device *, blk_mode_t, unsigned, unsigned long);
	unsigned int (*check_events)(struct gendisk *disk, unsigned int clearing);
	void (*unlock_native_capacity)(struct gendisk *);
	int  (*getgeo)(struct block_device *, struct hd_geometry *);
	int  (*set_read_only)(struct block_device *bdev, bool ro);
	void (*free_disk)(struct gendisk *disk);
	void (*swap_slot_free_notify)(struct block_device *, unsigned long);
	int  (*report_zones)(struct gendisk *, sector_t sector,
			     unsigned int nr_zones, report_zones_cb cb, void *data);
	char *(*devnode)(struct gendisk *disk, umode_t *mode);
	int  (*get_unique_id)(struct gendisk *disk, u8 id[16],
			      enum blk_unique_id id_type);
	struct module *owner;
	const struct pr_ops *pr_ops;
	int (*alternative_gpt_sector)(struct gendisk *disk, sector_t *sector);
};
```

`submit_bio` present means bio-based; absent means the driver uses the request queue. Both cannot be set.

`getgeo` exists for one reason: DOS-era partitioning tools want cylinders/heads/sectors. It is fiction, and every driver invents plausible numbers:

```c
static int my_getgeo(struct block_device *bdev, struct hd_geometry *geo)
{
	geo->heads = 4;
	geo->sectors = 16;
	geo->cylinders = get_capacity(bdev->bd_disk) / (4 * 16);
	geo->start = 0;
	return 0;
}
```

`free_disk` is the correct hook for freeing driver state tied to the disk's lifetime — it is called when the *last* reference drops, which may be long after `remove()`. Using it avoids §T.8's use-after-free entirely.

### 1.3 `brd`: the minimal bio-based driver

```c
static void brd_submit_bio(struct bio *bio)
{
	struct brd_device *brd = bio->bi_bdev->bd_disk->private_data;
	sector_t sector = bio->bi_iter.bi_sector;
	struct bio_vec bvec;
	struct bvec_iter iter;

	bio_for_each_segment(bvec, bio, iter) {
		unsigned int len = bvec.bv_len;
		int err;

		/* Never cross a page boundary in one operation */
		WARN_ON_ONCE((bvec.bv_offset & (SECTOR_SIZE - 1)) ||
			     (len & (SECTOR_SIZE - 1)));

		err = brd_do_bvec(brd, bvec.bv_page, len, bvec.bv_offset,
				  bio->bi_opf, sector);
		if (err) {
			if (err == -ENOMEM && bio->bi_opf & REQ_NOWAIT) {
				bio_wouldblock_error(bio);
				return;
			}
			bio_io_error(bio);
			return;
		}
		sector += len >> SECTOR_SHIFT;
	}

	bio_endio(bio);
}
```

That is the whole I/O path. Note:

- `bio_for_each_segment` with an explicit iterator (Ch. 63 §T.2) — the bio is not mutated.
- `REQ_NOWAIT` is honoured: `bio_wouldblock_error` returns `-EAGAIN` to the submitter rather than blocking.
- `bio_endio(bio)` exactly once, on every path.

### 1.4 A complete blk-mq driver skeleton

```c
// SPDX-License-Identifier: GPL-2.0
struct my_cmd {              /* per-request private data */
	struct request *rq;
	dma_addr_t dma;
	int hw_slot;
};

struct my_device {
	struct blk_mq_tag_set tag_set;
	struct gendisk *disk;
	spinlock_t lock;
	void __iomem *regs;
	sector_t capacity;
	bool dead;
};

static blk_status_t my_queue_rq(struct blk_mq_hw_ctx *hctx,
				const struct blk_mq_queue_data *bd)
{
	struct request *rq = bd->rq;
	struct my_device *dev = hctx->queue->queuedata;
	struct my_cmd *cmd = blk_mq_rq_to_pdu(rq);
	blk_status_t ret;

	if (unlikely(dev->dead))
		return BLK_STS_IOERR;

	/* MUST be called before the hardware can complete it. */
	blk_mq_start_request(rq);

	cmd->rq = rq;

	switch (req_op(rq)) {
	case REQ_OP_READ:
	case REQ_OP_WRITE:
		ret = my_submit_rw(dev, rq, cmd);
		break;
	case REQ_OP_FLUSH:
		ret = my_submit_flush(dev, rq, cmd);
		break;
	case REQ_OP_DISCARD:
		ret = my_submit_discard(dev, rq, cmd);
		break;
	default:
		return BLK_STS_NOTSUPP;
	}

	if (ret == BLK_STS_DEV_RESOURCE) {
		/* We could not submit. Undo the start. */
		blk_mq_stop_hw_queue(hctx);
		return ret;
	}

	/* `bd->last` tells us this is the last of a batch: ring the
	 * doorbell now rather than per request. */
	if (bd->last)
		my_ring_doorbell(dev);

	return BLK_STS_OK;
}

/* Called from the interrupt handler */
static void my_irq_complete(struct my_device *dev, int slot, int status)
{
	struct request *rq = dev->slots[slot];
	struct my_cmd *cmd;

	if (!rq)
		return;
	cmd = blk_mq_rq_to_pdu(rq);
	dev->slots[slot] = NULL;

	/* Steering happens here (Ch. 63 T.7) */
	blk_mq_complete_request(rq);
}

/* Called on the completion CPU */
static void my_complete_rq(struct request *rq)
{
	struct my_cmd *cmd = blk_mq_rq_to_pdu(rq);

	/* Unmap DMA, translate status */
	my_unmap(cmd);
	blk_mq_end_request(rq, cmd->status);
	/* rq is GONE after this line. */
}

static enum blk_eh_timer_return my_timeout(struct request *rq)
{
	struct my_cmd *cmd = blk_mq_rq_to_pdu(rq);
	struct my_device *dev = rq->q->queuedata;

	dev_warn(dev->dev, "request %d timed out\n", rq->tag);

	/* Is it actually still outstanding? The hardware may have
	 * completed it between the timer firing and now. */
	if (my_hw_completed(dev, cmd->hw_slot)) {
		my_irq_complete(dev, cmd->hw_slot, 0);
		return BLK_EH_DONE;
	}

	/* Abort it in hardware, then complete it with an error. */
	my_abort(dev, cmd->hw_slot);
	blk_mq_complete_request(rq);
	return BLK_EH_DONE;
}

static int my_init_request(struct blk_mq_tag_set *set, struct request *rq,
			   unsigned int hctx_idx, unsigned int numa_node)
{
	struct my_cmd *cmd = blk_mq_rq_to_pdu(rq);
	struct my_device *dev = set->driver_data;

	/* Called ONCE per tag, at setup. Preallocate here (T.7). */
	cmd->hw_slot = -1;
	cmd->dma = 0;
	return 0;
}

static const struct blk_mq_ops my_mq_ops = {
	.queue_rq	= my_queue_rq,
	.complete	= my_complete_rq,
	.timeout	= my_timeout,
	.init_request	= my_init_request,
	.exit_request	= my_exit_request,
	.map_queues	= my_map_queues,
};
```

Every line of `my_queue_rq` corresponds to an obligation from §T.1 and §T.4.

`bd->last` deserves a note: blk-mq tells the driver when it is submitting the last request of a dispatch batch, so the driver can ring the doorbell (an expensive MMIO write, Ch. 34) once instead of per request. NVMe's throughput depends heavily on this.

### 1.5 Tag-set setup

```c
static int my_setup_tagset(struct my_device *dev)
{
	struct blk_mq_tag_set *set = &dev->tag_set;

	memset(set, 0, sizeof(*set));
	set->ops		= &my_mq_ops;
	set->nr_hw_queues	= num_possible_cpus();   /* one per CPU */
	set->nr_maps		= 1;                     /* or 3 for read/poll */
	set->queue_depth	= MY_QUEUE_DEPTH;
	set->numa_node		= NUMA_NO_NODE;
	set->cmd_size		= sizeof(struct my_cmd); /* T.4: preallocated */
	set->flags		= BLK_MQ_F_SHOULD_MERGE;
	set->driver_data	= dev;

	return blk_mq_alloc_tag_set(set);
}
```

Flags worth knowing:

| Flag | Meaning |
|---|---|
| `BLK_MQ_F_SHOULD_MERGE` | allow merging (almost always yes) |
| `BLK_MQ_F_TAG_QUEUE_SHARED` | this tag set is shared across queues |
| `BLK_MQ_F_BLOCKING` | **`queue_rq` may sleep** (§T.7 rule 3) |
| `BLK_MQ_F_NO_SCHED` | do not attach an I/O scheduler |
| `BLK_MQ_F_STACKING` | this is a stacking driver; avoid recursion accounting |

`nr_hw_queues` should match the device's real parallelism. Setting it to `num_possible_cpus()` for a device with one hardware queue wastes memory and gains nothing; setting it to 1 for an NVMe device with 64 queues throws away Ch. 63 §T.5's entire benefit.

### 1.6 The `null_blk` configuration surface

`null_blk` is the reference implementation and a laboratory:

```sh
/sys/kernel/config/nullb/NAME/
	size            device size in MiB
	blocksize       logical block size
	queue_mode      0=bio, 1=rq(legacy), 2=multiqueue
	submit_queues   number of hardware queues
	poll_queues     dedicated polled queues (Ch. 63 T.5)
	irqmode         0=none, 1=softirq, 2=timer
	completion_nsec simulated device latency
	hw_queue_depth  tags per queue
	memory_backed   0=discard writes, 1=actually store
	discard         support REQ_OP_DISCARD
	cache_size      simulate a write cache (-> flush handling)
	zoned, zone_size, zone_nr_conv, zone_max_open   (Ch. 60 T.5)
	badblocks       inject errors at specific sectors
	shared_tags, shared_tag_bitmap
	blocking        set BLK_MQ_F_BLOCKING
	virt_boundary
	max_sectors
	power           1 to instantiate
```

**Everything in Chapters 63–64 can be isolated and measured with `null_blk`**, because you can vary one parameter at a time with no hardware in the way. Lab 65.1 exploits this.

### 1.7 Observability for your own driver

| Where | What |
|---|---|
| `/sys/block/DEV/queue/*` | your declared limits, reflected back |
| `/sys/kernel/debug/block/DEV/hctx*/` ★★★ | tags, dispatch list, busy requests |
| `/sys/kernel/debug/block/DEV/hctx*/busy` | **in-flight requests, with sectors** |
| `blktrace` ★★★ | every stage |
| `trace-cmd record -e block:\*` | same |
| your own tracepoints | `TRACE_EVENT` in your driver |
| `/sys/kernel/debug/fail_make_request/` | fault injection |
| `/sys/block/DEV/io-timeout-fail` | timeout injection |

Adding tracepoints to your own driver is worth the effort:

```c
#undef TRACE_SYSTEM
#define TRACE_SYSTEM mydrv

#if !defined(_TRACE_MYDRV_H) || defined(TRACE_HEADER_MULTI_READ)
#define _TRACE_MYDRV_H
#include <linux/tracepoint.h>

TRACE_EVENT(mydrv_submit,
	TP_PROTO(struct request *rq, int slot),
	TP_ARGS(rq, slot),
	TP_STRUCT__entry(
		__field(unsigned int, tag)
		__field(int, slot)
		__field(sector_t, sector)
		__field(unsigned int, nr_sectors)
		__field(unsigned int, op)
	),
	TP_fast_assign(
		__entry->tag = rq->tag;
		__entry->slot = slot;
		__entry->sector = blk_rq_pos(rq);
		__entry->nr_sectors = blk_rq_sectors(rq);
		__entry->op = req_op(rq);
	),
	TP_printk("tag=%u slot=%d op=%u sector=%llu+%u",
		  __entry->tag, __entry->slot, __entry->op,
		  (unsigned long long)__entry->sector, __entry->nr_sectors)
);
#endif
```

---

## 2. Practice

### Lab 65.1 — `null_blk` as a laboratory

Before writing anything, use the reference driver to see every parameter's effect.

```sh
sudo modprobe null_blk nr_devices=0
sudo mount -t configfs none /sys/kernel/config 2>/dev/null
cd /sys/kernel/config/nullb
```

Compare bio-based against blk-mq:

```sh
for mode in 0 2; do
  sudo mkdir -p m$mode && cd m$mode
  echo 1024 | sudo tee size > /dev/null
  echo $mode | sudo tee queue_mode > /dev/null
  echo 1 | sudo tee memory_backed > /dev/null
  echo 1 | sudo tee power > /dev/null
  cd ..
done

lsblk | grep nullb
for d in nullb0 nullb1; do
  echo "=== $d ==="
  ls /sys/block/$d/mq/ 2>/dev/null | wc -l    # 0 for bio-based
  sudo fio --name=t --filename=/dev/$d --direct=1 --rw=randread --bs=4k \
           --iodepth=32 --ioengine=libaio --runtime=8 --time_based \
           --numjobs=4 --group_reporting 2>/dev/null | grep -oP 'IOPS=\K[^,]+'
done
```

Queue count and scalability:

```sh
for sq in 1 2 4 $(nproc); do
  cd /sys/kernel/config/nullb
  sudo rm -rf q$sq 2>/dev/null
  sudo mkdir q$sq && cd q$sq
  echo 1024 | sudo tee size > /dev/null
  echo 2 | sudo tee queue_mode > /dev/null
  echo $sq | sudo tee submit_queues > /dev/null
  echo 1 | sudo tee power > /dev/null
  D=$(lsblk -ndo NAME | grep nullb | tail -1)
  echo -n "submit_queues=$sq: "
  sudo fio --name=t --filename=/dev/$D --direct=1 --rw=randread --bs=4k \
           --iodepth=32 --ioengine=libaio --runtime=8 --time_based \
           --numjobs=$(nproc) --group_reporting 2>/dev/null | grep -oP 'IOPS=\K[^,]+'
  cd ..
done
```

**That is Ch. 63 §T.5's argument, isolated from every other variable.**

Simulated latency, cache, and error injection:

```sh
cd /sys/kernel/config/nullb
sudo mkdir -p lat && cd lat
echo 1024 | sudo tee size > /dev/null
echo 2 | sudo tee queue_mode > /dev/null
echo 100000 | sudo tee completion_nsec > /dev/null    # 100 us
echo 1 | sudo tee memory_backed > /dev/null
echo 64 | sudo tee cache_size > /dev/null             # a write cache!
echo 1 | sudo tee power > /dev/null
D=$(lsblk -ndo NAME | grep nullb | tail -1)

cat /sys/block/$D/queue/write_cache        # "write back": flushes now matter
sudo blktrace -d /dev/$D -o - 2>/dev/null | blkparse -i - &
sudo dd if=/dev/zero of=/dev/$D bs=4k count=10 oflag=direct,dsync 2>/dev/null
sleep 1; sudo pkill blktrace
# Look for FLUSH/FUA requests (Ch. 61 T.3)
```

Bad blocks:

```sh
echo "+0-7" | sudo tee /sys/kernel/config/nullb/lat/badblocks
sudo dd if=/dev/$D of=/dev/null bs=4k count=1 iflag=direct 2>&1 | tail -2
dmesg | tail -3
echo "-0-7" | sudo tee /sys/kernel/config/nullb/lat/badblocks
```

Cleanup:

```sh
cd /sys/kernel/config/nullb
for d in */; do sudo rmdir "$d" 2>/dev/null; done
```

---

### Lab 65.2 — A bio-based ramdisk

```c
// SPDX-License-Identifier: GPL-2.0
/* rmd.c -- a bio-based RAM block device with a radix tree of pages. */
#include <linux/blkdev.h>
#include <linux/highmem.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/radix-tree.h>
#include <linux/slab.h>

#define RMD_SECTOR_SHIFT	9
#define RMD_PAGE_SECTORS	(PAGE_SIZE >> RMD_SECTOR_SHIFT)

static int rmd_size_mb = 64;
module_param(rmd_size_mb, int, 0444);
MODULE_PARM_DESC(rmd_size_mb, "device size in MiB");

struct rmd_device {
	struct gendisk		*disk;
	struct radix_tree_root	pages;
	spinlock_t		lock;
	sector_t		capacity;
};

static struct rmd_device *rmd;
static int rmd_major;

/* Look up, or allocate, the page backing this sector. */
static struct page *rmd_lookup_page(struct rmd_device *d, sector_t sector)
{
	pgoff_t idx = sector >> (PAGE_SHIFT - RMD_SECTOR_SHIFT);
	struct page *page;

	rcu_read_lock();
	page = radix_tree_lookup(&d->pages, idx);
	rcu_read_unlock();
	return page;
}

static struct page *rmd_insert_page(struct rmd_device *d, sector_t sector)
{
	pgoff_t idx = sector >> (PAGE_SHIFT - RMD_SECTOR_SHIFT);
	struct page *page;

	page = rmd_lookup_page(d, sector);
	if (page)
		return page;

	/* GFP_NOIO: we may be under reclaim (T.7 rule 4). */
	page = alloc_page(GFP_NOIO | __GFP_ZERO);
	if (!page)
		return NULL;

	if (radix_tree_preload(GFP_NOIO)) {
		__free_page(page);
		return NULL;
	}
	spin_lock(&d->lock);
	if (radix_tree_insert(&d->pages, idx, page)) {
		__free_page(page);
		page = radix_tree_lookup(&d->pages, idx);   /* someone raced */
	}
	spin_unlock(&d->lock);
	radix_tree_preload_end();
	return page;
}

static int rmd_do_bvec(struct rmd_device *d, struct page *page,
		       unsigned int len, unsigned int off,
		       blk_opf_t opf, sector_t sector)
{
	struct page *backing;
	void *src, *dst;
	unsigned int page_off = (sector & (RMD_PAGE_SECTORS - 1)) << RMD_SECTOR_SHIFT;

	if (op_is_write(opf)) {
		backing = rmd_insert_page(d, sector);
		if (!backing)
			return -ENOMEM;
		src = kmap_local_page(page) + off;
		dst = kmap_local_page(backing) + page_off;
		memcpy(dst, src, len);
		kunmap_local(dst - page_off);
		kunmap_local(src - off);
	} else {
		backing = rmd_lookup_page(d, sector);
		dst = kmap_local_page(page) + off;
		if (backing) {
			src = kmap_local_page(backing) + page_off;
			memcpy(dst, src, len);
			kunmap_local(src - page_off);
		} else {
			memset(dst, 0, len);        /* a hole reads as zeros */
		}
		kunmap_local(dst - off);
	}
	return 0;
}

static void rmd_submit_bio(struct bio *bio)
{
	struct rmd_device *d = bio->bi_bdev->bd_disk->private_data;
	sector_t sector = bio->bi_iter.bi_sector;
	struct bio_vec bvec;
	struct bvec_iter iter;

	if (unlikely(bio_end_sector(bio) > d->capacity)) {
		bio_io_error(bio);
		return;
	}

	if (unlikely(bio_op(bio) == REQ_OP_DISCARD ||
		     bio_op(bio) == REQ_OP_WRITE_ZEROES)) {
		/* Free the backing pages for this range. */
		sector_t s = sector, end = bio_end_sector(bio);

		while (s < end) {
			pgoff_t idx = s >> (PAGE_SHIFT - RMD_SECTOR_SHIFT);
			struct page *p;

			spin_lock(&d->lock);
			p = radix_tree_delete(&d->pages, idx);
			spin_unlock(&d->lock);
			if (p)
				__free_page(p);
			s += RMD_PAGE_SECTORS;
		}
		bio_endio(bio);
		return;
	}

	/* Ch. 63 T.2: an explicit iterator; the bio is not mutated. */
	bio_for_each_segment(bvec, bio, iter) {
		unsigned int len = bvec.bv_len;
		int err;

		/* Each bvec may span several backing pages; split it. */
		while (len) {
			unsigned int page_off =
				(sector & (RMD_PAGE_SECTORS - 1)) << RMD_SECTOR_SHIFT;
			unsigned int this = min_t(unsigned int, len,
						  PAGE_SIZE - page_off);

			err = rmd_do_bvec(d, bvec.bv_page, this,
					  bvec.bv_offset + (bvec.bv_len - len),
					  bio->bi_opf, sector);
			if (err) {
				if (err == -ENOMEM && (bio->bi_opf & REQ_NOWAIT))
					bio_wouldblock_error(bio);
				else
					bio_io_error(bio);
				return;
			}
			sector += this >> RMD_SECTOR_SHIFT;
			len -= this;
		}
	}

	bio_endio(bio);            /* EXACTLY ONCE, on every path (T.1) */
}

static int rmd_getgeo(struct block_device *bdev, struct hd_geometry *geo)
{
	geo->heads = 4;
	geo->sectors = 16;
	geo->cylinders = get_capacity(bdev->bd_disk) / (4 * 16);
	geo->start = 0;
	return 0;
}

static const struct block_device_operations rmd_fops = {
	.owner		= THIS_MODULE,
	.submit_bio	= rmd_submit_bio,
	.getgeo		= rmd_getgeo,
};

static void rmd_free_pages(struct rmd_device *d)
{
	struct page *pages[16];
	pgoff_t idx = 0;
	int nr, i;

	do {
		nr = radix_tree_gang_lookup(&d->pages, (void **)pages, idx, 16);
		for (i = 0; i < nr; i++) {
			void *p = radix_tree_delete(&d->pages,
				page_index(pages[i]));
			if (p)
				__free_page(pages[i]);
		}
		idx += nr;
	} while (nr);
}

static int __init rmd_init(void)
{
	struct queue_limits lim = {
		.logical_block_size	= 512,
		.physical_block_size	= PAGE_SIZE,
		.io_min			= PAGE_SIZE,
		.io_opt			= PAGE_SIZE,
		.max_hw_sectors		= 1024,
		.max_discard_sectors	= UINT_MAX,
		.discard_granularity	= PAGE_SIZE,
		.max_write_zeroes_sectors = UINT_MAX,
	};
	int ret;

	rmd = kzalloc(sizeof(*rmd), GFP_KERNEL);
	if (!rmd)
		return -ENOMEM;
	INIT_RADIX_TREE(&rmd->pages, GFP_ATOMIC);
	spin_lock_init(&rmd->lock);
	rmd->capacity = (sector_t)rmd_size_mb << (20 - RMD_SECTOR_SHIFT);

	rmd_major = register_blkdev(0, "rmd");
	if (rmd_major < 0) { ret = rmd_major; goto err_free; }

	rmd->disk = blk_alloc_disk(&lim, NUMA_NO_NODE);
	if (IS_ERR(rmd->disk)) { ret = PTR_ERR(rmd->disk); goto err_unreg; }

	rmd->disk->major	= rmd_major;
	rmd->disk->first_minor	= 0;
	rmd->disk->minors	= 16;
	rmd->disk->fops		= &rmd_fops;
	rmd->disk->private_data	= rmd;
	strscpy(rmd->disk->disk_name, "rmd0", DISK_NAME_LEN);
	set_capacity(rmd->disk, rmd->capacity);

	/* T.3: everything must be ready BEFORE this line. */
	ret = add_disk(rmd->disk);
	if (ret) goto err_put;

	pr_info("rmd: /dev/rmd0, %d MiB\n", rmd_size_mb);
	return 0;

err_put:
	put_disk(rmd->disk);
err_unreg:
	unregister_blkdev(rmd_major, "rmd");
err_free:
	kfree(rmd);
	return ret;
}

static void __exit rmd_exit(void)
{
	del_gendisk(rmd->disk);
	put_disk(rmd->disk);
	unregister_blkdev(rmd_major, "rmd");
	rmd_free_pages(rmd);
	kfree(rmd);
}

module_init(rmd_init);
module_exit(rmd_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("bio-based RAM block device");
```

```sh
cat > Makefile <<'EOF'
obj-m += rmd.o
all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
EOF
make && sudo insmod rmd.ko rmd_size_mb=128

lsblk /dev/rmd0
cat /sys/block/rmd0/queue/{logical_block_size,physical_block_size,max_sectors_kb,discard_max_bytes}

sudo mkfs.ext4 -qF /dev/rmd0
sudo mkdir -p /mnt/rmd && sudo mount /dev/rmd0 /mnt/rmd
sudo cp -r /usr/include /mnt/rmd/ 2>/dev/null
df -h /mnt/rmd
sudo umount /mnt/rmd
```

Sparse allocation — pages are allocated on demand:

```sh
free -m | head -2
sudo dd if=/dev/zero of=/dev/rmd0 bs=1M count=64 2>/dev/null
free -m | head -2                            # memory consumed
sudo blkdiscard /dev/rmd0
free -m | head -2                            # returned
```

Verify correctness:

```sh
sudo fio --name=verify --filename=/dev/rmd0 --direct=1 --rw=randwrite \
         --bs=4k --size=64m --verify=crc32c --verify_fatal=1 2>&1 | tail -5
```

Partitions:

```sh
sudo parted -s /dev/rmd0 mklabel gpt mkpart p1 ext4 1MiB 32MiB mkpart p2 ext4 32MiB 100%
sudo partprobe /dev/rmd0
lsblk /dev/rmd0
sudo mkfs.ext4 -qF /dev/rmd0p1 && sudo mount /dev/rmd0p1 /mnt/rmd && df -h /mnt/rmd
sudo umount /mnt/rmd
```

---

### Lab 65.3 — Convert it to blk-mq

```c
// SPDX-License-Identifier: GPL-2.0
/* rmq.c -- the same device, request-based. */
#include <linux/blk-mq.h>
#include <linux/blkdev.h>
#include <linux/highmem.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/vmalloc.h>

#define RMQ_SIZE_MB	64
#define RMQ_QUEUE_DEPTH	64

struct rmq_cmd {
	struct request *rq;
	blk_status_t status;
};

struct rmq_device {
	struct blk_mq_tag_set	tag_set;
	struct gendisk		*disk;
	u8			*data;
	sector_t		capacity;
	atomic_t		inflight;
	bool			inject_timeout;
};

static struct rmq_device *rmq;
static int rmq_major;

static int rmq_inject_timeout;
module_param(rmq_inject_timeout, int, 0644);

static blk_status_t rmq_transfer(struct rmq_device *d, struct request *rq)
{
	struct bio_vec bvec;
	struct req_iterator iter;
	sector_t sector = blk_rq_pos(rq);

	if (blk_rq_pos(rq) + blk_rq_sectors(rq) > d->capacity)
		return BLK_STS_IOERR;

	/* rq_for_each_segment walks every bvec of every bio in the request. */
	rq_for_each_segment(bvec, rq, iter) {
		void *kaddr = kmap_local_page(bvec.bv_page) + bvec.bv_offset;
		u8 *store = d->data + (sector << SECTOR_SHIFT);

		if (rq_data_dir(rq) == WRITE)
			memcpy(store, kaddr, bvec.bv_len);
		else
			memcpy(kaddr, store, bvec.bv_len);

		kunmap_local(kaddr - bvec.bv_offset);
		sector += bvec.bv_len >> SECTOR_SHIFT;
	}
	return BLK_STS_OK;
}

static blk_status_t rmq_queue_rq(struct blk_mq_hw_ctx *hctx,
				 const struct blk_mq_queue_data *bd)
{
	struct request *rq = bd->rq;
	struct rmq_device *d = hctx->queue->queuedata;
	struct rmq_cmd *cmd = blk_mq_rq_to_pdu(rq);
	blk_status_t ret;

	/* MUST come before the request can be completed (T.4). */
	blk_mq_start_request(rq);
	cmd->rq = rq;
	atomic_inc(&d->inflight);

	switch (req_op(rq)) {
	case REQ_OP_READ:
	case REQ_OP_WRITE:
		ret = rmq_transfer(d, rq);
		break;
	case REQ_OP_FLUSH:
		ret = BLK_STS_OK;        /* RAM: nothing to flush */
		break;
	case REQ_OP_DISCARD:
	case REQ_OP_WRITE_ZEROES:
		memset(d->data + (blk_rq_pos(rq) << SECTOR_SHIFT), 0,
		       blk_rq_bytes(rq));
		ret = BLK_STS_OK;
		break;
	default:
		ret = BLK_STS_NOTSUPP;
		break;
	}

	/* Deliberately drop the completion, to exercise ->timeout (Lab 65.5) */
	if (unlikely(rmq_inject_timeout) && (rq->tag % 100) == 0) {
		pr_warn("rmq: dropping completion for tag %d\n", rq->tag);
		return BLK_STS_OK;       /* never completed! */
	}

	atomic_dec(&d->inflight);
	blk_mq_end_request(rq, ret);
	/* rq is GONE. Do not touch it. (T.6) */
	return BLK_STS_OK;
}

static enum blk_eh_timer_return rmq_timeout(struct request *rq)
{
	struct rmq_device *d = rq->q->queuedata;

	pr_err("rmq: request tag %d timed out after %ums\n",
	       rq->tag, jiffies_to_msecs(jiffies - rq->start_time_ns / NSEC_PER_MSEC));

	atomic_dec(&d->inflight);
	blk_mq_end_request(rq, BLK_STS_TIMEOUT);
	return BLK_EH_DONE;
}

static int rmq_init_request(struct blk_mq_tag_set *set, struct request *rq,
			    unsigned int hctx_idx, unsigned int numa_node)
{
	struct rmq_cmd *cmd = blk_mq_rq_to_pdu(rq);

	cmd->rq = rq;
	cmd->status = BLK_STS_OK;
	return 0;
}

static const struct blk_mq_ops rmq_mq_ops = {
	.queue_rq	= rmq_queue_rq,
	.timeout	= rmq_timeout,
	.init_request	= rmq_init_request,
};

static const struct block_device_operations rmq_fops = {
	.owner = THIS_MODULE,
};

static int __init rmq_init(void)
{
	struct queue_limits lim = {
		.logical_block_size	= 512,
		.physical_block_size	= 4096,
		.io_min			= 4096,
		.io_opt			= 65536,
		.max_hw_sectors		= 2048,
		.max_segments		= 128,
		.max_segment_size	= 65536,
		.max_discard_sectors	= 2048,
		.discard_granularity	= 4096,
		.max_write_zeroes_sectors = 2048,
	};
	int ret;

	rmq = kzalloc(sizeof(*rmq), GFP_KERNEL);
	if (!rmq) return -ENOMEM;

	rmq->capacity = (sector_t)RMQ_SIZE_MB << (20 - SECTOR_SHIFT);
	rmq->data = vzalloc(RMQ_SIZE_MB << 20);
	if (!rmq->data) { ret = -ENOMEM; goto err_free; }

	rmq_major = register_blkdev(0, "rmq");
	if (rmq_major < 0) { ret = rmq_major; goto err_vfree; }

	rmq->tag_set.ops		= &rmq_mq_ops;
	rmq->tag_set.nr_hw_queues	= num_possible_cpus();
	rmq->tag_set.nr_maps		= 1;
	rmq->tag_set.queue_depth	= RMQ_QUEUE_DEPTH;
	rmq->tag_set.numa_node		= NUMA_NO_NODE;
	rmq->tag_set.cmd_size		= sizeof(struct rmq_cmd);
	rmq->tag_set.flags		= BLK_MQ_F_SHOULD_MERGE;
	rmq->tag_set.driver_data	= rmq;

	ret = blk_mq_alloc_tag_set(&rmq->tag_set);
	if (ret) goto err_unreg;

	rmq->disk = blk_mq_alloc_disk(&rmq->tag_set, &lim, rmq);
	if (IS_ERR(rmq->disk)) { ret = PTR_ERR(rmq->disk); goto err_tagset; }

	rmq->disk->major	= rmq_major;
	rmq->disk->first_minor	= 0;
	rmq->disk->minors	= 16;
	rmq->disk->fops		= &rmq_fops;
	rmq->disk->private_data	= rmq;
	rmq->disk->queue->queuedata = rmq;
	strscpy(rmq->disk->disk_name, "rmq0", DISK_NAME_LEN);
	set_capacity(rmq->disk, rmq->capacity);
	blk_queue_rq_timeout(rmq->disk->queue, 5 * HZ);

	ret = add_disk(rmq->disk);
	if (ret) goto err_cleanup;

	pr_info("rmq: /dev/rmq0, %d MiB, %d hw queues, depth %d\n",
		RMQ_SIZE_MB, rmq->tag_set.nr_hw_queues, RMQ_QUEUE_DEPTH);
	return 0;

err_cleanup:
	put_disk(rmq->disk);
err_tagset:
	blk_mq_free_tag_set(&rmq->tag_set);
err_unreg:
	unregister_blkdev(rmq_major, "rmq");
err_vfree:
	vfree(rmq->data);
err_free:
	kfree(rmq);
	return ret;
}

static void __exit rmq_exit(void)
{
	del_gendisk(rmq->disk);
	put_disk(rmq->disk);
	blk_mq_free_tag_set(&rmq->tag_set);
	unregister_blkdev(rmq_major, "rmq");
	vfree(rmq->data);
	kfree(rmq);
}

module_init(rmq_init);
module_exit(rmq_exit);
MODULE_LICENSE("GPL");
```

```sh
sudo insmod rmq.ko
ls /sys/block/rmq0/mq/                  # hardware queues exist now
cat /sys/block/rmq0/queue/scheduler
sudo cat /sys/kernel/debug/block/rmq0/hctx0/tags
```

Compare with the bio-based version:

```sh
for d in rmd0 rmq0; do
  echo "=== $d ==="
  sudo blktrace -d /dev/$d -o /tmp/bt-$d &
  sudo dd if=/dev/zero of=/dev/$d bs=4k count=2000 oflag=direct 2>/dev/null
  sudo pkill blktrace; sleep 1
  blkparse -i /tmp/bt-$d 2>/dev/null | awk '{print $6}' | sort | uniq -c | sort -rn | head -5
done
```

**`rmd0` shows only `Q`; `rmq0` shows `Q G I D C` and merges.** That is §T.2, demonstrated.

Merging:

```sh
sudo blktrace -d /dev/rmq0 -o /tmp/m &
sudo dd if=/dev/zero of=/dev/rmq0 bs=4k count=2000 oflag=direct 2>/dev/null
sudo pkill blktrace; sleep 1
echo -n "Q: "; blkparse -i /tmp/m | grep -c ' Q '
echo -n "M: "; blkparse -i /tmp/m | grep -cE ' M '
echo -n "D: "; blkparse -i /tmp/m | grep -c ' D '
```

Schedulers work now:

```sh
cat /sys/block/rmq0/queue/scheduler
for s in none mq-deadline kyber; do
  echo $s | sudo tee /sys/block/rmq0/queue/scheduler > /dev/null 2>&1 || continue
  echo -n "$s: "
  sudo fio --name=t --filename=/dev/rmq0 --direct=1 --rw=randread --bs=4k \
           --iodepth=32 --ioengine=libaio --runtime=5 --time_based \
           --numjobs=4 --group_reporting 2>/dev/null | grep -oP 'IOPS=\K[^,]+'
done
```

---

### Lab 65.4 — Break it deliberately

Each of these is a real bug class. Introduce it, observe the failure, fix it.

**Bug 1: forget to complete a request.**

```sh
sudo rmmod rmq
sudo insmod rmq.ko rmq_inject_timeout=1

sudo dd if=/dev/zero of=/dev/rmq0 bs=4k count=500 oflag=direct 2>&1 | tail -2
dmesg | tail -10
# "rmq: dropping completion for tag N"
# "rmq: request tag N timed out after 5000ms"
```

Now disable the timeout handler (comment out `.timeout`) and repeat:

```sh
# Without ->timeout, the process hangs FOREVER in D state.
ps -eo pid,stat,comm,wchan | grep ' D '
cat /proc/$(pgrep dd)/stack 2>/dev/null
sudo cat /sys/kernel/debug/block/rmq0/hctx0/busy    # the stuck request
```

**Bug 2: touch the request after completing it.**

```c
	blk_mq_end_request(rq, ret);
	pr_info("completed tag %d\n", rq->tag);      /* USE AFTER FREE */
```

```sh
# Build with CONFIG_KASAN=y and load it:
dmesg | grep -A20 'BUG: KASAN'
# "use-after-free in rmq_queue_rq"
```

**Bug 3: sleep in `queue_rq`.**

```c
	msleep(1);              /* or kmalloc(GFP_KERNEL) */
```

```sh
# With CONFIG_DEBUG_ATOMIC_SLEEP=y:
dmesg | grep -A15 'sleeping function called from invalid context'
```

Then do it correctly:

```c
	rmq->tag_set.flags = BLK_MQ_F_SHOULD_MERGE | BLK_MQ_F_BLOCKING;
	/* NOW queue_rq may sleep. */
```

**Bug 4: overstate `max_hw_sectors`.**

```c
	.max_hw_sectors = 65536,     /* claim 32 MiB, handle 1 MiB */
```

```sh
sudo dd if=/dev/zero of=/dev/rmq0 bs=8M count=5 oflag=direct 2>&1 | tail -2
dmesg | tail -5
```

**Bug 5: `add_disk` before the device is ready.**

```c
	ret = add_disk(rmq->disk);
	rmq->data = vzalloc(RMQ_SIZE_MB << 20);   /* TOO LATE */
```

```sh
# udev opens the device immediately -> NULL deref
dmesg | grep -A20 'BUG: kernel NULL pointer'
```

**Bug 6: return `BLK_STS_RESOURCE` when you meant `DEV_RESOURCE`.**

```c
	if (atomic_read(&d->inflight) >= RMQ_QUEUE_DEPTH)
		return BLK_STS_RESOURCE;      /* busy-loop! */
```

```sh
top -b -n1 | head -5                # 100% system time in one core
sudo perf top -e cycles --stdio 2>/dev/null | head -10
# blk_mq_dispatch_rq_list spinning
```

**Bug 7: unload the module with the device mounted.**

```c
static void __exit rmq_exit(void)
{
	/* MISSING: del_gendisk(rmq->disk); */
	vfree(rmq->data);
	kfree(rmq);
}
```

```sh
sudo mkfs.ext4 -qF /dev/rmq0 && sudo mount /dev/rmq0 /mnt/rmd
sudo rmmod rmq                       # oops on the next I/O
```

Run the whole sweep under the full debug configuration:

```sh
grep -E 'CONFIG_(KASAN|UBSAN|PROVE_LOCKING|DEBUG_ATOMIC_SLEEP)=' /boot/config-$(uname -r)
```

---

### Lab 65.5 — Fault and timeout injection

Rather than hand-inject, use the kernel's facilities.

```sh
grep -E 'CONFIG_FAIL_(MAKE_REQUEST|IO_TIMEOUT)' /boot/config-$(uname -r)
ls /sys/kernel/debug/fail_make_request/
```

Make requests fail randomly:

```sh
echo 50  | sudo tee /sys/kernel/debug/fail_make_request/probability
echo 100 | sudo tee /sys/kernel/debug/fail_make_request/times
echo 1   | sudo tee /sys/kernel/debug/fail_make_request/verbose
echo 1   | sudo tee /sys/block/rmq0/make-it-fail

sudo dd if=/dev/zero of=/dev/rmq0 bs=4k count=200 oflag=direct 2>&1 | tail -3
dmesg | tail -10

echo 0 | sudo tee /sys/block/rmq0/make-it-fail
```

With a filesystem on top — does it survive?

```sh
echo 0 | sudo tee /sys/block/rmq0/make-it-fail
sudo mkfs.ext4 -qF /dev/rmq0
sudo mount /dev/rmq0 /mnt/rmd
sudo cp -r /usr/include /mnt/rmd/ 2>/dev/null

echo 5 | sudo tee /sys/kernel/debug/fail_make_request/probability
echo 1 | sudo tee /sys/block/rmq0/make-it-fail
sudo cp -r /usr/share/doc /mnt/rmd/ 2>&1 | tail -3
dmesg | tail -20                    # ext4 should report errors, not crash
echo 0 | sudo tee /sys/block/rmq0/make-it-fail
sudo umount /mnt/rmd
sudo e2fsck -fn /dev/rmq0
```

Timeout injection:

```sh
echo 100 | sudo tee /sys/kernel/debug/fail_io_timeout/probability
echo 10  | sudo tee /sys/kernel/debug/fail_io_timeout/times
echo 1   | sudo tee /sys/block/rmq0/io-timeout-fail

sudo dd if=/dev/zero of=/dev/rmq0 bs=4k count=50 oflag=direct 2>&1 | tail -2
dmesg | tail -10                    # your ->timeout handler runs
echo 0 | sudo tee /sys/block/rmq0/io-timeout-fail
```

Hot-unplug during I/O (§T.8):

```sh
sudo fio --name=t --filename=/dev/rmq0 --direct=1 --rw=randwrite --bs=4k \
         --iodepth=32 --ioengine=libaio --runtime=30 --time_based > /dev/null 2>&1 &
FIO=$!
sleep 3
sudo rmmod rmq                       # while I/O is in flight
wait $FIO 2>/dev/null
dmesg | tail -10
lsmod | grep rmq
```

A correct driver survives this. An incorrect one oopses.

---

### Lab 65.6 — A network-backed driver: the hard cases

`nbd` is the model. Study it, then extend `rmq` to be asynchronous.

```sh
sudo apt install -y nbd-server nbd-client
dd if=/dev/zero of=/tmp/nbd.img bs=1M count=256 2>/dev/null
sudo nbd-server 10809 /tmp/nbd.img &
sleep 1
sudo modprobe nbd
sudo nbd-client 127.0.0.1 10809 /dev/nbd0
lsblk /dev/nbd0
```

Observe its structure:

```sh
cat /sys/block/nbd0/queue/{nr_requests,max_sectors_kb}
ls /sys/block/nbd0/mq/
sudo cat /sys/kernel/debug/block/nbd0/hctx0/flags    # BLK_MQ_F_BLOCKING

sudo blktrace -d /dev/nbd0 -o /tmp/nbd &
sudo dd if=/dev/zero of=/dev/nbd0 bs=4k count=100 oflag=direct 2>/dev/null
sudo pkill blktrace; sleep 1
btt -i /tmp/nbd 2>/dev/null | grep -A8 ALL | head -10
# D2C is dominated by the network round trip.
```

Network failure — the case that makes network drivers hard:

```sh
sudo fio --name=t --filename=/dev/nbd0 --direct=1 --rw=randwrite --bs=4k \
         --iodepth=8 --ioengine=libaio --runtime=30 --time_based > /dev/null 2>&1 &
FIO=$!
sleep 3
sudo pkill nbd-server                # the server vanishes
sleep 10
dmesg | tail -20
# nbd's timeout handler fires; requests are failed or the connection is
# retried, depending on -t and the dead-connection configuration.
kill $FIO 2>/dev/null
sudo nbd-client -d /dev/nbd0 2>/dev/null
```

Now extend `rmq` to complete asynchronously, which is what a real driver does:

```c
/* Add to rmq_device: */
	struct workqueue_struct *wq;
	spinlock_t pending_lock;
	struct list_head pending;

struct rmq_cmd {
	struct request *rq;
	struct list_head node;
	struct work_struct work;
	blk_status_t status;
};

static void rmq_complete_work(struct work_struct *work)
{
	struct rmq_cmd *cmd = container_of(work, struct rmq_cmd, work);
	struct request *rq = cmd->rq;
	struct rmq_device *d = rq->q->queuedata;

	/* Simulate device latency */
	usleep_range(50, 200);

	cmd->status = rmq_transfer(d, rq);
	atomic_dec(&d->inflight);

	/* Ch. 63 T.7: this respects rq_affinity and may IPI. */
	blk_mq_complete_request(rq);
}

static void rmq_complete_rq(struct request *rq)
{
	struct rmq_cmd *cmd = blk_mq_rq_to_pdu(rq);

	blk_mq_end_request(rq, cmd->status);
}

static blk_status_t rmq_queue_rq_async(struct blk_mq_hw_ctx *hctx,
				       const struct blk_mq_queue_data *bd)
{
	struct request *rq = bd->rq;
	struct rmq_device *d = hctx->queue->queuedata;
	struct rmq_cmd *cmd = blk_mq_rq_to_pdu(rq);

	blk_mq_start_request(rq);
	cmd->rq = rq;
	atomic_inc(&d->inflight);

	INIT_WORK(&cmd->work, rmq_complete_work);
	queue_work(d->wq, &cmd->work);

	return BLK_STS_OK;       /* completion happens LATER */
}

static const struct blk_mq_ops rmq_mq_ops = {
	.queue_rq	= rmq_queue_rq_async,
	.complete	= rmq_complete_rq,        /* now needed */
	.timeout	= rmq_timeout,
	.init_request	= rmq_init_request,
};
```

Measure the difference:

```sh
for mod in rmq_sync rmq_async; do
  sudo rmmod rmq 2>/dev/null
  sudo insmod $mod.ko
  echo "=== $mod ==="
  sudo fio --name=t --filename=/dev/rmq0 --direct=1 --rw=randread --bs=4k \
           --iodepth=32 --ioengine=libaio --runtime=8 --time_based \
           --numjobs=4 --group_reporting 2>/dev/null | \
    grep -E 'IOPS|clat.*avg'
done
```

And check where completions land:

```sh
for ra in 0 2; do
  echo $ra | sudo tee /sys/block/rmq0/queue/rq_affinity > /dev/null
  echo "=== rq_affinity=$ra ==="
  sudo bpftrace -e '
  tracepoint:block:block_rq_issue    { @submit[cpu] = count(); }
  tracepoint:block:block_rq_complete { @complete[cpu] = count(); }
  interval:s:8 { print(@submit); print(@complete); exit(); }' &
  sudo fio --name=t --filename=/dev/rmq0 --direct=1 --rw=randread --bs=4k \
           --iodepth=32 --ioengine=libaio --runtime=8 --time_based \
           --numjobs=$(nproc) > /dev/null 2>&1
  wait
done
```

---

### Lab 65.7 — `ioctl`s, sysfs, and identity

```c
/* Add to rmq.c */
#define RMQ_IOC_MAGIC		'R'
#define RMQ_IOC_GET_STATS	_IOR(RMQ_IOC_MAGIC, 1, struct rmq_stats)
#define RMQ_IOC_RESET_STATS	_IO(RMQ_IOC_MAGIC, 2)
#define RMQ_IOC_SET_LATENCY	_IOW(RMQ_IOC_MAGIC, 3, __u32)

struct rmq_stats {
	__u64 reads;
	__u64 writes;
	__u64 read_sectors;
	__u64 write_sectors;
	__u64 timeouts;
	__u32 inflight;
	__u32 pad;
};

static int rmq_ioctl(struct block_device *bdev, blk_mode_t mode,
		     unsigned int cmd, unsigned long arg)
{
	struct rmq_device *d = bdev->bd_disk->private_data;
	struct rmq_stats st;
	u32 lat;

	switch (cmd) {
	case RMQ_IOC_GET_STATS:
		memset(&st, 0, sizeof(st));
		st.reads		= atomic64_read(&d->reads);
		st.writes		= atomic64_read(&d->writes);
		st.read_sectors		= atomic64_read(&d->read_sectors);
		st.write_sectors	= atomic64_read(&d->write_sectors);
		st.timeouts		= atomic64_read(&d->timeouts);
		st.inflight		= atomic_read(&d->inflight);
		if (copy_to_user((void __user *)arg, &st, sizeof(st)))
			return -EFAULT;
		return 0;

	case RMQ_IOC_RESET_STATS:
		if (!(mode & BLK_OPEN_WRITE))
			return -EACCES;
		atomic64_set(&d->reads, 0);
		atomic64_set(&d->writes, 0);
		return 0;

	case RMQ_IOC_SET_LATENCY:
		if (!capable(CAP_SYS_ADMIN))
			return -EPERM;
		if (get_user(lat, (u32 __user *)arg))
			return -EFAULT;
		if (lat > 1000000)
			return -EINVAL;
		WRITE_ONCE(d->latency_us, lat);
		return 0;
	}
	return -ENOTTY;
}

/* A stable unique ID, for multipath and udev */
static int rmq_get_unique_id(struct gendisk *disk, u8 id[16],
			     enum blk_unique_id type)
{
	struct rmq_device *d = disk->private_data;

	if (type != BLK_UID_UUID)
		return -EOPNOTSUPP;
	memcpy(id, &d->uuid, 16);
	return 16;
}

static const struct block_device_operations rmq_fops = {
	.owner		= THIS_MODULE,
	.ioctl		= rmq_ioctl,
	.compat_ioctl	= blkdev_compat_ptr_ioctl,    /* Ch. 24 T.7 */
	.getgeo		= rmq_getgeo,
	.get_unique_id	= rmq_get_unique_id,
};
```

Sysfs attributes:

```c
static ssize_t latency_us_show(struct device *dev,
			       struct device_attribute *attr, char *buf)
{
	struct gendisk *disk = dev_to_disk(dev);
	struct rmq_device *d = disk->private_data;

	return sysfs_emit(buf, "%u\n", READ_ONCE(d->latency_us));
}

static ssize_t latency_us_store(struct device *dev,
				struct device_attribute *attr,
				const char *buf, size_t count)
{
	struct gendisk *disk = dev_to_disk(dev);
	struct rmq_device *d = disk->private_data;
	u32 val;
	int ret;

	ret = kstrtou32(buf, 10, &val);
	if (ret) return ret;
	if (val > 1000000) return -EINVAL;
	WRITE_ONCE(d->latency_us, val);
	return count;
}
static DEVICE_ATTR_RW(latency_us);

static struct attribute *rmq_attrs[] = {
	&dev_attr_latency_us.attr,
	NULL,
};
ATTRIBUTE_GROUPS(rmq);

/* Then in init: */
	rmq->disk->queue->limits.features |= BLK_FEAT_...;
	/* Attach the group via device_add_disk's groups argument, or: */
	ret = sysfs_create_group(&disk_to_dev(rmq->disk)->kobj, &rmq_group);
```

Test it:

```c
// SPDX-License-Identifier: GPL-2.0
/* rmqctl.c */
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <sys/ioctl.h>
#include <unistd.h>
#include <stdint.h>

#define RMQ_IOC_MAGIC     'R'
#define RMQ_IOC_GET_STATS _IOR(RMQ_IOC_MAGIC, 1, struct rmq_stats)

struct rmq_stats {
	uint64_t reads, writes, read_sectors, write_sectors, timeouts;
	uint32_t inflight, pad;
};

int main(int argc, char **argv)
{
	int fd = open(argv[1], O_RDONLY);
	struct rmq_stats st;

	if (fd < 0) { perror("open"); return 1; }
	if (ioctl(fd, RMQ_IOC_GET_STATS, &st)) { perror("ioctl"); return 1; }
	printf("reads=%lu writes=%lu rsect=%lu wsect=%lu timeouts=%lu inflight=%u\n",
	       st.reads, st.writes, st.read_sectors, st.write_sectors,
	       st.timeouts, st.inflight);
	return 0;
}
```

```sh
gcc -o rmqctl rmqctl.c
sudo dd if=/dev/zero of=/dev/rmq0 bs=1M count=10 oflag=direct 2>/dev/null
sudo ./rmqctl /dev/rmq0
cat /sys/block/rmq0/latency_us
echo 500 | sudo tee /sys/block/rmq0/latency_us
```

Compare with the standard ioctls everyone gets for free:

```sh
sudo blockdev --report /dev/rmq0
sudo blockdev --getsize64 /dev/rmq0
sudo blockdev --getss /dev/rmq0
sudo blockdev --getpbsz /dev/rmq0
sudo hdparm -I /dev/rmq0 2>&1 | head -5
```

---

### Lab 65.8 — Run `blktests`

```sh
git clone https://github.com/osandov/blktests.git
cd blktests

cat > config <<'EOF'
TEST_DEVS=(/dev/rmq0)
EOF

sudo ./check block/
sudo ./check block/001 block/002 block/005 block/013
sudo ./check loop/
sudo ./check nbd/
```

Each failure is a contract you have not met. Fix them in order.

Then filesystem-level testing:

```sh
sudo mkfs.xfs -f /dev/rmq0
sudo mkdir -p /mnt/rmq && sudo mount /dev/rmq0 /mnt/rmq

# fsstress: concurrent metadata chaos
sudo ~/xfstests-dev/ltp/fsstress -d /mnt/rmq -n 10000 -p 8 -v 2>&1 | tail -5

# fsx: data integrity under mixed operations
sudo ~/xfstests-dev/ltp/fsx -N 100000 /mnt/rmq/fsxfile

sudo umount /mnt/rmq
sudo xfs_repair -n /dev/rmq0
```

Concurrent stress with verification:

```sh
sudo fio --name=stress --filename=/dev/rmq0 --direct=1 \
         --rw=randrw --rwmixread=50 --bs=4k-128k --iodepth=32 \
         --ioengine=libaio --numjobs=$(nproc) --runtime=120 --time_based \
         --verify=crc32c --verify_fatal=1 --group_reporting 2>&1 | tail -10
```

The full debug sweep:

```sh
# On a kernel built with the T.9 configuration:
dmesg -C
sudo ./check block/ loop/
sudo fio --name=t --filename=/dev/rmq0 --direct=1 --rw=randrw --bs=4k \
         --iodepth=64 --ioengine=libaio --numjobs=$(nproc) \
         --runtime=60 --time_based --verify=crc32c > /dev/null 2>&1
dmesg | grep -E 'KASAN|BUG|WARNING|lockdep|sleeping function' | head -20
```

Zero output is the target.

---

## 3. Mastery drills

1. §T.1 lists eight obligations. For each, write the minimal driver bug that violates it and predict the exact symptom a user would report.

2. Give the decision rule for bio-based versus request-based, then classify: `dm-crypt`, `zram`, `virtio_blk`, `md-raid1`, `loop`, `nvme`. Justify each.

3. `add_disk()` is the publication point. Construct the race with udev precisely, and explain why it appears only on fast machines with many cores.

4. `BLK_STS_RESOURCE` versus `BLK_STS_DEV_RESOURCE`: write the block layer's response to each, then construct both the busy-loop and the stall.

5. `init_request` is called once per tag, not per I/O. Compute the memory a 64-queue, 1024-depth device with `cmd_size=256` preallocates, and explain why the alternative is unacceptable.

6. The timeout handler races with completion. Write both interleavings and show how the `cmpxchg` on `rq->state` makes exactly one win.

7. Declaring `BLK_FEAT_WRITE_CACHE` obliges you to implement `REQ_OP_FLUSH`. Trace the consequence of ignoring it from `fsync()` down to a power failure, naming every layer that is misled.

8. §T.7 gives four rules for allocation in the I/O path. For each, construct the deadlock it prevents, and find the analogous mechanism in three other chapters of Part 3.

9. `del_gendisk()` waits for in-flight I/O and open references. Construct the use-after-free that occurs without it, and explain what `free_disk` does differently.

10. Compute the throughput difference between ringing a doorbell per request and per batch, given a 500 ns MMIO write and a device doing 500K IOPS.

11. Your driver passes `fio --verify=crc32c` but corrupts data under `fsstress`. Give the three most likely causes and the diagnostic for each.

12. Design the complete error-handling strategy for a network-backed block driver: timeout, reconnection, in-flight request disposition, and what to tell the filesystem. State what you would do differently for a local device.

13. You are handed a block driver that achieves 50K IOPS where the hardware does 500K. Give the ordered diagnostic procedure using Chapters 63–65's tools, and the six most likely causes.

---

## 4. Further reading

**Kernel documentation**

- `Documentation/block/blk-mq.rst` ★★★
- `Documentation/block/queue-sysfs.rst` ★★★ — your limits, as users will see them.
- `Documentation/block/null_blk.rst` ★★★ — every configfs parameter; §1.6.
- `Documentation/block/biodoc.rst` — the layer's responsibilities.
- `Documentation/fault-injection/fault-injection.rst` ★★★ — Lab 65.5.
- `Documentation/core-api/kernel-api.rst` (the block section)
- `Documentation/driver-api/` — general driver conventions (Ch. 26–28).

**Source to read, in order**

1. `drivers/block/brd.c` ★★★ — **~400 lines; read all of it.** The minimal bio-based driver.
2. `drivers/block/null_blk/main.c` ★★★ — the reference blk-mq driver, with every option.
3. `drivers/block/virtio_blk.c` ★★★ — **the best model for a real driver.** Small, complete, production-quality; shows `queue_rqs` batching, `commit_rqs`, polling, and correct teardown.
4. `drivers/block/loop.c` ★★★ — `BLK_MQ_F_BLOCKING`, workqueue completion, and a realistic amount of complexity.
5. `drivers/block/nbd.c` ★★★ — network-backed; the timeout, reconnection, and error handling of §T.8 and Lab 65.6.
6. `drivers/nvme/host/pci.c` — high performance: multiple queue maps, polling, batched completion, `bd->last` doorbell handling.
7. `block/blk-mq.c`: `blk_mq_dispatch_rq_list`, `blk_mq_end_request`, `blk_mq_rq_timed_out` — the other side of your driver's interface.
8. `drivers/block/zram/zram_drv.c` — a more complex bio-based driver, with per-page state.

**Books**

- Corbet, Rubini, Kroah-Hartman, *Linux Device Drivers, 3rd ed.*, Chapter 16 ("Block Drivers") — **badly dated** (pre-blk-mq, pre-bio-iterator) but the *structure* of the explanation is still useful. Read it knowing what has changed.
- Love, *Linux Kernel Development*, Chapter 14 ("The Block I/O Layer") — conceptual, still accurate on the fundamentals.
- Venkateswaran, *Essential Linux Device Drivers*, Chapter 14 — similar caveats.

There is no current book on blk-mq drivers. The source and LWN are the documentation.

**LWN**

- "Block layer introduction part 1: the bio layer" and "part 2: the request layer" (Neil Brown) ★★★ — the best conceptual grounding.
- "The multiqueue block layer" ★★★
- "Immutable biovecs and biovec iterators" ★★★
- "A block layer introduction for driver writers" and the various driver-porting articles
- "Improving the block layer's queue limits API" (2024) — the `blk_mq_alloc_disk(&lim, ...)` change of §T.3.
- "Removing the legacy block layer" — what changed and why older driver examples mislead.
- "Fault injection in the block layer"
- "blktests: testing the block layer" ★★★

**Tools**

- `null_blk` ★★★ — **use it constantly.** Isolating one variable at a time is the only way to understand this layer.
- `blktests` ★★★ — `./check block/` is the conformance bar.
- `blktrace` + `blkparse` + `btt` ★★★
- `fio --verify=crc32c --verify_fatal=1` ★★★ — the correctness check
- `fsstress`, `fsx` (in `xfstests-dev/ltp/`) ★★★
- `/sys/kernel/debug/block/DEV/hctx*/busy` ★★★ — find the stuck request
- `/sys/kernel/debug/fail_make_request/`, `/sys/block/*/io-timeout-fail` ★★★
- KASAN, lockdep, `DEBUG_ATOMIC_SLEEP`, `DEBUG_KOBJECT_RELEASE` ★★★ — **build with all of them, always**
- `dmesg -w` in a second terminal, permanently
- `perf top`, `perf record -e block:\*`

---

→ Next: [66-device-mapper.md](66-device-mapper.md)

# Chapter 63 — Block layer I: `bio`, request queues, and the blk-mq architecture

> **Goal:** Understand the layer that turns "write these pages to this device" into hardware commands, and why it had to be rewritten. Understand the `bio` as the universal I/O description and why its iterator design matters, the bio→request→command pipeline, the single-queue architecture's three scalability bottlenecks and how blk-mq's software/hardware queue split eliminates each, tags as the unifying abstraction, the completion path and interrupt steering, bio splitting against device limits, and the stacking model that makes device-mapper and MD possible. By the end you can read `block/`, trace an I/O from `submit_bio` to completion, and diagnose where latency is spent.

---

## Theory & First Principles

### T.0 — Start here: 700 processes, one device

```bash
sudo iotop -o                                # who is doing I/O right now
cat /sys/block/nvme0n1/queue/nr_requests     # how deep the queue is
cat /sys/block/nvme0n1/queue/scheduler       # [none] mq-deadline kyber bfq
```

Every process on the machine wants the device. The device serves one request at a time (or
64, or 65,536 — that number turns out to matter enormously). **Something has to sit between
them, and that something is the block layer.**

It has exactly four jobs, and it is worth holding them apart because different eras of
hardware weight them completely differently:

| Job | Why | Worth it on HDD? | On NVMe? |
|---|---|---|---|
| **Merge** adjacent requests | one 128 KiB request beats 32 x 4 KiB | **yes, hugely** | yes — still saves per-request CPU |
| **Reorder** into an elevator | avoid seeks | **yes, 100x** | **no** — there is no seek |
| **Fairness / QoS** | one process must not starve another | yes | yes |
| **Uniform abstraction** (`bio`) | filesystems, dm, md all speak one language | yes | yes |

**Column 3 versus column 4 is the whole story of the modern block layer.** Three of the four
jobs survived the move to flash; the most famous one did not.

**Start with the unit of work.** The block layer's narrow waist is the `bio`:

```c
struct bio {
	struct block_device *bi_bdev;     /* which device              */
	unsigned int         bi_opf;      /* READ/WRITE | flags        */
	struct bvec_iter     bi_iter;     /* WHERE: sector + size      */
	struct bio_vec      *bi_io_vec;   /* WHAT: a scatter list of
	                                     (page, offset, len)       */
	bio_end_io_t        *bi_end_io;   /* call me when it's done    */
};
```

**Three observations that explain a lot of design downstream:**

1. **It is a scatter-gather list, not a buffer.** The pages of a 1 MiB read are not
   contiguous in physical memory, and they do not need to be — the device's DMA engine walks
   the list (Ch. 35).
2. **It is asynchronous by construction.** There is no "wait here"; there is a completion
   callback. Every synchronous-looking read in the kernel is a `bio` plus an explicit wait.
3. **It is stackable.** A `bio` can be *cloned and remapped* by device mapper or MD and
   passed down to another device. That is how LVM, dm-crypt, and RAID compose (Ch. 66) — they
   are filters over a stream of `bio`s, which is precisely why the `bio` had to be a
   self-contained description rather than a pointer into the caller's state.

**Now the historical break, which is Ch. 64's subject and which you should anticipate here.**
The original design had **one request queue per device, protected by one spinlock**:

```
   CPU0  CPU1  CPU2  ...  CPU63
     \     |     |        /
      \    |     |       /
       v   v     v      v
     +---------------------+
     |   queue_lock        |   <-- every submission, every completion,
     |   one request queue |       every merge attempt takes THIS lock
     +---------------------+
```

At 100 IOPS this is free. At **1,000,000 IOPS** it is Ch. 16 §T.0's shared counter: the lock
itself consumes more CPU than the I/O, and adding cores makes throughput *fall*.
Measurements around 2010–2013 showed the single-queue layer capping out near 800K–1M IOPS
**regardless of how fast the device was**.

**The fix (blk-mq, merged in 3.13) was not a faster lock — it was to remove the sharing**:
per-CPU software queues feeding per-device hardware queues, so the common path touches no
shared cacheline at all. That is the same answer as per-CPU counters (Ch. 16), RCU (Ch. 15),
and XFS allocation groups (Ch. 58 §T.0): **when a lock is the bottleneck, partition the data
so the lock is not needed.**

```bash
ls /sys/block/nvme0n1/mq/            # one directory per hardware queue
sudo blktrace -d /dev/nvme0n1 -o - | blkparse -i -   # Q G I D C per request
cat /sys/block/sda/queue/{max_sectors_kb,nr_requests,rotational}
sudo /usr/share/bcc/tools/biolatency -D 10 1
```

---

### T.1 What the block layer is for

Between the filesystem and the driver sits a layer whose job list is longer than it first appears:

| Job | Why it cannot live in the filesystem or the driver |
|---|---|
| **Merging** adjacent requests | requires seeing all I/O to a device, across filesystems |
| **Sorting/scheduling** | same |
| **Splitting** to device limits | the filesystem does not know the device's max transfer size |
| **Stacking** (dm, MD) | needs a device-independent I/O description |
| **Accounting** | one place to count |
| **Tagging** | must be coordinated across all submitters |
| **Plugging** (batching) | requires knowing when a submitter has finished a batch |
| **Bounce buffering** | address-limit handling belongs below the filesystem |
| **QoS** (throttling, latency targets) | needs the whole picture |

The layer's defining abstraction is the **`bio`**: a device-independent description of "transfer these memory pages to/from this device at this offset." Everything above produces bios; everything below consumes them.

Note what the block layer is *not*: it is not a driver, and it does not know about SCSI, NVMe, or SATA. The narrow waist of Ch. 51 §T.2 lives here.

### T.2 The `bio`: one structure, three roles

```c
struct bio {
	struct bio		*bi_next;	/* chaining */
	struct block_device	*bi_bdev;
	blk_opf_t		bi_opf;		/* op + flags: REQ_OP_READ | REQ_FUA | ... */
	unsigned short		bi_flags;
	unsigned short		bi_ioprio;
	enum req_op		bi_write_hint;
	blk_status_t		bi_status;
	atomic_t		__bi_remaining;	/* for chained bios */

	struct bvec_iter	bi_iter;	/* WHERE WE ARE -- see below */

	bio_end_io_t		*bi_end_io;
	void			*bi_private;
	...
	unsigned short		bi_vcnt;	/* how many bio_vecs */
	unsigned short		bi_max_vecs;
	atomic_t		__bi_cnt;	/* reference count */
	struct bio_vec		*bi_io_vec;	/* the actual vector */
	struct bio_set		*bi_pool;
	struct bio_vec		bi_inline_vecs[];  /* small-case optimisation */
};

struct bio_vec {
	struct page	*bv_page;
	unsigned int	bv_len;
	unsigned int	bv_offset;
};

struct bvec_iter {
	sector_t	bi_sector;	/* device offset, in 512-byte units */
	unsigned int	bi_size;	/* bytes REMAINING */
	unsigned int	bi_idx;		/* current index into bi_io_vec */
	unsigned int	bi_bvec_done;	/* bytes done in the current bvec */
};
```

The design decision worth understanding is the **separation of the vector from the iterator** (Kent Overstreet, ~2013). Before it, advancing through a bio destroyed information: splitting a bio meant allocating a new one and copying the relevant bio_vecs. After it, a "split" can share the same `bi_io_vec` array and differ only in the iterator:

```c
	/* Both bios point at the SAME bio_vec array. */
	struct bio *split = bio_split(bio, sectors, gfp, bs);
	/* split->bi_iter covers the first `sectors`;
	   bio->bi_iter has been advanced past them. */
```

This makes splitting cheap, and cheap splitting is what makes stacking (§T.8) viable. It also allows `bio_for_each_segment` to iterate with an explicit iterator rather than mutating the bio:

```c
	struct bio_vec bv;
	struct bvec_iter iter;

	bio_for_each_segment(bv, bio, iter) {
		/* bv is a copy; bio is untouched */
	}
```

**The three roles a bio plays:**

1. **A description** of work: what, where, how much.
2. **A unit of completion**: `bi_end_io` is called when done, with `bi_status`.
3. **A unit of composition**: bios chain (`bio_chain`) so a parent completes only when all children do — the mechanism behind `dm` splitting one I/O across several devices.

`__bi_remaining` implements the third: a chained child decrements the parent's count on completion, and the parent's `bi_end_io` fires at zero. Ch. 25's P7 (Completion) at the I/O layer.

### T.3 Operations and flags

```c
enum req_op {
	REQ_OP_READ		= 0,
	REQ_OP_WRITE		= 1,
	REQ_OP_FLUSH		= 2,
	REQ_OP_DISCARD		= 3,
	REQ_OP_SECURE_ERASE	= 5,
	REQ_OP_ZONE_APPEND	= 7,
	REQ_OP_WRITE_ZEROES	= 9,
	REQ_OP_ZONE_OPEN	= 10,
	REQ_OP_ZONE_CLOSE	= 11,
	REQ_OP_ZONE_FINISH	= 12,
	REQ_OP_ZONE_RESET	= 13,
	REQ_OP_ZONE_RESET_ALL	= 15,
	REQ_OP_DRV_IN		= 34,
	REQ_OP_DRV_OUT		= 35,
};

/* Flags, OR'd into bi_opf above REQ_OP_BITS */
#define REQ_FAILFAST_DEV, REQ_FAILFAST_TRANSPORT, REQ_FAILFAST_DRIVER
#define REQ_SYNC        /* someone is waiting: do not delay */
#define REQ_META        /* filesystem metadata */
#define REQ_PRIO        /* high priority */
#define REQ_NOMERGE     /* do not merge this */
#define REQ_IDLE        /* anticipate more from this submitter */
#define REQ_FUA         /* Ch. 61 T.3 */
#define REQ_PREFLUSH    /* Ch. 61 T.3 */
#define REQ_RAHEAD      /* readahead: droppable under pressure */
#define REQ_BACKGROUND  /* not latency-sensitive */
#define REQ_NOWAIT      /* fail rather than block */
#define REQ_POLLED      /* completion by polling, not interrupt */
#define REQ_SWAP, REQ_DRV, REQ_ATOMIC, REQ_NOUNMAP
```

These flags are the **only channel through which the filesystem communicates intent**. `REQ_SYNC` versus `REQ_BACKGROUND` is the difference between a read someone is blocked on and writeback. `REQ_RAHEAD` says "if this is expensive, skip it" — the scheduler and the driver may drop it under pressure. `REQ_META` lets a scheduler prioritise metadata, which matters enormously since metadata reads often gate many other operations.

`REQ_NOWAIT` is the io_uring-driven addition: "return `-EAGAIN` rather than block anywhere in the stack," which lets a non-blocking submitter stay non-blocking.

`REQ_OP_WRITE_ZEROES` and `REQ_OP_DISCARD` are worth distinguishing: discard says "I no longer need this data" (the device may return anything afterwards, unless `discard_zeroes_data`); write-zeroes says "make this read as zeros" and may be implemented as a real write or as a device hint. Filesystems need the latter for correctness (Ch. 55 §T.8's unwritten extents on devices without them).

### T.4 Why the single-queue design had to die

Until 3.13, Linux had one request queue per device, protected by one spinlock (`queue_lock`):

```c
struct request_queue {
	spinlock_t		__queue_lock;    /* ONE lock */
	struct list_head	queue_head;      /* ONE list */
	...
};
```

Every submission took the lock, searched for a merge candidate, inserted into the sorted list, and possibly ran the queue. Every completion took the lock.

This was fine when devices did 200 IOPS. It collapsed when they did 1,000,000.

**Three distinct bottlenecks**, and it is worth separating them because blk-mq's design addresses each differently:

**(a) Lock contention.** N cores, one spinlock. Classic (Ch. 16 §T.1). At high IOPS the lock's cache line ping-pongs and throughput *decreases* with more cores.

**(b) Cache-line bouncing.** Even without contention, the shared queue structures are written by every CPU. The request list head, the counters, the merge hash — all shared.

**(c) Completion on the wrong CPU.** A request submitted on CPU 0 might complete on CPU 12's interrupt. The completion touches data structures hot in CPU 0's cache, and the waking task must migrate or suffer remote-memory access.

Bjørling, Axboe, Nellans, and Bonnet measured this in 2013: **the single-queue block layer could not exceed roughly 1 million IOPS on any hardware**, and adding cores made it worse. Meanwhile NVMe devices were arriving that could do 1 million IOPS on their own and had hardware support for up to 64K queues.

The conclusion was not "optimise the locking" but "the architecture assumes one queue, and the hardware has many."

### T.5 blk-mq: two levels of queues

```
CPU0  CPU1  CPU2  CPU3  CPU4  CPU5  CPU6  CPU7
 │     │     │     │     │     │     │     │
 ▼     ▼     ▼     ▼     ▼     ▼     ▼     ▼
┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐
│SW ││SW ││SW ││SW ││SW ││SW ││SW ││SW │   software queues
│ctx││ctx││ctx││ctx││ctx││ctx││ctx││ctx│   ONE PER CPU, no locking
└─┬─┘└─┬─┘└─┬─┘└─┬─┘└─┬─┘└─┬─┘└─┬─┘└─┬─┘
  └──┬──┘    └──┬──┘    └──┬──┘    └──┬──┘
     ▼          ▼          ▼          ▼
  ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐
  │ HW  │   │ HW  │   │ HW  │   │ HW  │      hardware queues
  │ ctx │   │ ctx │   │ ctx │   │ ctx │      ONE PER DEVICE QUEUE
  └──┬──┘   └──┬──┘   └──┬──┘   └──┬──┘
     ▼         ▼         ▼         ▼
  ┌────────────────────────────────────┐
  │            device                  │
  └────────────────────────────────────┘
```

**Software queues (`blk_mq_ctx`)**: one per CPU. Submission from CPU N touches only CPU N's structure. **No lock is needed** because only one CPU uses it. Merging happens here against recent requests from the same CPU, where the cache is warm.

**Hardware queues (`blk_mq_hw_ctx`)**: one per device submission queue. For NVMe this maps to a real hardware queue pair with its own doorbell and its own MSI-X interrupt. For SATA (one hardware queue) all software queues map to one hardware queue.

The mapping is fixed at initialisation (`blk_mq_map_queues`), and modern drivers use `blk_mq_pci_map_queues`/`blk_mq_map_hw_queues` to align it with the device's interrupt affinity — so **the CPU that submits is the CPU that gets the completion interrupt**, addressing bottleneck (c) directly.

The result is that bottlenecks (a) and (b) largely disappear: the submission path writes only per-CPU data. Contention remains only where it is unavoidable — the tag bitmap (§T.6) and the device doorbell.

NVMe can have up to 65535 queues; in practice Linux allocates one per CPU (capped by the device). Separate queue *maps* exist for different purposes:

```c
enum hctx_type {
	HCTX_TYPE_DEFAULT,	/* reads and writes */
	HCTX_TYPE_READ,		/* if the device supports separate read queues */
	HCTX_TYPE_POLL,		/* polled completion, no interrupts */
	HCTX_MAX_TYPES,
};
```

`HCTX_TYPE_POLL` is how `io_uring` with `IORING_SETUP_IOPOLL` achieves sub-10 µs latency: dedicated queues with no interrupt at all, completions found by spinning.

### T.6 Tags: the unifying abstraction

A **tag** is an integer identifying an in-flight request. It serves four purposes simultaneously, which is why it is the right abstraction:

1. **The protocol needs one.** SCSI command queueing, NVMe command IDs, and AHCI NCQ slots all identify commands by a small integer.
2. **It is the request allocator.** Requests are preallocated as an array indexed by tag; getting a tag *is* allocating a request. No `kmalloc` in the I/O path.
3. **It bounds the queue depth.** The number of tags is the maximum in-flight count. Running out means the queue is full — which is the natural backpressure signal.
4. **It enables O(1) lookup** for timeout handling and completion: tag → request is an array index.

```c
struct blk_mq_tags {
	unsigned int		nr_tags;
	unsigned int		nr_reserved_tags;
	unsigned int		active_queues;
	struct sbitmap_queue	bitmap_tags;         /* the allocator */
	struct sbitmap_queue	breserved_tags;
	struct request		**rqs;               /* tag -> request */
	struct request		**static_rqs;
	struct list_head	page_list;
};
```

`sbitmap` (scalable bitmap) deserves attention: a naive shared bitmap with `find_first_zero_bit` has exactly the cache-line problem blk-mq was built to avoid. `sbitmap` splits the bitmap into **words spread across cache lines** and gives each CPU a hint of where to start searching, so different CPUs usually touch different words:

```c
struct sbitmap {
	unsigned int		depth;
	unsigned int		shift;
	unsigned int		map_nr;
	bool			round_robin;
	struct sbitmap_word	*map;        /* cache-line-aligned words */
	unsigned int __percpu	*alloc_hint; /* per-CPU starting point */
};
```

Plus `sbitmap_queue` adds **wait queues sharded by CPU** so that waiting for a free tag does not create a single wakeup point. This is a small, beautiful piece of engineering and a good example of "make the common case per-CPU, fall back to sharing only under contention."

**Shared tag sets**: multiple request queues (e.g. all LUNs behind one SCSI host adapter) can share one tag set, because the *hardware* limit is per-adapter, not per-LUN. `BLK_MQ_F_TAG_QUEUE_SHARED` handles the fairness problem this creates — one busy LUN must not starve others.

### T.7 The request lifecycle

```c
struct request {
	struct request_queue	*q;
	struct blk_mq_ctx	*mq_ctx;         /* the SW queue it came from */
	struct blk_mq_hw_ctx	*mq_hctx;        /* the HW queue it goes to */
	blk_opf_t		cmd_flags;
	req_flags_t		rq_flags;
	int			tag;
	int			internal_tag;    /* scheduler tag, Ch. 64 */
	unsigned int		timeout;
	unsigned int		__data_len;
	sector_t		__sector;
	struct bio		*bio;            /* the first bio */
	struct bio		*biotail;        /* the last, for merging */
	union {
		struct list_head queuelist;
		struct request	*rq_next;
	};
	struct gendisk		*rq_disk;
	unsigned long		start_time_ns;
	unsigned long		io_start_time_ns;
	...
	rq_end_io_fn		*end_io;
	void			*end_io_data;
};
```

The states:

```c
enum mq_rq_state {
	MQ_RQ_IDLE		= 0,	/* the tag is free */
	MQ_RQ_IN_FLIGHT		= 1,	/* issued to the driver */
	MQ_RQ_COMPLETE		= 2,	/* the driver said it is done */
};
```

Only three, and the transitions are guarded so that a **timeout and a completion cannot both act on the same request**. This matters: without it, a request that completes just as the timeout handler fires could be freed twice or completed twice. The state is changed with `cmpxchg` and the timeout handler backs off if it loses the race.

The submission path:

```
submit_bio(bio)
 └─ submit_bio_noacct()
     ├─ submit_bio_checks()           /* validate: size, alignment, ro, limits */
     ├─ blk_throtl_bio()              /* cgroup throttling (Ch. 64) */
     └─ disk->fops->submit_bio()  OR  blk_mq_submit_bio()
         │
         └─ blk_mq_submit_bio(bio)
             ├─ bio_split_to_limits()         /* T.9 */
             ├─ blk_attempt_plug_merge()      /* merge into the current plug */
             ├─ blk_mq_sched_bio_merge()      /* merge into the scheduler */
             ├─ blk_mq_get_new_requests()     /* get a TAG (T.6) */
             ├─ blk_mq_bio_to_request()       /* fill in the request */
             └─ either:
                 ├─ add to the plug list (batching, T.10)
                 ├─ blk_mq_try_issue_directly()   /* the fast path */
                 └─ blk_mq_sched_insert_request() /* to the scheduler */
```

The issue path:

```
blk_mq_run_hw_queue(hctx)
 └─ blk_mq_sched_dispatch_requests()
     ├─ collect requests from the scheduler or the SW queues
     └─ blk_mq_dispatch_rq_list()
         ├─ blk_mq_get_driver_tag()          /* the real hardware tag */
         ├─ blk_mq_start_request()           /* IN_FLIGHT; start the timer */
         └─ q->mq_ops->queue_rq(hctx, &bd)   /* THE DRIVER */
             └─ returns BLK_STS_OK / _RESOURCE / _DEV_RESOURCE / error
```

`BLK_STS_RESOURCE` versus `BLK_STS_DEV_RESOURCE` is a subtle and important distinction: the first means "I am out of a *shared* resource; retry later and something will free it"; the second means "this *device* is busy; do not retry until I complete something." Returning the wrong one causes either a busy-loop or a stall. The comment in `blk-mq.h` explaining this is worth reading.

Completion:

```
interrupt (or poll)
 └─ driver calls blk_mq_complete_request(rq)
     └─ blk_mq_complete_request_remote()
         ├─ same CPU / no steering  -> rq->q->mq_ops->complete(rq) directly
         ├─ different CPU, same node -> IPI to the submitting CPU
         └─ softirq -> BLOCK_SOFTIRQ
     └─ the driver's ->complete() -> blk_mq_end_request()
         ├─ blk_update_request()       /* advance the bio iterators */
         ├─ bio_endio() for each bio   /* -> the filesystem's callback */
         └─ blk_mq_free_request()      /* release the TAG */
```

The IPI-versus-local decision is controlled by `/sys/block/*/queue/rq_affinity`:

| Value | Behaviour |
|---|---|
| 0 | complete on whichever CPU took the interrupt |
| 1 | complete on a CPU in the same **group** as the submitter |
| 2 | complete on the **exact** submitting CPU |

`2` maximises cache locality and costs an IPI; `0` avoids the IPI and loses locality. For most workloads `1` is right, and it is the default. This is a genuine, measurable trade — Lab 63.5 measures it.

### T.8 Stacking: bio-based versus request-based

Two kinds of block driver:

| | **request-based** | **bio-based** |
|---|---|---|
| Entry point | `q->mq_ops->queue_rq()` | `disk->fops->submit_bio()` |
| Gets | merged, scheduled requests | raw bios |
| Examples | SCSI, NVMe, virtio-blk, loop | dm, MD, brd, zram, most stacking |
| Merging/scheduling | yes | **no** — it bypasses all of it |

A stacking driver receives a bio, possibly splits it, remaps it to one or more lower devices, and resubmits. It deliberately bypasses merging and scheduling because **the lower device's block layer will do that**, and doing it twice wastes work and can hurt (two schedulers reordering independently is worse than one).

The recursion problem: `submit_bio` from within a `submit_bio` handler could recurse arbitrarily deep — a stack of 10 dm targets is 10 stack frames plus everything each does. The solution:

```c
void submit_bio_noacct(struct bio *bio)
{
	...
	/* Are we already inside a submit_bio? Then just queue it. */
	if (current->bio_list) {
		bio_list_add(&current->bio_list[0], bio);
		return;
	}
	__submit_bio_noacct(bio);
}

static void __submit_bio_noacct(struct bio *bio)
{
	struct bio_list bio_list_on_stack[2];

	current->bio_list = bio_list_on_stack;
	do {
		...
		__submit_bio(bio);
		/* Process bios generated by THIS bio, breadth-first */
		bio_list_merge(&bio_list_on_stack[0], &lower);
	} while ((bio = bio_list_pop(&bio_list_on_stack[0])));
	current->bio_list = NULL;
}
```

**Recursion is converted to iteration via a per-task list.** Nested submissions are queued and processed at the top level, so stack depth is O(1) regardless of stacking depth. The two-list arrangement (`[0]` and `[1]`) maintains a breadth-first order that avoids a deadlock where a bio waits for another bio queued behind it.

This is a recurring kernel pattern: when recursion depth is unbounded and the stack is small, convert to an explicit worklist. Ch. 54 §T.7 did the same for symlinks.

**`bio_set` and mempools.** A stacking driver that splits a bio must allocate a new one — while under memory pressure, possibly *because of* memory pressure (swap). If the allocation fails, the I/O cannot proceed, and the memory cannot be freed because it needs the I/O. Every bio-based driver therefore uses a `bio_set` with a mempool reserve:

```c
	bioset_init(&md->bs, pool_size, front_pad, BIOSET_NEED_BVECS);
```

Ch. 22's `mempool` pattern, and the deadlock it prevents is the same one as FUSE's (Ch. 62 §T.7) and the AGFL's (Ch. 58 §1.2): **a reserve that breaks a circular dependency.**

### T.9 Queue limits and splitting

```c
struct queue_limits {
	unsigned int		max_hw_sectors;
	unsigned int		max_dev_sectors;
	unsigned int		chunk_sectors;
	unsigned int		max_sectors;         /* the effective limit */
	unsigned int		max_user_sectors;
	unsigned int		max_segment_size;
	unsigned int		physical_block_size;
	unsigned int		logical_block_size;
	unsigned int		alignment_offset;
	unsigned int		io_min;
	unsigned int		io_opt;
	unsigned int		max_discard_sectors;
	unsigned int		max_write_zeroes_sectors;
	unsigned int		max_zone_append_sectors;
	unsigned int		discard_granularity;
	unsigned short		max_segments;
	unsigned short		max_integrity_segments;
	unsigned char		misaligned;
	unsigned char		discard_misaligned;
	unsigned char		raid_partial_stripes_expensive;
	unsigned int		zoned;
	unsigned int		dma_alignment;
	...
};
```

A bio arriving from the filesystem may exceed any of these. `bio_split_to_limits()` checks and splits:

```c
struct bio *__bio_split_to_limits(struct bio *bio,
				  const struct queue_limits *lim,
				  unsigned int *nr_segs)
{
	switch (bio_op(bio)) {
	default:
		split = bio_split_rw(bio, lim, nr_segs, bs,
				     get_max_io_size(bio, lim) << SECTOR_SHIFT);
		...
	case REQ_OP_ZONE_APPEND:
		split = bio_split_zone_append(bio, lim, nr_segs);
		break;
	case REQ_OP_DISCARD:
	case REQ_OP_SECURE_ERASE:
		split = bio_split_discard(bio, lim, nr_segs, bs);
		break;
	case REQ_OP_WRITE_ZEROES:
		split = bio_split_write_zeroes(bio, lim, nr_segs, bs);
		break;
	}

	if (split) {
		split->bi_opf |= REQ_NOMERGE;   /* it was already sized */
		blkcg_bio_issue_init(split);
		bio_chain(split, bio);          /* T.2's composition */
		submit_bio_noacct(bio);         /* the remainder goes back */
		return split;
	}
	return bio;
}
```

`bio_chain` is what makes this correct: the caller's `bi_end_io` fires only when every split has completed. The caller never knows splitting happened.

**Stacked limits.** When device-mapper builds a device from several underlying ones, the limits must be the *intersection*:

```c
int blk_stack_limits(struct queue_limits *t, struct queue_limits *b,
		     sector_t start)
{
	t->max_sectors = min_not_zero(t->max_sectors, b->max_sectors);
	t->max_segments = min_not_zero(t->max_segments, b->max_segments);
	t->max_segment_size = min_not_zero(t->max_segment_size, b->max_segment_size);
	t->logical_block_size = max(t->logical_block_size, b->logical_block_size);
	t->physical_block_size = max(t->physical_block_size, b->physical_block_size);
	...
	/* Alignment is the hard part: offsets must be compatible */
	if (t->physical_block_size & (t->logical_block_size - 1)) {
		t->physical_block_size = t->logical_block_size;
		t->misaligned = 1;
		ret = -1;
	}
	...
}
```

The `misaligned` flag propagates up and is visible in `/sys/block/*/alignment_offset`. A misaligned stack (a partition starting at sector 63 on a 4K-physical drive, under LVM, under a filesystem) causes every write to become a read-modify-write on the device. This was extremely common on pre-2010 partition tables and is the origin of the "align partitions to 1 MiB" rule.

### T.10 Plugging: batching submissions

If a filesystem submits 256 bios for a 1 MiB write one at a time, each could be issued to the device separately, losing merge opportunities. **Plugging** batches them:

```c
	struct blk_plug plug;

	blk_start_plug(&plug);
	/* ... submit many bios ... */
	blk_finish_plug(&plug);      /* now flush them all */
```

```c
struct blk_plug {
	struct request *mq_list;	/* the batched requests */
	struct request *cached_rq;	/* a preallocated request */
	unsigned short nr_ios;
	unsigned short rq_count;
	bool multiple_queues;
	bool has_elevator;
	struct list_head cb_list;	/* callbacks for stacking drivers */
};
```

While plugged, new bios are merged against the plug list — **a per-task list, so no locking at all**. When the plug is flushed, the accumulated requests are issued in one batch, taking the hardware queue's lock once.

Three ways a plug flushes:
1. `blk_finish_plug()` — explicit.
2. `schedule()` — **if the task blocks, its plug is flushed automatically.** Otherwise a task that plugs and then sleeps would hold I/O indefinitely.
3. The plug is full (`BLK_MAX_REQUEST_COUNT`, 32).

Point 2 is essential and subtle: `io_schedule()` and `schedule()` call `blk_flush_plug()`. Without it, a thread that plugs, submits a read, and waits for it would deadlock — the read is in its own plug.

Callers that plug: `filemap_read`, `iomap_writepages`, `submit_bh` paths, `io_uring`'s submission loop, `md`'s raid write path. Getting a plug around a batch is one of the easiest real performance wins in filesystem code.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `block/blk-core.c` ★★★ | `submit_bio`, `submit_bio_noacct`, the recursion handling |
| `block/blk-mq.c` ★★★ | the whole of §T.5–T.7; ~5000 lines and the heart of the chapter |
| `block/blk-mq-tag.c` ★★★ | §T.6's tag allocation |
| `lib/sbitmap.c` ★★★ | the scalable bitmap |
| `block/blk-mq-sched.c` | the scheduler interface (Ch. 64) |
| `block/blk-merge.c` ★★★ | §T.9's splitting and all merging logic |
| `block/blk-settings.c` | `queue_limits`, `blk_stack_limits` |
| `block/bio.c` ★★★ | bio allocation, `bio_split`, `bio_chain`, `bio_endio` |
| `block/blk-flush.c` | Ch. 61 §T.3's flush machinery |
| `block/blk-timeout.c` | timeout handling and the state race |
| `block/blk-mq-cpumap.c`, `blk-mq-pci.c` | queue-to-CPU mapping |
| `block/genhd.c` | `gendisk`, partitions, sysfs |
| `include/linux/blk-mq.h` ★★★, `blk_types.h` ★★★, `bio.h` | the structures |
| `Documentation/block/` ★★★ | `blk-mq.rst`, `queue-sysfs.rst`, `biovecs.rst`, `stat.rst` |

### 1.2 `blk_mq_submit_bio`, annotated

```c
void blk_mq_submit_bio(struct bio *bio)
{
	struct request_queue *q = bdev_get_queue(bio->bi_bdev);
	struct blk_plug *plug = blk_mq_plug(bio);
	const int is_sync = op_is_sync(bio->bi_opf);
	struct blk_mq_hw_ctx *hctx;
	struct request *rq = NULL;
	unsigned int nr_segs = 1;
	blk_status_t ret;

	bio = blk_queue_bounce(bio, q);

	/* Did a previous submission leave us a cached request? */
	if (plug) {
		rq = rq_list_peek(&plug->cached_rq);
		if (rq && rq->q != q)
			rq = NULL;
	}
	if (rq) {
		if (unlikely(bio_may_exceed_limits(bio, &q->limits))) {
			bio = __bio_split_to_limits(bio, &q->limits, &nr_segs);
			if (!bio) return;
		}
		if (!bio_integrity_prep(bio)) return;
		if (blk_mq_attempt_bio_merge(q, bio, nr_segs)) return;
		...
		blk_mq_use_cached_rq(rq, plug, bio);
	} else {
		if (unlikely(bio_queue_enter(bio))) return;
		if (unlikely(bio_may_exceed_limits(bio, &q->limits))) {
			bio = __bio_split_to_limits(bio, &q->limits, &nr_segs);
			if (!bio) goto queue_exit;
		}
		if (!bio_integrity_prep(bio)) goto queue_exit;
		if (blk_mq_attempt_bio_merge(q, bio, nr_segs)) goto queue_exit;

		rq = blk_mq_get_new_requests(q, plug, bio, nr_segs);  /* TAG */
		if (unlikely(!rq)) goto queue_exit;
	}

	trace_block_getrq(bio);
	rq_qos_track(q, rq, bio);
	blk_mq_bio_to_request(rq, bio, nr_segs);

	ret = blk_crypto_rq_get_keyslot(rq);
	...

	if (op_is_flush(bio->bi_opf)) {
		blk_insert_flush(rq);          /* Ch. 61 §T.3 */
		return;
	}

	if (plug) {
		blk_add_rq_to_plug(plug, rq);  /* T.10: batch it */
		return;
	}

	hctx = rq->mq_hctx;
	if ((rq->rq_flags & RQF_USE_SCHED) ||
	    (hctx->dispatch_busy && (q->nr_hw_queues == 1 || !is_sync))) {
		blk_mq_insert_request(rq, 0);
		blk_mq_run_hw_queue(hctx, true);
	} else {
		blk_mq_run_dispatch_ops(q, blk_mq_try_issue_directly(hctx, rq));
	}
	return;
queue_exit:
	blk_queue_exit(q);
}
```

Every branch is a design decision from §T.5–T.10, and the function reads as a summary of the chapter.

Note the **cached request** path: a plug can hold a preallocated request, so a batch of submissions from one task avoids even the tag allocation for all but the first. Another layer of the "make the common case per-task" principle.

### 1.3 Tag allocation

```c
unsigned int blk_mq_get_tag(struct blk_mq_alloc_data *data)
{
	struct blk_mq_tags *tags = blk_mq_tags_from_data(data);
	struct sbitmap_queue *bt;
	struct sbq_wait_state *ws;
	DEFINE_SBQ_WAIT(wait);
	unsigned int tag_offset;
	int tag;
	...
	tag = __blk_mq_get_tag(data, bt);
	if (tag != BLK_MQ_NO_TAG)
		goto found_tag;

	if (data->flags & BLK_MQ_REQ_NOWAIT)
		return BLK_MQ_NO_TAG;          /* REQ_NOWAIT: do not block */

	ws = bt_wait_ptr(bt, data->hctx);   /* a SHARDED wait queue */
	do {
		struct sbitmap_queue *bt_prev;

		/* The queue may have been run; try again before sleeping. */
		blk_mq_run_hw_queue(data->hctx, false);

		tag = __blk_mq_get_tag(data, bt);
		if (tag != BLK_MQ_NO_TAG)
			break;

		sbitmap_prepare_to_wait(bt, ws, &wait, TASK_UNINTERRUPTIBLE);

		tag = __blk_mq_get_tag(data, bt);
		if (tag != BLK_MQ_NO_TAG)
			break;

		bt_prev = bt;
		io_schedule();                 /* <- flushes our plug (T.10) */
		...
	} while (1);
	...
found_tag:
	return tag + tag_offset;
}
```

The double-check around `prepare_to_wait` is Ch. 25's P3 (Wait/Wakeup) done correctly: check, prepare to sleep, check again, then sleep. Skipping the second check is the classic lost-wakeup bug.

`io_schedule()` rather than `schedule()` is important: it accounts the time as I/O wait and flushes the plug.

### 1.4 The timeout/completion race

```c
static bool blk_mq_req_expired(struct request *rq, unsigned long *next)
{
	unsigned long deadline;

	if (blk_mq_rq_state(rq) != MQ_RQ_IN_FLIGHT)
		return false;
	if (rq->rq_flags & RQF_TIMED_OUT)
		return false;

	deadline = READ_ONCE(rq->deadline);
	if (time_after_eq(jiffies, deadline))
		return true;
	...
}

void blk_mq_complete_request(struct request *rq)
{
	if (!blk_mq_complete_request_remote(rq))
		rq->q->mq_ops->complete(rq);
}

bool blk_mq_complete_request_remote(struct request *rq)
{
	WRITE_ONCE(rq->state, MQ_RQ_COMPLETE);
	...
}

/* The timeout handler must win or lose atomically: */
static bool blk_mq_mark_complete(struct request *rq)
{
	return cmpxchg(&rq->state, MQ_RQ_IN_FLIGHT, MQ_RQ_COMPLETE) ==
			MQ_RQ_IN_FLIGHT;
}
```

**A request can be completed exactly once**, and the `cmpxchg` decides whether the completion path or the timeout path gets to do it. Getting this wrong — and pre-blk-mq drivers frequently did — produces double-free and use-after-free bugs that appear only under load.

### 1.5 `bio_endio` and chaining

```c
void bio_endio(struct bio *bio)
{
again:
	if (!bio_remaining_done(bio))
		return;                      /* chained children still outstanding */
	if (!bio_integrity_endio(bio))
		return;

	blk_zone_bio_endio(bio);
	rq_qos_done_bio(bio);
	...
	/* A chained bio: complete the PARENT, iteratively not recursively */
	if (bio->bi_end_io == bio_chain_endio) {
		bio = __bio_chain_endio(bio);
		goto again;
	}

	blk_throtl_bio_endio(bio);
	bio_uninit(bio);
	if (bio->bi_end_io)
		bio->bi_end_io(bio);
}

static struct bio *__bio_chain_endio(struct bio *bio)
{
	struct bio *parent = bio->bi_private;

	if (bio->bi_status && !parent->bi_status)
		parent->bi_status = bio->bi_status;   /* propagate the error */
	bio_put(bio);
	return parent;
}
```

Note the `goto again` rather than a recursive call: a deeply chained bio (many splits of splits) would otherwise blow the stack. Same principle as §T.8.

### 1.6 Observability

| Where | What |
|---|---|
| `/sys/block/DEV/queue/` ★★★ | every limit and tunable |
| `/sys/kernel/debug/block/DEV/` ★★★ | **per-hctx state, tags, dispatch lists** |
| `/sys/kernel/debug/block/DEV/hctx*/tags` | tag bitmap state |
| `/sys/kernel/debug/block/DEV/hctx*/dispatch` | requests stuck in dispatch |
| `/sys/kernel/debug/block/DEV/hctx*/busy` | in-flight requests, with details |
| `/proc/diskstats` ★★★ | the numbers behind `iostat` |
| `iostat -xz 1` ★★★ | per-device latency, queue depth, utilisation |
| `blktrace`/`blkparse` ★★★ | **every event in a request's life** |
| `btt` | blktrace analysis: time in each stage |
| `trace-cmd record -e block:\*` ★★★ | the same events, via ftrace |
| `biolatency`, `biosnoop`, `biotop` (bcc) ★★★ | |
| `/sys/block/DEV/queue/rq_affinity` | §T.7's completion steering |
| `/sys/block/DEV/mq/*/` | per-hardware-queue CPU lists |

The `block:` tracepoints are the most useful set in the kernel for this layer:

```
block_bio_queue     -> Q: a bio entered the block layer
block_bio_frontmerge / block_bio_backmerge -> M: merged
block_getrq         -> G: a request was allocated (a tag was taken)
block_plug / block_unplug -> P/U
block_rq_insert     -> I: inserted into the scheduler
block_rq_issue      -> D: dispatched to the driver
block_rq_complete   -> C: completed
block_split         -> X
block_bio_remap / block_rq_remap -> A: remapped by a stacking driver
```

`Q → G → I → D → C` is the canonical lifecycle, and `btt` reports the time in each interval. **Q2D is queueing time; D2C is device time.** That single split answers most "is the device slow or is the queue deep?" questions.

---

## 2. Practice

### Lab 63.1 — Trace one I/O end to end

```sh
sudo apt install -y blktrace fio bpfcc-tools
sudo modprobe scsi_debug dev_size_mb=1024 delay=1
DEV=$(lsblk -ndo NAME,MODEL | awk '/scsi_debug/{print "/dev/"$1}')
DN=$(basename $DEV)

sudo blktrace -d $DEV -o - | blkparse -i - &
BT=$!
sleep 1
sudo dd if=$DEV of=/dev/null bs=4k count=1 iflag=direct 2>/dev/null
sleep 1
sudo kill $BT
```

```
  8,16   3        1     0.000000000  4242  Q   R 0 + 8 [dd]
  8,16   3        2     0.000002345  4242  G   R 0 + 8 [dd]
  8,16   3        3     0.000003123  4242  P   N [dd]
  8,16   3        4     0.000004567  4242  I   R 0 + 8 [dd]
  8,16   3        5     0.000005012  4242  U   N [dd] 1
  8,16   3        6     0.000005890  4242  D   R 0 + 8 [dd]
  8,16   1        7     0.001234567     0  C   R 0 + 8 [0]
```

**Q → G → P → I → U → D → C.** Every letter is a stage from §T.7. Map each to the function that emitted it.

The same via ftrace:

```sh
sudo trace-cmd record -e block:\* -- \
  sudo dd if=$DEV of=/dev/null bs=4k count=1 iflag=direct 2>/dev/null
sudo trace-cmd report
```

Now a larger I/O, to see splitting and merging:

```sh
sudo blktrace -d $DEV -o /tmp/bt &
sudo dd if=/dev/zero of=$DEV bs=1M count=10 oflag=direct 2>/dev/null
sudo pkill blktrace; sleep 1
blkparse -i /tmp/bt | head -40
blkparse -i /tmp/bt | awk '{print $6}' | sort | uniq -c | sort -rn
```

Timing analysis:

```sh
sudo blktrace -d $DEV -o /tmp/bt2 &
sudo fio --name=t --filename=$DEV --direct=1 --rw=randread --bs=4k \
         --iodepth=32 --ioengine=libaio --runtime=10 --time_based \
         --numjobs=4 > /dev/null 2>&1
sudo pkill blktrace; sleep 1
btt -i /tmp/bt2 | head -40
```

```
            ALL           MIN           AVG           MAX           N
Q2Q       0.000000012   0.000012345   0.000234567      160000
Q2G       0.000000234   0.000001234   0.000034567      160000
G2I       0.000000123   0.000000890   0.000012345      160000
I2D       0.000000456   0.000002345   0.000045678      160000
D2C       0.000891234   0.001123456   0.004567890      160000   <- the DEVICE
Q2C       0.000892345   0.001129123   0.004601234      160000   <- TOTAL
```

**D2C is device time; Q2C − D2C is everything Linux added.** On a fast device the kernel overhead should be a small fraction; if it is not, that is where to look.

---

### Lab 63.2 — See the queue architecture

```sh
ls /sys/block/$DN/mq/
for q in /sys/block/$DN/mq/*/; do
  echo "=== $(basename $q) ==="
  echo -n "  cpus: "; cat $q/cpu_list 2>/dev/null
  echo -n "  nr_tags: "; cat $q/nr_tags 2>/dev/null
  echo -n "  nr_reserved_tags: "; cat $q/nr_reserved_tags 2>/dev/null
done
```

An NVMe device (if present) shows the real thing:

```sh
for d in /sys/block/nvme*n1; do
  [ -d "$d" ] || continue
  echo "=== $(basename $d) ==="
  ls $d/mq/ | wc -l                         # one hw queue per CPU
  cat $d/queue/nr_requests
  for q in $d/mq/*/; do
    printf "  hctx%-3s cpus: %s\n" $(basename $q) "$(cat $q/cpu_list)"
  done | head -8
done
nproc
```

The debugfs view is where the real state lives:

```sh
sudo mount -t debugfs none /sys/kernel/debug 2>/dev/null
sudo ls /sys/kernel/debug/block/$DN/
sudo ls /sys/kernel/debug/block/$DN/hctx0/

sudo cat /sys/kernel/debug/block/$DN/hctx0/type
sudo cat /sys/kernel/debug/block/$DN/hctx0/state
sudo cat /sys/kernel/debug/block/$DN/hctx0/flags
sudo cat /sys/kernel/debug/block/$DN/hctx0/tags
```

```
nr_tags=64
nr_reserved_tags=0
active_queues=0
bitmap_tags:
depth=64
busy=17
cleared=3
bits_per_word=64
map_nr=1
alloc_hint={12, 33, 5, 41, ...}      <- T.6's per-CPU hints
wake_batch=8
wake_index=0
ws_active=0
```

Watch tags in flight:

```sh
sudo fio --name=t --filename=$DEV --direct=1 --rw=randread --bs=4k \
         --iodepth=32 --ioengine=libaio --runtime=30 --time_based \
         --numjobs=4 > /dev/null 2>&1 &
FIO=$!
for i in $(seq 1 10); do
  echo -n "busy: "
  sudo grep -h '^busy=' /sys/kernel/debug/block/$DN/hctx*/tags | \
    awk -F= '{s+=$2} END {print s}'
  sudo cat /sys/kernel/debug/block/$DN/hctx0/busy 2>/dev/null | wc -l
  sleep 1
done
kill $FIO 2>/dev/null
```

Tag exhaustion:

```sh
cat /sys/block/$DN/queue/nr_requests
echo 4 | sudo tee /sys/block/$DN/queue/nr_requests       # tiny

sudo bpftrace -e '
kprobe:blk_mq_get_tag { @get = count(); }
kretprobe:blk_mq_get_tag /retval == -1/ { @failed = count(); }
kprobe:io_schedule { @waits = count(); }
interval:s:3 { print(@get); print(@failed); print(@waits);
               clear(@get); clear(@failed); clear(@waits); }' &

sudo fio --name=t --filename=$DEV --direct=1 --rw=randread --bs=4k \
         --iodepth=64 --ioengine=libaio --runtime=10 --time_based \
         --numjobs=8 2>/dev/null | grep -E 'IOPS|lat.*avg'

echo 256 | sudo tee /sys/block/$DN/queue/nr_requests
sudo fio --name=t --filename=$DEV --direct=1 --rw=randread --bs=4k \
         --iodepth=64 --ioengine=libaio --runtime=10 --time_based \
         --numjobs=8 2>/dev/null | grep -E 'IOPS|lat.*avg'
```

---

### Lab 63.3 — Merging and plugging

```sh
sudo mkfs.ext4 -qF $DEV && sudo mkdir -p /mnt/b && sudo mount $DEV /mnt/b

sudo blktrace -d $DEV -o /tmp/merge &
sudo dd if=/dev/zero of=/mnt/b/seq bs=4k count=2000 conv=fsync 2>/dev/null
sudo pkill blktrace; sleep 1

blkparse -i /tmp/merge | awk '{print $6}' | sort | uniq -c | sort -rn
# Many Q, far fewer D: merging worked.
echo -n "bios queued: "; blkparse -i /tmp/merge | grep -c ' Q '
echo -n "merges:      "; blkparse -i /tmp/merge | grep -cE ' M '
echo -n "dispatched:  "; blkparse -i /tmp/merge | grep -c ' D '
```

Disable merging:

```sh
cat /sys/block/$DN/queue/nomerges      # 0=all, 1=simple only, 2=none
for nm in 0 1 2; do
  echo $nm | sudo tee /sys/block/$DN/queue/nomerges > /dev/null
  sudo blktrace -d $DEV -o /tmp/nm$nm &
  sudo dd if=/dev/zero of=/mnt/b/s$nm bs=4k count=2000 conv=fsync 2>/dev/null
  sudo pkill blktrace; sleep 1
  echo -n "nomerges=$nm: Q=$(blkparse -i /tmp/nm$nm | grep -c ' Q ') "
  echo "D=$(blkparse -i /tmp/nm$nm | grep -c ' D ')"
done
echo 0 | sudo tee /sys/block/$DN/queue/nomerges > /dev/null
```

Request sizes:

```sh
sudo bpftrace -e '
tracepoint:block:block_bio_queue  { @bio_kb = hist(args->nr_sector / 2); }
tracepoint:block:block_rq_issue   { @req_kb = hist(args->nr_sector / 2); }
interval:s:10 { print(@bio_kb); print(@req_kb); exit(); }' &
sudo dd if=/dev/zero of=/mnt/b/big bs=1M count=200 conv=fsync 2>/dev/null
wait
```

Bios are 4 KiB–1 MiB; requests are much larger. **That difference is merging.**

Plugging:

```sh
sudo trace-cmd record -e block:block_plug -e block:block_unplug \
                      -e block:block_rq_issue -- \
  sudo dd if=/dev/zero of=/mnt/b/plug bs=1M count=50 conv=fsync 2>/dev/null
sudo trace-cmd report | grep -E 'plug|unplug' | head -20
sudo trace-cmd report | grep unplug | grep -oP 'nr_rq=\K\d+' | \
  awk '{s+=$1; n++} END {print "avg requests per unplug:", s/n}'
```

Prove that blocking flushes the plug:

```sh
sudo bpftrace -e '
kprobe:blk_flush_plug {
	@from[kstack(3)] = count();
}
interval:s:10 { print(@from); exit(); }' &
sudo dd if=/dev/zero of=/mnt/b/p2 bs=4k count=5000 conv=fsync 2>/dev/null
wait
# You will see both blk_finish_plug and schedule/io_schedule paths (T.10).
```

Maximum request size:

```sh
cat /sys/block/$DN/queue/max_sectors_kb
cat /sys/block/$DN/queue/max_hw_sectors_kb
cat /sys/block/$DN/queue/max_segments
cat /sys/block/$DN/queue/max_segment_size

for sz in 64 256 1024; do
  echo $sz | sudo tee /sys/block/$DN/queue/max_sectors_kb > /dev/null 2>&1 || continue
  echo -n "max_sectors_kb=$sz: "
  sudo dd if=/dev/zero of=/mnt/b/x bs=1M count=200 oflag=direct 2>&1 | tail -1
done
```

---

### Lab 63.4 — Splitting

```sh
# A small max_sectors forces splitting
echo 64 | sudo tee /sys/block/$DN/queue/max_sectors_kb

sudo blktrace -d $DEV -o /tmp/split &
sudo dd if=/dev/zero of=$DEV bs=4M count=5 oflag=direct 2>/dev/null
sudo pkill blktrace; sleep 1

blkparse -i /tmp/split | grep ' X ' | head -10     # X = split
echo -n "splits: "; blkparse -i /tmp/split | grep -c ' X '
```

Watch it in the kernel:

```sh
sudo bpftrace -e '
kprobe:__bio_split_to_limits { @splits = count(); }
tracepoint:block:block_split {
	printf("split at sector %llu -> new sector %llu\n",
	       args->sector, args->new_sector);
}
interval:s:5 { print(@splits); clear(@splits); }' &
sudo dd if=/dev/zero of=$DEV bs=4M count=3 oflag=direct 2>/dev/null
```

`bio_chain` in action:

```sh
sudo bpftrace -e '
kprobe:bio_chain { @chain = count(); }
kprobe:bio_endio { @endio = count(); }
kprobe:__bio_chain_endio { @chain_endio = count(); }
interval:s:5 { print(@chain); print(@endio); print(@chain_endio);
               clear(@chain); clear(@endio); clear(@chain_endio); }' &
sudo dd if=/dev/zero of=$DEV bs=4M count=5 oflag=direct 2>/dev/null
```

Stacked limits:

```sh
echo 512 | sudo tee /sys/block/$DN/queue/max_sectors_kb

# Build a stack and watch the limits intersect
SZ=$(sudo blockdev --getsz $DEV)
sudo dmsetup create l1 --table "0 $SZ linear $DEV 0"
sudo dmsetup create l2 --table "0 $SZ linear /dev/mapper/l1 0"

for d in $DN dm-0 dm-1; do
  echo "=== $d ==="
  for f in max_sectors_kb max_segments logical_block_size physical_block_size \
           max_hw_sectors_kb optimal_io_size minimum_io_size; do
    printf "  %-22s %s\n" $f "$(cat /sys/block/$d/queue/$f 2>/dev/null)"
  done
done
```

Misalignment:

```sh
# A deliberately misaligned linear target
sudo dmsetup create bad --table "0 $((SZ-1)) linear $DEV 1"
cat /sys/block/dm-2/alignment_offset 2>/dev/null
cat /sys/block/dm-2/queue/optimal_io_size

sudo dmsetup remove bad l2 l1 2>/dev/null
```

Real-world alignment:

```sh
lsblk -o NAME,ALIGNMENT,PHY-SEC,LOG-SEC,MIN-IO,OPT-IO
for d in /sys/block/*/; do
  AO=$(cat $d/alignment_offset 2>/dev/null)
  [ "$AO" != "0" ] && [ -n "$AO" ] && echo "MISALIGNED: $(basename $d) offset=$AO"
done
```

---

### Lab 63.5 — Completion steering and scalability

```sh
cat /sys/block/$DN/queue/rq_affinity

for ra in 0 1 2; do
  echo $ra | sudo tee /sys/block/$DN/queue/rq_affinity > /dev/null
  echo "=== rq_affinity=$ra ==="
  sudo fio --name=t --filename=$DEV --direct=1 --rw=randread --bs=4k \
           --iodepth=32 --ioengine=libaio --runtime=10 --time_based \
           --numjobs=$(nproc) --group_reporting 2>/dev/null | \
    grep -E 'IOPS|clat.*avg|99.00th'
done
```

Where do completions land?

```sh
sudo bpftrace -e '
tracepoint:block:block_rq_issue    { @submit_cpu[cpu] = count(); }
tracepoint:block:block_rq_complete { @complete_cpu[cpu] = count(); }
interval:s:10 { print(@submit_cpu); print(@complete_cpu); exit(); }' &
sudo fio --name=t --filename=$DEV --direct=1 --rw=randread --bs=4k \
         --iodepth=32 --ioengine=libaio --runtime=10 --time_based \
         --numjobs=$(nproc) > /dev/null 2>&1
wait
```

IPI traffic:

```sh
grep -E 'CAL|TLB|LOC' /proc/interrupts | head -3
BEFORE=$(grep '^CAL' /proc/interrupts | awk '{s=0; for(i=2;i<=NF-2;i++) s+=$i; print s}')
sudo fio --name=t --filename=$DEV --direct=1 --rw=randread --bs=4k \
         --iodepth=32 --ioengine=libaio --runtime=10 --time_based \
         --numjobs=$(nproc) > /dev/null 2>&1
AFTER=$(grep '^CAL' /proc/interrupts | awk '{s=0; for(i=2;i<=NF-2;i++) s+=$i; print s}')
echo "function-call IPIs during the run: $((AFTER-BEFORE))"
```

`rq_affinity=2` should show noticeably more.

Scaling with core count:

```sh
for n in 1 2 4 8 $(nproc); do
  echo -n "$n jobs: "
  sudo fio --name=t --filename=$DEV --direct=1 --rw=randread --bs=4k \
           --iodepth=32 --ioengine=libaio --runtime=8 --time_based \
           --numjobs=$n --group_reporting 2>/dev/null | \
    grep -oP 'IOPS=\K[0-9.]+[km]?' | head -1
done
```

Where is the time going?

```sh
sudo perf record -a -g -- sudo fio --name=t --filename=$DEV --direct=1 \
  --rw=randread --bs=4k --iodepth=32 --ioengine=libaio --runtime=10 \
  --time_based --numjobs=$(nproc) > /dev/null 2>&1
sudo perf report --stdio --sort symbol 2>/dev/null | head -30
sudo perf report --stdio 2>/dev/null | grep -iE 'lock|spin|blk_mq|sbitmap' | head
```

On a properly working blk-mq stack, lock time should be a small fraction. Compare with a synthetic single-queue setup if you can build one.

---

### Lab 63.6 — Polling: latency without interrupts

```sh
# Requires a device supporting polled queues -- NVMe, or null_blk
sudo modprobe null_blk nr_devices=0
cd /sys/kernel/config/nullb
sudo mkdir poll0 && cd poll0
echo 1024 | sudo tee size > /dev/null
echo 1 | sudo tee memory_backed > /dev/null
echo 4 | sudo tee poll_queues > /dev/null
echo 1 | sudo tee submit_queues > /dev/null
echo 1 | sudo tee power > /dev/null
cd /

PDEV=/dev/nullb0
ls /sys/block/nullb0/mq/
for q in /sys/block/nullb0/mq/*/; do
  echo "$(basename $q): $(sudo cat /sys/kernel/debug/block/nullb0/$(basename $q)/type 2>/dev/null)"
done
# Some queues have type "poll" (T.5)
```

```sh
# Interrupt-driven
sudo fio --name=irq --filename=$PDEV --direct=1 --rw=randread --bs=4k \
         --iodepth=1 --ioengine=io_uring --runtime=10 --time_based 2>/dev/null | \
  grep -E 'clat.*avg|99.00th'

# Polled
sudo fio --name=poll --filename=$PDEV --direct=1 --rw=randread --bs=4k \
         --iodepth=1 --ioengine=io_uring --hipri=1 --runtime=10 --time_based 2>/dev/null | \
  grep -E 'clat.*avg|99.00th'
```

Polling trades CPU for latency:

```sh
for mode in "" "--hipri=1"; do
  echo "=== ${mode:-interrupt} ==="
  sudo fio --name=t --filename=$PDEV --direct=1 --rw=randread --bs=4k \
           --iodepth=1 --ioengine=io_uring $mode --runtime=10 --time_based 2>/dev/null | \
    grep -E 'clat.*avg|cpu.*usr'
done
```

Hybrid polling (sleep for part of the expected latency, then poll):

```sh
cat /sys/block/nvme0n1/queue/io_poll 2>/dev/null
cat /sys/block/nvme0n1/queue/io_poll_delay 2>/dev/null
# -1 = classic (spin the whole time), 0 = hybrid (adaptive), >0 = fixed delay
```

---

### Lab 63.7 — Write a bio-based driver

```c
// SPDX-License-Identifier: GPL-2.0
/* sbd.c -- a RAM-backed bio-based block driver. */
#include <linux/bio.h>
#include <linux/blkdev.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/vmalloc.h>

#define SBD_MINORS	16
#define SBD_SECTORS	(64 * 1024)		/* 32 MiB */
#define KERNEL_SECTOR_SIZE 512

static int sbd_major;
static struct gendisk *sbd_disk;
static u8 *sbd_data;

static void sbd_submit_bio(struct bio *bio)
{
	struct bio_vec bvec;
	struct bvec_iter iter;
	sector_t sector = bio->bi_iter.bi_sector;

	if (bio_end_sector(bio) > SBD_SECTORS) {
		bio_io_error(bio);
		return;
	}

	/* T.2's iterator: `bio` is not mutated */
	bio_for_each_segment(bvec, bio, iter) {
		void *kaddr = kmap_local_page(bvec.bv_page) + bvec.bv_offset;
		u8 *store = sbd_data + (sector << SECTOR_SHIFT);

		switch (bio_op(bio)) {
		case REQ_OP_READ:
			memcpy(kaddr, store, bvec.bv_len);
			break;
		case REQ_OP_WRITE:
			memcpy(store, kaddr, bvec.bv_len);
			break;
		default:
			kunmap_local(kaddr - bvec.bv_offset);
			bio->bi_status = BLK_STS_NOTSUPP;
			bio_endio(bio);
			return;
		}
		kunmap_local(kaddr - bvec.bv_offset);
		sector += bvec.bv_len >> SECTOR_SHIFT;
	}

	bio_endio(bio);
}

static const struct block_device_operations sbd_fops = {
	.owner		= THIS_MODULE,
	.submit_bio	= sbd_submit_bio,
};

static int __init sbd_init(void)
{
	struct queue_limits lim = {
		.logical_block_size	= 512,
		.physical_block_size	= 4096,
		.io_min			= 4096,
		.io_opt			= 65536,
		.max_hw_sectors		= 256,
	};
	int ret;

	sbd_data = vzalloc(SBD_SECTORS * KERNEL_SECTOR_SIZE);
	if (!sbd_data)
		return -ENOMEM;

	sbd_major = register_blkdev(0, "sbd");
	if (sbd_major < 0) { ret = sbd_major; goto out_free; }

	sbd_disk = blk_alloc_disk(&lim, NUMA_NO_NODE);
	if (IS_ERR(sbd_disk)) { ret = PTR_ERR(sbd_disk); goto out_unreg; }

	sbd_disk->major		= sbd_major;
	sbd_disk->first_minor	= 0;
	sbd_disk->minors	= SBD_MINORS;
	sbd_disk->fops		= &sbd_fops;
	sbd_disk->private_data	= NULL;
	snprintf(sbd_disk->disk_name, DISK_NAME_LEN, "sbd0");
	set_capacity(sbd_disk, SBD_SECTORS);

	ret = add_disk(sbd_disk);
	if (ret) goto out_cleanup;

	pr_info("sbd: /dev/sbd0, %u sectors\n", SBD_SECTORS);
	return 0;

out_cleanup:
	put_disk(sbd_disk);
out_unreg:
	unregister_blkdev(sbd_major, "sbd");
out_free:
	vfree(sbd_data);
	return ret;
}

static void __exit sbd_exit(void)
{
	del_gendisk(sbd_disk);
	put_disk(sbd_disk);
	unregister_blkdev(sbd_major, "sbd");
	vfree(sbd_data);
}

module_init(sbd_init);
module_exit(sbd_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Simple bio-based block device");
```

```sh
cat > Makefile <<'EOF'
obj-m += sbd.o
all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
EOF
make && sudo insmod sbd.ko
lsblk /dev/sbd0
cat /sys/block/sbd0/queue/{logical_block_size,physical_block_size,max_sectors_kb}

sudo mkfs.ext4 -qF /dev/sbd0
sudo mkdir -p /mnt/sbd && sudo mount /dev/sbd0 /mnt/sbd
sudo cp -r /usr/include /mnt/sbd/ 2>/dev/null
df -h /mnt/sbd
sudo umount /mnt/sbd
```

Prove it is bio-based, not request-based:

```sh
ls /sys/block/sbd0/mq/ 2>&1          # no mq directory: no request queue
sudo blktrace -d /dev/sbd0 -o - 2>/dev/null | blkparse -i - &
sudo dd if=/dev/sbd0 of=/dev/null bs=4k count=10 iflag=direct 2>/dev/null
sleep 1; sudo pkill blktrace
# Q events but no G/I/D: it bypasses the request layer (T.8).
```

Now convert it to blk-mq as an exercise — implement `queue_rq`, `blk_mq_alloc_tag_set`, and `blk_mq_alloc_disk` — and compare the trace. Chapter 65 does this properly.

Clean up:

```sh
sudo rmmod sbd
```

---

### Lab 63.8 — Queue limits, in full

```sh
for f in /sys/block/$DN/queue/*; do
  [ -f "$f" ] && printf "%-32s %s\n" "$(basename $f)" "$(cat $f 2>/dev/null | head -c 60)"
done
```

The ones that matter:

```sh
cat <<'EOF'
max_sectors_kb        the largest single request (tunable, <= max_hw_sectors_kb)
max_hw_sectors_kb     the device's hard limit
max_segments          max scatter-gather entries
max_segment_size      max bytes per SG entry
logical_block_size    the addressing unit (512 or 4096)
physical_block_size   the actual media unit -- writes smaller than this cost RMW
minimum_io_size       = physical_block_size; the smallest efficient I/O
optimal_io_size       the RAID stripe width, if known
nr_requests           queue depth per hw queue
rotational            1 = seek costs matter
nomerges              0/1/2
rq_affinity           completion steering (T.7)
add_random            contribute to the entropy pool (set 0 for SSDs)
read_ahead_kb         the readahead window (Ch. 52)
write_cache           "write back" or "write through" (Ch. 61 T.6)
fua                   does the device support FUA
discard_granularity   the smallest meaningful discard
discard_max_bytes     the largest discard
zoned                 none / host-managed / host-aware (Ch. 60 T.5)
io_poll               polled completion support
EOF
```

Logical versus physical block size:

```sh
lsblk -o NAME,LOG-SEC,PHY-SEC,MIN-IO,OPT-IO,ALIGNMENT

# Writes smaller than physical_block_size cost a read-modify-write
sudo modprobe -r scsi_debug
sudo modprobe scsi_debug dev_size_mb=512 sector_size=512 physblk_exp=3
DEV2=$(lsblk -ndo NAME,MODEL | awk '/scsi_debug/{print "/dev/"$1}')
cat /sys/block/$(basename $DEV2)/queue/{logical_block_size,physical_block_size}

for bs in 512 4k; do
  echo -n "bs=$bs: "
  sudo dd if=/dev/zero of=$DEV2 bs=$bs count=10000 oflag=direct 2>&1 | tail -1
done
```

Discard:

```sh
sudo modprobe -r scsi_debug
sudo modprobe scsi_debug dev_size_mb=1024 lbpu=1 lbpws=1
DEV3=$(lsblk -ndo NAME,MODEL | awk '/scsi_debug/{print "/dev/"$1}')
cat /sys/block/$(basename $DEV3)/queue/discard_{granularity,max_bytes,zeroes_data} 2>/dev/null

sudo blktrace -d $DEV3 -o /tmp/disc &
sudo blkdiscard -o 0 -l 100M $DEV3
sudo pkill blktrace; sleep 1
blkparse -i /tmp/disc | grep -i 'D  *D' | head

# The filesystem's discard paths
sudo mkfs.ext4 -qF $DEV3 && sudo mount -o discard $DEV3 /mnt/b
sudo dd if=/dev/zero of=/mnt/b/x bs=1M count=100 2>/dev/null; sync
sudo trace-cmd record -e block:block_rq_issue -- sudo rm /mnt/b/x
sudo trace-cmd report | grep -i discard | head
# vs fstrim (batched, usually better than mount -o discard)
sudo fstrim -v /mnt/b
```

---

## 3. Mastery drills

1. §T.1 lists nine block-layer jobs. For each, construct the argument that it cannot live in the filesystem, or in the driver, or in both.

2. Explain the separation of `bio_vec` from `bvec_iter`. Then show, concretely, what `bio_split` had to do before and after, and why the difference matters for stacking.

3. `bio_chain` and `__bi_remaining` implement composition. Write the completion sequence for a bio split three ways, one of which fails, and show how the error reaches the original submitter.

4. Enumerate the flags in §T.3 that carry *intent* rather than *mechanism*. For each, state what a scheduler or driver may legitimately do with it.

5. §T.4's three bottlenecks are distinct. For each, state whether blk-mq eliminates, reduces, or merely relocates it, and identify the remaining shared state in the submission path.

6. `sbitmap` uses per-CPU hints and sharded wait queues. Construct the workload where a naive shared bitmap performs identically, and the one where it is catastrophically worse.

7. Tags serve four purposes (§T.6). Design a system that separates them and state what you lose.

8. The timeout/completion race is resolved by `cmpxchg` on `rq->state`. Write both interleavings and show that exactly one path completes the request in each.

9. `BLK_STS_RESOURCE` versus `BLK_STS_DEV_RESOURCE`: construct the busy-loop that results from returning the first when you meant the second, and the stall from the reverse.

10. Explain why `submit_bio` recursion is converted to iteration, compute the stack depth a 10-deep dm stack would otherwise need, and name two other places in the kernel with the same conversion.

11. Every bio-based stacking driver needs a `bio_set` with a mempool. Construct the deadlock it prevents, and identify the same pattern in three other chapters of Part 3.

12. Plugs are flushed on `schedule()`. Construct the deadlock that occurs without this, and explain why the flush must happen in the scheduler rather than in the I/O wait path.

13. You are told that a NVMe device advertising 1M IOPS delivers 200K under your workload. Give the ordered diagnostic procedure using this chapter's tools, and the six most likely causes.

---

## 4. Further reading

**Kernel documentation**

- `Documentation/block/blk-mq.rst` ★★★ — §T.5–T.7 in the kernel's own words.
- `Documentation/block/queue-sysfs.rst` ★★★ — **every tunable, with its meaning.** Read it all.
- `Documentation/block/biovecs.rst` ★★★ — §T.2's iterator design, by Kent Overstreet.
- `Documentation/block/biodoc.rst` — older but still the best overview of the layer's responsibilities.
- `Documentation/block/writeback_cache_control.rst` ★★★ — Ch. 61 §T.3.
- `Documentation/block/stat.rst` — `/sys/block/*/stat` field by field.
- `Documentation/block/blktrace.rst` — the event letters.
- `Documentation/block/data-integrity.rst` — T10 PI / DIF, not covered here but relevant.
- `Documentation/admin-guide/cgroup-v2.rst` (the io controller section) — Ch. 64.

**Papers and primary sources**

- Bjørling, Axboe, Nellans, Bonnet, "Linux Block IO: Introducing Multi-queue SSD Access on Multi-core Systems," SYSTOR 2013 ★★★ — **the blk-mq design paper.** Contains the measurements that motivated §T.4. Read it.
- Axboe's LSFMM presentations on blk-mq and io_uring — the design rationale, updated over the years.
- Yang, Minturn, Hady, "When Poll is Better than Interrupt," FAST 2012 ★★★ — §T.5's `HCTX_TYPE_POLL`, argued from measurements.
- Lee et al., "Asynchronous I/O Stack: A Low-latency Kernel I/O Stack for Ultra-Low Latency SSDs," USENIX ATC 2019 — where the remaining latency goes.
- Bjørling, "I/O Determinism and NVMe," and the NVMe specification's queue model.

**LWN**

- "The multiqueue block layer" (2013) ★★★ — the introduction.
- "Block layer introduction part 1: the bio layer" and "part 2: the request layer" (Neil Brown, 2017) ★★★ — **the best available explanation of this chapter's material.** Read both.
- "Immutable biovecs and biovec iterators" ★★★ — §T.2's rewrite.
- "The end of the legacy block layer" (2018) — the removal of single-queue.
- "Improving the block layer's tag allocation" and the `sbitmap` coverage
- "Polled I/O and hybrid polling"
- "Zoned block devices" (Ch. 60 §T.5)
- "io_uring" series ★★★ — the submitter side of this story
- "Block-layer throttling and cgroup v2" — Ch. 64
- Jens Axboe's periodic block-layer status updates

**Source reading order**

1. `Documentation/block/blk-mq.rst`, then Neil Brown's two LWN articles. Do not start with the code.
2. `include/linux/blk_types.h` ★★★ — `bio`, `bio_vec`, `bvec_iter`, the ops and flags.
3. `block/bio.c`: `bio_split`, `bio_chain`, `bio_endio`, `bio_add_page` ★★★
4. `block/blk-core.c`: `submit_bio_noacct`, `__submit_bio_noacct` ★★★ — §T.8's recursion handling.
5. `block/blk-mq.c`: `blk_mq_submit_bio` ★★★, then `blk_mq_dispatch_rq_list`, `blk_mq_end_request`.
6. `block/blk-mq-tag.c` and `lib/sbitmap.c` ★★★ — §T.6.
7. `block/blk-merge.c`: `__bio_split_to_limits`, `blk_attempt_plug_merge`, `ll_back_merge_fn`.
8. `block/blk-settings.c`: `blk_stack_limits` — §T.9.
9. `drivers/block/null_blk/` ★★★ — the simplest complete blk-mq driver; excellent to read.

**Tools**

- `blktrace` + `blkparse` + `btt` ★★★ — **learn `btt`'s Q2D/D2C breakdown; it answers most questions.**
- `trace-cmd record -e block:\*` ★★★ — the same data, easier to filter
- `iostat -xz 1` ★★★ — `%util`, `aqu-sz`, `r_await`/`w_await`
- `biolatency`, `biosnoop`, `biotop`, `bitesize` (bcc/bpftrace) ★★★
- `/sys/kernel/debug/block/DEV/` ★★★ — the per-hctx ground truth
- `fio` ★★★ — `--ioengine=libaio/io_uring/psync`, `--iodepth`, `--numjobs`, `--hipri`, `--bs`
- `null_blk` ★★★ — a configurable device with no hardware; ideal for isolating the block layer
- `scsi_debug` — same, with SCSI semantics and configurable geometry
- `lsblk -o NAME,ALIGNMENT,PHY-SEC,LOG-SEC,MIN-IO,OPT-IO` ★★★
- `perf record -e block:\*` and `perf lock` for contention analysis

---

→ Next: [64-block-layer-2.md](64-block-layer-2.md)

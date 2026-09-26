# Chapter 64 — Block layer II: I/O schedulers, plugging, merging, and QoS

> **Goal:** Understand how Linux decides *which* I/O goes next and *how much* each competitor gets. Understand why the elevator algorithm existed and why it mostly stopped mattering, what `mq-deadline`, `bfq`, and `kyber` each optimise for and the evidence for choosing between them, why anticipatory scheduling was both brilliant and abandoned, how cgroup v2's io controller implements weights and limits, why `io.latency` and `io.cost` needed a cost model at all, the writeback-throttling feedback loop, and how to diagnose a latency problem rather than guess at it. By the end you can read `block/*-iosched.c`, choose a scheduler with evidence, and build a QoS configuration that does what you intended.

---

## Theory & First Principles

### T.0 — Start here: why the best I/O scheduler is usually no scheduler

```bash
cat /sys/block/sda/queue/scheduler        # [mq-deadline] kyber bfq none
cat /sys/block/nvme0n1/queue/scheduler    # [none] mq-deadline kyber bfq
#                                            ^^^^^^ the default on NVMe
```

A whole kernel subsystem — decades of algorithmic work, four implementations — and on modern
hardware the default is **to not use it**. Understanding exactly why is the best available
lesson in "an optimization is a contract with a hardware model, and hardware models expire."

**The original justification, made numerically.** On an HDD:

```
  seek + rotational latency:   ~10 ms
  transfer of 4 KiB:           ~0.04 ms

  -> 99.6% of the time spent servicing a random request is MECHANICAL POSITIONING.
```

So reordering requests into ascending sector order (the *elevator*) converts N seeks into
roughly one sweep. **A 100x throughput win.** Spending 10 µs of CPU on sorting to save 10 ms
of seek is a 1000:1 return — obviously correct.

On NVMe:

```
  "seek":                      ~0 (there is no arm)
  service time for 4 KiB:      ~80 us, with up to 65,536 queues x 65,536 deep

  -> sorting costs MORE CPU than it saves, and the device's own internal
     parallelism reorders far better than you possibly can.
```

**The optimization did not merely become smaller. It became negative.** `none` (the `blk-mq`
no-op path) is faster because the cheapest correct thing to do with a request is to submit
it.

**But — and this is why the chapter is not over — scheduling was never only about seeks.**
Pull the two questions apart:

| Question | Answered by | Still relevant on NVMe? |
|---|---|---|
| **"In what order does the device see requests?"** | the elevator | **no** |
| **"Which process gets to issue I/O, and how much?"** | fairness / QoS | **yes — more than ever** |

A `dd` and a latency-sensitive database on the same device still need arbitration. That is a
*resource control* problem, not a *geometry* problem, and it is why `bfq` (proportional
share, good for desktop and interactive latency), `kyber` (target-latency-driven throttling),
and the cgroup-v2 `io` controller (`io.latency`, `io.cost`, `io.max` — Ch. 92) all still
exist and still matter.

**The general principle, worth writing down:**

> **When hardware changes by orders of magnitude, an optimization does not merely lose value —
> it can invert.** The discipline is to ask what *model* an optimization assumes, and to check
> whether that model still holds — not merely whether the code still runs.

You have now seen this three times: ext4's block groups minimizing seeks (Ch. 57 §T.0),
`buffer_head`'s per-block mapping (Ch. 55 §T.0), and the elevator. **Go looking for the
fourth.**

**And the deeper reason `none` wins deserves stating separately:** `blk-mq` moved the work
from *reordering* to *not contending*. Per-CPU software queues, per-device hardware queues,
tag-based request allocation, and completing on the submitting CPU (so the completion touches
a warm cacheline and needs no IPI) — the performance now comes from cache locality and the
absence of shared state, not from cleverness about sector numbers.

```bash
echo none | sudo tee /sys/block/nvme0n1/queue/scheduler
sudo fio --name=t --filename=/dev/nvme0n1 --rw=randread --bs=4k \
         --iodepth=64 --numjobs=8 --ioengine=io_uring --runtime=10 --group_reporting
cat /sys/kernel/debug/block/nvme0n1/hctx0/{tags,busy}
sudo /usr/share/bcc/tools/biolatency -mT 1
```

---

### T.1 Two different questions

The block layer answers two questions that are often conflated:

| Question | Mechanism | Optimises |
|---|---|---|
| **In what order?** | I/O scheduler | device efficiency, latency fairness |
| **How much for whom?** | QoS / cgroup io controller | isolation between competing workloads |

A scheduler reorders a queue; a QoS controller decides who may put things in the queue at all. They compose: `blk-throttle` and `io.cost` run *before* the scheduler, on submission; the scheduler runs when dispatching.

Confusing them produces bad configurations. "My database is slow, I will switch to BFQ" addresses ordering when the problem is usually that a backup job is consuming all the device's capability — a QoS problem.

### T.2 The elevator, and why it stopped mattering

On a rotating disk, the dominant cost is seeking:

| Operation | Cost |
|---|---|
| Sequential read, next sector | ~0 |
| Seek to an adjacent track | ~0.5 ms |
| Full-stroke seek | ~15 ms |
| Rotational latency (7200 RPM) | ~4 ms average |

So servicing requests in sector order rather than arrival order can improve throughput by an order of magnitude. This is the **elevator algorithm** (SCAN, C-SCAN, LOOK), named for the obvious analogy: an elevator does not serve floors in the order buttons were pressed.

Linux's historical schedulers were all elevator variants:

| Scheduler | Era | Idea |
|---|---|---|
| Linus elevator | ≤2.4 | plain sorted insertion; starvation possible |
| **Deadline** | 2.5+ | sorted, plus per-request deadlines to bound starvation |
| **Anticipatory (AS)** | 2.5–2.6.33 | after a read, **wait briefly** for the next one (§T.3) |
| **CFQ** | 2.6–5.0 | per-process time slices, like CFS for disk |
| **noop** | — | FIFO with merging |

Then SSDs arrived and the cost model inverted:

| | HDD | SATA SSD | NVMe |
|---|---|---|---|
| Random 4K read | 5–15 ms | 50–100 µs | 10–80 µs |
| Sequential vs random ratio | **~1000×** | ~2× | **~1×** |
| Useful queue depth | 1–4 (NCQ 32) | 32 | **1000s** |
| Does sorting help? | **enormously** | slightly | **no** |

On NVMe, **sorting is pure overhead**. The device has its own queues, its own reordering, and enough parallelism that the host's ordering is irrelevant. Worse, a scheduler adds per-request CPU work and a serialisation point — exactly what Ch. 63 §T.4's rewrite was trying to remove.

Hence the modern defaults:

```sh
# Typical distribution udev rules
ACTION=="add|change", KERNEL=="sd[a-z]*", ATTR{queue/rotational}=="1", \
	ATTR{queue/scheduler}="bfq"
ACTION=="add|change", KERNEL=="sd[a-z]*", ATTR{queue/rotational}=="0", \
	ATTR{queue/scheduler}="mq-deadline"
ACTION=="add|change", KERNEL=="nvme[0-9]*n[0-9]*", \
	ATTR{queue/scheduler}="none"
```

**`none` on NVMe is not laziness; it is correct.** The measured difference is usually that any scheduler *costs* throughput on NVMe.

The residual value of a scheduler on fast devices is not throughput but **fairness and latency isolation** — and that is increasingly handled by the QoS layer (§T.6) instead.

### T.3 Anticipatory scheduling: the idea worth knowing

Even though it was removed in 2.6.33, anticipatory scheduling is worth understanding because its insight recurs.

**The problem (deceptive idleness):** a process doing synchronous sequential reads issues one request, waits, then issues the next. Between them, the scheduler sees an empty queue from that process and services someone else — seeking away. When the next request arrives, it seeks back. Two processes doing sequential reads can turn a 100 MB/s device into a 5 MB/s one purely through interleaving.

**The insight (Iyer & Druschel, SOSP 2001):** after servicing a synchronous read, **do nothing for a few milliseconds.** The process is very likely to issue an adjacent read, and waiting 3 ms to avoid a 15 ms seek is a clear win.

This is *voluntarily idling a busy resource because the expected future arrival is worth more than the present alternative* — the same reasoning as ski-rental (Ch. 48 §T.3), as CPU idle-state selection, and as TCP's delayed ACK. It is counterintuitive and correct.

CFQ incorporated it as `slice_idle`, and BFQ still does. On SSDs the seek saving is zero, so idling is pure loss — hence `slice_idle=0` on non-rotational devices, and hence the long tail of "BFQ is slow on my SSD" reports from configurations that did not disable it.

### T.4 The three modern schedulers

**`mq-deadline`** — sorted, with deadlines.

Two data structures per direction:

```
read:  RB-tree sorted by sector  +  FIFO list sorted by expiry
write: RB-tree sorted by sector  +  FIFO list sorted by expiry
```

The algorithm:

1. If a request has passed its deadline, service it (from the FIFO).
2. Otherwise continue in sector order from the current position (from the tree).
3. Prefer reads over writes, up to `writes_starved` consecutive batches.

Tunables:

| Knob | Default | Meaning |
|---|---|---|
| `read_expire` | 500 ms | read deadline |
| `write_expire` | 5000 ms | write deadline |
| `writes_starved` | 2 | read batches before a write batch |
| `fifo_batch` | 16 | requests dispatched per batch |
| `front_merges` | 1 | attempt front merges (usually useless; back merges dominate) |

The design point: **reads are synchronous, writes usually are not.** A process waiting on a read is blocked; a process that issued a buffered write is not. So a 10× deadline asymmetry is correct. This is the same reasoning as `REQ_SYNC` (Ch. 63 §T.3) and as the page cache's treatment of dirty pages (Ch. 52 §T.6).

`mq-deadline` is simple, predictable, low-overhead, and a good default for SATA SSDs and anything where you want bounded worst-case latency without much machinery.

**`bfq`** — Budget Fair Queueing.

BFQ gives each process (or cgroup) a **budget in sectors**, not a time slice, and services them in a fair-queueing order derived from WF²Q+. It adds:

- **Weights** (`io.bfq.weight`), per-process or per-cgroup.
- **Idling** (§T.3) to preserve sequentiality.
- **Low-latency heuristics**: newly-started or interactive applications get a temporary boost, which is why BFQ makes desktops feel responsive under load.
- **Injection**: dispatch other requests during an idle window if the device can absorb them without losing the benefit.

BFQ's strength is **isolation and interactive responsiveness**; its weakness is **CPU cost**. It is the most complex scheduler by a wide margin, and on a device doing 500K IOPS its per-request work becomes the bottleneck. The honest summary: excellent on rotating disks and on single-queue SSDs for desktop workloads; unsuitable for high-IOPS servers.

**`kyber`** — latency targets, by throttling.

Kyber is deliberately minimal. It has two knobs:

```
read_lat_nsec   (default 2 ms)
write_lat_nsec  (default 10 ms)
```

and one mechanism: it measures achieved latency and **adjusts the number of requests it allows in flight** to hit the target. If read latency exceeds the goal, it reduces the depth for writes and for "discard/other"; if there is headroom, it increases it.

It does no sorting. It maintains separate domains (read, sync write, other) with per-domain depth limits and adjusts them with a feedback loop.

Kyber is the right answer when you have a fast multi-queue device and want to bound read latency without paying for BFQ. It is used at scale (it came out of Facebook) and is a good example of **a controller rather than a policy**: it does not decide an order, it decides a rate, and lets the device do the rest.

The comparison:

| | `none` | `mq-deadline` | `kyber` | `bfq` |
|---|---|---|---|---|
| Sorts | no | yes | no | yes |
| Per-request CPU | ~0 | low | low | **high** |
| Bounds read latency | no | by deadline | **by feedback** | by fair queueing |
| Fairness between processes | no | no | weak | **strong** |
| cgroup weight support | via io.cost | no | no | **yes** |
| Idling | no | no | no | **yes** |
| Best for | NVMe, raw throughput | SATA SSD, predictability | fast device + latency SLO | HDD, desktop |

### T.5 Merging, revisited

Chapter 63 §T.10 covered plug merging. The scheduler adds two more opportunities, and the order matters:

```
1. Plug merge        -- per-task list, NO LOCK, cheapest
2. Scheduler merge   -- the elevator's hash/tree, under the scheduler's lock
3. Software-queue merge -- per-CPU list
```

```c
bool blk_mq_sched_bio_merge(struct request_queue *q, struct bio *bio,
			    unsigned int nr_segs)
{
	struct elevator_queue *e = q->elevator;
	struct blk_mq_ctx *ctx = blk_mq_get_ctx(q);
	struct blk_mq_hw_ctx *hctx = blk_mq_map_queue(q, bio->bi_opf, ctx);
	bool ret = false;
	enum hctx_type type;

	if (e && e->type->ops.bio_merge) {
		ret = e->type->ops.bio_merge(q, bio, nr_segs);
		goto out_put;
	}

	type = hctx->type;
	if (!(hctx->flags & BLK_MQ_F_SHOULD_MERGE) ||
	    list_empty_careful(&ctx->rq_lists[type]))
		goto out_put;

	/* No scheduler: try the software queue's list directly */
	spin_lock(&ctx->lock);
	if (blk_bio_list_merge(q, &ctx->rq_lists[type], bio, nr_segs)) {
		ctx->rq_merged++;
		ret = true;
	}
	spin_unlock(&ctx->lock);
out_put:
	return ret;
}
```

Three merge types:

| Type | Meaning |
|---|---|
| **Back merge** | the new bio starts where an existing request ends — the common case |
| **Front merge** | the new bio ends where an existing request starts — rare |
| **Request merge** | two requests become one, after a bio merge makes them adjacent |

Back merges dominate because writeback and readahead walk forward. `front_merges=0` on `mq-deadline` is a defensible micro-optimisation.

Merging is not free: it requires a hash lookup and a limits check (`ll_back_merge_fn` verifies the result still fits `max_sectors`, `max_segments`, and does not cross a `chunk_sectors` boundary). `/sys/block/*/queue/nomerges` can disable it, and on a device where sequential and random cost the same, disabling merging occasionally helps by removing the lookup.

### T.6 cgroup v2's io controller: four mechanisms

This is the part of the chapter that matters most operationally, and it has more moving parts than people expect.

**(a) `io.max` — hard limits (blk-throttle).**

```sh
echo "8:0 rbps=10485760 wbps=5242880 riops=1000 wiops=500" > io.max
```

A token-bucket throttle applied at submission. Simple, predictable, and **it wastes capacity**: if the limited cgroup is the only one running, the device sits idle. Use it when you need a hard ceiling (multi-tenant billing), not for general isolation.

**(b) `io.weight` — proportional share.**

```sh
echo "default 100" > io.weight
echo "8:0 500" > io.weight
```

Weights only make sense if something implements them. Two implementations:

- **BFQ** implements weights directly, in the scheduler (`io.bfq.weight`).
- **`io.cost`** implements them above the scheduler, and works with `none`.

**(c) `io.latency` — a latency target, by donation.**

```sh
echo "8:0 target=10" > io.latency     # 10 ms
```

The semantics are unusual and worth stating precisely: this is **not** a guarantee. It says "if this cgroup's average completion latency exceeds the target, throttle *other* cgroups until it does not." It is a **protection mechanism with a priority ordering**, not an SLO.

The lowest-latency-target cgroup is effectively the highest priority. A cgroup with no target set is a donor of last resort. This makes `io.latency` a good fit for "protect the database, let the batch jobs absorb the variance."

**(d) `io.cost` — a cost model.**

The hard problem: what does "50 % of the device" mean? Fifty percent of IOPS? Of bytes? Of time? For a device where a 4K random read costs 80 µs and a 1M sequential read costs 400 µs, neither IOPS nor bytes is proportional to the resource consumed.

`io.cost` builds a **linear cost model**:

```
cost(io) = ctrl->page_cost * pages
         + ctrl->seqio_cost * (is_sequential ? 1 : 0)
         + ctrl->randio_cost * (is_random ? 1 : 0)
```

with separate read and write coefficients, calibrated either from a built-in table (`io.cost.model` with a device class) or **auto-tuned at runtime** by observing achieved latency (`io.cost.qos`).

```sh
cat /sys/fs/cgroup/io.cost.model
# 8:0 ctrl=auto model=linear rbps=... rseqiops=... rrandiops=... \
#     wbps=... wseqiops=... wrandiops=...

cat /sys/fs/cgroup/io.cost.qos
# 8:0 enable=1 ctrl=auto rpct=95.00 rlat=5000 wpct=95.00 wlat=5000 \
#     min=50.00 max=150.00
```

The QoS parameters say: "keep the 95th-percentile read latency under 5 ms; if we are beating it, allow the device up to 150 % of its modelled capacity; if we are missing it, throttle to 50 %." So the controller has **two nested loops**: the cost model converts I/O to abstract cost units, and the QoS loop adjusts how many cost units per second the device is deemed capable of.

This is genuinely sophisticated, and it is the right answer to §T.6's hard problem. It is also why `io.cost` needs calibration (`iocost_coef_gen.py` in the kernel tree, or `resctl-bench`) to work well on an uncharacterised device.

**Which to use:**

| Goal | Use |
|---|---|
| Hard ceiling, billing | `io.max` |
| Proportional share, fast device | `io.cost` + weights |
| Proportional share, rotating disk | BFQ + `io.bfq.weight` |
| Protect one workload's latency | `io.latency` |
| Desktop responsiveness | BFQ |

And the critical caveat: **buffered writes are not attributed to the writer by default.** Writeback happens later, from kernel threads, often long after the process exited. **cgroup writeback** (Ch. 52 §T.8) fixes this by tracking the owning cgroup on the inode's `wb`, but it requires:

- cgroup v2 (not v1),
- a filesystem that supports it (ext4, btrfs, f2fs do; **XFS did not for a long time**),
- memory and io controllers enabled together on the same hierarchy.

Without all three, your io limits apply only to reads and direct writes, which is a very common and very confusing misconfiguration.

### T.7 Writeback throttling (`wbt`)

A separate mechanism from both scheduling and cgroups, and one of the most useful.

**The problem:** background writeback can fill the device queue, so a foreground read waits behind hundreds of writes. Read latency goes from 100 µs to 50 ms. The device is not overloaded — the *queue* is.

**The mechanism:** `wbt` monitors read latency and throttles background writes when it degrades.

```c
enum wbt_flags {
	WBT_TRACKED		= 1,
	WBT_READ		= 2,
	WBT_KSWAPD		= 4,
	WBT_DISCARD		= 8,
};

enum {
	WBT_STATE_ON_DEFAULT	= 1,
	WBT_STATE_ON_MANUAL	= 2,
	WBT_STATE_OFF_DEFAULT	= 3,
	WBT_STATE_OFF_MANUAL	= 4,
};
```

It maintains a scaling factor and adjusts the allowed depth for background writes:

```
if read latency > target:   scale down (fewer background writes in flight)
if read latency << target:  scale up
```

Controlled by:

```sh
cat /sys/block/sda/queue/wbt_lat_usec     # the target; 0 disables
echo 2000 > /sys/block/sda/queue/wbt_lat_usec
```

Defaults are 2 ms for rotational, 75 µs for non-rotational. The effect on mixed read/write workloads is often dramatic — Lab 64.6 measures it.

`wbt` is automatically disabled when an I/O scheduler that does its own latency management (`kyber`, `bfq`) is in use, because two controllers fighting over the same signal is worse than either alone. This is a recurring systems lesson: **nested feedback loops with the same input interfere.**

### T.8 What to measure

The most common mistake is optimising the wrong number. The hierarchy:

| Symptom | Measure | Then |
|---|---|---|
| "Slow" | `iostat -xz 1`: `%util`, `aqu-sz`, `await` | distinguish saturated from latent |
| High `await`, low `aqu-sz` | device latency | the device is slow; look at `btt` D2C |
| High `await`, high `aqu-sz` | queueing | too much concurrency; look at Q2D |
| High `%util`, low throughput | small I/Os | check merging and request sizes |
| p99 ≫ p50 | tail latency | `biolatency`, check scheduler and wbt |
| One workload starves another | isolation | cgroup io controller |

Note that **`%util` is nearly meaningless on modern devices**. It measures the fraction of time at least one request was outstanding. An NVMe device with 64 queues can be 100 % "utilised" at 1 % of its capability. `iostat`'s man page now says so. Use `aqu-sz` (average queue size) and latency instead.

The single most useful decomposition is `btt`'s:

```
Q2G   bio arrived -> request allocated     (tag contention)
G2I   request allocated -> inserted        (scheduler entry)
I2D   inserted -> dispatched               (SCHEDULER TIME)
D2C   dispatched -> completed              (DEVICE TIME)
Q2C   total
```

**I2D is the scheduler's contribution. D2C is the device's.** If I2D is large you have a queueing or scheduling problem; if D2C is large you have a device problem. Almost every "which scheduler should I use" question is answered by looking at this decomposition under the real workload.

### T.9 A decision procedure

```
1. Measure first: iostat -xz, btt, biolatency. Get a baseline.

2. Is the device saturated (D2C rising with load)?
   YES -> no scheduler will help. Reduce the load, add devices,
          or apply QoS to decide WHO gets the capacity.
   NO  -> continue.

3. Is the problem ordering (some I/O waits behind other I/O)?
   Rotating disk?      -> bfq (or mq-deadline for servers)
   SATA SSD?           -> mq-deadline
   NVMe?               -> none, plus wbt, plus io.cost if you need isolation
   Latency SLO?        -> kyber with an explicit target

4. Is the problem isolation (workload A hurts workload B)?
   -> cgroup v2 io controller (T.6), NOT a scheduler change.
   -> Verify cgroup writeback is actually working first.

5. Is the problem buffered writes drowning reads?
   -> wbt (T.7). Check wbt_lat_usec is non-zero.

6. Re-measure. If it did not help, revert it.
```

Step 6 is the one people skip. Scheduler changes are frequently cargo-culted and frequently make things worse; the only defence is measuring before and after with the actual workload.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `block/elevator.c` ★★★ | scheduler registration, switching, the `elevator_type` interface |
| `block/mq-deadline.c` ★★★ | §T.4's deadline scheduler; ~1100 lines and readable in one sitting |
| `block/kyber-iosched.c` ★★★ | §T.4's latency controller |
| `block/bfq-iosched.c`, `bfq-wf2q.c`, `bfq-cgroup.c` ★★★ | BFQ; ~10000 lines |
| `block/blk-mq-sched.c` | the glue between blk-mq and the scheduler |
| `block/blk-throttle.c` ★★★ | §T.6(a)'s `io.max` |
| `block/blk-iocost.c` ★★★ | §T.6(d)'s cost model; the header comment is a full design document |
| `block/blk-iolatency.c` ★★★ | §T.6(c) |
| `block/blk-wbt.c` ★★★ | §T.7 |
| `block/blk-rq-qos.c`, `blk-rq-qos.h` | the QoS hook chain |
| `block/blk-cgroup.c` | `blkcg` infrastructure, `blkg` per-(cgroup,device) state |
| `block/blk-stat.c` | latency statistics used by kyber, wbt, and iocost |
| `Documentation/block/deadline-iosched.rst`, `bfq-iosched.rst` ★★★ | |
| `Documentation/admin-guide/cgroup-v2.rst` ★★★ | the io controller section |

### 1.2 The scheduler interface

```c
struct elevator_mq_ops {
	int  (*init_sched)(struct request_queue *, struct elevator_type *);
	void (*exit_sched)(struct elevator_queue *);
	int  (*init_hctx)(struct blk_mq_hw_ctx *, unsigned int);
	void (*exit_hctx)(struct blk_mq_hw_ctx *, unsigned int);
	void (*depth_updated)(struct blk_mq_hw_ctx *);

	bool (*allow_merge)(struct request_queue *, struct request *, struct bio *);
	bool (*bio_merge)(struct request_queue *, struct bio *, unsigned int);
	int  (*request_merge)(struct request_queue *q, struct request **, struct bio *);
	void (*request_merged)(struct request_queue *, struct request *, enum elv_merge);
	void (*requests_merged)(struct request_queue *, struct request *, struct request *);

	void (*limit_depth)(blk_opf_t, struct blk_mq_alloc_data *);
	void (*prepare_request)(struct request *);
	void (*finish_request)(struct request *);
	void (*insert_requests)(struct blk_mq_hw_ctx *, struct list_head *,
				blk_insert_t);
	struct request *(*dispatch_request)(struct blk_mq_hw_ctx *);
	bool (*has_work)(struct blk_mq_hw_ctx *);
	void (*completed_request)(struct request *, u64);
	void (*requeue_request)(struct request *);
	struct request *(*former_request)(struct request_queue *, struct request *);
	struct request *(*next_request)(struct request_queue *, struct request *);
	void (*init_icq)(struct io_cq *);
	void (*exit_icq)(struct io_cq *);
};
```

The essential three are `insert_requests`, `dispatch_request`, and `has_work`. Everything else is optional. `limit_depth` is how a scheduler reserves tags (kyber uses it to enforce per-domain depth).

### 1.3 `mq-deadline`

```c
struct dd_per_prio {
	struct list_head dispatch;
	struct rb_root sort_list[DD_DIR_COUNT];      /* sorted by sector */
	struct list_head fifo_list[DD_DIR_COUNT];    /* sorted by expiry */
	struct request *next_rq[DD_DIR_COUNT];       /* the sector cursor */
	struct io_stats_per_prio stats;
};

struct deadline_data {
	struct dd_per_prio per_prio[DD_PRIO_COUNT];  /* RT / BE / IDLE */
	enum dd_data_dir last_dir;
	unsigned int batching;
	unsigned int starved;

	int fifo_expire[DD_DIR_COUNT];               /* 500ms / 5000ms */
	int fifo_batch;                              /* 16 */
	int writes_starved;                          /* 2 */
	int front_merges;
	u32 async_depth;
	int prio_aging_expire;

	spinlock_t lock;
	spinlock_t zone_lock;
};
```

The dispatch decision, which *is* §T.4's algorithm:

```c
static struct request *__dd_dispatch_request(struct deadline_data *dd,
					     struct dd_per_prio *per_prio,
					     unsigned long latest_start)
{
	struct request *rq, *next_rq;
	enum dd_data_dir data_dir;
	...
	/* 1. Anything already on the dispatch list? */
	if (!list_empty(&per_prio->dispatch)) {
		rq = list_first_entry(&per_prio->dispatch, struct request, queuelist);
		...
		goto done;
	}

	/* 2. Continue the current batch in sector order, if there is one. */
	rq = deadline_next_request(dd, per_prio, dd->last_dir);
	if (rq && dd->batching < dd->fifo_batch)
		goto dispatch_request;

	/* 3. Batch exhausted. Pick a direction: reads preferred, but
	 *    not more than `writes_starved` times in a row. */
	if (reads) {
		if (deadline_fifo_request(dd, per_prio, DD_WRITE) &&
		    (dd->starved++ >= dd->writes_starved))
			goto dispatch_writes;
		data_dir = DD_READ;
		goto dispatch_find_request;
	}
	if (writes) {
dispatch_writes:
		dd->starved = 0;
		data_dir = DD_WRITE;
		goto dispatch_find_request;
	}
	return NULL;

dispatch_find_request:
	/* 4. Expired request? Take it (bounded starvation). Else continue sorted. */
	next_rq = deadline_next_request(dd, per_prio, data_dir);
	if (deadline_check_fifo(per_prio, data_dir) || !next_rq) {
		rq = deadline_fifo_request(dd, per_prio, data_dir);
	} else {
		rq = next_rq;
	}
	...
	dd->batching = 0;

dispatch_request:
	dd->batching++;
	deadline_move_request(dd, per_prio, rq);
done:
	...
	return rq;
}
```

Read it against §T.4's four steps; the correspondence is exact. This is a scheduler you can hold entirely in your head, which is a large part of its value.

### 1.4 Kyber's feedback loop

```c
enum {
	KYBER_READ,
	KYBER_WRITE,
	KYBER_DISCARD,
	KYBER_OTHER,
	KYBER_NUM_DOMAINS,
};

static const unsigned int kyber_depth[] = {
	[KYBER_READ]	= 256,
	[KYBER_WRITE]	=  128,
	[KYBER_DISCARD]	=   64,
	[KYBER_OTHER]	=   16,
};

/* The latency buckets: good / great / bad */
enum {
	KYBER_LATENCY_SHIFT = 2,
	KYBER_GOOD_BUCKETS = 1 << KYBER_LATENCY_SHIFT,
	KYBER_LATENCY_BUCKETS = 2 * KYBER_GOOD_BUCKETS,
};

static void kyber_adjust_rw_depth(struct kyber_queue_data *kqd,
				  unsigned int sched_domain, unsigned int type,
				  unsigned int percentile, unsigned int cur_depth)
{
	unsigned int depth;

	switch (type) {
	case KYBER_TOTAL_LATENCY:
		if (percentile >= KYBER_GOOD_BUCKETS)
			depth = cur_depth - 1;        /* too slow: back off */
		else
			return;
		break;
	case KYBER_IO_LATENCY:
		if (percentile >= KYBER_GOOD_BUCKETS)
			depth = cur_depth - 1;
		else if (percentile < KYBER_GOOD_BUCKETS / 2)
			depth = cur_depth + 1;        /* headroom: allow more */
		else
			return;
		break;
	}
	kqd->domain_tokens[sched_domain].sb.depth =
		clamp(depth, 1U, kyber_depth[sched_domain]);
}
```

Kyber distinguishes **total latency** (including queueing) from **I/O latency** (device time only). If device latency is fine but total latency is bad, the problem is queueing and reducing depth helps. If device latency itself is bad, reducing depth will not help — the device is saturated. **Separating the two signals is what makes the controller stable**, and it is the key design idea.

### 1.5 The QoS hook chain

```c
enum rq_qos_id {
	RQ_QOS_WBT,
	RQ_QOS_LATENCY,
	RQ_QOS_COST,
	RQ_QOS_IOPRIO,
};

struct rq_qos_ops {
	void (*throttle)(struct rq_qos *, struct bio *);     /* may SLEEP */
	void (*track)(struct rq_qos *, struct request *, struct bio *);
	void (*merge)(struct rq_qos *, struct request *, struct bio *);
	void (*issue)(struct rq_qos *, struct request *);
	void (*requeue)(struct rq_qos *, struct request *);
	void (*done)(struct rq_qos *, struct request *);
	void (*done_bio)(struct rq_qos *, struct bio *);
	void (*cleanup)(struct rq_qos *, struct bio *);
	void (*queue_depth_changed)(struct rq_qos *);
	void (*exit)(struct rq_qos *);
	const struct blk_mq_debugfs_attr *debugfs_attrs;
};
```

They form a chain on the request queue, invoked at each stage:

```c
static inline void rq_qos_throttle(struct request_queue *q, struct bio *bio)
{
	if (q->rq_qos)
		__rq_qos_throttle(q->rq_qos, bio);
}

void __rq_qos_throttle(struct rq_qos *rqos, struct bio *bio)
{
	do {
		if (rqos->ops->throttle)
			rqos->ops->throttle(rqos, bio);
		rqos = rqos->next;
	} while (rqos);
}
```

`throttle` is called from `blk_mq_submit_bio` *before* tag allocation and **may sleep** — that is how `io.max`, `io.latency`, and `io.cost` delay a submitter. Note the implication: an I/O-throttled process blocks in `submit_bio`, which means it shows as `D` state and its plug is flushed (Ch. 63 §T.10).

### 1.6 iocost's cost model

The header comment in `block/blk-iocost.c` is a ~300-line design document and is worth reading in full. The core:

```c
/* The device's capability in "virtual time" units */
struct ioc {
	...
	u64				vtime_base_rate;
	atomic64_t			vtime_rate;
	u64				vtime_period_seq;
	...
	struct ioc_params		params;
	u32				margins[3];
	u32				period_us;
	...
};

static u64 calc_vtime_cost_builtin(struct bio *bio, struct ioc_gq *iocg,
				   bool is_merge, u64 *costp)
{
	struct ioc *ioc = iocg->ioc;
	u64 coef_seqio, coef_randio, coef_page;
	u64 pages = max_t(u64, bio_sectors(bio) >> IOC_SECT_TO_PAGE_SHIFT, 1);
	u64 seek_pages = 0;
	u64 cost = 0;

	switch (bio_op(bio)) {
	case REQ_OP_READ:
		coef_seqio	= ioc->params.lcoefs[LCOEF_RSEQIO];
		coef_randio	= ioc->params.lcoefs[LCOEF_RRANDIO];
		coef_page	= ioc->params.lcoefs[LCOEF_RPAGE];
		break;
	case REQ_OP_WRITE:
		coef_seqio	= ioc->params.lcoefs[LCOEF_WSEQIO];
		coef_randio	= ioc->params.lcoefs[LCOEF_WRANDIO];
		coef_page	= ioc->params.lcoefs[LCOEF_WPAGE];
		break;
	default:
		goto out;
	}

	if (iocg->cursor) {
		seek_pages = abs(bio->bi_iter.bi_sector - iocg->cursor);
		seek_pages >>= IOC_SECT_TO_PAGE_SHIFT;
	}

	if (!is_merge) {
		if (seek_pages > LCOEF_RANDIO_PAGES) {
			cost += coef_randio;        /* a seek: expensive */
		} else {
			cost += coef_seqio;         /* sequential: cheap */
		}
	}
	cost += pages * coef_page;
out:
	*costp = cost;
	return cost;
}
```

Note `iocg->cursor`: the controller tracks each cgroup's last sector to classify the next I/O as sequential or random. **The cost model has to reconstruct information the filesystem already knew and threw away** — an interesting consequence of the layering.

### 1.7 Observability

| Where | What |
|---|---|
| `/sys/block/DEV/queue/scheduler` ★★★ | the current and available schedulers |
| `/sys/block/DEV/queue/iosched/` ★★★ | the current scheduler's tunables |
| `/sys/block/DEV/queue/wbt_lat_usec` ★★★ | §T.7 |
| `/sys/fs/cgroup/*/io.{max,weight,latency,stat,pressure}` ★★★ | §T.6 |
| `/sys/fs/cgroup/io.cost.{model,qos}` ★★★ | |
| `/sys/kernel/debug/block/DEV/sched/` ★★★ | scheduler internal state |
| `/sys/kernel/debug/block/DEV/rqos/` | the QoS chain's state |
| `/proc/pressure/io` ★★★ | PSI: stall time due to I/O |
| `/sys/fs/cgroup/*/io.pressure` ★★★ | per-cgroup PSI |
| `iostat -xz 1` ★★★ | |
| `btt -i trace` ★★★ | §T.8's Q2G/G2I/I2D/D2C |
| `biolatency -D`, `biosnoop`, `biotop` | |
| `trace-cmd record -e block:\* -e bfq:\* -e wbt:\* -e iocost:\*` ★★★ | |

PSI (`/proc/pressure/io`) deserves a mention: it reports the fraction of time tasks were *stalled* on I/O, which is a far better saturation signal than `%util`. `some avg10=45.23` means 45 % of the last 10 seconds had at least one task stalled on I/O. It is the number to alert on.

---

## 2. Practice

### Lab 64.1 — Compare the schedulers honestly

```sh
sudo apt install -y fio blktrace bpfcc-tools sysstat
sudo modprobe scsi_debug dev_size_mb=2048 delay=1 ndelay=100000
DEV=$(lsblk -ndo NAME,MODEL | awk '/scsi_debug/{print "/dev/"$1}')
DN=$(basename $DEV)

cat /sys/block/$DN/queue/scheduler
cat /sys/block/$DN/queue/rotational
```

A mixed workload — the case where schedulers actually differ:

```sh
cat > mixed.fio <<'EOF'
[global]
filename=${DEV}
direct=1
runtime=20
time_based
group_reporting=0
ioengine=libaio

[reader]
rw=randread
bs=4k
iodepth=1
numjobs=1

[writer]
rw=write
bs=1M
iodepth=32
numjobs=4
EOF

for sched in none mq-deadline kyber bfq; do
  echo $sched | sudo tee /sys/block/$DN/queue/scheduler > /dev/null 2>&1 || {
    echo "$sched unavailable (modprobe ${sched}-iosched?)"; continue; }
  echo "=== $sched ==="
  sudo DEV=$DEV fio mixed.fio 2>/dev/null | \
    grep -A4 -E '^reader|^writer' | grep -E 'IOPS|clat.*avg|99.00th|BW='
done
```

The reader's latency is the number that matters: it shows how well each scheduler protects a synchronous read from a write flood.

Now the decomposition of §T.8:

```sh
for sched in none mq-deadline kyber bfq; do
  echo $sched | sudo tee /sys/block/$DN/queue/scheduler > /dev/null 2>&1 || continue
  sudo blktrace -d $DEV -o /tmp/bt-$sched &
  sudo DEV=$DEV fio mixed.fio > /dev/null 2>&1
  sudo pkill blktrace; sleep 1
  echo "=== $sched ==="
  btt -i /tmp/bt-$sched 2>/dev/null | grep -A8 'ALL' | head -10
done
```

**Compare I2D (scheduler time) and D2C (device time) across schedulers.** That comparison is the evidence base for a scheduler choice.

Per-scheduler CPU cost:

```sh
for sched in none mq-deadline kyber bfq; do
  echo $sched | sudo tee /sys/block/$DN/queue/scheduler > /dev/null 2>&1 || continue
  echo -n "$sched: "
  sudo fio --name=t --filename=$DEV --direct=1 --rw=randread --bs=4k \
           --iodepth=32 --ioengine=libaio --runtime=10 --time_based \
           --numjobs=$(nproc) --group_reporting 2>/dev/null | \
    grep -oP 'IOPS=\K[0-9.]+[km]?|cpu\s+:\s+usr=\K[0-9.]+|sys=\K[0-9.]+' | tr '\n' ' '
  echo
done
```

On a fast device, `none` wins on throughput and CPU. That is the point.

Tunables:

```sh
echo mq-deadline | sudo tee /sys/block/$DN/queue/scheduler > /dev/null
ls /sys/block/$DN/queue/iosched/
for f in /sys/block/$DN/queue/iosched/*; do
  printf "%-20s %s\n" "$(basename $f)" "$(cat $f)"
done

for we in 500 5000 50000; do
  echo $we | sudo tee /sys/block/$DN/queue/iosched/write_expire > /dev/null
  echo -n "write_expire=$we: reader clat="
  sudo DEV=$DEV fio mixed.fio 2>/dev/null | grep -A3 '^reader' | \
    grep -oP 'clat.*avg=\K[0-9.]+' | head -1
done
```

---

### Lab 64.2 — Demonstrate deceptive idleness

§T.3's problem, made visible.

```c
// SPDX-License-Identifier: GPL-2.0
/* seqread.c: strictly synchronous sequential reads -- one at a time. */
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <unistd.h>

int main(int argc, char **argv)
{
	int fd = open(argv[1], O_RDONLY | O_DIRECT);
	off_t off = argc > 2 ? atoll(argv[2]) : 0;
	void *buf;
	struct timespec a, b;
	long n = 2000, i;

	posix_memalign(&buf, 4096, 65536);
	clock_gettime(CLOCK_MONOTONIC, &a);
	for (i = 0; i < n; i++) {
		if (pread(fd, buf, 65536, off) <= 0) { off = 0; continue; }
		off += 65536;     /* strictly sequential, strictly synchronous */
	}
	clock_gettime(CLOCK_MONOTONIC, &b);
	printf("%.1f MB/s\n",
	       (double)n * 65536 / 1048576 /
	       ((b.tv_sec-a.tv_sec) + (b.tv_nsec-a.tv_nsec)/1e9));
	close(fd);
	return 0;
}
```

```sh
gcc -O2 -o seqread seqread.c

# Force rotational behaviour so idling matters
echo 1 | sudo tee /sys/block/$DN/queue/rotational

for sched in none mq-deadline bfq; do
  echo $sched | sudo tee /sys/block/$DN/queue/scheduler > /dev/null 2>&1 || continue
  echo "=== $sched ==="
  echo -n "  1 reader:  "; sudo ./seqread $DEV 0
  echo -n "  2 readers: "
  ( sudo ./seqread $DEV 0 & sudo ./seqread $DEV 1073741824 & wait ) 2>/dev/null
done
```

With two interleaved sequential readers, a non-idling scheduler seeks back and forth. BFQ's idling should preserve much more of the single-reader rate.

Watch the idling:

```sh
echo bfq | sudo tee /sys/block/$DN/queue/scheduler > /dev/null
cat /sys/block/$DN/queue/iosched/slice_idle
cat /sys/block/$DN/queue/iosched/slice_idle_us

sudo trace-cmd record -e bfq:\* -- sh -c \
  '(sudo ./seqread '$DEV' 0 & sudo ./seqread '$DEV' 1073741824 & wait)' 2>/dev/null
sudo trace-cmd report | grep -i idle | head -10

# Disable idling: the benefit should vanish
echo 0 | sudo tee /sys/block/$DN/queue/iosched/slice_idle > /dev/null
echo -n "bfq, slice_idle=0, 2 readers: "
( sudo ./seqread $DEV 0 & sudo ./seqread $DEV 1073741824 & wait ) 2>/dev/null
```

**That is why `slice_idle=0` is right on SSDs and wrong on HDDs.**

---

### Lab 64.3 — Kyber's feedback loop

```sh
sudo modprobe kyber-iosched 2>/dev/null
echo kyber | sudo tee /sys/block/$DN/queue/scheduler
ls /sys/block/$DN/queue/iosched/
cat /sys/block/$DN/queue/iosched/read_lat_nsec
cat /sys/block/$DN/queue/iosched/write_lat_nsec
```

Watch the depth adapt:

```sh
sudo cat /sys/kernel/debug/block/$DN/sched/read_tokens 2>/dev/null
sudo cat /sys/kernel/debug/block/$DN/sched/write_tokens 2>/dev/null

sudo DEV=$DEV fio mixed.fio > /dev/null 2>&1 &
FIO=$!
for i in $(seq 1 10); do
  echo -n "read_tokens: "
  sudo grep -h 'depth=' /sys/kernel/debug/block/$DN/sched/read_tokens 2>/dev/null
  echo -n "write_tokens: "
  sudo grep -h 'depth=' /sys/kernel/debug/block/$DN/sched/write_tokens 2>/dev/null
  sleep 2
done
kill $FIO 2>/dev/null
```

Tighten the target and watch the trade:

```sh
for lat in 100000 1000000 10000000; do
  echo $lat | sudo tee /sys/block/$DN/queue/iosched/read_lat_nsec > /dev/null
  echo "=== read_lat_nsec=$lat ($((lat/1000000)) ms) ==="
  sudo DEV=$DEV fio mixed.fio 2>/dev/null | grep -A4 -E '^reader|^writer' | \
    grep -E 'IOPS|clat.*avg|BW='
done
```

A tighter read target means lower read latency and less write throughput. **The knob does exactly what it says**, which is more than can be said for most tunables.

Kyber's two signals (§1.4):

```sh
sudo ls /sys/kernel/debug/block/$DN/sched/
sudo cat /sys/kernel/debug/block/$DN/sched/*latency* 2>/dev/null | head -20
```

---

### Lab 64.4 — cgroup v2: `io.max` and `io.weight`

```sh
mount | grep cgroup2
cat /sys/fs/cgroup/cgroup.controllers
echo "+io +memory" | sudo tee /sys/fs/cgroup/cgroup.subtree_control

sudo mkdir -p /sys/fs/cgroup/{hog,vip}
echo "+io" | sudo tee /sys/fs/cgroup/hog/cgroup.subtree_control 2>/dev/null
MAJMIN=$(lsblk -ndo MAJ:MIN $DEV | head -1)
echo "device: $MAJMIN"
```

Hard limits:

```sh
# Baseline
sudo fio --name=t --filename=$DEV --direct=1 --rw=read --bs=1M \
         --runtime=10 --time_based 2>/dev/null | grep -oP 'BW=\K[^ ]+'

# Limited to 10 MB/s
echo "$MAJMIN rbps=10485760" | sudo tee /sys/fs/cgroup/hog/io.max
sudo sh -c "echo \$\$ > /sys/fs/cgroup/hog/cgroup.procs; \
  exec fio --name=t --filename=$DEV --direct=1 --rw=read --bs=1M \
           --runtime=10 --time_based" 2>/dev/null | grep -oP 'BW=\K[^ ]+'

cat /sys/fs/cgroup/hog/io.stat
```

Watch the throttling:

```sh
sudo bpftrace -e '
kprobe:blk_throtl_bio { @throttled = count(); }
kprobe:throtl_schedule_next_dispatch { @delayed = count(); }
interval:s:3 { print(@throttled); print(@delayed);
               clear(@throttled); clear(@delayed); }' &

sudo sh -c "echo \$\$ > /sys/fs/cgroup/hog/cgroup.procs; \
  exec fio --name=t --filename=$DEV --direct=1 --rw=read --bs=1M \
           --runtime=10 --time_based" > /dev/null 2>&1
```

Weights with BFQ:

```sh
echo bfq | sudo tee /sys/block/$DN/queue/scheduler
echo "" | sudo tee /sys/fs/cgroup/hog/io.max     # clear the hard limit

echo 100 | sudo tee /sys/fs/cgroup/hog/io.bfq.weight 2>/dev/null || \
  echo "default 100" | sudo tee /sys/fs/cgroup/hog/io.weight
echo 900 | sudo tee /sys/fs/cgroup/vip/io.bfq.weight 2>/dev/null || \
  echo "default 900" | sudo tee /sys/fs/cgroup/vip/io.weight

run_in() {
  sudo sh -c "echo \$\$ > /sys/fs/cgroup/$1/cgroup.procs; \
    exec fio --name=$1 --filename=$DEV --direct=1 --rw=randread --bs=4k \
             --iodepth=16 --ioengine=libaio --runtime=15 --time_based" \
    2>/dev/null | grep -oP 'IOPS=\K[0-9.]+[km]?'
}

( echo -n "hog (weight 100): "; run_in hog ) &
( echo -n "vip (weight 900): "; run_in vip ) &
wait
```

Roughly a 9:1 split. Verify:

```sh
cat /sys/fs/cgroup/hog/io.stat
cat /sys/fs/cgroup/vip/io.stat
```

---

### Lab 64.5 — `io.cost` and `io.latency`

```sh
echo none | sudo tee /sys/block/$DN/queue/scheduler

# The cost model
cat /sys/fs/cgroup/io.cost.model
cat /sys/fs/cgroup/io.cost.qos

# Enable it
echo "$MAJMIN enable=1 ctrl=auto" | sudo tee /sys/fs/cgroup/io.cost.qos
cat /sys/fs/cgroup/io.cost.qos

echo "default 100" | sudo tee /sys/fs/cgroup/hog/io.weight
echo "default 900" | sudo tee /sys/fs/cgroup/vip/io.weight

( echo -n "hog: "; run_in hog ) &
( echo -n "vip: "; run_in vip ) &
wait
```

Watch iocost's internal state:

```sh
sudo ls /sys/kernel/debug/block/$DN/rqos/ 2>/dev/null
sudo cat /sys/kernel/debug/block/$DN/rqos/*/* 2>/dev/null | head -30

sudo trace-cmd record -e iocost:\* -- sh -c '
  (sudo sh -c "echo \$\$ > /sys/fs/cgroup/hog/cgroup.procs; \
     exec fio --name=h --filename='$DEV' --direct=1 --rw=randread --bs=4k \
     --iodepth=16 --ioengine=libaio --runtime=10 --time_based" &
   sudo sh -c "echo \$\$ > /sys/fs/cgroup/vip/cgroup.procs; \
     exec fio --name=v --filename='$DEV' --direct=1 --rw=randread --bs=4k \
     --iodepth=16 --ioengine=libaio --runtime=10 --time_based" &
   wait)' > /dev/null 2>&1
sudo trace-cmd report | head -30
# iocost_iocg_activate, iocost_ioc_vrate_adj, iocost_iocg_idle
```

The vrate adjustment is the outer feedback loop of §T.6(d):

```sh
sudo trace-cmd report | grep vrate_adj | head -10
```

Cost model coefficients — mixing block sizes shows why the model is needed:

```sh
for bs in 4k 64k 1m; do
  echo -n "bs=$bs: "
  sudo sh -c "echo \$\$ > /sys/fs/cgroup/hog/cgroup.procs; \
    exec fio --name=t --filename=$DEV --direct=1 --rw=randread --bs=$bs \
             --iodepth=16 --ioengine=libaio --runtime=8 --time_based" \
    2>/dev/null | grep -oP 'IOPS=\K[0-9.]+[km]?|BW=\K[^ ]+' | tr '\n' ' '
  echo
done
# IOPS and BW differ wildly -- neither alone is a fair currency (T.6(d)).
```

`io.latency`:

```sh
echo "" | sudo tee /sys/fs/cgroup/io.cost.qos 2>/dev/null
echo "$MAJMIN target=2" | sudo tee /sys/fs/cgroup/vip/io.latency

( sudo sh -c "echo \$\$ > /sys/fs/cgroup/hog/cgroup.procs; \
    exec fio --name=hog --filename=$DEV --direct=1 --rw=write --bs=1M \
             --iodepth=32 --ioengine=libaio --runtime=20 --time_based" \
    > /dev/null 2>&1 ) &
sleep 2
echo -n "vip latency with protection: "
sudo sh -c "echo \$\$ > /sys/fs/cgroup/vip/cgroup.procs; \
  exec fio --name=vip --filename=$DEV --direct=1 --rw=randread --bs=4k \
           --iodepth=1 --runtime=10 --time_based" 2>/dev/null | \
  grep -oP 'clat.*avg=\K[0-9.]+'
wait

cat /sys/fs/cgroup/vip/io.stat
cat /sys/fs/cgroup/hog/io.stat
```

PSI, the right saturation signal:

```sh
cat /proc/pressure/io
cat /sys/fs/cgroup/hog/io.pressure
cat /sys/fs/cgroup/vip/io.pressure

# Under load
( sudo sh -c "echo \$\$ > /sys/fs/cgroup/hog/cgroup.procs; \
    exec fio --name=t --filename=$DEV --direct=1 --rw=randwrite --bs=4k \
             --iodepth=64 --ioengine=libaio --runtime=15 --time_based" \
    > /dev/null 2>&1 ) &
for i in $(seq 1 5); do cat /proc/pressure/io; sleep 3; done
wait
```

---

### Lab 64.6 — Writeback throttling

```sh
cat /sys/block/$DN/queue/wbt_lat_usec
echo none | sudo tee /sys/block/$DN/queue/scheduler
sudo mkfs.ext4 -qF $DEV && sudo mkdir -p /mnt/w && sudo mount $DEV /mnt/w

cat > wbt.fio <<'EOF'
[global]
directory=/mnt/w
runtime=20
time_based
group_reporting=0

[reader]
rw=randread
bs=4k
size=256m
iodepth=1
ioengine=psync
numjobs=1

[bgwriter]
rw=write
bs=1M
size=1g
iodepth=1
ioengine=psync
numjobs=4
EOF

for lat in 0 75 2000 20000; do
  echo $lat | sudo tee /sys/block/$DN/queue/wbt_lat_usec > /dev/null
  echo "=== wbt_lat_usec=$lat ==="
  sudo fio wbt.fio 2>/dev/null | grep -A4 -E '^reader|^bgwriter' | \
    grep -E 'IOPS|clat.*avg|99.00th|BW='
done
```

`wbt_lat_usec=0` disables it and read latency should degrade noticeably.

Watch the scaling:

```sh
echo 2000 | sudo tee /sys/block/$DN/queue/wbt_lat_usec > /dev/null
sudo trace-cmd record -e wbt:\* -- sudo fio wbt.fio > /dev/null 2>&1
sudo trace-cmd report | head -30
# wbt_step: the scale factor going up and down
sudo trace-cmd report | grep -oP 'step=\K-?\d+' | sort | uniq -c
```

And the interaction of §T.7:

```sh
for sched in none mq-deadline kyber bfq; do
  echo $sched | sudo tee /sys/block/$DN/queue/scheduler > /dev/null 2>&1 || continue
  echo -n "$sched: wbt_lat_usec = "
  cat /sys/block/$DN/queue/wbt_lat_usec
done
# kyber and bfq disable wbt automatically: two controllers would fight.
```

---

### Lab 64.7 — cgroup writeback: verify it actually works

This is the most commonly misconfigured thing in §T.6.

```sh
echo none | sudo tee /sys/block/$DN/queue/scheduler
echo "+io +memory" | sudo tee /sys/fs/cgroup/cgroup.subtree_control

# Does the filesystem support cgroup writeback?
grep -r CONFIG_CGROUP_WRITEBACK /boot/config-$(uname -r)
mount | grep /mnt/w
```

```sh
# Buffered writes without cgroup writeback: NOT attributed
echo "$MAJMIN wbps=5242880" | sudo tee /sys/fs/cgroup/hog/io.max

echo "=== buffered write under io.max ==="
sudo sh -c "echo \$\$ > /sys/fs/cgroup/hog/cgroup.procs; \
  exec dd if=/dev/zero of=/mnt/w/buffered bs=1M count=500 conv=fsync" 2>&1 | tail -1

echo "=== direct write under io.max ==="
sudo sh -c "echo \$\$ > /sys/fs/cgroup/hog/cgroup.procs; \
  exec dd if=/dev/zero of=/mnt/w/direct bs=1M count=500 oflag=direct" 2>&1 | tail -1

cat /sys/fs/cgroup/hog/io.stat
```

If buffered is much faster than direct, the writes are escaping the limit. Check why:

```sh
# 1. Is memory enabled on the same hierarchy?
cat /sys/fs/cgroup/cgroup.subtree_control
cat /sys/fs/cgroup/hog/memory.current

# 2. Does the filesystem do cgroup writeback?
sudo bpftrace -e '
kprobe:wbc_attach_and_unlock_inode { @attach = count(); }
kprobe:wbc_account_cgroup_owner    { @account = count(); }
interval:s:5 { print(@attach); print(@account);
               clear(@attach); clear(@account); }' &
sudo sh -c "echo \$\$ > /sys/fs/cgroup/hog/cgroup.procs; \
  exec dd if=/dev/zero of=/mnt/w/b2 bs=1M count=200 conv=fsync" > /dev/null 2>&1
```

Non-zero counts mean cgroup writeback is active. Compare filesystems:

```sh
for fs in ext4 btrfs xfs; do
  sudo umount /mnt/w 2>/dev/null
  sudo mkfs.$fs -qf $DEV > /dev/null 2>&1 || sudo mkfs.$fs -qF $DEV > /dev/null 2>&1
  sudo mount $DEV /mnt/w
  echo -n "$fs: "
  sudo sh -c "echo \$\$ > /sys/fs/cgroup/hog/cgroup.procs; \
    exec dd if=/dev/zero of=/mnt/w/x bs=1M count=200 conv=fsync" 2>&1 | \
    grep -oP '[0-9.]+ [GM]B/s'
done
```

Which writeback threads are doing the work:

```sh
ps aux | grep -E 'kworker.*flush|writeback'
cat /sys/kernel/debug/bdi/$MAJMIN/stats 2>/dev/null
```

---

### Lab 64.8 — A complete diagnosis

Build a realistic problem and diagnose it by §T.9's procedure rather than by guessing.

```sh
sudo umount /mnt/w 2>/dev/null
sudo mkfs.ext4 -qF $DEV && sudo mount $DEV /mnt/w
sudo mkdir -p /mnt/w/{db,backup}

# The "database": small synchronous reads and fsyncs
cat > db.fio <<'EOF'
[db]
directory=/mnt/w/db
rw=randrw
rwmixread=70
bs=8k
size=512m
iodepth=4
ioengine=libaio
direct=1
fsync=16
runtime=60
time_based
EOF

# The "backup": large sequential buffered reads and writes
cat > backup.fio <<'EOF'
[backup]
directory=/mnt/w/backup
rw=write
bs=1M
size=2g
iodepth=1
ioengine=psync
numjobs=4
runtime=60
time_based
EOF
```

Step 1 — baseline:

```sh
echo "=== db alone ==="
sudo fio db.fio 2>/dev/null | grep -E 'read:|write:|clat.*avg|99.00th'
```

Step 2 — with contention:

```sh
sudo fio backup.fio > /dev/null 2>&1 &
BG=$!
sleep 3
echo "=== db with backup running ==="
sudo fio db.fio 2>/dev/null | grep -E 'read:|write:|clat.*avg|99.00th'
kill $BG 2>/dev/null; wait 2>/dev/null
```

Step 3 — measure, do not guess:

```sh
sudo fio backup.fio > /dev/null 2>&1 &
BG=$!
sleep 2
iostat -xz 2 3 $DEV
cat /proc/pressure/io
sudo blktrace -d $DEV -o /tmp/diag &
sudo fio db.fio > /dev/null 2>&1
sudo pkill blktrace; sleep 1
kill $BG 2>/dev/null; wait 2>/dev/null

btt -i /tmp/diag 2>/dev/null | grep -A8 ALL | head -10
sudo biolatency-bpfcc -D 10 1 2>/dev/null | head -30
```

Step 4 — is it ordering or capacity?

```
If D2C is large and rising -> the DEVICE is saturated. No scheduler helps.
If I2D is large            -> QUEUEING. A scheduler or wbt may help.
If both are small but the  -> the problem is upstream (fsync, metadata,
   application is slow        the filesystem).
```

Step 5 — apply the right fix and re-measure:

```sh
# Fix A: wbt (if buffered writes are the problem)
echo 2000 | sudo tee /sys/block/$DN/queue/wbt_lat_usec > /dev/null

# Fix B: a scheduler that protects reads
echo mq-deadline | sudo tee /sys/block/$DN/queue/scheduler > /dev/null

# Fix C: isolation via cgroups (the RIGHT fix for competing workloads)
sudo mkdir -p /sys/fs/cgroup/{database,batch}
echo "$MAJMIN target=5" | sudo tee /sys/fs/cgroup/database/io.latency

run_db()  { sudo sh -c "echo \$\$ > /sys/fs/cgroup/database/cgroup.procs; exec fio db.fio"; }
run_bk()  { sudo sh -c "echo \$\$ > /sys/fs/cgroup/batch/cgroup.procs; exec fio backup.fio"; }

run_bk > /dev/null 2>&1 &
BG=$!
sleep 3
echo "=== db with io.latency protection ==="
run_db 2>/dev/null | grep -E 'read:|clat.*avg|99.00th'
kill $BG 2>/dev/null; wait 2>/dev/null
```

Step 6 — the full matrix, so you have evidence:

```sh
for sched in none mq-deadline kyber bfq; do
  for wbt in 0 2000; do
    echo $sched | sudo tee /sys/block/$DN/queue/scheduler > /dev/null 2>&1 || continue
    echo $wbt | sudo tee /sys/block/$DN/queue/wbt_lat_usec > /dev/null 2>&1
    sudo fio backup.fio > /dev/null 2>&1 &
    BG=$!; sleep 2
    printf "%-12s wbt=%-5s db p99: " $sched $wbt
    sudo fio db.fio 2>/dev/null | grep -oP '99.00th=\[\s*\K[0-9]+' | head -1
    kill $BG 2>/dev/null; wait 2>/dev/null
  done
done
```

**This table is what a scheduler decision should be based on.** Not folklore.

---

## 3. Mastery drills

1. State the difference between scheduling and QoS precisely. Then classify each of the following as one or the other: `nomerges`, `io.max`, `wbt_lat_usec`, `slice_idle`, `read_expire`, `io.cost.qos`.

2. Derive the sequential/random cost ratio for a 7200 RPM disk and for a modern NVMe device. Then compute the maximum throughput improvement an elevator could deliver on each.

3. Explain deceptive idleness precisely. Compute the throughput of two interleaved sequential readers on a 15 ms-seek disk with and without a 3 ms idle window.

4. `mq-deadline`'s read and write expiries differ by 10×. Justify the asymmetry from first principles, then construct the workload where it is wrong.

5. Kyber separates total latency from I/O latency. Construct the instability that results from using only total latency, and prove the separation fixes it.

6. BFQ's per-request cost makes it unsuitable above some IOPS threshold. Estimate that threshold from a CPU-time measurement, and state what would have to change to raise it.

7. `io.max` wastes capacity; `io.weight` does not. Construct the scenario where `io.max` is nonetheless the correct choice.

8. `io.cost` needs a cost model. Prove that neither IOPS nor bytes alone is a fair currency, using a concrete two-workload example with numbers.

9. `io.latency` is not an SLO. State exactly what it guarantees, and construct the situation where a cgroup with a 10 ms target sees 200 ms.

10. cgroup writeback requires three conditions. For each, construct the misconfiguration and describe the symptom an operator would see.

11. `wbt` is disabled when kyber or bfq is active. Explain the interference, and name two other places in the kernel where nested feedback loops are deliberately avoided.

12. Given a `btt` output showing Q2G=2 µs, G2I=1 µs, I2D=8 ms, D2C=200 µs — diagnose it, and give the fix.

13. Design a complete QoS configuration for: one latency-sensitive database, three batch analytics jobs, and a nightly backup, on one NVMe device. Justify every choice and state how you would verify it works.

---

## 4. Further reading

**Kernel documentation**

- `Documentation/block/deadline-iosched.rst` ★★★ — short and complete.
- `Documentation/block/bfq-iosched.rst` ★★★ — long, detailed, and written by BFQ's authors. Explains the heuristics and when each applies.
- `Documentation/block/queue-sysfs.rst` ★★★ — every tunable.
- `Documentation/admin-guide/cgroup-v2.rst` ★★★ — **the io controller section is the authoritative reference for §T.6.** Read it carefully; the semantics of `io.latency` in particular are easy to misread.
- `Documentation/accounting/psi.rst` ★★★ — §T.8's pressure signals.
- `block/blk-iocost.c`'s header comment ★★★ — a ~300-line design document explaining the cost model, the vrate loop, and the debt/donation mechanics. **One of the best pieces of in-tree design documentation.**
- `block/blk-wbt.c`'s header comment — §T.7.

**Papers**

- Iyer & Druschel, "Anticipatory scheduling: A disk scheduling framework to overcome deceptive idleness in synchronous I/O," SOSP 2001 ★★★ — **§T.3.** Short, surprising, and the insight generalises well beyond disks.
- Valente & Andreolini, "Improving Application Responsiveness with the BFQ Disk I/O Scheduler," SYSTOR 2012 ★★★ — BFQ's design and its interactivity heuristics.
- Bennett & Zhang, "WF²Q: Worst-case Fair Weighted Fair Queueing," INFOCOM 1996 — the fair-queueing theory BFQ is built on; originally for networks.
- Demers, Keshav, Shenker, "Analysis and Simulation of a Fair Queueing Algorithm," SIGCOMM 1989 ★★★ — the origin of fair queueing; worth reading to see how thoroughly the network and storage problems are the same.
- Shakshober, "Choosing an I/O Scheduler for Red Hat Enterprise Linux" — dated but a good example of evidence-based selection.
- Axboe's blk-mq and scheduler presentations at LSFMM and Vault.
- Heo, Chinner et al., the iocost design discussions on linux-block ★★★ — §T.6(d)'s reasoning.

**LWN**

- "Two new block I/O schedulers for 4.12" (kyber and bfq) ★★★
- "The BFQ I/O scheduler" ★★★
- "Block-layer I/O scheduling" and the mq-deadline coverage
- "Anticipatory scheduling" (2003) and "The end of the anticipatory scheduler" (2009) ★★★ — both sides of §T.3's story
- "Toward less-annoying background writeback" ★★★ — §T.7's introduction
- "The io.latency controller" and "Controlling block I/O with iocost" ★★★
- "PSI: pressure stall information" ★★★ — §T.8
- "cgroup writeback" ★★★ — §T.6's most common misconfiguration
- "Fixing the I/O priority problem" and the `ioprio` coverage
- Facebook/Meta's resource-control posts and the `resctl-demo` material ★★★ — iocost, calibrated, in production

**Source reading order**

1. `Documentation/admin-guide/cgroup-v2.rst`'s io section, then `Documentation/block/deadline-iosched.rst`.
2. `block/mq-deadline.c` ★★★ — **read the whole file.** It is ~1100 lines and it is the clearest scheduler in the tree.
3. `block/elevator.c` — the interface and how switching works.
4. `block/kyber-iosched.c` ★★★ — the feedback loop; ~1000 lines.
5. `block/blk-wbt.c` ★★★ — §T.7; small and self-contained.
6. `block/blk-throttle.c` — the token bucket.
7. `block/blk-iocost.c` ★★★ — read the header comment first, then `calc_vtime_cost`, `ioc_timer_fn`.
8. `block/bfq-iosched.c` — last, and only if you need it. Start with the file's header comment, which is extensive.

**Tools**

- `fio` ★★★ — **the only reasonable way to compare schedulers.** Learn `--rwmixread`, `--iodepth`, `--numjobs`, `--fsync`, per-job sections, and `--group_reporting=0`.
- `btt` ★★★ — the Q2G/G2I/I2D/D2C decomposition (§T.8). This is the single most valuable diagnostic in this chapter.
- `iostat -xz 1` ★★★ — but read `aqu-sz` and `await`, **not `%util`**.
- `/proc/pressure/io` and `io.pressure` ★★★ — the correct saturation signal.
- `biolatency -D`, `biosnoop`, `biotop`, `bitesize` (bcc) ★★★
- `trace-cmd record -e block:\* -e bfq:\* -e wbt:\* -e iocost:\*` ★★★
- `/sys/kernel/debug/block/DEV/sched/` and `/rqos/` ★★★
- `resctl-bench` / `iocost_coef_gen.py` (in `tools/cgroup/`) — calibrate iocost for your device
- `systemd-run --scope -p IOWeight=... -p IOReadBandwidthMax=...` — the convenient way to apply cgroup io settings ad hoc
- `ionice` — sets `ioprio`, which only BFQ and mq-deadline's priority classes honour

---

→ Next: [65-write-block-driver.md](65-write-block-driver.md)

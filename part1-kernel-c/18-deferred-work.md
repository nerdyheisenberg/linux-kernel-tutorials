# Chapter 18 — Deferred Work: Softirqs, Tasklets, Workqueues, BH

> **Goal:** know which deferral mechanism to use, why softirqs cannot be allowed to starve
> userspace, and why every workqueue that can be entered during memory reclaim needs a rescuer
> thread. This chapter is about **forward progress guarantees**, which is the hardest part of
> kernel design.

---

## Theory & First Principles

### T.0 — Start here: four ways to "do it later", and how to choose

Ch. 17 established that interrupt handlers must defer. But defer *to what*? The kernel offers
four mechanisms and they are not interchangeable — choosing wrong produces bugs that appear
under load, months later.

```c
/* You are in an interrupt handler. You need to process a packet.
   Pick one: */

tasklet_schedule(&t);                    /* ① */
raise_softirq(NET_RX_SOFTIRQ);           /* ② */
queue_work(wq, &w);                      /* ⁢ */
return IRQ_WAKE_THREAD;                  /* ⁣ */
```

**The one question that decides it: can the deferred work sleep?**

```
                    Can it sleep?
                   /            \
                 NO              YES
                 │                │
        atomic context       process context
        │          │          │            │
     softirq    tasklet    workqueue   threaded IRQ
        ②         ①           ⁢            ⁣
```

And the properties that follow from that split are stark:

| | softirq ② | tasklet ① | workqueue ⁢ | threaded IRQ ⁣ |
|---|---|---|---|---|
| Can sleep | **no** | **no** | yes | yes |
| Runs concurrently on many CPUs | **yes** | **no** (serialized per tasklet) | yes | one per IRQ |
| Latency | lowest | low | higher (scheduled) | scheduled |
| Priority controllable | no (fixed table) | no | via the workqueue | **yes** (`chrt`) |
| Can be preempted | no | no | yes | yes |
| New code should use it | **rarely** | **never** | often | **usually** |

**Two of those rows are the whole chapter.**

**"Runs concurrently on many CPUs."** A softirq can execute on every CPU at once, so its data
needs real locking. A tasklet is serialized against itself — which sounds convenient and is
the reason tasklets are deprecated: **that serialization is a global property that makes them
a scalability bottleneck**, and it encourages code that is accidentally dependent on it
(§T.3). It is a classic case of an API whose convenience *is* its defect.

**"Priority controllable."** A softirq runs at a fixed point in a fixed table on interrupt
return. You cannot prioritize it, pin it, or preempt it. A threaded IRQ is a *task* — it has
a PID, you can `chrt` it, set its affinity, and see it in `ps`. That difference is what makes
real-time possible (Ch. 103 §T.4) and it is why `request_threaded_irq` is the modern answer.

**Now the failure mode that motivates §T.2.** Softirqs run on interrupt return, so a busy
system can generate them faster than it drains them:

```
  IRQ → raise softirq → return → run softirqs → another IRQ arrives → …
                                       ↑                              │
                                       └───────────────────────────┘
              userspace never runs again.  LIVELOCK.
```

The kernel's answer is a budget: after `MAX_SOFTIRQ_RESTART` iterations or 2 ms, stop and
hand the rest to `ksoftirqd`, a normal schedulable thread. **Watch it happen on a busy
network box:**

```bash
cat /proc/softirqs                # counts per CPU per type
top -H | grep ksoftirqd           # if this is burning CPU, you are at the limit
cat /proc/net/softnet_stat        # column 3 = times the NAPI budget was exhausted
```

**The general principle, which recurs in the scheduler, the block layer, and writeback:**

> Any mechanism that does work on behalf of an interrupt must have a **budget**, and a
> **fallback to a schedulable context** when the budget is exhausted. Without it, high load
> does not degrade performance — it starves everything else completely.

---

### T.1 Deferral is scheduling under a different name

Ch. 17 established *why* work must be deferred out of hardirq context. This chapter is about
the harder question: **once deferred, under what scheduling discipline does the work run?**

Every deferral mechanism is a scheduler with four design parameters:

1. **Context** — what is the deferred work allowed to do? (sleep? take mutexes? fault?)
2. **Priority** — does it preempt user tasks? other kernel work? itself?
3. **Concurrency** — can two instances run simultaneously? on the same CPU? on different CPUs?
4. **Forward progress** — under what conditions is it *guaranteed* to eventually run?

Linux offers four points in this space, and the differences are not cosmetic:

| | softirq | tasklet | workqueue | threaded IRQ |
|---|---|---|---|---|
| Context | softirq | softirq | **task** | **task** |
| Can sleep | no | no | **yes** | **yes** |
| Preempts user tasks | **yes, unconditionally** | yes | no (scheduled normally) | yes (RT prio) |
| Same handler on 2 CPUs | **yes** | **no** (serialized) | yes (unless ordered) | no (per-IRQ) |
| Per-CPU | yes | yes (runs where scheduled) | configurable | configurable |
| Number available | **fixed, 10** | unlimited | unlimited | one per IRQ |
| Forward progress | ksoftirqd fallback | via softirq | rescuer thread | scheduler |
| Recommended for new code | **no** (core only) | **no** (deprecated) | **yes** | **yes** |

### T.2 Softirqs: static priority, unbounded execution, and the starvation problem

A softirq is a **statically registered, per-CPU, non-preemptible callback** with a fixed
priority given by its index:

```c
/* include/linux/interrupt.h — this list is closed; you may not add to it */
enum {
	HI_SOFTIRQ = 0,      /* high-priority tasklets */
	TIMER_SOFTIRQ,       /* timer wheel */
	NET_TX_SOFTIRQ,
	NET_RX_SOFTIRQ,      /* ★ the big one */
	BLOCK_SOFTIRQ,       /* block I/O completion */
	IRQ_POLL_SOFTIRQ,
	TASKLET_SOFTIRQ,
	SCHED_SOFTIRQ,       /* load balancing */
	HRTIMER_SOFTIRQ,
	RCU_SOFTIRQ,         /* RCU callbacks */
	NR_SOFTIRQS
};
```

The list is **fixed at compile time and closed to drivers**. That is a deliberate
architectural decision: softirqs preempt *everything* in user space and cannot be controlled
by the scheduler, so allowing arbitrary drivers to register them would make system latency
unbounded and unattributable.

**Where softirqs run** (three places, and knowing all three is essential):

1. `irq_exit_rcu()` — on the way out of a hardirq, on the same CPU. The common case.
2. `local_bh_enable()` — when the last `spin_unlock_bh()`/`local_bh_enable()` unnests.
3. `ksoftirqd/N` — a per-CPU `SCHED_NORMAL` kthread, as a fallback.

**The starvation problem.** Softirq handlers can re-raise themselves (NAPI polls again, RCU
has more callbacks). If the CPU keeps running softirqs on every interrupt exit, and interrupts
keep arriving, **userspace never runs**. This is Ch. 17's livelock, reappearing one layer up.

Linux's answer is an **admission-control heuristic** in `__do_softirq()`:

```c
#define MAX_SOFTIRQ_TIME  msecs_to_jiffies(2)
#define MAX_SOFTIRQ_RESTART 10
/* Keep processing while: time budget not exhausted AND restart count not exceeded
 * AND no task needs rescheduling. Otherwise: wakeup_softirqd() and return. */
```

So after ~2 ms or 10 restarts, pending softirq work is **handed to `ksoftirqd`**, which is a
normal scheduled task at nice 0 — and therefore competes fairly with userspace instead of
preempting it. This is a **priority demotion under load**: cheap when load is low, fair when
load is high.

Consequences you will observe in production:

- High `%si` (softirq) in `mpstat` with low `ksoftirqd` CPU → softirqs are running in
  interrupt exit; latency-sensitive tasks are being preempted invisibly.
- High `ksoftirqd` CPU → you crossed the threshold; throughput is now bounded by the
  scheduler. Often accompanied by packet drops.
- Both are symptoms of the same overload; which one you see tells you where you are on the
  curve.

`Documentation/core-api/local_ops.rst` and the `softirq.c` comments are worth reading, but
the real text is the code: `kernel/softirq.c` is only ~1000 lines and is worth reading
completely.

### T.3 Why tasklets are deprecated (and the lesson in it)

A tasklet is a dynamically-registered callback that runs in softirq context with one extra
guarantee:

> **The same tasklet never runs on two CPUs simultaneously.**

That serialization is why people loved them: it acts as an *implicit lock*, so handler code
could touch shared state without taking one. That is also exactly why they are deprecated.

The problems, in order of severity:

1. **The implicit serialization is an undocumented lock.** It cannot be validated by lockdep,
   it doesn't appear in the code, and it silently constrains what can be parallelized later.
   The kernel has spent 20 years learning that *implicit* synchronization is technical debt
   (compare: `local_lock` in Ch. 16 T.7, which made the opposite choice explicit).
2. **All tasklets share one softirq priority level** (`TASKLET_SOFTIRQ`), so an unrelated
   slow tasklet delays yours. No isolation, no accounting, no attribution.
3. **They run in softirq context** — cannot sleep — while nearly every modern driver needs to
   sleep (regmap, runtime PM, DMA mapping on some paths).
4. **On PREEMPT_RT** they are forced into threads anyway, so the latency argument for using
   them evaporates.
5. The API has a long history of use-after-free on teardown (`tasklet_kill()` races), which
   `tasklet_unlock_wait()`/`tasklet_disable_in_atomic()` only partly fixed.

The official position (`Documentation/` and repeated maintainer statements) is:
**do not add new tasklet users.** Convert to a threaded IRQ or a workqueue. If you genuinely
need atomic-context deferral with softirq latency, use a dedicated mechanism
(`irq_poll`) or argue for it explicitly. There is an ongoing tree-wide removal effort — a
good source of first patches.

> **The general lesson, worth carrying into your own designs:** an API whose main selling
> point is an *implicit, invisible guarantee* will eventually be removed, because the
> guarantee cannot be checked, cannot be documented at the call site, and blocks future
> parallelism.

### T.4 Workqueues and the CMWQ concurrency-management theory

Workqueues run work items in **task context**, so they may sleep. The interesting theory is
in how many kernel threads back them.

**The old design (pre-2.6.36)** created one kthread *per workqueue per CPU*. With hundreds of
workqueues on a 64-CPU machine that is tens of thousands of threads — enormous memory waste
and PID exhaustion — and yet it *still* couldn't guarantee concurrency, because a single
thread per (wq, cpu) means one blocking work item stalls every other item on that queue.

**Concurrency Managed Workqueues (CMWQ)**, Tejun Heo, 2010, solves both by decoupling the
*user-visible workqueue* from the *worker pool*:

```
   user-visible:   struct workqueue_struct  (many; cheap; just a policy + a pwq per pool)
                          │
                          ▼
   implementation: struct worker_pool       (few: 2 per CPU [normal + highpri]
                                             + N unbound pools by attribute)
                          │
                          ▼
                   struct worker            (kthreads, created/destroyed dynamically)
```

The **concurrency management** algorithm is the elegant part. The invariant CMWQ maintains:

> **Exactly one runnable worker per CPU per pool.**

- A worker that is *running* keeps the pool's `nr_running` at 1. No new worker is started —
  because the CPU is already busy and more threads would only add context switches.
- The moment a worker **blocks** (the scheduler tells the workqueue core via
  `sched_submit_work`/`wq_worker_sleeping` hooks in `kernel/sched/core.c`), `nr_running`
  drops to 0 and the pool **immediately wakes or creates another worker**.
- When the blocked worker wakes up, `nr_running` goes back up and no further workers start.

The result: **concurrency is created exactly and only when it is needed** — i.e. when work
blocks — with no configuration and no idle thread hoarding. Idle workers are reaped after a
timeout (`IDLE_WORKER_TIMEOUT`, 300 s).

This is a genuinely beautiful piece of design: it requires a *scheduler hook*, which most
thread-pool implementations don't have access to, and that's exactly why an in-kernel pool
can do something userspace pools cannot. (Windows' "concurrency-managed" IOCP thread pools
use the same trick; Go's runtime does the analogous thing with `M`s and `P`s.)

### T.5 Forward-progress guarantees and the rescuer thread

Here is a deadlock that CMWQ creates and must then solve.

Worker pools **allocate new workers with `kthread_create()`**, which allocates memory. Now
consider: the system is out of memory; memory reclaim queues a work item (e.g. writeback);
all existing workers of that pool are blocked; the pool tries to create a new worker; the
creation needs memory; memory requires reclaim; reclaim needs that work item to run. **Circular
dependency → system hangs.**

The fix is a **pre-allocated rescuer thread**. A workqueue declared with `WQ_MEM_RECLAIM` gets
one kthread created at workqueue-creation time (when memory was available). If the pool cannot
create a worker in time, the rescuer is dispatched to execute the queue's pending items.

> **Rule, and reviewers will enforce it:** *any workqueue that may be used on a memory-reclaim
> path must be created with `WQ_MEM_RECLAIM`.* Block drivers, filesystems, writeback, and
> anything a swap write can reach.

This is a concrete instance of a general principle: **a system that needs resource X to free
resource X must pre-reserve X.** The same principle produces `mempool` (Ch. 11, T.7),
`PF_MEMALLOC` reserves, `bio_set` pools, and emergency page reserves. Recognizing this pattern
family is senior-level.

The debug knob `CONFIG_WQ_WATCHDOG` detects stalled pools and prints
`BUG: workqueue lockup - pool cpus=... stuck for Ns!` — one of the most useful diagnostics in
the kernel.

### T.6 Flush deadlocks and the dependency graph

`flush_work(w)` waits for `w` to finish. `flush_workqueue(wq)` waits for everything currently
queued. These create *scheduling dependencies*, and dependencies form cycles:

```
Work item A (on wq1) takes lock L, then does flush_work(B)
Work item B (on wq1) waits for lock L
→ If A and B are on the same single-threaded workqueue, B can never run. Deadlock.
```

Or the subtler one:

```
Work A calls flush_workqueue(wq2)
Work on wq2 calls flush_workqueue(wq1)
→ classic ABBA, but across workqueues
```

Workqueues participate in **lockdep**: `flush_work()` and `flush_workqueue()` acquire a
lockdep map, so these cycles are reported as normal lock inversions. That integration is why
`CONFIG_PROVE_LOCKING` catches workqueue deadlocks — a fact many people don't know.

Practical rules:
- Never `flush_workqueue()` from a work item on the same workqueue.
- Never hold a lock across `flush_work()` if the work takes that lock.
- Prefer `cancel_work_sync()`/`cancel_delayed_work_sync()` over flushing, and **always** call
  one of them before freeing the object containing the `struct work_struct`. Forgetting this
  is the workqueue equivalent of the missing `rcu_barrier()` (Ch. 15).
- `flush_scheduled_work()` (the system-wide flush) is **deprecated and being removed** — it
  waits for *unrelated* work and is a deadlock magnet.

### T.7 Ordering, concurrency limits, and unbound pools

Three policy dimensions you choose at creation time:

**Bound vs unbound.** Bound workqueues run work on the CPU that queued it — good for cache
locality, bad for CPU-intensive work (it can't be load-balanced and it adds latency to that
CPU). `WQ_UNBOUND` releases the work to the scheduler, which can place it anywhere. Rule of
thumb: *short and cache-sensitive → bound; long or CPU-bound → unbound.*

**Max active.** `max_active` bounds how many items of a workqueue may execute concurrently
per CPU (or per pool for unbound). `alloc_ordered_workqueue()` is exactly `max_active == 1` +
`WQ_UNBOUND`, giving **strict FIFO execution with no concurrency** — the correct choice when
work items have a required order (device state machines, firmware upload sequences).

**Priority.** `WQ_HIGHPRI` uses the high-priority worker pool, whose workers run at
nice -20 and are woken ahead of normal ones.

**Affinity (6.6+).** Unbound workqueues gained `affinity_scope` (`cpu`, `smt`, `cache`,
`numa`, `system`) configurable via sysfs — an explicit locality/utilization dial, again the
Ch. 16 cache theory surfacing in an API.

```bash
ls /sys/devices/virtual/workqueue/                 # per-wq tunables
cat /sys/devices/virtual/workqueue/nvme-wq/max_active
cat /sys/devices/virtual/workqueue/*/affinity_scope
sudo cat /sys/kernel/debug/workqueue/              # WQ debug (if enabled)
```

### T.8 BH (bottom half) disabling, and what `_bh` actually means

`local_bh_disable()` does **not** disable interrupts. It increments a per-CPU softirq
nesting counter, which prevents softirqs from being *executed* on this CPU (they still get
raised and stay pending). So:

- `spin_lock_bh(&l)` = `local_bh_disable()` + `spin_lock(&l)`. Use it when the lock is also
  taken from softirq context.
- Hardirqs still fire during a `_bh` region; they just don't run softirqs on exit.
- `local_bh_enable()` **runs pending softirqs immediately** if the count reaches zero. So an
  innocuous-looking `spin_unlock_bh()` can execute arbitrary softirq work — including your
  own — right there. This surprises people and matters for latency accounting.
- On PREEMPT_RT, `local_bh_disable()` becomes a per-CPU lock, and softirqs run in threads.

### T.9 Choosing: a decision procedure

```
Does the work need to sleep, allocate with GFP_KERNEL, take a mutex, or do I/O?
├─ YES ──▶ Task context required.
│          Is it tied to an interrupt and latency-critical?
│          ├─ YES → threaded IRQ (request_threaded_irq)
│          └─ NO  → workqueue
│                   Must items run strictly in order?      → alloc_ordered_workqueue()
│                   Can it be entered during memory reclaim? → add WQ_MEM_RECLAIM
│                   Is it CPU-intensive / long-running?     → add WQ_UNBOUND
│                   Is it latency-sensitive?                → add WQ_HIGHPRI
│                   Otherwise                               → system_wq via schedule_work()
└─ NO ───▶ Atomic context is acceptable. Do you *really* need softirq latency (< ~50 µs)?
           ├─ NO  → use a workqueue anyway. (This is the right answer 90% of the time.)
           └─ YES → Is this core networking/block/timer/RCU code?
                    ├─ YES → the existing softirq for your subsystem
                    └─ NO  → irq_poll, or a threaded IRQ with RT priority.
                             Do NOT add a tasklet. Do NOT add a softirq.
```

---

## 1. Concept — the APIs

### 1.1 Workqueues (what you will actually use)

```c
#include <linux/workqueue.h>

struct my_dev {
	struct work_struct         work;
	struct delayed_work        poll;
	struct workqueue_struct   *wq;
	...
};

static void my_work_fn(struct work_struct *w)
{
	struct my_dev *d = container_of(w, struct my_dev, work);

	mutex_lock(&d->lock);        /* legal: task context */
	do_slow_thing(d);
	mutex_unlock(&d->lock);
}

/* init */
INIT_WORK(&d->work, my_work_fn);
INIT_DELAYED_WORK(&d->poll, my_poll_fn);

d->wq = alloc_workqueue("mydev-%s", WQ_MEM_RECLAIM | WQ_UNBOUND, 0, dev_name(dev));
/* or: alloc_ordered_workqueue("mydev-ctl", WQ_MEM_RECLAIM); */

/* queue */
queue_work(d->wq, &d->work);                       /* no-op if already queued */
queue_delayed_work(d->wq, &d->poll, msecs_to_jiffies(100));
mod_delayed_work(d->wq, &d->poll, msecs_to_jiffies(10));   /* reschedule */
queue_work_on(cpu, d->wq, &d->work);               /* pin to a CPU */

/* teardown — ORDER MATTERS */
cancel_delayed_work_sync(&d->poll);
cancel_work_sync(&d->work);
destroy_workqueue(d->wq);          /* drains everything first */
/* only NOW may you free the object containing d->work */
```

System-provided workqueues (avoid for anything long-running; you're sharing with everyone):

| Queue | Helper | Notes |
|---|---|---|
| `system_wq` | `schedule_work()` | multi-CPU, `max_active` high |
| `system_highpri_wq` | `queue_work(system_highpri_wq, ...)` | nice -20 workers |
| `system_long_wq` | | for work expected to take a while |
| `system_unbound_wq` | | not bound to a CPU |
| `system_freezable_wq` | | frozen during suspend |
| `system_power_efficient_wq` | | respects `workqueue.power_efficient=1` |
| `system_bh_wq` (6.9+) | | **BH workqueues** — run in softirq context; the sanctioned tasklet replacement |

**BH workqueues (6.9+)** are the officially blessed tasklet replacement: the workqueue API
(`INIT_WORK`, `queue_work`, `cancel_work_sync`, lockdep integration) with softirq-context
execution. If you truly need atomic deferral, use these, not tasklets.

### 1.2 Softirq (for reference — you will read this, not write it)

```c
open_softirq(MY_SOFTIRQ, my_action);   /* core code only; index must be in the enum */
raise_softirq(MY_SOFTIRQ);             /* from anywhere */
raise_softirq_irqoff(MY_SOFTIRQ);      /* when IRQs already off — avoids a redundant save */

local_bh_disable(); ... local_bh_enable();
spin_lock_bh(&lock); ... spin_unlock_bh(&lock);
```

### 1.3 Tasklet (for reading legacy code only)

```c
static void my_tasklet_fn(struct tasklet_struct *t)
{
	struct my_dev *d = from_tasklet(d, t, tl);
	...
}
tasklet_setup(&d->tl, my_tasklet_fn);   /* modern form; old tasklet_init takes ulong data */
tasklet_schedule(&d->tl);
tasklet_kill(&d->tl);                   /* MUST be called before freeing d */
```

---

## 2. Internals

### 2.1 Source map

```
kernel/softirq.c              ★ __do_softirq, ksoftirqd, local_bh_*, tasklets
kernel/workqueue.c            ★ CMWQ — 7000 lines, extremely well commented
kernel/workqueue_internal.h   struct worker, worker_pool
kernel/sched/core.c           wq_worker_running / wq_worker_sleeping hooks (the CMWQ trick)
include/linux/workqueue.h     the API + flags
include/linux/interrupt.h     softirq enum, tasklet API
lib/irq_poll.c                the NAPI-style polled deferral
Documentation/core-api/workqueue.rst   ★ read this completely, it's excellent
```

### 2.2 `__do_softirq()`, annotated

```c
asmlinkage __visible void __softirq_entry __do_softirq(void)
{
	unsigned long end = jiffies + MAX_SOFTIRQ_TIME;   /* 2 ms budget */
	int max_restart = MAX_SOFTIRQ_RESTART;            /* 10 */

	pending = local_softirq_pending();
	softirq_handle_begin();            /* __local_bh_disable_ip() */

restart:
	set_softirq_pending(0);            /* clear, so new raises are visible */
	local_irq_enable();                /* ★ softirqs run with IRQs ENABLED */

	while ((softirq_bit = ffs(pending))) {
		h->action(h);              /* ← your handler; trace_softirq_entry/exit here */
		pending >>= softirq_bit;
	}

	local_irq_disable();
	pending = local_softirq_pending();
	if (pending) {
		if (time_before(jiffies, end) && !need_resched() && --max_restart)
			goto restart;      /* keep going: cheap path */
		wakeup_softirqd();         /* ★ demote to ksoftirqd: fair path */
	}
	softirq_handle_end();
}
```

Three things worth noting: softirqs run **with hardirqs enabled** (so they are interruptible,
just not preemptible); `need_resched()` is checked, so a waiting RT task cuts the loop short;
and the demotion is the admission control from T.2.

### 2.3 CMWQ structures

```c
struct workqueue_struct {                 /* user-visible */
	struct list_head      pwqs;       /* per-CPU (or per-node) pool_workqueues */
	unsigned int          flags;      /* WQ_UNBOUND | WQ_MEM_RECLAIM | ... */
	int                   saved_max_active;
	struct worker        *rescuer;    /* WQ_MEM_RECLAIM */
	struct wq_device     *wq_dev;     /* sysfs */
	char                  name[WQ_NAME_LEN];
};

struct pool_workqueue {                   /* the (wq, pool) binding */
	struct worker_pool   *pool;
	struct workqueue_struct *wq;
	int                   nr_active, max_active;
	struct list_head      inactive_works;   /* throttled by max_active */
};

struct worker_pool {                      /* the actual thread pool */
	raw_spinlock_t        lock;
	int                   cpu;        /* -1 for unbound */
	struct list_head      worklist;
	int                   nr_workers, nr_idle;
	atomic_t              nr_running;  /* ★ the concurrency-management counter */
	struct list_head      idle_list;
	struct timer_list     idle_timer;
	struct workqueue_attrs *attrs;
};
```

The `work_struct->data` field is another pointer-low-bits trick: it stores either the
`pool_workqueue` pointer (while queued) or the last pool id (while idle), plus the
`WORK_STRUCT_PENDING` bit. That's how `queue_work()` can be a no-op for an already-queued item
without any extra state, and how `cancel_work_sync()` knows where to look.

### 2.4 Observability

```bash
ps -eo pid,class,rtprio,pri,comm | grep -E 'ksoftirqd|kworker|irq/'
cat /proc/softirqs                            # per-CPU counts per softirq type
mpstat -P ALL 1                               # %si (softirq) and %hi (hardirq) per CPU

# Which work functions are running?
sudo perf trace -e 'workqueue:*' -a sleep 3
sudo bpftrace -e 'tracepoint:workqueue:workqueue_execute_start { @[ksym(args->function)] = count(); }'

# Work item latency (queue → execute) and duration:
sudo bpftrace -e '
tracepoint:workqueue:workqueue_queue_work   { @q[args->work] = nsecs; }
tracepoint:workqueue:workqueue_execute_start /@q[args->work]/ {
	@delay_us[ksym(args->function)] = hist((nsecs - @q[args->work])/1000);
	delete(@q[args->work]); @s[args->work] = nsecs; }
tracepoint:workqueue:workqueue_execute_end  /@s[args->work]/ {
	@run_us[ksym(args->function)] = hist((nsecs - @s[args->work])/1000);
	delete(@s[args->work]); }'

# Softirq latency:
sudo bpftrace -e 'tracepoint:irq:softirq_entry { @s[cpu]=nsecs; }
                  tracepoint:irq:softirq_exit /@s[cpu]/ {
                     @[args->vec] = hist((nsecs-@s[cpu])/1000); }'
```

`kworker/u8:3` naming: `u` = unbound, `8` = pool id (or nr_cpus for unbound), `3` = worker id.
`kworker/2:1H` = CPU 2, worker 1, **H**ighpri pool. Reading these names correctly tells you
instantly which pool is busy.

---

## 3. Practice

### 3.1 A driver with correct workqueue lifetime

```c
// SPDX-License-Identifier: GPL-2.0
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/workqueue.h>
#include <linux/platform_device.h>
#include <linux/slab.h>

struct mydev {
	struct device            *dev;
	struct workqueue_struct  *wq;
	struct work_struct        irq_work;
	struct delayed_work       health;
	struct mutex              lock;
	bool                      shutting_down;
};

static void mydev_irq_work(struct work_struct *w)
{
	struct mydev *d = container_of(w, struct mydev, irq_work);

	guard(mutex)(&d->lock);            /* legal: we are in task context */
	if (d->shutting_down)
		return;
	mydev_drain_fifo(d);               /* may sleep, may do I2C/SPI */
}

static void mydev_health(struct work_struct *w)
{
	struct mydev *d = container_of(to_delayed_work(w), struct mydev, health);

	scoped_guard(mutex, &d->lock) {
		if (d->shutting_down)
			return;
		mydev_check_temperature(d);
	}
	queue_delayed_work(d->wq, &d->health, msecs_to_jiffies(1000));  /* re-arm */
}

static irqreturn_t mydev_isr(int irq, void *data)
{
	struct mydev *d = data;

	mydev_mask_irq(d);
	queue_work(d->wq, &d->irq_work);
	return IRQ_HANDLED;
}

static int mydev_probe(struct platform_device *pdev)
{
	struct mydev *d;
	int ret;

	d = devm_kzalloc(&pdev->dev, sizeof(*d), GFP_KERNEL);
	if (!d)
		return -ENOMEM;
	d->dev = &pdev->dev;
	mutex_init(&d->lock);
	INIT_WORK(&d->irq_work, mydev_irq_work);
	INIT_DELAYED_WORK(&d->health, mydev_health);

	/*
	 * WQ_MEM_RECLAIM: this device sits under a block device, so our work
	 * can be reached from the writeback/swap path. Without a rescuer we
	 * could deadlock under memory pressure. (Ch. 18, T.5)
	 * Ordered: register programming must happen strictly in sequence.
	 */
	d->wq = alloc_ordered_workqueue("mydev/%s", WQ_MEM_RECLAIM,
					dev_name(&pdev->dev));
	if (!d->wq)
		return -ENOMEM;

	platform_set_drvdata(pdev, d);

	ret = mydev_hw_init(d);
	if (ret)
		goto err_wq;

	ret = devm_request_irq(&pdev->dev, platform_get_irq(pdev, 0),
			       mydev_isr, 0, dev_name(&pdev->dev), d);
	if (ret)
		goto err_hw;

	queue_delayed_work(d->wq, &d->health, msecs_to_jiffies(1000));
	return 0;

err_hw:
	mydev_hw_teardown(d);
err_wq:
	destroy_workqueue(d->wq);
	return ret;
}

static void mydev_remove(struct platform_device *pdev)
{
	struct mydev *d = platform_get_drvdata(pdev);

	/* devm_ frees the IRQ after remove() returns, so stop the source first. */
	mydev_mask_irq(d);

	scoped_guard(mutex, &d->lock)
		d->shutting_down = true;   /* stop the self-rearming health work */

	cancel_delayed_work_sync(&d->health);   /* ★ sync: waits for a running instance */
	cancel_work_sync(&d->irq_work);
	destroy_workqueue(d->wq);               /* drains, then frees */
	mydev_hw_teardown(d);
}
```

The `shutting_down` flag exists because `cancel_delayed_work_sync()` on a **self-rearming**
work item is racy on its own: the work can re-queue itself after the cancel. The flag +
cancel is the standard correct pattern. This is a bug found in real drivers constantly.

### 3.2 Reproduce softirq starvation

```bash
# Terminal 1 — generate heavy NET_RX softirq load
sudo pktgen  # or: iperf3 -c <host> -P 32  from another machine

# Terminal 2 — watch the demotion happen
mpstat -P ALL 1                    # watch %si climb
watch -d 'grep -E "NET_RX|TIMER|RCU" /proc/softirqs'
pidstat -p $(pgrep -d, ksoftirqd) 1

# Terminal 3 — measure the latency cost to a normal task
sudo cyclictest -m -p 0 -i 1000 -l 20000 -h 400 -q
```
You should see `%si` rise, then `ksoftirqd` CPU rise as the 2 ms/10-restart budget is
exceeded, and cyclictest's max latency jump. **This experiment makes T.2 concrete.**

### 3.3 Provoke and read a workqueue lockup

```c
/* In a module, on a WQ without WQ_MEM_RECLAIM: */
static void hog(struct work_struct *w) { ssleep(120); }
/* Queue max_active+1 of these, then queue one more and wait. */
```
With `CONFIG_WQ_WATCHDOG=y`, after `workqueue.watchdog_thresh` (default 30 s):
```
BUG: workqueue lockup - pool cpus=3 node=0 flags=0x0 nice=0 stuck for 33s!
```
Read the dump: it prints the pool, its workers, and the pending work functions.

---

## 3.4 Extended practice

### Lab 18.A — Watch softirq demotion to `ksoftirqd` happen (T.2)

```bash
# Terminal 1 — generate NET_RX softirq load
sudo modprobe pktgen 2>/dev/null
# simplest: from another host
iperf3 -c <this-host> -P 32 -t 60
# or locally with a veth pair + flood ping:
sudo ip link add v0 type veth peer name v1
sudo ip link set v0 up; sudo ip link set v1 up
sudo ping -f -s 1400 <peer>

# Terminal 2 — watch the transition
watch -n1 'grep -E "NET_RX|TIMER|RCU|SCHED" /proc/softirqs'
mpstat -P ALL 1                      # %si climbing = softirq in irq-exit
pidstat -p $(pgrep -d, ksoftirqd) 1  # ksoftirqd CPU rising = DEMOTED

# Terminal 3 — measure the latency cost to an ordinary task
sudo cyclictest -m -p 0 -i 1000 -l 30000 -h 400 -q

# Terminal 4 — see the budget being exhausted
sudo bpftrace -e '
kprobe:wakeup_softirqd { @demotions = count(); }
tracepoint:irq:softirq_entry { @s[cpu] = nsecs; }
tracepoint:irq:softirq_exit /@s[cpu]/ {
	@dur_us[args->vec] = hist((nsecs - @s[cpu])/1000); delete(@s[cpu]); }
interval:s:5 { print(@demotions); clear(@demotions); }'
```
**What to record:** the `%si` level at which `ksoftirqd` starts accumulating CPU, and the
cyclictest max at each stage. Then tune and re-measure:
```bash
sysctl net.core.netdev_budget net.core.netdev_budget_usecs
sudo sysctl -w net.core.netdev_budget=600 net.core.netdev_budget_usecs=8000
```
Explain the result in terms of the 2 ms / 10-restart admission control in T.2.

### Lab 18.B — Prove the CMWQ concurrency invariant (T.4)

```c
// SPDX-License-Identifier: GPL-2.0
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/workqueue.h>
#include <linux/delay.h>
#include <linux/ktime.h>

static int nwork = 16, sleep_ms = 200, spin_ms;
module_param(nwork, int, 0644);
module_param(sleep_ms, int, 0644);   /* blocking work */
module_param(spin_ms, int, 0644);    /* CPU-bound work */

static struct work_struct *works;
static struct workqueue_struct *wq;
static atomic_t concurrent, max_concurrent;

static void fn(struct work_struct *w)
{
	int c = atomic_inc_return(&concurrent);
	int old;

	do {
		old = atomic_read(&max_concurrent);
	} while (c > old && atomic_cmpxchg(&max_concurrent, old, c) != old);

	pr_info("work %ld on %s cpu%d (concurrent=%d)\n",
		w - works, current->comm, smp_processor_id(), c);

	if (sleep_ms)
		msleep(sleep_ms);            /* BLOCKS → pool starts another worker */
	if (spin_ms) {
		u64 end = ktime_get_ns() + spin_ms * 1000000ULL;

		while (ktime_get_ns() < end)
			cpu_relax();         /* does NOT block → no new worker */
	}
	atomic_dec(&concurrent);
}

static int __init cm_init(void)
{
	int i;

	wq = alloc_workqueue("cmdemo", WQ_MEM_RECLAIM, 0);
	if (!wq)
		return -ENOMEM;
	works = kcalloc(nwork, sizeof(*works), GFP_KERNEL);
	for (i = 0; i < nwork; i++) {
		INIT_WORK(&works[i], fn);
		queue_work(wq, &works[i]);
	}
	msleep(2000);
	pr_info("max concurrent workers: %d\n", atomic_read(&max_concurrent));
	return 0;
}
static void __exit cm_exit(void)
{
	int i;

	for (i = 0; i < nwork; i++)
		cancel_work_sync(&works[i]);
	destroy_workqueue(wq);
	kfree(works);
}
module_init(cm_init); module_exit(cm_exit);
MODULE_LICENSE("GPL");
```
```bash
# (a) Blocking work: expect MANY workers (one per blocked item)
sudo insmod cmdemo.ko nwork=16 sleep_ms=200 spin_ms=0
ps -eo comm | grep -c kworker ; dmesg | tail -20 ; sudo rmmod cmdemo

# (b) CPU-bound work: expect ~ONE runnable worker per CPU
sudo insmod cmdemo.ko nwork=16 sleep_ms=0 spin_ms=200
ps -eo comm | grep -c kworker ; dmesg | tail -20 ; sudo rmmod cmdemo

# (c) Ordered workqueue: expect exactly 1, in FIFO order
#     (change alloc_workqueue to alloc_ordered_workqueue and rebuild)
```
**This directly demonstrates T.4's invariant**: concurrency is created only when work
*blocks*. Then find the scheduler hooks that make it possible:
```bash
grep -n 'wq_worker_running\|wq_worker_sleeping\|wq_worker_tick' kernel/sched/core.c
```

### Lab 18.C — Provoke a workqueue lockup and read it (T.5)

```c
static void hog(struct work_struct *w) { ssleep(120); }
/* queue max_active + 2 of these on a wq created WITHOUT WQ_MEM_RECLAIM */
```
```bash
./scripts/config -e WQ_WATCHDOG --set-val DEFAULT_HUNG_TASK_TIMEOUT 30
# runtime:
echo 30 | sudo tee /sys/module/workqueue/parameters/watchdog_thresh
sudo insmod hog.ko
sleep 40 && dmesg | grep -A30 'workqueue lockup'

# Inspect pool state while it's stuck:
sudo ls /sys/devices/virtual/workqueue/
sudo cat /sys/devices/virtual/workqueue/hogwq/max_active
ps -eLo pid,comm,wchan:30 | grep kworker
sudo cat /proc/*/stack 2>/dev/null | head -40
# sysrq dumps all workqueue state:
echo t | sudo tee /proc/sysrq-trigger; dmesg | tail -60
```
Then add `WQ_MEM_RECLAIM` and show the rescuer thread (`kworker/R-hogwq` or `hogwq`) appears
and drains the queue:
```bash
ps -eo pid,comm | grep -i hogwq
```

### Lab 18.D — The self-rearming teardown race (§3.1)

```c
static void rearm_fn(struct work_struct *w)
{
	struct dev *d = container_of(to_delayed_work(w), struct dev, poll);

	do_poll(d);
	queue_delayed_work(d->wq, &d->poll, msecs_to_jiffies(1));  /* self-rearms */
}

/* BROKEN teardown: */
cancel_delayed_work_sync(&d->poll);   /* the running instance may re-queue AFTER this */
kfree(d);                             /* → use-after-free */

/* CORRECT teardown: */
WRITE_ONCE(d->shutting_down, true);   /* fn() checks this before re-queueing */
cancel_delayed_work_sync(&d->poll);
destroy_workqueue(d->wq);
kfree(d);
```
```bash
# Build with KASAN + DEBUG_OBJECTS_WORK and loop:
./scripts/config -e KASAN -e DEBUG_OBJECTS -e DEBUG_OBJECTS_WORK -e DEBUG_OBJECTS_FREE
while true; do sudo insmod rearm.ko && sudo rmmod rearm; done
dmesg -w | grep -A25 'KASAN\|ODEBUG'
```
The broken version fails within seconds. **This bug is present in real drivers today** —
finding one is a legitimate patch.

### Lab 18.E — Lockdep catches workqueue flush inversions (T.6)

```c
static DEFINE_MUTEX(m);
static struct workqueue_struct *wq;

static void work_a(struct work_struct *w)
{
	mutex_lock(&m);
	mutex_unlock(&m);
}

static void deadlock_path(void)
{
	mutex_lock(&m);
	flush_work(&wa);      /* waits for work_a, which needs m → ABBA */
	mutex_unlock(&m);
}
```
```bash
./scripts/config -e PROVE_LOCKING
sudo insmod flushdl.ko
dmesg | grep -A40 'possible circular locking dependency'
grep -n 'lockdep_map\|lock_map_acquire' kernel/workqueue.c | head -20
```
Note that **lockdep reports it before it hangs** — that's the value of the workqueue's
lockdep integration, which many developers don't know exists.

### Lab 18.F — Convert a tasklet to a BH workqueue (T.3)

```bash
git grep -l 'tasklet_schedule\|tasklet_setup' drivers/ | head -20
git log --oneline --grep='tasklet' --grep='convert\|remove\|replace' --all-match | head -20
```
Pick one and convert:
```c
/* Before */
struct tasklet_struct tl;
tasklet_setup(&tl, my_tasklet_fn);
tasklet_schedule(&tl);
tasklet_kill(&tl);

/* After — BH workqueue (6.9+): same softirq context, full workqueue API + lockdep */
struct work_struct bh_work;
INIT_WORK(&bh_work, my_bh_fn);
queue_work(system_bh_wq, &bh_work);      /* or system_bh_highpri_wq */
cancel_work_sync(&bh_work);

/* Or better — a threaded IRQ / ordinary workqueue if it can sleep */
```
```bash
grep -n 'system_bh_wq\|WQ_BH' include/linux/workqueue.h kernel/workqueue.c | head
```
Verify: same context (`in_softirq()` still true), same latency (measure with bpftrace), plus
lockdep coverage you didn't have before. **This is one of the best first-patch opportunities
in the kernel right now.**

### Lab 18.G — Full workqueue observability

```bash
# Which work functions run, how often, how long, and how long they waited:
sudo bpftrace -e '
tracepoint:workqueue:workqueue_queue_work    { @q[args->work] = nsecs; }
tracepoint:workqueue:workqueue_execute_start /@q[args->work]/ {
	@delay_us[ksym(args->function)] = hist((nsecs - @q[args->work])/1000);
	delete(@q[args->work]); @s[args->work] = nsecs; }
tracepoint:workqueue:workqueue_execute_end   /@s[args->work]/ {
	@run_us[ksym(args->function)]   = hist((nsecs - @s[args->work])/1000);
	delete(@s[args->work]); }'

# Pool and worker inventory
ps -eo pid,pri,ni,comm | grep -E 'kworker|ksoftirqd' | head -30
#   kworker/u8:3   = Unbound pool, worker 3
#   kworker/2:1H   = CPU 2, worker 1, Highpri pool
#   kworker/R-nvme = Rescuer for the nvme workqueue

# Per-workqueue tunables
for w in /sys/devices/virtual/workqueue/*/; do
  echo "$(basename $w): max_active=$(cat $w/max_active 2>/dev/null) \
affinity_scope=$(cat $w/affinity_scope 2>/dev/null) \
nice=$(cat $w/nice 2>/dev/null)"
done

# Tune an unbound wq's locality (6.6+) and measure
echo cache | sudo tee /sys/devices/virtual/workqueue/nvme-wq/affinity_scope
```

---

## 4. Mastery drills

1. **Read `kernel/workqueue.c`'s header comment** (~200 lines) and
   `Documentation/core-api/workqueue.rst` completely. Then explain CMWQ's concurrency
   invariant and where the scheduler hooks are (`wq_worker_sleeping`/`wq_worker_running` in
   `kernel/sched/core.c`).

2. **Find the rescuer in action.** Grep for `WQ_MEM_RECLAIM` in `fs/`, `block/`, `drivers/md/`.
   For three of them, explain the exact reclaim path that would deadlock without it.

3. **Tasklet conversion.** `git grep -l tasklet_schedule drivers/ | head -20`. Pick one, and
   write the patch converting it to a BH workqueue or a threaded IRQ. Check
   `git log --grep="tasklet" --oneline` for how others did it, and follow that pattern.
   **This is one of the best available first-patch opportunities in the kernel.**

4. **Missing cancel.** Find (or write) a driver that frees a struct containing a
   `work_struct` without `cancel_work_sync()`. Reproduce the use-after-free under KASAN with
   an insmod/rmmod loop.

5. **Ordering requirement.** Explain when `alloc_ordered_workqueue()` is required rather than
   just convenient. Find a real user in the tree and explain its sequencing constraint.

6. **Measure CMWQ.** Write a module that queues N work items that each `msleep(100)`.
   Instrument how many `kworker` threads appear (`ps`). Repeat with items that spin instead
   of sleeping. Explain the difference from the T.4 invariant.

7. **Flush deadlock.** Deliberately construct the flush-inversion from T.6 on an ordered
   workqueue. Run with `CONFIG_PROVE_LOCKING=y` and read the report. Note that lockdep
   catches it *before* it hangs.

8. **BH workqueues.** Read the 6.9 `system_bh_wq` merge (`git log --grep="BH workqueue"`).
   Explain what problem it solves that neither tasklets nor normal workqueues solved.

9. **Design question.** A NIC driver must, on link-down: (a) stop TX immediately, (b) drain
   in-flight DMA, (c) reprogram the PHY over MDIO (sleeps), (d) notify the netdev core.
   Which mechanism for each step, and why? What ordering constraints exist between them?

---

## 5. Further reading

**Kernel docs:**
- `Documentation/core-api/workqueue.rst` — **the single best doc in this area**
- `Documentation/core-api/irq/concepts.rst` (softirq interaction)
- `kernel/softirq.c` — read the whole file; it is short and load-bearing

**LWN (this area is unusually well covered):**
- "Concurrency-managed workqueues" (Corbet, 2010) — the CMWQ design writeup
- "The end of the tasklet?" / "Moving past tasklets"
- "BH workqueues" (2024)
- "Workqueue affinity scopes" (6.6)
- "Softirqs, the RT tree, and the future" — the long-running debate about replacing softirqs
- "Bottom halves and the realtime patch set"

**Papers / background:**
- Mogul & Ramakrishnan (1996) — as in Ch. 17; the theory that motivates budgeting
- Any thread-pool sizing literature (Little's Law applied to worker pools) for T.4's
  "one runnable worker" invariant

→ Next: [19-time-timers.md](19-time-timers.md)

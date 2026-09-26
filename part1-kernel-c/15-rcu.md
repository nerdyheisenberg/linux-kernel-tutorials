# Chapter 15 — RCU: Read-Copy-Update, In Depth

> **Goal:** understand RCU well enough to (a) use it correctly without thinking, (b) explain
> grace-period detection to a skeptic, and (c) know when RCU is the *wrong* answer.
> RCU is the single most distinctive piece of engineering in the Linux kernel. There are
> ~15,000 `rcu_read_lock()` call sites. You cannot be senior without this chapter.

---

## Theory & First Principles

> **How to read this section.** T.0 shows why a reader-writer lock fails at the thing it was
> designed for. T.1–T.5 build RCU from that failure. T.6–T.11 are the API, the variants, and
> the obligations.

---

### T.0 — Start here: why `rwlock` fails at read scaling

A reader-writer lock exists so that many readers can proceed in parallel. So read-heavy
workloads should scale linearly with cores. Measure it:

```
 read throughput
     │   ╱╲
     │  ╱   ╲────────────────────     rwlock: peaks at ~4 cores, then DECLINES
     │ ╱
     │╱                                    (measure it yourself: Ch. 105 Lab 5)
     └─────────────────────── cores
```

**Why does a *read* lock not scale for *reads*?** Look at what a reader does:

```c
read_lock(&lock);      /* atomic_inc(&lock->readers)  <- A WRITE */
…read the data…
read_unlock(&lock);    /* atomic_dec(&lock->readers)  <- ANOTHER WRITE */
```

The reader **writes** to the lock word. From Ch. 13 and Ch. 105 §T.3: a write requires the
cacheline in **exclusive** state, which invalidates every other CPU's copy. So *N* readers
generate *N* exclusive acquisitions of one line, continuously.

> **Read sharing is nearly free; write sharing is not.** A reader-writer lock turns a
> read-only workload into a write-heavy one *on the lock word*, and then wonders why it does
> not scale.

This is not an implementation flaw to be optimized away. It is inherent: **any mechanism
where readers announce themselves must have readers write somewhere**, and whatever they
write becomes the bottleneck.

**So ask the radical question:** can readers do *nothing at all*?

```c
rcu_read_lock();                       /* compiles to NOTHING (or preempt_disable) */
p = rcu_dereference(gbl_ptr);          /* an ordinary load */
use(p);
rcu_read_unlock();                     /* compiles to NOTHING */
```

Zero atomics. Zero writes. Zero cache traffic. **Readers are genuinely, exactly free**, and
read throughput is linear in cores forever.

**But now the writer has a problem.** It wants to remove an object:

```c
list_del_rcu(&obj->node);        /* unlink: new readers cannot find it */
kfree(obj);                      /* ← but a reader that found it BEFORE is still using it! */
```

Since readers left no trace, the writer cannot ask "is anyone using this?" — there is nobody
to ask. **The entire design problem of RCU is this one question**, and the answer is the
trick that makes it work:

> Do not track readers. Instead, wait for a **grace period**: a span of time after which
> every CPU has passed through a state in which it provably holds no RCU reference — a
> **quiescent state** (a context switch, an idle period, a return to userspace).
> After that, any reader that could have seen the old pointer has finished.

```
 writer: list_del_rcu(obj)
            │
            ├─ CPU0 ──[reader]────┐
            ├─ CPU1 ────[reader]─│──┐
            ├─ CPU2 ────────────│──│────┐        each │ is a quiescent state
            │                  └──┴────┴──► GRACE PERIOD ENDS
            │                              │
            └─ synchronize_rcu() returns ──┘  -> now kfree(obj) is safe
```

**The trade, stated honestly** — and this is what you say in an interview:

| | |
|---|---|
| **Buys** | Zero reader cost, perfect read scalability, no reader-side failure mode |
| **Costs** | Reclamation is *deferred* (memory held longer, typically 10–100 ms), writers are more expensive, and readers may briefly see stale data |

That is correct exactly when **reads massively outnumber writes** and the data is reachable
by pointer — which describes the dcache, the routing table, the module list, netfilter rules,
and the device tree. It is wrong for write-heavy data, and reaching for RCU there is a
recognizable novice mistake.

**The one thing to carry into §T.1:** RCU does not make a race disappear. It makes the
*reclamation* safe while leaving the staleness visible — so "my reader saw an object that has
been deleted" is not a bug, it is the design, and your code must tolerate it.

---

### T.1 The read-mostly problem

Consider the kernel's routing table, dentry cache, module list, or a driver's device list.
Measured access ratios are commonly **10⁴:1 to 10⁷:1 reads to writes**. Under such a
distribution, any synchronization that makes readers *write* (a lock counter, a hazard
pointer, an epoch pin) puts the entire cost of the system on the 99.9999% case to protect
the 0.0001% case.

From Ch. 14 T.5 (the Universal Scalability Law), reader-side writes contribute to the
$\kappa N^2$ coherency term — the one that makes throughput *decline* with more CPUs.
So the goal is sharp and quantitative:

> **Make the read side cost literally zero instructions, and pay for it entirely on the
> write side.**

RCU achieves exactly that. On `CONFIG_PREEMPT_NONE=y`, `rcu_read_lock()` compiles to
**nothing at all** — the empty statement. On preemptible kernels it is a non-atomic increment
of a per-task counter. Zero cache-line contention in either case.

This is not a free lunch; it is a *deliberate transfer of cost*, and understanding the bill
is what separates using RCU from understanding it.

### T.2 The three fundamental guarantees

RCU provides exactly three things (`Documentation/RCU/Design/Requirements/Requirements.rst`):

1. **Grace-period guarantee.** If any part of an RCU read-side critical section precedes the
   beginning of a grace period, then all of it precedes the end of that grace period.
   Informally: *`synchronize_rcu()` waits for all pre-existing readers to finish.*
2. **Publish/subscribe guarantee.** `rcu_assign_pointer()` + `rcu_dereference()` ensure that
   a reader which sees the new pointer also sees the fully-initialized object behind it.
   (This is the MP litmus test from Ch. 13, solved with release/address-dependency.)
3. **Memory-ordering guarantee.** RCU primitives supply the necessary barriers; you do not add
   your own.

Everything else — `call_rcu`, `kfree_rcu`, RCU lists, SRCU — is built from these.

Note what RCU does **not** provide: it is **not mutual exclusion**. Multiple readers *and* a
writer run concurrently, by design. RCU protects *existence*, not *consistency*. Writers
still need a lock (or atomics) among themselves.

### T.3 The core trick: quiescent states, not reader tracking

Here is the idea that makes RCU cheap, and it is genuinely clever.

Other SMR schemes ask: *"which readers currently hold a reference to this object?"* That
requires readers to announce themselves — a write — hence the cost.

RCU inverts the question. It asks: *"has every CPU passed through a state in which it
**definitionally cannot** be inside any RCU read-side critical section?"*

Such a state is a **quiescent state (QS)**. In a non-preemptible kernel, a read-side critical
section cannot span a context switch (you may not block inside one). Therefore:

> **A context switch is a quiescent state. So is returning to user mode. So is entering the
> idle loop.**

And crucially — these are events the kernel is *already tracking for other reasons*. The
scheduler already records context switches. The entry/exit code already knows about user-mode
transitions. So QS detection is nearly free: RCU piggybacks on work that already happens.

A **grace period (GP)** is then defined as:

> A time interval during which **every** CPU has passed through **at least one** quiescent
> state.

And the theorem follows immediately:

> **Any RCU read-side critical section that existed at the start of a grace period has
> completed by the end of it.**

*Proof sketch:* a reader on CPU *i* at GP start is inside a critical section. For CPU *i* to
report a QS, it must have executed a context switch / user transition / idle entry — none of
which can occur inside a read-side critical section. Hence the reader must have exited before
CPU *i*'s QS. Every CPU reports a QS before GP end. ∎

That is the entire foundation. Read it twice.

**The counterintuitive consequence:** RCU does not know, and does not care, *which* objects
readers are looking at. It is a coarse, global, time-based argument. This is why the read side
is free and why the write side must wait *milliseconds* — the grace period is bounded by the
slowest CPU's scheduling behaviour, not by the actual readers.

### T.4 RCU in the SMR design space

| Scheme | Reader announces? | Reader cost | GP/reclaim latency | Memory bound |
|---|---|---|---|---|
| Locking | yes (writes lock) | high, contended | immediate | tight |
| Hazard pointers | yes (writes HP slot) | store + fence | short | **bounded** |
| Epoch-based (EBR) | yes (writes epoch) | store | medium | unbounded |
| **RCU** | **no** | **zero** | **long (ms)** | unbounded |
| Reference counting | yes (atomic RMW) | very high | immediate | tight |

RCU sits at the extreme end: minimum reader cost, maximum reclamation latency and memory
overhead. The knobs you get to tune this are `synchronize_rcu_expedited()`,
`rcu_nocbs`, and offloading — all discussed below.

**When RCU is the wrong answer:**
- Write-heavy data (the copy + GP cost dominates).
- Memory-constrained systems where deferred frees are unacceptable.
- You need readers to see a *consistent multi-object snapshot* (RCU gives per-pointer
  atomicity only; you need a seqlock, a versioned structure, or a lock).
- Readers must block/sleep (→ use SRCU).
- You need an immediate, synchronous teardown guarantee (→ refcounts).

### T.5 Why "read-copy-update" is the name

The canonical update discipline for a linked structure:

```
1. READ    the existing element
2. COPY    it into a freshly allocated one
3. UPDATE  the copy
4. PUBLISH the copy with rcu_assign_pointer()   (atomic pointer swap)
5. WAIT    synchronize_rcu()                     (grace period)
6. FREE    the old element
```

Readers at any instant see either the entirely-old or the entirely-new element — never a
torn intermediate. **Multiple versions coexist during the grace period**, which is why RCU is
sometimes classed as a **multi-version concurrency control (MVCC)** scheme, related to what
databases do with snapshot isolation. That framing is useful: RCU readers are lock-free
snapshot readers over a versioned structure.

For *removal* only (no update), you skip copy: unlink, wait, free.
For *insertion* only, you initialize fully, then publish — no grace period needed at all,
because nothing is being freed.

> **Rule:** a grace period is needed when you **remove** something readers could still see.
> Not when you add.

### T.6 Grace-period detection at scale: the `rcu_node` tree

Naïvely, "has every CPU reported?" needs a shared counter or bitmask — a single cache line
touched by all CPUs at every GP. On a 1024-CPU machine that is a disaster.

Tree RCU (`kernel/rcu/tree.c`) solves it with a **combining tree** of `struct rcu_node`:

```
                     ┌────────────┐
                     │  root node │   ← GP starts/ends here
                     └─────┬──────┘
             ┌─────────────┴─────────────┐
      ┌──────▼─────┐               ┌─────▼──────┐
      │ rcu_node   │               │ rcu_node   │   (fan-out 16–64)
      └──┬───┬──┬──┘               └──┬───┬──┬──┘
      cpu0 cpu1 ...                cpu64 cpu65 ...
```

Each CPU reports its QS to its **leaf** node (per-leaf lock, contended by ≤64 CPUs). When all
CPUs under a leaf have reported, the leaf reports to its parent — *once*. Contention at the
root is therefore `O(log N)` in depth and the root is touched only a handful of times per GP,
regardless of CPU count.

This is a standard **combining tree / software combining** technique (Yew, Tzeng, Lawrie,
1987), and it is why RCU scales to thousands of CPUs. Tunables: `RCU_FANOUT`,
`RCU_FANOUT_LEAF`, visible in `/sys/kernel/debug/rcu/`.

Each CPU also keeps a `struct rcu_data` with its callback list, segmented into
"waiting for GP N", "waiting for GP N+1", "ready to invoke" — a **segmented callback list**
(`kernel/rcu/rcu_segcblist.c`) that lets callbacks be batched by GP number and invoked
without re-scanning.

### T.7 Preemptible RCU: when a reader *can* be preempted

`CONFIG_PREEMPT` breaks the T.3 argument: a task inside `rcu_read_lock()` **can** be
preempted, so a context switch is no longer a quiescent state.

Preemptible RCU (Tree-Preempt-RCU) restores the invariant by tracking explicitly:

```c
/* include/linux/rcupdate.h, CONFIG_PREEMPT_RCU */
static inline void rcu_read_lock(void)
{
	__rcu_read_lock();                       /* current->rcu_read_lock_nesting++ */
	rcu_lock_acquire(&rcu_lock_map);         /* lockdep only */
}
```

A non-atomic per-task counter — no shared cache line, so still nearly free. If a task is
preempted while nesting > 0, it is placed on its leaf `rcu_node`'s `blkd_tasks` list, and the
grace period additionally waits for that list to drain. This adds the concept of a
**blocked reader**, and with it **RCU priority boosting**: if a low-priority preempted reader
is holding up a grace period (and hence memory), the kernel boosts it to RT priority to get
it finished (`CONFIG_RCU_BOOST`). That is priority inheritance (Ch. 14 T.8) applied to a
grace period — a lovely bit of cross-pollination.

`CONFIG_PREEMPT_NONE` / `PREEMPT_VOLUNTARY` use classic Tree RCU where `rcu_read_lock()` is
literally `preempt_disable()` or nothing. Since 6.12, `CONFIG_PREEMPT_LAZY` and the
preempt-mode-at-boot work (`preempt=none|voluntary|full|lazy`) have blurred these; the
direction of travel is a single preemptible kernel with runtime policy.

### T.8 The RCU API families — and why there are five

| Flavour | Read side | Readers may sleep? | Wait for | Use |
|---|---|---|---|---|
| **RCU** (vanilla) | `rcu_read_lock()` | no | QS + blocked readers | 95% of uses |
| **SRCU** | `srcu_read_lock(&sp)` | **yes** | per-domain counters | sleepable readers, notifiers |
| **RCU-bh / RCU-sched** | — | — | — | **merged into vanilla in 4.20** |
| **RCU-Tasks** | (implicit) | yes | every task voluntarily context-switches | tracing trampolines |
| **RCU-Tasks-Rude** | (implicit) | yes | every CPU leaves the kernel | ftrace |
| **RCU-Tasks-Trace** | `rcu_read_lock_trace()` | yes | explicit | **BPF sleepable programs** |

The RCU-Tasks family exists for a problem RCU can't otherwise solve: *"has every task left
this piece of trampoline code that I want to free?"* — where the "readers" are arbitrary
instruction sequences with no `rcu_read_lock()`. The answer: wait for every task to
voluntarily context switch, which proves it is no longer executing the old trampoline.
That's a completely different QS definition serving the same theorem. Recognizing this
generalization — *RCU is a framework for "wait until everyone has left the old regime"* — is
the senior-level takeaway.

**SRCU** (Sleepable RCU) deserves its own explanation. It gives up the "readers never write"
property: readers increment one of **two per-CPU counter arrays**, selected by the current
index. The grace period flips the index, waits for the *old* array to drain to zero, flips
again, and waits again (the double flip is required for correctness — a classic subtlety).
Because each SRCU domain (`struct srcu_struct`) is independent, one domain's slow readers
cannot stall another's grace periods — unlike vanilla RCU, where one stuck reader stalls
*everything*. That isolation is the real reason SRCU exists, as much as sleepability.

### T.9 The cost of a grace period, and the three ways to pay it

A normal grace period takes **milliseconds** (it must wait for scheduling events on every
CPU; on a NO_HZ_IDLE system, idle CPUs are handled specially but timing is still coarse).

Three APIs, three cost profiles:

| API | Blocks caller? | Latency | Cost |
|---|---|---|---|
| `synchronize_rcu()` | **yes, sleeps** | ~10–100 ms typical | free for the system, expensive for you |
| `call_rcu(&h, fn)` | no | callback runs after GP | asynchronous; **you must `rcu_barrier()` on module unload** |
| `kfree_rcu(p, rcu)` | no | batched | optimized: no callback function, batched via `kvfree_rcu_bulk` |
| `synchronize_rcu_expedited()` | yes | ~10s of µs | **sends IPIs to every CPU** — disruptive; do not use on hot paths |
| `cond_synchronize_rcu(cookie)` | maybe | — | "wait only if a GP hasn't already elapsed" — `get_state_synchronize_rcu()` |

The **polled API** (`get_state_synchronize_rcu()` / `poll_state_synchronize_rcu()`, and the
`_full` variants) is the modern way to avoid paying for a GP you already got for free — used
heavily in `mm/` and networking. Know it exists; it is a frequent review suggestion.

**Callback flooding** is a real failure mode: if a writer loops issuing `call_rcu()`, callbacks
accumulate faster than grace periods retire them, and memory grows without bound. RCU defends
with callback-overload detection (`rcu_cblist` thresholds, `qhimark`, forcing expedited GPs)
but the *correct* fix is in your code: throttle, or use `synchronize_rcu()` occasionally to
self-limit. This is the concrete manifestation of "RCU has unbounded memory overhead".

**Offloading (`rcu_nocbs=`)**: by default callbacks are invoked in softirq on the CPU that
queued them, which adds jitter — unacceptable for RT and HPC. `rcu_nocbs` moves them to
`rcuo` kthreads that can be pinned elsewhere. Essential knowledge for latency-sensitive and
NO_HZ_FULL systems.

### T.10 RCU in the formal memory model

Remarkably, the LKMM (Ch. 13) includes RCU as a **first-class axiom**, not a hand-wave.
`tools/memory-model/linux-kernel.cat` defines `rcu-fence` and the `rcu` axiom encoding:

> If a grace period begins after a read-side critical section starts, and ends after it ends,
> then... (the full statement is the "RCU guarantee" expressed as an acyclicity constraint on
> a relation combining `gp`, `rcu-link`, `rscs`).

You can therefore write litmus tests containing `rcu_read_lock()` and `synchronize_rcu()` and
run them through `herd7`. Do it once — seeing RCU mechanically verified changes how seriously
you take it.

```
tools/memory-model/litmus-tests/MP+onceassign+derefonce.litmus
tools/memory-model/Documentation/explanation.txt   § "THE RCU AXIOM"
```

### T.11 The hazards, stated precisely

1. **Never hold a reference past `rcu_read_unlock()`.** The pointer is valid *only inside*
   the critical section. To keep it, take a real reference *inside* with
   `refcount_inc_not_zero()` (Ch. 12, T.3) — the composition is mandatory.
2. **Never sleep inside `rcu_read_lock()`** (vanilla RCU). `might_sleep()` /
   `CONFIG_PROVE_RCU` will catch it. If you need to sleep → SRCU.
3. **`rcu_dereference()` is not optional.** A plain load loses the address-dependency
   annotation, permits compiler value-speculation, and breaks on Alpha. `PROVE_RCU` checks
   that you're in a read-side critical section when you call it.
4. **`rcu_assign_pointer()` is not optional.** A plain store loses the release barrier, and
   readers can see an uninitialized object (the MP bug).
5. **`rcu_barrier()` before module unload.** Otherwise a queued callback jumps into unmapped
   text. Same class as `flush_workqueue()` / `del_timer_sync()`.
6. **RCU readers do not exclude writers.** Your data structure must be readable at every
   instant a writer could be observed mid-update. Practically: writers must not perform
   multi-step mutations visible to readers — use the copy-then-swap discipline, or a
   seqlock-style version counter on top.
7. **A stalled reader stalls the system.** A read-side critical section that runs for seconds
   (an infinite loop, a hardware wait) produces `rcu_sched detected stalls on CPUs/tasks`
   splats and unbounded memory growth. Read-side critical sections must be *short and
   non-blocking*. (This is the price of the coarse, global GP argument from T.3.)

---

## 1. Concept — the API you actually type

```c
/* Reader */
rcu_read_lock();
p = rcu_dereference(gp);
if (p)
	do_something_with(p->field);
rcu_read_unlock();
/* p is INVALID from here on. */

/* Writer: replace */
spin_lock(&update_lock);              /* writers still exclude each other */
old = rcu_dereference_protected(gp, lockdep_is_held(&update_lock));
new = kmalloc(...);  *new = *old;  new->field = v;
rcu_assign_pointer(gp, new);
spin_unlock(&update_lock);
synchronize_rcu();                    /* or kfree_rcu(old, rcu); */
kfree(old);
```

### 1.1 The full primitive list

| Primitive | Meaning |
|---|---|
| `rcu_read_lock()` / `_unlock()` | delimit a read-side critical section |
| `rcu_read_lock_bh()`, `_sched()` | legacy flavours, now aliases; still used for documentation |
| `rcu_dereference(p)` | subscribe: load with dependency ordering + lockdep check |
| `rcu_dereference_protected(p, cond)` | load when you hold the update lock (no reader checks) |
| `rcu_dereference_check(p, cond)` | load when *either* RCU or `cond` holds |
| `rcu_dereference_raw(p)` | no checks — justify in a comment |
| `rcu_assign_pointer(p, v)` | publish: `smp_store_release` |
| `rcu_replace_pointer(p, v, cond)` | assign and return old, in one expression |
| `synchronize_rcu()` | block until GP completes |
| `call_rcu(&h, fn)` | invoke `fn` after a GP |
| `kfree_rcu(p, rcu_field)` | free after a GP, batched, no callback needed |
| `rcu_barrier()` | wait for all outstanding `call_rcu` callbacks |
| `get_state_synchronize_rcu()` / `poll_state_synchronize_rcu()` | polled GP API |
| `RCU_INIT_POINTER(p, v)` | assign with no barrier — only for init/NULL |
| `rcu_access_pointer(p)` | load the pointer *value* without dereferencing it |

### 1.2 RCU-protected lists — the most common use by far

```c
#include <linux/rculist.h>

/* Reader */
rcu_read_lock();
list_for_each_entry_rcu(item, &my_list, node) {
	/* ... */
}
rcu_read_unlock();

/* Writer (holding update lock) */
list_add_rcu(&new->node, &my_list);
list_del_rcu(&old->node);
list_replace_rcu(&old->node, &new->node);
kfree_rcu(old, rcu);
```

Why these work: list insertion publishes the new node's `next` pointer *before* linking it in
(release), and removal leaves the removed node's `next` pointer **intact** so a concurrent
reader that is standing on it can still walk forward into the list. That "poisoning is
deferred" property is why you must use `list_del_rcu()` and not `list_del()`, and why
`hlist_nulls` exists for the harder case of moving between lists (used by the TCP/UDP socket
hash — a reader can be pushed onto a different chain and must detect it via the "nulls"
marker encoding the bucket number).

### 1.3 RCU-protected trees / hashes

- `struct xarray` and the maple tree are RCU-safe for lookups (Ch. 10).
- `rhashtable` (`lib/rhashtable.c`) — resizable, RCU-protected hash table. **Use this rather
  than rolling your own**; it handles the hard part (lock-free rehash with two tables live).
- `latch_tree` (`include/linux/rbtree_latch.h`) — an rbtree readable from NMI, using two
  copies and a latch seqcount. Used by the module address lookup on the `printk`/unwind path.

---

## 2. Internals

### 2.1 Source map

```
kernel/rcu/tree.c              ★ Tree RCU core: GP machinery, rcu_node tree
kernel/rcu/tree_plugin.h       preemptible RCU, boosting, no-CB offloading
kernel/rcu/tree_exp.h          expedited grace periods
kernel/rcu/tree_stall.h        stall detection & reporting
kernel/rcu/srcutree.c          SRCU
kernel/rcu/tasks.h             RCU-Tasks / Rude / Trace
kernel/rcu/rcu_segcblist.c     segmented callback lists
kernel/rcu/update.c            common API, rcu_barrier
kernel/rcu/rcutorture.c        ★ the torture test
include/linux/rcupdate.h       the API
include/linux/rculist.h        RCU list ops
Documentation/RCU/             ★ ~20 files; Design/Requirements is a small book
```

### 2.2 Grace-period state machine (Tree RCU)

```
rcu_gp_kthread()                        /* one per RCU flavour, kernel/rcu/tree.c */
  ├─ rcu_gp_init()
  │     mark GP start; record ->gp_seq; snapshot which CPUs are online/idle
  ├─ rcu_gp_fqs_loop()                  /* repeat until all QS reported */
  │     force_quiescent_state():
  │        - CPUs in dyntick-idle → counted as quiescent (no IPI needed)
  │        - CPUs running in user mode → quiescent
  │        - stubborn CPUs → resched IPI
  │        - after ~21s → RCU CPU stall warning
  └─ rcu_gp_cleanup()
        advance ->gp_seq; mark callbacks ready; wake rcu_core / rcuo kthreads
```

`->gp_seq` is a sequence number with the low bits encoding GP phase — the same
even/odd-style trick as a seqcount. Callback lists are tagged with the `gp_seq` they are
waiting for, so a single comparison decides readiness.

**dyntick-idle integration** is important and non-obvious: an idle CPU in a deep C-state must
not be woken just to report a QS. RCU tracks `->dynticks` per CPU; observing an even/odd
transition proves the CPU was idle (hence quiescent) at some point. This is how RCU stays
compatible with power management, and it's why `rcu_idle_enter()`/`rcu_user_enter()` exist in
the entry code.

### 2.3 Reading a stall warning

```
rcu: INFO: rcu_preempt detected stalls on CPUs/tasks:
rcu:     3-...0: (1 GPs behind) idle=a2c/1/0x4000000000000000 softirq=1234/1235 fqs=2
rcu:     (detected by 0, t=21002 jiffies, g=4517, q=871 ncpus=8)
```

- `3-...0` — CPU 3; the dots are flags (`.`=no, letter=yes) for:
  `[0]` GP kthread starved, `[1]` in dyntick-idle, `[2]` IRQ/NMI nesting, `[3]` offloaded.
- `t=21002 jiffies` — how long the GP has been stuck (default threshold 21 s,
  `rcupdate.rcu_cpu_stall_timeout`).
- `q=871` — callbacks queued.

**Diagnosis:** 99% of stalls are (a) an infinite/very long loop inside a read-side critical
section, (b) a CPU spinning with interrupts disabled, (c) a real deadlock, or (d) a
misbehaving hypervisor descheduling a vCPU. Look at the printed stack.

### 2.4 Debug configuration

```
CONFIG_PROVE_RCU=y              # lockdep integration: checks rcu_dereference usage
CONFIG_PROVE_RCU_LIST=y         # checks list_for_each_entry_rcu usage
CONFIG_RCU_EQS_DEBUG=y
CONFIG_RCU_TRACE=y
CONFIG_RCU_CPU_STALL_TIMEOUT=21
CONFIG_RCU_TORTURE_TEST=m
CONFIG_DEBUG_OBJECTS_RCU_HEAD=y # catches double call_rcu on the same head
```

```bash
ls /sys/kernel/debug/rcu/rcu_preempt/     # rcudata, rcugp, rcuexp, ...
sudo cat /sys/kernel/debug/rcu/rcu_preempt/rcugp
sudo modprobe rcutorture                  # then dmesg
sudo perf trace -e 'rcu:*' -a sleep 2
sudo bpftrace -e 'kprobe:synchronize_rcu { @[kstack] = count(); }'
```

---

## 3. Practice

### 3.1 A complete RCU-protected registry

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * Demonstrates the canonical composition: RCU for existence, refcount for
 * retention beyond the read-side critical section.
 *
 * Lock ordering: reg_lock is the only writer lock. Readers take no locks.
 */
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/slab.h>
#include <linux/rculist.h>
#include <linux/refcount.h>
#include <linux/kthread.h>
#include <linux/delay.h>

struct entry {
	u32               key;
	char              value[32];
	refcount_t        ref;
	struct list_head  node;
	struct rcu_head   rcu;
};

static LIST_HEAD(registry);
static DEFINE_SPINLOCK(reg_lock);

static void entry_free_rcu(struct rcu_head *h)
{
	kfree(container_of(h, struct entry, rcu));
}

static void entry_put(struct entry *e)
{
	if (refcount_dec_and_test(&e->ref))
		call_rcu(&e->rcu, entry_free_rcu);
}

static int entry_add(u32 key, const char *val)
{
	struct entry *e = kzalloc(sizeof(*e), GFP_KERNEL);

	if (!e)
		return -ENOMEM;
	e->key = key;
	strscpy(e->value, val, sizeof(e->value));
	refcount_set(&e->ref, 1);                  /* the registry's reference */

	spin_lock(&reg_lock);
	list_add_rcu(&e->node, &registry);         /* publishes with release semantics */
	spin_unlock(&reg_lock);
	return 0;
}

static void entry_remove(u32 key)
{
	struct entry *e;

	spin_lock(&reg_lock);
	list_for_each_entry(e, &registry, node) {  /* writer: plain iteration under lock */
		if (e->key == key) {
			list_del_rcu(&e->node);
			spin_unlock(&reg_lock);
			entry_put(e);                  /* drop registry's ref */
			return;
		}
	}
	spin_unlock(&reg_lock);
}

/* Lookup that returns a REFERENCED entry usable after rcu_read_unlock(). */
static struct entry *entry_get(u32 key)
{
	struct entry *e, *found = NULL;

	rcu_read_lock();
	list_for_each_entry_rcu(e, &registry, node) {
		if (e->key == key) {
			if (refcount_inc_not_zero(&e->ref))
				found = e;
			break;
		}
	}
	rcu_read_unlock();
	return found;
}

/* Lookup used entirely INSIDE the critical section — no refcount needed. */
static bool entry_exists(u32 key)
{
	struct entry *e;

	guard(rcu)();                              /* 6.4+ scoped guard for rcu_read_lock */
	list_for_each_entry_rcu(e, &registry, node)
		if (e->key == key)
			return true;
	return false;
}

static int __init rcudemo_init(void)
{
	struct entry *e;

	entry_add(1, "alpha");
	entry_add(2, "beta");

	e = entry_get(2);
	if (e) {
		pr_info("got %u = %s\n", e->key, e->value);
		msleep(10);                        /* legal: we hold a real reference */
		entry_put(e);
	}
	pr_info("exists(1)=%d exists(3)=%d\n", entry_exists(1), entry_exists(3));
	return 0;
}

static void __exit rcudemo_exit(void)
{
	struct entry *e, *tmp;

	spin_lock(&reg_lock);
	list_for_each_entry_safe(e, tmp, &registry, node) {
		list_del_rcu(&e->node);
		entry_put(e);
	}
	spin_unlock(&reg_lock);

	rcu_barrier();      /* ★ MANDATORY: wait for entry_free_rcu callbacks */
}

module_init(rcudemo_init);
module_exit(rcudemo_exit);
MODULE_LICENSE("GPL");
```

Study the two lookup functions. `entry_exists()` needs no refcount because the pointer never
escapes the critical section. `entry_get()` needs one because it does. **Deciding which you
need is the skill.**

### 3.2 SRCU when readers must sleep

```c
#include <linux/srcu.h>

static DEFINE_SRCU(my_srcu);        /* or: struct srcu_struct s; init_srcu_struct(&s); */

static void reader(void)
{
	int idx = srcu_read_lock(&my_srcu);
	struct thing *t = srcu_dereference(gp, &my_srcu);

	if (t)
		msleep(100);                /* LEGAL under SRCU */
	srcu_read_unlock(&my_srcu, idx);
}

static void writer(struct thing *new)
{
	struct thing *old = rcu_replace_pointer(gp, new, lockdep_is_held(&lock));

	synchronize_srcu(&my_srcu);         /* or call_srcu() */
	kfree(old);
}
/* cleanup_srcu_struct(&my_srcu) on teardown */
```
Note `srcu_read_lock()` returns an **index** that must be passed back to `unlock` — that's
the counter-array selector from T.8 leaking into the API.

### 3.3 Measure a grace period

```c
u64 t0 = ktime_get_ns();
synchronize_rcu();
pr_info("GP took %llu us\n", (ktime_get_ns() - t0) / 1000);

t0 = ktime_get_ns();
synchronize_rcu_expedited();
pr_info("expedited GP took %llu us\n", (ktime_get_ns() - t0) / 1000);
```
Run on an idle box and on a loaded box. Run with `nohz_full=` set. The spread will surprise
you and will permanently cure you of calling `synchronize_rcu()` in a loop.

---

## 3.4 Extended practice

### Lab 15.A — Measure the read side: RCU vs everything else

The headline claim of this chapter is "zero-cost readers". Verify it.

```c
// SPDX-License-Identifier: GPL-2.0
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/kthread.h>
#include <linux/rcupdate.h>
#include <linux/spinlock.h>
#include <linux/rwsem.h>
#include <linux/seqlock.h>
#include <linux/slab.h>
#include <linux/cpu.h>

struct cfg { int a, b, c; };
static struct cfg __rcu *gp;
static struct cfg  plain;
static DEFINE_SPINLOCK(sl);
static DECLARE_RWSEM(rws);
static DEFINE_SEQLOCK(seq);
static DEFINE_MUTEX(update_lock);

static int variant = 0;   /* 0=rcu 1=spinlock 2=rwsem 3=seqlock */
static int nreaders = 4;
module_param(variant, int, 0644);
module_param(nreaders, int, 0644);
static atomic64_t ops;
static atomic_t stop;

static int reader(void *u)
{
	u64 n = 0;
	int sum = 0;

	while (!atomic_read(&stop)) {
		switch (variant) {
		case 0: {
			struct cfg *p;

			rcu_read_lock();
			p = rcu_dereference(gp);
			sum += p->a + p->b + p->c;
			rcu_read_unlock();
			break;
		}
		case 1:
			spin_lock(&sl);
			sum += plain.a + plain.b + plain.c;
			spin_unlock(&sl);
			break;
		case 2:
			down_read(&rws);
			sum += plain.a + plain.b + plain.c;
			up_read(&rws);
			break;
		case 3: {
			unsigned int s;
			struct cfg local;

			do {
				s = read_seqbegin(&seq);
				local = plain;
			} while (read_seqretry(&seq, s));
			sum += local.a + local.b + local.c;
			break;
		}
		}
		if (!(++n & 0x3ff))
			cond_resched();
	}
	atomic64_add(n, &ops);
	return sum & 1;
}

static int __init rcubench_init(void)
{
	struct cfg *c = kzalloc(sizeof(*c), GFP_KERNEL);
	struct task_struct **t;
	u64 t0;
	int i;

	if (!c)
		return -ENOMEM;
	rcu_assign_pointer(gp, c);
	seqlock_init(&seq);

	t = kcalloc(nreaders, sizeof(*t), GFP_KERNEL);
	t0 = ktime_get_ns();
	for (i = 0; i < nreaders; i++) {
		t[i] = kthread_create(reader, NULL, "rcubench/%d", i);
		kthread_bind(t[i], i % num_online_cpus());
		wake_up_process(t[i]);
	}
	msleep(3000);
	atomic_set(&stop, 1);
	for (i = 0; i < nreaders; i++)
		kthread_stop(t[i]);

	pr_info("variant=%d readers=%2d -> %llu Mops/s (%llu ns/read)\n",
		variant, nreaders,
		atomic64_read(&ops) * 1000 / (ktime_get_ns() - t0),
		(ktime_get_ns() - t0) * nreaders / max(atomic64_read(&ops), 1LL));
	kfree(t);
	return 0;
}
static void __exit rcubench_exit(void)
{
	struct cfg *c = rcu_dereference_protected(gp, true);

	RCU_INIT_POINTER(gp, NULL);
	synchronize_rcu();
	kfree(c);
}
module_init(rcubench_init); module_exit(rcubench_exit);
MODULE_LICENSE("GPL");
```
```bash
for v in 0 1 2 3; do for n in 1 2 4 8 16; do
  sudo insmod rcubench.ko variant=$v nreaders=$n; sudo rmmod rcubench
done; done
dmesg | grep rcubench
```
**Expected:** RCU and seqlock scale *linearly* with reader count; spinlock and rwsem peak
early and then decline. Plot aggregate throughput vs `nreaders`. This one graph justifies
RCU's entire existence — and it is the graph to bring to a design review.

### Lab 15.B — Measure grace-period latency in four conditions

```c
static void measure_gp(const char *label)
{
	u64 t0 = ktime_get_ns();

	synchronize_rcu();
	pr_info("%-20s synchronize_rcu      : %8llu us\n", label,
		(ktime_get_ns() - t0) / 1000);

	t0 = ktime_get_ns();
	synchronize_rcu_expedited();
	pr_info("%-20s synchronize_rcu_exp  : %8llu us\n", label,
		(ktime_get_ns() - t0) / 1000);

	/* The polled API: often free, because a GP already elapsed */
	{
		unsigned long cookie = get_state_synchronize_rcu();

		msleep(50);
		t0 = ktime_get_ns();
		cond_synchronize_rcu(cookie);
		pr_info("%-20s cond_synchronize_rcu : %8llu us\n", label,
			(ktime_get_ns() - t0) / 1000);
	}
}
```
Run it: (a) on an idle box, (b) under `stress-ng --cpu $(nproc)`, (c) with
`nohz_full=1-7 rcu_nocbs=1-7`, (d) with `CONFIG_PREEMPT_RT`. Tabulate. The spread between
idle and loaded is usually 10–100× and it will cure you of calling `synchronize_rcu()`
anywhere near a hot path.

```bash
# Watch grace periods happen:
sudo cat /sys/kernel/debug/rcu/rcu_preempt/rcugp
watch -n1 'sudo cat /sys/kernel/debug/rcu/rcu_preempt/rcugp'
sudo bpftrace -e 'tracepoint:rcu:rcu_grace_period { @[str(args->gpevent)] = count(); }'
sudo perf stat -e 'rcu:*' -a -- sleep 5
```

### Lab 15.C — Provoke an RCU CPU stall and read it (T.11.7)

```c
static int staller(void *u)
{
	rcu_read_lock();
	pr_info("entering a 30-second read-side critical section (this is a BUG)\n");
	mdelay(30000);              /* busy-wait, cannot be preempted out of the RSCS */
	rcu_read_unlock();
	return 0;
}
```
```bash
./scripts/config --set-val RCU_CPU_STALL_TIMEOUT 21 -e RCU_TRACE
sudo insmod stall.ko
sleep 25 && dmesg | grep -A40 'detected stalls'
```
Annotate every field of the report using §2.3. Then watch memory grow:
```bash
while sleep 1; do grep -E 'Slab|MemFree' /proc/meminfo | tr '\n' ' '; echo; done
```
That growth is the "unbounded memory" cost from T.4, made visible.

### Lab 15.D — Callback flooding and the overload response (T.9)

```c
static int flooder(void *u)
{
	while (!kthread_should_stop()) {
		struct node *n = kmalloc(sizeof(*n), GFP_KERNEL);

		if (n)
			kfree_rcu(n, rcu);
		/* no throttling: deliberately outrun the grace-period machinery */
	}
	return 0;
}
```
```bash
sudo insmod flood.ko
watch -n1 'grep -E "Slab|SUnreclaim" /proc/meminfo; \
           sudo grep -h qlen /sys/kernel/debug/rcu/rcu_preempt/rcudata | head -4'
dmesg | grep -i 'callback\|qhimark\|rcu.*overload'
```
Then add throttling (`if (++n % 10000 == 0) synchronize_rcu();`) and show memory stabilizes.
Tune `rcutree.qhimark` / `rcutree.blimit` and observe the effect.

### Lab 15.E — SRCU with sleeping readers, and domain isolation

```c
static DEFINE_SRCU(fast_domain);
static DEFINE_SRCU(slow_domain);

static int slow_reader(void *u)
{
	int idx = srcu_read_lock(&slow_domain);

	ssleep(10);                        /* LEGAL under SRCU */
	srcu_read_unlock(&slow_domain, idx);
	return 0;
}

static void timed_gps(void)
{
	u64 t0 = ktime_get_ns();

	synchronize_srcu(&fast_domain);    /* unaffected by the slow reader */
	pr_info("fast domain GP: %llu us\n", (ktime_get_ns() - t0) / 1000);

	t0 = ktime_get_ns();
	synchronize_srcu(&slow_domain);    /* waits ~10 s */
	pr_info("slow domain GP: %llu us\n", (ktime_get_ns() - t0) / 1000);

	t0 = ktime_get_ns();
	synchronize_rcu();                 /* vanilla RCU: also unaffected */
	pr_info("vanilla RCU GP: %llu us\n", (ktime_get_ns() - t0) / 1000);
}
```
This demonstrates T.8's real point: **domain isolation**, not just sleepability. Then
contrast by making the slow reader use vanilla `rcu_read_lock()` + `schedule()` — lockdep
will catch it with `CONFIG_PROVE_RCU=y`.

### Lab 15.F — `hlist_nulls`: the race plain RCU lists cannot handle

```bash
$EDITOR include/linux/list_nulls.h include/linux/rculist_nulls.h
$EDITOR Documentation/RCU/rculist_nulls.rst     # read this fully; it's short and superb
$EDITOR net/ipv4/inet_hashtable.c               # __inet_lookup_established()
```
Write out the exact interleaving that breaks a plain RCU hash chain when an element is
**moved between buckets** (rehash), and show how encoding the bucket index in the NULL
terminator lets the reader detect it and restart. Then build a toy version and instrument
the restart count under concurrent rehashing.

### Lab 15.G — Verify RCU formally with `herd7` (T.10)

```
C rcu-mp
{}
P0(int **gp, int *data)
{
	WRITE_ONCE(*data, 42);
	rcu_assign_pointer(*gp, data);
}
P1(int **gp, int *data, int *old)
{
	int *r1;
	int r2;

	rcu_read_lock();
	r1 = rcu_dereference(*gp);
	r2 = READ_ONCE(*r1);
	rcu_read_unlock();
}
exists (1:r2=0)
```
```bash
cd linux/tools/memory-model
herd7 -conf linux-kernel.cfg /tmp/rcu-mp.litmus
# Then a grace-period test:
herd7 -conf linux-kernel.cfg litmus-tests/RCU+sync+read.litmus
herd7 -conf linux-kernel.cfg litmus-tests/RCU+sync+free.litmus
ls litmus-tests/ | grep -i rcu
```
Remove the `rcu_assign_pointer`/`rcu_dereference` (use plain accesses) and show the bad
outcome becomes allowed. **Seeing RCU mechanically verified is the point of this lab.**

### Lab 15.H — `rcutorture`, and how it verifies the guarantee

```bash
./scripts/config -m RCU_TORTURE_TEST -e PROVE_RCU -e RCU_TRACE
for t in rcu srcu srcud tasks tasks-tracing; do
  sudo modprobe rcutorture torture_type=$t nreaders=6 stat_interval=15 shutdown_secs=0
  sleep 45; sudo rmmod rcutorture; echo "=== $t"; dmesg | tail -8
done

# The kernel's own full test harness (uses QEMU; this is what maintainers run):
cd linux/tools/testing/selftests/rcutorture
./bin/kvm.sh --cpus 8 --duration 10 --configs "TREE01 TREE02 SRCU-P"
```
Then read `kernel/rcu/rcutorture.c` and find the code that **detects a grace-period
violation** (readers record which "generation" of the object they saw; a reader observing a
freed generation is a failure). That verification technique generalizes to your own
subsystems.

---

## 4. Mastery drills

1. **Prove the theorem out loud.** Explain to someone why a context switch cannot occur
   inside a non-preemptible RCU read-side critical section, and why that makes grace periods
   detectable for free. If you stumble, reread T.3.

2. **Read `Documentation/RCU/Design/Requirements/Requirements.rst`** end to end. It is long
   (~3000 lines) and it is the best-written design document in the kernel. Budget an evening.

3. **Break it deliberately.** Remove the `rcu_barrier()` from §3.1's exit path; build with
   `CONFIG_DEBUG_OBJECTS_RCU_HEAD=y` and KASAN; `insmod`/`rmmod` in a loop. Then remove
   `refcount_inc_not_zero()` from `entry_get()` and reproduce the UAF under load.

4. **Litmus-test RCU.** Write a litmus test with `rcu_read_lock()`/`synchronize_rcu()` and run
   it under `herd7`. Show that removing `synchronize_rcu()` makes the bad outcome allowed.

5. **Trace a real subsystem.** Read `net/ipv4/fib_trie.c` or `fs/dcache.c`. Identify every
   RCU primitive, the update lock, and the reclamation path. Explain how a reader that is
   preempted mid-traversal remains safe.

6. **`hlist_nulls`.** Read `include/linux/list_nulls.h` and the UDP/TCP socket lookup in
   `net/ipv4/udp.c`. Explain the exact race that plain RCU lists cannot handle and how the
   nulls value solves it. This is a classic senior interview question.

7. **Callback flood.** Write a module that calls `kfree_rcu()` in a tight loop. Watch
   `/proc/meminfo`, `rcu` debugfs `qlen`, and the kernel's overload response. Then add
   throttling and compare.

8. **Choose correctly.** For each, argue RCU vs seqlock vs refcount vs plain lock:
   (a) a driver's list of open contexts; (b) a global config struct read on every packet;
   (c) a hash table of 10M flows with 1M inserts/sec; (d) a timekeeping struct read from NMI;
   (e) a list of registered notifier callbacks that may sleep.

9. **Torture it.** `modprobe rcutorture torture_type=srcu nreaders=8 stat_interval=15` and
   interpret the output. Then read `rcutorture.c` to see how it *verifies* the grace-period
   guarantee at runtime — the verification technique is itself instructive.

---

## 5. Further reading

**Kernel documentation (this is where RCU is actually documented best):**
- `Documentation/RCU/whatisRCU.rst` — **start here**
- `Documentation/RCU/checklist.rst` — the review checklist; memorize it
- `Documentation/RCU/rcu_dereference.rst` — the subtleties of dependency ordering
- `Documentation/RCU/Design/Requirements/Requirements.rst` — the definitive design text
- `Documentation/RCU/Design/Data-Structures/Data-Structures.rst` — the `rcu_node` tree
- `Documentation/RCU/Design/Memory-Ordering/Tree-RCU-Memory-Ordering.rst`
- `Documentation/RCU/listRCU.rst`, `rculist_nulls.rst`, `rcuref.rst`, `stallwarn.rst`

**Papers:**
- McKenney & Slingwine, "Read-Copy Update: Using Execution History to Solve Concurrency
  Problems" (PDCS 1998) — **the original**
- McKenney et al., "RCU Usage In the Linux Kernel: One Decade Later" (2013)
- McKenney, Boyd-Wickizer, Walpole, "RCU Usage in the Linux Kernel: Eighteen Years Later"
  (OSR 2020)
- Michael, "Hazard Pointers: Safe Memory Reclamation for Lock-Free Objects" (TPDS 2004)
  — the main alternative; know the trade-offs
- Fraser, "Practical Lock-Freedom" (PhD thesis, 2004) — epoch-based reclamation
- Hart, McKenney, Brown, "Making Lockless Synchronization Fast" (IPDPS 2007) — **the
  quantitative comparison of RCU vs HP vs EBR vs refcounting**
- Yew, Tzeng, Lawrie, "Distributing Hot-Spot Addressing in Large-Scale Multiprocessors"
  (1987) — combining trees

**Books & talks:**
- McKenney, *Is Parallel Programming Hard…* — Chapter 9 is 100 pages on RCU by its author
- Paul McKenney's LWN series: "What is RCU, Fundamentally?" (Parts 1–3) — the classic
  three-part introduction
- LWN: "The RCU API, 2024 edition" (updated every few years — read the newest)
- Kernel Recipes / LPC talks by Paul McKenney and Joel Fernandes

→ Next: [16-percpu.md](16-percpu.md)

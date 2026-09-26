# Chapter 14 — Locking: Spinlocks, Mutexes, rwsems, Seqlocks, and Lockdep

> **Goal:** pick the right lock from first principles, design a lock hierarchy that scales,
> and read a lockdep splat like a sentence. Ch. 13 gave you ordering; this chapter gives you
> mutual exclusion built on top of it.

---

## Theory & First Principles

> **How to read this section.** T.0 builds a lock from an atomic and shows why the obvious
> implementation is a disaster. T.1–T.4 are the theory: mutual exclusion, ski-rental, the
> cache-coherence cost model. T.5–T.12 are the Linux primitives and how to choose.

---

### T.0 — Start here: build a lock, then discover why yours is bad

You have atomics (Ch. 13). A lock looks like five lines:

```c
/* Attempt 1: the test-and-set lock. */
void lock(atomic_t *l)   { while (atomic_xchg(l, 1) == 1) ; }
void unlock(atomic_t *l) { atomic_set(l, 0); }
```

It is correct. It provides mutual exclusion. **And it is catastrophically bad**, in three
specific ways — each of which motivates a real piece of the kernel.

**Problem 1: every spinner writes the same cacheline.**

`atomic_xchg` is a read-modify-write, so it must take the line in **exclusive** state. With
*N* CPUs spinning, the line ping-pongs between all of them continuously. From Ch. 00 §T.3, a
cross-socket cacheline transfer is 200–400 ns — so the spinners are generating O(N²)
coherence traffic and actively slowing down the CPU *holding* the lock, which is the one CPU
that needs to make progress.

Measure it and the curve goes the wrong way:

```
 throughput
     │      ╱╲
     │    ╱    ╲╲╲╲╲╲╲╲╲╲     <- adding cores makes it SLOWER
     │  ╱
     └────────────────── cores
```

That is the coherency term of the Universal Scalability Law (Ch. 105 §T.2), observed in five
lines of code. **The fix is not a faster lock; it is a lock where waiters do not share a
cacheline** — the MCS/`qspinlock` design of §T.3, where each waiter spins on its *own* node.

**Problem 2: it is unfair, and unfairness becomes starvation.**

Whoever's cacheline request wins the arbitration gets the lock. A CPU that just released it
still has the line and typically re-acquires it. Under contention one CPU can hold a lock
90% of the time while another waits indefinitely. Ticket locks and then `qspinlock` made
kernel spinlocks FIFO-fair, and the reason was measured starvation, not theory.

**Problem 3: spinning is sometimes exactly the wrong thing.**

```
 Hold time 50 ns   -> spin.  A context switch costs 1-2 µs: 40x worse.
 Hold time 10 ms   -> sleep. Spinning burns 10 ms of a CPU for nothing.
 Hold time UNKNOWN -> ???
```

You usually do not know. That is a known problem with a known answer — the **ski-rental**
problem, §T.2 — and it gives you a *provably* 2-competitive strategy: spin for about as long
as a sleep would cost, then sleep. Linux's adaptive mutex does exactly this, with a twist
you would not guess: it spins **only while the lock owner is still running on another CPU**,
because if the owner has been descheduled, spinning cannot possibly pay off.

**And the fourth thing, which is not about performance at all:**

```c
void handler(void) { lock(&l); ... }    /* interrupt handler */
void thread(void)  { lock(&l); /* <- interrupt fires HERE, on this CPU */ }
```

The handler spins waiting for a lock held by the thread it interrupted, on the same CPU. The
thread can never run to release it. **Instant, guaranteed self-deadlock.** No amount of
cleverness in the lock fixes this; it requires `spin_lock_irqsave` — and that requirement is
why "what context am I in?" (Ch. 00 §T.9) is the first question in kernel programming.

**So the five-line lock teaches all four lessons:** cacheline behaviour is the cost model,
fairness must be designed in, the spin-versus-sleep decision has a principled answer, and
context determines which primitive is even legal. Everything else in this chapter elaborates
those.

---

### T.1 The mutual exclusion problem

Dijkstra posed it in 1965: given `N` concurrent processes with shared memory, construct a
protocol such that at most one is in its critical section, with:

1. **Mutual exclusion** — never two at once (safety).
2. **Progress / deadlock freedom** — if nobody is in the CS and someone wants in, someone
   gets in (liveness).
3. **Starvation freedom / fairness** — every waiter eventually gets in (stronger liveness).

Dijkstra's original solution, Lamport's Bakery algorithm (1974), and Peterson's algorithm
(1981) solve it using only loads and stores — but they need sequential consistency (hence
`smp_mb()`, the SB litmus test from Ch. 13), they are `O(N)` in space, and they are far
slower than a single atomic RMW. Real kernels therefore build locks on hardware atomics.

Herlihy's consensus hierarchy (Ch. 13, T.12) tells us why: `test-and-set` has consensus
number 2 — enough for mutual exclusion — while `compare-and-swap` is universal. Locks only
need the former; lock-free data structures need the latter.

### T.2 Spin or sleep? The ski-rental problem

The central design question for any lock: when the lock is held, should the waiter **spin**
(burn CPU, keep the cache line, resume instantly) or **block** (yield the CPU, pay a context
switch on both sides)?

This is exactly the classical **ski-rental / rent-or-buy** online problem. Let:

- `S` = cost of a context switch pair (sleep + wake) ≈ **1–5 µs** (TLB, cache, scheduler).
- `H` = hold time of the lock (unknown to the waiter — that's what makes it *online*).

Spinning costs `H`. Blocking costs `S` regardless. Offline optimum: spin iff `H < S`.
The classic **2-competitive** online strategy: spin for exactly `S` time, then block.
Worst case you pay `2S` instead of the optimal `S`.

Linux implements exactly this, and refines it:

- **`spinlock_t`** — pure spinning. Correct choice when the hold time is *provably* short
  (a few hundred ns) and **mandatory** when you cannot sleep (hardirq/softirq context).
- **`mutex`** — blocking, but with **adaptive spinning**: spin *only while the current owner
  is running on another CPU*. That is an inspired heuristic, because "owner is running" is
  strong evidence the hold time is short; if the owner is descheduled, `H` is unbounded and
  spinning is pure waste. See `mutex_spin_on_owner()` in `kernel/locking/mutex.c`.

That single heuristic is why `mutex` is the default in Linux and why the old advice
"use a spinlock, it's faster" is obsolete for sleepable contexts.

### T.3 The real cost model is cache coherence, not instructions

A naive test-and-set lock:

```c
while (xchg(&lock, 1))
	;
```
Every spinner issues a `lock xchg` in a tight loop. Each one demands the cache line in
**Exclusive/Modified** state, invalidating everyone else. With `N` waiters you get `O(N)`
coherence traffic per attempt and `O(N²)` total to hand the lock around — the line
"ping-pongs" across the interconnect and *the lock holder itself* is slowed down trying to
release it.

**Test-and-test-and-set** fixes the worst of it:

```c
while (xchg(&lock, 1))
	while (READ_ONCE(lock))     /* spin on a SHARED read, no invalidation */
		cpu_relax();
```
Spinners now read from Shared state, generating no traffic — until the release, which
invalidates all of them at once (the **thundering herd**), and they all storm the line again.

**Key theoretical insight:** the problem is that all waiters contend on *one* cache line.
The fix is to give each waiter its **own** line to spin on. That is the **queue lock** family:

| Lock | Spins on | Traffic per handoff | Fair? |
|---|---|---|---|
| test-and-set | one shared line | O(N) | no (starvation possible) |
| ticket lock | one shared line (reads) | O(N) invalidations per release | **FIFO fair** |
| **MCS** (Mellor-Crummey & Scott, 1991) | **its own local node** | **O(1)** | FIFO fair |
| **CLH** (Craig, Landin, Hagersten) | predecessor's node | O(1) | FIFO fair |

MCS is the theoretical winner: each waiter enqueues a local node and spins on a flag *in its
own node*; the releaser writes to the successor's node. **Exactly one cache line transfer per
handoff, regardless of N.** This is a genuinely important algorithm — know it by name.

MCS's drawback: the lock word must be pointer-sized *and* each waiter needs a node, so the
lock structure grows. Linux cannot afford that — `spinlock_t` is embedded in millions of
objects (`struct inode`, `struct page`-adjacent structures) and must stay **4 bytes**.

### T.4 qspinlock: how Linux got MCS in 4 bytes

Linux's `spinlock_t` (since 4.2, Waiman Long) is a **queued spinlock**: a hybrid that keeps
the 32-bit lock word but layers MCS behind it.

```
 31          16 15   9  8   7      0
+--------------+------+---+--------+
| tail (cpu+idx)| pend |   | locked |
+--------------+------+---+--------+
```

Three-tier design, chosen by contention level:

1. **Uncontended** (the 99% case): one `cmpxchg` on the `locked` byte. Same cost as a simple
   TAS lock. No queue, no node.
2. **One waiter**: set the `pending` bit and spin on the lock byte. Avoids touching the MCS
   queue for the extremely common 2-CPU contention case — an empirical optimization that
   mattered a lot in benchmarks.
3. **Two or more waiters**: form a real MCS queue using per-CPU pre-allocated nodes
   (`qnodes[MAX_NODES]`, 4 per CPU for task/softirq/hardirq/NMI nesting). The `tail` field
   encodes `(cpu_id, nesting_index)` — that's how a pointer fits in 16 bits.

So Linux pays MCS's complexity **only when it pays off**. This tiering-by-contention pattern
recurs throughout the kernel (cf. `percpu_ref`, SLUB fast/slow path) and is worth recognizing
as a design template.

`qspinlock` also has a **paravirtualized** variant (`pv_qspinlock`): in a VM, a vCPU holding
a lock may be *preempted by the hypervisor*, making hold time unbounded. PV qspinlock
hypercalls to yield to the lock holder and to halt rather than spin. The "lock holder
preemption" problem is a fundamental consequence of spinning in a virtualized environment
and has no pure-guest solution.

### T.5 Fairness, convoys, and the Universal Scalability Law

**Fairness is not free.** A strictly FIFO lock forces the line to move to the next CPU in
queue order, even if that CPU is on a remote NUMA node and a local CPU wants the lock. Strict
FIFO can *halve* throughput on multi-socket machines. Hence `CONFIG_NUMA_AWARE_SPINLOCKS`
(CNA — Compact NUMA-aware locks), which batches handoffs within a node before passing to
another node — trading bounded unfairness for locality. This is a *deliberate, quantified*
fairness/throughput trade, and exactly the kind of judgement call a kernel architect makes.

**Lock convoy:** when a lock's hold time approaches the inter-arrival time of requests,
waiters queue up and the system enters a persistent queued state. Each release immediately
hands to a waiter who must be woken (context switch), so the system does a context switch per
critical section and throughput collapses. The convoy is self-sustaining — it does not decay
when load drops slightly. Recognizing a convoy in a profile (high context-switch rate +
high lock wait time + moderate CPU) is a senior diagnostic skill.

**Gunther's Universal Scalability Law** is the right model for capacity planning:

$$ C(N) = \frac{N}{1 + \sigma(N-1) + \kappa N(N-1)} $$

- $\sigma$ = **contention** (serialization; this is Amdahl's term)
- $\kappa$ = **coherency delay** (crosstalk; the cost of keeping caches consistent)

The $\kappa N^2$ term means throughput does not merely plateau — it *peaks and then
decreases*. That is why adding CPUs to a lock-bound workload can make it slower, and why
"just make the critical section shorter" eventually stops helping while "eliminate the shared
line entirely" (per-CPU, RCU, sharding) keeps working. **Every major Linux scalability
improvement of the last 20 years is an attack on $\kappa$, not $\sigma$.**

### T.6 Deadlock: Coffman's conditions and lock ordering as a partial order

Deadlock requires all four **Coffman conditions** (1971) simultaneously:

1. Mutual exclusion
2. Hold-and-wait
3. No preemption of resources
4. **Circular wait**

The kernel cannot give up 1–3, so it attacks **circular wait**, using the classical
**resource ordering** solution: define a global partial order `<` on lock classes and require
that locks be acquired in increasing order. If every thread respects a single partial order,
no cycle can exist — this is a one-line proof and it is the entire basis of kernel locking
discipline.

Consequences you live with:

- Lock ordering must be **documented** (top of file, or `Documentation/`), because it is a
  global invariant that no single function can enforce.
- `spin_lock_nested(lock, subclass)` exists for the legitimate case of taking two locks of
  the *same class* (e.g. two inodes in `rename`), where you need a *secondary* ordering rule
  (usually "lower address first" or "lower inode number first") that lockdep can't infer.
- Any function that takes a lock and calls a callback creates an **ordering obligation on
  code you don't control** — which is why "don't call user callbacks with a lock held" is
  such a strong API design rule.

### T.7 Lockdep: runtime verification of the partial order

`CONFIG_PROVE_LOCKING` implements a beautiful idea: instead of documenting the order, **infer
it and check it**.

Lockdep maintains a directed graph over **lock classes** (not instances — that's the trick
that makes it tractable; a class is typically identified by the `lockdep_map`'s static key,
so all `inode->i_mutex` instances share a class). Every time thread `T` holds `A` and
acquires `B`, it adds edge `A → B`. Before adding, it searches for a path `B → A`. If found,
that's a potential deadlock — reported **even if the deadlock never actually occurred**.

Properties that matter:

- **Sound-ish but incomplete**: it finds cycles in the *observed* graph. Never-executed
  orderings are never learned. So *coverage matters* — run your driver hard.
- **Not a dynamic detector**: it does not wait for a hang. One execution of each ordering is
  enough to prove the cycle exists. This is why lockdep catches bugs that would take years
  to reproduce.
- It also tracks **IRQ-safety**: if class `A` is ever taken in hardirq context, then taking
  `A` with interrupts enabled anywhere is a potential self-deadlock (an interrupt on the same
  CPU re-entering and blocking on a lock the interrupted code holds). This is the
  `HARDIRQ-safe → HARDIRQ-unsafe` class of report, and it is why `spin_lock_irqsave()` exists.
- It validates sleep-in-atomic (`might_sleep()`), RCU usage (`PROVE_RCU`), and more.

**Lockdep is not optional in a dev kernel.** Running without it is professional malpractice.

### T.8 Priority inversion and inheritance

On a real-time system: low-priority task `L` holds lock `X`; high-priority `H` blocks on `X`;
medium-priority `M` preempts `L`. Now `H` waits on `M` — **unbounded** inversion. This is
literally what nearly killed the Mars Pathfinder mission in 1997.

Sha, Rajkumar & Lehoczky (1990) give two solutions:

- **Priority inheritance (PI)**: while `L` holds a lock a higher-priority task wants, `L`
  temporarily inherits that priority. Bounds inversion to the length of the critical section.
  Linux implements this in `rt_mutex` (`kernel/locking/rtmutex.c`), with full transitive
  chain walking, and exposes it to userspace via `PTHREAD_PRIO_INHERIT` futexes.
- **Priority ceiling**: a lock carries the max priority of any task that can take it; the
  holder is raised immediately. Prevents deadlock too, but requires static analysis. Linux
  doesn't use it.

On `PREEMPT_RT`, `spinlock_t` is *converted into* an `rt_mutex` so that spinlock waiters can
be preempted and PI applies — which is why RT has `raw_spinlock_t` for the genuinely
atomic core (scheduler, IRQ core) that must remain non-preemptible. Ch. 28 covers this.
**Corollary:** using `raw_spinlock_t` outside those cores is an RT regression and reviewers
will reject it.

### T.9 Reader-writer locks: why they usually lose

Intuition says: readers don't conflict, so let them share. Theory says: be careful.

`rwlock_t`/`rw_semaphore` still require every reader to **write** the shared counter to
register itself. So you get the *same* cache-line ping-pong as a mutex, plus a more expensive
unlock path. Measured result: for short critical sections, a plain `mutex`/`spinlock` often
**beats** a reader-writer lock even with 100% readers.

Reader-writer locks win only when the critical section is *long* (so the counter contention
amortizes) and the reader:writer ratio is high.

There is also a **policy** problem with no right answer:

- **Reader-preferring**: writers can starve indefinitely.
- **Writer-preferring**: readers can starve; and it breaks **recursive read acquisition**
  (reader takes it, a writer queues, the same reader re-takes it → self-deadlock). This is
  why `rwlock_t` read-recursion is a documented hazard and why lockdep warns about it.

Linux `rw_semaphore` is roughly writer-preferring with reader "spin-on-owner" optimism and
handoff logic to bound starvation — a pragmatic compromise, not a clean policy.

**The correct answer is usually to escape the trade-off entirely:**

| Technique | Reader cost | Idea |
|---|---|---|
| **RCU** (Ch. 15) | **zero** | readers don't write *anything*; writers defer reclamation |
| **seqlock** | 2 reads + barrier, no write | optimistic; readers retry if a writer interfered |
| **per-CPU + `percpu_rwsem`** | local line | readers touch only their own CPU's state |
| **sharding** | independent lines | split the data structure; hash the key to a lock |

This is the single most important scalability lesson in the kernel: **the winning move is not
a better lock, it is a data structure where readers don't write.**

### T.10 Seqlocks: optimistic concurrency control

Seqlocks are Kung & Robinson's *optimistic concurrency control* (1981) applied to memory:

```
Writer:  seq++  (now odd = "in progress")   ... modify ...   seq++  (even = stable)
Reader:  s = seq; if (s & 1) retry;  ...read...  if (seq != s) retry;
```

Properties, and they are sharp:

- **Readers do no writes at all** → perfectly scalable, zero coherence traffic among readers.
- **Writers are never blocked by readers** → no writer starvation. (Opposite of rwlock!)
- **Readers can starve** under continuous writing.
- **Readers may observe an inconsistent snapshot mid-read.** Therefore the read section must
  be *pure*: no pointer dereferences of data that may have been freed, no side effects, no
  dividing by a value that might be torn to zero. You may only *copy* and then validate.
  Violating this is a real, recurring bug class.
- Requires `smp_rmb()`-style ordering around the sequence reads — and on the write side a
  `smp_wmb()` — otherwise the whole protocol is vacuous (see Ch. 13).

Canonical users: the timekeeper (`jiffies`, `ktime_get()` — read constantly, written once per
tick), and `mount` hash lookups. `seqcount_latch_t` is a variant that keeps **two copies** so
readers never have to retry — used for NMI-safe timekeeping, where retrying is not an option
because a reader in NMI could deadlock against an interrupted writer.

### T.11 Special-purpose lock theory

- **`ww_mutex` (wound/wait)** — for acquiring an *a priori unknown set* of locks, as in a GPU
  command submission that must lock N buffers. Global ordering is impossible because the set
  isn't known in advance. Solution: classical database **deadlock avoidance** via timestamps
  (Rosenkrantz et al., 1978) — an older transaction "wounds" a younger holder, forcing it to
  back off and retry. This is *avoidance*, not *prevention*: it accepts rollback in exchange
  for not needing a static order. Know this one; it's the answer to "how do you lock a set you
  can't order?"
- **`local_lock`** — on non-RT it's just `preempt_disable()`; on RT it's a per-CPU sleeping
  lock. It exists to give *semantic meaning* to "this data is protected by disabling
  preemption", which previously was an undocumented, unverifiable convention. A great example
  of turning implicit knowledge into a checkable type.
- **`percpu_rwsem`** — readers take only their own CPU's lock (fast); writers must take all
  of them (very slow, uses RCU to synchronize the mode switch). The extreme point of the
  read-cheap/write-expensive trade. Used for CPU hotplug and filesystem freezing.
- **bit spinlocks** — a spinlock that lives in one bit of an existing word, for when memory is
  so tight you cannot afford 4 bytes (`struct page`, dcache, buffer heads). No lockdep
  coverage, no fairness. Last resort.

### T.12 Choosing a lock: a decision procedure

```
Can this code sleep? (task context, no spinlock held, no RCU read section)
├─ NO  → spinlock_t.
│        Taken from hardirq?  → spin_lock_irqsave()
│        Taken from softirq?  → spin_lock_bh()
│        Core scheduler/irq/RT-critical? → raw_spinlock_t (justify it)
└─ YES → Is the critical section long, or does it sleep inside?
         ├─ YES → mutex (default), or rw_semaphore if genuinely read-heavy AND long
         └─ NO  → mutex anyway (adaptive spinning makes it ~spinlock cost when uncontended)

Then ask the scalability question BEFORE finalizing:
  Is this read-mostly, with readers that only traverse?          → RCU
  Is it a small, copyable snapshot read very frequently?         → seqlock
  Is the state naturally per-CPU?                                → per-CPU + local_lock
  Is it a big table with independent entries?                    → shard the lock
  Is it just a counter?                                          → atomic / per-CPU counter
  Is the "lock" protecting exactly one pointer swap?             → cmpxchg, no lock
```

The senior move is that **the second block runs first**. Junior engineers pick a lock;
seniors design the data structure so the lock barely matters.

---

## 1. Concept — the Linux locking API surface

| Primitive | Context | Sleeps | Notes |
|---|---|---|---|
| `spinlock_t` | any | no (yes on RT!) | qspinlock; 4 bytes; the workhorse |
| `raw_spinlock_t` | any | **never** | true spinning even on RT; core code only |
| `rwlock_t` | any | no | **discouraged** — reader-preferring, poor scaling |
| `struct mutex` | task | yes | adaptive spin; **the default** |
| `struct rw_semaphore` | task | yes | `down_read`/`down_write`; long read sections |
| `struct semaphore` | task | yes | counting; **legacy** — almost never correct today |
| `struct completion` | task | yes | one-shot event, not mutual exclusion |
| `seqlock_t` / `seqcount_t` | any | no | read-mostly, copyable data |
| `struct rt_mutex` | task | yes | priority inheritance; futex, RT |
| `struct ww_mutex` | task | yes | unordered lock sets (DRM) |
| `local_lock_t` | task | no (yes on RT) | per-CPU data protection |
| `struct percpu_rw_semaphore` | task | yes | read-cheap, write-catastrophic |
| bit spinlock | any | no | `bit_spin_lock()`; no lockdep |
| RCU | any | no (reader) | Ch. 15 — not a lock at all |
| SRCU | any | **reader may sleep** | Ch. 15 |

### 1.1 The IRQ-safety variants, and why they exist

```c
spin_lock(&l);                       /* no interrupt protection */
spin_lock_irq(&l);                   /* disables local IRQs; use only if you KNOW they were on */
spin_lock_irqsave(&l, flags);        /* ★ saves+disables; the safe default */
spin_lock_bh(&l);                    /* disables softirqs */
```

The rule follows directly from T.7: **if a lock is ever taken in interrupt context, every
other acquisition of that lock must disable that interrupt class on the local CPU.**
Otherwise: CPU0 takes `L` in task context → interrupt fires on CPU0 → handler tries `L` →
self-deadlock. Note it's a *local* CPU problem; another CPU's handler simply spins, which is
fine.

Modern kernels also have **cleanup/guard** helpers (6.4+, `include/linux/cleanup.h`) that
eliminate whole classes of "forgot to unlock on the error path" bugs:

```c
guard(mutex)(&my_mutex);              /* unlocked automatically at end of scope */

scoped_guard(spinlock_irqsave, &my_lock) {
	/* critical section */
}

struct foo *f __free(kfree) = kmalloc(...);   /* auto-freed on any return */
```
This is `__attribute__((cleanup))` — RAII in C. New code is expected to use it, and it
removes the most common reason for the `goto` unwind ladder.

---

## 2. Internals

### 2.1 Where the code lives

```
kernel/locking/qspinlock.c        queued spinlock (MCS-based)
kernel/locking/qspinlock_paravirt.h  PV variant
kernel/locking/mutex.c            mutex + adaptive spinning
kernel/locking/rwsem.c            rw_semaphore
kernel/locking/rtmutex.c          priority inheritance
kernel/locking/ww_mutex.h         wound/wait
kernel/locking/lockdep.c          ★ the validator
kernel/locking/osq_lock.c         optimistic spin queue (MCS for spin-on-owner)
kernel/locking/percpu-rwsem.c
include/linux/seqlock.h
include/linux/cleanup.h           guard()/__free()
Documentation/locking/            ★ all of it
```

### 2.2 Mutex: the fast path and the slow path

```c
struct mutex {
	atomic_long_t      owner;   /* task pointer + flag bits in low bits */
	raw_spinlock_t     wait_lock;
	struct optimistic_spin_queue osq;   /* MCS queue for spinners */
	struct list_head   wait_list;
};
```

- **Fast path**: `atomic_long_cmpxchg_acquire(&lock->owner, 0, curr)`. One atomic. Done.
- **Midpath (optimistic spinning)**: join the OSQ (an MCS queue, so spinners don't ping-pong)
  and spin while `owner` is unchanged **and** `owner_on_cpu(owner)` is true. Bail out the
  moment the owner is descheduled or a higher-priority task needs the CPU.
- **Slow path**: take `wait_lock`, add to `wait_list`, `set_current_state(TASK_UNINTERRUPTIBLE)`,
  `schedule()`.

The low bits of `owner` carry `MUTEX_FLAG_WAITERS`, `MUTEX_FLAG_HANDOFF` (for starvation
avoidance — after waiting too long, the unlocker hands the lock *directly* to the head waiter
rather than letting a spinner steal it), and `MUTEX_FLAG_PICKUP`. Packing state into pointer
low bits is a pervasive kernel idiom.

### 2.3 Reading a lockdep report

```
======================================================
WARNING: possible circular locking dependency detected
6.9.0 #1 Not tainted
------------------------------------------------------
kworker/2:1/68 is trying to acquire lock:
 ffff8881040a0118 (&dev->mutex){+.+.}-{3:3}, at: my_work_fn+0x2a/0x90

but task is already holding lock:
 ffffffffc0123456 (&priv->lock){+.+.}-{3:3}, at: my_work_fn+0x18/0x90

which lock already depends on the new lock.
...
 Possible unsafe locking scenario:
       CPU0                    CPU1
       ----                    ----
  lock(&priv->lock);
                               lock(&dev->mutex);
                               lock(&priv->lock);
  lock(&dev->mutex);

 *** DEADLOCK ***
```

Decode the annotations `{+.+.}-{3:3}`:

| Position | Meaning |
|---|---|
| 1st char | taken in **hardirq** context? `.` no, `+` yes-with-irqs-on, `-` yes-with-irqs-off, `?` both |
| 2nd char | taken with hardirqs **disabled**? |
| 3rd char | taken in **softirq** context? |
| 4th char | taken with softirqs disabled? |
| `{3:3}` | lock **class depth / usage** (read/write, and the lockdep subclass) |

The "Possible unsafe locking scenario" block is lockdep *constructing the counterexample for
you*. Read it, find both call sites in your code, and fix the order. It is almost never
wrong; if you think it is, you probably need `lockdep_set_class()` because two logically
independent locks share a class.

### 2.4 Contention measurement

```bash
# Enable CONFIG_LOCK_STAT=y
echo 1 | sudo tee /proc/sys/kernel/lock_stat
# ... run workload ...
sudo cat /proc/lock_stat | head -40
echo 0 | sudo tee /proc/sys/kernel/lock_stat
```

Columns: `con-bounces` (cache-line bounces — the $\kappa$ term!), `contentions`,
`waittime-min/avg/max/total`, `acq-bounces`, `acquisitions`, `holdtime-*`.
**`waittime-total` and `holdtime-avg` are the two numbers that matter.**

```bash
# Modern alternative — no rebuild needed:
sudo perf lock record -a -- sleep 5 && sudo perf lock contention
sudo perf lock contention -ab -- sleep 5      # BPF-based, low overhead
sudo bpftrace -e 'kprobe:mutex_lock { @[kstack] = count(); }'
```

---

## 3. Practice

### 3.1 A correctly-locked object with documented ordering

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * Lock ordering:
 *   registry_lock  ->  widget->lock
 * Never the reverse. widget->lock may be taken from softirq, so all
 * acquisitions must disable softirqs.
 */
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/mutex.h>
#include <linux/spinlock.h>
#include <linux/slab.h>
#include <linux/cleanup.h>

struct widget {
	spinlock_t        lock;      /* protects counter; taken from softirq */
	u64               counter;
	struct list_head  node;      /* protected by registry_lock */
	char              name[16];
};

static DEFINE_MUTEX(registry_lock);
static LIST_HEAD(registry);

static int widget_add(const char *name)
{
	struct widget *w = kzalloc(sizeof(*w), GFP_KERNEL);

	if (!w)
		return -ENOMEM;

	spin_lock_init(&w->lock);
	strscpy(w->name, name, sizeof(w->name));

	guard(mutex)(&registry_lock);          /* auto-unlock on every exit path */
	list_add_tail(&w->node, &registry);
	return 0;
}

/* Called from softirq — must not sleep, so spinlock, and _bh on the other side. */
void widget_bump(struct widget *w)
{
	guard(spinlock)(&w->lock);
	w->counter++;
}

static u64 widget_read(struct widget *w)
{
	guard(spinlock_bh)(&w->lock);          /* task context: must block softirq */
	return w->counter;
}

static void widget_dump_all(void)
{
	struct widget *w;

	guard(mutex)(&registry_lock);
	list_for_each_entry(w, &registry, node)
		pr_info("%s: %llu\n", w->name, widget_read(w));
		/* ^ registry_lock -> w->lock : matches the documented order */
}
```

### 3.2 Seqlock for a read-mostly snapshot

```c
#include <linux/seqlock.h>

struct stats {
	u64 packets;
	u64 bytes;
	u64 errors;
};

static struct stats cur_stats;
static DEFINE_SEQLOCK(stats_lock);

void stats_update(u64 pkts, u64 bytes)
{
	write_seqlock(&stats_lock);
	cur_stats.packets += pkts;
	cur_stats.bytes   += bytes;
	write_sequnlock(&stats_lock);
}

void stats_read(struct stats *out)
{
	unsigned int seq;

	do {
		seq = read_seqbegin(&stats_lock);
		*out = cur_stats;        /* PURE COPY. No pointer chasing, no side effects. */
	} while (read_seqretry(&stats_lock, seq));
}
```
Note what `stats_read()` does *not* do: it never dereferences a pointer read from the
protected data, never divides, never calls anything. That restriction is not style — it is
required by T.10.

### 3.3 Dev-kernel config for locking work

```
CONFIG_PROVE_LOCKING=y
CONFIG_DEBUG_SPINLOCK=y
CONFIG_DEBUG_MUTEXES=y
CONFIG_DEBUG_RWSEMS=y
CONFIG_DEBUG_LOCK_ALLOC=y
CONFIG_DEBUG_ATOMIC_SLEEP=y     # catches sleeping in atomic context
CONFIG_LOCK_STAT=y
CONFIG_PROVE_RCU=y
CONFIG_DEBUG_WW_MUTEX_SLOWPATH=y
CONFIG_KCSAN=y                  # data races (Ch. 13)
CONFIG_LOCK_TORTURE_TEST=m      # kernel/locking/locktorture.c
CONFIG_DETECT_HUNG_TASK=y
CONFIG_WQ_WATCHDOG=y
```

```bash
# Torture-test the locking primitives themselves:
sudo modprobe locktorture torture_type=mutex_lock nwriters_stress=4 stat_interval=10
dmesg | tail
```

---

## 3.4 Extended practice

### Lab 14.A — Find the ski-rental crossover on your machine (T.2)

```c
// SPDX-License-Identifier: GPL-2.0
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/kthread.h>
#include <linux/mutex.h>
#include <linux/spinlock.h>
#include <linux/delay.h>
#include <linux/ktime.h>
#include <linux/cpu.h>

static int hold_ns = 100;      /* critical-section length */
static int nthreads = 4;
static int use_mutex;
module_param(hold_ns, int, 0644);
module_param(nthreads, int, 0644);
module_param(use_mutex, int, 0644);

static DEFINE_SPINLOCK(slock);
static DEFINE_MUTEX(mlock);
static atomic64_t total_ops;
static atomic_t   stop;
static struct task_struct **tasks;

static void spin_ns(int ns)
{
	u64 end = ktime_get_ns() + ns;

	while (ktime_get_ns() < end)
		cpu_relax();
}

static int worker(void *unused)
{
	u64 ops = 0;

	while (!atomic_read(&stop) && !kthread_should_stop()) {
		if (use_mutex) {
			mutex_lock(&mlock);
			spin_ns(hold_ns);
			mutex_unlock(&mlock);
		} else {
			spin_lock(&slock);
			spin_ns(hold_ns);
			spin_unlock(&slock);
		}
		ops++;
		if (!(ops & 0xff))
			cond_resched();
	}
	atomic64_add(ops, &total_ops);
	return 0;
}

static int __init lockbench_init(void)
{
	int i;
	u64 t0;

	tasks = kcalloc(nthreads, sizeof(*tasks), GFP_KERNEL);
	if (!tasks)
		return -ENOMEM;

	t0 = ktime_get_ns();
	for (i = 0; i < nthreads; i++) {
		tasks[i] = kthread_create(worker, NULL, "lockbench/%d", i);
		kthread_bind(tasks[i], i % num_online_cpus());
		wake_up_process(tasks[i]);
	}
	msleep(3000);
	atomic_set(&stop, 1);
	for (i = 0; i < nthreads; i++)
		kthread_stop(tasks[i]);

	pr_info("%-8s threads=%2d hold=%6dns -> %llu ops/s\n",
		use_mutex ? "mutex" : "spinlock", nthreads, hold_ns,
		atomic64_read(&total_ops) * 1000000000ULL / (ktime_get_ns() - t0));
	kfree(tasks);
	return 0;
}
static void __exit lockbench_exit(void) { }
module_init(lockbench_init); module_exit(lockbench_exit);
MODULE_LICENSE("GPL");
```
```bash
for m in 0 1; do for h in 100 500 1000 5000 20000 100000; do
  for n in 1 2 4 8; do
    sudo insmod lockbench.ko use_mutex=$m hold_ns=$h nthreads=$n; sudo rmmod lockbench
  done
done; done
dmesg | grep lockbench
```
**Plot ops/s vs hold time for each primitive.** You are looking for two things:
1. The **crossover** where `mutex` overtakes `spinlock` — that is your machine's `S` from T.2.
2. The **peak-then-decline** shape as `nthreads` grows — that is the USL's $\kappa N^2$ term
   (T.5). Fit $\sigma$ and $\kappa$; the numbers will make you permanently suspicious of
   shared locks.

### Lab 14.B — Induce, read, and fix every lockdep report class (T.7)

```c
/* mode=1 : ABBA deadlock between two mutexes */
/* mode=2 : IRQ-unsafe -> IRQ-safe inversion */
/* mode=3 : recursive read on a rwlock */
/* mode=4 : sleeping in atomic context */
/* mode=5 : same-class nesting without _nested() */
static int mode = 1;
module_param(mode, int, 0644);

static DEFINE_MUTEX(a), DEFINE_MUTEX(b);
static DEFINE_SPINLOCK(irq_lock);

static int t1(void *u)
{
	if (mode == 1) { mutex_lock(&a); msleep(10); mutex_lock(&b);
			 mutex_unlock(&b); mutex_unlock(&a); }
	return 0;
}
static int t2(void *u)
{
	if (mode == 1) { mutex_lock(&b); msleep(10); mutex_lock(&a);
			 mutex_unlock(&a); mutex_unlock(&b); }
	return 0;
}

/* mode 2: take irq_lock WITHOUT disabling irqs here, but WITH it in an ISR */
static void inversion(void)
{
	spin_lock(&irq_lock);        /* lockdep: HARDIRQ-unsafe -> HARDIRQ-safe */
	udelay(10);
	spin_unlock(&irq_lock);
}
static irqreturn_t isr(int irq, void *d)
{
	spin_lock(&irq_lock);
	spin_unlock(&irq_lock);
	return IRQ_HANDLED;
}

/* mode 4 */
static void sleep_in_atomic(void)
{
	spin_lock(&irq_lock);
	msleep(1);                   /* BUG: sleeping function called from invalid context */
	spin_unlock(&irq_lock);
}

/* mode 5: two inodes, same lock class */
static void same_class(struct mutex *m1, struct mutex *m2)
{
	mutex_lock(m1);
	mutex_lock(m2);              /* lockdep: possible recursive locking */
	/* FIX: mutex_lock_nested(m2, SINGLE_DEPTH_NESTING); and order by address */
	mutex_unlock(m2);
	mutex_unlock(m1);
}
```
```bash
./scripts/config -e PROVE_LOCKING -e DEBUG_ATOMIC_SLEEP -e DEBUG_MUTEXES -e DEBUG_SPINLOCK
for m in 1 2 3 4 5; do
  sudo insmod lockbug.ko mode=$m; sleep 2; sudo rmmod lockbug 2>/dev/null
  echo "### mode=$m"; sudo dmesg -c | head -60
done
```
For each report, annotate: the `{+.+.}-{3:3}` flags, the "possible unsafe locking scenario"
block, and the two stack traces. Then fix each and confirm silence. **Keep the annotated
output as a reference sheet.**

### Lab 14.C — Measure contention like a professional

```bash
# (a) Built-in lock statistics
./scripts/config -e LOCK_STAT && make -j$(nproc) && boot
echo 1 | sudo tee /proc/sys/kernel/lock_stat
fio --name=rw --rw=randrw --bs=4k --numjobs=$(nproc) --size=1G --runtime=30 --group_reporting
echo 0 | sudo tee /proc/sys/kernel/lock_stat
sudo head -3 /proc/lock_stat
sudo awk 'NR>3 && NF>6' /proc/lock_stat | sort -k6 -rn | head -20   # by waittime-total

# (b) perf lock — no rebuild required, BPF-backed
sudo perf lock contention -ab --  sleep 10
sudo perf lock contention -abt -- sleep 10        # per-thread
sudo perf lock contention -abl -- sleep 10        # by lock address/caller

# (c) Where are we spinning?
sudo perf record -F 999 -a -g -- sleep 10
sudo perf report --stdio --sort symbol | grep -iE 'lock|osq|mutex|rwsem' | head -20

# (d) Cache-line level truth (Ch. 16)
sudo perf c2c record -a -- sleep 10 && sudo perf c2c report --stdio | head -60

# (e) bpftrace: mutex hold-time histogram
sudo bpftrace -e '
kprobe:mutex_lock   { @s[tid] = nsecs; }
kretprobe:mutex_lock /@s[tid]/ { @acq_ns = hist(nsecs - @s[tid]); delete(@s[tid]); }'
```

### Lab 14.D — Convert a `goto`-unlock ladder to `guard()` (T.4/Ch. 05)

```bash
# Find candidates:
git grep -n -A2 'goto.*unlock' drivers/ | head -30
```
Pick a function, convert it, and **verify the generated code is equivalent**:
```bash
make drivers/foo/bar.o && objdump -d drivers/foo/bar.o > /tmp/before.s
# ...edit...
make drivers/foo/bar.o && objdump -d drivers/foo/bar.o > /tmp/after.s
diff /tmp/before.s /tmp/after.s
```
Then read the pitfalls:
```bash
$EDITOR include/linux/cleanup.h     # the header comment; note the `return_ptr()` rule
git log --oneline --grep='guard(' | head -20
```

### Lab 14.E — Escape the lock entirely: four rewrites of one problem

Take a shared counter under a spinlock and rewrite it four ways, measuring each:

```c
/* 1. spinlock + plain counter    — the baseline */
/* 2. atomic64_t                  — removes the lock, keeps the shared line */
/* 3. per-CPU counter             — removes the shared line entirely (Ch. 16) */
/* 4. percpu_counter with batch   — per-CPU with a bounded-error global view */
```
Run each with 1…N threads. You should see (1) and (2) peak and decline, (3) go linear, and
(4) sit just below (3). **This is the Ch. 14 T.12 decision procedure, empirically justified.**

### Lab 14.F — Torture the primitives

```bash
./scripts/config -m LOCK_TORTURE_TEST -e PROVE_LOCKING
sudo modprobe locktorture torture_type=mutex_lock  nwriters_stress=4 stat_interval=10
sleep 60; sudo rmmod locktorture; dmesg | tail -20

for t in spin_lock spin_lock_irq rw_lock mutex_lock rtmutex_lock rwsem_lock ww_mutex_lock percpu_rwsem_lock; do
  sudo modprobe locktorture torture_type=$t nwriters_stress=4 stat_interval=15
  sleep 30; sudo rmmod locktorture
  echo "=== $t"; dmesg | tail -5
done
$EDITOR kernel/locking/locktorture.c    # read how it VERIFIES mutual exclusion
```
The verification trick (a shared counter that must never be observed changing inside a
critical section) is worth stealing for your own subsystem tests.

### Lab 14.G — Read `qspinlock.c` with a debugger

```bash
$EDITOR kernel/locking/qspinlock.c
# Answer in writing:
#  1. What are the three tiers, and what is the exact condition for entering each?
#  2. How does `tail` encode (cpu, idx) in 16 bits? Why exactly 4 idx values?
#  3. What does the `pending` bit optimize, and for which contention level?
#  4. Where does the MCS node come from, and why is it never allocated?
grep -n 'MAX_NODES\|encode_tail\|decode_tail\|_Q_PENDING' kernel/locking/qspinlock.c | head -20

# Then watch it in QEMU:
(gdb) b queued_spin_lock_slowpath
(gdb) p/x *(struct qspinlock *)lock
```

---

## 4. Mastery drills

1. **Induce a lockdep splat deliberately.** Write a module with two mutexes and two kthreads
   that take them in opposite orders. Run with `PROVE_LOCKING=y`. Read the report and map
   every line back to your code. Then fix it and confirm silence.

2. **Measure the ski-rental crossover.** Build a module with a configurable critical-section
   length. Benchmark `spinlock_t` vs `mutex` at hold times of 100 ns, 1 µs, 10 µs, 100 µs
   with 1/2/4/N threads. Plot it. Find your machine's crossover point and compare with the
   theoretical `S`.

3. **Demonstrate the USL.** Take a shared counter under a spinlock and measure aggregate
   throughput at 1, 2, 4, 8, … CPUs. Show the *peak-then-decline* shape. Then replace with
   a per-CPU counter and show it goes linear. Fit $\sigma$ and $\kappa$.

4. **Read `qspinlock.c`.** Trace all three tiers. Explain how `tail` encodes a CPU and why
   there are exactly 4 per-CPU MCS nodes. Explain what `pv_wait()` solves.

5. **Find a real deadlock fix.** `git log --grep="lockdep" --oneline -- drivers/ | head -30`.
   Pick three. For each, state which Coffman condition was violated and what the fix did.

6. **Convert to guards.** Find a function in `drivers/` with a long `goto out_unlock` ladder.
   Rewrite using `guard()`/`scoped_guard()`. Verify the semantics are identical. (Several
   such cleanup series have been merged — a legitimate first-patch area.)

7. **Seqlock misuse hunt.** Construct a broken seqlock reader that dereferences a pointer
   read inside the read section. Explain the exact interleaving that crashes. Then explain
   why `latch` seqcounts exist for NMI.

8. **ww_mutex.** Read `Documentation/locking/ww-mutex-design.rst` and the DRM usage in
   `drivers/gpu/drm/drm_exec.c`. Explain why a global lock order is impossible there, and
   trace the wound/backoff/retry loop.

9. **RT impact.** Explain to a reviewer why converting a `spinlock_t` to `raw_spinlock_t`
   "for performance" is a PREEMPT_RT regression. Cite what happens to latency.

---

## 5. Further reading

**Kernel docs (all short, all mandatory):**
- `Documentation/locking/locktypes.rst` — **start here**; the definitive "which lock" doc
- `Documentation/locking/lockdep-design.rst`
- `Documentation/locking/mutex-design.rst`, `rt-mutex-design.rst`, `ww-mutex-design.rst`
- `Documentation/locking/seqlock.rst`
- `Documentation/locking/spinlocks.rst`, `hwspinlock.rst`
- `Documentation/kernel-hacking/locking.rst`

**Papers:**
- Dijkstra, "Solution of a Problem in Concurrent Programming Control" (CACM 1965)
- Lamport, "A New Solution of Dijkstra's Concurrent Programming Problem" (1974) — Bakery
- Mellor-Crummey & Scott, "Algorithms for Scalable Synchronization on Shared-Memory
  Multiprocessors" (TOCS 1991) — **MCS locks; read this one**
- Sha, Rajkumar & Lehoczky, "Priority Inheritance Protocols" (IEEE ToC 1990)
- Kung & Robinson, "On Optimistic Methods for Concurrency Control" (TODS 1981) — seqlocks
- Rosenkrantz, Stearns & Lewis, "System Level Concurrency Control for Distributed Database
  Systems" (1978) — wound/wait
- Coffman, Elphick & Shoshani, "System Deadlocks" (1971)
- Boyd-Wickizer et al., "Non-scalable locks are dangerous" (OLS 2012) — **why ticket locks
  collapse; the paper that motivated qspinlock**

**Books/other:**
- McKenney, *Is Parallel Programming Hard…* — Ch. 6–7, 10
- Herlihy & Shavit, *The Art of Multiprocessor Programming* — the academic reference
- Gunther, *Guerrilla Capacity Planning* — the Universal Scalability Law
- LWN: "The ticket spinlock", "MCS locks and qspinlocks", "Cleaning up with `guard()`",
  "NUMA-aware qspinlocks", "The mutex handoff"

→ Next: [15-rcu.md](15-rcu.md)

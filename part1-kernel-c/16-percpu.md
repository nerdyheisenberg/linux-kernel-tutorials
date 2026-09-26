# Chapter 16 — Per-CPU Data, `local_lock`, and the Theory of Scalable Design

> **Goal:** stop asking "which lock?" and start asking "how do I make the sharing disappear?"
> This chapter is the constructive counterpart to Ch. 14's analysis of why locks don't scale.

---

## Theory & First Principles

### T.0 — Start here: a counter that gets slower with more CPUs

The simplest possible shared state. Count packets.

```c
atomic64_t packets;

void rx(void) { atomic64_inc(&packets); }        /* on every packet, every CPU */
```

Correct, obvious, and it does not scale. Measure increments per second as you add cores:

```
 increments/sec
     │ ╱╲
     │╱   ╲╲╲╲╲╲╲╲╲╲╲╲╲╲╲     1 core: ~100M/s
     │                             8 cores: ~15M/s   ← SEVEN times SLOWER per core
     └──────────────── cores    (Ch. 105 Lab 2 measures this)
```

**There is no lock here and no contention in any logical sense** — every CPU is doing an
independent increment of a shared total. And yet adding CPUs makes the aggregate *worse*.

**Why.** An atomic RMW needs the cacheline in **exclusive** state. Only one cache can hold it
exclusively at a time. So the line migrates between every CPU that touches it, continuously,
and each migration is 100–400 ns (Ch. 00 §T.3). The counter is not slow because of the `lock`
prefix; it is slow because **64 bytes of memory can only be in one place at a time.**

> **The real unit of contention is the cache line, not the lock and not the variable.**

That sentence is the entire chapter. Two consequences follow, and the second is the one
people miss:

**Consequence 1 — sharing a line you did not mean to share costs the same.**

```c
struct stats {
	u64 rx_packets;    /* CPU 0 writes this */
	u64 tx_packets;    /* CPU 1 writes this */
};                         /* SAME 64-byte line. */
```

No shared variable. No race. No lock. **And exactly the same ping-pong**, because coherence
operates on lines, not variables. This is **false sharing** (§T.2), it is invisible in code
review, and `perf c2c` is how you find it (Ch. 105 Lab 2).

**Consequence 2 — the fix is not a faster atomic. It is to stop sharing.**

```c
DEFINE_PER_CPU(u64, packets);

void rx(void) { this_cpu_inc(packets); }     /* NOT atomic. Not shared. */

u64 total(void) {                            /* the cost moves HERE */
	u64 sum = 0; int cpu;
	for_each_possible_cpu(cpu)
		sum += per_cpu(packets, cpu);
	return sum;
}
```

Each CPU writes only its own line, which stays in its own L1 forever. The increment becomes
a plain `inc` — no atomic, no barrier, no coherence traffic — and throughput is genuinely
linear in cores.

**The trade, stated explicitly**, because this is the pattern you will apply dozens of times:

| | Shared atomic | Per-CPU + fold |
|---|---|---|
| Write | ~20–300 ns, scales *negatively* | **~1 ns, scales linearly** |
| Read | ~1 ns, exact | **O(N_cpus)**, and *approximate* while writes continue |
| Memory | 8 bytes | 8 × N_cpus, padded to lines |

**You have moved the cost from the frequent operation to the rare one.** That is correct
whenever writes outnumber reads — which is true of essentially every statistic, and false for
things read on every operation. Recognizing which case you are in is the skill; `percpu_counter`
(§T.6) is the hybrid for when you need both.

And there is a deeper result waiting in §T.3: the **Scalable Commutativity Rule** says that
whenever interface operations *commute*, they can be implemented to scale. `rx()` commutes
with itself; `total()` does not commute with `rx()`. **The scalability was determined by the
interface, before anyone wrote an implementation** — which is why this is an architecture
chapter and not an optimization chapter.

---

### T.1 The real unit of contention is the cache line

A modern CPU's coherence protocol operates on **cache lines** (64 B on x86-64/ARM64, 128 B
effective on some parts due to adjacent-line prefetch, 256 B on some POWER). The protocol
(MESI/MOESI/MESIF) enforces the single-writer/multiple-reader invariant *per line*:

| State | Meaning | Cost to write |
|---|---|---|
| **M**odified | this core has the only dirty copy | free |
| **E**xclusive | this core has the only clean copy | free (M transition is local) |
| **S**hared | several cores have clean copies | **must invalidate all others** |
| **I**nvalid | not present | **must fetch, possibly from another core's cache** |

Rough cost model on a modern server part:

| Access | Latency |
|---|---|
| L1 hit (line in M/E) | ~1 ns (4 cycles) |
| L3 hit, same socket | ~15 ns |
| **Line owned by another core, same socket** | **~40–80 ns** |
| **Line owned by another socket** | **~150–300 ns** |
| Atomic RMW on a contended line | 100 ns – several µs under load |

**The decisive fact:** a contended atomic increment can cost *100× more* than an uncontended
one. It is not the atomic instruction that is expensive — it is the line transfer.

So the objective function for scalable design is not "fewer instructions" but
**"fewer cache lines written by more than one CPU."**

### T.2 False sharing: paying the price for nothing

```c
struct bad {
	unsigned long cpu0_counter;   /* offset 0  */
	unsigned long cpu1_counter;   /* offset 8  — SAME CACHE LINE */
};
```
Two CPUs updating *logically independent* variables that happen to share a line will ping-pong
that line forever. Throughput can drop 10–100×. This is **false sharing**, and it is invisible
in source code — you must reason about layout.

Kernel tools:

```c
struct good {
	unsigned long a;
	unsigned long b ____cacheline_aligned_in_smp;
};

struct hot {
	spinlock_t lock;
	int state;
} ____cacheline_aligned;

/* Group read-mostly data away from written data: */
static int config __read_mostly;
static DEFINE_PER_CPU_ALIGNED(struct stats, my_stats);
```

`__read_mostly` places the symbol in a separate section (`.data..read_mostly`) so that
rarely-written globals don't share lines with hot counters. `____cacheline_aligned_in_smp`
pads to `L1_CACHE_BYTES`. `struct_group()` and `CACHELINE_PADDING()` help express intent.

Detection is empirical:

```bash
sudo perf c2c record -a -- sleep 10 && sudo perf c2c report --stdio
# "HITM" (hit-modified) counts are the smoking gun for true/false sharing
pahole -C task_struct vmlinux      # see the actual layout + holes
pahole --reorganize -C my_struct vmlinux
```
`perf c2c` is the professional tool for this. Learn it now; almost nobody uses it, and it
makes you look like a wizard.

### T.3 The Scalable Commutativity Rule

This is the deepest result in the field, and it should change how you design APIs.

Clements, Kaashoek, Zeldovich, Morris & Kohler, *"The Scalable Commutativity Rule: Designing
Scalable Software for Multicore Processors"* (SOSP 2013, best paper):

> **Whenever interface operations commute, they can be implemented in a way that scales.**

More precisely: if two operations commute (executing them in either order yields the same
results and the same subsequent observable behaviour), then there exists an implementation in
which they access **no common cache line** — hence they scale perfectly.

The contrapositive is what matters to you:

> **If your interface forces operations to *not* commute, no implementation can scale.**
> The bottleneck is in the *specification*, not the code.

The canonical example is POSIX `open()`: it is specified to return the **lowest available**
file descriptor. Two concurrent `open()` calls therefore do *not* commute — they must agree on
which one gets the lower number — so they must share state. No amount of clever locking fixes
this; the API is unscalable *by definition*. (Linux's fd table uses a per-process bitmap and
a spinlock; it's fine because fd tables are per-process, but the principle stands, and
`O_TMPFILE`/`close_range()` style APIs were designed with this in mind.)

Practical consequences for kernel API design — this is architect-level material:

- Avoid specifying "lowest/first/smallest available". Return *an* identifier, not *the*
  identifier. (`xa_alloc()` vs a strict counter.)
- Avoid global ordering guarantees you don't need. A globally ordered event log is
  unscalable; per-CPU logs merged at read time are not.
- Avoid exact global counters in an API contract. `/proc/stat` is read rarely; make the
  counters per-CPU and sum on read.
- Prefer operations on disjoint objects to operations on a shared namespace.

The authors built a tool (COMMUTER) that automatically found non-commutative cases in the
POSIX spec, and rewrote a kernel (sv6) to scale where Linux did not. Cite this paper in a
design review and people will take you seriously.

### T.4 Partitioning: the constructive technique

If sharing is the disease, **partitioning** is the cure. Three flavours, in increasing order
of strength:

| Technique | Partition key | Aggregation cost | Example |
|---|---|---|---|
| **Sharding / lock striping** | hash of object | none (independent) | `inode_hashtable` buckets, futex hash |
| **Per-CPU** | `smp_processor_id()` | O(nr_cpus) on read | counters, slab caches, `percpu_ref` |
| **Per-task / per-thread** | `current` | O(nr_tasks) | `current->io_context`, `rss_stat` |

Per-CPU is special because the partition key is *free* (it's where you're running) and
because the kernel can guarantee no other CPU touches your slot — which means **you can use
non-atomic operations**. That's the real win: `this_cpu_inc()` is a single non-`lock`-prefixed
instruction on x86 (`incq %gs:offset`), ~1 ns, versus ~20–100 ns for a contended atomic.

The catch is the **read side inverts**: reading the true total is now `O(nr_cpus)` and is
only ever *approximately* consistent (you cannot atomically snapshot 256 counters). That's
the trade, and it is almost always worth it because these counters are written billions of
times and read by `/proc` occasionally.

### T.5 Approximate counters and the error/cost curve

`percpu_counter` (`lib/percpu_counter.c`) generalizes the idea with a tunable error bound:

```c
struct percpu_counter {
	raw_spinlock_t  lock;
	s64             count;        /* the shared, approximate total */
	s32 __percpu   *counters;     /* per-CPU deltas */
};
```

`percpu_counter_add(fbc, n)` adds to the local delta; when `|delta| > batch`, it folds the
delta into the shared `count` under the lock. So:

- Shared-line traffic is reduced by a factor of `batch`.
- The error in `percpu_counter_read()` is bounded by `nr_cpus × batch`.
- `percpu_counter_sum()` takes the lock and walks all CPUs for an *exact-ish* value.

`percpu_counter_batch` is auto-tuned as `max(32, 2*nr_cpus)`. Used by ext4 free-block counts,
network connection counts, memcg, and dirty-page accounting — all places where a wrong answer
by 1000 is harmless but a contended counter is fatal.

The general principle: **relaxing exactness buys scalability, and the exchange rate is
explicit.** When you propose an approximate counter in review, state the bound.

A related family:
- `percpu_ref` (Ch. 12, T.8) — per-CPU while alive, atomic while dying.
- `atomic_long_t` + `__percpu` "local" variants — `local_t`/`local64_t`, atomic with respect
  to **interrupts on this CPU only**, so no `lock` prefix. Used by perf ring buffers and
  ftrace, which must be NMI-safe but never cross-CPU.

### T.6 What protects per-CPU data? (the preemption problem)

Per-CPU data is not automatically safe. Two threats:

1. **Migration.** If you compute `this_cpu_ptr(&x)` and are then migrated, you're operating
   on another CPU's slot. → must disable preemption (or migration) across the access.
2. **Interrupts.** A hardirq/softirq on the *same* CPU can touch the same per-CPU variable
   mid-update. → must disable IRQs/softirqs if that's possible.

This yields the accessor families:

| API | Guarantees | Notes |
|---|---|---|
| `this_cpu_inc(v)` | atomic w.r.t. preemption **and** interrupts on this CPU | single instruction on x86; safe everywhere |
| `__this_cpu_inc(v)` | **no** guarantees | caller must already have disabled preemption; debug-checked |
| `get_cpu_var(v)` / `put_cpu_var(v)` | disables preemption around access | legacy style |
| `this_cpu_ptr(&v)` | just computes the address | **caller must pin the CPU** |
| `per_cpu_ptr(&v, cpu)` | address for a specific CPU | used for aggregation |
| `get_cpu()` / `put_cpu()` | disable preemption, return cpu id | |

x86's `this_cpu_*` are single instructions with a `%gs` segment prefix, so they are
*inherently* atomic against preemption and interrupts without any disabling. On
architectures without such addressing, the generic fallback disables interrupts. **This
architecture asymmetry is why `this_cpu_*` is preferred over hand-rolled
`preempt_disable(); __this_cpu_...; preempt_enable();`** — it lets each arch use its best
option.

### T.7 `local_lock`: turning a convention into a checkable type

For years, "this per-CPU data is protected by `preempt_disable()`" was an *undocumented
convention*. You could not tell from the code which data a `preempt_disable()` protected, and
lockdep could not check anything.

`local_lock_t` (5.8+) fixes this by making the protection an explicit, named object:

```c
struct my_percpu_state {
	local_lock_t lock;
	struct list_head pending;
	int count;
};
static DEFINE_PER_CPU(struct my_percpu_state, mps) = {
	.lock = INIT_LOCAL_LOCK(lock),
};

void add_pending(struct item *it)
{
	struct my_percpu_state *s;

	local_lock(&mps.lock);            /* on !RT: preempt_disable() + lockdep tracking */
	s = this_cpu_ptr(&mps);
	list_add(&it->node, &s->pending);
	s->count++;
	local_unlock(&mps.lock);
}
```

Variants: `local_lock_irq()`, `local_lock_irqsave()`, `local_lock_nested_bh()`.

On `!PREEMPT_RT` it compiles to preemption/IRQ disabling plus lockdep bookkeeping — **zero
runtime cost, full static and dynamic checking**. On `PREEMPT_RT` it becomes a real per-CPU
sleeping lock, because RT cannot tolerate unbounded `preempt_disable()` regions (Ch. 28).

This is a beautiful piece of engineering to study: *the same source expresses two very
different runtime strategies because the **semantics** were made explicit.* Whenever you see
bare `preempt_disable()` in new code protecting per-CPU data, the reviewer's answer is
"use `local_lock`".

### T.8 The read-side cost inversion, and when per-CPU is wrong

Per-CPU trades a cheap write for an expensive read. Therefore it is **wrong** when:

- Reads are frequent relative to writes (you've just inverted the problem).
- You need an *exact*, *atomic* snapshot (e.g. a quota limit that must never be exceeded —
  though `percpu_counter_compare()` with a threshold handles the common case).
- `nr_cpus` is large and the read is on a latency-critical path (summing 512 counters touches
  512 cache lines).
- Memory is tight: per-CPU allocation costs `nr_cpus × size`. On a 256-CPU machine a
  per-CPU `struct` of 1 KiB costs 256 KiB. This is why `alloc_percpu()` uses its own chunked
  allocator rather than `kmalloc`.

**Hybrid answer for "must not exceed a limit":** keep per-CPU deltas but check a shared
approximate total against a *conservative* threshold; fall back to exact summation only when
near the limit. This is exactly what `percpu_counter_compare()` and memcg's
`page_counter` do. The pattern — *fast approximate path, slow exact path, switch near the
boundary* — recurs everywhere in the kernel.

### T.9 NUMA: partitioning in space as well as by CPU

On multi-socket machines, per-CPU allocation must also be **node-local**, or you've moved the
contention into the interconnect. Linux's per-CPU allocator allocates each CPU's chunk from
its own NUMA node, and `alloc_pages_node()` / `kmalloc_node()` let subsystems do the same for
their own structures.

The next level is **node-level** partitioning: one structure per node, shared by the cores on
that node. This is the right granularity when per-CPU is too memory-hungry but global is too
contended — e.g. SLUB's `kmem_cache_node`, the zone/`pgdat` structures, and CNA spinlocks
(Ch. 14 T.5). Recognizing "this wants to be per-node, not per-CPU" is a mature judgement.

### T.10 A hierarchy of scalability techniques

Arrange in order of preference. Reach for the highest one that works:

```
1. Don't share at all           — per-CPU / per-task / per-object state
2. Shard the sharing            — lock striping, hashed locks, per-node
3. Make readers not write       — RCU, seqlock
4. Make writes commutative      — atomics on disjoint lines, approximate counters
5. Amortize                     — batching, per-CPU queues flushed in bulk
6. Shorten the critical section — the last resort, and the least effective
7. Pick a better lock           — qspinlock is already there; this buys you almost nothing
```

Most engineers start at 7 and work up. Start at 1 and work down. When you present a
scalability fix in review, state which level you used and why the higher levels didn't apply.

---

## 1. Concept — the per-CPU API

### 1.1 Static per-CPU variables

```c
#include <linux/percpu.h>

DEFINE_PER_CPU(unsigned long, my_counter);
DEFINE_PER_CPU_SHARED_ALIGNED(struct stats, my_stats);   /* cacheline-aligned */
DEFINE_PER_CPU_ALIGNED(struct foo, my_foo);
DECLARE_PER_CPU(unsigned long, my_counter);              /* in a header */

this_cpu_inc(my_counter);
this_cpu_add(my_counter, 7);
this_cpu_write(my_counter, 0);
unsigned long v = this_cpu_read(my_counter);

/* Aggregate */
unsigned long total = 0;
int cpu;
for_each_possible_cpu(cpu)
	total += per_cpu(my_counter, cpu);
```

> Use `for_each_possible_cpu()`, **not** `for_each_online_cpu()`, when summing. A CPU that
> went offline still holds its accumulated count. This is a classic bug.

### 1.2 Dynamic per-CPU allocation

```c
struct ring __percpu *rings = alloc_percpu(struct ring);
if (!rings)
	return -ENOMEM;

for_each_possible_cpu(cpu)
	init_ring(per_cpu_ptr(rings, cpu));

/* fast path */
struct ring *r = get_cpu_ptr(rings);   /* preempt_disable + this_cpu_ptr */
ring_push(r, item);
put_cpu_ptr(rings);

free_percpu(rings);
```

`__percpu` is a `sparse` address-space annotation (`__attribute__((noderef, address_space(3)))`).
Dereferencing a `__percpu` pointer directly is a sparse error — you *must* go through
`per_cpu_ptr()`/`this_cpu_ptr()`. Run `make C=1` to check.

### 1.3 `percpu_counter`

```c
#include <linux/percpu_counter.h>

struct percpu_counter conns;

percpu_counter_init(&conns, 0, GFP_KERNEL);
percpu_counter_inc(&conns);
percpu_counter_add_batch(&conns, 1, 1024);        /* explicit batch */

s64 approx = percpu_counter_read_positive(&conns); /* cheap, approximate */
s64 exact  = percpu_counter_sum_positive(&conns);  /* expensive, accurate */
if (percpu_counter_compare(&conns, limit) >= 0)    /* smart: exact only when near */
	return -EMFILE;

percpu_counter_destroy(&conns);
```

### 1.4 CPU hotplug interaction

Per-CPU data must survive CPU offline/online. The modern API is the **state machine** in
`include/linux/cpuhotplug.h`:

```c
static int my_cpu_online(unsigned int cpu)  { /* init this CPU's state */ return 0; }
static int my_cpu_offline(unsigned int cpu) { /* flush/migrate its state */ return 0; }

ret = cpuhp_setup_state(CPUHP_AP_ONLINE_DYN, "my/driver:online",
			my_cpu_online, my_cpu_offline);
/* ret is the dynamically allocated state id; free with cpuhp_remove_state(ret) */
```
The callbacks run in a defined order relative to every other subsystem — that ordering is the
whole point of the enum in `cpuhotplug.h`. Picking the right state is a real design decision;
read the comments in that header.

---

## 2. Internals

### 2.1 How per-CPU addressing works

At build time, per-CPU variables live in a special section `.data..percpu` at **offset zero**.
At boot, `setup_per_cpu_areas()` allocates one copy of that whole section per CPU (node-local)
and records each CPU's base in `__per_cpu_offset[cpu]`.

```c
#define per_cpu_ptr(ptr, cpu)  ((typeof(ptr))((void *)(ptr) + __per_cpu_offset[cpu]))
```

On x86-64, the **current CPU's** offset is kept in the `%gs` base register, so:

```c
this_cpu_inc(my_counter);
/*  →  incq  %gs:my_counter(%rip)     — one instruction, no locking, no lookup */
```

On arm64 it's `TPIDR_EL1`; on RISC-V, `tp`. This is why `this_cpu_*` is so cheap and why the
API is written the way it is.

### 2.2 The per-CPU allocator

`mm/percpu.c` is a bespoke allocator. It cannot use `kmalloc` because it must allocate
`nr_cpus` copies at a **fixed relative offset** from each CPU's base. It manages "chunks"
(each covering a range of the per-CPU address space across all CPUs), with first-chunk
special-casing for early boot, and supports `alloc_percpu_gfp()` including atomic context
(with a pre-populated reserve, since populating page tables can sleep).

Read `mm/percpu.c`'s header comment — it is one of the better-documented files in the tree.

```bash
cat /proc/vmallocinfo | grep pcpu
sudo cat /sys/kernel/debug/percpu_stats     # CONFIG_PERCPU_STATS=y
```

### 2.3 Where to look in the tree

```
include/linux/percpu.h            alloc_percpu, per_cpu_ptr
include/linux/percpu-defs.h       DEFINE_PER_CPU, this_cpu_* generic
arch/x86/include/asm/percpu.h     the %gs implementation — read this
include/linux/local_lock.h        local_lock_t
include/linux/local_lock_internal.h
lib/percpu_counter.c
lib/percpu-refcount.c
mm/percpu.c                       ★ the allocator
include/linux/cpuhotplug.h        ★ the hotplug state enum
Documentation/core-api/this_cpu_ops.rst    ★ normative
Documentation/core-api/local_ops.rst
Documentation/core-api/cpu_hotplug.rst
```

---

## 3. Practice

### 3.1 A scalable statistics module

```c
// SPDX-License-Identifier: GPL-2.0
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/percpu.h>
#include <linux/local_lock.h>
#include <linux/debugfs.h>
#include <linux/cpu.h>

struct cpu_stats {
	local_lock_t   lock;
	u64            hits;
	u64            misses;
	u64            bytes;
} ____cacheline_aligned_in_smp;

static DEFINE_PER_CPU(struct cpu_stats, stats) = {
	.lock = INIT_LOCAL_LOCK(lock),
};

/* Fast path: no shared cache line, no atomics, no lock in the !RT build. */
void stats_hit(u64 bytes)
{
	struct cpu_stats *s;

	local_lock(&stats.lock);
	s = this_cpu_ptr(&stats);
	s->hits++;
	s->bytes += bytes;
	local_unlock(&stats.lock);
}
EXPORT_SYMBOL_GPL(stats_hit);

/* Single-counter update needs no lock at all — this_cpu_* is atomic per-CPU. */
void stats_miss(void)
{
	this_cpu_inc(stats.misses);
}
EXPORT_SYMBOL_GPL(stats_miss);

/* Slow path: O(nr_cpus), approximate (no global snapshot is possible). */
static int stats_show(struct seq_file *m, void *v)
{
	u64 hits = 0, misses = 0, bytes = 0;
	int cpu;

	for_each_possible_cpu(cpu) {
		struct cpu_stats *s = per_cpu_ptr(&stats, cpu);

		/* Reading another CPU's counters races with its updates.
		 * That is accepted: the result is approximate by design. */
		hits   += READ_ONCE(s->hits);
		misses += READ_ONCE(s->misses);
		bytes  += READ_ONCE(s->bytes);
	}
	seq_printf(m, "hits %llu\nmisses %llu\nbytes %llu\n", hits, misses, bytes);
	return 0;
}
DEFINE_SHOW_ATTRIBUTE(stats);

static struct dentry *dbg;

static int __init pcpu_demo_init(void)
{
	dbg = debugfs_create_file("pcpu_demo", 0444, NULL, NULL, &stats_fops);
	stats_hit(4096);
	stats_miss();
	return 0;
}

static void __exit pcpu_demo_exit(void)
{
	debugfs_remove(dbg);
}

module_init(pcpu_demo_init);
module_exit(pcpu_demo_exit);
MODULE_LICENSE("GPL");
```

Note the two update paths: `stats_hit()` needs `local_lock` because it updates **two** fields
that a reader might want consistent-ish and because it must be RT-safe; `stats_miss()` needs
nothing because a single `this_cpu_inc()` is already atomic against preemption and interrupts.
Knowing which you need is the skill.

### 3.2 Measure the difference

Write two variants of a counter — `atomic64_inc(&global)` and `this_cpu_inc(percpu)` — and
hammer them from `nr_cpus` kthreads:

```c
static int hammer(void *arg)
{
	u64 i;

	for (i = 0; i < 10000000 && !kthread_should_stop(); i++)
		this_cpu_inc(my_counter);       /* vs atomic64_inc(&global) */
	return 0;
}
```

Expected on an 8-core box: per-CPU stays flat at ~1 ns/op as you add threads; the atomic
degrades from ~10 ns to ~200+ ns/op. **Plot it.** This graph is the single most persuasive
artifact you can bring to a design discussion.

### 3.3 Hunt false sharing for real

```bash
sudo perf c2c record -F 60000 -a -- sleep 10
sudo perf c2c report --stdio | head -60
```
Look at the "Shared Data Cache Line Table": `Total records`, `LclHitm`/`RmtHitm` (local /
remote hit-modified). Any line with high HITM is a contention point; the report shows the
offsets within the line and the source lines touching them — which immediately tells you
*true* sharing (same offset) vs *false* sharing (different offsets).

```bash
pahole -C cpu_stats my_module.ko          # verify your alignment actually happened
pahole --hex -C task_struct vmlinux | head -40
```

---

## 3.4 Extended practice

### Lab 16.A — The scalability ladder, measured end to end (T.10)

One counter, seven implementations, one graph. This is the definitive experiment of Part 1.

```c
// SPDX-License-Identifier: GPL-2.0
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/kthread.h>
#include <linux/percpu.h>
#include <linux/percpu_counter.h>
#include <linux/local_lock.h>
#include <linux/spinlock.h>
#include <linux/cpu.h>

static int variant, nthreads = 4;
module_param(variant, int, 0644);
module_param(nthreads, int, 0644);

static DEFINE_SPINLOCK(lock);
static unsigned long guarded;
static atomic64_t atom;
static DEFINE_PER_CPU(unsigned long, pc);
static DEFINE_PER_CPU_ALIGNED(unsigned long, pc_aligned);
static struct percpu_counter pcc;
struct sharded { spinlock_t l; unsigned long v; } ____cacheline_aligned;
static struct sharded shards[64];

/* variant 6: deliberate FALSE SHARING, for contrast */
static struct { unsigned long v; } false_shared[64];

static atomic_t stop;
static atomic64_t total;

static int worker(void *arg)
{
	int id = (long)arg;
	u64 n = 0;

	while (!atomic_read(&stop)) {
		switch (variant) {
		case 0: spin_lock(&lock); guarded++; spin_unlock(&lock); break;
		case 1: atomic64_inc(&atom);                             break;
		case 2: this_cpu_inc(pc);                                break;
		case 3: this_cpu_inc(pc_aligned);                        break;
		case 4: percpu_counter_add(&pcc, 1);                     break;
		case 5: { int s = id & 63;
			  spin_lock(&shards[s].l); shards[s].v++;
			  spin_unlock(&shards[s].l); }                   break;
		case 6: false_shared[id & 63].v++;                       break;
		}
		if (!(++n & 0x3ff))
			cond_resched();
	}
	atomic64_add(n, &total);
	return 0;
}

static const char *names[] = { "spinlock", "atomic64", "percpu", "percpu_aligned",
			       "percpu_counter", "sharded", "false_shared" };

static int __init scale_init(void)
{
	struct task_struct **t;
	u64 t0;
	int i;

	for (i = 0; i < 64; i++)
		spin_lock_init(&shards[i].l);
	percpu_counter_init(&pcc, 0, GFP_KERNEL);

	t = kcalloc(nthreads, sizeof(*t), GFP_KERNEL);
	t0 = ktime_get_ns();
	for (i = 0; i < nthreads; i++) {
		t[i] = kthread_create(worker, (void *)(long)i, "scale/%d", i);
		kthread_bind(t[i], i % num_online_cpus());
		wake_up_process(t[i]);
	}
	msleep(3000);
	atomic_set(&stop, 1);
	for (i = 0; i < nthreads; i++)
		kthread_stop(t[i]);

	pr_info("%-16s threads=%2d -> %6llu Mops/s\n", names[variant], nthreads,
		atomic64_read(&total) * 1000 / (ktime_get_ns() - t0));
	percpu_counter_destroy(&pcc);
	kfree(t);
	return 0;
}
static void __exit scale_exit(void) { }
module_init(scale_init); module_exit(scale_exit);
MODULE_LICENSE("GPL");
```
```bash
for v in 0 1 2 3 4 5 6; do for n in 1 2 4 8 16 $(nproc); do
  sudo insmod scale.ko variant=$v nthreads=$n; sudo rmmod scale
done; done
dmesg | grep -E 'spinlock|atomic64|percpu|sharded|false_shared'
```
**What you should see, and must be able to explain:**
- `percpu` / `percpu_aligned`: **linear** in thread count.
- `sharded`: near-linear until shards collide.
- `percpu_counter`: just below per-CPU (the periodic fold costs a little).
- `atomic64` and `spinlock`: peak at 2–4 threads, then **decline** — the $\kappa N^2$ term.
- `false_shared`: **worse than the atomic**, despite having no synchronization at all.
  That last result is the one that teaches T.2.

### Lab 16.B — Find false sharing with `perf c2c` (T.2)

```bash
sudo perf c2c record -F 60000 -a -- sh -c 'sudo insmod scale.ko variant=6 nthreads=8; sudo rmmod scale'
sudo perf c2c report --stdio | head -80
```
Read the **"Shared Data Cache Line Table"**. Key columns:
- `Total records`, `LclHitm` / `RmtHitm` — local/remote hit-modified. **HITM is the signal.**
- The per-offset breakdown below each line: *different* offsets being written by different
  CPUs = **false** sharing; the *same* offset = **true** sharing.

Now do it on the real system:
```bash
sudo perf c2c record -a -- sleep 10           # while running your real workload
sudo perf c2c report --stdio -c tid,iaddr | head -60
```
Then verify your fix with `pahole`:
```bash
pahole -C sharded scale.ko
pahole --hex -C sk_buff vmlinux | head -50
pahole -C task_struct --reorganize vmlinux | tail -20
```

### Lab 16.C — `local_lock` and the `preempt_disable()` conversion (T.7)

```c
struct region {
	local_lock_t     lock;
	struct list_head pending;
	int              count;
	u64              bytes;
};
static DEFINE_PER_CPU(struct region, regions) = {
	.lock = INIT_LOCAL_LOCK(lock),
};

/* BEFORE: an undocumented, uncheckable convention */
static void old_add(struct item *it)
{
	struct region *r;

	preempt_disable();              /* protects... what, exactly? */
	r = this_cpu_ptr(&regions);
	list_add(&it->node, &r->pending);
	r->count++;
	preempt_enable();
}

/* AFTER: the protected data is named, and lockdep checks it */
static void new_add(struct item *it)
{
	struct region *r;

	local_lock(&regions.lock);
	r = this_cpu_ptr(&regions);
	list_add(&it->node, &r->pending);
	r->count++;
	local_unlock(&regions.lock);
}

/* Now deliberately violate it and watch lockdep catch what it never could before: */
static void violate(void)
{
	struct region *r = this_cpu_ptr(&regions);   /* no local_lock held! */

	lockdep_assert_held(&this_cpu_ptr(&regions)->lock);   /* fires */
	r->count++;
}
```
```bash
./scripts/config -e PROVE_LOCKING -e DEBUG_PREEMPT -e PREEMPT
# and then the RT comparison:
git log --oneline --grep='local_lock' | head -30
$EDITOR include/linux/local_lock_internal.h   # see the !RT vs RT definitions side by side
```

### Lab 16.D — Per-CPU across CPU hotplug (§1.4)

```c
static int my_online(unsigned int cpu)
{
	pr_info("cpu%u online: initializing state\n", cpu);
	*per_cpu_ptr(&mycount, cpu) = 0;
	return 0;
}
static int my_offline(unsigned int cpu)
{
	/* Fold this CPU's state somewhere durable BEFORE it goes away. */
	atomic64_add(*per_cpu_ptr(&mycount, cpu), &grand_total);
	*per_cpu_ptr(&mycount, cpu) = 0;
	pr_info("cpu%u offline: folded\n", cpu);
	return 0;
}

state = cpuhp_setup_state(CPUHP_AP_ONLINE_DYN, "mydrv:online", my_online, my_offline);
...
cpuhp_remove_state(state);
```
```bash
# Hammer it:
sudo insmod hp.ko
while true; do
  for c in 2 3; do
    echo 0 | sudo tee /sys/devices/system/cpu/cpu$c/online >/dev/null
    echo 1 | sudo tee /sys/devices/system/cpu/cpu$c/online >/dev/null
  done
done &
# ...under load. Verify no counts are lost and nothing crashes.
cat /sys/devices/system/cpu/hotplug/states | head -40
```
**The classic bug this catches:** summing with `for_each_online_cpu()` instead of
`for_each_possible_cpu()`. Introduce it deliberately and watch counts vanish.

### Lab 16.E — `percpu_counter`: quantify the error/cost trade (T.5)

```c
static void batch_sweep(void)
{
	int batches[] = { 1, 8, 32, 128, 1024 };
	int i;

	for (i = 0; i < ARRAY_SIZE(batches); i++) {
		/* hammer percpu_counter_add_batch(&pcc, 1, batches[i]) from N cpus */
		/* then compare: */
		s64 approx = percpu_counter_read(&pcc);
		s64 exact  = percpu_counter_sum(&pcc);

		pr_info("batch=%4d  approx=%lld exact=%lld error=%lld (bound=%d)\n",
			batches[i], approx, exact, exact - approx,
			batches[i] * num_possible_cpus());
	}
}
```
Confirm the error stays within `nr_cpus × batch`, and plot throughput vs batch. Then find
the real users and the thresholds they chose:
```bash
git grep -n 'percpu_counter_add_batch\|percpu_counter_compare' fs/ mm/ net/ | head -20
$EDITOR lib/percpu_counter.c   # percpu_counter_batch auto-tuning
```

### Lab 16.F — Read the per-CPU allocator and the `%gs` trick (§2.1–2.2)

```bash
# See the single-instruction access:
cat > /tmp/pc.c <<'EOF'
#include <linux/percpu.h>
DEFINE_PER_CPU(unsigned long, demo);
void bump(void) { this_cpu_inc(demo); }
EOF
make M=/tmp pc.o && objdump -d /tmp/pc.o | grep -A4 '<bump>:'
# expect on x86-64:   incq %gs:0x0(%rip)     — ONE instruction, no lock prefix

# The offsets table:
sudo grep -E '__per_cpu_offset|__per_cpu_start|__per_cpu_end' /proc/kallsyms
$EDITOR arch/x86/include/asm/percpu.h        # the %gs implementation
$EDITOR mm/percpu.c                          # read the 100-line header comment

# Allocator statistics:
./scripts/config -e PERCPU_STATS
sudo cat /sys/kernel/debug/percpu_stats
grep pcpu /proc/vmallocinfo | head
```

### Lab 16.G — Design review: the hierarchical quota problem

Implement drill 8 for real. Requirements: a global limit of 1,000,000 concurrent objects,
**enforced exactly** (never exceeded), on a many-core machine at millions of ops/sec.

Sketch:
```c
struct budget {
	atomic64_t       global_remaining;   /* the shared, exact authority */
	int __percpu    *local_reserve;      /* pre-claimed tokens, per CPU */
};

static bool budget_get(struct budget *b)
{
	int *res;

	preempt_disable();
	res = this_cpu_ptr(b->local_reserve);
	if (*res > 0) { (*res)--; preempt_enable(); return true; }  /* fast path: no sharing */
	preempt_enable();
	return budget_refill_slow(b);        /* slow path: one atomic per BATCH acquisitions */
}
```
Implement `budget_refill_slow()` (claim `BATCH` from the global counter atomically; if fewer
remain, claim what's left; if zero, fail) and a drain-on-offline hotplug callback. Then
benchmark against a plain `atomic64_dec_if_positive()` and show both correctness (never
exceeds the limit) and scalability. Finally compare your design with
`percpu_counter_compare()` and memcg's `page_counter` + `memcg_stock`.

---

## 4. Mastery drills

1. **Read the Commutativity Rule paper** (SOSP 2013). Then find one Linux interface you
   believe is unscalable *by specification* and explain why. Propose a commutative variant.

2. **Layout audit.** Run `pahole` on `struct task_struct`, `struct sock`, and `struct
   net_device`. Find the cacheline-alignment annotations and explain what each is protecting
   against. Find a hole and explain why it's there (or isn't worth fixing).

3. **`for_each_possible_cpu` bug.** `git grep -n "for_each_online_cpu" -- lib/ kernel/` and
   find a summation loop. Determine whether it's a bug. (Several real fixes exist; find one
   with `git log --grep="possible_cpu"`.)

4. **Convert a global counter.** Pick a driver with an `atomic_t` statistics counter.
   Convert it to per-CPU. Measure. Write the commit message including the measurement.

5. **`local_lock` conversion.** `git log --grep="local_lock" --oneline | head -40`.
   Read three conversion patches. Explain in each case what convention was being made
   explicit and what RT problem it solved.

6. **Per-CPU allocator.** Read `mm/percpu.c`'s top comment and `pcpu_alloc()`. Explain why a
   separate allocator is required and how the "first chunk" bootstrap works.

7. **Hotplug correctness.** Write a module with per-CPU state and a `cpuhp` callback pair.
   Offline and online a CPU (`echo 0 > /sys/devices/system/cpu/cpu3/online`) while it's under
   load. Verify no counts are lost and no crash occurs.

8. **Design question.** You need a global limit of 1,000,000 concurrent connections, enforced
   exactly, on a 256-CPU machine handling 5M connections/sec. Design it. (Expected answer:
   per-CPU reservation pools with batch refill from a shared budget, exact check only when a
   CPU's local reservation is exhausted — i.e. hierarchical token allocation. Compare with
   `percpu_counter_compare()`.)

---

## 5. Further reading

**Kernel docs:**
- `Documentation/core-api/this_cpu_ops.rst` — normative, short
- `Documentation/core-api/local_ops.rst`
- `Documentation/core-api/cpu_hotplug.rst`
- `Documentation/core-api/padata.rst` — parallel work distribution built on these ideas
- `mm/percpu.c` header comment

**Papers:**
- Clements, Kaashoek, Zeldovich, Morris, Kohler, "The Scalable Commutativity Rule"
  (SOSP 2013) — **read this one; it will change how you design APIs**
- Boyd-Wickizer et al., "An Analysis of Linux Scalability to Many Cores" (OSDI 2010) — the
  paper that drove a decade of Linux scalability work
- Gunther, *Guerrilla Capacity Planning* — the USL (see Ch. 14)
- Drepper, "What Every Programmer Should Know About Memory" (2007) — **the** cache primer

**Talks / articles:**
- LWN: "Per-CPU variables and the realtime tree", "The seqcount latch lock type",
  "local_lock and PREEMPT_RT", "Cache-line contention and perf c2c"
- Joe Mario, "C2C — False Sharing Detection in Linux Perf" (the `perf c2c` author's writeup)

→ Next: [17-interrupts.md](17-interrupts.md)

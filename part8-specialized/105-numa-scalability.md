# Chapter 105 — NUMA and Large-System Scalability

> The final chapter of Part 8, and the one that retroactively explains a great deal of what
> came before. Per-CPU data (Ch. 16), RCU (Ch. 15), `sbitmap` and per-CPU software queues
> (Ch. 63), the folio/memdesc work (Ch. 52), `percpu_rwsem`, `local_lock` — these are not
> unrelated optimizations. They are all answers to the same question: **how do you build a
> shared-memory system where nothing is actually shared?**

---

## Theory & First Principles

### T.0 — Start here: memory is not one thing

```bash
numactl --hardware
# node distances:
#       0   1
#   0: 10  21     <- accessing node 1's memory from node 0 costs ~2.1x
#   1: 21  10
```

**Every mental model you have used so far assumed memory is uniform. It is not, and has not
been since roughly 2003.**

```
   +------------------+          +------------------+
   |     SOCKET 0     |  UPI /   |     SOCKET 1     |
   |  cores 0-31      |<-------->|  cores 32-63     |
   |  L3 cache        |  Infinity|  L3 cache        |
   |  memory ctrl ----|--+       |  memory ctrl ----|--+
   +------------------+  |       +------------------+  |
                      [DRAM 0]                      [DRAM 1]
                      ~80 ns                        ~80 ns local
                                                    ~140 ns REMOTE from socket 0
```

**A thread on socket 0 touching memory on node 1 pays ~1.8× the latency and consumes
interconnect bandwidth that is shared with everything else.** A process whose threads and
memory are scattered at random runs at roughly 60% of the speed of the identical process
placed correctly — **with no code change whatsoever.**

**So the kernel must answer two questions it did not have to ask in 1995:**

```
   WHERE does this page live?     -> first-touch policy: the page is allocated on
                                     the node of the CPU that first WRITES it.
                                     Not the one that malloc'd it -- the one that
                                     touched it.  (Parallel initialization matters!)

   WHERE does this thread run?    -> the scheduler balances load, which may move a
                                     thread AWAY from its memory.

   ...and these two decisions fight each other.
```

**AutoNUMA is the kernel's attempt to reconcile them:** it periodically unmaps pages to force
faults, observes which node actually touches them, and then either migrates the page to the
thread or the thread to the page. **It is a feedback controller, and like all feedback
controllers it can oscillate** — which is why on a carefully-placed workload (a database that
already pins itself) `numa_balancing=0` is often faster.

**Now the deeper point, which is the right note to end the book on.** NUMA is one instance of
a general law about parallel systems. **Adding cores does not add throughput linearly, and
past a point it *subtracts*:**

```
   Universal Scalability Law (Gunther):

                        N
     C(N) = --------------------------------
            1 + a(N-1) + b*N*(N-1)
                 ^            ^
                 |            |  CROSSTALK / COHERENCE
                 |            |  cost grows as N^2 -- this is why
                 |            |  throughput eventually FALLS
                 |
                 |  CONTENTION (serialization -- Amdahl's law)
```

**Amdahl's law says you plateau. The USL says you *retrograde*.** The `b` term — the cost of
keeping N caches coherent about shared data — is the one that bites, and it is exactly Ch. 16
§T.0's counter that gets slower with more CPUs, given a formula.

**And now look back at Part 3, Part 4, and Part 5 with that formula in hand.** Every one of
these was an attack on the `b` term:

| | How it reduces crosstalk |
|---|---|
| Per-CPU counters (Ch. 16) | no shared cacheline at all |
| RCU (Ch. 15) | readers write nothing |
| RCU-walk path resolution (Ch. 54) | no refcount writes on the hot path |
| XFS allocation groups (Ch. 58) | independent allocation domains |
| `blk-mq` (Ch. 63) | per-CPU software queues |
| NVMe queue pairs (Ch. 69) | one queue and one interrupt vector per CPU |
| `io_uring` (Ch. 76) | shared ring instead of a shared lock |

**Seven subsystems, one equation.** That is the final compression of this book: **the central
problem of modern systems software is shared mutable state, and essentially every advanced
technique you have learned is a different way of not having any.**

```bash
numactl --hardware && numastat -m
sudo perf stat -e node-load-misses,node-store-misses -a -- sleep 10
sudo perf c2c record -a -- sleep 10 && sudo perf c2c report   # FALSE SHARING, located
cat /proc/sys/kernel/numa_balancing
numactl --cpunodebind=0 --membind=0 ./app     # measure the difference yourself
```

---

### T.1 — The hardware you are actually programming

The mental model of "the CPU accesses memory" has been wrong since roughly 2003. What you
have is:

```
 Socket 0                              Socket 1
 ┌──────────────────────────┐          ┌──────────────────────────┐
 │ core core core core ...  │          │ core core core core ...  │
 │   L1   L1   L1   L1      │          │   L1   L1   L1   L1      │
 │   L2   L2   L2   L2      │          │   L2   L2   L2   L2      │
 │ ├──── shared L3 ───────┤ │          │ ├──── shared L3 ───────┤ │
 │      memory controller   │◄────────►│      memory controller   │
 └──────────┬───────────────┘   UPI/   └──────────┬───────────────┘
            │                  Infinity            │
       ┌────▼────┐              Fabric        ┌────▼────┐
       │ DRAM n0 │                            │ DRAM n1 │
       └─────────┘                            └─────────┘
```

| Access | Latency | Bandwidth |
|---|---|---|
| L1 hit | ~1 ns (4 cycles) | enormous |
| L2 hit | ~4 ns | |
| L3 hit (local socket) | ~15–20 ns | |
| **Local DRAM** | **~80–100 ns** | full |
| **Remote DRAM (1 hop)** | **~130–200 ns** | ~60% of local |
| Remote DRAM (2 hops, 4+ sockets) | ~250–350 ns | lower still |
| **Cacheline transfer from another socket's cache (HITM)** | **~200–400 ns** | — |
| CXL-attached memory | ~250–400 ns | lower |

Two derived facts that drive everything:

1. **The NUMA ratio** (remote/local latency) is typically 1.5–2.2×. That is the number
   `numactl --hardware` reports in its distance matrix (10 = local, 21 = one hop, by
   convention).
2. **A contended cacheline is more expensive than a remote memory access.** A HITM — one
   core writing a line that another socket's cache owns modified — costs 200–400 ns *and*
   invalidates the other core's copy, so the cost recurs. This is why **contention, not
   latency, is the scalability limit.**

Also note: modern large sockets are internally NUMA (Sub-NUMA Clustering on Intel, NPS on
AMD EPYC), so a single-socket machine can have 4+ NUMA nodes. "One socket, so no NUMA" is
wrong on current hardware.

### T.2 — The Universal Scalability Law

Amdahl's law bounds speedup by the serial fraction:

$$S(N) = \frac{1}{(1-p) + p/N}$$

It says you asymptote. It does **not** say you can get *worse*. Real systems do get worse,
and the Universal Scalability Law (Gunther) explains why:

$$C(N) = \frac{N}{1 + \alpha(N-1) + \beta N(N-1)}$$

| Term | Name | Physical meaning |
|---|---|---|
| $\alpha$ | **Contention** | Serialization — waiting for a lock. Amdahl's term |
| $\beta$ | **Coherency** | Crosstalk — the cost of keeping caches consistent |

The $\beta N^2$ term means throughput **peaks and then declines**. Adding cores past the peak
makes the system slower in absolute terms. This is not theoretical: it is exactly what
happens when a hot cacheline is written by every core, because the invalidation traffic
grows quadratically.

**The engineering consequence:** shortening a critical section reduces $\alpha$ but does
nothing for $\beta$. Past a few dozen cores, $\beta$ dominates, and the only fixes are ones
that *eliminate sharing* — per-CPU data, RCU, sharding. That is why the ladder in §T.6 has
the shape it does.

$$N_{peak} = \sqrt{\frac{1-\alpha}{\beta}}$$

Measure $\alpha$ and $\beta$ by fitting a throughput-vs-cores curve, and you can *predict*
the core count at which your system will stop improving — a genuinely useful thing to do
before buying hardware.

### T.3 — Cache coherence is the hidden cost model

MESI (and its MOESI/MESIF variants) is the protocol, and the state transitions are the price
list:

| State | Meaning |
|---|---|
| **M**odified | This cache has the only copy, and it is dirty |
| **E**xclusive | Only copy, clean |
| **S**hared | Multiple caches have a clean copy |
| **I**nvalid | Not present |

The operations that cost:

| Pattern | Cost |
|---|---|
| Read a line nobody has | DRAM latency |
| Read a line in S elsewhere | cheap — sharing reads is nearly free |
| **Write a line in S elsewhere** | **invalidate all sharers — broadcast + wait** |
| **Write a line in M elsewhere** | **HITM: fetch from the other cache + invalidate** |
| Atomic RMW on a shared line | acquire exclusive ownership: the above, every time |

**The single most important consequence:** *read* sharing scales; *write* sharing does not.
A million cores can read the same line concurrently at full speed. Two cores alternately
writing it will run at a fraction of one core's speed.

This is the entire justification for RCU. An `rwlock` has readers *write* to the lock word to
take a reference — so "read-only" workload still has write sharing on the lock, and read
throughput **decreases** as cores are added. RCU readers write nothing at all, so read
throughput is linear. Being able to state that in exactly these terms is the difference
between knowing what RCU is and knowing why it exists. → Ch. 15.

### T.4 — False sharing, the bug with no symptoms

```c
/* Looks fine. Is a disaster. */
struct stats {
	unsigned long rx_packets;   /* written by CPU 0's NAPI */
	unsigned long tx_packets;   /* written by CPU 1's xmit */
};                              /* both in ONE 64-byte cacheline */
```

There is no logical sharing, no race, no correctness problem — and the line ping-pongs
between the two cores on every increment. Throughput can be 10× lower than the
padded version.

```c
struct stats {
	unsigned long rx_packets ____cacheline_aligned_in_smp;
	unsigned long tx_packets ____cacheline_aligned_in_smp;
};
```

Detection is the hard part, because the code looks correct and the profiler shows time in an
innocent-looking increment. **`perf c2c`** is the tool: it uses precise load/store sampling
to attribute HITM events to specific cachelines and specific *offsets within* them, and
tells you which functions and which CPUs collided.

Two adjacent traps:
- **True sharing** presented as false sharing: sometimes the fix is not padding but removing
  the shared counter entirely (`percpu_counter`).
- **Padding is not free** — it costs memory and cache footprint. Padding every field in a
  hot struct can be a net loss. Measure.
- **Cacheline size varies**: 64 bytes on most x86 and ARM, 128 on Apple silicon and some
  POWER, and x86 prefetches adjacent line pairs, which makes the *effective* unit 128 bytes
  for some patterns. Hence `L1_CACHE_BYTES` and `SMP_CACHE_BYTES` rather than a literal.

### T.5 — NUMA policy: where memory comes from

Linux's default is **first-touch**: a page is allocated on the node of the CPU that first
*writes* it, not the one that `malloc`'d it. This is usually right and occasionally
catastrophic — a common failure is an initialization thread touching a large array, placing
all of it on one node, after which every worker on other nodes makes remote accesses
forever.

The policies (`set_mempolicy`, `mbind`, `numactl`):

| Policy | Behaviour | Use |
|---|---|---|
| `MPOL_DEFAULT` | First touch, local node | Most things |
| `MPOL_BIND` | Strictly from a node set; **OOM rather than go remote** | Hard partitioning |
| `MPOL_PREFERRED` | Prefer a node, fall back | Softer version of BIND |
| `MPOL_PREFERRED_MANY` | Prefer a *set*, fall back | Tiered memory (DRAM set, CXL fallback) |
| `MPOL_INTERLEAVE` | Round-robin across nodes | **Bandwidth-bound** workloads; trades latency for aggregate bandwidth |
| `MPOL_WEIGHTED_INTERLEAVE` | Interleave proportional to node bandwidth | Heterogeneous memory (CXL) |

**The interleave insight**, which is counterintuitive and interview-worthy: if a workload is
*bandwidth*-bound rather than *latency*-bound, interleaving across all nodes is faster than
local allocation, because you get the sum of all memory controllers' bandwidth. A big
in-memory analytics scan is the classic case. Local-first is a latency optimization, and
applying it to a bandwidth problem is a mistake.

**Automatic NUMA balancing** (`kernel.numa_balancing=1`): periodically unmaps pages to
induce a "NUMA hinting fault", observes which node actually touches them, and migrates pages
(or the task). It helps unaware workloads and *hurts* carefully-tuned ones by fighting your
explicit placement and by adding fault overhead. For a tuned workload, turn it off; for a
general-purpose fleet, leave it on. Knowing that this is a decision, not a default, is the
point.

### T.6 — The scalability ladder

The escalation, with what each step attacks:

| Step | Technique | Attacks | Ceiling |
|---|---|---|---|
| 0 | **Measure** (`perf c2c`, `lock_stat`) | — | — |
| 1 | Shrink the critical section | $\alpha$ | 2–4× |
| 2 | Split the lock (per-bucket, per-object) | $\alpha$ | tens of cores |
| 3 | Reader/writer split (`rwsem`, `seqlock`) | $\alpha$, only if reads dominate | limited — readers still write the lock |
| 4 | **RCU** | $\beta$ — readers write nothing | hundreds, read-mostly |
| 5 | **Per-CPU + fold on read** | $\beta$ | near-linear, if reads are rare |
| 6 | Shard by NUMA node, hierarchical rollup | $\beta$ across sockets | thousands |
| 7 | **Eliminate the shared state** | both | the only true answer |

Steps 1–3 are $\alpha$ work and stop paying off around a few dozen cores. Steps 4–7 are
$\beta$ work. The mistake that defines a mid-level engineer is spending all the effort on
steps 1–3 and concluding that "the lock is already short, so this must be inherent."

**Per-CPU counters** deserve elaboration because they are the workhorse:

```c
struct percpu_counter c;
percpu_counter_add(&c, 1);           /* per-CPU increment, batched      */
sum = percpu_counter_sum(&c);        /* EXPENSIVE: walks every CPU      */
approx = percpu_counter_read(&c);    /* cheap, approximate              */
```

The design trade is explicit: writes are $O(1)$ and contention-free; reads are $O(N_{cpus})$
and only approximate unless you pay for the full sum. **That is the right trade whenever
writes outnumber reads**, which is true of essentially every statistic. When it is not true —
when you read frequently — you need something else, and recognizing that is the skill.

Related primitives worth naming: `local_lock_t` (per-CPU protection that is explicit and
RT-correct → Ch. 16, Ch. 103), `percpu_rwsem` (readers are per-CPU and nearly free; writers
are extremely expensive — correct for "almost never written" global state like CPU hotplug),
`sbitmap` (per-CPU-cached bitmap allocation, used for blk-mq tags → Ch. 63), and
`per-CPU page lists (PCP)` in the page allocator.

### T.7 — Where the kernel itself stops scaling

Known bottlenecks at high core counts, each with its own story:

| Bottleneck | Symptom | State of the art |
|---|---|---|
| **`mmap_lock`** | Multithreaded processes serializing on page faults and `mmap` | Per-VMA locks (6.4+) let faults proceed without the mmap lock — one of the biggest recent wins |
| **`i_rwsem`** | Concurrent writes to one file serialize | XFS allows concurrent `O_DIRECT` writes; buffered writes still serialize |
| **Page allocator zone lock** | High allocation rates | Per-CPU page lists (PCP), per-node lists |
| **`dcache` / `d_lock`** | Path lookup on shared directories | RCU-walk (lockless path walk), then REF-walk fallback |
| **TLB shootdown IPIs** | Unexplained jitter on unrelated cores | `mmu_gather` batching, PCID/ASID, ARM64 broadcast TLBI |
| **Timer/RCU callbacks** | Per-CPU kthread interference | `rcu_nocbs` offload |
| **`stop_machine`** | Global stall | Avoided; still used for module load, hotplug |
| **`inode_hash_lock`, `sb_lock`** | Metadata-heavy workloads | Progressively split |
| **Futex hash buckets** | Many threads on few futexes | Larger hash, per-process futex hash (recent work) |
| **Network `qdisc` lock** | Single TX queue | Multiqueue NICs, XPS, `NOLOCK` qdiscs |

The pattern in every row: the fix is never "make the lock faster." It is always "make the
thing not shared" — per-CPU, per-node, per-object, or RCU.

### T.8 — Cache and memory-bandwidth isolation (RDT / MPAM)

The resource that has no cgroup controller and that everyone forgets: **last-level cache and
memory bandwidth are completely unisolated by default.** A noisy neighbour can evict your
entire working set from L3 and consume all the memory bandwidth, and neither `cpu.max` nor
`memory.max` does anything about it.

Intel RDT (Resource Director Technology) / ARM MPAM, exposed via `resctrl`:

| Feature | Does |
|---|---|
| **CMT** (Cache Monitoring) | Measure LLC occupancy per group |
| **MBM** (Memory Bandwidth Monitoring) | Measure local and total bandwidth per group |
| **CAT** (Cache Allocation) | *Partition* the LLC by way-mask per group |
| **MBA** (Memory Bandwidth Allocation) | Throttle a group's bandwidth |

```bash
mount -t resctrl resctrl /sys/fs/resctrl
mkdir /sys/fs/resctrl/critical
echo "L3:0=0xff0;1=0xff0" > /sys/fs/resctrl/critical/schemata   # reserve 8 ways
echo $PID > /sys/fs/resctrl/critical/tasks
cat /sys/fs/resctrl/critical/mon_data/mon_L3_00/llc_occupancy
```

For any latency-sensitive workload sharing a socket, this is often the highest-impact
control available and is very rarely used. Mentioning it in a multi-tenancy design
discussion is a distinctive signal.

### T.9 — Memory tiering and CXL

The newest dimension. CXL-attached memory sits at ~250–400 ns — between DRAM and NVMe by a
wide margin, and close enough to DRAM to be used *as memory* rather than as storage.

Linux models it as a **CPU-less NUMA node**, which is an elegant reuse: all the existing NUMA
machinery (distance matrices, memory policy, migration) applies immediately.

The new machinery:
- **`MPOL_WEIGHTED_INTERLEAVE`** — interleave proportional to each node's bandwidth, so a
  slower tier gets proportionally less traffic.
- **Demotion/promotion** — reclaim on a fast node *demotes* cold pages to the slow tier
  instead of swapping; NUMA hinting faults *promote* hot pages back.
  (`/sys/kernel/mm/numa/demotion_enabled`.)
- **DAMON** — access-frequency monitoring with far lower overhead than page-table scanning,
  plus `DAMOS` to act on it (e.g. "demote regions colder than X").

The architectural point: **the memory hierarchy has grown a new level, and the OS's job is
to place pages by temperature.** This is the working-set model (→ `reference/os-fundamentals.md`
§4) applied at a new boundary, and the same problem the page cache solves for disk.

### T.10 — The discipline, stated as principles

1. **Measure before you theorize.** At scale the bottleneck is usually two or three specific
   cachelines, not a diffuse problem. `perf c2c` finds them in minutes.
2. **Read sharing is free; write sharing is not.** Every design decision follows from this.
3. **Contention ($\beta$), not latency, sets the ceiling.** A shorter critical section on a
   shared line does not save you.
4. **Locality is a first-class design property**, not a tuning step. Data should live near
   the CPU that owns it, and *ownership* should be a design decision.
5. **Partition, don't share.** Per-CPU, per-node, per-object. The best lock is the one that
   is never contended because only one CPU can reach it.
6. **Fast paths must not write shared state** — not even statistics. That is what
   `percpu_counter` is for.
7. **Batch to amortize.** One expensive synchronization per N operations beats N cheap ones,
   because the expensive one is $O(N_{cpus})$ and the cheap ones are $O(N \cdot N_{cpus})$.
8. **Know your topology and pin accordingly** — but remember pinning is brittle; a design
   that requires perfect placement will degrade badly when placement fails.

---

## 1. Internals

### Source map

| Path | Contents |
|---|---|
| `mm/mempolicy.c` | NUMA memory policy: `set_mempolicy`, `mbind`, interleave |
| `mm/migrate.c` | Page migration, including `move_pages()` and demotion |
| `mm/numa_balancing.c`, `kernel/sched/fair.c` | Automatic NUMA balancing, task and page placement |
| `mm/page_alloc.c` | Zonelists, node fallback order, per-CPU page lists |
| `mm/memory-tiers.c` | Tiered memory, demotion targets |
| `mm/damon/` | DAMON access monitoring and DAMOS schemes |
| `lib/percpu_counter.c` | Batched per-CPU counters |
| `include/linux/percpu.h`, `mm/percpu.c` | Per-CPU allocator |
| `include/linux/local_lock.h` | `local_lock_t` |
| `kernel/locking/percpu-rwsem.c` | Per-CPU rwsem |
| `lib/sbitmap.c` | Scalable bitmap with per-CPU caching (blk-mq tags) |
| `kernel/locking/osq_lock.c` | MCS/OSQ queued spinlock — the per-CPU-node spin queue |
| `kernel/locking/qspinlock.c` | The queued spinlock: **spinners wait on their own cacheline** |
| `arch/x86/kernel/cpu/resctrl/` | RDT/CAT/MBA |
| `drivers/base/node.c` | NUMA node sysfs representation |
| `Documentation/admin-guide/mm/numa_memory_policy.rst` | Policy semantics |
| `Documentation/mm/numa.rst` | The model |

### `qspinlock`: why a spinlock is not a spin on a shared word

A naive test-and-set spinlock has every waiter hammering the same cacheline — pure $\beta$,
$O(N^2)$ coherence traffic, and throughput that *decreases* with core count. The MCS queued
lock fixes this:

```
 lock word: [ tail_cpu | tail_idx | pending | locked ]
                    │
                    ▼
    per-CPU node ──► per-CPU node ──► per-CPU node
     (CPU 3)          (CPU 7)          (CPU 12)
     spins on         spins on         spins on
     its OWN line     its OWN line     its OWN line
```

Each waiter enqueues a per-CPU node and **spins on a flag in its own node** — its own
cacheline, which no one else writes until it is that waiter's turn. The unlocker writes
exactly one remote line: the next waiter's flag.

Result: $O(1)$ coherence traffic per handoff instead of $O(N)$, and FIFO fairness as a
bonus. Linux's `qspinlock` is a hybrid — it has fast paths for the uncontended and
single-waiter cases (the `pending` bit) before falling back to the full MCS queue, because
the queue setup is wasted work when there is no contention.

**This is the best single example of the chapter's thesis:** the fix for a scalability
problem was not a faster lock, it was restructuring so that waiters stop sharing a line.

### Per-CPU counter internals

```c
struct percpu_counter {
	raw_spinlock_t	lock;
	s64		count;        /* the global, approximate total   */
	s32 __percpu	*counters;    /* per-CPU deltas                  */
};

void percpu_counter_add_batch(struct percpu_counter *fbc, s64 amount, s32 batch)
{
	s64 count = __this_cpu_read(*fbc->counters) + amount;

	if (abs(count) >= batch) {
		/* Rare: fold into the global under the lock. */
		raw_spin_lock(&fbc->lock);
		fbc->count += count;
		__this_cpu_write(*fbc->counters, 0);
		raw_spin_unlock(&fbc->lock);
	} else {
		/* Common: purely local, no shared line touched at all. */
		__this_cpu_write(*fbc->counters, count);
	}
}
```

The `batch` parameter is the tunable trade between accuracy and contention: larger batch
means fewer global folds (less contention) but larger error in `percpu_counter_read()`. The
default scales with CPU count, because the total error is `batch × nr_cpus`.

### Observability surface

```bash
# Topology: nodes, distances, per-node memory
numactl --hardware
lscpu | grep -i numa
cat /sys/devices/system/node/node*/distance
cat /sys/devices/system/node/node*/meminfo | grep -E 'MemTotal|MemFree'

# Where is a process's memory, actually?
numastat -p $(pidof myapp)
cat /proc/$(pidof myapp)/numa_maps | head
cat /proc/$(pidof myapp)/status | grep -i numa

# Remote vs local hits, system-wide:
numastat            # numa_hit, numa_miss, numa_foreign, other_node

# THE tool for contention: which cacheline, which offset, which function.
sudo perf c2c record -a -- sleep 10
sudo perf c2c report --stdio

# Remote memory access rate via PMU:
sudo perf stat -e node-loads,node-load-misses,node-stores,node-store-misses -a sleep 10
# Intel uncore / offcore:
sudo perf stat -e offcore_response.all_data_rd.l3_miss.remote_dram -a sleep 10

# Lock contention:
echo 1 | sudo tee /proc/sys/kernel/lock_stat   # needs CONFIG_LOCK_STAT
sudo cat /proc/lock_stat | head -40
sudo perf lock record -a -- sleep 10 && sudo perf lock contention

# NUMA balancing activity:
grep -E 'numa_pages_migrated|pgmigrate|numa_hint' /proc/vmstat
sysctl kernel.numa_balancing

# Cache/bandwidth monitoring:
sudo mount -t resctrl resctrl /sys/fs/resctrl 2>/dev/null
cat /sys/fs/resctrl/info/L3/num_closids /sys/fs/resctrl/info/L3/cbm_mask
```

---

## 2. Practice

### Lab 105.1 — Measure your machine's NUMA reality

```c
/* numa_latency.c — measure local vs remote memory latency directly.
 * Build: gcc -O2 -o numa_latency numa_latency.c -lnuma
 * Run:   ./numa_latency
 *
 * Uses a pointer-chasing loop, which defeats prefetching and hardware
 * parallelism, so it measures true dependent-load latency.            */
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <sched.h>
#include <numa.h>
#include <numaif.h>

#define BUF_MB   256
#define STRIDE   64                       /* one cacheline */
#define NCHASE   (BUF_MB * 1024 * 1024 / STRIDE)

static double chase(void **buf, long iters)
{
	struct timespec t0, t1;
	void **p = buf;

	/* warm up */
	for (long i = 0; i < NCHASE; i++)
		p = (void **)*p;

	clock_gettime(CLOCK_MONOTONIC, &t0);
	for (long i = 0; i < iters; i++)
		p = (void **)*p;
	clock_gettime(CLOCK_MONOTONIC, &t1);

	/* keep the compiler honest */
	if (p == (void **)0x1) printf("");

	return ((t1.tv_sec - t0.tv_sec) * 1e9 +
		(t1.tv_nsec - t0.tv_nsec)) / iters;
}

static void **make_chain(int node)
{
	size_t sz = (size_t)BUF_MB * 1024 * 1024;
	void **buf = numa_alloc_onnode(sz, node);
	long n = sz / STRIDE;
	long *idx;

	if (!buf) { perror("numa_alloc_onnode"); exit(1); }
	memset(buf, 0, sz);

	/* Build a random permutation so the chase is unpredictable. */
	idx = malloc(n * sizeof(long));
	for (long i = 0; i < n; i++) idx[i] = i;
	for (long i = n - 1; i > 0; i--) {
		long j = random() % (i + 1);
		long t = idx[i]; idx[i] = idx[j]; idx[j] = t;
	}
	for (long i = 0; i < n; i++)
		*(void **)((char *)buf + idx[i] * STRIDE) =
			(char *)buf + idx[(i + 1) % n] * STRIDE;
	free(idx);
	return buf;
}

int main(void)
{
	int nnodes;

	if (numa_available() < 0) {
		fprintf(stderr, "NUMA not available on this system\n");
		return 1;
	}
	nnodes = numa_max_node() + 1;
	printf("NUMA nodes: %d\n\n", nnodes);
	printf("%-8s %-8s %12s %8s\n", "cpu@node", "mem@node", "ns/access", "ratio");

	double local_ref = 0;
	for (int cnode = 0; cnode < nnodes; cnode++) {
		struct bitmask *cpus = numa_allocate_cpumask();
		numa_node_to_cpus(cnode, cpus);
		/* Pin to the first CPU of cnode. */
		for (int c = 0; c < numa_num_configured_cpus(); c++) {
			if (numa_bitmask_isbitset(cpus, c)) {
				cpu_set_t s; CPU_ZERO(&s); CPU_SET(c, &s);
				sched_setaffinity(0, sizeof(s), &s);
				break;
			}
		}
		numa_free_cpumask(cpus);

		for (int mnode = 0; mnode < nnodes; mnode++) {
			void **buf = make_chain(mnode);
			double ns = chase(buf, 20L * 1000 * 1000);
			if (cnode == 0 && mnode == 0) local_ref = ns;
			printf("%-8d %-8d %12.1f %8.2fx\n",
			       cnode, mnode, ns, ns / local_ref);
			numa_free(buf, (size_t)BUF_MB * 1024 * 1024);
		}
	}

	printf("\nKernel's distance matrix (10 = local):\n");
	system("cat /sys/devices/system/node/node*/distance");
	return 0;
}
```

Compare your measured ratio against the ACPI SLIT distance matrix the kernel reports. They
often disagree, because SLIT values are firmware-declared and sometimes wrong. **Trust your
measurement.**

### Lab 105.2 — Reproduce and fix false sharing

```c
/* false_sharing.c — the same work, three layouts, wildly different speed.
 * Build: gcc -O2 -pthread -o false_sharing false_sharing.c
 * Run:   ./false_sharing 8                                             */
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <time.h>
#include <sched.h>

#define ITERS (200L * 1000 * 1000)
#define CL    64

static int nthreads;

/* (a) Adjacent longs -- all in one or two cachelines. Disaster. */
static volatile long shared_adjacent[64];

/* (b) Padded to a cacheline each. */
struct padded { volatile long v; char pad[CL - sizeof(long)]; }
	__attribute__((aligned(CL)));
static struct padded shared_padded[64];

/* (c) Thread-local, summed at the end. The real answer. */
static long results[64];

struct arg { int id; int mode; };

static void *worker(void *p)
{
	struct arg *a = p;
	cpu_set_t s;

	CPU_ZERO(&s); CPU_SET(a->id, &s);
	sched_setaffinity(0, sizeof(s), &s);

	switch (a->mode) {
	case 0:
		for (long i = 0; i < ITERS; i++)
			shared_adjacent[a->id]++;
		break;
	case 1:
		for (long i = 0; i < ITERS; i++)
			shared_padded[a->id].v++;
		break;
	default: {
		long local = 0;                 /* lives in a register */
		for (long i = 0; i < ITERS; i++)
			local++;
		results[a->id] = local;
		break;
	}
	}
	return NULL;
}

static double run(int mode)
{
	pthread_t t[64];
	struct arg a[64];
	struct timespec t0, t1;

	clock_gettime(CLOCK_MONOTONIC, &t0);
	for (int i = 0; i < nthreads; i++) {
		a[i].id = i; a[i].mode = mode;
		pthread_create(&t[i], NULL, worker, &a[i]);
	}
	for (int i = 0; i < nthreads; i++)
		pthread_join(t[i], NULL);
	clock_gettime(CLOCK_MONOTONIC, &t1);

	return (t1.tv_sec - t0.tv_sec) + (t1.tv_nsec - t0.tv_nsec) / 1e9;
}

int main(int argc, char **argv)
{
	double a, b, c;
	nthreads = (argc > 1) ? atoi(argv[1]) : 4;
	if (nthreads > 64) nthreads = 64;

	a = run(0); b = run(1); c = run(2);
	printf("threads = %d, %ld increments each\n\n", nthreads, ITERS);
	printf("  adjacent (false sharing) : %7.3f s   (1.00x)\n", a);
	printf("  cacheline padded         : %7.3f s   (%.2fx faster)\n", b, a / b);
	printf("  thread-local             : %7.3f s   (%.2fx faster)\n", c, a / c);
	return 0;
}
```

```bash
./false_sharing 1   # no sharing possible -> all three similar
./false_sharing 2
./false_sharing 8   # the gap opens dramatically
./false_sharing $(nproc)

# Now PROVE it is false sharing, do not just infer it:
sudo perf c2c record -- ./false_sharing 8
sudo perf c2c report --stdio | head -60
#   -> look for "Shared Data Cache Line Table": it names the line,
#      the offset, the reading and writing CPUs, and the source line.
```

The essential observation: case (a) can get *slower* as you add threads — a direct
demonstration of the $\beta N^2$ term from §T.2. Case (c) is perfectly linear. Same
arithmetic, different memory layout.

### Lab 105.3 — First-touch and the initialization trap

```c
/* first_touch.c — demonstrate the most common NUMA bug in real software.
 * Build: gcc -O2 -fopenmp -o first_touch first_touch.c -lnuma           */
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <omp.h>
#include <numa.h>

#define N (400L * 1000 * 1000)     /* ~3.2 GB of doubles */

static double bench(double *a, double *b, const char *label)
{
	struct timespec t0, t1;
	double sum = 0;

	clock_gettime(CLOCK_MONOTONIC, &t0);
	#pragma omp parallel for reduction(+:sum)
	for (long i = 0; i < N; i++)
		sum += a[i] * b[i];
	clock_gettime(CLOCK_MONOTONIC, &t1);

	double s = (t1.tv_sec - t0.tv_sec) + (t1.tv_nsec - t0.tv_nsec) / 1e9;
	printf("%-28s %6.3f s   %6.1f GB/s   (sum=%.3e)\n",
	       label, s, 2.0 * N * sizeof(double) / s / 1e9, sum);
	return s;
}

int main(void)
{
	double *a1, *b1, *a2, *b2;
	double bad, good;

	printf("threads=%d nodes=%d\n\n", omp_get_max_threads(),
	       numa_max_node() + 1);

	/* --- WRONG: the master thread touches everything first, so every
	 *     page lands on the master's node. All other threads then make
	 *     remote accesses for the entire life of the program.        */
	a1 = malloc(N * sizeof(double));
	b1 = malloc(N * sizeof(double));
	for (long i = 0; i < N; i++) { a1[i] = 1.0; b1[i] = 2.0; }

	/* --- RIGHT: initialize in parallel with the SAME access pattern
	 *     used later, so first-touch places each page on the node of
	 *     the thread that will actually use it.                     */
	a2 = malloc(N * sizeof(double));
	b2 = malloc(N * sizeof(double));
	#pragma omp parallel for
	for (long i = 0; i < N; i++) { a2[i] = 1.0; b2[i] = 2.0; }

	bad  = bench(a1, b1, "serial init (all one node)");
	good = bench(a2, b2, "parallel init (first-touch)");
	printf("\nspeedup from correct placement: %.2fx\n", bad / good);
	printf("\nverify with: numastat -p %d\n", getpid());
	return 0;
}
```

```bash
gcc -O2 -fopenmp -o first_touch first_touch.c -lnuma
./first_touch

# Confirm the placement difference directly:
numactl --hardware
numactl --interleave=all ./first_touch    # interleaving fixes the bad case too
numactl --cpunodebind=0 --membind=0 ./first_touch   # single node: no gap
```

On a 2-socket machine expect 1.5–2× from correct placement alone, with **no algorithmic
change**. Note also that `--interleave=all` rescues the badly-initialized case — because the
workload is bandwidth-bound, which is §T.5's counterintuitive point made concrete.

### Lab 105.4 — Find the contended cacheline in a real workload

```bash
# perf c2c is the single most valuable tool in this chapter.
sudo perf c2c record -F 60000 -a -- <your workload or sleep 10>
sudo perf c2c report --stdio > c2c.txt

# Read it in this order:
#  1. "Shared Data Cache Line Table" -- ranked by HITM count.
#     Columns: Total records, LLC Load Hitm (Local/Remote), Store Refs,
#              Data address, Node, PID, and the symbol.
#  2. For the top line, the per-offset breakdown: which BYTES within the
#     64-byte line are being touched, by which functions, on which CPUs.
#     Two different offsets touched by two different CPUs == FALSE sharing.
#     The same offset touched by both == TRUE sharing.
#  3. That distinction determines the fix: padding vs. redesign.

# Cross-check with the raw event on Intel:
sudo perf stat -e mem_load_l3_hit_retired.xsnp_hitm,\
mem_load_l3_miss_retired.remote_hitm -a sleep 10

# On AMD:
sudo perf c2c record -a -- sleep 10   # uses IBS
```

Apply it to something real: a kernel build, `redis-benchmark`, `pgbench`, or the false-sharing
lab above. **Practise reading the output until the local-vs-remote HITM distinction is
automatic**, because that is the entire diagnostic.

### Lab 105.5 — Scaling curve and USL fit

```bash
#!/bin/bash
# usl.sh — measure a scaling curve and fit alpha and beta.
BENCH=${1:-"hackbench -l 20000 -g 10"}
MAX=$(nproc)

echo "cores,seconds,throughput,speedup"
BASE=""
for n in 1 2 4 8 16 24 32 48 64; do
	[ "$n" -gt "$MAX" ] && break
	CPUS=$(seq -s, 0 $((n-1)))
	T=$( { /usr/bin/time -f "%e" taskset -c "$CPUS" $BENCH >/dev/null; } 2>&1 )
	TP=$(echo "scale=4; 1/$T" | bc)
	[ -z "$BASE" ] && BASE=$TP
	SU=$(echo "scale=3; $TP/$BASE" | bc)
	echo "$n,$T,$TP,$SU"
done
```

Then fit $C(N) = N / (1 + \alpha(N-1) + \beta N(N-1))$ to the throughput column (any
nonlinear least-squares tool; `scipy.optimize.curve_fit` is two lines). Report $\alpha$,
$\beta$, and the predicted $N_{peak} = \sqrt{(1-\alpha)/\beta}$.

The deliverable that matters: **a prediction.** "This workload peaks at 38 cores; buying a
64-core machine will make it slower." Then verify. Being able to produce that statement from
a measurement is a genuinely senior capability and almost nobody does it.

### Lab 105.6 — Cache partitioning with resctrl

```bash
#!/bin/bash
# resctrl_demo.sh — protect a latency-sensitive workload's LLC share.
set -e
sudo mount -t resctrl resctrl /sys/fs/resctrl 2>/dev/null || true
R=/sys/fs/resctrl

echo "LLC ways available: $(cat $R/info/L3/cbm_mask)  (bitmask)"
echo "Monitoring features: $(cat $R/info/L3_MON/mon_features 2>/dev/null)"

# Two groups: 'critical' gets 8 exclusive ways, 'noisy' gets the rest.
sudo mkdir -p $R/critical $R/noisy
echo "L3:0=0x00ff" | sudo tee $R/critical/schemata   # low 8 ways
echo "L3:0=0xff00" | sudo tee $R/noisy/schemata      # high 8 ways

# Baseline: run the latency-sensitive workload alone.
echo "--- alone ---"
./latency_sensitive_bench

# Contended, WITHOUT partitioning: put both in the default group.
echo "--- with noisy neighbour, no partitioning ---"
stress-ng --cache 8 --cache-level 3 -t 60s &
NOISY=$!
./latency_sensitive_bench
kill $NOISY

# Contended, WITH partitioning.
echo "--- with noisy neighbour, LLC partitioned ---"
stress-ng --cache 8 --cache-level 3 -t 60s &
NOISY=$!
echo $NOISY | sudo tee $R/noisy/tasks
echo $$     | sudo tee $R/critical/tasks
./latency_sensitive_bench
kill $NOISY

# Observe occupancy and bandwidth per group:
cat $R/critical/mon_data/mon_L3_00/llc_occupancy
cat $R/critical/mon_data/mon_L3_00/mbm_local_bytes
```

The result to internalize: a cache-thrashing neighbour can degrade a co-located workload by
2–5× with no CPU or memory-cgroup violation whatsoever, and `resctrl` is the only control
that addresses it. **This is the isolation gap that almost every multi-tenancy design
misses.**

---

## 3. Mastery drills

1. Build a complete performance profile of your largest available machine: NUMA latency
   matrix (measured, not SLIT), local and remote bandwidth per node, LLC size and
   associativity, cacheline size, and the cost of a HITM. Publish it as a one-page reference
   and use it to sanity-check every subsequent measurement.

2. Take a real kernel subsystem (the page allocator, the dcache, or blk-mq) and trace its
   scalability history through git: find the commits that changed a global lock to per-CPU
   or RCU, read the cover letters, and reproduce the benchmark each claimed.

3. Write a benchmark that exhibits the USL $\beta$ term — throughput that *decreases* with
   added cores. Fit $\alpha$ and $\beta$. Then fix it and show $\beta$ approaching zero.

4. Instrument a multithreaded application of your choice with `perf c2c`. Find its top three
   contended cachelines, classify each as true or false sharing, fix them, and report the
   throughput change per fix.

5. Implement a NUMA-aware hash table: per-node shards, node-local allocation, and a
   migration policy for entries accessed predominantly from a remote node. Compare against a
   naive global hash table at 1, 2, and 4 nodes.

6. Measure the cost of `percpu_counter_sum()` versus `percpu_counter_read()` as a function of
   core count. Determine the read frequency at which per-CPU counters become the *wrong*
   choice, and propose what you would use instead.

7. Compare `qspinlock` against a naive test-and-set spinlock (write one in a module) at 2,
   8, 32, and all cores. Plot throughput and explain the shapes using the MESI state
   transitions from §T.3.

8. Take a bandwidth-bound workload and measure it under `--localalloc`, `--interleave=all`,
   and `--membind` to a single node. Explain the ordering and identify the workload
   characteristic that determines which wins.

9. Enable and disable automatic NUMA balancing on a workload with explicit `numactl`
   placement. Measure the interference. Write the rule you would put in an operations
   runbook.

10. Set up a tiered-memory system (or emulate one with a CPU-less NUMA node via
    `memmap=` / a slow QEMU node). Configure demotion and promotion, use DAMON to observe
    page temperature, and measure the effect on a workload whose working set exceeds the
    fast tier.

11. Use `resctrl` to quantify the cost of LLC contention for three different workloads
    (latency-sensitive, bandwidth-bound, compute-bound). Produce the table showing which
    workloads need partitioning and which do not.

12. Profile TLB shootdown activity on a large multithreaded process
    (`perf stat -e tlb_flush.*` plus the `tlb:tlb_flush` tracepoint). Correlate IPI rate
    with `munmap`/`madvise` rate, and estimate the latency inflicted on unrelated cores.

13. Design the scalability architecture for a subsystem that must handle 100M operations/sec
    across 512 cores on 8 NUMA nodes. Specify the data layout, the synchronization strategy
    at each level, the statistics approach, and the memory placement policy — and state the
    measurement that would tell you the design is failing.

---

## 4. Further reading

**Kernel documentation**
- `Documentation/mm/numa.rst` — the model
- `Documentation/admin-guide/mm/numa_memory_policy.rst` — policy semantics in detail
- `Documentation/admin-guide/mm/damon/` — DAMON and DAMOS
- `Documentation/arch/x86/resctrl.rst` — RDT/CAT/MBA, with worked examples
- `Documentation/locking/locktypes.rst` — which lock type for which situation
- `Documentation/core-api/local_ops.rst`, `this_cpu_ops.rst` — per-CPU primitives
- `Documentation/scheduler/sched-domains.rst` — how the scheduler models topology

**Papers**
- Boyd-Wickizer et al., "An Analysis of Linux Scalability to Many Cores," *OSDI*, 2010 —
  **the** paper on this topic; identifies the bottlenecks and fixes them one by one
- Clements et al., "The Scalable Commutativity Rule," *SOSP*, 2013 — a profound result:
  *whenever interface operations commute, they can be implemented to scale*. This turns
  scalability into an **interface design** property, not an implementation property. Read
  this one if you read only one
- Mellor-Crummey & Scott, "Algorithms for Scalable Synchronization on Shared-Memory
  Multiprocessors," *TOCS*, 1991 — the MCS lock
- Gunther, *Guerrilla Capacity Planning* — the Universal Scalability Law
- Lameter, "NUMA (Non-Uniform Memory Access): An Overview," *ACM Queue*, 2013
- David, Guerraoui, Trigonakis, "Everything You Always Wanted to Know About
  Synchronization but Were Afraid to Ask," *SOSP*, 2013 — an empirical survey across
  hardware; excellent data
- Dashti et al., "Traffic Management: A Holistic Approach to Memory Placement on NUMA
  Systems," *ASPLOS*, 2013 — why congestion, not just latency, matters
- Gouicem et al., "Application-Informed Kernel Synchronization Primitives," *OSDI*, 2022

**LWN**
- "Per-VMA locks" series — the `mmap_lock` scalability fix
- "A new API for scatter/gather" and the folio/memdesc series (memory footprint at scale)
- "Memory tiering" and the CXL coverage
- "DAMON" series
- "The 'queued' spinlock" and Waiman Long's scalability work
- Paul McKenney's RCU articles, particularly on RCU's scalability properties

**Books**
- Gunther, *Guerrilla Capacity Planning* — USL, and how to fit it
- McKenney, *Is Parallel Programming Hard, And, If So, What Can You Do About It?* — free,
  and the best treatment of the counting, partitioning, and deferred-processing chapters
  in particular. Chapter 5 (counting) is directly this chapter's material
- Herlihy & Shavit, *The Art of Multiprocessor Programming*, 2nd ed. — the theory
- Drepper, "What Every Programmer Should Know About Memory" (2007) — long, free, and still
  the best explanation of cache and NUMA hardware behaviour

**Tools**
- `perf c2c` — cacheline contention attribution. The essential one
- `numactl`, `numastat`, `migratepages`, `numademo`
- `likwid` — topology-aware performance counters and pinning (`likwid-topology`,
  `likwid-perfctr`, `likwid-bench`)
- `Intel MLC` (Memory Latency Checker) — authoritative latency/bandwidth matrices
- `stream` — the standard memory bandwidth benchmark
- `lstopo` / `hwloc` — visualize the topology, including cache sharing
- `pcm` (Intel Processor Counter Monitor) — UPI traffic, memory controller utilization
- `resctrl` / `pqos` — cache and bandwidth monitoring and allocation

---

## Part 8 completion checkpoint

You have now covered the four cross-cutting domains. Before moving on, confirm you can:

- [ ] State a threat model and derive which mitigations follow from it, including their cost
- [ ] Explain the exploitation pipeline and which mitigation breaks each stage
- [ ] Write a seccomp filter, and say why it cannot dereference pointers
- [ ] Define real time as *bounded*, decompose a latency budget into its five terms, and name
      the tool that attributes each
- [ ] Explain what `PREEMPT_RT` changes and what it deliberately does not
- [ ] Describe priority inversion and the two protocols that bound it
- [ ] Draw the KVM vCPU run loop and identify which exits stay in the kernel and why
- [ ] State the Popek–Goldberg criterion and why x86 failed it
- [ ] Explain virtio, vhost, and vDPA as three points on one curve
- [ ] Place containers, gVisor, microVMs, and VMs on the isolation/overhead spectrum
- [ ] Explain why read sharing scales and write sharing does not, in MESI terms
- [ ] Use `perf c2c` to find a contended cacheline and classify it as true or false sharing
- [ ] Describe the seven-step scalability ladder and which term ($\alpha$ or $\beta$) each
      step attacks
- [ ] Name the isolation gap that cgroups do not cover, and the mechanism that does

→ Next: [../part4-net/76-io-uring.md](../part4-net/76-io-uring.md)

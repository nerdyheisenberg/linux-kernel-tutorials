# OS Fundamentals — the rapid reference this curriculum assumes

> **Why this exists.** The 106 chapters teach Linux. They assume you already have the
> classic operating-systems theory that a CS degree provides — and interviews ask that
> theory *directly*, often as a warm-up, often verbatim from Tanenbaum or Silberschatz.
> Fumbling "what are the four conditions for deadlock" after an hour of brilliant RCU
> discussion is disproportionately damaging, because it reads as "learned Linux by
> osmosis, never studied systems."
>
> This is a dense reference, not a tutorial. Every section ends with **→ Linux** showing
> where the abstraction actually lives in the tree, and which chapter derives it.

---

## 1. Processes, threads, and context

### The classic definitions

| Term | Definition |
|---|---|
| **Program** | A passive file on disk |
| **Process** | A program in execution: address space + execution context + resources |
| **Thread** | An execution context sharing an address space with its peers |
| **Context switch** | Saving one execution context and restoring another |
| **Process control block (PCB)** | The kernel's per-process record |

**Process states** (the canonical five-state model):

```
        admitted            dispatch           exit
  NEW ───────────► READY ──────────► RUNNING ──────────► TERMINATED
                     ▲                 │
                     │  I/O complete   │ I/O request / wait
                     └──── WAITING ◄───┘
                            (blocked)
```

Linux adds: `TASK_RUNNING` covers both READY and RUNNING (distinguished by whether it is
`rq->curr`), plus `TASK_INTERRUPTIBLE`, `TASK_UNINTERRUPTIBLE`, `TASK_KILLABLE`,
`TASK_STOPPED`, `TASK_TRACED`, `EXIT_ZOMBIE`, `EXIT_DEAD`.

### Context switch cost, and why it is the number that matters

| Component | Typical cost |
|---|---|
| Direct: save/restore registers, `switch_to` | 1–2 µs |
| Page-table switch (`switch_mm`, CR3 write) | +0.5–1 µs, or ~0 with PCID |
| **Indirect: cache and TLB pollution** | **5–50 µs** of degraded execution afterward |

The indirect cost dominates and is the reason for almost every design decision that avoids
switching: spinlocks over mutexes for short holds, softirqs over threads, NAPI polling,
`io_uring`'s batching, thread pools over thread-per-request.

**→ Linux:** `kernel/sched/core.c:context_switch()`. Ch. 20, Ch. 21.

### Thread models

| Model | Mapping | Notes |
|---|---|---|
| **N:1** (user-level) | many user threads : 1 kernel thread | Fast switches; one blocking syscall blocks all; no SMP |
| **1:1** (kernel-level) | 1:1 | **Linux's model.** Every thread is a schedulable entity |
| **M:N** (hybrid) | many:many | Solaris, early FreeBSD, Go's runtime. Complex; largely abandoned in kernels |

Linux's 1:1 choice follows from §T.1 of Ch. 20: there is only `task_struct`, and "thread"
vs "process" is a matter of which `clone()` flags shared which resources.

**→ Linux:** Ch. 20 §T.1.

---

## 2. Scheduling theory

### The metrics (and their conflicts)

| Metric | Definition | Favours |
|---|---|---|
| **Throughput** | Jobs completed per unit time | Long timeslices, batch |
| **Turnaround time** | Completion − arrival | SJF |
| **Waiting time** | Time in ready queue | SJF |
| **Response time** | First response − arrival | Short timeslices, RR |
| **Fairness** | Proportional share | Fair-share schedulers |
| **Predictability / jitter** | Variance in the above | RT schedulers |

Throughput and response time are in direct conflict: every context switch buys response
time and costs throughput.

### The classic algorithms

| Algorithm | Preemptive | Optimal for | Fatal flaw |
|---|---|---|---|
| **FCFS / FIFO** | No | Nothing | Convoy effect: one long job blocks everything |
| **SJF** (shortest job first) | No | Minimizes *average* waiting time — provably | Requires knowing job length; starves long jobs |
| **SRTF** (shortest remaining time) | Yes | Same, preemptive | Same |
| **Round robin** | Yes | Response time | Quantum too small → switch overhead; too large → FCFS |
| **Priority** | Either | Meeting explicit importance | **Starvation** — fixed by aging |
| **MLFQ** (multi-level feedback queue) | Yes | Approximating SJF *without knowing job length* | Gameable; needs periodic boost |
| **Fair-share / proportional** | Yes | Guaranteed shares | Says nothing about *when* within the period |
| **Lottery / stride** | Yes | Proportional share, simple | No latency guarantee |
| **EDF** (earliest deadline first) | Yes | **Optimal** on uniprocessor: schedulable iff U ≤ 1 | Domino failure on overload |
| **RM** (rate monotonic) | Yes | Optimal *fixed-priority* assignment | Bound is only U ≤ n(2^(1/n) − 1) → ln 2 ≈ 0.693 |

**MLFQ is worth understanding deeply**, because it is the answer to "how do you approximate
SJF when you cannot see the future?" The rules (Arpaci-Dusseau's formulation):

1. If priority(A) > priority(B), A runs.
2. If equal, round-robin.
3. A new job enters at the **highest** priority (assume it is short).
4. If a job uses its entire time allotment, its priority **drops** (it is CPU-bound).
5. After period S, **boost everything** back to the top (prevents starvation and gaming).

Rule 4 originally said "if it yields before the quantum, keep priority" — which was gameable
by yielding at 99% of the quantum. Fixing that required accounting for *total allotment*,
not per-quantum behaviour. This is a nice small example of a mechanism being gamed and
repaired.

### Real-time schedulability

For periodic tasks with period $T_i$ and worst-case execution time $C_i$, utilization is

$$U = \sum_{i=1}^{n} \frac{C_i}{T_i}$$

| Scheme | Schedulable if |
|---|---|
| **RM** (fixed priority) | $U \le n(2^{1/n}-1)$ — sufficient, not necessary |
| **RM** exact | Response-time analysis: $R_i = C_i + \sum_{j \in hp(i)} \lceil R_i/T_j \rceil C_j$ |
| **EDF** (dynamic) | $U \le 1$ — necessary *and* sufficient |

**Why anyone uses RM given EDF is optimal:** EDF's failure mode on overload is a domino
effect — one overrun cascades into cascading misses with no predictable victim. RM degrades
predictably (low-priority tasks miss first). Linux's `SCHED_DEADLINE` gets both by pairing
EDF with the **Constant Bandwidth Server**, which enforces each task's budget so an overrun
cannot damage others.

**→ Linux:** Ch. 21 §T.2 (proportional share, lag), §T.4 (EEVDF), §T.6 (RM/EDF/CBS).

### Priority inversion

```
   High-priority H needs lock L
   Low-priority  L1 holds lock L
   Medium-priority M preempts L1 (it can — M > L1)
   => H waits on M, indirectly. Inversion is UNBOUNDED.
```

The Mars Pathfinder failure (1997) was exactly this, and the fix uplinked to Mars was to
enable priority inheritance.

| Protocol | Mechanism | Bound |
|---|---|---|
| **Priority inheritance (PI)** | Holder temporarily inherits the highest waiter's priority | Bounded by the length of the critical sections |
| **Priority ceiling (PCP)** | Each lock has a ceiling = max priority of any user; holder runs at ceiling | Bounds inversion *and* prevents deadlock |
| **Immediate ceiling (ICPP)** | Raise to ceiling on acquire, unconditionally | Simplest; used in Ada, POSIX `PTHREAD_PRIO_PROTECT` |

**→ Linux:** `rt_mutex` with full transitive PI chain walking; `FUTEX_LOCK_PI` for
userspace. Ch. 14 §T.8, Ch. 103.

---

## 3. Concurrency and synchronization

### Race conditions and critical sections

A **race condition** exists when the result depends on the timing of uncontrolled events.
A **critical section** is code that accesses shared state and must not interleave.

Requirements for a correct critical-section solution (Dijkstra):

1. **Mutual exclusion** — at most one thread inside.
2. **Progress** — if nobody is inside, selection of who enters cannot be postponed
   indefinitely.
3. **Bounded waiting** — a bound on how many others may enter before you do.

Peterson's algorithm satisfies all three for two threads using only loads and stores —
**and it is broken on any real CPU without memory barriers**, which is a superb interview
follow-up. (It requires sequential consistency; x86 and ARM both reorder store-then-load.)

### The four Coffman conditions for deadlock

Deadlock requires **all four** simultaneously (Coffman, Elphick, Shoshani, 1971):

| # | Condition | Break it by |
|---|---|---|
| 1 | **Mutual exclusion** — resource is non-shareable | Make it shareable (rare; e.g. read-only) |
| 2 | **Hold and wait** — holds one, requests another | Acquire all at once, or release before requesting |
| 3 | **No preemption** — cannot be forcibly taken | Allow rollback / `trylock` with backoff |
| 4 | **Circular wait** — a cycle in the wait-for graph | **Impose a global lock order** ← what Linux does |

Linux's answer is #4, enforced by `lockdep`, which builds the lock-order graph at runtime
and reports a cycle the *first* time an inverted order is observed — even if no deadlock
actually occurred. That is the key insight: lockdep finds the *possibility*, not the event.

**The three strategies:**

| Strategy | Method | Used by |
|---|---|---|
| **Prevention** | Structurally break one of the four | Linux (lock ordering) |
| **Avoidance** | Only grant if the resulting state is safe (banker's algorithm) | Essentially nobody — needs max-claims in advance |
| **Detection and recovery** | Let it happen, detect cycles, kill/rollback | Databases |

**Banker's algorithm** (Dijkstra) — know it exists and why it is impractical: it requires
each process to declare its maximum resource claim up front, then grants a request only if
the resulting state is *safe* (there exists some completion order). The reason no OS uses
it: processes cannot predict their maximum claims, and the check is O(n²m).

**Livelock** — threads are running but making no progress (e.g. two threads politely
backing off in lockstep). **Starvation** — a thread never gets the resource, though others
progress. Neither is deadlock.

**→ Linux:** Ch. 14 §T.7 (lockdep), Ch. 25 (patterns), `Documentation/locking/`.

### The classic problems

Know these three by name; interviewers use them as shared vocabulary.

**Producer–consumer (bounded buffer).** Needs mutual exclusion on the buffer *plus*
counting: producers wait when full, consumers when empty. The classic semaphore solution:

```c
sem_t empty = N, full = 0;  mutex_t m;

producer: wait(empty); lock(m); insert(); unlock(m); post(full);
consumer: wait(full);  lock(m); remove(); unlock(m); post(empty);
```

**The order matters:** acquiring `m` before `empty` deadlocks. This is a good small test of
whether someone has actually thought about it.
→ Linux: `kfifo` + wait queues; Ch. 10 §T.5, Ch. 25 P6.

**Readers–writers.** Many readers *or* one writer. Three variants, and the choice is a
policy decision:
- *Readers-preference* → writers can starve.
- *Writers-preference* → readers can starve.
- *Fair* (queue-based) → neither.

→ Linux: `rw_semaphore` (writer-preferring, to avoid writer starvation), `seqlock`
(writer-preferring and wait-free for writers), RCU (readers never block, ever). Ch. 14, 15.

**Dining philosophers** (Dijkstra). Five philosophers, five forks, each needs two.
Demonstrates deadlock (all grab left simultaneously) and starvation. Solutions: resource
ordering (one philosopher grabs right first — breaks the cycle, Coffman #4), an arbiter, or
limiting concurrency to N−1.
→ The resource-ordering solution is exactly Linux's lock-ordering discipline.

### Synchronization primitives compared

| Primitive | Blocks? | Counting? | Ownership | Linux |
|---|---|---|---|---|
| **Spinlock** | Busy-waits | No | Implicit | `spinlock_t` |
| **Mutex** | Sleeps | No | **Yes** — only the owner may unlock | `struct mutex` |
| **Semaphore** | Sleeps | **Yes** | No — anyone may post | `struct semaphore` (legacy) |
| **Condition variable** | Sleeps | No | Paired with a mutex | wait queues |
| **Monitor** | — | — | Language-level mutex + condvars | — |
| **Barrier** | Sleeps | — | All-must-arrive | `completion`, `rcu_barrier` |
| **RW lock** | Either | — | Shared/exclusive | `rwlock_t`, `rw_semaphore` |

**Mutex vs binary semaphore** is a classic question. The answer: a mutex has an **owner**,
which enables priority inheritance, deadlock detection, and the rule that only the locker
may unlock. A semaphore is a signalling counter and may be posted by a different thread —
which is why `completion` (a signalling primitive) is separate from `mutex` in Linux.

**→ Linux:** Ch. 14 (all primitives), Ch. 25 (which to choose).

---

## 4. Memory management

### Address translation

| Scheme | Unit | Internal frag | External frag | Notes |
|---|---|---|---|---|
| **Contiguous allocation** | variable | low | **high** | Needs compaction |
| **Segmentation** | logical segments | low | high | Matches program structure; x86 legacy |
| **Paging** | fixed pages | ≤ 1 page/region | **none** | The universal answer |
| **Segmented paging** | both | | | x86-32's actual model |

**Why paging won:** fixed-size units eliminate external fragmentation entirely, which turns
allocation into a free-list pop rather than a best-fit search.

**Multi-level page tables** exist because a flat table for 64-bit addresses is impossible:
2⁵² entries × 8 bytes = 32 PiB *per process*. A radix trie makes the cost proportional to
what is actually mapped. → Ch. 22 §T.2.

**Inverted page tables** (one entry per *physical* frame, hashed) bound the size by RAM
rather than by address space — used on PowerPC and IA-64, but they make sharing awkward and
require a hash lookup on every miss.

### TLB

| Property | Consequence |
|---|---|
| Small (64 L1 / ~1500 L2 entries) | Coverage = entries × page size ≈ 6 MiB with 4K pages |
| Not coherent in hardware | **Software must shoot down** — the only such cache in the machine |
| Tagged by ASID/PCID (modern) | Avoids full flush on context switch |

**Effective access time** with hit ratio $h$, TLB time $t$, memory time $m$:

$$\text{EAT} = h(t+m) + (1-h)(t + 2m)$$

(Two memory accesses on a miss: one for the page table, one for the data — more with
multi-level tables.)

**→ Linux:** Ch. 22 §T.3 (shootdown, `mmu_gather`, PCID).

### Page replacement algorithms

| Algorithm | Idea | Property |
|---|---|---|
| **OPT / MIN** (Belady) | Evict the page used furthest in the future | **Provably optimal; unimplementable.** The benchmark |
| **FIFO** | Evict oldest | Simple; suffers **Belady's anomaly** |
| **LRU** | Evict least recently used | Good; exact implementation needs per-access updating → impossible |
| **Clock / second-chance** | Circular scan; reference bit gives a second chance | LRU approximation in O(1) with **one bit** |
| **NFU / aging** | Shift a counter with the reference bit | Better approximation |
| **2Q / LRU-K** | Two queues: probationary + protected | **Scan-resistant** |
| **ARC** | Adaptively balance recency vs frequency | Patented; not in Linux |
| **CLOCK-Pro / LIRS** | Uses reuse *distance*, not just recency | Stronger |

**Belady's anomaly:** for FIFO, *more* frames can cause *more* faults. Reference string
`1,2,3,4,1,2,5,1,2,3,4,5` gives 9 faults with 3 frames and 10 with 4. It is worth being able
to produce this example — it is asked.

**Stack algorithms** (LRU, OPT) cannot exhibit the anomaly, because the set of pages in
memory with $n$ frames is always a subset of the set with $n+1$ frames.

**→ Linux:** Ch. 23 §T.2 — Linux uses a two-list (2Q-like) scheme precisely for scan
resistance, plus refault distance (§T.3) to reconstruct reuse distance, plus MGLRU (§T.4).

### Working set and thrashing

**Working set** $W(t,\tau)$ = the set of pages referenced in the last $\tau$ time units
(Denning, 1968). The model's claim, empirically validated for 60 years: $W$ is small,
slowly varying, and predictive.

**Thrashing** occurs when $\sum |W_i| >$ available frames. It is a *phase transition*, not
gradual degradation: nearly every access faults, CPU utilization collapses, and — in a naive
system — the scheduler responds by admitting *more* processes, making it worse. That
positive feedback loop is why thrashing is catastrophic rather than merely slow.

**→ Linux:** Ch. 23 §T.1; `/proc/pressure/memory` (PSI) measures exactly this.

### Allocation

| Algorithm | Method | Problem |
|---|---|---|
| **First fit** | First hole big enough | Fragments the front |
| **Best fit** | Smallest sufficient hole | Leaves unusable slivers; slow |
| **Worst fit** | Largest hole | Bad in practice |
| **Buddy system** | Power-of-two splitting/merging | Up to 2× internal fragmentation; **O(log n) coalescing** |
| **Slab** | Per-object-type caches | Solves buddy's internal fragmentation for small objects |

**Robson's bound (1971):** any allocator can be forced into fragmentation requiring
$\Theta(M \log(n_{max}/n_{min}))$ memory to satisfy $M$ bytes of live requests. This is why
"just defragment better" is not a solution — the worst case is inherent.

**→ Linux:** Ch. 11 §T.2 (buddy), §T.3 (slab/SLUB), Ch. 23 §T.8 (compaction).

---

## 5. Virtual memory mechanisms

| Mechanism | What it defers |
|---|---|
| **Demand paging** | Loading until first touch |
| **Copy-on-write** | Copying until first write |
| **Memory-mapped files** | Reading until access; unifies file and memory |
| **Swapping / paging out** | Backing anonymous memory |
| **Shared memory** | Copying between processes entirely |

All five are the same trick: **the page fault is a programmable interception point.**
Understanding this unifies half of MM. → Ch. 22 §T.1.

**Page fault cost:** minor (resolved from memory) ≈ 0.5–2 µs; major (requires I/O) ≈ 100 µs
(NVMe) to 10 ms (HDD). A 10⁴ ratio, which is why the distinction matters in every
measurement.

---

## 6. File systems

### Allocation strategies

| Strategy | Random access | Fragmentation | Used by |
|---|---|---|---|
| **Contiguous** | O(1) | External, severe | CD-ROM, some RT systems |
| **Linked list** | **O(n)** | None | Rare |
| **FAT** (linked, table in memory) | O(n) but in RAM | None | FAT12/16/32 |
| **Indexed (inode)** | O(1) with direct blocks | None | Unix family |
| **Extents** | O(log n) | Low | ext4, XFS, Btrfs, NTFS |

**The Unix inode's multi-level scheme** — 12 direct, 1 single-indirect, 1 double, 1 triple —
is a classic exam question. With 4 KiB blocks and 4-byte pointers (1024 per block):

- Direct: 12 × 4 KiB = 48 KiB
- Single: 1024 × 4 KiB = 4 MiB
- Double: 1024² × 4 KiB = 4 GiB
- Triple: 1024³ × 4 KiB = 4 TiB

The design deliberately optimizes small files (most files are small) while still supporting
huge ones. Extents replaced it because for large contiguous files, storing (start, length)
beats storing every block number.

### Consistency

| Approach | Mechanism | Cost |
|---|---|---|
| **fsck** | Scan and repair after crash | O(filesystem size) — hours on large volumes |
| **Soft updates** | Order writes so on-disk state is always consistent | Complex dependency tracking; FFS/UFS |
| **Journaling** | Write intent to a log, then do the work | 2× writes for metadata (or data) |
| **Copy-on-write** | Never overwrite; atomically swap a root pointer | Fragmentation; ZFS, Btrfs |
| **Log-structured** | The whole filesystem is a log | Garbage collection; LFS, F2FS |

**Journaling modes** (ext4 terminology, worth knowing exactly):

| Mode | Journals | Guarantee |
|---|---|---|
| `data=journal` | Metadata **and** data | Strongest; ~2× write amplification |
| `data=ordered` (default) | Metadata; data written **before** the commit | No stale data exposure |
| `data=writeback` | Metadata only, no ordering | Fastest; a crash can expose stale blocks in a file |

**→ Linux:** Ch. 61 (journaling), Ch. 57–60 (real filesystems).

### Directory structures

Linear (O(n) lookup) → hashed (ext4's htree) → B-tree (XFS, Btrfs, NTFS). The progression
is driven by directories with millions of entries, where O(n) is fatal.

### Hard vs symbolic links

| | Hard link | Symbolic link |
|---|---|---|
| Points to | **Inode** | **Path string** |
| Cross-filesystem | No | Yes |
| To directories | No (would create cycles) | Yes |
| Survives target rename | **Yes** | No (dangles) |
| Extra inode | No | Yes |

The "no hard links to directories" rule exists because the directory graph must remain a
tree (plus `.` and `..`), or `find`, `rm -r`, and reference counting all break.

**→ Linux:** Ch. 53 §T.2 — this is exactly why dentry and inode are separate objects.

---

## 7. I/O

### The device-interaction progression

| Method | CPU cost | Latency |
|---|---|---|
| **Programmed I/O (polling)** | 100% busy | Lowest |
| **Interrupt-driven** | Per-transfer interrupt | Interrupt overhead per unit |
| **DMA** | Per-*transfer* interrupt only | Best for bulk |
| **Hybrid (NAPI-style)** | Adaptive | Best of both |

Linux's NAPI (Ch. 46 §T.2) is the canonical hybrid: interrupt-driven at low load, polled at
high load, switching automatically. It exists because pure interrupt-driven receive suffers
**receive livelock** — beyond a threshold arrival rate, throughput collapses to zero because
interrupts preempt the very code that would drain the queue.

### Disk scheduling (historical but asked)

| Algorithm | Method |
|---|---|
| **FCFS** | No reordering |
| **SSTF** | Shortest seek time first — can starve |
| **SCAN / elevator** | Sweep one direction, then reverse |
| **C-SCAN** | Sweep one direction only, jump back — more uniform wait |
| **LOOK / C-LOOK** | SCAN but reverse at the last request, not the end |

These matter historically (Linux's first scheduler was literally "the Linus elevator") and
matter *not at all* on SSDs, where there is no seek. The modern question is fairness and
latency isolation, not seek minimization. **Saying that** is the senior-level answer.

**→ Linux:** Ch. 64.

### RAID levels

| Level | Layout | Redundancy | Read | Write | Usable |
|---|---|---|---|---|---|
| **0** | Striping | **None** | Fast | Fast | 100% |
| **1** | Mirroring | 1 disk | Fast | 1× | 50% |
| **5** | Striping + distributed parity | 1 disk | Fast | **RMW penalty** | (n−1)/n |
| **6** | Two parity | 2 disks | Fast | Worse RMW | (n−2)/n |
| **10** | Mirror then stripe | 1+/mirror | Fast | Fast | 50% |

**The RAID-5 write hole:** a partial-stripe write must read old data + old parity, compute
new parity, write both. If power is lost between the data and parity writes, the stripe is
inconsistent *and there is no way to detect which is wrong*. Fixes: battery-backed cache,
journaling (`md`'s write-journal), or CoW (ZFS/Btrfs avoid it structurally by never
overwriting).

**→ Linux:** Ch. 67.

---

## 8. Protection and security

| Concept | Definition |
|---|---|
| **Principle of least privilege** | Grant the minimum authority necessary |
| **Complete mediation** | Check every access, every time |
| **Fail-safe defaults** | Deny by default |
| **Separation of privilege** | Require two conditions |
| **Economy of mechanism** | Keep the TCB small |
| **Open design** | Security must not depend on secrecy of the design |

These are Saltzer & Schroeder (1975) — eight principles, still the canonical list, and
still cited in kernel design discussions.

**Access control models:**

| Model | Mechanism | Example |
|---|---|---|
| **DAC** (discretionary) | Owner sets permissions | Unix mode bits, ACLs |
| **MAC** (mandatory) | System policy overrides owner | SELinux, AppArmor |
| **RBAC** | Roles between users and permissions | Many enterprise systems |
| **Capability-based** | Unforgeable token *is* the authority | File descriptors, `pidfd`, seccomp notify |

**ACL vs capability** is the classic distinction: an ACL is indexed by object ("who may
touch this?"), a capability is indexed by subject ("what may I touch?"). File descriptors
are genuine capabilities — which is why passing one over `SCM_RIGHTS` transfers authority,
and why "return an fd" is the recommended kernel API design (Ch. 24 §T.5).

**→ Linux:** Ch. 102.

---

## 9. Virtualization

| Type | Definition |
|---|---|
| **Type 1 (bare metal)** | Hypervisor on hardware — Xen, ESXi |
| **Type 2 (hosted)** | Hypervisor in an OS — VirtualBox, **KVM** (arguably hybrid) |
| **Paravirtualization** | Guest is modified to cooperate — Xen PV, virtio |
| **Containers** | OS-level isolation, one kernel — namespaces + cgroups |

**Popek & Goldberg (1974)** — the formal criterion, and a good interview answer:

> An architecture is virtualizable iff the set of **sensitive** instructions (those that
> change or expose privileged state) is a subset of the set of **privileged** instructions
> (those that trap in user mode).

x86 famously failed this — 17 sensitive-but-unprivileged instructions — which is why
VMware needed binary translation and why Intel VT-x / AMD-V were added.

**→ Linux:** Ch. 104.

---

## 10. Distributed systems (for architect roles)

Increasingly asked once you are past senior, because kernel work now sits inside distributed
systems.

**CAP theorem** (Brewer/Gilbert-Lynch): under a network **P**artition, you must choose
between **C**onsistency and **A**vailability. The common misstatement is "pick two of
three" — partitions are not optional, so the real choice is CP or AP.

**The eight fallacies of distributed computing** (Deutsch/Gosling): the network is reliable;
latency is zero; bandwidth is infinite; the network is secure; topology does not change;
there is one administrator; transport cost is zero; the network is homogeneous.

**Consensus:** Paxos, Raft. Know that consensus requires a majority quorum and that FLP
proves no deterministic consensus is possible in a fully asynchronous system with even one
faulty process — hence timeouts everywhere.

**Idempotency, at-least-once vs exactly-once:** exactly-once delivery is impossible;
exactly-once *processing* is achievable via idempotency + deduplication. This matters for
anything involving retries, which includes every storage and network path.

---

## 11. The numbers to have memorized

Interviewers ask for order-of-magnitude estimates. Knowing these lets you sanity-check any
design in your head.

| Operation | Latency | Relative |
|---|---|---|
| L1 cache reference | 1 ns | 1× |
| Branch mispredict | 3 ns | 3× |
| L2 cache reference | 4 ns | 4× |
| Mutex lock/unlock (uncontended) | 17 ns | 17× |
| Main memory reference | 100 ns | 100× |
| Compress 1 KB (Snappy) | 2 µs | |
| **Context switch (direct)** | **1–2 µs** | |
| Send 1 KB over 10 Gbps | 1 µs | |
| **Syscall (no mitigations)** | **~60 ns** | |
| **Syscall (with Spectre mitigations)** | **200–500 ns** | |
| SSD random read (NVMe) | 20–80 µs | |
| Read 1 MB sequentially from memory | 50 µs | |
| Read 1 MB sequentially from NVMe | 150 µs | |
| **TLB shootdown IPI** | **1–10 µs** | |
| **Device cache flush (NVMe)** | **20–500 µs** | |
| Round trip within datacenter | 500 µs | |
| **HDD seek** | **8–15 ms** | |
| Read 1 MB from HDD | 5 ms | |
| Packet CA → Netherlands → CA | 150 ms | |

Derived facts worth stating:

- **Memory is the new disk; disk is the new tape.** The DRAM/NVMe gap (~500×) is now
  smaller than the L1/DRAM gap (~100×) was historically significant — cache misses matter
  as much as I/O now.
- **A syscall costs ~1000 L1 hits.** This is the entire argument for `io_uring`, vDSO, and
  batching.
- **HDD random/sequential ratio is ~1000×; NVMe's is ~2×.** Every seek-avoidance
  optimization in the storage stack was designed for the first number and is overhead under
  the second. → Ch. 51 §T.7.

---

## 12. Fifteen questions you should be able to answer in 60 seconds

Self-test. If any takes longer, that section above is your homework.

1. What are the four Coffman conditions, and which one does Linux break?
2. Why is exact LRU unimplementable, and what does Linux do instead?
3. State Belady's anomaly and give a reference string that exhibits it.
4. Mutex vs binary semaphore — name the structural difference and one consequence.
5. Why is EDF optimal but rate-monotonic still used?
6. Explain priority inversion and name two protocols that bound it.
7. Why did paging beat segmentation?
8. What is the working set model, and why is thrashing a phase transition?
9. Compare inode block-pointer schemes with extents, with a size calculation.
10. Why can you not hard-link a directory?
11. State Popek & Goldberg's criterion and why x86 failed it.
12. Give the three journaling modes and what each risks.
13. What is the RAID-5 write hole and what are three fixes?
14. Why does a context switch cost more than the register save/restore suggests?
15. What is the difference between an ACL and a capability, and which are file descriptors?

---

## Sources

- **Arpaci-Dusseau & Arpaci-Dusseau, *Operating Systems: Three Easy Pieces*** — free online,
  and the best single book for everything in this document. If you read one thing, read this.
- Silberschatz, Galvin, Gagne, *Operating System Concepts* — the standard curriculum text;
  the source of most textbook-phrased interview questions.
- Tanenbaum & Bos, *Modern Operating Systems* — the other standard; stronger on distributed
  and security.
- Coffman, Elphick, Shoshani, "System Deadlocks," *Computing Surveys*, 1971.
- Denning, "The Working Set Model for Program Behavior," *CACM*, 1968.
- Belady, "A Study of Replacement Algorithms," *IBM Systems Journal*, 1966.
- Liu & Layland, "Scheduling Algorithms for Multiprogramming in a Hard-Real-Time
  Environment," *JACM*, 1973.
- Sha, Rajkumar, Lehoczky, "Priority Inheritance Protocols," *IEEE Trans. Computers*, 1990.
- Popek & Goldberg, "Formal Requirements for Virtualizable Third Generation Architectures,"
  *CACM*, 1974.
- Saltzer & Schroeder, "The Protection of Information in Computer Systems," 1975.
- Dijkstra, "Cooperating Sequential Processes," 1965 — the origin of semaphores and the
  dining philosophers.

→ Next: [question-bank.md](question-bank.md)

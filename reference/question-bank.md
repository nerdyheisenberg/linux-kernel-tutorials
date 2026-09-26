# Question Bank — graded by level, with model answers

> **How to use this.** Each question carries a level marker and a model answer written the
> way you should actually say it — not an essay. The **↪ follow-up** line is what a good
> interviewer asks next; if you cannot answer that, you have not finished studying the
> topic. The **→** link points at the chapter that derives the answer.
>
> | Marker | Level | What it tests |
> |---|---|---|
> | **[S]** | Senior | Correct mechanism, correct vocabulary |
> | **[St]** | Staff | Why the mechanism exists; what it costs; the alternative |
> | **[P]** | Principal | Judgement across subsystems; when to reject the question |
>
> A senior candidate answers all **[S]** fluently and most **[St]** with prompting. A staff
> candidate answers **[St]** unprompted. A principal candidate reframes **[P]** questions
> before answering them.

---

## Part A — Concurrency and synchronization

### A1 [S] When would you use a spinlock instead of a mutex?

When the critical section is short (shorter than two context switches, ~1–2 µs), or when
you cannot sleep — interrupt context, or holding another spinlock. The tradeoff is burning
CPU while waiting versus paying ~2× context-switch cost to sleep and wake.

↪ *Follow-up:* "What if you're not sure how long it will be held?" → `mutex` already
optimistically spins (adaptive/OSQ) while the owner is running, so the modern default is
mutex unless you are in atomic context. The rule "spinlock for short" is partly obsolete.
→ Ch. 14 §T.3, §T.5

### A2 [S] What is the difference between `spin_lock()` and `spin_lock_irqsave()`?

`spin_lock_irqsave()` additionally disables local interrupts. You need it when the same lock
is also taken from hard-IRQ context on the same CPU — otherwise the interrupt deadlocks
against the thread that holds the lock. `spin_lock_bh()` is the softirq equivalent.

↪ *Follow-up:* "Why `irqsave` rather than `irq`?" → `spin_lock_irq()` unconditionally
re-enables on release; if you were already in an IRQ-disabled region, that is a bug. Save/
restore is composable. On PREEMPT_RT, `spin_lock_irqsave` does **not** disable interrupts
at all — it becomes an `rt_mutex`. → Ch. 14 §T.4, Ch. 17

### A3 [St] Explain RCU to me.

Three layers:

1. **Mechanism.** Readers run with no locks, no atomics, no writes at all — `rcu_read_lock()`
   compiles to nothing (or a preempt-count increment). Writers publish a new version with
   `rcu_assign_pointer()` and defer freeing the old one until every CPU has passed through a
   **quiescent state** — a point where it provably holds no reference. The sum of a
   quiescent state on every CPU is a **grace period**.
2. **Forces.** It buys zero reader-side cost and perfect reader scalability by paying with
   deferred reclamation: memory is held longer, and updates are more expensive. That trade
   is correct exactly when reads massively outnumber writes and the data is
   pointer-reachable.
3. **Alternative.** A `rwlock` gives immediate reclamation but readers write to the shared
   lock word — cacheline ping-pong makes read throughput go *down* as you add cores. On the
   dcache, that is the difference between scaling and not.

↪ *Follow-ups:* "What if the reader needs to sleep?" → SRCU. "What if you free something
that is not a pointer-reachable object?" → RCU only defers; anything with side effects
(unregistering, stopping a timer) still needs explicit synchronization. "How do you get the
grace period to end faster?" → `synchronize_rcu_expedited()`, at the cost of IPIs to every
CPU. → Ch. 15

### A4 [St] What memory barrier problem does `rcu_assign_pointer()` solve?

Publication ordering. Initializing an object's fields and then storing the pointer are
independent stores; the CPU or compiler may reorder them, so another CPU can observe a
non-NULL pointer to an uninitialized object. `rcu_assign_pointer()` is a release store — it
orders all prior stores before the pointer publication.

↪ *Follow-up:* "And on the read side?" → `rcu_dereference()` is a consume/dependency-ordered
load. On every architecture except DEC Alpha, address dependency alone suffices; Alpha's
split cache could reorder dependent loads, which is why the API exists at all rather than
being a plain load. Also, the compiler can break dependencies via value speculation, hence
`READ_ONCE()`. → Ch. 13 §T.5, Ch. 15 §T.3

### A5 [S] What is a seqlock and when is it appropriate?

A writer increments a sequence counter on entry and exit; readers snapshot the counter,
read, and re-read the counter — if it changed or was odd, retry. Appropriate for
**small, read-mostly, write-rare** data where readers can safely be retried and the data
contains no pointers to follow. The canonical user is the timekeeper.

↪ *Follow-up:* "What is its failure mode?" → Writers are never blocked by readers, so a
steady stream of writers starves readers indefinitely. And a reader can observe a torn,
inconsistent intermediate state — it must not *act* on the data before validating the
counter, which means no dereferencing pointers read inside the section. → Ch. 14 §T.6

### A6 [St] Why does `smp_mb()` exist if the CPU has cache coherence?

Coherence and ordering are different properties. Coherence guarantees all CPUs agree on the
order of writes **to a single location**. It says nothing about the relative order of
accesses to *different* locations, which is what store buffers, invalidate queues, and
speculative execution reorder. A barrier constrains that relative order.

↪ *Follow-up:* "Give the minimal example." → Store-buffer / Dekker: two CPUs each store to
their own flag then load the other's; without a full barrier both can read zero, which no
sequentially-consistent execution permits. → Ch. 13 §T.2

### A7 [P] A team proposes a new global lock for a new feature. What do you ask?

- What is the *contention shape* — how many CPUs, how often, what hold time? A global lock
  is fine at 10 acquisitions/sec and fatal at 10M.
- Is the state genuinely global, or is it per-CPU/per-object state that has been globalized
  by the data structure choice? Most global locks are a data-structure smell.
- What is the *lock order* relative to existing locks, and has lockdep seen it?
- Does anything sleep inside it? Does it get taken from IRQ context? Those constrain the
  type.
- What is the RT story — will this become an `rt_mutex` and can it then invert?
- Can this be RCU, per-CPU + fold-on-read, or a seqlock instead?

The principal-level move is to note that "add a lock" and "choose a data structure" are the
same decision, and push back to the latter. → Ch. 25, Ch. 16, Ch. 105

### A8 [St] What is false sharing and how do you find it?

Two variables that are logically independent land in the same cacheline; writes from
different CPUs bounce the line between caches even though there is no logical contention.
Find it with `perf c2c record/report`, which attributes HITM (hit-modified) events to
specific cachelines and offsets.

↪ *Follow-up:* "How do you fix it?" → `____cacheline_aligned_in_smp` on the struct member,
reorder fields so read-mostly and write-hot are in different lines, or convert to per-CPU.
Note that padding costs memory and cache footprint, so it is not free. → Ch. 16 §T.4,
Ch. 105

### A9 [S] Can you call `kmalloc(GFP_KERNEL)` in an interrupt handler?

No. `GFP_KERNEL` permits sleeping (direct reclaim). In atomic context use `GFP_ATOMIC`,
which cannot sleep and instead dips into emergency reserves — so it fails more often and
you must handle failure.

↪ *Follow-up:* "What is the better design?" → Preallocate. The allocation-in-IRQ question
usually means the design is wrong; use a mempool, a per-CPU cache, or move the work to a
threaded IRQ / workqueue where you can sleep. → Ch. 11 §T.5, Ch. 17

### A10 [St] Explain the ABA problem and how Linux avoids it.

A lock-free algorithm CASes on a pointer; between read and CAS, the value changes A→B→A. The
CAS succeeds but the world has changed underneath — typically the node was freed and
reallocated. Linux's dominant answer is to not free at all during the operation: RCU
guarantees the object cannot be reused while any reader holds it. Alternatives are tagged
pointers (version counter in unused bits) and hazard pointers.

↪ *Follow-up:* "Where does `cmpxchg_double` come in?" → It CASes a pointer and its tag
atomically, which SLUB uses for its lockless freelist. → Ch. 13 §T.7, Ch. 15

---

## Part B — Memory management

### B1 [S] Walk me through what happens on a page fault.

Hardware traps → `do_page_fault()` reads the faulting address (CR2 / FAR) and error code →
find the VMA covering the address. Then:

- No VMA, and not a stack-growth case → SIGSEGV.
- VMA exists but permissions disallow the access → SIGSEGV.
- VMA exists, PTE is not present, anonymous → allocate a zero page (or map the shared zero
  page for a read).
- Present but read-only and this is a write to a COW page → allocate, copy, remap.
- Not present, file-backed → look in the page cache; if absent, issue readahead + I/O
  (this is the **major** fault).
- Present and permissions fine → spurious; another CPU already fixed it. Just return.

Modern kernels attempt this under a per-VMA lock first (`lock_vma_under_rcu`) and fall back
to `mmap_lock` — that change is recent and worth mentioning.

↪ *Follow-up:* "What is the cost difference between minor and major?" → ~1 µs vs ~100 µs on
NVMe. Four orders of magnitude on spinning disk. → Ch. 22 §T.1

### B2 [S] What is the difference between `kmalloc` and `vmalloc`?

`kmalloc` returns **physically contiguous** memory from the slab allocator, with a cheap
virt↔phys translation, size-limited (practically a few MiB, backed by the buddy allocator's
order limit). `vmalloc` stitches together arbitrary physical pages into a virtually
contiguous range — it can satisfy large requests but costs page-table setup, a TLB entry per
page, and cannot be used for DMA that requires physical contiguity.

↪ *Follow-up:* "So why not always vmalloc?" → TLB pressure, slower allocation, the vmalloc
address space is finite on 32-bit, and `virt_to_phys()` does not work on it. Also
`kvmalloc()` exists precisely to say "I don't care, try kmalloc then fall back." → Ch. 11

### B3 [St] Why do folios exist?

`struct page` was a 64-byte-per-4KiB tax with a fatal ambiguity: a function taking a
`struct page *` could not tell whether it was given a head page, a tail page, or a
single page. Every compound-page user reimplemented `compound_head()` defensively.
A **folio** is a type-level statement: "this is a head page, never a tail." That converts a
runtime convention into a compile-time guarantee, removes millions of redundant
`compound_head()` calls, and is the prerequisite for large-block filesystem support and
eventually for shrinking `struct page` to 8 bytes (memdesc).

↪ *Follow-up:* "Give the general principle." → When a correctness obligation is routinely
forgotten, move it into the type/primitive rather than into documentation. Same principle as
`devm_` (Ch. 28), `guard()` (Ch. 05), `mmiowb()` folding into `spin_unlock` (Ch. 34).
→ Ch. 52 §T.2

### B4 [St] How does the kernel decide what to reclaim?

Two LRU list pairs — anon and file — each split into **active** and **inactive**. New pages
enter inactive; a second reference promotes to active. Reclaim scans inactive. The split is a
2Q variant and exists for **scan resistance**: a `cat bigfile` must not evict the working
set. The anon/file balance is driven by measured refault distances (`workingset_refault`),
which reconstruct reuse distance from evicted-entry shadow records — i.e. the kernel
*measures* whether its previous decision was wrong and adjusts. `swappiness` biases the
ratio; memcg adds per-cgroup limits and protections (`memory.low`, `memory.min`).

↪ *Follow-up:* "What does MGLRU change?" → Replaces the two-list scheme with multiple
generations and page-table-walk-based aging, which is much cheaper at large scale and gives
finer-grained recency. → Ch. 23 §T.2–T.4

### B5 [St] What is a TLB shootdown and why is it expensive?

The TLB is the only cache in the machine with no hardware coherence. When one CPU changes a
page-table entry, every other CPU that might have cached that translation must be told —
by **IPI**. Cost: 1–10 µs, scaling with the number of target CPUs, and each target's
interrupt latency adds jitter. Batching (`mmu_gather`) amortizes it across a whole `munmap`;
`ASID`/`PCID` avoids full flushes on context switch; some architectures (ARM64) have
broadcast TLB invalidation in hardware, removing the IPI.

↪ *Follow-up:* "Where does this show up in production?" → A multithreaded process doing
frequent `munmap`/`madvise(DONTNEED)` on a 96-core box generates a storm of IPIs that
appears as unexplained latency on *unrelated* threads. → Ch. 22 §T.3

### B6 [S] What is the difference between RSS, PSS, and virtual size?

**VSZ** is address space reserved — meaningless as a memory metric (a `malloc` of 1 TiB with
no touches counts). **RSS** is resident pages, but counts shared pages fully in *every*
sharer, so summing RSS over processes double-counts. **PSS** divides each shared page by the
number of sharers, so PSS *does* sum correctly. **USS** is private-only — what you would get
back by killing the process.

↪ *Follow-up:* "What about page cache?" → None of these tell you about reclaimable page
cache, which is why "free memory" is a bad metric and PSI (`/proc/pressure/memory`) is a good
one — it measures stall time, which is the thing you actually care about. → Ch. 23 §T.7

### B7 [P] Design a memory limit for untrusted tenants on a shared host.

Reframe first: "memory limit" under-specifies. You must decide the failure mode.

- **cgroup v2 `memory.max`** gives a hard cap with OOM-kill at the boundary. Correct for
  tenants that must never affect neighbours, wrong if you want graceful degradation.
- **`memory.high`** throttles instead of killing — reclaim pressure is applied and the
  allocator is slowed, giving the workload a chance to shed load.
- **`memory.min` / `memory.low`** protect a working set from reclaim, which is the *other*
  half — a limit without a protection floor means a noisy neighbour steals your page cache.
- **PSI per-cgroup** is how you detect distress before the kill, and is what your autoscaler
  should read.
- **Kernel memory** (slab, page tables, socket buffers) counts against v2's limit; make sure
  the tenant cannot force unbounded kernel allocation — that is the real attack.
- Swap: `memory.swap.max` separately, or you have not actually bounded anything.

State the choice explicitly: "high + PSI-driven shedding, max as a backstop, min as a floor."
→ Ch. 23 §T.6, Ch. 105

---

## Part C — Scheduling and time

### C1 [S] How does CFS/EEVDF decide what to run next?

EEVDF (6.6+) tracks each entity's **lag** — the difference between the service it *should*
have received under an ideal fluid-share model and what it actually got. Positive lag means
under-served. Among entities that are **eligible** (lag ≥ 0), it picks the earliest
**virtual deadline**, where the deadline is computed from the entity's requested timeslice.
This gives proportional fairness *and* a latency knob (`sched_attr::sched_runtime`) that CFS
lacked — short-slice tasks get earlier deadlines and hence lower latency without needing a
higher weight.

↪ *Follow-up:* "What did CFS do and why was it replaced?" → CFS picked minimum `vruntime`
via an rbtree, which is fair but has no latency dimension; latency was hacked in via
`sched_latency` and wakeup preemption heuristics that were hard to reason about. → Ch. 21 §T.4

### C2 [St] A latency-sensitive thread occasionally sees 5 ms delays. Where do you look?

Enumerate the sources in order of likelihood:

1. **Preemption by another task** — check `sched_switch` tracepoints, run with
   `SCHED_FIFO`/`SCHED_DEADLINE` to test.
2. **IRQ / softirq** — `/proc/interrupts`, `perf` on `irq_handler_entry`. A NIC on the same
   CPU is a classic. Fix with IRQ affinity.
3. **Long non-preemptible section** — `preemptirqsoff` tracer, `hwlat` detector.
4. **Page fault**, especially a major one or a COW storm — `mlockall`, prefault.
5. **CPU frequency / idle state transitions** — C-state exit latency can be hundreds of µs;
   `cpu_dma_latency` QoS or `idle=poll`.
6. **SMT sibling** contention, or another cgroup's throttling (`cpu.max` throttled periods in
   `cpu.stat`).
7. **TLB shootdown IPI** from an unrelated process.
8. **Hardware SMI/SMM** — invisible to Linux; `hwlatdetect` is the only way to see it.

The senior answer names the tools; the staff answer names the *order* and why.
→ Ch. 21, Ch. 103

### C3 [S] What is the difference between `SCHED_FIFO`, `SCHED_RR`, and `SCHED_DEADLINE`?

FIFO and RR are fixed-priority (1–99, above all normal tasks); RR adds round-robin among
equal priorities, FIFO runs until it blocks or yields. `SCHED_DEADLINE` is EDF + Constant
Bandwidth Server: you declare (runtime, deadline, period) and the kernel **admission-controls**
— it refuses the request if the set becomes unschedulable — then enforces the budget so an
overrunning task is throttled rather than damaging others.

↪ *Follow-up:* "Which would you use for a real-time audio thread?" → `SCHED_DEADLINE` if the
work is genuinely periodic with known WCET, because it gives isolation and provable
admission. FIFO if the work is aperiodic. Note the RT throttling default
(`sched_rt_runtime_us` = 950000/1000000) which caps all RT at 95% — a runaway FIFO task will
be throttled, and people are frequently surprised by this. → Ch. 21 §T.6, Ch. 103

### C4 [St] Why are there both `hrtimer` and the timer wheel?

Different cost/precision tradeoffs. The **timer wheel** (`timer_list`) is O(1) insert with
cascading, millisecond-ish granularity, optimized for the case that dominates: timeouts that
are **cancelled before they fire** (every TCP retransmit timer). **`hrtimer`** is an rbtree
ordered by absolute ktime, nanosecond resolution, programmed into a one-shot clock event
device — precise but with per-timer setup cost.

↪ *Follow-up:* "Which does `schedule_timeout` use?" → the wheel, because a timeout is a
"probably won't fire" object. `nanosleep` uses hrtimer, because a sleep is a "definitely will
fire, and precision matters" object. The design principle: **optimize the operation that
actually dominates**, which for timeouts is cancellation, not expiry. → Ch. 19 §T.2–T.4

### C5 [St] Why do we have both `CLOCK_MONOTONIC` and `CLOCK_BOOTTIME`?

`CLOCK_MONOTONIC` does not advance during suspend; `CLOCK_BOOTTIME` does. Which one is
correct depends entirely on whether your interval is about *elapsed CPU-observable time* or
*wall-clock duration*. Timeouts usually want MONOTONIC (a 5-second network timeout should not
fire instantly after a 3-hour suspend)... except when they should
(`CLOCK_BOOTTIME` alarm timers exist for exactly that). Never use `CLOCK_REALTIME` for
intervals — it jumps with NTP and settimeofday.

↪ *Follow-up:* "What about `CLOCK_MONOTONIC_RAW`?" → Unadjusted by NTP slewing; use it when
you want the raw hardware rate, e.g. calibrating something. → Ch. 19 §T.1

---

## Part D — Drivers and the device model

### D1 [S] Walk me through what happens when you plug in a PCIe device.

Hotplug interrupt → PCI core rescans the bus, reads config space (vendor/device ID, BARs,
capabilities) → creates a `struct pci_dev` and registers it with the driver core →
`bus_probe_device` matches against every registered `pci_driver`'s ID table → on a match,
`->probe()` runs: `pci_enable_device()` (power up, decode enable), `pci_request_regions()`
(claim the BARs), `pci_iomap()`, set DMA mask, allocate MSI/MSI-X vectors, `pci_set_master()`
for bus-master DMA, register the class-level interface (netdev, blockdev, ...), and finally
enable interrupts. Unwind in exact reverse — or use `devm_` and let the core do it.

↪ *Follow-up:* "What if `->probe` is called before a dependency is ready?" → Return
`-EPROBE_DEFER`; the core retries after other devices probe. The whole deferred-probe
mechanism exists because the probe order is not a topological sort of the resource
dependency graph. → Ch. 27, Ch. 37

### D2 [S] What problem does `devm_*` solve?

Error-path leaks. A probe function with eight resources needs eight labelled `goto`s and a
reverse-order unwind; every new resource is an opportunity to forget a line. `devm_`
allocations are tracked on a per-device list and released automatically when probe fails or
the device is removed. Same principle as folios and `guard()`: when a correctness obligation
is routinely forgotten, move it into the primitive.

↪ *Follow-up:* "When is `devm_` wrong?" → When the lifetime is not the device's. If an
object outlives `remove()` — because a userspace fd still references it — `devm_` frees it
too early and you get a UAF. That is the char-device lifetime problem (Ch. 29), and the
answer is `struct kref` on a separate object, not `devm_`. Also note `devm_` release order is
strictly reverse-registration, which occasionally is not the order you need. → Ch. 28

### D3 [St] A driver works fine but corrupts data at high load. Where do you look?

Rank by base rate:

1. **Missing DMA ownership transitions.** No `dma_sync_single_for_cpu/device`, or accessing a
   buffer still owned by the device. Turn on `CONFIG_DMA_API_DEBUG` and run with an IOMMU in
   strict mode — that converts silent corruption into a faulting error.
2. **Missing memory barriers between descriptor writes and the doorbell.** `writel()` orders
   against prior MMIO but you need `dma_wmb()` between descriptor-in-memory writes and the
   MMIO doorbell.
3. **Cacheline sharing between a DMA buffer and other data** on a non-coherent platform —
   the cache invalidate destroys the neighbour. Buffers must be cacheline-aligned and
   -sized.
4. **Locking**: a race between the IRQ handler and the submit path. Turn on lockdep,
   KASAN, and `PROVE_LOCKING`.
5. **Ring wrap / index arithmetic** — only manifests at the wrap boundary, hence "high load
   only."

↪ *Follow-up:* "Which of those does KASAN catch?" → 4 and 5 (if they touch out of bounds);
not 1–3, which are hardware-visibility problems. That distinction is the point.
→ Ch. 35, Ch. 34, Ch. 50

### D4 [St] Why does the kernel have both `platform_device` and Device Tree?

They answer different questions. `platform_device` is the **software model** for
non-discoverable devices — it gives them a place in the driver-model hierarchy so probe,
power management, and sysfs work uniformly. Device Tree is the **data source** that describes
which such devices exist and with what resources, moving that description out of board files
and into a firmware-supplied blob. Before DT, every board was C code in `arch/arm/mach-*/`,
which is why the ARM tree was 100k lines of board files that Torvalds famously objected to.

↪ *Follow-up:* "And ACPI?" → Same role as DT on x86/server ARM, but with a bytecode
interpreter (AML) that can execute methods, which makes it far more powerful and far harder
to debug. → Ch. 31, Ch. 32, Ch. 33

### D5 [S] How do you pass data between an interrupt handler and process context?

The handler does the minimum — acknowledge the device, grab the data — and defers. Options in
increasing heaviness: **softirq/tasklet** (atomic, no sleeping, runs soon), **threaded IRQ**
(`request_threaded_irq`, sleepable, own kthread, RT-friendly), **workqueue** (sleepable,
shared pool, higher latency), **kfifo + wait queue** for the data itself.

↪ *Follow-up:* "Which is the modern default?" → Threaded IRQ. It is the only option that is
RT-correct, gives the work a schedulable identity you can prioritize and pin, and it is what
`PREEMPT_RT` forces anyway. Tasklets are deprecated. → Ch. 17, Ch. 18

---

## Part E — Storage and filesystems

### E1 [S] Trace a `write()` to a file on ext4 down to the disk.

`write()` → `vfs_write` → `file->f_op->write_iter` → `generic_perform_write`: find/allocate
the folio in the page cache, `->write_begin` (allocate blocks or set up delayed allocation),
copy from user, `->write_end`, mark dirty. Return — **nothing has hit the disk yet**. Later,
writeback (`wb_workfn`, triggered by dirty ratio, expiry, or `fsync`) calls
`->writepages` → iomap/ext4 maps folios to extents, builds bios → submit to blk-mq → the
request queue → I/O scheduler (or none) → driver → device. Journal commits interleave to
record metadata changes. `fsync()` forces writeback *plus* a journal commit *plus* a
`REQ_OP_FLUSH` to defeat the device's volatile write cache.

↪ *Follow-up:* "Where can data be lost if the machine loses power at each stage?" → Anywhere
before the FLUSH completes. And a `FUA` write without a preceding flush only guarantees *that*
write is durable, not prior ones. → Ch. 52, Ch. 57, Ch. 63

### E2 [St] What does `fsync()` actually guarantee, and what does it not?

It guarantees the file's data and the metadata needed to retrieve it are durable on the
device — after the device acknowledges a cache flush. It does **not** guarantee: that the
*directory entry* is durable (you must `fsync` the parent directory after a create/rename),
that other files are durable, or that an ordering exists between fsyncs on different files.
And historically it did not reliably report errors — a failed writeback could be reported to
whoever called `fsync` first and lost for everyone else. That is the "fsyncgate" problem
(PostgreSQL, 2018), fixed by errseq_t giving each file description its own error-seen
marker.

↪ *Follow-up:* "So how do you write a file atomically?" → Write to a temp file, `fsync` it,
`rename` over the target, `fsync` the directory. Four steps, and people skip two of them.
→ Ch. 53, Ch. 61

### E3 [St] Compare ext4, XFS, and Btrfs for a database workload.

- **ext4**: journaled overwrite-in-place. Predictable, low metadata overhead, lowest latency
  variance. `data=ordered` is enough since the DB does its own WAL. Weak at very high
  parallel metadata rates (single journal).
- **XFS**: extent-based, B+tree metadata, **allocation groups give parallel metadata
  scaling**, delayed logging. Best choice for large files and many-core hosts. This is why
  RHEL defaults to it.
- **Btrfs**: CoW. Gives snapshots and checksums, but CoW on random in-place writes of a DB
  file causes severe fragmentation and write amplification — you must set `nodatacow` on the
  DB directory, which disables checksums, which removes most of the reason you chose Btrfs.

The senior answer is "XFS, or ext4 if you want boring." The staff answer explains *why*
CoW and a DB's random-overwrite pattern are structurally mismatched.
→ Ch. 57, Ch. 58, Ch. 59

### E4 [S] What is the block layer's plugging mechanism for?

A process about to submit multiple related I/Os starts a **plug**; requests accumulate in a
per-task list rather than going straight to the hardware queue, allowing merging of adjacent
requests and batched submission. Unplug happens on explicit finish or when the task sleeps.
It reduces per-request overhead and lock traffic, and it is per-task so it needs no locking
at all.

↪ *Follow-up:* "What happens if the task sleeps while plugged?" → `schedule()` flushes the
plug automatically — otherwise the I/O you are waiting for would never be submitted, a
self-deadlock. → Ch. 63 §T.5

### E5 [St] Why did blk-mq replace the single-queue block layer?

The old layer had a single request queue with a single spinlock per device. At NVMe rates
(millions of IOPS) that lock is the bottleneck — you cannot reach a million IOPS through one
contended cacheline. blk-mq has **per-CPU software queues** (no cross-CPU contention on
submit) mapped onto **hardware queues** matching the device's actual submission queues, with
per-queue tag allocation (`sbitmap`). It also matches the hardware: NVMe genuinely has one
submission queue per core.

↪ *Follow-up:* "What did we lose?" → Global I/O scheduling decisions. Reordering across CPUs
is much harder, which is why the modern schedulers (`mq-deadline`, `bfq`, `kyber`) are
structured differently, and why `none` is the right answer for NVMe — there is nothing useful
to reorder. → Ch. 63 §T.3, Ch. 64

### E6 [P] Design the storage stack for a system that must never lose an acknowledged write.

Enumerate every layer that can buffer, and state the barrier at each:

1. **Application** — must `fsync()`/`fdatasync()` and check the return; must fsync the
   directory for namespace operations.
2. **Page cache** — defeated by fsync, or bypassed by `O_DIRECT` (which shifts the alignment
   and buffering burden to you).
3. **Filesystem journal** — `data=ordered` at minimum; journal on a separate durable device
   if you want to decouple.
4. **Device Mapper / MD** — must pass FLUSH/FUA through (they do), but RAID-5 has a **write
   hole**: use RAID-1/10, or RAID-6 with a write journal, or a CoW filesystem.
5. **Device volatile write cache** — defeated by `REQ_OP_FLUSH`; or disable it
   (`hdparm -W0`, NVMe `VWC`), or use a device with **power-loss protection** (enterprise SSD
   capacitors), which makes flush nearly free.
6. **Controller cache** — battery/flash-backed, or disabled.
7. **The write itself may tear.** Sector atomicity is only guaranteed at the device sector
   size; larger atomic writes need `RWF_ATOMIC` (6.11+) or application-level checksums.

Then the judgement: state the cost. A true fsync per write caps you at device flush latency
(20–500 µs), so you need **group commit** — batch N writers into one flush and acknowledge
them together. That is what every database does, and it is the answer they are listening for.
→ Ch. 51, Ch. 61, Ch. 67, Ch. 69

---

## Part F — Networking

### F1 [S] What is an `sk_buff` and why is it shaped that way?

The universal packet container. Its key property is **headroom and tailroom**: the data
pointer sits in the middle of the buffer so each protocol layer can prepend its header by
moving a pointer (`skb_push`) rather than copying. `skb_pull` on receive is the inverse. That
single design choice makes a layered stack O(1) per layer instead of O(copies).

↪ *Follow-up:* "What is `skb_clone` vs `skb_copy`?" → Clone shares the data (`skb_shared_info`
refcount) and copies only the metadata — used by `tcpdump`/taps. Copy duplicates everything.
Writing to a cloned skb requires `skb_unshare` first — the COW pattern again. → Ch. 71

### F2 [St] Why does XDP exist when we already have netfilter?

Position in the pipeline. XDP runs in the driver, on the DMA buffer, **before** `sk_buff`
allocation — which is the single most expensive per-packet operation in the receive path. For
drop/redirect workloads (DDoS mitigation, load balancing), avoiding allocation is the entire
performance story: ~10× throughput. Netfilter runs after skb allocation and has access to
conntrack and the full stack — much more capable, much more expensive.

↪ *Follow-up:* "What is XDP's cost?" → No fragmentation/reassembly, no conntrack, limited
metadata, driver support required, and you must be careful with XDP_TX/REDIRECT buffer
lifetimes. It is a fast path, not a replacement. → Ch. 74, Ch. 73

### F3 [St] How does eBPF stay safe?

The **verifier**: it simulates all paths with abstract interpretation over value ranges,
proves the program terminates (originally by forbidding loops, now bounded loops with a proof
of decreasing bound), proves every memory access is in-bounds, proves no uninitialized read,
enforces per-program-type context access rules, and checks helper call signatures. Then JIT
compilation to native code. Plus: no arbitrary kernel function calls except an allowlist
(kfuncs), no unbounded loops, instruction-count and complexity limits.

↪ *Follow-up:* "What is the verifier's failure mode?" → It rejects correct programs
(incompleteness). It is a sound but not complete analysis, so the practical experience is
fighting it into accepting things you know are fine. Also: the verifier is itself a large
attack surface — several CVEs have been verifier bugs allowing OOB access. → Ch. 75

### F4 [S] What does NAPI solve?

**Receive livelock.** Under high packet rates, per-packet interrupts consume all CPU in
interrupt context, preempting the softirq/thread that would actually drain the ring — so
goodput drops to zero as offered load rises. NAPI disables the device's interrupt on the
first packet and switches to polling with a budget, then re-enables when the ring drains.
Interrupt-driven at low load (low latency), polled at high load (high throughput), switching
automatically.

↪ *Follow-up:* "What is the budget for?" → Bounding how long one device can hold the softirq,
so a fast NIC cannot starve other devices or push work to `ksoftirqd` unfairly. → Ch. 46 §T.2

### F5 [St] Why is `io_uring` faster than epoll + read?

Three separate wins, worth separating:

1. **Batching** — one `io_uring_enter` submits N operations; epoll needs N syscalls. At
   ~200–500 ns per syscall with mitigations, that dominates at high IOPS.
2. **Zero syscalls at all** in polled mode (`SQPOLL`) — a kernel thread picks up submissions
   from the shared ring.
3. **Genuine async for file I/O** — `epoll` never worked for regular files (they are always
   "ready"), so file I/O needed a thread pool. `io_uring` makes it first-class.

↪ *Follow-up:* "What is the cost?" → A large new attack surface (many CVEs, disabled by
default in some distros and by Google in Android/ChromeOS), a complex ownership model for
registered buffers, and completion-ordering subtleties. → Ch. 76

---

## Part G — Architecture and judgement (staff/principal)

### G1 [P] When should code go in the kernel versus userspace?

Four criteria, and I would apply them in order:

1. **Privilege** — does it require access to hardware or to state no user process may have?
   If not, that is a strong argument for userspace.
2. **Performance** — does the crossing cost dominate? A syscall is ~1000 L1 hits; if you must
   do it per-packet or per-microsecond, in-kernel wins. If you can batch, the argument
   evaporates — which is why `io_uring` and XDP changed several of these debates.
3. **Policy vs mechanism** — the kernel provides mechanism; policy belongs in userspace
   because policy changes faster than the ABI can. A kernel policy decision is a permanent
   decision.
4. **Blast radius** — kernel code has no fault isolation and an unbreakable ABI. A bug is a
   panic or a privilege escalation, and a mistake in the interface is forever (Hyrum's Law).

Then note the third option that people forget: **eBPF** lets you put *userspace-authored
policy* at a kernel-speed location, which resolves the tension for a growing set of problems.
FUSE, VFIO/UIO, and io_uring are similar "userspace with kernel-provided fast path" answers.
→ Ch. 00 §T.4, Ch. 24, Ch. 75

### G2 [P] How would you design a new system call?

- **Does it need to be a syscall?** Prefer an existing extensible interface: an ioctl on an
  existing fd, a netlink family, a filesystem. A new syscall is 400+ lines of arch plumbing
  and is permanent.
- **Extensibility from day one**: pass a versioned struct with a size field and use
  `copy_struct_from_user()`, so unknown-future fields work in both directions
  (`clone3`, `openat2`, `sched_setattr` set the pattern). Never pass a bare flags int and
  hope.
- **Return an fd** for anything with a lifetime — fds are capabilities, they are
  refcounted, pollable, passable over `SCM_RIGHTS`, and closed automatically on exit.
  `pidfd`, `memfd`, `userfaultfd`, `io_uring` all do this.
- **Reject unknown flags with `-EINVAL`** so you can add flags later. Accepting-and-ignoring
  makes the flag unusable forever.
- **`*at()` form** — take a dirfd. Relative-path races are a whole CVE category.
- **32/64-bit compat**: no `long` in the ABI struct, explicit padding, no implicit holes
  (structleak will find them), `__u64` for pointers.
- **Think about the failure mode of partial success** — what does a partially-completed
  operation return?
- Then: **tests in `selftests/`, documentation in `man-pages`, and a real user.** The kernel
  will not merge an interface with no in-tree or announced consumer.

↪ *Follow-up:* "What is your biggest worry after it merges?" → Hyrum's Law: every observable
behaviour, including the ones I did not intend, is now depended upon. I would want to
constrain observability deliberately — do not expose what you are not prepared to keep.
→ Ch. 24

### G3 [P] You inherit a subsystem with 40% of its bugs in one driver family. What do you do?

Diagnose the *class* before fixing instances:

- Categorize the bugs. If they cluster on error paths → the answer is `devm_`/`guard()`/a
  helper that makes the correct thing automatic, not 40 individual fixes.
- If they cluster on concurrency → the locking model is wrong; look for a data-structure
  change (per-CPU, RCU) rather than more locks.
- If they cluster on hardware-specific quirks → the abstraction boundary is in the wrong
  place; consider a library (like `regmap` did for MMIO/I2C access, eliminating a whole class
  of bugs across hundreds of drivers).
- Add the missing *detection*: KASAN/KCSAN/UBSAN in CI, fuzzing with syzkaller, a
  `CONFIG_*_DEBUG` option that validates the API contract at runtime.
- Then: can I delete code? Converting N drivers onto a common framework usually removes more
  bugs than it introduces.

The principle is the one this curriculum repeats: **when a correctness obligation is
routinely forgotten, move it into the primitive.** → Ch. 28, Ch. 34, Ch. 50

### G4 [St] How do you evaluate whether an abstraction is worth adding?

Four questions:

1. **Does it have at least three real users?** Two is coincidence. Abstractions built for one
   caller encode that caller's assumptions.
2. **Does it remove more code than it adds?** Net-negative diffs are the strongest signal.
3. **Does it make an incorrect use impossible, or merely inconvenient?** Type-level
   guarantees (folio, `__must_check`, sparse `__user`) beat documentation.
4. **What is the escape hatch, and what does it cost?** Every abstraction leaks; the ones
   that survive have a clean way to drop down a level (regmap's raw accessors, DMA API's
   `dma_alloc_noncoherent`).

→ Ch. 26, Ch. 43

### G5 [P] Argue against a feature you think should not be merged.

The structure that works:

1. **Agree on the problem.** "You need X" — never dispute the need; disputing the need makes
   it personal.
2. **Name the cost precisely and in their currency.** Not "this is complex" but "this adds a
   lock to the fast path that every driver pays for, to serve one device."
3. **Offer the alternative**, and be honest about *its* cost.
4. **Say what would change your mind.** "If you show me three drivers that need this, I
   withdraw the objection." This converts an argument into a test.

The failure mode is arguing about taste. The recovery is always to find the measurable
statement underneath.

### G6 [St] Something is slow. Walk me through your approach.

USE, then split queueing from service time:

1. **Utilization, Saturation, Errors** for every resource: CPU (`mpstat`, `top`), memory
   (PSI, `vmstat`), disk (`iostat -x` — look at `%util` *and* `aqu-sz`), network
   (`ss -ti`, `ethtool -S`), and importantly the *software* resources: lock contention, tag
   depth, workqueue backlog.
2. Ask **queueing or service?** If `await` is high but `svctm` is low, you are queued —
   the fix is concurrency or admission control. If service time itself is high, the fix is
   the device or the work.
3. **Off-CPU vs on-CPU.** If CPU utilization is low and it is still slow, profiling on-CPU
   tells you nothing. Use off-CPU analysis (`offcputime`, `wakeuptime`) or blocked-time
   tracing.
4. **Then** profile: `perf record -g`, flame graph, and look for one dominant frame.
5. Always form a hypothesis with a **predicted measurement** before changing anything. "If
   this is lock contention, `perf lock` will show > X% and pinning to one NUMA node will
   improve it by Y."

The mistake candidates make is starting at step 4. → Ch. 06, `reference/debugging-scenarios.md`

---

## Part H — Rapid fire (60 seconds each)

| # | Question | Answer |
|---|---|---|
| 1 | Difference between `fork()` and `vfork()`? | `vfork` suspends the parent and shares the address space until exec; exists to avoid COW setup cost. Obsolete — use `posix_spawn`/`clone(CLONE_VM\|CLONE_VFORK)` |
| 2 | What is a zombie process? | Exited but not reaped; the `task_struct` persists to hold the exit status until the parent `wait()`s |
| 3 | What happens if the parent dies first? | Re-parented to `init` (or nearest subreaper, `PR_SET_CHILD_SUBREAPER`), which reaps |
| 4 | Why is `printf` in a signal handler unsafe? | Not async-signal-safe: it takes a lock on `stdout` that the interrupted code may hold |
| 5 | What is the difference between `exit()` and `_exit()`? | `exit()` runs atexit handlers and flushes stdio; `_exit()` is the raw syscall |
| 6 | Where does `strace` get its data? | `ptrace(PTRACE_SYSCALL)` — or `PTRACE_SEIZE` + seccomp for the faster modes. It roughly doubles syscall cost |
| 7 | How does `gdb` set a breakpoint? | Replaces the instruction with `int3` (0xCC); the trap delivers SIGTRAP to the tracer |
| 8 | What is the vDSO? | A kernel-provided shared object mapped into every process, so `gettimeofday`/`clock_gettime` read a shared page without a syscall |
| 9 | What is copy-on-write? | Shared mappings marked read-only; the write fault allocates and copies. Makes `fork` O(page tables) instead of O(memory) |
| 10 | What does `mmap(MAP_POPULATE)` do? | Prefaults the mapping, trading upfront cost for no later faults — use for latency-sensitive code |
| 11 | What is the OOM killer's heuristic? | `oom_score` from RSS + swap + page tables, adjusted by `oom_score_adj`; picks the largest offender in the constrained scope (cgroup or global) |
| 12 | What is `/proc/sys/vm/overcommit_memory`? | 0 heuristic, 1 always allow, 2 strict (never overcommit, guided by `overcommit_ratio`). Mode 2 turns OOM kills into honest `malloc` failures |
| 13 | What is a `kthread`? | A kernel-only task with no user address space (`mm == NULL`), schedulable like any other task |
| 14 | What does `wait_event_interruptible` actually do? | Adds to a wait queue, sets `TASK_INTERRUPTIBLE`, re-checks the condition, `schedule()`. The re-check before sleeping is the whole point — it closes the lost-wakeup race |
| 15 | What is a lost wakeup? | Waker signals between your condition check and your `schedule()`; you sleep forever. Prevented by setting the task state *before* the final check |
| 16 | What is `container_of`? | Given a pointer to a member, recover the enclosing struct via `offsetof` arithmetic. Enables intrusive data structures — no allocation, no type erasure |
| 17 | Why intrusive lists? | The node is embedded, so insert/remove never allocates and can never fail — essential in paths where allocation is forbidden |
| 18 | What is `ERR_PTR`/`IS_ERR`? | Encode small negative errnos in the top page of the address space, so a function can return either a pointer or an error without an out-param |
| 19 | What is `likely()`/`unlikely()`? | Branch hints to the compiler for layout. Only use where you have measured or where it is structurally obvious (error paths) |
| 20 | What does `__init` do? | Places the function in a section freed after boot. `__initdata` likewise. Saves a few hundred KiB |
| 21 | Why is `sleeping while atomic` fatal? | The scheduler would switch away while holding a spinlock with preemption/IRQs off — any other CPU taking that lock spins forever |
| 22 | What is `might_sleep()`? | A debug annotation that warns if called in atomic context; it is how the above bug is caught early rather than as a hang |
| 23 | Difference between `EXPORT_SYMBOL` and `EXPORT_SYMBOL_GPL`? | GPL-only symbols are refused to modules without a GPL-compatible license string. A licensing boundary, not a technical one |
| 24 | What is kABI and does Linux have one? | A stable binary interface for modules. Upstream Linux deliberately has **none**; distros synthesize one via whitelists and symbol checksums |
| 25 | Why does Linux have no stable in-kernel API? | Stated in `Documentation/process/stable-api-nonsense.rst`: it would freeze design mistakes forever and the maintenance cost would exceed the benefit. The *userspace* ABI, by contrast, is inviolable |

---

## Part I — Questions with no right answer (principal signal)

These test whether you can reason under genuine uncertainty. There is no model answer; what
is graded is whether you enumerate the forces and commit to a position with stated
conditions.

1. Should the kernel adopt Rust for all new drivers? Under what conditions would you say no?
2. Is the "no stable in-kernel API" policy still correct in 2025, given the maintenance burden
   on out-of-tree vendor code and the size of the Android kernel fork?
3. When is a microkernel the right answer? Name a workload where you would choose seL4 over
   Linux, and one where the reverse is obviously true.
4. eBPF is becoming a programmable kernel. Where should the line be — what should never be
   expressible in BPF?
5. Should `io_uring` be enabled by default given its CVE history? How would you decide?
6. Monolithic kernel, unikernel, or library OS for a 10,000-node fleet running one workload?
7. Is `CONFIG_PREEMPT_RT` in mainline a net win for non-RT users?
8. How much should the kernel sacrifice for Spectre-class mitigations? Who decides, and what
   would you measure to support the decision?
9. Given a fixed engineering budget, would you spend it on performance, on test
   infrastructure, or on deleting code?
10. Your company's product depends on a 200-patch out-of-tree kernel delta. Make the business
    case for upstreaming it — or for not.

---

## How to practise

1. **Cover the answer, speak aloud, time yourself.** Reading an answer creates recognition,
   not recall. Interviews test recall under mild stress.
2. **Always answer in three layers** (mechanism → forces → alternative and its cost). See
   `interview-playbook.md` §4.
3. **Answer the follow-up too.** The follow-up is where the level is decided.
4. For every **[S]** you get right, ask yourself the **[St]** version: *why does this exist,
   what does it cost, and what would I use instead?*
5. Maintain a list of the ones you fumbled and re-test at 1 day, 1 week, 1 month.

→ Next: [system-design.md](system-design.md)

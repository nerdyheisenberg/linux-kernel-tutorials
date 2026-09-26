# Debugging Scenarios — worked, out loud

> **What this round tests.** Not whether you know `gdb`. It tests whether you have a
> *procedure* — whether, given a wall of hostile evidence, you narrow the space
> systematically instead of guessing. Interviewers have watched many candidates stare at an
> oops and start reading it top to bottom. The ones who get hired read four specific fields
> first and say why.
>
> Each scenario below is presented as the interviewer presents it. Read the evidence, decide
> what you would say, *then* read the walkthrough.

---

## 0. The universal procedure

Say these steps out loud as you use them. Structure is graded as heavily as the answer.

1. **Classify.** Which of the six kinds is this?
   | Class | Signature |
   |---|---|
   | Crash | oops, panic, BUG |
   | Hang | task blocked, no progress, softlockup/hungtask |
   | Corruption | wrong data, no crash (or a crash far from the cause) |
   | Performance | correct but slow |
   | Latency | correct and fast on average, occasionally terrible |
   | Resource leak | degrades over hours/days |

   The class determines the tool. Reaching for `perf` on a corruption bug wastes an hour.

2. **Establish reproducibility.** Always? 1 in 1000? Only under load? Only on one machine?
   A reliable reproducer is worth more than any amount of log reading, so *spend effort
   building one* — say that explicitly.

3. **Bound the change.** What changed? Kernel version, config, hardware, workload, load
   level. If something changed, `git bisect` is usually faster than reasoning.

4. **Read the evidence in priority order.** For an oops that is: the reason line, the
   faulting address, `RIP`/`PC`, then the top of the stack. Not the register dump.

5. **Form a hypothesis with a predicted observation.** "If this is X, then enabling KASAN
   will report a use-after-free in `foo_free`." Then test it. One hypothesis at a time.

6. **Use the tool that turns silence into noise.** Most kernel bugs are silent until they
   are catastrophic. The debugging skill is knowing which `CONFIG_` or runtime knob converts
   the silent failure into an immediate, localized report.

### The conversion table — memorize this

| Silent failure | Turn it loud with |
|---|---|
| Use-after-free, out-of-bounds | `CONFIG_KASAN` |
| Uninitialized memory use | `CONFIG_KMSAN` |
| Data race | `CONFIG_KCSAN` |
| Undefined behaviour (shifts, overflow) | `CONFIG_UBSAN` |
| Lock-order inversion | `CONFIG_PROVE_LOCKING` (lockdep) |
| Sleeping in atomic | `CONFIG_DEBUG_ATOMIC_SLEEP` |
| RCU misuse | `CONFIG_PROVE_RCU` + `rcu_dereference_check` |
| Bad DMA API usage | `CONFIG_DMA_API_DEBUG` + IOMMU `strict` |
| Slab corruption | `CONFIG_SLUB_DEBUG_ON`, `slub_debug=FZP` |
| Use of freed pages | `CONFIG_DEBUG_PAGEALLOC`, `init_on_free=1` |
| List corruption | `CONFIG_DEBUG_LIST` |
| Stack overflow | `CONFIG_VMAP_STACK` (guard page → clean oops) |
| Refcount over/underflow | `CONFIG_REFCOUNT_FULL` / `refcount_t` (default now) |
| Memory leak | `CONFIG_DEBUG_KMEMLEAK` |
| Long non-preemptible section | `preemptirqsoff` tracer |
| Hardware/SMI latency | `hwlatdetect`, `osnoise` tracer |

**"I would rebuild with KASAN and lockdep" is a correct first answer to a large fraction of
kernel bugs, and saying it immediately is a positive signal — it shows you know that
reasoning about memory bugs by inspection is a losing game.**

---

## 1. Scenario: read the oops

> *Interviewer hands you this.*

```
BUG: kernel NULL pointer dereference, address: 0000000000000028
#PF: supervisor read access in kernel mode
#PF: error_code(0x0000) - not-present page
PGD 0 P4D 0
Oops: 0000 [#1] PREEMPT SMP NOPTI
CPU: 3 PID: 1842 Comm: kworker/3:1 Tainted: G     U  O      6.6.21 #1
Hardware name: QEMU Standard PC (Q35 + ICH9, 2009)
Workqueue: events mydrv_work_fn [mydrv]
RIP: 0010:mydrv_process+0x2f/0x180 [mydrv]
Code: 48 8b 7f 20 48 85 ff 74 3a 48 8b 47 28 <48> 8b 40 28 48 89 ...
RSP: 0018:ffffb3c1c0d7fe08 EFLAGS: 00010246
RAX: 0000000000000000 RBX: ffff9a2b04c1a800 RCX: 0000000000000000
...
Call Trace:
 <TASK>
 mydrv_work_fn+0x3c/0x90 [mydrv]
 process_one_work+0x1e2/0x3b0
 worker_thread+0x50/0x3a0
 kthread+0xe8/0x120
 ret_from_fork+0x34/0x50
 </TASK>
```

### What to read, in order

1. **The reason line.** "NULL pointer dereference, address 0x28". Not NULL itself — **offset
   0x28 from NULL**, which means the code dereferenced a NULL *struct pointer* and reached a
   member at offset 40. That is a much more specific fact than "null deref."
2. **`error_code(0x0000)`** — bit 0 clear = not-present, bit 1 clear = **read**, bit 2 clear =
   kernel mode. So: a kernel-mode read of an unmapped address. (Know these three bits;
   they are asked.)
3. **`RIP: mydrv_process+0x2f`** — the exact instruction. With the module, `faddr2line
   mydrv.ko mydrv_process+0x2f` gives file and line. Say that command name.
4. **`Tainted: G U O`** — `O` = out-of-tree module, `U` = userspace-defined taint. Always
   check taint; `P` (proprietary) or `D` (previous oops) changes how much you trust the
   report.
5. **Context: `Workqueue: events mydrv_work_fn`** — this is process context in a kworker, so
   sleeping was allowed and this is not an IRQ-context problem.
6. **`RAX: 0`** and the `Code:` line: `48 8b 47 28` is `mov rax,[rdi+0x28]`, then
   `<48> 8b 40 28` is `mov rax,[rax+0x28]` — the `<>` marks the faulting instruction. So the
   code loaded a pointer from `rdi+0x28`, got NULL, and dereferenced it. **The first load
   succeeded; the object exists, but one of its members is NULL.**

### The diagnosis you say out loud

"This is not `mydrv_process` being called with a NULL argument — `rdi` was valid, the first
load worked. A *member* at offset 0x28 of a valid object is NULL. The two usual causes are:
the work was queued before that member was initialized, or something tore it down
concurrently and set it to NULL. Given this is a workqueue item, I would look first at the
probe/remove path — specifically whether the work is queued before initialization completes,
or whether `remove()` NULLs the pointer without a `cancel_work_sync()`."

### What you would do next

```bash
# 1. Exact line
faddr2line mydrv.ko mydrv_process+0x2f
# or:  gdb -batch -ex 'list *(mydrv_process+0x2f)' mydrv.ko

# 2. Which member is at 0x28?
pahole -C mydrv_ctx mydrv.ko

# 3. Is this a lifetime bug? Turn it loud:
#    KASAN will catch it if the object was freed rather than NULLed.
```

And structurally: check that `remove()` does `cancel_work_sync()` **before** freeing or
NULLing anything the work touches, and that `probe()` queues work only as its last step.
This is the single most common driver bug shape. → Ch. 18, Ch. 28

---

## 2. Scenario: the hung task

```
INFO: task dbwriter:4412 blocked for more than 122 seconds.
      Tainted: G           O       6.6.21 #1
"echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
task:dbwriter  state:D stack:0  pid:4412 ppid:4001
Call Trace:
 __schedule+0x2d8/0x900
 schedule+0x5b/0xd0
 io_schedule+0x46/0x80
 folio_wait_bit_common+0x13a/0x330
 folio_wait_writeback+0x28/0x80
 ...
```

### Classify and read

State D = `TASK_UNINTERRUPTIBLE`. The hung-task detector only fires for D state, and D means
**waiting on something the kernel refuses to let a signal interrupt** — almost always I/O or
a mutex.

`io_schedule` + `folio_wait_writeback` says: waiting for a page's writeback to complete. So
this is not a locking deadlock; it is **I/O that never completed**.

### The fork in the road

"A blocked task is either (a) waiting for I/O that is stuck, or (b) waiting for a lock whose
holder is stuck. The stack tells me which. Here it is (a), so I stop looking at locks
entirely and start looking at the block layer and the device."

### The investigation

```bash
# Every blocked task and its stack — is dbwriter alone, or is everything stuck?
echo w > /proc/sysrq-trigger; dmesg

# Are there in-flight requests that never completed?
cat /sys/kernel/debug/block/nvme0n1/hctx*/busy
cat /sys/kernel/debug/block/nvme0n1/hctx*/tags

# Per-device I/O state; look at aqu-sz and await, not just %util
iostat -x 1

# Did the device report errors / timeouts?
dmesg | grep -iE 'nvme|timeout|reset|I/O error|medium'

# Is it actually the device, or are we throttled?
cat /sys/fs/cgroup/<cg>/io.stat
cat /proc/pressure/io
```

### The three answers this usually has

1. **Device is genuinely stuck** — firmware hang, a lost completion interrupt. Evidence:
   requests in `hctx*/busy` with no progress, eventual `nvme timeout` messages. The driver's
   timeout handler should reset; if it does not, that is the bug.
2. **cgroup throttling** — `io.max` set too low; the task is blocked *by policy*, and `io.stat`
   plus `/proc/pressure/io` shows it. This looks identical from the stack and is very
   commonly the real answer in containerized environments. **Mentioning it is a strong
   signal**, because it is the non-obvious one.
3. **Writeback deadlock** — the task is waiting for writeback that requires an allocation
   that requires reclaim that requires this writeback. Classic on memory-constrained systems;
   evidence is high `/proc/pressure/memory` alongside the I/O stall.

### The general lesson to state

"Hung task in D state is a *dependency chain* problem. My job is to walk the chain — A waits
on B waits on C — until I find something that is not waiting on anything, which is the actual
culprit. `echo w > /proc/sysrq-trigger` gives me every link at once." → Ch. 52, Ch. 63

---

## 3. Scenario: lockdep report

```
======================================================
WARNING: possible circular locking dependency detected
6.6.21 #1 Tainted: G           O
------------------------------------------------------
kworker/2:1/87 is trying to acquire lock:
 ffff9a2b1c0a8060 (&dev->config_lock){+.+.}-{3:3}, at: mydrv_reconfig+0x2a/0x120

but task is already holding lock:
 ffff9a2b1c0a8118 (&dev->io_lock){+.+.}-{3:3}, at: mydrv_work_fn+0x1e/0x90

which lock already depends on the other.
...
 Possible unsafe locking scenario:

       CPU0                    CPU1
       ----                    ----
  lock(&dev->io_lock);
                               lock(&dev->config_lock);
                               lock(&dev->io_lock);
  lock(&dev->config_lock);

 *** DEADLOCK ***
```

### The single most important thing to say first

**"This is not a deadlock that happened. Lockdep observed both orders at different times and
is reporting that the deadlock is *possible*."** That is the whole value of lockdep and the
thing candidates most often get wrong. It found a latent bug that might have taken years to
manifest.

Also: it fires **once**. After the first report lockdep disables itself, so the first report
is the only one you get — do not dismiss it and keep testing.

### How to read it

- The `{+.+.}` is the usage mask: acquired in process context, acquired with softirqs
  enabled, etc. A `-` vs `+` in the IRQ positions tells you whether an IRQ-context deadlock
  is also in play (a different and more urgent class).
- `{3:3}` is the lock class's position in the dependency graph.
- The "Possible unsafe locking scenario" block is lockdep telling you the exact interleaving.
  Read that, not the raw stacks.

### The fix

Pick a global order and enforce it everywhere: `config_lock` before `io_lock`, or the
reverse. Document it in a comment at the struct definition — this is one of the few places a
comment is genuinely load-bearing:

```c
struct mydrv_dev {
	/* Lock order: config_lock -> io_lock. Never the reverse. */
	struct mutex config_lock;
	struct mutex io_lock;
	...
};
```

Then ask the better question: **why are there two locks?** Two locks on the same object with
an ordering constraint is often one lock that was split prematurely. If `config_lock` is
rarely taken, merging them removes the bug class entirely. That is the staff-level answer.

### The three lockdep report types you should recognize

| Report | Meaning |
|---|---|
| "possible circular locking dependency" | A→B and B→A observed. Fix with ordering |
| "inconsistent {SOFTIRQ-ON-W} -> {IN-SOFTIRQ-W} usage" | Lock taken both with and without softirqs disabled → use `spin_lock_bh` consistently |
| "possible irq lock inversion dependency" | Same, for hard IRQs → `spin_lock_irqsave` |

The latter two are about **context**, not order, and the fix is a consistent lock *variant*,
not a consistent order. Distinguishing them is a real signal. → Ch. 14 §T.7

---

## 4. Scenario: memory corruption with no obvious cause

> *"A long-running service gets random crashes in unrelated code — sometimes slab, sometimes
> a list, sometimes userspace. Never the same place twice."*

### Classify

Random crashes in unrelated code = **memory corruption**, near-certainly. The crash site is
the *victim*, not the culprit; reading the crash stacks is nearly worthless, which you
should say immediately to avoid spending the round there.

### The procedure

1. **Make it deterministic.** Boot with KASAN. KASAN turns "crash somewhere later" into
   "report at the exact instruction, with the allocation and free stacks of the object." This
   is the single highest-value move and should be your first sentence.

```bash
# .config
CONFIG_KASAN=y
CONFIG_KASAN_GENERIC=y        # or CONFIG_KASAN_SW_TAGS on arm64 (cheaper)
CONFIG_KASAN_INLINE=y         # faster instrumentation
CONFIG_SLUB_DEBUG_ON=y
CONFIG_DEBUG_LIST=y
CONFIG_DEBUG_OBJECTS=y        # catches timer/workqueue reuse-after-free
# boot:
init_on_alloc=1 init_on_free=1 slub_debug=FZPU page_poison=1
```

2. **If you cannot run KASAN** (performance, production, hardware): `slub_debug=FZP` gives
   redzones and poison at ~10% cost, and `CONFIG_DEBUG_PAGEALLOC` catches use-after-free of
   whole pages. On arm64, **MTE** (`kasan.mode=async`) is cheap enough for production.

3. **Read a KASAN report properly.** The three stacks are:
   - The **access** stack — who touched it (the victim).
   - The **allocated by** stack — where the object came from.
   - The **freed by** stack — **this is the culprit**. Read it first.

4. **If the free stack looks legitimate**, you have a refcount bug, not a free bug — someone
   dropped a reference they did not own, or failed to take one. Look at `refcount_t` warnings
   in the log preceding the crash; a `refcount_t: underflow` warning minutes earlier is the
   real report.

5. **If KASAN finds nothing**, suspect:
   - **DMA**: the device wrote into memory the CPU had reclaimed. KASAN cannot see device
     writes. Turn on `CONFIG_DMA_API_DEBUG` and put the IOMMU in `strict` mode —
     `iommu.strict=1 iommu.passthrough=0`. A device DMAing to a stale address then faults
     visibly instead of corrupting silently. **This is the answer KASAN cannot give and is a
     strong differentiator to mention.**
   - **Stack overflow**: `CONFIG_VMAP_STACK` converts it into a clean guard-page oops.
   - **Hardware**: ECC errors (`edac-util`, `rasdaemon`). Rare, but if it is one machine
     only, test it.

### The classic shapes

| Shape | Evidence | Cause |
|---|---|---|
| UAF | free stack in `remove`/`release` | missing `cancel_work_sync`/`del_timer_sync`, or `devm_` on an object outliving the device |
| Double free | two free stacks | error path frees, then caller frees |
| Slab OOB | redzone overwrite | off-by-one, or `sizeof` of the pointer not the struct |
| Refcount | `refcount_t` warning earlier | `get` missing on a path that stores the pointer |
| DMA | KASAN silent, IOMMU fault with strict | buffer freed before `dma_unmap`, or device still writing |

→ Ch. 12, Ch. 28, Ch. 35, Ch. 06

---

## 5. Scenario: p99 latency regression after a kernel upgrade

> *"We went from 5.15 to 6.6. Mean latency is 3% better. p99.9 went from 2 ms to 40 ms."*

### The framing that scores

"A mean improvement with a tail regression means it is not a throughput problem — something
is now *occasionally* blocking. Tail latency is almost always one of: a lock held across a
sleep, reclaim, an unexpected preemption, a throttling boundary, or a migration. I would not
profile on-CPU first, because a 40 ms stall is almost certainly off-CPU."

### The measurement

```bash
# 1. Is it off-CPU? Where?
offcputime-bpfcc -f -p $(pidof svc) 30 > off.stacks   # then flame graph
# 2. Scheduler latency specifically
perf sched latency -s max
# 3. Per-syscall tail, cheap and immediately informative
funclatency-bpfcc -m 'do_sys_openat2'   # or whatever dominates
# 4. Is it reclaim?
cat /proc/pressure/{cpu,memory,io}
# 5. Is it cgroup throttling? nr_throttled and throttled_usec:
cat /sys/fs/cgroup/<cg>/cpu.stat
```

### The candidates, ranked by base rate for this exact symptom

1. **cgroup CPU bandwidth throttling.** `cpu.max` with a 100 ms period: a burst exhausts the
   quota and the task is stopped until the period boundary — producing stalls quantized to
   ~tens of ms. **40 ms is suspiciously period-shaped.** Check `cpu.stat`'s `nr_throttled`
   and `throttled_usec`. This is the highest-probability answer and you should name it first.
2. **Memory reclaim / compaction.** Direct reclaim in the allocation path, or
   `khugepaged`/compaction stalls. Check `/proc/pressure/memory` and `vmstat`'s
   `allocstall_*` and `compact_stall`.
3. **A scheduler behaviour change.** 6.6 introduced EEVDF. Wakeup preemption and timeslice
   behaviour changed; a workload tuned to CFS's heuristics can regress. Test by pinning, by
   `SCHED_BATCH`, or by adjusting `sched_latency`-equivalent knobs.
4. **Writeback / fsync**. A periodic flusher stalling the writer.
5. **NUMA balancing** migrating pages (and therefore stalling on migration faults). Test with
   `numa_balancing=0`.
6. **THP allocation stalls** — `defrag=always` causes synchronous compaction. Set
   `defrag=defer+madvise`.

### The discipline to demonstrate

Change **one** variable, and predict the result before measuring. "If it is (1), setting
`cpu.max=max` removes the tail entirely and `nr_throttled` stops incrementing. If the tail
persists, I have eliminated it." A candidate who changes three sysctls at once and reports
improvement has learned nothing.

And: **`git bisect` is available.** 5.15→6.6 is a large range, but bisecting a
reproducible p99 regression over ~15 builds is often faster than reasoning, and saying you
would do it shows you value evidence over cleverness.

→ Ch. 21, Ch. 23, Ch. 06

---

## 6. Scenario: softlockup / RCU stall

```
watchdog: BUG: soft lockup - CPU#5 stuck for 23s! [swapper/5:0]
...
rcu: INFO: rcu_preempt detected stalls on CPUs/tasks:
rcu:     5-...0: (1 GPs behind) idle=b2c/1/0x4000000000000000 softirq=1234/1235 fqs=2601
rcu:     (detected by 2, t=15002 jiffies, g=8841, q=19)
```

### What each means — know the distinction

| Detector | Fires when | Implies |
|---|---|---|
| **Soft lockup** | A CPU did not schedule for `watchdog_thresh`×2 (default 20 s) | A loop in kernel mode with preemption disabled, or interrupts on but never yielding |
| **Hard lockup** | A CPU did not take the NMI/perf watchdog interrupt | **Interrupts disabled** in a loop, or a hardware hang. Much more serious |
| **RCU stall** | A CPU has not reported a quiescent state for ~21 s | The CPU is stuck in an RCU read-side section, or stuck with preemption off, or `rcu_nocbs` offload is wedged |
| **Hung task** | A task in D state for 120 s | Waiting, not spinning — a *different* class entirely (see §2) |

**Softlockup and RCU stall mean spinning; hung task means blocked.** Saying that distinction
first tells the interviewer you know which half of the tool space to use.

### The investigation

The report includes the stuck CPU's stack (or you can get it with `echo l >
/proc/sysrq-trigger` for all CPUs). Then:

1. **Is it an infinite loop?** Read the stack. A loop with a bad exit condition is the simple
   case.
2. **Is it lock contention so severe it looks like a hang?** A spinlock held for seconds by
   another CPU. `echo l > /proc/sysrq-trigger` shows all CPUs — look for another CPU inside
   the same lock's critical section.
3. **Is it an interrupt storm?** A device asserting a level-triggered IRQ that is never
   cleared: the handler returns, the IRQ fires again, forever. `/proc/interrupts` sampled
   twice shows a count climbing by millions. The kernel may print `nobody cared` and disable
   the IRQ — that message is the giveaway.
4. **Is it a very long legitimate operation?** Walking a huge list, zeroing a huge allocation,
   `stop_machine` on a large system. Fix: add `cond_resched()` — which is exactly what these
   detectors exist to force you to do.
5. **RCU stall specifically:** a long RCU read-side section is a bug (they must be short), but
   the more common cause is a CPU stuck for another reason and RCU is merely the messenger.
   Check whether a softlockup fired on the same CPU — if so, chase that.

### The boot options that make the next one easier

```
nmi_watchdog=1 softlockup_panic=1 hung_task_panic=1 panic_on_rcu_stall=1
panic=1 crashkernel=256M
```

Panic-and-dump turns an unreproducible stall into a `vmcore` you can open in `crash`. In
production, this is usually the right configuration — a 30-second reboot with a dump beats a
wedged machine with no evidence. Saying this shows operational judgement. → Ch. 06

---

## 7. Scenario: it works on my machine

> *"Driver works on the dev board, corrupts data on the customer's board. Same kernel, same
> driver."*

### The differences worth enumerating, in order

1. **Cache coherency.** Is the device coherent with the CPU on one board and not the other?
   `dma-coherent` in DT, or the ACPI `_CCA` attribute. A driver that works on a coherent
   platform and skips `dma_sync_*` calls breaks silently on a non-coherent one. **This is the
   number one answer** and should be your opening.
2. **IOMMU present/absent.** With an IOMMU, a bad DMA address faults; without one, it
   silently corrupts random memory. So "works on the board with the IOMMU" can mean the
   driver has a latent bug being *masked* — the fault would have been visible.
3. **Endianness / register width.** A big-endian SoC, or a bus that requires 32-bit accesses
   where you used `writeb`.
4. **Memory layout.** DMA mask too small for the customer's higher memory map, causing
   bounce-buffering (swiotlb) — which then exposes missing `dma_sync` calls, again.
5. **Timing.** Faster CPU or slower bus changes a race's outcome. A missing barrier is
   invisible on one and fatal on the other.
6. **Cache line size.** 64 vs 128 bytes; a buffer that was accidentally aligned on one
   platform is not on the other, so a cache invalidate destroys an adjacent structure.
7. **Firmware/clock/regulator differences** — a clock at a different rate, a regulator not
   enabled because a different PMIC driver is in play.
8. **Config differences** — `CONFIG_PREEMPT` vs `CONFIG_PREEMPT_RT` vs `PREEMPT_NONE` changes
   every timing assumption.

### The move that resolves it

"I would stop diffing the boards and instead make the dev board *behave like* the customer's:
force non-coherent DMA (`dma-noncoherent` or by allocating with
`dma_alloc_noncoherent`), force swiotlb (`swiotlb=force`), force the IOMMU off, and enable
`CONFIG_DMA_API_DEBUG`. If any of those reproduces it on my bench, I have the answer in an
hour instead of a week."

**Turning the failing environment into a local reproducer is almost always faster than
remote debugging, and proposing it is a strong senior signal.** → Ch. 35, Ch. 34, Ch. 32

---

## 8. Scenario: a leak over days

> *"Memory use climbs 200 MB/day. Never OOMs in testing because testing runs for an hour."*

### Narrow the location first

```bash
# 1. Is it userspace or kernel? Compare total to the sum of process RSS.
free -m; ps -eo rss= | awk '{s+=$1} END {print s/1024" MB"}'
# 2. If kernel: which allocator?
cat /proc/meminfo     # Slab, SUnreclaim, KernelStack, PageTables, VmallocUsed
slabtop -o -s c       # which cache is growing?
cat /proc/vmallocinfo | sort -k2 -n | tail
# 3. Kernel object leak with stacks:
echo scan > /sys/kernel/debug/kmemleak; cat /sys/kernel/debug/kmemleak
# 4. Per-allocation-site accounting (very useful, often forgotten):
cat /proc/allocinfo | sort -k1 -n | tail -20    # needs CONFIG_MEM_ALLOC_PROFILING
```

The decision tree:
- `SUnreclaim` growing → slab leak; `slabtop` names the cache, which names the subsystem.
- `VmallocUsed` growing → a `vmalloc`/module/stack leak.
- `KernelStack` growing → thread leak.
- `PageTables` growing → a process mapping without unmapping, or a fork leak.
- None of the above growing but `MemFree` shrinking → page leak or fragmentation; check
  `/proc/buddyinfo` for order-0 pile-up.
- Everything accounted for and still shrinking → **not a leak**: it is page cache doing its
  job, and the reporter is reading `free` wrong. Check `MemAvailable`, not `MemFree`. Say
  this — a meaningful fraction of "leaks" are this.

### Kmemleak's caveat, worth mentioning

It is a conservative scanner: it reports objects with no pointer to them anywhere in scanned
memory. It **false-negatives** on pointers stored with the low bits used as tags, in
registers, or obfuscated — and it false-positives on legitimately unreferenced-but-tracked
allocations. Treat it as a strong hint, not proof. `CONFIG_MEM_ALLOC_PROFILING`
(`/proc/allocinfo`, 6.10+) is often better because it gives you per-call-site totals with
no guessing at all.

### The structural causes

| Growing | Usual cause |
|---|---|
| A driver's slab cache | error path that does not free; or an object freed only on a success path |
| `dentry`/`inode` | a workload creating files; usually not a leak — reclaimable. Check with `echo 2 > drop_caches` |
| `kmalloc-*` generic caches | hard; needs kmemleak or allocinfo |
| Network buffers | skb leak, or a socket leak (`ss -s`, `/proc/net/sockstat`) |
| `KernelStack` | `kthread` created per-event and never stopped |

→ Ch. 11, Ch. 23, Ch. 06

---

## 9. Scenario: reproduce it before you fix it

> *"The bug happens once a week in a fleet of 5000. You cannot reproduce it."*

This is the question that separates people who have shipped from people who have not. The
answer is **not** "read the code harder."

### The moves, in order of value

1. **Make the failure cheaper to observe.** Deploy the panic-and-dump configuration (§6) to a
   subset of the fleet. One `vmcore` is worth a hundred hypotheses.
2. **Add instrumentation that costs nothing when idle.** Tracepoints + a `bpftrace` script
   deployed fleet-wide, triggering a dump of relevant state *when the precondition is
   detected*, not after the crash. You are converting a post-mortem into a live capture.
3. **Widen the aperture.** Enable `CONFIG_DEBUG_*` on 1% of the fleet. A 5% throughput cost
   on 50 machines is cheap; a week per bug is not.
4. **Compress the timeline.** What makes it once-a-week? If it is a rare interleaving, add
   artificial delay at the suspected point (`CONFIG_FAULT_INJECTION`, `msleep` under a
   debugfs knob, `ftrace` with a stall trigger) to make the window enormous. Racing code that
   loses a 100 ns race weekly loses a 100 ms race instantly.
5. **Fault injection.** `failslab`, `fail_page_alloc`, `fail_make_request` — most rare bugs
   are error paths that are never exercised. Injecting failure turns a 1-in-10⁹ path into a
   1-in-10 path. This is the single most productive technique for "rare" bugs and is
   badly under-used.
6. **Fuzz it.** `syzkaller` with a focused config on the subsystem. If the interface is
   reachable from userspace, syzkaller will find things you will not.
7. **Stress the specific dimension.** `stress-ng`, `trinity`, memory pressure via a cgroup,
   CPU hotplug in a loop (`torture` tests), module load/unload in a loop. Many rare bugs are
   teardown bugs and only teardown stress finds them.
8. **Correlate across the fleet.** 5000 machines is a *dataset*. What do the failing ones
   share — kernel build, hardware revision, firmware version, workload mix, uptime, NUMA
   topology? An hour with the fleet data often beats a week with the code.

### The sentence to say

"My first goal is not to fix it, it is to reproduce it, because an unreproducible fix is
indistinguishable from no fix. I would spend the first two days building a reproducer, using
fault injection and artificial delays to compress a weekly event into a per-minute one."

→ Ch. 50, Ch. 06

---

## 10. The tool inventory

Organized by what you need, not alphabetically — this is the form you want it in under
pressure.

| Need | Tool |
|---|---|
| Where is it spending CPU? | `perf record -g` + flame graph |
| Where is it *not* on CPU? | `offcputime-bpfcc`, `wakeuptime-bpfcc` |
| Which cacheline is hot? | `perf c2c` |
| Lock contention | `perf lock`, `/proc/lock_stat` (`CONFIG_LOCK_STAT`) |
| Latency distribution of anything | `funclatency-bpfcc`, `bpftrace` hist |
| What is the kernel doing right now? | `echo l > sysrq-trigger`, `echo w`, `echo t` |
| Trace a subsystem | `trace-cmd record -e 'block:*'`, `perf trace` |
| Ad-hoc kernel instrumentation | `bpftrace`, kprobes/fprobes |
| Post-mortem | `crash` on a `vmcore` (kdump), `drgn` (prefer `drgn` — Python, scriptable, works live too) |
| Live kernel inspection | `drgn`, `/proc/kcore` + gdb |
| Source-level debugging | KGDB over serial, or QEMU + `gdb -ex 'target remote :1234'` |
| Memory bugs | KASAN, KFENCE (production-safe sampling!), kmemleak |
| Races | KCSAN |
| Static analysis | `sparse`, `smatch`, `coccinelle`, `clang-analyzer` |
| Build-time struct layout | `pahole` |
| Address → line | `faddr2line`, `decode_stacktrace.sh` |
| Fleet-safe memory-bug detection | **KFENCE** — ~0% overhead sampling allocator, designed to run in production |

**`drgn` and `KFENCE` are the two most under-known tools on this list.** Mentioning KFENCE
in response to "how do you find memory bugs in production, where KASAN is too slow" is a
notably strong answer.

---

## 11. The meta-answers

Have these ready; they are asked directly.

**"What do you do first when given a bug?"**
> Classify it, then determine reproducibility. Those two facts determine every subsequent
> decision, and answering them takes minutes while guessing wrong costs days.

**"How do you debug something you can't reproduce?"**
> I invest in reproducing it — fault injection, artificial delays, stress on the specific
> dimension, and fleet-wide correlation. If I truly cannot, I add instrumentation that
> captures state at the moment of failure and wait, rather than reasoning from a post-mortem.

**"What's the hardest bug you've debugged?"**
> Have a real one prepared, told as: symptom → what made it hard (the misleading evidence) →
> the move that broke it open → the systemic fix. The last part matters most: what did you
> change so this *class* of bug could not recur?

**"When do you give up on debugging and rewrite?"**
> When the bug density per KLoC in a module is an order of magnitude above the rest of the
> subsystem, the bugs cluster in one dimension (error paths, locking), and the module has no
> tests. Then the bugs are a symptom of the structure and fixing them individually is
> treading water.

**"How do you know your fix is correct?"**
> A test that fails before and passes after; an explanation of the mechanism that accounts
> for *all* the observed symptoms, not just the crash; and a check for the same pattern
> elsewhere in the tree — because if I made this mistake, so did someone else.

→ Next: [../README.md](../README.md)

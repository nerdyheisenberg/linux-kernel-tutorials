# Chapter 103 — Real-Time Linux (PREEMPT_RT)

> You have met real time in fragments: `rt_mutex` and priority inheritance in Ch. 14,
> threaded IRQs in Ch. 17, `hrtimer` in Ch. 19, `SCHED_DEADLINE` in Ch. 21, RCU's
> preemptible readers in Ch. 15. This chapter assembles them into a single discipline,
> because real time is not a feature you enable — it is a property of the whole system that
> any one component can destroy.

---

## Theory & First Principles

### T.0 — Start here: a system that is fast on average and useless

Two systems respond to an external event:

```
   System A:   average  10 us    p99   15 us    WORST  50 ms
   System B:   average 200 us    p99  210 us    WORST 250 us
```

**System A is twenty times faster and completely unusable** for a motor controller, a defibrillator,
or a robot arm. **System B is correct.** If your control loop runs at 1 kHz and one iteration takes
50 ms, you have missed fifty deadlines, and depending on what is attached, something has crashed
into something else.

> **Real time means *bounded*, not *fast*. It is a statement about the worst case, and only
> about the worst case. Throughput and average latency are irrelevant — and the trade is
> usually that you give up some of both to get the bound.**

**So: where does the 50 ms come from in a normal kernel?** There are exactly four sources of
unbounded delay, and `PREEMPT_RT` is precisely the project of eliminating each:

| Source | Why it is unbounded | What RT does |
|---|---|---|
| **Long spinlock sections** | preemption is disabled inside them; a long one delays everything | convert most `spinlock_t` to **sleeping rt_mutexes** |
| **Interrupt handlers** | run before any task, for as long as they want | move handlers into **schedulable kernel threads** |
| **Softirqs / tasklets** | run after IRQs, unbounded work | made preemptible, accounted to threads |
| **Priority inversion** | a low-priority task holding a lock a high-priority task needs | **priority inheritance** on the rt_mutex |

**Priority inversion deserves the space**, because it is the classic and it has a famous
victim:

```
  LOW  task L  takes lock X
  MED  task M  becomes runnable and PREEMPTS L (it is higher priority)
  HIGH task H  wants lock X -- blocked on L, which is not running,
               because M is running.

  -> H, the highest-priority task in the system, waits on M, an UNRELATED
     medium-priority task, for an UNBOUNDED time.
```

This is what repeatedly reset the **Mars Pathfinder** in 1997. The fix, **priority
inheritance**, is elegant: while L holds a lock that H wants, L temporarily *inherits* H's
priority, so M cannot preempt it. **A lock is not only a mutual-exclusion device — it is a
scheduling relationship**, and a scheduler that does not know about locks cannot bound
anything.

**Now the trade, stated plainly, because RT is not free:**

| `PREEMPT_RT` buys | and costs |
|---|---|
| bounded worst-case latency (tens of µs) | **lower throughput** — typically 10–30% |
| every kernel path preemptible | more context switches, more cache pollution |
| predictability | some code paths (a few raw spinlocks) must stay non-preemptible |

**You are deliberately making the average case worse to make the worst case bounded.** That
is the definition of the field, and if someone asks for "real-time performance" without
naming a deadline, they do not yet know what they are asking for.

**Three things people forget, which will ruin an RT system regardless of the kernel:**

1. **SMIs.** System Management Interrupts are invisible to the OS, cannot be masked, and can
   last hundreds of microseconds. `hwlatdetect` exists to find them. **Firmware can violate
   your latency budget and the kernel cannot stop it.**
2. **Power management and frequency scaling.** Exiting a deep C-state takes tens of µs. RT
   systems disable idle states and fix the frequency — trading a great deal of power for
   determinism.
3. **Page faults.** Your RT thread must `mlockall()` and pre-fault its stack and heap; one
   major fault is milliseconds (Ch. 22 §T.0). **The deadline is missed in userspace, by
   demand paging, not by the kernel.**

**And `SCHED_DEADLINE` is worth knowing as the theoretically right answer:** instead of
priorities (which are a *relative* statement and compose badly), you declare
`(runtime, deadline, period)` — an *absolute* requirement — and the kernel runs an admission
test and refuses tasks it cannot schedule. **Declare the requirement, let the system verify
feasibility** — far better engineering than tuning priority numbers until it seems to work.

```bash
uname -v | grep -i preempt_rt
sudo cyclictest -m -S -p95 -i200 -h400 -D 60      # the standard latency benchmark
sudo hwlatdetect --duration=60                    # find SMIs
cat /sys/kernel/debug/tracing/available_tracers   # wakeup_rt, irqsoff, preemptoff
chrt -d --sched-runtime 1000000 --sched-deadline 10000000 --sched-period 10000000 -- ./app
```

---

### T.1 — Real time means *bounded*, not *fast*

The single most common misunderstanding, and the first thing to say in an interview:

> A real-time system is one whose correctness depends on **when** the answer is produced,
> not only on what the answer is. The engineering goal is a **provable upper bound on
> latency**, not a low average.

A system with 1 µs average latency and an unbounded tail is **not** real time. A system with
500 µs latency that never, under any circumstance, exceeds 500 µs **is**. Throughput-oriented
Linux optimizes the mean; RT Linux optimizes the maximum, and **deliberately sacrifices
throughput to do it** (typically 5–30%).

| Class | Consequence of a miss | Examples |
|---|---|---|
| **Hard** | System failure; possibly physical harm | flight control, motor commutation, airbag |
| **Firm** | Result is worthless but harmless | video frame, sensor fusion sample |
| **Soft** | Value degrades with lateness | audio, video playback, UI |

Linux with `PREEMPT_RT` reliably serves firm and soft real time and a large fraction of hard
real time with deadlines above ~50 µs. Below that, or where *certification* is required, you
want a dedicated RTOS or a co-processor — and saying so is a mark of judgement, not defeat.

### T.2 — The latency budget: decompose it

```
 event at device
   │  (1) hardware/interrupt-controller delivery
   ▼
 CPU takes the interrupt
   │  (2) interrupt DISABLED time — the top of the budget
   ▼
 IRQ handler runs
   │  (3) handler duration
   ▼
 wakes the RT thread
   │  (4) scheduling latency: preemption disabled? higher-prio task? migration?
   ▼
 RT thread runs
   │  (5) execution: page faults, cache misses, lock waits
   ▼
 response
```

**Worst-case latency is the sum of the worst case of each term**, not the sum of the
averages. Each term has a distinct mitigation:

| Term | Dominant cause | Mitigation |
|---|---|---|
| (1) | SMI/SMM, firmware, interrupt controller | `hwlatdetect`; firmware settings; nothing in software |
| (2) | Long `local_irq_disable()` regions | `PREEMPT_RT` shortens them; `irqsoff` tracer finds them |
| (3) | Work done in hard IRQ | Threaded IRQs move it to a schedulable context |
| (4) | Non-preemptible kernel code; another RT task | `PREEMPT_RT`; priority design; CPU isolation |
| (5) | Major faults, allocation, lock contention | `mlockall`, prefault, no allocation, PI mutexes |

The discipline this implies: **you cannot bound what you do not measure per-term.** The
`osnoise` and `timerlat` tracers exist specifically to attribute an outlier to one of these.

### T.3 — The four preemption models

```
 CONFIG_PREEMPT_NONE      Throughput.  Preempt only at explicit points
                          (syscall return, cond_resched). Latency: ~ms.

 CONFIG_PREEMPT_VOLUNTARY Adds might_sleep() preemption points.
                          Latency: ~hundreds of µs. Desktop distro default (historically).

 CONFIG_PREEMPT           Kernel is preemptible except in critical sections
                          (spinlocks, IRQ-disabled, preempt_disable).
                          Latency: ~tens to hundreds of µs.

 CONFIG_PREEMPT_RT        Nearly everything is preemptible. Spinlocks become
                          sleeping rt_mutexes; IRQs become threads; softirqs
                          become threads. Latency: single-digit to tens of µs.
```

Since 6.12 there is also **`CONFIG_PREEMPT_LAZY`** and the `preempt=` boot-time selection
work, which lets one kernel binary behave as NONE/VOLUNTARY/FULL — the long-running effort
to stop shipping four different kernels. This is live mainline work worth tracking.

`PREEMPT_RT` was merged into mainline in **6.12** after roughly twenty years out of tree — a
useful piece of history: the reason it took so long is that making the kernel preemptible
everywhere required *auditing every assumption about atomicity in the entire tree*, and most
of that work landed as generic improvements (threaded IRQs, `hrtimer`, generic locking,
lockdep, `local_lock`) that non-RT users have been benefiting from for a decade.

### T.4 — What `PREEMPT_RT` actually changes

This is the heart of the chapter. Five transformations:

**1. Spinlocks become sleeping locks.** `spinlock_t` and `rwlock_t` are mapped onto
`rt_mutex`. A task holding one is **preemptible**, and a task waiting on one **sleeps**
rather than spinning. This is what removes the largest source of non-preemptible time.

Consequence: code that relies on `spin_lock()` implying `preempt_disable()` — for example
to protect per-CPU data — **breaks**. Hence `raw_spinlock_t`, which remains a true spinning,
preemption-disabling lock, used in the scheduler, the timer core, and other places where
sleeping is impossible. Converting the tree meant correctly classifying thousands of locks
into "can sleep" and "cannot", which is the bulk of the twenty years.

Also consequence: `local_lock_t` was invented, because "disable preemption to protect
per-CPU data" needs an explicit primitive once `spin_lock` no longer does it. → Ch. 16.

**2. Interrupt handlers become threads.** Every `request_irq()` handler runs in a kthread
(`irq/NN-name`) at `SCHED_FIFO` 50 by default, unless flagged `IRQF_NO_THREAD`. The hard-IRQ
portion shrinks to a stub that masks the interrupt and wakes the thread. This makes IRQ
handling **schedulable**: you can prioritize it, pin it, and preempt it. A high-priority RT
task can now preempt a network interrupt, which on a non-RT kernel is impossible.

**3. Softirqs become threads.** Same reasoning. On `PREEMPT_RT` softirqs run in per-CPU
`ksoftirqd` context with a priority, rather than in a non-preemptible section on IRQ return.

**4. `rt_mutex` with full priority inheritance everywhere.** Since every lock is now an
`rt_mutex`, PI applies universally, and the PI chain walk can be deep and transitive.

**5. Per-CPU and RCU adjustments.** RCU read-side sections become preemptible
(`CONFIG_PREEMPT_RCU`), callbacks can be offloaded (`rcu_nocbs`), and
`local_bh_disable()` no longer disables preemption.

**What does *not* change:** `raw_spinlock_t`, `preempt_disable()`, `local_irq_disable()`,
NMI context. Those remain the bounded, non-preemptible core — and the size of the largest
such region is now your latency floor.

### T.5 — Priority inversion and the protocols that bound it

```
 t0: L (prio 10) acquires lock X
 t1: H (prio 90) blocks on X                 -- correct, bounded so far
 t2: M (prio 50) becomes runnable, preempts L
 t3: ...M runs for as long as it likes.
     H is now blocked behind M, which it outranks. UNBOUNDED inversion.
```

**Priority inheritance:** at t1, L's priority is boosted to 90 for as long as it holds X. M
cannot preempt it. The inversion is now bounded by the length of L's critical section. The
boost must be **transitive** — if L is itself blocked on a lock held by K, K must be boosted
too, which is the "PI chain walk" and is the source of most of `rt_mutex`'s complexity.

**Priority ceiling:** each lock carries a ceiling = the maximum priority of any task that
may acquire it; a holder immediately runs at the ceiling. This bounds inversion **and
prevents deadlock** (a task can only block on a lock whose ceiling exceeds its current
priority, which makes circular waits impossible) and needs no chain walk. Its cost is that
the ceiling must be known statically, which does not fit a general-purpose kernel — hence
Linux uses PI.

Historical anchor worth knowing: **Mars Pathfinder, 1997.** A high-priority bus-management
task blocked on a mutex held by a low-priority meteorological task, which was preempted by a
medium-priority communications task. The watchdog reset the spacecraft repeatedly. The fix,
uplinked to Mars, was a single flag enabling priority inheritance in VxWorks. It is the
canonical story and interviewers do use it.

**Userspace** gets the same via `FUTEX_LOCK_PI` / `pthread_mutexattr_setprotocol(...,
PTHREAD_PRIO_INHERIT)`. If your RT thread takes a pthread mutex without PI, you have
reintroduced the problem.

### T.6 — Scheduling for real time

| Policy | Model | Use |
|---|---|---|
| `SCHED_FIFO` | Fixed priority 1–99, runs until it blocks or yields | Aperiodic RT work |
| `SCHED_RR` | Same, plus a quantum among equals | Rare; FIFO with a tiebreak |
| `SCHED_DEADLINE` | EDF + Constant Bandwidth Server | **Periodic work with known WCET** |

`SCHED_DEADLINE` is the one to understand deeply, because it is the only policy that offers
a *guarantee* rather than a *priority*:

```c
struct sched_attr attr = {
	.size           = sizeof(attr),
	.sched_policy   = SCHED_DEADLINE,
	.sched_runtime  =  2 * 1000 * 1000,  /* 2 ms  WCET   */
	.sched_deadline = 10 * 1000 * 1000,  /* 10 ms relative deadline */
	.sched_period   = 10 * 1000 * 1000,  /* 10 ms period */
};
```

Two mechanisms make this a guarantee:
- **Admission control.** `sched_setattr()` returns `-EBUSY` if adding this task would push
  total utilization past what is schedulable. The kernel *refuses* to make a promise it
  cannot keep.
- **Budget enforcement (CBS).** A task that exceeds `sched_runtime` in a period is throttled
  until its next period. An overrunning task damages *itself*, not its neighbours — which is
  precisely the fix for EDF's domino-failure problem.

**RT throttling**, which surprises everyone: by default
`/proc/sys/kernel/sched_rt_runtime_us` = 950000 and `sched_rt_period_us` = 1000000, capping
*all* `SCHED_FIFO`/`RR` tasks at 95% of each CPU. It exists so a runaway RT task cannot lock
you out of the machine. If your RT task mysteriously stalls for 50 ms every second, this is
why. Disabling it (`-1`) is legitimate on an isolated core, but only with a watchdog.

**Priority assignment**, briefly: rate-monotonic (shorter period ⇒ higher priority) is the
optimal fixed-priority assignment, which gives you a principled starting point instead of
"pick 80." Leave headroom above your tasks for the IRQ threads that serve them — a common
mistake is running an RT task at 99, above the `irq/` thread that delivers its data, which
deadlocks progress.

### T.7 — CPU isolation: removing the other tenants

Priority alone is insufficient, because much of what perturbs you is not a task.

| Perturbation | Removal |
|---|---|
| Other tasks | `cpuset` / `isolcpus=` |
| Scheduler tick | `nohz_full=` (tickless when 1 runnable task) |
| RCU callbacks | `rcu_nocbs=` + pin `rcuo*` kthreads elsewhere |
| Device interrupts | `irqaffinity=`, per-IRQ `smp_affinity` |
| Workqueues | `WQ_SYSFS` → `/sys/devices/virtual/workqueue/*/cpumask` |
| Kernel threads | `kthread_cpus=`, `tuna`/`taskset` |
| Timer migration | `sysctl kernel.timer_migration=0` |
| Frequency changes | `performance` governor, disable turbo for determinism |
| Deep C-states | write to `/dev/cpu_dma_latency`, or `idle=poll` |
| SMT sibling | `nosmt`, or leave the sibling idle |
| Cross-CPU IPIs | hardest: TLB shootdowns and `stop_machine` still reach you |

The last row is the important caveat. **`stop_machine()` preempts everything on every CPU**,
including an isolated RT task, for potentially milliseconds. It is triggered by module
load/unload, CPU hotplug, some static-key updates, and certain MTRR/microcode operations.
Therefore: **no module loading, no hotplug, no `tracefs` enable/disable of static keys during
the RT phase.** This is a real operational constraint that people discover the hard way.

Modern preference: `cpuset` (cgroup v2 `cpuset.cpus.partition=isolated`) over the static
`isolcpus=` boot parameter, because it is dynamic and composable. `isolcpus` is effectively
deprecated but still widely used.

### T.8 — What the RT application itself must do

The kernel can only bound what it controls. The application contributes its own unbounded
terms:

```c
/* The mandatory preamble for any RT thread. */
mlockall(MCL_CURRENT | MCL_FUTURE);   /* no page faults, ever */
/* Prefault the stack: touch it now so the faults happen now. */
{ char dummy[64 * 1024]; memset(dummy, 0, sizeof(dummy)); }
/* Prefault and pre-size the heap; then never malloc again. */
mallopt(M_TRIM_THRESHOLD, -1);
mallopt(M_MMAP_MAX, 0);
```

Forbidden in the RT loop:
- **`malloc`/`free`** — may take a lock, may `mmap`, may fault.
- **Any page fault** — a major fault is 100 µs, which is often the entire budget.
- **Unbounded I/O**, `printf`, logging to a file, syslog.
- **Non-PI mutexes** shared with non-RT threads.
- **Anything with unbounded loop count** over a shared data structure.

Required:
- PI mutexes (`PTHREAD_PRIO_INHERIT`) for any lock shared with a lower-priority thread, or
  better: **lock-free SPSC rings** to communicate with the non-RT side, so no shared lock
  exists at all.
- `clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME, ...)` for periodic wakeups — **absolute**,
  so scheduling jitter does not accumulate into drift. Relative sleeps drift; this is a
  classic bug.
- `sched_setaffinity` to the isolated CPU, and `sched_setattr` for the policy.

### T.9 — Measuring: the only thing that makes any of this real

**`cyclictest`** is the standard. It creates RT threads that sleep for a fixed interval and
measures the difference between the requested and actual wakeup time — i.e. it measures
terms (1)–(4) of §T.2 directly.

```bash
# The canonical run. -m locks memory, -p99 sets priority, -i100 is 100 µs period,
# -h builds a histogram, --smi reports SMI counts.
cyclictest -m -S -p99 -i100 -h400 -l100000000 --smi
```

The rules of a valid measurement, each of which people violate:
1. **Run under representative load.** An idle system proves nothing. Use `stress-ng`,
   `hackbench`, and your actual workload concurrently.
2. **Run long.** The tail you care about appears at 10⁸ samples, not 10⁵. Hours to days.
3. **Report the maximum**, and the full histogram. The mean is irrelevant to a hard deadline.
4. **Separate hardware from software.** `hwlatdetect` measures SMI/firmware stalls that
   Linux cannot see or fix. If `hwlatdetect` reports 200 µs, no kernel configuration will
   get you below that, and you need a firmware/BIOS change or different hardware.

**The modern tracers, which are better than `cyclictest` for *attribution*:**

- **`osnoise`** — measures how much CPU time is stolen from a spinning thread and **by what**
  (IRQ, softirq, thread, NMI), with a per-source breakdown. This answers "why 200 µs?",
  which `cyclictest` cannot.
- **`timerlat`** — like `cyclictest` but in-kernel, distinguishing IRQ latency from thread
  latency, and able to trigger a trace dump on an outlier above a threshold. This is the
  single most useful RT debugging tool in the modern kernel.

```bash
# Attribute every outlier above 50 µs automatically:
rtla timerlat top -a 50
rtla osnoise hist -P F:99 -c 3 -d 10m
```

### T.10 — When Linux is the wrong answer

Judgement, and a good answer to "when would you not use Linux?":

| Need | Answer |
|---|---|
| < 10 µs worst case | Bare-metal loop, an RTOS on a co-processor (Zephyr, FreeRTOS), or an FPGA |
| Certification (DO-178C, IEC 61508 SIL3, ISO 26262 ASIL-D) | A certifiable RTOS, or Linux in a partition under a certified hypervisor (Jailhouse, PikeOS) |
| Determinism with a rich OS alongside | **Asymmetric multiprocessing**: RT core running an RTOS, Linux on the rest, communicating via `remoteproc`/`rpmsg` (→ Ch. 49) |
| Hard deadline + general-purpose stack | `PREEMPT_RT` on an isolated core, with the caveats in §T.7 |

The honest framing: Linux gives you excellent, *measurable*, but not *provable* bounds. If
your requirement is a mathematical proof rather than a 30-day histogram, the millions of
lines of kernel code are not a provable artifact and no amount of tuning changes that.

---

## 1. Internals

### Source map

| Path | Contents |
|---|---|
| `kernel/locking/rtmutex.c` | `rt_mutex` and the PI chain walk — the core of RT locking |
| `kernel/locking/rtmutex_api.c` | The `spinlock_t`→`rt_mutex` mapping used by RT |
| `kernel/locking/spinlock_rt.c` | RT spinlock implementation |
| `include/linux/spinlock_rt.h` | The RT definitions of `spin_lock` et al. |
| `include/linux/local_lock.h` | `local_lock_t` — explicit per-CPU protection |
| `kernel/irq/manage.c` | `request_threaded_irq`, IRQ thread creation, `irq_thread()` |
| `kernel/softirq.c` | Softirq handling, including the RT variant |
| `kernel/sched/deadline.c` | `SCHED_DEADLINE`: EDF + CBS, admission control |
| `kernel/sched/rt.c` | `SCHED_FIFO`/`RR`, RT throttling, push/pull balancing |
| `kernel/time/hrtimer.c` | High-resolution timers |
| `kernel/trace/trace_osnoise.c` | `osnoise` and `timerlat` tracers |
| `kernel/trace/trace_irqsoff.c` | `irqsoff`/`preemptoff`/`preemptirqsoff` tracers |
| `drivers/misc/hwlat_detector` / `trace_hwlat.c` | SMI/hardware latency detection |
| `Documentation/scheduler/sched-deadline.rst` | The formal model, worth reading fully |
| `Documentation/locking/rt-mutex-design.rst` | The PI algorithm, explained by its author |

### The PI chain walk

The interesting part of `rt_mutex`. When H blocks on a lock held by L:

```c
/* kernel/locking/rtmutex.c, conceptually */
rt_mutex_adjust_prio_chain(task, ...)
{
	for (;;) {
		/* 1. Is `task`'s priority still correct given its waiters? */
		if (!task_has_pi_waiters(task) || prio_unchanged)
			break;
		/* 2. Boost it. */
		rt_mutex_setprio(task, new_prio);
		/* 3. Is `task` itself blocked on another lock?
		 *    If so, the owner of THAT lock must be boosted too. */
		lock = task->pi_blocked_on ? task->pi_blocked_on->lock : NULL;
		if (!lock)
			break;
		/* 4. Requeue `task` in that lock's waiter tree at its new prio. */
		rt_mutex_enqueue(lock, waiter);
		task = rt_mutex_owner(lock);
		/* 5. Loop: walk up the chain. */
	}
}
```

Three properties to understand:

- The walk is **bounded** only by the chain depth, which is bounded by the lock nesting
  depth. A cycle would be a deadlock, and lockdep exists to prevent that.
- It runs with `raw_spinlock`s held (the lock's `wait_lock` and the task's `pi_lock`), in a
  careful hand-over-hand pattern — this is some of the most delicate code in the kernel.
- **Deboosting** on unlock is symmetric and equally necessary; forgetting it leaves a
  low-priority task permanently boosted.

### Threaded IRQ flow

```c
request_threaded_irq(irq, handler, thread_fn, flags, name, dev);
                     │        │         └── runs in a kthread, may sleep
                     │        └── hard-IRQ; returns IRQ_WAKE_THREAD
                     └── if NULL, the core supplies irq_default_primary_handler()
                         which just masks the interrupt and wakes the thread

 hard IRQ context:
   handler() -> IRQ_WAKE_THREAD
     -> __irq_wake_thread() -> wake_up_process(action->thread)

 thread context (irq/NN-name, SCHED_FIFO 50):
   irq_thread()
     -> irq_thread_fn() -> action->thread_fn()
     -> irq_finalize_oneshot()   /* unmask if IRQF_ONESHOT */
```

`IRQF_ONESHOT` matters: it keeps the interrupt masked until the thread finishes, which is
mandatory for level-triggered interrupts (otherwise the interrupt re-fires immediately and
you get the storm described in `debugging-scenarios.md` §6).

On `PREEMPT_RT`, `request_irq()` is threaded by default — a driver written for non-RT gets
the behaviour automatically, which was the whole point of pushing threaded IRQs into
mainline years before RT merged.

### Observability surface

```bash
# Preemption model of the running kernel
grep PREEMPT /boot/config-$(uname -r)
cat /sys/kernel/debug/sched/preempt   # 6.12+ with dynamic preemption

# IRQ threads and their priorities
ps -eLo pid,tid,class,rtprio,comm | grep -E 'irq/|ksoftirqd|rcu'

# Per-IRQ affinity and counts
cat /proc/interrupts
cat /proc/irq/*/smp_affinity_list

# RT throttling
sysctl kernel.sched_rt_runtime_us kernel.sched_rt_period_us

# Isolated CPUs as the kernel sees them
cat /sys/devices/system/cpu/isolated
cat /sys/devices/system/cpu/nohz_full

# Available tracers
cat /sys/kernel/debug/tracing/available_tracers
#   -> wakeup, wakeup_rt, irqsoff, preemptoff, preemptirqsoff, osnoise, timerlat, hwlat

# Latency of the longest IRQ-disabled section seen so far
echo irqsoff > /sys/kernel/debug/tracing/current_tracer
cat /sys/kernel/debug/tracing/tracing_max_latency
```

---

## 2. Practice

### Lab 103.1 — Establish a baseline

```bash
#!/bin/bash
# rt_baseline.sh — measure before you tune. Run this first, always.
set -e
DUR=${1:-300}   # seconds

echo "=== Kernel ==="
uname -a
grep -E 'CONFIG_PREEMPT|CONFIG_HZ|CONFIG_NO_HZ|CONFIG_HIGH_RES_TIMERS' \
	/boot/config-$(uname -r) 2>/dev/null | grep -v '^#'

echo; echo "=== Hardware latency (SMI/firmware) — the floor you cannot beat ==="
sudo hwlatdetect --duration=60 --threshold=10 || echo "hwlatdetect unavailable"

echo; echo "=== Idle system ==="
sudo cyclictest -m -S -p99 -i200 -h200 -q -D30 | tail -5

echo; echo "=== Under load (this is the number that matters) ==="
stress-ng --cpu $(nproc) --io 4 --vm 2 --vm-bytes 1G -t ${DUR}s &
SPID=$!
hackbench -l 100000 >/dev/null 2>&1 &
HPID=$!
sudo cyclictest -m -S -p99 -i200 -h200 -q -D${DUR} | tail -5
kill $SPID $HPID 2>/dev/null || true
wait 2>/dev/null || true
```

Record: max under load, and the ratio of loaded-max to idle-max. On a stock
`CONFIG_PREEMPT` kernel expect loaded max in the hundreds of µs to low ms. On `PREEMPT_RT`
with isolation, expect tens of µs. **The delta between those two runs is what this entire
chapter buys you.**

### Lab 103.2 — Attribute an outlier with `rtla timerlat`

```bash
# Stop and dump a trace whenever latency exceeds 30 µs. This is the tool that
# turns "sometimes it's slow" into "IRQ 34 from the NIC, here is the stack."
sudo rtla timerlat top -a 30 -c 3 -d 5m

# Histogram with a full breakdown of IRQ vs thread latency:
sudo rtla timerlat hist -P f:95 -c 3 -d 5m -T 30

# Who is stealing time from CPU 3, and how much?
sudo rtla osnoise top -c 3 -d 5m
sudo rtla osnoise hist -c 3 -d 5m
```

`osnoise` output columns are the lesson: it reports noise attributed to `NMI`, `IRQ`,
`SOFTIRQ`, and `THREAD` separately, plus the count of each. An outlier attributed to `IRQ`
sends you to `/proc/interrupts` and affinity; one attributed to `THREAD` sends you to
`ps -eLo class,rtprio`; one attributed to none of them is hardware, and `hwlatdetect`
confirms it. **That decision tree is the whole skill.**

### Lab 103.3 — A correct periodic RT thread

```c
/* rt_periodic.c — the reference implementation of an RT control loop.
 * Build: gcc -O2 -Wall -o rt_periodic rt_periodic.c -lpthread
 * Run:   sudo ./rt_periodic 3 1000     # CPU 3, 1000 µs period
 *
 * Demonstrates every mandatory element: memory locking, stack prefault,
 * absolute-time periodic wakeup, affinity, RT policy, and jitter accounting.
 */
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sched.h>
#include <time.h>
#include <errno.h>
#include <malloc.h>
#include <sys/mman.h>

#define STACK_PREFAULT   (512 * 1024)
#define NSEC_PER_SEC     1000000000L

static long max_jitter_ns;
static long long samples, sum_jitter;

static void prefault_stack(void)
{
	/* Touch the stack now so page faults happen here, not in the loop. */
	unsigned char dummy[STACK_PREFAULT];
	memset(dummy, 0, sizeof(dummy));
	/* volatile read prevents the compiler eliminating the whole thing */
	*(volatile unsigned char *)dummy = 0;
}

static void ts_add_ns(struct timespec *ts, long ns)
{
	ts->tv_nsec += ns;
	while (ts->tv_nsec >= NSEC_PER_SEC) {
		ts->tv_nsec -= NSEC_PER_SEC;
		ts->tv_sec++;
	}
}

static long ts_diff_ns(const struct timespec *a, const struct timespec *b)
{
	return (a->tv_sec - b->tv_sec) * NSEC_PER_SEC + (a->tv_nsec - b->tv_nsec);
}

int main(int argc, char **argv)
{
	int cpu = (argc > 1) ? atoi(argv[1]) : 0;
	long period_us = (argc > 2) ? atol(argv[2]) : 1000;
	long period_ns = period_us * 1000;
	struct timespec next, now;
	struct sched_param sp = { .sched_priority = 80 };
	cpu_set_t set;

	/* 1. Lock ALL memory, current and future. A major fault is ~100 µs,
	 *    which for a 1 ms loop is 10% of the budget -- and unbounded.   */
	if (mlockall(MCL_CURRENT | MCL_FUTURE)) {
		perror("mlockall");
		return 1;
	}

	/* 2. Stop glibc returning memory to the kernel, and stop it using
	 *    mmap for large allocations -- both cause faults later.        */
	mallopt(M_TRIM_THRESHOLD, -1);
	mallopt(M_MMAP_MAX, 0);

	/* 3. Prefault the stack. */
	prefault_stack();

	/* 4. Pin to the isolated CPU. Migration is a latency source and
	 *    destroys cache and NUMA locality.                            */
	CPU_ZERO(&set);
	CPU_SET(cpu, &set);
	if (sched_setaffinity(0, sizeof(set), &set)) {
		perror("sched_setaffinity");
		return 1;
	}

	/* 5. Become a real-time task. Note: 80, not 99 -- leave headroom
	 *    above us for the IRQ threads that feed us.                   */
	if (sched_setscheduler(0, SCHED_FIFO, &sp)) {
		perror("sched_setscheduler (need CAP_SYS_NICE / root?)");
		return 1;
	}

	printf("RT loop: cpu=%d period=%ld us prio=%d. Ctrl-C to stop.\n",
	       cpu, period_us, sp.sched_priority);

	clock_gettime(CLOCK_MONOTONIC, &next);

	for (;;) {
		long jitter;

		/* 6. ABSOLUTE sleep. A relative sleep accumulates the wakeup
		 *    error every cycle and the loop drifts. This single
		 *    detail is the most common bug in RT application code.  */
		ts_add_ns(&next, period_ns);
		if (clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME, &next, NULL)) {
			if (errno == EINTR)
				continue;
			perror("clock_nanosleep");
			break;
		}

		clock_gettime(CLOCK_MONOTONIC, &now);
		jitter = ts_diff_ns(&now, &next);   /* how late were we? */

		/* ---- the actual work would go here ----
		 * Rules: bounded execution time, no allocation, no I/O,
		 * no non-PI locks, no unbounded loops.                    */

		samples++;
		sum_jitter += jitter;
		if (jitter > max_jitter_ns) {
			max_jitter_ns = jitter;
			printf("new max jitter: %ld ns (avg %lld ns over %lld)\n",
			       max_jitter_ns, sum_jitter / samples, samples);
			fflush(stdout);
		}
	}
	return 0;
}
```

Run it on a non-isolated CPU under load, then on an isolated CPU, and compare `max`. Then
deliberately break one rule at a time — remove `mlockall`, use a relative sleep, add a
`malloc` in the loop, drop the affinity — and observe which one costs the most. **The
ranking is the lesson**, and it is usually: relative sleep (unbounded drift) > no
`mlockall` > allocation > no affinity.

### Lab 103.4 — Build an isolated RT partition

```bash
#!/bin/bash
# isolate.sh — carve CPU 3 out of the general-purpose system.
# Run as root. Assumes an 8-CPU machine; adjust masks.
set -e
RT_CPU=3
HOUSEKEEPING=0-2,4-7
HK_MASK=$(printf '%x' $(( (1<<8) - 1 - (1<<RT_CPU) )))   # all but RT_CPU

echo "=== 1. Move all movable IRQs off CPU $RT_CPU ==="
for d in /proc/irq/[0-9]*; do
	echo "$HK_MASK" > "$d/smp_affinity" 2>/dev/null || true
done
echo "$HK_MASK" > /proc/irq/default_smp_affinity

echo "=== 2. Move workqueues off ==="
for w in /sys/devices/virtual/workqueue/*/cpumask; do
	echo "$HK_MASK" > "$w" 2>/dev/null || true
done
echo "$HK_MASK" > /sys/devices/virtual/workqueue/cpumask 2>/dev/null || true

echo "=== 3. Disable timer migration onto us, and RT throttling ==="
sysctl -w kernel.timer_migration=0
sysctl -w kernel.sched_rt_runtime_us=-1     # DANGER: no safety net. Use a watchdog.

echo "=== 4. Performance governor, no deep C-states ==="
for g in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
	echo performance > "$g" 2>/dev/null || true
done
# Hold a 0 µs DMA latency constraint for as long as this fd is open:
exec 9<> /dev/cpu_dma_latency && printf '\x00\x00\x00\x00' >&9

echo "=== 5. cgroup v2 isolated partition (preferred over isolcpus=) ==="
CG=/sys/fs/cgroup
mkdir -p $CG/rt
echo "+cpuset" > $CG/cgroup.subtree_control 2>/dev/null || true
echo "$RT_CPU"        > $CG/rt/cpuset.cpus
echo isolated         > $CG/rt/cpuset.cpus.partition
echo "$HOUSEKEEPING"  > $CG/cpuset.cpus 2>/dev/null || true

echo "=== 6. Verify ==="
cat /sys/devices/system/cpu/isolated
cat $CG/rt/cpuset.cpus.partition
grep -c . /proc/interrupts >/dev/null
awk -v c=$((RT_CPU+2)) 'NR>1 {s+=$c} END {print "IRQs still landing on CPU '"$RT_CPU"': " s}' \
	/proc/interrupts

echo
echo "Boot-time additions for full isolation (add to the kernel cmdline):"
echo "  nohz_full=$RT_CPU rcu_nocbs=$RT_CPU rcu_nocb_poll irqaffinity=$HOUSEKEEPING"
echo "  skew_tick=1 tsc=reliable intel_pstate=disable processor.max_cstate=1 idle=poll"
echo "Then run your task with:  taskset -c $RT_CPU chrt -f 80 ./rt_periodic $RT_CPU 1000"
```

Measure with `cyclictest` before and after. Then check which perturbations remain: run
`rtla osnoise top -c 3` and see what still steals time. The residue is usually TLB shootdown
IPIs and the occasional `stop_machine` — the things §T.7 warned you cannot remove.

### Lab 103.5 — Reproduce priority inversion, then fix it

```c
/* inversion.c — demonstrate unbounded priority inversion and PI's fix.
 * Build: gcc -O2 -o inversion inversion.c -lpthread
 * Run:   sudo taskset -c 2 ./inversion 0     # no PI  -> huge H delay
 *        sudo taskset -c 2 ./inversion 1     # PI     -> bounded
 *
 * All three threads share ONE cpu, which is what makes the inversion
 * observable.                                                        */
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>
#include <time.h>
#include <sched.h>
#include <sys/mman.h>

static pthread_mutex_t lock;
static volatile int m_spin = 1;
static struct timespec t_h_block, t_h_acquire;

static void busy_ms(long ms)
{
	struct timespec s, n;
	clock_gettime(CLOCK_MONOTONIC, &s);
	do {
		clock_gettime(CLOCK_MONOTONIC, &n);
	} while ((n.tv_sec - s.tv_sec) * 1000L +
		 (n.tv_nsec - s.tv_nsec) / 1000000L < ms);
}

static void set_prio(int prio)
{
	struct sched_param sp = { .sched_priority = prio };
	if (pthread_setschedparam(pthread_self(), SCHED_FIFO, &sp))
		perror("pthread_setschedparam");
}

static void *low(void *arg)
{
	set_prio(10);
	pthread_mutex_lock(&lock);
	busy_ms(2000);              /* hold the lock for 2 s */
	pthread_mutex_unlock(&lock);
	return NULL;
}

static void *medium(void *arg)
{
	set_prio(50);
	usleep(100 * 1000);         /* let L acquire and H block first */
	/* Pure CPU burn. Does NOT touch the lock -- that is the point. */
	while (m_spin)
		;
	return NULL;
}

static void *high(void *arg)
{
	set_prio(90);
	usleep(50 * 1000);          /* let L acquire */
	clock_gettime(CLOCK_MONOTONIC, &t_h_block);
	pthread_mutex_lock(&lock);
	clock_gettime(CLOCK_MONOTONIC, &t_h_acquire);
	pthread_mutex_unlock(&lock);
	m_spin = 0;
	return NULL;
}

int main(int argc, char **argv)
{
	int use_pi = (argc > 1) ? atoi(argv[1]) : 0;
	pthread_mutexattr_t attr;
	pthread_t tl, tm, th;
	double ms;

	mlockall(MCL_CURRENT | MCL_FUTURE);

	pthread_mutexattr_init(&attr);
	if (use_pi)
		pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
	pthread_mutex_init(&lock, &attr);

	pthread_create(&tl, NULL, low, NULL);
	pthread_create(&tm, NULL, medium, NULL);
	pthread_create(&th, NULL, high, NULL);

	pthread_join(th, NULL);
	pthread_join(tl, NULL);
	pthread_join(tm, NULL);

	ms = (t_h_acquire.tv_sec - t_h_block.tv_sec) * 1000.0 +
	     (t_h_acquire.tv_nsec - t_h_block.tv_nsec) / 1e6;
	printf("PI %s: high-priority thread blocked for %.1f ms\n",
	       use_pi ? "ON " : "OFF", ms);
	printf("  expected: OFF -> ~2000+ ms (bounded only by M finishing)\n");
	printf("            ON  -> ~2000 ms but L runs at 90, M never runs\n");
	return 0;
}
```

The instructive observation: with PI **off** on a single CPU, M (priority 50) runs *instead
of* L (priority 10), so H waits for M — and M here never finishes on its own, so H waits
forever until H's own completion flag is set. With PI **on**, L is boosted to 90, M cannot
preempt it, L finishes its critical section promptly, and H proceeds. Instrument with
`chrt -p` from another terminal to watch L's priority change in real time — seeing the boost
happen is worth more than reading about it.

### Lab 103.6 — `SCHED_DEADLINE` and admission control

```c
/* deadline.c — demonstrate that SCHED_DEADLINE refuses impossible promises.
 * Build: gcc -O2 -o deadline deadline.c
 * Run:   sudo ./deadline 2 10     # 2 ms runtime in a 10 ms period = 20%
 *        sudo ./deadline 9 10     # 90% -- watch several instances fail    */
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <linux/sched.h>
#include <linux/sched/types.h>
#include <sys/syscall.h>
#include <time.h>

static int sched_setattr_(pid_t pid, const struct sched_attr *a, unsigned int f)
{ return syscall(__NR_sched_setattr, pid, a, f); }

int main(int argc, char **argv)
{
	long runtime_ms = (argc > 1) ? atol(argv[1]) : 2;
	long period_ms  = (argc > 2) ? atol(argv[2]) : 10;
	struct sched_attr attr;
	long long cycles = 0;
	time_t start;

	memset(&attr, 0, sizeof(attr));
	attr.size           = sizeof(attr);
	attr.sched_policy   = SCHED_DEADLINE;
	attr.sched_runtime  = runtime_ms * 1000000ULL;
	attr.sched_deadline = period_ms  * 1000000ULL;
	attr.sched_period   = period_ms  * 1000000ULL;

	if (sched_setattr_(0, &attr, 0)) {
		if (errno == EBUSY) {
			fprintf(stderr,
			  "sched_setattr: EBUSY -- ADMISSION CONTROL REFUSED.\n"
			  "The kernel will not promise %ld ms every %ld ms because\n"
			  "the total utilization would exceed what is schedulable.\n"
			  "This is the whole point: a guarantee it cannot keep is\n"
			  "not offered at all.\n", runtime_ms, period_ms);
		} else {
			perror("sched_setattr");
		}
		return 1;
	}

	printf("admitted: %ld ms / %ld ms (%.0f%% utilization)\n",
	       runtime_ms, period_ms, 100.0 * runtime_ms / period_ms);

	start = time(NULL);
	while (time(NULL) - start < 10) {
		/* Burn roughly the budget, then yield the remainder of this
		 * period. sched_yield() under DEADLINE means "I'm done for
		 * this period" -- different from its meaning elsewhere.    */
		struct timespec s, n;
		clock_gettime(CLOCK_MONOTONIC, &s);
		do {
			clock_gettime(CLOCK_MONOTONIC, &n);
		} while ((n.tv_sec - s.tv_sec) * 1000000000L +
			 (n.tv_nsec - s.tv_nsec) < (runtime_ms - 1) * 1000000L);
		cycles++;
		sched_yield();
	}
	printf("completed %lld periods in 10 s (expected ~%ld)\n",
	       cycles, 10000 / period_ms);
	return 0;
}
```

```bash
# Launch instances until admission fails. On an N-CPU system the total
# admissible utilization is roughly N (minus the RT-throttling reserve).
for i in $(seq 1 12); do sudo ./deadline 8 10 & sleep 0.2; done
```

Then deliberately overrun the budget (increase the busy loop past `sched_runtime`) and
observe the CBS throttle the task rather than letting it steal time. Compare with
`SCHED_FIFO`, where an overrunning task simply runs and starves everything below it. **That
comparison is the argument for `SCHED_DEADLINE` in one experiment.**

---

## 3. Mastery drills

1. Build both a `CONFIG_PREEMPT` and a `CONFIG_PREEMPT_RT` kernel from the same source.
   Measure: `cyclictest` max under load, kernel build time, `hackbench` throughput, and a
   network `iperf3` result. Produce the throughput-vs-latency tradeoff table with real
   numbers, then write the one-paragraph recommendation for a workload of your choosing.

2. Using `rtla osnoise hist`, attribute every source of noise on a fully isolated CPU over
   an hour. Rank them. For each, either eliminate it or explain precisely why you cannot.

3. Take the `irqsoff` tracer and find the ten longest interrupt-disabled regions in your
   kernel under a realistic workload. For the worst one, read the code and determine whether
   it is reducible.

4. Write a driver whose IRQ handler deliberately does 500 µs of work in hard-IRQ context.
   Measure the effect on `cyclictest`. Convert it to a threaded IRQ and re-measure. Then set
   the IRQ thread's priority above and below the RT task and explain both results.

5. Implement a lock-free SPSC ring buffer for RT↔non-RT communication. Prove (by argument
   and by `KCSAN`) that it needs no locks, and identify exactly which memory barriers are
   required and why. Measure its worst-case latency versus a PI mutex.

6. Construct a three-level PI chain (H blocks on A held by M, M blocks on B held by L) and
   observe the transitive boost with `trace-cmd record -e sched:sched_pi_setprio`. Then
   construct a chain deep enough to measure the chain-walk cost.

7. Determine empirically how long `stop_machine()` stalls an isolated RT CPU by loading and
   unloading a module in a loop while `cyclictest` runs. Write the operational policy that
   follows from your measurement.

8. Configure a system where a `SCHED_DEADLINE` task, a `SCHED_FIFO` task, and a
   `SCHED_OTHER` task share one CPU. Predict the behaviour of each, then measure. Explain
   any discrepancy, including the effect of RT throttling.

9. Port the RT periodic loop (Lab 103.3) to use `io_uring` for its I/O. Determine whether
   `io_uring` is RT-safe: what allocations does submission perform, what locks does it take,
   and is `IORING_SETUP_SQPOLL` better or worse for determinism?

10. Build an AMP system in QEMU: Linux on most cores, a bare-metal or Zephyr image on one,
    communicating via shared memory and `rpmsg` (→ Ch. 49). Compare its worst-case latency
    against `PREEMPT_RT` on the same hardware and state the crossover point where AMP wins.

11. Take a `PREEMPT_RT` kernel and find three places where `raw_spinlock_t` is used. For
    each, explain why a sleeping lock is impossible there, and bound the section's execution
    time by reading the code.

12. Design and document the full RT configuration for a motor-control application with a
    100 µs cycle and a hard deadline. Include the hardware requirements, kernel config, boot
    parameters, isolation setup, application structure, the validation plan, and — most
    importantly — the list of things you would refuse to allow on that machine.

13. Argue the case, with measurements, for and against enabling `PREEMPT_RT` on a fleet of
    general-purpose servers that occasionally run latency-sensitive workloads. What would
    you measure to decide, and what result would change your answer?

---

## 4. Further reading

**Kernel documentation**
- `Documentation/scheduler/sched-deadline.rst` — the CBS model with the formal treatment
- `Documentation/scheduler/sched-rt-group.rst` — RT throttling and group scheduling
- `Documentation/locking/rt-mutex-design.rst` — the PI algorithm from its implementer
- `Documentation/timers/` — `hrtimers.rst`, `no_hz.rst` (read `no_hz.rst` fully; it explains
  exactly what `nohz_full` does and does not remove)
- `Documentation/tools/rtla/` — the `rtla` tool suite
- `Documentation/admin-guide/kernel-per-CPU-kthreads.rst` — **the definitive checklist** for
  removing kernel threads from an isolated CPU

**Papers**
- Liu & Layland, "Scheduling Algorithms for Multiprogramming in a Hard-Real-Time
  Environment," *JACM*, 1973 — RM and EDF, and the utilization bounds
- Sha, Rajkumar, Lehoczky, "Priority Inheritance Protocols: An Approach to Real-Time
  Synchronization," *IEEE Trans. Computers*, 1990 — PI and PCP, formally
- Abeni & Buttazzo, "Integrating Multimedia Applications in Hard Real-Time Systems,"
  *RTSS*, 1998 — the Constant Bandwidth Server behind `SCHED_DEADLINE`
- Lelli et al., "Deadline Scheduling in the Linux Kernel," *Software: Practice and
  Experience*, 2016 — the implementation paper
- Reghenzani, Massari, Fornaciari, "The Real-Time Linux Kernel: A Survey on PREEMPT_RT,"
  *ACM Computing Surveys*, 2019 — the best single survey
- de Oliveira et al., "Demystifying the Real-Time Linux Scheduling Latency," *ECRTS*, 2020 —
  a formal model of where latency comes from; the basis of the `osnoise` tracer

**LWN**
- "Realtime preemption locking core merged" and the full `PREEMPT_RT` merge coverage (6.12)
- Steven Rostedt's `rtla`, `osnoise`, and `timerlat` articles
- "The real-time preemption endgame" series
- "Deadline scheduling for Linux" and follow-ups
- "Lazy preemption" (2024–25) — the `PREEMPT_LAZY` work unifying the preemption models

**Books**
- Buttazzo, *Hard Real-Time Computing Systems*, 3rd ed. — the standard text on the theory;
  read ch. 4 (periodic scheduling) and ch. 7 (resource access protocols)
- Liu, *Real-Time Systems* — more formal, excellent on schedulability analysis
- Kopetz, *Real-Time Systems: Design Principles for Distributed Embedded Applications* — the
  systems-architecture view, including time-triggered design

**Tools**
- `rt-tests` suite: `cyclictest`, `hackbench`, `pi_stress`, `signaltest`, `ptsematest`
- `rtla`: `timerlat`, `osnoise`, `hwnoise` — the modern, attribution-capable tools
- `hwlatdetect` — separate hardware/firmware latency from software
- `tuna` — interactive IRQ and thread affinity/priority manipulation
- `stalld` — a userspace daemon that boosts starving tasks; useful on RT systems where a
  priority-inverted low-priority thread would otherwise never run
- `trace-cmd` / `kernelshark` — visualize the scheduling timeline around an outlier
- The `tuned` `realtime` profile (RHEL) as a reference configuration to read, not to trust

→ Next: [104-virtualization.md](104-virtualization.md)

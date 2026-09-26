# Chapter 21 — The Scheduler: CFS → EEVDF, Real-Time, Deadline, Load Balancing

> **Goal:** explain *why* EEVDF replaced CFS using the underlying theory, read
> `kernel/sched/fair.c` without drowning, diagnose a latency problem with data, and know when
> `SCHED_DEADLINE` or `sched_ext` is the right answer.

---

## Theory & First Principles

### T.0 — Start here: write a scheduler, then watch it fail

You have runnable tasks and one CPU. Pick the next one. How hard can it be?

**Attempt 1: round robin.** Everyone gets 10 ms in turn. Fair, simple, and:

```
  A text editor waiting for a keystroke is runnable for 50 us every keypress.
  A compiler is runnable always.

  Round robin gives each 10 ms in turn. With 8 compiler jobs running, your
  keystroke waits up to 80 ms.  The editor feels broken.
```

**Attempt 2: prioritize interactive tasks.** Give a bonus to tasks that sleep a lot. Now:

```
  A task learns to sleep 1 ms out of every 10 to keep its "interactive" bonus,
  and starves everything else. This is not hypothetical -- the O(1) scheduler's
  interactivity heuristics (2003-2007) were exactly this: they were gamed, and
  they misclassified real workloads in ways nobody could debug, because the
  heuristics were unprincipled.
```

**Attempt 3: shortest job first.** Provably minimizes average wait time. And:

```
  It requires knowing how long each task will run. You cannot know this.
  And long jobs starve forever.
```

**The problem is not that scheduling is hard to implement. It is that the objectives
contradict each other:**

| Objective | Wants |
|---|---|
| **Throughput** | Long timeslices. Never switch. Batch everything |
| **Latency** | Short timeslices. Switch immediately on wakeup |
| **Fairness** | Equal shares over time |
| **Priority** | *Un*equal shares, on purpose |
| **Determinism** | Bounded worst case, even at the cost of the average |
| **Energy** | Race to idle, or run slow and steady -- opposite strategies |
| **Cache/NUMA locality** | Never migrate a task |
| **Load balance** | Migrate tasks |

**Throughput and latency are in direct opposition** -- every context switch buys response
time and costs 1-5 us of throughput (Ch. 00 §T.3). Locality and load balance are in direct
opposition. **There is no optimum**, only a chosen point on a trade surface — which is why
there is no "correct" scheduler and why Linux has five scheduling classes.

**What Linux actually does** is refuse the heuristics and pick a principled model:

```
  "Each task should receive a share of CPU proportional to its weight,
   as if the CPU were infinitely divisible and ran all tasks simultaneously
   at proportional speeds."        <- the IDEAL FLUID model

  Then: track how far each task has DEVIATED from its ideal share,
        and always run the one that is furthest behind.
```

That is CFS (2007), and it removed an entire class of unexplainable behaviour — because
there is now a *definition* of correct that you can measure a deviation from.

**But it was still not enough**, and the reason is the point of §T.3: *proportional share
says nothing about **when** within a period you get your share.* A task needing 1 ms of CPU
every 10 ms for audio, and a task needing 100 ms every second, may have identical weights and
identical fairness — and wildly different latency requirements. CFS could express "more CPU,"
not "sooner."

**EEVDF (6.6+) adds the missing dimension**: each task declares a *requested timeslice*, from
which a virtual **deadline** is computed. Among tasks that are *eligible* (owed CPU), run the
earliest deadline. Short-slice tasks get earlier deadlines and hence lower latency — **without
needing more CPU.** Fairness and latency become independent knobs, which is what the model
needed all along.

```bash
cat /sys/kernel/debug/sched/debug | head -40
cat /proc/$$/sched                       # your shell's vruntime, wakeups, migrations
chrt -p $$                               # scheduling class and priority
```

---

### T.1 Scheduling is a multi-objective optimization with no optimum

A scheduler allocates one resource (CPU time) among competing tasks. The objectives are
mutually contradictory:

| Objective | Definition | Conflicts with |
|---|---|---|
| **Throughput** | useful work per second | latency (context switches cost) |
| **Latency / responsiveness** | time from runnable → running | throughput, cache warmth |
| **Fairness** | proportional share of CPU | latency for interactive tasks |
| **Determinism** | bounded worst-case response | throughput, fairness |
| **Energy** | joules per unit work | latency (race-to-idle vs slow-and-steady) |
| **Isolation** | one task can't hurt another | utilization |

There is provably no single "best" schedule. General multiprocessor scheduling with
precedence constraints is NP-hard; even the single-processor case becomes intractable once
you add arbitrary arrival times and deadlines. So every real scheduler is a **heuristic
tuned for an assumed workload mix**, and every scheduler argument is really an argument about
which workload matters.

This is why Linux has **five scheduling classes** consulted in strict priority order —
different classes encode different objective functions:

```c
/* kernel/sched/sched.h — checked highest first */
stop_sched_class      /* CPU stopper: migration, hotplug. Preempts everything. */
dl_sched_class        /* SCHED_DEADLINE — EDF + CBS.  Determinism.            */
rt_sched_class        /* SCHED_FIFO / SCHED_RR — fixed priority. Determinism.  */
fair_sched_class      /* SCHED_OTHER / BATCH / IDLE — EEVDF. Fairness+latency. */
ext_sched_class       /* sched_ext — BPF-defined policy (6.12+)                */
idle_sched_class      /* the idle task                                         */
```

`pick_next_task()` walks this list. A single runnable `SCHED_FIFO` task starves every
`SCHED_OTHER` task on that CPU, by design — hence `sched_rt_runtime_us` as a safety valve.

### T.2 Proportional share: the theory CFS and EEVDF both implement

The fairness notion Linux uses is **proportional share** (a.k.a. weighted fair queueing,
borrowed from network packet scheduling). Given tasks with weights $w_i$, over any interval
$[t_1, t_2]$ each task should receive

$$ S_i(t_1,t_2) = \frac{w_i}{\sum_j w_j}\,(t_2 - t_1) $$

An **ideal fluid-flow (GPS, Generalized Processor Sharing) server** would run all tasks
simultaneously at fractional rates. Real CPUs are discrete, so the practical question is:
*how far can a real schedule deviate from the fluid ideal?* That deviation is called **lag**:

$$ \text{lag}_i(t) = S_i^{\text{ideal}}(t) - S_i^{\text{actual}}(t) $$

- lag > 0 → the task is **owed** CPU time (it has been under-served)
- lag < 0 → the task has **over-consumed**
- $\sum_i \text{lag}_i = 0$ at all times (what one task gains, others lose)

A scheduler is **fair** if it keeps $|\text{lag}|$ bounded by a small constant.

Three classical algorithms implement this, and Linux has used two of them:

| Algorithm | Idea | Lag bound | Latency guarantee |
|---|---|---|---|
| **Lottery scheduling** (Waldspurger & Weihl, OSDI 1994) | random draw weighted by tickets | probabilistic only | none |
| **Stride scheduling** (Waldspurger, 1995) | each task has `stride = K/w`; always run the minimum `pass`; then `pass += stride` | **deterministic, O(1)** | none |
| **EEVDF** (Stoica & Abdel-Wahab, 1995) | stride, **plus** eligibility and virtual deadlines | deterministic | **yes — provably optimal** |

**CFS was stride scheduling.** Its `vruntime` *is* the stride scheduler's `pass` value:

```c
vruntime += delta_exec * NICE_0_LOAD / weight;
```
"Always run the task with the smallest `vruntime`" is exactly "always run the minimum-pass
task". The red-black tree keyed on `vruntime` with a cached leftmost node (Ch. 09 T.5) is
just an efficient min-queue.

### T.3 Why CFS had to be replaced: fairness ≠ latency

CFS guaranteed **long-run proportional fairness** but said *nothing* about **when** within
that period a task runs. Two tasks each entitled to 50% can be scheduled as
`AAAA...BBBB...` (100 ms slices) or `ABABAB...` (1 ms slices) — both perfectly fair, wildly
different latency.

CFS's only latency control was `sched_latency_ns` / `sched_min_granularity_ns` — a single
global knob computing a time slice as `latency × weight / total_weight`. Problems:

- **No per-task latency requirement.** A video player needing 5 ms response and a compiler
  needing throughput got the same treatment. `nice` conflated *share* with *latency*, which
  is wrong: you often want "small share, but when I wake, run me now" (an audio thread) or
  "large share, latency irrelevant" (a batch job).
- **The heuristics accreted.** `GENTLE_FAIR_SLEEPERS`, wakeup preemption with
  `sched_wakeup_granularity_ns`, `START_DEBIT`, `place_entity` sleeper bonuses — each a
  patch for a specific complaint, each interacting unpredictably with the others. By 2022
  nobody could reason about the combined behaviour.
- **No theoretical framework** to say whether a proposed change was *correct*, only whether
  it improved a benchmark.

Peter Zijlstra's EEVDF work (merged in **6.6**) replaced the heuristic pile with an algorithm
that has a **proof**.

### T.4 EEVDF: Earliest Eligible Virtual Deadline First

EEVDF (Stoica & Abdel-Wahab, 1995) adds two concepts to stride scheduling.

**(1) Eligibility.** A task is *eligible* at time $t$ if its lag ≥ 0 — i.e. it has not yet
received more than its fair share up to now. Tasks that have over-consumed are **ineligible**
and simply not considered, until virtual time advances enough to make them eligible again.
This directly bounds how far ahead any task can get.

**(2) Virtual deadline.** Each task declares a **request size** $r_i$ (its desired time
slice). Its virtual deadline is

$$ vd_i = ve_i + \frac{r_i}{w_i} $$

where $ve_i$ is its virtual eligible time. Among all *eligible* tasks, run the one with the
**earliest virtual deadline**.

The consequence is the whole point:

> **A task that asks for a *smaller* slice gets an *earlier* deadline, and therefore runs
> sooner — without receiving any more total CPU.**

**Latency and bandwidth become independent knobs.** This is what CFS structurally could not
express. It is exposed to userspace as:

```c
struct sched_attr attr = {
	.size            = sizeof(attr),
	.sched_policy    = SCHED_OTHER,
	.sched_nice      = 0,       /* BANDWIDTH: how much CPU  */
	.sched_flags     = SCHED_FLAG_UTIL_CLAMP | SCHED_FLAG_RESET_ON_FORK,
	.sched_runtime   = 0,
};
sched_setattr(0, &attr, 0);

/* and the new latency knob: */
prctl(PR_SET_TIMERSLACK, ...);            /* unrelated, but often confused */
/* the real one: */
attr.sched_flags |= SCHED_FLAG_LATENCY_NICE? ;  /* latency_nice: -20..19 */
```
`latency_nice` (and the cgroup `cpu.latency.nice`) maps to the request size $r_i$: lower
latency-nice → smaller slice → earlier deadline → faster wakeup response, **same share**.

EEVDF's theoretical guarantee: lag is bounded by $\max_i r_i$, and it is **optimal** in the
sense that no algorithm can achieve a smaller worst-case lag for the same request sizes.
That is the difference between "a pile of heuristics that benchmark well" and "an algorithm".

The fields you will see in `struct sched_entity`:

```c
struct sched_entity {
	struct load_weight	load;       /* w_i, from nice */
	struct rb_node		run_node;
	u64			deadline;   /* vd_i  — EEVDF */
	u64			min_vruntime;
	u64			vruntime;   /* ve_i  — the stride "pass" */
	s64			vlag;       /* lag_i — EEVDF */
	u64			slice;      /* r_i   — request size */
	u64			sum_exec_runtime;
	struct sched_avg	avg;        /* PELT */
	...
};
```

```bash
grep -n 'vlag\|deadline\|slice' include/linux/sched.h | head -20
grep -n 'entity_eligible\|update_deadline\|place_entity' kernel/sched/fair.c
```

### T.5 PELT: estimating "how much CPU does this task need?"

Fairness needs weights; **load balancing and frequency selection need demand estimates**.
That is a signal-processing problem: from a binary runnable/not-runnable history, estimate
future utilization.

**PELT (Per-Entity Load Tracking)** uses a geometric series — an exponentially weighted
moving average with a 32 ms half-life:

$$ u(n) = u(n-1)\cdot y + \text{contribution}(n), \qquad y^{32} = 0.5 $$

Implemented with integer arithmetic and a precomputed table (`decay_load()`,
`runnable_avg_yN_inv[]`). Each `sched_entity` and each `cfs_rq` tracks:

- `util_avg` — fraction of time actually **running** (0..1024). Drives **cpufreq** (schedutil).
- `load_avg` — `util` weighted by priority. Drives **load balancing**.
- `runnable_avg` — fraction of time **runnable** (including waiting). Detects contention.

Why an EWMA and not a simple average? Because you need **recency** (a task that just became
busy should be seen as busy) with **noise immunity** (one idle tick shouldn't collapse the
estimate). The half-life is the tuning knob, and 32 ms was chosen empirically against mobile
workloads.

**The key insight that makes `schedutil` work:** the *scheduler* knows a task's utilization
before the CPU frequency governor could ever measure it — the scheduler knows a task is
about to run. So frequency selection moved *into* the scheduler (`cpufreq_update_util()`),
replacing the old sampling governors (`ondemand`) that reacted after the fact. That is a
layering violation (Ch. 02 T.2) made deliberately, for a large win.

`util_est` refines this further: remember the utilization a task had *last time it ran* so a
periodically-sleeping task doesn't get its estimate decayed to zero while asleep — the
"periodic task looks idle" pathology.

### T.6 Real-time scheduling theory

`SCHED_FIFO` / `SCHED_RR` implement **fixed-priority preemptive scheduling**. The classical
theory is Liu & Layland (JACM 1973):

- **Rate Monotonic (RM)**: assign priority by *rate* (shorter period = higher priority).
  RM is the **optimal fixed-priority** assignment. Schedulability bound for $n$ tasks:
  $$ U = \sum_i \frac{C_i}{T_i} \le n(2^{1/n} - 1) \xrightarrow[n\to\infty]{} \ln 2 \approx 0.693 $$
  So with fixed priorities you can only guarantee ~69% utilization in the worst case.

- **Earliest Deadline First (EDF)**: dynamic priority by absolute deadline. EDF is
  **optimal among all** scheduling algorithms on one CPU, with bound
  $$ U = \sum_i \frac{C_i}{T_i} \le 1 $$
  i.e. **100% utilization is schedulable**.

So why does anyone use fixed priority? Because EDF degrades catastrophically on overload
(a single task overrunning can cause a **domino effect** of missed deadlines), and because
fixed priority is simpler to reason about and implement. Linux offers both.

**`SCHED_DEADLINE`** implements EDF plus the **Constant Bandwidth Server** (CBS, Abeni &
Buttazzo, 1998), which solves the overload problem: each task declares

```c
struct sched_attr attr = {
	.sched_policy   = SCHED_DEADLINE,
	.sched_runtime  =  10 * 1000 * 1000,   /* 10 ms of CPU  */
	.sched_deadline =  50 * 1000 * 1000,   /* within 50 ms  */
	.sched_period   = 100 * 1000 * 1000,   /* every 100 ms  */
};
sched_setattr(0, &attr, 0);
```
The CBS **enforces** the budget: a task that exceeds `sched_runtime` in a period is
throttled until its next period. Therefore **a misbehaving task cannot damage others** —
temporal isolation. And the kernel performs **admission control**: `sched_setattr()` returns
`-EBUSY` if adding this task would exceed the schedulable bound. That is a genuinely rare
property — an OS interface that *refuses* a request it cannot honour.

`SCHED_DEADLINE` > `SCHED_FIFO` in priority, so a deadline task preempts all RT tasks.

**Priority inversion** (Ch. 14 T.8) is the other half of RT theory: `rt_mutex` implements
priority inheritance with full transitive chain walking, and PI futexes
(`FUTEX_LOCK_PI`) extend it to userspace.

### T.7 SMP: the scheduler is really a distributed system

With $N$ CPUs there is no global runqueue (that would be the ultimate contended cache line —
Ch. 16 T.1). Instead: **one runqueue per CPU**, and a separate mechanism to keep them
balanced. That makes the scheduler a *distributed* system with all the attendant problems.

**Scheduling domains** encode the hardware topology as a hierarchy, because migration cost
is not uniform:

```
   NUMA domain        (cross-socket:  ~300 ns memory, full cache loss)
     └─ MC domain     (shared LLC:    cheap-ish, L3 retained)
          └─ SMT domain (same core:   nearly free, L1/L2 shared)
```

Balancing runs at each level with a different frequency and a different `imbalance_pct` —
you balance aggressively within a core, reluctantly across sockets. This is *cost-aware
work stealing*, and the parameters come straight from the memory hierarchy:

```bash
ls /sys/kernel/debug/sched/domains/cpu0/
for d in /sys/kernel/debug/sched/domains/cpu0/domain*/; do
  echo "$(cat $d/name): flags=$(cat $d/flags) min=$(cat $d/min_interval) max=$(cat $d/max_interval)"
done
cat /proc/schedstat | head -20
```

Three balancing mechanisms:

1. **Wakeup placement** (`select_task_rq_fair`) — the most important for latency. On wakeup,
   pick a CPU: prefer an idle sibling sharing cache with the waker (`wake_affine`), else
   search the LLC domain for an idle CPU (`select_idle_sibling`). This search is
   latency-critical and is bounded by `SIS_UTIL` heuristics, because scanning 128 CPUs on
   every wakeup is itself a cost.
2. **Periodic load balancing** (`run_rebalance_domains`, in `SCHED_SOFTIRQ`) — walk the
   domain hierarchy, find the busiest group, pull tasks.
3. **Idle balancing** (`newidle_balance`) — a CPU about to go idle tries to steal work
   first. Bounded by `sysctl_sched_migration_cost` to avoid wasting more time searching than
   the work is worth.

**NUMA balancing** (`CONFIG_NUMA_BALANCING`) is a separate feedback loop: periodically unmap
pages to force faults, use the faults to learn which node a task actually touches, then
migrate either the task or the pages. This is *profile-guided placement at runtime* —
elegant, but with real overhead, which is why it is tunable and often disabled for
latency-sensitive work.

### T.8 Group scheduling and cgroups

`CONFIG_FAIR_GROUP_SCHED` makes the fairness hierarchy **recursive**: a `cfs_rq` can contain
`sched_entity`s that are themselves groups containing their own `cfs_rq`. Fairness is applied
at each level, so two cgroups with equal weight get 50% each regardless of how many tasks are
inside them.

```
                 root cfs_rq
                /           \
         [group A w=1024]  [group B w=1024]        ← 50/50 between groups
            /     \              |
        taskA1  taskA2       taskB1                ← A1 and A2 get 25% each
```

This is what makes container CPU limits meaningful. Two interfaces:

```bash
# cgroup v2
cat /sys/fs/cgroup/mygroup/cpu.weight        # 1..10000, default 100 (proportional)
cat /sys/fs/cgroup/mygroup/cpu.max           # "200000 100000" = 2 CPUs worth (hard cap)
cat /sys/fs/cgroup/mygroup/cpu.max.burst     # allow accumulated slack to be spent
cat /sys/fs/cgroup/mygroup/cpu.pressure      # PSI: how much time was LOST to contention
cat /sys/fs/cgroup/mygroup/cpu.stat          # nr_throttled, throttled_usec
cat /sys/fs/cgroup/mygroup/cpu.idle          # SCHED_IDLE for the whole group
```

**CFS bandwidth control** (`cpu.max`) is where production problems live: a quota is
distributed to per-CPU "slices", and a task that exhausts its CPU's slice is throttled even
if global quota remains. On a many-core machine with a small quota this causes surprising
latency spikes. `cpu.max.burst` (5.14+) mitigates it. **`nr_throttled` in `cpu.stat` is the
number to watch in any containerized production system.**

**PSI (Pressure Stall Information)** deserves emphasis: `some`/`full` pressure measures
*time lost to resource contention*, not utilization. 100% CPU utilization with zero pressure
means perfectly sized; 60% utilization with high pressure means badly distributed. It is a
far better signal than load average and is what modern autoscalers should use.

### T.9 `sched_ext`: policy as a loadable BPF program

Merged in **6.12**, `sched_ext` lets a BPF program implement `enqueue`/`dequeue`/`dispatch`/
`select_cpu` and become the scheduler for `SCHED_EXT` tasks.

Why this matters, in policy/mechanism terms (Ch. 00 T.1): scheduling policy is the classic
case where the "right" answer depends entirely on the workload, and a single in-kernel policy
must compromise. `sched_ext` moves **policy** to a loadable program while the kernel keeps
**mechanism** (runqueues, context switching, topology). Safety comes from the BPF verifier
plus a **watchdog that reverts to CFS/EEVDF** if the BPF scheduler stalls a task — so a bad
scheduler degrades rather than hangs the machine.

Real schedulers built on it: `scx_rusty` (multi-domain, load-balancing, written in Rust),
`scx_lavd` (latency-aware, for gaming), `scx_layered` (per-workload layers, used in
production at Meta), `scx_simple`. Being able to prototype a scheduler in an afternoon
changed what is researchable.

```bash
ls /sys/kernel/sched_ext/ 2>/dev/null
cat /sys/kernel/sched_ext/root/ops 2>/dev/null
$EDITOR Documentation/scheduler/sched-ext.rst
```

---

## 1. Internals

### 1.1 The core dispatch

```c
/* kernel/sched/core.c */
__schedule(unsigned int sched_mode)
{
	prev = rq->curr;
	...
	if (!preempt && prev_state)              /* voluntary sleep */
		deactivate_task(rq, prev, DEQUEUE_SLEEP);

	next = pick_next_task(rq, prev, &rf);    /* ★ walks sched_class list */

	if (likely(prev != next)) {
		rq->curr = next;
		++*switch_count;
		rq = context_switch(rq, prev, next, &rf);   /* ★ switch_mm + switch_to */
	}
}
```

`context_switch()` does two things: `switch_mm_irqs_off()` (change page tables — skipped for
kthreads and for same-`mm` threads, which is why thread switches are cheaper) and
`switch_to()` (arch asm: save/restore callee-saved registers and the stack pointer).

**Where `__schedule()` is called from:**
1. Voluntarily — a task blocks (`schedule()` inside `wait_event`, `mutex_lock`, I/O).
2. Preemption on return to user mode.
3. Preemption on return from interrupt to kernel mode (`CONFIG_PREEMPT`).
4. `cond_resched()` — explicit yield points in long kernel loops.
5. On wakeup, if the woken task should preempt the current one (`check_preempt_curr`).

**Preemption models** (6.12+ can switch at boot with `preempt=`):

| Model | Kernel preemptible? | Latency | Throughput |
|---|---|---|---|
| `PREEMPT_NONE` | no | worst | best |
| `PREEMPT_VOLUNTARY` | at `might_sleep()` points | better | good |
| `PREEMPT_LAZY` (6.13+) | yes, but deferred for SCHED_OTHER | good | good |
| `PREEMPT_FULL` | yes | good | slight cost |
| `PREEMPT_RT` | yes, incl. spinlocks & IRQs | best | measurable cost |

### 1.2 Source map

```
kernel/sched/core.c        ★ __schedule, try_to_wake_up, sched_setattr, context_switch
kernel/sched/fair.c        ★★ EEVDF/CFS: pick_next_entity, place_entity, load balancing
kernel/sched/rt.c          SCHED_FIFO/RR
kernel/sched/deadline.c    SCHED_DEADLINE: EDF + CBS + admission control
kernel/sched/ext.c         sched_ext (6.12+)
kernel/sched/pelt.c        PELT signals
kernel/sched/cpufreq_schedutil.c   frequency selection from util_avg
kernel/sched/topology.c    scheduling domains
kernel/sched/psi.c         pressure stall information
kernel/sched/sched.h       ★ struct rq, struct cfs_rq, the class definitions
kernel/sched/debug.c       /sys/kernel/debug/sched/*
include/uapi/linux/sched/types.h   struct sched_attr
Documentation/scheduler/   ★ sched-design-CFS.rst, sched-deadline.rst, sched-ext.rst,
                             sched-domains.rst, sched-bwc.rst, sched-energy.rst
```

### 1.3 Observability surface

```bash
# Per-task
cat /proc/<pid>/sched                    # vruntime, deadline, slice, nr_switches, wait_sum
cat /proc/<pid>/schedstat                # cpu_time, wait_time, nr_timeslices
cat /proc/<pid>/status | grep -E 'voluntary|nonvoluntary'
chrt -p <pid>                            # policy and priority
taskset -pc <pid>                        # affinity

# System
cat /proc/schedstat                       # per-CPU, per-domain balancing counters
ls /sys/kernel/debug/sched/               # features, domains, debug, latency_warn_ms
cat /sys/kernel/debug/sched/debug | head -60    # every runqueue, every task
cat /sys/kernel/debug/sched/features      # runtime-togglable scheduler features
cat /proc/pressure/cpu                    # PSI

# Tracing
sudo perf sched record -- sleep 5 && sudo perf sched latency --sort max
sudo perf sched timehist | head -40
sudo perf sched map | head -40            # a visual CPU-by-CPU timeline
sudo trace-cmd record -e sched -- sleep 3 && trace-cmd report | head -40
```

---

## 2. Practice

### Lab 21.1 — Prove proportional share, then break it with `nice` (T.2)

```bash
# Three CPU hogs pinned to ONE cpu, different nice values
taskset -c 3 nice -n  0 sh -c 'while :; do :; done' & A=$!
taskset -c 3 nice -n  5 sh -c 'while :; do :; done' & B=$!
taskset -c 3 nice -n 10 sh -c 'while :; do :; done' & C=$!
sleep 20
for p in $A $B $C; do
  echo "pid $p nice=$(ps -o ni= -p $p) cpu=$(ps -o %cpu= -p $p)"
done
kill $A $B $C
```
`nice` weights are a table (`sched_prio_to_weight[]`) with **each nice level worth ~1.25×**:
```bash
grep -A12 'sched_prio_to_weight\[\]' kernel/sched/core.c
```
Verify: nice 0 : nice 5 should be ≈ 1024 : 335 ≈ 3:1. Compute the predicted percentages and
compare with measured. **This is T.2, verified in 30 seconds.**

### Lab 21.2 — EEVDF: separate latency from bandwidth (T.4)

The headline claim of EEVDF. Demonstrate it.

```c
/* latbench.c — a "responsive" task competing with CPU hogs.
   gcc -O2 -o latbench latbench.c -lrt  */
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>
#include <sched.h>
#include <sys/syscall.h>
#include <linux/sched.h>
#include <linux/sched/types.h>

static double now_us(void) {
	struct timespec t; clock_gettime(CLOCK_MONOTONIC, &t);
	return t.tv_sec * 1e6 + t.tv_nsec / 1e3;
}

int main(int argc, char **argv)
{
	int latency_nice = argc > 1 ? atoi(argv[1]) : 0;
	double worst = 0, sum = 0;
	int i, n = 2000;

#ifdef SCHED_FLAG_UTIL_CLAMP
	/* Set latency_nice via sched_setattr (kernel >= 6.5) */
	struct sched_attr { __u32 size, sched_policy; __u64 sched_flags;
	                    __s32 sched_nice; __u32 sched_priority;
	                    __u64 sched_runtime, sched_deadline, sched_period;
	                    __u32 sched_util_min, sched_util_max;
	                    __s32 sched_latency_nice; } a;
	memset(&a, 0, sizeof(a));
	a.size = sizeof(a);
	a.sched_policy = SCHED_OTHER;
	a.sched_flags  = 0x20 /* SCHED_FLAG_LATENCY_NICE */;
	a.sched_latency_nice = latency_nice;
	if (syscall(SYS_sched_setattr, 0, &a, 0) < 0)
		perror("sched_setattr (latency_nice unsupported?)");
#endif

	for (i = 0; i < n; i++) {
		double t0 = now_us();
		struct timespec ts = { 0, 1000000 };     /* sleep 1 ms */
		nanosleep(&ts, NULL);
		double lat = now_us() - t0 - 1000.0;     /* wakeup latency */
		if (lat > worst) worst = lat;
		sum += lat;
	}
	printf("latency_nice=%3d  avg=%8.1f us  max=%8.1f us\n",
	       latency_nice, sum / n, worst);
	return 0;
}
```
```bash
# Background load: one hog per CPU
for c in $(seq 0 $(($(nproc)-1))); do taskset -c $c sh -c 'while :; do :; done' & done

./latbench   19    # high latency-nice  -> larger slice, later deadline
./latbench    0
./latbench  -20    # low  latency-nice  -> smaller slice, EARLIER deadline

# Verify total CPU share is UNCHANGED across all three (that's the point):
#   run each under `perf stat -e task-clock` and compare.
kill %1 %2 %3 %4 2>/dev/null; jobs -p | xargs -r kill
```
**Expected:** `max` latency drops substantially at `latency_nice=-20` while `task-clock`
(total CPU consumed) stays the same. That is $r_i$ shrinking the virtual deadline without
changing $w_i$ — T.4's central claim.

Also inspect the EEVDF state directly:
```bash
sudo cat /proc/$(pgrep -n latbench)/sched | grep -E 'vruntime|deadline|slice|vlag'
sudo cat /sys/kernel/debug/sched/debug | grep -A20 'cfs_rq\[3\]'
```

### Lab 21.3 — Watch PELT converge (T.5)

```bash
# A task with a 30% duty cycle
cat > /tmp/duty.sh <<'EOF'
while :; do
  end=$(( $(date +%s%N) + 30000000 ))   # 30 ms busy
  while [ $(date +%s%N) -lt $end ]; do :; done
  sleep 0.07                            # 70 ms idle
done
EOF
taskset -c 2 bash /tmp/duty.sh & P=$!

# util_avg should converge to ~0.3 * 1024 ≈ 307
for i in $(seq 1 20); do
  sudo grep -E 'se.avg.util_avg|se.avg.load_avg|se.avg.util_est' /proc/$P/sched | tr '\n' ' '
  echo; sleep 1
done
kill $P

# And the frequency response:
sudo bpftrace -e 'tracepoint:power:cpu_frequency { @[args->cpu_id] = hist(args->state/1000); }'
cat /sys/devices/system/cpu/cpu2/cpufreq/scaling_governor    # should be schedutil
sudo trace-cmd record -e sched:sched_pelt_se -e power:cpu_frequency -- sleep 10
trace-cmd report | head -30
```
Change the duty cycle to 10%, 50%, 90% and confirm `util_avg` tracks it. Then make it sleep
for 500 ms and observe `util_est` preserving the estimate while `util_avg` decays.

### Lab 21.4 — SCHED_DEADLINE with admission control (T.6)

```c
/* dl.c — gcc -O2 -o dl dl.c */
#define _GNU_SOURCE
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <time.h>
#include <errno.h>
#include <sys/syscall.h>
#include <linux/types.h>

struct sched_attr {
	__u32 size, sched_policy;  __u64 sched_flags;
	__s32 sched_nice;          __u32 sched_priority;
	__u64 sched_runtime, sched_deadline, sched_period;
};
#define SCHED_DEADLINE 6

int main(int argc, char **argv)
{
	struct sched_attr a;
	__u64 runtime = (argc > 1 ? atoll(argv[1]) : 10) * 1000000ULL;   /* ms */
	int i;

	memset(&a, 0, sizeof(a));
	a.size            = sizeof(a);
	a.sched_policy    = SCHED_DEADLINE;
	a.sched_runtime   = runtime;
	a.sched_deadline  =  50 * 1000000ULL;
	a.sched_period    = 100 * 1000000ULL;

	if (syscall(SYS_sched_setattr, 0, &a, 0) < 0) {
		printf("sched_setattr FAILED: %s  <-- ADMISSION CONTROL\n", strerror(errno));
		return 1;
	}
	printf("admitted: %llu ms every 100 ms\n", (unsigned long long)runtime / 1000000);

	for (i = 0; i < 50; i++) {
		struct timespec t0, t1;
		clock_gettime(CLOCK_MONOTONIC, &t0);
		do { clock_gettime(CLOCK_MONOTONIC, &t1); }
		while ((t1.tv_sec - t0.tv_sec) * 1e9 + (t1.tv_nsec - t0.tv_nsec) < runtime * 0.8);
		sched_yield();                     /* give back the rest of the period */
	}
	return 0;
}
```
```bash
sudo ./dl 10    # 10% -> admitted
sudo ./dl 40    # 40% -> admitted
sudo ./dl 95    # 95% -> likely -EBUSY (admission control!)

# Run several at once until admission fails:
for i in 1 2 3 4 5 6 7 8 9 10; do sudo ./dl 10 & done

# Watch the CBS throttle an overrunning task:
sudo bpftrace -e 'kprobe:dl_runtime_exceeded { @throttles = count(); }'
sudo cat /sys/kernel/debug/sched/debug | grep -A10 dl_rq
cat /proc/sys/kernel/sched_rt_runtime_us /proc/sys/kernel/sched_rt_period_us
```
Then **prove temporal isolation**: make one deadline task deliberately overrun its runtime
and show that the *other* deadline tasks still meet their deadlines. That is CBS working, and
it is the property `SCHED_FIFO` cannot give you.

### Lab 21.5 — RT priorities and the starvation safety valve

```bash
# A SCHED_FIFO spinner will monopolize its CPU...
sudo chrt -f 50 taskset -c 3 sh -c 'while :; do :; done' & R=$!
taskset -c 3 sh -c 'while :; do :; done' & N=$!
sleep 10
ps -o pid,cls,rtprio,pcpu,comm -p $R $N

# ...but only up to sched_rt_runtime_us / sched_rt_period_us
cat /proc/sys/kernel/sched_rt_runtime_us   # 950000 of 1000000 = 95%
sudo sysctl -w kernel.sched_rt_runtime_us=500000   # now only 50%
sleep 10; ps -o pid,cls,rtprio,pcpu,comm -p $R $N
sudo sysctl -w kernel.sched_rt_runtime_us=950000
kill $R $N

# Priority inversion + inheritance, live:
sudo bpftrace -e 'kprobe:rt_mutex_setprio { printf("PI boost: %s -> prio %d\n", comm, arg1); }'
```

### Lab 21.6 — Diagnose a latency problem with `perf sched`

```bash
# Record everything scheduling-related under your workload
sudo perf sched record -- your-workload
sudo perf sched latency --sort max | head -30        # who waits longest?
sudo perf sched timehist -Vw | head -50              # per-event timeline with wait times
sudo perf sched map | head -40                       # visual: which task on which CPU

# The three numbers that matter for a task:
cat /proc/<pid>/schedstat     # <cpu_time_ns> <wait_time_ns> <nr_timeslices>
#   wait_time / (wait_time + cpu_time)  ≈ how much of your latency is scheduling

# Per-CPU runqueue latency histogram:
sudo bpftrace -e '
tracepoint:sched:sched_wakeup      { @w[args->pid] = nsecs; }
tracepoint:sched:sched_wakeup_new  { @w[args->pid] = nsecs; }
tracepoint:sched:sched_switch /@w[args->next_pid]/ {
	@runqlat_us = hist((nsecs - @w[args->next_pid])/1000);
	delete(@w[args->next_pid]); }'
# (this is exactly bcc's runqlat)
sudo /usr/share/bcc/tools/runqlat 5 2
sudo /usr/share/bcc/tools/runqslower 10000      # tasks waiting > 10 ms

# Involuntary context switches = preemption pressure
cat /proc/<pid>/status | grep -i switches
```

### Lab 21.7 — Scheduling domains and migration cost (T.7)

```bash
# Read your machine's topology as the scheduler sees it
lscpu -e
lstopo-no-graphics --of console 2>/dev/null | head -30
for d in /sys/kernel/debug/sched/domains/cpu0/domain*/; do
  echo "$(basename $d): name=$(cat $d/name) flags=$(cat $d/flags)"
  echo "   busy=$(cat $d/busy_factor) imbalance=$(cat $d/imbalance_pct) cache_nice=$(cat $d/cache_nice_tries)"
done

# Measure migration cost directly: ping-pong two threads,
# (a) on the same core (SMT siblings), (b) same LLC, (c) different sockets
cat /sys/devices/system/cpu/cpu0/topology/thread_siblings_list
cat /sys/devices/system/cpu/cpu0/topology/core_siblings_list
numactl --hardware

# lmbench does this properly:
lat_ctx -s 0 2 ; taskset -c 0,1 lat_ctx -s 0 2 ; taskset -c 0,16 lat_ctx -s 0 2

# Watch migrations happen
sudo bpftrace -e 'tracepoint:sched:sched_migrate_task {
	@[args->orig_cpu, args->dest_cpu] = count(); }'
cat /proc/schedstat | awk '/^cpu/{cpu=$1} /^domain/{print cpu, $0}' | head -20

sysctl kernel.sched_migration_cost_ns      # below this, a task is "cache hot"
```

### Lab 21.8 — cgroup CPU control and the throttling trap (T.8)

```bash
sudo mkdir -p /sys/fs/cgroup/demo
echo "50000 100000" | sudo tee /sys/fs/cgroup/demo/cpu.max     # 0.5 CPU
echo 100           | sudo tee /sys/fs/cgroup/demo/cpu.weight

# Put a multi-threaded hog in it — THIS is where the trap bites
( echo $BASHPID | sudo tee /sys/fs/cgroup/demo/cgroup.procs >/dev/null
  for i in $(seq 1 $(nproc)); do sh -c 'while :; do :; done' & done
  sleep 20; jobs -p | xargs -r kill ) &

watch -n1 'cat /sys/fs/cgroup/demo/cpu.stat; echo ---; cat /sys/fs/cgroup/demo/cpu.pressure'
```
**Watch `nr_throttled` and `throttled_usec` climb.** With 16 threads sharing 0.5 CPU of
quota, per-CPU slice distribution causes throttling far more often than the average would
suggest. Then:
```bash
echo 20000 | sudo tee /sys/fs/cgroup/demo/cpu.max.burst    # allow bursting
# and compare nr_throttled
$EDITOR Documentation/scheduler/sched-bwc.rst
```
**This is the single most common performance problem in containerized production systems.**
Be able to diagnose it from `cpu.stat` alone.

### Lab 21.9 — Toggle scheduler features at runtime

```bash
cat /sys/kernel/debug/sched/features
# e.g.  GENTLE_FAIR_SLEEPERS START_DEBIT NEXT_BUDDY LAST_BUDDY CACHE_HOT_BUDDY
#       WAKEUP_PREEMPTION HRTICK SIS_UTIL RT_PUSH_IPI ...

# Turn one off and measure:
echo NO_NEXT_BUDDY | sudo tee /sys/kernel/debug/sched/features
./latbench 0
echo NEXT_BUDDY | sudo tee /sys/kernel/debug/sched/features

# Other live knobs:
ls /sys/kernel/debug/sched/
cat /sys/kernel/debug/sched/latency_warn_ms
cat /sys/kernel/debug/sched/base_slice_ns       # EEVDF's default request size r_i
sudo sysctl -a | grep ^kernel.sched
```
Pick three features, read what they do in `kernel/sched/features.h`, predict the effect of
disabling each, then measure. **Prediction-before-measurement is the discipline that builds
real intuition.**

### Lab 21.10 — Write a scheduler with `sched_ext` (T.9)

```bash
# Kernel: CONFIG_SCHED_CLASS_EXT=y, plus BPF + BTF
./scripts/config -e SCHED_CLASS_EXT -e BPF_SYSCALL -e BPF_JIT -e DEBUG_INFO_BTF
make -j$(nproc)

# Userspace schedulers:
git clone https://github.com/sched-ext/scx && cd scx
meson setup build --prefix ~ && meson compile -C build
sudo ./build/scheds/c/scx_simple
# in another terminal:
cat /sys/kernel/sched_ext/root/ops
chrt -p $$            # tasks now show SCHED_EXT
sudo ./build/scheds/rust/scx_rusty/debug/scx_rusty
```
Then read `tools/sched_ext/scx_simple.bpf.c` (~150 lines — a complete working scheduler) and
modify it: make it strictly FIFO, then strictly LIFO, and measure the latency difference on
your workload. **Being able to prototype a scheduling policy in an hour is a genuinely new
capability.**

```bash
$EDITOR Documentation/scheduler/sched-ext.rst
$EDITOR tools/sched_ext/scx_simple.bpf.c
```

### Lab 21.11 — Energy-aware scheduling (big.LITTLE)

On an arm64 board with heterogeneous cores (or a QEMU model):
```bash
cat /sys/devices/system/cpu/cpu*/cpu_capacity
cat /sys/kernel/debug/sched/debug | grep -i 'capacity\|energy'
ls /sys/devices/system/cpu/cpu0/cpufreq/
cat /sys/kernel/debug/energy_model/*/ps:*/    2>/dev/null

# uclamp: force a task onto a big core by raising its utilization floor
sudo ./util_clamp_tool --pid $$ --min 800      # or via sched_setattr SCHED_FLAG_UTIL_CLAMP
$EDITOR Documentation/scheduler/sched-energy.rst
$EDITOR Documentation/scheduler/sched-util-clamp.rst
```

---

## 3. Mastery drills

1. **Derive EEVDF.** Starting from the definition of lag in T.2, show why "run the eligible
   task with the earliest virtual deadline" bounds lag by $\max_i r_i$. Then find
   `entity_eligible()` and `pick_eevdf()` in `kernel/sched/fair.c` and map each line to the
   formula.

2. **Read the EEVDF merge.** `git log --oneline --grep='EEVDF' -- kernel/sched/ | head -30`,
   then find Peter Zijlstra's original posting on lore.kernel.org. List every CFS heuristic
   the series *deleted*. Explain, for three of them, what problem they had been patching and
   why EEVDF makes them unnecessary.

3. **Stride scheduling.** Read Waldspurger's stride scheduling paper. Prove that CFS's
   `vruntime` update is exactly stride scheduling's `pass += stride`. What is CFS's `K`?

4. **Liu & Layland.** For a task set with periods (10, 20, 50) ms and WCETs (3, 4, 10) ms:
   compute $U$; determine RM schedulability by the bound *and* by exact response-time
   analysis; determine EDF schedulability. Then actually run it with `SCHED_FIFO` (RM
   priorities) and with `SCHED_DEADLINE`, and see whether theory matches practice.

5. **The wakeup path.** Read `try_to_wake_up()` and `select_task_rq_fair()`. Enumerate every
   decision point in CPU selection. Explain `wake_affine_idle`, `select_idle_sibling`, and
   `SIS_UTIL`. Why is scanning for an idle CPU itself a scalability problem?

6. **PELT arithmetic.** Read `kernel/sched/pelt.c`. Explain `decay_load()`, the
   `runnable_avg_yN_inv[]` table, and why the half-life is 32 ms. What breaks if you change
   it? (`git log --grep='PELT half' --oneline`.)

7. **Group scheduling recursion.** Draw the `sched_entity`/`cfs_rq` structure for: root →
   two cgroups → three tasks each. Show where each `vruntime` lives and how throttling at
   the group level propagates. Then verify with `/sys/kernel/debug/sched/debug`.

8. **CFS bandwidth pathology.** Explain, precisely, why a 16-thread application with
   `cpu.max = 200000 100000` (2 CPUs) gets throttled even though it uses less than 2 CPUs on
   average. Read `sched-bwc.rst` and `distribute_cfs_runtime()`. Propose two fixes and
   evaluate each.

9. **PSI vs load average.** Explain why load average is a poor signal and what PSI measures
   instead. Construct two workloads with the same load average but very different
   `/proc/pressure/cpu`. Read `kernel/sched/psi.c`.

10. **Priority inversion end to end.** Construct the three-task inversion from Ch. 14 T.8
    using `SCHED_FIFO` and a plain futex; measure the high-priority task's delay. Then
    switch to a PI futex (`pthread_mutexattr_setprotocol(PTHREAD_PRIO_INHERIT)`) and
    re-measure. Explain the mechanism via `rt_mutex_setprio()`.

11. **Load balancing cost.** Find `sysctl_sched_migration_cost_ns`. Explain what it trades.
    Then measure: pin a workload's threads so they *must* migrate, vs. letting them stay,
    and compare throughput and LLC miss rate (`perf stat -e LLC-load-misses`).

12. **Write a `sched_ext` scheduler** that implements a policy the in-kernel scheduler cannot:
    e.g. strict priority to one cgroup, or gang scheduling for a set of threads. Measure it
    against EEVDF on a workload where your policy should win. Write up the result.

13. **Design question.** A latency-sensitive network service and a batch analytics job share
    a 64-core machine. The service needs p99 < 500 µs; the batch job should use everything
    left over. Design the scheduling configuration. Consider: cgroup weights vs quotas,
    `SCHED_IDLE` for batch, `latency_nice`, `nohz_full` + `isolcpus` for the service,
    IRQ affinity (Ch. 17 T.8), and `cpu.pressure` as the feedback signal. Justify each
    choice and state what you would measure to validate it.

---

## 4. Further reading

**Kernel documentation (unusually good here):**
- `Documentation/scheduler/sched-design-CFS.rst` — still the best conceptual intro
- `Documentation/scheduler/sched-eevdf.rst` (6.12+)
- `Documentation/scheduler/sched-deadline.rst` ★★ — contains the CBS/EDF theory *properly*,
  with references. One of the best documents in the kernel.
- `Documentation/scheduler/sched-domains.rst`, `sched-capacity.rst`, `sched-energy.rst`
- `Documentation/scheduler/sched-bwc.rst` — CFS bandwidth control and its pathologies
- `Documentation/scheduler/sched-ext.rst`, `sched-util-clamp.rst`, `sched-rt-group.rst`
- `Documentation/accounting/psi.rst`
- `Documentation/admin-guide/cgroup-v2.rst` (CPU controller section)

**Papers (this is a rare area where the papers are directly load-bearing):**
- Stoica & Abdel-Wahab, "Earliest Eligible Virtual Deadline First: A Flexible and Accurate
  Mechanism for Proportional Share Resource Allocation" (1995) — **the EEVDF paper**
- Waldspurger & Weihl, "Lottery Scheduling: Flexible Proportional-Share Resource Management"
  (OSDI 1994)
- Waldspurger, "Lottery and Stride Scheduling" (PhD thesis, MIT 1995) — stride = CFS
- Liu & Layland, "Scheduling Algorithms for Multiprogramming in a Hard-Real-Time
  Environment" (JACM 1973) — **RM, EDF, and the utilization bounds**
- Abeni & Buttazzo, "Integrating Multimedia Applications in Hard Real-Time Systems"
  (RTSS 1998) — the Constant Bandwidth Server
- Demers, Keshav & Shenker, "Analysis and Simulation of a Fair Queueing Algorithm"
  (SIGCOMM 1989) — where proportional share came from
- Lozi et al., "The Linux Scheduler: a Decade of Wasted Cores" (EuroSys 2016) — four real
  load-balancing bugs, found with visualization tools. **Read this; it teaches you how to
  find scheduler bugs.**
- Blagodurov et al., "A Case for NUMA-aware Contention Management" (USENIX ATC 2011)
- Sha, Rajkumar & Lehoczky, "Priority Inheritance Protocols" (1990) — as in Ch. 14

**Books:**
- Buttazzo, *Hard Real-Time Computing Systems* — the RT theory in T.6, done properly
- Love, *Linux Kernel Development*, Ch. 4 (Process Scheduling) — pre-EEVDF but the framing
  holds
- Gregg, *Systems Performance*, 2nd ed., Ch. 6 (CPUs) — the measurement side

**LWN (the scheduler is the best-covered subsystem on LWN):**
- "An EEVDF CPU scheduler for Linux" (Corbet, 2023) — the clearest explanation available
- "Completing the EEVDF scheduler" (2024)
- "Per-entity load tracking" (2013)
- "The pluggable CPU scheduler revisited" / "The extensible scheduler class" (sched_ext)
- "CPU scheduling for real-time: SCHED_DEADLINE"
- "Scheduling for the tail" / "Latency nice"
- "Realtime response, virtual machines, and the scheduler"

**Tools:**
- `sched-ext` schedulers: https://github.com/sched-ext/scx
- `perf sched`, `runqlat`/`runqslower` (bcc), `trace-cmd`, KernelShark
- `rt-tests` (`cyclictest`, `hackbench`, `rt-migrate-test`)

→ Next: [22-mm-internals-1.md](22-mm-internals-1.md)

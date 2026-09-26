# Chapter 19 — Time: Clocksources, Timekeeping, Timer Wheels, hrtimers, NO_HZ

> **Goal:** understand why the kernel has *seven* notions of "time", why the timer wheel is
> O(1) and the hrtimer tree is O(log n) *on purpose*, and how a tickless kernel can keep
> accurate time while doing nothing for ten seconds.

---

## Theory & First Principles

### T.0 — Start here: five clocks, and why your timeout is wrong

Ask the system what time it is. There is no single answer:

```c
struct timespec ts;
clock_gettime(CLOCK_REALTIME,       &ts);  /* wall clock. JUMPS. */
clock_gettime(CLOCK_MONOTONIC,      &ts);  /* since boot. Never jumps. STOPS in suspend. */
clock_gettime(CLOCK_BOOTTIME,       &ts);  /* since boot, INCLUDING suspend. */
clock_gettime(CLOCK_MONOTONIC_RAW,  &ts);  /* monotonic, NOT NTP-adjusted. */
clock_gettime(CLOCK_PROCESS_CPUTIME_ID, &ts); /* CPU time this process burned. */
```

**Now the bug.** Write a five-second timeout the obvious way:

```c
time_t deadline = time(NULL) + 5;          /* CLOCK_REALTIME */
while (time(NULL) < deadline)
	do_work();
```

Three ways this fails in production, all of which have caused real outages:

| Event | Result |
|---|---|
| NTP steps the clock **back** 3 seconds | Your 5-second timeout becomes 8 seconds |
| NTP steps it **forward** 10 seconds | Your timeout fires **immediately** |
| A **leap second** is inserted | 2012: a kernel bug here hung Reddit, Mozilla, Qantas, and a large fraction of the internet simultaneously |

**The rule, and it is absolute:**

> **`CLOCK_REALTIME` is for telling humans what time it is. `CLOCK_MONOTONIC` is for
> measuring intervals. Never use the first for the second.**

And the second-order question, which is where judgement enters: should a 5-second timeout
fire after a 3-hour suspend? `CLOCK_MONOTONIC` stops during suspend, so it fires 5 seconds
after resume. `CLOCK_BOOTTIME` keeps running, so it fires immediately. **Neither is right in
general** — a network timeout wants the first, a "check for updates daily" wants the second.
The kernel provides both because the question has no universal answer, and knowing which you
need is the skill.

**Now the second half of the chapter, which is a different problem entirely.** Inside the
kernel, timers are not primarily about time — they are about *cancellation*:

```c
mod_timer(&tcp_retransmit_timer, jiffies + rto);
/* … the ACK arrives 2 ms later … */
timer_delete(&tcp_retransmit_timer);      /* cancelled. It NEVER FIRED. */
```

**The overwhelming majority of kernel timers are cancelled before they expire.** Every TCP
retransmit timer, every I/O timeout, every watchdog on a healthy system. That single fact
determines the data structure:

| | timer wheel (`timer_list`) | `hrtimer` |
|---|---|---|
| Structure | hashed buckets, cascading | rbtree ordered by absolute time |
| Insert / cancel | **O(1)** | O(log n) |
| Resolution | jiffies (1–10 ms), inexact | nanoseconds, exact |
| Optimized for | **cancellation** | **expiry** |
| Used by | timeouts | `nanosleep`, scheduler, RT |

> **Optimize the operation that actually dominates.** For timeouts that is cancellation, not
> expiry — so the wheel accepts imprecision and cascading to make insert and delete O(1).

That principle (Ch. 89 §T.1, #6) is why the kernel has *two* timer subsystems rather than one
good one, and it is a better answer to "why are there two?" than any amount of history.

```bash
cat /proc/timer_list | head -40      # every armed timer, both kinds
cat /sys/devices/system/clocksource/clocksource0/available_clocksource
cat /sys/devices/system/clocksource/clocksource0/current_clocksource   # usually tsc
```

---

### T.1 There is no such thing as "the time"

An operating system must provide several mutually incompatible notions of time, and
conflating them is a classic source of production bugs.

| Clock | Property | Breaks when | Use for |
|---|---|---|---|
| **Monotonic** (`CLOCK_MONOTONIC`) | never jumps, never goes backwards; stops during suspend | — | **measuring intervals**, timeouts |
| **Boottime** (`CLOCK_BOOTTIME`) | monotonic **including** suspend time | — | timeouts that must survive suspend |
| **Realtime** (`CLOCK_REALTIME`) | wall-clock, UTC; can jump forward/backward | NTP steps, `settimeofday`, leap seconds | timestamps, file times, user display |
| **TAI** (`CLOCK_TAI`) | realtime without leap seconds | — | PTP, precise scheduling |
| **Raw monotonic** (`CLOCK_MONOTONIC_RAW`) | hardware counter, **not** NTP-disciplined | — | measuring the discipline itself |
| **Process/thread CPU time** | consumed CPU, not elapsed | — | accounting, `getrusage` |
| **jiffies** | tick counter; coarse (1–10 ms) | wraps | cheap, coarse timeouts |

**The rule that prevents most bugs:** *never measure an interval with a realtime clock.*
`CLOCK_REALTIME` can move backwards (NTP step, admin change), which makes `end - start`
negative and every derived computation wrong. In-kernel this means `ktime_get()` (monotonic),
not `ktime_get_real()`. There is a whole genre of CVEs and outages from this.

**Suspend semantics** are the second trap. `CLOCK_MONOTONIC` **stops** across suspend-to-RAM.
A 60-second timeout armed before a 3-hour suspend fires 60 seconds after resume, not
immediately. If you need "60 seconds of wall time regardless", use `CLOCK_BOOTTIME` /
`ktime_get_boottime()`, or an alarm timer (`alarmtimer`, which can also *wake the system*).

### T.2 Timekeeping is a control system

The hardware gives you a free-running counter (TSC, ARM architected timer, HPET). It ticks at
some nominal frequency, but:

- the crystal has an error of tens to hundreds of ppm (1 ppm ≈ 86 ms/day drift),
- the frequency varies with temperature and voltage,
- multiple CPUs' counters may not be perfectly synchronized.

So the kernel runs a **phase-locked loop** disciplined by NTP/PTP (`kernel/time/ntp.c`,
implementing the NTP kernel discipline of Mills, RFC 1589/5905):

```
       hardware counter (cycles)
              │ × mult >> shift          ← the discipline knobs
              ▼
        nanoseconds since boot
              │ + wall_to_monotonic + offsets
              ▼
    CLOCK_MONOTONIC / CLOCK_REALTIME / CLOCK_TAI
```

`adjtimex()` feeds a frequency correction (`freq`) and a phase correction (`offset`) into the
loop. The kernel applies **frequency correction by adjusting `mult`** — i.e. it *slews* rather
than *steps*, because stepping would violate monotonicity. Only a large error causes a step,
and only on `CLOCK_REALTIME`.

Why `mult`/`shift` instead of floating point: the kernel has no FP in most contexts, and this
form makes `cycles → ns` a multiply and a shift with bounded rounding error. Choosing `shift`
is a precision-vs-overflow trade computed in `clocks_calc_mult_shift()` — a nice small piece
of numerical engineering worth reading.

**Leap seconds** are where this gets ugly: UTC occasionally inserts a 61st second. Linux
historically handled this by repeating a second, which broke assumptions all over user space
(the famous 2012 leap-second outage that took down Reddit, Mozilla, and others via a
futex/hrtimer interaction). Modern practice is **leap smearing** in NTP userspace and
`CLOCK_TAI` for anything that cares. Know this story; it is a favourite interview question
about why monotonic clocks exist.

### T.3 The clocksource/clockevent split

Linux separates two orthogonal hardware roles — a genuinely clarifying abstraction:

- **Clocksource** — *"what time is it?"* A monotonically increasing counter you can read.
  Rated by `.rating` (0–499); the highest-rated usable one wins.
- **Clockevent device** — *"interrupt me at time T."* A programmable comparator/one-shot.

```bash
cat /sys/devices/system/clocksource/clocksource0/available_clocksource
cat /sys/devices/system/clocksource/clocksource0/current_clocksource
# typical x86: tsc hpet acpi_pm
cat /sys/devices/system/clockevents/clockevent0/current_device
dmesg | grep -iE 'clocksource|tsc|clockevent'
```

**The TSC saga** is essential background. x86's Time Stamp Counter is the fastest possible
clocksource (a single `rdtsc`, ~20 cycles, usable from userspace via the vDSO), but
historically it was unusable because it varied with CPU frequency, stopped in deep C-states,
and drifted between sockets. Modern CPUs advertise `constant_tsc`, `nonstop_tsc`
(a.k.a. invariant TSC), which restores it. The kernel *watchdogs* the TSC against HPET/ACPI-PM
and will demote it at runtime if it misbehaves:

```
clocksource: timekeeping watchdog on CPU3: Marking clocksource 'tsc' as unstable
```
If you ever see this on a production box, your `clock_gettime()` just went from ~20 ns to
~500 ns, and everything slowed down. It is a classic "mysterious performance cliff".

**The vDSO** matters here: `clock_gettime()` is implemented *in userspace* by reading the
clocksource directly from a shared page, with no syscall at all — provided the clocksource is
vDSO-capable (TSC, ARM `CNTVCT_EL0`). That's a 20 ns call instead of a 300 ns syscall. This
is why "the clocksource fell back to HPET" is catastrophic for tracing/benchmarking workloads.

### T.4 Timer wheels: the O(1) algorithm you must know

This is the most important algorithm in the chapter.

The problem: maintain a set of `N` pending timers supporting `insert`, `cancel`, and
`expire_all_due(now)`, called on every tick. Naive options:

| Structure | Insert | Cancel | Expire |
|---|---|---|---|
| Sorted list | O(N) | O(1) | O(1) |
| Min-heap | O(log N) | O(log N) | O(log N) |
| Balanced tree | O(log N) | O(log N) | O(log N) |
| **Timing wheel** | **O(1)** | **O(1)** | **O(1) amortized** |

**Varghese & Lauck, "Hashed and Hierarchical Timing Wheels" (SOSP 1987)** is the paper.
The insight: a timer's expiry is a *time*, and time advances at a known rate. So use time
itself as a hash function.

A single wheel of `W` buckets: a timer expiring in `d` ticks goes in bucket
`(now + d) mod W`. Each tick, advance the cursor and fire everything in that bucket.
Insert = compute an index = O(1). Expire = walk one bucket = O(1) amortized.
Limitation: only handles `d < W`.

**Hierarchical wheels** handle arbitrary `d` with multiple levels of decreasing resolution:

```
Level 0: granularity 1 tick      covers     0 – 63 ticks
Level 1: granularity 8 ticks     covers    64 – 511
Level 2: granularity 64 ticks    covers   512 – 4095
...
Level 8: granularity ~2.7 min    covers up to weeks
```

Linux (`kernel/time/timer.c`) uses `LVL_DEPTH = 8` (9 on 1000 Hz), `LVL_SIZE = 64` buckets per
level, `LVL_CLK_SHIFT = 3` (×8 granularity per level).

**The crucial design decision (4.8, Thomas Gleixner):** the old implementation **cascaded** —
when a higher-level bucket expired, its timers were redistributed down to lower levels, an
O(N) operation with unbounded latency spikes. The current implementation **does not cascade**.
Instead, a timer placed in a coarse level simply fires with that level's granularity, i.e.
**it may fire late, by up to the granularity of its level**.

That is a profound API change justified by an empirical observation:

> **The overwhelming majority of kernel timers are cancelled before they expire.**
> They are *timeouts*, not *alarms*: "if the device doesn't respond in 5 seconds, error out."
> The normal case is the device responds and the timer is removed.

So the design optimizes for **insert and cancel being O(1) and cache-friendly**, and accepts
imprecision on expiry — which is exactly wrong for a real alarm and exactly right for a
timeout. Hence the split:

> **`timer_list` = imprecise timeouts (may fire late). `hrtimer` = precise alarms.**

If you need precision, you must use an hrtimer. If you use a `timer_list` and complain that it
fired 30 ms late, you have misunderstood the contract.

### T.5 hrtimers: precision costs O(log n)

`hrtimer` (Thomas Gleixner & Ingo Molnar, 2006) makes the opposite trade:

- Stored in a **timerqueue**: a red-black tree ordered by expiry, with a cached leftmost node
  (`timerqueue_head->rb_leftmost`) so "what expires next" is O(1).
- Insert/remove are O(log n).
- **Not tied to the tick.** The next expiry is programmed directly into the clockevent device,
  so resolution is limited only by hardware and interrupt latency (~1 µs, not 1 ms).
- Per-CPU, with several **bases**: `MONOTONIC`, `REALTIME`, `BOOTTIME`, `TAI`, plus `_SOFT`
  variants that run in softirq rather than hardirq context.

The `_SOFT` distinction matters: hardirq-context hrtimer callbacks add directly to system
interrupt latency (Ch. 17, T.2). Since 4.16, hrtimers default to the *soft* base unless
`HRTIMER_MODE_..._HARD` is specified, and on PREEMPT_RT almost everything is soft. If you set
`HRTIMER_MODE_ABS_HARD`, you are asserting that your callback is bounded and trivial — and a
reviewer will ask you to prove it.

**Choosing between them — the decision rule:**

```
Is this a timeout you expect to cancel?          → timer_list
Do you need sub-millisecond accuracy?            → hrtimer
Is it periodic at high frequency?                → hrtimer (with forwarding)
Must it wake the system from suspend?            → alarmtimer (CLOCK_*_ALARM)
Is it per-task CPU time?                         → posix-cpu-timers
Does it just need to run "soon, roughly"?        → timer_list with a slack-friendly deadline
```

### T.6 Timer slack and coalescing: trading precision for power

Every timer expiry wakes a CPU. A CPU that wakes 1000 times/sec never reaches a deep C-state,
and deep C-states are where nearly all idle power savings live (the exit latency is
microseconds but the residency requirement is milliseconds).

So the kernel deliberately **batches** timers:

- **Deferrable timers** (`TIMER_DEFERRABLE`) do not wake an idle CPU at all; they wait until
  the CPU wakes for another reason. Correct for statistics, cache cleanup, garbage collection.
- **Timer slack** — each task has `current->timer_slack_ns` (default 50 µs, settable via
  `prctl(PR_SET_TIMERSLACK)`), and `poll`/`select`/`nanosleep` round the deadline up by it,
  so nearby timers coalesce into one wakeup.
- **Range timers** — `hrtimer_start_range_ns(timer, expiry, delta, mode)` gives the kernel an
  explicit `[expiry, expiry+delta]` window to fire in.

This is a first-class design principle for any timer you add: **state the precision you
actually need, and no more.** An API that demands exact timing when approximate would do is
imposing a power cost on every user forever. Reviewers ask about this.

### T.7 The tick, and the theory of tickless operation

Historically Linux had a **periodic tick** at `CONFIG_HZ` (100/250/300/1000 Hz) that drove
timekeeping, timer expiry, scheduler accounting, and RCU quiescent states. Simple, but it
means the CPU wakes `HZ` times per second even when completely idle.

Three modes exist today:

| Mode | Config | Behaviour |
|---|---|---|
| Periodic | `HZ_PERIODIC` | always tick |
| **Idle-tickless** | `NO_HZ_IDLE` (default) | stop the tick when the CPU is idle |
| **Full-tickless** | `NO_HZ_FULL` | stop the tick even when **one** task is running |

**NO_HZ_IDLE** is straightforward: when idle, compute the next timer expiry and program the
clockevent for exactly then, possibly seconds away. Power win, no downside. Note this
interacts with RCU (Ch. 15, T.2.2): an idle CPU must still be counted as quiescent without
being woken — that's the `dynticks` counter.

**NO_HZ_FULL** is much harder and is where the interesting theory is. To stop the tick while a
task is running, you must eliminate *everything* the tick was doing:

1. **Timekeeping** — one CPU must remain the "timekeeper" and keep ticking
   (`tick_do_timer_cpu`). NO_HZ_FULL cannot cover all CPUs.
2. **Scheduler accounting** — with no tick, how do you measure CPU time? Answer:
   `CONFIG_VIRT_CPU_ACCOUNTING_GEN`, accounting at every kernel/user transition instead.
   More accurate, slightly more expensive per syscall.
3. **Preemption / time-slice enforcement** — if only one runnable task exists there is
   nothing to preempt *to*, hence the "≤1 runnable task" condition for going tickless.
4. **RCU** — no tick means no quiescent-state reports. Solved by `rcu_nocbs` (offload
   callbacks) plus context-tracking-based QS reporting on kernel/user transitions.
5. **Everything else that assumed a tick** — vmstat updates, workqueue timers, watchdogs —
   all had to be made explicitly schedulable or offloaded.

The result is a CPU that can run a userspace thread for **seconds with zero kernel
interruptions**, which is what HPC, DPDK-style packet processing, and hard real-time need.
Achieving it required auditing *every* periodic activity in the kernel — a multi-year effort
and a great case study in how a seemingly small change (stop the tick) cascades through an
entire architecture.

```bash
# NO_HZ_FULL setup for CPUs 2-7:
# boot: isolcpus=2-7 nohz_full=2-7 rcu_nocbs=2-7 irqaffinity=0-1
cat /sys/devices/system/cpu/nohz_full
grep -E 'LOC|TLB|RES' /proc/interrupts        # local timer interrupts per CPU
sudo perf stat -e irq_vectors:local_timer_entry -C 4 -- sleep 10   # should be ~0
```

### T.8 jiffies: wraparound and the comparison macros

`jiffies` is an `unsigned long` incremented `HZ` times per second. At `HZ=1000` on 32-bit it
wraps in **49.7 days**. To catch wraparound bugs early, the kernel deliberately initializes it
to `-5 minutes` worth of ticks (`INITIAL_JIFFIES`) so any latent bug fires in the first five
minutes of uptime rather than in month two. That is a wonderful piece of defensive
engineering — *make latent bugs manifest immediately*.

**Never compare jiffies with `<` or `>`.** Use the macros, which are wraparound-safe because
they compare *signed differences*:

```c
time_after(a, b)          /* a > b, wrap-safe */
time_before(a, b)
time_after_eq(a, b)
time_before_eq(a, b)
time_in_range(a, b, c)
/* 64-bit variants for jiffies_64: time_after64(), etc. */

/* Correct timeout loop: */
unsigned long deadline = jiffies + msecs_to_jiffies(500);
while (!device_ready()) {
	if (time_after(jiffies, deadline))
		return -ETIMEDOUT;
	cpu_relax();
}
```
Why it works: `time_after(a,b)` is `((long)((b) - (a)) < 0)`. Modular subtraction is correct
across a wrap as long as the true interval is less than half the range.

Conversions: `msecs_to_jiffies()`, `usecs_to_jiffies()`, `jiffies_to_msecs()`,
`nsecs_to_jiffies()`. Never hand-compute with `HZ` — `HZ` is a config option and your
arithmetic will be wrong on someone's kernel.

### T.9 The 2038 problem

`time_t` as a signed 32-bit count of seconds since 1970 overflows on
**2038-01-19 03:14:07 UTC**. The kernel's response was a multi-year, tree-wide conversion:

- All internal time is `ktime_t` (signed 64-bit nanoseconds) or `struct timespec64`.
- `struct timespec` and `struct timeval` are **banned in new kernel code**.
- New 64-bit-time syscalls were added for 32-bit architectures
  (`clock_gettime64`, `futex_time64`, …), and 32-bit `y2038`-safe userspace requires
  `-D_TIME_BITS=64` with glibc 2.34+ / musl.
- Filesystem on-disk formats had to be extended (ext4 uses extra bits in the inode for
  nanoseconds *and* an epoch extension; XFS added bigtime in v5).

For embedded work (Part 7) this is not academic: a 32-bit ARM device shipping today with a
pre-2020 toolchain will fail in 2038. Check `_TIME_BITS`, check your filesystem, check your
RTC driver's range.

---

## 1. Concept — the API

### 1.1 Reading time

```c
#include <linux/ktime.h>
#include <linux/timekeeping.h>

ktime_t   t   = ktime_get();              /* CLOCK_MONOTONIC as ktime_t (s64 ns) */
u64       ns  = ktime_get_ns();
u64       bns = ktime_get_boottime_ns();  /* includes suspend */
u64       rns = ktime_get_real_ns();      /* wall clock — NOT for intervals */
u64       tai = ktime_get_clocktai_ns();

/* Coarse variants: no hardware read, just the last tick's value. ~1 ns, ms accuracy. */
ktime_t   c   = ktime_get_coarse();
u64       cns = ktime_get_coarse_ns();

/* struct forms */
struct timespec64 ts;
ktime_get_ts64(&ts);
ktime_get_real_ts64(&ts);

/* Raw cycle counters for micro-benchmarks */
u64 cycles = get_cycles();                /* rdtsc / CNTVCT — no ordering! */
```

> **Use the `coarse` variants for anything not needing sub-tick accuracy.** They read a
> cached value instead of the hardware counter — an order of magnitude cheaper. Filesystem
> timestamps use them (which is why `stat` mtimes have tick granularity).

### 1.2 `timer_list` (timeouts)

```c
#include <linux/timer.h>

struct my_dev {
	struct timer_list timeout;
	...
};

static void my_timeout(struct timer_list *t)
{
	struct my_dev *d = timer_container_of(d, t, timeout);   /* 6.x name; older: from_timer() */

	/* ★ SOFTIRQ CONTEXT: cannot sleep. */
	dev_warn(d->dev, "device timed out\n");
	schedule_work(&d->reset_work);
}

timer_setup(&d->timeout, my_timeout, 0);
mod_timer(&d->timeout, jiffies + msecs_to_jiffies(5000));   /* arm or re-arm */
timer_delete(&d->timeout);                                  /* async */
timer_delete_sync(&d->timeout);     /* ★ waits for a running callback — MUST use before free */
timer_shutdown_sync(&d->timeout);   /* delete + prevent re-arming (6.2+) */
```

> Naming note: `del_timer()`/`del_timer_sync()` were renamed to
> `timer_delete()`/`timer_delete_sync()` in 6.2+; both exist during the transition.
> `timer_shutdown_sync()` is the correct call in a teardown path for a **self-rearming**
> timer — it makes subsequent `mod_timer()` calls no-ops, closing the same race we saw with
> self-rearming delayed work in Ch. 18.

Flags: `TIMER_DEFERRABLE` (don't wake an idle CPU), `TIMER_PINNED` (must run on the arming
CPU), `TIMER_IRQSAFE`.

### 1.3 `hrtimer` (precision)

```c
#include <linux/hrtimer.h>

static enum hrtimer_restart my_hr_fn(struct hrtimer *t)
{
	struct my_dev *d = container_of(t, struct my_dev, hrt);

	do_precise_thing(d);
	hrtimer_forward_now(t, ms_to_ktime(1));   /* next period, absorbing overruns */
	return HRTIMER_RESTART;                   /* or HRTIMER_NORESTART */
}

hrtimer_setup(&d->hrt, my_hr_fn, CLOCK_MONOTONIC, HRTIMER_MODE_REL_SOFT);  /* 6.15+ API */
hrtimer_start(&d->hrt, ms_to_ktime(1), HRTIMER_MODE_REL_SOFT);
hrtimer_start_range_ns(&d->hrt, expiry, 50 * NSEC_PER_USEC, HRTIMER_MODE_ABS);
hrtimer_cancel(&d->hrt);            /* waits for the callback */
hrtimer_try_to_cancel(&d->hrt);     /* non-blocking */
```
`hrtimer_forward_now()` is how you write a jitter-free periodic timer: it advances the
expiry by whole periods from *the previous expiry*, not from *now*, so errors do not
accumulate. Computing `now + period` yourself causes drift — a classic mistake.

### 1.4 Sleeping and delaying

```c
/* Busy-wait — BLOCKS THE CPU. Use only in atomic context and only for short waits. */
ndelay(100); udelay(10); mdelay(1);    /* mdelay > ~10ms is essentially always a bug */

/* Sleeping — task context only */
msleep(100);                  /* uses timer_list; may over-sleep considerably */
msleep_interruptible(100);
usleep_range(100, 200);       /* ★ preferred for 10 µs – 20 ms: gives the kernel a window */
fsleep(usecs);                /* picks udelay/usleep_range/msleep by magnitude */
schedule_timeout_uninterruptible(HZ);
wait_event_timeout(wq, cond, msecs_to_jiffies(500));

/* Polling a register with a timeout — use the helpers, don't hand-roll */
#include <linux/iopoll.h>
ret = readl_poll_timeout(base + STATUS, val, val & READY, 10 /*us sleep*/, 1000 /*us total*/);
ret = readl_poll_timeout_atomic(base + STATUS, val, val & READY, 1, 100);
ret = regmap_read_poll_timeout(regmap, REG, val, val & BIT(0), 100, 10000);
```

> **`msleep(1)` typically sleeps 1–20 ms**, because `timer_list` granularity plus rounding.
> For 1 ms – 20 ms use `usleep_range(min, max)`, which uses an hrtimer with a coalescing
> window. `Documentation/timers/timers-howto.rst` states this explicitly and it is one of
> the most frequently violated rules in driver code.

---

## 2. Internals

### 2.1 Source map

```
kernel/time/timekeeping.c     ★ the timekeeper, mult/shift, ktime_get*
kernel/time/clocksource.c     registration, rating, the watchdog
kernel/time/clockevents.c     programmable event devices
kernel/time/tick-common.c     periodic tick
kernel/time/tick-sched.c      ★ NO_HZ: tick_nohz_idle_enter/exit, nohz_full
kernel/time/timer.c           ★ the timer wheel (read the header comment!)
kernel/time/hrtimer.c         ★ hrtimers, timerqueue bases
kernel/time/ntp.c             NTP/PLL discipline, adjtimex
kernel/time/alarmtimer.c      RTC-backed timers that wake from suspend
kernel/time/posix-cpu-timers.c
lib/timerqueue.c              rbtree + cached leftmost
drivers/clocksource/          every SoC timer driver
Documentation/timers/         ★ timers-howto.rst, hrtimers.rst, highres.rst, no_hz.rst
```

### 2.2 The wheel, in code

```c
/* kernel/time/timer.c */
#define LVL_CLK_SHIFT   3
#define LVL_BITS        6
#define LVL_SIZE        (1UL << LVL_BITS)          /* 64 buckets per level */
#define LVL_DEPTH       8                          /* 9 for HZ=1000 */
#define WHEEL_SIZE      (LVL_SIZE * LVL_DEPTH)

struct timer_base {
	raw_spinlock_t   lock;
	struct timer_list *running_timer;   /* for timer_delete_sync() */
	unsigned long    clk;               /* this base's notion of "now" */
	unsigned long    next_expiry;
	unsigned int     cpu;
	bool             is_idle;
	DECLARE_BITMAP(pending_map, WHEEL_SIZE);   /* ★ which buckets are non-empty */
	struct hlist_head vectors[WHEEL_SIZE];
};
```

The `pending_map` bitmap is the trick that makes "find the next expiry" fast without
scanning 512 buckets — essential for NO_HZ, which must compute the next wakeup on every idle
entry. Each CPU has two bases: `BASE_LOCAL`/`BASE_GLOBAL` (and a deferrable one), separating
pinned from migratable timers.

### 2.3 Where timers actually execute

```
tick interrupt (clockevent)
  → tick_handle_periodic / hrtimer_interrupt
      → update_process_times()
          → run_local_timers()
              → raise_softirq(TIMER_SOFTIRQ)
  ...
  → irq_exit() → __do_softirq()
       → run_timer_softirq()
            → __run_timers(base)        ← your timer_list callback runs HERE (softirq)
       → hrtimer_run_softirq()          ← _SOFT hrtimer callbacks

hrtimer_interrupt()                      ← hard hrtimers run directly in HARDIRQ context
```

So: `timer_list` callbacks are **softirq context**; `hrtimer` callbacks are hardirq *or*
softirq depending on the mode. Neither may sleep. Both may be delayed by softirq backlog
(Ch. 18, T.2) — which is why an overloaded network CPU also has bad timer accuracy.

### 2.4 Observability

```bash
cat /proc/timer_list | head -80            # ★ every pending timer, per CPU, per base
cat /proc/timer_list | grep -A5 'cpu: 0'
cat /proc/timer_stats                      # (older kernels)

sudo perf stat -e timer:* -a sleep 5
sudo bpftrace -e 'tracepoint:timer:timer_start { @[ksym(args->function)] = count(); }'
sudo bpftrace -e 'tracepoint:timer:hrtimer_start { @[ksym(args->function)] = count(); }'

sudo trace-cmd record -e timer -e irq_vectors sleep 2 && trace-cmd report | head

# Who is waking up an idle CPU?
sudo powertop --html=out.html
sudo turbostat --interval 1                # C-state residency: proves timers are batching
```

`/proc/timer_list` is underused and extremely informative — it shows every armed timer with
its function, expiry, and slack. When chasing a wakeup problem, start here.

---

## 3. Practice

### 3.1 Timeout with a `timer_list` (correct teardown)

```c
struct mydev {
	struct timer_list  watchdog;
	struct work_struct reset;
	struct device     *dev;
	atomic_t           armed;
};

static void mydev_watchdog(struct timer_list *t)
{
	struct mydev *d = timer_container_of(d, t, watchdog);

	/* softirq context: defer anything that sleeps */
	dev_err(d->dev, "hardware watchdog expired\n");
	schedule_work(&d->reset);
}

static void mydev_start_op(struct mydev *d)
{
	mod_timer(&d->watchdog, jiffies + msecs_to_jiffies(2000));
	mydev_kick_hw(d);
}

static void mydev_op_complete(struct mydev *d)
{
	timer_delete(&d->watchdog);        /* the common case: cancelled, never fired */
	mydev_finish(d);
}

static void mydev_remove(struct platform_device *pdev)
{
	struct mydev *d = platform_get_drvdata(pdev);

	timer_shutdown_sync(&d->watchdog); /* ★ delete, wait, and prevent re-arm */
	cancel_work_sync(&d->reset);
	/* only now is it safe to free anything the callbacks touch */
}
```

### 3.2 Jitter-free periodic hrtimer

```c
#define PERIOD_NS (250 * NSEC_PER_USEC)   /* 4 kHz */

static enum hrtimer_restart sample_fn(struct hrtimer *t)
{
	struct mydev *d = container_of(t, struct mydev, hrt);
	u64 overruns;

	mydev_take_sample(d);

	/* Advance from the PREVIOUS expiry, not from now: no drift accumulation.
	 * Returns how many periods were missed, which is a useful health metric. */
	overruns = hrtimer_forward_now(t, ns_to_ktime(PERIOD_NS));
	if (overruns > 1)
		d->missed += overruns - 1;

	return HRTIMER_RESTART;
}

hrtimer_setup(&d->hrt, sample_fn, CLOCK_MONOTONIC, HRTIMER_MODE_REL_SOFT);
hrtimer_start(&d->hrt, ns_to_ktime(PERIOD_NS), HRTIMER_MODE_REL_SOFT);
...
hrtimer_cancel(&d->hrt);
```

### 3.3 Measure real timer accuracy

```c
static int __init jitter_init(void)
{
	ktime_t t0 = ktime_get();
	int i;

	for (i = 0; i < 100; i++)
		msleep(1);
	pr_info("100x msleep(1)      = %lld us\n", ktime_us_delta(ktime_get(), t0));

	t0 = ktime_get();
	for (i = 0; i < 100; i++)
		usleep_range(1000, 1100);
	pr_info("100x usleep_range   = %lld us\n", ktime_us_delta(ktime_get(), t0));
	return 0;
}
```
Expected: `msleep(1)` ×100 takes far more than 100 ms (often 200–400 ms at HZ=250);
`usleep_range` lands close to 100–110 ms. **Run this. It is the most persuasive possible
argument for `usleep_range`.**

### 3.4 NO_HZ_FULL experiment

```bash
# Boot with: isolcpus=nohz,domain,managed_irq,3-7 nohz_full=3-7 rcu_nocbs=3-7
# Then, on CPU 4, run a spin loop and count local timer interrupts:
grep LOC /proc/interrupts
taskset -c 4 ./spinner &
sleep 10
grep LOC /proc/interrupts        # CPU4's count should barely move
sudo perf stat -e irq_vectors:local_timer_entry -C 4 -- sleep 10
```

---

## 3.5 Extended practice

### Lab 19.A — The sleep-accuracy experiment (§1.4)

The single most behaviour-changing measurement in this chapter.

```c
// SPDX-License-Identifier: GPL-2.0
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/delay.h>
#include <linux/ktime.h>

#define N 100
#define REPORT(label, stmt) do {                                        \
	u64 t0 = ktime_get_ns(), worst = 0; int i;                      \
	for (i = 0; i < N; i++) {                                       \
		u64 a = ktime_get_ns();                                 \
		stmt;                                                   \
		worst = max(worst, ktime_get_ns() - a);                 \
	}                                                               \
	pr_info("%-28s avg %6llu us  worst %6llu us\n", label,          \
		(ktime_get_ns() - t0) / N / 1000, worst / 1000);        \
} while (0)

static int __init sleepbench_init(void)
{
	pr_info("HZ=%d  jiffies granularity=%u us\n", HZ, jiffies_to_usecs(1));

	REPORT("msleep(1)",                 msleep(1));
	REPORT("msleep(10)",                msleep(10));
	REPORT("usleep_range(1000,1100)",   usleep_range(1000, 1100));
	REPORT("usleep_range(1000,1000)",   usleep_range(1000, 1000));
	REPORT("usleep_range(100,200)",     usleep_range(100, 200));
	REPORT("fsleep(1000)",              fsleep(1000));
	REPORT("schedule_timeout(1 jiffy)", { set_current_state(TASK_UNINTERRUPTIBLE);
					      schedule_timeout(1); });
	REPORT("udelay(1000) [BUSY WAIT]",  udelay(1000));
	return 0;
}
static void __exit sleepbench_exit(void) { }
module_init(sleepbench_init); module_exit(sleepbench_exit);
MODULE_LICENSE("GPL");
```
```bash
sudo insmod sleepbench.ko && dmesg | tail -12 && sudo rmmod sleepbench
# Repeat under load:
stress-ng --cpu $(nproc) --timeout 30s &
sudo insmod sleepbench.ko && dmesg | tail -12 && sudo rmmod sleepbench
```
**Expected:** `msleep(1)` averages 2–20 ms depending on `HZ`; `usleep_range(1000,1100)`
lands at ~1.0–1.1 ms. That is a **10–20× error**, and it is why
`Documentation/timers/timers-howto.rst` forbids `msleep()` under 20 ms. Most drivers in the
wild get this wrong.

### Lab 19.B — Timer wheel accuracy vs hrtimer accuracy (T.4/T.5)

```c
struct probe { struct timer_list t; ktime_t armed, requested; };
struct hprobe { struct hrtimer h; ktime_t armed, requested; };
static struct probe  *tl;
static struct hprobe *hr;
static int n = 2000;

static void tl_fn(struct timer_list *t)
{
	struct probe *p = timer_container_of(p, t, t);
	s64 err_us = ktime_us_delta(ktime_get(), p->requested);

	atomic64_add(err_us, &tl_err_total);
	if (err_us > atomic64_read(&tl_err_max))
		atomic64_set(&tl_err_max, err_us);
}

static enum hrtimer_restart hr_fn(struct hrtimer *h)
{
	struct hprobe *p = container_of(h, struct hprobe, h);
	s64 err_us = ktime_us_delta(ktime_get(), p->requested);

	atomic64_add(err_us, &hr_err_total);
	return HRTIMER_NORESTART;
}

/* Arm n timers of each kind with random delays spread over [1ms, 10s]. */
```
```bash
for load in idle busy; do
  [ $load = busy ] && stress-ng --cpu $(nproc) --timeout 60s &
  sudo insmod timeracc.ko n=2000; sleep 15; dmesg | tail -4; sudo rmmod timeracc
done
```
**Expected:** `timer_list` error grows with the requested delay (coarser wheel level =
larger granularity — exactly T.4's no-cascade trade), while hrtimer error stays ~tens of µs
regardless. Plot error vs delay for both; the *staircase* in the `timer_list` curve is the
wheel's level boundaries made visible. **That staircase is the single best artifact in this
chapter.**

### Lab 19.C — Inspect every armed timer in the system

```bash
# The underused goldmine:
sudo cat /proc/timer_list | head -100
sudo cat /proc/timer_list | grep -c '^ #'              # total armed timers
sudo awk '/^ #/{print $NF}' /proc/timer_list | sort | uniq -c | sort -rn | head -20

# Per-CPU clockevent programming and NO_HZ state:
sudo grep -A12 'cpu: 0' /proc/timer_list

# Who is arming timers right now?
sudo bpftrace -e 'tracepoint:timer:timer_start   { @[ksym(args->function)] = count(); }'
sudo bpftrace -e 'tracepoint:timer:hrtimer_start { @[ksym(args->function)] = count(); }'

# Who is EXPIRING, and how late?
sudo bpftrace -e '
tracepoint:timer:hrtimer_start  { @exp[args->hrtimer] = args->expires; }
tracepoint:timer:hrtimer_expire_entry /@exp[args->hrtimer]/ {
	@late_us[ksym(args->function)] = hist((args->now - @exp[args->hrtimer])/1000);
	delete(@exp[args->hrtimer]); }'
```

### Lab 19.D — Clocksource: measure the TSC cliff (T.3)

```bash
cat /sys/devices/system/clocksource/clocksource0/available_clocksource
CUR=/sys/devices/system/clocksource/clocksource0/current_clocksource
cat $CUR

cat > /tmp/cg.c <<'EOF'
#define _GNU_SOURCE
#include <stdio.h>
#include <time.h>
#define N 2000000
int main(void) {
	struct timespec ts, a, b;
	long i;
	clock_gettime(CLOCK_MONOTONIC, &a);
	for (i = 0; i < N; i++) clock_gettime(CLOCK_MONOTONIC, &ts);
	clock_gettime(CLOCK_MONOTONIC, &b);
	printf("%.1f ns/call\n",
	   ((b.tv_sec-a.tv_sec)*1e9 + (b.tv_nsec-a.tv_nsec)) / N);
	return 0;
}
EOF
gcc -O2 -o /tmp/cg /tmp/cg.c

for cs in $(cat /sys/devices/system/clocksource/clocksource0/available_clocksource); do
  echo $cs | sudo tee $CUR >/dev/null
  echo -n "$cs: "; /tmp/cg
done
echo tsc | sudo tee $CUR >/dev/null

# Prove the vDSO is doing the work:
strace -c /tmp/cg 2>&1 | tail -5     # almost NO clock_gettime syscalls with tsc
strace -c -e trace=clock_gettime /tmp/cg 2>&1 | tail -3

# Watchdog behaviour:
dmesg | grep -i 'clocksource\|tsc'
grep -E 'constant_tsc|nonstop_tsc|tsc_reliable' /proc/cpuinfo | head -1
```
**Expect ~20 ns (tsc, vDSO) vs ~300–600 ns (hpet/acpi_pm, real syscall).** A production box
that demotes to HPET gets a system-wide slowdown for every `clock_gettime`, every log
timestamp, and every `perf` sample — a classic mystery performance cliff.

### Lab 19.E — Deferrable timers, slack, and power (T.6)

```c
static struct timer_list normal, deferrable;

timer_setup(&normal, fn, 0);
timer_setup(&deferrable, fn, TIMER_DEFERRABLE);
/* arm both for +1s, repeatedly, on an otherwise idle system */
```
```bash
# Measure wakeups and C-state residency:
sudo turbostat --quiet --show Core,CPU,Busy%,C1%,C3%,C6%,C7s%,PkgWatt --interval 5
sudo powertop --time=20 --html=/tmp/pt.html
cat /sys/devices/system/cpu/cpu0/cpuidle/state*/usage
grep LOC /proc/interrupts                # local timer interrupt count

# Userspace timer slack:
chrt -o 0 ./myapp &
cat /proc/$!/timerslack_ns
echo 50000000 | sudo tee /proc/$!/timerslack_ns    # 50 ms slack → heavy coalescing
```
Run your workload with the normal timer, then the deferrable one, then with range timers.
Report package watts and C6 residency for each. **This is the power argument, quantified.**

### Lab 19.F — NO_HZ_FULL: achieve a truly quiet CPU (T.7)

```bash
# Boot parameters:
#   isolcpus=nohz,domain,managed_irq,4-7 nohz_full=4-7 rcu_nocbs=4-7
#   irqaffinity=0-3 rcu_nocb_poll nosoftlockup tsc=reliable
cat /sys/devices/system/cpu/nohz_full
cat /sys/devices/system/cpu/isolated
ps -eo pid,comm,psr | grep rcuo | head

# Baseline: count local timer interrupts on an isolated CPU
B=$(grep LOC /proc/interrupts | awk '{print $6}')       # column for CPU4
taskset -c 4 ./spinner & SPIN=$!
sleep 10
A=$(grep LOC /proc/interrupts | awk '{print $6}')
echo "local timer interrupts on cpu4 in 10s: $((A-B))"   # target: single digits
kill $SPIN

# Find ALL remaining interference on that CPU:
sudo perf stat -e irq_vectors:local_timer_entry,sched:sched_switch,irq:softirq_entry \
     -C 4 -- sleep 10
sudo trace-cmd record -C 4 -e sched -e timer -e irq_vectors -e workqueue -- sleep 5
trace-cmd report | awk '{print $4}' | sort | uniq -c | sort -rn | head -20

# The definitive checklist:
$EDITOR Documentation/admin-guide/kernel-per-CPU-kthreads.rst
```
Eliminate every remaining source one at a time (vmstat updates: `sysctl vm.stat_interval`;
workqueues: `/sys/devices/virtual/workqueue/*/cpumask`; watchdog:
`sysctl kernel.watchdog_cpumask`). **Getting to zero interrupts per second is a rite of
passage for HPC/RT/DPDK work.**

### Lab 19.G — Jiffies wraparound, proven (T.8)

```c
static int __init wrap_init(void)
{
	unsigned long a = 0xFFFFFFF0UL;   /* near wrap on 32-bit */
	unsigned long b = 0x00000010UL;   /* just after wrap */

	pr_info("naive  a > b        : %d  (WRONG: b is LATER)\n", a > b);
	pr_info("safe   time_after(b,a): %d  (correct)\n", time_after(b, a));
	pr_info("diff   (long)(b - a) : %ld\n", (long)(b - a));
	pr_info("INITIAL_JIFFIES = %lu, jiffies now = %lu, uptime ~%lu s\n",
		(unsigned long)INITIAL_JIFFIES, jiffies,
		(jiffies - INITIAL_JIFFIES) / HZ);
	return 0;
}
```
Then find the real thing:
```bash
grep -n 'INITIAL_JIFFIES' include/linux/jiffies.h
git grep -n 'jiffies >\|jiffies <' -- drivers/ | head -20   # candidate bugs
git log --oneline --grep='time_after' --grep='wrap' --all-match | head -20
```
Every hit in that `git grep` is a potential patch. Evaluate five.

### Lab 19.H — Register polling: use the helpers, measure the difference

```c
#include <linux/iopoll.h>

/* BAD: busy-waits, blocks the CPU, no timeout */
while (!(readl(base + STATUS) & READY))
	;

/* BAD: msleep(1) in a loop → 10–20x over-sleep (Lab 19.A) */
while (!(readl(base + STATUS) & READY))
	msleep(1);

/* GOOD: sleeping poll with a proper timeout */
ret = readl_poll_timeout(base + STATUS, val, val & READY,
			 10 /* us between polls */, 100000 /* us total */);

/* GOOD: atomic context (udelay-based) */
ret = readl_poll_timeout_atomic(base + STATUS, val, val & READY, 1, 1000);

/* GOOD: over a sleeping bus */
ret = regmap_read_poll_timeout(rm, REG_STATUS, val, val & READY, 100, 10000);
```
Benchmark all four against a simulated device that becomes ready after 3 ms. Report actual
wait time and CPU consumed. Then audit a real driver:
```bash
git grep -n 'while.*readl\|while.*ioread' drivers/ | head -20
git log --oneline --grep='readl_poll_timeout' | head -20   # the conversion campaign
```

---

## 4. Mastery drills

1. **Read the Varghese & Lauck paper** (SOSP 1987). Then read the header comment of
   `kernel/time/timer.c` and map the paper's "hierarchical wheel" onto `LVL_DEPTH`,
   `LVL_CLK_SHIFT`, and `calc_index()`. Explain precisely why the current implementation
   does not cascade and what accuracy it gives up.

2. **Clock choice audit.** Find three places in `drivers/` using `ktime_get_real*()`.
   For each, decide whether it should be monotonic. (Real bugs exist here;
   `git log --grep="use ktime_get instead"` shows the fixes.)

3. **Wraparound.** Write code that compares jiffies with `>` and demonstrate the bug by
   setting `INITIAL_JIFFIES` or simulating with `unsigned long` arithmetic. Then prove
   `time_after()` is correct across the wrap.

4. **TSC demotion.** Find the clocksource watchdog in `kernel/time/clocksource.c`. Explain
   the algorithm. Then benchmark `clock_gettime(CLOCK_MONOTONIC)` with
   `current_clocksource=tsc` vs `hpet` (you can switch at runtime via sysfs). Report the
   ns/call difference and explain the vDSO's role.

5. **hrtimer vs timer_list accuracy.** Arm 10,000 of each with random expiries in
   [1 ms, 10 s] and record actual vs requested expiry. Plot the error distributions. Explain
   the shape from T.4/T.5.

6. **Power cost of timers.** Use `powertop` to find the top wakeup sources on an idle laptop
   or VM. Pick one, find its timer in `/proc/timer_list`, and determine whether it could be
   `TIMER_DEFERRABLE` or use a range. (Several such patches have been merged.)

7. **Read `tick-sched.c`.** Trace `tick_nohz_idle_enter()` → `tick_nohz_next_event()` →
   `tick_nohz_stop_tick()`. Explain how the next timer expiry is computed and how the
   `pending_map` bitmap makes it fast.

8. **The 2038 audit.** Pick an embedded target from Part 7. Verify `_TIME_BITS=64`, verify
   the filesystem supports post-2038 timestamps, and verify the RTC driver's range. Write
   down what would break and when.

9. **Explain leap seconds** and the 2012 outage to a colleague, including why `CLOCK_TAI`
   exists and what leap smearing does.

---

## 5. Further reading

**Kernel docs:**
- `Documentation/timers/timers-howto.rst` — **which delay/sleep function to use; mandatory**
- `Documentation/timers/hrtimers.rst`, `highres.rst`
- `Documentation/timers/no_hz.rst` — **the definitive NO_HZ_FULL guide**
- `Documentation/core-api/timekeeping.rst` — which `ktime_get*` variant to use
- `Documentation/admin-guide/kernel-per-CPU-kthreads.rst` — removing every source of
  interference from a CPU (the practical NO_HZ_FULL companion)

**Papers:**
- Varghese & Lauck, "Hashed and Hierarchical Timing Wheels: Data Structures for the
  Efficient Implementation of a Timer Facility" (SOSP 1987) — **the algorithm**
- Mills, "Internet Time Synchronization: The Network Time Protocol" / RFC 5905 — the PLL
- Gleixner & Niehaus, "Hrtimers and Beyond: Transforming the Linux Time Subsystems"
  (OLS 2006) — the hrtimer design paper

**LWN:**
- "Reinventing the timer wheel" (2015) — the no-cascade redesign
- "The return of the timer wheel" / "Timers and the tickless kernel"
- "(Nearly) full tickless operation in 3.10"
- "The 2038 problem and the kernel" series
- "Clocksource watchdog and the TSC"

→ Next: [20-processes.md](20-processes.md)

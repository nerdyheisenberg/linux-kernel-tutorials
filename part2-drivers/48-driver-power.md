# Chapter 48 — Runtime PM, system suspend, and driver power states

> **Goal:** Understand power management as a *distributed consensus problem over a dependency graph*, why runtime PM and system suspend are two different problems that share one callback set, why the reference count is the entire design, why `->suspend` may not sleep waiting for userspace, how wakeup sources make "suspend" safe against races, and what the `_noirq` phases exist to solve. By the end you can add correct runtime PM to any driver, debug a machine that will not suspend or will not stay asleep, and reason precisely about the ordering guarantees the PM core does and does not give you.

---

## Theory & First Principles

### T.0 — Start here: two problems that look like one, and are not

"Power management" sounds like one topic. It is two, with different mechanisms, different
failure modes, and different code paths — and conflating them is the most common conceptual
error in this area.

```
  RUNTIME PM                          SYSTEM SUSPEND
  +----------------------------+      +--------------------------------+
  | The MACHINE is running.    |      | The WHOLE MACHINE goes down.   |
  | ONE idle device powers     |      | Every device, in dependency    |
  | down, right now, while     |      | order, then the CPUs.          |
  | everything else runs.      |      |                                |
  |                            |      | Triggered by: user, lid, idle  |
  | Triggered by: a refcount   |      | policy                         |
  | hitting zero.              |      |                                |
  | Latency budget: microsec   |      | Latency budget: ~1 second      |
  | Userspace: unaware         |      | Userspace: frozen              |
  +----------------------------+      +--------------------------------+
```

**Runtime PM is a refcount**, and once you see that it becomes simple — it is Ch. 12 again:

```c
pm_runtime_get_sync(dev);     /* 0 -> 1: resume the device. 1 -> 2: just count. */
do_the_work(dev);
pm_runtime_put_autosuspend(dev);   /* 1 -> 0: schedule a suspend, after a delay */
```

The `autosuspend` delay exists because suspending and resuming costs time and energy; if the
device will be used again in 10 ms, powering it down was a net loss. **That is a ski-rental
problem** (Ch. 14 §T.2) with a tunable rather than a computed answer, exposed as
`/sys/.../power/autosuspend_delay_ms`.

**System suspend is an ordered graph traversal**, and the order is what the device model
exists for (Ch. 26 §T.0):

```
 suspend:  children first, then parents    (a USB mouse, before its controller,
                                             before the PCI bridge)
 resume:   parents first, then children
```

**Now the part that makes system suspend genuinely hard**, and it is worth knowing before
you debug one: there are *four* callback phases, not one, and they exist because of a race.

```
  ->prepare()        userspace frozen; may still sleep and allocate
  ->suspend()        normal context; may sleep. Most work happens here.
  ->suspend_late()
  ->suspend_noirq()  INTERRUPTS DISABLED. No sleeping. No IRQs.
  ------------------ the machine is now genuinely off ------------------
```

The `_noirq` phase exists because a device that has already been told to suspend may still
receive an interrupt, and its handler would touch hardware that is now powered off.
Disabling interrupts and doing final state saves in a separate phase closes that window.
**Nearly every suspend bug is a wrong phase or a wrong order**, and `initcall_debug` plus
`/sys/power/pm_test` is how you find it.

**And the thing that makes this different from every other chapter:** you cannot test power
management by making it work once. A device that resumes correctly 999 times out of 1000 is
a device that hangs a laptop weekly. **The only meaningful test is thousands of cycles under
varying load**, which is why §T.8's automated suspend-resume loop matters more than any
amount of code review.

```bash
cat /sys/devices/.../power/runtime_status      # active / suspended / suspending
cat /sys/devices/.../power/runtime_active_time
cat /sys/kernel/debug/pm_genpd/pm_genpd_summary
echo devices > /sys/power/pm_test && echo mem > /sys/power/state   # test WITHOUT
                                                                   # really suspending
dmesg | grep -i 'PM:.*fail\|did not resume'
```

---

### T.1 Two problems that look like one

"Power management" names two problems with different inputs, different timescales, and different correctness criteria. Conflating them is the source of most PM bugs.

| | **Runtime PM** | **System suspend** |
|---|---|---|
| Trigger | a device became idle | the user/policy decided to sleep the machine |
| Scope | one device (and its ancestors) | **everything, simultaneously** |
| Frequency | thousands of times per second | a few times per day |
| Userspace | **still running** | frozen |
| Latency budget | microseconds to milliseconds | hundreds of milliseconds |
| Failure mode | device left on: wasted power | machine hangs, or wakes on nothing, or loses data |
| Who decides | the driver, from its own idleness | the PM core, globally |
| Can it be refused? | no — it is driven by a refcount | yes, `->prepare`/`->suspend` may return an error and abort |

Runtime PM is a *local optimisation* running continuously. System suspend is a *global state transition* that must be atomic with respect to the entire machine. They share `struct dev_pm_ops` because the hardware operations often coincide ("turn this block off"), not because the problems are the same.

The deep reason they cannot be unified: during runtime PM, the rest of the system is live and can generate work for your device at any moment; during system suspend, it cannot, because everything has been quiesced in a defined order. Those are different environments, and code that is correct in one may deadlock in the other (§T.6).

### T.2 Runtime PM as a distributed refcount over a DAG

The entire runtime PM design is one idea:

> **A device may be powered off when nothing needs it, and "nothing needs it" is a reference count.**

```c
	pm_runtime_get_sync(dev);     /* count 0 -> 1: resume the device */
	/* ... use the hardware ... */
	pm_runtime_put(dev);          /* count 1 -> 0: schedule a suspend */
```

This is Ch. 12's refcounting applied to *hardware state* rather than object lifetime, and Ch. 43 §T.5's enable counts generalised. What makes it more than a counter is three additions:

**(a) The count propagates up the device tree.** A child's resume resumes its parent first; a parent may not suspend while any child is active (`pm_runtime_get_noresume()` on the parent, held by each active child). So the count is really a count over a DAG — the device hierarchy of Ch. 26, augmented by **device links** (Ch. 27 §T.5) for dependencies the hierarchy does not express (a device needing a clock provider that is not its parent). This is exactly why `fw_devlink` matters: it builds the graph that PM then traverses.

**(b) Suspend is deferred, not immediate.** `pm_runtime_put()` drops to zero and calls `->runtime_idle()`, which by default schedules a suspend. Most drivers use **autosuspend**:

```c
	pm_runtime_set_autosuspend_delay(dev, 100);   /* ms */
	pm_runtime_use_autosuspend(dev);
	...
	pm_runtime_mark_last_busy(dev);
	pm_runtime_put_autosuspend(dev);
```

The delay exists because **powering a device down and back up costs energy and latency**, and a device that just did work is likely to do more. Formally, this is the *ski-rental problem* again (Ch. 14 §T.2): you do not know how long the idle period will be; paying the suspend cost immediately is optimal if idle is long, staying on is optimal if idle is short, and a fixed threshold gives a 2-competitive online algorithm. The autosuspend delay is that threshold, and the right value is roughly "the energy cost of a power cycle divided by the idle power draw."

**(c) The state machine has more than two states.** `RPM_ACTIVE`, `RPM_SUSPENDED`, `RPM_RESUMING`, `RPM_SUSPENDING`, plus `disable_depth`, `runtime_error`, and `power.is_suspended`. The transient states exist because the callbacks can sleep, so a resume request can arrive while a suspend is in flight. The core handles this by making the requester *wait* for the in-flight transition and then act, which is why `pm_runtime_get_sync()` can block.

### T.3 The API, and which variant to use

This is the table to memorise; choosing wrong is the most common runtime PM bug.

| Call | Increments? | Resumes? | Blocks? | Use when |
|---|---|---|---|---|
| `pm_runtime_get_sync(dev)` | yes | yes | yes | **process context, need hardware now**; legacy — see below |
| `pm_runtime_resume_and_get(dev)` | yes | yes | yes | **preferred** — on error it drops the count for you |
| `pm_runtime_get(dev)` | yes | asynchronously | no | you will be told later; rare |
| `pm_runtime_get_noresume(dev)` | yes | **no** | no | just pin it active; you know it already is |
| `pm_runtime_get_if_active(dev)` | if active | no | no | opportunistic: use hardware only if it happens to be on |
| `pm_runtime_get_if_in_use(dev)` | if count>0 | no | no | same, stricter |
| `pm_runtime_put(dev)` | dec | idles if 0 | no | ordinary release |
| `pm_runtime_put_sync(dev)` | dec | suspends **now** | yes | teardown paths |
| `pm_runtime_put_autosuspend(dev)` | dec | idles after delay | no | **the normal choice** |
| `pm_runtime_put_noidle(dev)` | dec | no | no | undoing a `get_noresume` |

`pm_runtime_get_sync()` has a notorious trap: **it increments the count even when it returns an error.** Code like

```c
	ret = pm_runtime_get_sync(dev);
	if (ret < 0)
		return ret;            /* BUG: count leaked, device pinned on forever */
```

appears in hundreds of historical drivers and was the subject of a tree-wide cleanup. `pm_runtime_resume_and_get()` (5.10+) exists solely to make the correct behaviour the default. **Use it.** This is a good example of a general principle:

> When an API's correct use requires remembering a counterintuitive rule, the fix is a new API that encodes the rule, not more documentation.

Two more things drivers must do and often forget:

- **Tell the core the initial state.** After probe, the core assumes the device is suspended unless told otherwise. If the bootloader left it on and you call `pm_runtime_enable()` without `pm_runtime_set_active()`, the core believes it is off, will not resume it, and your first register access hits a powered-down block.

```c
	pm_runtime_set_active(dev);        /* it is ON right now */
	pm_runtime_enable(dev);
	pm_runtime_get_noresume(dev);      /* hold it while we probe */
	... probe hardware ...
	pm_runtime_put_autosuspend(dev);   /* now it may sleep */
```

- **Disable it in remove, symmetrically.** `pm_runtime_disable()` before tearing down, or a pending autosuspend timer fires into freed memory.

### T.4 Why `->runtime_suspend` must never fail for "busy"

A subtle but important rule: `->runtime_suspend()` is called when the count has already reached zero. If it returns `-EBUSY`, the core retries later — but the device is *supposed* to be idle. A driver returning `-EBUSY` is saying "my refcount does not reflect my actual usage," which means the refcount is wrong.

The correct design is:

> **Every path that touches hardware must hold a runtime PM reference. If you cannot suspend, it is because you failed to take a reference somewhere.**

Where "every path" includes: ioctls, read/write, interrupt handlers (via a reference held by whatever armed the interrupt), work items, and timers. Tracking this down is precisely the "who is holding it" debugging of Lab 48.5.

There is one legitimate use of `-EBUSY`: hardware that can only be suspended at certain points (e.g. between frames). Even then, `-EAGAIN`/`-EBUSY` should be rare and bounded.

### T.5 System suspend: the phases, and why there are so many

`ACPI S3`/`suspend-to-RAM` and `s2idle` both run the same driver callback sequence. The PM core walks the device tree in a defined order:

```
  SUSPEND (children before parents)          RESUME (parents before children)
  ┌─────────────────────────────────┐        ┌─────────────────────────────────┐
  │ ->prepare                       │        │ ->complete                      │
  │ ->suspend                       │        │ ->resume                        │
  │ ->suspend_late                  │        │ ->resume_early                  │
  │ ->suspend_noirq   [IRQs off]    │        │ ->resume_noirq    [IRQs off]    │
  └─────────────────────────────────┘        └─────────────────────────────────┘
          ↓                                            ↑
     freeze processes, disable nonboot CPUs, enter the sleep state
```

Each phase exists to solve a specific ordering problem:

| Phase | Runs with | Purpose |
|---|---|---|
| `->prepare` | everything normal | **may abort the suspend**; block new children from registering; last chance to refuse |
| `->suspend` | userspace frozen, IRQs on, other devices still up | the main work: quiesce, save state. **May sleep. May use other devices.** |
| `->suspend_late` | IRQs on, but device drivers are largely quiesced | things that must happen after all `->suspend` |
| `->suspend_noirq` | **interrupts disabled** | the final step: disable the device's interrupt, check for pending wakeups, turn off clocks/power that the IRQ path needed |
| `->resume_noirq` | IRQs disabled | restore enough to be safe with IRQs on |
| `->resume_early` / `->resume` | IRQs on | full restore |
| `->complete` | everything normal | undo `->prepare` |

The **`_noirq` phases are the interesting design**. Why do they exist? Consider a bus controller (say an I²C controller) with a device on it. The child's `->suspend` may need to send an I²C message to put the peripheral to sleep — so the controller must still work. But the controller cannot be turned off until all its children are done. Ordering alone (children before parents) handles that.

The harder case: **interrupt handlers running during suspend.** After `->suspend` returns, an interrupt could still arrive and cause the driver to touch hardware that a later phase turned off. The `_noirq` phase runs with interrupts disabled, so drivers can safely do the final teardown knowing nothing can re-enter them. It also gives a race-free place to check "did a wakeup event arrive?" — see §T.7.

The rule that follows, and it is absolute:

> **`->suspend` may sleep and may use other devices. `->suspend_noirq` may not sleep on anything that requires an interrupt, and may not use devices that are already `_noirq`-suspended.**

`SET_SYSTEM_SLEEP_PM_OPS` / `DEFINE_SIMPLE_DEV_PM_OPS` fill in `->freeze`/`->thaw`/`->poweroff`/`->restore` (the hibernation variants) with the same functions, which is correct for the vast majority of drivers.

### T.6 Why `->suspend` must not wait for userspace

Userspace is **frozen** before `->suspend` runs (`freeze_processes()`). Therefore:

> A `->suspend` callback that blocks waiting for anything userspace must do will deadlock the machine.

This sounds obvious and is violated constantly, usually indirectly:

- Waiting on a completion that a userspace thread signals.
- Allocating memory with `GFP_KERNEL` in a way that triggers reclaim needing writeback to a device already suspended (this is why `PF_MEMALLOC_NOIO` / `dev_pm_set_driver_flags(DPM_FLAG_...)` and `memalloc_noio_save()` exist — a direct parallel to Ch. 11's `memalloc_nofs_save()`).
- Calling a firmware-loading API that reads from a filesystem (Ch. 49): `request_firmware()` during resume can deadlock because the storage device may not be back. This is the reason `request_firmware_nowait()` and firmware caching (`FW_ACTION_NOUEVENT` / the PM firmware cache) exist.

The general principle:

> **During a global state transition, any dependency on a component that is also transitioning is a potential deadlock. Enumerate your dependencies and verify each is either already restored or not needed.**

The freezer itself is worth understanding: `freeze_processes()` sends every user task through a "freezing point" (`try_to_freeze()` at signal-handling and some syscall boundaries) where it parks. Kernel threads freeze only if they opt in (`set_freezable()`). Tasks in uninterruptible sleep cannot be frozen, which is why "Freezing of tasks failed after 20.00 seconds" names the offending task — that message is a complete diagnosis.

### T.7 Wakeup sources: the race that suspend must not lose

The fundamental correctness problem of suspend:

> A wakeup event (key press, network packet, USB insertion, RTC alarm) arrives *while the system is going to sleep*. If suspend completes anyway, the event is lost and the machine sleeps forever. If suspend aborts, the machine stays awake.

This is a classic lost-wakeup race (Ch. 25 P3), but distributed across every device and userspace policy. Linux's solution has two parts.

**(a) Wakeup sources with a global counter.** Every device that can wake the system registers a wakeup source (`device_init_wakeup()`, `device_set_wakeup_enable()`). When an event occurs, the driver calls `pm_wakeup_event(dev, msec)` or `pm_stay_awake(dev)`/`pm_relax(dev)`. The core maintains a global count of *in-progress* wakeup events and a monotonically increasing *total* count.

**(b) The suspend sequence checks it atomically.** `pm_wakeup_pending()` is checked at multiple points in the suspend path, including in `->suspend_noirq` (which is why that phase must exist — it is the last point where a check is meaningful). If any wakeup event occurred since the sequence started, suspend **aborts**.

For userspace, the same mechanism is exposed via `/sys/power/wakeup_count`:

```sh
count=$(cat /sys/power/wakeup_count)   # read the current total
# ... make the decision to suspend ...
echo $count > /sys/power/wakeup_count  # fails if count changed meanwhile
echo mem > /sys/power/state
```

Writing back the value you read is a **compare-and-swap on the wakeup counter**. If an event arrived between read and write, the write fails with `-EBUSY` and the daemon knows not to suspend. This is a beautiful, minimal solution to a distributed race, and it is worth studying as a general technique: *expose a version counter and require the client to prove it acted on current information.*

Per-device wakeup control lives in sysfs:

```sh
cat /sys/devices/.../power/wakeup           # enabled / disabled
cat /sys/kernel/debug/wakeup_sources        # who woke us, how often, how long
```

Two distinctions that confuse people:

- **Wakeup-capable ≠ wakeup-enabled.** The hardware can wake; policy decides whether it may. `device_init_wakeup(dev, true)` sets both; `device_set_wakeup_enable()` sets only the latter.
- **Remote wakeup during runtime suspend is a different thing** from system wakeup, though the same `->runtime_suspend` typically arms it. A USB device in runtime suspend can signal resume; the driver must arm that before suspending and must handle a resume it did not request.

### T.8 `s2idle` versus `S3`: why modern laptops changed

Historically suspend-to-RAM meant **ACPI S3**: firmware takes over, the CPU is powered off, RAM is in self-refresh, resume re-enters through firmware. Power draw is very low, resume takes ~1 s, and the *firmware* owns the transition.

Modern platforms increasingly support only **s2idle** (`suspend-to-idle`, "Modern Standby" / S0ix): the kernel freezes everything, puts all devices into their lowest runtime-PM states, and then simply idles the CPUs in a deep C-state. No firmware transition.

| | S3 | s2idle |
|---|---|---|
| Who owns the transition | firmware | **the kernel** |
| Devices must be | suspended by drivers | suspended by drivers **and actually reach low-power states** |
| Power draw | very low, guaranteed by hardware | **only as low as the worst device** |
| Resume latency | ~1 s | ~10–100 ms |
| Wakeup sources | limited to what firmware supports | any device, including network |
| Failure mode | rare, firmware-level | **a single device that fails to reach its low state ruins battery life** |

That last row is why s2idle generates so many bug reports. In S3, a buggy driver still got low power because the firmware cut the rails. In s2idle, the system is only as good as the sum of its drivers, and one device stuck in D0 costs watts.

The practical consequence: **on s2idle platforms, runtime PM correctness directly determines suspend power.** The two problems of §T.1 are not unified, but on modern hardware they became coupled — s2idle's device quiescing is largely "put everything into its runtime-suspended state." Checking `/sys/kernel/debug/pmc_core/*` (Intel) or the equivalent tells you which sub-block blocked the package from entering S0ix, and the answer is always a specific driver.

### T.9 Hibernation, and why it is a different problem again

Suspend-to-disk (`S4`) adds a requirement nothing else has: **the memory image must be written to disk by a system that is itself part of the image.**

The sequence:

1. Freeze processes.
2. **Free enough memory** that the remaining image fits in half of RAM (so it can be snapshotted).
3. `->freeze` all devices (like suspend, but the device must be quiesced *without* losing state that the snapshot needs).
4. Snapshot memory.
5. `->thaw` devices enough to write.
6. Write the image to swap.
7. Power off.

On resume, a fresh kernel boots, reads the image, and then the "restore kernel" hands control to the resumed image, calling `->restore` on every device — which must handle the fact that the hardware was **fully reinitialised by firmware and a different kernel** in between.

Hence the four extra callbacks:

| Callback | Meaning |
|---|---|
| `->freeze` | quiesce for snapshotting; do **not** power down (we still need to write the image) |
| `->thaw` | undo `->freeze` after the snapshot |
| `->poweroff` | like `->suspend`, but we will never resume from this state |
| `->restore` | like `->resume`, but assume **nothing** about hardware state |

`->restore` being stricter than `->resume` is the key insight: after S3 the hardware often retained its configuration, so drivers could get away with partial restores. After hibernation it definitely did not. A driver that works across suspend but fails across hibernate has a `->resume` that assumes retained state.

This is also why hibernation and Secure Boot/lockdown interact: restoring an arbitrary memory image is equivalent to loading arbitrary kernel code, hence image signing (`CONFIG_HIBERNATION_SNAPSHOT_VERIFY` and the ongoing work in this area).

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `drivers/base/power/runtime.c` | the runtime PM state machine — **read this** |
| `drivers/base/power/main.c` | the system suspend phase walker (`dpm_suspend_*`, `dpm_resume_*`) |
| `drivers/base/power/wakeup.c` | wakeup sources, `pm_wakeup_pending()`, the counter |
| `drivers/base/power/sysfs.c` | `/sys/devices/*/power/*` |
| `drivers/base/power/domain.c` | genpd (Ch. 43 §T.8) — the consumer of runtime PM |
| `kernel/power/suspend.c` | `pm_suspend()`, s2idle and S3 paths |
| `kernel/power/process.c` | the freezer |
| `kernel/power/hibernate.c`, `snapshot.c`, `swap.c` | hibernation |
| `kernel/power/main.c` | `/sys/power/*` |
| `include/linux/pm.h` | `dev_pm_ops`, the phase definitions |
| `include/linux/pm_runtime.h` | the consumer API of §T.3 |
| `include/linux/pm_wakeup.h` | wakeup sources |
| `Documentation/power/` | excellent and short |

### 1.2 `struct dev_pm_ops` in full

```c
struct dev_pm_ops {
	int (*prepare)(struct device *dev);
	void (*complete)(struct device *dev);

	int (*suspend)(struct device *dev);
	int (*resume)(struct device *dev);
	int (*freeze)(struct device *dev);
	int (*thaw)(struct device *dev);
	int (*poweroff)(struct device *dev);
	int (*restore)(struct device *dev);

	int (*suspend_late)(struct device *dev);
	int (*resume_early)(struct device *dev);
	int (*freeze_late)(struct device *dev);
	int (*thaw_early)(struct device *dev);
	int (*poweroff_late)(struct device *dev);
	int (*restore_early)(struct device *dev);

	int (*suspend_noirq)(struct device *dev);
	int (*resume_noirq)(struct device *dev);
	int (*freeze_noirq)(struct device *dev);
	int (*thaw_noirq)(struct device *dev);
	int (*poweroff_noirq)(struct device *dev);
	int (*restore_noirq)(struct device *dev);

	int (*runtime_suspend)(struct device *dev);
	int (*runtime_resume)(struct device *dev);
	int (*runtime_idle)(struct device *dev);
};
```

Twenty-four callbacks. Almost no driver implements more than four, because of the macros:

```c
static DEFINE_RUNTIME_DEV_PM_OPS(my_pm_ops,
				 my_runtime_suspend,
				 my_runtime_resume,
				 NULL /* idle */);
/* ^ also wires up system suspend to call the runtime callbacks via
 *   pm_runtime_force_suspend/resume — usually exactly right. */

static DEFINE_SIMPLE_DEV_PM_OPS(my_pm_ops, my_suspend, my_resume);
/* ^ system sleep only; fills freeze/thaw/poweroff/restore too. */
```

and the driver references it with `pm_ptr(&my_pm_ops)` / `pm_sleep_ptr()`, which compile to `NULL` when `CONFIG_PM` is off, letting the callbacks be `__maybe_unused` and dropped — avoiding the `#ifdef` maze that used to surround every driver's PM code (Ch. 03 §T.5's `IS_ENABLED` argument, applied structurally).

`pm_runtime_force_suspend()` / `pm_runtime_force_resume()` deserve a note: they make system suspend reuse the runtime callbacks *correctly*, including handling the case where the device was already runtime-suspended (in which case system suspend should do nothing) and restoring the right runtime state on resume. Hand-rolling this is a classic source of "device is dead after resume, but only sometimes."

### 1.3 The runtime PM state machine

```c
struct dev_pm_info {
	atomic_t		usage_count;
	atomic_t		child_count;
	unsigned int		disable_depth:3;
	unsigned int		runtime_auto:1;
	unsigned int		ignore_children:1;
	enum rpm_status		runtime_status;
	enum rpm_request	request;
	int			runtime_error;
	u64			autosuspend_delay;  /* ns */
	u64			last_busy;
	struct wakeup_source	*wakeup;
	/* ... accounting: suspended_time, active_time ... */
};
```

`rpm_resume()` and `rpm_suspend()` in `runtime.c` are ~200 lines each and contain the whole design: the retry loop for transient states, the parent propagation, the `disable_depth` gate, and the error latching. Reading them once removes all mystery from this subsystem.

Note `runtime_error`: if a runtime callback fails, the error is **latched** and all further runtime PM on that device fails until `pm_runtime_set_suspended()`/`pm_runtime_enable()` clears it. This is deliberate — a device whose power state is unknown must not be assumed.

### 1.4 The sysfs and debugfs surface

```sh
# Per device
/sys/devices/.../power/control              # "auto" (RPM on) or "on" (pinned)
/sys/devices/.../power/runtime_status       # active / suspended / suspending
/sys/devices/.../power/runtime_usage        # the refcount
/sys/devices/.../power/runtime_active_kids
/sys/devices/.../power/runtime_active_time  # ms
/sys/devices/.../power/runtime_suspended_time
/sys/devices/.../power/autosuspend_delay_ms
/sys/devices/.../power/wakeup               # enabled / disabled
/sys/devices/.../power/wakeup_count         # how many times it woke us
/sys/devices/.../power/async                # parallel suspend allowed

# System
/sys/power/state                    # mem / disk / freeze / standby
/sys/power/mem_sleep                # [s2idle] deep      <-- which "mem" means
/sys/power/disk                     # platform/shutdown/reboot/suspend
/sys/power/wakeup_count             # the CAS counter of T.7
/sys/power/pm_debug_messages
/sys/power/pm_test                  # freezer/devices/platform/processors/core
/sys/power/suspend_stats/           # success, fail, last_failed_dev, last_failed_step

# Debug
/sys/kernel/debug/wakeup_sources
/sys/kernel/debug/suspend_stats
/sys/kernel/debug/pm_genpd/pm_genpd_summary
/sys/kernel/debug/pmc_core/          # Intel: what blocked S0ix
```

`/sys/power/suspend_stats/last_failed_dev` and `last_failed_step` together name the exact driver and phase that aborted a suspend. Most people debugging suspend do not know these exist.

---

## 2. Practice

### Lab 48.1 — Survey your machine's power state

```sh
# What sleep states exist?
cat /sys/power/state
cat /sys/power/mem_sleep          # [s2idle] or [deep] -- T.8

# Which devices have runtime PM enabled, and are they using it?
for d in /sys/bus/*/devices/*/power/control; do
  dev=${d%/power/control}
  printf "%-50s %-6s %s\n" "$(basename $dev)" "$(cat $d)" \
         "$(cat ${dev}/power/runtime_status 2>/dev/null)"
done | sort -k2 | head -40

# Who is pinned active with a nonzero count?
grep -H . /sys/bus/pci/devices/*/power/runtime_usage 2>/dev/null | grep -v ':0$'

# Runtime PM effectiveness, per device
for d in /sys/bus/pci/devices/*/power; do
  a=$(cat $d/runtime_active_time); s=$(cat $d/runtime_suspended_time)
  [ $((a+s)) -gt 0 ] && printf "%-40s active %6ss suspended %6ss\n" \
    "$(basename $(dirname $d))" $((a/1000)) $((s/1000))
done
```

Questions to answer from your own machine:

1. Which devices are `control = on` (runtime PM pinned off)? For each, is that the driver's choice or a userspace policy (often `powertop`, `udev` rules, or `/sys` writes)?
2. Find a device with `runtime_status = active` and `runtime_usage = 0`. What state is that, and why is it not a contradiction? (Autosuspend timer pending.)
3. Which device has the worst active/suspended ratio? That is your battery.

Then:

```sh
sudo powertop --auto-tune       # sets control=auto everywhere; observe what changes
sudo powertop                   # the "Tunables" tab is exactly this sysfs surface
```

---

### Lab 48.2 — Add runtime PM to a driver, correctly

Start from a platform driver and add the full, correct pattern. This template is worth memorising.

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/pm_runtime.h>
#include <linux/clk.h>
#include <linux/io.h>

struct mydev {
	struct device	*dev;
	void __iomem	*regs;
	struct clk	*clk;
	atomic_t	hw_on;		/* for the lab's assertions */
};

/* ---- the only two functions that touch power ---- */

static int __maybe_unused mydev_runtime_suspend(struct device *dev)
{
	struct mydev *md = dev_get_drvdata(dev);

	dev_dbg(dev, "runtime suspend\n");
	/* Quiesce, save any state the hardware will lose, then power off. */
	writel(0, md->regs + REG_CTRL);
	atomic_set(&md->hw_on, 0);
	clk_disable_unprepare(md->clk);
	return 0;
}

static int __maybe_unused mydev_runtime_resume(struct device *dev)
{
	struct mydev *md = dev_get_drvdata(dev);
	int ret;

	dev_dbg(dev, "runtime resume\n");
	ret = clk_prepare_enable(md->clk);
	if (ret)
		return ret;
	atomic_set(&md->hw_on, 1);
	/* Restore anything the hardware lost. */
	writel(md->saved_ctrl, md->regs + REG_CTRL);
	return 0;
}

/* System sleep reuses them; force_suspend/resume handle the
 * "already runtime suspended" case correctly (1.2). */
static const struct dev_pm_ops mydev_pm_ops = {
	SET_RUNTIME_PM_OPS(mydev_runtime_suspend, mydev_runtime_resume, NULL)
	SET_SYSTEM_SLEEP_PM_OPS(pm_runtime_force_suspend,
				pm_runtime_force_resume)
};

/* ---- every hardware access is bracketed ---- */

static int mydev_do_work(struct mydev *md, u32 val)
{
	int ret;

	ret = pm_runtime_resume_and_get(md->dev);   /* NOT get_sync (T.3) */
	if (ret < 0)
		return ret;

	WARN_ON(!atomic_read(&md->hw_on));	/* lab assertion */
	writel(val, md->regs + REG_DATA);

	pm_runtime_mark_last_busy(md->dev);
	pm_runtime_put_autosuspend(md->dev);
	return 0;
}

/* ---- probe: the ordering matters ---- */

static int mydev_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	struct mydev *md;
	int ret;

	md = devm_kzalloc(dev, sizeof(*md), GFP_KERNEL);
	if (!md)
		return -ENOMEM;
	md->dev = dev;
	platform_set_drvdata(pdev, md);

	md->regs = devm_platform_ioremap_resource(pdev, 0);
	if (IS_ERR(md->regs))
		return PTR_ERR(md->regs);

	md->clk = devm_clk_get(dev, NULL);
	if (IS_ERR(md->clk))
		return dev_err_probe(dev, PTR_ERR(md->clk), "no clock\n");

	/* Bring hardware up manually for probe. */
	ret = clk_prepare_enable(md->clk);
	if (ret)
		return ret;
	atomic_set(&md->hw_on, 1);

	/* Tell the core the truth about the current state (T.3). */
	pm_runtime_set_active(dev);
	pm_runtime_set_autosuspend_delay(dev, 100);
	pm_runtime_use_autosuspend(dev);
	pm_runtime_enable(dev);
	pm_runtime_get_noresume(dev);		/* pin while we probe */

	ret = mydev_identify_hardware(md);
	if (ret)
		goto err_pm;

	ret = mydev_register_with_subsystem(md);
	if (ret)
		goto err_pm;

	pm_runtime_mark_last_busy(dev);
	pm_runtime_put_autosuspend(dev);	/* release: may now suspend */
	return 0;

err_pm:
	pm_runtime_put_noidle(dev);
	pm_runtime_disable(dev);
	pm_runtime_set_suspended(dev);
	clk_disable_unprepare(md->clk);
	return ret;
}

static void mydev_remove(struct platform_device *pdev)
{
	struct mydev *md = platform_get_drvdata(pdev);
	struct device *dev = &pdev->dev;

	mydev_unregister_from_subsystem(md);	/* no new users */

	/* Resume so teardown can touch hardware, then stop RPM entirely. */
	pm_runtime_get_sync(dev);
	pm_runtime_disable(dev);		/* cancels pending timers */
	pm_runtime_put_noidle(dev);
	pm_runtime_set_suspended(dev);

	clk_disable_unprepare(md->clk);
}

static struct platform_driver mydev_driver = {
	.driver = {
		.name = "mydev",
		.pm = pm_ptr(&mydev_pm_ops),   /* NULL if !CONFIG_PM */
	},
	.probe	= mydev_probe,
	.remove_new = mydev_remove,
};
module_platform_driver(mydev_driver);
MODULE_LICENSE("GPL");
```

Then exercise it:

```sh
D=/sys/bus/platform/devices/mydev
cat $D/power/runtime_status              # suspended, after the autosuspend delay
echo 0 | sudo tee $D/power/autosuspend_delay_ms
# trigger work through your driver's interface, then:
watch -n0.2 "cat $D/power/runtime_status $D/power/runtime_usage"
```

Deliberate bugs to introduce, one at a time:

1. **Replace `pm_runtime_resume_and_get` with `pm_runtime_get_sync` and return early on error.** Make `runtime_resume` fail (return `-EIO`). Watch `runtime_usage` climb without bound and the device pin on forever (§T.3).
2. **Remove `pm_runtime_set_active()` from probe.** The core believes the device is suspended when it is on; the first `pm_runtime_resume_and_get()` calls `runtime_resume` on already-enabled clocks → enable-count imbalance (Ch. 43 §T.5).
3. **Remove `pm_runtime_disable()` from remove.** `rmmod` while an autosuspend timer is pending → use-after-free in the timer. Confirm with KASAN.
4. **Access `md->regs` without a reference** (call `writel` from a debugfs file with no `get`). The `WARN_ON(!hw_on)` fires. This is §T.4's rule made testable — consider shipping such an assertion in debug builds.

---

### Lab 48.3 — Observe and control system suspend

```sh
# Enable verbose PM messages BEFORE suspending
echo 1 | sudo tee /sys/power/pm_debug_messages
echo N | sudo tee /sys/module/printk/parameters/console_suspend   # keep console alive

# The staged test mode: suspend only up to a point, then resume.
# This is the single best suspend debugging tool.
cat /sys/power/pm_test                # none [freezer devices platform processors core]
echo devices | sudo tee /sys/power/pm_test
sudo sh -c 'echo mem > /sys/power/state'   # suspends devices, waits 5s, resumes
sudo dmesg | tail -100
echo none | sudo tee /sys/power/pm_test
```

`pm_test` bisects the suspend path:

| Value | Stops after | Tells you |
|---|---|---|
| `freezer` | freezing processes | is a task refusing to freeze? |
| `devices` | all device callbacks | is a driver's suspend/resume broken? |
| `platform` | platform prepare | firmware/ACPI issue? |
| `processors` | disabling nonboot CPUs | CPU hotplug issue? |
| `core` | just before the real sleep | everything except the actual transition |

Then the real thing, with timing:

```sh
sudo dmesg -C
sudo systemctl suspend      # or: echo mem | sudo tee /sys/power/state
# ... wake it ...
sudo dmesg | grep -E 'PM:|suspend|resume' | head -60
cat /sys/power/suspend_stats/success
cat /sys/power/suspend_stats/fail
cat /sys/power/suspend_stats/last_failed_dev
cat /sys/power/suspend_stats/last_failed_step
```

Find the slow devices:

```sh
echo 1 | sudo tee /sys/power/pm_print_times      # or initcall_debug on cmdline
sudo systemctl suspend
sudo dmesg | grep -E 'call .* returned' | sort -t' ' -k7 -rn | head -20
```

Then the proper tool:

```sh
git clone https://github.com/intel/pm-graph
sudo ./sleepgraph.py -config config/suspend.cfg -m mem
# produces an HTML timeline of every device callback in every phase
```

`sleepgraph` output is the definitive artefact for suspend performance work: it shows each device's time in each phase, laid out on a timeline, with the dependency-forced serialisation visible.

---

### Lab 48.4 — Wakeup sources and the `wakeup_count` race

```sh
# Who can wake the machine?
cat /proc/acpi/wakeup                    # ACPI-level, x86
grep -r . /sys/devices/*/*/power/wakeup 2>/dev/null | grep enabled | head

# Who actually woke it, and how often?
sudo cat /sys/kernel/debug/wakeup_sources | column -t | head -20
```

The columns are: name, active_count, event_count, wakeup_count, expire_count, active_since, total_time, max_time, last_change, prevent_suspend_time. `prevent_suspend_time` is the one to sort by when the machine will not stay asleep.

Now demonstrate the CAS protocol of §T.7 by hand:

```sh
#!/bin/sh
# safe-suspend.sh -- the correct way for a userspace daemon to suspend
count=$(cat /sys/power/wakeup_count) || exit 1
echo "wakeup_count was $count"
if ! echo "$count" > /sys/power/wakeup_count 2>/dev/null; then
	echo "a wakeup event arrived; not suspending"
	exit 1
fi
echo mem > /sys/power/state
echo "resumed"
```

Test the race:

```sh
# Terminal 1
sudo ./safe-suspend.sh
# Terminal 2, immediately: press a key / move the mouse / plug USB
```

With enough attempts you will see the write fail — the machine correctly declined to sleep through a wakeup event. Now write the *naive* version (skip the write-back) and observe that it can sleep through the event.

Control a wakeup source:

```sh
# Stop USB from waking the machine
for d in /sys/bus/usb/devices/*/power/wakeup; do
  echo disabled | sudo tee $d >/dev/null
done
# Enable RTC wake and test
sudo rtcwake -m mem -s 20
```

Then, in a driver, add a wakeup source:

```c
	device_init_wakeup(dev, true);              /* capable + enabled */
	...
	/* in the IRQ handler for the wake event: */
	pm_wakeup_event(dev, 500);                  /* keep awake 500 ms */
	/* or, for a longer operation: */
	pm_stay_awake(dev);
	... handle ...
	pm_relax(dev);
```

Verify it appears in `/sys/kernel/debug/wakeup_sources` and that triggering it during a suspend attempt aborts the suspend.

---

### Lab 48.5 — Find what is keeping a device (or the machine) awake

**Device will not runtime-suspend.**

```sh
D=/sys/bus/pci/devices/0000:00:14.0
cat $D/power/control            # must be "auto"
cat $D/power/runtime_usage      # who holds references?
cat $D/power/runtime_active_kids  # a child is active
cat $D/power/runtime_status
```

Decision tree:

| Reading | Cause |
|---|---|
| `control = on` | userspace/udev pinned it; `echo auto >` |
| `runtime_usage > 0` | some code path holds a reference |
| `runtime_active_kids > 0` | a child device is active — go look at it |
| `runtime_status = active`, usage 0, kids 0 | autosuspend timer pending, or `runtime_suspend` returned an error |

To find *who* holds the reference:

```sh
sudo bpftrace -e '
kprobe:__pm_runtime_resume,
kprobe:__pm_runtime_get_if_active {
  @[kstack, comm] = count();
}
kprobe:__pm_runtime_idle,
kprobe:__pm_runtime_suspend {
  @put[kstack] = count();
}'
```

or, more precisely, trace the usage count transitions:

```sh
sudo trace-cmd record -e power -e rpm
sudo trace-cmd report | grep rpm_
```

The `rpm_suspend`, `rpm_resume`, `rpm_idle`, `rpm_return_int` tracepoints print the device name, the usage count, and the child count at each transition — enough to reconstruct exactly who is holding what.

**Machine will not stay asleep.**

```sh
sudo dmesg | grep -i 'wakeup\|resume'
cat /sys/power/suspend_stats/last_failed_dev
sudo cat /sys/kernel/debug/wakeup_sources | sort -k10 -rn | head
```

**Machine sleeps but drains battery (s2idle).**

```sh
# Intel: what blocked the package from S0ix?
sudo cat /sys/kernel/debug/pmc_core/slp_s0_residency_usec
sudo cat /sys/kernel/debug/pmc_core/package_cstate_show
sudo cat /sys/kernel/debug/pmc_core/ltr_show
sudo cat /sys/kernel/debug/pmc_core/substate_residencies

# AMD:
sudo cat /sys/kernel/debug/amd_pmc/s0ix_stats
```

If `slp_s0_residency` does not increase across a suspend, some device kept the package awake. Cross-reference with `pm_genpd_summary` and with each device's `runtime_status` immediately before suspend. This is §T.8's coupling in practice: **the fix is always a specific driver's runtime PM.**

---

### Lab 48.6 — Make a driver survive hibernation

Take the Lab 48.2 driver and test it across all four transitions:

```sh
# 1. Runtime suspend/resume
echo 0 | sudo tee $D/power/autosuspend_delay_ms
# exercise it, watch runtime_status toggle

# 2. System suspend (S3 or s2idle)
sudo rtcwake -m mem -s 10

# 3. s2idle explicitly
echo s2idle | sudo tee /sys/power/mem_sleep
sudo rtcwake -m mem -s 10

# 4. Hibernation
sudo swapon --show                  # need swap >= RAM
sudo rtcwake -m disk -s 20
```

Now deliberately create the bug of §T.9: make `runtime_resume()` assume a register retained its value across suspend:

```c
static int mydev_runtime_resume(struct device *dev)
{
	clk_prepare_enable(md->clk);
	/* BUG: assumes REG_CTRL survived */
	writel(readl(md->regs + REG_CTRL) | CTRL_ENABLE, md->regs + REG_CTRL);
	return 0;
}
```

Observe: works across runtime PM (the block retained state), works across s2idle on some platforms, **fails across hibernation always**. Fix it by saving state in suspend and writing it unconditionally in resume. Write down the general rule:

> `->restore` must assume the hardware is in its power-on-reset state. If `->resume` is correct under that assumption too, you only need one function.

Also test the firmware deadlock of §T.6 if your driver loads firmware:

```c
	/* In probe: cache the firmware so resume does not need the filesystem. */
	request_firmware(&fw, "mydev.bin", dev);
	/* The PM core caches firmware across suspend automatically when it was
	 * requested with request_firmware() — verify with:
	 *   /sys/kernel/debug/... or by unmounting /lib/firmware before suspend. */
```

---

### Lab 48.7 — Measure the benefit

Power management is only worth doing if it measurably helps. Build the measurement.

```sh
# Battery-based (laptop)
cat /sys/class/power_supply/BAT0/power_now      # µW, on many laptops
# or
sudo powerstat -R 10 60

# Package energy (Intel/AMD RAPL)
sudo perf stat -a -e power/energy-pkg/ -e power/energy-ram/ sleep 60

# C-state residency: the proxy for "is the CPU actually idle"
sudo turbostat --quiet --interval 5
sudo cpupower monitor
```

Experiment:

1. Baseline: idle machine, 60 s, record package energy and `turbostat` PC8/PC10 residency.
2. `echo on | sudo tee /sys/bus/pci/devices/*/power/control` — disable runtime PM everywhere. Re-measure.
3. Restore `auto`, run `powertop --auto-tune`, re-measure.
4. Suspend for 60 s with `rtcwake` and compute average power from battery charge delta.

Record the numbers. Then compute, for one device you control, the autosuspend delay that minimises energy given (a) the energy cost of one power cycle and (b) the idle power draw, and compare to the value the driver actually uses. Most drivers use a round number chosen by intuition; being able to derive it is a differentiator.

---

## 3. Mastery drills

1. Prove that the `/sys/power/wakeup_count` read-then-write-back protocol correctly prevents sleeping through a wakeup event, stating precisely the atomicity the kernel must provide for the proof to hold.

2. The autosuspend delay is a ski-rental threshold. Given a device with power-cycle energy $E_c$, active idle power $P_a$, and suspended power $P_s$, derive the break-even idle duration and show that setting the delay to that value is 2-competitive.

3. `pm_runtime_get_sync()` increments the count even on failure. Write the three-line wrapper that fixes it, then explain why the kernel added `pm_runtime_resume_and_get()` rather than changing `get_sync`'s semantics.

4. Runtime PM propagates to parents. Construct a four-device chain where a leaf's autosuspend delay effectively determines the root's, and compute the total idle time before the root can suspend.

5. `->suspend_noirq` runs with interrupts disabled. Enumerate everything a driver is forbidden from doing there, and for each give the failure mode if it does it anyway.

6. Userspace is frozen before `->suspend`. Construct three distinct deadlocks caused by a `->suspend` callback depending on a frozen component, and name the kernel mechanism that prevents each.

7. On s2idle, system power is the max over devices' states rather than a firmware-enforced floor. Model this and show why a single device failing to reach its low-power state can cost more than all the others save.

8. `->restore` must assume power-on-reset hardware state. Give an example of a driver where `->resume` can legitimately be cheaper than `->restore`, and one where they must be identical.

9. Device links (Ch. 27) add edges to the PM graph beyond the parent/child hierarchy. Construct a case where omitting a device link causes a *correct-looking* driver to fail at resume, and explain why the failure is order-dependent and therefore intermittent.

10. `runtime_error` is latched. Argue for and against automatic retry, and describe what a driver should do to recover from a transient resume failure.

11. genpd powers a domain off when all members are runtime-suspended. Design the protocol that guarantees no member's DMA is in flight at that moment, and relate each step to Ch. 25's P12.

12. Compare the runtime PM refcount to (a) `kref` (Ch. 12), (b) clock enable counts (Ch. 43), and (c) the NAPI `SCHED` bit (Ch. 46). Identify the property that makes runtime PM's counter need a full state machine while the others do not.

13. Design the instrumentation you would add to a kernel to answer, in production, "which driver is responsible for this machine's poor suspend residency?" — without requiring a reproduction or a developer.

---

## 4. Further reading

**Kernel documentation** (`Documentation/power/`)

- `runtime_pm.rst` ★★★ — the definitive reference for §T.2–T.4. Long, precise, and it answers every question about the API semantics. Read it fully at least once.
- `devices.rst` ★★★ — the system sleep phases of §T.5, including exactly what each phase may and may not do. The `_noirq` explanation is here.
- `suspend-and-interrupts.rst` ★★★ — short and essential: `IRQF_NO_SUSPEND`, `enable_irq_wake()`, and why they are different.
- `pm_qos_interface.rst` ★★ — latency constraints, `dev_pm_qos_*`; how a driver says "do not put me in a C-state deeper than 50 µs."
- `swsusp.rst`, `basic-pm-debugging.rst` ★★★ — the `pm_test` procedure of Lab 48.3 in the maintainers' words.
- `energy-model.rst`, `cpuidle/`, `cpufreq.rst` — the CPU side, adjacent to this chapter.
- `Documentation/driver-api/pm/devices.rst` and `notifiers.rst` ★★★ — the driver-author view; `devices.rst` there overlaps `power/devices.rst` with more API detail.
- `Documentation/ABI/testing/sysfs-devices-power` and `sysfs-power` ★★ — every file in §1.4, documented.

**Source worth reading**

- `drivers/base/power/runtime.c` ★★★ — `rpm_resume()`, `rpm_suspend()`, `rpm_idle()`. About 800 lines total and it is the whole subsystem. Read with §1.3's struct in front of you.
- `drivers/base/power/main.c` ★★★ — `dpm_suspend()`, `device_suspend()`, `dpm_noirq_suspend_devices()`. The phase walker, including the async parallelism.
- `drivers/base/power/wakeup.c` ★★ — `pm_wakeup_event()`, `pm_wakeup_pending()`, `wakeup_count` handling. Short.
- `kernel/power/process.c` ★★ — the freezer; `try_to_freeze_tasks()` and the "Freezing of tasks failed" message.
- `kernel/power/suspend.c` — `suspend_enter()` shows exactly where `pm_wakeup_pending()` is checked.
- Good driver examples: `drivers/i2c/busses/i2c-imx.c` (clean runtime PM), `drivers/usb/core/hub.c` (hard case: remote wakeup), `drivers/net/ethernet/intel/e1000e/netdev.c` (WoL + all four sleep states).

**Papers and background**

- L. Benini, A. Bogliolo, G. De Micheli, "A Survey of Design Techniques for System-Level Dynamic Power Management," *IEEE TVLSI* 8(3), 2000 — the theoretical framing of §T.2, including the break-even analysis.
- A. R. Karlin et al., "Competitive Randomized Algorithms for Nonuniform Problems," *Algorithmica*, 1994 — the ski-rental result behind autosuspend delays.
- E. Le Sueur and G. Heiser, "Dynamic Voltage and Frequency Scaling: The Laws of Diminishing Returns," HotPower 2010 — why device-level PM increasingly matters more than DVFS.
- The Android **wakelock** papers and LWN coverage (2010–2012) — the argument that produced `wakeup_count` and `pm_stay_awake`; historically important and still the clearest statement of the problem in §T.7.

**LWN**

- "Rethinking suspend-to-RAM" and the `wakeup_count` discussions (2010–2011) ★★★
- "Suspend blockers and the Android debate" — one of the most instructive kernel design arguments ever conducted in public; read it for the *process* as much as the content.
- "The runtime power management framework" (2009) and its follow-ups
- "System-wide suspend and runtime PM" / `pm_runtime_force_suspend()` coverage
- "Modern Standby and Linux" / s2idle coverage (2019–2022) — §T.8
- "Fixing the pm_runtime_get_sync() API" (2020–2021) — the tree-wide cleanup of §T.3

**Tools**

- `pm-graph` (`sleepgraph.py`, `bootgraph.py`) ★★★ — the essential suspend analysis tool
- `powertop` ★★★ — tunables, per-device wakeup accounting, estimated power
- `turbostat`, `cpupower monitor`, `perf stat -e power/energy-pkg/` — C-states and energy
- `rtcwake` — scripted, reproducible suspend/resume cycles
- `/sys/power/pm_test` ★★★ — bisect the suspend path
- `/sys/power/suspend_stats/`, `/sys/kernel/debug/wakeup_sources` ★★★
- `trace-cmd record -e power -e rpm` — the runtime PM tracepoints
- `/sys/kernel/debug/pmc_core/` (Intel), `/sys/kernel/debug/amd_pmc/` — S0ix blockers
- `s2ram`/`systemctl suspend`/`hibernate` and `journalctl -b -1` for post-mortem

---

→ Next: [49-firmware-remoteproc.md](49-firmware-remoteproc.md)

# Chapter 17 — Interrupts: Controllers, IRQ Domains, Threaded Handlers, MSI

> **Goal:** understand what happens between a wire toggling and your C function running,
> write handlers that are correct under every race, and know why modern hardware abandoned
> interrupt *lines* for interrupt *messages*.

---

## Theory & First Principles

### T.0 — Start here: the handler that must not do anything

A network packet arrives. The NIC raises an interrupt. Write the handler:

```c
static irqreturn_t nic_irq(int irq, void *dev_id)
{
	struct nic *n = dev_id;
	struct sk_buff *skb;

	skb = alloc_skb(2048, GFP_KERNEL);   /* ① */
	read_packet(n, skb);
	mutex_lock(&n->lock);                /* ② */
	netif_rx(skb);                       /* ⁢ */
	mutex_unlock(&n->lock);
	return IRQ_HANDLED;
}
```

**Every numbered line is a bug**, and the reasons are not about style:

① **`GFP_KERNEL` may sleep.** You are on a borrowed stack, in a context with no `task_struct`
to block, with interrupts disabled. `schedule()` here does not put "you" to sleep — there is
no "you" — it corrupts the interrupted task's state. The kernel catches it (`BUG: sleeping
function called from invalid context`) only because someone added `might_sleep()`.

⁡ **A mutex sleeps.** Same problem, and worse: if the interrupted task on *this CPU* holds
that mutex, you are waiting for a task that cannot run until you return. Self-deadlock,
guaranteed, 100% of the time.

⁢ **The whole handler runs with interrupts disabled on this CPU** — and on many
architectures, with the interrupt line masked globally. Every microsecond you spend here is a
microsecond of added latency for *every other device*, and for every real-time task on the
system (Ch. 103 §T.2).

**So the constraint is brutal and it is not negotiable:**

> An interrupt handler runs in a context with **no task**, **no ability to sleep**, **a
> borrowed stack**, and **interrupts disabled**. It must finish in microseconds.

But a packet needs protocol processing, which needs allocation, which may need reclaim, which
sleeps. **The work is unavoidable and the context forbids it.** That contradiction is what
this chapter is about, and the resolution is to **split the work in two**:

```
 hardware IRQ                      deferred context
 ┌───────────────────┐              ┌────────────────────────┐
 │ TOP HALF          │              │ BOTTOM HALF            │
 │ - ack the device  │   schedule   │ - allocate             │
 │ - grab the data   │ ─────────► │ - take mutexes         │
 │ - note what to do │              │ - run the protocol     │
 │ - RETURN FAST     │              │ - can be preempted     │
 └───────────────────┘              └────────────────────────┘
  microseconds                        milliseconds are fine
```

**The corrected handler:**

```c
static irqreturn_t nic_irq(int irq, void *dev_id)
{
	struct nic *n = dev_id;
	u32 status = readl(n->base + REG_STATUS);

	if (!(status & IRQ_PENDING))
		return IRQ_NONE;              /* shared line: not ours. Say so. */
	writel(status, n->base + REG_ACK);    /* ack so it stops asserting */
	return IRQ_WAKE_THREAD;               /* do the real work elsewhere */
}

static irqreturn_t nic_thread(int irq, void *dev_id)
{
	/* Process context. May sleep, allocate, take mutexes. */
	process_packets(dev_id);
	return IRQ_HANDLED;
}
```

**Four things to carry forward:**

1. **`IRQ_NONE` is not optional on a shared line.** Returning `IRQ_HANDLED` when it was not
   yours breaks the core's spurious-interrupt detection; returning `IRQ_NONE` from the real
   owner gets your interrupt disabled with `nobody cared`.
2. **Acknowledging the device is the top half's actual job.** A level-triggered interrupt
   that is not acked re-fires immediately — an interrupt storm that wedges the CPU
   (`debugging-scenarios.md` §6).
3. **Threaded IRQs are the modern default**, not tasklets and not softirqs. The work gets a
   schedulable identity you can prioritize, pin, and preempt — which is what makes
   `PREEMPT_RT` possible at all (Ch. 103 §T.4).
4. **There is a deeper question underneath.** At high packet rates, interrupting per packet
   is *itself* the problem: past a threshold, interrupt overhead consumes the CPU that would
   have drained the queue, and throughput collapses to zero. That is **receive livelock**
   (§T.1), the answer is NAPI (Ch. 46 §T.2), and it is a queueing-theory result rather than
   an engineering trick.

---

### T.1 Interrupts vs polling: a queueing-theory question

Two ways to learn that a device needs attention:

- **Polling**: the CPU asks. Cost is `O(poll_rate)` regardless of event rate. Latency is
  bounded by the polling interval. Predictable, wasteful at low load.
- **Interrupts**: the device tells you. Cost is `O(event_rate)`. Latency is near-minimal.
  Efficient at low load, **catastrophic at high load**.

The crossover is not a matter of taste. Let $\lambda$ be the event arrival rate and $C$ the
per-interrupt overhead (pipeline flush, register save, controller ack, cache pollution,
return-from-interrupt — realistically **1–5 µs** of *total system* cost, far more than the
handler itself). Interrupt cost is $\lambda C$. When $\lambda C \to 1$, the CPU spends 100% of
its time entering and leaving interrupts and **zero time doing useful work**.

That end state is **receive livelock**, characterized by Mogul & Ramakrishnan (TOCS 1996) in
one of the most important systems papers ever written. Their finding: an interrupt-driven
network stack under overload delivers **decreasing** throughput as offered load increases —
eventually zero — because interrupt processing has strict priority over the work that
actually drains the queue. The system is busy, and useless.

Their prescribed fix is exactly what Linux does today:

1. **Disable the interrupt** once you know there is work.
2. **Poll** while work remains (amortize `C` over many events).
3. **Re-enable** when the queue drains.

That is NAPI (`New API`, `net/core/dev.c`), and it's also the block layer's completion
batching, and `irq_poll` (`lib/irq_poll.c`) used by storage drivers. **Hybrid
interrupt/polling is not an optimization — it is a stability requirement.**

Modern variants you should know by name: **interrupt coalescing** (hardware delays the IRQ
until `N` events or `T` µs — `ethtool -C`), **interrupt moderation**, and
`io_uring`/NVMe **polled I/O** (`io_poll`), which removes interrupts entirely for
ultra-low-latency storage.

### T.2 Decomposing interrupt latency

When someone says "our interrupt latency is 30 µs", make them decompose it:

```
 device asserts
   │  t1: controller propagation + CPU interrupt recognition       (~0.1–1 µs)
   │  t2: WAIT — local interrupts disabled by someone else         (UNBOUNDED, yours to fix)
   │  t3: CPU vectoring: pipeline drain, mode switch, stack switch (~0.1–0.5 µs)
   │  t4: kernel entry + irqentry_enter + controller dispatch      (~0.5–2 µs)
   ▼  t5: your handler runs
   │  t6: scheduling the deferred half (softirq/thread wakeup)     (~1–10 µs)
   ▼  t7: the deferred half actually runs                          (depends on load)
```

Only `t2` and `t5`–`t7` are under your control, and **`t2` is the one that ruins real-time
systems**. Every `spin_lock_irqsave()` region anywhere in the kernel extends `t2` for *all*
interrupts on that CPU. This single fact explains:

- Why long `local_irq_disable()` regions are a bug, not a style issue.
- Why PREEMPT_RT converts nearly everything to threaded handlers (Ch. 28).
- Why `ftrace`'s `irqsoff` tracer exists and is the first tool you reach for.

```bash
sudo trace-cmd record -p irqsoff -M 1 sleep 5      # longest irqs-off region per CPU
sudo cat /sys/kernel/tracing/tracing_max_latency
sudo cyclictest -m -p 99 -i 200 -h 400             # the standard RT latency benchmark
```

### T.3 Why top half / bottom half exists

While a hardirq handler runs, **interrupts of the same line (and on many designs, all
interrupts) are masked on that CPU**. Every microsecond spent there is added directly to
`t2` for every other device. So the design rule follows mathematically, not aesthetically:

> **The hardirq handler must do the minimum required to (a) determine the device interrupted,
> (b) silence it, and (c) record enough state to finish later.** Everything else is deferred.

This is a **priority inversion avoidance** structure: it prevents a slow, unimportant device's
processing from blocking a fast, critical device's *recognition*.

The three deferral mechanisms (Ch. 18 covers them fully) trade latency for context:

| Mechanism | Context | Can sleep | Latency | Preemptible |
|---|---|---|---|---|
| softirq | softirq | no | ~µs | no |
| tasklet | softirq | no | ~µs | no (deprecated) |
| threaded IRQ | task | **yes** | ~10 µs | yes |
| workqueue | task | **yes** | ~10–100 µs | yes |

### T.4 Edge vs level triggering: the lost-interrupt problem

This is a correctness issue that bites everyone once.

**Level-triggered**: the line is asserted and *stays* asserted until the device is serviced.
- **Robust**: if you miss it, it's still there. You cannot lose an interrupt.
- **Shareable**: several devices can pull the same line; the handler chain runs until all say
  "not mine".
- **Requires**: the handler must clear the device's condition, or you re-enter forever
  (an "interrupt storm"). The kernel detects this and disables the IRQ:
  `irq N: nobody cared (try booting with the "irqpoll" option)`.

**Edge-triggered**: the device signals a *transition*.
- **Cheap and fast**, no need to hold a line.
- **Interrupts can be lost**: if two edges occur while masked, you get one. If the device
  raises a new event after you've read the status register but before you've acked, that
  event is lost forever and the device hangs.
- Not shareable in practice.

**The canonical edge-triggered race, and the canonical fix:**

```c
/* BROKEN: an event arriving between the check and the ack is lost. */
static irqreturn_t bad_isr(int irq, void *d)
{
	u32 status = readl(base + STATUS);
	handle(status);
	writel(status, base + ACK);
	return IRQ_HANDLED;
}

/* CORRECT: ack first, then loop until the device reports idle. */
static irqreturn_t good_isr(int irq, void *d)
{
	u32 status;
	int handled = 0;

	while ((status = readl(base + STATUS)) != 0) {
		writel(status, base + ACK);   /* ack exactly what we observed */
		handle(status);               /* new events set STATUS again */
		handled = 1;
	}
	return IRQ_RETVAL(handled);
}
```
The loop closes the window: any event that arrives during `handle()` re-asserts `STATUS` and
is caught by the next iteration. Bound the loop if the hardware can lie to you.

Linux abstracts triggering via `irq_set_irq_type()` / DT `interrupts` properties
(`IRQ_TYPE_LEVEL_HIGH`, `IRQ_TYPE_EDGE_RISING`, …), and the `irq_chip` implements the correct
mask/ack/eoi sequence per type — that's what the `handle_level_irq`, `handle_edge_irq`,
`handle_fasteoi_irq` flow handlers in `kernel/irq/chip.c` are.

### T.5 MSI: interrupts as memory writes

Traditional interrupts are **out-of-band sideband signals**: a physical wire, routed through
a controller, with a number assigned by firmware. This has three fatal problems at scale:

1. **Pin scarcity.** A PCIe card with 64 queues cannot have 64 pins.
2. **Ordering.** The interrupt travels on a different path than the DMA data. It can *arrive
   before the data it describes has landed in memory* — the classic "interrupt overtakes
   DMA" race, which forced drivers to perform a dummy read of a device register in the ISR to
   flush posted writes.
3. **Sharing.** Line sharing means every handler in the chain runs on every interrupt.

**MSI (Message Signalled Interrupts)** replaces the wire with a **posted memory write** by the
device to a magic address. The consequences are elegant:

- **Unlimited vectors** (MSI: up to 32; MSI-X: up to 2048), each independently maskable and
  independently targetable at a CPU.
- **Ordering is solved for free.** PCIe ordering rules require that posted writes complete in
  order, so the DMA data write *precedes* the interrupt write. If you receive the MSI, the
  data is in memory. **This is why MSI is not merely "more vectors" — it removes a whole
  class of driver bug.**
- **No sharing**, so `IRQF_SHARED` is unnecessary and handlers are simpler.
- Per-vector CPU affinity enables **multi-queue** devices: one queue + one vector per CPU,
  giving you an entire I/O path with zero cross-CPU sharing (the Ch. 16 principle applied to
  hardware).

MSI-X adds a per-vector table in device memory (address + data + mask per vector), which is
what makes per-vector affinity and masking possible.

`pci_alloc_irq_vectors()` is the modern one-call API that tries MSI-X, falls back to MSI,
falls back to legacy INTx:

```c
nvec = pci_alloc_irq_vectors(pdev, 1, nr_cpus,
			     PCI_IRQ_MSIX | PCI_IRQ_MSI | PCI_IRQ_INTX | PCI_IRQ_AFFINITY);
irq = pci_irq_vector(pdev, i);
```
`PCI_IRQ_AFFINITY` asks the core to spread vectors across CPUs automatically — the same
machinery `blk-mq` uses to build its queue mapping (Ch. 63).

### T.6 IRQ domains: a hierarchical namespace problem

Embedded SoCs have *trees* of interrupt controllers: a GIC at the root, a GPIO controller
hanging off one GIC line, a PMIC on I²C hanging off a GPIO, each multiplexing dozens of
sources. Two problems arise:

1. **Naming.** "IRQ 5" is meaningless — 5 on *which* controller? Historically, boards
   hard-coded a global numbering scheme in a header, which made a single kernel image for
   multiple boards impossible.
2. **Composition.** Masking an interrupt at the leaf may require masking at every level.

`irq_domain` (`kernel/irq/irqdomain.c`) solves (1) by making each controller a **translation
domain** mapping *hwirq* (the hardware-local number) → *virq* (a global, dynamically
allocated Linux IRQ number). Domain types: `linear` (array), `tree` (radix, for sparse
hwirqs), `nomap` (hwirq == virq).

`irq_domain_hierarchy` solves (2): a virq can have a **stack** of `irq_data` structures, one
per controller level. `irq_chip_mask_parent()`, `irq_chip_eoi_parent()` etc. let each level
forward operations upward. MSI is itself implemented as a domain hierarchy
(`msi_domain` → `vector_domain` → `apic`), which is why the same `irq_domain` code serves
both an i.MX GPIO expander and x86 interrupt remapping.

This is a genuinely good piece of architecture and worth studying as an example of
"turn a global namespace into composable local namespaces" — the same move as PIDs→PID
namespaces and device numbers→`fwnode`.

```bash
sudo cat /sys/kernel/debug/irq/domains/default
ls /sys/kernel/debug/irq/irqs/            # per-virq: hwirq, chip, domain, affinity
cat /proc/interrupts
```

### T.7 Threaded interrupts: moving the handler into a task

`request_threaded_irq()` splits the handler in two:

```c
request_threaded_irq(irq,
		     hard_handler,   /* hardirq context; may be NULL */
		     thread_fn,      /* task context; MAY SLEEP */
		     flags, name, dev);
```

The hard handler returns `IRQ_WAKE_THREAD` to schedule `thread_fn` on a dedicated RT kthread
(`irq/NN-name`, default SCHED_FIFO 50). The IRQ line stays masked until the thread completes.

Why this matters theoretically:

- It converts an **unbounded non-preemptible region** into a **preemptible, schedulable
  entity**. Now the scheduler — not interrupt priority — decides what runs, and you can set
  priorities, affinities, and cgroups on interrupt work.
- It makes `t2` (Ch. T.2) bounded and small, which is the entire premise of PREEMPT_RT.
- It permits sleeping in the handler, which is **mandatory** for devices behind a sleeping
  bus (I²C, SPI, regmap-over-I²C). A PMIC interrupt handler *must* do I²C reads to find out
  what happened; that cannot happen in hardirq context. Hence
  `request_threaded_irq(irq, NULL, my_thread_fn, IRQF_ONESHOT, ...)`.

`IRQF_ONESHOT` is required when there is no hard handler: it keeps the line masked until the
thread finishes, which is essential for level-triggered lines (otherwise you re-interrupt
immediately, forever).

`IRQF_NO_THREAD` marks handlers that must *not* be threaded even on RT (timer, IPIs).

On `PREEMPT_RT`, **all** handlers registered with `request_irq()` are forcibly threaded
(`force_irqthreads`), which you can also request on a normal kernel with the `threadirqs`
boot parameter. Try it; measure the latency change.

### T.8 Affinity, steering, and the NUMA dimension

Each interrupt has an affinity mask. Placing it wrong costs you:

- IRQ on CPU A, softirq on CPU A, but the application thread on CPU B → data crosses the
  interconnect twice, cache is cold everywhere.
- All IRQs on CPU 0 → CPU 0 saturates while 63 CPUs idle (the classic pre-MSI-X problem).

```bash
cat /proc/irq/42/smp_affinity_list
echo 3 | sudo tee /proc/irq/42/smp_affinity_list
cat /proc/irq/42/effective_affinity_list   # what the controller ACTUALLY does
systemctl status irqbalance                # userspace daemon; usually disable it for RT/perf
```

`IRQD_AFFINITY_MANAGED` (set by `PCI_IRQ_AFFINITY`) means **the kernel owns the mask and
userspace cannot change it** — because `blk-mq`/networking have built queue→CPU mappings that
must stay consistent. Trying to `echo` to `smp_affinity` on a managed IRQ returns `-EIO`, and
that surprises people constantly.

Related steering mechanisms, all solving "get the packet to the right CPU":
**RSS** (hardware hashes flows to queues), **RPS** (software equivalent), **RFS** (steer to
the CPU where the *application* runs), **XPS** (TX queue selection), **aRFS** (hardware
programmed from flow tables). Ch. 46 covers these.

### T.9 Spurious, shared, and the `nobody cared` problem

A shared level IRQ requires each handler to *positively identify* its device and return
`IRQ_NONE` if not involved. If every handler returns `IRQ_NONE` 100,000 consecutive times,
`note_interrupt()` in `kernel/irq/spurious.c` disables the line and prints:

```
irq 16: nobody cared (try booting with the "irqpoll" option)
```

Causes, in order of frequency: (1) you enabled the IRQ before the device was initialized —
a classic probe-ordering bug, where `request_irq()` comes before hardware setup; (2) you fail
to clear the source; (3) hardware/firmware routing is wrong. The fix for (1) is structural:
**always initialize hardware fully, then request the IRQ, and free the IRQ before tearing
hardware down.** Note that `request_irq()` can fire the handler *before it returns*, so your
handler must be able to run against a fully-set-up driver state at that instant.

### T.10 NMI: the context with no rules

A Non-Maskable Interrupt cannot be blocked by `local_irq_disable()`. Used for: hardware error
reporting (MCE), the hard lockup watchdog, `perf` PMU sampling, and KGDB entry.

NMI context is brutal:
- It can interrupt code holding *any* lock → **you may take no lock that is ever taken
  outside NMI**. In practice, no locks at all.
- It can nest with itself on some architectures.
- It can interrupt code mid-update of any per-CPU structure.
- `printk` from NMI must be lock-free — this is why
  `kernel/printk/printk_ringbuffer.c` is a carefully-constructed lock-free multi-writer
  ring buffer, one of the most sophisticated pieces of lock-free code in the tree.

Rules: use `nmi_enter()`/`nmi_exit()` (done by the core), use only per-CPU data with
`local_t`/`this_cpu_*`, use `printk_deferred`/the NMI-safe paths, and use
`in_nmi()` to branch. Anything else is a bug.

---

## 1. Concept — the API

### 1.1 Requesting an interrupt

```c
/* Classic */
ret = request_irq(irq, handler, IRQF_SHARED, "mydev", dev);

/* Threaded — the modern default for anything non-trivial */
ret = request_threaded_irq(irq, hard_isr, thread_fn,
			   IRQF_ONESHOT | IRQF_TRIGGER_FALLING, "mydev", dev);

/* Managed: freed automatically at driver detach (Ch. 28) */
ret = devm_request_threaded_irq(&pdev->dev, irq, NULL, thread_fn,
				IRQF_ONESHOT, dev_name(&pdev->dev), priv);

/* Any-context (RT-safe) handler that must never be threaded */
ret = request_irq(irq, handler, IRQF_NO_THREAD, "timer", dev);
```

Key flags:

| Flag | Meaning |
|---|---|
| `IRQF_SHARED` | line may be shared; handler must identify its device |
| `IRQF_ONESHOT` | keep masked until the threaded handler completes (**required if `hard_isr == NULL`**) |
| `IRQF_TRIGGER_*` | set edge/level type (usually comes from DT instead) |
| `IRQF_NO_SUSPEND` | keep armed across system suspend (wakeup-capable core IRQs) |
| `IRQF_NO_THREAD` | never force-thread, even on RT |
| `IRQF_PERCPU` | per-CPU interrupt (timers, IPIs); use `request_percpu_irq()` |
| `IRQF_NOBALANCING` | exclude from irqbalance |
| `IRQF_EARLY_RESUME` | resume before the normal resume phase |

Return values from a handler:

```c
IRQ_NONE          /* not my device (shared lines) */
IRQ_HANDLED       /* handled */
IRQ_WAKE_THREAD   /* run the threaded handler */
```

### 1.2 Masking and control

```c
disable_irq(irq);          /* waits for in-flight handlers to finish — MAY SLEEP */
disable_irq_nosync(irq);   /* returns immediately; safe from the handler itself */
enable_irq(irq);           /* nested: must balance */
synchronize_irq(irq);      /* wait for handlers to complete */

irq_set_affinity_hint(irq, mask);
enable_irq_wake(irq) / disable_irq_wake(irq);   /* wakeup source for suspend */
```

> `disable_irq()` from within the handler for that IRQ **deadlocks** (it waits for itself).
> Use `disable_irq_nosync()`. This is a recurring bug.

### 1.3 Getting an IRQ number

```c
irq = platform_get_irq(pdev, 0);            /* from DT/ACPI; logs its own errors */
irq = platform_get_irq_byname(pdev, "rx");
irq = of_irq_get(np, 0);
irq = fwnode_irq_get(fwnode, 0);
irq = gpiod_to_irq(gpiod);                  /* GPIO as an interrupt source */
irq = pci_irq_vector(pdev, vec);            /* after pci_alloc_irq_vectors() */
irq = i2c_client->irq;                      /* filled by the I2C core from DT/ACPI */
```
`platform_get_irq()` returns a negative errno (including `-EPROBE_DEFER` if the interrupt
controller isn't up yet — see Ch. 27). **Never** treat 0 as valid; virq 0 is reserved.

---

## 2. Internals

### 2.1 Source map

```
kernel/irq/manage.c        request_irq, threaded IRQ machinery, affinity
kernel/irq/handle.c        handle_irq_event(), the handler chain
kernel/irq/chip.c          ★ flow handlers: handle_level_irq, handle_edge_irq, handle_fasteoi_irq
kernel/irq/irqdomain.c     ★ hwirq → virq mapping, hierarchy
kernel/irq/msi.c           MSI domain core
kernel/irq/spurious.c      "nobody cared" detection
kernel/irq/irqdesc.c       struct irq_desc allocation
kernel/irq/cpuhotplug.c    migrating IRQs off a dying CPU
kernel/irq/matrix.c        vector allocation (x86)
arch/x86/kernel/apic/      APIC, IO-APIC, vector domain
drivers/irqchip/           ★ every SoC interrupt controller (GIC, GICv3, etc.)
include/linux/interrupt.h  the API
include/linux/irq.h        irq_chip, irq_data
Documentation/core-api/irq/   ★ irq-domain.rst, irqflags-tracing.rst, concepts.rst
```

### 2.2 The descriptor and chip

```c
struct irq_desc {
	struct irq_data      irq_data;      /* hwirq, chip, domain, affinity */
	struct irqaction    *action;        /* handler chain (shared IRQs) */
	irq_flow_handler_t   handle_irq;    /* handle_level_irq / handle_edge_irq / ... */
	unsigned int         irq_count, irqs_unhandled;   /* spurious detection */
	unsigned int __percpu *kstat_irqs;  /* /proc/interrupts counts */
	raw_spinlock_t       lock;
	...
};

struct irq_chip {                        /* the controller driver */
	void (*irq_mask)(struct irq_data *);
	void (*irq_unmask)(struct irq_data *);
	void (*irq_ack)(struct irq_data *);
	void (*irq_eoi)(struct irq_data *);
	int  (*irq_set_type)(struct irq_data *, unsigned int type);
	int  (*irq_set_affinity)(struct irq_data *, const struct cpumask *, bool force);
	int  (*irq_set_wake)(struct irq_data *, unsigned int on);
	...
};
```

Three layers, cleanly separated — study this, it's a model of good kernel design:

1. **`irq_chip`** — *how to talk to the controller* (mask/unmask/ack/eoi). Written once per
   controller.
2. **flow handler** — *what sequence to perform for this trigger type*. Written once per
   trigger discipline, in core code.
3. **`irqaction`** — *your driver's handler*. Knows nothing about controllers.

### 2.3 The dispatch path (x86 MSI example)

```
device writes MSI address/data
  → LAPIC delivers vector V on CPU N
  → arch/x86/entry/entry_64.S: asm_common_interrupt
  → common_interrupt()  (irqentry_enter → RCU/context tracking, irq_enter_rcu)
  → DEFINE_IDTENTRY_IRQ → handle_irq() → generic_handle_irq_desc()
  → desc->handle_irq()   e.g. handle_edge_irq()
        ├─ chip->irq_ack()
        ├─ handle_irq_event(desc)
        │     └─ for each action: action->handler(irq, action->dev_id)
        │           returns IRQ_HANDLED / IRQ_WAKE_THREAD
        │           if WAKE_THREAD → wake_up_process(action->thread)
        └─ chip->irq_unmask() (if it was masked)
  → irq_exit_rcu() → invoke pending softirqs (Ch. 18)
  → irqentry_exit() → maybe preempt → IRET/ERET
```

Note where softirqs run: **on the way out of the interrupt**, on the same CPU. That single
fact explains most of Ch. 18.

### 2.4 Statistics

```bash
cat /proc/interrupts            # per-CPU counts, chip, type, and device name
watch -d -n1 'cat /proc/interrupts'
cat /proc/stat | grep ^intr
sudo cat /sys/kernel/debug/irq/irqs/42
sudo perf stat -e irq:irq_handler_entry -a sleep 5
sudo bpftrace -e 'tracepoint:irq:irq_handler_entry { @[args->name] = count(); }'
sudo bpftrace -e 'tracepoint:irq:irq_handler_entry { @s[cpu,args->irq]=nsecs; }
                  tracepoint:irq:irq_handler_exit  /@s[cpu,args->irq]/ {
                    @us = hist((nsecs-@s[cpu,args->irq])/1000); delete(@s[cpu,args->irq]); }'
```
That last one gives you a histogram of hardirq handler durations — run it on any production
box and you will find something surprising.

---

## 3. Practice

### 3.1 A correct threaded handler with a sleeping bus

```c
// SPDX-License-Identifier: GPL-2.0
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/i2c.h>
#include <linux/interrupt.h>
#include <linux/regmap.h>

struct sensor {
	struct regmap  *regmap;
	struct device  *dev;
	int             irq;
	atomic_t        events;
};

/*
 * Hard handler: runs in hardirq context. We cannot touch I2C here, so all we
 * can do is hand off. With a level-triggered line the core keeps it masked
 * (IRQF_ONESHOT) until the thread completes, so no storm.
 */
static irqreturn_t sensor_hard_isr(int irq, void *data)
{
	struct sensor *s = data;

	atomic_inc(&s->events);
	return IRQ_WAKE_THREAD;
}

/* Threaded handler: task context, may sleep, may do I2C. */
static irqreturn_t sensor_thread_isr(int irq, void *data)
{
	struct sensor *s = data;
	unsigned int status;
	int ret;

	ret = regmap_read(s->regmap, REG_INT_STATUS, &status);   /* sleeps on I2C */
	if (ret) {
		dev_err_ratelimited(s->dev, "status read failed: %d\n", ret);
		return IRQ_NONE;
	}
	if (!status)
		return IRQ_NONE;                 /* shared line: not ours */

	regmap_write(s->regmap, REG_INT_STATUS, status);         /* W1C ack */

	if (status & INT_DATA_READY)
		sensor_push_sample(s);
	if (status & INT_OVERFLOW)
		dev_warn_ratelimited(s->dev, "FIFO overflow\n");

	return IRQ_HANDLED;
}

static int sensor_probe(struct i2c_client *client)
{
	struct sensor *s;
	int ret;

	s = devm_kzalloc(&client->dev, sizeof(*s), GFP_KERNEL);
	if (!s)
		return -ENOMEM;
	s->dev = &client->dev;
	s->irq = client->irq;
	s->regmap = devm_regmap_init_i2c(client, &sensor_regmap_config);
	if (IS_ERR(s->regmap))
		return PTR_ERR(s->regmap);
	i2c_set_clientdata(client, s);

	/* ★ ORDER MATTERS: fully initialize hardware and mask interrupts
	 * BEFORE requesting the IRQ, because the handler can fire immediately. */
	ret = sensor_hw_init(s);
	if (ret)
		return ret;

	ret = devm_request_threaded_irq(&client->dev, s->irq,
					sensor_hard_isr, sensor_thread_isr,
					IRQF_ONESHOT,
					dev_name(&client->dev), s);
	if (ret)
		return dev_err_probe(&client->dev, ret, "failed to request irq\n");

	return sensor_enable_interrupts(s);      /* only now unmask at the device */
}
```

Three things to internalize from this:
1. **Hardware init → request_irq → enable device interrupts.** Never reorder.
2. `devm_request_threaded_irq()` frees the IRQ at detach *in the right order* relative to
   other `devm_` resources (Ch. 28).
3. `dev_err_probe()` handles `-EPROBE_DEFER` silently and logs everything else — use it in
   every probe error path.

### 3.2 Interrupt-to-poll transition (the Mogul & Ramakrishnan fix)

```c
#include <linux/irq_poll.h>

struct dev_priv {
	struct irq_poll iop;
	...
};

static int mydev_poll(struct irq_poll *iop, int budget)
{
	struct dev_priv *p = container_of(iop, struct dev_priv, iop);
	int done = 0;

	while (done < budget && mydev_has_completion(p)) {
		mydev_process_one(p);
		done++;
	}

	if (done < budget) {
		irq_poll_complete(iop);           /* queue drained */
		mydev_enable_irq(p);              /* re-arm hardware interrupt */
	}
	return done;
}

static irqreturn_t mydev_isr(int irq, void *data)
{
	struct dev_priv *p = data;

	mydev_disable_irq(p);                     /* ★ stop the interrupt source */
	irq_poll_sched(&p->iop);                  /* drain in softirq, budgeted */
	return IRQ_HANDLED;
}

/* init: irq_poll_init(&p->iop, weight, mydev_poll);  teardown: irq_poll_disable() */
```
This is the same shape as NAPI (`napi_schedule`/`napi_complete_done`) — recognize it and you
recognize half the drivers in the tree.

### 3.3 Latency measurement lab

```bash
# 1. Find your worst irqs-off region
sudo sh -c 'echo 0 > /sys/kernel/tracing/tracing_max_latency'
sudo sh -c 'echo irqsoff > /sys/kernel/tracing/current_tracer; echo 1 > /sys/kernel/tracing/tracing_on'
# ... run workload ...
sudo cat /sys/kernel/tracing/trace | head -50

# 2. Compare threaded vs non-threaded
#    boot with  threadirqs   and re-run cyclictest
sudo cyclictest -m -S -p 99 -i 200 -d 0 -l 100000 -h 400 -q

# 3. Watch a specific IRQ's handler duration
sudo funclatency-bpfcc 'mydev_isr'      # bcc
```

---

## 3.4 Extended practice

### Lab 17.A — Decompose interrupt latency on your machine (T.2)

```bash
# 1. The worst irqs-off region in the whole kernel (t2 from T.2)
sudo sh -c 'echo 0 > /sys/kernel/tracing/tracing_max_latency'
sudo sh -c 'echo irqsoff > /sys/kernel/tracing/current_tracer'
sudo sh -c 'echo 1 > /sys/kernel/tracing/tracing_on'
# ...run a real workload (fio, iperf3, kernel build)...
sudo cat /sys/kernel/tracing/tracing_max_latency
sudo head -60 /sys/kernel/tracing/trace         # the actual offending call path

# Repeat with the other latency tracers:
for t in preemptoff preemptirqsoff wakeup wakeup_rt; do
  sudo sh -c "echo 0 > /sys/kernel/tracing/tracing_max_latency"
  sudo sh -c "echo $t > /sys/kernel/tracing/current_tracer"
  sleep 20
  echo "$t: $(sudo cat /sys/kernel/tracing/tracing_max_latency) us"
done
sudo sh -c 'echo nop > /sys/kernel/tracing/current_tracer'

# 2. Hardirq handler duration, per IRQ (t5)
sudo bpftrace -e '
tracepoint:irq:irq_handler_entry { @s[cpu] = nsecs; @n[args->irq] = str(args->name); }
tracepoint:irq:irq_handler_exit  /@s[cpu]/ {
	@us[@n[args->irq]] = hist((nsecs - @s[cpu]) / 1000); delete(@s[cpu]); }'

# 3. hardirq -> threaded-handler wakeup delay (t6)
sudo bpftrace -e '
tracepoint:irq:irq_handler_exit { @t[args->irq] = nsecs; }
tracepoint:sched:sched_wakeup /str(args->comm) =~ "irq/*"/ { @wake = hist(nsecs); }'

# 4. End-to-end RT latency (the number customers care about)
sudo cyclictest -m -S -p 99 -i 200 -d 0 -l 200000 -h 400 -q
```
Produce a table: `t2_max`, `t5_p99` per device, and cyclictest `max`. **Then repeat every
measurement after booting with `threadirqs`** and compare. That before/after table is the
entire argument for threaded interrupts.

### Lab 17.B — A complete driver for QEMU's `edu` device (real MMIO + IRQ)

This gives you a real interrupt source you fully control. Boot QEMU with `-device edu`.

```c
// SPDX-License-Identifier: GPL-2.0
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/pci.h>
#include <linux/interrupt.h>
#include <linux/io.h>
#include <linux/completion.h>

#define EDU_ID          0x00
#define EDU_FACT        0x08    /* write N -> device factorializes it */
#define EDU_STATUS      0x20
#define  EDU_STATUS_BUSY BIT(0)
#define  EDU_STATUS_IRQ  BIT(7)
#define EDU_IRQ_STATUS  0x24
#define EDU_IRQ_RAISE   0x60
#define EDU_IRQ_ACK     0x64

struct edu {
	void __iomem      *mmio;
	struct pci_dev    *pdev;
	struct completion  done;
	atomic_t           irqs;
};

static irqreturn_t edu_isr(int irq, void *data)
{
	struct edu *e = data;
	u32 status = readl(e->mmio + EDU_IRQ_STATUS);

	if (!status)
		return IRQ_NONE;               /* shared INTx line: not ours */

	writel(status, e->mmio + EDU_IRQ_ACK);   /* ack EXACTLY what we observed */
	atomic_inc(&e->irqs);
	complete(&e->done);
	return IRQ_HANDLED;
}

static int edu_probe(struct pci_dev *pdev, const struct pci_device_id *id)
{
	struct edu *e;
	int ret, nvec;

	e = devm_kzalloc(&pdev->dev, sizeof(*e), GFP_KERNEL);
	if (!e)
		return -ENOMEM;
	e->pdev = pdev;
	init_completion(&e->done);

	ret = pcim_enable_device(pdev);
	if (ret)
		return ret;
	e->mmio = pcim_iomap_region(pdev, 0, KBUILD_MODNAME);
	if (IS_ERR(e->mmio))
		return PTR_ERR(e->mmio);
	pci_set_master(pdev);
	pci_set_drvdata(pdev, e);

	dev_info(&pdev->dev, "edu id=0x%08x\n", readl(e->mmio + EDU_ID));

	/* Try MSI first, fall back to INTx — T.5 */
	nvec = pci_alloc_irq_vectors(pdev, 1, 1, PCI_IRQ_MSI | PCI_IRQ_INTX);
	if (nvec < 0)
		return nvec;
	dev_info(&pdev->dev, "using %s\n", pdev->msi_enabled ? "MSI" : "INTx");

	/* ★ hardware is mapped and quiesced BEFORE we request the IRQ */
	ret = devm_request_irq(&pdev->dev, pci_irq_vector(pdev, 0), edu_isr,
			       pdev->msi_enabled ? 0 : IRQF_SHARED,
			       KBUILD_MODNAME, e);
	if (ret)
		return dev_err_probe(&pdev->dev, ret, "request_irq\n");

	/* Fire one interrupt and wait for it */
	writel(0xABCD, e->mmio + EDU_IRQ_RAISE);
	if (!wait_for_completion_timeout(&e->done, HZ))
		dev_err(&pdev->dev, "no interrupt!\n");
	else
		dev_info(&pdev->dev, "got %d irq(s)\n", atomic_read(&e->irqs));
	return 0;
}

static const struct pci_device_id edu_ids[] = {
	{ PCI_DEVICE(0x1234, 0x11e8) },
	{ }
};
MODULE_DEVICE_TABLE(pci, edu_ids);

static struct pci_driver edu_driver = {
	.name = KBUILD_MODNAME, .id_table = edu_ids, .probe = edu_probe,
};
module_pci_driver(edu_driver);
MODULE_LICENSE("GPL");
```
```bash
qemu-system-x86_64 ... -device edu
# in the guest:
sudo insmod edu.ko && dmesg | tail
grep edu /proc/interrupts
sudo cat /sys/kernel/debug/irq/irqs/$(awk '/edu/{gsub(":","",$1); print $1}' /proc/interrupts)
```
Then run it both ways — `-device edu` (MSI capable) and forcing INTx with
`pci=nomsi` — and compare `/proc/interrupts` and the handler's `IRQ_NONE` rate.

### Lab 17.C — Reproduce the edge-triggered lost-interrupt race (T.4)

Use a GPIO loopback on real hardware (a Pi or a BeagleBone), or simulate with `gpio-mockup`:

```bash
sudo modprobe gpio-mockup gpio_mockup_ranges=-1,32 gpio_mockup_named_lines
ls /sys/kernel/debug/gpio-mockup/
```
Write two ISR variants — the broken read-then-ack and the correct ack-then-loop — and drive
the line at high frequency. Count events produced vs events handled.

```c
/* Instrument both variants: */
static atomic_t produced, handled, lost;

static irqreturn_t bad_isr(int irq, void *d)
{
	u32 st = readl(base + STATUS);   /* event arriving HERE is lost */
	handle(st);
	writel(st, base + ACK);
	atomic_inc(&handled);
	return IRQ_HANDLED;
}

static irqreturn_t good_isr(int irq, void *d)
{
	u32 st;
	int n = 0;

	while ((st = readl(base + STATUS)) && n++ < 64) {
		writel(st, base + ACK);
		handle(st);
		atomic_inc(&handled);
	}
	return n ? IRQ_HANDLED : IRQ_NONE;
}
```
At high event rates `bad_isr` will show `handled < produced`. **That gap is the lost
interrupt, and on real hardware it means a permanently hung device.**

### Lab 17.D — Walk an IRQ domain hierarchy (T.6)

```bash
# On arm64 QEMU virt, or any SoC:
sudo ls /sys/kernel/debug/irq/domains/
sudo cat /sys/kernel/debug/irq/domains/default
for d in /sys/kernel/debug/irq/domains/*; do echo "== $d"; sudo head -6 "$d"; done

# Per-virq detail: hwirq, chip, domain chain, affinity
for i in $(ls /sys/kernel/debug/irq/irqs/ | head -12); do
  echo "--- virq $i"; sudo cat /sys/kernel/debug/irq/irqs/$i
done

# On x86, see the MSI domain stack:
sudo cat /sys/kernel/debug/irq/domains/* | grep -A4 -i 'msi\|vector\|remap'
```
Draw the hierarchy for one GPIO interrupt: `gpio chip → (parent) GIC` or
`PCI-MSI → IR-PCI-MSI → VECTOR → APIC`. Then find where each level's `irq_chip` lives
in `drivers/irqchip/` and read its `mask`/`unmask`/`eoi`.

### Lab 17.E — Affinity: measure the cost of getting it wrong (T.8)

```bash
NIC=eth0
IRQS=$(grep "$NIC" /proc/interrupts | awk '{gsub(":","",$1); print $1}')

# (a) All NIC IRQs on CPU 0
for i in $IRQS; do echo 0 | sudo tee /proc/irq/$i/smp_affinity_list >/dev/null; done
sudo systemctl stop irqbalance
iperf3 -c <peer> -t 20 -P 8 ; mpstat -P ALL 1 5 | tail -20

# (b) Spread across all CPUs
c=0; for i in $IRQS; do echo $c | sudo tee /proc/irq/$i/smp_affinity_list >/dev/null;
     c=$(( (c+1) % $(nproc) )); done
iperf3 -c <peer> -t 20 -P 8 ; mpstat -P ALL 1 5 | tail -20

# (c) Check what the hardware ACTUALLY did (managed IRQs ignore you):
for i in $IRQS; do
  echo "$i req=$(cat /proc/irq/$i/smp_affinity_list) eff=$(cat /proc/irq/$i/effective_affinity_list)"
done

# Steering knobs:
cat /sys/class/net/$NIC/queues/rx-0/rps_cpus
cat /proc/sys/net/core/rps_sock_flow_entries
ethtool -l $NIC; ethtool -x $NIC; ethtool -c $NIC     # queues, RSS table, coalescing
```
Record throughput, `%si` per CPU, and p99 latency for each case. Then explain the difference
using Ch. 16 T.1's cache-line cost model.

### Lab 17.F — Interrupt coalescing: find the latency/throughput knee (T.1)

```bash
for usecs in 0 1 8 32 128; do
  sudo ethtool -C eth0 rx-usecs $usecs rx-frames 0 2>/dev/null || continue
  echo "=== rx-usecs=$usecs"
  IRQ0=$(grep eth0 /proc/interrupts | awk '{s=0; for(i=2;i<=NF-2;i++) s+=$i; print s}' | head -1)
  iperf3 -c <peer> -t 10 | tail -3
  sudo ping -f -c 10000 <peer> | tail -2         # latency side
  IRQ1=$(grep eth0 /proc/interrupts | awk '{s=0; for(i=2;i<=NF-2;i++) s+=$i; print s}' | head -1)
  echo "interrupts: $((IRQ1-IRQ0))"
done
```
Plot throughput, interrupt count, and p99 latency vs `rx-usecs`. You are empirically
locating the point on the Mogul & Ramakrishnan curve where $\lambda C$ stops dominating.

### Lab 17.G — Provoke "nobody cared" (T.9)

Write a driver that requests a shared level IRQ and *always* returns `IRQ_NONE`, then share
it with a device that actually interrupts.

```bash
sudo insmod badshare.ko
# generate interrupts on the shared line, then:
dmesg | grep -A20 'nobody cared'
cat /proc/interrupts | grep <line>      # count stops advancing: the IRQ was disabled
$EDITOR kernel/irq/spurious.c           # note_interrupt(): 99900/100000 heuristic
```
Then fix it by requesting the IRQ *after* hardware init and returning `IRQ_HANDLED`
correctly, and confirm the warning disappears.

---

## 4. Mastery drills

1. **Read the livelock paper** (Mogul & Ramakrishnan, 1996). Then find NAPI's implementation
   of their fix in `net/core/dev.c` (`napi_poll`, `net_rx_action`, the `netdev_budget` and
   `netdev_budget_usecs` limits). Map each paper concept to each function.

2. **Edge race.** Write a module + a QEMU device (or use `null_blk`/a GPIO loopback) that
   reproduces the lost-edge race in §T.4. Fix it with the loop. Prove the fix with a counter.

3. **IRQ domain walk.** On an ARM64 board or QEMU `virt` machine, dump
   `/sys/kernel/debug/irq/domains/*` and draw the hierarchy. Trace one GPIO interrupt from
   `gpiod_to_irq()` through every domain level to the GIC.

4. **Read a real `irq_chip`.** `drivers/irqchip/irq-gic-v3.c`. Identify `irq_mask`,
   `irq_eoi`, the `handle_fasteoi_irq` flow, and how affinity is programmed. Then read a
   simple one (`drivers/irqchip/irq-sifive-plic.c`) and compare.

5. **Spurious interrupt.** Deliberately request an IRQ *before* initializing hardware in a
   driver you control, on a shared level line. Trigger "nobody cared". Read
   `kernel/irq/spurious.c` and explain the heuristic.

6. **Affinity experiment.** Pin a NIC's IRQs to CPU 0 only, run `iperf3`, measure throughput
   and `%si` in `mpstat`. Then spread across all CPUs with `PCI_IRQ_AFFINITY`-managed
   vectors. Explain the difference in terms of Ch. 16's cache-line theory.

7. **MSI ordering.** Explain to a colleague why a legacy-INTx driver often needs a dummy
   register read in the ISR and an MSI-X driver does not. Cite PCIe posted-write ordering.

8. **Threaded conversion.** Take a driver in `drivers/` using `request_irq()` with a long
   handler. Convert it to `request_threaded_irq()`. Measure `irqsoff` before and after.
   This is a legitimate upstream patch category.

9. **NMI thought experiment.** Why can't you call `spin_lock()` in an NMI handler even with
   `_irqsave`? Why is `printk()` allowed? Read `printk_ringbuffer.c`'s design comment.

---

## 5. Further reading

**Kernel docs:**
- `Documentation/core-api/irq/concepts.rst`, `irq-domain.rst`, `irqflags-tracing.rst`,
  `irq-affinity.rst`
- `Documentation/PCI/msi-howto.rst`
- `Documentation/devicetree/bindings/interrupt-controller/` — the binding conventions
- `Documentation/driver-api/basics.rst` (IRQ section)

**Papers / classics:**
- Mogul & Ramakrishnan, "Eliminating Receive Livelock in an Interrupt-Driven Kernel"
  (USENIX 1996 / TOCS 1997) — **the foundational paper; read it**
- Druschel & Banga, "Lazy Receiver Processing" (OSDI 1996) — the companion idea
- Salah et al., surveys of interrupt-handling schemes (polling/NAPI analysis with queueing
  models) — for the quantitative treatment of T.1

**Specs worth skimming:**
- PCI Express Base Spec, §6.1 (MSI/MSI-X) and §2.4 (transaction ordering)
- ARM Generic Interrupt Controller Architecture Spec (GICv3/v4) — the shape of modern
  interrupt hardware

**LWN:**
- "Threaded interrupt handlers" (Corbet)
- "The irq_domain interrupt number mapping library"
- "Interrupt handling and the realtime tree"
- "Toward less-annoying background tasks" (softirq/threadirqs discussions)

→ Next: [18-deferred-work.md](18-deferred-work.md)

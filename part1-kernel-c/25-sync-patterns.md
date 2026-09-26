# Chapter 25 — Kernel Synchronization Design Patterns

> **Goal:** stop *inventing* concurrency solutions and start *recognizing* them. This chapter
> is the capstone of Part 1: a catalogue of the ~15 structures that account for essentially
> all correct concurrent code in the kernel, each with its forces, its proof obligation, its
> failure mode, and real in-tree users.

---

## Theory & First Principles

### T.0 — Start here: you now know the primitives and still cannot write this

Chapters 12–19 gave you every tool: atomics, barriers, spinlocks, mutexes, RCU, refcounts,
per-CPU, wait queues, workqueues. Now write this function, which every driver needs:

> *Look up an object in a hash table by id, and return it ready to use. Another CPU may be
> deleting it at the same moment. The lookup is on a fast path and must not take a lock.*

Here is the attempt almost everyone writes first:

```c
struct obj *lookup(u32 id)
{
	struct obj *o;

	rcu_read_lock();
	hash_for_each_possible_rcu(table, o, node, id)
		if (o->id == id) {
			rcu_read_unlock();
			return o;               /* ← BUG */
		}
	rcu_read_unlock();
	return NULL;
}
```

**Every primitive is used correctly and the function is still wrong.** `rcu_read_unlock()`
ended the grace-period protection, so `o` may be freed before the caller touches it. You have
written Ch. 12 §T.0's use-after-free with RCU decoration.

Second attempt — take a reference:

```c
	hash_for_each_possible_rcu(table, o, node, id)
		if (o->id == id) {
			refcount_inc(&o->ref);  /* ← STILL A BUG */
			rcu_read_unlock();
			return o;
		}
```

If the last reference was dropped just before this, the count is 0 and destruction is already
underway. You have **resurrected a dying object**. The correct primitive is
`refcount_inc_not_zero`, and it must be *inside* the RCU read-side section:

```c
struct obj *lookup(u32 id)
{
	struct obj *o;

	rcu_read_lock();
	hash_for_each_possible_rcu(table, o, node, id) {
		if (o->id != id)
			continue;
		if (!refcount_inc_not_zero(&o->ref))   /* already dying: skip */
			continue;
		rcu_read_unlock();
		return o;                              /* caller must obj_put() */
	}
	rcu_read_unlock();
	return NULL;
}
```

**Nothing here required a primitive you did not already know.** What it required was knowing
that *this exact configuration of forces* — lockless lookup, concurrent deletion, an object
outliving its lookup — has a known solution with known obligations. That is **P2, Refcounted
Lookup**, and there are about fifteen such patterns covering essentially all correct
concurrent kernel code.

**Why a catalogue beats more primitives**, stated as the three facts that make concurrency
different:

1. **You cannot test your way to correctness.** A missing barrier or a lost wakeup may work
   for years and fail on a new CPU. Ch. 13 §T.0 showed an outcome that occurs once in a
   million iterations on one microarchitecture and never on another. Correctness must be
   *constructed*.
2. **The primitives are not the hard part.** Knowing `refcount_inc_not_zero` exists does not
   tell you it belongs inside the RCU section. **The combination is the pattern.**
3. **Review is pattern-matching.** A maintainer reading your patch asks "*which pattern is
   this, and did they discharge all its obligations?*" Code that matches no known pattern has
   an enormous review cost — justifiably, because the reviewer must now verify it from first
   principles against every interleaving.

**Two obligations account for nearly every bug in this territory**, and learning to look for
them first — in your own code and in review — is most of the value of this chapter:

- **Publication:** if a reader can see a pointer, it must see everything written before it
  was published. (§T.4a, Ch. 13 §T.0)
- **Quiescence:** before freeing or unloading, prove nobody can still be executing that code
  or touching that data. (§T.4b — every `synchronize_rcu`, `cancel_work_sync`, `free_irq`,
  `del_timer_sync`, `kthread_stop` is an instance)

---

### T.1 Why a pattern catalogue, and not more primitives

Christopher Alexander, *A Pattern Language* (1977), and Gamma et al., *Design Patterns*
(1994), made the same observation about different domains:

> Expert practitioners do not solve each problem from scratch. They recognize a *recurring
> configuration of forces* and apply a known *structure* that resolves them, with known
> trade-offs and known failure modes.

Concurrency is the domain where this matters most, because:

1. **The failure modes are not observable by testing.** A missing barrier or a lost wakeup
   may work for years and then fail on a new CPU or under a new load. You cannot debug your
   way to correctness; you must *construct* it.
2. **The primitives are not the hard part.** You learned `spin_lock`, `rcu_read_lock`,
   `refcount_inc_not_zero`, `smp_store_release` in Ch. 12–19. Knowing them does not tell you
   which *combination* is correct for "look up an object in a hash table that another CPU may
   be deleting". The combination is the pattern.
3. **Review is pattern-matching.** A maintainer reading your patch is asking *"which pattern
   is this, and did they get all its obligations?"* If your code does not match a known
   pattern, the review cost explodes — and rightly so.

### T.2 Every pattern is a protocol with a proof obligation

For each pattern in this chapter, four things must be stated. Get in the habit of writing
them in your commit messages and comments:

| | Question |
|---|---|
| **Invariant** | What is true at every instant, for every observer? |
| **Protocol** | The ordered steps each participant must perform. |
| **Obligation** | What the *compiler and CPU* must guarantee (barriers, atomicity). |
| **Failure mode** | What breaks, and how it manifests, if a step is omitted. |

A pattern is correct when the protocol, executed by all participants, preserves the invariant
under *every* interleaving permitted by the memory model (Ch. 13). "Every interleaving" is
why you cannot test your way there.

### T.3 The three questions that select a pattern

Before writing any concurrent code, answer these. They map almost mechanically onto the
catalogue below.

**1. What is the shape of the sharing?**

| Shape | Patterns |
|---|---|
| One writer publishes, many readers consume | P1 Publish/Subscribe, P11 Seqlock Snapshot |
| Many readers look things up; writers add/remove | P2 Refcounted Lookup, P9 Read-Mostly Registry |
| One side produces work, another consumes it | P6 Producer/Consumer, P13 Deferred Work |
| One side waits for a condition the other establishes | P3 Wait/Wakeup, P7 Completion |
| State evolves through defined transitions | P8 State Machine |
| A counter or statistic | P10 Per-CPU Fold, P16 Split Counter |
| Something must be created exactly once | P5 Lazy Init |
| Something must be destroyed exactly once, safely | P12 Stop-Drain-Free |

**2. What is the read:write ratio, and what is the read-side cost budget?**
Read-dominated + zero-cost budget → RCU. Balanced → a lock. Write-dominated → a lock, and
think about sharding.

**3. What contexts are involved?**
Any atomic-context participant forbids sleeping primitives and forces `_irqsave`/`_bh`
variants. Any userspace-visible lifetime (an open fd) forbids `devm_`-style parent ownership.

### T.4 The two universal obligations

Nearly every bug in this chapter's territory is one of two omissions. Learn to look for them
first, in your own code and in review.

**(a) The publication obligation.** *If a reader can observe a pointer, it must be able to
observe everything the writer did before publishing it.* Requires release/acquire or
dependency ordering (Ch. 13 T.3, the MP litmus test). Omitting it yields a reader that sees
a half-initialized object — usually a NULL deref inside a "valid" object.

**(b) The quiescence obligation.** *Before freeing or unloading, you must prove no participant
can still be executing code or touching data that is about to disappear.* Requires a grace
period, a `_sync` cancel, a refcount drain, or a barrier. Omitting it yields use-after-free
at teardown — the single most common driver crash.

Every `rcu_barrier()`, `cancel_work_sync()`, `timer_shutdown_sync()`, `free_irq()`,
`kthread_stop()`, `synchronize_rcu()`, `percpu_ref_kill()+wait`, and `flush_workqueue()` in
the kernel is an instance of (b). When you see one, name it: *"that's the quiescence step."*

---

## 1. The catalogue

### P1 — Publish / Subscribe (initialize, then publish)

**Problem.** A writer builds an object and makes it visible to readers.
**Invariant.** Any reader that observes the pointer observes a fully-initialized object.
**Obligation.** Release on the store, acquire (or dependency) on the load.

```c
/* Publisher */
struct cfg *new = kzalloc(sizeof(*new), GFP_KERNEL);
new->a = 1; new->b = 2; new->c = 3;
rcu_assign_pointer(gp, new);          /* == smp_store_release() */

/* Subscriber */
rcu_read_lock();
p = rcu_dereference(gp);              /* dependency-ordered load */
if (p)
	use(p->a, p->b, p->c);        /* guaranteed to see 1,2,3 */
rcu_read_unlock();
```

**Failure mode.** Plain `gp = new;` + plain `p = gp;` → on ARM64/PowerPC a reader sees the
new pointer with stale field values. Reproduces only on weakly-ordered hardware, months
later, at a customer.
**Variants.** `smp_store_release`/`smp_load_acquire` when RCU is not involved;
`list_add_rcu()` for lists; `xa_store()` (which contains the release internally).
**In tree.** Everywhere: `net/core/dev.c` (`rcu_assign_pointer(dev->...)`), sysctl tables,
`cred`, routing tables.

---

### P2 — Refcounted Lookup (the composition of Ch. 12 and Ch. 15)

**Problem.** Look up an object in a shared container and use it *after* the lookup returns.
**Invariant.** The returned pointer is valid until the caller `put()`s it.
**Obligation.** The memory must remain readable during the attempt (RCU), and the acquire
must be atomic and fallible (`inc_not_zero`).

```c
struct obj *obj_get(u32 key)
{
	struct obj *o;

	rcu_read_lock();                              /* memory stays valid */
	o = xa_load(&table, key);                     /* or list_for_each_entry_rcu */
	if (o && !refcount_inc_not_zero(&o->ref))     /* object may be dying */
		o = NULL;
	rcu_read_unlock();
	return o;                                     /* caller must obj_put() */
}

static void obj_release(struct kref *kref)        /* refcount hit zero */
{
	struct obj *o = container_of(kref, struct obj, ref);

	spin_lock(&table_lock);
	xa_erase(&table, o->key);                     /* UNPUBLISH first */
	spin_unlock(&table_lock);
	kfree_rcu(o, rcu);                            /* then defer the free */
}
```

**Failure mode.** `refcount_inc()` instead of `_not_zero` → incrementing a corpse.
Dropping `rcu_read_lock()` → the memory itself may be gone before you read the counter.
**Decision rule.** If the pointer never escapes the read-side critical section, you do *not*
need the refcount (that is P9). If it does, you need both halves.
**In tree.** `fs/file.c` (`fget`), socket lookup, `drivers/base/core.c`.

---

### P3 — Wait / Wakeup without lost wakeups ★

**The single most important pattern in this chapter**, and the one most often written wrong.

**Problem.** Task A waits for a condition that task B will establish.
**The race.** A checks the condition (false), and *before* A sleeps, B sets the condition and
sends a wakeup. A then sleeps — forever. This is the **lost wakeup**.

**The protocol.** The waiter must be *registered on the wait queue and marked non-running*
**before** re-checking the condition, so that any wakeup after the registration is not lost:

```c
/* WAITER — the raw form, so you can see the ordering */
DEFINE_WAIT(wait);
for (;;) {
	prepare_to_wait(&wq, &wait, TASK_INTERRUPTIBLE);  /* 1. enqueue + set state */
	                                                   /*    contains smp_mb()     */
	if (condition)                                     /* 2. RE-CHECK after that  */
		break;
	if (signal_pending(current)) { ret = -ERESTARTSYS; break; }
	schedule();                                        /* 3. sleep */
}
finish_wait(&wq, &wait);

/* WAKER */
WRITE_ONCE(condition, true);      /* 1. establish the condition */
wake_up(&wq);                     /* 2. wake — contains the matching barrier */
```

**Obligation.** `set_current_state()` / `prepare_to_wait()` contain a full memory barrier
precisely so that the state change is visible before the condition is re-read, and
`wake_up()` contains one so the condition store is visible before the wakeup. **This is an SB
litmus test** (Ch. 13 T.3) — both sides store then load, and only a full barrier forbids the
bad outcome. That is why nothing weaker than `smp_mb()` works here.

**In practice, use the helper and never hand-roll it:**

```c
ret = wait_event_interruptible(wq, condition);        /* -ERESTARTSYS on signal */
ret = wait_event_killable(wq, condition);             /* ★ prefer this over uninterruptible */
ret = wait_event_timeout(wq, condition, msecs_to_jiffies(500));   /* 0 on timeout */
ret = wait_event_interruptible_timeout(wq, cond, to);
wait_event(wq, condition);                            /* only if the wait is PROVABLY bounded */
```

**Failure modes.**
- `if (!cond) schedule();` — classic lost wakeup; hangs under load, never in testing.
- Checking the condition **before** `prepare_to_wait()` — same race.
- `wake_up()` before establishing the condition — the waiter re-checks and sleeps again.
- Forgetting `finish_wait()` — the waiter stays on the queue; later list corruption.
- Using `wait_event()` (uninterruptible) where the wait can be unbounded → unkillable
  `D`-state tasks (Ch. 20 T.6).

**Variant — exclusive wakeups.** With many waiters on one resource, `wake_up()` wakes all of
them and all but one go back to sleep: the **thundering herd**. Use
`wait_event_exclusive()`/`prepare_to_wait_exclusive()` + `wake_up()` (which wakes one
exclusive waiter), or `wake_up_nr()`.

**In tree.** `kernel/sched/wait.c`, and roughly every driver in the tree.

---

### P4 — Double-Checked Locking (the correct forms)

**Problem.** Take an expensive lock only when initialization is actually needed.
**The classic bug.** The naive form is broken without barriers — the reader can see the
pointer before the object's fields (it is P1's failure, hidden inside an `if`).

```c
/* BROKEN — the textbook C++/Java bug, equally broken here */
if (!obj) {
	mutex_lock(&lock);
	if (!obj)
		obj = create();        /* plain store: fields may be invisible */
	mutex_unlock(&lock);
}
use(obj);                              /* plain load: may see a half-built object */

/* CORRECT — release/acquire */
p = smp_load_acquire(&obj);
if (!p) {
	mutex_lock(&lock);
	p = obj;                       /* under the lock, a plain read is fine */
	if (!p) {
		p = create();
		smp_store_release(&obj, p);
	}
	mutex_unlock(&lock);
}
use(p);

/* CORRECT — and usually better: let the infrastructure do it */
static DEFINE_STATIC_KEY_FALSE(feature_enabled);
static DEFINE_MUTEX(init_lock);

if (static_branch_unlikely(&feature_enabled))   /* zero-cost when off */
	use_feature();
```

**Better still, avoid the pattern.** The kernel prefers:
- `DO_ONCE()` / `DO_ONCE_SLOW()` — run something exactly once, cheaply thereafter.
- `static_key` / `static_branch` — runtime-patched branches with no load at all.
- Initialize eagerly at `module_init`/`probe` time and remove the question.

**In tree.** `net/core/utils.c` (`DO_ONCE`), `lib/once.c`, tracepoint enablement.

---

### P5 — Lazy One-Time Initialization

**Problem.** Exactly one caller must perform an initialization; all others must wait for it
or skip it.

```c
/* (a) Cheap, no sleeping: */
static DEFINE_STATIC_KEY_FALSE(inited);
DO_ONCE(expensive_setup, arg);

/* (b) Sleeping init, callers must wait for completion: */
static struct completion init_done;
static atomic_t init_state = ATOMIC_INIT(0);   /* 0=none 1=in progress 2=done */

int get_resource(void)
{
	if (atomic_read(&init_state) == 2)
		return 0;                          /* fast path */
	if (atomic_cmpxchg(&init_state, 0, 1) == 0) {
		int ret = do_slow_init();          /* only one winner runs this */

		atomic_set(&init_state, ret ? 0 : 2);
		complete_all(&init_done);
		return ret;
	}
	wait_for_completion(&init_done);           /* losers wait */
	return atomic_read(&init_state) == 2 ? 0 : -EIO;
}
```

**Failure mode.** Two callers both initializing (missing the CAS) → double allocation, leaked
resource, or double registration. Or: the loser proceeds before the winner finishes → uses an
uninitialized resource.
**In tree.** Firmware loading, crypto algorithm registration, deferred hardware bring-up.

---

### P6 — Producer / Consumer Ring

**Problem.** One side generates items at high rate; the other consumes them.
**Invariant.** Items are neither lost nor duplicated; the consumer never reads a slot the
producer has not finished writing.

```c
/* SPSC — no lock at all, just release/acquire (Ch. 13 Lab 13.E, Ch. 10 T.5) */
static bool push(struct ring *r, void *item)
{
	u32 head = r->head;                          /* only we write head */
	u32 tail = smp_load_acquire(&r->tail);       /* see the consumer's progress */

	if (head - tail > r->mask)
		return false;
	r->buf[head & r->mask] = item;               /* write the DATA */
	smp_store_release(&r->head, head + 1);       /* then publish the INDEX */
	return true;
}

/* Or just use the kernel's: */
DEFINE_KFIFO(fifo, struct sample, 256);
kfifo_put(&fifo, s);  kfifo_get(&fifo, &s);
```

**Combine with P3** for blocking semantics:
```c
/* producer */  kfifo_put(&fifo, s); wake_up_interruptible(&rq);
/* consumer */  wait_event_interruptible(rq, !kfifo_is_empty(&fifo));
```

**Failure mode.** Publishing the index before the data (P1's failure). Masking the indices
*in storage* rather than on use, destroying the empty/full distinction (Ch. 10 T.5).
MPSC/MPMC written as if SPSC — kfifo's lockless guarantee is **single**-producer,
**single**-consumer only; more than one of either needs a lock.
**In tree.** `kfifo`, ftrace's ring buffer, `printk_ringbuffer`, virtio rings, io_uring SQ/CQ.

---

### P7 — Completion / One-Shot Handoff

**Problem.** A caller starts an asynchronous operation and must wait for it to finish once.

```c
struct op {
	struct completion done;
	int               result;
};

/* initiator */
init_completion(&op->done);
submit_to_hardware(op);
if (!wait_for_completion_timeout(&op->done, msecs_to_jiffies(5000))) {
	abort_operation(op);        /* ★ you MUST handle the timeout path */
	return -ETIMEDOUT;
}
return op->result;

/* completer (often an ISR or a threaded handler) */
op->result = status;            /* establish the data FIRST */
complete(&op->done);            /* then signal */
```

**Failure mode.** The timeout path that does not actually stop the operation — the hardware
completes later and writes into a freed `struct op`. **A timeout without an abort is a
use-after-free waiting to happen**, and it is one of the most common driver bugs.
`complete_all()` vs `complete()`: `complete()` releases exactly one waiter, `complete_all()`
releases all and marks the completion permanently done.
**In tree.** Every DMA-based driver, firmware upload, `drivers/base/dd.c` deferred probe.

---

### P8 — State Machine Under a Lock

**Problem.** An object moves through defined states; transitions must be atomic and illegal
transitions must be impossible.

```c
enum dev_state { DEV_IDLE, DEV_STARTING, DEV_RUNNING, DEV_STOPPING, DEV_DEAD };

static int dev_start(struct mydev *d)
{
	guard(mutex)(&d->lock);

	switch (d->state) {
	case DEV_IDLE:
		d->state = DEV_STARTING;
		break;
	case DEV_RUNNING:
		return 0;                       /* idempotent */
	case DEV_DEAD:
		return -ENODEV;
	default:
		return -EBUSY;                  /* ★ explicit about every state */
	}
	/* NOTE: lock is dropped while the slow hardware work happens; the
	 * STARTING state is what prevents a concurrent second start. */
	...
}
```

**Design rules that make this pattern work:**
- **Enumerate every state explicitly**, including the `default:` case. A silent fall-through
  is how illegal transitions happen.
- Use **intermediate states** (`STARTING`, `STOPPING`) so you can drop the lock during slow
  work without allowing a concurrent transition.
- Make operations **idempotent** where possible — it removes whole classes of races.
- Consider `WARN_ONCE()` on a transition you believe is impossible; it documents the
  invariant *and* checks it (Ch. 07 T.5).

**In tree.** `drivers/base/dd.c`, USB device states, NVMe controller states
(`nvme_change_ctrl_state()` is a textbook example — read it).

---

### P9 — Read-Mostly Registry

**Problem.** A list of registered handlers/devices, read constantly, modified rarely.
**Key decision.** Does the pointer escape the read-side critical section? If not, **no
refcount is needed** — this is P2 minus the refcount, and it is much cheaper.

```c
/* Registration (rare, under a lock) */
spin_lock(&reg_lock);
list_add_rcu(&h->node, &handlers);
spin_unlock(&reg_lock);

/* Unregistration */
spin_lock(&reg_lock);
list_del_rcu(&h->node);
spin_unlock(&reg_lock);
synchronize_rcu();              /* ★ quiescence: no reader can still see it */
/* now the caller may free h, or return from module_exit */

/* Use (hot path, no locks, no atomics) */
rcu_read_lock();
list_for_each_entry_rcu(h, &handlers, node)
	h->fn(arg);             /* pointer never escapes this section */
rcu_read_unlock();
```

**Failure mode.** Calling a handler that can sleep from inside `rcu_read_lock()` → use SRCU.
Forgetting `synchronize_rcu()` before the module unloads → a call into freed text.
**In tree.** Notifier chains, `netdev` hooks, LSM hook lists, tracepoint callbacks.

---

### P10 — Per-CPU Accumulate, Fold on Read

**Problem.** A statistic updated on every operation, read occasionally.

```c
static DEFINE_PER_CPU(struct stats, pcpu_stats);

/* fast path: no shared cache line, no atomics (Ch. 16) */
this_cpu_inc(pcpu_stats.packets);
this_cpu_add(pcpu_stats.bytes, len);

/* slow path: O(nr_cpus), APPROXIMATE by construction */
static void read_stats(struct stats *out)
{
	int cpu;

	memset(out, 0, sizeof(*out));
	for_each_possible_cpu(cpu) {           /* ★ possible, not online */
		struct stats *s = per_cpu_ptr(&pcpu_stats, cpu);

		out->packets += READ_ONCE(s->packets);
		out->bytes   += READ_ONCE(s->bytes);
	}
}
```

**Obligation.** Accept that the read is not an atomic snapshot. If you need a consistent
multi-field snapshot per CPU, add a per-CPU seqcount (`u64_stats_sync`, which is what
networking uses on 32-bit to avoid torn 64-bit counters).
**Failure mode.** `for_each_online_cpu()` loses an offlined CPU's accumulated counts.
**In tree.** `netdev` stats (`u64_stats_t`), `vmstat`, `percpu_counter`.

---

### P11 — Seqlock Snapshot

**Problem.** A small, copyable structure read very frequently, written rarely, where readers
must never block writers.

```c
/* writer */
write_seqlock(&tk_lock);
tk.a = ...; tk.b = ...; tk.c = ...;
write_sequnlock(&tk_lock);

/* reader — must be PURE */
do {
	seq = read_seqbegin(&tk_lock);
	local = tk;                 /* copy only; no pointer chasing, no side effects */
} while (read_seqretry(&tk_lock, seq));
use(&local);
```

**Obligation.** The read section may observe a torn intermediate state, so it may only
*copy* and then validate. No dereferencing pointers read inside, no division by values read
inside, no side effects.
**Failure mode.** Dereferencing a pointer read in the read section → the writer freed it →
UAF. Using a torn value before the retry check → arbitrary behaviour.
**Variant.** `seqcount_latch_t` keeps two copies so readers never retry — required when a
reader can run in NMI and could otherwise deadlock against an interrupted writer.
**In tree.** `kernel/time/timekeeping.c`, `fs/namespace.c` mount hash, `latch_tree` for
module address lookup.

---

### P12 — Stop, Drain, Free (the quiescence pattern) ★

**Problem.** Tear down an object while other participants may still be using it.
**This is T.4(b), and it is the most common source of driver crashes.**

The protocol is always the same three steps, in this order:

```c
static void my_teardown(struct mydev *d)
{
	/* 1. STOP: make it impossible for NEW users to start. */
	mydev_mask_irq(d);                 /* hardware stops generating work */
	WRITE_ONCE(d->shutting_down, true);/* self-rearming work stops re-arming */
	unregister_chrdev_region(...);     /* no new open() */
	list_del_rcu(&d->node);            /* no new lookups */

	/* 2. DRAIN: wait for every EXISTING user to finish. */
	synchronize_rcu();                 /* RCU readers */
	free_irq(d->irq, d);               /* waits for in-flight handlers */
	timer_shutdown_sync(&d->timer);    /* waits for a running callback */
	cancel_delayed_work_sync(&d->poll);
	cancel_work_sync(&d->work);
	kthread_stop(d->thread);
	flush_workqueue(d->wq);
	wait_for_completion(&d->last_io_done);
	rcu_barrier();                     /* outstanding call_rcu callbacks */

	/* 3. FREE: now, and only now, nothing can reach it. */
	destroy_workqueue(d->wq);
	kfree(d);                          /* or kref_put() for the last reference */
}
```

**The checklist** (memorize it; apply it to every teardown you write or review):

| If you started… | You must drain with… |
|---|---|
| `request_irq()` | `free_irq()` — it waits for in-flight handlers |
| `mod_timer()` | `timer_shutdown_sync()` |
| `hrtimer_start()` | `hrtimer_cancel()` |
| `queue_work()` / `queue_delayed_work()` | `cancel_work_sync()` / `cancel_delayed_work_sync()` |
| `kthread_run()` | `kthread_stop()` (and the thread must poll `kthread_should_stop()`) |
| `call_rcu()` | `rcu_barrier()` |
| `list_add_rcu()` / published a pointer | `synchronize_rcu()` after unpublishing |
| a DMA transfer | wait for completion **and** `dma_unmap_*` |
| a `percpu_ref` | `percpu_ref_kill()` + wait for the release callback |
| anything userspace can hold an fd to | a refcount — **not** `devm_` (Ch. 12 Lab 12.E) |

**Failure mode.** Freeing before draining → KASAN `slab-use-after-free`, or a jump into
unmapped module text at `rmmod`.
**Ordering matters:** stop *before* drain, or new users appear while you are draining.

---

### P13 — Deferred Work with Safe Cancellation

**Problem.** A handler defers work; the object may be destroyed before the work runs.

```c
/* Self-rearming work needs a flag AND a sync cancel (Ch. 18 Lab 18.D) */
static void poll_fn(struct work_struct *w)
{
	struct mydev *d = container_of(to_delayed_work(w), struct mydev, poll);

	scoped_guard(mutex, &d->lock) {
		if (d->shutting_down)
			return;                /* ★ do not re-arm */
		do_poll(d);
	}
	queue_delayed_work(d->wq, &d->poll, HZ);
}

/* teardown — P12 applied */
scoped_guard(mutex, &d->lock)
	d->shutting_down = true;
cancel_delayed_work_sync(&d->poll);
```

**Failure mode.** `cancel_delayed_work_sync()` alone on self-rearming work is racy: the
running instance can re-queue *after* the cancel returns.
**In tree.** Countless drivers; also the reason `disable_work_sync()` was added (6.10+) —
it disables *and* cancels atomically, removing the need for the flag.

---

### P14 — Optimistic Retry (CAS loop)

**Problem.** Update shared state without a lock, when contention is rare.

```c
static void add_to_max(atomic_long_t *v, long new)
{
	long old, prev = atomic_long_read(v);

	do {
		old = prev;
		if (new <= old)
			return;
	} while ((prev = atomic_long_cmpxchg(v, old, new)) != old);
}

/* Modern preferred form — try_cmpxchg updates `old` for you and generates better code */
long old = atomic_long_read(v);
do { if (new <= old) return; } while (!atomic_long_try_cmpxchg(v, &old, new));
```

**Obligations.** A *failed* `cmpxchg` provides **no ordering** (Ch. 13 T.11) — do not rely on
it. Beware **ABA** (Ch. 13 T.12) if the value is a pointer. Bound the loop or prove progress.
**Failure mode.** Unbounded retry under heavy contention (livelock) — at which point a lock
is faster. Measure before choosing this.
**In tree.** `percpu_ref`, `qspinlock`, SLUB's fast path, `refcount_t` itself.

---

### P15 — Hand-Over-Hand Locking (lock coupling)

**Problem.** Traverse a linked structure while other threads modify it, without holding a
global lock.

```c
spin_lock(&head->lock);
cur = head->next;
while (cur) {
	spin_lock(&cur->lock);          /* acquire the next BEFORE releasing the previous */
	spin_unlock(&prev->lock);
	if (match(cur))
		break;
	prev = cur;
	cur = cur->next;
}
```

**Invariant.** At every instant you hold at least one lock on the path, so the node you are
standing on cannot be freed.
**Obligation.** A strict global order (always forward) or you get ABBA deadlock.
**Reality check.** The kernel rarely uses this — RCU (P9) is almost always better, since it
achieves the same thing with *zero* reader cost. Know it so you can recognize it, and know
that proposing it in review will get you asked "why not RCU?"
**In tree.** Some filesystem directory traversals; `dcache`'s `d_lock` sequences use a
related discipline.

---

### P16 — Split Counter / Hierarchical Budget

**Problem.** Enforce a global limit exactly, at a rate that makes a shared counter impossible.

```c
struct budget {
	atomic64_t    global;          /* the exact authority */
	int __percpu *local;           /* pre-claimed tokens */
};
#define BATCH 64

static bool budget_get(struct budget *b)
{
	int *res;

	preempt_disable();
	res = this_cpu_ptr(b->local);
	if (*res > 0) { (*res)--; preempt_enable(); return true; }   /* fast: no sharing */
	preempt_enable();
	return budget_refill_slow(b);        /* one atomic per BATCH acquisitions */
}
```

**Idea.** Amortize the shared atomic over `BATCH` operations by pre-claiming. Exactness is
preserved because the global counter is the only authority; per-CPU reserves are *claims
already deducted*.
**Obligation.** CPU-hotplug must return an offlined CPU's reserve.
**In tree.** `percpu_counter_compare()`, memcg's `memcg_stock`, `percpu_ref`, blk-mq tags
(`sbitmap`).

---

## 2. Anti-patterns

Recognize these in review; each has a named correct replacement.

| Anti-pattern | Why it's wrong | Use instead |
|---|---|---|
| `volatile` for concurrency | no ordering, no atomicity (Ch. 08 T.8) | `READ_ONCE`/`WRITE_ONCE`, or a lock |
| `if (!cond) schedule();` | lost wakeup (P3) | `wait_event_*()` |
| `msleep(10)` in a loop to "wait for" something | timing-dependent; breaks under load | `wait_event_timeout()` or `readl_poll_timeout()` |
| `while (busy) ;` | burns a CPU, may never exit | a proper wait, with a timeout |
| A "big driver lock" for everything | serializes unrelated work | per-object locks; document the order |
| `spin_lock()` held across a sleeping call | deadlock / "scheduling while atomic" | restructure; `DEBUG_ATOMIC_SLEEP` catches it |
| Taking a lock in an interrupt handler that is also taken without `_irqsave` | self-deadlock (Ch. 14 T.7) | `spin_lock_irqsave()` |
| `atomic_t` used as a refcount | silent overflow → UAF (Ch. 12 T.6) | `refcount_t` / `kref` |
| A flag checked then acted on without a lock | TOCTOU | `cmpxchg`, or hold the lock across both |
| `kfree()` without draining | UAF at teardown | P12 |
| `devm_kzalloc()` for an object userspace holds an fd to | freed while referenced | `kref` |
| Reader-writer lock "because there are many readers" | often slower than a mutex (Ch. 14 T.9) | measure; consider RCU/seqlock |
| Retry loop with no bound | livelock | bound it, or take a lock |
| A lock whose protected data you cannot name | it protects nothing | name the data in a comment, or delete the lock |

---

## 3. How to document a pattern in your code

Reviewers look for this. Put it at the top of the file or above the struct:

```c
/*
 * Locking and lifetime model
 * ==========================
 *
 * Object graph:
 *   mydev (kref)  owns  ->  queue[] (embedded)  ->  request (kref, from a mempool)
 *   A request holds a reference on its mydev for its whole lifetime.
 *
 * Locks, in acquisition order:
 *   1. mydev->reg_mutex   - registration/teardown; sleeps; task context only
 *   2. queue->lock        - spinlock; taken from hardirq  => ALWAYS _irqsave
 *   3. request->lock      - spinlock; leaf lock, never nests
 *
 * Patterns in use (Ch. 25):
 *   - P2  Refcounted lookup: mydev_get() uses RCU + refcount_inc_not_zero().
 *   - P3  Completion of a request: wait_event_timeout() + an explicit abort path.
 *   - P9  Read-mostly registry for the device list.
 *   - P12 Teardown order is: mask IRQ -> synchronize_rcu -> free_irq ->
 *         cancel_work_sync -> drain requests -> kref_put.
 *
 * Contexts:
 *   mydev_isr()        - hardirq. May not sleep. Takes queue->lock only.
 *   mydev_work()       - task context (workqueue). May sleep.
 *   mydev_ioctl()      - task context, user-triggered. Validates all input.
 *
 * RCU: mydev->config is RCU-protected; writers hold reg_mutex and use
 *      rcu_replace_pointer(); readers use rcu_dereference() and must not
 *      retain the pointer past rcu_read_unlock().
 */
```

Writing this **before** the code is the single highest-leverage habit in kernel development.
It forces you to answer T.2's four questions, and it is what a maintainer reads first.

---

## 4. Practice

### Lab 25.1 — Build a driver that uses eight patterns correctly

Write one module implementing a virtual device with:
a registry of instances (**P9**), lookup by id returning a reference (**P2**),
an RCU-protected config (**P1**), a submit/complete request path (**P7**),
a kfifo of results with blocking reads (**P6** + **P3**), per-CPU statistics (**P10**),
a state machine (**P8**), and a correct teardown (**P12**).

Then verify with the full debug config (Ch. 06 Lab 6.6):
```bash
./scripts/config -e PROVE_LOCKING -e PROVE_RCU -e PROVE_RCU_LIST -e KASAN -e KCSAN \
                 -e DEBUG_OBJECTS -e DEBUG_OBJECTS_WORK -e DEBUG_OBJECTS_TIMERS \
                 -e DEBUG_OBJECTS_RCU_HEAD -e DEBUG_ATOMIC_SLEEP -e DEBUG_LIST
# Then hammer it:
while true; do sudo insmod pat.ko && ./stress_test && sudo rmmod pat; done
```
**The module must survive an insmod/rmmod loop under concurrent load with zero splats.**
That is the acceptance criterion.

### Lab 25.2 — Reproduce the lost wakeup (P3)

```c
static bool cond;
static DECLARE_WAIT_QUEUE_HEAD(wq);
static int mode;   /* 0 = broken, 1 = correct */

static int waiter(void *unused)
{
	if (mode == 0) {
		/* BROKEN: check, then sleep — the window is between them */
		while (!READ_ONCE(cond)) {
			set_current_state(TASK_INTERRUPTIBLE);
			/* widen the race window artificially */
			udelay(50);
			schedule();
		}
		__set_current_state(TASK_RUNNING);
	} else {
		wait_event_interruptible(wq, READ_ONCE(cond));
	}
	pr_info("waiter woke\n");
	return 0;
}

static int waker(void *unused)
{
	udelay(50);
	WRITE_ONCE(cond, true);
	wake_up(&wq);
	return 0;
}
```
Run `mode=0` in a loop; it will hang (detected by `CONFIG_DETECT_HUNG_TASK`). Run `mode=1`;
it never does. Then read `prepare_to_wait()` and find the `smp_mb()`, and write down which
litmus test from Ch. 13 it is solving.

```bash
./scripts/config -e DETECT_HUNG_TASK --set-val DEFAULT_HUNG_TASK_TIMEOUT 20
dmesg | grep -A20 'blocked for more than'
sudo cat /proc/$(pgrep waiter)/stack
```

### Lab 25.3 — Break each pattern deliberately, and identify the detector

For each row, write the broken version, run it under the debug config, and record **which
tool caught it and what the message was**. This table is the deliverable.

| Pattern | Break it by… | Expected detector |
|---|---|---|
| P1 Publish | plain store instead of `rcu_assign_pointer` | KCSAN; or a reader crash on arm64 |
| P2 Lookup | `refcount_inc` instead of `_not_zero` | KASAN `slab-use-after-free` |
| P3 Wait | check-then-sleep | `hung_task` watchdog |
| P4 DCL | plain load/store | KCSAN data race |
| P7 Completion | no abort on timeout | KASAN UAF from the late completion |
| P9 Registry | `list_for_each_entry` inside `rcu_read_lock` | `PROVE_RCU_LIST` |
| P9 Registry | no `synchronize_rcu` before free | KASAN |
| P10 Per-CPU | `for_each_online_cpu` in the fold | wrong counts after hotplug |
| P11 Seqlock | dereference a pointer in the read section | KASAN |
| P12 Teardown | drop `rcu_barrier()` | KASAN / `DEBUG_OBJECTS_RCU_HEAD` |
| P12 Teardown | drop `cancel_work_sync()` | `DEBUG_OBJECTS_WORK` |
| P13 Rearm | cancel without the flag | KASAN, intermittently |
| — | sleep under a spinlock | `DEBUG_ATOMIC_SLEEP` |
| — | ABBA lock order | `PROVE_LOCKING` |

### Lab 25.4 — Pattern-match real subsystems

Pick three subsystems you have not read. For each, produce the "Locking and lifetime model"
comment block from §3, purely by reading the code. Suggested targets:

```bash
$EDITOR drivers/char/random.c        # per-CPU, seqlock-ish, crediting; lots of patterns
$EDITOR drivers/base/dd.c            # probe/remove state machine, deferred probe
$EDITOR net/core/dev.c               # netdev registry, RCU, per-CPU, NAPI
$EDITOR block/blk-mq.c               # percpu_ref, tags, state machine, completion
$EDITOR kernel/workqueue.c           # pools, rescuers, cancellation
$EDITOR drivers/nvme/host/core.c     # ★ nvme_change_ctrl_state() is a textbook P8
```
Then check yourself against `Documentation/` and the file's own comments. **Count how many of
the sixteen patterns you can find in a single driver** — a mature driver usually uses six to
ten.

### Lab 25.5 — Review a real patch series

Find a driver patch on lore.kernel.org that adds concurrency. Before reading the replies,
write the review *you* would send, using T.2's four questions and §2's anti-pattern table.
Then compare with what the maintainers actually said.

```bash
b4 mbox -o /tmp <message-id>
b4 shazam <message-id>     # apply it locally to read in context
```
Do this five times. It is the fastest way to calibrate your judgement against the community's.

---

## 5. Mastery drills

1. **Write the catalogue from memory.** Name all sixteen patterns, their invariant, and
   their failure mode. Repeat weekly until automatic.

2. **For each pattern, find two real in-tree users** from different subsystems. Note how the
   implementations differ and why.

3. **The P3 barrier.** Read `set_current_state()`, `prepare_to_wait()`, and
   `wake_up()`/`try_to_wake_up()`. Identify every memory barrier and state exactly what each
   orders. Then write the litmus test and run it through `herd7` (Ch. 13 Lab 13.A).

4. **The quiescence audit.** Take any driver in `drivers/`. Enumerate everything it starts,
   and verify there is a matching drain in `remove()`. Find one that is missing or in the
   wrong order. (They exist. This is a real patch.)

5. **Pattern substitution.** Take a subsystem using a reader-writer lock. Determine whether
   P9 (RCU) or P11 (seqlock) would apply. Estimate the read-side cost change. If it's a win,
   write the patch.

6. **Design under constraints.** Design the synchronization for: a driver whose hardware
   completes requests in an ISR, whose requests are submitted from userspace via an ioctl,
   which supports `poll()`, which can be unbound at any time, and which must support 1M
   requests/sec on a 64-core machine. Name every pattern you use and justify each.

7. **Find the missing barrier.** `git log --oneline --grep='smp_' --grep='Fixes' --all-match
   -- drivers/ | head -20`. For each, identify the pattern that was broken and which
   obligation from T.4 was missed.

8. **Anti-pattern hunt.** `git grep -n 'volatile' -- drivers/ | head -40` and
   `git grep -n -B2 -A2 'msleep' -- drivers/ | grep -B2 -A2 'while' | head -40`.
   Classify each hit against §2's table. Fix one.

9. **Write the model comment** (§3) for a driver you maintain or have read. Post it as a
   documentation patch. Maintainers are consistently receptive to these, and writing one is
   how you *discover* whether the model is actually coherent.

10. **The composition question.** Explain, with reference to Ch. 12 T.3 and Ch. 15 T.4, why
    P2 requires *both* RCU and a fallible refcount, and why neither alone is sufficient.
    Then explain when `SLAB_TYPESAFE_BY_RCU` changes the answer and what extra obligation it
    adds.

11. **Rust preview.** For each of P1, P2, P3, P7, P12, describe how Rust's type system
    would express or enforce the obligation (`Arc`, `ARef`, `Pin`, `Drop`, lifetimes, `Send`/
    `Sync`). Which obligations does Rust make *impossible to forget*, and which remain the
    programmer's responsibility? (This is the bridge to Part 5.)

12. **Build your own checklist.** Produce a one-page concurrency review checklist derived
    from T.2, T.3, T.4, and §2. Use it on every patch you write for a month. Refine it.

---

## 6. Further reading

**Kernel documentation:**
- `Documentation/kernel-hacking/locking.rst` ★★ — the closest thing to an official version of
  this chapter; read it completely
- `Documentation/locking/locktypes.rst` — which primitive, by context
- `Documentation/RCU/checklist.rst` ★★ — **a pattern-obligation checklist in disguise**
- `Documentation/RCU/rcuref.rst` — P2, formally
- `Documentation/core-api/kref.rst`, `refcount-vs-atomic.rst`
- `Documentation/memory-barriers.txt` §"Sleep and wake-up functions" — P3's barriers
- `Documentation/core-api/workqueue.rst` — P13's cancellation semantics

**Books:**
- McKenney, *Is Parallel Programming Hard, And, If So, What Can You Do About It?* —
  **free**; Ch. 5–9 are this chapter with full derivations, by the author of RCU
- Herlihy & Shavit, *The Art of Multiprocessor Programming* — the algorithmic foundations
- Gamma, Helm, Johnson & Vlissides, *Design Patterns* (1994) — for the *form* of a pattern
  catalogue
- Schmidt et al., *Pattern-Oriented Software Architecture Vol. 2: Patterns for Concurrent and
  Networked Objects* — the closest prior art to this chapter, from the userspace world
- Love, *Linux Kernel Development*, Ch. 9–10 — the introductory framing
- Corbet, Rubini & Kroah-Hartman, *Linux Device Drivers* 3e, Ch. 5 — dated APIs, correct
  patterns

**Papers:**
- All of Ch. 13–16's references apply here; this chapter is their synthesis.
- Engler & Ashcraft, "RacerX: Effective, Static Detection of Race Conditions and Deadlocks"
  (SOSP 2003) — what static analysis can and cannot find in exactly this territory
- Lu et al., "Learning from Mistakes: A Comprehensive Study on Real World Concurrency Bug
  Characteristics" (ASPLOS 2008) — **an empirical taxonomy of concurrency bugs**; maps
  closely onto §2's anti-patterns

**LWN:**
- "Sleepable RCU", "The RCU API" series
- "Cleaning up with guard()", "The many ways to cancel work"
- "Lockless patterns" series (Paolo Bonzini) — ★ an excellent four-part tour of exactly
  these patterns with memory-model rigour
- "Concurrency bugs should fear the big bad data-race detector" (KCSAN)

---

## Part 1 complete

You now have the full kernel-C toolkit: the dialect (08), data structures (09–10), memory
(11), lifetimes (12), the memory model (13), locking (14), RCU (15), per-CPU (16), interrupts
(17), deferred work (18), time (19), processes (20), scheduling (21), memory management
(22–23), userspace interfaces (24), and the patterns that combine them (25).

**Before moving on, complete this checkpoint:**

- [ ] You can state the four-axes checklist (Ch. 00 T.9) for any function on sight.
- [ ] You can choose an allocator, a lock, and a lifetime strategy from first principles.
- [ ] You can write and run a litmus test.
- [ ] You can read a KASAN, lockdep, RCU-stall, and OOM report fluently.
- [ ] You can write the "Locking and lifetime model" comment for a driver you did not write.
- [ ] Your QEMU edit→boot→test loop is under 30 seconds.

→ Next: **Part 2 — Device Drivers**, starting with
[../part2-drivers/26-driver-model.md](../part2-drivers/26-driver-model.md)

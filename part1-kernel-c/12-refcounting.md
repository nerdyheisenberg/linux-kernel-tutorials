# Chapter 12 — Reference Counting, Lifetimes, and Object Ownership

> **Goal:** never write a use-after-free again. This chapter is the C-language precursor to
> everything Rust enforces at compile time (Part 5). Master it and Rust will feel obvious.

---

## Theory & First Principles

> **How to read this section.** T.0 is the bug, shown concretely, before any theory. T.1–T.4
> are why it is genuinely hard and what refcounting does and does not guarantee. T.5–T.9 are
> the APIs, the patterns, and the failure modes.

---

### T.0 — Start here: the bug that refcounting exists to prevent

Two CPUs. One object. No exotic hardware, no compiler cleverness — just two threads.

```c
/* A lookup table of devices, protected by a lock. */
struct device *lookup(int id)
{
	struct device *d;
	spin_lock(&dev_lock);
	d = find_in_table(id);
	spin_unlock(&dev_lock);      /* ← the lock is released HERE */
	return d;
}

/* CPU 0                             CPU 1                                   */
/* ------------------------------    ------------------------------------- */
d = lookup(7);                  /*                                          */
                                /*  spin_lock(&dev_lock);                   */
                                /*  remove_from_table(7);                   */
                                /*  spin_unlock(&dev_lock);                 */
                                /*  kfree(d);           ← FREED             */
d->ops->start(d);               /*  USE AFTER FREE                          */
```

**Every line here is "correct."** The lock was held for the lookup. The lock was held for the
removal. Neither critical section is wrong. The bug is that **the lock protected the *table*,
not the *object*** — and `d`'s lifetime extends past the critical section that found it.

This is Ch. 00 §T.9's "locks protect data, not code" in its most expensive form, and it is
the shape of an enormous fraction of real kernel CVEs. Search for it:

```bash
cd ~/src/linux
git log --oneline --grep='use-after-free' --since=2.years | wc -l
git log --oneline --grep='refcount' --grep='UAF' --all-match --since=2.years | head
```

**Three fixes that do not work**, and understanding why each fails is most of the chapter:

| Attempt | Why it fails |
|---|---|
| **Hold the lock longer** — until the caller is done | The caller may sleep, may take other locks, may return to userspace. You have converted a UAF into a deadlock and a scalability disaster |
| **Check `if (still_in_table(d))` before use** | TOCTOU. It can be removed between the check and the use |
| **Never free anything** | Works, and is genuinely used (RCU-protected caches that only grow). Does not generalize |

**The fix that does work** is to make the object itself track how many references exist:

```c
struct device *lookup(int id)
{
	struct device *d;
	spin_lock(&dev_lock);
	d = find_in_table(id);
	if (d && !refcount_inc_not_zero(&d->ref))   /* ← acquire UNDER the lock */
		d = NULL;                            /*    and handle "already dying" */
	spin_unlock(&dev_lock);
	return d;
}

/* caller: */
d->ops->start(d);
put_device(d);        /* refcount_dec_and_test -> free at zero */
```

**Two details in those four lines carry the whole chapter:**

1. **The increment happens *under the same lock* that protects the table.** Take it after
   unlocking and you are back where you started. This atomicity between "I found it" and "I
   claimed it" is the *lookup–acquire race*, §T.3, and it is the single most common place
   people get this wrong.
2. **`refcount_inc_not_zero`, not `refcount_inc`.** A count of zero means "the last reference
   is gone and destruction is in progress." Resurrecting it is a use-after-free with extra
   steps. The `_not_zero` variant returns false and the lookup reports "not found" — which is
   correct, because it *is* going away.

**The question to hold through this chapter:** what if you want the lookup to take *no lock
at all*, because it is on a fast path and lock contention is the bottleneck? That is the
read-mostly problem, and the answer — RCU — is Ch. 15. This chapter is the foundation it
builds on: **RCU tells you when it is safe to free; refcounting tells you when it is safe to
stop using.** You need both.

---

### T.1 The problem, stated formally

An object `O` is **reachable** if some execution can still dereference a pointer to it.
Memory may be reclaimed only when `O` is unreachable *for all future executions*. In a
single-threaded program with a static call graph, a compiler can often prove this. In a
preemptible, SMP, interrupt-driven kernel with pointers stashed in lists, trees, hardware
descriptors, and userspace file descriptors, **reachability is not statically decidable**.

So you need a *runtime* discipline. There are exactly four known families:

| Family | Mechanism | Cost | Used in Linux? |
|---|---|---|---|
| **Static/lexical** | ownership ends with scope | free | yes (stack, `devm_`) |
| **Tracing GC** | periodically compute reachability from roots | pauses, needs pointer map | **no** — C has no pointer map, and pauses are unacceptable |
| **Reference counting** | maintain in-degree; free at 0 | atomic RMW per get/put; cycles leak | yes, pervasively |
| **Deferred / epoch reclamation** | prove all pre-existing readers finished, then free | near-zero read cost | yes — RCU |

Linux is built almost entirely on the last two, **composed together**. Understanding *why
they must be composed* is the central insight of this chapter.

### T.2 Reference counting: what it actually guarantees, and what it doesn't

A reference count is a **conservative under-approximation of reachability**: it counts
*registered* pointers, not all pointers. Its correctness rests on an invariant you must
maintain by hand:

> **Invariant R:** every pointer that may be dereferenced in the future is accounted for by
> exactly one increment, and each such increment is matched by exactly one decrement after
> the last dereference through that pointer.

Two failure modes follow directly:

- **Under-counting** → premature free → **use-after-free** (exploitable).
- **Over-counting** → never freed → **leak** (a DoS, and on embedded, fatal).

And one structural limitation: **reference counting cannot collect cycles.** If `A` holds a
ref to `B` and `B` to `A`, both counts stay at 1 forever. Tracing GCs handle this; refcounting
cannot, by construction. The kernel's answer is *architectural*: object graphs are designed as
**DAGs with a designated parent direction**, and back-pointers are made **non-owning** (raw,
weak) with the child guaranteed to outlive-or-notify. When you see a `struct device *dev;`
field inside a driver's private struct that does *not* take `get_device()`, that is a
deliberate weak back-edge, valid only because the driver is unbound before the device dies.

*Recognizing owning vs. non-owning pointers by reading a struct definition is a
senior-level reading skill. C gives you no syntax for it; you infer it from the teardown path.*

### T.3 The fundamental race: lookup–acquire

Every refcounting system faces one irreducible race:

```
Thread A (reader)                      Thread B (last owner)
  p = lookup(table, key)   ───┐
                              │        put(p): count 1 → 0
                              │        remove(table, key)
                              │        free(p)
  get(p)   ← DEREFERENCES  ───┘        ← p is already freed
```

The window between *obtaining the pointer* and *taking the reference* is where use-after-free
lives. There are exactly three sound solutions, and all real kernel code uses one of them:

1. **Mutual exclusion over the whole window.** Hold the table lock across lookup *and*
   `get()`. Correct, simple, and serializes readers — a scalability bottleneck.

2. **Make the acquire fallible and atomic.** `refcount_inc_not_zero()` is a CAS loop that
   *fails* if the count already reached zero. Zero is now a permanent "tombstone" state
   meaning *"destruction has begun; you may not resurrect me."* This is why
   `refcount_t` treats inc-from-zero as a bug rather than a normal operation — it is not a
   style choice, it is what makes the tombstone protocol sound.

3. **Guarantee the memory is still readable.** Even a *failed* `refcount_inc_not_zero()` must
   *read* the counter, so the memory must not be unmapped or recycled to a different type.
   That guarantee is exactly what an SMR (safe memory reclamation) scheme provides — in
   Linux, RCU.

**Therefore:** lockless lookup requires *both* an atomic fallible acquire *and* a deferred
reclamation scheme. Neither alone is sufficient. That is the composition claim from T.1, and
it is the single most important idea in this chapter:

```
    rcu_read_lock()          ← guarantees the MEMORY is still valid
      p = lookup(...)
      ok = refcount_inc_not_zero(&p->ref)   ← guarantees the OBJECT is still alive
    rcu_read_unlock()
```

### T.4 Safe memory reclamation: the wider literature

The lookup–acquire race is an instance of the general SMR problem in lock-free programming.
Three canonical solutions exist; Linux uses one and you should know all three, because
maintainers will reference them in review:

| Scheme | Idea | Reader cost | Reclamation latency | Bounded memory? |
|---|---|---|---|---|
| **Hazard pointers** (Michael, 2004) | reader publishes the pointer it is using; reclaimer scans all hazard slots before freeing | a store + memory barrier per access | short | **yes** |
| **Epoch-based** (Fraser, 2004) | global epoch counter; readers pin an epoch; free once all readers advanced | a store per critical section | medium | no (one stuck reader blocks all) |
| **RCU** (McKenney, 1998→) | readers are *unannotated*; grace period = "every CPU has passed a quiescent state" | **zero** in `CONFIG_PREEMPT_NONE` | long (ms) | no |

Linux chose RCU because **read-side cost dominates** in kernel workloads: routing tables,
dcache, and module lists are read millions of times per write. RCU's trade is explicit and
principled: *pay unbounded memory and long reclamation latency to make readers free.*

The consequence you live with daily: **an object freed by RCU is not freed when you drop the
last reference.** It is freed "later". So anything that must happen *synchronously* at the
last put (unpublishing from a table, releasing hardware, signalling a completion) must be
done in the release callback, and only `kfree()` may be deferred.

### T.5 Type-stability and the ABA problem

The ABA problem: a reader reads pointer `P`, is preempted; the object is freed, the memory
reused for a *different* object, and a new pointer with the *same bit pattern* is installed.
The reader resumes, compares pointers, sees "unchanged", and proceeds on the wrong object.

Greenwald & Cheriton's notion of **type-stable memory** (1996) is the fix Linux adopted as
`SLAB_TYPESAFE_BY_RCU`. Its guarantee is deliberately weaker — and cheaper — than full RCU
deferral:

> The *memory* will never be returned to the page allocator or repurposed for a different
> type within an RCU read-side critical section, **but an individual object may be freed and
> immediately reallocated as another object of the same type.**

Why is that useful? Because it makes *dereferencing always memory-safe* (the fields exist,
the locks are initialized, the refcount field is a valid `refcount_t`), which is precisely
what you need to *attempt* `refcount_inc_not_zero()`. You then re-validate identity:

```
p = lookup();                     /* memory guaranteed valid & correctly typed */
if (!refcount_inc_not_zero(&p->ref)) retry;
if (p->key != key) { put(p); retry; }   /* ← ABA check: it got recycled */
```

This is how `struct sock` achieves lockless lookup at tens of millions of packets/sec.
It is also why `SLAB_TYPESAFE_BY_RCU` caches must **never** have their identity fields
zeroed on free, and why their constructors matter. Getting this wrong is a subtle,
exploitable class of bug — use it only when profiling proves the need.

### T.6 Why `atomic_t` was the wrong type: overflow as a security primitive

`atomic_inc()` on a 32-bit counter wraps silently: `INT_MAX → INT_MIN`, and continued
incrementing reaches 0. An attacker who can leak a reference in a loop (e.g., a missing
`put()` in an error path reachable from an ioctl) can drive the count to zero **while
references still exist**, then trigger a free and get a UAF on a still-referenced object.
This was a real, repeatedly exploited pattern (CVE-2014-2851, CVE-2016-0728, and others).

`refcount_t` makes the counter **saturating**: on overflow it pins at `UINT_MAX` and warns;
it never wraps. The object then leaks instead of being freed early. That is the right trade:
**a leak is a bug; a premature free is a privilege escalation.** This is a general security
principle worth internalizing — *when a safety property cannot be maintained, fail toward the
non-exploitable state.*

`refcount_t` also encodes a **state machine** that `atomic_t` does not:

```
       [1..N] ──dec──▶ [1..N-1]
          │
          │ dec_and_test → 0
          ▼
        [ 0 ]  ── DEAD. inc() is a BUG. inc_not_zero() must fail.
```

Only `refcount_inc_not_zero()` may *observe* 0; nothing may *leave* 0.

### T.7 Memory ordering: why the destructor sees your writes

This is the part people hand-wave. Consider:

```
CPU 0                              CPU 1
obj->data = 42;                    /* holds another ref */
put(obj);      /* 2 → 1 */         put(obj);  /* 1 → 0 → runs destructor */
                                   read obj->data;  /* must see 42 */
```

For the destructor to safely observe *all* writes made by *every* prior reference holder,
the refcount must provide:

- **Release** semantics on the decrement — all prior writes by this CPU become visible to
  whoever observes the decremented value.
- **Acquire** semantics on the 1→0 transition — the CPU that wins the race and runs the
  destructor sees all writes released by everyone else.

`refcount_dec_and_test()` implements exactly `RELEASE` + `ACQUIRE`-on-zero. This is the
standard "release/acquire handoff" pattern, and it is the same reasoning behind
`std::shared_ptr`'s `memory_order_release` / `acquire` fence in C++.

Corollary: **you do not need — and must not add — an `smp_mb()` around refcount ops.**
Adding one signals you didn't understand this. See
`Documentation/core-api/refcount-vs-atomic.rst` for the full ordering table; a maintainer
*will* check this in review.

### T.8 Amortizing the atomic: per-CPU reference counting

An atomic RMW on a shared cache line costs ~20–100 ns under contention because the line must
be acquired exclusively (MESI/MOESI). At scale, a single global refcount is a serialization
point — Amdahl's law applied to a cache line.

`percpu_ref` exploits an asymmetry: **the sum only needs to be exact when you are asking
"is it zero?"**, which happens once, at teardown. So:

- **Live phase:** each CPU increments its own counter. No sharing, no atomics contended.
  The true count is `Σ per-cpu counters + bias`, which nobody computes.
- **Kill:** switch to atomic mode via RCU — wait for a grace period so no CPU is mid-flight
  in the per-CPU path, then collapse all per-CPU counters into one atomic and remove the
  bias. Subsequent `tryget` fails.
- **Drain:** the atomic reaching zero fires the release callback.

This is a **mode-switching** algorithm, and the RCU grace period is what makes the switch
safe. Note the pattern: *RCU is used here not to free memory but to synchronize a protocol
transition.* That generalization — RCU as "wait until all CPUs have left the old regime" — is
what lets you apply it far beyond list traversal.

### T.9 Ownership without a type system — and what Rust adds

Everything above is a set of **invariants maintained by convention and review**. C can express
none of them: it has no way to say "this pointer is owning", "this pointer is borrowed for the
duration of this RCU critical section", or "this function consumes the reference".

Rust's ownership model encodes precisely these invariants in the type system:

| Kernel C convention | Rust type-system encoding |
|---|---|
| "you own this; you must `put()`" | `Arc<T>` (move semantics, `Drop`) |
| "borrowed, valid while caller holds a ref" | `&T` with a lifetime `'a` |
| "always-refcounted foreign object" | `ARef<T>` / `AlwaysRefCounted` |
| "valid only inside `rcu_read_lock()`" | a `Guard` with a lifetime tying the borrow to the guard |
| "may be null / may be dying" | `Option<ARef<T>>` from a fallible `try_get` |
| "cycles are impossible here" | `Weak<T>` for back-edges |

This is why Part 5 will feel like a *notation* for what you already do, rather than a new
paradigm — **provided you internalize this chapter first.** Engineers who skip straight to
Rust never understand why `ARef` and `Pin` exist.

### T.10 Choosing a strategy: a decision procedure

```
Does the object's lifetime end inside one function?
  └─ yes → scoped: alloc + free, goto error ladder.
Does it die exactly when a parent dies, with no other holders?
  └─ yes → parent-owned: devm_*, or embed it in the parent struct.
Can more than one subsystem hold a pointer, with no lock covering all uses?
  └─ yes → refcount (kref / refcount_t).
        Do readers look it up from a shared container without a lock?
          └─ yes → add RCU: publish with rcu_assign_pointer, look up under
                   rcu_read_lock, acquire with *_inc_not_zero, free with kfree_rcu.
              Is the object recycled so fast that a grace period per free is too costly?
                └─ yes → SLAB_TYPESAFE_BY_RCU + explicit identity re-check.
        Is get/put itself a measured hotspot?
          └─ yes → percpu_ref.
Must the last put run in atomic context, but the destructor sleeps?
  └─ defer the destructor to a workqueue; keep only the unpublish in the release fn.
```

Run this procedure explicitly for every new object you add. Writing the answer in the commit
message is what distinguishes a patch that gets merged from one that gets three rounds of
review.

---

## 1. Concept — the hardest problem in the kernel

In userspace, lifetime bugs crash your process. In the kernel they are **CVEs**.
Roughly 40–50% of exploitable Linux kernel vulnerabilities in the last decade were
use-after-free or double-free.

Every kernel object needs an answer to four questions:

1. **Who owns it?** (who is responsible for freeing)
2. **Who can reach it?** (which lists/trees/pointers hold references)
3. **When is the last reference dropped?** (and is that context allowed to sleep?)
4. **How do readers know it's still alive?** (lock, refcount, or RCU grace period)

There are exactly four lifetime strategies in the kernel. Learn to name them.

| Strategy | Mechanism | Example |
|---|---|---|
| **Scoped** | allocate + free in the same function/scope | temp buffers |
| **Parent-owned** | freed when parent dies | `devm_kmalloc()` (Ch. 28) |
| **Refcounted** | `refcount_t` / `kref` / `get_*`/`put_*` | `struct file`, `struct dentry`, `struct device` |
| **RCU-deferred** | freed after a grace period | routing table entries, `struct task_struct` pieces |

Most real objects use **refcount + RCU together**.

---

## 2. Internals

### 2.1 `atomic_t` is *not* a refcount

Historically everyone used `atomic_t`. This was a security disaster: `atomic_inc()` wraps
silently at `INT_MAX`, letting an attacker overflow a counter to zero and trigger a free.

Since 4.11 the kernel has `refcount_t` (`include/linux/refcount.h`), which is
**saturating**: overflow and use-after-free transitions are detected and WARN.

```c
refcount_t r;
refcount_set(&r, 1);

refcount_inc(&r);            /* WARNs if r was 0 — inc-from-zero is a UAF signature */
bool last = refcount_dec_and_test(&r);   /* true if it hit 0 */
bool got  = refcount_inc_not_zero(&r);   /* the ONLY safe way to grab a weak ref */
```

Semantics you must memorize:

- `refcount_inc()` on 0 → **bug**, because 0 means "being destroyed".
- `refcount_inc_not_zero()` → the lookup-then-grab primitive. Returns false if the object
  is already dying. **This is the heart of every RCU lookup.**
- `refcount_dec_and_test()` → returns true exactly once, for exactly one caller.
- `refcount_dec_and_lock(&r, &lock)` → atomically: if dec reaches 0, take `lock` and return
  true. Solves the classic race between "drop last ref" and "someone looks me up in a list".

**Memory ordering:** `refcount_dec_and_test()` has release semantics on the decrement and
acquire semantics on the zero transition. That means all your writes before the `put()` are
visible to the thread that runs the destructor. You get this for free; don't add barriers.

### 2.2 `kref` — refcount + destructor, the canonical pattern

```c
#include <linux/kref.h>

struct my_obj {
	struct kref     kref;
	struct list_head list;
	spinlock_t      lock;
	int             data;
};

static void my_obj_release(struct kref *kref)
{
	struct my_obj *o = container_of(kref, struct my_obj, kref);

	/* Called with refcount == 0. Nobody else can reach us... if we did it right. */
	kfree(o);
}

static struct my_obj *my_obj_alloc(void)
{
	struct my_obj *o = kzalloc(sizeof(*o), GFP_KERNEL);

	if (!o)
		return NULL;
	kref_init(&o->kref);        /* refcount = 1, owned by the caller */
	spin_lock_init(&o->lock);
	return o;
}

static struct my_obj *my_obj_get(struct my_obj *o)
{
	kref_get(&o->kref);
	return o;
}

static void my_obj_put(struct my_obj *o)
{
	kref_put(&o->kref, my_obj_release);
}
```

Rules from `Documentation/core-api/kref.rst`:

1. If you have a pointer you did not create, you must have taken a reference, **or** hold
   a lock/RCU that guarantees the owner can't drop theirs.
2. `kref_get()` requires you *already* hold a valid reference.
3. Getting a reference from a shared structure requires the structure's lock, or
   `kref_get_unless_zero()` under RCU.
4. The release function must not sleep unless every `put()` caller can sleep.

### 2.3 The lookup race, and the three correct solutions

The bug everyone writes once:

```c
/* BROKEN */
struct my_obj *lookup(int id)
{
	struct my_obj *o;

	spin_lock(&table_lock);
	o = find_in_table(id);
	spin_unlock(&table_lock);
	kref_get(&o->kref);      /* ← object may have been freed between unlock and here */
	return o;
}
```

**Solution A — take the ref under the lock:**

```c
struct my_obj *lookup(int id)
{
	struct my_obj *o;

	spin_lock(&table_lock);
	o = find_in_table(id);
	if (o)
		kref_get(&o->kref);
	spin_unlock(&table_lock);
	return o;
}

void my_obj_put(struct my_obj *o)
{
	/* the matching side: remove from table atomically with the last put */
	if (refcount_dec_and_lock(&o->kref.refcount, &table_lock)) {
		list_del(&o->list);
		spin_unlock(&table_lock);
		kfree(o);
	}
}
```

**Solution B — RCU + `kref_get_unless_zero()`** (scales, readers never block):

```c
struct my_obj *lookup_rcu(int id)
{
	struct my_obj *o;

	rcu_read_lock();
	o = radix_lookup(id);                     /* or xa_load() */
	if (o && !kref_get_unless_zero(&o->kref))
		o = NULL;                          /* it's dying; pretend not found */
	rcu_read_unlock();
	return o;
}

static void my_obj_release(struct kref *kref)
{
	struct my_obj *o = container_of(kref, struct my_obj, kref);

	spin_lock(&table_lock);
	xa_erase(&table, o->id);                   /* unpublish */
	spin_unlock(&table_lock);
	kfree_rcu(o, rcu);                         /* free after grace period */
}
```

This requires `struct rcu_head rcu;` in the object and that the object be allocated from a
`SLAB_TYPESAFE_BY_RCU` cache *or* freed with `kfree_rcu()`. See Ch. 15.

**Solution C — never hand out raw pointers.** Use a handle/ID (`xa_alloc()`), and resolve it
under the lock each time. Slower, but bulletproof. This is what fd tables effectively do.

### 2.4 Weak references, `SLAB_TYPESAFE_BY_RCU`, and the type-stability trick

`SLAB_TYPESAFE_BY_RCU` (formerly `SLAB_DESTROY_BY_RCU`) guarantees that a *slab page* is not
returned to the page allocator until a grace period passes — but an individual object **can
be reused immediately** for another object *of the same type*.

Consequence: under `rcu_read_lock()`, the memory is always valid and always a `struct
my_obj`, but it may be a *different* one. So you must re-validate after taking the ref:

```c
rcu_read_lock();
o = lookup(id);
if (o && kref_get_unless_zero(&o->kref)) {
	if (o->id != id) {       /* ← re-check identity: it got recycled */
		my_obj_put(o);
		o = NULL;
	}
}
rcu_read_unlock();
```

This is how `struct sock` and `struct file` (via `files_lookup_fd_rcu`) achieve lockless
lookup at millions of ops/sec. It is also a rich source of subtle bugs — use only when
profiling proves you need it.

### 2.5 `percpu_ref` — for objects with extremely hot get/put

A plain `refcount_t` on a shared object is a cacheline ping-pong. `percpu_ref`
(`include/linux/percpu-refcount.h`) starts in **per-CPU mode** (increments are per-CPU,
essentially free) and switches to **atomic mode** when you call `percpu_ref_kill()`.

```c
struct percpu_ref ref;

percpu_ref_init(&ref, my_release, 0, GFP_KERNEL);
percpu_ref_get(&ref);                    /* per-CPU increment, ~1 ns */
percpu_ref_put(&ref);
...
percpu_ref_kill(&ref);                   /* switch to atomic, no new tryget succeeds */
wait_for_completion(&done);              /* release() fires when it hits 0 */
percpu_ref_exit(&ref);
```

Used by: `blk-mq` queue usage counters, cgroups, io_uring, DAX. If a refcount shows up in a
profile, this is the answer.

### 2.6 `struct device` refcounting and `devres`

`struct device` embeds a `kobject`, which embeds a `struct kref`.

```c
get_device(dev);   /* kobject_get */
put_device(dev);   /* kobject_put → device_release → bus->release or type->release */
```

**Critical rule:** a `struct device` must have a `release()` callback. Freeing a device with
`kfree()` directly is a classic driver bug the kernel WARNs about loudly:
`"Device 'xyz' does not have a release() function, it is broken and must be fixed."`

`devm_*` APIs (Ch. 28) attach allocations to the device's `devres` list, freed in reverse
order at driver detach. They solve *parent-owned* lifetime, **not** refcounted lifetime —
a common mistake is using `devm_kzalloc()` for an object that outlives the driver binding
(e.g., something userspace holds an open fd to). That is a UAF waiting to happen.

### 2.7 Module lifetime

```c
try_module_get(THIS_MODULE);   /* prevents rmmod while you hold it */
module_put(THIS_MODULE);
```
File operations already do this via `.owner = THIS_MODULE`. Almost the only time you call
these manually is when you hand a callback to another subsystem that will outlive your call.

---

## 3. Practice

### 3.1 Refcounted object with RCU lookup — full working module

```c
// SPDX-License-Identifier: GPL-2.0
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/slab.h>
#include <linux/xarray.h>
#include <linux/kref.h>
#include <linux/rcupdate.h>

struct node {
	u32              id;
	struct kref      kref;
	struct rcu_head  rcu;
	char             payload[32];
};

static DEFINE_XARRAY_ALLOC(nodes);
static DEFINE_SPINLOCK(nodes_lock);

static void node_free_rcu(struct rcu_head *head)
{
	struct node *n = container_of(head, struct node, rcu);

	pr_info("freeing node %u after grace period\n", n->id);
	kfree(n);
}

static void node_release(struct kref *kref)
{
	struct node *n = container_of(kref, struct node, kref);

	spin_lock(&nodes_lock);
	xa_erase(&nodes, n->id);
	spin_unlock(&nodes_lock);

	call_rcu(&n->rcu, node_free_rcu);
}

static void node_put(struct node *n) { kref_put(&n->kref, node_release); }

static struct node *node_create(const char *name)
{
	struct node *n = kzalloc(sizeof(*n), GFP_KERNEL);
	u32 id;
	int err;

	if (!n)
		return ERR_PTR(-ENOMEM);

	kref_init(&n->kref);
	strscpy(n->payload, name, sizeof(n->payload));

	err = xa_alloc(&nodes, &id, n, xa_limit_32b, GFP_KERNEL);
	if (err) {
		kfree(n);
		return ERR_PTR(err);
	}
	n->id = id;
	return n;
}

/* Lockless lookup: returns a node with a reference held, or NULL. */
static struct node *node_lookup(u32 id)
{
	struct node *n;

	rcu_read_lock();
	n = xa_load(&nodes, id);
	if (n && !kref_get_unless_zero(&n->kref))
		n = NULL;
	rcu_read_unlock();
	return n;
}

static int __init refdemo_init(void)
{
	struct node *a, *b;

	a = node_create("alpha");
	if (IS_ERR(a))
		return PTR_ERR(a);
	pr_info("created id=%u\n", a->id);

	b = node_lookup(a->id);
	pr_info("lookup -> %s (refs now 2)\n", b ? b->payload : "NULL");
	if (b)
		node_put(b);

	node_put(a);            /* last ref: erase + call_rcu */
	return 0;
}

static void __exit refdemo_exit(void)
{
	rcu_barrier();          /* ★ wait for outstanding call_rcu before module text vanishes */
	xa_destroy(&nodes);
}

module_init(refdemo_init);
module_exit(refdemo_exit);
MODULE_LICENSE("GPL");
```

> **The `rcu_barrier()` in the exit path is not optional.** If `node_free_rcu` is still
> queued when the module text is unmapped, you get a jump into freed memory. Any module
> using `call_rcu()` must `rcu_barrier()` on unload. Same for `flush_workqueue()`,
> `del_timer_sync()`, `cancel_delayed_work_sync()`.

### 3.2 Catch lifetime bugs automatically

```bash
# In your dev kernel config (Ch. 06):
CONFIG_KASAN=y                 # use-after-free / out-of-bounds detector (the big one)
CONFIG_KASAN_VMALLOC=y
CONFIG_KFENCE=y                # low-overhead sampling detector, safe for production
CONFIG_DEBUG_OBJECTS=y
CONFIG_DEBUG_OBJECTS_FREE=y
CONFIG_DEBUG_LIST=y            # list_head corruption checks
CONFIG_DEBUG_KOBJECT_RELEASE=y # delays kobject release to expose UAF
CONFIG_REFCOUNT_FULL=y         # (older kernels; now always-on)
CONFIG_PROVE_RCU=y
CONFIG_SLUB_DEBUG_ON=y         # poisoning, redzones, last-free stack traces
```

A KASAN report tells you three stacks: allocation, free, and bad access. Reading it fluently
is a core skill:

```
BUG: KASAN: slab-use-after-free in node_put+0x12/0x80
Read of size 4 at addr ffff888107a3c008 by task demo/1234
...
Allocated by task 1234:  kmalloc → node_create
Freed by task 1235:      kfree → node_free_rcu
```

### 3.3 Prove the saturation behaviour

```c
refcount_t r;
refcount_set(&r, UINT_MAX - 1);
refcount_inc(&r);
refcount_inc(&r);   /* saturates, WARNs once, never wraps to 0 */
```

---

## 3.4 Extended practice

### Lab 12.A — Reproduce the lookup–acquire race (T.3)

Build the broken version, then each of the three fixes, and prove each works.

```c
// SPDX-License-Identifier: GPL-2.0
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/slab.h>
#include <linux/kthread.h>
#include <linux/xarray.h>
#include <linux/refcount.h>
#include <linux/delay.h>

static int mode = 0;   /* 0=broken 1=lock 2=rcu */
module_param(mode, int, 0644);

struct node { u32 id; refcount_t ref; struct rcu_head rcu; u32 magic; };
#define MAGIC 0x600DF00D

static DEFINE_XARRAY(tbl);
static DEFINE_SPINLOCK(tbl_lock);
static struct task_struct *adder, *remover, *looker;
static atomic_t hits, misses, corrupt;

static void node_free_rcu(struct rcu_head *h)
{
	kfree(container_of(h, struct node, rcu));
}

static void node_put(struct node *n)
{
	if (refcount_dec_and_test(&n->ref)) {
		n->magic = 0xDEADDEAD;
		if (mode == 2)
			call_rcu(&n->rcu, node_free_rcu);
		else
			kfree(n);
	}
}

static struct node *lookup(u32 id)
{
	struct node *n = NULL;

	switch (mode) {
	case 0:                                  /* BROKEN: window between load and get */
		n = xa_load(&tbl, id);
		if (n) {
			cpu_relax();                 /* widen the window */
			refcount_inc(&n->ref);       /* ← may inc a freed object */
		}
		break;
	case 1:                                  /* Solution A: lock covers the window */
		spin_lock(&tbl_lock);
		n = xa_load(&tbl, id);
		if (n)
			refcount_inc(&n->ref);
		spin_unlock(&tbl_lock);
		break;
	case 2:                                  /* Solution B: RCU + inc_not_zero */
		rcu_read_lock();
		n = xa_load(&tbl, id);
		if (n && !refcount_inc_not_zero(&n->ref))
			n = NULL;
		rcu_read_unlock();
		break;
	}
	return n;
}

static int look_fn(void *unused)
{
	while (!kthread_should_stop()) {
		struct node *n = lookup(1);

		if (!n) { atomic_inc(&misses); continue; }
		if (READ_ONCE(n->magic) != MAGIC)
			atomic_inc(&corrupt);        /* ← we touched a dead object */
		else
			atomic_inc(&hits);
		node_put(n);
		cond_resched();
	}
	return 0;
}
/* adder/remover kthreads: repeatedly xa_store()/xa_erase()+node_put() id 1 */
```
```bash
for m in 0 1 2; do
  sudo insmod race.ko mode=$m; sleep 5; sudo rmmod race
  echo "mode=$m"; dmesg | tail -2
done
```
Build with **KASAN** (Ch. 06). `mode=0` will produce `slab-use-after-free` within seconds;
modes 1 and 2 will not. Note the `corrupt` counter: it detects the infection (Ch. 06 T.1)
even when KASAN is off.

### Lab 12.B — Watch `refcount_t` saturate instead of wrapping (T.6)

```c
static int __init sat_init(void)
{
	refcount_t r;
	atomic_t a;
	int i;

	/* atomic_t: wraps silently → 0 → a free while references exist */
	atomic_set(&a, INT_MAX);
	atomic_inc(&a);
	pr_info("atomic_t  INT_MAX+1 = %d  (wrapped!)\n", atomic_read(&a));

	/* refcount_t: saturates and warns ONCE */
	refcount_set(&r, UINT_MAX - 2);
	for (i = 0; i < 5; i++)
		refcount_inc(&r);
	pr_info("refcount_t saturated at %u (UINT_MAX=%u)\n",
		refcount_read(&r), UINT_MAX);

	/* inc-from-zero is a BUG: it means resurrecting a dying object */
	refcount_set(&r, 0);
	refcount_inc(&r);                 /* → WARN: refcount_t: addition on 0 */
	pr_info("after inc-from-zero: %u\n", refcount_read(&r));
	return 0;
}
```
Read the resulting `REFCOUNT_WARN` splat carefully. Then find the real CVEs:
```bash
git log --oneline --grep='refcount' --grep='overflow' --all-match | head -20
git log --oneline -S'refcount_t' -- drivers/ | head -20   # the conversion campaign
```

### Lab 12.C — `SLAB_TYPESAFE_BY_RCU` and the ABA re-check (T.5)

```c
static struct kmem_cache *tsc;

static void ctor(void *p)
{
	struct node *n = p;

	refcount_set(&n->ref, 0);     /* constructor runs once per object, not per alloc */
}

/* create with SLAB_TYPESAFE_BY_RCU */
tsc = kmem_cache_create("tsc_node", sizeof(struct node), 0,
			SLAB_TYPESAFE_BY_RCU | SLAB_HWCACHE_ALIGN, ctor);

/* Reader MUST re-validate identity after acquiring the reference: */
rcu_read_lock();
n = xa_load(&tbl, id);
if (n && refcount_inc_not_zero(&n->ref)) {
	if (READ_ONCE(n->id) != id) {      /* ← recycled into a DIFFERENT node */
		atomic_inc(&aba_detected);
		node_put(n);
		n = NULL;
	}
}
rcu_read_unlock();
```
Run with a high alloc/free churn rate on many CPUs and watch `aba_detected` become nonzero.
**Then delete the re-check and watch the wrong object get used.** This is the single best way
to understand why `SLAB_TYPESAFE_BY_RCU` is dangerous. Then read the real user:
```bash
git grep -n 'SLAB_TYPESAFE_BY_RCU' | head
$EDITOR net/core/sock.c   # sk_prot_alloc / proto_register, and __inet_lookup_established
```

### Lab 12.D — `percpu_ref` vs `refcount_t` under contention (T.8)

```c
#include <linux/percpu-refcount.h>

static struct percpu_ref pref;
static refcount_t plain;
static DECLARE_COMPLETION(done);

static void pref_release(struct percpu_ref *r) { complete(&done); }

static int hammer(void *arg)
{
	bool use_percpu = (long)arg;
	u64 t0 = ktime_get_ns();
	int i;

	for (i = 0; i < 1000000; i++) {
		if (use_percpu) { percpu_ref_get(&pref); percpu_ref_put(&pref); }
		else            { refcount_inc(&plain); refcount_dec(&plain); }
	}
	pr_info("cpu%d %s: %llu ns/op\n", smp_processor_id(),
		use_percpu ? "percpu_ref" : "refcount_t",
		(ktime_get_ns() - t0) / 1000000);
	return 0;
}
/* launch one kthread per CPU for each variant; then: */
percpu_ref_kill(&pref);
wait_for_completion(&done);
percpu_ref_exit(&pref);
```
Run on 1, 2, 4, and all CPUs. `refcount_t` should degrade from ~10 ns to 200+ ns/op;
`percpu_ref` should stay flat at ~1–2 ns. **Plot it.** Then observe the mode switch:
```bash
sudo bpftrace -e 'kprobe:percpu_ref_switch_to_atomic_rcu { printf("mode switch\n"); }'
```

### Lab 12.E — `devm_` is the wrong tool for a refcounted object

Write two versions of a misc-device driver whose private data is reachable from an open fd:

```c
/* WRONG: devm_kzalloc frees at driver detach, while userspace still holds an fd */
priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);

/* RIGHT: kref; the driver holds one ref, each open() holds one */
priv = kzalloc(sizeof(*priv), GFP_KERNEL);
kref_init(&priv->ref);
...
static int my_open(struct inode *i, struct file *f)
{
	struct priv *p = container_of(i->i_cdev, struct priv, cdev);

	if (!kref_get_unless_zero(&p->ref))
		return -ENODEV;          /* driver already removed */
	f->private_data = p;
	return 0;
}
static int my_release(struct inode *i, struct file *f)
{
	kref_put(&((struct priv *)f->private_data)->ref, priv_release);
	return 0;
}
/* remove(): drop the driver's reference; last fd close frees it */
```
```bash
# Reproduce the UAF in the WRONG version:
sudo insmod mydrv.ko
exec 3</dev/mydev              # hold an fd open
sudo rmmod mydrv               # devm_ frees priv HERE
dd if=/dev/fd/3 bs=1 count=1   # → KASAN slab-use-after-free
exec 3<&-
```
This is a real, recurring driver bug class. Fixing one upstream is a genuine contribution.

### Lab 12.F — Read a real refcount design end to end

```bash
# struct file — the most performance-critical refcount in the kernel
$EDITOR fs/file_table.c fs/file.c include/linux/file_ref.h
git log --oneline -- include/linux/file_ref.h | head
# Questions to answer in writing:
#  1. How does fget() avoid a UAF against a concurrent close()?
#  2. What does files_lookup_fd_rcu() guarantee, and what does it NOT?
#  3. Why was f_count converted from atomic_long_t to file_ref_t in 6.13?
#  4. What is the "dead" state in file_ref_t and how does it differ from 0?

# struct device — the refcount you will touch most often as a driver writer
$EDITOR drivers/base/core.c   # device_release(), put_device(), the "no release()" WARN
git grep -n 'does not have a release' drivers/base/core.c
```

---

## 4. Mastery drills

1. **Audit a real driver.** Pick any driver in `drivers/char/`. Draw its object graph:
   every allocation, who owns it, how it's freed. Find at least one object whose lifetime
   isn't obvious and explain it.

2. **`struct file` lifetime.** Read `fs/file_table.c` and `fs/file.c`. Explain:
   how does `fget()` avoid a UAF against a concurrent `close()`? What role does
   `files_lookup_fd_rcu()` + `get_file_rcu()` play? Why did `f_count` become
   `file_ref_t` in 6.13?

3. **Find a real CVE.** `git log --oneline --grep='use-after-free' -- drivers/ | head -50`.
   Pick one, read the fix, and explain the exact interleaving that triggered it.

4. **percpu_ref study.** Read `block/blk-core.c`'s `q_usage_counter`. Why can't it be a
   plain `refcount_t`? What does `blk_freeze_queue()` do and why does it need the
   atomic-mode switch?

5. **Write the bug, then catch it.** Deliberately remove the `rcu_barrier()` from §3.1,
   build with KASAN, and `rmmod` in a loop until it explodes. Read the splat.

6. **Design question:** you're adding an object that userspace can hold an fd to, but which
   is created by a platform driver. Where does it live? `devm_kzalloc()` or `kref`?
   Justify. (Answer: `kref`, with the driver holding one reference it drops at `remove()`,
   and the fd holding another. `devm_` would free it while userspace still has the fd.)

---

## 5. Further reading

- `Documentation/core-api/kref.rst` — short, mandatory
- `Documentation/core-api/refcount-vs-atomic.rst` — memory-ordering table, mandatory
- `Documentation/RCU/rcuref.rst` — reference counting with RCU, the definitive text
- `include/linux/refcount.h` — read the comments, they are a tutorial
- LWN: "The rest of the refcount_t story", "A new API for reference counting"
- Kees Cook's talks on refcount hardening (KSPP)

→ Next: [13-atomics-memory-model.md](13-atomics-memory-model.md)

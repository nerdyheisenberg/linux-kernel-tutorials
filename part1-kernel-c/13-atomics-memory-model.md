# Chapter 13 — Atomics and the Linux Kernel Memory Model (LKMM)

> **Goal:** be able to reason about, and *formally check*, what one CPU can observe of another
> CPU's writes. This is the deepest theory chapter in Part 1. Everything in locking (Ch. 14),
> RCU (Ch. 15), per-CPU (Ch. 16), and lock-free driver rings (Ch. 34) rests on it.

---

## Theory & First Principles

> **How to read this section.** T.0 shows you a program whose output is impossible — and then
> runs it. T.1–T.5 build the model: coherence versus consistency, litmus tests, barriers.
> T.6–T.11 are the Linux API and the LKMM.

---

### T.0 — Start here: an impossible result, on your machine

Read this program and predict what it can print.

```c
/* Shared, both initially 0. */
int x = 0, y = 0;
int r1, r2;

/* CPU 0            CPU 1        */
x = 1;          /*  y = 1;       */
r1 = y;         /*  r2 = x;      */
```

Enumerate every interleaving of four statements. In **every** one of them, at least one CPU
runs its store before the other's load, so at least one of `r1`, `r2` must be 1:

| Order | r1 | r2 |
|---|---|---|
| x=1, r1=y, y=1, r2=x | 0 | **1** |
| x=1, y=1, r1=y, r2=x | **1** | **1** |
| y=1, r2=x, x=1, r1=y | **1** | 0 |
| …and so on | | |

**`r1 == 0 && r2 == 0` appears in no interleaving whatsoever.** It is not merely unlikely; it
is not in the state space.

Now run it:

```bash
cd ~/src/linux/tools/memory-model
cat > /tmp/SB.litmus <<'EOF'
C SB
{}
P0(int *x, int *y) { WRITE_ONCE(*x, 1); int r1 = READ_ONCE(*y); }
P1(int *x, int *y) { WRITE_ONCE(*y, 1); int r2 = READ_ONCE(*x); }
exists (0:r1=0 /\ 1:r2=0)
EOF
herd7 -conf linux-kernel.cfg /tmp/SB.litmus
#   Positive: 1   <-- the "impossible" outcome IS ALLOWED
```

Or observe it directly on x86 hardware with `litmus7`, where it occurs in roughly 1 in 10⁶
iterations. **It is real, it happens on the machine on your desk, and it happens on x86 — the
*strongest* mainstream memory model.**

**Why.** Every modern CPU has a **store buffer**: a store retires into a small queue and
drains to cache later, so the core can continue. A subsequent load checks the store buffer
for its *own* address, but not for others'. So:

```
 CPU 0                          CPU 1
  store x=1 -> store buffer      store y=1 -> store buffer
  load  y   -> cache: 0          load  x   -> cache: 0
  (drains later)                 (drains later)
```

Both loads see 0. No interleaving of *program* statements produced this, because the
assumption that hardware executes your program as written was false.

**Three things this destroys, and they are the reason this chapter is here:**

1. **Sequential consistency is not what hardware provides.** Lamport's SC — "the result is as
   if all operations executed in *some* sequential order consistent with program order" — is
   a model you *pay* for, not one you get. §T.1.
2. **Testing cannot find these bugs.** This outcome occurs once in a million on one
   microarchitecture and never on another. Your test passes. Your customer's ARM server
   fails. **You must construct correctness, not observe it.** Ch. 25 §T.2.
3. **"It works on x86" means nothing.** x86 is TSO: it reorders only store→load. ARM64,
   POWER, and RISC-V reorder far more aggressively. Code developed on x86 and deployed on
   ARM is the classic way to meet this in production.

**The one pattern to hold onto**, because it is 90% of what you will actually need. The
*message-passing* case:

```c
/* WRONG                          */  /* RIGHT                                */
obj->field = 42;                      obj->field = 42;
WRITE_ONCE(shared_ptr, obj);          smp_store_release(&shared_ptr, obj);

p = READ_ONCE(shared_ptr);            p = smp_load_acquire(&shared_ptr);
if (p) use(p->field);   /* may be    if (p) use(p->field);   /* guaranteed 42 */
                           garbage */
```

The reader can observe the pointer before the field. `smp_store_release` /
`smp_load_acquire` forbid it. **That pair, and the reason for it, is the single most
important thing in this chapter** — it is what `rcu_assign_pointer`/`rcu_dereference`
(Ch. 15), every lock's acquire/release semantics (Ch. 14), and `pin-init`'s publication rule
(Ch. 80) are all instances of.

---

### T.1 The illusion you have been programming against

When you write C, you assume:

1. Statements execute in program order.
2. A write becomes visible to everyone the instant it executes.
3. All CPUs agree on the order in which writes happened.

**All three are false on real hardware**, and the compiler violates them before the hardware
even gets a chance. Assumption (1)–(3) together are called **sequential consistency (SC)**,
formalized by Leslie Lamport in 1979:

> *"the result of any execution is the same as if the operations of all the processors were
> executed in some sequential order, and the operations of each individual processor appear
> in this sequence in the order specified by its program."*

SC is what every programmer intuitively assumes and what **no** production CPU implements,
because it forbids the two optimizations that make CPUs fast:

- **Store buffering.** A store retires into a per-core FIFO buffer and drains to the cache
  hierarchy later. The storing core can read its own store immediately (store-to-load
  forwarding), but other cores cannot see it yet. This alone breaks SC.
- **Out-of-order / speculative loads.** Loads issue as soon as their address is known,
  possibly before earlier loads or stores complete.

### T.2 Coherence ≠ consistency

A distinction people constantly conflate:

- **Cache coherence** is a *per-location* guarantee: all CPUs observe writes *to a single
  memory location* in one consistent total order, and a write eventually becomes visible.
  MESI/MOESI protocols give you this. It is essentially free and always present.
- **Memory consistency** is an *inter-location* guarantee: how writes to *different*
  locations are ordered relative to each other, as seen by other CPUs. This is where all
  the difficulty lives, and it is what memory barriers control.

Coherence means `x` will eventually be 1 everywhere. Consistency is about whether someone
can see `x == 1` but still see the older value of `y` that you wrote *before* `x`.

### T.3 Litmus tests: the vocabulary of the field

Memory-model discussions are conducted in **litmus tests** — tiny multi-threaded programs
with an assertion about which final states are possible. Learn these six names; maintainers
use them like words.

All variables start at 0. `r1`, `r2` are local registers.

**MP — Message Passing.** The most important one; it is what every lock, every RCU
publication, and every ring buffer is doing.

```
CPU0                    CPU1
WRITE_ONCE(data, 42);   r1 = READ_ONCE(flag);
WRITE_ONCE(flag, 1);    r2 = READ_ONCE(data);

Can we observe r1 == 1 && r2 == 0 ?   ← "I saw the flag but not the data"
```
On x86 (TSO): no, stores are not reordered with stores and loads not with loads.
On ARM64/PowerPC/RISC-V: **yes**. This is the classic bug. Fix with
`smp_wmb()`/`smp_rmb()` or, preferably, `smp_store_release()`/`smp_load_acquire()`.

**SB — Store Buffering.** The one that x86 *also* fails.

```
CPU0                    CPU1
WRITE_ONCE(x, 1);       WRITE_ONCE(y, 1);
r1 = READ_ONCE(y);      r2 = READ_ONCE(x);

Can r1 == 0 && r2 == 0 ?
```
**Yes, even on x86.** Both stores sit in store buffers while both loads are satisfied from
cache. This is why x86 needs `mfence`/`lock`-prefixed ops for `smp_mb()` and why "x86 is
strongly ordered so I don't need barriers" is wrong. Dekker/Peterson mutual exclusion and
the `wake_up`/`prepare_to_wait` pairing both depend on fixing SB.

**LB — Load Buffering.**
```
CPU0                    CPU1
r1 = READ_ONCE(x);      r2 = READ_ONCE(y);
WRITE_ONCE(y, 1);       WRITE_ONCE(x, 1);

Can r1 == 1 && r2 == 1 ?   (a "value out of thin air"-adjacent cycle)
```
Forbidden on x86 and, under LKMM, forbidden if there's any dependency or ordering; allowed
on weak hardware without ordering.

**IRIW — Independent Reads of Independent Writes.** Tests *multi-copy atomicity*.
```
CPU0            CPU1            CPU2                    CPU3
x = 1;          y = 1;          r1 = x; r2 = y;         r3 = y; r4 = x;

Can CPU2 see x-then-not-y while CPU3 sees y-then-not-x?
```
If yes, the machine is **non-multi-copy-atomic**: different observers disagree on the global
order of independent stores. Classic PowerPC allows this; ARMv8 is *other*-multi-copy-atomic
(all observers *except* the storer agree), x86 TSO is multi-copy atomic. LKMM requires
`smp_mb()` on the readers to forbid it.

**WRC / ISA2 / R / S** — the remaining standard names, used to probe **cumulativity** and
**transitivity** (below). You'll meet them in `tools/memory-model/litmus-tests/`.

### T.4 Data races and the DRF-SC theorem

Adve & Hill (ISCA 1990) proved the result that underpins every modern memory model:

> **Data-Race-Free ⇒ Sequential Consistency.**
> If a program contains no data races *under SC execution*, then executing it on a weakly
> ordered machine that respects the model's synchronization primitives produces only SC
> behaviours.

This is the bargain: **"you give me correctly-annotated synchronization, I give you back the
simple model."** A *data race* is two accesses to the same location, from different threads,
at least one a write, not ordered by synchronization.

The practical consequence for kernel code:

- If a variable is only ever touched under a lock → no race → you may use plain C accesses
  and reason sequentially. **You do not need barriers inside a critical section.**
- If a variable is touched concurrently without a lock → you are writing *racy* code
  deliberately, and you must use `READ_ONCE()`/`WRITE_ONCE()`/atomics and reason with the
  memory model explicitly. There is no middle ground, and "it's just an int, it'll be fine"
  is not engineering.

In C11 a data race is **undefined behaviour**. In the kernel, marked racy accesses
(`READ_ONCE`/`WRITE_ONCE`/`data_race()`) are *defined* — LKMM gives them semantics that C11
does not. That divergence is intentional and is why the kernel has its own model.

### T.5 The compiler is an adversary

Before any hardware reordering, the *compiler* will wreck unannotated shared-memory code.
A plain `x = foo;` gives the compiler license to:

| Optimization | What breaks |
|---|---|
| **Load tearing** | split a 64-bit load into two 32-bit loads → you observe half of each value |
| **Store tearing** | same on the write side |
| **Load fusing** | hoist a load out of a loop → `while (!flag);` becomes `if (!flag) for(;;);` — an infinite loop |
| **Store fusing** | drop all but the last store → a state machine's intermediate states vanish, so another CPU never sees them |
| **Invented loads** | reload the variable under register pressure → two "reads" of what you thought was one value disagree |
| **Invented stores** | write a value speculatively then correct it → another CPU sees a value that was never legal |
| **Dead-store elimination / value speculation** | `if (x == 1) y = x;` → `y = 1;`, destroying the data dependency you were relying on |

`READ_ONCE()` / `WRITE_ONCE()` (in `include/asm-generic/rwonce.h`) are `volatile` casts with
a compiler barrier. They do **not** emit CPU barriers (except on Alpha for `READ_ONCE`'s
dependency ordering). Their entire job is to forbid the table above — to make the access
happen **exactly once, exactly as written, in full width**.

```c
/* WRONG: compiler may fuse the load, tear it, or invent extra loads */
while (!shared->ready)
    cpu_relax();

/* RIGHT */
while (!READ_ONCE(shared->ready))
    cpu_relax();
```

`READ_ONCE` is also documentation: it announces *"this location is concurrently modified."*
Reviewers read it that way. `data_race()` is the sibling annotation meaning *"this race is
intentional and benign; KCSAN, don't report it"* — for statistics counters and heuristics.

### T.6 Relations: the formal machinery of LKMM

LKMM is a real, executable formal model written in the `cat` language, living in
`tools/memory-model/linux-kernel.cat`. It defines relations over memory events. You do not
need to write `cat`, but you must know the vocabulary:

| Relation | Meaning |
|---|---|
| `po` | **program order** — within one CPU, source order |
| `rf` | **reads-from** — links a write to the read that returns its value |
| `co` | **coherence order** — the total order of writes to one location |
| `fr` | **from-reads** — a read precedes a write that overwrites what it read |
| `ppo` | **preserved program order** — the subset of `po` the hardware actually maintains |
| `addr` / `data` / `ctrl` | address, data, control **dependencies** |
| `hb` | **happens-before** — the model's derived ordering |
| `prop` | propagation — how stores propagate to other CPUs (cumulativity) |

A litmus test outcome is **forbidden** iff the model's required relations form a **cycle**.
That's the whole theory in one sentence: *memory-model reasoning is cycle detection in a
graph of events.* Barriers exist to insert edges into that graph so that the bad cycle
becomes impossible.

LKMM's core axioms:
- **sc-per-loc** — coherence: `po-loc ∪ rf ∪ co ∪ fr` is acyclic (per-location SC).
- **happens-before** — `hb` is acyclic (no effect precedes its cause).
- **propagation** — `prop` axiom, covering cumulativity across CPUs.
- **rcu** — the RCU axiom, which makes grace-period guarantees part of the *same* formal
  model (a genuinely remarkable achievement; see Ch. 15).

### T.7 Dependencies: ordering for free, and the trap

Weak hardware preserves some order without barriers, because it *must*:

- **Address dependency:** `p = READ_ONCE(ptr); x = p->field;` — the second load cannot issue
  before the first returns, because its address isn't known. Ordering is guaranteed.
  **This is the entire basis of `rcu_dereference()`.**
- **Data dependency:** `r = READ_ONCE(a); WRITE_ONCE(b, r);` — the store's *value* depends on
  the load. Ordered.
- **Control dependency:** `if (READ_ONCE(a)) WRITE_ONCE(b, 1);` — orders the load before the
  **store**, because the CPU may not make a store visible speculatively. But it does **not**
  order the load before a subsequent **load**, because loads *can* be speculated past a
  branch. This asymmetry is a notorious bug source.

Three fatal traps with dependencies:

1. **Compilers destroy dependencies.** If the compiler can prove `p` has only one possible
   value, it substitutes the constant and the dependency evaporates. Hence
   `rcu_dereference()` exists rather than a plain load, and hence the long-standing
   `dependency_barrier` discussion. Never write `if (p == &known) use(&known);`.
2. **Control dependencies vanish under branch-collapsing.** If both arms of an `if` store
   the same value to the same variable, the compiler hoists the store out and the branch —
   and your ordering — disappears. `Documentation/memory-barriers.txt` devotes a long section
   to this; read it.
3. **Alpha.** Historically the only production CPU that could reorder *dependent* loads
   (its split cache could deliver stale data through a fresh pointer). This forced
   `smp_read_barrier_depends()` into the kernel for two decades. Alpha support was dropped
   from LKMM's concern in 5.9 and `READ_ONCE()` now carries the fix internally. The reason
   to know this: it explains why `rcu_dereference()` exists as a separate concept from a
   plain load, and it is the standard "does this person really know memory models" question.

### T.8 Cumulativity and transitivity

A barrier on CPU 1 must order not only CPU 1's own accesses but also stores by *other* CPUs
that CPU 1 has already observed. That property is **cumulativity**.

- **A-cumulative** barrier: orders prior stores *that this CPU has read* before the barrier.
- **B-cumulative**: orders later stores propagated after the barrier.

`smp_mb()` is fully cumulative. `smp_store_release()`/`smp_load_acquire()` chains give
**transitive** ordering — if A releases to B and B releases to C, then C sees A's writes.
This is why release/acquire is usable for multi-hop protocols.

`smp_wmb()`/`smp_rmb()` are weaker and **non-cumulative in general** on some architectures.
LKMM's rule of thumb, straight from the maintainers: *prefer release/acquire; use `smp_mb()`
when you need a full fence; use `smp_wmb`/`smp_rmb` only when you know exactly why.*

### T.9 The out-of-thin-air problem (why the C11 model is broken and LKMM is pragmatic)

The C11/C++11 model, using `memory_order_relaxed`, cannot rule out "out-of-thin-air" (OOTA)
values: self-justifying speculative executions in which `r1 == r2 == 42` arises from nowhere.
No hardware does this; no compiler does this; but the *formal model* permits it, and 15 years
of committee work have not produced an agreed fix.

LKMM sidesteps this by **not** being a purely axiomatic value-based model in that region:
it treats dependencies as ordering (which real hardware honours) rather than trying to
define semantic dependency. The trade-off: LKMM is not fully compositional with the C11
model, and the kernel therefore *does not* use C11 atomics. That is a deliberate,
much-debated engineering decision — and a favourite senior interview topic.

### T.10 Hardware models you must recognize

| ISA | Model | Notes |
|---|---|---|
| **x86-64** | **TSO** (Total Store Order) | Only StoreLoad reordering. Loads are acquire, stores are release, *for free*. `lock`-prefixed RMW is a full fence. `smp_mb()` = `mfence`/`lock addl $0,-4(%rsp)` |
| **ARM64** | weak, other-multi-copy-atomic | `LDAR`/`STLR` give acquire/release *cheaply in hardware* — release/acquire is nearly free, full `DMB ISH` is not |
| **RISC-V** | **RVWMO** | weak; `fence r,rw` etc.; `.aq`/`.rl` bits on AMOs |
| **PowerPC** | weak, non-multi-copy-atomic | `lwsync`, `sync`, `isync`; historically the hardest target |
| **Alpha** | weakest ever shipped | reordered dependent loads; shaped the kernel's APIs permanently |

**Portability consequence:** code developed and tested only on x86 is *systematically*
under-barriered. x86 silently provides MP ordering. Your ARM64 users find the bug. The
correct workflow is: reason with LKMM, verify with `herd7`, then test on ARM64.

### T.11 Why atomics are more than "atomic"

An atomic operation gives **indivisibility** (no torn/interleaved result) — but indivisibility
says nothing about **ordering** relative to *other* locations. Linux therefore defines, for
most RMW ops, four ordering variants:

```
atomic_add_return()        — fully ordered (implies smp_mb() before and after)
atomic_add_return_relaxed() — atomic, no ordering at all
atomic_add_return_acquire() — atomic + acquire (later accesses can't move before)
atomic_add_return_release() — atomic + release (earlier accesses can't move after)
```

And the kernel-specific rules that catch everyone:

- **Non-value-returning RMWs are unordered.** `atomic_inc()`, `atomic_dec()`, `atomic_add()`
  provide *no* ordering. If you need ordering around one, use
  `smp_mb__before_atomic()` / `smp_mb__after_atomic()`.
- **Value-returning RMWs are fully ordered.** `atomic_inc_return()`, `atomic_dec_and_test()`,
  `atomic_xchg()`, `atomic_cmpxchg()` (on success).
- **Conditional RMWs are ordered only on success.** `atomic_cmpxchg()` that fails provides
  no ordering. `atomic_try_cmpxchg()` and `atomic_add_unless()` likewise. This is a
  frequent, subtle bug.
- `atomic_read()`/`atomic_set()` are just `READ_ONCE`/`WRITE_ONCE`. **No ordering, no
  atomicity beyond single-copy.**

The full, authoritative table is `Documentation/atomic_t.txt`. Read it three times.

### T.12 CAS, ABA, and progress guarantees

`cmpxchg` is the universal primitive: Herlihy's 1991 hierarchy proves compare-and-swap has
**infinite consensus number**, i.e. it can implement any lock-free object for any number of
threads — unlike test-and-set or fetch-and-add, which cannot. That is the theoretical reason
every lock-free structure in the kernel bottoms out in `cmpxchg`.

Progress classes (know the words, they appear in review):

- **Obstruction-free** — a thread makes progress if it runs alone.
- **Lock-free** — *some* thread always makes progress (system-wide progress; individual
  starvation possible).
- **Wait-free** — *every* thread makes progress in bounded steps.

Kernel lock-free code is almost always merely *lock-free*, and that's deliberate: wait-free
algorithms are dramatically more complex and slower in the common case. What the kernel
actually wants from lock-freedom is usually not throughput but **context-safety**: the ability
to run in NMI or to be interrupted at any point without deadlock. `printk`'s ringbuffer
(`kernel/printk/printk_ringbuffer.c`) is the canonical example — it must work from NMI, so it
cannot take a lock, so it is lock-free.

**ABA:** a CAS sees the expected value and succeeds, but the location was changed to
something else and back in between. The CAS cannot tell. Mitigations: version/tag counters
packed alongside the pointer (needs `cmpxchg_double`/128-bit CAS), pointer tagging in unused
low bits, or — the kernel's usual answer — avoid the problem structurally with RCU and
`SLAB_TYPESAFE_BY_RCU` identity re-checks (Ch. 12, T.5).

### T.13 The practical rule set (derived, not memorized)

1. Unshared or lock-protected data → plain C accesses. Reason sequentially.
2. Concurrently accessed without a lock → `READ_ONCE`/`WRITE_ONCE` **always**, no exceptions.
3. Need a counter → `atomic_t`; pick the ordering variant from what you need, not by habit.
4. Publishing an initialized object → `smp_store_release()` (or `rcu_assign_pointer()`).
5. Consuming a published object → `smp_load_acquire()` (or `rcu_dereference()`).
6. Need SB-style ordering (store then load a *different* location) → nothing weaker than
   `smp_mb()` will do.
7. Unsure → **write the litmus test and run `herd7`.** This is not exotic; it is a 30-second
   loop and it is what maintainers will ask you to do.

---

## 1. Concept — the Linux barrier and atomic API surface

### 1.1 Compiler-only barriers

```c
barrier();                 /* compiler may not move memory accesses across this */
READ_ONCE(x); WRITE_ONCE(x, v);
data_race(x);              /* "this race is intentional", silences KCSAN */
ASSERT_EXCLUSIVE_ACCESS(x) /* KCSAN assertion: nobody else may touch x here */
```

### 1.2 CPU barriers

| Barrier | Orders | Cost on x86 | Cost on ARM64 |
|---|---|---|---|
| `smp_mb()` | all ← → all | `lock addl`/`mfence` (~20–30 cyc) | `dmb ish` |
| `smp_rmb()` | loads ← → loads | nop (compiler barrier) | `dmb ishld` |
| `smp_wmb()` | stores ← → stores | nop | `dmb ishst` |
| `smp_store_release(p, v)` | all earlier ← store | plain store + compiler barrier | `stlr` (cheap!) |
| `smp_load_acquire(p)` | load → all later | plain load + compiler barrier | `ldar` (cheap!) |
| `smp_mb__before_atomic()` | before unordered RMW | `lock addl` | `dmb ish` |
| `smp_mb__after_atomic()` | after unordered RMW | `lock addl` | `dmb ish` |
| `smp_cond_load_acquire(p, cond)` | spin-wait with acquire | | uses WFE on ARM |

Non-`smp_` variants (`mb()`, `rmb()`, `wmb()`) also order against **devices** (MMIO/DMA) and
are *not* compiled out on UP. For device I/O you additionally have `dma_rmb()`, `dma_wmb()`,
`__iormb()`, and — critically — the fact that `readl()`/`writel()` already contain ordering.
Ch. 34 covers MMIO ordering in full.

### 1.3 The atomic families

```c
atomic_t        /* int-sized;  atomic_read/set/add/sub/inc/dec/and/or/xor/cmpxchg/... */
atomic64_t      /* 64-bit */
atomic_long_t
refcount_t      /* saturating; Ch. 12 */
local_t         /* atomic w.r.t. interrupts on THIS cpu only — cheaper, no LOCK prefix */
DECLARE_BITMAP  /* set_bit/clear_bit/test_and_set_bit — atomic per-bit */
```

Bitops ordering mirrors the atomic rules exactly: `set_bit()` unordered,
`test_and_set_bit()` fully ordered, `__set_bit()` non-atomic entirely, plus
`clear_bit_unlock()` / `test_and_set_bit_lock()` for release/acquire.

---

## 2. Internals

### 2.1 Where the code lives

```
include/linux/atomic/            generated atomic API (atomic-arch-fallback.h etc.)
scripts/atomic/                  ★ the generator — read gen-atomic-fallback.sh
arch/x86/include/asm/atomic.h    x86 implementations (lock xadd, lock cmpxchg)
arch/arm64/include/asm/atomic_ll_sc.h / atomic_lse.h   LL/SC vs LSE atomics
include/asm-generic/barrier.h    generic barrier definitions
Documentation/atomic_t.txt       ★ the normative API doc
Documentation/memory-barriers.txt ★ 3000 lines, the canonical text
tools/memory-model/              ★ the formal model + herd7 harness
```

The atomic API is **generated**, not hand-written: `scripts/atomic/` produces the
`_relaxed`/`_acquire`/`_release`/fully-ordered variants and the fallbacks for architectures
that only implement some. Reading the generator teaches you the ordering matrix better than
any prose.

### 2.2 x86: `lock` prefix is the whole story

```asm
lock xaddl  %eax, (%rdi)    ; atomic_add_return
lock cmpxchgl %esi, (%rdi)  ; atomic_cmpxchg
lock addl $0, -4(%rsp)      ; smp_mb() — cheaper than mfence on most parts
```
Any `lock`-prefixed instruction is a full memory fence on x86. That's why
`atomic_inc_return()` needs no extra barriers there — and why porting to ARM64 exposes bugs.

### 2.3 ARM64: LL/SC vs LSE

```asm
1:  ldxr  w1, [x0]          ; load-exclusive
    add   w1, w1, w2
    stxr  w3, w1, [x0]      ; store-exclusive, w3 = 0 on success
    cbnz  w3, 1b            ; retry on contention
```
ARMv8.1-LSE replaces the loop with single instructions (`ldadd`, `casal`, `swpal`), which
scale far better under contention. The kernel picks at boot via alternatives patching —
`arch/arm64/include/asm/atomic_lse.h`. Acquire/release are *encoded in the instruction*
(`ldar`/`stlr`, `.a`/`.l` suffixes), which is why release/acquire is the cheap idiom on ARM64
and full `smp_mb()` is comparatively expensive. **Design your protocols in release/acquire.**

---

## 3. Practice

### 3.1 Run the formal model — this is the headline skill of this chapter

```bash
sudo apt install opam && opam init && opam install herdtools7   # or: nix-shell -p herdtools7
cd linux/tools/memory-model
cat litmus-tests/MP+fencewmbonceonce+fencermbonceonce.litmus
herd7 -conf linux-kernel.cfg litmus-tests/MP+fencewmbonceonce+fencermbonceonce.litmus
```

Write your own. Save as `MY-mp-broken.litmus`:

```
C MY-mp-broken

{}

P0(int *data, int *flag)
{
	WRITE_ONCE(*data, 42);
	WRITE_ONCE(*flag, 1);
}

P1(int *data, int *flag)
{
	int r1;
	int r2;

	r1 = READ_ONCE(*flag);
	r2 = READ_ONCE(*data);
}

exists (1:r1=1 /\ 1:r2=0)
```

```bash
herd7 -conf linux-kernel.cfg MY-mp-broken.litmus
# => Sometimes 1 ... "Positive: 1" — the bad outcome IS possible
```

Now fix it and re-run:

```
P0(int *data, int *flag)
{
	WRITE_ONCE(*data, 42);
	smp_store_release(flag, 1);
}

P1(int *data, int *flag)
{
	int r1;
	int r2;

	r1 = smp_load_acquire(flag);
	r2 = READ_ONCE(*data);
}
```
```bash
herd7 -conf linux-kernel.cfg MY-mp-fixed.litmus
# => Never 0 0 ... "Positive: 0"   ← proven forbidden
```

**Then run it on real hardware** with `klitmus7`, which turns a litmus test into a kernel
module:

```bash
mkdir -p /tmp/lit && klitmus7 -o /tmp/lit MY-mp-broken.litmus
cd /tmp/lit && make && sudo sh run.sh
```
Run this on x86 (bad outcome never observed) and on an ARM64 box (bad outcome observed).
Seeing it happen on real silicon is the moment memory models stop being abstract.

### 3.2 The publish/consume idiom, correctly

```c
struct config {
	int a, b, c;
};
static struct config __rcu *cur_config;

/* Publisher */
static int publish(int a, int b, int c)
{
	struct config *new = kmalloc(sizeof(*new), GFP_KERNEL);
	struct config *old;

	if (!new)
		return -ENOMEM;
	new->a = a; new->b = b; new->c = c;

	old = rcu_replace_pointer(cur_config, new, lockdep_is_held(&cfg_mutex));
	/* rcu_assign_pointer() == smp_store_release(): all three stores above
	 * are guaranteed visible to anyone who sees the new pointer. */
	kfree_rcu(old, rcu);
	return 0;
}

/* Consumer */
static void consume(void)
{
	struct config *c;

	rcu_read_lock();
	c = rcu_dereference(cur_config);
	/* address dependency orders these loads after the pointer load */
	pr_info("%d %d %d\n", c->a, c->b, c->c);
	rcu_read_unlock();
}
```

### 3.3 Atomic ordering traps — spot the bug

```c
/* BUG 1: atomic_inc() provides NO ordering. */
obj->ready = 1;
atomic_inc(&obj->count);           /* other CPU may see count but not ready */
/* FIX: */
smp_mb__before_atomic();
atomic_inc(&obj->count);

/* BUG 2: failed cmpxchg gives no ordering. */
if (atomic_cmpxchg(&state, IDLE, BUSY) != IDLE)
	return read_shared_thing();  /* unordered! */

/* BUG 3: waiting without READ_ONCE — compiler hoists the load out of the loop. */
while (!obj->done)
	cpu_relax();
/* FIX: */
while (!READ_ONCE(obj->done))
	cpu_relax();
/* BETTER: */
smp_cond_load_acquire(&obj->done, VAL);
```

### 3.4 Enable the race detector

```
CONFIG_KCSAN=y
CONFIG_KCSAN_STRICT=y
CONFIG_KCSAN_REPORT_VALUE_CHANGE_ONLY=n
```
KCSAN (Kernel Concurrency Sanitizer) uses sampled watchpoints with delays to catch real data
races at runtime. It is the single best tool for finding missing `READ_ONCE()`. Run your
module under it before posting a patch.

---

## 3.5 Extended practice

### Lab 13.A — The complete litmus-test workbench

Set this up once; you will use it for the rest of your career.

```bash
# Install herdtools7
sudo apt install -y opam && opam init -y && eval $(opam env)
opam install -y herdtools7
# or: nix-shell -p herdtools7   |   brew install herdtools7

cd linux/tools/memory-model
ls litmus-tests/ | head -30
cat README

# Run the whole standard suite (takes a few minutes):
./scripts/checkalllitmus.sh
# Compare model output against the documented expectations:
./scripts/checklitmushist.sh
```

Now build the six canonical tests from T.3 yourself. Create `/tmp/lit/` and write each:

```
C SB
{}
P0(int *x, int *y) { int r1; WRITE_ONCE(*x, 1); r1 = READ_ONCE(*y); }
P1(int *x, int *y) { int r2; WRITE_ONCE(*y, 1); r2 = READ_ONCE(*x); }
exists (0:r1=0 /\ 1:r2=0)
```
```
C LB
{}
P0(int *x, int *y) { int r1; r1 = READ_ONCE(*x); WRITE_ONCE(*y, 1); }
P1(int *x, int *y) { int r2; r2 = READ_ONCE(*y); WRITE_ONCE(*x, 1); }
exists (0:r1=1 /\ 1:r2=1)
```
```
C IRIW
{}
P0(int *x) { WRITE_ONCE(*x, 1); }
P1(int *y) { WRITE_ONCE(*y, 1); }
P2(int *x, int *y) { int r1; int r2; r1 = READ_ONCE(*x); smp_rmb(); r2 = READ_ONCE(*y); }
P3(int *x, int *y) { int r3; int r4; r3 = READ_ONCE(*y); smp_rmb(); r4 = READ_ONCE(*x); }
exists (2:r1=1 /\ 2:r2=0 /\ 3:r3=1 /\ 3:r4=0)
```
```bash
for t in SB LB IRIW MP; do
  echo "=== $t"; herd7 -conf linux-kernel.cfg /tmp/lit/$t.litmus | tail -6
done
```
Record, for each: **Never / Sometimes / Always**, and the `Positive:` count. Then add the
appropriate barrier to each and show the bad outcome becomes `Never`. Keep this table.

### Lab 13.B — Observe SB on real silicon with `klitmus7`

```bash
mkdir -p /tmp/klit && cd /tmp/klit
klitmus7 -o . /tmp/lit/SB.litmus
make
sudo sh run.sh | tail -20
```
Output looks like:
```
Histogram (4 states)
1234  *>0:r1=0; 1:r2=0;      ← the "forbidden by SC" outcome, OBSERVED
...
Ok  Sometimes 1234 9998766
```
**You just watched your x86 CPU violate sequential consistency.** Now add `smp_mb()` between
the store and the load in both threads, rebuild, and confirm the count drops to zero.

Repeat the whole exercise for **MP** on an ARM64 machine (a Raspberry Pi 4/5, an Ampere
instance, an Apple Silicon Linux VM, or `qemu-system-aarch64` with `-smp 4` — note TCG
is sequentially consistent so you need real hardware or KVM on arm64).

### Lab 13.C — Demonstrate every compiler hazard from T.5

```c
/* /tmp/hazards.c — compile with: gcc -O2 -S -o - /tmp/hazards.c  */
int flag, data;
long big;

/* 1. Load fusing: the loop becomes `if (!flag) for(;;);` */
void fuse(void)        { while (!flag) ; }
void fuse_fixed(void)  { while (!*(volatile int *)&flag) ; }

/* 2. Store fusing: only the last store survives */
void store_fuse(void)  { data = 1; data = 2; data = 3; }

/* 3. Invented load: two "reads" of one variable may disagree */
int invent(int *p)     { if (flag > 0) return flag * 2; return flag + 1; }

/* 4. Dependency destruction by value speculation */
int dep(int *p)        { int v = *p; if (v == 42) return 42; return v; }
```
```bash
gcc -O2 -S -o /tmp/hz.s /tmp/hazards.c
grep -A8 '^fuse:' /tmp/hz.s          # infinite loop, one load
grep -A8 '^fuse_fixed:' /tmp/hz.s    # load every iteration
grep -A8 '^store_fuse:' /tmp/hz.s    # ONE store of 3
```
Now rewrite each using `READ_ONCE`/`WRITE_ONCE` semantics (`volatile` cast + `barrier()`)
and diff the assembly. **This is T.5, proven, in ten minutes.**

### Lab 13.D — Measure barrier cost on your hardware

```c
#define TIME(name, stmt) do {                                   \
	u64 t0 = ktime_get_ns(); int i;                         \
	for (i = 0; i < 10000000; i++) { stmt; }                \
	pr_info("%-24s %5llu ps/op\n", name,                    \
		(ktime_get_ns() - t0) / 10000);                 \
} while (0)

static int x, y;
static atomic_t a;

static int __init barrier_bench(void)
{
	TIME("baseline store",   WRITE_ONCE(x, i));
	TIME("barrier()",        { WRITE_ONCE(x, i); barrier(); });
	TIME("smp_wmb()",        { WRITE_ONCE(x, i); smp_wmb(); });
	TIME("smp_rmb()",        { y = READ_ONCE(x); smp_rmb(); });
	TIME("smp_store_release",smp_store_release(&x, i));
	TIME("smp_load_acquire", y = smp_load_acquire(&x));
	TIME("smp_mb()",         { WRITE_ONCE(x, i); smp_mb(); });
	TIME("atomic_inc",       atomic_inc(&a));
	TIME("atomic_inc_return",atomic_inc_return(&a));
	TIME("cmpxchg",          atomic_cmpxchg(&a, i, i + 1));
	return 0;
}
```
Run on x86-64 **and** on arm64. The two profiles are strikingly different:
- x86: `smp_wmb`/`smp_rmb` are free (nops); `smp_mb` costs ~20–30 cycles; release/acquire are
  free.
- arm64: `smp_store_release`/`smp_load_acquire` (`stlr`/`ldar`) are **much cheaper** than
  `dmb ish`.

**Conclusion you should draw and then apply forever: design protocols in release/acquire,
not in `smp_mb()`.**

### Lab 13.E — A correct lock-free SPSC ring, verified

```c
struct ring {
	u32   head;              /* producer writes, consumer reads */
	u32   tail;              /* consumer writes, producer reads */
	u32   mask;
	void **buf;
};

static bool ring_push(struct ring *r, void *item)
{
	u32 head = r->head;                          /* only we write head */
	u32 tail = smp_load_acquire(&r->tail);       /* ACQUIRE: see consumer's frees */

	if (head - tail > r->mask)
		return false;                        /* full */

	r->buf[head & r->mask] = item;
	smp_store_release(&r->head, head + 1);       /* RELEASE: publish AFTER the write */
	return true;
}

static void *ring_pop(struct ring *r)
{
	u32 tail = r->tail;
	u32 head = smp_load_acquire(&r->head);       /* ACQUIRE: see producer's data */
	void *item;

	if (head == tail)
		return NULL;                         /* empty */

	item = r->buf[tail & r->mask];
	smp_store_release(&r->tail, tail + 1);       /* RELEASE: free the slot AFTER reading */
	return item;
}
```
Now **prove it** with a litmus test capturing the essential MP shape, and **test it** with
two kthreads on different CPUs plus a sequence-number check. Then compare against
`include/linux/kfifo.h` (Ch. 10 T.5) and explain any differences.

### Lab 13.F — Find a real missing-barrier bug

```bash
cd linux
git log --oneline --grep='memory barrier' --grep='smp_mb' --all-match | head -30
git log --oneline -S'smp_store_release' -- drivers/ | head -20
git log --oneline --grep='READ_ONCE' --grep='data race' --all-match | head -20
```
Pick three. For each, write down: (1) which litmus test the bug corresponds to,
(2) which architectures could observe it, (3) how it was found (review? KCSAN? a bug report
from an arm64 user?). The answer to (3) is almost always "an arm64 user", which is T.10's
portability point made concrete.

### Lab 13.G — Run KCSAN and find real races

```bash
./scripts/config -e KCSAN -e KCSAN_STRICT -d KCSAN_REPORT_VALUE_CHANGE_ONLY \
                 --set-val KCSAN_UDELAY_TASK 80 --set-val KCSAN_UDELAY_INTERRUPT 20
make -j$(nproc) && boot

# Then run a workload and watch:
sudo dmesg -w | grep -A25 'BUG: KCSAN'
# Tunables at runtime:
ls /sys/kernel/debug/kcsan
echo on | sudo tee /sys/kernel/debug/kcsan
```
A KCSAN report shows **both** accesses, both stacks, and the values. Annotate benign races
with `data_race()` and genuine ones with `READ_ONCE`/`WRITE_ONCE` or a lock — and be able to
explain which you chose and why.

---

## 4. Mastery drills

1. **Reproduce SB on x86.** Write the SB litmus test, run `herd7` (expect: allowed), then
   `klitmus7` on a real x86 box. You *will* observe `0 0`. Then add `smp_mb()` and show it
   disappears. Explain the store buffer to someone out loud.

2. **Read `Documentation/memory-barriers.txt` cover to cover.** It is ~3000 lines. Take two
   evenings. Write a one-page summary. Everyone senior has done this.

3. **Audit an atomic.** Find five uses of `atomic_inc()` in `drivers/`. For each, determine
   whether ordering is required and whether a barrier is present. If you find a genuine bug,
   that's a real patch.

4. **Control-dependency hunt.** `git log --grep="control dependenc"` in the kernel. Read three
   commits. Explain in each case what the compiler was permitted to do.

5. **Derive the lock.** Using only `atomic_cmpxchg`, `smp_load_acquire`, `smp_store_release`,
   implement a correct test-and-set spinlock. Prove with `herd7` that a critical-section
   invariant holds (the `tools/memory-model/litmus-tests/` directory has lock tests to copy).
   Then explain why the kernel uses a queued spinlock instead (Ch. 14).

6. **Cumulativity.** Construct the WRC litmus test. Show it is allowed with `smp_wmb`/`smp_rmb`
   on some models and forbidden with release/acquire. Explain A-cumulativity from the result.

7. **Architecture comparison.** Take `MP` and compile the same C function for x86-64 and
   arm64 (`make ARCH=arm64 ... .s`). Diff the generated assembly for `smp_store_release`.
   Explain why ARM64 costs less than you expected.

8. **Explain OOTA** to a colleague and say why the kernel does not use C11 atomics. If you can
   do this convincingly you are ahead of most working kernel developers.

---

## 5. Further reading

**Primary (kernel):**
- `Documentation/memory-barriers.txt` — the canonical text
- `Documentation/atomic_t.txt`, `Documentation/atomic_bitops.txt`
- `tools/memory-model/Documentation/` — `explanation.txt` (a *textbook*, ~2000 lines,
  the single best explanation of LKMM), `recipes.txt`, `litmus-tests.txt`,
  `simple.txt`, `ordering.txt`, `control-dependencies.txt`
- `Documentation/dev-tools/kcsan.rst`

**Papers:**
- Lamport, "How to Make a Multiprocessor Computer That Correctly Executes Multiprocess
  Programs" (1979) — the definition of SC
- Adve & Hill, "Weak Ordering — A New Definition" (ISCA 1990) — DRF-SC
- Alglave, Maranget, Tautschnig, "Herding Cats" (TOPLAS 2014) — the `cat`/herd framework
- Alglave et al., "Frightening Small Children and Disconcerting Grown-ups: Concurrency in
  the Linux Kernel" (ASPLOS 2018) — **the LKMM paper**
- Herlihy, "Wait-Free Synchronization" (TOPLAS 1991) — consensus numbers, why CAS is universal
- Boehm & Adve, "Foundations of the C++ Concurrency Memory Model" (PLDI 2008)
- Batty et al., "Mathematizing C++ Concurrency" (POPL 2011)

**Books/notes:**
- McKenney, *Is Parallel Programming Hard, And, If So, What Can You Do About It?*
  — **free, and the best book in the field.** Chapters 14–15 are this chapter, expanded
  to 200 pages. https://mirrors.edge.kernel.org/pub/linux/kernel/people/paulmck/perfbook/
- Sorin, Hill, Wood, *A Primer on Memory Consistency and Cache Coherence* (Morgan & Claypool)
- Preshing's blog series on memory ordering — the gentlest correct introduction

**Talks:**
- Paul McKenney, any of his LinuxCon/Kernel Recipes talks on LKMM
- Will Deacon, "The Linux Kernel Memory Model" (Linux Plumbers)

→ Next: [14-locking.md](14-locking.md)

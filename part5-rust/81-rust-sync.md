# Chapter 81 — Synchronization in Kernel Rust

> Ch. 13–16 established the kernel's concurrency model: barriers, atomics, spinlocks,
> mutexes, RCU, per-CPU. This chapter is about how that model is *expressed in types* —
> and where the expression is genuinely stronger than C, where it is merely equivalent, and
> where it is still a convention you must follow by hand.

---

## Theory & First Principles

### T.0 — Start here: two traits that nobody writes and everything depends on

Ask a kernel C question: **which of these is safe to hand to another thread?**

```c
struct mutex          *m;   /* yes */
struct completion     *c;   /* yes */
void                  *percpu_ptr;  /* NO -- it is bound to THIS CPU */
struct task_struct    *t;   /* only with a reference held */
char                   local_buf[64];  /* only if its lifetime allows */
```

In C the answer lives in your head and in review comments. Get it wrong and you have a data
race — which does not crash, does not reproduce, and shows up once a month in production
(Ch. 13 §T.0).

**Rust encodes the answer in two marker traits, and the entire concurrency model is these
two definitions:**

```
   Send  =  "it is safe to MOVE a value of this type to another thread"
   Sync  =  "it is safe to SHARE a &T of this type with another thread"
            (equivalently: &T is Send)
```

That is it. They have no methods. They are **automatically derived** — a struct is `Send` if
all its fields are `Send` — and the compiler consults them at every thread boundary.

**Work through why they must be two separate traits, because it is the sharpest part:**

| Type | `Send`? | `Sync`? | Why |
|---|---|---|---|
| `u32`, `String`, `KVec<u8>` | yes | yes | ordinary owned data |
| `Arc<T>` (atomic refcount) | yes (if `T: Send+Sync`) | yes | the count is atomic |
| `Rc<T>` (non-atomic refcount) | **no** | **no** | two threads would race on the count |
| `Cell<T>` / `RefCell<T>` | yes | **no** | movable, but sharing gives unsynchronized mutation |
| `MutexGuard<T>` | **no** | yes | a lock must be released by the thread that took it |
| raw per-CPU pointer | **no** | **no** | bound to one CPU |

**`MutexGuard` is the one to remember.** "You may look at it from another thread, but you may
not *move* it there" is a real, subtle, load-bearing rule in every OS — `pthread_mutex_unlock`
from a different thread is undefined behaviour, and the kernel's `mutex` has the same
requirement. In C that rule lives in documentation; in Rust it is `!Send`, and violating it
is a compile error.

**Now the second idea, which is where this connects to Ch. 78 §T.0:** the C convention is
"take the lock, then touch the data." Two separate things, related only by a comment. Rust
inverts it:

```rust
struct Inner { count: u32, buf: KVec<u8> }

let m: Mutex<Inner> = ...;

// There is NO WAY to reach `count` without going through the lock:
{
    let mut g = m.lock();   // returns Guard<'_, Inner>
    g.count += 1;           // Deref through the guard
}                           // Drop releases it -- on EVERY path, including `?`
```

**The data is inside the lock, so "forgot to take the lock" is not expressible, and "forgot to
release on the error path" is not expressible.** Those are two of the three most common
locking bugs in kernel C. This is, once again, "move the obligation into the primitive"
(Ch. 05, 28, 34, 52, 82).

**And the honest limit, which you must state or the chapter is propaganda:**

> **Rust prevents data races. It does not prevent deadlocks.**

Lock *ordering* is a global property of a program; the type system sees one function at a
time. `lock(A); lock(B)` in one place and `lock(B); lock(A)` in another compiles perfectly
and deadlocks exactly as it would in C. **You still need lockdep** (Ch. 14, Ch. 06), and
kernel Rust locks are integrated with it for precisely this reason. Similarly, Rust does not
prevent priority inversion, lock convoying, or choosing a mutex where you needed RCU.

**A final note on context**, because it is the kernel-specific wrinkle: a `Mutex` may sleep, a
`SpinLock` may not, and calling a sleeping function from atomic context is a bug the type
system does **not** currently catch. The `kernel` crate's `SpinLockIrq`, `Guard`, and
`might_sleep` machinery are where this is actively being worked out — and it is a good
example of a kernel invariant that is *not yet* encoded in a type, and might be.

```bash
sed -n '1,100p' rust/kernel/sync/lock/mutex.rs
grep -rn 'unsafe impl Send\|unsafe impl Sync' rust/kernel/ | head -20
# each of those is a hand-written safety claim -- read the SAFETY comment on each
```

---

### T.1 — The two marker traits that carry the whole model

```rust
pub unsafe auto trait Send { }   // "this value can be MOVED to another thread"
pub unsafe auto trait Sync { }   // "&T can be SHARED with another thread"
```

They are `auto` — automatically implemented for any type whose fields all implement them —
and `unsafe` — implementing them manually is a claim you must justify.

The definitional relationship, which is the single most important sentence:

> `T: Sync` **if and only if** `&T: Send`.

So "can I share a reference across CPUs?" and "can I send a shared reference across CPUs?"
are the same question. From that, everything else follows:

| Type | `Send` | `Sync` | Because |
|---|---|---|---|
| `u32`, `KVec<u8>` | yes | yes | plain data |
| `&T` | if `T: Sync` | if `T: Sync` | sharing a reference *is* sharing |
| `&mut T` | if `T: Send` | if `T: Sync` | exclusive, so moving it is moving `T` |
| `Cell<T>`, `RefCell<T>` | if `T: Send` | **no** | unsynchronized interior mutability |
| `Arc<T>` | if `T: Send + Sync` | if `T: Send + Sync` | clones can go anywhere |
| `Mutex<T>` | if `T: Send` | **if `T: Send`** | the lock supplies the synchronization |
| `*const T`, `*mut T` | **no** | **no** | raw pointers carry no guarantees |

**The `Mutex<T>: Sync where T: Send` row is the one to understand.** A `Mutex` makes a
`Send`-but-not-`Sync` type shareable, because the lock provides the missing synchronization.
That is the type system stating, precisely, what a lock is *for*.

**Why this matters in the kernel.** In C, "is this structure safe to touch from another CPU?"
is answered by a comment, by convention, or by reading the whole subsystem. In Rust the
compiler answers it, and a violation is a compile error — not a KCSAN report if you happen to
have it enabled and happen to hit the race.

Concretely: raw pointers are `!Send + !Sync`, so any type holding one is too, which means
**every wrapper over a C object must explicitly claim `Send`/`Sync` with a justification.**
That forcing function is the mechanism. When you see:

```rust
// SAFETY: `Registration` holds a pointer to a C object that is internally
// synchronized by the subsystem's own lock, so it is safe to access from
// multiple threads.
unsafe impl Sync for Registration {}
```

you are looking at a claim that a reviewer can check — and a claim that C never requires
anyone to write down.

### T.2 — Data inside the lock, not beside it

```c
/* C: adjacency is the only relationship. */
struct foo {
	spinlock_t lock;
	int        counter;   /* "protected by lock" -- says a comment */
};
foo->counter++;           /* compiles fine, races */
```

```rust
// Rust: containment.
struct Foo {
    counter: SpinLock<i32>,
}
// foo.counter += 1;          // error: no `+=` on SpinLock<i32>
*foo.counter.lock() += 1;     // the ONLY way in
```

This is the single largest practical win of Rust locking, and the one to lead with. There is
no path to the data that does not go through the lock, because the data is *inside* it.

Two corollaries:
- Grouping matters. `SpinLock<Stats>` protects all of `Stats` together; three separate
  `SpinLock<u64>`s are three separate locks. **The type states the granularity**, which in C
  is another comment.
- Data protected by two locks, or by a lock held by a *different* object, needs
  `LockedBy<T, U>` — see §T.6.

### T.3 — The guard, and RAII unlocking

```rust
pub struct Guard<'a, T: ?Sized, B: Backend> { ... }

impl Deref for Guard   { type Target = T; }   // &T
impl DerefMut for Guard { }                   // &mut T
impl Drop for Guard    { fn drop(&mut self) { B::unlock(...) } }
```

```rust
{
    let mut g = state.lock();
    g.counter += 1;
    if g.counter > LIMIT {
        return Err(EOVERFLOW);   // unlocked here. Automatically.
    }
    g.flags |= DIRTY;
}   // and here
```

Three properties, each eliminating a real C bug class:

1. **Unlock on every path**, including `?`, `return`, and `break`. The "forgot to unlock on
   the error path" bug does not exist.
2. **The guard's lifetime is tied to the lock's.** You cannot return a `&T` derived from the
   guard past the guard's scope — the borrow checker rejects it. "Do not use the data after
   unlocking" is enforced.
3. **You cannot unlock twice**, and you cannot unlock a lock you do not hold, because the
   guard is the only unlock mechanism and it is consumed by `Drop`.

`drop(g)` is the explicit early unlock, which is the Rust spelling of narrowing a critical
section. Reviewers should look for it where a critical section looks long.

### T.4 — Backends: one `Lock`, many kernel primitives

```rust
pub struct Lock<T: ?Sized, B: Backend> {
    state: Opaque<B::State>,     // struct mutex, or spinlock_t, ...
    _pin: PhantomPinned,
    data: UnsafeCell<T>,
}

pub type Mutex<T>   = Lock<T, MutexBackend>;
pub type SpinLock<T> = Lock<T, SpinLockBackend>;
```

`B` is a type parameter, so the choice is compile-time and there is no indirection — a
`SpinLock::lock()` compiles to a direct call to the C `spin_lock`, inlined.

The available backends and their C equivalents:

| Rust | C | Context | Sleeps? |
|---|---|---|---|
| `Mutex<T>` | `struct mutex` | process only | yes |
| `SpinLock<T>` | `spinlock_t` | any, incl. softirq | no (but RT-sleeps) |
| `SpinLockIrq<T>` | `spin_lock_irqsave` | any, incl. hardirq | no |
| `CondVar` | `wait_queue_head_t` | with a lock | yes |
| `Arc<T>` | `kref`-style refcount | any | no |
| `GlobalLock` | statically-declared lock | any | depends |

Missing, as of writing: `rw_semaphore`, `seqlock`, `percpu_rwsem`, and full RCU
abstractions. Knowing the *gaps* is as important as knowing the API — it determines whether
a given driver can be written in Rust today.

### T.5 — `CondVar`: waiting, correctly

```rust
#[pin_data]
struct Queue {
    #[pin] items: Mutex<KVec<Item>>,
    #[pin] not_empty: CondVar,
}

impl Queue {
    fn pop(&self) -> Result<Item> {
        let mut guard = self.items.lock();
        while guard.is_empty() {
            // Atomically: release the lock, sleep, reacquire on wake.
            // Returns false if interrupted by a signal.
            if self.not_empty.wait_interruptible(&mut guard) {
                return Err(ERESTARTSYS);
            }
        }
        guard.pop().ok_or(EAGAIN)
    }

    fn push(&self, item: Item) -> Result {
        let mut guard = self.items.lock();
        guard.push(item, GFP_KERNEL)?;
        self.not_empty.notify_one();
        Ok(())
    }
}
```

Two things the API forces you to get right:

1. **`wait` takes `&mut Guard`.** You cannot wait without holding the lock — which is
   exactly the lost-wakeup prevention from Ch. 18. In C, `wait_event_interruptible` uses
   macro magic to re-check the condition before sleeping; here the structure makes the
   relationship explicit.
2. **`while`, not `if`.** Spurious wakeups are possible, so the condition must be re-checked.
   The API cannot force this — it is still a convention — but the shape of the call makes
   the `while` natural, and its absence is visible in review.

Signal handling is explicit: `wait_interruptible` returns a `bool` saying "a signal arrived,"
and you must decide. Compare with C's `if (wait_event_interruptible(...)) return -ERESTARTSYS;`
— same discipline, more visible.

### T.6 — `LockedBy<T, U>`: data protected by someone else's lock

A common kernel pattern that neither adjacency nor containment expresses: many objects, one
lock.

```c
struct controller {
	struct mutex lock;
	struct list_head devices;
};
struct device_entry {
	int state;         /* protected by controller->lock */
};
```

```rust
struct Controller {
    #[pin] inner: Mutex<ControllerInner>,
}

struct DeviceEntry {
    // "You can only get at `state` if you show me a guard for the
    // Controller's lock."
    state: LockedBy<i32, ControllerInner>,
}

// Usage:
let guard = controller.inner.lock();
let s = entry.state.access(&guard);      // compile error without the guard
```

`LockedBy` checks at runtime that the guard belongs to the *right* lock instance (by
address), and at compile time that a guard of the right *type* was presented. It is the
closest Rust gets to C's `__guarded_by` annotation, and it is strictly stronger because it
cannot be omitted.

### T.7 — Atomics

Kernel Rust does **not** use `core::sync::atomic` directly. It uses bindings to the kernel's
own `atomic_t`/`atomic64_t` and `refcount_t`, because:

- The Linux Kernel Memory Model (LKMM, Ch. 13) and the C11/Rust memory model are *different
  formalisms*. Mixing them in one kernel is not obviously sound, and the kernel community
  insisted on one model.
- `refcount_t`'s saturation and underflow-`WARN` semantics are a hardening feature that
  `core`'s `AtomicUsize` does not have.
- `READ_ONCE`/`WRITE_ONCE`, `smp_mb()`, and the acquire/release helpers must map onto the
  kernel's, so that `herd7`/LKMM litmus tests remain meaningful.

The practical rule: **use the kernel's atomic abstractions, and when you need a barrier,
reach for the kernel's barrier, not `core::sync::atomic::fence`.** This is an area of active
work; check `rust/kernel/sync/atomic.rs` for current status.

### T.8 — RCU

The abstraction is partial and worth understanding precisely.

```rust
let _guard = rcu::read_lock();          // rcu_read_lock()
let p = some_pointer.read();            // rcu_dereference()
use_it(p);
// _guard dropped -> rcu_read_unlock()
```

What the type system gives you:
- The guard's lifetime bounds the read-side section, so "do not hold the pointer past
  `rcu_read_unlock()`" becomes a borrow-checker rule rather than a convention.
- Getting a raw pointer out requires `unsafe`, so stashing it is visible.

What it does not give you:
- Nothing prevents you from calling a sleeping function inside the read-side section (§T.10).
- The update side (`rcu_assign_pointer`, `synchronize_rcu`, `call_rcu`) is thinner and more
  `unsafe`-heavy.

This is the clearest current example of "the abstraction exists but is incomplete," and it is
a good honest answer to "how mature is kernel Rust?"

### T.9 — Workqueues and deferred work

```rust
#[pin_data]
struct MyDriver {
    #[pin] work: Work<MyDriver>,
    data: u32,
}

impl_has_work! { impl HasWork<Self> for MyDriver { self.work } }

impl WorkItem for MyDriver {
    type Pointer = Arc<MyDriver>;
    fn run(this: Arc<MyDriver>) {
        pr_info!("work running, data = {}\n", this.data);
    }
}

// Enqueue:
let _ = workqueue::system().enqueue(driver.clone());
```

The type-level improvement: `Work<T>` is embedded in `T` (the intrusive pattern), the
`Pointer` associated type states what owns the item while queued (`Arc<T>` — so the object
cannot be freed while the work is pending), and enqueueing takes that pointer by value.

**That single design decision eliminates the most common workqueue bug in C**: freeing an
object with work still queued, because in C `cancel_work_sync()` in `remove()` is something
you must remember (→ `reference/debugging-scenarios.md` §1). Here, the work holds a reference.

### T.10 — What the types still do *not* say

Be able to list these; it is the honest boundary of the claim.

| Not encoded | Consequence | Tool |
|---|---|---|
| **May sleep** | `Mutex::lock()` in atomic context still compiles | `might_sleep()`, lockdep, runtime splat |
| **Lock ordering** | A→B and B→A still compiles | `lockdep` |
| **IRQ context** | Nothing distinguishes hardirq from process context in the type | review, `in_interrupt()` |
| **Preemption state** | Not tracked | `preempt_count` at runtime |
| **RCU section correctness** | Sleeping inside a read-side section compiles | `CONFIG_PROVE_RCU` |
| **Memory model subtleties** | Barrier placement is still yours to get right | LKMM, `herd7`, KCSAN |

There have been proposals — `klint`, a static context checker — and none has landed. The
accurate statement for an interview:

> **Rust eliminates data races entirely** — that is a hard guarantee from `Send`/`Sync` plus
> the borrow rules. **It does not eliminate deadlocks, sleeping-in-atomic, or memory-ordering
> bugs**, which remain runtime-detected by lockdep, `might_sleep`, and KCSAN exactly as in C.

That distinction — races eliminated, deadlocks not — is the crisp version, and it is the
answer that separates someone who has used it from someone who has read a blog post.

---

## 1. Internals

### Source map

| Path | Contents |
|---|---|
| `rust/kernel/sync.rs` | Module root; `LockClassKey`, re-exports |
| `rust/kernel/sync/lock.rs` | `Lock<T, B>`, `Guard`, the `Backend` trait |
| `rust/kernel/sync/lock/mutex.rs` | `Mutex`, `MutexBackend`, `new_mutex!` |
| `rust/kernel/sync/lock/spinlock.rs` | `SpinLock`, `SpinLockBackend`, `new_spinlock!` |
| `rust/kernel/sync/lock/global.rs` | `GlobalLock` — statically-declared locks |
| `rust/kernel/sync/condvar.rs` | `CondVar` over `wait_queue_head_t` |
| `rust/kernel/sync/arc.rs` | `Arc`, `ArcBorrow`, `UniqueArc` |
| `rust/kernel/sync/locked_by.rs` | `LockedBy<T, U>` |
| `rust/kernel/sync/poll.rs` | `PollCondVar`, `PollTable` for file `poll()` |
| `rust/kernel/workqueue.rs` | `Work<T>`, `WorkItem`, `Queue`, `impl_has_work!` |
| `rust/kernel/rcu.rs` | RCU read-side guard |
| `rust/kernel/task.rs` | `Task`, `current!()` |

### The `Backend` trait

```rust
pub unsafe trait Backend {
    type State;                    // struct mutex / spinlock_t
    type GuardState;               // e.g. saved IRQ flags for irqsave

    unsafe fn init(ptr: *mut Self::State, name: *const c_char,
                   key: *mut bindings::lock_class_key);
    unsafe fn lock(ptr: *mut Self::State) -> Self::GuardState;
    unsafe fn unlock(ptr: *mut Self::State, guard_state: &Self::GuardState);
}
```

`GuardState` is the elegant bit: for `spin_lock_irqsave` it carries the saved flags, so the
guard's `Drop` can restore them. That is `spin_lock_irqsave`/`spin_unlock_irqrestore` paired
by construction — you cannot restore the wrong flags or forget to restore them.

```rust
unsafe impl Backend for SpinLockBackend {
    type State = bindings::spinlock_t;
    type GuardState = ();
    unsafe fn lock(ptr: *mut Self::State) -> () {
        // SAFETY: `ptr` points to a valid, initialized spinlock_t.
        unsafe { bindings::spin_lock(ptr) }
    }
    unsafe fn unlock(ptr: *mut Self::State, _: &()) {
        // SAFETY: as above, and we hold the lock.
        unsafe { bindings::spin_unlock(ptr) }
    }
    ...
}
```

Zero-sized `GuardState` for the plain spinlock means the guard costs nothing beyond a
pointer. Compile-time dispatch plus a ZST equals literally the same machine code as the C
call.

### `Send`/`Sync` for `Lock<T, B>`

```rust
// SAFETY: `Lock` owns its data. Sending the lock to another thread is
// equivalent to sending T, so it requires T: Send.
unsafe impl<T: ?Sized + Send, B: Backend> Send for Lock<T, B> {}

// SAFETY: `Lock` serializes access to T, so sharing &Lock<T> across
// threads is sound as long as T can be *sent* to whichever thread
// acquires the lock. Notably this does NOT require T: Sync -- the lock
// supplies the synchronization.
unsafe impl<T: ?Sized + Send, B: Backend> Sync for Lock<T, B> {}
```

**Read those two impls carefully.** They are the formal statement of what a lock does, in
four lines, and they are checkable by a reviewer. There is no equivalent artifact anywhere in
the C kernel.

### `new_mutex!` and the lock class key

```rust
#[macro_export]
macro_rules! new_mutex {
    ($inner:expr $(, $name:literal)? $(,)?) => {
        $crate::sync::Mutex::new(
            $inner,
            $crate::optional_name!($($name)?),
            $crate::static_lock_class!(),   // a distinct static per call site
        )
    };
}
```

`static_lock_class!()` creates a `static` `lock_class_key` at the macro's expansion site,
which is exactly what `DEFINE_MUTEX`/`mutex_init` do in C via `__mutex_init` and
`__lockdep_no_validate__`. One key per *site*, not per *instance* — so a thousand devices'
locks share a class and lockdep can reason about ordering between classes.

---

## 2. Practice

### Lab 81.1 — Let the compiler refuse a data race

```rust
// SPDX-License-Identifier: GPL-2.0
//! The compiler rejects what KCSAN would only sometimes catch.

use kernel::prelude::*;
use kernel::sync::{new_spinlock, Arc, SpinLock};
use kernel::workqueue::{self, impl_has_work, new_work, Work, WorkItem};

// ---- (1) Unprotected shared mutable state: does not compile ----
struct Bad { counter: u64 }
// If you try to share &Bad across threads and mutate, you need &mut,
// and you cannot have &mut through a shared Arc. The compiler stops you
// at the point you try, not three months later in production.

// ---- (2) The correct version ----
#[pin_data]
struct Good {
    #[pin] counter: SpinLock<u64>,
    #[pin] work: Work<Good>,
}

impl_has_work! { impl HasWork<Self> for Good { self.work } }

impl WorkItem for Good {
    type Pointer = Arc<Good>;
    fn run(this: Arc<Good>) {
        // Runs on some other CPU, concurrently with the submitter.
        let mut c = this.counter.lock();
        *c += 1;
        pr_info!("work: counter = {}\n", *c);
    }
}

impl Good {
    fn new() -> impl PinInit<Self, Error> {
        try_pin_init!(Self {
            counter <- new_spinlock!(0u64, "good::counter"),
            work    <- new_work!("good::work"),
        })
    }
}

fn demo() -> Result {
    let g = Arc::pin_init(Good::new(), GFP_KERNEL)?;

    // Enqueue several times. Each enqueue holds a reference, so the
    // object cannot be freed while work is pending -- the C bug of
    // kfree()ing something with queued work is not expressible.
    for _ in 0..4 {
        let _ = workqueue::system().enqueue(g.clone());
    }

    // Meanwhile, this CPU touches it too -- correctly.
    {
        let mut c = g.counter.lock();
        *c += 100;
    }
    Ok(())
}
```

Then try to write the race:

```rust
// Attempt: hand out a &mut across the boundary.
// fn race(g: &Arc<Good>) {
//     let c: &mut u64 = &mut g.counter;   // error: no DerefMut on Arc
// }
//
// Attempt: hold the guard past the lock.
// fn escape(g: &Good) -> &u64 {
//     let guard = g.counter.lock();
//     &*guard                              // error[E0515]: cannot return
// }                                        // reference to local `guard`
```

**Error E0515 there is the type system saying "do not use the data after unlocking."** Write
the C version and observe that it compiles silently.

### Lab 81.2 — Producer/consumer with `CondVar`

```rust
// SPDX-License-Identifier: GPL-2.0
//! The classic bounded-buffer problem (→ reference/os-fundamentals.md §3)
//! with the kernel's Rust primitives.

use kernel::prelude::*;
use kernel::sync::{new_condvar, new_mutex, Arc, CondVar, Mutex};

const CAPACITY: usize = 8;

#[pin_data]
pub struct Channel<T> {
    #[pin] buf: Mutex<KVec<T>>,
    #[pin] not_empty: CondVar,
    #[pin] not_full: CondVar,
}

impl<T> Channel<T> {
    pub fn new() -> impl PinInit<Self, Error> {
        try_pin_init!(Self {
            buf       <- new_mutex!(KVec::new(), "Channel::buf"),
            not_empty <- new_condvar!("Channel::not_empty"),
            not_full  <- new_condvar!("Channel::not_full"),
        })
    }

    pub fn send(&self, value: T) -> Result {
        let mut guard = self.buf.lock();
        // WHILE, not if: spurious wakeups are permitted, and the
        // condition may have been consumed by another waiter.
        while guard.len() >= CAPACITY {
            if self.not_full.wait_interruptible(&mut guard) {
                return Err(ERESTARTSYS);
            }
        }
        guard.push(value, GFP_KERNEL)?;
        // Notify while still holding the lock: simplest and correct.
        // (Notifying after unlocking is a valid optimization; notifying
        //  before acquiring is a lost wakeup.)
        self.not_empty.notify_one();
        Ok(())
    }

    pub fn recv(&self) -> Result<T> {
        let mut guard = self.buf.lock();
        while guard.is_empty() {
            if self.not_empty.wait_interruptible(&mut guard) {
                return Err(ERESTARTSYS);
            }
        }
        let v = guard.remove(0);
        self.not_full.notify_one();
        Ok(v)
    }
}
```

Points to draw out:
- `wait_interruptible(&mut guard)` **takes the guard**, so waiting without the lock is not
  expressible. That is the lost-wakeup fix (Ch. 18 §T.4), structurally.
- Two condition variables, because there are two distinct conditions. One would work but
  would require `notify_all` and wake spurious waiters — the same reasoning as C.
- `?` on `push` means an allocation failure propagates *with the lock released
  automatically*.

### Lab 81.3 — `LockedBy` for the many-objects-one-lock pattern

```rust
// SPDX-License-Identifier: GPL-2.0
use kernel::prelude::*;
use kernel::sync::{new_mutex, Arc, LockedBy, Mutex};

struct ControllerInner {
    total_active: u32,
}

#[pin_data]
struct Controller {
    #[pin] inner: Mutex<ControllerInner>,
}

struct Port {
    index: u32,
    /// Protected by the *controller's* lock, not by one of its own.
    /// The type says so, and there is no way to read it without proof.
    active: LockedBy<bool, ControllerInner>,
}

impl Port {
    fn new(index: u32, ctrl: &Arc<Controller>) -> Self {
        Port {
            index,
            active: LockedBy::new(&ctrl.inner, false),
        }
    }

    fn activate(&self, ctrl: &Controller) -> Result {
        let mut guard = ctrl.inner.lock();
        // Presenting the guard is the proof. Without it: compile error.
        let active = self.active.access_mut(&mut guard);
        if *active {
            return Err(EBUSY);
        }
        *active = true;
        guard.total_active += 1;
        Ok(())
    }

    fn is_active(&self, ctrl: &Controller) -> bool {
        let guard = ctrl.inner.lock();
        *self.active.access(&guard)
    }
}

// The attempt that fails:
// fn peek(port: &Port) -> bool {
//     *port.active          // error: no Deref on LockedBy
// }
```

Compare with the C version, where `port->active` is a plain `bool` with a comment saying
`/* protected by ctrl->lock */`, and nothing at all checks it.

### Lab 81.4 — Prove `Send`/`Sync` do real work

```rust
// SPDX-License-Identifier: GPL-2.0
//! Each block below should FAIL to compile. Uncomment one at a time.

use kernel::prelude::*;
use kernel::sync::{Arc, Mutex};
use core::cell::Cell;

fn requires_send<T: Send>(_: T) {}
fn requires_sync<T: Sync>(_: &T) {}

fn demo() {
    // 1. Cell is Send but NOT Sync -- unsynchronized interior mutability.
    let c = Cell::new(0u32);
    requires_send(c);                 // ok
    // requires_sync(&Cell::new(0u32));
    // error: `Cell<u32>` cannot be shared between threads safely

    // 2. Raw pointers are neither.
    struct HoldsPtr(*mut u8);
    // requires_send(HoldsPtr(core::ptr::null_mut()));
    // error: `*mut u8` cannot be sent between threads safely
    //   ==> ANY wrapper over a C object must justify Send/Sync explicitly.
    //       This is the forcing function.

    // 3. Arc<T> requires T: Send + Sync, because clones may go anywhere.
    // let a = Arc::new(Cell::new(0u32), GFP_KERNEL).unwrap();
    // requires_send(a);
    // error: `Cell<u32>` cannot be shared between threads safely
    //   ==> Arc<Cell<T>> is rejected; Arc<Mutex<T>> is accepted.

    // 4. The fix:
    // let a = Arc::pin_init(new_mutex!(0u32, "demo"), GFP_KERNEL).unwrap();
    // requires_send(a);              // ok: Mutex supplies the synchronization
}
```

Then write the `unsafe impl` you would need to make a C-pointer wrapper `Send`, and the
`// SAFETY:` justification:

```rust
struct Registration { ptr: *mut bindings::some_c_object }

// SAFETY: `some_c_object` is internally synchronized by the subsystem's
// own lock (see drivers/foo/core.c, `foo_lock`), and no operation we
// perform on it requires being on a particular CPU. Therefore moving a
// `Registration` between threads is sound.
unsafe impl Send for Registration {}

// SAFETY: all methods taking `&self` only perform operations the C code
// documents as safe to call concurrently (`foo_read`, `foo_stat`).
unsafe impl Sync for Registration {}
```

**Note what just happened:** to write this type at all you were forced to state, in reviewable
prose, the concurrency contract of a C API. That statement did not previously exist anywhere.
That is the real value, and it is worth saying explicitly in an interview.

### Lab 81.5 — Measure that the abstractions are free

```bash
cd $KDIR

# 1. Build a module with a SpinLock-protected counter and one with a raw
#    binding call to spin_lock/spin_unlock. Compare the disassembly.
objdump -d --no-show-raw-insn my_rust_lock.ko | \
	sed -n '/<increment>:/,/ret/p'

# Expect: call _raw_spin_lock / inc / call _raw_spin_unlock.
# The Guard, the Deref, and the Drop have all vanished.

# 2. Check code size of the two forms:
size my_rust_lock.ko my_c_lock.ko

# 3. Benchmark contention identically in both:
#    (a lockless-increment loop under N threads, measured with perf)
sudo perf stat -e cycles,instructions -- ./bench_lock
```

Then examine where the abstraction is *not* free: `Arc::clone` is an atomic increment (as
`kref_get` is), `LockedBy::access` does a runtime address comparison (a branch), and
`try_pin_init!` generates the unwind code. **Be able to name those three.** "Zero-cost
abstraction" is a claim about the *common* case, not a universal one, and treating it as
universal is a red flag.

### Lab 81.6 — Find what lockdep still catches

```rust
// Deliberate lock inversion. This COMPILES -- Rust does not track order.
fn path_a(x: &Resources) {
    let _a = x.lock_a.lock();
    let _b = x.lock_b.lock();
}
fn path_b(x: &Resources) {
    let _b = x.lock_b.lock();
    let _a = x.lock_a.lock();     // inversion. Compiles fine.
}
```

```bash
# Build with lockdep and run both paths:
./scripts/config --enable PROVE_LOCKING --enable DEBUG_LOCK_ALLOC
make LLVM=1 -j$(nproc)
# Load the module, exercise both paths, read dmesg:
dmesg | grep -A40 'possible circular locking dependency'
```

You should get the identical report you would get from C (→
`reference/debugging-scenarios.md` §3), naming your `new_mutex!` strings as the lock classes.

**This lab is the honest counterweight to the rest of the chapter.** Rust eliminated the data
race; it did nothing about the deadlock. Being able to demonstrate both facts in one session
is a strong thing to have done.

Similarly, verify that sleeping in atomic context is still a runtime bug:

```rust
let _g = spinlock.lock();      // atomic context now
let b = KBox::new(0u8, GFP_KERNEL)?;   // may sleep. COMPILES.
// -> "BUG: sleeping function called from invalid context" at runtime
```

Enable `CONFIG_DEBUG_ATOMIC_SLEEP` and confirm.

---

## 3. Mastery drills

1. Implement a `RwSemaphore<T>` abstraction over `struct rw_semaphore`, with separate read
   and write guards. Get the `Send`/`Sync` bounds right and justify them in writing. Compare
   your bounds with `Lock<T, B>`'s and explain the difference.

2. Implement a `SeqLock<T>` abstraction. Explain why the reader API cannot simply return
   `&T` (hint: the data may be torn mid-read), and design an API that makes the retry loop
   unavoidable.

3. Write a benchmark comparing Rust `SpinLock<u64>` against C `spinlock_t` + `u64` at 1, 4,
   16, and all cores. Confirm they are identical, or explain any difference.

4. Read `rust/kernel/sync/lock.rs` in full and write out the safety argument for every
   `unsafe` block and every `unsafe impl`. Then check your reasoning against the comments in
   the source.

5. Design a type-level encoding of "this function may sleep." Sketch it (a marker trait? a
   context token passed by value?), then explain why it has not been adopted in the kernel
   and what it would cost.

6. Port a C driver's complete locking scheme to Rust types. Document every place where the C
   version relied on a comment and the Rust version relies on a type — and every place where
   it still relies on a comment.

7. Build a lock hierarchy of three locks and demonstrate lockdep catching an inversion in
   Rust code. Confirm the report names your `new_*!` strings and explain why choosing good
   names matters.

8. Implement `LockedBy` yourself, including the runtime owner check. Explain what the check
   costs and when it can be elided.

9. Investigate the state of atomics in kernel Rust: read `rust/kernel/sync/atomic.rs` (or the
   current equivalent) and the LKML discussion about LKMM versus the Rust memory model.
   Summarize both positions and state which you find more persuasive, with reasons.

10. Write a `Work<T>` based driver where the work item must be cancelled on teardown. Show
    that the `Arc` reference makes the use-after-free impossible, then determine what happens
    if the work is still running when the module unloads, and what `cancel_work_sync` maps to.

11. Explore `PollCondVar` and implement a char device's `poll()`. Explain how the wait-queue
    lifetime problem (the queue must outlive every waiter) is handled.

12. Write the two-page memo: "Which classes of concurrency bug does Rust eliminate in the
    kernel, which does it not, and what should we keep in CI as a result?" This is exactly
    the document a principal engineer would be asked for.

---

## 4. Further reading

**Kernel documentation and code**
- `rust/kernel/sync/lock.rs` — read completely; it is the best-documented file in the crate
- `rust/kernel/sync/condvar.rs` — the wait-queue abstraction, and the lost-wakeup reasoning
- `rust/kernel/sync/arc.rs` — `refcount_t` integration and `UniqueArc`
- `rust/kernel/workqueue.rs` — an unusually thorough module doc explaining the ownership model
- `Documentation/locking/` — the C rules still apply in full; nothing here supersedes them
- `tools/memory-model/` — LKMM, litmus tests, `herd7`

**Books**
- **Mara Bos, *Rust Atomics and Locks*** — free online, and the best explanation of
  `Send`/`Sync`, the memory model, and how locks are built. Chapters 1–4 are directly
  applicable; read them before reviewing any Rust locking patch
- McKenney, *Is Parallel Programming Hard…* — the kernel's concurrency bible; unchanged by
  the language
- Herlihy & Shavit, *The Art of Multiprocessor Programming* — the theory

**Papers and articles**
- Jung et al., "RustBelt," *POPL*, 2018 — the formal soundness proof, including how
  `unsafe`-implemented libraries like `Mutex` are shown to be safe
- Jung's "Understanding and Evolving the Rust Programming Language" (thesis) — the Stacked
  Borrows model, relevant when reasoning about raw pointers in abstractions
- LWN: "Rust and the kernel memory model" — the LKMM-versus-Rust-atomics discussion
- LWN: "Concurrency in Rust for the kernel" and the Kangrejos coverage

**Reference**
- The `core::marker` docs for `Send`/`Sync` — short and precise
- `Documentation/rust/coding-guidelines.rst` — the `unsafe impl Send/Sync` documentation
  requirement
- `rust/kernel/sync/locked_by.rs` — small, and a good model for designing your own
  proof-carrying API

→ Next: [82-rust-drivers-1.md](82-rust-drivers-1.md)

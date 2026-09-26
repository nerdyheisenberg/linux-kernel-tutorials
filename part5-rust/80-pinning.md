# Chapter 80 — Pinning and `pin-init`

> The hardest concept in kernel Rust, and the one that most clearly exposes the mismatch
> between Rust's model and the kernel's. Rust assumes values can be moved freely — a move is
> a `memcpy` plus a promise. The kernel is built from self-referential structures: a
> `list_head` whose `prev`/`next` point at itself when empty, a `spinlock_t` that a lockdep
> key identifies by address, a `struct device` whose address C stores in a dozen places.
> **Moving any of those is instant corruption.** `Pin` is how Rust says "this must not move,"
> and `pin-init` is how you build such a thing without ever having it on the stack.

---

## Theory & First Principles

### T.0 — Start here: a structure that must never move

Here is Ch. 09's intrusive list, drawn as memory:

```
   struct my_obj @ 0x1000                struct my_obj @ 0x2000
   +-------------------+                 +-------------------+
   | data              |                 | data              |
   | list.next ------------------------> | list.next ---> ... |
   | list.prev <-----------------------  | list.prev         |
   +-------------------+                 +-------------------+
            ^                                     |
            +-------------------------------------+

   The NEIGHBOURS store this object's ADDRESS.
   Move the object to 0x3000 and both neighbours now point at freed memory.
```

The same is true of `struct mutex` (lockdep identifies a lock by its address), `struct
work_struct`, `struct timer_list`, `struct completion`, and essentially every kernel object
that participates in a kernel data structure. **"This object's address is part of its
identity" is the normal case in kernel C.**

**Now the collision.** In Rust, *moving is the default* (Ch. 78 §T.0):

```rust
let a = make_thing();   // built at some stack address
let b = a;              // MOVED -- bitwise copied to a new address,
                        //          and `a` is no longer usable
vec.push(b);            // MOVED again, into the vector's heap buffer
```

The compiler moves values freely because for ordinary Rust types, a bitwise move is always
valid — nothing stores its own address. **Kernel types break that assumption, so the compiler
must be told.**

**`Pin<P>` is how you tell it:**

```rust
Pin<&mut T>       // "this T will never move again, for the rest of its life"
Pin<KBox<T>>      // an owned, heap-allocated, never-moving T
```

`Pin` is not a runtime thing — it is a **type-level promise**: once a value is pinned, you can
no longer obtain a `&mut T` (which would let you `mem::swap` it away) unless `T: Unpin`.

**But pinning alone creates a chicken-and-egg problem**, and this is the part that makes
kernel Rust look unusual:

```rust
// The obvious thing does not work:
let m = Mutex::new(inner);    // constructed HERE, at this stack address
let m = KBox::pin(m)?;        // then MOVED to the heap -- too late!
                              // lockdep already registered the stack address.
```

You cannot build a self-referential object somewhere and then move it into place; it must be
built **at its final address**. Hence `pin-init`:

```rust
#[pin_data]
struct MyDriver {
    #[pin] lock: Mutex<Inner>,      // must be initialized in place
           count: u32,              // ordinary; can be moved
}

impl MyDriver {
    fn new() -> impl PinInit<Self, Error> {
        try_pin_init!(Self {
            lock <- new_mutex!(Inner::default(), "my_driver::lock"),
            count: 0,
        })
    }
}
// `impl PinInit` is a RECIPE, not a value. The allocation happens first;
// the recipe then runs AT the final address. Nothing is ever moved.
```

**The idea generalizes well beyond Rust, and it is the reason to read this chapter carefully
even if you never write kernel Rust:**

> **Return a plan for constructing the object, not the object.** The caller allocates, then
> executes the plan in place. This is also why C kernel APIs look like
> `foo_init(&obj)` rather than `obj = foo_create()` — the C convention exists for exactly the
> same reason, and `pin-init` is that convention made explicit and type-checked, with
> automatic cleanup of already-initialized fields if a later field's initialization fails.

That last clause is the part C gets wrong constantly: if field 4 of 6 fails to initialize,
fields 1–3 must be torn down in reverse order. In C this is the `goto err_*` ladder that
Ch. 28 §T.0 counted bugs in. `try_pin_init!` generates it correctly, every time.

**The mental model to hold:** `Unpin` means "moving me is harmless" and is the default for
ordinary types; the kernel's self-referential types are `!Unpin`, and `Pin` is the marker
that carries that fact through the type system so the compiler can enforce it.

```bash
sed -n '1,120p' rust/kernel/init.rs
grep -rn 'pin_init\|#\[pin\]' rust/kernel/ | head -20
grep -rn 'PhantomPinned' rust/ | head
```

---

### T.1 — Why moving is the default, and why that breaks

```rust
let a = Foo { .. };
let b = a;          // a MOVE: memcpy the bytes, forget the source
```

For an ordinary value this is sound: nothing pointed at the old address. The compiler relies
on this everywhere — returning a value from a function, pushing into a `Vec` (which may
realloc and move every element), passing by value.

Now consider the kernel's most common data structure:

```c
struct list_head { struct list_head *next, *prev; };

static inline void INIT_LIST_HEAD(struct list_head *list)
{
	list->next = list;     /* points to ITSELF */
	list->prev = list;
}
```

`memcpy` that to a new address and `next`/`prev` still point at the **old** address. The list
is now corrupt, silently, and the corruption surfaces later in an unrelated place. Every
intrusive kernel structure has this property:

| Structure | Why it cannot move |
|---|---|
| `struct list_head` | self-pointers when empty; neighbours point back at it |
| `struct rb_node` | parent and children point at it |
| `struct mutex` / `spinlock_t` | wait list is a `list_head`; lockdep keys are addresses |
| `struct device` | the driver core, sysfs, and the parent all hold pointers |
| `wait_queue_head_t` | waiters hold a pointer |
| `struct hlist_node` | `pprev` points at the *previous element's next pointer* |
| Anything registered with C | C stored the address |

So Rust needs a way to say: **"this value's address is part of its identity."**

### T.2 — `Pin<P>`: a wrapper that withholds `&mut`

```rust
pub struct Pin<P> { pointer: P }
```

`Pin<P>` where `P` is a pointer type (`&mut T`, `KBox<T>`, `Arc<T>`) makes a single promise:

> Once a value is pinned, it will not move until it is dropped.

The enforcement mechanism is elegantly minimal: **`Pin<P>` does not give you `&mut T`.** And
without `&mut T`, you cannot call `mem::swap`, `mem::replace`, or `mem::take` — which are the
only safe ways to move a value out of a place.

```rust
let mut boxed: Pin<KBox<Foo>> = KBox::pin_init(..., GFP_KERNEL)?;
let r: &Foo = &*boxed;               // fine: shared reference
// let m: &mut Foo = &mut *boxed;    // NOT available
let m: Pin<&mut Foo> = boxed.as_mut(); // you get a PINNED mutable reference
```

To get `&mut T` out of a `Pin<&mut T>` you need `unsafe { Pin::get_unchecked_mut() }`, whose
safety contract is "you must not move the value." So the obligation is explicit and
auditable.

### T.3 — `Unpin`: the escape hatch, and why `PhantomPinned` exists

Most types do not care about their address, so pinning them should cost nothing. Hence:

```rust
pub auto trait Unpin {}     // auto-implemented for almost everything
```

If `T: Unpin`, then `Pin<&mut T>` **does** give you `&mut T` safely, because moving `T` is
harmless. So `Pin` is a no-op for `u32`, `KVec<u8>`, and ordinary structs.

To opt *out* — to say "my address matters" — you include a field that is not `Unpin`:

```rust
use core::marker::PhantomPinned;

struct SelfRef {
    data: u64,
    _pin: PhantomPinned,     // zero-sized; makes SelfRef !Unpin
}
```

`PhantomPinned` is zero-sized and exists purely to poison the auto-trait. This is why you
saw it inside `Opaque<T>` in Ch. 79 — every C struct Rust holds is `!Unpin` by construction.

**The summary sentence:** `Pin` is only meaningful for `!Unpin` types, and a type becomes
`!Unpin` by containing `PhantomPinned` (directly or transitively via `Opaque`).

### T.4 — The initialization problem, which is the real difficulty

`Pin` says "do not move it *after* it exists." But the ordinary way to create a value is:

```rust
let m = Mutex::new(data);              // construct on the STACK
let b = KBox::new(m, GFP_KERNEL)?;     // MOVE it into the heap
```

That move happens *before* pinning, and for a `struct mutex` it is already fatal —
`__mutex_init()` has written a self-pointer and registered a lockdep key at the stack
address.

Worse, a kernel object can be large. Constructing a 4 KiB struct on a 16 KiB kernel stack and
then moving it is an overflow waiting to happen.

**What you actually need is C's pattern:** allocate the memory first, then initialize it *in
place*:

```c
d = kzalloc(sizeof(*d), GFP_KERNEL);
mutex_init(&d->lock);           /* initialize IN the final location */
INIT_LIST_HEAD(&d->entries);
```

Rust has no native syntax for this. `pin-init` supplies it.

### T.5 — `pin-init`: in-place construction

```rust
use kernel::prelude::*;
use kernel::sync::{new_mutex, Mutex};

#[pin_data]
struct Device {
    id: u32,
    #[pin]
    lock: Mutex<State>,
    #[pin]
    entries: ListHead,
}

impl Device {
    fn new(id: u32) -> impl PinInit<Self, Error> {
        try_pin_init!(Self {
            id,
            lock  <- new_mutex!(State::default(), "Device::lock"),
            entries <- ListHead::new(),
        })
    }
}

// Allocate, then initialize IN PLACE. Nothing is ever on the stack.
let dev: Pin<KBox<Device>> = KBox::pin_init(Device::new(7), GFP_KERNEL)?;
```

Read the syntax:

| Token | Means |
|---|---|
| `#[pin_data]` | Generates the machinery; marks this as a pin-initializable struct |
| `#[pin]` on a field | This field is structurally pinned — pinning the struct pins it |
| `id,` | An ordinary field: just a value |
| `lock <- initializer` | **In-place initialization.** The initializer writes directly into the final address |
| `pin_init!` | Build an infallible initializer |
| `try_pin_init!` | Build a fallible one (returns `Result`); fields may fail |
| `PinInit<T, E>` | A *recipe* for constructing a `T` at a given address |

The key abstraction: **`impl PinInit<T>` is not a `T`. It is a closure-like value that knows
how to build a `T` at an address you give it later.** So the construction is deferred until
the memory exists.

```rust
pub unsafe trait PinInit<T: ?Sized, E = Infallible>: Sized {
    /// # Safety
    /// `slot` must be valid, properly aligned, uninitialized memory for T,
    /// and must remain pinned (not move) until dropped.
    unsafe fn __pinned_init(self, slot: *mut T) -> Result<(), E>;
}
```

`KBox::pin_init` allocates, calls `__pinned_init` on the allocation, and — crucially — **if
initialization fails partway, it drops the fields that were already initialized, in reverse
order, then frees the allocation.** That is the `goto` ladder again, generated by a macro.

### T.6 — The `<-` operator and failure semantics

```rust
try_pin_init!(Self {
    a <- init_a(),        // if this fails: nothing to unwind, just return Err
    b <- init_b(),        // if this fails: drop a, return Err
    c <- init_c(),        // if this fails: drop b, drop a, return Err
    d: plain_value,
}? Error)
```

This is the single most important property to understand and to say in review: **`pin-init`
generates the correct partial-teardown for every failure point.** In C this is the
`goto err_c; err_b:; err_a:;` ladder, and getting it wrong — freeing in the wrong order,
skipping a label, adding a resource without adding a label — is a classic bug class.

`pin_init!` (infallible) versus `try_pin_init!` (fallible) versus `init!`/`try_init!` (the
unpinned variants) is a four-way choice; use the `try_` pinned form for anything containing a
lock or a C struct.

### T.7 — Structural vs non-structural pinning

A subtle but reviewable distinction.

- A field marked `#[pin]` is **structurally pinned**: if the struct is pinned, that field is
  pinned, and you can only ever get `Pin<&mut Field>`.
- A field *not* marked is **not** structurally pinned: even in a pinned struct, you can get
  `&mut Field` and move it.

```rust
#[pin_data]
struct Dev {
    #[pin] lock: Mutex<u32>,   // pinned: cannot be moved out
    counter: u64,              // not pinned: freely mutable
    buffer: KVec<u8>,          // not pinned: can be replaced wholesale
}
```

The rule: **mark `#[pin]` exactly those fields whose address matters.** Marking too much
makes the type painful to use; marking too little is unsound if the field is self-referential.
Getting this wrong is a real review finding.

### T.8 — `Arc::pin_init` and the shared case

The common driver shape is a shared, locked state object:

```rust
let state: Arc<Mutex<State>> = Arc::pin_init(
    new_mutex!(State::new(), "mydrv::state"),
    GFP_KERNEL,
)?;
```

Note `Arc<Mutex<T>>`, not `Pin<Arc<Mutex<T>>>`. `Arc` never exposes `&mut` to its contents
(it cannot — it is shared), so an `Arc<T>` already guarantees `T` will not move. **`Arc` is
implicitly a pinning container**, which is why you see `Arc::pin_init` producing a plain
`Arc`. This is a frequent point of confusion and worth being able to explain.

`UniqueArc<T>` is the construction-time form: refcount provably 1, so `&mut` is available,
and it converts into `Arc` when you publish it. That is the type-level version of the kernel
C pattern "initialize before making visible."

### T.9 — The lockdep connection, and why the name matters

```rust
new_mutex!(value, "Device::lock")
```

That string is not decoration. It becomes the **lockdep class name**. Every kernel lock
needs a static `lock_class_key` so lockdep can build the lock-order graph (Ch. 14 §T.7), and
because the key must be a distinct static per lock *site*, the macro creates one.

This is why `Mutex::new()` is not an ordinary function — it needs a macro to generate the
static key and to capture the name at the call site. Same reason `DEFINE_MUTEX` and
`mutex_init` are macros in C.

Practical consequence: **if you see `new_mutex!` with a duplicated or meaningless name in
review, flag it** — lockdep reports will be unreadable.

### T.10 — The wider lesson

Pinning is where Rust's model and the kernel's model genuinely disagree, and the resolution
is instructive:

- Rust could not simply say "kernel types are special." It needed a *type-level* way to
  express address-stability, usable by ordinary library code. `Pin` is that.
- `Pin` alone was insufficient, because it says nothing about *construction*. `pin-init` is
  the kernel community's contribution — and it has since been extracted as a standalone crate
  useful outside the kernel.
- The macro generates the error-unwinding that C writes by hand. **The abstraction absorbed a
  bug class** — the same pattern as `devm_` (Ch. 28) and folios (Ch. 52), one more time.

When asked "what was the hardest part of putting Rust in the kernel?", pinning and
initialization is the strongest technical answer, and the fact that it produced a reusable
crate is the strongest evidence that it was solved properly rather than hacked around.

---

## 1. Internals

### Source map

| Path | Contents |
|---|---|
| `rust/pin-init/` | The standalone `pin-init` crate (moved out of `kernel` in 6.13-ish) |
| `rust/pin-init/src/lib.rs` | `PinInit`, `Init`, `pin_init!`, `try_pin_init!`, docs |
| `rust/pin-init/src/macros.rs` | The macro expansion — heavy reading, but definitive |
| `rust/kernel/init.rs` | Kernel glue: `InPlaceInit` for `KBox`, `Arc`, `UniqueArc` |
| `rust/kernel/sync/lock.rs` | `Lock<T, B>`, `Guard` — the generic lock over a backend |
| `rust/kernel/sync/lock/mutex.rs` | `Mutex`, `new_mutex!` |
| `rust/kernel/sync/lock/spinlock.rs` | `SpinLock`, `new_spinlock!` |
| `rust/kernel/types.rs` | `Opaque` — `PhantomPinned` in action |
| `rust/kernel/list.rs` | `List`, `ListLinks` — the pinned intrusive list |
| `core::pin` module docs | The canonical explanation of `Pin`; genuinely good |

### What `#[pin_data]` generates, sketched

```rust
#[pin_data]
struct Dev { #[pin] lock: Mutex<u32>, id: u32 }
```

expands to approximately:

```rust
struct Dev { lock: Mutex<u32>, id: u32 }

// A "projection" type letting the initializer write each field in place:
impl Dev {
    fn __pin_data() -> __ThePinData { ... }
}
struct __ThePinData;
impl __ThePinData {
    // For #[pin] fields: accepts a PinInit
    unsafe fn lock<E>(self, slot: *mut Mutex<u32>, init: impl PinInit<Mutex<u32>, E>)
        -> Result<(), E> { unsafe { init.__pinned_init(slot) } }
    // For plain fields: accepts a value
    unsafe fn id<E>(self, slot: *mut u32, init: impl Init<u32, E>)
        -> Result<(), E> { unsafe { init.__init(slot) } }
}

// Marks the type !Unpin if any field is #[pin]:
impl !Unpin for Dev {}   // conceptually; done via PhantomPinned
```

And `try_pin_init!` expands to a `PinInit` implementation that calls those in order, with a
`ScopeGuard`-style drop of already-initialized fields on failure. **Reading the expansion
once with `cargo expand` (or the kernel's `make rust-analyzer` + IDE) removes all the
mystery.**

### `Lock<T, B>`: one lock, many backends

```rust
pub struct Lock<T: ?Sized, B: Backend> {
    state: Opaque<B::State>,      // the C lock (struct mutex / spinlock_t)
    _pin: PhantomPinned,
    data: UnsafeCell<T>,          // the protected data, INSIDE the lock
}

pub struct Guard<'a, T: ?Sized, B: Backend> { ... }

impl<T, B: Backend> Lock<T, B> {
    pub fn lock(&self) -> Guard<'_, T, B> { ... }
}

impl<T, B: Backend> Deref for Guard<'_, T, B> { type Target = T; ... }
impl<T, B: Backend> DerefMut for Guard<'_, T, B> { ... }
impl<T, B: Backend> Drop for Guard<'_, T, B> { fn drop(&mut self) { B::unlock(...) } }
```

Three properties fall out of this layout, and they are the whole argument for Rust locking:

1. **The data is inside the lock.** There is no way to reach `T` without going through
   `lock()`. In C, `struct foo { spinlock_t lock; int counter; }` puts them side by side and
   nothing stops you touching `counter` directly.
2. **Unlock is `Drop`.** Every path unlocks, including early returns and `?`.
3. **The guard's lifetime is tied to the lock.** You cannot return a `&T` that outlives the
   guard — the borrow checker rejects it. That is "do not use the data after unlocking,"
   enforced.

`B: Backend` is `MutexBackend` or `SpinLockBackend`, selected at compile time, so there is no
indirection. Ch. 81 develops this.

---

## 2. Practice

### Lab 80.1 — Feel the problem before the solution

```rust
// SPDX-License-Identifier: GPL-2.0
//! Why Pin exists: a self-referential struct, broken by a move.
//! This is a THOUGHT EXPERIMENT -- the unsafe here is deliberately unsound.

use kernel::prelude::*;
use core::ptr;

struct SelfRef {
    value: u64,
    /// Points at `self.value`. If the struct moves, this dangles.
    ptr_to_value: *const u64,
}

impl SelfRef {
    /// UNSOUND on purpose: constructs on the stack, so the pointer is
    /// correct only until the value is moved.
    fn new(value: u64) -> Self {
        let mut s = SelfRef { value, ptr_to_value: ptr::null() };
        s.ptr_to_value = &s.value;      // points into the STACK temporary
        s                               // ... and then we MOVE it. Broken.
    }
}

fn demonstrate() {
    let a = SelfRef::new(42);
    // SAFETY: NONE. This is the bug.
    let via_ptr = unsafe { *a.ptr_to_value };
    pr_info!("direct = {}  via pointer = {}\n", a.value, via_ptr);
    //   Frequently prints garbage, or the right answer by luck.

    let b = a;      // another move
    let via_ptr2 = unsafe { *b.ptr_to_value };
    pr_info!("after move: direct = {}  via pointer = {}\n",
             b.value, via_ptr2);
    //   b.ptr_to_value still points at a's old address.
}
```

Now substitute `list_head` for `ptr_to_value` and you have the kernel's situation exactly.
The C equivalent compiles without complaint, runs, and corrupts a list.

Then fix it the Rust way:

```rust
use core::marker::PhantomPinned;
use core::pin::Pin;

struct SelfRefFixed {
    value: u64,
    ptr_to_value: *const u64,
    _pin: PhantomPinned,        // now !Unpin: pinning is meaningful
}

impl SelfRefFixed {
    fn new(value: u64) -> impl PinInit<Self> {
        pin_init!(Self {
            value,
            ptr_to_value: ptr::null(),
            _pin: PhantomPinned,
        })
    }

    /// Only callable on a pinned value, so the address is stable.
    fn setup(self: Pin<&mut Self>) {
        // SAFETY: we do not move out of the mutable reference; we only
        // write a pointer field.
        let this = unsafe { self.get_unchecked_mut() };
        this.ptr_to_value = &this.value;
    }
}

// Construction: allocate first, initialize in place, never on the stack.
let mut s = KBox::pin_init(SelfRefFixed::new(42), GFP_KERNEL)?;
s.as_mut().setup();
// There is now NO WAY to move it: Pin<KBox<_>> does not give &mut.
```

### Lab 80.2 — A realistic pinned driver structure

```rust
// SPDX-License-Identifier: GPL-2.0
//! The shape almost every Rust driver has.

use kernel::prelude::*;
use kernel::sync::{new_mutex, new_spinlock, Arc, Mutex, SpinLock};
use kernel::list::{List, ListArc, ListLinks};

module! {
    type: PinDemo,
    name: "pin_demo",
    author: "You",
    description: "pin-init in a realistic driver shape",
    license: "GPL",
}

#[derive(Default)]
struct Stats {
    submitted: u64,
    completed: u64,
    errors: u64,
}

/// An entry on the driver's intrusive list. It MUST be pinned: the list
/// links point at it, and its neighbours point back.
#[pin_data]
struct Request {
    id: u64,
    len: usize,
    #[pin]
    links: ListLinks<0>,
}

impl Request {
    fn new(id: u64, len: usize) -> impl PinInit<Self> {
        pin_init!(Self {
            id,
            len,
            links <- ListLinks::new(),
        })
    }
}

kernel::list::impl_has_list_links! {
    impl HasListLinks<0> for Request { self.links }
}
kernel::list::impl_list_arc_safe! {
    impl ListArcSafe<0> for Request { untracked; }
}
kernel::list::impl_list_item! {
    impl ListItem<0> for Request { using ListLinks; }
}

/// The driver's shared state. Note the mixture:
///  - `id` is plain data
///  - `stats` is behind a spinlock (touched from IRQ context)
///  - `pending` is behind a mutex (touched from process context)
/// Both locks are #[pin] because a C `struct mutex`/`spinlock_t` cannot move.
#[pin_data]
struct DriverState {
    id: u32,
    #[pin]
    stats: SpinLock<Stats>,
    #[pin]
    pending: Mutex<List<Request, 0>>,
}

impl DriverState {
    fn new(id: u32) -> impl PinInit<Self, Error> {
        try_pin_init!(Self {
            id,
            stats   <- new_spinlock!(Stats::default(), "pin_demo::stats"),
            pending <- new_mutex!(List::new(),         "pin_demo::pending"),
        })
    }

    fn submit(&self, req: ListArc<Request, 0>) {
        {
            let mut s = self.stats.lock();     // spinlock: atomic context ok
            s.submitted += 1;
        }                                      // unlock here
        let mut p = self.pending.lock();       // mutex: may sleep
        p.push_back(req);
    }

    fn complete_one(&self) -> Option<u64> {
        let mut p = self.pending.lock();
        let req = p.pop_front()?;
        let id = req.id;
        drop(p);                               // explicit early unlock

        let mut s = self.stats.lock();
        s.completed += 1;
        Some(id)
    }
}

struct PinDemo {
    state: Arc<DriverState>,
}

impl kernel::Module for PinDemo {
    fn init(_m: &'static ThisModule) -> Result<Self> {
        // Allocate first, initialize in place. Neither lock is ever
        // constructed at a temporary address.
        let state = Arc::pin_init(DriverState::new(1), GFP_KERNEL)?;

        for i in 0..4u64 {
            let req = ListArc::pin_init(Request::new(i, 4096), GFP_KERNEL)?;
            state.submit(req);
        }

        while let Some(id) = state.complete_one() {
            pr_info!("completed request {}\n", id);
        }

        let s = state.stats.lock();
        pr_info!("submitted={} completed={} errors={}\n",
                 s.submitted, s.completed, s.errors);
        drop(s);

        Ok(PinDemo { state })
    }
}

impl Drop for PinDemo {
    fn drop(&mut self) {
        pr_info!("pin_demo: unloading (id {})\n", self.state.id);
    }
}
```

Points to extract:
- `Arc<DriverState>` has no `Pin` wrapper — §T.8.
- Two locks with different backends, chosen by which contexts touch the data.
- `drop(p)` for an explicit early unlock, which is the Rust spelling of "narrow the critical
  section."
- The intrusive list is pinned; the boilerplate `impl_has_list_links!` family is how Rust
  expresses C's `container_of` relationship (Ch. 09).

### Lab 80.3 — Watch the failure unwinding

```rust
// SPDX-License-Identifier: GPL-2.0
//! Prove that pin-init generates correct partial teardown.

use kernel::prelude::*;

struct Noisy(&'static str);

impl Noisy {
    fn new(name: &'static str, fail: bool) -> impl PinInit<Self, Error> {
        pin_init::pin_init_from_closure(move |slot: *mut Self| {
            if fail {
                pr_info!("  init {}: FAILING\n", name);
                return Err(ENOMEM);
            }
            pr_info!("  init {}: ok\n", name);
            // SAFETY: `slot` is valid uninitialized memory for Self, per
            // the PinInit contract.
            unsafe { slot.write(Noisy(name)) };
            Ok(())
        })
    }
}

impl Drop for Noisy {
    fn drop(&mut self) {
        pr_info!("  drop {}\n", self.0);
    }
}

#[pin_data]
struct Composite {
    #[pin] a: Noisy,
    #[pin] b: Noisy,
    #[pin] c: Noisy,
}

impl Composite {
    fn new(fail_at: usize) -> impl PinInit<Self, Error> {
        try_pin_init!(Self {
            a <- Noisy::new("a", fail_at == 0),
            b <- Noisy::new("b", fail_at == 1),
            c <- Noisy::new("c", fail_at == 2),
        })
    }
}

fn demo() -> Result {
    for fail_at in 0..4 {
        pr_info!("--- failing at index {} ---\n", fail_at);
        match KBox::pin_init(Composite::new(fail_at), GFP_KERNEL) {
            Ok(v)  => { pr_info!("  built ok\n"); drop(v); }
            Err(e) => pr_info!("  failed: {}\n", e.to_errno()),
        }
    }
    Ok(())
}
```

Expected output:

```
--- failing at index 0 ---
  init a: FAILING
  failed: -12
--- failing at index 1 ---
  init a: ok
  init b: FAILING
  drop a                 <-- exactly the right unwind, generated
  failed: -12
--- failing at index 2 ---
  init a: ok
  init b: ok
  init c: FAILING
  drop b                 <-- reverse order
  drop a
  failed: -12
--- failing at index 3 ---
  init a: ok / b: ok / c: ok
  built ok
  drop c / drop b / drop a
```

**Write the C equivalent and its `goto` ladder, then have a colleague introduce one subtle
bug into it (a swapped label, a missing one) and see how long it takes to spot.** That
exercise is the entire argument for this machinery.

### Lab 80.4 — Diagnose the common `pin-init` errors

Each of these is a real error you will hit. Predict the message, then compile.

```rust
// 1. Forgot #[pin] on a field that needs it
#[pin_data]
struct Bad1 { lock: Mutex<u32> }          // no #[pin]
// try_pin_init!(Self { lock <- new_mutex!(0, "x") })
// error: cannot use `<-` on a field that is not `#[pin]`

// 2. Used `:` where `<-` is needed
#[pin_data]
struct Bad2 { #[pin] lock: Mutex<u32> }
// pin_init!(Self { lock: Mutex::new(0) })
// error: `Mutex::new` does not exist / expected PinInit
//   -> locks have NO plain constructor, by design. You MUST use new_mutex!.

// 3. Tried to move a pinned value
// let b: Pin<KBox<Dev>> = ...;
// let inner: Dev = *b;
// error[E0507]: cannot move out of dereference of `Pin<KBox<Dev>>`

// 4. Tried to get &mut from a Pin
// let m: &mut Dev = &mut *b;
// error: the trait bound `Dev: Unpin` is not satisfied
//   -> this is the ENTIRE mechanism working. Use b.as_mut() for Pin<&mut Dev>.

// 5. Mixed fallible and infallible
// pin_init!(Self { v <- thing_that_returns_Result() })
// error: mismatched types, PinInit<_, Error> vs PinInit<_, Infallible>
//   -> use try_pin_init! and annotate the error type: `}? Error)`

// 6. Stack-constructed a lock
// let m = Mutex { .. };
// error: cannot construct; fields are private
//   -> deliberate. Locks can only be built via new_mutex! into a slot.
```

**Error 4 is the one worth dwelling on.** `the trait bound Dev: Unpin is not satisfied` is
Rust telling you, at compile time, "you are about to move a `struct mutex`." In C this would
be a silent corruption found three months later by lockdep or a list-corruption splat.

### Lab 80.5 — Read the expansion

```bash
cd $KDIR

# Read the crate docs -- pin-init's documentation is unusually thorough
# and includes a full worked explanation of the macro.
$EDITOR rust/pin-init/src/lib.rs        # read the module doc comment in full
$EDITOR rust/kernel/init.rs             # the kernel's InPlaceInit glue

# Find real uses in tree:
grep -rn 'try_pin_init!' rust/ drivers/ samples/ | head -20
grep -rn '#\[pin_data\]'  rust/ drivers/ samples/ | head -20

# The lock implementation, which is the best worked example:
$EDITOR rust/kernel/sync/lock.rs
$EDITOR rust/kernel/sync/lock/mutex.rs

# Generated docs include the macros with examples:
make LLVM=1 rustdoc
xdg-open Documentation/output/rust/rustdoc/pin_init/index.html
```

Specific things to find and understand:
1. Where `PhantomPinned` appears in `Opaque` and why.
2. How `Lock<T, B>` puts `data` behind `UnsafeCell` and the C lock behind `Opaque`.
3. How `Guard`'s `Drop` calls the backend's unlock.
4. How `new_mutex!` creates the static `lock_class_key`.

### Lab 80.6 — Port a C structure with embedded locks and lists

Take a real driver struct from Part 2:

```c
struct my_device {
	struct device      *dev;
	void __iomem       *base;
	spinlock_t          lock;
	struct list_head    pending;
	struct completion   done;
	struct work_struct  work;
	u32                 flags;
};
```

Write the Rust equivalent. For each field, answer:

| Field | `#[pin]`? | Why |
|---|---|---|
| `dev` | no | it is an `ARef<Device>` — a pointer, freely movable |
| `base` | no | `IoMem` is a wrapper over an address; movable |
| `lock` | **yes** | `spinlock_t` has a wait list and a lockdep key |
| `pending` | **yes** | `list_head` is self-referential |
| `done` | **yes** | `struct completion` contains a wait queue head |
| `work` | **yes** | `work_struct` is queued by address |
| `flags` | no | plain data |

Then write the `try_pin_init!` and compare the total line count with the C `probe()` that
initializes all seven plus its error ladder. **Report the ratio.**

---

## 3. Mastery drills

1. Read `core::pin`'s module documentation completely, then explain `Pin` to a colleague in
   under three minutes without using the word "pin." (Suggested framing: "a wrapper that
   refuses to hand out `&mut`.")

2. Implement a self-referential type correctly with `Pin` and `pin-init`, with a test that
   demonstrates the pointer remains valid across every operation. Then try to break it and
   document what the compiler stops you from doing.

3. Read `rust/pin-init/src/macros.rs`. Produce a two-page explanation of how `try_pin_init!`
   expands, including exactly where the partial-teardown guard is generated.

4. Determine which C kernel structures are `!Unpin` in the Rust bindings and why. Build the
   table. Find one that arguably should be `!Unpin` but is not, if any exists.

5. Measure the cost of `pin-init` versus naive stack-construct-then-move for a 4 KiB struct:
   stack usage, instruction count, and whether the move is even legal. Report the stack
   figure against `THREAD_SIZE`.

6. Implement `PinInit` manually (without the macro) for a three-field struct, including
   correct unwinding on partial failure. Then compare your implementation with the macro's
   expansion and list every difference.

7. Take the `List`/`ListArc`/`ListLinks` machinery in `rust/kernel/list.rs` and explain how it
   encodes `container_of`. Compare the safety properties with C's `list_entry()`.

8. Deliberately mark a `Mutex` field without `#[pin]` and determine what breaks and at what
   point (compile error, or unsoundness?). Write up the finding as a review note.

9. Explain why `Arc<T>` does not need `Pin<Arc<T>>` while `KBox<T>` does need
   `Pin<KBox<T>>`. Write it as you would explain it on a mailing list.

10. Find a kernel C structure whose move-unsafety is *not* obvious (no `list_head`, no lock)
    and explain why it still cannot move. (Hint: anything whose address was registered, or
    anything containing a `kref` that C dereferences by address.)

11. Design and implement a `pin-init`-based abstraction for a C API that requires
    `init`-then-`register`-then-`unregister`-then-`destroy`, with correct behaviour if any
    step fails. Get the `Drop` ordering right.

12. Write the one-page "pinning for C developers" explainer you would put in your team's
    onboarding docs. Include the three things a reviewer must check and the six error
    messages from Lab 80.4.

---

## 4. Further reading

**Primary documentation**
- `core::pin` module documentation — the canonical explanation; long, careful, and worth the
  full read. The "Drop guarantee" section is the crux
- `rust/pin-init/src/lib.rs` module doc — the best explanation of the initialization problem
  and the solution, written by the author
- `make rustdoc` → the `pin_init` crate docs, with runnable examples

**Code to read, in order**
1. `rust/kernel/types.rs` — `Opaque` and its `PhantomPinned`
2. `rust/kernel/sync/lock.rs` — the generic lock; the clearest use of pinning in tree
3. `rust/kernel/sync/lock/mutex.rs` — `new_mutex!` and the lockdep key
4. `rust/kernel/list.rs` — pinned intrusive lists; the hardest and most rewarding
5. `rust/pin-init/src/macros.rs` — the expansion, when you are ready

**Articles and talks**
- "Pin and suffering" and the various Rust-community `Pin` explainers — most were written for
  async, but the mechanism is identical
- Benno Lossin's Kangrejos talks on `pin-init` — the design rationale from its author
- LWN: "Safe pinned initialization for Rust" and the merge coverage
- The `pin-init` crate on crates.io — it is now usable outside the kernel, and its README is
  a good standalone introduction

**Background**
- *The Rustonomicon*, chapters on `Drop`, `PhantomData`, and `#[may_dangle]`
- The original `Pin` RFC (rust-lang/rfcs#2349) — the design discussion, including the
  alternatives that were rejected
- Withoutboats' blog series on `Pin` and self-referential types — the clearest account of
  *why* the design is shaped this way

→ Next: [81-rust-sync.md](81-rust-sync.md)

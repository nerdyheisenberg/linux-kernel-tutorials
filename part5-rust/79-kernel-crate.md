# Chapter 79 — The `kernel` Crate: Types, Errors, Allocation, Refcounting

> This is the API reference chapter. Everything a Rust driver touches comes from here:
> `Result` and `Error`, the fallible containers, the refcounting types, the C-interop
> wrappers, and the string handling. If Ch. 78 taught you to read the language, this chapter
> teaches you to read the library.

---

## Theory & First Principles

### T.0 — Start here: no `std`, no allocator, no panics

Rust's standard library assumes an operating system. You are *writing* the operating system.
So the first line of every kernel Rust file is, in effect:

```rust
#![no_std]  // no std::fs, no std::thread, no std::println!, no std::collections
```

**What survives is `core`** — `Option`, `Result`, slices, iterators, traits, `Ordering` —
everything that needs no allocator and no OS. Everything else must be rebuilt against kernel
primitives. That rebuild *is* the `kernel` crate.

**And it is not a port. Three things are fundamentally different**, and each forces a design
decision you will meet throughout Part 5:

| `std` assumes | The kernel requires |
|---|---|
| allocation always succeeds; OOM aborts the process | **allocation can fail and you must handle it** — there is nothing to abort to |
| allocation is context-free | `GFP_KERNEL` vs `GFP_ATOMIC` vs `GFP_NOIO` — context-dependent (Ch. 11 §T.0) |
| `panic!` unwinds and kills a thread | **a panic is a kernel oops**; there is no safe unwinding |

Hence:

```rust
// std:                          kernel:
let v = vec![1, 2, 3];        // let mut v = KVec::new();
                              // v.push(1, GFP_KERNEL)?;   <- fallible
                              //                ^^^^^^^^^ and explicit
let b = Box::new(x);          // let b = KBox::new(x, GFP_KERNEL)?;
```

**Every allocation returns a `Result`, and every allocation names its context.** That is
tedious, and it is also exactly right: the `GFP` rules are a real invariant that C leaves to
comments and to a runtime `might_sleep()`. Here the flag is a mandatory argument and the
failure is a value you cannot ignore. **Another comment turned into a type-checked
parameter** (Ch. 78 §T.0).

**Now the crate's actual job, which is the thing to understand structurally:**

```
   +--------------------------------------------------+
   |  YOUR DRIVER     -- 100% safe Rust, ideally       |
   +--------------------------------------------------+
   |  kernel crate    -- SAFE ABSTRACTIONS             |
   |    Mutex<T>  ARef<T>  KVec<T>  Io<SIZE>  Device   |
   |    Registration  Module  miscdev  chrdev  ...     |
   |                                                   |
   |    Internally unsafe, with a documented SAFETY    |
   |    argument for every unsafe block.               |
   +--------------------------------------------------+
   |  bindings crate  -- raw, machine-generated,       |
   |                     entirely unsafe (Ch. 84)      |
   +--------------------------------------------------+
   |  the C kernel    -- hostile: null pointers,       |
   |                     aliasing, no lifetimes        |
   +--------------------------------------------------+
```

**The whole value proposition rests on one property of that picture:** unsafe code is
*concentrated*, not *distributed*. Reviewing 5,000 lines of carefully-argued unsafe
abstraction is tractable; reviewing 5,000,000 lines of driver code for the same bug classes
is not. **Rust in the kernel is primarily a strategy for making review tractable**, not a
claim that unsafety disappears.

**Two patterns you will see everywhere in the crate, worth recognizing now:**

1. **`# Safety` on every `unsafe fn`, and `// SAFETY:` on every `unsafe` block.** The kernel
   build enforces this (`clippy::undocumented_unsafe_blocks`). The comment *is* the proof;
   the reviewer's job is to check it; and the proof has a fixed, greppable location — which
   is precisely what C lacks.
2. **`impl PinInit` instead of constructors.** Many kernel structures cannot be built on the
   stack and moved: a `struct mutex` registers itself with lockdep at its final address, and
   an intrusive list node is pointed at by its neighbours (Ch. 09). So the crate initializes
   **in place** — Ch. 80's subject, and the single most kernel-specific thing about kernel
   Rust.

```bash
ls rust/kernel/ && wc -l rust/kernel/*.rs | tail -1
grep -rn 'unsafe' rust/kernel/ | wc -l
grep -rn 'SAFETY:' rust/kernel/ | wc -l     # compare these two numbers
sed -n '1,60p' rust/kernel/alloc/kbox.rs
```

---

### T.1 — The crate's job: one safe façade over a hostile C world

The `kernel` crate exists to answer one question for every C API: **what is the smallest
Rust-visible type that makes misuse impossible?**

Four recurring answers:

| C situation | Rust answer |
|---|---|
| A C struct Rust must hold but never construct or move | `Opaque<T>` |
| A C object with a refcount Rust must participate in | `AlwaysRefCounted` + `ARef<T>` |
| A Rust object C must own for a while | `ForeignOwnable` (`into_foreign`/`from_foreign`) |
| A resource that must be released | ownership + `Drop` |

Everything else is composition of those four. Learning them is learning the crate.

### T.2 — `Result` and `Error`: errno as a first-class value

```rust
pub type Result<T = (), E = Error> = core::result::Result<T, E>;

pub struct Error(core::ffi::c_int);   // always in -MAX_ERRNO..0
```

The constants are in the prelude with their C names: `EINVAL`, `ENOMEM`, `EIO`, `EAGAIN`,
`EPROBE_DEFER`, `ENOTSUPP`, … so a Rust driver reads like a C one:

```rust
if !valid {
    return Err(EINVAL);
}
```

Three conversion helpers you will use constantly at the FFI boundary:

```rust
// C returns 0 or -errno:
to_result(unsafe { bindings::clk_prepare_enable(clk) })?;

// C returns a pointer or an ERR_PTR:
let ptr = from_err_ptr(unsafe { bindings::clk_get(dev, name) })?;

// C returns a pointer or NULL:
let ptr = NonNull::new(raw).ok_or(ENOMEM)?;
```

And going the other way, in a callback C will call:

```rust
unsafe extern "C" fn probe_callback(pdev: *mut bindings::platform_device) -> c_int {
    match Self::probe_inner(pdev) {
        Ok(())  => 0,
        Err(e)  => e.to_errno(),
    }
}
```

**`Result` is `#[must_use]`.** Ignoring it is a warning that the kernel build turns into a
hard error — which is `__must_check` applied universally and unforgettably. That alone
eliminates a recurring C bug: an unchecked `clk_prepare_enable`.

### T.3 — Fallible allocation

The single largest divergence from standard Rust.

```rust
// std / userspace: infallible, aborts the process on OOM
let b = Box::new(value);
let mut v = Vec::new();  v.push(x);

// kernel: fallible, explicit context flag
let b = KBox::new(value, GFP_KERNEL)?;
let mut v = KVec::new(); v.push(x, GFP_KERNEL)?;
```

Three allocators, matching the C functions you know from Ch. 11:

| Type | C equivalent | Properties |
|---|---|---|
| `Kmalloc` / `KBox` / `KVec` | `kmalloc` | Physically contiguous, size-limited, fast |
| `Vmalloc` / `VBox` / `VVec` | `vmalloc` | Virtually contiguous, large, page-granular |
| `KVmalloc` / `KVBox` / `KVVec` | `kvmalloc` | Try kmalloc, fall back to vmalloc |

And the GFP flags are values, not magic numbers:

```rust
GFP_KERNEL      // may sleep, may reclaim
GFP_ATOMIC      // may not sleep; uses reserves
GFP_KERNEL | __GFP_ZERO
__GFP_NOWARN    // composable with |
```

Why this matters architecturally: **the allocation context is now part of the function
signature's data flow.** A helper that allocates must be given a flag by its caller, so the
question "can this path sleep?" is pushed up to someone who knows the answer, rather than
being baked in by whoever wrote the helper. That is a genuine improvement on C, where
`GFP_KERNEL` is usually hardcoded and the bug surfaces as a `might_sleep()` splat years
later.

`KBox` also has the in-place forms, which matter for large objects:

```rust
KBox::new_uninit(GFP_KERNEL)?          // allocate without initializing
KBox::init(pin_init!(...), GFP_KERNEL)?  // initialize IN PLACE (Ch. 80)
```

The in-place form is not a micro-optimization: constructing a 4 KiB struct on the stack and
then moving it into a box would overflow the kernel's 16 KiB stack. **In-place
initialization is a correctness requirement in the kernel, not a performance nicety.**

### T.4 — `Arc<T>`: shared ownership, kernel edition

```rust
use kernel::sync::Arc;

let a = Arc::new(Data { .. }, GFP_KERNEL)?;
let b = a.clone();      // refcount++ (an atomic increment)
drop(b);                // refcount--; frees at zero
```

Differences from `alloc::sync::Arc`:
- **Fallible construction** — returns `Result`.
- Uses `refcount_t` semantics: saturating, with overflow detection, matching the kernel's own
  hardening (→ Ch. 12).
- `ArcBorrow<'_, T>` exists for passing a borrowed reference without touching the refcount —
  the equivalent of passing a raw pointer under a caller-held reference, which is the common
  kernel pattern.

The critical property, and the thing to say in review: **`Arc<T>` gives shared ownership, not
shared mutability.** `Arc<T>` derefs to `&T` only. To mutate you need `Arc<Mutex<T>>` or
`Arc<SpinLock<T>>` — and the type therefore *states* that the data is locked. In C, `kref` on
a struct tells you nothing about whether its fields are locked, and the answer lives in a
comment.

```rust
Arc<Mutex<State>>      // shared, mutable, sleepable lock  -- the common driver shape
Arc<SpinLock<State>>   // shared, mutable, atomic-context lock
Arc<Data>              // shared, immutable
```

`Arc` cycles leak, exactly as `kref` cycles do. `Weak` exists but is less used in kernel code
so far. **Rust does not prevent leaks, and saying so unprompted is a mark of honesty.**

### T.5 — `ARef<T>` and `AlwaysRefCounted`: participating in C's refcounts

`Arc` is for objects Rust allocated. But most kernel objects are allocated by C and carry
their own refcount (`struct device`, `struct file`, `struct inode`). Rust must be able to
hold a reference without owning the allocation.

```rust
/// # Safety
/// Implementers must ensure the object is always refcounted, and that
/// `inc_ref`/`dec_ref` manipulate that refcount correctly.
pub unsafe trait AlwaysRefCounted {
    fn inc_ref(&self);
    unsafe fn dec_ref(obj: NonNull<Self>);
}

pub struct ARef<T: AlwaysRefCounted> { ptr: NonNull<T>, ... }
```

```rust
// For struct device, inc_ref calls get_device() and dec_ref calls put_device().
let dev: ARef<Device> = some_device.into();
// Clone -> get_device(); drop -> put_device(). Automatically, on every path.
```

**This is the abstraction that eliminates the most common refcount bug in kernel C**: an
error path that returns without a `put_device()`. In Rust the `put` is the `Drop`, and `Drop`
runs on every path including `?`-propagation and panics-that-abort.

Read `rust/kernel/types.rs` for `ARef` and `rust/kernel/device.rs` for the `Device` impl —
together they are about 80 lines and they demonstrate the entire pattern.

### T.6 — `Opaque<T>`: holding a C struct honestly

```rust
#[repr(transparent)]
pub struct Opaque<T> {
    value: UnsafeCell<MaybeUninit<T>>,
    _pin: PhantomPinned,
}
```

Each field encodes one fact about C structs:

| Field | States |
|---|---|
| `UnsafeCell` | C may mutate this behind Rust's back, so it is **not** `noalias` — the optimizer must not assume otherwise |
| `MaybeUninit` | C may not have initialized it; Rust must not assume a valid value exists |
| `PhantomPinned` | It must never move; C holds pointers to it |
| `#[repr(transparent)]` | Layout is exactly `T`, so a `*mut Opaque<T>` is a `*mut T` |

Usage:

```rust
#[repr(transparent)]
pub struct Device(Opaque<bindings::device>);

impl Device {
    /// # Safety
    /// `ptr` must be valid and the caller must hold a reference for 'a.
    pub unsafe fn from_raw<'a>(ptr: *mut bindings::device) -> &'a Self {
        // SAFETY: Device is repr(transparent) over Opaque<device>, which
        // is repr(transparent) over device, and the caller guarantees
        // validity for 'a.
        unsafe { &*ptr.cast() }
    }

    pub fn as_raw(&self) -> *mut bindings::device {
        self.0.get()
    }
}
```

`Opaque` is why kernel Rust can hold a `struct device` at all without either copying it
(impossible — C holds pointers) or lying to the optimizer (unsound).

### T.7 — `ForeignOwnable`: lending an object to C

The `drvdata` pattern, made safe.

```rust
pub trait ForeignOwnable: Sized {
    type Borrowed<'a>;
    fn into_foreign(self) -> *mut c_void;            // give ownership to C
    unsafe fn from_foreign(p: *mut c_void) -> Self;  // take it back
    unsafe fn borrow<'a>(p: *mut c_void) -> Self::Borrowed<'a>;  // peek
}
```

Implemented for `KBox<T>`, `Arc<T>`, and `Pin<KBox<T>>`. The lifecycle in a driver:

```
 probe:    let d = KBox::new(state, GFP_KERNEL)?;
           dev_set_drvdata(dev, d.into_foreign());     // C owns it now
 callback: let d = unsafe { KBox::<State>::borrow(dev_get_drvdata(dev)) };
                                                        // C still owns it
 remove:   let d = unsafe { KBox::<State>::from_foreign(dev_get_drvdata(dev)) };
                                                        // Rust owns it again
           // d dropped at end of scope -> freed
```

The three-way distinction (`into`/`borrow`/`from`) is exactly the ownership question that C's
`void *drvdata` leaves unanswered. **Every review of a Rust driver should check that
`into_foreign` and `from_foreign` are balanced on every path**, because an unbalanced pair is
a leak (or, worse, a double free) and it is the one class of bug the compiler cannot catch
here — the raw pointer round-trip is exactly where ownership tracking is suspended.

### T.8 — Strings, and the C boundary

Three string types, for three different problems:

| Type | Is | Use |
|---|---|---|
| `&str` | UTF-8, no NUL, Rust-native | Rust-internal text |
| `CStr` | NUL-terminated, borrowed | Passing to C |
| `CString` | NUL-terminated, owned, heap | Building a string for C |
| `BStr` | bytes, no encoding assumption | Arbitrary byte strings |

```rust
let s = c_str!("my_device");          // compile-time CStr, no allocation
let owned = CString::try_from_fmt(fmt!("dev{}", id))?;   // runtime, fallible
```

`fmt!` is the kernel's `format_args!`, and `pr_info!`/`dev_err!` take the same format syntax
as Rust's `println!` (`{}`, `{:?}`, `{:#x}`) — **not** printf's `%d`. This is one of the
first things that trips up C developers reading Rust patches.

```rust
pr_info!("value = {} hex = {:#x} debug = {:?}\n", v, v, obj);
dev_err!(dev, "probe failed: {:?}\n", e);
```

Note that `\n` is still required — `pr_info!` maps onto `printk` and inherits its line
discipline.

### T.9 — `UserSlice`: `__user` as a type

```rust
let mut reader = UserSlice::new(user_ptr, len).reader();
let mut buf = KVec::new();
reader.read_all(&mut buf, GFP_KERNEL)?;      // copy_from_user, checked

let mut writer = UserSlice::new(user_ptr, len).writer();
writer.write_slice(&data)?;                  // copy_to_user, checked
```

There is **no way to dereference a `UserSlice`**. The only operations are the copy helpers,
which go through `copy_from_user`/`copy_to_user` and return `Result`. Compare with C, where
`__user` is a `sparse` annotation that is advisory and frequently missing on new code.

Two further properties worth noting in review:
- A `UserSliceReader` is **consumed as it reads**, so a double-fetch (reading the same user
  memory twice and getting different values — the TOCTOU bug class from Ch. 24 and Ch. 76)
  requires deliberately constructing a second reader, which is visible in the diff.
- Lengths are checked, so the hardened-usercopy bug from Ch. 102 Lab 4 is not expressible.

### T.10 — What is still missing

Read this list before promising anyone a Rust driver:

- Subsystem coverage is **partial**. As of 6.12-ish: platform, PCI (growing), misc device,
  char device, block (via `rnull`), net PHY, DRM (via Nova), clk/regulator/gpio (partial),
  DMA (recent and contentious). Many subsystems have **no** abstraction at all.
- If the abstraction does not exist, you must write it — which means `unsafe`, `bindgen`
  coverage, and getting a maintainer to review a new safety contract.
- Some C APIs are **structurally hard to wrap**: anything with an implicit context
  requirement (may-sleep, IRQ-disabled), anything with a lock held across a callback,
  anything whose lifetime rules are stated only in prose.

The honest planning answer: **"Rust is ready for new drivers in subsystems that already have
abstractions. Everywhere else, budget for writing the abstraction, and budget for the review
cycle it will take."**

---

## 1. Internals

### Source map

| Path | Contents |
|---|---|
| `rust/kernel/error.rs` | `Error`, `Result`, `to_result`, `from_err_ptr`, errno constants |
| `rust/kernel/alloc.rs` | `Allocator` trait, `Flags` (`GFP_*`), `AllocError` |
| `rust/kernel/alloc/kbox.rs` | `KBox`, `VBox`, `KVBox` |
| `rust/kernel/alloc/kvec.rs` | `KVec`, `VVec`, `KVVec` |
| `rust/kernel/sync/arc.rs` | `Arc`, `ArcBorrow`, `UniqueArc` |
| `rust/kernel/types.rs` | `Opaque`, `ARef`, `AlwaysRefCounted`, `ForeignOwnable`, `ScopeGuard`, `Either` |
| `rust/kernel/str.rs` | `CStr`, `CString`, `BStr`, `fmt!`, `c_str!` |
| `rust/kernel/uaccess.rs` | `UserSlice`, `UserSliceReader`, `UserSliceWriter` |
| `rust/kernel/device.rs` | `Device`, its `AlwaysRefCounted` impl |
| `rust/kernel/print.rs` | `pr_*!` macros |
| `rust/kernel/prelude.rs` | The auto-imported set |

### `KBox`, annotated

```rust
#[repr(transparent)]
pub struct Box<T: ?Sized, A: Allocator>(NonNull<T>, PhantomData<A>);

pub type KBox<T>  = Box<T, Kmalloc>;
pub type VBox<T>  = Box<T, Vmalloc>;
pub type KVBox<T> = Box<T, KVmalloc>;

impl<T, A: Allocator> Box<T, A> {
    pub fn new(x: T, flags: Flags) -> Result<Self, AllocError> {
        let b = Self::new_uninit(flags)?;
        Ok(Box::write(b, x))
    }
}

impl<T: ?Sized, A: Allocator> Drop for Box<T, A> {
    fn drop(&mut self) {
        let ptr = self.0.as_ptr();
        // SAFETY: the value is valid per the type invariant and is being
        // dropped exactly once.
        unsafe { core::ptr::drop_in_place(ptr) };
        // SAFETY: the memory came from A and has not been freed.
        unsafe { A::free(self.0.cast(), Layout::for_value_raw(ptr)) };
    }
}
```

Two things to notice: `#[repr(transparent)]` over a `NonNull` means `Option<KBox<T>>` is
exactly one pointer (the `None` uses the null niche), and the `Allocator` is a **type
parameter**, so choosing `kmalloc` versus `vmalloc` is a compile-time decision with no
runtime dispatch.

### `Arc`, annotated

```rust
#[repr(transparent)]
pub struct Arc<T: ?Sized> {
    ptr: NonNull<ArcInner<T>>,
    _p: PhantomData<ArcInner<T>>,
}

#[repr(C)]
struct ArcInner<T: ?Sized> {
    refcount: Opaque<bindings::refcount_t>,   // the KERNEL's refcount_t
    data: T,
}
```

`refcount_t`, not a plain atomic — so it inherits the kernel's saturation-on-overflow and
`WARN` on underflow (→ Ch. 12 §T.4). A Rust `Arc` bug therefore produces the same `refcount_t:
underflow` splat a C bug would, which is good for debuggability and good for consistency.

`UniqueArc<T>` is worth knowing: an `Arc` with a refcount provably equal to 1, so it gives
`&mut T`. It is how you build an object that will *become* shared but needs mutation during
construction — and it converts to `Arc` with a simple `.into()`. This is a type-level
encoding of "not published yet," which C expresses by convention.

### `ScopeGuard` — the `goto out` replacement for foreign resources

```rust
let guard = ScopeGuard::new(|| unsafe { bindings::release_thing(p) });
do_something_fallible()?;      // if this fails, the guard runs
guard.dismiss();               // success: do not release
```

Used where the resource is not a Rust value with a `Drop` — typically mid-way through
wrapping a C API. It is the direct analogue of `guard()` from Ch. 05, and it appears
throughout `rust/kernel/` in exactly the places C would have a `goto`.

---

## 2. Practice

### Lab 79.1 — A tour module exercising the crate

```rust
// SPDX-License-Identifier: GPL-2.0
//! Exercises the core `kernel` crate types.

use kernel::prelude::*;
use kernel::alloc::{flags, KVec};
use kernel::str::CString;
use kernel::sync::{new_mutex, Arc, Mutex};

module! {
    type: CrateTour,
    name: "crate_tour",
    author: "You",
    description: "A tour of the kernel crate",
    license: "GPL",
}

struct Shared {
    counter: u64,
    label: CString,
}

struct CrateTour {
    shared: Arc<Mutex<Shared>>,
}

/// Demonstrates Result and `?` with a real failure path.
fn parse_id(s: &str) -> Result<u32> {
    // No unwrap: turn the parse failure into an errno.
    s.parse::<u32>().map_err(|_| EINVAL)
}

/// Demonstrates fallible allocation with three allocators.
fn allocate_things() -> Result {
    // kmalloc: small, physically contiguous
    let small = KBox::new([0u8; 128], GFP_KERNEL)?;
    pr_info!("kmalloc'd {} bytes\n", core::mem::size_of_val(&*small));

    // vmalloc: large, virtually contiguous
    let big: VBox<[u8; 1 << 20]> = VBox::new_uninit(GFP_KERNEL)?
        .write([0u8; 1 << 20]);
    pr_info!("vmalloc'd {} bytes\n", core::mem::size_of_val(&*big));

    // kvmalloc: try kmalloc, fall back
    let either = KVBox::new([0u32; 4096], GFP_KERNEL)?;
    pr_info!("kvmalloc'd {} bytes\n", core::mem::size_of_val(&*either));

    // Deliberately fail: __GFP_NOWARN so the OOM message is suppressed.
    let huge: Result<KBox<[u8; 1 << 30]>, _> =
        KBox::new_uninit(flags::GFP_KERNEL | flags::__GFP_NOWARN)
            .map(|b| b.write([0u8; 1 << 30]));
    match huge {
        Ok(_)  => pr_info!("1 GiB kmalloc unexpectedly succeeded\n"),
        Err(_) => pr_info!("1 GiB kmalloc failed, as expected, gracefully\n"),
    }

    Ok(())
}

impl kernel::Module for CrateTour {
    fn init(_m: &'static ThisModule) -> Result<Self> {
        pr_info!("crate_tour: init\n");

        // --- Result and ? ---
        pr_info!("parse_id(\"42\")   = {:?}\n", parse_id("42"));
        pr_info!("parse_id(\"oops\") = {:?}\n",
                 parse_id("oops").map_err(|e| e.to_errno()));

        // --- allocation ---
        allocate_things()?;

        // --- KVec with error propagation ---
        let mut v: KVec<u32> = KVec::with_capacity(16, GFP_KERNEL)?;
        for i in 0..16u32 {
            v.push(i, GFP_KERNEL)?;
        }
        pr_info!("vec len {} cap {} sum {}\n",
                 v.len(), v.capacity(), v.iter().sum::<u32>());

        // --- CString built at runtime, fallibly ---
        let label = CString::try_from_fmt(fmt!("tour-{}", v.len()))?;
        pr_info!("label = {}\n", &*label);

        // --- Arc<Mutex<T>>: shared, mutable, locked ---
        let shared = Arc::pin_init(
            new_mutex!(Shared { counter: 0, label }, "crate_tour::shared"),
            GFP_KERNEL,
        )?;

        // Clone the Arc -- refcount++, cheap, and the clone is independent.
        let second = shared.clone();
        {
            let mut g = second.lock();
            g.counter += 1;
        }   // unlock on scope exit

        pr_info!("counter = {}\n", shared.lock().counter);
        // `second` dropped here -> refcount--

        Ok(CrateTour { shared })
    }
}

impl Drop for CrateTour {
    fn drop(&mut self) {
        pr_info!("crate_tour: exit, final counter = {}\n",
                 self.shared.lock().counter);
        // Arc dropped -> refcount 0 -> Shared dropped -> CString freed.
        // None of that is written anywhere.
    }
}
```

### Lab 79.2 — Implement `AlwaysRefCounted` for a C object

```rust
// SPDX-License-Identifier: GPL-2.0
//! Participating in a C object's refcount, correctly.

use kernel::bindings;
use kernel::prelude::*;
use kernel::types::{ARef, AlwaysRefCounted, Opaque};
use core::ptr::NonNull;

/// A wrapper over `struct device`.
///
/// # Invariants
///
/// The pointer wrapped by `Opaque` always points to a valid `struct device`
/// for which we hold at least one reference while an `ARef<Device>` exists.
#[repr(transparent)]
pub struct Device(Opaque<bindings::device>);

// SAFETY: `struct device` is always refcounted via get_device/put_device,
// and the implementations below call exactly those.
unsafe impl AlwaysRefCounted for Device {
    fn inc_ref(&self) {
        // SAFETY: by the type invariant, self.0 points to a valid device
        // for which we already hold a reference, so incrementing is sound.
        unsafe { bindings::get_device(self.0.get()) };
    }

    unsafe fn dec_ref(obj: NonNull<Self>) {
        // SAFETY: the caller guarantees they are relinquishing a reference
        // they own, per the AlwaysRefCounted contract.
        unsafe { bindings::put_device(obj.cast().as_ptr()) };
    }
}

impl Device {
    /// Borrow a device from a raw pointer.
    ///
    /// # Safety
    ///
    /// `ptr` must be valid and the caller must guarantee it remains valid
    /// for the entire lifetime `'a`.
    pub unsafe fn from_raw<'a>(ptr: *mut bindings::device) -> &'a Self {
        // SAFETY: Device is repr(transparent) over Opaque<device> which is
        // repr(transparent) over device; validity is the caller's promise.
        unsafe { &*ptr.cast() }
    }

    pub fn as_raw(&self) -> *mut bindings::device {
        self.0.get()
    }
}

/// Now the payoff: this function cannot leak a reference.
fn hold_for_a_while(dev: &Device) -> ARef<Device> {
    let owned: ARef<Device> = dev.into();   // get_device()
    do_something_fallible().ok();           // even if this panicked-aborts,
    owned                                   // the count is consistent
}   // If we had returned early, `owned` would have been dropped -> put_device()
```

Compare with the C version of the same logic and count the `put_device()` calls that must be
written by hand — one per exit path. Then search the tree for a real bug of this shape;
`git log --grep="put_device"` finds many fixes.

### Lab 79.3 — `ForeignOwnable` round trip

```rust
// SPDX-License-Identifier: GPL-2.0
//! The drvdata pattern, done safely.

use kernel::bindings;
use kernel::prelude::*;
use kernel::types::ForeignOwnable;
use core::ffi::c_void;

struct DriverState {
    id: u32,
    buffer: KVec<u8>,
}

/// Called from probe. Ownership moves to C.
fn attach(dev: *mut bindings::device, id: u32) -> Result {
    let mut buffer = KVec::new();
    buffer.resize(4096, 0u8, GFP_KERNEL)?;

    let state = KBox::new(DriverState { id, buffer }, GFP_KERNEL)?;

    // From here the KBox is NOT dropped by Rust -- C owns the pointer.
    let raw: *mut c_void = state.into_foreign();

    // SAFETY: `dev` is valid; we are storing a pointer we just created.
    unsafe { bindings::dev_set_drvdata(dev, raw) };
    Ok(())
}

/// Called from any callback. C keeps ownership; we only look.
///
/// # Safety
/// `dev` must be a device previously passed to `attach` and not yet detached.
unsafe fn peek(dev: *mut bindings::device) -> u32 {
    // SAFETY: caller guarantees attach() ran and detach() has not.
    let raw = unsafe { bindings::dev_get_drvdata(dev) };
    // SAFETY: `raw` came from KBox::<DriverState>::into_foreign and C still
    // owns it, so borrowing for this scope is sound.
    let state = unsafe { KBox::<DriverState>::borrow(raw) };
    state.id
    // Nothing is dropped. This is the crucial difference from from_foreign.
}

/// Called from remove. Ownership returns to Rust.
///
/// # Safety
/// `dev` must be a device previously passed to `attach`, and this must be
/// called exactly once per `attach`.
unsafe fn detach(dev: *mut bindings::device) {
    // SAFETY: caller guarantees the pairing above.
    let raw = unsafe { bindings::dev_get_drvdata(dev) };
    // SAFETY: `raw` came from into_foreign; this is the single matching
    // from_foreign, so ownership transfers back exactly once.
    let state = unsafe { KBox::<DriverState>::from_foreign(raw) };
    pr_info!("detaching device {}\n", state.id);
    // `state` dropped here: DriverState dropped, KVec freed, KBox freed.
}
```

Then **deliberately introduce the bugs** and reason about what each costs:
- Call `from_foreign` twice → double free. The compiler does not catch it (the pointer
  round-trip suspended tracking). KASAN does. This is exactly why the `# Safety` contract
  says "exactly once."
- Never call `from_foreign` → leak. `kmemleak` finds it.
- Use `from_foreign` where `borrow` was intended → premature free, then UAF.

**The lesson to write down: `ForeignOwnable` is the one place in a Rust driver where you are
back to C-level guarantees, and it is therefore the place review must concentrate.**

### Lab 79.4 — Safe user-memory access

```rust
use kernel::prelude::*;
use kernel::uaccess::{UserSlice, UserSliceReader, UserSliceWriter};

/// A read() implementation.
fn do_read(user_buf: UserSlice, state: &State) -> Result<usize> {
    let mut writer = user_buf.writer();
    let data = state.snapshot();          // a &[u8]
    let n = core::cmp::min(writer.len(), data.len());
    writer.write_slice(&data[..n])?;      // copy_to_user, checked
    Ok(n)
}

/// A write() implementation, showing the double-fetch hazard explicitly.
fn do_write(user_buf: UserSlice, state: &mut State) -> Result<usize> {
    let mut reader = user_buf.reader();

    // Read a header ONCE.
    let mut hdr = [0u8; 8];
    reader.read_slice(&mut hdr)?;
    let len = u32::from_le_bytes(hdr[4..8].try_into().map_err(|_| EINVAL)?);

    // VALIDATE the value we already copied -- not the user memory, which
    // may have changed. This is the correct discipline; the reader has
    // already advanced past those bytes, so re-reading them is not
    // accidentally possible.
    if len as usize > reader.len() || len > MAX_LEN {
        return Err(EINVAL);
    }

    let mut payload = KVec::new();
    reader.read_all(&mut payload, GFP_KERNEL)?;
    state.apply(&payload)?;
    Ok(8 + payload.len())
}
```

Contrast with the C bug: read the length with `get_user`, validate it, then `copy_from_user`
using a **second** read of the same field. Between the two, userspace changes it. That is the
double-fetch CVE class (Ch. 24). The Rust reader's consume-as-you-go design makes the mistake
require visible extra effort.

### Lab 79.5 — Error handling under pressure

```rust
/// A probe-shaped function with six fallible steps and no goto ladder.
fn probe_inner(dev: &Device) -> Result<KBox<State>> {
    let clk    = Clk::get(dev, None)?;             // (1)
    let clk    = clk.prepare_enable()?;            // (2)
    let regs   = dev.ioremap_resource(0)?;         // (3)
    let irq    = dev.irq_by_index(0)?;             // (4)
    let mut buf: KVec<u8> = KVec::new();
    buf.resize(PAGE_SIZE, 0, GFP_KERNEL)?;         // (5)
    let state  = KBox::new(State { clk, regs, irq, buf }, GFP_KERNEL)?;  // (6)
    register_with_subsystem(&state)?;              // (7)
    Ok(state)
}
```

Now answer, for each failure point, **what gets released and in what order** — then verify by
adding `pr_info!` to each `Drop` impl and forcing each failure with a debugfs knob. The
expected answer is reverse declaration order, automatically, with no unwinding code written.

Write the C equivalent with its `goto` ladder and count the lines. Then introduce the classic
bug — swap two labels in the ladder — and note that the Rust version has no place to put that
bug.

### Lab 79.6 — Read the crate

```bash
cd $KDIR
make LLVM=1 rustdoc
xdg-open Documentation/output/rust/rustdoc/kernel/index.html

# Read these four files completely. Together they are ~1500 lines and they
# are the core of the crate.
$EDITOR rust/kernel/types.rs        # Opaque, ARef, ForeignOwnable, ScopeGuard
$EDITOR rust/kernel/error.rs        # Error, Result, conversions
$EDITOR rust/kernel/alloc/kbox.rs   # ownership + Drop, textbook unsafe discipline
$EDITOR rust/kernel/sync/arc.rs     # refcount_t integration, UniqueArc

# Count and categorize the unsafe:
grep -c 'unsafe' rust/kernel/types.rs
grep -B2 'unsafe {' rust/kernel/sync/arc.rs | grep 'SAFETY' | wc -l
# Every unsafe block should have a SAFETY comment. Verify the ratio.
```

For each `unsafe` block in `arc.rs`, **write down in your own words the invariant it relies
on and where that invariant is established**. If you can do that for all of them, you can
review a kernel Rust abstraction.

---

## 3. Mastery drills

1. Write a complete `AlwaysRefCounted` implementation for a C type that does not yet have one
   (`struct inode`, `struct net_device`, `struct fwnode_handle`). Get the `# Safety` contract
   right and submit it for review by a colleague.

2. Implement your own minimal `Arc<T>` from scratch over `refcount_t`, including
   `UniqueArc`. Explain every `unsafe` block. Then diff against `rust/kernel/sync/arc.rs` and
   account for every difference.

3. Measure the code size and runtime cost of `KBox<T>` versus a raw `kmalloc` +
   manual free, for a hot path. Confirm the zero-cost claim or find where it breaks.

4. Take a C driver with a complex probe and port only its error handling to the Rust shape
   (pseudocode is fine). Count the eliminated lines and identify any error path that was
   *wrong* in the original.

5. Write a `ForeignOwnable` implementation for a custom type and prove the balance property:
   construct a test that loads and unloads 10,000 times and shows no growth in `slabtop`.

6. Explore the `Allocator` trait: implement a third allocator (say, a fixed-size pool) and
   make `Box<T, MyPool>` work. Explain what the trait's safety contract requires of you.

7. Enumerate every type in `rust/kernel/types.rs` and write a one-line statement of the C
   problem it solves. Turn it into a table and add it to `reference/cheatsheets.md`.

8. Find three C kernel APIs with no Rust abstraction. For each, sketch the safe wrapper: what
   type wraps what, what the invariants are, what `unsafe` is needed, and which C contract is
   unstated and would need to be pinned down with the maintainer.

9. Study how `Result` composes across the FFI boundary in both directions in a real in-tree
   driver. Identify every place an errno is created, converted, or consumed.

10. Determine what happens on allocation failure at each of the six steps in Lab 79.5. Prove
    it experimentally using `CONFIG_FAILSLAB` and `/sys/kernel/debug/failslab/`, and verify
    the release order with tracing.

11. Compare `CString::try_from_fmt` with C's `kasprintf`. What can fail, what is checked, and
    what happens on failure in each? Write the comparison as you would explain it to a C
    maintainer reviewing a Rust patch.

12. Audit a real Rust driver for the `into_foreign`/`from_foreign` balance. Draw the state
    machine of the driver's lifecycle and mark each transition. Report any path where the
    balance is not obviously maintained.

---

## 4. Further reading

**Kernel documentation**
- `make rustdoc` — the generated `kernel` crate documentation. This is the primary reference;
  everything below is supplementary
- `Documentation/rust/coding-guidelines.rst` — `# Safety`, `# Invariants`, `// SAFETY:`
- `rust/kernel/lib.rs` — the module list; read it as a table of contents

**Code to read, in order**
1. `rust/kernel/error.rs` — small, complete, and you will use it constantly
2. `rust/kernel/types.rs` — the four core patterns of §T.1
3. `rust/kernel/alloc/kbox.rs` — ownership and `Drop` done properly
4. `rust/kernel/sync/arc.rs` — refcounting across the FFI
5. `samples/rust/rust_misc_device.rs` — all of the above in one working driver

**Books and references**
- *The Rustonomicon* — chapters on ownership, `Drop`, `PhantomData`, and FFI are directly
  relevant to every type in this chapter
- Gjengset, *Rust for Rustaceans* — ch. 2 (types), ch. 3 (designing interfaces), ch. 9
  (unsafe) map closely onto the crate's design decisions
- *Rust Atomics and Locks* (Mara Bos) — ch. 1–3 for `Arc` and refcounting semantics
- The `std::ptr` and `core::mem` module docs — `MaybeUninit`, `drop_in_place`, and
  `NonNull` are load-bearing here and their docs are unusually good

**LWN and talks**
- "Rust in the kernel: the `kernel` crate" coverage
- Kangrejos talks on `pin-init`, `Arc`, and the allocator design
- The LKML threads on fallible allocation and the `alloc` fork — good context for *why* the
  kernel does not just use `alloc`

→ Next: [80-pinning.md](80-pinning.md)

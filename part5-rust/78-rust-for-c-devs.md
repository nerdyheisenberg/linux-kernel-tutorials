# Chapter 78 — Rust for C Kernel Developers

> A crash course in the language, written for someone who knows C, knows the kernel, and has
> no interest in web servers. Every concept is introduced by the C problem it solves. The
> aim is that by the end you can *read* any file in `rust/kernel/` and review a driver
> patch — which is a lower bar than "write idiomatic Rust" and a much more useful one first.

---

## Theory & First Principles

### T.0 — Start here: read the comment, then read the type

Here is real kernel C, with its contract written in the only place C has for it:

```c
/*
 * Caller must hold dev->lock.
 * Returns a pointer valid only while the lock is held.
 * Caller must NOT free the result.
 * May not be called from interrupt context.
 */
struct config *dev_get_config(struct device *dev);
```

**Four invariants, zero of them checked.** A reviewer must notice each one, at every call
site, forever. Ch. 00 §T.1's mechanism/policy split has an evil twin here: *documentation as
a substitute for enforcement*.

Now the same contract, expressed so the compiler enforces it:

```rust
impl Device {
    fn config<'a>(&self, guard: &'a Guard<'_, Mutex<Inner>>) -> &'a Config {
        // You cannot CALL this without holding the lock: you must
        // produce the guard to pass it.
        //
        // The returned reference is tied to 'a -- the guard's lifetime --
        // so it CANNOT outlive the lock. That is a borrow-checker error,
        // not a latent bug.
        //
        // &Config is a borrow, not ownership: there is no way to free it.
    }
}
```

**Three of the four comment lines became compile errors.** Not documentation — *errors*.

**That is the whole chapter in one idea:**

> **Every invariant the kernel currently documents in a comment is a candidate for being
> encoded in a type.** The question for each one is: can the compiler check it, and what does
> the encoding cost in ergonomics?

The translation table is worth studying as a map of what you are about to learn:

| Kernel C convention | Encoded in Rust as | What you can no longer do |
|---|---|---|
| "caller must hold the lock" | `Mutex<T>` — the data lives *inside* the lock | touch the data without locking |
| "caller must `put` when done" | `ARef<T>` and `Drop` | forget the `put` |
| "valid only while X is alive" | a lifetime parameter `'a` | outlive X |
| "do not free this" | `&T`, a borrow rather than ownership | free it |
| "this may be NULL" | `Option<&T>` | dereference without checking |
| "returns an error or a pointer" (`ERR_PTR`) | `Result<T, Error>` | silently ignore the error |
| "safe to share between threads" | the `Sync` marker | share a non-`Sync` type |
| "this buffer is `len` bytes" | a slice `&[u8]` — length travels with the pointer | read past the end |

**The last row deserves a pause**, because it is the most mundane and the most valuable. In C
a pointer and its length are two arguments the compiler will never relate, and passing the
wrong one is the mechanism behind an enormous share of kernel CVEs. In Rust a slice *is* the
pair, indivisibly, and indexing is bounds-checked. **The bug class does not get harder; it
stops existing.**

**And the part C programmers find genuinely disorienting**, worth naming now so it is not a
surprise later: **moving is the default**. Assigning a value transfers ownership and the
source becomes unusable; `memcpy`-like duplication happens only for `Copy` types. This is
exactly why Ch. 80 (`Pin`) has to exist — the kernel is full of self-referential structures
(intrusive lists, Ch. 09; a `struct mutex` known to lockdep by its address) that *cannot* be
moved, and Rust must be told so explicitly.

**Two things Rust does *not* give you, so you do not waste effort looking:** it does not
prevent deadlock (lock ordering is still yours — Ch. 14), and it does not prevent leaks
(`mem::forget` is a safe function, and reference cycles leak). **Memory safety and resource
correctness are different properties**, and conflating them is the most common error in
arguments about this topic.

```bash
ls rust/kernel/*.rs
sed -n '1,80p' rust/kernel/sync/lock.rs       # Mutex<T>: the data is inside
grep -rn '# Safety' rust/kernel/ | head -20   # every unsafe fn states its contract
```

---

### T.1 — The type system is where kernel conventions go

C kernel code is full of conventions enforced by review and by `sparse`:

| C convention | How it is enforced | Rust equivalent |
|---|---|---|
| "returns NULL on failure" | documentation | `Option<T>` |
| "returns -errno or a pointer, use `IS_ERR`" | `ERR_PTR` encoding, `__must_check` | `Result<T, Error>` |
| `__user` pointers must not be dereferenced | `sparse` | `UserSlice` — a distinct type |
| `__iomem` pointers need `readl`/`writel` | `sparse` | `Io<SIZE>` — distinct type, no deref |
| "caller must hold the lock" | a comment | the data lives *inside* the lock |
| "caller must free this" | a comment | ownership; `Drop` |
| "this may sleep" | `might_sleep()`, lockdep | (still a convention — see §T.9) |
| "this struct is refcounted" | `kref`, and hope | `Arc<T>`, `ARef<T>` |

The single sentence that summarizes Part 5: **Rust moves kernel conventions from
documentation into types.** That is why a Rust driver is shorter than its C equivalent
despite Rust being more verbose per line — the invariant-checking code disappears into the
type.

### T.2 — Values, ownership, and moves

```rust
let a = KBox::new(Foo, GFP_KERNEL)?;
let b = a;              // MOVE. `a` is no longer usable.
// use(a);              // error[E0382]: borrow of moved value

let x = 5;
let y = x;              // COPY. i32 is `Copy`; `x` is still usable.
```

Types that are `Copy` (integers, `bool`, `char`, references, and structs of `Copy` types you
opt in) are duplicated. Everything else **moves**. A move is a `memcpy` of the value plus a
promise that the source will not be used or dropped — so there is no double free, ever, by
construction.

This is the C pattern of "who owns this pointer?" turned into a compiler-checked property.

**The three ways to pass a value to a function:**

```rust
fn consume(x: KBox<Foo>)      { }  // takes ownership; caller loses it
fn inspect(x: &Foo)           { }  // shared borrow; caller keeps it, read-only
fn modify(x: &mut Foo)        { }  // exclusive borrow; caller keeps it, mutable
```

In C all three are `Foo *`, and which one it is lives in a comment. The number of kernel bugs
caused by a function unexpectedly taking or not taking ownership is not small.

### T.3 — `Option` and `Result`: no NULL, no `ERR_PTR`

```rust
enum Option<T> { Some(T), None }
enum Result<T, E> { Ok(T), Err(E) }
```

These are **not** special. They are ordinary enums, and Rust enums are tagged unions that can
carry data. The compiler's **niche optimization** means `Option<&T>` and `Option<KBox<T>>`
are exactly pointer-sized, with `None` represented as the null pattern. So the safety costs
nothing.

The critical property: **you cannot use the value without handling the absent case.**

```rust
// C:
struct foo *f = find_foo(id);
f->field = 1;              // BOOM if find_foo returned NULL, and nothing
                           // in the language stopped you.

// Rust:
let f = find_foo(id);      // Option<&Foo>
// f.field = 1;            // error: no field `field` on `Option<&Foo>`
match f {
    Some(f) => { /* use f */ }
    None    => { /* handle it */ }
}
```

The kernel's `Result<T> = core::result::Result<T, Error>`, where `Error` wraps an errno:

```rust
fn do_thing() -> Result<u32> {
    let x = might_fail()?;          // `?` = if Err, return it. Like `goto out`
    let y = might_also_fail()?;     // but it cannot be forgotten
    Ok(x + y)
}
```

`?` is the direct replacement for the kernel's `if (ret) goto err_foo;` ladder — and because
every intermediate value's `Drop` runs on the early return, **the unwind is automatic**. That
one property removes the entire category of error-path leaks that `devm_` was invented for
(Ch. 28).

Converting between the two worlds:

```rust
// C returned an int errno-or-zero:
to_result(unsafe { bindings::some_c_call(ptr) })?;
// Rust returning to C:
match do_thing() { Ok(_) => 0, Err(e) => e.to_errno() }
```

### T.4 — Pattern matching

```rust
match value {
    Some(x) if x > 100  => pr_info!("big: {}\n", x),
    Some(0)             => pr_info!("zero\n"),
    Some(x)             => pr_info!("got {}\n", x),
    None                => pr_info!("nothing\n"),
}
```

`match` is **exhaustive**: forgetting a variant is a compile error. Adding a variant to an
enum makes every `match` on it fail to compile until updated — which is exactly what you want
when you add a new device state or a new opcode, and exactly what C's `switch` fails to give
you (`-Wswitch` helps but only for enums, only without a `default`).

Ergonomic forms you will see constantly:

```rust
if let Some(x) = opt { use_it(x); }
let Some(x) = opt else { return Err(EINVAL); };   // let-else: early exit
while let Some(item) = queue.pop() { process(item); }
```

### T.5 — Structs, enums, traits — mapping from C

```rust
struct Device {            // struct device { ... }
    id: u32,
    name: CString,
}

impl Device {              // methods; `self` is the C `this` pointer
    fn new(id: u32) -> Result<Self> { ... }   // associated fn (no self)
    fn id(&self) -> u32 { self.id }           // takes &self
    fn set_id(&mut self, id: u32) { self.id = id; }
}
```

```rust
enum State {               // a tagged union, with the tag handled for you
    Idle,
    Running { pid: u32 },
    Failed(Error),
}
```

C's equivalent is a `struct` with an explicit `enum` tag and a `union`, plus the discipline
to only read the right arm. Rust makes reading the wrong arm impossible.

**Traits are the big one.** A trait is a set of methods a type can implement — and the direct
analogue of a kernel `*_ops` vtable:

```rust
// C:
struct file_operations {
    ssize_t (*read)(struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *);
};

// Rust:
#[vtable]
pub trait Operations {
    fn read(...) -> Result<usize>;
    fn write(...) -> Result<usize>;
}
```

The `#[vtable]` proc macro generates the C `file_operations` struct from the trait impl, with
`HAS_READ`/`HAS_WRITE` associated constants so unimplemented methods become NULL function
pointers exactly as C expects. **You write a trait impl; the macro produces the C vtable.**
That is the central FFI trick of the whole `kernel` crate.

Two ways to use a trait:

```rust
fn generic<T: Operations>(x: &T)   { }   // STATIC dispatch: monomorphized,
                                         // inlined, zero cost
fn dynamic(x: &dyn Operations)     { }   // DYNAMIC dispatch: a fat pointer
                                         // (data + vtable). Exactly a C
                                         // struct+ops pair
```

Use static dispatch by default; it compiles to the same code as a direct call. Use `dyn` when
you genuinely need heterogeneous types behind one interface, which is when C would have used
a function-pointer table anyway.

### T.6 — Generics and the where-clause

```rust
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T { ... }
```

Rust generics are **monomorphized**: the compiler emits one specialized copy per concrete
type, like C++ templates and unlike Java. So they are zero-cost at runtime and cost compile
time and code size instead.

Unlike C++ templates, the bounds (`T: PartialOrd + Copy`) are checked *at the definition*, so
the error appears where you wrote the generic function, not inside a 400-line instantiation
trace. This matters when reading error messages.

The kernel uses generics heavily for exactly the things C does with `void *` and macros:
`KVec<T>`, `Arc<T>`, `SpinLock<T>`, `Registration<T>`.

### T.7 — Lifetimes, in the smallest useful dose

You will read lifetimes far more often than you write them.

```rust
fn first_word<'a>(s: &'a str) -> &'a str { ... }
```

Read `'a` as "some region of code." The signature says: *the returned reference is valid for
the same region as the input.* The compiler then refuses any call where you would keep the
result longer than the input lives.

**Elision** means you rarely write them. The three rules:
1. Each input reference gets its own lifetime.
2. If there is exactly one input lifetime, it is assigned to all outputs.
3. If one of the inputs is `&self`, its lifetime is assigned to all outputs.

So `fn name(&self) -> &str` needs no annotation — rule 3 covers it.

`'static` means "lives for the whole program." In the kernel, `&'static ThisModule` and
string literals are the common cases. **`'static` is not a promise that memory is leaked;
it is a bound on how long a reference may be held.**

Where lifetimes get hard, and where you will see them fought over in review, is
**self-referential structures** — a struct holding a pointer into itself, which is
ubiquitous in kernel C (every `list_head` embedded in an object that is pointed to). That is
what Ch. 80's pinning is about.

### T.8 — Interior mutability, and why it exists

The borrow rule says "one `&mut` or many `&`." But a lock is precisely a thing that gives you
mutable access through a shared reference — many threads hold `&Mutex<T>` and one at a time
gets `&mut T`. That requires an escape hatch, and the escape hatch is `UnsafeCell<T>`.

```
 UnsafeCell<T>      -- the ONLY legal way to get &mut T from &T. A compiler
                       primitive: it opts out of the noalias guarantee.
   ├─ Cell<T>       -- get/set by value, single-threaded
   ├─ RefCell<T>    -- runtime-checked borrows, panics on violation
   │                   (NOT used in kernel: it can panic)
   └─ SpinLock<T>, Mutex<T>  -- the kernel's versions, checked by an
                                actual lock
```

**Every synchronization primitive in existence is built on `UnsafeCell`.** When you see it in
`rust/kernel/sync/`, that is why. Ch. 81 develops this.

### T.9 — What is still a convention

Honesty matters here, and interviewers probe it.

- **"May sleep" is not in the type system.** A `Mutex::lock()` in an atomic context is still
  a bug caught at runtime by `might_sleep()`, exactly as in C. There have been proposals for
  a `klint`-style context checker; none has landed.
- **Lock ordering** is not in the type system. `lockdep` is still the tool.
- **Interrupt context** is not tracked.
- **RCU read-side sections** have a `Guard` type that helps, but nothing prevents you from
  stashing a pointer past its end without `unsafe`... because getting the pointer out
  requires `unsafe` in the first place. So the *abstraction* enforces it, not the language.

The pattern: **Rust gives you the tools to encode a rule as a type, but someone has to do the
encoding.** Where the `kernel` crate has done it, you get compile-time safety; where it has
not, you get C's guarantees. Knowing which is which is what makes you useful in review.

---

## 1. Internals

### Source map — the files to read as a language reference

| Path | Read it for |
|---|---|
| `rust/kernel/prelude.rs` | What is in scope everywhere; a good index |
| `rust/kernel/error.rs` | `Error`, `Result`, `to_result`, `from_err_ptr`, the errno constants |
| `rust/kernel/types.rs` | `Opaque`, `ARef`, `AlwaysRefCounted`, `ForeignOwnable`, `ScopeGuard` |
| `rust/kernel/alloc/kbox.rs`, `kvec.rs` | Fallible containers; excellent examples of `unsafe` discipline |
| `rust/kernel/str.rs` | `CStr`, `CString`, `BStr`, `fmt!` — string handling across the FFI |
| `rust/kernel/uaccess.rs` | `UserSlice`, `UserSliceReader/Writer` — `__user` as a type |
| `rust/macros/vtable.rs` | How a trait becomes a C ops struct |
| `samples/rust/rust_misc_device.rs` | The most complete small example in-tree |

### `Error` — how errno becomes a type

```rust
pub struct Error(core::ffi::c_int);   // always negative, always a valid errno

// Construction from a C return value:
pub fn to_result(err: core::ffi::c_int) -> Result {
    if err < 0 { Err(Error::from_errno(err)) } else { Ok(()) }
}

// And back:
impl Error {
    pub fn to_errno(self) -> core::ffi::c_int { self.0 }
    pub fn to_ptr<T>(self) -> *mut T { /* ERR_PTR equivalent */ }
}
```

`Result<T>` in kernel Rust is `core::result::Result<T, Error>`, and because `Error` is a
single `c_int` with a niche, `Result<()>` is one word and `Result<KBox<T>>` is one pointer.
**The `ERR_PTR`/`PTR_ERR` trick is unnecessary because the type system can express the sum
directly, at the same cost.** That is a clean example of §T.1.

### `Opaque<T>` — wrapping C structs you must not inspect

```rust
pub struct Opaque<T> {
    value: UnsafeCell<MaybeUninit<T>>,
    _pin: PhantomPinned,
}
```

Used for every C type Rust holds but must not construct, move, or assume is initialized —
`bindings::device`, `bindings::file`, `bindings::mutex`. Three properties encoded at once:
- `UnsafeCell`: C may mutate it behind our back, so it must not be `noalias`.
- `MaybeUninit`: C may leave it uninitialized.
- `PhantomPinned`: it must never move (C code holds pointers to it).

Reading this one 4-line struct teaches more about FFI safety than any amount of prose.

### `#[vtable]` expansion, sketched

```rust
#[vtable]
pub trait Operations {
    fn read(...) -> Result<usize> { kernel::build_error!("not implemented") }
    fn write(...) -> Result<usize> { kernel::build_error!("not implemented") }
}
```
expands to add:
```rust
pub trait Operations {
    const HAS_READ: bool = false;    // set to true if the impl overrides read
    const HAS_WRITE: bool = false;
    ...
}
```
and the adapter code builds the C struct with `if T::HAS_READ { Some(read_trampoline::<T>) }
else { None }`. Since the constants are compile-time, the branch vanishes. **A NULL function
pointer in the C struct, chosen at compile time by whether you wrote the method.**

---

## 2. Practice

### Lab 78.1 — Translate a C driver idiom to Rust, line by line

```c
/* The classic C probe error ladder. */
static int myprobe(struct platform_device *pdev)
{
	struct mydev *d;
	int ret;

	d = kzalloc(sizeof(*d), GFP_KERNEL);
	if (!d)
		return -ENOMEM;

	d->clk = clk_get(&pdev->dev, NULL);
	if (IS_ERR(d->clk)) {
		ret = PTR_ERR(d->clk);
		goto err_free;
	}

	ret = clk_prepare_enable(d->clk);
	if (ret)
		goto err_clk_put;

	d->base = ioremap(res->start, resource_size(res));
	if (!d->base) {
		ret = -ENOMEM;
		goto err_clk_disable;
	}

	platform_set_drvdata(pdev, d);
	return 0;

err_clk_disable:
	clk_disable_unprepare(d->clk);
err_clk_put:
	clk_put(d->clk);
err_free:
	kfree(d);
	return ret;
}
```

```rust
// The Rust equivalent. Note: no goto ladder, no remove() teardown,
// and no possibility of getting the unwind order wrong.
fn probe(pdev: &mut platform::Device) -> Result<Pin<KBox<Self>>> {
    let clk = Clk::get(pdev.as_ref(), None)?;   // `?` on failure: nothing
                                                //  allocated yet to leak
    let clk = clk.prepare_enable()?;            // returns an EnabledClk whose
                                                //  Drop disables+puts it
    let base = pdev.ioremap_resource(0)?;       // Drop iounmaps

    let d = KBox::new(MyDev { clk, base }, GFP_KERNEL)?;
    Ok(d.into())
    // If KBox::new fails HERE, `clk` and `base` are dropped in reverse
    // declaration order -- iounmap, then disable, then put. Correct,
    // and written nowhere.
}
```

Do this translation for three real drivers from Part 2. The exercise that teaches the most is
**counting the error-path lines that disappear** — typically 25–40% of a probe function in C
is unwinding.

Notice also: there is no `remove()`. Dropping the returned box runs every field's `Drop` in
reverse order. The C version needs a `remove()` that mirrors `probe()` exactly, and the class
of bugs where it does not is large.

### Lab 78.2 — Build a small program that exercises the language

```rust
// SPDX-License-Identifier: GPL-2.0
//! A module that exercises the constructs you will meet in review.

use kernel::prelude::*;
use kernel::sync::{new_mutex, Arc, Mutex};

module! {
    type: LangTour,
    name: "lang_tour",
    author: "You",
    description: "Rust language constructs in a kernel context",
    license: "GPL",
}

/// An enum with data: a device state machine. Compare with a C struct
/// containing an enum tag and a union.
#[derive(Debug)]
enum DevState {
    Idle,
    Running { jobs: u32 },
    Failed(i32),
}

impl DevState {
    /// Exhaustive match: adding a variant breaks this at compile time.
    fn describe(&self) -> &'static str {
        match self {
            DevState::Idle                      => "idle",
            DevState::Running { jobs } if *jobs > 100 => "busy",
            DevState::Running { .. }            => "running",
            DevState::Failed(_)                 => "failed",
        }
    }
}

/// A trait used as a vtable.
trait Sink {
    fn emit(&self, v: u32) -> Result;
    /// Default method -- like a C ops struct with a fallback helper.
    fn emit_all(&self, vs: &[u32]) -> Result {
        for v in vs {
            self.emit(*v)?;         // `?` propagates the first failure
        }
        Ok(())
    }
}

struct LogSink { prefix: &'static str }

impl Sink for LogSink {
    fn emit(&self, v: u32) -> Result {
        if v == 0 {
            return Err(EINVAL);     // errno as a value
        }
        pr_info!("{}: {}\n", self.prefix, v);
        Ok(())
    }
}

/// Generic over any Sink: monomorphized, so this call is inlined.
fn drive<S: Sink>(s: &S) -> Result {
    s.emit_all(&[1, 2, 3])
}

/// Dynamic dispatch: a fat pointer. Use when the type varies at runtime.
fn drive_dyn(s: &dyn Sink) -> Result {
    s.emit_all(&[4, 5, 6])
}

struct LangTour {
    _state: Arc<Mutex<DevState>>,
}

impl kernel::Module for LangTour {
    fn init(_m: &'static ThisModule) -> Result<Self> {
        // --- Option: no NULL ---
        let v: KVec<u32> = KVec::new();
        match v.first() {
            Some(x) => pr_info!("first = {}\n", x),
            None    => pr_info!("empty vector\n"),
        }

        // --- let-else: the idiomatic early return ---
        let mut names: KVec<&str> = KVec::new();
        names.push("alpha", GFP_KERNEL)?;
        let Some(first) = names.first() else {
            pr_err!("no names\n");
            return Err(EINVAL);
        };
        pr_info!("first name: {}\n", first);

        // --- enums with data ---
        for st in [DevState::Idle,
                   DevState::Running { jobs: 5 },
                   DevState::Running { jobs: 500 },
                   DevState::Failed(-EIO.to_errno())] {
            pr_info!("state {:?} -> {}\n", st, st.describe());
        }

        // --- traits, static and dynamic ---
        let sink = LogSink { prefix: "static" };
        drive(&sink)?;
        let sink2 = LogSink { prefix: "dynamic" };
        drive_dyn(&sink2)?;

        // --- error propagation: this one fails, and `?` returns it ---
        match sink.emit(0) {
            Ok(())  => pr_info!("unexpected success\n"),
            Err(e)  => pr_info!("emit(0) failed as expected: {}\n",
                                e.to_errno()),
        }

        // --- iterators: no index, no off-by-one ---
        let squares: KVec<u32> = {
            let mut k = KVec::new();
            for i in 1u32..=5 {
                k.push(i * i, GFP_KERNEL)?;
            }
            k
        };
        let total: u32 = squares.iter().sum();
        pr_info!("squares {:?} sum {}\n", &*squares, total);

        // --- shared, locked state ---
        let state = Arc::pin_init(
            new_mutex!(DevState::Idle, "lang_tour::state"),
            GFP_KERNEL,
        )?;
        {
            let mut g = state.lock();
            *g = DevState::Running { jobs: 1 };
        }   // guard dropped here -> unlock. Cannot forget.
        pr_info!("state now {}\n", state.lock().describe());

        Ok(LangTour { _state: state })
    }
}

impl Drop for LangTour {
    fn drop(&mut self) {
        pr_info!("lang_tour: unloading\n");
    }
}
```

### Lab 78.3 — Read the error messages

Take the module above and break it deliberately, one line at a time. The exercise is to
**predict the error before compiling.**

```rust
// 1. Use a moved value
let a = KBox::new(1u32, GFP_KERNEL)?;
let b = a;
pr_info!("{}", a);
// predict: E0382 borrow of moved value

// 2. Two mutable borrows
let mut v = KVec::new();
v.push(1u32, GFP_KERNEL)?;
let r1 = &mut v;
let r2 = &mut v;
r1.push(2, GFP_KERNEL)?;
// predict: E0499 cannot borrow `v` as mutable more than once

// 3. Non-exhaustive match (add a variant to DevState, do not update describe)
// predict: E0004 non-exhaustive patterns

// 4. Ignoring a Result
sink.emit(1);
// predict: warning unused_must_use -- and in the kernel this is an error,
//          because #[must_use] on Result is what replaces __must_check

// 5. Returning a reference to a local
fn bad() -> &u32 { let x = 5; &x }
// predict: E0106 missing lifetime specifier, then E0515 returns reference
//          to local variable -- i.e. use-after-return, caught at compile time
```

**Reading rustc's errors fluently is the single highest-leverage skill for reviewing Rust
patches.** The messages name the exact borrow, the exact scope, and usually the fix.

### Lab 78.4 — Ownership across the FFI boundary

This is where C developers most often get confused, and it is the crux of reviewing bindings.

```rust
use kernel::types::ForeignOwnable;

// Pattern 1: hand ownership TO C, get it back later.
//   C equivalent: platform_set_drvdata / platform_get_drvdata
fn give_to_c(data: KBox<MyData>) -> *mut core::ffi::c_void {
    // into_foreign consumes the box and yields a raw pointer.
    // Rust now considers the value MOVED OUT -- it will not be dropped.
    data.into_foreign()
}

unsafe fn take_back_from_c(ptr: *mut core::ffi::c_void) -> KBox<MyData> {
    // SAFETY: `ptr` came from `into_foreign` on a KBox<MyData>, has not
    // been passed to `from_foreign` before, and C has relinquished it.
    unsafe { KBox::from_foreign(ptr) }
    // Ownership returns to Rust; it WILL be dropped at the end of scope.
}

// Pattern 2: BORROW from C, do not take ownership.
unsafe fn borrow_from_c<'a>(ptr: *mut core::ffi::c_void)
        -> <KBox<MyData> as ForeignOwnable>::Borrowed<'a> {
    // SAFETY: `ptr` is valid for at least 'a and C retains ownership.
    unsafe { KBox::<MyData>::borrow(ptr) }
    // Nothing is dropped. This is the common case in a callback.
}
```

The three rules to internalize, and to check in every review:

1. **`into_foreign` = "C now owns this."** Exactly one matching `from_foreign` must eventually
   happen, or it leaks.
2. **`borrow` = "C still owns this."** The result must not outlive the C object's lifetime,
   which is why it carries a lifetime parameter that you must justify.
3. **Every `from_foreign` needs a `// SAFETY:` comment naming the `into_foreign` that
   produced the pointer.** If you cannot name it, the code is wrong.

Now find these patterns in `samples/rust/rust_misc_device.rs` and in a real driver, and
verify each safety comment.

### Lab 78.5 — Iterators instead of index loops

```rust
// C:  for (i = 0; i < n; i++) if (a[i] > max) max = a[i];
let max = a.iter().copied().max().unwrap_or(0);

// C:  count matching entries
let n = devices.iter().filter(|d| d.is_active()).count();

// C:  build a list of transformed values, with error handling
let results: Result<KVec<u32>> = inputs
    .iter()
    .map(|x| transform(*x))       // each returns Result<u32>
    .try_fold(KVec::new(), |mut acc, r| {
        acc.push(r?, GFP_KERNEL)?;
        Ok(acc)
    });

// Iterate with index when you genuinely need it:
for (i, dev) in devices.iter().enumerate() {
    pr_info!("device {}: {}\n", i, dev.name());
}

// Two slices in lockstep:
for (src, dst) in input.iter().zip(output.iter_mut()) {
    *dst = *src * 2;
}
```

Iterators are zero-cost: they compile to the same loop. What you gain is that **off-by-one
and bounds errors are structurally impossible** — there is no index to get wrong. Verify it
yourself with the assembly diff from Lab 77.6.

### Lab 78.6 — Review a real Rust patch

```bash
cd $KDIR
# Find recent Rust changes:
git log --oneline --since=1.year -- rust/ drivers/ | grep -i rust | head -20

# Pick one and review it against this checklist:
git show <commit>
```

**The kernel Rust review checklist:**

- [ ] Does every `unsafe fn` have a `# Safety` doc section stating the caller's obligation?
- [ ] Does every `unsafe { }` block have a `// SAFETY:` comment that *discharges a specific
      stated invariant*, not just "this is fine"?
- [ ] Does every type with invariants have an `# Invariants` doc section?
- [ ] Any `unwrap()`, `expect()`, `panic!()`, or `[]` indexing on a runtime value? Those can
      panic, and a kernel panic is a BUG.
- [ ] Is every allocation fallible, with an explicit `GFP_` flag, and is the flag correct for
      the context?
- [ ] Are `Send`/`Sync` impls justified? An `unsafe impl Send` is a claim about cross-CPU
      safety and needs an argument.
- [ ] Are C pointers wrapped in `Opaque` where C may mutate or where initialization is C's
      job?
- [ ] Is `into_foreign`/`from_foreign` balanced, and is each `from_foreign` traceable to its
      `into_foreign`?
- [ ] Does the code sleep? If so, is it reachable from atomic context?
- [ ] Do `Drop` impls do the right thing on the *partial construction* path?

Practise on three real patches. Write the review you would send. This is directly what a
Rust-capable maintainer does, and the checklist is short enough to memorize.

---

## 3. Mastery drills

1. Translate five real C drivers' `probe`/`remove` pairs into Rust signatures (no need for
   working code — the resource types and their `Drop` behaviour). Count the error-path lines
   eliminated and produce the table.

2. Implement `Option` and `Result` yourself as plain enums, plus `?` via the `Try` trait.
   Then explain, with `size_of`, why `Option<&T>` is one word.

3. Take a C ops struct with eight function pointers and design the equivalent Rust trait.
   Handle the optional methods. Read `rust/macros/vtable.rs` and explain exactly how the
   generated code decides between `Some(f)` and `None`.

4. Write a function whose signature is only expressible with an explicit lifetime annotation,
   and one where elision suffices. Explain which elision rule applies in the second case.

5. Build a small state machine as an enum with data, with transitions as methods returning
   `Result<NewState>`. Then add a variant and observe every place that must be updated. Argue
   whether that is a feature or a burden at kernel scale.

6. Implement a safe wrapper over a C API that uses a callback with a `void *` context.
   Get the ownership right: who owns the context, when is it freed, and what happens if the
   callback fires after teardown?

7. Write three functions taking `T`, `&T`, and `&mut T`, and a caller that exercises all
   three. Then write the C equivalents and enumerate the invariants the C version relies on
   that nothing checks.

8. Find five uses of `unwrap()` or `expect()` anywhere in `rust/` (or prove there are none in
   reachable code) and explain, for each, why it is or is not acceptable.

9. Measure the code-size impact of monomorphization: instantiate a generic function over ten
   types and compare `.text` size against a `dyn`-dispatched version. State the rule you
   would apply for kernel code.

10. Read `rust/kernel/uaccess.rs` in full. Explain how `UserSlice` makes it impossible to
    dereference a user pointer, and compare with `sparse`'s `__user` checking — including
    what `sparse` catches that the type system does not, and vice versa.

11. Take a bug from a real Linux CVE that is a memory-safety bug. Write the equivalent Rust
    and determine whether the compiler would have rejected it, and if not, why not.

12. Write the "Rust for C kernel developers" one-page cheat sheet you wish you had had at the
    start of this chapter. Compare it with `reference/cheatsheets.md` and merge.

---

## 4. Further reading

**Kernel documentation**
- `Documentation/rust/coding-guidelines.rst` — the `# Safety` / `// SAFETY:` /
  `# Invariants` conventions; this *is* the review standard
- `make rustdoc` output for the `kernel` crate — searchable and always current
- `samples/rust/` — `rust_minimal.rs`, `rust_print.rs`, `rust_misc_device.rs`

**Books**
- Klabnik & Nichols, *The Rust Programming Language* — free. Ch. 4, 6, 10, 13, 15, 17, 18
  are the ones that matter here
- *Rust by Example* — free; good for quick syntax lookup
- Blandy, Orendorff, Tindall, *Programming Rust*, 2nd ed. — the systems programmer's Rust
  book; the best treatment of ownership for people who already think in pointers
- *The Rustonomicon* — free; read before writing any abstraction
- Gjengset, *Rust for Rustaceans* — the intermediate book; the chapters on types, traits, and
  unsafe are directly applicable

**References**
- The Rust Reference (`doc.rust-lang.org/reference/`) — the language definition
- The `rustc` error index (`doc.rust-lang.org/error_codes/`) — look up E0382, E0499, E0502,
  E0515, E0597 and understand each fully; they are 90% of what you will hit
- `std`/`core` documentation — even though `std` is unavailable, `core`'s docs are the same

**Talks**
- Niko Matsakis on the borrow checker and NLL
- "Rust: A Language for the Next 40 Years" (Carol Nichols)
- Kangrejos talks on the `kernel` crate's design decisions

→ Next: [79-kernel-crate.md](79-kernel-crate.md)

# Chapter 77 — Rust Foundations: Why, and How It Builds

> **Part 5 — Rust for Linux.** Nine chapters. The goal is not to make you a Rust programmer;
> it is to make you a kernel engineer who can read, review, and write kernel Rust, and who
> can argue about it at an architectural level. You already know the C dialect (Ch. 08), the
> driver model (Ch. 26–28), and the locking rules (Ch. 14–16). Rust is a different way of
> *encoding* those same rules — in the type system rather than in documentation and review.

---

## Theory & First Principles

### T.0 — Start here: the same bug, twenty years apart

```c
struct my_dev *dev = find_dev(id);   /* returns a pointer */
spin_unlock(&list_lock);
use(dev);                            /* the object may already be freed */
```

You met this in Ch. 12 §T.0 (refcounted lookup), again in Ch. 29 §T.0 (the device's two
lifetimes), and again in Ch. 65 §T.0 (completing a request twice). **It is the same bug every
time, and the kernel has been fixing instances of it continuously since 1991.**

**The empirical case for this part of the book is one number:** across Microsoft's, Google's,
and Mozilla's independent analyses of their own large C/C++ codebases, roughly **70% of
security-critical vulnerabilities are memory-safety errors** — use-after-free, buffer
overflow, double free, uninitialized read, data race. In Linux specifically, CVE analyses put
drivers at the majority of the count.

**And the reason is not that kernel developers are careless.** It is structural:

| The rule | Who enforces it in C |
|---|---|
| "free this exactly once" | you, by convention |
| "do not use it after freeing" | you, by remembering |
| "hold the lock when touching this field" | you, by reading a comment |
| "this pointer must not outlive the structure" | you, by being careful |
| "this function may not sleep" | you, by knowing the context |

**Every one of those is a *proof obligation carried in a human's head*.** Review catches most
of them. Sanitizers (Ch. 06) catch some at runtime. Neither scales to 30 million lines and
4,000 contributors per release.

**Rust's claim is narrow and should be stated precisely, because the overstated version is
wrong and invites bad arguments:**

> **Rust moves a specific set of proof obligations from the programmer's head into the type
> system, where the compiler checks them mechanically.** Specifically: ownership (who frees),
> lifetimes (how long a reference stays valid), aliasing (shared XOR mutable), and
> thread-safety (`Send`/`Sync`). It does not eliminate logic bugs, deadlocks, leaks, or the
> need to understand the hardware.

**Now the move that makes this work, and it is the one to actually understand:**

```
   The rule:  at any moment, a value has EITHER
                one mutable reference  (&mut T)
              OR any number of shared references (&T)
              -- NEVER both.
```

That single rule, checked statically, makes **use-after-free, double-free, iterator
invalidation, and data races unrepresentable** — not "unlikely," but rejected at compile
time. A data race requires two accesses, at least one a write, unsynchronized; the aliasing
rule forbids exactly that configuration.

**And notice the recurring principle of this whole book, carried to its conclusion:** "when a
correctness obligation is routinely forgotten, move it into the primitive" (Ch. 05, 28, 34,
52, 82). `devm_` did it for one resource class by making cleanup automatic. Ownership and
RAII do it for *every* resource, and make forgetting a **compile error** rather than a
runtime leak.

**Two honest counterweights before §T.1:**

1. **`unsafe` does not go away in a kernel.** MMIO, DMA, page tables, and every C FFI call are
   inherently unverifiable by the compiler. Rust's value is not that unsafe code vanishes — it
   is that unsafe code becomes **a small, greppable, reviewable minority** with an explicitly
   stated invariant, instead of being 100% of the codebase (Ch. 84).
2. **The costs are real:** a second toolchain, a hard learning curve, an unstable set of
   kernel abstractions, limited architecture support, and a bindings layer that must be
   maintained in lockstep with C APIs that change freely. Ch. 85 argues this honestly rather
   than as advocacy.

```bash
rustc --version && cargo --version
make LLVM=1 rustavailable                  # does this tree build Rust?
ls rust/kernel/                            # the abstractions, in the tree
grep -rn 'unsafe' rust/kernel/ | wc -l     # how much is unsafe, really?
```

---

### T.1 — The argument, in one paragraph

Roughly **two thirds of serious kernel CVEs are memory-safety bugs** — use-after-free,
out-of-bounds, double free, data races, uninitialized use. That figure is consistent across
Microsoft's analysis of Windows (~70%), Google's of Chrome (~70%) and Android (~65–75%
falling to ~24% of new code as Rust adoption grew), and the Linux CVE record. These are not
exotic bugs; they are the *same five mistakes* made repeatedly by competent engineers over
thirty years.

Rust's claim is narrow and checkable: **in safe Rust, those five classes are impossible by
construction**, verified at compile time, with no runtime cost. It does not prevent logic
errors, deadlocks, resource leaks, or misusing hardware. It prevents exactly the class that
dominates the CVE record.

The interview-grade framing: *every other item on a kernel-hardening list is a tradeoff —
KPTI costs 5–30%, CFI costs 1–5%, `init_on_free` costs 1–5%. Rust is the only intervention
that removes a bug class at **zero runtime cost**. That asymmetry, not performance or
fashion, is the argument.*

### T.2 — What the borrow checker actually enforces

Three rules. Everything else follows.

**1. Ownership.** Every value has exactly one owner. When the owner goes out of scope, the
value is dropped. There is no `free()` to forget and no `free()` to call twice.

```rust
{
    let b = KBox::new(Foo::new(), GFP_KERNEL)?;   // b owns the allocation
    use_it(&b);
}   // b dropped here, memory freed. Automatically. Always. On every path,
    // including early returns and `?` propagation.
```

This is RAII, the same idea as `devm_` (Ch. 28) and `guard()` (Ch. 05) — but enforced by the
compiler rather than by remembering to use the right helper.

**2. Borrowing.** You may have **either** any number of shared references `&T` **or**
exactly one mutable reference `&mut T` — never both, never two mutable.

```rust
let mut v = KVec::new();
let r1 = &v;        // shared borrow
let r2 = &v;        // another shared borrow -- fine
let m  = &mut v;    // ERROR: cannot borrow as mutable while borrowed as immutable
```

This single rule eliminates:
- **Iterator invalidation** — you cannot push to a vector while iterating it.
- **Aliasing bugs** — a function taking `&mut T` knows nothing else can see `T`.
- **Data races** — because a race requires two references, at least one mutable.

And it gives the optimizer information C cannot express: `&mut T` is `noalias`, always, for
free.

**3. Lifetimes.** A reference may not outlive the thing it refers to. The compiler proves
this; the lifetime annotations (`'a`) are how you *state* relationships it cannot infer.

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str { ... }
// "the returned reference lives at most as long as the shorter of x and y"
```

**Use-after-free is a lifetime violation, and lifetimes are checked at compile time.** That
is the whole trick.

### T.3 — What `unsafe` means, and what it does not

The single most misunderstood point, and the one to get right in an interview.

`unsafe` does **not** turn off the borrow checker. It unlocks exactly five additional
abilities:

1. Dereference a raw pointer (`*const T`, `*mut T`).
2. Call an `unsafe` function (including all FFI / C functions).
3. Access or modify a mutable `static`.
4. Implement an `unsafe` trait (`Send`, `Sync`).
5. Access fields of a `union`.

Everything else — ownership, borrowing, lifetimes, type checking — still applies inside an
`unsafe` block.

The correct mental model:

> `unsafe` is not "unsafe code." It is **"code where I, the author, am responsible for an
> invariant the compiler cannot check."** It is a *proof obligation marker*.

Which leads to the kernel's actual discipline, and the thing reviewers enforce:

```rust
/// # Safety
///
/// `ptr` must point to a valid, initialized `Foo` that remains valid for
/// the duration of `'a`, and no other mutable reference to it may exist.
unsafe fn from_raw<'a>(ptr: *mut Foo) -> &'a mut Foo {
    // SAFETY: the caller guarantees `ptr` is valid and uniquely owned for
    // `'a`, per the contract above.
    unsafe { &mut *ptr }
}
```

**Every `unsafe fn` carries a `# Safety` section stating the caller's obligation. Every
`unsafe` block carries a `// SAFETY:` comment discharging it.** This is mandatory in
`rust/` — `checkpatch` and reviewers enforce it. The value is that the audit surface is
*enumerable*: `grep -c unsafe` gives you the size of the code you must review by hand,
where in C the answer is "all of it."

### T.4 — The architectural pattern: safe abstractions over unsafe primitives

This is the design shape of the entire `kernel` crate and the thing to understand
structurally:

```
 ┌───────────────────────────────────────────────────────┐
 │  Driver code — 100% SAFE Rust                         │  <- most code
 │  Cannot cause UB, no matter what it does              │
 ├───────────────────────────────────────────────────────┤
 │  `kernel` crate — safe API, unsafe internals          │  <- audited once
 │  Each `unsafe` discharged against a documented C       │
 │  contract; the safety argument is written down         │
 ├───────────────────────────────────────────────────────┤
 │  `bindings` crate — raw, machine-generated FFI         │  <- all unsafe
 │  Produced by bindgen from the C headers                │
 ├───────────────────────────────────────────────────────┤
 │  The C kernel                                          │
 └───────────────────────────────────────────────────────┘
```

A device driver written against the `kernel` crate contains **zero `unsafe`** in the normal
case. The `unsafe` was paid once, by the abstraction author, and reviewed by people who
understand both sides of the boundary. Then a thousand drivers inherit the correctness.

This is the same principle the curriculum keeps returning to — *when a correctness
obligation is routinely forgotten, move it into the primitive* (`devm_` in Ch. 28, folios in
Ch. 52, `mmiowb` folding in Ch. 34, `guard()` in Ch. 05). Rust simply makes the move
**enforceable** rather than advisory.

### T.5 — What Rust does *not* fix

State these unprompted; it is the difference between an advocate and an engineer.

| Not prevented | Why |
|---|---|
| **Deadlock** | Lock ordering is not a type-level property. `lockdep` is still your tool |
| **Memory leaks** | Leaking is *safe* in Rust. `Arc` cycles leak exactly as in C |
| **Logic errors** | The type system checks types, not intent |
| **Incorrect hardware programming** | Writing the wrong value to a register is well-typed |
| **Integer overflow in release builds** | Wraps (panics in debug); use `checked_*`/`saturating_*` |
| **Panics** | A panic in the kernel is a BUG. Kernel Rust must avoid panicking paths entirely |
| **Unsafe code's own bugs** | The `unsafe` blocks are exactly as dangerous as C |
| **Side channels** | Timing is not a type |

The panic issue is important and specific: userspace Rust panics freely (`unwrap()`,
`expect()`, array indexing, integer division). The kernel cannot unwind. So kernel Rust:
- compiles with `panic=abort` and `-Cforce-unwind-tables=n`,
- forbids `unwrap()`/`expect()` in review,
- uses `Result` everywhere with `?` propagation,
- uses **fallible allocation** — `KBox::new(x, GFP_KERNEL)?` returns `Result`, unlike
  userspace `Box::new` which aborts on OOM. This is one of the largest differences from
  standard Rust and the reason kernel Rust uses `alloc` only in a heavily modified form.

### T.6 — Why `core` and not `std`

`#![no_std]`. The standard library assumes an OS: heap, threads, files, `println!`, unwinding.

| Crate | Provides | In kernel? |
|---|---|---|
| `core` | Language fundamentals: `Option`, `Result`, iterators, traits, slices, primitives. No allocation | **Yes** |
| `alloc` | `Box`, `Vec`, `String`, `Arc` — requires a heap, **aborts on OOM** | Partially / replaced |
| `std` | Everything else: OS services | **No** |

The kernel supplies its own: `KBox`, `KVec`, `Arc`, `CString` in the `kernel` crate, all
**fallible** — allocation returns `Result<_, AllocError>` and takes an explicit `GFP_` flag,
because `GFP_KERNEL` vs `GFP_ATOMIC` is a real decision the kernel must make (Ch. 11 §T.5)
and the type system should carry it.

```rust
// Userspace Rust:    aborts the process on OOM
let b = Box::new(Foo);
// Kernel Rust:       explicit context, explicit failure
let b = KBox::new(Foo, GFP_KERNEL)?;
```

### T.7 — Status, honestly

| Milestone | When |
|---|---|
| RFC and initial support merged | **6.1** (Dec 2022) — infrastructure only, no drivers |
| `pin-init`, `Arc`, `SpinLock`, `Mutex` | 6.2–6.6 |
| Binder rewrite in Android | 6.x, shipping in production |
| `rnull` null block driver | 6.x, in-tree |
| Nova (NVIDIA GSP) DRM driver | active development, the flagship |
| PHY drivers, Asix/Ath, misc drivers | merged incrementally |
| Stated policy: Rust for *new* drivers welcomed | 2024 onward |

The kernel's position, as stated by Torvalds and the maintainers: **Rust is not replacing C.**
The 30 million existing lines stay. Rust is for *new* code, especially new drivers, where
the memory-safety argument has the highest value and the least migration cost.

The genuine friction points, which you should be able to name:
- **No stable ABI between Rust and C**, and Rust itself has no stable ABI — so the whole
  tree must be built with one toolchain.
- **Minimum supported Rust version** churn; the kernel now pins a version and moves
  deliberately.
- **Maintainer burden**: a C maintainer who does not read Rust now has code in their
  subsystem they cannot review. This is a *social* problem, and it has produced real
  friction on the mailing lists — and it is the honest answer to "what is the hardest part?"
- **`bindgen` fragility** with complex headers, inline functions, and macros.

---

## 1. Internals

### Source map

| Path | Contents |
|---|---|
| `rust/` | Everything Rust |
| `rust/kernel/` | **The `kernel` crate** — the safe abstraction layer |
| `rust/kernel/lib.rs` | Crate root; the module list is a map of what is abstracted |
| `rust/kernel/prelude.rs` | What every kernel Rust file imports |
| `rust/kernel/error.rs` | `Error`, `Result`, `to_result()`, errno mapping |
| `rust/kernel/alloc/` | `KBox`, `KVec`, `Allocator`, `Kmalloc`/`Vmalloc`/`KVmalloc` |
| `rust/kernel/sync/` | `Arc`, `SpinLock`, `Mutex`, `CondVar`, `LockedBy` |
| `rust/kernel/init.rs`, `rust/pin-init/` | `pin-init`: in-place initialization |
| `rust/kernel/types.rs` | `Opaque`, `ARef`, `AlwaysRefCounted`, `ForeignOwnable` |
| `rust/bindings/` | **Machine-generated** bindings; `bindings_helper.h` selects headers |
| `rust/helpers/` | C shims for things bindgen cannot handle (static inlines, macros) |
| `rust/macros/` | Proc macros: `module!`, `vtable`, `pin_data`, `pinned_drop` |
| `rust/uapi/` | Bindings for UAPI headers, kept separate on purpose |
| `rust/compiler_builtins.rs` | The handful of intrinsics `core` needs |
| `Documentation/rust/` | `quick-start.rst`, `general-information.rst`, `coding-guidelines.rst` |
| `scripts/generate_rust_analyzer.py` | Generates `rust-project.json` for IDE support |
| `samples/rust/` | Minimal in-tree examples — the best starting code |

### The build pipeline

```
 .rs source
   │
   ├─ rustc --edition 2021 -Zbinary-dep-depinfo ... --emit=obj
   │     │
   │     ├─ core (prebuilt or built from rust-src)
   │     ├─ kernel crate      <-- built once per kernel build
   │     └─ bindings crate    <-- generated by bindgen from C headers
   │
   ▼
 .o  ──► same as any C object: ld, modpost, MODULE_INFO, .ko
```

Key facts:
- **`bindgen` runs at build time** against `rust/bindings/bindings_helper.h`, which
  `#include`s the C headers to expose. If a symbol is not reachable from that header, Rust
  cannot see it — adding a binding means adding an `#include` there.
- **Static inline functions and macros are invisible to bindgen.** They get hand-written C
  shims in `rust/helpers/helpers.c` (`rust_helper_spin_lock()` etc.), which is why that file
  exists and why it grows.
- The output is an ordinary ELF object. **`modpost`, `MODULE_LICENSE`, symbol versioning,
  and `insmod` all work unchanged** — a Rust module is not special to the module loader.
- `CONFIG_RUST` depends on `RUST_IS_AVAILABLE`, which `scripts/rust_is_available.sh` computes
  by checking `rustc`, `bindgen`, and `libclang` versions.

### The `module!` macro

```rust
module! {
    type: MyModule,
    name: "my_module",
    author: "Your Name",
    description: "A sample kernel module in Rust",
    license: "GPL",
}
```

This expands to: the `.modinfo` section entries (exactly the same strings
`MODULE_LICENSE`/`MODULE_AUTHOR` produce in C), an `init_module`/`cleanup_module` pair with
C ABI, and the glue that constructs your type and stores it in a static. The lifecycle is
`Module::init()` returning `Result<Self>`, and `Drop::drop()` as the exit path — **so module
cleanup is RAII and cannot be forgotten.**

---

## 2. Practice

### Lab 77.1 — Get a Rust-capable kernel building

```bash
#!/bin/bash
# rust_setup.sh — from zero to a kernel with CONFIG_RUST=y
set -e
KDIR=${1:-$HOME/src/linux}

echo "=== 1. Install rustup and the pinned toolchain ==="
command -v rustup >/dev/null || \
	curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"

cd "$KDIR"
# The kernel pins an exact version. Ask it which.
RUSTVER=$(scripts/min-tool-version.sh rustc)
BINDGENVER=$(scripts/min-tool-version.sh bindgen)
echo "kernel wants rustc $RUSTVER, bindgen $BINDGENVER"

rustup toolchain install "$RUSTVER"
rustup component add rust-src --toolchain "$RUSTVER"
rustup override set "$RUSTVER"

echo "=== 2. bindgen and libclang ==="
cargo install --locked --version "$BINDGENVER" bindgen-cli || true
# Debian/Ubuntu:  sudo apt install libclang-dev clang llvm
# Fedora:         sudo dnf install clang-devel llvm-devel

echo "=== 3. Ask the kernel whether it is satisfied ==="
make LLVM=1 rustavailable
# On failure this prints EXACTLY what is missing. Read it; do not guess.

echo "=== 4. Configure ==="
make LLVM=1 defconfig
./scripts/config --enable RUST
./scripts/config --enable SAMPLES
./scripts/config --enable SAMPLES_RUST
./scripts/config --module SAMPLE_RUST_MINIMAL
./scripts/config --module SAMPLE_RUST_PRINT
./scripts/config --enable RUST_DEBUG_ASSERTIONS
make LLVM=1 olddefconfig
grep -E '^CONFIG_RUST' .config

echo "=== 5. Build ==="
make LLVM=1 -j"$(nproc)"

echo "=== 6. Documentation and IDE support ==="
make LLVM=1 rustdoc                     # -> Documentation/output/rust/rustdoc/
make LLVM=1 rust-analyzer               # -> rust-project.json for your editor
ls samples/rust/*.ko
```

**Use `LLVM=1`.** Building the C parts with clang and the Rust parts with rustc means both
frontends share LLVM, which avoids a class of ABI and LTO mismatches. GCC + rustc works but
is the less-travelled path.

If `make rustavailable` fails, read its output literally — it names the exact missing
component and version. The three usual causes: no `rust-src` component, `bindgen` version
mismatch, and `libclang` not found (set `LIBCLANG_PATH`).

### Lab 77.2 — The minimal module, read line by line

```rust
// SPDX-License-Identifier: GPL-2.0
//! A minimal Rust kernel module.
//!
//! Build in-tree: place at samples/rust/rust_hello.rs, add to
//! samples/rust/Makefile and Kconfig, then `make LLVM=1 M=samples/rust`.

use kernel::prelude::*;

module! {
    type: RustHello,
    name: "rust_hello",
    author: "You",
    description: "Minimal Rust module",
    license: "GPL",
}

/// Module state. Anything the module owns lives here; when the module is
/// unloaded, this is dropped, and `Drop` runs. There is no separate
/// "exit function" to forget to write.
struct RustHello {
    numbers: KVec<i32>,
}

impl kernel::Module for RustHello {
    fn init(_module: &'static ThisModule) -> Result<Self> {
        pr_info!("rust_hello: loading\n");

        // Fallible allocation. Note the explicit GFP flag and the `?`.
        // Userspace Rust's Vec::push cannot fail; ours returns Result.
        let mut numbers = KVec::new();
        for i in 0..8 {
            numbers.push(i * i, GFP_KERNEL)?;
        }

        pr_info!("rust_hello: squares = {:?}\n", &*numbers);

        // Returning Err here means init failed; nothing partially built
        // leaks, because everything constructed so far is dropped.
        Ok(RustHello { numbers })
    }
}

impl Drop for RustHello {
    fn drop(&mut self) {
        pr_info!("rust_hello: unloading, had {} numbers\n",
                 self.numbers.len());
        // `numbers`' allocation is freed here. Automatically.
    }
}
```

The five things to notice, each contrasted with C:

| Rust | C equivalent, and what can go wrong |
|---|---|
| `module!` macro | `MODULE_LICENSE` etc. — same output, but the type and init are tied together |
| `fn init(...) -> Result<Self>` | `static int __init x_init(void)` — but in C, a failure after partial setup must unwind manually |
| `KVec::push(x, GFP_KERNEL)?` | `krealloc` + a NULL check you might forget |
| `impl Drop` | `module_exit` — but Drop runs for *every* owned field, recursively, automatically |
| No `unsafe` anywhere | — |

### Lab 77.3 — Make the compiler reject the classic bugs

```rust
// SPDX-License-Identifier: GPL-2.0
//! Deliberately-broken code. Each block should FAIL to compile.
//! Uncomment one at a time and read the error message carefully --
//! the errors are the lesson.

use kernel::prelude::*;

fn use_after_free() {
    let r;
    {
        let b = KBox::new(42, GFP_KERNEL).unwrap();
        r = &*b;
    }   // b dropped here
    // pr_info!("{}", r);
    // error[E0597]: `b` does not live long enough
    //   ==> USE AFTER FREE, caught at compile time.
    let _ = r;
}

fn double_free() {
    let b = KBox::new(42, GFP_KERNEL).unwrap();
    let c = b;          // ownership MOVED to c
    // pr_info!("{}", b);
    // error[E0382]: borrow of moved value: `b`
    //   ==> There is no way to free twice: only one owner exists.
    drop(c);
}

fn iterator_invalidation() {
    let mut v: KVec<i32> = KVec::new();
    v.push(1, GFP_KERNEL).unwrap();
    for _x in v.iter() {
        // v.push(2, GFP_KERNEL).unwrap();
        // error[E0502]: cannot borrow `v` as mutable because it is also
        //               borrowed as immutable
        //   ==> The realloc that would dangle the iterator is impossible.
    }
}

fn out_of_bounds() {
    let v: [i32; 4] = [1, 2, 3, 4];
    let i = 10;
    // let _x = v[i];
    // Compiles, but PANICS at runtime -- and a kernel panic is a BUG.
    // The correct kernel-Rust form:
    match v.get(i) {
        Some(x) => pr_info!("got {}\n", x),
        None    => pr_info!("index {} out of range\n", i),
    }
}

fn uninitialized() {
    let x: i32;
    // pr_info!("{}", x);
    // error[E0381]: used binding `x` isn't initialized
    let _ = x;
}
```

The pedagogical point: **run these and read the errors.** Rust's diagnostics name the exact
line where the value dies and the exact line where you used it. Compare with what these bugs
look like in C: nothing at compile time, a KASAN report hours later if you are lucky, a
silent corruption if you are not (→ `reference/debugging-scenarios.md` §4).

### Lab 77.4 — Look at what `unsafe` actually costs you

```rust
// SPDX-License-Identifier: GPL-2.0
//! Writing a safe abstraction over an unsafe primitive -- the pattern
//! that the whole `kernel` crate is built from.

use kernel::prelude::*;
use core::ptr::NonNull;

/// A safe wrapper around a raw pointer to a `T` that we own.
///
/// # Invariants
///
/// `ptr` always points to a valid, initialized, uniquely-owned `T`
/// allocated by `KBox::into_raw`.
pub struct MyBox<T> {
    ptr: NonNull<T>,
}

impl<T> MyBox<T> {
    /// Allocate and take ownership. Entirely safe to call.
    pub fn new(value: T, flags: kernel::alloc::Flags) -> Result<Self> {
        let b = KBox::new(value, flags)?;
        let raw = KBox::into_raw(b);
        // INVARIANT: KBox::into_raw returns a valid, uniquely-owned,
        // non-null pointer to an initialized T.
        // SAFETY: `raw` is non-null for the reason above.
        Ok(Self { ptr: unsafe { NonNull::new_unchecked(raw) } })
    }

    pub fn get(&self) -> &T {
        // SAFETY: by the type invariant, `ptr` is valid and initialized,
        // and `&self` guarantees no mutable alias exists.
        unsafe { self.ptr.as_ref() }
    }

    pub fn get_mut(&mut self) -> &mut T {
        // SAFETY: by the type invariant, `ptr` is valid and initialized,
        // and `&mut self` guarantees exclusive access.
        unsafe { self.ptr.as_mut() }
    }
}

impl<T> Drop for MyBox<T> {
    fn drop(&mut self) {
        // SAFETY: by the type invariant we uniquely own this allocation,
        // it came from KBox::into_raw, and `drop` runs exactly once.
        drop(unsafe { KBox::from_raw(self.ptr.as_ptr()) });
    }
}

// SAFETY: MyBox owns its T exclusively, so sending it across threads is
// sound exactly when T can be sent.
unsafe impl<T: Send> Send for MyBox<T> {}
// SAFETY: &MyBox<T> only hands out &T, so sharing is sound exactly when
// &T can be shared.
unsafe impl<T: Sync> Sync for MyBox<T> {}
```

Count the `unsafe` blocks: five, each with a one-sentence justification referring to a stated
invariant. Now every *user* of `MyBox` writes zero `unsafe` and cannot misuse it. **That
ratio — five audited lines versus unbounded safe usage — is the economic argument for the
abstraction layer.**

Note the two `unsafe impl` lines at the bottom. `Send` and `Sync` are the type-level encoding
of "can this cross CPUs / be shared between them," which in C is a comment at best. Ch. 81
develops this.

### Lab 77.5 — Read the generated bindings

```bash
cd $KDIR
# Where bindgen's output actually lands:
find . -name 'bindings_generated.rs' | head
wc -l $(find . -name 'bindings_generated.rs' | head -1)

# What does a C function look like after binding?
grep -A6 'pub fn kmalloc' $(find . -name 'bindings_generated.rs' | head -1)
grep -A20 'pub struct device' $(find . -name 'bindings_generated.rs' | head -1)

# What headers are exposed? THIS is the file you edit to add a binding.
cat rust/bindings/bindings_helper.h

# Static inlines and macros bindgen cannot see get C shims here:
grep -n 'rust_helper' rust/helpers/*.c | head -30

# Browse the safe API instead:
make LLVM=1 rustdoc
xdg-open Documentation/output/rust/rustdoc/kernel/index.html
```

Compare a raw binding with its safe wrapper. For example `bindings::spin_lock_init` (unsafe,
takes a raw pointer, no lifetime information) versus `kernel::sync::SpinLock` (safe, owns its
data, releases on scope exit, cannot be used before initialization). **Writing out that
comparison for three primitives is the fastest way to understand what the `kernel` crate is
for.**

### Lab 77.6 — Prove the zero-cost claim

```bash
# Build the same trivial logic in C and Rust and diff the assembly.
cd $KDIR

# 1. Rust: a bounds-checked slice sum
cat > /tmp/sum.rs <<'EOF'
#![no_std]
#[no_mangle]
pub fn sum(v: &[i32]) -> i32 {
    let mut t = 0;
    for x in v { t += *x; }
    t
}
EOF
rustc --edition 2021 -O --emit asm --crate-type lib -o /tmp/sum_rs.s /tmp/sum.rs

# 2. C equivalent
cat > /tmp/sum.c <<'EOF'
int sum(const int *v, unsigned long n) {
    int t = 0;
    for (unsigned long i = 0; i < n; i++) t += v[i];
    return t;
}
EOF
clang -O2 -S -o /tmp/sum_c.s /tmp/sum.c

diff <(grep -v '^\s*\.' /tmp/sum_rs.s) <(grep -v '^\s*\.' /tmp/sum_c.s) || true
```

The iterator, the slice bounds, and the ownership all vanish — Rust's abstractions are
compiled away. Then try the version that *cannot* be proven in bounds (indexing with a
runtime value) and see the bounds check appear. **That is the honest picture: zero-cost where
the compiler can prove it, an explicit check where it cannot — and you choose, via `get()`
versus `[]`, whether the failure is a `None` or a panic.**

---

## 3. Mastery drills

1. Build a kernel with `CONFIG_RUST=y` from scratch on your machine, documenting every
   toolchain problem you hit and its fix. Then write the setup script you would give a new
   team member.

2. Read `rust/kernel/lib.rs` and produce a table of every module in the `kernel` crate, what
   C subsystem it abstracts, and how complete the abstraction is. This is the map of what
   you can and cannot write in Rust today.

3. Take the minimal module and add, one at a time: a module parameter, a `/proc` entry, a
   debugfs file. For each, determine whether the abstraction exists; where it does not,
   describe what writing it would require.

4. Count `unsafe` blocks in `rust/kernel/` and categorize them by what invariant each
   discharges (FFI call, raw pointer deref, `Send`/`Sync`, union access). Report the ratio of
   `unsafe` lines to total lines, and compare it with the ratio you would get for equivalent
   C (where it is 1.0 by definition).

5. Write a safe abstraction over a C kernel API that does not yet have one (pick a small
   one). Write the `# Safety` and `// SAFETY:` documentation to the standard in
   `Documentation/rust/coding-guidelines.rst`, then have someone try to break it.

6. Demonstrate each of the five memory-safety bug classes in C, build with KASAN, and show
   the report. Then write the equivalent in Rust and show the compile error. Produce the
   side-by-side table — this is the artifact to bring to an interview.

7. Investigate `panic=abort` in the kernel: find where it is configured, what `core`'s panic
   handler does in `rust/kernel/`, and enumerate the operations in safe Rust that can panic.
   Write the review checklist for avoiding them.

8. Compare compile times: build `samples/rust` and an equivalently-sized C sample. Measure
   incremental and full rebuild times. Then form a view on whether the cost is acceptable at
   the scale of a subsystem.

9. Read the LKML thread for the original Rust-for-Linux merge and one of the later friction
   threads (e.g. the DMA API or the maintainer-burden discussions). Summarize the strongest
   argument on each side. Do not pick a winner — state the conditions under which each is
   right.

10. Determine what happens when a Rust module and a C module share a data structure. Write
    one of each that operate on the same C `struct`, and document every invariant that is
    now unenforceable because half the code is C.

11. Trace a single `pr_info!` call from Rust source to the C `printk`. Identify every layer:
    the macro, the format machinery, the binding, the helper, the C function. Count the
    layers and assess whether any is removable.

12. Take a real in-tree Rust driver (`drivers/block/rnull.rs` or a PHY driver) and review it
    as if it were submitted to you. Check every `unsafe` block against its stated invariant.
    Write the review you would send.

---

## 4. Further reading

**Kernel documentation**
- `Documentation/rust/quick-start.rst` — the setup guide; follow it exactly
- `Documentation/rust/general-information.rst` — how Rust fits into the build
- `Documentation/rust/coding-guidelines.rst` — **the style and `# Safety` rules**; read this
  before writing any kernel Rust
- `Documentation/rust/arch-support.rst` — which architectures work
- `make rustdoc` output — the generated API docs for the `kernel` crate; the single most
  useful reference once you are writing code

**Primary sources**
- The Rust for Linux project: `rust-for-linux.com`, and the `Rust-for-Linux/linux` tree
- Miguel Ojeda's RFC and the LKML merge thread (2021), and the Kangrejos workshop reports
- Android's Rust-in-the-platform reports — the source of the "24% of new code" memory-safety
  figure and the best real-world evidence

**Books**
- Klabnik & Nichols, *The Rust Programming Language* ("the book") — free online. Chapters
  4 (ownership), 10 (lifetimes), 15 (smart pointers), 16 (concurrency) are the load-bearing
  ones for kernel work
- *The Rustonomicon* — free; the book about `unsafe`. Required reading before writing an
  abstraction
- *Rust Atomics and Locks* (Mara Bos) — free online; the best explanation of `Send`/`Sync`
  and the memory model, and it maps directly onto Ch. 13
- Blandy, Orendorff, Tindall, *Programming Rust*, 2nd ed. — the best systems-oriented Rust
  book

**Papers and reports**
- Microsoft MSRC, "A proactive approach to more secure code" (2019) — the ~70% figure
- Google Security Blog, "Eliminating Memory Safety Vulnerabilities at the Source" (2024) —
  the Android data, and the argument that *new* code is what matters
- Jung et al., "RustBelt: Securing the Foundations of the Rust Programming Language,"
  *POPL*, 2018 — the formal proof that Rust's type system is sound, including `unsafe`
  encapsulation
- Astrauskas et al., "How Do Programmers Use Unsafe Rust?," *OOPSLA*, 2020 — empirical data
  on how `unsafe` is actually used

**LWN**
- "Rust for Linux" tag archive — the complete history
- "Coming to terms with Rust in the kernel" and the maintainer-burden discussions
- "A pair of Rust kernel modules" and the driver-by-driver merge coverage
- Kangrejos conference reports, annually

→ Next: [78-rust-for-c-devs.md](78-rust-for-c-devs.md)

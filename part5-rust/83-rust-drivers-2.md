# Chapter 83 — Writing Rust Drivers II: PCI, MMIO, DMA, IRQ, Devres

> Chapter 82 covered the non-discoverable, memory-mapped case. This one covers the hard
> parts: a discoverable bus with BARs and capabilities, DMA with its ownership rules, and
> interrupts with their context rules. These are where Rust's guarantees are most valuable
> — and where they most visibly stop, because **the compiler cannot reason about what the
> device does.**

---

## Theory & First Principles

### T.0 — Start here: where exactly does the guarantee stop?

Write a Rust driver and you will hear "it is memory safe." **Be precise about what that
sentence covers**, because an imprecise version of it is actively dangerous.

```
  +-------------------------------------------------------------+
  |  Your safe Rust driver code                                 |
  |    guaranteed: no UAF, no double free, no OOB, no data race |  <- the claim
  +-------------------------------------------------------------+
  |  kernel crate abstractions (unsafe inside, audited)          |
  +-------------------------------------------------------------+
  |  bindgen bindings                                            |
  +======= THE GUARANTEE ENDS HERE ==============================+
  |  C kernel code -- may pass you a NULL, may free your object,  |
  |  may call your callback from IRQ context, may not hold the   |
  |  lock its comment says it holds                              |
  +-------------------------------------------------------------+
  |  THE HARDWARE -- a DMA engine writing into your memory       |
  |  concurrently, with no regard for any type system at all     |
  +-------------------------------------------------------------+
```

**Three categories of thing the compiler cannot check, and every driver author must handle by
hand:**

1. **MMIO semantics.** `writel(0x1, regs + 0x20)` is type-safe and may still brick the
   device. Register semantics, write-ordering requirements, and "you must wait 10 µs after
   this write" are datasheet facts (Ch. 34 §T.0). The type system can enforce *bounds*, not
   *meaning*.
2. **DMA.** A device writing into a buffer is, by definition, a mutation the compiler does
   not see — an aliasing violation of the exact kind Rust's model forbids. The ownership
   handoff of Ch. 35 §T.0 (`dma_map` / `dma_unmap`, CPU-owned vs device-owned) has to be
   modelled explicitly in the API, and the modelling is imperfect.
3. **Execution context.** "This may not sleep" is not in the type system. Calling a
   `Mutex::lock()` from an IRQ handler compiles fine and deadlocks (Ch. 17, Ch. 81 §T.0).

**So what *is* the claim?** This, and it is worth memorizing as the accurate form:

> **In safe Rust code, the memory-safety bug classes are absent. At every boundary where
> those guarantees cannot hold, the code must be marked `unsafe` and carry a written safety
> argument — which makes the set of places a memory-safety bug can live small, explicit, and
> greppable.**

The value is **auditability**, not omniscience. A reviewer of a 3,000-line C driver must
consider every line a candidate for a UAF. A reviewer of the Rust equivalent looks at the
forty lines inside `unsafe` blocks. **That is a 75× reduction in the surface a human must
reason about, and it is the entire practical argument.**

**Now the thing this chapter is really about: how do you build a *sound* abstraction over an
unsound world?** The discipline has three parts, and they are the transferable skill:

```
  1. STATE THE INVARIANT.   "# Invariants: `self.ptr` is a valid, live
                             pointer to a `struct foo` whose refcount
                             this object owns one reference of."

  2. ESTABLISH IT at every construction site, in unsafe code, with a
     `// SAFETY:` comment proving it holds.

  3. PRESERVE IT in every method. If any safe method can break the
     invariant, the abstraction is UNSOUND -- even if no current caller
     does so. Soundness is a property of what is POSSIBLE, not of what
     happens.
```

**Step 3 is the one people get wrong.** "No caller currently does that" is not a soundness
argument. If a safe function can be called in a way that causes UB, the *abstraction* has the
bug, not the caller — and that is a genuinely different standard of review from C, where the
caller is traditionally blamed for violating a documented contract.

**This is the same shape as Ch. 24 §T.0's UAPI rule** (your interface must be safe against a
hostile caller, not merely against the callers you imagined) and Ch. 65 §T.0's trust-boundary
validation. **Rust just makes the standard explicit and gives it a name.**

```bash
grep -rn '# Invariants' rust/kernel/ | head -20
grep -rn '# Safety' rust/kernel/ | wc -l
sed -n '1,80p' rust/kernel/types.rs        # ARef, Opaque, AlwaysRefCounted
```

---

### T.1 — The boundary of the guarantee

State this before anything else, because it frames the whole chapter:

> Rust's memory safety covers what the **CPU** does to memory. DMA is what the **device**
> does to memory. The borrow checker cannot see it, KASAN cannot see it, and no type system
> in existence can see it.

So a DMA buffer freed while the device is still writing to it is a use-after-free that Rust
does not prevent. What the DMA abstractions do instead is make the **ownership transitions
explicit and hard to skip** — which is a real improvement, but a different kind of one.

The three kinds of safety in this chapter:

| Kind | Example | Enforced by |
|---|---|---|
| **Compile-time** | MMIO offset in range; no `&mut` aliasing; unmap on drop | the compiler |
| **Type-guided** | DMA ownership transitions; coherent vs streaming | the API shape + review |
| **Runtime only** | Device actually stopped DMA before free; barrier correctness | `DMA_API_DEBUG`, IOMMU strict, review |

Being able to sort a given hazard into the right row is what this chapter is for.

### T.2 — PCI: the discoverable-bus shape

```rust
impl pci::Driver for MyPciDriver {
    type IdInfo = Config;
    const ID_TABLE: pci::IdTable<Self::IdInfo> = &PCI_TABLE;

    fn probe(pdev: &pci::Device<device::Core>, info: &Self::IdInfo)
        -> Result<Pin<KBox<Self>>>
    {
        pdev.enable_device_mem()?;         // pci_enable_device_mem
        pdev.set_master();                 // pci_set_master (bus-master DMA)

        let bar = pdev.iomap_region_sized::<BAR_SIZE>(0, c_str!("mydrv"))?;
        // ^ pci_request_region + pci_iomap, with Drop doing both releases

        ...
    }
}

kernel::pci_device_table!(
    PCI_TABLE, MODULE_PCI_TABLE,
    <MyPciDriver as pci::Driver>::IdInfo,
    [
        (pci::DeviceId::from_id(VENDOR_ID, DEVICE_ID_V1), CONFIG_V1),
        (pci::DeviceId::from_id(VENDOR_ID, DEVICE_ID_V2), CONFIG_V2),
    ]
);
```

The C sequence from Ch. 37 is preserved exactly — enable, request regions, map, set master,
allocate vectors, register — because it is a hardware requirement, not a language artifact.
What changes is that **each step returns a value whose `Drop` performs the matching release
in the right order**, so the six-label `goto` ladder disappears.

The ordering rule from Ch. 82 §T.5 becomes critical here: declare fields in *acquisition*
order, because `Drop` runs in reverse. If `pci_disable_device` must happen after the BAR is
unmapped, the enable-guard must be declared before the BAR.

### T.3 — MMIO: `IoMem` and why there is no `Deref`

```rust
let regs: IoMem<0x1000> = pdev.iomap_region_sized::<0x1000>(0, name)?;

let id = regs.read32(REG_ID);            // bounds checked AT COMPILE TIME
regs.write32(CTRL_ENABLE, REG_CONTROL);
// let x = regs.read32(0x2000);          // compile error: 0x2000 >= 0x1000
```

Three properties, each replacing a C convention:

1. **No `Deref`.** There is no way to turn an `IoMem` into a `&u32` and dereference it. In C,
   `__iomem` is a `sparse` annotation that is advisory; here the operation simply does not
   exist. Every access goes through `read*`/`write*`, which map to `readl`/`writel` and carry
   the correct barriers and byte-ordering.
2. **Const-generic bounds.** With a compile-time `SIZE`, offsets are checked by the compiler.
   With `SIZE = 0` (runtime-sized), you must use `try_read32` and handle `Result`. **A
   register-offset typo becomes a build failure**, which is a genuinely new capability.
3. **Ordering is inherited, not reinvented.** `write32` is `writel`, with the same ordering
   guarantees and the same limits: it orders against other MMIO to the same device, not
   against DMA-memory writes. You still need `dma_wmb()` between descriptor writes and a
   doorbell (Ch. 34 §T.5, Ch. 35). **Rust does not help here at all**, and pretending
   otherwise in an interview is a mistake.

### T.4 — `Devres`: binding a resource to the device's lifetime

```rust
pub struct Devres<T> { ... }

impl<T> Devres<T> {
    pub fn new(dev: &Device<Bound>, data: impl PinInit<T, Error>)
        -> impl PinInit<Self, Error>;
    pub fn try_access(&self) -> Option<Borrow<'_, T>>;
}
```

`Devres<T>` is the Rust `devm_`, and it exists for the same reason: **some resources must be
released when the device is unbound, even if a Rust object still references them.**

The MMIO case makes it concrete. If the device is hot-unplugged or unbound, the BAR must be
unmapped *now* — but a file descriptor or a work item might still hold a reference to the
driver state. So `Devres` separates the two lifetimes:

- The *mapping* dies at unbind.
- The *wrapper* lives as long as anyone holds it.
- `try_access()` returns `None` after unbind.

**That `Option` is the whole point.** It is the "is this still valid?" check that C's
`devm_ioremap` does not give you, and the reason a hot-unplug oops is possible in C and not
in safe Rust.

```rust
// Every MMIO access after probe goes through this shape:
let io = self.regs.try_access().ok_or(ENXIO)?;
let v = io.read32(REG_STATUS);
```

Reviewers should treat `try_access().unwrap()` as a defect: it turns a clean `-ENXIO` into a
kernel panic.

### T.5 — Interrupts

```rust
// Threaded IRQ: the modern default (Ch. 17 §T.6, Ch. 103 §T.4).
let registration = ThreadedRegistration::register(
    irq,
    irq::flags::SHARED,
    c_str!("mydrv"),
    handler_data,      // Arc<Self> -- the handler cannot outlive its data
)?;

impl ThreadedHandler for MyDriver {
    /// Hard-IRQ context. Cannot sleep. Cannot allocate with GFP_KERNEL.
    fn handle_irq(&self) -> irq::Return {
        let io = match self.regs.try_access() {
            Some(io) => io,
            None => return irq::Return::None,   // device gone
        };
        let status = io.read32(REG_STATUS);
        if status & IRQ_PENDING == 0 {
            return irq::Return::None;           // shared IRQ: not ours
        }
        io.write32(status, REG_STATUS);         // ack
        irq::Return::WakeThread
    }

    /// Thread context. May sleep.
    fn handle_threaded_irq(&self) -> irq::Return {
        self.process_completions();
        irq::Return::Handled
    }
}
```

What the types give you:

- **The registration is a value.** Dropping it calls `free_irq`. The C bug of freeing driver
  state before `free_irq()` — leaving a live handler pointing at freed memory — requires the
  registration to outlive the data, which the lifetime bounds prevent.
- **The handler data is an `Arc<Self>`**, so the object cannot be freed while the handler is
  registered. Same mechanism as `Work<T>` in Ch. 81 §T.9.
- **`irq::Return` is an enum**, so `IRQ_NONE`/`IRQ_HANDLED`/`IRQ_WAKE_THREAD` is exhaustive
  rather than an int.

What they do **not** give you: the hard-IRQ handler is an ordinary `fn` and nothing stops you
calling something that sleeps inside it. `might_sleep()` at runtime is still your detector
(Ch. 81 §T.10).

### T.6 — DMA: the part where you must think

Ch. 35 established the model: the device has its own address space (bus/IOVA), the CPU has
its own, and a *mapping* connects them for a bounded period during which **the device owns
the memory**.

**Coherent (consistent) DMA** — allocated once, shared for the device's lifetime, no explicit
sync:

```rust
let ring: CoherentAllocation<Descriptor> =
    CoherentAllocation::alloc_coherent(dev, NUM_DESCS, GFP_KERNEL)?;

let dma_handle = ring.dma_handle();      // give this to the device
// CPU access:
kernel::dma_write!(ring[i] = Descriptor { addr, len, flags })?;
let d = kernel::dma_read!(ring[i])?;
```

Use for: descriptor rings, small control structures, anything both sides touch constantly.
Cost: on non-coherent platforms this is uncached memory, so CPU access is slow.

**Streaming DMA** — map an existing buffer for one transfer:

```
 CPU writes buffer
   │
 dma_map_single(DMA_TO_DEVICE)      <-- ownership transfers TO the device
   │                                    CPU MUST NOT touch it now
 device reads
   │
 dma_unmap_single(DMA_TO_DEVICE)    <-- ownership returns to the CPU
```

The rule to state precisely, because it is the one interviewers probe: **between map and
unmap, the buffer belongs to the device. CPU access in that window is undefined**, and on a
non-coherent platform it produces silent corruption because the cache maintenance has already
happened.

If you must peek mid-transfer, `dma_sync_single_for_cpu()` / `..._for_device()` bracket the
access — and those are exactly the calls whose omission causes the "works on one board,
corrupts on another" bug of `reference/debugging-scenarios.md` §7.

**What Rust adds, and what it does not:**

| | |
|---|---|
| Adds | `Drop` unmaps — the "forgot to unmap" leak is gone |
| Adds | The mapping is a value with a lifetime, so the buffer cannot be freed while the mapping exists |
| Adds | `dma_read!`/`dma_write!` macros make CPU access to coherent memory explicit and greppable |
| **Does not add** | Any check that the *device* has stopped. That is a hardware fact |
| **Does not add** | Barrier correctness between descriptor writes and the doorbell |
| **Does not add** | Cacheline alignment of the buffer on non-coherent platforms |

The DMA abstractions were among the most contentious to merge, precisely because the safety
argument is subtle: a `dma_handle` handed to a device is a capability the type system cannot
track once it crosses the MMIO boundary.

### T.7 — The descriptor-ring pattern, correctly

This is the shape of every high-performance driver (Ch. 46, Ch. 69), and the place where
every ordering rule matters at once:

```rust
fn submit(&self, buf: &DmaBuffer) -> Result {
    let ring = self.ring.try_access().ok_or(ENXIO)?;
    let mut idx = self.tail.lock();

    // 1. Write the descriptor into coherent memory.
    kernel::dma_write!(ring[*idx] = Descriptor {
        addr: buf.dma_handle(),
        len: buf.len() as u32,
        flags: DESC_OWN_DEVICE,
    })?;

    // 2. BARRIER: ensure the descriptor is visible to the device before
    //    the doorbell. This is dma_wmb(), and it is YOUR responsibility.
    //    Nothing in the type system requires it.
    kernel::dma_wmb();

    // 3. Ring the doorbell (an MMIO write).
    let io = self.regs.try_access().ok_or(ENXIO)?;
    io.write32(*idx as u32, REG_DOORBELL);

    *idx = (*idx + 1) % NUM_DESCS;
    Ok(())
}
```

Step 2 is the one that is invisible when it is missing and fatal at high load. Make it a
review checklist item.

### T.8 — MSI/MSI-X

```rust
let nvec = pdev.alloc_irq_vectors(1, MAX_VECTORS,
                                  pci::IrqType::MSIX | pci::IrqType::MSI)?;
for i in 0..nvec {
    let irq = pdev.irq_vector(i)?;
    let reg = ThreadedRegistration::register(irq, 0, name, queue[i].clone())?;
    registrations.push(reg, GFP_KERNEL)?;
}
```

The shape mirrors Ch. 38 exactly: request a range, get what you get, one handler per vector,
per-vector data so there is no shared state on the interrupt path. The Rust improvement is
that each `Registration` owns its `free_irq`, so a partial failure halfway through the loop
unwinds the already-registered vectors automatically — which in C is another `goto` ladder
with an index.

### T.9 — The honest scorecard

| Bug class | C | Rust | Why |
|---|---|---|---|
| Forgot `iounmap` | possible | **impossible** | `Drop` |
| Forgot `free_irq` | possible | **impossible** | `Drop` |
| Forgot `dma_unmap` | possible | **impossible** | `Drop` |
| Wrong unwind order in probe | common | **impossible** | reverse declaration order |
| Register offset out of BAR | possible | **compile error** | const generics |
| Dereferenced `__iomem` | possible | **impossible** | no `Deref` |
| MMIO after unbind | possible | **impossible in safe code** | `Devres` → `Option` |
| Handler outlives its data | possible | **impossible** | `Arc` in registration |
| Freed buffer while device DMAs | possible | **possible** | device behaviour |
| Missing `dma_wmb` before doorbell | possible | **possible** | ordering is not a type |
| Missing `dma_sync` | possible | **possible** | — |
| Cacheline sharing in DMA buffer | possible | **possible** | layout, not safety |
| Deadlock | possible | **possible** | lockdep |
| Sleeping in hard IRQ | possible | **possible** | `might_sleep` |

**Roughly the top half is eliminated; the bottom half is not.** That table is the answer to
"is Rust worth it for drivers?" and the fact that you can draw the line precisely is what
makes the answer credible.

---

## 1. Internals

### Source map

| Path | Contents |
|---|---|
| `rust/kernel/pci.rs` | `pci::Driver`, `pci::Device`, BAR mapping, MSI/MSI-X, `pci_device_table!` |
| `rust/kernel/io.rs` | `Io<SIZE>`, the const-generic accessors |
| `rust/kernel/io/mem.rs` | `IoMem`, `ExclusiveIoMem` |
| `rust/kernel/devres.rs` | `Devres<T>`, `try_access` |
| `rust/kernel/dma.rs` | `CoherentAllocation`, `Device` DMA mask, `dma_read!`/`dma_write!` |
| `rust/kernel/irq.rs` | `Registration`, `ThreadedRegistration`, `Handler`, `irq::Return` |
| `rust/kernel/device.rs` | `Device<Core>`, `Device<Bound>`, `Device<Normal>` — the state typestates |
| `samples/rust/rust_driver_pci.rs` | Minimal PCI driver |
| `drivers/block/rnull.rs` | A real driver using the block layer |
| `Documentation/core-api/dma-api.rst` | **Unchanged and still authoritative** |

### Device typestates

A recent and elegant addition:

```rust
pub struct Device<Ctx: DeviceContext = Normal>(Opaque<bindings::device>, PhantomData<Ctx>);

pub struct Normal;    // any reference to a device
pub struct Bound;     // the driver is bound -- resources may be acquired
pub struct Core;      // inside probe/remove -- the strongest guarantees
```

Some operations are only valid while the driver is bound (acquiring a `Devres` resource, for
instance). Encoding that as a type parameter means `Devres::new(dev, ...)` simply will not
accept a `&Device<Normal>` — **the "you can only do this in probe" rule becomes a signature**
rather than a sentence in a document.

This is the same technique as `Pin` (Ch. 80) and `LockedBy` (Ch. 81): make the precondition a
type that only the right context can produce.

### `CoherentAllocation`

```rust
pub struct CoherentAllocation<T: AsBytes + FromBytes> {
    dev: ARef<Device>,
    dma_handle: bindings::dma_addr_t,
    count: usize,
    cpu_addr: *mut T,
    dma_attrs: Attrs,
}

impl<T> Drop for CoherentAllocation<T> {
    fn drop(&mut self) {
        // SAFETY: allocated by dma_alloc_attrs with these exact arguments,
        // and this is the single matching free.
        unsafe { bindings::dma_free_attrs(...) };
    }
}
```

The `T: AsBytes + FromBytes` bound is the interesting part. It restricts `T` to types that
are safe to reinterpret as raw bytes — no padding with undefined contents, no pointers, no
enums with invalid bit patterns. That is the type-level statement of "this struct is a
hardware descriptor," and it prevents two real bugs: leaking uninitialized padding to a
device (an info leak, Ch. 102) and reading a device-written value into a type where some bit
patterns are UB.

### `irq::Registration`

```rust
pub struct Registration<T: Handler + 'static> {
    inner: Devres<RegistrationInner>,
    handler: Arc<T>,
}
```

Note both mechanisms at once: `Devres` so the IRQ is freed at unbind, and `Arc<T>` so the
handler data outlives any in-flight interrupt. Between them, the two classic C bugs — IRQ
still registered after unbind, and handler data freed while an interrupt is in flight — are
both structurally excluded.

---

## 2. Practice

### Lab 83.1 — A PCI driver with BAR, MSI-X, and a DMA ring

```rust
// SPDX-License-Identifier: GPL-2.0
//! A PCI driver exercising BAR mapping, MSI-X, coherent DMA, and
//! threaded IRQs. Test against QEMU's `edu` device or `pci-testdev`.

use kernel::prelude::*;
use kernel::{
    bindings, c_str, device,
    devres::Devres,
    dma::{CoherentAllocation, Device as DmaDevice},
    dma_read, dma_write,
    io::mem::IoMem,
    irq::{self, ThreadedHandler, ThreadedRegistration},
    pci,
    sync::{new_spinlock, Arc, SpinLock},
    types::ARef,
};

module! {
    type: RustPciDriver,
    name: "rust_pci_demo",
    author: "You",
    description: "PCI + DMA + MSI-X in Rust",
    license: "GPL",
}

const VENDOR_ID: u32 = 0x1234;
const DEVICE_ID: u32 = 0x11e8;      // QEMU edu device

const BAR_SIZE: usize = 0x100000;
const REG_ID:        usize = 0x00;
const REG_STATUS:    usize = 0x20;
const REG_IRQ_STATUS:usize = 0x24;
const REG_IRQ_ACK:   usize = 0x64;
const REG_DMA_SRC:   usize = 0x80;
const REG_DMA_DST:   usize = 0x88;
const REG_DMA_CNT:   usize = 0x90;
const REG_DMA_CMD:   usize = 0x98;
const REG_DOORBELL:  usize = 0x60;

const NUM_DESCS: usize = 64;

/// A hardware descriptor. AsBytes + FromBytes is the type-level statement
/// that this is a plain byte layout with no padding surprises.
#[repr(C)]
#[derive(Copy, Clone, Default)]
struct Descriptor {
    addr: u64,
    len: u32,
    flags: u32,
}

// SAFETY: Descriptor is repr(C), contains only integers, and has no
// padding (8 + 4 + 4 = 16, all naturally aligned).
unsafe impl kernel::types::AsBytes for Descriptor {}
// SAFETY: every bit pattern is a valid Descriptor.
unsafe impl kernel::types::FromBytes for Descriptor {}

struct RingState {
    head: usize,
    tail: usize,
    completions: u64,
    spurious: u64,
}

#[pin_data(PinnedDrop)]
struct DeviceState {
    dev: ARef<device::Device>,
    regs: Devres<IoMem<BAR_SIZE>>,
    ring: CoherentAllocation<Descriptor>,
    #[pin]
    state: SpinLock<RingState>,
}

impl DeviceState {
    fn submit(&self, dma_addr: u64, len: u32) -> Result {
        let io = self.regs.try_access().ok_or(ENXIO)?;
        let mut s = self.state.lock();

        let next = (s.head + 1) % NUM_DESCS;
        if next == s.tail {
            return Err(EAGAIN);                 // ring full
        }

        // 1. Write the descriptor into coherent memory.
        dma_write!(self.ring[s.head] = Descriptor {
            addr: dma_addr,
            len,
            flags: 1,                            // OWN_DEVICE
        })?;

        // 2. Ensure the descriptor is visible BEFORE the doorbell.
        //    The type system does not require this. You do.
        kernel::dma_wmb();

        // 3. Doorbell.
        io.write32(s.head as u32, REG_DOORBELL);

        s.head = next;
        Ok(())
    }

    fn drain_completions(&self) -> Result<u32> {
        let mut s = self.state.lock();
        let mut n = 0u32;

        while s.tail != s.head {
            let d = dma_read!(self.ring[s.tail])?;
            if d.flags & 1 != 0 {
                break;                           // still owned by device
            }
            s.tail = (s.tail + 1) % NUM_DESCS;
            s.completions += 1;
            n += 1;
        }
        Ok(n)
    }
}

#[vtable]
impl ThreadedHandler for DeviceState {
    /// Hard IRQ. Minimum work: identify, acknowledge, defer.
    fn handle_irq(&self) -> irq::Return {
        let Some(io) = self.regs.try_access() else {
            return irq::Return::None;            // device gone
        };
        let status = io.read32(REG_IRQ_STATUS);
        if status == 0 {
            // Shared IRQ line and this was not us. Returning None is
            // what lets the core find the real handler.
            self.state.lock().spurious += 1;
            return irq::Return::None;
        }
        io.write32(status, REG_IRQ_ACK);
        irq::Return::WakeThread
    }

    /// Thread context: may sleep, may allocate, may take mutexes.
    fn handle_threaded_irq(&self) -> irq::Return {
        match self.drain_completions() {
            Ok(n) if n > 0 => dev_dbg!(self.dev, "drained {} completions\n", n),
            Ok(_) => {}
            Err(e) => dev_err!(self.dev, "drain failed: {:?}\n", e),
        }
        irq::Return::Handled
    }
}

struct RustPciDriver {
    state: Arc<DeviceState>,
    _irqs: KVec<ThreadedRegistration<DeviceState>>,
}

kernel::pci_device_table!(
    PCI_TABLE, MODULE_PCI_TABLE,
    <RustPciDriver as pci::Driver>::IdInfo,
    [ (pci::DeviceId::from_id(VENDOR_ID, DEVICE_ID), ()) ]
);

impl pci::Driver for RustPciDriver {
    type IdInfo = ();
    const ID_TABLE: pci::IdTable<Self::IdInfo> = &PCI_TABLE;

    fn probe(pdev: &pci::Device<device::Core>, _info: &Self::IdInfo)
        -> Result<Pin<KBox<Self>>>
    {
        let dev = pdev.as_ref();

        // --- enable, master, map. Each returns a value with a Drop. ---
        pdev.enable_device_mem()?;
        pdev.set_master();

        let regs = pdev.iomap_region_sized::<BAR_SIZE>(
            0, c_str!("rust_pci_demo"))?;

        // Confirm we are talking to the right thing.
        {
            let io = regs.try_access().ok_or(ENXIO)?;
            let id = io.read32(REG_ID);
            dev_info!(dev, "device id register = {:#010x}\n", id);
        }

        // --- DMA mask BEFORE any allocation. Order matters. ---
        dma_set_mask_and_coherent(dev, 0xffff_ffff_ffff_ffff)?;

        let ring = CoherentAllocation::<Descriptor>::alloc_coherent(
            dev, NUM_DESCS, GFP_KERNEL)?;
        dev_info!(dev, "ring at dma {:#x}, {} descriptors\n",
                  ring.dma_handle(), NUM_DESCS);

        // Zero the ring before telling hardware about it.
        for i in 0..NUM_DESCS {
            dma_write!(ring[i] = Descriptor::default())?;
        }

        let state = Arc::pin_init(
            try_pin_init!(DeviceState {
                dev: dev.into(),
                regs,
                ring,
                state <- new_spinlock!(
                    RingState { head: 0, tail: 0, completions: 0, spurious: 0 },
                    "DeviceState::state"
                ),
            }),
            GFP_KERNEL,
        )?;

        // --- MSI-X. Partial failure unwinds automatically. ---
        let nvec = pdev.alloc_irq_vectors(
            1, 4, pci::IrqType::MSIX | pci::IrqType::MSI)?;
        dev_info!(dev, "allocated {} irq vectors\n", nvec);

        let mut irqs = KVec::with_capacity(nvec as usize, GFP_KERNEL)?;
        for i in 0..nvec {
            let irq = pdev.irq_vector(i)?;
            let reg = ThreadedRegistration::register(
                irq, irq::flags::SHARED, c_str!("rust_pci_demo"),
                state.clone(),
            )?;
            irqs.push(reg, GFP_KERNEL)?;
            // If push fails here, `reg` is dropped -> free_irq, and the
            // already-pushed registrations unwind with `irqs`. The C
            // version needs a loop with an index and a label.
        }

        Ok(KBox::new(RustPciDriver { state, _irqs: irqs }, GFP_KERNEL)?.into())
    }
}

#[pinned_drop]
impl PinnedDrop for DeviceState {
    fn drop(self: Pin<&mut Self>) {
        // CRITICAL: stop the device's DMA before the ring allocation is
        // freed. Nothing in the type system enforces this; it is a
        // hardware fact and it is your job.
        if let Some(io) = self.regs.try_access() {
            io.write32(0, REG_DMA_CMD);
            let _ = io.read32(REG_DMA_CMD);   // read back to flush the write
        }
        let s = self.state.lock();
        dev_info!(self.dev, "teardown: {} completions, {} spurious\n",
                  s.completions, s.spurious);
    }
}
```

Test against QEMU:

```bash
qemu-system-x86_64 -enable-kvm -m 2G \
	-kernel arch/x86/boot/bzImage \
	-append "console=ttyS0 dyndbg=+p" \
	-device edu \
	-nographic

# In the guest:
insmod rust_pci_demo.ko
dmesg | tail -20
lspci -vv -d 1234:11e8
cat /proc/interrupts | grep rust_pci

# Then the important test:
echo 1 > /sys/bus/pci/devices/0000:00:04.0/remove   # hot-remove
# Verify: no oops, clean teardown message, and any in-flight access
#         returns ENXIO rather than touching unmapped memory.
echo 1 > /sys/bus/pci/rescan
```

### Lab 83.2 — Streaming DMA and the ownership window

```rust
// SPDX-License-Identifier: GPL-2.0
//! Streaming DMA: the map/unmap window, made explicit.

use kernel::prelude::*;
use kernel::dma::{DataDirection, DmaMapping};

/// One transfer's worth of buffer plus its mapping.
struct Transfer {
    buffer: KVec<u8>,
    /// While this is Some, the DEVICE owns `buffer`. CPU access is UB.
    mapping: Option<DmaMapping>,
}

impl Transfer {
    fn new(len: usize) -> Result<Self> {
        let mut buffer = KVec::new();
        // Cacheline-align and -size the buffer. On a non-coherent
        // platform, sharing a cacheline with anything else means the
        // cache invalidate destroys the neighbour.
        buffer.resize(round_up(len, L1_CACHE_BYTES), 0u8, GFP_KERNEL)?;
        Ok(Transfer { buffer, mapping: None })
    }

    /// Fill the buffer. Only legal while unmapped.
    fn fill(&mut self, data: &[u8]) -> Result {
        if self.mapping.is_some() {
            // The type system does not stop this; we do, explicitly.
            return Err(EBUSY);
        }
        self.buffer[..data.len()].copy_from_slice(data);
        Ok(())
    }

    /// Hand ownership to the device.
    fn map_for_device(&mut self, dev: &Device) -> Result<u64> {
        let m = DmaMapping::map_single(dev, &self.buffer,
                                       DataDirection::ToDevice)?;
        let handle = m.dma_address();
        self.mapping = Some(m);
        // From HERE until unmap, `self.buffer` belongs to the device.
        Ok(handle)
    }

    /// Take ownership back.
    fn unmap(&mut self) {
        self.mapping = None;      // Drop -> dma_unmap_single
        // The CPU may touch the buffer again.
    }

    /// Peek mid-transfer -- requires an explicit sync in both directions.
    fn peek_with_sync(&mut self, dev: &Device) -> Result<u8> {
        let m = self.mapping.as_ref().ok_or(EINVAL)?;
        m.sync_for_cpu(dev)?;                 // invalidate caches
        let b = self.buffer[0];
        m.sync_for_device(dev)?;              // give it back
        Ok(b)
    }
}
```

Now **prove the hazard**:

```bash
# 1. Build with DMA API debugging and a strict IOMMU.
./scripts/config --enable DMA_API_DEBUG --enable DMA_API_DEBUG_SG
# boot: iommu=force intel_iommu=on iommu.strict=1 iommu.passthrough=0

# 2. Deliberately touch the buffer while mapped and observe what happens
#    on a coherent platform (nothing visible) versus a non-coherent one
#    (corruption). Emulate non-coherent with swiotlb=force.

# 3. Deliberately free the buffer while mapped:
#    -> DMA_API_DEBUG reports "device driver frees DMA memory with
#       different size" or "cacheline tracking" errors.

# 4. Check the debug counters:
cat /sys/kernel/debug/dma-api/error_count
cat /sys/kernel/debug/dma-api/num_free_entries
```

**The lesson to write down:** Rust's `Drop` guarantees the unmap happens. It guarantees
nothing about whether the device was finished. `CONFIG_DMA_API_DEBUG` plus a strict IOMMU is
still your detector, exactly as in C.

### Lab 83.3 — Break the compile-time guarantees

```rust
// 1. Register offset outside the BAR.
let regs: IoMem<0x1000> = ...;
// regs.read32(0x2000);
// error: evaluation of constant value failed: assertion failed
//   ==> A typo'd register offset is a BUILD failure. This is new.

// 2. Dereference the MMIO pointer.
// let v: u32 = *regs;
// error[E0614]: type `IoMem<4096>` cannot be dereferenced
//   ==> No accidental non-volatile access.

// 3. Use MMIO after unbind.
// let io = self.regs.try_access().unwrap();
//   Compiles. Panics after unbind. THIS IS A REVIEW DEFECT.
// Correct:
let io = self.regs.try_access().ok_or(ENXIO)?;

// 4. Let the IRQ handler outlive its data.
//    Not expressible: ThreadedRegistration holds Arc<T>, and the
//    registration must be dropped (freeing the IRQ) before the last
//    Arc can be. The lifetime bounds make it a compile error.

// 5. A descriptor type with padding.
#[repr(C)]
struct BadDesc { a: u8, b: u64 }     // 7 bytes of padding
// unsafe impl AsBytes for BadDesc {}
//   Compiles -- the unsafe impl is YOUR claim. But it is WRONG: the
//   padding is uninitialized and would be exposed to the device, which
//   is an info leak (Ch. 102). Use #[repr(C, packed)] or reorder.
//   ==> This is a review finding, not a compile error. Know the difference.
```

Item 5 is the one to dwell on. **`unsafe impl AsBytes` is a claim the compiler accepts on
your word**, and getting it wrong is a real vulnerability. Add "check every `AsBytes`/
`FromBytes` impl for padding with `pahole`" to your review checklist.

### Lab 83.4 — Resource ordering

```rust
// The ordering rule made concrete. Drop runs in REVERSE declaration order.
#[pin_data]
struct Correct {
    // Acquired first, released last:
    enable_guard: PciEnableGuard,    // pci_disable_device
    regs: Devres<IoMem<SIZE>>,       // iounmap + release_region
    dma: CoherentAllocation<Desc>,   // dma_free_coherent
    irqs: KVec<ThreadedRegistration<Self>>,  // free_irq  <-- FIRST to drop
}
// Drop order: irqs, dma, regs, enable_guard.
//   free_irq before freeing the DMA ring the handler touches:  correct.
//   iounmap before pci_disable_device:                          correct.

#[pin_data]
struct Wrong {
    irqs: KVec<ThreadedRegistration<Self>>,  // <-- drops LAST
    dma: CoherentAllocation<Desc>,           // freed while IRQ still armed!
    regs: Devres<IoMem<SIZE>>,
}
// An interrupt arriving between the dma free and the free_irq touches
// freed memory. COMPILES FINE. This is the one ordering bug Rust does
// not take away from you.
```

Verify empirically: build both, generate interrupts continuously, and unbind under KASAN.
The `Wrong` version should produce a use-after-free report.

**Add to the review checklist: "fields are declared in acquisition order."**

### Lab 83.5 — Hot-unplug survival

```bash
# The test that finds the most real bugs.
for i in $(seq 1 200); do
	echo 1 > /sys/bus/pci/devices/0000:00:04.0/remove
	sleep 0.05
	echo 1 > /sys/bus/pci/rescan
	sleep 0.05
done

# While that runs, in another shell, hammer the device:
while :; do ./exercise_device /dev/rust_pci0 2>/dev/null; done

# Expected:
#   - no oops
#   - userspace gets -ENODEV or -ENXIO, cleanly
#   - no growth in slabtop
#   - no kmemleak reports
dmesg -w | grep -iE 'BUG|KASAN|oops|refcount'
cat /sys/kernel/debug/kmemleak
```

This exercises exactly the `Devres` → `Option` path from §T.4. In C the equivalent test finds
oopses in a large fraction of drivers; in Rust it should find none, and if it does, the bug
is in your `unsafe` or your ordering.

---

## 3. Mastery drills

1. Port a complete C PCI driver from Ch. 37 to Rust. Produce the bug-class scorecard of §T.9
   for *your* driver specifically, marking which rows your C version actually got wrong.

2. Implement a full descriptor-ring driver (submit, complete, wrap, full/empty) against QEMU's
   `edu` device. Verify correctness under `stress-ng` and KASAN, then deliberately remove the
   `dma_wmb()` and find the load level at which corruption appears.

3. Write a scatter-gather DMA abstraction. Determine what does not yet exist in
   `rust/kernel/dma.rs` and what you would need to add. Write the safety contract.

4. Measure the cost of `Devres::try_access()` on a hot MMIO path. Determine whether the
   `Option` check is measurable, and if so, design an API that hoists it out of the loop
   safely.

5. Build a driver that must survive surprise hot-removal mid-DMA. Enumerate every hazard and
   show which are handled by types and which by your code.

6. Implement MSI-X with per-vector affinity and per-queue state (the NAPI shape from Ch. 46).
   Verify each vector lands on its intended CPU and that there is no shared cacheline between
   queues (`perf c2c`, → Ch. 105).

7. Use `pahole` on every `#[repr(C)]` descriptor type in your driver. Find any padding, and
   determine whether the `AsBytes` impl is sound. Fix or justify each.

8. Compare the generated assembly for `IoMem::read32` against C's `readl`. Confirm they are
   identical, including barriers, on both x86 and arm64.

9. Implement a driver using `Devres` for every resource and one using none (raw acquisition
   with explicit `Drop`). Compare behaviour under hot-unplug and explain the difference.

10. Study `drivers/block/rnull.rs` and the Nova DRM driver. Identify every place they use raw
    `bindings` with `unsafe` because an abstraction is missing, and pick one to write.

11. Build the "device is hostile" version of your driver for the confidential-computing threat
    model (Ch. 104 §T.8): every MMIO read and every DMA-written descriptor is
    attacker-controlled. List every place you currently trust the device and fix them.

12. Write the review checklist for Rust PCI/DMA drivers. It should be at most one page and
    should cover: field ordering, `try_access().unwrap()`, `AsBytes` padding, `dma_wmb`
    before doorbells, DMA stop before free, and `irq::Return::None` for shared lines.

---

## 4. Further reading

**Kernel documentation — still the authority**
- `Documentation/core-api/dma-api.rst` and `dma-api-howto.rst` — **the DMA rules are
  language-independent**; nothing in this chapter supersedes them
- `Documentation/PCI/pci.rst` — the initialization sequence
- `Documentation/core-api/irq/` — IRQ handling, threaded IRQs
- `Documentation/driver-api/device-io.rst` — MMIO accessors and ordering

**Code to read**
1. `rust/kernel/pci.rs` — the adapter; compare with `drivers/pci/` C
2. `rust/kernel/io.rs` — the const-generic bounds trick; short and clever
3. `rust/kernel/devres.rs` — the two-lifetime solution
4. `rust/kernel/dma.rs` — read the safety comments especially
5. `rust/kernel/irq.rs` — `Devres` + `Arc` together
6. `samples/rust/rust_driver_pci.rs`
7. `drivers/block/rnull.rs` — a real driver at scale

**Cross-references**
- Ch. 34 — MMIO and ordering, in C. Everything there still applies
- Ch. 35 — the DMA API and its ownership model. **Re-read §T.3 before writing DMA code**
- Ch. 36 — IOMMU, and why strict mode is your debugging tool
- Ch. 37–38 — PCI and PCIe, in C
- Ch. 17 — interrupts, top and bottom halves
- Ch. 105 — cacheline behaviour, which governs DMA buffer layout

**LWN and discussion**
- The LKML threads on the Rust DMA abstractions — the most instructive reading in this area,
  because the disagreements are about exactly where the safety claim stops
- Kangrejos talks on device lifetimes and `Devres`
- "Rust for PCI drivers" coverage and the Nova driver series

→ Next: [84-bindings-unsafe.md](84-bindings-unsafe.md)

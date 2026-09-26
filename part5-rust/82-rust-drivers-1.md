# Chapter 82 — Writing Rust Drivers I: Misc, Platform, Device Tree

> Everything so far has been preparation. This chapter writes actual drivers. The structure
> mirrors Part 2 exactly — Ch. 26 (driver model), Ch. 27 (probe/bind), Ch. 29 (char devices),
> Ch. 30 (misc/debugfs), Ch. 31 (platform devices), Ch. 32 (device tree) — so you can read
> the two side by side and see the same object model expressed twice.

---

## Theory & First Principles

### T.0 — Start here: the same driver, both languages

Ch. 28 §T.0 asked you to count the bugs in a C probe function. Here is the shape of the
correct C, and the shape of the Rust that cannot be written incorrectly.

```c
static int my_probe(struct platform_device *pdev)
{
	struct my_dev *d = devm_kzalloc(&pdev->dev, sizeof(*d), GFP_KERNEL);
	if (!d)
		return -ENOMEM;
	d->regs = devm_platform_ioremap_resource(pdev, 0);
	if (IS_ERR(d->regs))
		return PTR_ERR(d->regs);            /* ← IS_ERR, not NULL. Easy to get wrong. */
	d->clk = devm_clk_get_enabled(&pdev->dev, NULL);
	if (IS_ERR(d->clk))
		return PTR_ERR(d->clk);
	platform_set_drvdata(pdev, d);          /* ← forget this and remove() crashes */
	return 0;
}
```

```rust
impl platform::Driver for MyDriver {
    type IdInfo = ();
    const OF_ID_TABLE: Option<of::IdTable<()>> = Some(&OF_TABLE);

    fn probe(pdev: &platform::Device<Core>, _info: Option<&()>)
        -> Result<Pin<KBox<Self>>>
    {
        let regs = pdev.io_request_and_map(0, c_str!("my-regs"))?;
        let clk  = Clk::get(pdev.as_ref(), None)?.prepare_enable()?;

        KBox::pin_init(try_pin_init!(MyDriver { regs, clk }), GFP_KERNEL)
        //  ^ returning the object IS registering it. There is no
        //    separate set_drvdata step to forget.
    }
}
// No remove(). Drop runs in reverse declaration order, on every path.
```

**Go through what structurally cannot go wrong in the second version:**

| C failure mode | Why it is absent |
|---|---|
| Check `NULL` where you should check `IS_ERR` | errors are `Result`; there is one representation |
| Ignore a returned error | `Result` is `#[must_use]`; ignoring it is a warning-as-error |
| Forget `platform_set_drvdata` | the returned value *is* the driver data |
| Unwind in the wrong order on the error path | `Drop` runs in reverse declaration order, generated |
| Free something twice in `remove()` | ownership: there is exactly one owner |
| Use `d->regs` after `remove()` | lifetimes: it cannot outlive `MyDriver` |

**Notice what this is *not*.** The Rust driver is not architecturally different: it still
probes, still matches on a device-tree compatible string, still maps registers, still enables
a clock, still gets removed. **The device model of Ch. 26–28 is unchanged.** What changed is
the *encoding* of its rules — and that is the point:

> **A driver framework's safety rules are its real content. `devm_` was the kernel's first
> systematic attempt to encode them (Ch. 28 §T.0: "when a correctness obligation is routinely
> forgotten, move it into the primitive"). Rust's `Drop` is the same idea with the escape
> hatches closed.**

`devm_` is opt-in: you can still call `kzalloc` and forget the `kfree`, and reviewers must
notice. `Drop` is not opt-in — you must go out of your way (`mem::forget`, `ManuallyDrop`) to
skip it.

**Two kernel-specific things that are genuinely harder in Rust**, stated so the chapter is
balanced:

1. **MMIO cannot be safe, and pretending otherwise is dishonest.** `Io<SIZE>` makes *bounds*
   checkable (a compile-time size parameter plus runtime checks for dynamic offsets), but
   whether writing `0x1` to offset `0x20` is correct depends on a datasheet the compiler has
   never read. Ch. 34 §T.0's barrier rules are equally invisible to it.
2. **The abstractions are incomplete and unstable.** A subsystem with no Rust bindings means
   you write them (Ch. 84), and they change as the C side changes. This is a real, ongoing
   engineering cost, not a transitional annoyance.

```bash
ls drivers/**/*.rs samples/rust/*.rs 2>/dev/null
sed -n '1,120p' samples/rust/rust_misc_device.rs
grep -rn 'impl platform::Driver' --include=*.rs . | head
```

---

### T.1 — The same driver model, a different encoding

Nothing about the driver model changes. There is still a bus, a driver, a device, a match
table, a `probe`, and a `remove`. What changes is who enforces the rules:

| Driver-model rule | C | Rust |
|---|---|---|
| `remove()` must undo `probe()` exactly | discipline + review | `Drop`, generated |
| drvdata ownership | `void *` and a convention | `ForeignOwnable` |
| Match table must be NULL-terminated | a macro and hope | `kernel::of_device_table!` generates it |
| Resource release order | reverse of the `goto` ladder | reverse declaration order, automatic |
| Refcounts on `struct device` | `get_device`/`put_device` pairs | `ARef<Device>` |
| The ops struct's NULL slots | manual struct initializer | `#[vtable]` + `HAS_*` constants |

The shape of a Rust driver is therefore:

```rust
1. define the state struct        (what the device owns)
2. impl the driver trait          (probe, and optionally others)
3. define the match table         (of_device_table! / module_platform_driver!)
4. module! { type: TheDriver }    (registration + unregistration)
```

and there is **no `remove()`** in the common case, because dropping the state object is the
removal.

### T.2 — `probe` returns the state; `Drop` is `remove`

```rust
impl platform::Driver for MyDriver {
    type IdInfo = ();
    const OF_ID_TABLE: Option<of::IdTable<Self::IdInfo>> = Some(&OF_TABLE);

    fn probe(pdev: &mut platform::Device, _id: Option<&Self::IdInfo>)
        -> Result<Pin<KBox<Self>>>
    {
        let state = KBox::pin_init(MyDriver::new(pdev)?, GFP_KERNEL)?;
        Ok(state)
    }
}

impl Drop for MyDriver {
    fn drop(&mut self) {
        // Only for things NOT already handled by a field's own Drop.
        dev_info!(self.dev, "removed\n");
    }
}
```

The returned `Pin<KBox<Self>>` is handed to the driver core via `ForeignOwnable`; on removal
the core hands it back and it is dropped. Every field's `Drop` runs in reverse declaration
order. **That is the entire lifecycle.**

The consequence worth internalizing: in C, `probe` and `remove` are two functions that must
be kept in exact mirror correspondence as the driver evolves, and drift between them is one
of the most common driver bug sources. In Rust the correspondence is not written down and
therefore cannot drift.

### T.3 — Misc devices: the smallest complete driver

`miscdevice` is the right starting point for the same reason it is in C (Ch. 30): it gives
you a `/dev` node with a dynamic minor and no `cdev`/class/region boilerplate.

```rust
use kernel::miscdevice::{MiscDevice, MiscDeviceOptions, MiscDeviceRegistration};

#[vtable]
impl MiscDevice for MyDevice {
    type Ptr = Arc<MyDevice>;

    fn open(_file: &File, _misc: &MiscDeviceRegistration<Self>)
        -> Result<Arc<MyDevice>> { ... }

    fn ioctl(me: ArcBorrow<'_, MyDevice>, _file: &File,
             cmd: u32, arg: usize) -> Result<isize> { ... }

    fn read_iter(...) -> Result<usize> { ... }
}
```

Three things to notice in that signature:

- **`type Ptr = Arc<MyDevice>`** states what owns the per-open state. `open` produces one;
  the kernel stores it in `file->private_data` via `ForeignOwnable`; `release` drops it. The
  "who frees `private_data`?" question has a typed answer.
- **`ArcBorrow<'_, MyDevice>`** in the callbacks: you get a borrowed reference with a
  lifetime bounded by the call. You cannot stash it, and you do not touch the refcount.
- **`#[vtable]`** generates the C `file_operations` with `NULL` for anything you did not
  implement.

### T.4 — The lifetime problem char devices actually have

Ch. 29 §T.6 established the hard part of char devices in C: **an open file descriptor can
outlive `remove()`**. Userspace holds an `fd`; the device is unbound; a callback then runs
against freed state. The C answer is a `kref` on a separate object plus careful ordering,
and `devm_` is *wrong* here because the lifetime is not the device's.

Rust encodes this directly:

```rust
// Per-device state, owned by the driver core, dropped at remove():
struct DeviceState { regs: IoMem, ... }

// Per-open state, owned by the file, dropped at close():
struct OpenState {
    shared: Arc<Shared>,      // shared with the device, refcounted
    position: u64,            // per-fd
}
```

The `Arc<Shared>` is the `kref`. As long as any open file holds one, `Shared` lives. When the
device is removed, the driver drops *its* `Arc`, but the object survives until the last file
closes — and the remaining operations must then fail gracefully, which you express with a
flag or an `Option` inside a lock:

```rust
struct Shared {
    inner: Mutex<Option<LiveResources>>,   // None after remove()
}
// In a file op:
let g = shared.inner.lock();
let live = g.as_ref().ok_or(ENODEV)?;      // clean -ENODEV after unbind
```

**This is the single most important driver-lifetime pattern in the chapter, in either
language.** Rust does not invent it; it makes the two lifetimes into two types so that
conflating them does not compile.

### T.5 — Platform devices and the resource dance

```rust
fn probe(pdev: &mut platform::Device, _id: Option<&()>)
    -> Result<Pin<KBox<Self>>>
{
    let dev = pdev.as_ref();

    // Each of these returns a value whose Drop releases the resource.
    let regs = pdev.ioremap_resource(0)?;         // Drop -> iounmap
    let clk  = Clk::get(dev, None)?.prepare_enable()?;  // Drop -> disable+put
    let irq  = pdev.irq_by_index(0)?;

    // Registration is also a value: Drop -> unregister.
    let state = KBox::pin_init(Self::new(regs, clk, irq)?, GFP_KERNEL)?;
    let _reg  = Registration::new(&state)?;

    Ok(state)
}
```

Two rules that follow:

1. **Declaration order is release order (reversed).** If `clk` must be disabled after `regs`
   is unmapped, declare `clk` *before* `regs`. Getting this wrong is the one ordering bug
   still available to you, and it is a legitimate review finding.
2. **Anything that must be undone should be a value with a `Drop`.** If a C API has no
   natural Rust owner, wrap it or use `ScopeGuard`. A `probe` that calls a C `*_register()`
   and relies on `remove()` to unregister has reintroduced the C problem.

### T.6 — `-EPROBE_DEFER` still exists and still matters

```rust
let clk = match Clk::get(dev, Some(c_str!("core"))) {
    Ok(c) => c,
    Err(e) if e == EPROBE_DEFER => return Err(EPROBE_DEFER),
    Err(e) => {
        dev_err!(dev, "failed to get core clock: {:?}\n", e);
        return Err(e);
    }
};
```

Or, idiomatically, just `?` — because `EPROBE_DEFER` propagates like any other error and the
core interprets it. The important discipline is the same as C (Ch. 27 §T.5): **do not log an
error for `EPROBE_DEFER`**, because it is expected and will spam the log on every retry.
`dev_err_probe()` exists in C for exactly this; the Rust equivalent is the `match` above or a
helper.

### T.7 — Device tree matching

```rust
kernel::module_platform_driver! {
    type: MyDriver,
    name: "my_driver",
    author: "You",
    description: "A platform driver in Rust",
    license: "GPL",
}

kernel::of_device_table!(
    OF_TABLE,
    MODULE_OF_TABLE,
    <MyDriver as platform::Driver>::IdInfo,
    [
        (of::DeviceId::new(c_str!("vendor,my-device-v1")), Config::V1),
        (of::DeviceId::new(c_str!("vendor,my-device-v2")), Config::V2),
    ]
);
```

The macro generates the C `of_device_id` array **with the NULL terminator**, plus the
`MODULE_DEVICE_TABLE` alias so `depmod` can autoload the module. Forgetting the terminator —
a real C bug that causes the matcher to walk off the end — is not expressible.

`IdInfo` is the typed replacement for C's `.data = (void *)&config`:

```rust
type IdInfo = Config;   // an enum, a struct, whatever you like

fn probe(pdev: &mut platform::Device, id: Option<&Config>) -> Result<...> {
    let cfg = id.ok_or(ENODEV)?;   // typed, no cast from void *
    match cfg { Config::V1 => ..., Config::V2 => ... }
}
```

In C this is `of_device_get_match_data()` returning a `const void *` that you cast and hope
is the right type. Here the type is checked and the `match` is exhaustive.

Reading DT properties:

```rust
let freq: u32 = dev.property_read::<u32>(c_str!("clock-frequency"))?;
let name = dev.property_read_string(c_str!("label")).ok();
let enabled = dev.property_read_bool(c_str!("vendor,feature-enabled"));
```

Note these are fallible and typed. `of_property_read_u32` in C returns an int you must check
and writes through an out-parameter you might forget to initialize.

### T.8 — Where the abstractions run out

The realistic scoping answer, and what you must check before promising a Rust driver:

| Available (as of ~6.12–6.14) | Missing or partial |
|---|---|
| `platform::Driver` | most other buses |
| `miscdevice`, char device basics | full `cdev`/class/sysfs attribute machinery |
| `of` matching, property reads | full DT graph/phandle traversal |
| `io::Io`, `IoMem` MMIO | `regmap` |
| `Clk` (basic get/enable) | clock-provider side, rates, parents |
| `Device`, `ARef<Device>` | most `devm_` C helpers (unneeded — `Drop` replaces them) |
| PCI (growing), `pci::Driver` | many PCI capabilities |
| `irq` (recent) | threaded IRQ ergonomics, IRQ domains |
| DMA (recent, contentious) | scatter-gather, IOMMU APIs |
| `net::phy` | most of netdev |
| block (via `rnull`) | most of blk-mq |

If the abstraction you need does not exist, your options are: write it (Ch. 84), use raw
`bindings` with `unsafe` (acceptable only temporarily and only with a plan to upstream an
abstraction), or write the driver in C. **Choosing the third is often the correct engineering
answer and saying so is a sign of judgement, not defeat.**

---

## 1. Internals

### Source map

| Path | Contents |
|---|---|
| `rust/kernel/driver.rs` | Generic `Registration`, `Adapter` — the bus-agnostic machinery |
| `rust/kernel/platform.rs` | `platform::Driver`, `platform::Device`, resource helpers |
| `rust/kernel/of.rs` | `of::DeviceId`, `of::IdTable`, `of_device_table!` |
| `rust/kernel/device.rs` | `Device`, `ARef<Device>`, `dev_*!` macros |
| `rust/kernel/device_id.rs` | The generic ID-table machinery |
| `rust/kernel/miscdevice.rs` | `MiscDevice`, `MiscDeviceRegistration` |
| `rust/kernel/fs/file.rs` | `File`, `LocalFile`, file flags |
| `rust/kernel/io.rs`, `io/mem.rs` | `Io<SIZE>`, `IoMem`, `readl`/`writel` equivalents |
| `rust/kernel/clk.rs` | `Clk`, `EnabledClk` |
| `samples/rust/rust_misc_device.rs` | **The best complete in-tree example** |
| `samples/rust/rust_driver_platform.rs` | Minimal platform driver |
| `samples/rust/rust_driver_pci.rs` | Minimal PCI driver |

### The `Adapter` pattern

```rust
pub trait Adapter {
    type IdInfo: 'static;
    fn of_id_table() -> Option<of::IdTable<Self::IdInfo>>;
    fn id_info(dev: &Device) -> Option<&'static Self::IdInfo> { ... }
}
```

Each bus implements a small adapter that converts between the C callback signatures and the
Rust trait. The generated C `platform_driver` looks like:

```rust
unsafe extern "C" fn probe_callback(pdev: *mut bindings::platform_device) -> c_int {
    // SAFETY: the driver core guarantees `pdev` is valid for this call.
    let pdev = unsafe { &mut *pdev.cast() };
    let info = <Self as Adapter>::id_info(pdev.as_ref());
    match T::probe(pdev, info) {
        Ok(data) => {
            // Hand ownership to the driver core.
            let ptr = data.into_foreign();
            // SAFETY: pdev is valid; we just created ptr.
            unsafe { bindings::dev_set_drvdata(pdev.as_raw(), ptr) };
            0
        }
        Err(e) => e.to_errno(),
    }
}

unsafe extern "C" fn remove_callback(pdev: *mut bindings::platform_device) {
    // SAFETY: probe_callback stored exactly one into_foreign pointer here.
    let ptr = unsafe { bindings::dev_get_drvdata((*pdev).dev.as_raw()) };
    // SAFETY: this is the single matching from_foreign.
    let _data = unsafe { <T::Data as ForeignOwnable>::from_foreign(ptr) };
    // _data dropped here -> the whole driver state tears down.
}
```

**Read that `remove_callback`.** It is four lines, it is the only place ownership tracking is
suspended, and it is written once per bus rather than once per driver. That is the leverage.

### `IoMem` — `__iomem` as a type

```rust
pub struct Io<const SIZE: usize = 0> { addr: usize, maxsize: usize }

impl<const SIZE: usize> Io<SIZE> {
    pub fn read32(&self, offset: usize) -> u32 { ... }       // bounds-checked
    pub fn write32(&self, value: u32, offset: usize) { ... }
    pub fn try_read32(&self, offset: usize) -> Result<u32> { ... }
}
```

The `SIZE` const generic is the clever part: if the region size is known at compile time, the
bounds check is **compile-time** and costs nothing:

```rust
let regs: IoMem<0x1000> = pdev.ioremap_resource_sized::<0x1000>(0)?;
let v = regs.read32(0x40);       // checked at COMPILE time: 0x40 < 0x1000
// let w = regs.read32(0x2000);  // compile error
```

With `SIZE = 0` (unknown at compile time) it falls back to a runtime check via `try_read32`.
There is **no `Deref`**, so a `readl`-free dereference is not expressible — which is what
`sparse`'s `__iomem` tries to achieve advisorily.

---

## 2. Practice

### Lab 82.1 — A complete misc device

```rust
// SPDX-License-Identifier: GPL-2.0
//! A misc character device in Rust: open/read/write/ioctl, with correct
//! per-open and per-device state separation.
//!
//! Build in-tree at samples/rust/rust_scratchpad.rs.

use kernel::prelude::*;
use kernel::{
    c_str,
    fs::File,
    ioctl::{_IO, _IOC_SIZE, _IOR, _IOW},
    miscdevice::{MiscDevice, MiscDeviceOptions, MiscDeviceRegistration},
    sync::{new_mutex, Arc, ArcBorrow, Mutex},
    uaccess::{UserSlice, UserSliceReader, UserSliceWriter},
};

module! {
    type: ScratchpadModule,
    name: "rust_scratchpad",
    author: "You",
    description: "A scratchpad misc device in Rust",
    license: "GPL",
}

const SCRATCH_IOC_MAGIC: u32 = b'S' as u32;
const SCRATCH_IOC_CLEAR: u32 = _IO(SCRATCH_IOC_MAGIC, 1);
const SCRATCH_IOC_GETLEN: u32 = _IOR::<u32>(SCRATCH_IOC_MAGIC, 2);
const SCRATCH_IOC_SETCAP: u32 = _IOW::<u32>(SCRATCH_IOC_MAGIC, 3);

const MAX_CAPACITY: usize = 64 * 1024;

/// Device-wide state, shared by every open file. Lives as long as ANY
/// file holds an Arc -- which is the whole point.
struct Shared {
    buffer: Mutex<KVec<u8>>,
    capacity: Mutex<usize>,
}

/// Per-open state. One per file descriptor.
struct Scratchpad {
    shared: Arc<Shared>,
    /// Per-fd read position: two processes reading get independent cursors.
    position: Mutex<usize>,
}

#[vtable]
impl MiscDevice for Scratchpad {
    type Ptr = Arc<Scratchpad>;

    fn open(_file: &File, misc: &MiscDeviceRegistration<Self>)
        -> Result<Arc<Scratchpad>>
    {
        dev_info!(misc.device(), "scratchpad: open\n");

        let shared = Arc::pin_init(
            pin_init!(Shared {
                buffer <- new_mutex!(KVec::new(), "Shared::buffer"),
                capacity <- new_mutex!(4096usize, "Shared::capacity"),
            }),
            GFP_KERNEL,
        )?;

        Arc::pin_init(
            pin_init!(Scratchpad {
                shared,
                position <- new_mutex!(0usize, "Scratchpad::position"),
            }),
            GFP_KERNEL,
        )
    }

    fn read_iter(me: ArcBorrow<'_, Scratchpad>, _file: &File,
                 writer: &mut UserSliceWriter) -> Result<usize>
    {
        let buf = me.shared.buffer.lock();
        let mut pos = me.position.lock();

        if *pos >= buf.len() {
            return Ok(0);                 // EOF for this fd
        }
        let avail = buf.len() - *pos;
        let n = core::cmp::min(avail, writer.len());
        writer.write_slice(&buf[*pos..*pos + n])?;   // copy_to_user, checked
        *pos += n;
        Ok(n)
    }

    fn write_iter(me: ArcBorrow<'_, Scratchpad>, _file: &File,
                  reader: &mut UserSliceReader) -> Result<usize>
    {
        let cap = *me.shared.capacity.lock();
        let mut buf = me.shared.buffer.lock();

        let n = reader.len();
        if buf.len() + n > cap {
            return Err(ENOSPC);
        }
        let start = buf.len();
        buf.resize(start + n, 0, GFP_KERNEL)?;
        // If read_all fails partway we must not leave garbage visible.
        if let Err(e) = reader.read_slice(&mut buf[start..]) {
            buf.truncate(start);
            return Err(e);
        }
        Ok(n)
    }

    fn ioctl(me: ArcBorrow<'_, Scratchpad>, _file: &File,
             cmd: u32, arg: usize) -> Result<isize>
    {
        match cmd {
            SCRATCH_IOC_CLEAR => {
                me.shared.buffer.lock().clear();
                *me.position.lock() = 0;
                Ok(0)
            }
            SCRATCH_IOC_GETLEN => {
                let len = me.shared.buffer.lock().len() as u32;
                let mut w = UserSlice::new(arg as *mut u8,
                                           _IOC_SIZE(cmd)).writer();
                w.write::<u32>(&len)?;
                Ok(0)
            }
            SCRATCH_IOC_SETCAP => {
                let mut r = UserSlice::new(arg as *mut u8,
                                           _IOC_SIZE(cmd)).reader();
                let new_cap: u32 = r.read()?;
                let new_cap = new_cap as usize;
                // Validate the COPIED value, never re-read user memory.
                if new_cap == 0 || new_cap > MAX_CAPACITY {
                    return Err(EINVAL);
                }
                *me.shared.capacity.lock() = new_cap;
                Ok(0)
            }
            _ => Err(ENOTTY),          // the correct errno for a bad ioctl
        }
    }
}

struct ScratchpadModule {
    _reg: Pin<KBox<MiscDeviceRegistration<Scratchpad>>>,
}

impl kernel::Module for ScratchpadModule {
    fn init(_m: &'static ThisModule) -> Result<Self> {
        let opts = MiscDeviceOptions { name: c_str!("rust_scratchpad") };
        let reg = KBox::pin_init(MiscDeviceRegistration::register(opts),
                                 GFP_KERNEL)?;
        pr_info!("rust_scratchpad: /dev/rust_scratchpad ready\n");
        Ok(ScratchpadModule { _reg: reg })
    }
}
// No explicit exit: dropping _reg deregisters the misc device.
```

Test it:

```c
/* test_scratchpad.c — gcc -O2 -o test_scratchpad test_scratchpad.c */
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <sys/ioctl.h>

#define SCRATCH_IOC_MAGIC 'S'
#define SCRATCH_IOC_CLEAR  _IO(SCRATCH_IOC_MAGIC, 1)
#define SCRATCH_IOC_GETLEN _IOR(SCRATCH_IOC_MAGIC, 2, unsigned int)
#define SCRATCH_IOC_SETCAP _IOW(SCRATCH_IOC_MAGIC, 3, unsigned int)

int main(void)
{
	char buf[128];
	unsigned int len, cap = 8192;
	int fd = open("/dev/rust_scratchpad", O_RDWR);
	if (fd < 0) { perror("open"); return 1; }

	ioctl(fd, SCRATCH_IOC_SETCAP, &cap);
	write(fd, "hello rust driver", 17);
	ioctl(fd, SCRATCH_IOC_GETLEN, &len);
	printf("length after write: %u\n", len);

	int fd2 = open("/dev/rust_scratchpad", O_RDONLY);
	/* Independent per-fd cursor, shared per-device buffer -- verify. */
	printf("fd  read: %zd bytes\n", read(fd,  buf, sizeof(buf)));
	printf("fd2 read: %zd bytes\n", read(fd2, buf, sizeof(buf)));

	/* Error paths */
	if (ioctl(fd, _IO('X', 99)) < 0) perror("bad ioctl (expect ENOTTY)");
	cap = 0;
	if (ioctl(fd, SCRATCH_IOC_SETCAP, &cap) < 0)
		perror("cap=0 (expect EINVAL)");

	close(fd); close(fd2);
	return 0;
}
```

Then **remove the module while a file descriptor is open** and observe what happens. Compare
with the C version of Ch. 29 and the class of bug it has to guard against by hand.

### Lab 82.2 — A platform driver with device tree

```rust
// SPDX-License-Identifier: GPL-2.0
//! A platform driver with OF matching, MMIO, clocks, and per-variant config.

use kernel::prelude::*;
use kernel::{
    c_str, device, devres::Devres, io::mem::IoMem, of, platform,
    sync::{new_mutex, Mutex},
};

module! {
    type: MyPlatDriver,
    name: "rust_myplat",
    author: "You",
    description: "A platform driver in Rust",
    license: "GPL",
}

const REG_SIZE: usize = 0x1000;
const REG_ID:      usize = 0x00;
const REG_CONTROL: usize = 0x04;
const REG_STATUS:  usize = 0x08;

const CTRL_ENABLE: u32 = 1 << 0;
const CTRL_RESET:  u32 = 1 << 1;

/// Per-variant configuration, the typed replacement for
/// `of_device_get_match_data()` returning `const void *`.
#[derive(Copy, Clone, Debug)]
struct Config {
    max_channels: u32,
    has_dma: bool,
}

const CONFIG_V1: Config = Config { max_channels: 4,  has_dma: false };
const CONFIG_V2: Config = Config { max_channels: 16, has_dma: true  };

struct Inner {
    enabled: bool,
    channels_used: u32,
}

#[pin_data(PinnedDrop)]
struct MyPlatDriver {
    dev: ARef<device::Device>,
    regs: Devres<IoMem<REG_SIZE>>,
    config: Config,
    #[pin]
    inner: Mutex<Inner>,
}

kernel::of_device_table!(
    OF_TABLE,
    MODULE_OF_TABLE,
    <MyPlatDriver as platform::Driver>::IdInfo,
    [
        (of::DeviceId::new(c_str!("vendor,myplat-v1")), CONFIG_V1),
        (of::DeviceId::new(c_str!("vendor,myplat-v2")), CONFIG_V2),
    ]
);

impl platform::Driver for MyPlatDriver {
    type IdInfo = Config;
    const OF_ID_TABLE: Option<of::IdTable<Self::IdInfo>> = Some(&OF_TABLE);

    fn probe(pdev: &platform::Device<device::Core>,
             info: Option<&Self::IdInfo>) -> Result<Pin<KBox<Self>>>
    {
        let dev = pdev.as_ref();
        let config = *info.ok_or(ENODEV)?;

        dev_info!(dev, "probing: max_channels={} dma={}\n",
                  config.max_channels, config.has_dma);

        // MMIO. Devres ties the iounmap to the device's lifetime, which
        // is exactly what devm_ioremap_resource does in C.
        let regs = pdev.ioremap_resource_sized::<REG_SIZE>(0)?;

        // Optional DT property with a sensible default.
        let channels: u32 = dev
            .property_read::<u32>(c_str!("vendor,num-channels"))
            .unwrap_or(config.max_channels);
        if channels > config.max_channels {
            dev_err!(dev, "num-channels {} exceeds max {}\n",
                     channels, config.max_channels);
            return Err(EINVAL);
        }

        // Touch the hardware: read the ID register to confirm it is there.
        {
            let io = regs.try_access().ok_or(ENXIO)?;
            let id = io.read32(REG_ID);
            dev_info!(dev, "hardware id = {:#010x}\n", id);
            io.write32(CTRL_RESET, REG_CONTROL);
            io.write32(CTRL_ENABLE, REG_CONTROL);
        }

        let state = KBox::pin_init(
            try_pin_init!(MyPlatDriver {
                dev: dev.into(),
                regs,
                config,
                inner <- new_mutex!(
                    Inner { enabled: true, channels_used: 0 },
                    "MyPlatDriver::inner"
                ),
            }),
            GFP_KERNEL,
        )?;

        dev_info!(state.dev, "probe complete\n");
        Ok(state)
    }
}

#[pinned_drop]
impl PinnedDrop for MyPlatDriver {
    fn drop(self: Pin<&mut Self>) {
        // Quiesce the hardware. Everything else -- the iomap, the ARef,
        // the mutex -- releases itself.
        if let Some(io) = self.regs.try_access() {
            io.write32(0, REG_CONTROL);
        }
        dev_info!(self.dev, "removed\n");
    }
}
```

With the matching device tree node:

```dts
myplat@10000000 {
	compatible = "vendor,myplat-v2";
	reg = <0x10000000 0x1000>;
	interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>;
	clocks = <&clkc CLK_MYPLAT>;
	clock-names = "core";
	vendor,num-channels = <8>;
	status = "okay";
};
```

Test with QEMU and a DT overlay exactly as in Ch. 32 Lab 3. The driver should probe, print
the hardware ID, and unbind cleanly:

```bash
echo 10000000.myplat > /sys/bus/platform/drivers/rust_myplat/unbind
echo 10000000.myplat > /sys/bus/platform/drivers/rust_myplat/bind
rmmod rust_myplat
```

**Run the bind/unbind loop a thousand times and watch `slabtop`.** A leak here means an
unbalanced `into_foreign`/`from_foreign` or a missing `Drop`, and this is the fastest way to
find it.

### Lab 82.3 — Deliberately break the lifetime rules

Each of these is a real bug you would have to review for in C. Verify the compiler's answer.

```rust
// 1. Return a reference derived from a guard.
fn bad1(s: &Shared) -> &[u8] {
    let g = s.buffer.lock();
    &g[..]              // error[E0515]: cannot return value referencing
}                       //   local variable `g`
                        // ==> "do not use data after unlocking", enforced.

// 2. Keep drvdata after remove.
// There is no way to express this: from_foreign consumes the pointer and
// the driver core zeroes it. The type system plus the adapter make the
// use-after-free unreachable from safe code.

// 3. Use MMIO after the device is unbound.
fn bad3(d: &MyPlatDriver) -> u32 {
    let io = d.regs.try_access().unwrap();   // <-- unwrap can PANIC
    io.read32(REG_ID)
}
// Correct:
fn good3(d: &MyPlatDriver) -> Result<u32> {
    let io = d.regs.try_access().ok_or(ENXIO)?;
    Ok(io.read32(REG_ID))
}
// Devres returns Option because after device removal the mapping is gone.
// The Option IS the "is this still valid?" check that C omits.

// 4. Out-of-range MMIO offset.
// io.read32(0x2000);   // with IoMem<0x1000>: COMPILE error.
//   ==> a whole class of register-offset typos, caught at build time.

// 5. Log an error on EPROBE_DEFER.
//   Not a compile error -- still a review finding. Rust does not fix
//   everything, and knowing which is which is the point.
```

### Lab 82.4 — Side-by-side with the C version

Take your C platform driver from Ch. 31 and put the two in adjacent editor panes. Produce
this table for your own code:

| Concern | C lines | Rust lines | Who enforces |
|---|---|---|---|
| State struct definition | | | — |
| Resource acquisition | | | — |
| **Error unwinding in probe** | | **0** | compiler |
| **`remove()`** | | **0–5** | `Drop` |
| Match table | | | macro (terminator) |
| Match data typing | cast from `void *` | typed | compiler |
| MMIO bounds | none | compile-time | compiler |
| drvdata lifecycle | manual | `ForeignOwnable` | adapter, reviewed once |
| Locking annotation | comment | type | compiler |

The number that usually surprises people is the third row. **In a driver with six resources,
the C error ladder is typically 30–45 lines that exist only to undo things, and every one of
them is a place to introduce a bug.**

### Lab 82.5 — The two-lifetime test

This is the lab that proves you understand §T.4.

```bash
# 1. Load the scratchpad module and open a file descriptor that stays open.
exec 7<> /dev/rust_scratchpad
echo "data" >&7

# 2. Unload the module while the fd is open.
rmmod rust_scratchpad
#    Expect: EBUSY, because the module's refcount is held by the open file.
#    That is the C behaviour too (try_module_get in the fops).

# 3. Now for a platform driver with a chardev: unbind (not unload) while
#    an fd is open.
echo 10000000.myplat > /sys/bus/platform/drivers/rust_myplat/unbind

# 4. Use the still-open fd.
cat <&7
#    CORRECT behaviour: -ENODEV, cleanly.
#    INCORRECT behaviour: oops, or silent garbage.

exec 7<&-
```

Implement the `Option<LiveResources>` pattern from §T.4 and verify step 4 returns `-ENODEV`.
Then remove the `Option` and observe what the compiler will and will not let you do wrong —
**this is a place where the type system helps but does not decide for you**, which is a
useful calibration.

---

## 3. Mastery drills

1. Port a complete C platform driver from Part 2 to Rust, feature for feature. Produce the
   line-count table of Lab 82.4 and a written list of every C invariant that became a type.

2. Write a misc device that supports `poll()`/`epoll` using `PollCondVar`. Verify with a
   userspace `epoll` program that blocks and is woken.

3. Implement `mmap()` on a misc device (a shared buffer). Determine what the abstraction
   offers, what you must do with raw `bindings`, and write the safety contract for the
   `unsafe` you need.

4. Build the two-lifetime scenario deliberately wrong (device state freed while an fd is
   open), run it under KASAN, and capture the report. Then fix it with `Arc` and show the
   report disappears. This is the single most convincing demo you can build.

5. Write a platform driver that requests a threaded IRQ, handles it, and communicates with
   process context through a `CondVar`. Verify with a QEMU device that raises interrupts.

6. Implement `-EPROBE_DEFER` handling correctly for a driver with three deferred
   dependencies. Verify the probe order with `initcall_debug` and
   `/sys/kernel/debug/devices_deferred`.

7. Add sysfs attributes to your Rust driver. Determine whether the abstraction exists; if
   not, write it, including the `show`/`store` signature and the string handling.

8. Measure the binary size and probe latency of a Rust driver versus its C equivalent. Account
   for every difference. Report whether the Rust module's `.text` is larger and why.

9. Write a driver for a device that appears on **both** OF and ACPI, with a single `IdInfo`
   type. Explain how the adapter dispatches and what happens when neither table matches.

10. Stress-test bind/unbind 10,000 times under `CONFIG_DEBUG_KMEMLEAK` and
    `CONFIG_DEBUG_OBJECTS`. Report any growth and trace it to its cause.

11. Take the `Adapter` machinery in `rust/kernel/driver.rs` and write a new bus adapter for a
    bus that does not have one (i2c, spi, or a toy bus of your own). Get the
    `into_foreign`/`from_foreign` pairing right and defend it in review.

12. Write the onboarding document: "How to write a Rust platform driver in this codebase."
    Include the state-struct shape, the resource-ordering rule, the two-lifetime pattern, and
    the review checklist.

---

## 4. Further reading

**Code to read, in this order**
1. `samples/rust/rust_misc_device.rs` — the most complete small example; read it fully
2. `samples/rust/rust_driver_platform.rs` — the minimal platform driver
3. `rust/kernel/platform.rs` — the adapter; ~300 lines and it explains the whole model
4. `rust/kernel/miscdevice.rs` — the `Ptr` associated type and `ForeignOwnable` in practice
5. `rust/kernel/io.rs` — the const-generic bounds checking
6. `drivers/net/phy/ax88796b_rust.rs` — a real, merged, production Rust driver

**Kernel documentation**
- `Documentation/driver-api/driver-model/` — unchanged by the language; the model is the model
- `Documentation/devicetree/bindings/` — DT bindings are language-agnostic
- `make rustdoc` → `kernel::platform`, `kernel::miscdevice`, `kernel::of`

**Cross-references in this curriculum**
- Ch. 26–28 — the driver model, probe/bind, and `devres`, in C
- Ch. 29–31 — char devices, misc devices, and platform devices, in C
- Ch. 32 — device tree
- Ch. 79 — `ForeignOwnable`, `ARef`, `Result`
- Ch. 80 — `pin-init` for the state struct
- Ch. 81 — the locking used inside it

**LWN and talks**
- "A first look at Rust in the 6.1 kernel" and the driver-by-driver merge coverage
- Kangrejos talks on the driver abstractions, particularly the `Devres` and device-lifetime
  discussions — these are the hardest design questions in the area and are still evolving
- The LKML threads on the platform and PCI abstractions — good examples of what a maintainer
  asks of a new abstraction

→ Next: [83-rust-drivers-2.md](83-rust-drivers-2.md)

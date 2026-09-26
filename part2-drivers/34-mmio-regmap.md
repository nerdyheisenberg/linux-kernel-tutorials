# Chapter 34 — MMIO, `ioremap`, Ordering, and `regmap`

> **Goal:** talk to hardware registers correctly. Understand why `writel()` is not a memory
> write, why your register access can be reordered, cached, coalesced, or simply lost, and
> why `regmap` exists.

---

## Theory & First Principles

### T.0 — Start here: why `*ptr = 1` is a bug

You have a device register at a mapped address. Write to it the obvious way:

```c
u32 *reg = ioremap(0x02020000, 0x1000);
reg[0x40/4] = ENABLE;        /* ← every part of this is wrong */
```

**Five distinct failures, and each one is real:**

**① The compiler may delete it.**
```c
reg[0x10] = RESET;
reg[0x10] = ENABLE;   /* compiler: "the first store is dead" -> DELETED */
```
For memory that is correct. For a register where writing `RESET` *does something*, it is
catastrophic. The compiler has no idea this address has side effects.

**② The compiler may reorder, merge, or split it.** It can turn two adjacent 32-bit stores
into one 64-bit store, or split a 32-bit store into bytes. A device that latches on a 32-bit
write to a specific offset sees neither.

**③ The CPU may reorder it.** Stores retire into a store buffer and drain later (Ch. 13
§T.0). Your "write the descriptor, then ring the doorbell" becomes "ring the doorbell, then
write the descriptor" as observed by the device.

**④ The byte order may be wrong.** The device is little-endian; your CPU may not be. A raw
store does no conversion.

**⑤ It may not be a memory access at all.** On some architectures device access uses
different instructions or a different address space entirely.

**So the kernel does not let you dereference it:**

```c
void __iomem *base = ioremap(0x02020000, 0x1000);

writel(ENABLE, base + 0x40);      /* volatile, ordered, endian-correct */
u32 v = readl(base + 0x44);
```

`__iomem` is a `sparse` annotation (Ch. 08 §T.2) marking the pointer as *not dereferenceable*.
`make C=1` flags `*base` as an error. **The type system is being used to prevent a class of
bug that the language cannot otherwise express** — and in Rust this becomes a real type with
no `Deref` at all (Ch. 83 §T.3).

**Now the part that catches experienced people.** `writel` orders MMIO against *other MMIO*.
It does **not** order MMIO against *normal memory*:

```c
/* WRONG -- and it works on your desk, and corrupts data under load */
desc->addr = dma_addr;
desc->len  = len;
desc->flags = OWN_DEVICE;          /* normal memory, the device DMAs this */
writel(idx, base + DOORBELL);      /* MMIO -- "go" */

/* RIGHT */
desc->addr = dma_addr;
desc->len  = len;
desc->flags = OWN_DEVICE;
dma_wmb();                         /* ← descriptor visible BEFORE the doorbell */
writel(idx, base + DOORBELL);
```

Without `dma_wmb()`, the device can see the doorbell and read a descriptor that is not
finished. **The symptom is silent data corruption that appears only at high load, on some
platforms, sometimes** — which is to say, the worst possible symptom. This is the single most
important line in the chapter (§T.5), and it is invisible when it is missing.

**And the final observation, which motivates `regmap`.** Every driver that talks to registers
writes the same things: read-modify-write of bitfields, caching of write-only registers,
endianness handling, a spinlock for RMW atomicity, and — for I²C/SPI devices — an entirely
different transport with the same register model. That duplication was hundreds of drivers
deep, with the same bugs in each. `regmap` (§T.7) abstracts *register access itself*, and it
is one of the clearest wins in the kernel's history: net-negative diff, fewer bugs, and a
driver that works over MMIO or I²C or SPI with one line changed.

```bash
cat /proc/iomem | head -20                    # claimed physical regions
sudo devmem2 0xfed00000 w                     # read a register directly (careful!)
ls /sys/kernel/debug/regmap/                  # every regmap device
cat /sys/kernel/debug/regmap/*/registers | head
```

---

### T.1 MMIO is not memory, and the difference is the whole chapter

A device register at physical address `0x1c020000` looks like memory: you load and store to
it. It is not memory, and every difference matters:

| | Normal memory | **MMIO** |
|---|---|---|
| Reads are idempotent | yes | **no** — reading a FIFO *consumes* data |
| Writes are idempotent | yes | **no** — writing a W1C bit clears it |
| Speculative access is safe | yes | **no** — a speculative read can trigger a transaction |
| Caching is transparent | yes | **fatal** — a cached register is a stale register |
| Write combining is safe | yes | **usually not** — merging two writes loses one |
| Reordering is invisible | mostly | **no** — register writes have required order |
| Access width is flexible | yes | **no** — many registers require exactly 32-bit access |
| Access has side effects | no | **yes, by design** |

The consequence: **you must prevent the compiler and the CPU from treating register accesses
as ordinary memory operations.** That is exactly what `readl()`/`writel()` do, and it is why
a raw pointer dereference to `ioremap`ped memory is a bug even though it compiles.

```c
/* ★ WRONG — the compiler may reorder, merge, split, elide, or speculate these */
u32 *regs = (u32 *)base;
regs[0] = CTRL_RESET;
regs[0] = CTRL_ENABLE;      /* the compiler may delete the first store entirely */
while (regs[1] & BUSY) ;    /* the compiler may hoist the load out of the loop */

/* ★ RIGHT */
writel(CTRL_RESET,  base + CTRL);
writel(CTRL_ENABLE, base + CTRL);
readl_poll_timeout(base + STATUS, val, !(val & BUSY), 10, 10000);
```

The `__iomem` sparse annotation (Ch. 08 T.2) exists precisely to make the wrong version a
*compile-time* error: `__iomem` pointers are `noderef`, so dereferencing one fails
`make C=1`. **Run sparse on every driver you write.**

### T.2 `ioremap`: creating an uncached window

Physical device registers are not in the kernel's direct map (Ch. 22 T.1) — that map covers
RAM, with RAM's caching attributes. `ioremap()` creates a *new* virtual mapping with
**device memory attributes**:

```c
void __iomem *base = ioremap(phys_addr, size);          /* strongly-ordered / device-nGnRE */
void __iomem *base = ioremap_wc(phys_addr, size);       /* write-combining (framebuffers) */
void __iomem *base = ioremap_cache(phys_addr, size);    /* cacheable — ONLY for real RAM */
void __iomem *base = ioremap_np(phys_addr, size);       /* non-posted (arm64, PCI config) */
iounmap(base);
```

What "device memory attributes" means per architecture:

| Arch | Attribute | Guarantees |
|---|---|---|
| **x86** | UC (uncacheable, PAT/MTRR) | no cache, no speculation, no reordering, no merging |
| **arm64** | `Device-nGnRE` | **n**on-**G**athering, non-**R**e-ordering, **E**arly-write-ack |
| **arm64** (strictest) | `Device-nGnRnE` | also no early ack — writes wait for the device |
| **write-combining** | Normal-NC / WC | **may merge and reorder writes** — much faster for bulk |

The arm64 naming is worth decoding because it names exactly the three hazards:
- **Gathering** — merging two accesses into one. Fatal for FIFOs and W1C registers.
- **Reordering** — device accesses passing each other. Fatal for command sequences.
- **Early ack** — the write completes at a buffer, not the device. Affects when you *know*
  it landed.

**Write-combining is a deliberate, dangerous optimization.** A framebuffer or a doorbell
ring benefits enormously (8 separate 4-byte writes become one 32-byte burst), but a control
register must never be WC. Use `ioremap_wc()` only where the device documentation says the
region tolerates merging, and use `wmb()`/`writel()` to flush before anything order-sensitive.

Managed forms (Ch. 28) are what you actually write:

```c
base = devm_ioremap_resource(dev, res);                    /* request + map */
base = devm_platform_ioremap_resource(pdev, 0);            /* + get the resource */
base = devm_platform_ioremap_resource_byname(pdev, "regs");/* ★ by name */
base = devm_ioremap_wc(dev, start, size);
base = pcim_iomap_region(pdev, bar, name);                 /* PCI (Ch. 37) */
```

### T.3 The accessors, and the ordering each one provides

```c
/* Little-endian, with implicit ordering — the default */
u8  readb(addr);   u16 readw(addr);   u32 readl(addr);   u64 readq(addr);
void writeb(v, addr); writew(v, addr); writel(v, addr); writeq(v, addr);

/* Relaxed: NO ordering against other memory or MMIO — much faster */
readb_relaxed / readw_relaxed / readl_relaxed / readq_relaxed
writeb_relaxed / writew_relaxed / writel_relaxed / writeq_relaxed

/* Explicit endianness (on-device register endianness ≠ CPU endianness) */
ioread32be / iowrite32be / __raw_readl / __raw_writel

/* Bulk: repeatedly access ONE register (FIFO draining) */
readsl(addr, buf, count);   writesl(addr, buf, count);
ioread32_rep / iowrite32_rep

/* Copy to/from a memory-like MMIO region (framebuffers) */
memcpy_fromio(dst, src_io, n);  memcpy_toio(dst_io, src, n);  memset_io(dst_io, c, n);
```

**The ordering semantics of `readl()`/`writel()` are the part everyone gets wrong.** In
Linux's model:

- `writel()` = `dma_wmb()` (or stronger) **before** the store. So all *prior* memory writes
  — including DMA descriptors you just built — are visible to the device before the register
  write reaches it.
- `readl()` = the load, then `dma_rmb()` (or stronger) **after**. So data the device DMA'd
  before setting a status bit is visible to you after you read that bit.
- MMIO accesses to the *same* device are ordered with respect to each other (on `Device-nGnRE`
  and on x86 UC).

This is precisely calibrated to the **canonical driver pattern**:

```c
/* TX path: build a descriptor in RAM, then ring the doorbell */
desc->addr = dma_addr;
desc->len  = len;
desc->flags = DESC_OWN;
writel(1, base + DOORBELL);     /* ★ the implicit barrier guarantees the device
                                 *   sees the descriptor before the doorbell    */

/* RX path: read a status register, then read the data the device DMA'd */
status = readl(base + STATUS);  /* ★ the implicit barrier guarantees we see the
                                 *   DMA'd data written before the status bit   */
if (status & RX_DONE)
	process(rx_buffer);
```

`*_relaxed()` removes those barriers. It is correct and much faster when you are doing
several accesses to the same device with no RAM interaction between them — a register-read
loop, a bulk FIFO drain, an interrupt-status poll. It is **wrong** anywhere the doorbell or
status pattern above applies.

```c
/* Idiomatic use of relaxed: many accesses, one barrier */
for (i = 0; i < n; i++)
	writel_relaxed(data[i], base + FIFO);
writel(CMD_GO, base + CTRL);        /* ★ the last one is ordered — flushes the rest */
```

**On arm64 this matters enormously** — `readl()` emits a `dmb` that `readl_relaxed()` does
not, and in a hot interrupt handler the difference is measurable. On x86 the relaxed and
ordered forms often generate identical code, which is exactly why x86-only testing hides
these bugs (Ch. 13 T.10).

### T.4 Posted writes: the write that has not happened yet

A `writel()` to a PCIe or AXI device is **posted**: the CPU's store buffer hands it to the
interconnect, which acknowledges immediately. The write is *in flight*, not *complete*.

Three consequences that produce real, hard-to-find bugs:

**(1) Clearing an interrupt does not take effect immediately.**
```c
static irqreturn_t my_isr(int irq, void *d)
{
	u32 status = readl(base + STATUS);

	writel(status, base + STATUS);     /* W1C ack — POSTED, may not have landed */
	handle(status);
	return IRQ_HANDLED;                /* ★ the IRQ may re-assert: spurious storm */
}
```
The classic fix is a **read-back to flush**:
```c
	writel(status, base + STATUS);
	readl(base + STATUS);              /* ★ forces the write to complete */
```
A read to the same device cannot pass a posted write, so the read-back is a flush. **This is
why legacy INTx drivers are full of apparently-pointless reads** — and why MSI drivers are
not (Ch. 17 T.5: MSI ordering makes it unnecessary).

**(2) Delays after a write do not start when you think.**
```c
	writel(RESET, base + CTRL);
	udelay(10);                        /* ★ the write may still be in flight! */
	writel(ENABLE, base + CTRL);
/* Correct: */
	writel(RESET, base + CTRL);
	readl(base + CTRL);                /* flush */
	udelay(10);
	writel(ENABLE, base + CTRL);
```

**(3) Unmapping or powering down after a write can lose it.** Flush before
`iounmap()`, before disabling the clock, and before `pm_runtime_put()`.

arm64 offers `ioremap_np()` (non-posted, `Device-nGnRnE`) where the architecture or device
requires writes to be acknowledged by the endpoint — PCI config space, and some
interconnects. It is much slower; use it only when specified.

### T.5 The barrier zoo, and which one you actually need

```c
mb() / rmb() / wmb()          /* full system barriers: CPU + device + DMA */
dma_rmb() / dma_wmb()         /* order accesses to DMA-coherent memory — CHEAPER */
smp_mb() / smp_rmb() / smp_wmb()  /* CPU↔CPU only (Ch. 13); compiled out on UP */
__iowmb() / __iormb()         /* what readl/writel use internally */
mmiowb()                      /* REMOVED in 5.2 — see below */
```

The decision rule:

| You are ordering… | Use |
|---|---|
| CPU ↔ CPU (shared kernel data) | `smp_*` (Ch. 13) |
| CPU ↔ device via **DMA-coherent RAM** | `dma_rmb()` / `dma_wmb()` |
| CPU ↔ device via **MMIO** | nothing — `readl`/`writel` already do it |
| MMIO relative to **non-coherent** DMA | `mb()`/`wmb()`, plus `dma_sync_*` (Ch. 35) |
| Bulk relaxed MMIO, then a doorbell | `*_relaxed()` × N, then one ordered `writel()` |

**`mmiowb()` is a good story.** It existed to solve a real problem: on some architectures
(ia64, mips, sh), MMIO writes from two CPUs holding the same spinlock could arrive at the
device out of order, because the spinlock's release barrier did not order MMIO. Drivers had
to call `mmiowb()` before `spin_unlock()` — and essentially nobody did, correctly.

In 5.2 the kernel **moved the responsibility into `spin_unlock()` itself** (tracking whether
MMIO was performed inside the critical section) and deleted `mmiowb()` from driver code.
That is an important general lesson: *when a correctness obligation is routinely forgotten,
move it into the primitive rather than documenting it harder.* (Compare: `devm_` in Ch. 28,
`guard()` in Ch. 05, `dev_groups` in Ch. 26.)

### T.6 Port I/O, and why it barely matters now

x86 has a separate 16-bit I/O address space accessed with `in`/`out` instructions:

```c
u8 inb(port); u16 inw(port); u32 inl(port);
void outb(v, port); outw(v, port); outl(v, port);
request_region(port, n, "mydrv"); release_region(port, n);
```

It exists for historical PC compatibility (8250 UARTs at 0x3f8, the PIC at 0x20, the PS/2
controller at 0x60). It is:
- x86-only in spirit (other architectures emulate it over a PCI I/O window),
- 64 KiB total,
- **always non-posted and slow** (~1 µs per access — 100× an MMIO write),
- deprecated for new hardware; PCIe devices are expected to use MMIO BARs.

`ioport_map()` and the `ioread32`/`iowrite32` family let one driver handle both spaces
(`pci_iomap()` returns a cookie that works with either), which is the modern approach.

```bash
cat /proc/ioports | head -20
cat /proc/iomem | head -20
```

### T.7 `regmap`: abstracting the bus away from the register model

Consider a PMIC with 200 registers. It might be attached over I²C, SPI, or memory-mapped —
and the *same chip family* often offers all three. Without abstraction you write the driver
three times.

Worse, each bus has different concerns: I²C/SPI accesses **sleep** (so no spinlocks, no
interrupt context); MMIO does not. Register caching is valuable over a 400 kHz I²C bus and
pointless over MMIO.

**`regmap`** (Mark Brown, 3.1+) abstracts *register access* as its own layer:

```c
static const struct regmap_config my_regmap_config = {
	.reg_bits        = 8,                       /* register address width */
	.val_bits        = 8,                       /* register value width */
	.max_register    = 0xff,
	.reg_defaults    = my_defaults,             /* for cache initialization */
	.num_reg_defaults = ARRAY_SIZE(my_defaults),
	.cache_type      = REGCACHE_MAPLE,          /* ★ or RBTREE / FLAT / NONE */
	.writeable_reg   = my_writeable_reg,        /* callbacks: which regs are valid */
	.readable_reg    = my_readable_reg,
	.volatile_reg    = my_volatile_reg,         /* ★ never cache these */
	.precious_reg    = my_precious_reg,         /* reading has side effects */
	.val_format_endian = REGMAP_ENDIAN_BIG,
	.use_single_read = false,                   /* can we do bulk? */
};

/* One line per bus — the driver above is identical */
regmap = devm_regmap_init_i2c(client, &my_regmap_config);
regmap = devm_regmap_init_spi(spi, &my_regmap_config);
regmap = devm_regmap_init_mmio(dev, base, &my_regmap_config);
regmap = devm_regmap_init_mmio_clk(dev, "apb", base, &cfg);   /* clocks the bus */

/* Bus-agnostic access */
regmap_read(regmap, REG_STATUS, &val);
regmap_write(regmap, REG_CTRL, 0x42);
regmap_update_bits(regmap, REG_CTRL, MASK_ENABLE, MASK_ENABLE);  /* ★ read-modify-write */
regmap_bulk_read(regmap, REG_DATA, buf, 16);
regmap_bulk_write(regmap, REG_DATA, buf, 16);
regmap_read_poll_timeout(regmap, REG_STATUS, val, val & READY, 100, 10000);
regmap_multi_reg_write(regmap, init_seq, ARRAY_SIZE(init_seq));  /* an init sequence */
```

What you get beyond bus abstraction — and each item is a real win:

1. **Register cache.** `REGCACHE_MAPLE` (6.4+, replacing rbtree) or `REGCACHE_FLAT`. A read
   of a non-volatile register never touches the bus. On a 100 kHz I²C PMIC this is the
   difference between usable and unusable.
2. **Locking, done once.** regmap takes a mutex (or spinlock for `fast_io` MMIO) around
   read-modify-write, so `regmap_update_bits()` is atomic without every driver reimplementing
   it.
3. **Suspend/resume for free.** `regcache_mark_dirty()` + `regcache_sync()` restores the
   entire register state after the chip loses power — a substantial amount of code you do
   not write.
4. **debugfs.** Every regmap appears under `/sys/kernel/debug/regmap/<name>/` with a
   `registers` dump and a `cache_only`/`cache_bypass` control. **This is the single most
   useful debugging surface in embedded driver work.**
5. **`regmap_irq`** — a generic implementation of the "status register + mask register"
   interrupt-controller pattern that nearly every PMIC has, so you get a real `irq_chip`
   (Ch. 17 T.6) without writing one.
6. **Endianness and paging** handled declaratively.

```c
/* regmap_irq: an entire irq_chip from a table */
static const struct regmap_irq my_irqs[] = {
	REGMAP_IRQ_REG(MY_IRQ_ALARM,  0, BIT(0)),
	REGMAP_IRQ_REG(MY_IRQ_OVERTEMP, 0, BIT(1)),
};
static const struct regmap_irq_chip my_irq_chip = {
	.name           = "mypmic",
	.status_base    = REG_INT_STATUS,
	.mask_base      = REG_INT_MASK,
	.ack_base       = REG_INT_STATUS,
	.num_regs       = 1,
	.irqs           = my_irqs,
	.num_irqs       = ARRAY_SIZE(my_irqs),
};
devm_regmap_add_irq_chip(dev, regmap, client->irq, IRQF_ONESHOT, 0,
			 &my_irq_chip, &irq_data);
virq = regmap_irq_get_virq(irq_data, MY_IRQ_ALARM);
```

**When *not* to use regmap:** a performance-critical MMIO datapath. `regmap_write()` on MMIO
costs a function call, a lock, a cache lookup, and a validity check — perhaps 50–100 ns
versus 5 ns for `writel_relaxed()`. For a NIC's TX doorbell, use `writel()`. For a PMIC's
configuration, use regmap. Many drivers correctly use both.

### T.8 Register definition style

```c
/* ★ Use bit-field helpers, not hand-rolled shifts */
#define MY_CTRL              0x00
#define  MY_CTRL_ENABLE      BIT(0)
#define  MY_CTRL_RESET       BIT(1)
#define  MY_CTRL_MODE        GENMASK(5, 4)
#define  MY_CTRL_SPEED       GENMASK(15, 8)

u32 v = readl(base + MY_CTRL);

v &= ~MY_CTRL_MODE;
v |= FIELD_PREP(MY_CTRL_MODE, mode);        /* ★ include/linux/bitfield.h */
v |= FIELD_PREP(MY_CTRL_SPEED, speed);
writel(v, base + MY_CTRL);

mode = FIELD_GET(MY_CTRL_MODE, readl(base + MY_CTRL));

/* Compile-time checked: FIELD_PREP warns if the value doesn't fit the mask */
```

`FIELD_PREP`/`FIELD_GET` (`include/linux/bitfield.h`) are checked at compile time and
eliminate the classic `(val << SHIFT) & MASK` errors. **Use them; reviewers ask for them.**

Also: `regmap_field` does the same for regmap, precomputing the register/mask/shift:
```c
static const struct reg_field my_fields[] = {
	[F_MODE]  = REG_FIELD(REG_CTRL, 4, 5),
	[F_SPEED] = REG_FIELD(REG_CTRL, 8, 15),
};
devm_regmap_field_bulk_alloc(dev, regmap, priv->fields, my_fields, ARRAY_SIZE(my_fields));
regmap_field_write(priv->fields[F_MODE], mode);
```

---

## 1. Internals

### 1.1 Source map

```
include/asm-generic/io.h        ★★ the generic readl/writel and their barriers
arch/x86/include/asm/io.h       x86: UC via PAT, in/out
arch/arm64/include/asm/io.h     ★ arm64: __iormb/__iowmb, Device-nGnRE
arch/arm64/mm/ioremap.c
mm/ioremap.c, lib/devres.c      ioremap, devm_ioremap
include/linux/iopoll.h          ★ readl_poll_timeout and friends
include/linux/bitfield.h        ★ FIELD_PREP / FIELD_GET
drivers/base/regmap/            ★★ regmap: regmap.c, regcache*.c, regmap-irq.c,
                                   regmap-i2c.c, regmap-spi.c, regmap-mmio.c
include/linux/regmap.h          ★ read the whole header
Documentation/driver-api/device-io.rst   ★★ the normative MMIO document
Documentation/driver-api/regmap.rst
Documentation/memory-barriers.txt §"Kernel I/O barrier effects"  ★
```

### 1.2 What `writel()` expands to

```c
/* include/asm-generic/io.h */
#define writel(v, c) ({ __io_bw(); __raw_writel(cpu_to_le32(v), c); __io_aw(); })
#define readl(c)     ({ u32 __v; __io_br(); __v = le32_to_cpu(__raw_readl(c)); __io_ar(__v); __v; })

/* arm64 (arch/arm64/include/asm/io.h): */
#define __io_ar(v)   __iormb(v)      /* dmb oshld + a dependency on v */
#define __io_bw()    __iowmb()       /* dmb oshst */
#define __io_br()    do {} while (0)
#define __io_aw()    do {} while (0)

/* x86: all four are compiler barriers only — the UC memory type does the rest */
```

Read this and T.3 becomes concrete: on arm64 `writel()` really does emit a `dmb` and
`writel_relaxed()` really does not.

---

## 2. Practice

### Lab 34.1 — Prove the compiler wrecks raw MMIO (T.1)

```c
/* /tmp/mmio.c — compile, don't run */
#include <stdint.h>
void raw(volatile void *b) { }      /* stand-in */

void bad(uint32_t *regs)
{
	regs[0] = 1;
	regs[0] = 2;                /* the compiler deletes the first store */
	while (regs[1] & 1) ;       /* the compiler hoists the load out */
}

void good(volatile uint32_t *regs)
{
	regs[0] = 1;
	regs[0] = 2;
	while (regs[1] & 1) ;
}
```
```bash
gcc -O2 -S -o - /tmp/mmio.c | sed -n '/^bad:/,/ret/p'
gcc -O2 -S -o - /tmp/mmio.c | sed -n '/^good:/,/ret/p'
```
**Count the stores in each.** Then find the real definitions:
```bash
grep -n -A5 'define __raw_writel' include/asm-generic/io.h arch/arm64/include/asm/io.h
grep -n -A10 'define writel' include/asm-generic/io.h
```

### Lab 34.2 — Catch `__iomem` violations with sparse (T.1)

```c
static int bad_probe(struct platform_device *pdev)
{
	void __iomem *base = devm_platform_ioremap_resource(pdev, 0);
	u32 v;

	v = *(u32 *)base;               /* sparse: dereference of noderef expression */
	*(u32 *)(base + 4) = 0x42;      /* sparse: dereference of noderef expression */
	v = readl(base);                /* correct */
	writel(0x42, base + 4);         /* correct */
	return 0;
}
```
```bash
sudo apt install -y sparse
make C=2 M=$PWD modules 2>&1 | grep -i noderef
```
**Do this once and `__iomem` stops looking like decoration.** Then count them in a real
driver: `git grep -c '__iomem' drivers/net/ethernet/intel/e1000e/`.

### Lab 34.3 — Measure `readl` vs `readl_relaxed` on arm64 (T.3)

```c
#define N 1000000
static int bench_probe(struct platform_device *pdev)
{
	void __iomem *base = devm_platform_ioremap_resource(pdev, 0);
	volatile u32 sink = 0;
	u64 t0;
	int i;

	t0 = ktime_get_ns();
	for (i = 0; i < N; i++) sink += readl(base + STATUS);
	pr_info("readl        : %llu ns/access\n", (ktime_get_ns() - t0) / N);

	t0 = ktime_get_ns();
	for (i = 0; i < N; i++) sink += readl_relaxed(base + STATUS);
	pr_info("readl_relaxed: %llu ns/access\n", (ktime_get_ns() - t0) / N);

	t0 = ktime_get_ns();
	for (i = 0; i < N; i++) writel_relaxed(i, base + SCRATCH);
	pr_info("writel_relaxed: %llu ns/access\n", (ktime_get_ns() - t0) / N);

	t0 = ktime_get_ns();
	for (i = 0; i < N; i++) writel(i, base + SCRATCH);
	pr_info("writel        : %llu ns/access\n", (ktime_get_ns() - t0) / N);
	return 0;
}
```
Run on **x86 and arm64** (QEMU virt with a real MMIO region, or a Raspberry Pi). Compare the
generated assembly:
```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- M=$PWD mmiobench.o
aarch64-linux-gnu-objdump -d mmiobench.o | grep -B2 -A2 'dmb'
```
**On arm64 you will see `dmb` instructions that vanish with `_relaxed`.** On x86 the code is
often identical — which is the whole portability trap of Ch. 13 T.10.

### Lab 34.4 — Reproduce the posted-write interrupt storm (T.4)

Using QEMU's `edu` device (Ch. 17 Lab 17.B) or a device you model:

```c
/* ★ BROKEN: ack is posted, IRQ may re-assert before it lands */
static irqreturn_t storm_isr(int irq, void *d)
{
	struct mydev *m = d;
	u32 st = readl(m->base + IRQ_STATUS);

	if (!st)
		return IRQ_NONE;
	writel(st, m->base + IRQ_ACK);
	m->count++;
	return IRQ_HANDLED;
}

/* ★ FIXED: read back to flush the posted write */
static irqreturn_t fixed_isr(int irq, void *d)
{
	struct mydev *m = d;
	u32 st = readl(m->base + IRQ_STATUS);

	if (!st)
		return IRQ_NONE;
	writel(st, m->base + IRQ_ACK);
	readl(m->base + IRQ_STATUS);     /* ★ flush */
	m->count++;
	return IRQ_HANDLED;
}
```
```bash
# Compare interrupt counts for the same number of device events:
watch -n1 'grep mydev /proc/interrupts'
sudo bpftrace -e 'tracepoint:irq:irq_handler_entry /str(args->name) == "mydev"/ { @ = count(); }'
# The broken version shows many more interrupts per event.

# Find real examples of the read-back idiom:
git grep -n -B2 -A2 'flush.*posted\|posted.*write' drivers/ | head -20
git grep -n 'readl(.*);.*/\* flush \*/' drivers/ | head -10
```

### Lab 34.5 — Write-combining, measured (T.2)

```c
static int wc_probe(struct platform_device *pdev)
{
	struct resource *res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
	void __iomem *uc = devm_ioremap(&pdev->dev, res->start, resource_size(res));
	void __iomem *wc = devm_ioremap_wc(&pdev->dev, res->start, resource_size(res));
	u64 t0;
	int i;

	t0 = ktime_get_ns();
	for (i = 0; i < 100000; i++) writel_relaxed(i, uc + (i & 0x3fc));
	pr_info("UC writes: %llu ns/op\n", (ktime_get_ns() - t0) / 100000);

	t0 = ktime_get_ns();
	for (i = 0; i < 100000; i++) writel_relaxed(i, wc + (i & 0x3fc));
	wmb();
	pr_info("WC writes: %llu ns/op\n", (ktime_get_ns() - t0) / 100000);
	return 0;
}
```
```bash
# On x86, inspect the memory types in use:
sudo cat /sys/kernel/debug/x86/pat_memtype_list 2>/dev/null | head
sudo cat /proc/mtrr
# On arm64:
sudo cat /sys/kernel/debug/kernel_page_tables 2>/dev/null | grep -i device | head
```
Expect WC to be several times faster for sequential writes — and **completely wrong** for a
control register. Explain why in writing.

### Lab 34.6 — A complete regmap-based driver

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * A PMIC-style I2C driver using regmap: cache, irq_chip, and PM restore.
 */
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

#include <linux/i2c.h>
#include <linux/module.h>
#include <linux/regmap.h>
#include <linux/bitfield.h>
#include <linux/pm.h>

#define MYPMIC_REG_ID         0x00
#define MYPMIC_REG_CTRL       0x01
#define  CTRL_ENABLE          BIT(0)
#define  CTRL_MODE            GENMASK(3, 1)
#define MYPMIC_REG_VOLTAGE    0x02
#define MYPMIC_REG_STATUS     0x10   /* volatile */
#define MYPMIC_REG_INT_STATUS 0x11   /* volatile, W1C */
#define MYPMIC_REG_INT_MASK   0x12
#define MYPMIC_REG_MAX        0x1f

struct mypmic { struct regmap *regmap; struct device *dev;
		struct regmap_irq_chip_data *irq_data; };

static bool mypmic_volatile_reg(struct device *dev, unsigned int reg)
{
	switch (reg) {
	case MYPMIC_REG_STATUS:
	case MYPMIC_REG_INT_STATUS:
		return true;                 /* ★ never cache: value changes underneath us */
	default:
		return false;
	}
}
static bool mypmic_writeable_reg(struct device *dev, unsigned int reg)
{
	return reg != MYPMIC_REG_ID && reg <= MYPMIC_REG_MAX;
}

static const struct reg_default mypmic_defaults[] = {
	{ MYPMIC_REG_CTRL,     0x00 },
	{ MYPMIC_REG_VOLTAGE,  0x20 },
	{ MYPMIC_REG_INT_MASK, 0xff },
};

static const struct regmap_config mypmic_regmap_config = {
	.reg_bits         = 8,
	.val_bits         = 8,
	.max_register     = MYPMIC_REG_MAX,
	.volatile_reg     = mypmic_volatile_reg,
	.writeable_reg    = mypmic_writeable_reg,
	.reg_defaults     = mypmic_defaults,
	.num_reg_defaults = ARRAY_SIZE(mypmic_defaults),
	.cache_type       = REGCACHE_MAPLE,     /* ★ */
};

static const struct regmap_irq mypmic_irqs[] = {
	REGMAP_IRQ_REG(0, 0, BIT(0)),          /* overtemp */
	REGMAP_IRQ_REG(1, 0, BIT(1)),          /* undervoltage */
};
static const struct regmap_irq_chip mypmic_irq_chip = {
	.name        = "mypmic",
	.status_base = MYPMIC_REG_INT_STATUS,
	.mask_base   = MYPMIC_REG_INT_MASK,
	.ack_base    = MYPMIC_REG_INT_STATUS,
	.num_regs    = 1,
	.irqs        = mypmic_irqs,
	.num_irqs    = ARRAY_SIZE(mypmic_irqs),
};

static irqreturn_t mypmic_overtemp(int irq, void *data)
{
	dev_warn(((struct mypmic *)data)->dev, "overtemperature!\n");
	return IRQ_HANDLED;
}

static int mypmic_probe(struct i2c_client *client)
{
	struct mypmic *p;
	unsigned int id;
	int ret, virq;

	p = devm_kzalloc(&client->dev, sizeof(*p), GFP_KERNEL);
	if (!p)
		return -ENOMEM;
	p->dev = &client->dev;
	i2c_set_clientdata(client, p);

	p->regmap = devm_regmap_init_i2c(client, &mypmic_regmap_config);
	if (IS_ERR(p->regmap))
		return dev_err_probe(p->dev, PTR_ERR(p->regmap), "regmap init\n");

	ret = regmap_read(p->regmap, MYPMIC_REG_ID, &id);
	if (ret)
		return dev_err_probe(p->dev, ret, "cannot read ID\n");
	if (id != 0xA5)
		return dev_err_probe(p->dev, -ENODEV, "bad ID 0x%02x\n", id);

	/* Atomic read-modify-write, locked by regmap */
	ret = regmap_update_bits(p->regmap, MYPMIC_REG_CTRL,
				 CTRL_ENABLE | CTRL_MODE,
				 CTRL_ENABLE | FIELD_PREP(CTRL_MODE, 2));
	if (ret)
		return ret;

	if (client->irq) {
		ret = devm_regmap_add_irq_chip(p->dev, p->regmap, client->irq,
					       IRQF_ONESHOT, 0, &mypmic_irq_chip,
					       &p->irq_data);
		if (ret)
			return dev_err_probe(p->dev, ret, "irq chip\n");

		virq = regmap_irq_get_virq(p->irq_data, 0);
		ret = devm_request_threaded_irq(p->dev, virq, NULL, mypmic_overtemp,
						IRQF_ONESHOT, "mypmic-overtemp", p);
		if (ret)
			return ret;
	}
	dev_info(p->dev, "probed, id 0x%02x\n", id);
	return 0;
}

static int __maybe_unused mypmic_suspend(struct device *dev)
{
	struct mypmic *p = dev_get_drvdata(dev);

	regcache_cache_only(p->regmap, true);   /* ★ no bus access while suspended */
	regcache_mark_dirty(p->regmap);
	return 0;
}
static int __maybe_unused mypmic_resume(struct device *dev)
{
	struct mypmic *p = dev_get_drvdata(dev);

	regcache_cache_only(p->regmap, false);
	return regcache_sync(p->regmap);        /* ★ restore EVERY register, for free */
}
static SIMPLE_DEV_PM_OPS(mypmic_pm, mypmic_suspend, mypmic_resume);

static const struct of_device_id mypmic_of_match[] = {
	{ .compatible = "acme,mypmic" }, { }
};
MODULE_DEVICE_TABLE(of, mypmic_of_match);

static struct i2c_driver mypmic_driver = {
	.driver = { .name = "mypmic", .of_match_table = mypmic_of_match,
		    .pm = pm_ptr(&mypmic_pm) },
	.probe  = mypmic_probe,
};
module_i2c_driver(mypmic_driver);
MODULE_LICENSE("GPL");
```

```bash
# Test against an emulated I2C device (Ch. 40) or i2c-stub:
sudo modprobe i2c-stub chip_addr=0x48
i2cdetect -l
sudo i2cset -y $BUS 0x48 0x00 0xa5        # fake the ID register

# THE debugging surface:
sudo ls /sys/kernel/debug/regmap/
sudo cat /sys/kernel/debug/regmap/*/registers      # ★ full register dump
sudo cat /sys/kernel/debug/regmap/*/name
sudo cat /sys/kernel/debug/regmap/*/range
echo Y | sudo tee /sys/kernel/debug/regmap/*/cache_only     # stop touching the bus
echo Y | sudo tee /sys/kernel/debug/regmap/*/cache_bypass   # ignore the cache

# Watch the cache work:
sudo bpftrace -e 'tracepoint:regmap:regmap_hw_read_start { @hw_reads = count(); }
                  tracepoint:regmap:regmap_reg_read       { @api_reads = count(); }'
sudo trace-cmd record -e regmap -- sleep 5 && trace-cmd report | head -30
```
**The `api_reads` vs `hw_reads` ratio is the cache hit rate.** On a PMIC it should be very
high.

### Lab 34.7 — `FIELD_PREP`/`FIELD_GET` and their compile-time checks

```c
#define CTRL_MODE   GENMASK(5, 4)
#define CTRL_SPEED  GENMASK(15, 8)

static void fields_demo(void __iomem *base)
{
	u32 v = readl(base + CTRL);

	/* Old style — easy to get wrong */
	v = (v & ~0x30) | ((2 << 4) & 0x30);

	/* Modern — checked */
	v &= ~CTRL_MODE;
	v |= FIELD_PREP(CTRL_MODE, 2);

	/* ★ This FAILS TO COMPILE: 7 does not fit in a 2-bit field */
	/* v |= FIELD_PREP(CTRL_MODE, 7); */

	pr_info("mode=%lu speed=%lu\n",
		FIELD_GET(CTRL_MODE, v), FIELD_GET(CTRL_SPEED, v));
	writel(v, base + CTRL);
}
```
```bash
# Uncomment the bad line and read the error:
make M=$PWD 2>&1 | grep -A5 'FIELD_PREP'
$EDITOR include/linux/bitfield.h     # read __BF_FIELD_CHECK
# Conversion opportunities in the tree:
git grep -n '<< [0-9]*) &' -- drivers/ | head -20
git log --oneline --grep='use FIELD_PREP' | head
```

### Lab 34.8 — `readl_poll_timeout`, the right way to wait

```c
#include <linux/iopoll.h>

/* ★ Sleeping poll — task context */
ret = readl_poll_timeout(base + STATUS, val, val & READY,
			 10 /* us between polls */, 100000 /* us total */);
if (ret)
	return dev_err_probe(dev, ret, "timed out waiting for READY\n");

/* ★ Atomic poll — interrupt context, uses udelay */
ret = readl_poll_timeout_atomic(base + STATUS, val, val & READY, 1, 1000);

/* regmap version */
ret = regmap_read_poll_timeout(regmap, REG_STATUS, val, val & READY, 100, 10000);

/* ★ WRONG, and very common: */
while (!(readl(base + STATUS) & READY))
	;                              /* no timeout: hangs the machine on broken hw */
while (!(readl(base + STATUS) & READY))
	msleep(1);                     /* 10-20x oversleep (Ch. 19 Lab 19.A) */
```
```bash
git grep -n -B1 -A3 'while.*readl.*&' -- drivers/ | head -30   # audit candidates
git log --oneline --grep='readl_poll_timeout' | head
```

---

## 3. Mastery drills

1. **Read `Documentation/driver-api/device-io.rst`** completely. It is the normative document
   and it is short. Then read `Documentation/memory-barriers.txt`'s "Kernel I/O barrier
   effects" section.

2. **Decode the accessors.** For your architecture, expand `writel()` fully through
   `include/asm-generic/io.h` and `arch/*/include/asm/io.h`. Write out the exact instruction
   sequence. Repeat for `writel_relaxed()` and explain the difference.

3. **Find posted-write flushes.** `git grep -n 'flush' -- drivers/net/ethernet/ | head -30`.
   Identify three genuine posted-write flushes. Explain what would break without each.

4. **The `mmiowb` story.** Read the 5.2 removal series (`git log --oneline --grep=mmiowb`).
   Explain the original problem, why drivers got it wrong, and how moving it into
   `spin_unlock()` works. Relate it to Ch. 28's `devm_` argument.

5. **Write-combining audit.** Find every `ioremap_wc()` in the tree. For three, explain why
   merging is safe for that region. Find one you think is questionable.

6. **regmap cost.** Benchmark `regmap_write()` vs `writel()` on an MMIO regmap. Explain the
   difference and state a rule for when each is appropriate.

7. **regcache.** Read `drivers/base/regmap/regcache.c` and `regcache-maple.c`. Explain
   `volatile_reg`, `precious_reg`, `regcache_sync()`, and why `REGCACHE_MAPLE` replaced
   `REGCACHE_RBTREE` (hint: Ch. 10 T.4).

8. **regmap_irq.** Read `drivers/base/regmap/regmap-irq.c`. Explain how it implements an
   `irq_chip` (Ch. 17 §2.2) over a sleeping bus, and why `IRQF_ONESHOT` is mandatory.

9. **Endianness.** Find a driver with `REGMAP_ENDIAN_BIG` or `ioread32be`. Explain the
   hardware situation and how it interacts with Ch. 02 T.3's portability rules.

10. **Design question.** You have a sensor family: one variant on I²C, one on SPI, one
    memory-mapped, all with identical register layouts but different endianness on the SPI
    part, and the MMIO part is in a hot interrupt path. Design the driver structure. Which
    parts use regmap, which use raw MMIO, and how do you share the register definitions?

11. **Audit a driver for ordering bugs.** Pick a NIC or storage driver. Find every
    `*_relaxed()` use and determine whether the surrounding code establishes the required
    ordering by other means. Report anything suspicious.

---

## 4. Further reading

**Kernel documentation:**
- `Documentation/driver-api/device-io.rst` ★★ — **normative; read first**
- `Documentation/driver-api/regmap.rst`
- `Documentation/memory-barriers.txt` — "Kernel I/O barrier effects" ★
- `Documentation/driver-api/bus-virt-phys-mapping.rst` — the address-space vocabulary
- `Documentation/core-api/bus-virt-phys-mapping.rst`

**Source:**
- `include/asm-generic/io.h` ★★, `arch/arm64/include/asm/io.h` ★
- `include/linux/iopoll.h`, `include/linux/bitfield.h`
- `drivers/base/regmap/` ★★ — `regmap.c`, `regcache-maple.c`, `regmap-irq.c`
- Exemplary users: `drivers/mfd/` (almost any PMIC), `drivers/iio/`,
  `drivers/net/ethernet/stmicro/stmmac/` (raw MMIO in a hot path)

**Architecture references:**
- ARM Architecture Reference Manual, §B2 (memory model), §D5 (memory types: Device-nGnRnE
  etc.) — the source of T.2's terminology
- Intel SDM Vol. 3A, §11.3 (memory types, PAT, MTRR) and §8.2 (memory ordering)
- PCI Express Base Specification §2.4 (transaction ordering) — posted vs non-posted

**LWN:**
- "The rise and fall of mmiowb()" ★ — the T.5 story, told properly
- "Relaxed I/O accessors" / "MMIO ordering and the kernel"
- "regmap: a register map abstraction" (Mark Brown's introduction)
- "Write-combining and the kernel"

→ Next: [35-dma-api.md](35-dma-api.md)

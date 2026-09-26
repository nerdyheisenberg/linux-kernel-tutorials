# Chapter 35 — The DMA API: Coherent, Streaming, Scatter-Gather

> **Goal:** let hardware read and write your memory without corruption. Understand the three
> address spaces, the cache-coherency ownership protocol, and why `dma_map_single()` on a
> stack buffer is a bug that works on your laptop and fails in the field.

---

## Theory & First Principles

### T.0 — Start here: the device does not speak your addresses

You have a buffer. Tell the device where it is.

```c
void *buf = kmalloc(4096, GFP_KERNEL);
writel((u32)buf, dev->base + REG_DMA_ADDR);   /* ← wrong in four ways */
```

**Wrong 1: that is a *virtual* address.** The device has no MMU and no page tables. It cannot
translate `0xffff888104a3d000` into anything.

```c
writel(virt_to_phys(buf), dev->base + REG_DMA_ADDR);   /* better. still wrong. */
```

**Wrong 2: the device may not see the same physical addresses you do.** There can be an
IOMMU translating, a bus offset on embedded SoCs, or an address-width limit (a 32-bit device
cannot reach a buffer above 4 GiB, so the kernel must bounce it through a low buffer).

> **There are three address spaces, not two:** virtual (CPU, via the MMU), physical (the CPU's
> view of the bus), and **DMA/bus** (the *device's* view). They coincide often enough on x86
> that people write code that only works there.

**Wrong 3: the CPU's caches.** You wrote into `buf`; on a non-coherent platform those bytes
are sitting in L1 and the device reads stale DRAM. Or the device writes DRAM and the CPU
reads a stale cache line. Somebody must issue cache maintenance, and it must happen at
exactly the right moments.

**Wrong 4 — and this is the one that matters most:** even with a correct address, **who owns
the buffer right now?**

```
   dma_map_single(dev, buf, len, DMA_TO_DEVICE)
        │
        │   ◄───── THE DEVICE OWNS THIS MEMORY ─────►
        │   CPU access here is UNDEFINED. Not "racy". Undefined.
        │   On a non-coherent platform, the cache maintenance has
        │   ALREADY happened -- your write will be silently lost,
        │   or will corrupt a neighbouring cache line.
        │
   dma_unmap_single(...)          <- ownership returns to the CPU
```

**So the DMA API is not an address-translation API. It is an *ownership transfer* API**, and
the address is almost a side effect:

```c
dma_addr_t handle = dma_map_single(dev, buf, len, DMA_TO_DEVICE);
if (dma_mapping_error(dev, handle))        /* it CAN fail: IOMMU, bounce buffers */
	return -ENOMEM;
writel(lower_32_bits(handle), dev->base + REG_DMA_ADDR);
/* ... device runs ... */
dma_unmap_single(dev, handle, len, DMA_TO_DEVICE);
```

That one call does, as needed: translate to a bus address, program the IOMMU, allocate and
copy through a bounce buffer if the device cannot reach the memory, and perform cache
maintenance in the right direction. **On x86 with a coherent device it is nearly a no-op —
which is exactly why x86-only code omits it and then fails on ARM.**

**The honest boundary, and this is worth saying now because it recurs in Part 5:** no type
system, no sanitizer, and no verifier can check this. KASAN sees CPU accesses, not device
accesses. The compiler does not know the device exists. **Correct DMA is a discipline, not a
property that can be enforced** — which is why `CONFIG_DMA_API_DEBUG` plus a strict IOMMU is
the only real detector (Ch. 83 §T.9), and why "works on my board" is the characteristic
failure mode of this chapter (`debugging-scenarios.md` §7).

```bash
cat /sys/kernel/debug/dma-api/error_count     # needs CONFIG_DMA_API_DEBUG
dmesg | grep -i 'swiotlb\|DMAR\|IOMMU'        # bounce buffers / IOMMU in use?
cat /proc/meminfo | grep -i bounce
```

---

### T.1 Three address spaces, not two

Ch. 22 taught virtual vs physical. DMA introduces a third, and conflating any two of them is
the root of most DMA bugs:

```
   CPU virtual address        void *cpu_addr        (what your code holds)
        │ MMU / page tables
        ▼
   CPU physical address       phys_addr_t           (what the memory controller sees)
        │ ???
        ▼
   DMA / bus address          dma_addr_t            (what the DEVICE must be told)
```

That last arrow used to be the identity function on PCs, which is why decades of code got
away with `virt_to_phys()`. It is **not** the identity in general:

| Cause | Effect |
|---|---|
| **IOMMU** | the device's address is an *IOVA*, translated by page tables the kernel owns |
| **Bus offset** | some SoCs put RAM at CPU `0x80000000` but the device sees `0x00000000` |
| **DMA window** | a bridge maps only a subrange of physical memory |
| **Bounce buffering** | the device cannot reach your buffer, so the data is copied elsewhere |
| **Virtualization** | a guest's "physical" address is not the host's |

**Therefore: `virt_to_phys()` is never a valid way to get a DMA address.** The only valid
source is the DMA API. This is not pedantry — it is the difference between a driver that
works on one board and a driver that works everywhere.

```c
/* ★ WRONG — works on x86 without an IOMMU, fails everywhere else */
writel(virt_to_phys(buf), base + DMA_ADDR);

/* ★ RIGHT */
dma_addr_t dma = dma_map_single(dev, buf, len, DMA_TO_DEVICE);
if (dma_mapping_error(dev, dma))
	return -ENOMEM;
writel(lower_32_bits(dma), base + DMA_ADDR_LO);
writel(upper_32_bits(dma), base + DMA_ADDR_HI);
```

`dma_addr_t` is a distinct type for exactly this reason (Ch. 02 T.3's portability rules) —
it may be 32 or 64 bits independently of `phys_addr_t`.

### T.2 The second problem: cache coherency

Even with the right address, the data may be wrong.

```
   CPU writes buf[0] = 42          → lands in the L1 cache, dirty, not in RAM
   Device DMAs from RAM            → reads STALE data

   Device DMAs into RAM            → RAM updated, CPU cache still holds the old line
   CPU reads buf[0]                → reads STALE data from cache

   CPU speculatively prefetches buf → cache line loaded
   Device DMAs into RAM            → CPU later uses the prefetched stale line
```

Whether this happens depends on the **interconnect**, not on software:

| Platform | Coherency |
|---|---|
| x86/x86-64 | **hardware-coherent** — DMA snoops the caches; nothing to do |
| ARM64 servers, many SoCs with `dma-coherent` | coherent |
| Many ARM/ARM64 SoCs, most ARM32, RISC-V, MIPS | **non-coherent** — software must maintain caches |
| Per-device on the same SoC | **varies** — hence the DT `dma-coherent` property |

This is why **x86-only testing hides DMA bugs**, in exactly the way it hides memory-ordering
bugs (Ch. 13 T.10). A driver missing `dma_sync_*` works perfectly on your desktop and
corrupts data on an i.MX board.

The DMA API abstracts this: on coherent platforms the sync operations compile to nothing; on
non-coherent ones they emit cache maintenance (`dc civac` on arm64, etc.). **You write the
same code either way.**

### T.3 Coherent vs streaming: two models for two lifetimes

```c
/* ---- COHERENT (a.k.a. "consistent"): long-lived, simultaneously accessed ---- */
void *cpu = dma_alloc_coherent(dev, size, &dma_handle, GFP_KERNEL);
/* ... both CPU and device may access it at any time, no sync needed ... */
dma_free_coherent(dev, size, cpu, dma_handle);

/* ---- STREAMING: short-lived, ownership transfers explicitly ---- */
dma_addr_t dma = dma_map_single(dev, buf, len, DMA_TO_DEVICE);
/* ... the DEVICE owns the buffer now; CPU must not touch it ... */
dma_unmap_single(dev, dma, len, DMA_TO_DEVICE);
/* ... the CPU owns it again ... */
```

| | Coherent | Streaming |
|---|---|---|
| Lifetime | allocated once, lives for the driver's life | per-transfer |
| Who allocates | the DMA API (may be uncached memory) | you (any kernel memory) |
| Cache maintenance | none needed | **explicit, via the ownership protocol** |
| CPU access cost | possibly **uncached — very slow** on non-coherent platforms | normal |
| Use for | descriptor rings, command queues, status blocks | packet/block data buffers |
| Can be mapped from an existing buffer | no | **yes** — that's the point |

The split exists because the two use cases have opposite requirements. A descriptor ring is
touched by both sides constantly and is small — so make it permanently coherent (on
non-coherent platforms, that means *uncached*, which is slow per-access but avoids constant
cache maintenance). A network packet is touched by one side at a time and is large — so keep
it cached and pay a single cache operation at each handover.

> **Rule of thumb:** *descriptors and doorbell state → coherent. Payload data → streaming.*
> A driver that uses `dma_alloc_coherent()` for packet buffers on an ARM SoC will be
> mysteriously slow, because every CPU access to an uncached page costs a memory round trip.

### T.4 The ownership protocol — the heart of streaming DMA

This is the part to internalize. Streaming DMA is a **formal ownership transfer**, and
violating it is undefined behaviour in the same way a data race is.

```
   ┌──────────────┐  dma_map_*()  /  dma_sync_*_for_device()   ┌──────────────┐
   │  CPU owns    │ ─────────────────────────────────────────▶ │ DEVICE owns  │
   │ (may read &  │                                            │ (CPU must    │
   │  write)      │ ◀───────────────────────────────────────── │  NOT touch)  │
   └──────────────┘  dma_unmap_*() /  dma_sync_*_for_cpu()      └──────────────┘
```

**While the device owns the buffer, the CPU must not read or write it — at all.** Not "should
not"; *must not*. On a non-coherent platform a CPU read pulls a line into the cache, and a
later eviction of that (possibly speculatively-dirtied) line overwrites what the device
wrote. The corruption appears far from the cause.

```c
/* The canonical streaming pattern */
dma_addr_t dma = dma_map_single(dev, buf, len, DMA_FROM_DEVICE);
if (dma_mapping_error(dev, dma))            /* ★ ALWAYS check */
	return -ENOMEM;

start_hardware_dma(dev, dma, len);
wait_for_completion(&done);

dma_unmap_single(dev, dma, len, DMA_FROM_DEVICE);
/* ONLY NOW may you read buf */
process(buf);
```

For a buffer reused many times (a ring of RX buffers), map once and use `sync`:

```c
/* map once at setup */
rx->dma = dma_map_single(dev, rx->buf, RX_SIZE, DMA_FROM_DEVICE);

/* each cycle: */
dma_sync_single_for_cpu(dev, rx->dma, len, DMA_FROM_DEVICE);   /* take ownership */
process(rx->buf);                                              /* CPU reads */
dma_sync_single_for_device(dev, rx->dma, RX_SIZE, DMA_FROM_DEVICE); /* give it back */
post_to_hardware(rx->dma);

/* unmap once at teardown */
dma_unmap_single(dev, rx->dma, RX_SIZE, DMA_FROM_DEVICE);
```

**Direction matters and is not a hint:**

| Direction | Meaning | Cache op on map/sync-for-device | on unmap/sync-for-cpu |
|---|---|---|---|
| `DMA_TO_DEVICE` | CPU wrote, device reads | **clean** (flush dirty lines out) | nothing |
| `DMA_FROM_DEVICE` | device writes, CPU reads | **invalidate** | **invalidate** |
| `DMA_BIDIRECTIONAL` | both | clean **and** invalidate | invalidate |
| `DMA_NONE` | debugging only | — | — |

Getting the direction wrong is a silent data-corruption bug on non-coherent platforms and a
no-op on x86 — the worst possible failure profile.

The similarity to Rust's borrow checker is not accidental: this *is* a move/borrow
discipline, enforced by convention and by `CONFIG_DMA_API_DEBUG` rather than by types. Part 5
shows how Rust's DMA abstractions encode it in the type system.

### T.5 DMA masks: what the device can address

```c
ret = dma_set_mask_and_coherent(dev, DMA_BIT_MASK(64));
if (ret) {
	ret = dma_set_mask_and_coherent(dev, DMA_BIT_MASK(32));
	if (ret)
		return dev_err_probe(dev, ret, "no usable DMA configuration\n");
}
```

The mask declares **how many address bits the device's DMA engine has**. It must be set in
`probe()` **before any DMA allocation or mapping**, because it determines:

1. Which memory `dma_alloc_coherent()` may return (`GFP_DMA32` zone if the mask is 32-bit).
2. Whether a streaming mapping needs **bounce buffering**.

Also relevant:
```c
dma_set_max_seg_size(dev, 65535);          /* max bytes per scatterlist entry */
dma_set_seg_boundary(dev, 0xffff);         /* segments must not cross this boundary */
dma_set_min_align_mask(dev, 4095);         /* preserve low bits through bouncing */
```

**Getting the mask wrong is a top-tier bug.** Declaring 64 bits on a device that only wires
32 address lines produces DMA to a wrapped-around address — silent memory corruption of
whatever happens to live at the low address. Many real drivers have shipped this.

### T.6 SWIOTLB: the bounce buffer of last resort

If a device cannot reach a buffer (mask too small, no IOMMU, memory too high), the kernel
**bounces**: it copies the data through a pre-reserved low-memory pool.

```
   Your buffer at 0x2_4000_0000 (9 GB)     device mask = 32 bits
              │  dma_map_single(DMA_TO_DEVICE)
              │  memcpy → SWIOTLB slot at 0x0_0800_0000
              ▼
   Device DMAs from the bounce buffer
              │  dma_unmap_single(DMA_FROM_DEVICE)
              │  memcpy back
              ▼
   Your buffer has the data
```

Costs, all of them significant:

- **A memcpy per transfer**, in both directions for `DMA_BIDIRECTIONAL`.
- A **fixed reserved pool** (64 MiB by default) that can be exhausted →
  `swiotlb buffer is full` and I/O failures under load.
- Latency and cache pollution.

SWIOTLB is also mandatory infrastructure for **confidential computing** (SEV-SNP, TDX), where
the guest must bounce all DMA through explicitly-shared unencrypted pages — which is why it
got significant attention and a dynamic-growth capability (6.x) after years of neglect.

```bash
dmesg | grep -i swiotlb
cat /sys/kernel/debug/swiotlb/io_tlb_nslabs 2>/dev/null
cat /proc/meminfo | grep -i bounce
# boot params: swiotlb=65536  swiotlb=force  swiotlb=noforce
sudo bpftrace -e 'kprobe:swiotlb_tbl_map_single { @[comm] = count(); }'
```
**If `swiotlb_tbl_map_single` shows up in a profile, something is misconfigured** — usually a
too-small DMA mask or a missing IOMMU.

### T.7 Scatter-gather: physically fragmented buffers

A 1 MiB userspace buffer is 256 physically-discontiguous pages. Most devices can handle a
*list* of segments:

```c
struct scatterlist *sgl;
int nents, i;

sg_alloc_table(&sgt, npages, GFP_KERNEL);
for_each_sg(sgt.sgl, sg, npages, i)
	sg_set_page(sg, pages[i], PAGE_SIZE, 0);

nents = dma_map_sg(dev, sgt.sgl, sgt.nents, DMA_TO_DEVICE);
if (!nents)
	return -ENOMEM;

for_each_sg(sgt.sgl, sg, nents, i) {          /* ★ iterate over the RETURNED count */
	dma_addr_t addr = sg_dma_address(sg);
	unsigned int len = sg_dma_len(sg);
	program_descriptor(dev, i, addr, len);
}
...
dma_unmap_sg(dev, sgt.sgl, sgt.nents, DMA_TO_DEVICE);   /* original nents here */
```

**The subtlety that catches everyone:** `dma_map_sg()` returns a count that may be **smaller**
than what you passed, because an IOMMU can *coalesce* physically-separate pages into one
contiguous IOVA range. So:

- Iterate the mapped list with the **returned** `nents` and use `sg_dma_address()`/
  `sg_dma_len()`.
- Pass the **original** `nents` to `dma_unmap_sg()`.
- Never use `sg->length`/`sg_phys()` after mapping; use the `sg_dma_*` accessors.

This coalescing is a major IOMMU benefit: 256 pages can become **one** descriptor.

The modern `dma_map_sgtable()` wraps this correctly and is preferred:
```c
ret = dma_map_sgtable(dev, &sgt, DMA_TO_DEVICE, 0);
for_each_sgtable_dma_sg(&sgt, sg, i) { ... }
dma_unmap_sgtable(dev, &sgt, DMA_TO_DEVICE, 0);
```

### T.8 The bug catalogue

Every one of these is common, and every one is invisible on a coherent x86 machine.

| Bug | Why it breaks |
|---|---|
| **DMA to/from a stack buffer** | the stack may be in vmalloc space (`VMAP_STACK`); not physically contiguous; and the frame may be reused |
| **DMA to/from `vmalloc()` memory** | not physically contiguous; `virt_to_phys()` is meaningless on it |
| **DMA to a buffer sharing a cache line with other data** | the invalidate on unmap destroys the neighbour's data. This is why `kmalloc` guarantees `ARCH_DMA_MINALIGN` alignment |
| **CPU touching the buffer while the device owns it** | T.4; stale-cache corruption |
| **Wrong direction** | wrong cache operation; silent corruption |
| **Missing `dma_mapping_error()` check** | `DMA_MAPPING_ERROR` programmed into the device as a real address |
| **Missing unmap** | IOVA space leak → eventual `iommu: map failed`; SWIOTLB slot leak |
| **Wrong `dev`** | a parent's mask/IOMMU domain is not the child's |
| **Mask set after allocating** | the allocation used the wrong constraints |
| **DMA from a `devm_kzalloc()` buffer with DMA still running at unbind** | UAF (Ch. 28 T.4) |
| **Using `page_address()` on a highmem page** | NULL on 32-bit |

The cache-line-sharing rule deserves emphasis:

```c
/* ★ BROKEN on non-coherent platforms */
struct mydev {
	int             flag;
	char            dma_buf[64];     /* may share a cache line with `flag` */
};
/* The invalidate on unmap can discard a concurrent write to `flag`. */

/* ★ CORRECT */
struct mydev {
	int             flag;
	char           *dma_buf;         /* separately kmalloc'd: ARCH_DMA_MINALIGN aligned */
};
/* or: */
	char            dma_buf[64] ____cacheline_aligned;
```
`kmalloc()` guarantees `ARCH_DMA_MINALIGN` alignment precisely so that a `kmalloc`'d buffer
is safe to DMA to. Buffers embedded in larger structures are not.

### T.9 The newer, more precise allocators

`dma_alloc_coherent()` is a blunt instrument: it always gives you coherent (often uncached)
memory. Newer APIs let you say what you actually need:

```c
/* Explicitly non-coherent, cached — you do the sync. Best for streaming rings. */
void *cpu = dma_alloc_noncoherent(dev, size, &dma, DMA_FROM_DEVICE, GFP_KERNEL);
dma_sync_single_for_cpu(dev, dma, size, DMA_FROM_DEVICE);
dma_free_noncoherent(dev, size, cpu, dma, DMA_FROM_DEVICE);

/* Pages, not a mapped VA — map them yourself, or give them to userspace */
struct page *pg = dma_alloc_pages(dev, size, &dma, DMA_BIDIRECTIONAL, GFP_KERNEL);

/* Non-contiguous but IOMMU-contiguous — a big buffer with no CMA pressure */
struct sg_table *sgt = dma_alloc_noncontiguous(dev, size, DMA_TO_DEVICE, GFP_KERNEL, 0);
void *vaddr = dma_vmap_noncontiguous(dev, size, sgt);

/* Small, frequent, aligned allocations (descriptors) */
struct dma_pool *pool = dmam_pool_create("desc", dev, 64, 64, 0);
void *d = dma_pool_zalloc(pool, GFP_ATOMIC, &dma);
dma_pool_free(pool, d, dma);
```

`dma_alloc_noncontiguous()` is significant: with an IOMMU, a 16 MiB buffer needs **no
physical contiguity at all** — the IOMMU makes it contiguous for the device. That removes the
CMA/compaction pressure (Ch. 11 T.3, Ch. 23 T.8) that large `dma_alloc_coherent()` calls
create.

**Attributes** refine behaviour further:
```c
DMA_ATTR_WEAK_ORDERING        /* the device tolerates reordered writes */
DMA_ATTR_WRITE_COMBINE        /* map WC (Ch. 34 T.2) */
DMA_ATTR_NO_KERNEL_MAPPING    /* ★ don't create a kernel VA at all — saves vmalloc space */
DMA_ATTR_SKIP_CPU_SYNC        /* caller handles cache maintenance (careful!) */
DMA_ATTR_FORCE_CONTIGUOUS     /* must be physically contiguous even with an IOMMU */
DMA_ATTR_NO_WARN              /* allocation failure is handled */
```

---

## 1. Internals

### 1.1 The dispatch layers

```
   driver
     │ dma_map_single(dev, ...)
     ▼
   include/linux/dma-mapping.h        (inline dispatch)
     │
     ├─ dev->dma_ops set?  ──yes──▶ dev->dma_ops->map_page()   (arch/bus-specific)
     │
     └─ no ──▶ dma_direct_map_page()       kernel/dma/direct.c
                 ├─ is it reachable within the mask?
                 │    no ──▶ swiotlb_map()             kernel/dma/swiotlb.c
                 ├─ is the device non-coherent?
                 │    yes ─▶ arch_sync_dma_for_device()  arch/*/mm/dma-mapping.c
                 └─ return phys_to_dma(dev, phys)      (applies dma_pfn_offset)

   with an IOMMU:  dev->dma_ops = &iommu_dma_ops        drivers/iommu/dma-iommu.c
                   → iommu_dma_map_page() → allocate an IOVA → iommu_map()
```

### 1.2 Source map

```
include/linux/dma-mapping.h   ★★ the API and its inline dispatch
kernel/dma/mapping.c          ★ the generic layer
kernel/dma/direct.c           ★ the no-IOMMU path, masks, phys_to_dma
kernel/dma/swiotlb.c          ★ bounce buffering (T.6)
kernel/dma/coherent.c         per-device coherent pools
kernel/dma/pool.c             atomic-context coherent pools
kernel/dma/contiguous.c       CMA integration
kernel/dma/debug.c            ★★ CONFIG_DMA_API_DEBUG — read what it checks
drivers/iommu/dma-iommu.c     ★ the IOMMU dma_ops (Ch. 36)
arch/arm64/mm/dma-mapping.c   arch_sync_dma_for_{cpu,device}
include/linux/scatterlist.h   ★ sg helpers and iterators
include/linux/dmapool.h
Documentation/core-api/dma-api.rst        ★★ normative
Documentation/core-api/dma-api-howto.rst  ★★ the tutorial — read both
Documentation/core-api/dma-attributes.rst
Documentation/core-api/dma-isa-lpc.rst
```

---

## 2. Practice

### Lab 35.1 — A complete DMA-capable driver

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * dmademo — descriptor ring (coherent) + data buffers (streaming).
 *
 * Lifetime & ownership (Ch. 25 §3, Ch. 35 T.4):
 *   ring        : dma_alloc_coherent, freed at remove. CPU+device access freely.
 *   rx_buf[i]   : kmalloc'd, mapped once, ownership cycles via dma_sync_*.
 *   Teardown    : STOP the engine, wait for in-flight DMA, THEN unmap and free.
 */
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

#include <linux/dma-mapping.h>
#include <linux/interrupt.h>
#include <linux/io.h>
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/slab.h>

#define NR_DESC     64
#define RX_BUF_SIZE 2048

struct hw_desc {          /* shared with the device — fixed-width, little-endian */
	__le64 addr;
	__le32 len;
	__le32 flags;
} __packed;

#define DESC_OWN_DEVICE   BIT(31)
#define DESC_DONE         BIT(30)

struct rx_slot {
	void       *buf;
	dma_addr_t  dma;
};

struct dmademo {
	struct device   *dev;
	void __iomem    *base;

	struct hw_desc  *ring;          /* coherent */
	dma_addr_t       ring_dma;

	struct rx_slot   rx[NR_DESC];   /* streaming */
	unsigned int     head;
	struct dma_pool *cmd_pool;      /* small coherent allocations */
	bool             running;
};

static int dmademo_alloc_rx(struct dmademo *d)
{
	int i;

	for (i = 0; i < NR_DESC; i++) {
		/* ★ kmalloc guarantees ARCH_DMA_MINALIGN alignment (T.8) */
		d->rx[i].buf = kmalloc(RX_BUF_SIZE, GFP_KERNEL);
		if (!d->rx[i].buf)
			return -ENOMEM;

		d->rx[i].dma = dma_map_single(d->dev, d->rx[i].buf,
					      RX_BUF_SIZE, DMA_FROM_DEVICE);
		if (dma_mapping_error(d->dev, d->rx[i].dma))   /* ★ ALWAYS check */
			return -ENOMEM;

		/* Publish the descriptor; the DEVICE now owns this buffer */
		d->ring[i].addr  = cpu_to_le64(d->rx[i].dma);
		d->ring[i].len   = cpu_to_le32(RX_BUF_SIZE);
		d->ring[i].flags = cpu_to_le32(DESC_OWN_DEVICE);
	}
	return 0;
}

static void dmademo_free_rx(struct dmademo *d)
{
	int i;

	for (i = 0; i < NR_DESC; i++) {
		if (d->rx[i].dma && !dma_mapping_error(d->dev, d->rx[i].dma))
			dma_unmap_single(d->dev, d->rx[i].dma,
					 RX_BUF_SIZE, DMA_FROM_DEVICE);
		kfree(d->rx[i].buf);
	}
}

static irqreturn_t dmademo_isr(int irq, void *data)
{
	struct dmademo *d = data;
	u32 st = readl(d->base + IRQ_STATUS);
	unsigned int i;

	if (!st)
		return IRQ_NONE;
	writel(st, d->base + IRQ_ACK);
	readl(d->base + IRQ_STATUS);            /* ★ flush the posted write (Ch. 34 T.4) */

	for (i = d->head; ; i = (i + 1) % NR_DESC) {
		u32 flags = le32_to_cpu(READ_ONCE(d->ring[i].flags));

		if (!(flags & DESC_DONE))
			break;

		/* ★ Take ownership of the data buffer before reading it (T.4) */
		dma_sync_single_for_cpu(d->dev, d->rx[i].dma,
					le32_to_cpu(d->ring[i].len), DMA_FROM_DEVICE);

		process_packet(d->rx[i].buf, le32_to_cpu(d->ring[i].len));

		/* ★ Hand it back before re-arming */
		dma_sync_single_for_device(d->dev, d->rx[i].dma,
					   RX_BUF_SIZE, DMA_FROM_DEVICE);

		d->ring[i].len   = cpu_to_le32(RX_BUF_SIZE);
		/* ★ publish the descriptor last; smp_wmb pairs with the device's read */
		smp_wmb();
		WRITE_ONCE(d->ring[i].flags, cpu_to_le32(DESC_OWN_DEVICE));
	}
	d->head = i;
	writel(i, d->base + RX_TAIL);           /* doorbell: ordered by writel (Ch. 34 T.3) */
	return IRQ_HANDLED;
}

static int dmademo_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	struct dmademo *d;
	int irq, ret;

	d = devm_kzalloc(dev, sizeof(*d), GFP_KERNEL);
	if (!d)
		return -ENOMEM;
	d->dev = dev;
	platform_set_drvdata(pdev, d);

	d->base = devm_platform_ioremap_resource(pdev, 0);
	if (IS_ERR(d->base))
		return PTR_ERR(d->base);

	/* ★ T.5: the mask MUST be set before any allocation or mapping */
	ret = dma_set_mask_and_coherent(dev, DMA_BIT_MASK(64));
	if (ret) {
		ret = dma_set_mask_and_coherent(dev, DMA_BIT_MASK(32));
		if (ret)
			return dev_err_probe(dev, ret, "no usable DMA mask\n");
		dev_info(dev, "falling back to 32-bit DMA\n");
	}

	/* Descriptors: coherent, because both sides touch them constantly (T.3) */
	d->ring = dmam_alloc_coherent(dev, NR_DESC * sizeof(*d->ring),
				      &d->ring_dma, GFP_KERNEL);
	if (!d->ring)
		return -ENOMEM;
	dev_info(dev, "ring cpu=%p dma=%pad\n", d->ring, &d->ring_dma);

	/* Small command blocks: a pool avoids per-allocation overhead */
	d->cmd_pool = dmam_pool_create("dmademo-cmd", dev, 64, 64, 0);
	if (!d->cmd_pool)
		return -ENOMEM;

	ret = dmademo_alloc_rx(d);
	if (ret)
		goto err_free_rx;

	/* Hardware setup BEFORE the IRQ (Ch. 17 §3.1) */
	writel(lower_32_bits(d->ring_dma), d->base + RING_BASE_LO);
	writel(upper_32_bits(d->ring_dma), d->base + RING_BASE_HI);
	writel(NR_DESC, d->base + RING_SIZE);

	irq = platform_get_irq(pdev, 0);
	if (irq < 0) {
		ret = irq;
		goto err_free_rx;
	}
	ret = devm_request_irq(dev, irq, dmademo_isr, 0, dev_name(dev), d);
	if (ret)
		goto err_free_rx;

	writel(CTRL_RUN, d->base + CTRL);
	d->running = true;
	return 0;

err_free_rx:
	dmademo_free_rx(d);
	return ret;
}

static void dmademo_remove(struct platform_device *pdev)
{
	struct dmademo *d = platform_get_drvdata(pdev);

	/* ★ Ch. 25 P12: STOP the engine and WAIT for in-flight DMA before unmapping */
	writel(0, d->base + CTRL);
	readl(d->base + CTRL);                     /* flush */
	while (readl(d->base + STATUS) & STATUS_DMA_ACTIVE)
		cpu_relax();                       /* (bound this in real code) */

	dmademo_free_rx(d);                        /* now safe to unmap */
	/* dmam_* and devm_* handle the rest, in reverse order */
}
```

### Lab 35.2 — Catch every DMA bug automatically

```bash
./scripts/config -e DMA_API_DEBUG -e DMA_API_DEBUG_SG -e DEBUG_VIRTUAL \
                 -e DMA_CMA -e KASAN
make -j$(nproc) && boot

# Then deliberately commit each bug from T.8 and read the report:
dmesg | grep -A10 'DMA-API'
```

Bugs to plant, one at a time:

```c
/* 1. Stack buffer */
static void bad1(struct device *dev)
{
	char buf[64];                                  /* ON THE STACK */
	dma_addr_t d = dma_map_single(dev, buf, 64, DMA_TO_DEVICE);
	/* DMA-API: device driver maps memory from stack */
}

/* 2. vmalloc buffer */
static void bad2(struct device *dev)
{
	void *p = vmalloc(4096);
	dma_map_single(dev, p, 4096, DMA_TO_DEVICE);   /* DMA-API: ... from vmalloc area */
}

/* 3. Missing unmap */
static void bad3(struct device *dev, void *p)
{
	dma_map_single(dev, p, 64, DMA_TO_DEVICE);     /* never unmapped */
	/* DMA-API: device driver has pending DMA allocations while released */
}

/* 4. Wrong direction on unmap */
static void bad4(struct device *dev, void *p)
{
	dma_addr_t d = dma_map_single(dev, p, 64, DMA_TO_DEVICE);
	dma_unmap_single(dev, d, 64, DMA_FROM_DEVICE); /* DMA-API: ... DMA direction mismatch */
}

/* 5. No mapping-error check */
static void bad5(struct device *dev, void *p)
{
	dma_addr_t d = dma_map_single(dev, p, 64, DMA_TO_DEVICE);
	writel(d, base + ADDR);                        /* DMA-API: device driver failed to
							* check map error */
}
```
```bash
# Tune the debug facility:
echo 0 | sudo tee /sys/kernel/debug/dma-api/disabled
cat /sys/kernel/debug/dma-api/error_count
cat /sys/kernel/debug/dma-api/num_free_entries
echo 1 | sudo tee /sys/kernel/debug/dma-api/driver_filter    # restrict to one driver
$EDITOR kernel/dma/debug.c      # read what each check actually verifies
```
**Run every driver you write under `DMA_API_DEBUG` at least once.** It catches things review
does not.

### Lab 35.3 — Measure coherent vs streaming vs uncached

```c
#define N 100000
static int dmabench(struct device *dev)
{
	void *coh, *str;
	dma_addr_t cdma, sdma;
	u64 t0;
	int i;
	volatile char sink = 0;

	coh = dma_alloc_coherent(dev, 4096, &cdma, GFP_KERNEL);
	str = kmalloc(4096, GFP_KERNEL);
	sdma = dma_map_single(dev, str, 4096, DMA_BIDIRECTIONAL);

	/* CPU access cost */
	t0 = ktime_get_ns();
	for (i = 0; i < N; i++) memset(coh, i, 4096);
	pr_info("memset coherent   : %llu ns\n", (ktime_get_ns() - t0) / N);

	t0 = ktime_get_ns();
	for (i = 0; i < N; i++) memset(str, i, 4096);
	pr_info("memset streaming  : %llu ns\n", (ktime_get_ns() - t0) / N);

	/* Sync cost (zero on coherent platforms) */
	t0 = ktime_get_ns();
	for (i = 0; i < N; i++) {
		dma_sync_single_for_device(dev, sdma, 4096, DMA_TO_DEVICE);
		dma_sync_single_for_cpu(dev, sdma, 4096, DMA_FROM_DEVICE);
	}
	pr_info("sync pair         : %llu ns\n", (ktime_get_ns() - t0) / N);

	/* Map/unmap cost */
	t0 = ktime_get_ns();
	for (i = 0; i < N; i++) {
		dma_addr_t d = dma_map_single(dev, str, 4096, DMA_TO_DEVICE);
		dma_unmap_single(dev, d, 4096, DMA_TO_DEVICE);
	}
	pr_info("map+unmap         : %llu ns\n", (ktime_get_ns() - t0) / N);

	dma_unmap_single(dev, sdma, 4096, DMA_BIDIRECTIONAL);
	kfree(str);
	dma_free_coherent(dev, 4096, coh, cdma);
	return (int)sink;
}
```
Run on **x86 (coherent)** and on **arm64 (often non-coherent)**. On x86 the sync cost is ~0;
on a non-coherent arm64 SoC `memset` to coherent memory can be **10–50× slower** than to
normal memory. That number is the entire argument of T.3.

```bash
# Is the device coherent?
cat /sys/bus/platform/devices/*/of_node/dma-coherent 2>/dev/null
grep -rn 'dma-coherent' /proc/device-tree/ 2>/dev/null | head
dmesg | grep -i 'coherent\|dma_ops'
```

### Lab 35.4 — Force and measure SWIOTLB (T.6)

```bash
# Force bouncing for everything
# boot with: swiotlb=force swiotlb=131072

dmesg | grep -i swiotlb
# "software IO TLB: mapped [mem 0x...] (64MB)"

# Run an I/O workload and watch
sudo fio --name=t --filename=/dev/nvme0n1 --rw=randread --bs=128k \
         --iodepth=64 --ioengine=libaio --runtime=30 --time_based
sudo bpftrace -e '
kprobe:swiotlb_tbl_map_single { @bounces = count(); @bytes = sum(arg3); }
interval:s:5 { print(@bounces); print(@bytes); clear(@bounces); clear(@bytes); }'

# Exhaust it:
cat /sys/kernel/debug/swiotlb/io_tlb_used 2>/dev/null
dmesg | grep -i 'swiotlb buffer is full'

# Compare throughput with and without
# boot without swiotlb=force and rerun fio
```

### Lab 35.5 — Scatter-gather and IOMMU coalescing (T.7)

```c
static int sg_demo(struct device *dev)
{
	struct sg_table sgt;
	struct scatterlist *sg;
	struct page **pages;
	int i, n = 64, ret;

	pages = kcalloc(n, sizeof(*pages), GFP_KERNEL);
	for (i = 0; i < n; i++)
		pages[i] = alloc_page(GFP_KERNEL);       /* deliberately scattered */

	ret = sg_alloc_table_from_pages(&sgt, pages, n, 0, n * PAGE_SIZE, GFP_KERNEL);
	pr_info("before map: %u entries\n", sgt.orig_nents);

	ret = dma_map_sgtable(dev, &sgt, DMA_TO_DEVICE, 0);
	pr_info("after  map: %u entries  ← IOMMU coalescing if < orig\n", sgt.nents);

	for_each_sgtable_dma_sg(&sgt, sg, i)
		pr_info("  [%d] dma=%pad len=%u\n", i, &sg_dma_address(sg), sg_dma_len(sg));

	dma_unmap_sgtable(dev, &sgt, DMA_TO_DEVICE, 0);
	sg_free_table(&sgt);
	for (i = 0; i < n; i++) __free_page(pages[i]);
	kfree(pages);
	return 0;
}
```
```bash
# With the IOMMU off:  intel_iommu=off / amd_iommu=off / iommu.passthrough=1
#   → nents == orig_nents (64 entries)
# With the IOMMU on:
#   → nents may be 1 (one contiguous IOVA range)
dmesg | grep -iE 'DMAR|AMD-Vi|iommu'
ls /sys/class/iommu/
```
**That single measurement explains why IOMMUs improve DMA-heavy performance** despite adding
a translation step: 64 descriptors become 1.

### Lab 35.6 — The cache-line-sharing corruption (T.8)

On a **non-coherent** platform (an arm32/arm64 SoC, or QEMU with a non-coherent device):

```c
struct corrupt_demo {
	u32  guard_before;
	char dma_buf[64];          /* ★ shares cache lines with the guards */
	u32  guard_after;
};

static void demo(struct device *dev, struct corrupt_demo *c)
{
	dma_addr_t d;

	c->guard_before = 0xAAAAAAAA;
	c->guard_after  = 0xBBBBBBBB;

	d = dma_map_single(dev, c->dma_buf, 64, DMA_FROM_DEVICE);
	start_device_dma(d, 64);
	/* while the device writes, the CPU dirties an adjacent field: */
	c->guard_before = 0xCCCCCCCC;
	wait_for_dma();
	dma_unmap_single(dev, d, 64, DMA_FROM_DEVICE);   /* invalidate discards the line */

	pr_info("guards: 0x%08x 0x%08x (expect CCCCCCCC BBBBBBBB)\n",
		c->guard_before, c->guard_after);
}
```
```bash
grep -rn 'ARCH_DMA_MINALIGN\|ARCH_KMALLOC_MINALIGN' arch/arm64/include/asm/cache.h \
	include/linux/slab.h
# Why kmalloc'd buffers are safe:
grep -n -B3 -A10 'ARCH_DMA_MINALIGN' Documentation/core-api/dma-api-howto.rst
```

### Lab 35.7 — `dma_pool` for descriptors

```c
struct dma_pool *pool = dmam_pool_create("desc", dev,
					 sizeof(struct hw_desc),  /* size */
					 64,                      /* alignment */
					 0);                      /* boundary (0 = none) */
void *desc = dma_pool_zalloc(pool, GFP_ATOMIC, &dma);   /* ★ works in atomic context */
dma_pool_free(pool, desc, dma);
```
Benchmark against `dma_alloc_coherent()` per descriptor:
```bash
sudo cat /sys/kernel/debug/dmapool/pools 2>/dev/null
# or:
grep -rn 'dma_pool_create' drivers/usb/host/ drivers/scsi/ | head
```
**`dma_alloc_coherent()` is page-granular.** Allocating a 64-byte descriptor with it wastes
4032 bytes and cannot be done in atomic context. That is what `dma_pool` fixes.

---

## 3. Mastery drills

1. **Read both DMA documents.** `Documentation/core-api/dma-api-howto.rst` (tutorial) and
   `dma-api.rst` (reference). They are the normative text and they are excellent.

2. **Trace `dma_map_single()`** from the inline in `include/linux/dma-mapping.h` through
   `dma_direct_map_page()` to `arch_sync_dma_for_device()` on arm64. Identify every decision
   point: mask check, SWIOTLB, coherency, offset.

3. **The three address spaces.** For a device behind an IOMMU, write out a concrete example
   with real numbers for the CPU virtual, CPU physical, and DMA addresses of the same buffer.
   Verify with `/proc/iomem`, `/proc/self/pagemap`, and the driver's `%pad` prints.

4. **Ownership discipline.** Write a formal statement of the streaming-DMA ownership protocol
   (T.4) as pre/post conditions for each API call. Then compare with what
   `CONFIG_DMA_API_DEBUG` actually checks — what can it *not* catch?

5. **Mask bugs.** Find three drivers that call `dma_set_mask_and_coherent()`. For each,
   determine from the hardware documentation whether the mask is right. Then find one that
   sets the mask *after* an allocation (`git grep -B20 dma_set_mask` and look).

6. **SWIOTLB and confidential computing.** Read the SEV/TDX SWIOTLB usage
   (`git log --grep='swiotlb' --grep='SEV\|TDX' --all-match`). Explain why all DMA must
   bounce in a confidential guest and what that costs.

7. **Coalescing.** Read `drivers/iommu/dma-iommu.c`'s `iommu_dma_map_sg()`. Explain exactly
   when segments can be merged and what `dma_get_merge_boundary()` is for.

8. **Non-coherent performance.** On an ARM SoC, profile a driver that uses
   `dma_alloc_coherent()` for bulk data. Convert it to `dma_alloc_noncoherent()` + explicit
   sync and measure. Report the difference. (Several real conversions exist —
   `git log --grep=dma_alloc_noncoherent`.)

9. **Find a real DMA bug.** `git log --oneline --grep='DMA' --grep='stack\|vmalloc\|coherent'
   --all-match -- drivers/ | head -30`. Read five. Classify each against T.8's table.

10. **Design question.** A NIC does 10 Gb/s with 1500-byte packets on a non-coherent ARM64
    SoC with an IOMMU. Design the buffer strategy: coherent or streaming for descriptors and
    for data? Page pool or per-packet kmalloc? Map-once or map-per-packet? What is the
    per-packet DMA API cost budget at line rate, and which choices fit in it?
    (Then read `net/core/page_pool.c` and compare.)

11. **Rust preview.** Read `rust/kernel/dma.rs`. Explain how `CoherentAllocation` and the
    ownership types encode T.4's protocol. What does the borrow checker catch that
    `DMA_API_DEBUG` catches only at runtime? (Part 5, Ch. 83.)

---

## 4. Further reading

**Kernel documentation:**
- `Documentation/core-api/dma-api-howto.rst` ★★ — **read this first, completely**
- `Documentation/core-api/dma-api.rst` ★★ — the normative reference
- `Documentation/core-api/dma-attributes.rst`
- `Documentation/driver-api/dma-buf.rst` (Ch. 36)
- `Documentation/arch/arm64/memory.rst` (coherency and `dma-coherent`)
- `Documentation/devicetree/bindings/dma/` — the DT side

**Source:**
- `include/linux/dma-mapping.h` ★★ — read the whole header
- `kernel/dma/direct.c`, `kernel/dma/swiotlb.c`, `kernel/dma/debug.c` ★
- `include/linux/scatterlist.h`
- `net/core/page_pool.c` — the state of the art in high-rate DMA buffer management
- Exemplary drivers: `drivers/net/ethernet/intel/igb/` (classic ring),
  `drivers/nvme/host/pci.c` (PRP/SGL, `dma_pool`), `drivers/usb/host/xhci-mem.c`

**Papers & background:**
- "Cache coherence" chapters of Hennessy & Patterson, or Sorin/Hill/Wood's
  *Primer on Memory Consistency and Cache Coherence* — the hardware underneath T.2
- PCI Express Base Specification §2 (transaction ordering, posted writes)
- ARM AMBA ACE / CHI specifications — how ARM SoCs achieve (or don't) coherency

**LWN:**
- "A deep dive into the DMA API" / "The DMA API and its discontents"
- "Fixing the DMA API for non-coherent devices"
- "dma_alloc_noncontiguous() and friends"
- "SWIOTLB and confidential computing"
- "Page pool and the network stack"

→ Next: [36-iommu-dmabuf.md](36-iommu-dmabuf.md)

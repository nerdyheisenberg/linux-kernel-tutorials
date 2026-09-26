# Chapter 36 — IOMMU, VFIO, and DMA-BUF

> **Goal:** understand the MMU for devices — why it is a *security boundary* and not just a
> translation layer — and how buffers are shared between subsystems and processes without
> copying.

---

## Theory & First Principles

### T.0 — Start here: a device can read all of your memory

Everything in Ch. 00 §T.2 about protection rests on the MMU: a process cannot *name* memory
it was not given. Now notice what that argument assumes — that all accesses go through the
CPU.

**DMA does not go through the CPU.**

```
  CPU  ───► [ MMU ] ───► DRAM        translated, checked, protected

  Device ─────────────► DRAM        NO TRANSLATION. NO CHECK.
                               A bus-mastering device can read or write
                               ANY physical address, including kernel text,
                               page tables, and every process's secrets.
```

> **Without an IOMMU, every DMA-capable device is equivalent to root.** Not "a security
> risk" — exactly equivalent. A device that can write physical memory can write the page
> tables, and whoever controls the page tables controls everything.

This is not theoretical:

| Attack | How |
|---|---|
| **Thunderbolt/PCIe DMA attacks** (Thunderclap, 2019) | Plug in a malicious device; read the whole of RAM; extract keys |
| **Firewire DMA** (2004–) | The spec *required* unrestricted memory access. `dd` over Firewire read any machine's RAM |
| **A compromised NIC or BMC** | Firmware runs on the device and the device owns memory |
| **A buggy — not malicious — driver** | A wrong DMA address silently corrupts an unrelated page. This is far more common than attacks |

**The IOMMU is an MMU for devices.** Same idea, same shape, applied on the other side:

```
  Device ───► [ IOMMU ] ───► DRAM
            page tables, per DEVICE (or per group)
            an unmapped address FAULTS instead of succeeding
```

That one addition buys four different things, which is why the chapter covers so much ground:

1. **Protection** — a device reaches only what was mapped for it.
2. **Address extension** — a 32-bit device can reach memory above 4 GiB without bouncing.
3. **Contiguity** — scattered physical pages become one contiguous *device* address, so
   Ch. 11 §T.0's fragmentation problem stops being fatal for DMA.
4. **Safe passthrough** — you can hand a device to a VM or to userspace, because the IOMMU
   contains it. VFIO, SPDK, DPDK, and every GPU passthrough setup depend on this (Ch. 104).

**And the cost, which is a real engineering decision and not a footnote:**

| Mode | Behaviour | Risk |
|---|---|---|
| `iommu.strict=1` | Invalidate the IOTLB synchronously on every unmap | none; costs real throughput |
| lazy (often default) | Batch invalidations | **a freed buffer stays device-reachable for a window** |
| `iommu.passthrough=1` | Identity-map: no protection at all | full DMA access; used for performance |

That table is a security/performance trade you will be asked to make (Ch. 102 §T.9), and
"which mode, and why" is a much better answer than "we enabled the IOMMU."

**The second half of the chapter is a different problem with the same shape.** A camera
captures into a buffer; a GPU composites it; a video encoder consumes it; a display scans it
out. Four devices, one buffer, and copying between them would destroy the power budget of
every phone ever made. **`dma-buf` is a refcounted, shareable buffer with explicit
attach/map/fence semantics** — in effect, a file descriptor for a piece of memory that
multiple devices can agree on. It is the narrow waist (Ch. 00 §T.3b) applied to buffers
rather than to calls.

```bash
dmesg | grep -iE 'DMAR|IOMMU|AMD-Vi'
ls /sys/kernel/iommu_groups/                    # the unit of ISOLATION (see T.6)
for g in /sys/kernel/iommu_groups/*/devices/*; do
    echo "group $(basename $(dirname $(dirname $g))): $(basename $g)"
done | sort -V | head
cat /sys/kernel/debug/dma_buf/bufinfo 2>/dev/null | head
```

---

### T.1 Without an IOMMU, every device is root

This is the security fact that motivates everything in this chapter, and it is
under-appreciated:

> **A bus-mastering device can DMA to any physical address.** It does not go through the CPU's
> page tables. It is not subject to user/kernel separation. It can read your encryption keys
> and it can write to kernel text.

So without an IOMMU:

- A malicious or buggy **PCIe card** owns the machine.
- A **Thunderbolt/USB4** device — plugged in by anyone with physical access — owns the
  machine. This is a real, demonstrated attack class: FireWire DMA attacks (2004),
  *Thunderstrike* (2014), *PCILeech* (still works on unprotected systems), *Thunderclap*
  (2019, which showed IOMMU misconfiguration was widespread).
- A **firmware bug** in a NIC corrupts arbitrary memory with no diagnostic.
- **Device passthrough to a VM is impossible** — a guest programming a device's DMA engine
  could read the host's memory.

An IOMMU applies Ch. 22 T.1's move — *insert a level of indirection* — to device accesses:

```
   Device issues address 0x1000  (an IOVA)
          │
          ▼  IOMMU page tables, owned by the KERNEL (or by the VM)
          ▼
   Physical address 0x2_4A17_3000
          │
          ▼ (or: FAULT — the device is denied)
        memory
```

The four benefits, in the order they historically mattered:

1. **Addressing** — a 32-bit device can reach memory above 4 GiB without bouncing (Ch. 35 T.6).
2. **Coalescing** — scattered pages become one contiguous IOVA range (Ch. 35 T.7).
3. **Isolation** — ★ the device can only touch what you mapped, when you mapped it.
4. **Passthrough** — a guest can own a device safely.

**(3) is the one that matters most today**, and it is why IOMMU is enabled by default on
servers and on any machine with external DMA-capable ports.

```bash
dmesg | grep -iE 'DMAR|IOMMU|AMD-Vi|SMMU'
ls /sys/class/iommu/
ls /sys/kernel/iommu_groups/
cat /sys/kernel/iommu_groups/*/type 2>/dev/null | sort | uniq -c
# boot params: intel_iommu=on  amd_iommu=on  iommu=pt  iommu.passthrough=1
#              iommu.strict=1  intel_iommu=sm_on (scalable mode)
```

### T.2 IOMMU groups: the real unit of isolation

This is the concept people find surprising, and it is forced by hardware.

An IOMMU can assign different page tables (**domains**) to different devices — *in
principle*. In practice, several devices may be **indistinguishable to the IOMMU**, so they
must share a domain. The set of devices that cannot be isolated from one another is an
**IOMMU group**, and it is the **minimum granularity of ownership**.

Why devices end up grouped:

| Cause | Explanation |
|---|---|
| **Multi-function devices** | functions of one device may not tag transactions distinctly |
| **PCIe bridges without ACS** | a bridge may route peer-to-peer traffic *without* it reaching the IOMMU |
| **PCIe-to-PCI bridges** | legacy PCI has no requester ID per device |
| **Aliasing** | some devices emit a requester ID that is not their own (broken hardware) |

**ACS (Access Control Services)** is the PCIe capability that forces peer-to-peer traffic
upstream to the IOMMU. Without ACS on every bridge in the path, two devices under that bridge
can talk to each other directly, bypassing any isolation you think you have. Hence the famous
`pcie_acs_override` out-of-tree patch that consumer motherboard users apply to split groups
for GPU passthrough — **and why doing so is genuinely unsafe**, not merely unsupported.

```bash
# See your groups
for g in /sys/kernel/iommu_groups/*/; do
  echo "=== group $(basename $g)"
  for d in $g/devices/*; do
    echo "    $(basename $d)  $(lspci -nns $(basename $d) 2>/dev/null | cut -d' ' -f2-)"
  done
done

# ACS capability on the bridges
sudo lspci -vvv | grep -B12 'Access Control Services' | grep -E '^[0-9a-f]|ACSCtl'
sudo lspci -vvv -s 00:1c.0 | grep -A4 ACS
```

**The consequence for VFIO:** you cannot pass through *one device* from a group — you must
pass through the **whole group**. If your GPU shares a group with the audio function and a
USB controller, the guest gets all three.

### T.3 Domains, and the strict-vs-lazy trade-off

```c
struct iommu_domain {
	unsigned type;        /* IDENTITY | BLOCKED | UNMANAGED | DMA | DMA_FQ */
	const struct iommu_domain_ops *ops;
	unsigned long pgsize_bitmap;
	...
};
```

| Domain type | Meaning |
|---|---|
| `IOMMU_DOMAIN_IDENTITY` | 1:1 passthrough — translation but no protection (`iommu=pt`) |
| `IOMMU_DOMAIN_BLOCKED` | all DMA denied (a device with no driver) |
| `IOMMU_DOMAIN_DMA` | ★ the kernel's DMA API manages it; **strict** invalidation |
| `IOMMU_DOMAIN_DMA_FQ` | same, but **lazy** (flush queue) invalidation |
| `IOMMU_DOMAIN_UNMANAGED` | someone else owns the page tables — VFIO, a GPU driver |

**The strict/lazy choice is a genuine security-vs-performance dial**, and you should be able
to explain it:

- Every `dma_unmap_*()` removes an IOVA mapping. To make that effective, the **IOMMU's TLB**
  must be invalidated — which, like the CPU TLB (Ch. 22 T.3), is not coherent and costs a
  round trip to the IOMMU hardware.
- **Strict** (`iommu.strict=1`): invalidate on every unmap. Correct, and **slow** — it can cost
  30–50% of small-I/O throughput on a fast NVMe device.
- **Lazy** (`DMA_FQ`, the default on many systems): batch invalidations in a flush queue and
  process them periodically. Much faster, **but there is a window in which a page that has
  been freed and reallocated is still reachable by the device.**

That window is a real weakness: a compromised or buggy device can read or write memory that
has already been returned to the allocator and handed to something else. For a
threat model that includes malicious devices (Thunderbolt, passthrough), `iommu.strict=1` is
mandatory. For a trusted server, lazy is usually the right trade.

```bash
cat /sys/kernel/iommu_groups/*/type          # DMA vs DMA-FQ vs identity
cat /sys/module/iommu/parameters/strict 2>/dev/null
# Measure it:
sudo fio --name=t --filename=/dev/nvme0n1 --rw=randread --bs=4k --iodepth=64 \
         --ioengine=libaio --runtime=30 --time_based
# then reboot with iommu.strict=1 and compare IOPS
sudo perf stat -e 'iommu:*' -a -- sleep 5
```

### T.4 The IOMMU API

Most drivers never touch it — the DMA API (Ch. 35) uses it transparently via
`iommu_dma_ops`. You use it directly only when you manage your own address space (GPU
drivers, VFIO, some accelerators):

```c
struct iommu_domain *domain = iommu_paging_domain_alloc(dev);
iommu_attach_device(domain, dev);

iommu_map(domain, iova, paddr, size, IOMMU_READ | IOMMU_WRITE, GFP_KERNEL);
iommu_unmap(domain, iova, size);
phys = iommu_iova_to_phys(domain, iova);

iommu_detach_device(domain, dev);
iommu_domain_free(domain);

/* Fault reporting — essential for debugging */
iommu_set_fault_handler(domain, my_fault_handler, priv);
```

Advanced features worth knowing by name:

- **PASID / SVA (Shared Virtual Addressing)** — a device can use a *process's* page tables
  directly, so `malloc()`ed pointers are valid device addresses with no mapping at all.
  This is what makes GPU/accelerator programming models like CUDA unified memory and SYCL
  work natively (`iommu_sva_bind_device()`).
- **ATS/PRI (Address Translation Services / Page Request Interface)** — the device caches
  translations locally and can *take a page fault*, letting the OS demand-page device memory.
- **Nested translation** — two levels (guest IOVA → guest physical → host physical) for
  passthrough with a guest-managed IOMMU.
- **iommufd** (6.6+) — the modern replacement for VFIO's container/group model, exposing
  IOMMU address spaces as file descriptors with much better semantics for nesting and
  userspace page-table management.

### T.5 VFIO: safe device access from userspace

VFIO lets an **unprivileged** userspace process drive a device directly, safely, because the
IOMMU constrains what that device can reach.

```
   userspace process
      │ ioctl(VFIO_GROUP_GET_DEVICE_FD)
      │ mmap() the device's BARs      ──────▶ direct MMIO access
      │ ioctl(VFIO_IOMMU_MAP_DMA)     ──────▶ IOMMU maps THIS PROCESS's pages only
      │ eventfd                       ──────▶ interrupt delivery
      ▼
   the device can only DMA to what this process mapped
```

Uses, all significant:

- **VM device passthrough** (QEMU/KVM) — the dominant use.
- **DPDK / SPDK** — userspace network and storage stacks that bypass the kernel entirely for
  latency. `vfio-pci` is how they get at the hardware safely.
- **Userspace drivers** for accelerators and FPGAs.

```bash
# Bind a device to vfio-pci (Ch. 27 Lab 27.7)
sudo modprobe vfio-pci
D=0000:03:00.0
echo $D | sudo tee /sys/bus/pci/devices/$D/driver/unbind
echo vfio-pci | sudo tee /sys/bus/pci/devices/$D/driver_override
echo $D | sudo tee /sys/bus/pci/drivers_probe
ls -l /dev/vfio/
readlink /sys/bus/pci/devices/$D/iommu_group     # the group number == the /dev/vfio node

# QEMU passthrough
qemu-system-x86_64 -enable-kvm -m 4G -device vfio-pci,host=$D ...
```

**The `no-IOMMU` mode** (`vfio_iommu_type1.allow_unsafe_interrupts`, `enable_unsafe_noiommu_mode`)
exists for systems without an IOMMU and is exactly as dangerous as it sounds — it is
`CAP_SYS_RAWIO`-gated and taints the kernel. Knowing *why* it taints is the point: without an
IOMMU, userspace driving a bus-mastering device is equivalent to giving it kernel write
access.

### T.6 DMA-BUF: the buffer-sharing problem

Consider a video pipeline: a **camera** captures into a buffer, a **GPU** processes it, a
**display controller** scans it out, and a **video encoder** compresses it. Four devices, four
subsystems (V4L2, DRM, DRM again, V4L2/media), and possibly four different drivers.

Without a shared mechanism, moving data between them means **copying** — at 4K60 that is
~1.5 GB/s of pure waste, and it breaks the latency budget entirely.

**DMA-BUF** (Sumit Semwal, 3.3+) is a kernel-wide buffer-sharing framework where a buffer is
represented as a **file descriptor**:

```
   Exporter (e.g. the GPU driver)          Importer (e.g. the display driver)
      dma_buf_export()                        dma_buf_get(fd)
          │ → returns a struct dma_buf            │
          │ → dma_buf_fd() → an fd  ─────────────▶│ (passed via ioctl or SCM_RIGHTS)
          │                                       │ dma_buf_attach(dmabuf, dev)
          │  ops->map_dma_buf()  ◀────────────────│ dma_buf_map_attachment()
          │  → returns an sg_table valid          │ → sg_table with DMA addresses
          │    for THAT device                    │   for the importer's IOMMU domain
```

Why an **fd** is exactly the right representation (Ch. 29 T.1, Ch. 24 T.5):

- Lifetime is refcounted by the VFS for free.
- It can be passed between processes over a Unix socket (`SCM_RIGHTS`).
- It can be `poll()`ed for readiness.
- Permissions and sandboxing (seccomp, LSM) apply.
- `close()` releases it.

The genuinely clever part is **`map_dma_buf()` being per-attachment**: the same physical
buffer gets a *different* scatter-gather table for each importing device, because each device
may be behind a different IOMMU domain with different IOVAs. The exporter owns the pages; each
importer gets addresses valid for itself.

```c
/* Exporter */
static const struct dma_buf_ops my_ops = {
	.attach          = my_attach,
	.detach          = my_detach,
	.map_dma_buf     = my_map,        /* ★ returns a per-device sg_table */
	.unmap_dma_buf   = my_unmap,
	.release         = my_release,
	.mmap            = my_mmap,       /* CPU access from userspace */
	.vmap            = my_vmap,       /* CPU access from the kernel */
	.begin_cpu_access = my_begin_cpu, /* ★ cache maintenance (Ch. 35 T.4!) */
	.end_cpu_access   = my_end_cpu,
};

DEFINE_DMA_BUF_EXPORT_INFO(exp_info);
exp_info.ops   = &my_ops;
exp_info.size  = size;
exp_info.flags = O_RDWR | O_CLOEXEC;
exp_info.priv  = my_buffer;

dmabuf = dma_buf_export(&exp_info);
fd = dma_buf_fd(dmabuf, O_CLOEXEC);

/* Importer */
dmabuf = dma_buf_get(fd);
attach = dma_buf_attach(dmabuf, my_dev);
sgt    = dma_buf_map_attachment_unlocked(attach, DMA_BIDIRECTIONAL);
/* program the hardware from sg_dma_address(sgt->sgl) ... */
dma_buf_unmap_attachment_unlocked(attach, sgt, DMA_BIDIRECTIONAL);
dma_buf_detach(dmabuf, attach);
dma_buf_put(dmabuf);
```

**`begin_cpu_access`/`end_cpu_access`** exist because the Ch. 35 T.4 ownership protocol still
applies across the sharing boundary — userspace calls
`ioctl(DMA_BUF_IOCTL_SYNC)` before and after CPU access so the exporter can do cache
maintenance.

**DMA heaps** (`/dev/dma_heap/*`) are the userspace allocator for dma-bufs, replacing
Android's ION:

```bash
ls /dev/dma_heap/           # system, system-uncached, cma, ...
cat /sys/kernel/debug/dma_buf/bufinfo 2>/dev/null | head -40
```

### T.7 `dma_fence`: cross-device completion, and its deadlock hazard

Sharing a buffer is only half the problem. The other half: **when is the producer done?**

`dma_fence` is a cross-driver completion primitive — a refcounted object that transitions
once from *unsignalled* to *signalled*, with callbacks and `poll()` support.

```c
struct dma_fence *fence = my_create_fence();
dma_fence_add_callback(fence, &cb, my_callback);
dma_fence_wait(fence, true);
dma_fence_signal(fence);
```

Attached to a buffer via `dma_resv` (the reservation object), which holds:
- one **write** (exclusive) fence — the last writer,
- a set of **read** (shared) fences — concurrent readers.

That gives **implicit synchronization**: an importer that wants to read waits for the write
fence; a writer waits for all read fences. Producers and consumers never coordinate directly.

**The rule that makes this work, and the hazard it creates:**

> **A `dma_fence` MUST signal in bounded time. Always. Unconditionally.**

Because arbitrary other drivers, and userspace, may be blocked on it. A fence that never
signals hangs the display, the compositor, and eventually the session. Therefore:

- You **may not allocate memory** while holding a fence unsignalled in a way that could
  require waiting on that fence (the classic reclaim inversion — Ch. 11 T.7's shape).
- You **may not take a lock** that could be held by something waiting on the fence.
- GPU drivers must implement **timeouts and reset** so a hung job still signals (with an
  error).

This is enforced socially and by `CONFIG_DEBUG_WW_MUTEX_SLOWPATH` + lockdep annotations
(`dma_fence_begin_signalling()`/`dma_fence_end_signalling()`), which teach lockdep the
"signalling critical section" so inversions are caught. Read
`Documentation/driver-api/dma-buf.rst`'s "Indefinite DMA Fences" section — it is one of the
best-argued pieces of design documentation in the kernel, and the argument generalizes to any
cross-subsystem completion primitive.

**Explicit sync** (`sync_file`, a fence exposed as an fd) is the modern alternative: userspace
passes fences explicitly between producer and consumer, which is more composable and is what
Android and modern Wayland compositors use. `drm_syncobj` adds named, reusable fence
containers with timeline semantics.

---

## 1. Internals

### 1.1 Source map

```
drivers/iommu/iommu.c            ★★ the core: groups, domains, attach/detach
drivers/iommu/dma-iommu.c        ★ iommu_dma_ops — how the DMA API uses the IOMMU
drivers/iommu/iova.c             IOVA allocator (a per-CPU rcache + an rbtree)
drivers/iommu/intel/             Intel VT-d
drivers/iommu/amd/               AMD-Vi
drivers/iommu/arm/smmu-v3/       ARM SMMUv3
drivers/iommu/iommufd/           ★ (6.6+) the modern userspace IOMMU interface
drivers/vfio/                    ★ vfio.c, pci/, vfio_iommu_type1.c
drivers/dma-buf/                 ★★ dma-buf.c, dma-fence.c, dma-resv.c, sync_file.c,
                                    heaps/ (system_heap.c, cma_heap.c)
include/linux/iommu.h, dma-buf.h, dma-fence.h, dma-resv.h
Documentation/driver-api/dma-buf.rst   ★★ read the whole thing
Documentation/driver-api/vfio.rst      ★
Documentation/userspace-api/iommufd.rst
Documentation/arch/x86/intel-iommu.rst
```

### 1.2 The IOVA allocator

Mapping is only half the cost; **allocating an IOVA range** is the other half and was
historically the bottleneck. `drivers/iommu/iova.c` uses a per-CPU **magazine cache**
(`iova_rcache`) in front of an rbtree of free ranges — the Ch. 16 T.4 per-CPU pattern applied
to address-space allocation, because a global rbtree lock at 1M IOPS is impossible.

Worth reading: it is a compact, real example of the scalability techniques from Part 1
applied to a non-obvious resource.

---

## 2. Practice

### Lab 36.1 — Map your machine's isolation topology (T.2)

```bash
# Groups, with device names and ACS status
for g in $(ls -1v /sys/kernel/iommu_groups/); do
  echo "=== IOMMU group $g  (type: $(cat /sys/kernel/iommu_groups/$g/type 2>/dev/null))"
  for d in /sys/kernel/iommu_groups/$g/devices/*; do
    b=$(basename $d)
    printf "    %s  %s\n" "$b" "$(lspci -nns $b 2>/dev/null | cut -d' ' -f2- | head -c 70)"
  done
done

# Why are devices grouped? Check ACS on every bridge:
sudo lspci -vvv 2>/dev/null | awk '/^[0-9a-f]{2}:/{dev=$0} /Access Control Services/{print dev}'

# Walk the path from a device to the root, checking ACS at each bridge:
D=0000:03:00.0
P=/sys/bus/pci/devices/$D
while [ -n "$P" ] && [ "$(basename $P)" != "pci0000:00" ]; do
  echo "  $(basename $P)"
  P=$(dirname $(readlink -f $P))
done
```
**Deliverable:** explain, for one multi-device group on your machine, exactly why those
devices cannot be isolated from each other.

### Lab 36.2 — Measure the strict/lazy trade-off (T.3)

```bash
# Baseline: what mode are we in?
cat /sys/kernel/iommu_groups/*/type | sort | uniq -c
dmesg | grep -i 'iommu.*\(strict\|lazy\|flush queue\|passthrough\)'

run_bench() {
  sudo fio --name=t --filename=/dev/nvme0n1 --rw=randread --bs=4k \
           --iodepth=64 --numjobs=4 --ioengine=libaio --direct=1 \
           --runtime=30 --time_based --group_reporting 2>&1 | grep -E 'IOPS|lat.*avg'
}

# Reboot with each and compare:
#   (a) intel_iommu=off                      — no IOMMU at all
#   (b) intel_iommu=on iommu=pt              — identity domain: translation, no protection
#   (c) intel_iommu=on iommu.strict=0        — lazy (default)
#   (d) intel_iommu=on iommu.strict=1        — strict
run_bench

# Where does the time go?
sudo perf record -e 'iommu:*' -a -- sleep 5 && sudo perf report --stdio | head -20
sudo bpftrace -e '
kprobe:iommu_map    { @maps = count(); }
kprobe:iommu_unmap  { @unmaps = count(); }
kprobe:intel_flush_iotlb_all,kprobe:qi_flush_iotlb { @flushes = count(); }
interval:s:5 { print(@maps); print(@unmaps); print(@flushes); clear(@maps); clear(@unmaps); clear(@flushes); }'
```
**Produce a four-row table of IOPS and p99 latency.** Then write two sentences justifying a
choice for (a) a laptop with Thunderbolt, (b) a trusted database server.

### Lab 36.3 — Watch an IOMMU fault

```c
/* Deliberately DMA to an unmapped IOVA */
static void trigger_fault(struct mydev *d)
{
	writel(0xdeadbe00, d->base + DMA_ADDR_LO);   /* never mapped */
	writel(0x0,        d->base + DMA_ADDR_HI);
	writel(4096,       d->base + DMA_LEN);
	writel(DMA_GO,     d->base + DMA_CTRL);
}
```
```bash
dmesg -w &
# Expect something like:
#  DMAR: [DMA Read NO_PASID] Request device [03:00.0] fault addr 0xdeadbe000
#        [fault reason 0x06] PTE Read access is not set
#  or: arm-smmu-v3: event 0x10 received: ... C_BAD_STE / F_TRANSLATION

# Install a handler in your own driver:
iommu_set_fault_handler(domain, my_fault_handler, priv);

# Intel/AMD fault registers:
sudo cat /sys/kernel/debug/iommu/intel/dmar_translation_struct 2>/dev/null | head
sudo cat /sys/kernel/debug/iommu/intel/invalidation_queue 2>/dev/null | head
sudo ls /sys/kernel/debug/iommu/
```
**An IOMMU fault is a *gift*** — without an IOMMU that DMA would have silently corrupted
memory. Reading fault reports fluently is a real skill; the "fault reason" code tells you
exactly what the device did wrong.

### Lab 36.4 — VFIO end to end (T.5)

```bash
# 1. Pick a device you can spare (a second NIC, a spare NVMe)
D=0000:03:00.0
lspci -nns $D
readlink /sys/bus/pci/devices/$D/iommu_group

# 2. Check the whole group is free
G=$(basename $(readlink /sys/bus/pci/devices/$D/iommu_group))
for x in /sys/kernel/iommu_groups/$G/devices/*; do
  echo "$(basename $x) -> $(basename $(readlink $x/driver 2>/dev/null))"
done

# 3. Bind everything in the group to vfio-pci
sudo modprobe vfio-pci
for x in /sys/kernel/iommu_groups/$G/devices/*; do
  b=$(basename $x)
  echo $b | sudo tee /sys/bus/pci/devices/$b/driver/unbind >/dev/null 2>&1
  echo vfio-pci | sudo tee /sys/bus/pci/devices/$b/driver_override >/dev/null
  echo $b | sudo tee /sys/bus/pci/drivers_probe >/dev/null
done
ls -l /dev/vfio/$G

# 4. Drive it from userspace
sudo apt install -y dpdk-dev || true
# Minimal VFIO program:
cat > /tmp/vfio.c <<'EOF'
#include <fcntl.h>
#include <linux/vfio.h>
#include <stdio.h>
#include <string.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <unistd.h>

int main(int argc, char **argv) {
	int container = open("/dev/vfio/vfio", O_RDWR);
	char gpath[64]; snprintf(gpath, sizeof(gpath), "/dev/vfio/%s", argv[1]);
	int group = open(gpath, O_RDWR);
	struct vfio_group_status gs = { .argsz = sizeof(gs) };

	ioctl(group, VFIO_GROUP_GET_STATUS, &gs);
	printf("group viable: %d\n", !!(gs.flags & VFIO_GROUP_FLAGS_VIABLE));

	ioctl(group, VFIO_GROUP_SET_CONTAINER, &container);
	ioctl(container, VFIO_SET_IOMMU, VFIO_TYPE1_IOMMU);

	int dev = ioctl(group, VFIO_GROUP_GET_DEVICE_FD, argv[2]);
	struct vfio_device_info di = { .argsz = sizeof(di) };
	ioctl(dev, VFIO_DEVICE_GET_INFO, &di);
	printf("regions=%u irqs=%u\n", di.num_regions, di.num_irqs);

	struct vfio_region_info ri = { .argsz = sizeof(ri), .index = 0 };
	ioctl(dev, VFIO_DEVICE_GET_REGION_INFO, &ri);
	printf("BAR0 size=0x%llx offset=0x%llx\n",
	       (unsigned long long)ri.size, (unsigned long long)ri.offset);

	void *bar = mmap(NULL, ri.size, PROT_READ|PROT_WRITE, MAP_SHARED, dev, ri.offset);
	printf("BAR0[0] = 0x%08x\n", *(volatile unsigned *)bar);   /* real MMIO, from userspace */

	/* Map some of OUR memory for the device to DMA to */
	void *buf = mmap(NULL, 1<<20, PROT_READ|PROT_WRITE,
			 MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
	struct vfio_iommu_type1_dma_map dm = {
		.argsz = sizeof(dm), .vaddr = (unsigned long)buf,
		.size = 1<<20, .iova = 0x10000000,
		.flags = VFIO_DMA_MAP_FLAG_READ | VFIO_DMA_MAP_FLAG_WRITE,
	};
	printf("dma map: %d\n", ioctl(container, VFIO_IOMMU_MAP_DMA, &dm));
	return 0;
}
EOF
gcc -O2 -o /tmp/vfio /tmp/vfio.c && sudo /tmp/vfio $G $D

# 5. Restore
for x in /sys/kernel/iommu_groups/$G/devices/*; do
  b=$(basename $x)
  echo "" | sudo tee /sys/bus/pci/devices/$b/driver_override >/dev/null
  echo $b | sudo tee /sys/bus/pci/drivers/vfio-pci/unbind >/dev/null 2>&1
  echo $b | sudo tee /sys/bus/pci/drivers_probe >/dev/null
done
```
**You just drove a PCIe device from unprivileged-ish userspace, safely.** The IOMMU is the
only reason that is not a root exploit.

### Lab 36.5 — A DMA-BUF exporter and importer (T.6)

```c
// SPDX-License-Identifier: GPL-2.0
/* A minimal dma-buf exporter, exposed via a misc device ioctl. */
#include <linux/dma-buf.h>
#include <linux/dma-mapping.h>
#include <linux/miscdevice.h>
#include <linux/module.h>
#include <linux/slab.h>

struct mybuf {
	struct device *dev;
	void          *vaddr;
	dma_addr_t     dma;
	size_t         size;
};

static struct sg_table *mybuf_map(struct dma_buf_attachment *att,
				  enum dma_data_direction dir)
{
	struct mybuf *b = att->dmabuf->priv;
	struct sg_table *sgt;
	int ret;

	sgt = kzalloc(sizeof(*sgt), GFP_KERNEL);
	if (!sgt)
		return ERR_PTR(-ENOMEM);

	/* ★ build a table valid for THIS attachment's device (T.6) */
	ret = dma_get_sgtable(b->dev, sgt, b->vaddr, b->dma, b->size);
	if (ret)
		goto err;
	ret = dma_map_sgtable(att->dev, sgt, dir, 0);
	if (ret)
		goto err_free;
	return sgt;

err_free:
	sg_free_table(sgt);
err:
	kfree(sgt);
	return ERR_PTR(ret);
}

static void mybuf_unmap(struct dma_buf_attachment *att, struct sg_table *sgt,
			enum dma_data_direction dir)
{
	dma_unmap_sgtable(att->dev, sgt, dir, 0);
	sg_free_table(sgt);
	kfree(sgt);
}

static void mybuf_release(struct dma_buf *dmabuf)
{
	struct mybuf *b = dmabuf->priv;

	dma_free_coherent(b->dev, b->size, b->vaddr, b->dma);
	kfree(b);
}

static int mybuf_mmap(struct dma_buf *dmabuf, struct vm_area_struct *vma)
{
	struct mybuf *b = dmabuf->priv;

	return dma_mmap_coherent(b->dev, vma, b->vaddr, b->dma, b->size);
}

static int mybuf_begin_cpu(struct dma_buf *dmabuf, enum dma_data_direction dir)
{
	struct mybuf *b = dmabuf->priv;

	dma_sync_single_for_cpu(b->dev, b->dma, b->size, dir);   /* ★ Ch. 35 T.4 */
	return 0;
}
static int mybuf_end_cpu(struct dma_buf *dmabuf, enum dma_data_direction dir)
{
	struct mybuf *b = dmabuf->priv;

	dma_sync_single_for_device(b->dev, b->dma, b->size, dir);
	return 0;
}

static const struct dma_buf_ops mybuf_ops = {
	.map_dma_buf      = mybuf_map,
	.unmap_dma_buf    = mybuf_unmap,
	.release          = mybuf_release,
	.mmap             = mybuf_mmap,
	.begin_cpu_access = mybuf_begin_cpu,
	.end_cpu_access   = mybuf_end_cpu,
};

static int mybuf_alloc_fd(struct device *dev, size_t size)
{
	DEFINE_DMA_BUF_EXPORT_INFO(exp);
	struct mybuf *b;
	struct dma_buf *dmabuf;

	b = kzalloc(sizeof(*b), GFP_KERNEL);
	if (!b)
		return -ENOMEM;
	b->dev = dev;
	b->size = PAGE_ALIGN(size);
	b->vaddr = dma_alloc_coherent(dev, b->size, &b->dma, GFP_KERNEL);
	if (!b->vaddr) {
		kfree(b);
		return -ENOMEM;
	}

	exp.ops   = &mybuf_ops;
	exp.size  = b->size;
	exp.flags = O_RDWR | O_CLOEXEC;
	exp.priv  = b;

	dmabuf = dma_buf_export(&exp);
	if (IS_ERR(dmabuf)) {
		dma_free_coherent(dev, b->size, b->vaddr, b->dma);
		kfree(b);
		return PTR_ERR(dmabuf);
	}
	return dma_buf_fd(dmabuf, O_CLOEXEC);      /* ★ the buffer IS an fd */
}
```
```bash
# Userspace: allocate, mmap, and pass the fd to another process
sudo insmod dmabufdemo.ko
# Then:
ls /proc/self/fd/ -l | grep dmabuf
sudo cat /sys/kernel/debug/dma_buf/bufinfo

# Compare with the standard heaps:
ls /dev/dma_heap/
cat > /tmp/heap.c <<'EOF'
#include <fcntl.h>
#include <linux/dma-heap.h>
#include <stdio.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
int main(void) {
	int h = open("/dev/dma_heap/system", O_RDWR);
	struct dma_heap_allocation_data d = { .len = 1<<20, .fd_flags = O_RDWR|O_CLOEXEC };
	ioctl(h, DMA_HEAP_IOCTL_ALLOC, &d);
	printf("dmabuf fd = %d\n", d.fd);
	void *p = mmap(NULL, 1<<20, PROT_READ|PROT_WRITE, MAP_SHARED, d.fd, 0);
	((char *)p)[0] = 42;
	printf("mapped and written\n");
	return 0;
}
EOF
gcc -O2 -o /tmp/heap /tmp/heap.c && /tmp/heap
sudo cat /sys/kernel/debug/dma_buf/bufinfo
```

### Lab 36.6 — Fences and explicit sync (T.7)

```bash
# Watch fences in a real GPU pipeline
sudo cat /sys/kernel/debug/dri/0/gem_names 2>/dev/null | head
sudo cat /sys/kernel/debug/dri/0/framebuffer 2>/dev/null
sudo ls /sys/kernel/debug/sync/ 2>/dev/null
sudo cat /sys/kernel/debug/sync/info 2>/dev/null | head -40

sudo bpftrace -e '
tracepoint:dma_fence:dma_fence_init     { @init = count(); }
tracepoint:dma_fence:dma_fence_signaled { @signaled = count(); }
tracepoint:dma_fence:dma_fence_wait_start { @s[tid] = nsecs; }
tracepoint:dma_fence:dma_fence_wait_end /@s[tid]/ {
	@wait_us = hist((nsecs - @s[tid])/1000); delete(@s[tid]); }'

# Run something graphical (glxgears, a Wayland client, vkcube) and observe.

# The lockdep annotations that enforce the "must signal" rule:
grep -n 'dma_fence_begin_signalling\|dma_fence_end_signalling' drivers/ -r | head
$EDITOR Documentation/driver-api/dma-buf.rst    # the "Indefinite DMA Fences" section
```

---

## 3. Mastery drills

1. **Read `Documentation/driver-api/dma-buf.rst`** completely, especially "Indefinite DMA
   Fences". Summarize the argument in 300 words. Why can a fence not wait on memory
   allocation? Relate it to Ch. 11 T.7 and Ch. 18 T.5.

2. **Group forensics.** For every IOMMU group on your machine with more than one device,
   determine the specific hardware reason (multifunction, no ACS, bridge type). Use
   `lspci -vvv` and the PCIe spec.

3. **The ACS override question.** Read the out-of-tree `pcie_acs_override` patch and the
   upstream rejection discussion. Write both sides: why users want it, and why merging it
   would be irresponsible.

4. **Lazy-invalidation window.** Write a precise description of the attack: what an adversary
   with a compromised device can do between `dma_unmap` and the deferred flush, and what it
   takes to exploit. Then explain what `iommu.strict=1` costs.

5. **IOVA allocation.** Read `drivers/iommu/iova.c`. Explain the rcache design and map it onto
   Ch. 16's per-CPU patterns. What happens when the cache misses?

6. **SVA.** Read `drivers/iommu/iommu-sva.c` and
   `Documentation/arch/x86/sva.rst`. Explain how a device can use a process's page tables,
   what PASID is, and how device page faults (PRI) are serviced. What new failure modes does
   this introduce?

7. **iommufd.** Read `Documentation/userspace-api/iommufd.rst`. Explain what problems it
   solves that VFIO's container/group model could not, particularly for nested translation.

8. **dma-buf per-attachment mapping.** Explain why `map_dma_buf()` is per-attachment rather
   than per-buffer, with a concrete example of two importers behind different IOMMUs.

9. **Find a fence deadlock fix.** `git log --oneline --grep='dma_fence' --grep='deadlock\|
   lockdep' --all-match | head -20`. Read three. What was the inversion in each?

10. **Design question.** A camera, an NPU, and a display share frames at 60 fps on an ARM SoC
    where the camera is non-coherent, the NPU is behind an SMMU, and the display has no IOMMU
    and needs physically contiguous memory. Design the buffer allocation and synchronization.
    Which heap? Implicit or explicit fences? Where does cache maintenance happen?

11. **Security review.** Your laptop has Thunderbolt. Enumerate every setting that affects DMA
    security (`iommu`, `iommu.strict`, IOMMU_DEFAULT_DMA_STRICT, Thunderbolt security levels,
    kernel lockdown, `pci=nommconf`). Produce a hardened configuration and justify each
    choice.

---

## 4. Further reading

**Kernel documentation:**
- `Documentation/driver-api/dma-buf.rst` ★★
- `Documentation/driver-api/vfio.rst` ★, `vfio-mediated-device.rst`, `vfio-pci-device-specific-driver-acceptance.rst`
- `Documentation/userspace-api/iommufd.rst`
- `Documentation/arch/x86/intel-iommu.rst`, `Documentation/arch/x86/sva.rst`
- `Documentation/admin-guide/kernel-parameters.txt` — every `iommu*`, `intel_iommu`, `amd_iommu`, `vfio*` option

**Specifications:**
- Intel VT-d specification; AMD I/O Virtualization Technology (IOMMU) specification
- ARM System MMU Architecture Specification (SMMUv3)
- PCIe Base Spec: §6.12 (ACS), §6.20 (ATS), §10 (PASID/PRI)

**Source:**
- `drivers/iommu/iommu.c`, `dma-iommu.c`, `iova.c` ★
- `drivers/vfio/vfio_iommu_type1.c`
- `drivers/dma-buf/dma-buf.c`, `dma-fence.c`, `dma-resv.c` ★
- `drivers/dma-buf/heaps/` — the allocator side

**Papers & security research:**
- Markettos et al., **"Thunderclap: Exploring Vulnerabilities in Operating System IOMMU
  Protection via DMA from Untrustworthy Peripherals"** (NDSS 2019) ★ — read this; it is the
  definitive treatment of why T.1 and T.3 matter
- Boileau, "Hit by a Bus: Physical Access Attacks with Firewire" (2006)
- Frisk's PCILeech documentation
- Ben-Yehuda et al., "The Price of Safety: Evaluating IOMMU Performance" (OLS 2007)
- Malka et al., "rIOMMU: Efficient IOMMU for I/O Devices that Employ Ring Buffers"
  (ASPLOS 2015)

**LWN:**
- "IOMMU groups, inside and out" (Alex Williamson) ★ — the best explanation of T.2
- "Safe device assignment with VFIO"
- "The iommufd subsystem"
- "Sharing buffers with dma-buf" / "DMA buffer sharing infrastructure"
- "Indefinite DMA fences" — the design debate itself
- "Strict vs lazy IOMMU invalidation"

→ Next: [37-pci.md](37-pci.md)

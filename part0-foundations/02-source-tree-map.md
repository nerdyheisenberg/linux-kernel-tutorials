# Chapter 02 — The Source Tree: A Guided Map

> **Goal:** given any kernel symptom, filename, or feature name, you can locate the relevant
> code in under 60 seconds without a search engine.

---

## Theory & First Principles

> **How to read this section.** T.0 is for someone who has just cloned the tree and is
> intimidated by it. T.1–T.4 are the theory of decomposition that explains *why* it is shaped
> this way. T.5–T.8 are what you need to navigate and to argue about it.

---

### T.0 — Start here: the tree is not 40 million lines you must read

Clone it and look at the actual distribution. The number that frightens people is the wrong
number.

```bash
cd ~/src/linux
cloc --quiet . 2>/dev/null | tail -20   ||  find . -name '*.[ch]' | xargs wc -l | tail -1

# Where the mass actually is:
for d in drivers arch fs net sound kernel mm include lib crypto block security tools; do
    printf '%-12s %8s files  %10s lines\n' "$d" \
        "$(find $d -name '*.[ch]' 2>/dev/null | wc -l)" \
        "$(find $d -name '*.[ch]' -exec cat {} + 2>/dev/null | wc -l)"
done | sort -k4 -rn
```

The result is always roughly this shape:

```
 drivers/   ~60%   ██████████████████████████████   you read ONE driver at a time
 arch/      ~12%   ██████                         you read ONE architecture
 fs/         ~5%   ███                            you read ONE filesystem
 net/        ~5%   ███
 sound/      ~4%   ██
 kernel/     ~2%   █        <- the actual core: scheduler, locking, time, RCU
 mm/         ~1%   █        <- the memory manager, entire
```

**The three facts that make the tree tractable, and this is the whole point of T.0:**

1. **~75% of the tree is drivers and architectures, and you will never read most of it.**
   Drivers are *leaves*. They depend on the core; the core does not depend on them. You read
   one driver when you need that device.
2. **The part that everyone must understand is small.** `kernel/` + `mm/` + the core of `fs/`
   and `net/` is under a million lines, and the *load-bearing* portion of that is perhaps
   100 k. `kernel/sched/core.c` is ~12 k lines. `mm/page_alloc.c` is ~7 k. These are
   readable artefacts.
3. **The tree has a grammar.** File and directory names are predictable, so you can locate
   things without searching. Verify it right now:

```bash
ls kernel/sched/          # core.c fair.c rt.c deadline.c idle.c ...
ls mm/ | head -20         # page_alloc.c slab.c vmscan.c mmap.c memory.c ...
ls fs/ext4/ | head        # super.c inode.c namei.c file.c extents.c ...
ls drivers/net/ethernet/  # one directory per vendor
```

`super.c` is the mount/superblock code in **every** filesystem. `namei.c` is path resolution
in every filesystem. `core.c` is the main logic in every subsystem. `main.c` in a driver
directory is the probe path. **Learn the grammar and navigation becomes free** — that is
§T.6's subject, and it is worth more than any IDE.

**The practical stance to adopt now:** you are not reading a 40-million-line program. You are
reading a ~100-line function, in a ~3000-line file, in a subsystem with a documented
interface, and the rest of the tree is *reference material you query*, not text you consume.
Ch. 07 turns that stance into a technique.

---

### T.1 Decomposition: Parnas, and why directories are not modules

David Parnas, *"On the Criteria To Be Used in Decomposing Systems into Modules"*
(CACM, 1972) — arguably the most important paper in software architecture — establishes the
criterion that still governs kernel design:

> **Decompose by *secrets*, not by *steps*.** Each module should encapsulate one design
> decision that is likely to change, and hide it behind an interface that does not reveal it.

Parnas contrasted two decompositions of the same program: one by *processing sequence*
(read input → process → format → write) and one by *information hiding* (one module owns the
data structure, one owns the format, …). The second is dramatically more resilient to change,
even though the first looks more "logical".

Linux applies this, and the applications are visible in the tree:

| Secret being hidden | Module | Interface |
|---|---|---|
| "how does *this* filesystem store data?" | `fs/ext4/`, `fs/xfs/` | VFS ops tables |
| "how does *this* CPU do atomics/MMU/traps?" | `arch/*/` | `asm/` headers, `asm-generic/` fallbacks |
| "how does *this* NIC work?" | `drivers/net/ethernet/*` | `net_device_ops` |
| "which reclamation policy?" | `mm/vmscan.c` | `shrinker` interface |
| "which security policy?" | `security/selinux/` | LSM hooks |

**Crucial distinction:** the *directory hierarchy* is a filing convenience; the *dependency
hierarchy* is the real architecture, and the two do not coincide. `drivers/net/ethernet/intel/`
depends on `net/core/`, `lib/`, `mm/`, and `kernel/` — none of which are its ancestors.
When you "learn the tree", you are learning the **dependency graph**, and the only reliable
way to see it is `#include` analysis and symbol references, not `ls`.

```bash
# The dependency graph, empirically:
cd linux
grep -h '^#include <linux/' fs/ext4/*.c | sort | uniq -c | sort -rn | head -20
nm -u fs/ext4/ext4.o 2>/dev/null | head -30     # what ext4 needs from elsewhere
```

### T.2 Layering, and where Linux deliberately breaks it

The idealized layering is:

```
   syscall / UAPI
        │
   subsystem core   (VFS, net core, block, driver model)
        │
   subsystem implementations (ext4, TCP, blk-mq schedulers, bus drivers)
        │
   device drivers
        │
   arch / firmware
```

Strict layering is violated on purpose in several well-known places, and knowing them is a
sign you've actually read the code:

- **The page cache spans VFS, MM, and the block layer.** A `folio` is simultaneously an MM
  object, a filesystem cache entry, and an I/O unit. There is no clean seam, and attempts to
  create one (e.g. the long `struct page` → `folio` conversion) have taken years.
- **Networking bypasses layers for performance**: XDP runs *before* the stack; `sendfile`/
  `splice` connect the page cache directly to sockets; GSO/GRO cross L2–L4 boundaries.
- **`arch/` reaches upward**: `arch/x86/mm/fault.c` calls into generic MM; `arch/*/kvm/`
  is entangled with both MM and the scheduler.
- **The scheduler is consulted by non-scheduler code** (workqueue's CMWQ hooks, Ch. 18).

The lesson, and it generalizes: **performance-critical systems trade layering purity for
vertical integration**, and the discipline is not "never break layers" but "break them
explicitly, in named places, with documented invariants." An architect's job is to keep the
number of such places small and their contracts written down.

### T.3 Portability: the `arch/` boundary (Love, LKD3 Ch. 19)

Linux runs on ~25 architectures. That portability is not accidental — it is a *design
constraint enforced by a specific boundary*, and the variables it abstracts are worth
enumerating because **every one of them is a bug you can write today**:

| Variable | Range in Linux | How you get it wrong |
|---|---|---|
| **Word size** | 32 / 64-bit | storing a pointer in `int`; assuming `sizeof(long) == 8` |
| **Endianness** | LE (x86, arm64 usually), BE (s390, some PowerPC/MIPS) | reading a network/on-disk field without `be32_to_cpu()` |
| **Page size** | 4 K, 16 K, 64 K (arm64 supports all three!) | assuming `PAGE_SIZE == 4096`; hard-coded shifts |
| **Alignment** | strict on some (SPARC, older ARM) | casting a `char *` to `u32 *` at an arbitrary offset |
| **Signedness of `char`** | signed on x86, **unsigned on ARM/PowerPC** | `char c = getchar(); if (c == -1)` |
| **Stack size / growth** | 8–16 K; grows down everywhere in Linux | large locals, recursion |
| **Atomic width** | some arches lack 64-bit atomics on 32-bit | using `atomic64_t` in a hot path on arm32 |
| **Cache coherence** | I/D caches not coherent on many non-x86 | self-modifying code, DMA without flushes |
| **Memory model** | TSO on x86, weak on arm64/ppc/riscv (Ch. 13) | missing barriers; "works on my x86 laptop" |

The portability rules that follow — memorize these, they are review comments you will
receive:

1. **Use the fixed-width or semantic types**: `u8/u16/u32/u64`, `s8…s64`, `size_t`,
   `ptrdiff_t`, `dma_addr_t`, `phys_addr_t`, `resource_size_t`, `pid_t`, `loff_t`,
   `sector_t`, `gfp_t`. Never `int` for a pointer, never `long` for a 64-bit quantity.
2. **Annotate externally-defined data**: `__le32`, `__be16` for on-disk/on-wire fields, and
   convert explicitly (`le32_to_cpu()`, `cpu_to_be16()`). `sparse` checks this
   (`make C=1`) — this is a *type system for endianness*, and it works.
3. **Never assume `PAGE_SIZE`.** Use `PAGE_SIZE`, `PAGE_SHIFT`, `PAGE_MASK`, and
   `PAGE_ALIGN()`. An arm64 kernel with 64 K pages breaks a surprising amount of code.
4. **Be explicit about `char` signedness**: use `s8`/`u8` when the sign matters.
5. **Respect alignment**: use `get_unaligned()/put_unaligned()` (or the `_le32` variants) for
   packed on-wire data. `__packed` structs are *not* free — on strict-alignment arches the
   compiler generates byte-at-a-time access.
6. **Assume nothing about ordering** — write to the Linux memory model, not to x86 (Ch. 13).
7. **Keep architecture-specific code in `arch/`.** If you find yourself writing `#ifdef
   CONFIG_X86` in `drivers/`, you almost certainly need a new `asm-generic` hook instead.

The `asm-generic` mechanism is the elegant part: an architecture provides only what differs,
and `include/asm-generic/` supplies a working default for the rest via
`generic-y += foo.h` in `arch/*/include/asm/Kbuild`. New architectures (RISC-V, LoongArch)
were bootstrapped largely by *not writing* most of `asm/`.

```bash
cat arch/riscv/include/asm/Kbuild        # see how much is inherited
ls include/asm-generic/ | head -40
```

### T.4 Measuring architecture: coupling, cohesion, and churn

You can measure a codebase's structure, and you should — it turns "I feel like this is
messy" into an argument.

- **Fan-in / fan-out** — how many modules call this, and how many does it call? High fan-in +
  low fan-out = a good *core* abstraction (e.g. `kmalloc`). High fan-out + low fan-in = a
  *policy/glue* layer. High both = a god module and a refactoring target.
- **Cohesion** — do the functions in this file operate on the same data? `fs/ext4/inode.c`
  is cohesive; a `misc.c` never is.
- **Churn × complexity** — files that are both frequently changed *and* complex are where
  bugs live (Adam Tornhill's *Your Code as a Crime Scene* formalizes this).

```bash
cd linux
# Churn: which files change most?
git log --since=1.year --name-only --pretty=format: | grep '\.c$' \
  | sort | uniq -c | sort -rn | head -25

# Fan-in for a symbol:
git grep -l 'kmem_cache_create' -- '*.c' | wc -l

# Lines per subsystem — where is the mass?
for d in arch block crypto drivers fs include kernel lib mm net security sound rust; do
  printf "%-10s %8s\n" "$d" "$(git ls-files "$d" | xargs wc -l 2>/dev/null | tail -1 | awk '{print $1}')"
done

# Who owns it? (Conway's Law, made executable)
./scripts/get_maintainer.pl -f mm/page_alloc.c
```

Run the churn command. The top of that list — `drivers/gpu/drm/`, `drivers/net/`,
`fs/btrfs/`, `arch/x86/` — *is* where the kernel's active complexity lives, and it tells you
where review attention and test investment belong.

### T.5 `include/uapi/` — the ABI boundary made physical

In 2012 the kernel performed a mechanical split: every header that defines something
userspace can see moved to `include/uapi/` (and `arch/*/include/uapi/`). This was not
tidying — it made an **invisible, error-prone social contract into a checkable, physical
one**:

- A change under `include/uapi/` is an ABI change and gets ABI-level review.
- `make headers_install` produces exactly the sanitized set userspace needs.
- The `__KERNEL__` `#ifdef` mess that used to guard these headers largely disappeared.

This is a generalizable architectural technique: **when a rule is important and frequently
violated, encode it in the directory structure or the type system so violations become
mechanically detectable.** (Same move as `__user`/`__iomem` sparse annotations, `__percpu`,
and `local_lock`.)

```bash
ls include/uapi/linux/ | head -30
git log --oneline -- include/uapi/linux/io_uring.h | head    # ABI evolution, done carefully
make headers_install INSTALL_HDR_PATH=/tmp/uapi && ls /tmp/uapi/include/linux | head
```

### T.6 Where the mass is, and what that implies for your career

```
drivers/   ~ 60–65 % of lines   — shallow but enormous; hardware-specific
arch/      ~ 8–10 %             — deep, subtle, per-CPU
fs/        ~ 5 %                — deep, correctness-critical
net/       ~ 5 %                — deep, performance-critical
sound/     ~ 3 %
kernel/ mm/ lib/ block/ security/ crypto/  ~ 6 % combined — **the deepest code in the tree**
```

The strategic reading: `drivers/` is where the *jobs* are and where you will start; `kernel/`,
`mm/`, `block/`, and `fs/` are where *architects* are made. The last 6% contains nearly all
the ideas. Budget your study time inversely to line count.

---

### T.7 — What is still argued about

**1. Is `drivers/` at 60% a success or a failure?**
The success reading: driver breadth *is* the market, and the tree's structure is what lets
thousands of vendors contribute in parallel without colliding. The failure reading: much of
it is near-duplicate code that a better abstraction would have eliminated, and the review
burden per subsystem maintainer is now unsustainable (Ch. 87 §T.10). Both are true; the
question is what to do about it, and `regmap`, the driver model, and Rust abstractions are
three different answers.

**2. Should `arch/` be smaller?**
Every generic-ification of arch code (generic entry, generic irq, generic clockevents,
`asm-generic/`) has been a net win — but each took years and broke things en route. The
remaining arch code is genuinely irreducible in some places (entry, atomics, page-table
format) and merely un-refactored in others. Being able to tell the two apart is a real skill.

**3. Does the tree's structure actually reflect the architecture, or just the maintainers?**
Conway's Law (Ch. 00 §T.6) says these are the same question. `drivers/staging/` is a pure
social construct with no architectural meaning; `drivers/base/` is a genuine architectural
layer. `fs/` contains both a real layer (the VFS) and a pile of independent implementations.
**Ask of every directory: is this an architectural boundary or an organizational one?**

**4. Is a single tree still right?**
The alternative — split the kernel into separately-versioned components — is what every
microkernel and every "modular" OS has tried. It requires stable internal interfaces
(Ch. 00 §T.5), which is exactly the thing Linux refuses to provide. The monorepo is not an
accident; it is the enabling condition for tree-wide refactoring.

### T.8 — The compressed model

```
 The tree is ~75% LEAVES (drivers, arch) that you query, and ~25% CORE that
 you must actually understand. The core that matters is ~100k lines.

 It has a GRAMMAR: core.c, main.c, super.c, namei.c, file.c, inode.c,
 *-core.c, Kconfig, Makefile. Learn it and navigation is free.

 DIRECTORIES ARE NOT MODULES. The module boundary is the HEADER in
 include/linux/ plus the ops struct -- that is what Parnas means by an
 interface, and it is what you must not break.

 include/uapi/ IS THE ABI BOUNDARY MADE PHYSICAL. A change there is forever
 (Ch. 24). A change anywhere else is not.

 LAYERING IS DELIBERATELY VIOLATED where performance demands it, and the
 violations are named and documented. An undocumented violation is a bug;
 a documented one is a design decision.
```

Five questions for any unfamiliar directory:

1. **Is this a leaf or a layer?** Does the core depend on it, or only it on the core?
2. **Where is its interface?** Find the header in `include/linux/` and the ops struct.
3. **Who maintains it, and how active is it?** `./scripts/get_maintainer.pl -f <dir>` and
   `git log --oneline --since=1.year <dir> | wc -l`.
4. **What is the canonical example?** Every subsystem has one implementation everyone copies.
5. **Is anything here in `uapi/`?** If so, that part is permanent.

---

## 1. Concept: the tree has a grammar

Linux directory structure encodes an architecture. Learn the grammar and navigation becomes
deterministic.

```
Rule 1: arch/<arch>/        → "this only makes sense on one CPU family"
Rule 2: include/linux/x.h   → "internal API for subsystem x"
Rule 3: include/uapi/       → "frozen forever; userspace sees this"
Rule 4: drivers/<class>/    → grouped by DEVICE CLASS, not by vendor
Rule 5: kernel/<topic>/     → core mechanisms with no hardware
Rule 6: lib/                → generic algorithms; also usable by userspace tests
Rule 7: *_core.c            → the subsystem's engine
Rule 8: *-main.c / core.c   → driver entry point in a multi-file driver
Rule 9: Documentation/      → mirrors the code structure
```

---

## 2. Top-level directories in depth

### `arch/` — architecture support
```
arch/x86/
├── boot/            real-mode stub, decompressor, bzImage assembly
├── entry/           ★ syscall & interrupt entry/exit (entry_64.S, syscall_64.c)
├── include/asm/     arch-specific headers (atomic, barrier, pgtable, io)
├── kernel/          head_64.S, setup.c, traps.c, cpu/, apic/, smpboot.c
├── mm/              page table code, fault.c, init_64.c, TLB
├── kvm/             KVM x86 implementation
├── lib/             optimized memcpy/memset, usercopy
└── configs/         defconfigs
```
```
arch/arm64/
├── boot/dts/        ★ ALL device trees for arm64 boards
├── kernel/          head.S, entry.S, cpufeature.c, smp.c, psci.c
├── mm/              context.c (ASIDs), fault.c, mmu.c
├── include/asm/     sysreg.h (system registers!), assembler.h
└── kvm/             KVM arm64 (incl. pKVM)
```

**Navigation tip:** when a C function is missing, it's in `arch/*/`. E.g. `set_pte_at`,
`__switch_to`, `arch_atomic_add`, `copy_to_user` — all per-arch.

### `kernel/` — core mechanisms (small, deep, high-value)
| Path | Contents |
|---|---|
| `kernel/sched/` | `core.c`, `fair.c` (EEVDF), `rt.c`, `deadline.c`, `ext.c` (sched_ext), `sched.h` |
| `kernel/locking/` | `mutex.c`, `rwsem.c`, `spinlock.c`, `qspinlock.c`, `lockdep.c`, `rtmutex.c` |
| `kernel/rcu/` | `tree.c` (tree RCU), `tasks.h`, `srcutree.c`, `rcutorture.c` |
| `kernel/time/` | `timer.c`, `hrtimer.c`, `clocksource.c`, `tick-sched.c`, `timekeeping.c` |
| `kernel/trace/` | ftrace, tracepoints, kprobes glue, `bpf_trace.c`, ring buffer |
| `kernel/bpf/` | verifier, JIT glue, maps, `core.c` |
| `kernel/cgroup/` | cgroup v1/v2 core, `cpuset.c`, `rstat.c` |
| `kernel/irq/` | `irqdesc.c`, `manage.c`, `chip.c`, `irqdomain.c`, `msi.c` |
| `kernel/power/` | suspend, hibernate, `qos.c` |
| `kernel/` (root) | `fork.c`, `exit.c`, `signal.c`, `sys.c`, `workqueue.c`, `kthread.c`, `module/` |

### `mm/` — memory management
| File | Role |
|---|---|
| `page_alloc.c` | buddy allocator, `__alloc_pages()` |
| `slub.c` | the slab allocator (SLAB and SLOB are gone; SLUB is it) |
| `vmalloc.c` | virtually contiguous allocations |
| `memory.c` | ★ page fault handling, `handle_mm_fault()` |
| `mmap.c` | `mmap`/`munmap`, VMA management (now maple-tree backed) |
| `vmscan.c` | ★ reclaim, LRU, `shrink_folio_list()`, MGLRU |
| `filemap.c` | ★ page cache: `filemap_read()`, `folio` lookup |
| `page-writeback.c` | dirty throttling, writeback thresholds |
| `compaction.c` | anti-fragmentation |
| `huge_memory.c` | transparent huge pages |
| `memcontrol.c` | memory cgroup |
| `rmap.c` | reverse mapping (folio → VMAs) |
| `swap*.c` | swap subsystem, zswap, zram interface |
| `gup.c` | `get_user_pages()` — critical for drivers/DMA |
| `mmu_notifier.c` | for KVM, IOMMU SVA, GPU drivers |

### `fs/` — VFS + filesystems
| Path | Role |
|---|---|
| `fs/namei.c` | ★ path resolution — the hardest file in the kernel; read it 3×|
| `fs/dcache.c` | dentry cache |
| `fs/inode.c` | inode lifecycle |
| `fs/super.c` | superblock, mounting, `fs_context` |
| `fs/namespace.c` | mount namespaces, `mount()`/`open_tree()` |
| `fs/open.c`, `read_write.c` | syscall implementations |
| `fs/buffer.c` | legacy buffer_head layer |
| `fs/iomap/` | ★ modern extent-based I/O (XFS, ext4 DAX, btrfs, gfs2) |
| `fs/ext4/`, `fs/xfs/`, `fs/btrfs/`, `fs/f2fs/` | the big filesystems |
| `fs/overlayfs/`, `fs/fuse/`, `fs/nfs/`, `fs/smb/` | stacking/network |
| `fs/proc/`, `fs/sysfs/`, `fs/kernfs/`, `fs/debugfs/` | pseudo-filesystems |
| `fs/notify/` | inotify, fanotify |
| `fs/crypto/`, `fs/verity/` | fscrypt, fs-verity |

### `block/` — block layer
| File | Role |
|---|---|
| `blk-core.c` | entry: `submit_bio()` |
| `blk-mq.c` | ★ multi-queue block layer — the engine |
| `blk-mq-sched.c`, `mq-deadline.c`, `bfq-iosched.c`, `kyber-iosched.c` | schedulers |
| `bio.c` | bio allocation, splitting, chaining |
| `blk-merge.c` | request merging, segment limits |
| `blk-settings.c` | queue limits (`blk_queue_*`, now `queue_limits`) |
| `genhd.c`, `partitions/` | gendisk, partition parsing |
| `blk-cgroup.c`, `blk-iocost.c`, `blk-iolatency.c` | I/O QoS |
| `blk-zoned.c` | zoned block devices (ZNS, SMR) |
| `bdev.c` | block device inode/file ops |

### `drivers/` — by class, not vendor
```
drivers/
├── base/          ★ THE DRIVER MODEL: core.c, bus.c, driver.c, dd.c, devres.c, platform.c
├── of/            device tree runtime: base.c, address.c, irq.c, platform.c, overlay.c
├── acpi/          ACPI core + ACPICA
├── pci/           PCI/PCIe core: probe.c, msi/, hotplug/, controller/
├── usb/           core/, host/ (xhci), gadget/, storage/, serial/
├── i2c/, spi/     bus cores + controller drivers (busses/) + client drivers
├── gpio/, pinctrl/
├── clk/, reset/, regulator/, power/domain
├── net/           ethernet/, wireless/, phy/, dsa/, virtio_net.c, veth.c
├── nvme/          host/ and target/
├── scsi/          midlayer + HBA drivers + libata under ata/
├── ata/           libata (SATA/PATA)
├── mmc/           core/ + host/ (eMMC, SD)
├── mtd/           raw NAND, SPI-NOR, ubi/
├── md/            ★ MD RAID + device-mapper (dm-*.c)
├── block/         null_blk, loop, brd (ramdisk), virtio_blk, zram
├── gpu/drm/       DRM/KMS — the largest single subsystem
├── media/         V4L2, DVB, camera
├── iio/           industrial I/O sensors
├── hwmon/, thermal/, watchdog/
├── input/         evdev, HID under hid/
├── char/          misc char drivers, tpm/, hw_random/
├── misc/          the junk drawer (but eeprom/, lkdtm/ live here)
├── firmware/      EFI, SCMI, ARM FFA, `firmware_class`
├── remoteproc/, rpmsg/, mailbox/
├── virtio/        virtio core
├── vfio/          userspace device assignment
├── iommu/         IOMMU core + Intel/AMD/ARM SMMU
├── nvmem/, soundwire/, phy/ (generic PHY)
└── staging/       not yet upstream-quality; do not learn from here
```

### `net/` — networking
| Path | Role |
|---|---|
| `net/core/` | `dev.c` (★ netdev core), `skbuff.c`, `sock.c`, `filter.c` (BPF net), `neighbour.c` |
| `net/ipv4/`, `net/ipv6/` | IP, TCP (`tcp_input.c`, `tcp_output.c`), UDP, routing |
| `net/netfilter/` | conntrack, nf_tables, xtables |
| `net/sched/` | qdiscs, TC classifiers/actions |
| `net/packet/`, `net/unix/`, `net/netlink/` | AF_PACKET, AF_UNIX, netlink |
| `net/xdp/` | AF_XDP |
| `net/bridge/`, `net/dsa/`, `net/8021q/` | L2 |
| `net/tls/`, `net/mptcp/`, `net/smc/` | modern transports |

### `include/`
```
include/linux/      internal API (fs.h, sched.h, mm.h, device.h, blkdev.h, skbuff.h, ...)
include/uapi/       ★ STABLE ABI. Every change here is forever.
include/asm-generic/ generic fallbacks arches can opt into
include/net/        networking internal headers
include/drm/, include/media/, include/scsi/, include/sound/  subsystem headers
include/trace/events/  ★ tracepoint definitions — great documentation of data flow
```

### `rust/`
```
rust/
├── kernel/         ★ the `kernel` crate — safe abstractions
│   ├── sync/       Arc, SpinLock, Mutex, CondVar, LockedBy
│   ├── alloc/      KBox, KVec, allocators (Kmalloc, Vmalloc, KVmalloc)
│   ├── drm/, net/, pci.rs, platform.rs, miscdevice.rs, io.rs, dma.rs
│   └── lib.rs
├── bindings/       auto-generated bindgen output (bindings_generated.rs)
├── uapi/           bindgen over include/uapi
├── helpers/        C shims for things bindgen can't express (inline funcs, macros)
├── macros/         proc macros: #[vtable], module!, #[pin_data]
└── pin-init/       the pin-init crate (also on crates.io)
```

### `tools/`
```
tools/perf/                 perf(1)
tools/bpf/bpftool/          bpftool
tools/testing/selftests/    ★ kselftest — real test suites
tools/testing/kunit/        KUnit runner
tools/lib/bpf/              libbpf
tools/objtool/              control-flow validation
tools/power/x86/turbostat/  etc.
```

### `scripts/`
| Tool | Use |
|---|---|
| `checkpatch.pl` | style check — run before every patch |
| `get_maintainer.pl` | who to CC |
| `faddr2line` | `func+0x1f/0x40` → file:line |
| `decode_stacktrace.sh` | symbolize an Oops |
| `bloat-o-meter` | size diff between two vmlinux |
| `coccinelle/` | semantic patches |
| `gdb/linux/` | GDB python helpers (`lx-*` commands) |
| `kernel-doc` | doc extraction |
| `extract-vmlinux` | get vmlinux from bzImage |
| `spelling.txt`, `const_structs.checkpatch` | checkpatch data |

---

## 3. Practice: navigation tooling

### 3.1 cscope + ctags (fast, offline)
```bash
make O=b ARCH=x86 cscope tags -j$(nproc)
# in vim:  :cs find g handle_mm_fault
```
Restrict to your arch — otherwise you get 20 definitions of `atomic_add`.

### 3.2 clangd (best experience)
```bash
make O=b -j$(nproc) compile_commands.json
ln -sf b/compile_commands.json .
```
`.clangd` in tree root:
```yaml
CompileFlags:
  Add: [-Wno-unknown-warning-option, -Wno-unused-function]
  Remove: [-mabi=lp64, -mno-fp-ret-in-387, -fconserve-stack, -mpreferred-stack-boundary=*, -mindirect-branch*, -fno-allow-store-data-races]
Diagnostics:
  UnusedIncludes: None
```

### 3.3 Elixir cross-referencer (web)
https://elixir.bootlin.com/linux/latest/source — the fastest way to browse across versions.
Use the version dropdown to see how an API evolved.

### 3.4 git archaeology — your most powerful tool
```bash
git log --oneline -20 -- fs/ext4/inode.c
git log -S'folio_mark_dirty' --oneline          # when was this string added/removed?
git log -L :ext4_write_begin:fs/ext4/inode.c    # evolution of ONE function
git blame -L 100,150 mm/vmscan.c
git show <sha>                                   # the commit message IS the documentation
git log --grep='EEVDF' --oneline
git describe --contains <sha>                    # which release contains it?
git tag --contains <sha> | head -1
```

> **The commit message is the design document.** Kernel commit messages are famously
> detailed. When you don't understand code, `git blame` it and read the commit.

### 3.5 Finding "who calls this?"
```bash
git grep -n 'blk_mq_alloc_disk' -- '*.c' '*.h'
git grep -nW 'static.*ext4_file_operations'      # -W = show whole function
git grep -n --heading --break 'EXPORT_SYMBOL_GPL(dma_map_sg'
```

`git grep` beats `grep -r` by 10× in this tree. Learn its flags:
`-n` line numbers, `-W` function context, `-p` show enclosing function,
`-l` filenames only, `--and`/`--or`, `-e`.

---

## 4. A repeatable subsystem-reading method

When dropped into an unknown subsystem, do exactly this:

1. **Read `Documentation/<subsystem>/`** — 20 minutes.
2. **Find the central struct.** Usually in `include/linux/<subsys>.h`.
   For block: `struct request_queue`, `struct bio`. For VFS: `struct inode`.
   For drivers: `struct device`.
3. **Find the ops table.** `struct *_operations` / `*_ops`. This enumerates the
   entire contract between core and drivers. **This is the API surface.**
4. **Find registration.** `grep 'register_.*(' ` — how does a client join the subsystem?
5. **Trace 3 paths:**
   - **Init:** `module_init` → `register_*` → probe
   - **Fast path:** the one function called per-operation (`submit_bio`, `netif_receive_skb`)
   - **Teardown:** `unregister_*`, refcount drop, RCU free
6. **Read the tracepoints:** `include/trace/events/<subsys>.h` — a curated list of the
   moments the maintainers consider significant.
7. **Read one simple driver** that uses it. `drivers/block/brd.c`, `drivers/net/veth.c`,
   `fs/ramfs/`, `drivers/i2c/busses/i2c-gpio.c` are excellent references.
8. **Read the most recent 50 commits** to the subsystem. You learn what's actively changing.

---

## 5. Portability practice — prove T.3 to yourself

### Lab 2.A — Break your code on another architecture

```bash
# 1. Find every PAGE_SIZE assumption in a subsystem you care about
git grep -n '4096\|>> 12\|<< 12' -- drivers/ | head -30
# How many of those are genuinely page-size-independent?

# 2. Build a kernel with 64 KiB pages and see what breaks
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- O=b64k defconfig
./scripts/config --file b64k/.config -d ARM64_4K_PAGES -e ARM64_64K_PAGES
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- O=b64k olddefconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- O=b64k -j$(nproc) 2>&1 | grep -i warn | head

# 3. Build big-endian and watch sparse complain about missing conversions
make ARCH=s390 CROSS_COMPILE=s390x-linux-gnu- O=bbe defconfig
make ARCH=s390 CROSS_COMPILE=s390x-linux-gnu- O=bbe C=1 -j$(nproc) 2>&1 | grep -i 'restricted\|endian' | head -20
```

### Lab 2.B — Use sparse as an endianness type checker

```c
/* /tmp/endian_demo.c — put this in a module and run `make C=1 M=/tmp` */
#include <linux/types.h>
#include <asm/byteorder.h>

struct on_disk_hdr {
	__le32 magic;
	__be16 port;
	__u8   version;
} __packed;

static u32 bad(struct on_disk_hdr *h)
{
	return h->magic;              /* sparse: cast to restricted __le32 */
}

static u32 good(struct on_disk_hdr *h)
{
	return le32_to_cpu(h->magic); /* correct */
}
```
```bash
sudo apt install -y sparse
make C=2 M=/tmp modules          # C=2 checks even unchanged files
# expect: "warning: cast from restricted __le32"
```
**This is a real type system.** Once you have seen sparse catch an endianness bug, you will
annotate every on-disk and on-wire structure for the rest of your career.

### Lab 2.C — Alignment faults, demonstrated

```c
/* In a module: */
static void align_demo(void)
{
	u8 buf[16] = { 0 };
	u32 *p = (u32 *)(buf + 1);      /* deliberately misaligned */

	pr_info("unaligned direct : %u\n", *p);              /* fine on x86, faults elsewhere */
	pr_info("unaligned helper : %u\n", get_unaligned((u32 *)(buf + 1)));  /* always fine */
	pr_info("unaligned le32   : %u\n", get_unaligned_le32(buf + 1));
}
```
```bash
# On arm64 you can see the kernel's alignment-fixup counters:
cat /proc/cpu/alignment 2>/dev/null       # arm32
dmesg | grep -i 'alignment\|unaligned'
```

### Lab 2.D — Map the dependency graph of one subsystem

```bash
cd linux
SUB=fs/ext4

# What does it include from elsewhere?
grep -rh '^#include <' $SUB/*.c $SUB/*.h | sort | uniq -c | sort -rn | head -25

# What symbols does it import? (build it first)
make $SUB/ -j$(nproc)
nm -u $SUB/ext4.o | sed 's/^ *U //' | sort -u | head -40

# What does it export to the rest of the kernel?
grep -rn 'EXPORT_SYMBOL' $SUB/ | head

# Who depends on IT?
git grep -l 'ext4_' -- ':!fs/ext4' | head

# Render it (graphviz):
{ echo 'digraph G {'; \
  grep -rh '^#include <linux/\(.*\)\.h>' $SUB/*.c \
   | sed 's|#include <linux/\(.*\)\.h>|"ext4" -> "\1";|' | sort -u; \
  echo '}'; } > /tmp/ext4.dot
dot -Tsvg /tmp/ext4.dot -o /tmp/ext4.svg
```

### Lab 2.E — Churn-and-complexity hotspot analysis (T.4)

```bash
cd linux
# Files changed most in the last year, with their size:
git log --since=1.year --name-only --pretty=format: -- '*.c' \
 | grep -v '^$' | sort | uniq -c | sort -rn | head -30 \
 | while read n f; do [ -f "$f" ] && printf "%5s changes %6s lines  %s\n" "$n" "$(wc -l < "$f")" "$f"; done

# Bug-fix density: which files get the most "Fixes:" tags?
git log --since=1.year --format='%H' | while read c; do
  git show --stat --format='' "$c" 2>/dev/null | head -1
done 2>/dev/null | head   # (slow; better: use git log --grep='Fixes:' --name-only)

git log --since=1.year --grep='^Fixes:' --name-only --pretty=format: -- '*.c' \
 | sort | uniq -c | sort -rn | head -20
```
The second list is a map of where the kernel's bugs actually are. Compare it with the first.
Files that are high on *both* lists are the ones worth reviewing carefully — and the ones
where your careful patch will be most valued.

---

## 6. Mastery drills

1. Without searching the web, find: (a) where `O_DIRECT` is handled for ext4,
   (b) where MSI-X vectors are allocated for PCI, (c) where `TCP_NODELAY` takes effect,
   (d) where the arm64 `Image` header is defined. Time yourself; target < 60 s each.
2. Pick `drivers/block/brd.c`. Apply the 8-step method. Write a one-page summary.
3. `git log -L` the function `handle_mm_fault` across 5 years. Summarize how it changed.
4. Find every `struct file_operations` in `drivers/char/`. Which one is simplest? Read it.
5. Build a personal `bookmarks.md` with the 50 file paths you'll use most.
6. **Parnas exercise.** Pick three subsystems and, for each, state in one sentence the
   *secret* it hides. Then find a place where the secret leaks (a caller that depends on an
   implementation detail) and describe what would break if the implementation changed.
7. **Layering violations.** Find and document three deliberate layering violations in the
   tree beyond the ones in T.2. For each, find the commit or comment that justifies it.
8. **`asm-generic` archaeology.** Compare `arch/riscv/include/asm/Kbuild` with
   `arch/x86/include/asm/Kbuild`. Count how many headers RISC-V inherits. Pick one RISC-V
   *does* override and explain the hardware reason.
9. **Portability bug hunt.** `git log --oneline --grep='endian' -- drivers/ | head -30`.
   Read five. Categorize each by which row of the T.3 table it violated.
10. **Write the portability checklist.** Produce a one-page review checklist from T.3 that
    you will apply to every patch you write or review. Keep it with you.
11. **UAPI discipline.** Find a commit that added a field to a UAPI struct. Explain how
    backward *and* forward compatibility were preserved (hint: look for size/flags fields,
    `__reserved`, and `copy_struct_from_user()`).

---

## 7. Further reading

**Kernel documentation:**
- `Documentation/` index: `make htmldocs && xdg-open Documentation/output/index.html`
- `MAINTAINERS` — read the top comment; it explains the `F:`/`K:`/`N:` syntax
- `Documentation/process/adding-syscalls.rst` — the UAPI discipline in practice
- `Documentation/driver-api/` and `Documentation/core-api/` indexes — skim the tables of
  contents once so you know what exists

**Books/papers on the architectural theory:**
- Parnas, "On the Criteria To Be Used in Decomposing Systems into Modules" (CACM 1972)
- Parnas, "Designing Software for Ease of Extension and Contraction" (1979)
- Love, *Linux Kernel Development* 3rd ed., **Ch. 19 "Portability"** — the concise version
  of T.3, still accurate
- Tornhill, *Your Code as a Crime Scene* — the churn×complexity method in T.4
- Bowman, Holt & Brewster, "Linux as a Case Study: Its Extracted Software Architecture"
  (ICSE 1999) — an actual reverse-engineered architecture of Linux; dated but the *method*
  is what you want
- Brooks, *The Mythical Man-Month*, Ch. 4 "Aristocracy, Democracy and System Design" —
  conceptual integrity, which is what a maintainer is defending

**Tools:**
- Bootlin Elixir: https://elixir.bootlin.com
- LWN subsystem indexes: https://lwn.net/Kernel/Index/
- `scripts/get_maintainer.pl`, `scripts/checkpatch.pl`, `scripts/coccicheck`

→ Next: [03-kbuild-kconfig.md](03-kbuild-kconfig.md)

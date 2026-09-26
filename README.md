# Linux Kernel & OS Mastery — 106 Chapters, Zero to Principal Kernel Architect

> A complete, self-contained apprenticeship covering Linux kernel internals in **C** and **Rust**,
> with **deep dedicated tracks for device drivers (Part 2, 25 chapters)** and
> **storage & filesystems (Part 3, 20 chapters)**, plus the full OS/userspace/embedded stack
> (boot, init, containers, Yocto, BSP bring-up).
>
> Baseline: **Linux 6.6 LTS → 6.12 LTS → mainline 6.1x**. Modern APIs only —
> folios, EEVDF, maple tree, blk-mq, `devm_`/`devres`, faux bus, `pin-init`, Rust 1.78+.

---

## How to use this

Each chapter is structured identically:

| Section | Meaning |
|---|---|
| **Theory & First Principles** | The computer science underneath: the formal problem, the literature, the theorems and pathologies that forced the design, and the alternatives that were rejected. This is what lets you *derive* the API instead of memorizing it. |
| **Concept** | The mental model. Why this exists in Linux specifically, what problem it solves here. |
| **Internals** | Real structs, real call paths, real file:line references into the tree. |
| **Practice** | Code you type, build, boot, and break. |
| **Mastery drills** | Open problems that force you to read the tree yourself. |
| **Further reading** | Documentation/, LWN, original papers, books, talks. |

> **On the theory sections:** they are not optional background. Senior kernel engineers are
> distinguished by the ability to say *"this design is a response to Robson's bound"* or
> *"that's the lookup–acquire race; you need `inc_not_zero` under RCU"* — and to recognize
> a proposed API as a rediscovery of something that already failed in 2007. Each theory
> section cites the original papers so you can go to the source.

**Rule:** *the source tree is the textbook; this curriculum is the map.* Every chapter
cites paths like `drivers/base/core.c` or `rust/kernel/sync/lock/mutex.rs`. Open them.

### Pace (aggressive but real)

```
Weeks  1–4     Part 0   (ch 00–07)   Environment, kbuild, QEMU, first module
Weeks  5–18    Part 1   (ch 08–25)   Kernel C core: memory, locking, RCU, IRQ, sched, MM
Weeks 19–34    Part 2   (ch 26–50)   Device drivers — the big one
Weeks 35–48    Part 3   (ch 51–70)   Storage & filesystems — the other big one
Weeks 49–54    Part 4   (ch 71–76)   Networking
Weeks 55–62    Part 5   (ch 77–85)   Rust for Linux
Weeks 63–66    Part 6   (ch 86–89)   Upstream engineering & architecture
Weeks 67–80    Part 7   (ch 90–101)  OS stack, Yocto, BSP, production
Weeks 81–88    Part 8   (ch 102–105) Security, real-time, virtualization, scale
Ongoing        reference/            Interview & architect readiness
```

---

# Curriculum map — 106 chapters

## Part 0 — Foundations (8 chapters)

| # | Chapter | File |
|---|---|---|
| 00 | Orientation: what the kernel actually is | [part0-foundations/00-orientation.md](part0-foundations/00-orientation.md) |
| 01 | Toolchain, cross-compilation, reproducible builds | [part0-foundations/01-toolchain.md](part0-foundations/01-toolchain.md) |
| 02 | The source tree: a guided map | [part0-foundations/02-source-tree-map.md](part0-foundations/02-source-tree-map.md) |
| 03 | Kbuild & Kconfig deep dive | [part0-foundations/03-kbuild-kconfig.md](part0-foundations/03-kbuild-kconfig.md) |
| 04 | QEMU, initramfs, and a bootable dev loop | [part0-foundations/04-qemu-boot-lab.md](part0-foundations/04-qemu-boot-lab.md) |
| 05 | Your first module; what `insmod` really does | [part0-foundations/05-first-module.md](part0-foundations/05-first-module.md) |
| 06 | Kernel debugging setup: gdb, KGDB, QEMU stubs, `pahole` | [part0-foundations/06-debug-setup.md](part0-foundations/06-debug-setup.md) |
| 07 | Reading kernel code fast: cscope, elixir, `b4`, git archaeology | [part0-foundations/07-reading-code.md](part0-foundations/07-reading-code.md) |

## Part 1 — Kernel C core (18 chapters)

| # | Chapter | File |
|---|---|---|
| 08 | The kernel C dialect | [part1-kernel-c/08-kernel-c-dialect.md](part1-kernel-c/08-kernel-c-dialect.md) |
| 09 | Core data structures I: lists, hlists, rbtree | [part1-kernel-c/09-data-structures-1.md](part1-kernel-c/09-data-structures-1.md) |
| 10 | Core data structures II: XArray, maple tree, IDR, kfifo, bitmaps | [part1-kernel-c/10-data-structures-2.md](part1-kernel-c/10-data-structures-2.md) |
| 11 | Memory allocation APIs: kmalloc, slab, vmalloc, page allocator | [part1-kernel-c/11-memory-apis.md](part1-kernel-c/11-memory-apis.md) |
| 12 | Reference counting, lifetimes, `kref`, `refcount_t` | [part1-kernel-c/12-refcounting.md](part1-kernel-c/12-refcounting.md) |
| 13 | Atomics & the Linux memory model (LKMM) | [part1-kernel-c/13-atomics-memory-model.md](part1-kernel-c/13-atomics-memory-model.md) |
| 14 | Locking: spinlocks, mutexes, rwsem, seqlock, lockdep | [part1-kernel-c/14-locking.md](part1-kernel-c/14-locking.md) |
| 15 | RCU in depth | [part1-kernel-c/15-rcu.md](part1-kernel-c/15-rcu.md) |
| 16 | Per-CPU data, local_lock, and scalability patterns | [part1-kernel-c/16-percpu.md](part1-kernel-c/16-percpu.md) |
| 17 | Interrupts: controllers, IRQ domains, threaded IRQs, MSI | [part1-kernel-c/17-interrupts.md](part1-kernel-c/17-interrupts.md) |
| 18 | Deferred work: softirq, tasklet, workqueue | [part1-kernel-c/18-deferred-work.md](part1-kernel-c/18-deferred-work.md) |
| 19 | Time: clocksource, clockevents, timers, hrtimers, NO_HZ | [part1-kernel-c/19-time-timers.md](part1-kernel-c/19-time-timers.md) |
| 20 | Processes, threads, `task_struct`, fork/exec/exit | [part1-kernel-c/20-processes.md](part1-kernel-c/20-processes.md) |
| 21 | The scheduler: CFS → EEVDF, RT, deadline, load balancing | [part1-kernel-c/21-scheduler.md](part1-kernel-c/21-scheduler.md) |
| 22 | Memory management internals I: page tables, folios, page fault | [part1-kernel-c/22-mm-internals-1.md](part1-kernel-c/22-mm-internals-1.md) |
| 23 | Memory management internals II: reclaim, compaction, THP, memcg | [part1-kernel-c/23-mm-internals-2.md](part1-kernel-c/23-mm-internals-2.md) |
| 24 | System calls, UAPI design, ioctl, netlink, seccomp surface | [part1-kernel-c/24-syscalls-uapi.md](part1-kernel-c/24-syscalls-uapi.md) |
| 25 | Kernel synchronization design patterns (case studies) | [part1-kernel-c/25-sync-patterns.md](part1-kernel-c/25-sync-patterns.md) |

## Part 2 — Device drivers: the deep track (25 chapters)

| # | Chapter | File |
|---|---|---|
| 26 | Driver model foundations: kobject, kset, sysfs, uevent | [part2-drivers/26-driver-model.md](part2-drivers/26-driver-model.md) |
| 27 | Bus / device / driver binding, probe & deferred probe | [part2-drivers/27-bus-probe-binding.md](part2-drivers/27-bus-probe-binding.md) |
| 28 | Resource management: `devres`, `devm_*`, teardown ordering | [part2-drivers/28-devres.md](part2-drivers/28-devres.md) |
| 29 | Character devices from scratch | [part2-drivers/29-char-devices.md](part2-drivers/29-char-devices.md) |
| 30 | misc devices, `faux` bus, and debugfs interfaces | [part2-drivers/30-misc-faux-debugfs.md](part2-drivers/30-misc-faux-debugfs.md) |
| 31 | Platform devices & the platform bus | [part2-drivers/31-platform-devices.md](part2-drivers/31-platform-devices.md) |
| 32 | Device Tree: syntax, bindings, overlays, `fwnode` | [part2-drivers/32-device-tree.md](part2-drivers/32-device-tree.md) |
| 33 | ACPI for driver writers | [part2-drivers/33-acpi.md](part2-drivers/33-acpi.md) |
| 34 | MMIO, `ioremap`, barriers, `regmap` | [part2-drivers/34-mmio-regmap.md](part2-drivers/34-mmio-regmap.md) |
| 35 | The DMA API: coherent, streaming, scatter-gather, `dma_fence` | [part2-drivers/35-dma-api.md](part2-drivers/35-dma-api.md) |
| 36 | IOMMU, DMA-BUF, and device isolation | [part2-drivers/36-iommu-dmabuf.md](part2-drivers/36-iommu-dmabuf.md) |
| 37 | PCI & PCIe: enumeration, BARs, config space, capabilities | [part2-drivers/37-pci.md](part2-drivers/37-pci.md) |
| 38 | PCIe advanced: MSI/MSI-X, SR-IOV, AER, hotplug, P2PDMA | [part2-drivers/38-pcie-advanced.md](part2-drivers/38-pcie-advanced.md) |
| 39 | USB: descriptors, URBs, host/gadget, xHCI | [part2-drivers/39-usb.md](part2-drivers/39-usb.md) |
| 40 | I2C & SMBus drivers | [part2-drivers/40-i2c.md](part2-drivers/40-i2c.md) |
| 41 | SPI, QSPI, and SPI-NOR | [part2-drivers/41-spi.md](part2-drivers/41-spi.md) |
| 42 | GPIO, pinctrl, and the descriptor API | [part2-drivers/42-gpio-pinctrl.md](part2-drivers/42-gpio-pinctrl.md) |
| 43 | Clocks, resets, regulators, power domains | [part2-drivers/43-clk-reset-regulator.md](part2-drivers/43-clk-reset-regulator.md) |
| 44 | IIO, hwmon, and sensor subsystems | [part2-drivers/44-iio-hwmon.md](part2-drivers/44-iio-hwmon.md) |
| 45 | Input subsystem: evdev, HID, touchscreens | [part2-drivers/45-input-hid.md](part2-drivers/45-input-hid.md) |
| 46 | Network device drivers: `netdev`, NAPI, XDP, ethtool | [part2-drivers/46-netdev-drivers.md](part2-drivers/46-netdev-drivers.md) |
| 47 | Media & display: V4L2, DRM/KMS, GPU driver architecture | [part2-drivers/47-drm-v4l2.md](part2-drivers/47-drm-v4l2.md) |
| 48 | Runtime PM, system suspend, and driver power states | [part2-drivers/48-driver-power.md](part2-drivers/48-driver-power.md) |
| 49 | Firmware loading, remoteproc, rpmsg, mailbox | [part2-drivers/49-firmware-remoteproc.md](part2-drivers/49-firmware-remoteproc.md) |
| 50 | Driver testing, fault injection, KUnit, and virtual hardware | [part2-drivers/50-driver-testing.md](part2-drivers/50-driver-testing.md) |

## Part 3 — Storage & filesystems: the deep track (20 chapters)

| # | Chapter | File |
|---|---|---|
| 51 | Storage stack overview: from `write()` to spinning rust | [part3-storage/51-storage-overview.md](part3-storage/51-storage-overview.md) |
| 52 | The page cache, folios, and writeback | [part3-storage/52-page-cache-writeback.md](part3-storage/52-page-cache-writeback.md) |
| 53 | VFS I: superblock, inode, dentry, file, mount | [part3-storage/53-vfs-1.md](part3-storage/53-vfs-1.md) |
| 54 | VFS II: path walking, RCU-walk, dcache, namespaces | [part3-storage/54-vfs-2.md](part3-storage/54-vfs-2.md) |
| 55 | VFS III: address_space ops, iomap, direct I/O | [part3-storage/55-vfs-3-iomap.md](part3-storage/55-vfs-3-iomap.md) |
| 56 | Writing a filesystem from scratch (ramfs → simplefs) | [part3-storage/56-write-a-filesystem.md](part3-storage/56-write-a-filesystem.md) |
| 57 | ext4 internals | [part3-storage/57-ext4.md](part3-storage/57-ext4.md) |
| 58 | XFS internals | [part3-storage/58-xfs.md](part3-storage/58-xfs.md) |
| 59 | Btrfs internals: CoW, B-trees, subvolumes, RAID | [part3-storage/59-btrfs.md](part3-storage/59-btrfs.md) |
| 60 | F2FS, ZFS, and log-structured design | [part3-storage/60-f2fs-zfs.md](part3-storage/60-f2fs-zfs.md) |
| 61 | Journaling & crash consistency (JBD2, fsync, barriers, FUA) | [part3-storage/61-journaling-consistency.md](part3-storage/61-journaling-consistency.md) |
| 62 | Network & stacking filesystems: NFS, SMB, overlayfs, FUSE | [part3-storage/62-network-stacking-fs.md](part3-storage/62-network-stacking-fs.md) |
| 63 | Block layer I: `bio`, request queues, blk-mq architecture | [part3-storage/63-block-layer-1.md](part3-storage/63-block-layer-1.md) |
| 64 | Block layer II: I/O schedulers, plugging, merging, QoS | [part3-storage/64-block-layer-2.md](part3-storage/64-block-layer-2.md) |
| 65 | Writing a block driver (`null_blk`, ramdisk, blk-mq driver) | [part3-storage/65-write-block-driver.md](part3-storage/65-write-block-driver.md) |
| 66 | Device mapper: targets, dm-crypt, dm-thin, dm-raid, LVM | [part3-storage/66-device-mapper.md](part3-storage/66-device-mapper.md) |
| 67 | MD RAID and software RAID internals | [part3-storage/67-md-raid.md](part3-storage/67-md-raid.md) |
| 68 | SCSI & ATA: libata, SAS, the SCSI midlayer, UFS | [part3-storage/68-scsi-ata.md](part3-storage/68-scsi-ata.md) |
| 69 | NVMe: PCIe, NVMe-oF, zoned namespaces, passthrough | [part3-storage/69-nvme.md](part3-storage/69-nvme.md) |
| 70 | MTD, raw NAND, eMMC/SD, UBI/UBIFS, and flash reality | [part3-storage/70-mtd-flash.md](part3-storage/70-mtd-flash.md) |

## Part 4 — Networking (6 chapters)

| # | Chapter | File |
|---|---|---|
| 71 | `sk_buff`, the network device layer, and RX/TX paths | [part4-net/71-skb-netdev.md](part4-net/71-skb-netdev.md) |
| 72 | Protocol stack: IP, TCP, UDP, sockets | [part4-net/72-protocol-stack.md](part4-net/72-protocol-stack.md) |
| 73 | Netfilter, nftables, conntrack, traffic control | [part4-net/73-netfilter-tc.md](part4-net/73-netfilter-tc.md) |
| 74 | XDP, AF_XDP, and high-performance networking | [part4-net/74-xdp.md](part4-net/74-xdp.md) |
| 75 | eBPF: verifier, maps, programs, CO-RE | [part4-net/75-ebpf.md](part4-net/75-ebpf.md) |
| 76 | io_uring: architecture and driver implications | [part4-net/76-io-uring.md](part4-net/76-io-uring.md) |

## Part 5 — Rust for Linux (9 chapters)

| # | Chapter | File |
|---|---|---|
| 77 | Why Rust; build integration; `rustc` in kbuild | [part5-rust/77-rust-foundations.md](part5-rust/77-rust-foundations.md) |
| 78 | Rust language crash course *for kernel engineers* | [part5-rust/78-rust-for-c-devs.md](part5-rust/78-rust-for-c-devs.md) |
| 79 | The `kernel` crate: `Result`, `Error`, `Box`, `Vec`, `Arc`, `ARef` | [part5-rust/79-kernel-crate.md](part5-rust/79-kernel-crate.md) |
| 80 | Pinning, `pin-init`, and self-referential kernel objects | [part5-rust/80-pinning.md](part5-rust/80-pinning.md) |
| 81 | Synchronization in kernel Rust: `SpinLock`, `Mutex`, `Send`/`Sync` | [part5-rust/81-rust-sync.md](part5-rust/81-rust-sync.md) |
| 82 | Writing Rust drivers I: misc, platform, DT/ACPI matching | [part5-rust/82-rust-drivers-1.md](part5-rust/82-rust-drivers-1.md) |
| 83 | Writing Rust drivers II: PCI, MMIO, DMA, IRQ, `devres` | [part5-rust/83-rust-drivers-2.md](part5-rust/83-rust-drivers-2.md) |
| 84 | Bindings, `unsafe`, and writing safety contracts | [part5-rust/84-bindings-unsafe.md](part5-rust/84-bindings-unsafe.md) |
| 85 | Case studies: Binder, Nova, `rnull`, Android, upstream status | [part5-rust/85-case-studies.md](part5-rust/85-case-studies.md) |

## Part 6 — Upstream engineering & architecture (4 chapters)

| # | Chapter | File |
|---|---|---|
| 86 | Patch workflow: git, `b4`, mailing lists, review etiquette | [part6-engineering/86-patch-workflow.md](part6-engineering/86-patch-workflow.md) |
| 87 | Reviewing, maintaining, and subsystem stewardship | [part6-engineering/87-maintainership.md](part6-engineering/87-maintainership.md) |
| 88 | Stable/LTS, backporting, distro & product kernels | [part6-engineering/88-stable-backporting.md](part6-engineering/88-stable-backporting.md) |
| 89 | The architect's playbook: API design & technical judgement | [part6-engineering/89-architect-playbook.md](part6-engineering/89-architect-playbook.md) |

## Part 7 — The OS around the kernel (12 chapters)

| # | Chapter | File |
|---|---|---|
| 90 | Boot flow: firmware, UEFI, U-Boot, ATF, secure boot | [part7-os/90-boot-flow.md](part7-os/90-boot-flow.md) |
| 91 | initramfs, init systems, systemd internals | [part7-os/91-init-systemd.md](part7-os/91-init-systemd.md) |
| 92 | ELF, libc, dynamic linking, and the userspace ABI | [part7-os/92-elf-libc.md](part7-os/92-elf-libc.md) |
| 93 | Namespaces, cgroups v2, containers from scratch | [part7-os/93-containers.md](part7-os/93-containers.md) |
| 94 | Storage administration in production | [part7-os/94-storage-admin.md](part7-os/94-storage-admin.md) |
| 95 | Observability: perf, ftrace, bpftrace, crash dumps | [part7-os/95-observability.md](part7-os/95-observability.md) |
| 96 | Yocto I — architecture & concepts | [part7-os/96-yocto-1-concepts.md](part7-os/96-yocto-1-concepts.md) |
| 97 | Yocto II — BitBake language, recipes, layers, classes | [part7-os/97-yocto-2-recipes.md](part7-os/97-yocto-2-recipes.md) |
| 98 | Yocto III — BSPs, kernel recipes, images, distro config | [part7-os/98-yocto-3-bsp-kernel.md](part7-os/98-yocto-3-bsp-kernel.md) |
| 99 | Yocto IV — production: SDK, CVE, SBOM, OTA, reproducibility | [part7-os/99-yocto-4-production.md](part7-os/99-yocto-4-production.md) |
| 100 | Buildroot, OpenWrt, and alternatives | [part7-os/100-buildroot-alternatives.md](part7-os/100-buildroot-alternatives.md) |
| 101 | Board bring-up end-to-end: a full BSP from bare metal | [part7-os/101-board-bringup.md](part7-os/101-board-bringup.md) |

## Part 8 — Specialized domains (4 chapters)

> These four subjects are *cross-cutting*: each one is referenced from a dozen earlier
> chapters but owned by none of them. They are also, empirically, the four topics that
> senior and staff-level kernel interviews probe hardest — because each one is where
> a plausible-sounding design is most likely to be catastrophically wrong.

| # | Chapter | File |
|---|---|---|
| 102 | Kernel security architecture: LSM, seccomp, capabilities, hardening, CPU mitigations | [part8-specialized/102-kernel-security.md](part8-specialized/102-kernel-security.md) |
| 103 | Real-time Linux: PREEMPT_RT, priority inversion, latency engineering | [part8-specialized/103-realtime.md](part8-specialized/103-realtime.md) |
| 104 | Virtualization & KVM internals: VMX/EPT, virtio, vhost, VFIO, confidential computing | [part8-specialized/104-virtualization.md](part8-specialized/104-virtualization.md) |
| 105 | NUMA, scalability, and performance engineering as a discipline | [part8-specialized/105-numa-scalability.md](part8-specialized/105-numa-scalability.md) |

## Labs & reference

| Item | File |
|---|---|
| Lab environment, conventions, and index (~790 labs) | [labs/README.md](labs/README.md) |
| Capstone projects (10 multi-week projects) | [labs/capstones.md](labs/capstones.md) |
| Cheat sheets | [reference/cheatsheets.md](reference/cheatsheets.md) |
| Reading list | [reference/reading-list.md](reference/reading-list.md) |
| Glossary | [reference/glossary.md](reference/glossary.md) |

### Interview & architect readiness

> The curriculum teaches you the material. This track teaches you to **perform it under
> time pressure, out loud, to a skeptical stranger** — a genuinely different skill.

| Item | File | What it is for |
|---|---|---|
| **Interview playbook** | [reference/interview-playbook.md](reference/interview-playbook.md) | How senior kernel/OS loops are actually structured, what signal each round probes, the rubric interviewers score against, red flags, and the architect-track behavioral questions |
| **OS fundamentals rapid reference** | [reference/os-fundamentals.md](reference/os-fundamentals.md) | The classic CS-degree operating-systems material this curriculum *assumes* — Coffman conditions, Belady's anomaly, banker's algorithm, MLFQ, working-set model, the classic concurrency problems. Interviews ask these verbatim as warm-ups |
| **Question bank** | [reference/question-bank.md](reference/question-bank.md) | Several hundred questions by topic and by level (senior → staff → principal), each with a model answer, the follow-up the interviewer will ask, and a link to the chapter that derives it |
| **System design track** | [reference/system-design.md](reference/system-design.md) | Architect-level design problems worked end-to-end: design a storage stack, design a UAPI, kernel-or-userspace, design for 10M IOPS, design an RT partition, design a secure BSP |
| **Debugging scenarios** | [reference/debugging-scenarios.md](reference/debugging-scenarios.md) | "Here is an oops / a latency spike / a corruption report — diagnose it live." Worked walkthroughs in the format live debugging rounds use |

### Mapping to the classic textbooks

If you are cross-checking against a standard text, this curriculum is a strict superset:

| Book | Coverage |
|---|---|
| Love, *Linux Kernel Development* 3e (20 ch) | **All 20 covered.** Love's ch 19 *Portability* is distributed across Ch 02 §T.3, Ch 08 and Ch 13; his ch 20 *Patches & Community* is Part 6. Everything after 2.6 — EEVDF, folios, blk-mq, maple tree, io_uring, XDP, Rust — is additive |
| Corbet/Rubini/Kroah-Hartman, *Linux Device Drivers* 3e | Superseded by Part 2 (25 chapters vs LDD3's 18, on modern APIs) |
| Bovet & Cesati, *Understanding the Linux Kernel* | Parts 1 and 3 |
| Gorman, *Understanding the Linux Virtual Memory Manager* | Ch 11, 22, 23, 52, 105 |
| Tanenbaum, *Modern Operating Systems* | [reference/os-fundamentals.md](reference/os-fundamentals.md) |

---

## The four skills that separate principals from everyone else

1. **You model an unfamiliar subsystem in a day** — by finding the *central data structure*
   and the *three paths* that touch it (init, fast path, teardown), not by reading every line.
2. **You reason about concurrency statically.** "Who calls this? Under what lock? In what
   context? What can sleep? What's the memory ordering? What's the RCU grace period?"
3. **You think in lifetimes and ownership even in C** — refcounts, RCU, `devm_` teardown
   order, `module_get`. This is why strong C kernel devs pick up Rust quickly.
4. **You design for the maintainer, not the compiler.** Upstreamability is a design constraint.

---

## Prerequisites

- **C**: pointers, structs, function pointers, storage classes, preprocessor, UB.
  Shaky? Chapter 08 re-teaches C from a kernel angle.
- **Shell, git, computer architecture** (caches, MMU, rings, interrupts, DMA).
- Rust is **not** a prerequisite — Part 5 teaches Rust *as a kernel language*.

→ Start: [part0-foundations/00-orientation.md](part0-foundations/00-orientation.md)

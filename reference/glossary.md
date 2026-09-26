# Glossary

> Every term used in the curriculum, defined in one or two sentences, with the chapter that
> develops it. Where a term is commonly misunderstood, the misunderstanding is named.

---

## A

**ABI (Application Binary Interface)** — The binary contract between components: calling
conventions, struct layouts, syscall numbers. The kernel's *userspace* ABI is inviolable;
its *in-kernel* ABI deliberately does not exist. → Ch. 24, Ch. 88

**ACL (Access Control List)** — Permissions indexed by *object* ("who may touch this?").
Contrast with a capability, indexed by *subject*. → Ch. 102

**ACPI** — Firmware-provided hardware description and runtime methods on x86 and server ARM,
with an interpreted bytecode (AML). The x86 analogue of device tree, but executable. → Ch. 33

**ACS (Access Control Services)** — A PCIe capability that prevents peer-to-peer DMA between
devices, making them separable into distinct IOMMU groups. Without it, devices under a switch
share an IOMMU group. → Ch. 104

**Amdahl's law** — Speedup is bounded by the serial fraction. Does *not* predict the
throughput *decrease* that real systems show; see USL. → Ch. 105

**AML** — ACPI Machine Language; the bytecode ACPI methods are written in. → Ch. 33

**Anti-rollback** — A monotonic counter (usually in fuses) preventing installation of an
older, signed, vulnerable image. → Ch. 90, Ch. 99

**Arc** (Rust) — Atomically refcounted shared ownership. In kernel Rust it uses the kernel's
`refcount_t`. `Arc<T>` gives shared *ownership*, not shared *mutability*. → Ch. 79

**ARef** (Rust) — A reference to a C object that participates in the object's own refcount
(`get_device`/`put_device`). → Ch. 79

**Atomic context** — Any context where sleeping is forbidden: interrupt handlers, spinlock-
held regions, `preempt_disable()` regions. → Ch. 17

**Auxiliary vector (auxv)** — The kernel→loader ABI passed on the stack at `execve`:
`AT_PHDR`, `AT_BASE`, `AT_HWCAP`, `AT_SYSINFO_EHDR` (the vDSO), `AT_RANDOM`, `AT_SECURE`.
→ Ch. 92

---

## B

**Banker's algorithm** — Deadlock *avoidance* by granting only requests leaving a safe state.
Requires maximum claims declared in advance, which is why no OS uses it. → `os-fundamentals.md`

**Barrier** — See *memory barrier*. Not to be confused with block-layer "barriers," which
were replaced by FLUSH/FUA. → Ch. 13

**bbappend** — A Yocto file extending a recipe without forking it. The mechanism that keeps a
BSP maintainable. → Ch. 96

**Belady's anomaly** — FIFO page replacement can fault *more* with *more* frames. Stack
algorithms (LRU, OPT) cannot. → `os-fundamentals.md`

**BitBake** — The build engine underneath Yocto/OpenEmbedded. A Python task executor with a
dependency graph; knows nothing about Linux specifically. → Ch. 96

**blk-mq** — The multi-queue block layer: per-CPU software queues mapped onto hardware
queues, replacing the single-queue-single-lock design that could not reach NVMe rates.
→ Ch. 63

**BL1/BL2/BL31/BL32/BL33** — ARM Trusted Firmware boot stages. BL31 is the EL3 Secure Monitor
and **stays resident forever**, serving PSCI calls. → Ch. 90

**Bonding of hardware/policy config** — See *hardware vs non-hardware fragments*. → Ch. 98

**Borrow checker** — Rust's compile-time enforcement of ownership, borrowing (one `&mut` xor
many `&`), and lifetimes. → Ch. 77

**BPF / eBPF** — Verified, JIT-compiled programs attached to kernel hooks. Enables
userspace-authored policy at kernel-resident speed. → Ch. 75

**BSP (Board Support Package)** — The machine configuration, kernel recipe, bootloader, and
device tree that make a board buildable. Should contain *hardware* facts only. → Ch. 98

**BTF (BPF Type Format)** — Compact kernel type information enabling CO-RE: a BPF program
compiled once can relocate field offsets for a different kernel. → Ch. 75, Ch. 95

**Buildroot** — A Kconfig-driven build system producing a rootfs image directly, with no
packages. Fast to learn, poor incremental rebuilds. → Ch. 100

**buildpaths** — A Yocto QA check firing when a host build path leaks into a binary; a
reproducibility failure and often an information leak. → Ch. 97, Ch. 99

---

## C

**Capability (kernel)** — A slice of root's power: `CAP_NET_ADMIN`, `CAP_SYS_ADMIN`, ~40
others. `CAP_SYS_ADMIN` is effectively equivalent to root. → Ch. 102

**Capability (security theory)** — An unforgeable token that *is* the authority, indexed by
subject. File descriptors are genuine capabilities. → Ch. 24, Ch. 102

**CBS (Constant Bandwidth Server)** — The budget-enforcement mechanism pairing with EDF in
`SCHED_DEADLINE`, so an overrunning task damages only itself. → Ch. 21, Ch. 103

**cgroup** — Resource *accounting and limits*. Orthogonal to namespaces, which control
*visibility*. v2 has one unified hierarchy and forbids internal processes. → Ch. 93

**CFI (Control-Flow Integrity)** — Restricting indirect calls to valid targets. Constrains
only the forward edge; the backward edge needs shadow stacks. Neither prevents data-only
attacks. → Ch. 102

**Coffman conditions** — The four simultaneous requirements for deadlock: mutual exclusion,
hold-and-wait, no preemption, circular wait. Linux breaks the fourth. → `os-fundamentals.md`

**Coherency (USL β term)** — The cost of keeping caches consistent; grows as N², which is why
adding cores can *reduce* throughput. → Ch. 105

**Compound page / folio** — A folio is a head page, guaranteed by type. Introduced to remove
the head/tail ambiguity of `struct page`. → Ch. 52

**CO-RE (Compile Once, Run Everywhere)** — BPF programs relocating struct offsets at load
time using BTF. → Ch. 95

**CoW (Copy-on-Write)** — Share read-only; copy on first write. The mechanism behind `fork`,
CoW filesystems, and `Arc::make_mut`. → Ch. 22

**CRA (Cyber Resilience Act)** — EU regulation requiring SBOMs, defined support periods, and
vulnerability handling for products with digital elements. → Ch. 99

**crash / drgn** — Post-mortem kernel analysis tools. `drgn` is Python-based and scriptable,
works on live kernels, and is the better choice for most work. → Ch. 95

**CVE** — Since 2024 the kernel is a CNA and assigns thousands of CVEs annually. Per-CVE
triage is explicitly *not* the recommended strategy; continuous stable adoption is. → Ch. 88

---

## D

**DAC (Discretionary Access Control)** — The Unix model: the object owner sets permissions.
Contrast MAC. → Ch. 102

**DAMON** — Low-overhead memory-access monitoring, with DAMOS for acting on it (e.g. demoting
cold pages to a slow tier). → Ch. 105

**Delta** — Out-of-tree patches carried against upstream. A recurring cost, not a one-time
one; the central object of embedded kernel maintenance. → Ch. 88

**DEPENDS vs RDEPENDS** (Yocto) — Build-time versus runtime dependencies. Confusing them is
the most common recipe bug. → Ch. 96

**devm_** — Managed resource allocation, released automatically on probe failure or device
removal. Wrong when an object's lifetime is not the device's (e.g. an open fd). → Ch. 28

**Devres** (Rust) — The Rust equivalent, returning `Option` from `try_access()` so
post-unbind access is impossible in safe code. → Ch. 83

**Device tree** — A firmware-supplied data description of non-discoverable hardware. Check
`/sys/firmware/devicetree/base` for what the kernel *actually* received, which differs from
the source DTS after bootloader fixups. → Ch. 32, Ch. 90

**dm-verity** — Block-level integrity via a Merkle tree, verified at read time. → Ch. 66,
Ch. 102

**DMA ownership** — Between `dma_map` and `dma_unmap`, the memory belongs to the *device*.
CPU access in that window is undefined. No type system can enforce this. → Ch. 35, Ch. 83

**Double fetch** — Reading the same user memory twice, allowing userspace to change it
between a check and a use. A TOCTOU CVE class. → Ch. 24, Ch. 76

---

## E

**EDF (Earliest Deadline First)** — Optimal uniprocessor scheduling; schedulable iff U ≤ 1.
Its overload failure is a domino effect, which is why fixed-priority schemes persist.
→ Ch. 21, `os-fundamentals.md`

**EEVDF** — Linux's scheduler since 6.6: eligibility by *lag*, selection by virtual
*deadline*. Adds a latency dimension CFS lacked. → Ch. 21

**EL0/EL1/EL2/EL3** — ARM exception levels. Linux normally runs at EL2 so KVM can use it;
firmware handing off at EL1 silently disables KVM. → Ch. 90

**ELF** — The executable format. Has a *linking* view (sections) and an *execution* view
(segments); the kernel reads only program headers. → Ch. 92

**Ephemeral vs durable** — See *fsync*. An acknowledged write is durable only after the
device's cache flush completes. → Ch. 94

**ET_DYN** — An ELF type covering both shared objects *and* PIE executables, which is why
`file` calls a PIE binary a "shared object." → Ch. 92

---

## F

**False sharing** — Logically independent variables in one cacheline, causing ping-pong with
no logical contention. Find with `perf c2c`. → Ch. 105

**FIT image** — U-Boot's Flattened Image Tree: kernel, DTB, and initramfs in one signed
container. Signing the *configuration*, not just the images, prevents mix-and-match. → Ch. 90

**Folio** — See *compound page*. A type-level guarantee replacing a runtime convention.
→ Ch. 52

**ForeignOwnable** (Rust) — The `drvdata` pattern made safe: `into_foreign` gives ownership to
C, `from_foreign` takes it back (exactly once), `borrow` peeks. → Ch. 79

**fsync** — Forces a file's data and retrieval metadata to durable storage, *including* a
device cache flush. Does **not** make the directory entry durable — `fsync` the parent too.
→ Ch. 53, Ch. 94

---

## G

**GFP flags** — Allocation context: `GFP_KERNEL` may sleep, `GFP_ATOMIC` may not. Kernel Rust
makes the flag an explicit parameter. → Ch. 11, Ch. 79

**GKI (Generic Kernel Image)** — Android's one-kernel-for-all-devices programme, with a
stable KMI and everything vendor-specific as a module. The best-documented delta-reduction
effort. → Ch. 88

**Group commit** — Batching multiple writers into one durability operation, amortizing the
flush cost. → Ch. 94

---

## H

**Hardware vs non-hardware fragments** — A Yocto kernel-config split: hardware facts belong
in the BSP, policy in the distro. Most BSPs ignore the distinction. → Ch. 98

**Hash equivalence** — BitBake's optimization treating different signatures as equivalent when
they produce identical output, preventing rebuild cascades. → Ch. 96

**HITM (Hit-Modified)** — A cacheline fetched from another cache's modified copy;
200–400 ns and the signal `perf c2c` reports. → Ch. 105

**hrtimer** — High-resolution timers: rbtree-ordered, nanosecond precision, programmed into a
clock-event device. Contrast the timer wheel, optimized for *cancellation*. → Ch. 19

**Hyrum's Law** — Every observable behaviour of an interface will be depended upon, including
the accidents and the bugs. → Ch. 24, Ch. 89

---

## I

**IMA/EVM** — Integrity Measurement Architecture: measures files into TPM PCRs before use,
optionally enforcing signatures. → Ch. 102

**Initramfs** — A cpio archive unpacked into a tmpfs, breaking the chicken-and-egg of needing
drivers to mount the root. Not a ramdisk (that was `initrd`). → Ch. 91

**Intrusive data structure** — The list node is embedded in the object, so insert/remove
never allocates and cannot fail. Requires `container_of`. → Ch. 09

**IOMMU** — Address translation and protection for device DMA. The only defence against a
malicious peripheral. `strict` mode invalidates synchronously and costs throughput; lazy mode
leaves a window. → Ch. 36

**io_uring** — Shared-memory submission/completion rings making the syscall optional. Three
tiers: inline non-blocking, internal poll, io-wq worker fallback. → Ch. 76

**IPC (instructions per cycle)** — Under 1.0 usually means stalled on memory. → Ch. 95

---

## K

**kABI** — A stable in-kernel module ABI. Upstream Linux deliberately has none; distributions
synthesize one with symbol whitelists, checksums, and struct padding. → Ch. 88

**KASAN / KCSAN / KMSAN / UBSAN** — Compile-time-instrumented detectors for memory errors,
data races, uninitialized memory, and undefined behaviour. → Ch. 06

**KASLR / KPTI** — Kernel address randomization; kernel page-table isolation (the Meltdown
mitigation, 5–30% syscall cost). → Ch. 102

**KFENCE** — Sampling memory-error detector with ~0% overhead, designed for production. The
answer to "KASAN is too slow for the fleet." → Ch. 95

**kprobe vs tracepoint** — A kprobe attaches anywhere and breaks on rename or inlining; a
tracepoint is a stable-ish declared interface with zero cost when disabled. Prefer
tracepoints. → Ch. 95

---

## L

**Lag** (EEVDF) — The difference between the service an entity *should* have received under
ideal fluid sharing and what it got. Positive lag means eligible. → Ch. 21

**Landlock** — An LSM allowing an **unprivileged** process to restrict *itself*, inherited
across `execve` and irreversible. → Ch. 102

**Lifetime** (Rust) — A compile-time region for which a reference is valid. Use-after-free is
a lifetime violation. → Ch. 78

**Lock holder preemption** — A vCPU descheduled while holding a spinlock, causing other vCPUs
to spin for a full timeslice. Why CPU-oversubscribed VMs perform catastrophically rather than
proportionally. → Ch. 104

**Lockdep** — Runtime lock-order validation. Reports a *possible* deadlock the first time both
orders are observed, not an actual one. Fires once. → Ch. 14

**Lockdown (LSM)** — Enforces that root ≠ kernel under Secure Boot. `integrity` blocks
unsigned modules and `/dev/mem`; `confidentiality` additionally blocks reading kernel memory,
which breaks most debugging. → Ch. 102

**LSM (Linux Security Module)** — A hook framework for mandatory access control. LSMs can only
*further restrict*, never grant — the invariant that made a pluggable framework acceptable.
→ Ch. 102

**LTS** — A long-term-support kernel, maintained 2 years (extendable to 6). Check the EOL
date against your product's support commitment. → Ch. 88

---

## M

**MAC (Mandatory Access Control)** — Policy the object owner cannot override: SELinux,
AppArmor, Smack. → Ch. 102

**maple tree** — The modern VMA index, replacing the rbtree. RCU-safe and range-oriented.
→ Ch. 22

**MCS lock / qspinlock** — A queued spinlock where each waiter spins on **its own** cacheline,
giving O(1) coherence traffic per handoff instead of O(N). The best example of "fix the data
structure, not the lock." → Ch. 105

**memcg** — The cgroup memory controller. `memory.max` kills, `memory.high` throttles,
`memory.min`/`.low` protect. People set limits and forget protections. → Ch. 23, Ch. 93

**MESI** — The cache coherence protocol. Read sharing is nearly free; **write sharing is
not** — the fact underlying all scalability work. → Ch. 105

**MLFQ (Multi-Level Feedback Queue)** — Approximates SJF without knowing job length, by
demoting CPU-bound tasks and periodically boosting everything. → `os-fundamentals.md`

**Multiconfig** (Yocto) — Building several configurations in one invocation, with
cross-configuration dependencies. → Ch. 97

---

## N

**Namespace** — Controls *visibility*: mount, UTS, IPC, PID, network, user, cgroup, time.
Orthogonal to cgroups. → Ch. 93

**NAPI** — Interrupt-driven at low load, polled at high load, switching automatically. Exists
to prevent *receive livelock*. → Ch. 46

**Narrow waist** — A small stable interface with many implementations below and many users
above: syscalls, VFS, `file_operations`, blk-mq. → Ch. 89

**NUMA** — Non-uniform memory access. Remote DRAM is 1.5–2.2× local. Default placement is
*first touch*, which is usually right and occasionally catastrophic. → Ch. 105

---

## O

**Off-CPU analysis** — Measuring where time is spent *blocked*. When the CPU is idle and the
system is slow, on-CPU profiling tells you nothing. → Ch. 95

**Opaque\<T\>** (Rust) — Wraps a C struct: `UnsafeCell` (C may mutate), `MaybeUninit` (C may
not have initialized), `PhantomPinned` (C holds pointers). Four lines that teach FFI safety.
→ Ch. 79

**Overlay root** — OpenWrt's design: an immutable squashfs base plus a writable overlay.
Factory reset is wiping the overlay. → Ch. 100

---

## P

**Page fault** — The programmable interception point behind demand paging, COW, mmap,
swapping, and shared memory. Minor ~1 µs, major ~100 µs. → Ch. 22

**PACKAGECONFIG** (Yocto) — The correct mechanism for optional features: five comma-separated
fields (enable, disable, DEPENDS, RDEPENDS, conflicts). → Ch. 97

**Per-CPU counter** — Writes are local and contention-free; reads are O(N_cpus) and
approximate. The right trade whenever writes outnumber reads. → Ch. 16, Ch. 105

**Pin / pin-init** (Rust) — `Pin<P>` withholds `&mut`, so a value cannot be moved. `pin-init`
initializes in place, with generated partial-teardown on failure. Required because kernel
structs are self-referential. → Ch. 80

**Pinmux** — Pin multiplexing. A peripheral whose driver probes but whose hardware does
nothing is usually a pinmux problem. → Ch. 42, Ch. 101

**pivot_root vs chroot** — `pivot_root` moves the old root's *mount* so it can be unmounted;
`chroot` leaves it reachable. Containers use `pivot_root`. → Ch. 93

**PLT / GOT** — Procedure Linkage Table and Global Offset Table: the indirection enabling lazy
symbol binding. Full RELRO (`-z now -z relro`) makes the GOT read-only after relocation.
→ Ch. 92

**Popek–Goldberg** — An architecture is efficiently virtualizable iff sensitive instructions
⊆ privileged instructions. x86 failed with 17. → Ch. 104

**PREEMPT_RT** — Merged in 6.12 after ~20 years. Spinlocks become sleeping `rt_mutex`es, IRQs
and softirqs become threads. `raw_spinlock_t` remains non-preemptible. → Ch. 103

**Priority inversion** — H blocks on L's lock; M preempts L; H waits behind M, unbounded.
Fixed by priority inheritance or ceiling. Mars Pathfinder, 1997. → Ch. 14, Ch. 103

**PSCI** — The ARM power-state interface. Every `cpu_up`, suspend, and reboot on ARM64 is an
SMC into the resident EL3 firmware. → Ch. 90

**PSI (Pressure Stall Information)** — Measures *stall time*, not utilization. The metric an
autoscaler should read. → Ch. 23, Ch. 93

---

## R

**RAID-5 write hole** — Power loss between a partial-stripe data write and its parity write
leaves an undetectably inconsistent stripe. Fixes: battery-backed cache, a write journal, or
CoW. → Ch. 67, Ch. 94

**RCU** — Readers write nothing at all; writers defer reclamation until a grace period. Buys
perfect read scalability with delayed reclamation. Why it beats `rwlock`: rwlock readers write
the lock word. → Ch. 15

**RELRO** — Making the GOT read-only after relocation (`-z relro -z now`), removing a classic
exploitation primitive. → Ch. 92, Ch. 102

**Reproducible build** — Same inputs, bit-identical output. Enables verification, debugging,
caching, and provenance. `diffoscope` finds the differences. → Ch. 99

**RPO / RTO** — Recovery Point Objective (how much data may be lost — determines backup
frequency) and Recovery Time Objective (how long recovery may take — determines the method).
These two numbers drive the whole design. → Ch. 94

---

## S

**sbitmap** — A scalable bitmap with per-CPU caching, used for blk-mq tag allocation. → Ch. 63

**SBOM** — A machine-readable component inventory (SPDX or CycloneDX). Required by US EO
14028 and the EU CRA. → Ch. 99

**sched_ext** — BPF-authored schedulers: kernel mechanism, userspace policy. → Ch. 21

**Seccomp-BPF** — Syscall filtering. Deliberately **cannot dereference pointers**, to avoid a
TOCTOU window. **Does not filter `io_uring` operations.** → Ch. 102, Ch. 76

**Send / Sync** (Rust) — `T: Send` = movable across threads; `T: Sync` ⟺ `&T: Send`. Raw
pointers are neither, which forces every C wrapper to state its concurrency contract in
writing. → Ch. 81

**Seqlock** — Writers bump a counter; readers snapshot and retry. Writer-preferring, so
readers can starve; readers may observe torn state and must not act on it before validating.
→ Ch. 14

**SEV / TDX / CCA** — Confidential computing: the guest does not trust the hypervisor. Forces
explicit shared memory for I/O, makes every device input attacker-controlled, and requires
attestation before provisioning secrets. → Ch. 104

**Socket activation** — systemd creates all sockets before starting anything, so the
dependency is on the *interface*, not the *implementation*. The reason parallel startup
works. → Ch. 91

**sstate** — Yocto's shared-state cache, keyed on a task signature. The reason Yocto is usable
and Buildroot's incremental builds are painful. → Ch. 96

**Static key** — A patched NOP/JMP making a disabled tracepoint literally free. The mechanism
that lets the kernel carry thousands of tracepoints. → Ch. 06, Ch. 95

**Stop-drain-free** — The teardown order: stop new work, drain in-flight work, then free.
Reversing it is the most common driver bug shape. → Ch. 25, Ch. 83

---

## T

**Tainted** — Kernel flags indicating reduced trustworthiness of a bug report: `P`
proprietary, `O` out-of-tree, `D` previous oops, `W` warning. Always check it. → Ch. 06

**TLB shootdown** — The TLB is the only cache with no hardware coherence, so a page-table
change requires an IPI to every CPU that might have cached it. 1–10 µs, and a source of
unexplained jitter on *unrelated* cores. → Ch. 22

**TOCTOU** — Time-of-check to time-of-use. The reason seccomp cannot dereference pointers and
the reason `*at()` syscalls take a dirfd. → Ch. 24, Ch. 102

**Two-phase commit (validate-then-apply)** — Check everything, commit nothing until all checks
pass. KMS atomic, V4L2 `TRY_FMT`, clock rate changes, filesystem transactions. → Ch. 43,
Ch. 47, Ch. 89

---

## U

**UAPI** — The userspace-facing API. Extensibility idioms: a size field plus
`copy_struct_from_user`, rejecting unknown flags, returning an fd, the `*at()` form.
→ Ch. 24

**unsafe** (Rust) — Unlocks exactly five abilities and disables *none* of the borrow checker.
A proof-obligation marker, not "unsafe code." → Ch. 77, Ch. 84

**Upstream-first** — Every fix goes to mainline first, always. Anything else is a recurring
rebase tax forever. → Ch. 88

**USL (Universal Scalability Law)** — C(N) = N / (1 + α(N−1) + βN(N−1)). The βN² coherency
term makes throughput peak and then *decline*. → Ch. 105

---

## V

**vDSO** — A kernel-provided shared object mapped into every process, giving syscall-free
`clock_gettime` (~20 ns vs ~300 ns) via a seqlock-protected shared page. → Ch. 92

**VFIO** — Safe userspace device access via IOMMU isolation. The basis of device passthrough
and userspace drivers. → Ch. 36

**virtio / vhost / vDPA** — Paravirtualized I/O (shared-memory rings, one exit per batch);
the backend moved into the kernel; and real hardware speaking the virtio datapath while
keeping virtio's control plane and live migration. → Ch. 104

**vtable** (Rust `#[vtable]`) — A proc macro generating a C `*_ops` struct from a trait impl,
with `HAS_*` constants producing NULL for unimplemented methods. → Ch. 78

---

## W

**wic** — Yocto's disk-image creator, driven by a `.wks` kickstart file. → Ch. 98

**Working set** — W(t, τ), the pages referenced in the last τ. Thrashing occurs when the sum
of working sets exceeds memory — a phase transition, not gradual degradation. →
`os-fundamentals.md`, Ch. 23

**Write hole** — See *RAID-5 write hole*.

**Writeback** — Deferred flushing of dirty page-cache pages. On large-memory machines set
`vm.dirty_bytes`, not `dirty_ratio` — 20% of 512 GB is a multi-minute stall. → Ch. 52, Ch. 94

---

## X

**XDP** — eBPF in the driver, on the DMA buffer, **before** `sk_buff` allocation. ~10× for
drop/redirect workloads because it skips the most expensive per-packet operation. → Ch. 74

**XArray** — The modern radix-tree-based index, used by the page cache. → Ch. 10

---

## Y

**Yocto** — The umbrella project; **BitBake** is the engine, **OE-Core** is the metadata,
**Poky** is the reference distribution. Machine = hardware, distro = policy, image =
contents. → Ch. 96

---

## Z

**Zero-cost abstraction** — An abstraction compiling to the same code as the hand-written
version. True for Rust generics, `Guard`, `IoMem` bounds, and `Option<&T>` — **not**
universally: `Arc::clone` is an atomic, and `LockedBy::access` is a branch. Treating it as
universal is a red flag. → Ch. 77, Ch. 81

**Zombie** — A process that has exited but not been reaped. PID 1 must `wait()`, which is why
running a bare shell as PID 1 in a container leaks them. → Ch. 20, Ch. 91

→ Back: [../README.md](../README.md) | Next: [reading-list.md](reading-list.md)

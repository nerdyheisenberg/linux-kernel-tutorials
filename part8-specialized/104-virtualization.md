# Chapter 104 — KVM and Virtualization Internals

> You have used QEMU since Ch. 04 as a tool. You met VFIO in Ch. 36 as a way to hand a
> device to userspace. This chapter is about what is underneath: how a CPU runs a guest
> kernel, what a VM exit costs, why virtio exists, and where the boundary between the
> hypervisor and the guest actually is. It is also where containers, microVMs, and
> confidential computing get placed on a single map.

---

## Theory & First Principles

### T.0 — Start here: run a kernel inside a kernel

A guest OS was written on the assumption that it owns the machine. It will set up page tables,
program the interrupt controller, execute `cli`, and talk to devices. **Your job is to let it
do all of that while it is actually an ordinary process under `ps`.**

```bash
ps aux | grep qemu        # a Linux VM is a PROCESS. With threads, one per vCPU.
```

**The classical requirement, from Popek and Goldberg (1974), is one sentence:**

> **Every instruction that could reveal or alter privileged machine state must *trap* when
> executed in unprivileged mode**, so the hypervisor can emulate it.

**x86 failed this for ~17 instructions.** The infamous one: `POPF` executed in user mode
*silently ignores* the interrupt-flag bits instead of faulting. So a guest kernel doing `cli`
would neither disable interrupts nor trap — **it would just quietly do the wrong thing.** No
trap means no chance to emulate.

**Three historical answers, and they are a good lesson in how a hard problem gets solved:**

| Approach | Mechanism | Cost |
|---|---|---|
| **Binary translation** (VMware, 1999) | scan guest kernel code and rewrite the unsafe instructions at runtime | enormously complex; a genuine engineering triumph |
| **Paravirtualization** (Xen, 2003) | change the guest to call the hypervisor explicitly | **you must modify the guest OS** |
| **Hardware virtualization** (VT-x / AMD-V, 2006) | a new CPU mode: the guest runs at its own ring 0, and privileged operations cause a **VM exit** | needs hardware; the exits themselves cost ~1–10 µs |

**Option 3 is why KVM is tiny.** The CPU does the hard part; KVM is a driver (`/dev/kvm`) that
sets up the VMCS and handles exits, and **Linux itself becomes the hypervisor** — reusing its
scheduler for vCPUs, its allocator for guest memory, its cgroups for limits, and its drivers
for hardware. **Compare Xen, which had to implement all of that itself.** Reusing an existing
kernel rather than writing a parallel one is the single largest architectural decision in KVM,
and it is why it won.

**Now the remaining hard problems, because the CPU only solved the CPU:**

**1. Memory: two levels of translation.** The guest translates GVA → GPA; the host must then
translate GPA → HPA. Software "shadow page tables" were fast to run and miserable to maintain
(every guest page-table write had to trap). **EPT/NPT put the second translation in hardware**
— a 2-level walk, with a deeper TLB miss penalty, but no traps.

**2. I/O: emulation is far too slow.** Emulating a real NIC means a VM exit per register
access — dozens per packet, at ~2 µs each. **So: stop pretending.**

```
   virtio: the guest KNOWS it is virtualized.

     shared ring in guest memory  <-- the host reads it directly
     guest fills descriptors (cheap stores)
     ONE doorbell write -> ONE vm exit -> the host processes a BATCH
```

**That is Ch. 69 §T.0's NVMe doorbell and Ch. 76 §T.0's `io_uring` ring, for the third time.**
Whenever two agents cannot cheaply call each other, the answer is a shared-memory ring with a
batched notification. **`vhost` then moves the host side into the kernel** to avoid a further
userspace round trip, and **SR-IOV/VFIO** goes furthest of all — giving the guest a real PCIe
function, with the IOMMU (Ch. 36 §T.0) providing the isolation that software used to.

**And the honest summary of the containers-versus-VMs question** (Ch. 93 §T.0), which you
should be able to give in one breath:

| | Containers | VMs |
|---|---|---|
| Isolation boundary | ~400 syscalls into 30M lines | a narrow hardware interface + ~50K lines |
| Startup | milliseconds | ~100 ms (Firecracker) to seconds |
| Overhead | ~0 | a few % CPU, plus guest memory |
| Different kernel/OS per tenant | no | yes |

**Firecracker is the interesting synthesis:** a VMM that implements only virtio-net,
virtio-block, and a serial port — under 50K lines of Rust — booting in ~125 ms. **VM-grade
isolation at near-container cost, achieved by deleting features rather than adding them.**
That is a design lesson worth more than the specific product.

```bash
lsmod | grep kvm && ls -l /dev/kvm
sudo perf kvm stat live                  # VM exits by reason, live
cat /sys/kernel/debug/kvm/*              # exit counters
lspci -vvv | grep -i 'SR-IOV\|Virtual Function'
ls /sys/kernel/iommu_groups/
```

---

### T.1 — The Popek–Goldberg criterion, and why x86 failed it

The formal foundation, 1974, and a question that is asked directly:

> An architecture is **efficiently virtualizable** iff the set of **sensitive** instructions
> is a subset of the set of **privileged** instructions.

- **Privileged**: traps when executed in user mode.
- **Sensitive**: either changes privileged state (*control-sensitive*) or reveals it
  (*behaviour-sensitive*).

If every sensitive instruction traps, you can build a hypervisor by **trap-and-emulate**:
run the guest directly on the CPU in user mode, catch every trap, emulate it, resume. Fast,
simple, provably correct.

x86 had **17 sensitive-but-unprivileged instructions** — `SGDT`, `SIDT`, `SLDT`, `SMSW`,
`PUSHF`/`POPF`, `LAR`, `LSL`, `VERR`, `VERW`, and others. `POPF` is the classic: executed in
user mode it *silently ignores* the interrupt-flag bits instead of faulting. A guest kernel
doing `cli` would appear to succeed and then take interrupts — silent incorrect behaviour,
not a trap you can intercept.

The three historical responses:

| Approach | Method | Cost |
|---|---|---|
| **Binary translation** (VMware, 1999) | Scan and rewrite guest kernel code, replacing sensitive instructions | Enormous engineering; translation cache; still fast |
| **Paravirtualization** (Xen, 2003) | Modify the guest to call the hypervisor explicitly (hypercalls) | Requires guest source; cannot run Windows unmodified |
| **Hardware extensions** (VT-x 2005 / AMD-V 2006) | Add a new *mode* below ring 0 where all sensitive instructions exit | Made the other two largely obsolete |

KVM (merged in 2.6.20, 2007) exists **because** of the third. It is a thin driver that
exposes the hardware virtualization extensions through `/dev/kvm`; it was ~10k lines at
merge precisely because the hardware did the hard part.

### T.2 — The hardware mechanism: VMX / SVM

Intel VT-x introduces two modes and a control structure:

```
   VMX root mode (host / hypervisor)          VMX non-root mode (guest)
   ┌──────────────────────────────┐           ┌──────────────────────────┐
   │ ring 0: KVM, Linux kernel    │  VMLAUNCH │ ring 0: guest kernel     │
   │ ring 3: QEMU                 │ ────────► │ ring 3: guest userspace  │
   │                              │           │                          │
   │                              │ ◄──────── │                          │
   └──────────────────────────────┘  VM exit  └──────────────────────────┘
                     ▲
                     │
               VMCS (in memory): guest state, host state,
                                 execution controls, exit info
```

Crucially, the guest runs with its **own ring 0** — the guest kernel really executes in ring
0, in non-root mode. There is no ring de-privileging. What changes is that a configurable
set of events causes a **VM exit** back to root mode.

**The VMCS** (`VMCB` on AMD) is the per-vCPU control block with four regions:
- *Guest state*: all registers, saved on exit and restored on entry by hardware.
- *Host state*: what to restore on exit.
- *VM-execution controls*: **which events exit.** This is the tuning surface — you choose
  whether `CPUID`, `HLT`, `RDTSC`, specific MSR accesses, I/O ports, or CR accesses exit.
- *VM-exit information*: the reason, plus qualification data.

**VM exit cost** is the number that governs all virtualization performance:

| Era | Round-trip exit cost |
|---|---|
| 2006 (first generation) | ~3000–4000 cycles |
| Modern (2020s) | **~1000–1500 cycles ≈ 300–500 ns** |
| With mitigations (IBPB, L1D flush) | considerably more |

So: **an exit costs roughly a syscall, or a bit more.** Everything in virtualization
performance is "avoid exits." Once you internalize that, virtio, posted interrupts, EPT, and
APICv all become obviously motivated rather than a list of acronyms.

AMD SVM differs in details worth knowing: `VMRUN`/`VMEXIT` instead of `VMLAUNCH`/`VMRESUME`,
a `VMCB` with a cleaner layout, and Nested Page Tables instead of EPT (same idea, different
name).

### T.3 — Memory virtualization: two levels of translation

The guest has its own page tables mapping Guest Virtual → Guest Physical. But Guest Physical
is not Host Physical. Somebody must compose the two.

**Shadow page tables** (the software era): the hypervisor maintains a hidden page table
mapping GVA→HPA directly, and write-protects the guest's page tables so every guest page
table modification exits. Correct, but a page-fault storm — the guest cannot touch its own
page tables without an exit.

**EPT / NPT** (the hardware era): a second-level page table, walked by hardware.

```
 Guest CR3 ──► Guest page tables ──► GPA ──► EPT ──► HPA
                (guest controls,               (host controls,
                 no exits needed)               no exits needed)
```

A TLB miss now requires walking *both* tables. For 4-level paging on each side that is up to
**24 memory accesses** (5×4 + 4) instead of 4. This is why:
- **Huge pages matter enormously in VMs** — they cut both walks.
- Tagged TLBs (**VPID** on Intel, **ASID** on AMD) matter, so a VM entry does not flush.

The practical guidance that follows: back guest memory with hugepages (`hugetlbfs` or THP),
and pin it for latency-sensitive guests. A guest whose memory is swapped by the host suffers
a fault the guest cannot see, diagnose, or account for.

**Memory overcommit techniques**, each with a sharp edge:
- **Ballooning** (`virtio-balloon`): a guest driver allocates pages and tells the host it may
  reclaim them. Requires guest cooperation, and is slow to react.
- **Free page reporting**: the guest proactively reports free pages. Better.
- **KSM** (kernel same-page merging): the host scans and merges identical pages. Saves a lot
  with many similar guests — and is a **side channel**: an attacker can detect whether a page
  exists in another VM by timing a write (a merged page is COW and therefore slow). Do not
  enable KSM across security boundaries.
- **Host swapping**: works, invisible to the guest, and destroys latency predictability.

### T.4 — The vCPU run loop, which is the core of KVM

This is the single most important structure to be able to draw:

```
 QEMU/userspace thread (one per vCPU)
   │
   └─► ioctl(vcpu_fd, KVM_RUN)
         │
         └─► kvm_arch_vcpu_ioctl_run()
               │
               ├─► vcpu_enter_guest()
               │     ├─ handle pending requests (TLB flush, MMU reload, ...)
               │     ├─ disable interrupts, load guest FPU/MSRs
               │     ├─ VMLAUNCH/VMRESUME  ────────────────► guest runs
               │     │                                        │
               │     │   ◄──────────────────────────────────  VM exit
               │     ├─ restore host state
               │     └─ kvm_x86_handle_exit()
               │          │
               │          ├─ handled IN KERNEL  ──► loop again  (FAST: ~0.5 µs)
               │          │    e.g. EPT violation, CPUID, MSR, HLT,
               │          │         in-kernel APIC/PIT/PIC, coalesced MMIO,
               │          │         ioeventfd-registered MMIO/PIO writes
               │          │
               │          └─ needs USERSPACE  ──► return from KVM_RUN
               │               e.g. unhandled MMIO, PIO to a QEMU device
               │               │
               │               ▼
               │          QEMU emulates the device, updates kvm_run->mmio
               │               │
               └───────────────┘  ioctl(KVM_RUN) again   (SLOW: ~5-10 µs)
```

**The kernel/userspace split is the central design decision.** KVM handles in-kernel
whatever is hot (MMU, local APIC, timers, IPIs); everything else returns to userspace, where
QEMU has the full device model. That is a 10–20× cost difference, which is why the
in-kernel irqchip and `ioeventfd` exist.

`struct kvm_run` is the shared memory page between KVM and userspace — userspace `mmap`s the
vCPU fd and reads the exit reason and payload from it, avoiding a copy on every exit.

### T.5 — I/O virtualization: four levels, four prices

| Approach | Mechanism | Cost per I/O | Live migration |
|---|---|---|---|
| **Full emulation** | Guest MMIO/PIO exits to QEMU, which emulates e1000/IDE | ~10 µs, several exits | yes |
| **virtio** | Paravirtualized: shared-memory rings, one exit per *batch* | ~1–3 µs | yes |
| **vhost** | virtio backend moved **into the kernel** (`vhost-net`, `vhost-scsi`) | sub-µs, no QEMU round-trip | yes |
| **vhost-user / DPDK** | Backend in another *userspace* process, shared memory | sub-µs, polling | yes |
| **SR-IOV / VFIO passthrough** | Guest drives a real VF directly; IOMMU isolates | near-native | **hard** (needs vendor support or vDPA) |
| **vDPA** | Real hardware speaking the virtio datapath, virtio control plane | near-native | **yes** — this is the point |

**virtio is the most important idea here.** The insight: emulating a real NIC is absurd —
the guest writes a register, exits, QEMU decodes a hardware protocol designed for a
1999 chip. Instead, define a device interface *designed for virtualization*: a shared-memory
ring (the virtqueue) with descriptors, so the guest enqueues N requests and **kicks once**.
One exit amortized over a batch.

```
   Guest driver                          Host (QEMU or vhost)
   ┌────────────────┐                    ┌─────────────────┐
   │ descriptor tbl │◄──── shared ──────►│                 │
   │ avail ring     │──── guest writes ─►│ reads           │
   │ used ring      │◄─── host writes ───│ writes          │
   └────────────────┘                    └─────────────────┘
          │ kick (MMIO write -> ONE exit per batch)
          └────────────────────────────────────►
          ◄──────────────── interrupt (or polled) ──────────
```

The modern `packed` virtqueue layout replaces the three separate structures with one ring,
improving cache behaviour — a nice example of a design being revised once the cost model was
understood.

**vDPA** deserves the last word: it lets real hardware implement the virtio *datapath* while
the control plane stays virtio, so you get passthrough performance **and** live migration
**and** a single guest driver. It is the convergence point of this whole table.

### T.6 — Interrupt virtualization

Delivering an interrupt to a guest naively costs an exit. Three generations of fix:

1. **In-kernel irqchip** — KVM emulates the local APIC and I/O APIC in the kernel, so an
   interrupt injection does not round-trip to QEMU. (`KVM_CREATE_IRQCHIP`.)
2. **`irqfd` / `ioeventfd`** — an eventfd wired directly into KVM. A host thread writes the
   eventfd; KVM injects the interrupt with no userspace involvement. `ioeventfd` is the
   reverse: a guest write to a specific MMIO address signals an eventfd without exiting to
   userspace. Together they are what make vhost fast.
3. **Posted interrupts (APICv / AVIC)** — the hardware delivers an interrupt **directly into
   the running guest with no VM exit at all**. A remote CPU writes a posted-interrupt
   descriptor and sends a special IPI; the guest's vCPU takes the interrupt in non-root mode.

The progression is the same story as everything else in this chapter: **each generation
removes an exit.**

### T.7 — Nested virtualization

L0 (bare metal) runs L1 (a hypervisor) which runs L2. The hardware only has one level of
VMX, so L0 must emulate VMX for L1:

- L1 thinks it executes `VMLAUNCH`; that instruction exits to L0.
- L0 merges L1's VMCS (`vmcs12`) with its own controls into a real `vmcs02` and runs L2.
- Every L2 exit goes to **L0**, which decides whether to handle it or reflect it to L1.
- The same shadowing applies to EPT: L0 must compose L1's EPT with its own.

Cost: an exit that L1 must handle costs **two** exits plus merging. Nested is usable
(it is what runs CI, nested containers-in-VMs, and Windows with Hyper-V/VBS enabled by
default on modern Windows) but is measurably slower and has historically been a rich source
of security bugs — the merging logic is where L1 controls data that L0 must validate.

**VMCS shadowing** (hardware) lets L1's `VMREAD`/`VMWRITE` access a shadow VMCS without
exiting, which removes the largest nested overhead.

### T.8 — Confidential computing: inverting the trust model

Classically the guest trusts the hypervisor completely. Confidential computing removes that:

| Technology | Protects | Mechanism |
|---|---|---|
| **AMD SEV** | Guest memory confidentiality | Per-VM memory encryption key in the memory controller |
| **SEV-ES** | + register state | Encrypted guest state on exit; `#VC` exception for explicit exits |
| **SEV-SNP** | + **integrity** | Reverse Map Table (RMP) prevents host remapping/replay |
| **Intel TDX** | Full | TD-module (signed firmware) mediates; SEAM mode |
| **ARM CCA** | Full | Realm Management Extension, Realm world |
| **Intel SGX** | Process-level enclaves | Different model: enclave, not VM |

The architectural consequences ripple everywhere and are what makes this an *architecture*
topic rather than a feature:

1. **DMA and MMIO must use explicitly shared memory.** The host cannot read private guest
   pages, so every I/O buffer must be in a page marked shared. Hence `swiotlb` becomes
   mandatory (`CONFIG_AMD_MEM_ENCRYPT` forces bounce buffering) and there is a real
   throughput cost.
2. **The host is now an attacker.** Every value the guest reads from a virtual device —
   every MMIO read, every virtio descriptor, every ACPI table — is attacker-controlled.
   Decades of drivers that trust their hardware are now attack surface. This is the
   "device filter" / hardening work (`CONFIG_ARCH_HAS_CC_PLATFORM`, restricted device
   allowlists in CoCo guests).
3. **Interrupts are attacker-controlled.** The host can inject or withhold interrupts;
   SEV-SNP's "restricted injection" and alternate injection exist for this.
4. **Attestation is mandatory before provisioning secrets.** The guest proves its launch
   measurement to a remote party, which only then releases the disk-encryption key. Without
   attestation, encryption is pointless — the host could have launched a modified image.
5. **Live migration becomes hard**: the memory is encrypted with a key the host lacks, so
   migration requires a firmware-mediated protocol between source and destination.

### T.9 — The isolation spectrum, and how to choose

This is the framing an architect is expected to supply:

```
 weaker isolation                                          stronger isolation
 ◄──────────────────────────────────────────────────────────────────────────►
 process       container      gVisor       Kata/Firecracker      full VM
 (namespace)   (ns+cgroup)    (userspace   (microVM, minimal     (QEMU,
                              kernel)       device model)         full devices)

 shared kernel ────────────────────┤├──────────── separate kernel
 ~0 overhead    ~0 overhead    ~10-50%      ~5 MiB, ~125 ms boot   ~50-200 MiB
```

| Boundary | Attack surface | When it is right |
|---|---|---|
| Process | full kernel, ~400 syscalls | same trust domain |
| Container | full kernel + namespace code | **mutually non-hostile** tenants, operational convenience |
| gVisor | ~60 host syscalls (userspace kernel intercepts the rest) | hostile code, tolerant of syscall-heavy slowdown |
| **microVM** (Firecracker/Cloud Hypervisor) | KVM + ~5 virtio devices | **hostile multi-tenant with container economics** — the modern default for FaaS |
| Full VM | KVM + full QEMU device model | legacy OS, device passthrough, full fidelity |
| CoCo VM | + hostile *host* | regulated data on untrusted infrastructure |

The judgement to voice: **Firecracker changed this calculus.** In 2016 the argument for
containers-as-a-security-boundary was economic — VMs were too slow and too fat. A 125 ms
boot and 5 MiB of overhead removes most of that argument, which is why AWS Lambda and
Fargate run on microVMs rather than on containers. If someone asks "containers or VMs for
untrusted tenants," the answer is "microVMs," and the reason is that the economic premise of
the original question expired.

### T.10 — What the guest should know it is a guest

Paravirtualization did not disappear; it moved from "replacing privileged instructions" to
"cooperating where cooperation pays":

| PV feature | Problem solved |
|---|---|
| **PV clock** (`kvmclock`) | TSC is unreliable across migration and vCPU scheduling |
| **PV spinlocks / paravirt ticketlocks** | **Lock holder preemption**: a vCPU holding a spinlock gets descheduled and other vCPUs spin for a whole timeslice. PV locks `HLT`/yield instead of spinning, and the host wakes them |
| **PV TLB shootdown** | Do not IPI a vCPU that is not running; just mark it |
| **PV EOI** | Avoid an exit on every APIC end-of-interrupt |
| **PV sched yield** | Directed yield to the lock holder |
| **`virtio-balloon` / free page reporting** | Memory overcommit |
| **async page fault** | Host page fault on guest memory: tell the guest so it can schedule another task instead of stalling the vCPU |

**Lock holder preemption is the one to understand.** It is the core reason a
CPU-oversubscribed VM performs catastrophically rather than proportionally: with 2× vCPU
oversubscription you do not get 50% performance, you can get 10%, because spinlock hold
times become scheduler-quantum-sized. The mitigations are PV locks, **not** oversubscribing
latency-sensitive VMs, and CPU pinning.

---

## 1. Internals

### Source map

| Path | Contents |
|---|---|
| `virt/kvm/kvm_main.c` | Architecture-independent core: VM/vCPU creation, memslots, the `KVM_RUN` entry |
| `virt/kvm/eventfd.c` | `irqfd` and `ioeventfd` |
| `arch/x86/kvm/x86.c` | x86 vCPU run loop, exit dispatch, MSR/CPUID handling |
| `arch/x86/kvm/vmx/vmx.c` | Intel VT-x: VMCS setup, entry/exit, exit handlers |
| `arch/x86/kvm/vmx/nested.c` | Nested VMX — `vmcs12`/`vmcs02` merging |
| `arch/x86/kvm/svm/svm.c` | AMD SVM |
| `arch/x86/kvm/svm/sev.c` | SEV / SEV-ES / SEV-SNP |
| `arch/x86/kvm/mmu/` | EPT/NPT, shadow paging, the MMU page-fault path |
| `arch/x86/kvm/mmu/tdp_mmu.c` | The two-dimensional paging MMU (the modern, scalable one) |
| `arch/x86/kvm/lapic.c`, `ioapic.c`, `irq.c` | In-kernel irqchip |
| `arch/arm64/kvm/` | ARM: `hyp/` contains the EL2 hypervisor code, including pKVM |
| `drivers/vhost/` | `vhost.c`, `net.c`, `scsi.c`, `vdpa.c` — in-kernel virtio backends |
| `drivers/virtio/` | Guest-side virtio core and transports |
| `drivers/vfio/` | Device passthrough, IOMMU groups, `vfio-pci` |
| `include/uapi/linux/kvm.h` | **The KVM ABI** — read this to understand the API shape |
| `Documentation/virt/kvm/api.rst` | The authoritative API reference, very complete |

### The KVM API shape — an object lesson in fd-as-capability

```c
int kvm  = open("/dev/kvm", O_RDWR);              /* the subsystem       */
int vm   = ioctl(kvm, KVM_CREATE_VM, 0);          /* a VM                */
int vcpu = ioctl(vm,  KVM_CREATE_VCPU, 0);        /* a vCPU              */
struct kvm_run *run = mmap(NULL, run_size, PROT_READ|PROT_WRITE,
                           MAP_SHARED, vcpu, 0);  /* shared state page   */
```

Note the design (→ Ch. 24 §T.5): a hierarchy of file descriptors, each a capability, each
independently passable and automatically cleaned up on process exit. Killing QEMU tears down
the VM with no kernel-side reference leak. This is held up as one of the best-designed
interfaces in the kernel, and being able to say *why* — fd capabilities, an `mmap`'d shared
page to avoid per-exit copies, versioned extensions via `KVM_CHECK_EXTENSION` — is a strong
answer.

### Memory slots

```c
struct kvm_userspace_memory_region2 {
	__u32 slot;
	__u32 flags;              /* LOG_DIRTY_PAGES, READONLY, GUEST_MEMFD */
	__u64 guest_phys_addr;    /* GPA base                               */
	__u64 memory_size;
	__u64 userspace_addr;     /* HVA -- a normal userspace mapping!     */
	__u64 guest_memfd_offset; /* for confidential guests                */
	__u32 guest_memfd;
	...
};
```

The elegance: **guest physical memory is just a userspace mapping in QEMU.** That means all
of Linux's MM applies to it — you can back it with `hugetlbfs`, `memfd`, a file, THP, NUMA
policy, or `mbind` it. KVM composes GPA→HVA (memslots) with HVA→HPA (the host page tables)
to build the EPT. One design decision that inherits an entire subsystem.

`guest_memfd` is the newer addition for confidential VMs: memory that is **not** mappable by
userspace at all, because the host must not be able to read it.

### Exit reasons worth knowing

```c
/* include/uapi/linux/kvm.h */
#define KVM_EXIT_IO             2   /* PIO to a userspace device       */
#define KVM_EXIT_MMIO           6   /* MMIO to a userspace device      */
#define KVM_EXIT_HLT            5   /* guest idle -> host may sleep    */
#define KVM_EXIT_INTR           10  /* host signal; return to userspace*/
#define KVM_EXIT_SHUTDOWN       8   /* triple fault                    */
#define KVM_EXIT_INTERNAL_ERROR 17  /* KVM could not proceed           */
#define KVM_EXIT_HYPERCALL      3
```

And the in-kernel-only reasons (never seen by userspace) — EPT violation, CPUID, MSR access,
CR access, external interrupt. Those are the common ones, and the fact that they never reach
userspace is the whole performance story.

### Observability surface

```bash
# Is virtualization available and what does it support?
lscpu | grep -E 'Virtualization|Hypervisor'
ls /sys/module/kvm_intel/parameters/    # nested, ept, unrestricted_guest, ...
cat /sys/module/kvm_intel/parameters/nested

# Per-VM exit statistics -- the single most useful virtualization metric.
ls /sys/kernel/debug/kvm/*/          # per-VM directories
grep . /sys/kernel/debug/kvm/*/vcpu0/* 2>/dev/null | head -40
# Look at: exits, mmio_exits, io_exits, halt_exits, irq_injections,
#          nested_run, tlb_flush, pf_taken, pf_fixed

# Live exit breakdown -- the tool to reach for first:
sudo perf kvm stat live
sudo perf kvm --host --guest top

# Trace every exit with its reason:
sudo trace-cmd record -e kvm:kvm_exit -e kvm:kvm_entry -e kvm:kvm_mmio
sudo trace-cmd report | head -50

# Guest-side: am I a guest, and of what?
systemd-detect-virt
dmesg | grep -iE 'hypervisor|kvm|virtio|paravirt'
cat /sys/hypervisor/type 2>/dev/null
```

`perf kvm stat` is the tool to name in an interview: it gives you a histogram of exit
reasons with counts and time, which immediately tells you whether a VM is slow because of
MMIO (bad device model), EPT violations (memory setup), or halt exits (idle, fine).

---

## 2. Practice

### Lab 104.1 — Write a hypervisor in 200 lines

The best possible way to understand KVM. This runs real 16-bit x86 code in a VM.

```c
/* tinyvm.c — a complete KVM hypervisor.
 * Build: gcc -O2 -Wall -o tinyvm tinyvm.c
 * Run:   ./tinyvm            (needs read/write on /dev/kvm; add yourself to the kvm group)
 *
 * The guest is 16-bit real-mode code that adds two numbers and writes the
 * result to an I/O port, which exits to us.                              */
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <linux/kvm.h>

/* Guest code, 16-bit real mode:
 *   mov al, [0x100]   ; load a
 *   add al, [0x101]   ; add b
 *   out 0x10, al      ; -> VM exit, we see the result
 *   hlt               ; -> VM exit, we stop                       */
static const unsigned char guest_code[] = {
	0xa0, 0x00, 0x01,       /* mov al, [0x100]      */
	0x02, 0x06, 0x01, 0x01, /* add al, [0x101]      */
	0xe6, 0x10,             /* out 0x10, al         */
	0xf4,                   /* hlt                  */
};

#define MEM_SIZE (1 << 20)   /* 1 MiB of guest physical memory */

int main(void)
{
	int kvm, vmfd, vcpufd, ret;
	unsigned char *mem;
	struct kvm_sregs sregs;
	struct kvm_regs regs;
	struct kvm_run *run;
	struct kvm_userspace_memory_region region;
	size_t run_size;
	long exits = 0;

	/* 1. Open the subsystem and sanity-check the API version. */
	kvm = open("/dev/kvm", O_RDWR | O_CLOEXEC);
	if (kvm < 0) { perror("/dev/kvm"); return 1; }
	if (ioctl(kvm, KVM_GET_API_VERSION, NULL) != 12) {
		fprintf(stderr, "unexpected KVM API version\n"); return 1;
	}

	/* 2. Create a VM. This is just an fd. */
	vmfd = ioctl(kvm, KVM_CREATE_VM, (unsigned long)0);
	if (vmfd < 0) { perror("KVM_CREATE_VM"); return 1; }

	/* 3. Allocate guest physical memory -- an ordinary anonymous mapping.
	 *    This is the key insight: guest RAM is just our memory.         */
	mem = mmap(NULL, MEM_SIZE, PROT_READ | PROT_WRITE,
		   MAP_SHARED | MAP_ANONYMOUS, -1, 0);
	if (mem == MAP_FAILED) { perror("mmap"); return 1; }

	memcpy(mem + 0x1000, guest_code, sizeof(guest_code));
	mem[0x100] = 40;   /* a */
	mem[0x101] = 2;    /* b */

	/* 4. Tell KVM: GPA 0 .. MEM_SIZE maps to this HVA. */
	memset(&region, 0, sizeof(region));
	region.slot            = 0;
	region.guest_phys_addr = 0;
	region.memory_size     = MEM_SIZE;
	region.userspace_addr  = (unsigned long)mem;
	if (ioctl(vmfd, KVM_SET_USER_MEMORY_REGION, &region) < 0) {
		perror("KVM_SET_USER_MEMORY_REGION"); return 1;
	}

	/* 5. Create a vCPU and mmap its shared kvm_run page. */
	vcpufd = ioctl(vmfd, KVM_CREATE_VCPU, (unsigned long)0);
	if (vcpufd < 0) { perror("KVM_CREATE_VCPU"); return 1; }

	run_size = ioctl(kvm, KVM_GET_VCPU_MMAP_SIZE, NULL);
	run = mmap(NULL, run_size, PROT_READ | PROT_WRITE, MAP_SHARED, vcpufd, 0);
	if (run == MAP_FAILED) { perror("mmap kvm_run"); return 1; }

	/* 6. Set up real mode: CS base 0, IP 0x1000. */
	if (ioctl(vcpufd, KVM_GET_SREGS, &sregs) < 0) { perror("GET_SREGS"); return 1; }
	sregs.cs.base     = 0;
	sregs.cs.selector = 0;
	if (ioctl(vcpufd, KVM_SET_SREGS, &sregs) < 0) { perror("SET_SREGS"); return 1; }

	memset(&regs, 0, sizeof(regs));
	regs.rip    = 0x1000;
	regs.rflags = 0x2;          /* bit 1 is reserved-and-must-be-1 */
	if (ioctl(vcpufd, KVM_SET_REGS, &regs) < 0) { perror("SET_REGS"); return 1; }

	/* 7. THE RUN LOOP. This is the whole of a hypervisor. */
	for (;;) {
		ret = ioctl(vcpufd, KVM_RUN, NULL);
		if (ret < 0) {
			if (errno == EINTR) continue;
			perror("KVM_RUN");
			return 1;
		}
		exits++;

		switch (run->exit_reason) {
		case KVM_EXIT_IO:
			if (run->io.direction == KVM_EXIT_IO_OUT &&
			    run->io.port == 0x10) {
				unsigned char v =
					*((char *)run + run->io.data_offset);
				printf("guest computed: %u  (expected 42)\n", v);
			} else {
				printf("unhandled IO port 0x%x\n", run->io.port);
			}
			break;

		case KVM_EXIT_HLT:
			printf("guest halted after %ld userspace exits\n", exits);
			return 0;

		case KVM_EXIT_MMIO:
			printf("MMIO %s at 0x%llx len %u\n",
			       run->mmio.is_write ? "write" : "read",
			       run->mmio.phys_addr, run->mmio.len);
			break;

		case KVM_EXIT_FAIL_ENTRY:
			fprintf(stderr, "entry failed, reason 0x%llx\n",
			  run->fail_entry.hardware_entry_failure_reason);
			return 1;

		case KVM_EXIT_INTERNAL_ERROR:
			fprintf(stderr, "internal error, suberror %u\n",
				run->internal.suberror);
			return 1;

		default:
			fprintf(stderr, "exit_reason = %u\n", run->exit_reason);
			return 1;
		}
	}
}
```

Extend it: add a second vCPU (one thread each), add an MMIO region and implement a device,
add `KVM_EXIT_IO` on port `0x3f8` to implement a serial console, then load a real Linux
`bzImage`. Each step teaches more than a chapter of reading. **Only two userspace exits
occur in the program above** — count them, then reason about what a real guest's exit count
would look like.

### Lab 104.2 — Measure the exit cost

```bash
# 1. Start a guest and find its debugfs directory.
qemu-system-x86_64 -enable-kvm -m 2G -smp 2 -nographic \
	-kernel /boot/vmlinuz-$(uname -r) -append "console=ttyS0" &

VM=$(ls -d /sys/kernel/debug/kvm/*/ | head -1)
echo "VM stats at $VM"

# 2. Exit breakdown, sorted.
grep . $VM/vcpu0/* 2>/dev/null | sed 's|.*/||' | sort -t: -k2 -rn | head -20

# 3. Live breakdown by reason -- the tool to use first.
sudo perf kvm stat live -d 10

# 4. Exit latency histogram from tracepoints:
sudo bpftrace -e '
tracepoint:kvm:kvm_exit { @t[tid] = nsecs; @reason[args->exit_reason] = count(); }
tracepoint:kvm:kvm_entry /@t[tid]/ { @ns = hist(nsecs - @t[tid]); delete(@t[tid]); }
interval:s:15 { exit(); }'
```

Then compare device models directly:

```bash
# Emulated e1000 vs virtio-net vs vhost-net. Run iperf3 in each and compare
# throughput AND the exit count from /sys/kernel/debug/kvm/*/vcpu*/exits.
qemu-system-x86_64 ... -netdev user,id=n0 -device e1000,netdev=n0
qemu-system-x86_64 ... -netdev user,id=n0 -device virtio-net-pci,netdev=n0
qemu-system-x86_64 ... -netdev tap,id=n0,vhost=on -device virtio-net-pci,netdev=n0
```

**The expected result is the lesson:** e1000 does an exit per register access and tops out
around 1 Gbps with high CPU; virtio does one exit per batch; vhost-net removes the QEMU
round-trip entirely and approaches line rate. You are measuring §T.5's table yourself.

### Lab 104.3 — Observe lock holder preemption

```bash
# 1. A VM with MORE vCPUs than the host has physical CPUs.
HOST_CPUS=$(nproc)
qemu-system-x86_64 -enable-kvm -m 4G -smp $((HOST_CPUS * 2)) ...

# 2. Inside the guest, run a lock-heavy workload:
#    (hackbench hammers the scheduler and its locks)
hackbench -l 10000 -g 20

# 3. Compare with -smp $HOST_CPUS. The degradation is superlinear.

# 4. Now check whether PV spinlocks are in use:
#    guest:
dmesg | grep -i 'paravirtual\|pv-spinlock\|KVM setup pv spinlock'
cat /sys/module/kvm/parameters/* 2>/dev/null
#    Disable them to see the difference:
#    boot the guest with  no-kvmapf nopvspin
```

Record throughput at 1×, 1.5×, and 2× oversubscription, with and without `nopvspin`. The
result — that 2× oversubscription can cost far more than 2× — is one of the most
practically important facts in virtualization capacity planning, and it is the direct
consequence of §T.10.

### Lab 104.4 — Device passthrough with VFIO

```bash
# DANGER: this unbinds a real device from its host driver. Use a spare NIC,
# and be sure it is not your only network path.

DEV=0000:03:00.0

# 1. IOMMU must be on. Check the kernel cmdline: intel_iommu=on / amd_iommu=on
dmesg | grep -i -e DMAR -e IOMMU | head

# 2. IOMMU GROUPS are the unit of isolation -- you pass through a whole
#    group or nothing, because devices in a group can DMA to each other
#    behind the IOMMU. This is the concept to understand.
for g in /sys/kernel/iommu_groups/*/devices/*; do
	echo "group $(basename $(dirname $(dirname $g))): $(basename $g)"
done | sort -V

# 3. Unbind from the host driver, bind to vfio-pci.
echo "$DEV" | sudo tee /sys/bus/pci/devices/$DEV/driver/unbind
VD=$(cat /sys/bus/pci/devices/$DEV/vendor) 
DD=$(cat /sys/bus/pci/devices/$DEV/device)
echo "${VD#0x} ${DD#0x}" | sudo tee /sys/bus/pci/drivers/vfio-pci/new_id

# 4. Give it to a guest.
sudo qemu-system-x86_64 -enable-kvm -m 4G -smp 4 \
	-device vfio-pci,host=$DEV ...

# 5. Inside the guest, the device appears as real hardware.
#    Measure throughput/latency vs virtio. Then try to live-migrate,
#    and observe that you cannot -- that is the tradeoff made concrete.
```

The ACS (Access Control Services) question is the real lesson: if the upstream PCIe switch
does not enforce ACS, all devices below it are in one IOMMU group, because peer-to-peer DMA
between them bypasses the IOMMU. The infamous `pcie_acs_override` patch lets you lie about
this — and understanding *why that is unsafe* is exactly the kind of judgement question that
distinguishes levels.

### Lab 104.5 — Build a microVM and measure boot time

```bash
# Firecracker-style minimal VM with QEMU's microvm machine type.
# No PCI, no ACPI, no legacy devices -- just virtio-mmio.

qemu-system-x86_64 \
	-M microvm,x-option-roms=off,pit=off,pic=off,rtc=off,isa-serial=off \
	-enable-kvm -cpu host -m 512M -smp 1 \
	-kernel vmlinux \
	-append "console=hvc0 root=/dev/vda rw acpi=off reboot=t panic=-1 \
	         tsc=reliable no_timer_check noreplace-smp quiet" \
	-nodefaults -no-user-config -nographic \
	-chardev stdio,id=c0 -device virtio-serial-device \
	-device virtconsole,chardev=c0 \
	-drive id=root,file=rootfs.ext4,format=raw,if=none \
	-device virtio-blk-device,drive=root

# Measure boot time precisely:
#   guest: systemd-analyze  (or add `initcall_debug` and read dmesg timestamps)
#   host:  time until the guest writes its first byte to the console
```

Then strip further and measure the effect of each: `acpi=off`, removing the PIT/PIC/RTC,
`CONFIG_` reduction in the guest kernel, and `-cpu host` versus an emulated model.
Firecracker reaches ~125 ms; see how close you get and where your remaining time goes
(`initcall_debug` in the guest tells you exactly). The point of this lab is §T.9: you are
measuring whether a VM boundary is economically viable for the workload in question.

### Lab 104.6 — Nested virtualization

```bash
# 1. Enable on the host.
cat /sys/module/kvm_intel/parameters/nested     # want Y
# If N: modprobe -r kvm_intel; modprobe kvm_intel nested=1

# 2. L1 must see the VMX feature -> use -cpu host (or add vmx explicitly).
qemu-system-x86_64 -enable-kvm -cpu host -m 8G -smp 4 ...   # this is L1

# 3. Inside L1, confirm and run L2.
lscpu | grep Virtualization
qemu-system-x86_64 -enable-kvm -m 2G ...                    # this is L2

# 4. Measure the tax: run the same benchmark in L1 and L2.
#    Then, from L0, watch the nested exit counters:
grep nested /sys/kernel/debug/kvm/*/vcpu0/*
```

Expect a substantial slowdown for exit-heavy workloads and a small one for compute-bound
ones. That difference — nested virtualization taxes *exits*, not *instructions* — is the
insight, and it explains why nested CI runners are fine and nested I/O-heavy workloads are
not.

---

## 3. Mastery drills

1. Extend `tinyvm.c` to boot a real Linux kernel: implement enough of a serial console, a
   boot protocol (`linux,boot-params`), and `virtio-mmio` to reach a shell. Count the
   userspace exits from boot to login prompt and categorize them.

2. Implement a `virtio-blk` device in `tinyvm.c`: parse the virtqueue, service descriptors,
   write the used ring, and inject an interrupt via `irqfd`. Then measure the exits-per-IO
   and compare against `vhost-blk`.

3. Instrument the KVM MMU: use tracepoints to count EPT violations during a guest boot and a
   guest workload. Then back the guest with 2 MiB hugepages and re-measure. Explain the
   difference quantitatively in terms of page-walk depth.

4. Build a table of VM exit cost by reason on your hardware — CPUID, MSR read, MSR write,
   EPT violation, MMIO to in-kernel device, MMIO to userspace device, HLT. Use
   `bpftrace` on `kvm_exit`/`kvm_entry`. Explain the ordering.

5. Take a real workload (a database, a web server) and profile it in a VM versus bare metal.
   Attribute every percentage point of the difference to a specific mechanism: exits, EPT
   walk depth, interrupt delivery, TLB pressure, or scheduling.

6. Configure a VM with CPU pinning, hugepage backing, `isolcpus` on the host for the vCPU
   threads, and a `SCHED_FIFO` vCPU priority. Measure `cyclictest` **inside** the guest
   (→ Ch. 103) and determine how close a VM can get to bare-metal RT latency, and what the
   irreducible floor is.

7. Study the `vhost-net` code path end to end: a packet arriving on a physical NIC to it
   appearing in the guest. Identify every context switch, every copy, and every interrupt.
   Then do the same for `vhost-user` with DPDK and explain where the differences come from.

8. Implement a minimal vDPA-style split: a control plane in userspace and a datapath that
   bypasses it. Explain what makes live migration possible in vDPA but not in VFIO
   passthrough.

9. Set up an SEV or TDX guest (or read the code if you lack the hardware). Enumerate every
   place the guest kernel must stop trusting the host, and estimate the auditing effort for
   one subsystem of your choice.

10. Compare gVisor, Kata Containers, and Firecracker on the same workload: startup time,
    memory overhead, syscall-heavy throughput, I/O throughput, and host attack surface
    (count the host syscalls each can reach). Produce the decision matrix.

11. Reproduce lock holder preemption at increasing oversubscription ratios and plot
    throughput. Find the knee. Then repeat with PV spinlocks disabled and with vCPU pinning,
    and explain all three curves.

12. Trace a nested exit end to end: L2 executes `CPUID`, and you follow it through L0's
    handling, the decision to reflect to L1, L1's handling, and the re-entry. Count the total
    cycles and compare to the non-nested case.

13. Design the virtualization architecture for a multi-tenant FaaS platform: 10,000
    invocations/sec, 100 ms average duration, hostile tenant code, sub-200 ms cold start.
    State every choice — isolation boundary, device model, memory strategy, snapshot/restore,
    CPU allocation — and price each one.

---

## 4. Further reading

**Kernel documentation**
- `Documentation/virt/kvm/api.rst` — the complete KVM ABI; unusually good documentation
- `Documentation/virt/kvm/locking.rst` — the KVM locking hierarchy
- `Documentation/virt/kvm/x86/mmu.rst` — the shadow/EPT MMU explained by its authors
- `Documentation/virt/coco/` — confidential computing
- `Documentation/driver-api/vfio.rst` — IOMMU groups and passthrough
- The virtio specification (OASIS) — read the virtqueue chapter at minimum

**Papers**
- Popek & Goldberg, "Formal Requirements for Virtualizable Third Generation Architectures,"
  *CACM*, 1974 — the criterion
- Bugnion, Devine, Rosenblum, et al., "Bringing Virtualization to the x86 Architecture with
  the Original VMware Workstation," *TOCS*, 2012 — how binary translation actually worked
- Barham et al., "Xen and the Art of Virtualization," *SOSP*, 2003 — paravirtualization
- Kivity et al., "kvm: the Linux Virtual Machine Monitor," *Linux Symposium*, 2007 — the
  original KVM paper; short and very readable
- Ben-Yehuda et al., "The Turtles Project: Design and Implementation of Nested
  Virtualization," *OSDI*, 2010
- Russell, "virtio: Towards a De-Facto Standard for Virtual I/O Devices," *OSR*, 2008
- Agache et al., "Firecracker: Lightweight Virtualization for Serverless Applications,"
  *NSDI*, 2020 — **read this one**; it is the clearest statement of the modern isolation
  tradeoff
- Young et al., "The True Cost of Containing: A gVisor Case Study," *HotCloud*, 2019
- Li et al., "A Comparison of Confidential Computing Technologies" and the SEV-SNP /
  TDX whitepapers from AMD and Intel

**LWN**
- "KVM" tag archive, particularly the KVM Forum coverage each year
- "The Firecracker virtual machine monitor"
- "Confidential computing" series
- "vDPA: a new virtio device type" and follow-ups
- "Nested virtualization" and the nested-VMX merge coverage

**Books**
- Bugnion, Nieh, Tsafrir, *Hardware and Software Support for Virtualization* (Synthesis
  Lectures, 2017) — the best single book on this material, concise and current
- Intel SDM Volume 3C, chapters 23–33 — the VMX specification itself; dense but definitive
- AMD APM Volume 2, chapter 15 — SVM

**Tools**
- `kvm_stat` — curses-mode live exit statistics
- `perf kvm stat` / `perf kvm top` — host+guest profiling with symbol resolution
- `virsh`, `virt-manager`, `libvirt` — management
- `cloud-hypervisor`, `firecracker` — read their source; both are far smaller than QEMU and
  therefore much more readable as reference VMMs
- `kvmtool` (`lkvm`) — a ~10k-line VMM maintained in the kernel community; the best
  intermediate step between `tinyvm.c` and QEMU

→ Next: [105-numa-scalability.md](105-numa-scalability.md)

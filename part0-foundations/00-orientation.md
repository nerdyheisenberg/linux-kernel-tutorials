# Module 00 — Orientation: What the Linux Kernel Actually Is

> **Goal:** by the end of this module you can draw the entire Linux system on a whiteboard
> from memory, and explain every arrow.

---

## Theory & First Principles

> **How to read this section.** T.0–T.2 assume nothing and build the model from the hardware
> up. T.3–T.7 are the working engineer's model — the costs, the trade-offs, the history you
> are expected to know. T.8–T.11 are the architect's model: the arguments that are still
> live, the organizational forces, and the compressed mental model to carry with you.

---

### T.0 — Start here: what happens when you type `ls`

Before any theory, follow one concrete thing all the way down. Everything in this chapter is
an abstraction of this sequence.

```
 you press Enter in bash
   │
 ① bash calls fork()        -> a TRAP into the kernel. CPU switches to privileged mode.
   │                            Kernel creates a second process: a copy of bash.
   ▼
 ② child calls execve("/bin/ls")   -> another TRAP.
   │      Kernel: opens the file, reads the ELF header, throws away the old
   │      address space, maps /bin/ls's code and data, maps the dynamic
   │      linker, sets the instruction pointer, RETURNS to userspace.
   ▼
 ③ /lib/ld-linux.so runs first (not ls!). It maps libc, resolves symbols.
   │      Each mapping is an mmap() TRAP.
   ▼
 ④ ls finally runs main(). It calls opendir(".")
   │      -> libc issues openat(AT_FDCWD, ".", O_RDONLY|O_DIRECTORY)  -> TRAP
   │      Kernel: walks the path through the VFS, finds the inode via the
   │      filesystem driver, which may issue block I/O, which goes to a
   │      device driver, which pokes hardware and then SLEEPS.
   │      The scheduler runs something else. An interrupt eventually fires.
   ▼
 ⑤ getdents64() in a loop -> TRAP each time. write(1, ...) -> TRAP.
   ▼
 ⑥ exit_group() -> TRAP. Kernel tears down the address space, reparents
        children, wakes up the parent blocked in wait4().
```

Count the transitions: a bare `ls` in an empty directory makes **roughly 60–120 syscalls**.
Verify it yourself right now:

```bash
strace -c ls /tmp            # summary by syscall
strace -f ls /tmp 2>&1 | wc -l
```

**Four observations that this chapter exists to explain:**

1. **The kernel never "runs" on its own.** Every one of those steps is the kernel being
   *entered* — by a trap, an interrupt, or an exception — doing work on behalf of someone,
   and leaving. There is no kernel main loop. (§T.1, §T.2)
2. **Every entry costs.** At ~300 ns per syscall with mitigations, 100 syscalls is 30 µs of
   pure boundary crossing. For `ls` that is irrelevant. For a database doing 1M ops/sec it is
   the entire budget. (§T.3)
3. **The kernel did wildly different things** — process creation, ELF parsing, path
   resolution, block I/O, scheduling, interrupt handling — through *one* interface. That
   uniformity is a deliberate design choice, not an accident. (§T.4)
4. **At step ④ the kernel put the process to sleep and ran something else.** That single
   ability — suspend a computation, run another, resume the first with nothing observing the
   gap — is the hardest and most valuable thing an OS does. (§T.1)

If you want one exercise before reading further: run `strace -f` on a program you wrote and
account for every syscall. The gap between what you thought your program did and what it
actually did is the gap this curriculum closes.

---

### T.1 What is an operating system, formally?

Strip away everything Linux-specific. An OS exists to solve exactly three problems:

1. **Multiplexing** — one set of physical resources (CPU, RAM, disks, NICs) must be shared by
   many mutually distrusting programs. The OS is a *time-* and *space-division multiplexer*.
2. **Abstraction** — raw hardware interfaces are unusable (a disk is a set of LBAs; you want
   files). The OS provides *virtual* resources: virtual CPUs (threads), virtual memory
   (address spaces), virtual disks (files), virtual networks (sockets).
3. **Protection** — the multiplexing must be enforceable against a hostile participant.
   Without hardware support this is impossible; with it, it reduces to a small
   **reference monitor** that mediates every access.

Everything else — schedulers, filesystems, the network stack — is an *implementation
technique* for one of these three. When you meet an unfamiliar subsystem, ask which of the
three it serves, and you immediately have a frame for it.

**Work the frame on examples**, because it is only useful if it is automatic:

| Subsystem | Primary | Secondary |
|---|---|---|
| Scheduler | multiplexing (CPU, in time) | — |
| Virtual memory | multiplexing (RAM, in space) | abstraction (a flat address space) |
| VFS | abstraction (one API over 60 filesystems) | — |
| Page cache | abstraction (memory *is* the disk) | multiplexing (shared between processes) |
| Namespaces | protection (visibility) | — |
| cgroups | multiplexing (with explicit shares) | — |
| Device drivers | abstraction (a NIC becomes `send()`) | — |
| seccomp/LSM | protection | — |
| `io_uring` | **none of the three** — it is a *cost* optimization on the boundary itself |

That last row is the interesting one. When a subsystem does not fit the three-problem frame,
it is almost always there because of the cost model of §T.3 — and noticing that is the first
genuinely architectural observation in this curriculum.

**The virtualization insight, stated once properly.** Each abstraction is a *virtual* version
of a physical resource, and each is sold on the same promise: *you may pretend you have the
whole machine.*

| Physical | Virtual | The lie |
|---|---|---|
| 8 cores | unlimited threads | "you have a CPU to yourself" |
| 16 GiB RAM | 256 TiB address space per process | "you have more memory than exists" |
| A disk of 512-byte sectors | files that grow | "storage is a named, resizable byte array" |
| A NIC with rings and descriptors | a socket | "the network is a reliable byte stream" |

Every one of these lies is *leaky*, and every leak is a chapter in this curriculum. Threads
leak through scheduling latency (Ch. 21) and cache pollution (Ch. 105). Virtual memory leaks
through page-fault cost (Ch. 22) and the OOM killer (Ch. 23). Files leak through `fsync`
semantics (Ch. 53) and `ENOSPC`. Sockets leak through `EAGAIN`, Nagle, and head-of-line
blocking (Ch. 72). **Mastery is knowing exactly where each abstraction stops being true.**

Three classical design principles apply throughout, and you should be able to cite them:

- **Separation of policy and mechanism** (Levin, Cohen, Corwin, Pollack & Wulf, *Hydra*,
  SOSP 1975). The kernel provides *mechanism* ("you can set a scheduling priority"),
  userspace chooses *policy* ("web servers get priority 10"). Linux violates this
  deliberately in places (the scheduler has a built-in policy) and honours it religiously in
  others (LSM, cgroups, `sched_ext`). Every argument about "should this be in the kernel?"
  is really an argument about this principle.
- **The end-to-end argument** (Saltzer, Reed & Clark, TOCS 1984). A function is best placed
  at the endpoints; lower layers should only implement it as a performance optimization.
  This is why the kernel does not do application-level retry, encryption of your data, or
  transaction semantics — and why it *does* do TCP checksums (as an optimization, with
  end-to-end checks still expected above).
- **Lampson's Hints** (SOSP 1983). "Do one thing well", "don't hide power", "use hints",
  "split resources rather than sharing", "make actions atomic or restartable", "end-to-end".
  Read this paper; it is nine pages and it is the distilled wisdom of the field.

### T.2 The dual-mode model: protection reduced to one hardware bit

Protection in every mainstream OS rests on a single hardware primitive: the CPU has a
**privilege level**, and certain operations are legal only at the privileged level.

| Arch | Privileged | Unprivileged | Entry instruction | Return |
|---|---|---|---|---|
| x86-64 | ring 0 | ring 3 | `syscall` | `sysret`/`iret` |
| ARM64 | EL1 | EL0 | `svc #0` | `eret` |
| RISC-V | S-mode | U-mode | `ecall` | `sret` |
| (virt) | EL2/root/HS | — | `hvc`/`vmcall` | — |

What "privileged" actually gates, in every architecture:

- Writing the page-table base register (`CR3`, `TTBR0/1_EL1`, `satp`) — **this is the one
  that matters**, because whoever controls the page tables controls all memory.
- Enabling/disabling interrupts.
- Executing I/O instructions / accessing device MMIO regions.
- Modifying the interrupt vector table, the privilege state itself, and cache/TLB control.

Given those four, protection is *complete*: an unprivileged program cannot name memory it was
not given, cannot ignore preemption, and cannot talk to hardware. That is the entire security
model, and everything else (users, capabilities, namespaces, SELinux) is built on top of it in
software.

**Why "cannot name memory it was not given" is the load-bearing one.** Suppose only that
single guarantee held and nothing else. Then:

- A process cannot read another's secrets, because it cannot construct an address that refers
  to them — the MMU translates only what the page tables map.
- A process cannot corrupt the kernel, for the same reason: kernel pages are mapped with the
  supervisor bit set, and a user-mode access to them faults.
- A process cannot forge the page tables, because `CR3`/`TTBR` is privileged.

Everything else is defence in depth. **Memory protection is not one of several security
mechanisms; it is the one on which the others rest.** This is why Meltdown (§T.3) was
catastrophic rather than merely bad: it broke the base case, and every layer above it
inherited the break.

**The trap is the only door.** A `syscall` instruction does not "call a function". It:
1. Switches privilege level atomically with the jump (so you cannot land at an arbitrary
   address with privilege).
2. Jumps to a **fixed, kernel-controlled entry point** (`MSR_LSTAR` on x86-64,
   `VBAR_EL1` on ARM64).
3. Switches the stack pointer to a kernel stack (via `swapgs`/`SP_EL1`).

Step 2 is the crux. The kernel chooses the entry point; userspace chooses only the *arguments*.

**Work through why each of the three steps is necessary**, because the necessity is not
obvious and the attacks that motivated them are real:

| If you omitted… | The attack |
|---|---|
| **(1) atomicity** of privilege-raise and jump | Raise privilege, then be interrupted and resumed at an attacker-chosen address. Privileged execution of arbitrary code |
| **(2) the fixed entry point** | Enter the kernel *in the middle* of a function, past its validation. This is exactly why `sysenter`-style interfaces take a syscall *number* and not an *address* |
| **(3) the stack switch** | The kernel would run on a user-controlled stack pointer. Every `push` corrupts memory the attacker chose, and every `ret` jumps where the attacker wrote. Total compromise |

Step 3 also explains a piece of x86 arcana you will meet: `swapgs`. On entry the kernel needs
a per-CPU pointer to find its stack — but it cannot use a register (userspace controls them
all) or a fixed address (there are many CPUs). So `GS` is set to a per-CPU area and swapped
atomically on the transition. Forgetting a `swapgs` on some path has been a real CVE more than
once, which tells you how delicate this boundary is even for the people who own it.

**The reference-monitor criteria** (Anderson, 1972) — a reference monitor must be:

| Criterion | Linux |
|---|---|
| **Tamper-proof** | ✅ hardware-enforced by the privilege bit |
| **Always invoked** | ✅ there is no path into kernel memory except a trap |
| **Small enough to verify** | ❌ **~40 million lines** |

Linux fails the third criterion spectacularly, and that failure is the origin of most of
Part 8. If you cannot verify the monitor, you must (a) shrink what is reachable — seccomp
(Ch. 102 §T.5); (b) add a second monitor inside — LSM (Ch. 102 §T.4); (c) make the code
memory-safe — Rust (Part 5); or (d) put a *smaller* monitor underneath — a hypervisor
(Ch. 104). Those four responses are not independent ideas; they are four answers to one
failed criterion, and seeing them that way is worth more than knowing any of them
individually.

### T.2b — A worked trap: `getpid()` to the instruction

The smallest possible syscall, end to end. Everything else is this plus work.

```
 USERSPACE                                    │ KERNEL
                                              │
 mov  eax, 39          ; __NR_getpid          │
 syscall               ; ──────────────────────┐
                                              ││ hardware, atomically:
                                              ││  - save RIP  -> RCX
                                              ││  - save RFLAGS -> R11
                                              ││  - mask interrupts per MSR_SFMASK
                                              ││  - load RIP from MSR_LSTAR
                                              ││  - switch CS/SS to ring 0
                                              │▼
                                              │ entry_SYSCALL_64:
                                              │   swapgs                ; per-CPU base
                                              │   mov %rsp, PER_CPU(rsp_scratch)
                                              │   mov PER_CPU(cpu_current_top_of_stack), %rsp
                                              │   ; build struct pt_regs by pushing registers
                                              │   ; (this IS the "saved user context")
                                              │   call do_syscall_64
                                              │     -> if (nr < NR_syscalls)
                                              │          regs->ax = sys_call_table[nr](regs);
                                              │        else regs->ax = -ENOSYS;
                                              │        ^ bounds check. Miss it and you have
                                              │          an arbitrary-call primitive.
                                              │   ; restore registers from pt_regs
                                              │   swapgs
                                              │   sysretq  ─────────────┐
                                              │                          │
 ; execution resumes here, RAX = pid         │◄─────────────────────────┘
```

Four things to extract, each of which recurs for the rest of the curriculum:

1. **`struct pt_regs` is the saved user context.** Signal delivery rewrites it (Ch. 20),
   `ptrace` reads and modifies it (Ch. 06), and a core dump serializes it (Ch. 95). It is not
   an implementation detail — it is a first-class object.
2. **The syscall number is an index into a table, and the bounds check is the entire
   security of that dispatch.** Ch. 24 §T.2.
3. **Return values are `-errno` in a register.** There is no exception mechanism, no
   `errno` global in the kernel; libc is what turns `-2` into `errno = ENOENT` and a `-1`
   return. That convention propagates into `ERR_PTR` (Ch. 08) and into Rust's `Result`
   (Ch. 79).
4. **The mitigations bolt on here.** KPTI adds a `CR3` write on both edges; Spectre
   mitigations add `IBPB`/`VERW`. That is *why* the number in §T.3 went from 60 ns to 300 ns —
   not because the kernel got slower, but because this transition got more expensive. Watch
   for it in Ch. 102 Lab 2, where you measure it yourself.

This is a **narrow, total interface** — narrow because ~450 entry points is small relative to
40 M lines, total because there is no other way in. The tension between those two facts is the
subject of Ch. 24.

### T.3 Cost model: why the boundary shapes every design

Numbers you should have memorized, because every architectural argument eventually bottoms
out in them (modern server-class x86-64, order of magnitude):

| Operation | Cost |
|---|---|
| Function call | ~1 ns |
| L1 cache hit | ~1 ns |
| Atomic RMW, uncontended | ~10 ns |
| **Syscall (no KPTI)** | **~60–100 ns** |
| **Syscall (with KPTI/Meltdown mitigation)** | **~200–500 ns** |
| Main memory access | ~80 ns |
| Contended cache line transfer | ~100–300 ns |
| **Context switch (same address space)** | **~1–2 µs** |
| **Context switch (different address space)** | **~2–5 µs** (TLB/cache pollution dominates) |
| Interrupt entry+exit | ~1–5 µs total system cost |
| NVMe read (4 KiB) | ~80 µs |
| Network RTT, same datacenter | ~100 µs |

Three design consequences follow *directly*, and they explain most of modern Linux:

1. **The syscall is not free, so batch it.** This is why `io_uring`, `epoll`, `sendmmsg`,
   `readv`/`writev`, and `futex` exist. `futex` is the canonical example: take the lock in
   userspace with an atomic; enter the kernel *only* when contended. Uncontended locking
   costs 10 ns instead of 100 ns.
2. **The kernel is mapped into every address space** (top half of the virtual address space)
   so a syscall is a *mode switch*, not an *address-space switch*. This is a ~30× saving,
   and it is exactly what Meltdown broke — KPTI reintroduces the address-space switch
   and costs 2–5× on syscall-heavy workloads.
3. **Copying is expensive relative to everything else**, hence zero-copy: `sendfile`,
   `splice`, `mmap`, `MSG_ZEROCOPY`, DMA straight into page cache, and the entire design of
   `io_uring`'s registered buffers.

#### Using the table as a reasoning tool

The numbers are not trivia. Here is how a senior engineer actually uses them — normalize
everything to the cheapest operation and the conclusions become forced.

**Scale them to human time** (1 ns → 1 second):

| Operation | Real | Human scale |
|---|---|---|
| L1 hit | 1 ns | 1 second |
| Main memory | 80 ns | 1.5 minutes |
| Syscall (mitigated) | 300 ns | **5 minutes** |
| Context switch | 3 µs | **50 minutes** |
| NVMe read | 80 µs | **22 hours** |
| Datacenter RTT | 100 µs | 27 hours |
| HDD seek | 10 ms | **4 months** |

Now the design rules stop being memorized and become obvious. Doing a syscall to save a
memory access is like driving five minutes to avoid a 1.5-minute walk. Doing a context switch
to avoid spinning for 100 ns is like sleeping for an hour to avoid waiting a minute — which is
exactly the ski-rental calculation behind adaptive mutexes (Ch. 14 §T.5).

**Derive a budget, which is the actual skill.** Suppose a requirement of **1 million
operations per second on one core**:

```
 1 core-second / 1,000,000 ops  =  1000 ns per operation, TOTAL
```

Now subtract:

| Item | Cost | Remaining |
|---|---|---|
| Budget | | 1000 ns |
| One syscall (mitigated) | −300 ns | 700 ns |
| One context switch (if you block) | −3000 ns | **−2300 ns ✗ impossible** |

**The arithmetic, not taste, tells you the design:** at 1 M ops/sec you may afford roughly one
syscall per operation and you may not block. At 10 M ops/sec (Ch. 76, `system-design.md` §2)
you get 100 ns and may not syscall at all — hence shared-memory rings. You have just derived
`io_uring` from a division.

**Do this in interviews.** "What's the requirement? 10 M IOPS? That's 100 ns per I/O, and a
syscall is 300, so a per-I/O syscall is arithmetically impossible — which forces a
shared-memory ring." That reasoning, out loud, in the first two minutes, is worth more than
any amount of subsystem knowledge.

#### The three ratios that never change

Specific numbers drift with hardware. These ratios have held for twenty years and are what
you should actually retain:

| Ratio | Value | Consequence |
|---|---|---|
| **memory : L1** | ~80× | Cache-miss behaviour dominates algorithmic complexity at small *n* (Ch. 09 §T.3) |
| **syscall : function call** | ~300× | Batch, or move the work across the boundary (Ch. 24, Ch. 76) |
| **context switch : syscall** | ~10× | Never block to save a syscall; spin briefly instead (Ch. 14) |

And the two that *did* change, which is why modern Linux looks different from 2010 Linux:

- **HDD → NVMe collapsed random/sequential from ~1000× to ~2×.** Every seek-avoidance
  optimization in the storage stack — elevator scheduling, aggressive readahead, extent
  packing — was designed for the first number and is overhead under the second. This single
  change motivated blk-mq (Ch. 63), the `none` scheduler, and much of Ch. 51 §T.7.
- **Mitigations made the syscall 3–5× more expensive** (2018 onward), which is why
  `io_uring` (2019) arrived when it did. The interface did not become a good idea in 2019;
  the boundary became expensive enough that it became a *necessary* one.

**The general lesson, and it is the most transferable thing in this chapter:** when a ratio in
the cost model moves by an order of magnitude, an entire layer of the software stack becomes
wrong. Recognizing that a ratio has moved — before the layer's badness is obvious — is what
architectural foresight actually consists of.

### T.3b — The narrow waist: why one interface serves everything

Observation ③ in T.0 was that wildly different subsystems were reached through one uniform
interface. That is the **hourglass** or **narrow waist** pattern, and it is the most important
structural idea in the kernel.

```
        many, and growing                many, and growing
   ┌──────────────────────────┐    ┌──────────────────────────┐
   │ bash  nginx  postgres    │    │ ext4 xfs btrfs nfs fuse  │
   │ python  ffmpeg  systemd  │    │ tmpfs procfs squashfs    │
   └────────────┬─────────────┘    └────────────┬─────────────┘
                │                                │
          ┌─────▼──────┐                   ┌─────▼──────┐
          │  ~450      │                   │   ~20      │  <- the WAISTS
          │  syscalls  │                   │  VFS ops   │
          └─────┬──────┘                   └─────┬──────┘
                │                                │
   ┌────────────▼─────────────┐    ┌─────────────▼────────────┐
   │ x86 arm64 riscv ppc s390 │    │ nvme scsi mmc md dm      │
   └──────────────────────────┘    └──────────────────────────┘
```

**Why this shape and not another.** With *m* things above and *n* below, direct coupling costs
you *m × n* relationships to build and maintain. A waist costs *m + n*. At m = 10,000
applications and n = 60 filesystems the difference is not an optimization, it is the
difference between a system that can exist and one that cannot.

**The properties a waist must have**, and every one of them is a constraint you will meet
again:

| Property | Why | Where it bites |
|---|---|---|
| **Narrow** | Each new operation must be implemented by *every* provider | Adding a `file_operations` member means auditing 60 filesystems |
| **Stable** | Both sides evolve independently only if the middle does not move | Ch. 24 — the syscall ABI is permanent |
| **Sufficient** | If it cannot express what providers can do, they route around it | `ioctl` exists precisely because the waist was insufficient |
| **Neutral** | It must not encode one provider's assumptions | `readdir`'s design leaked from 1970s Unix and constrains every filesystem since |

Find the waists as you go; the whole curriculum is organized around them:

| Waist | Above | Below | Chapter |
|---|---|---|---|
| syscalls | all userspace | all of the kernel | 24 |
| VFS `file_operations` | `read`/`write`/`mmap` | 60+ filesystems | 53 |
| `struct bio` / blk-mq | filesystems | NVMe, SCSI, MMC, dm, md | 63 |
| `sk_buff` + netdev ops | the protocol stack | every NIC driver | 71, 46 |
| the driver model (`probe`) | the core | every bus and device | 26 |
| `dma_map_*` | every driver | IOMMU, swiotlb, coherent/non-coherent | 35 |
| `clk_get/enable` | every driver | every SoC's clock tree | 43 |

**"Everything is a file" is one waist, not a philosophy.** A file descriptor is a small
integer naming a kernel object that supports `read`, `write`, `poll`, `close`, `mmap`, and
`ioctl`. Once a thing is an fd it inherits, for free: `select`/`poll`/`epoll`, passing over
`SCM_RIGHTS`, automatic close on exit, refcounting, and per-fd permissions. That is why the
modern answer to "how should this new interface work?" is almost always **return a file
descriptor** — `pidfd`, `memfd`, `userfaultfd`, `timerfd`, `signalfd`, `io_uring`, `bpf`
(Ch. 24 §T.5).

**And where the waist leaked.** `ioctl` is the honest admission that `read`/`write` could not
express everything devices do. It is a per-driver, untyped, unversioned escape hatch — and it
is responsible for a large share of the kernel's CVE history and its 32/64-bit compatibility
pain. **Every waist leaks; the engineering question is whether the leak is deliberate and
bounded or accidental and unbounded.** Hold that thought until Ch. 24 §T.7 and Ch. 89 §T.2.

---

### T.4 Monolithic vs microkernel: the argument, settled empirically

This is the most famous design debate in OS history and you will be asked about it.

**Frame it properly first.** The question is *where the isolation boundaries go*, and the
trade is always the same:

```
   fewer boundaries                              more boundaries
   ◄──────────────────────────────────────────────────────────►
   monolithic          hybrid            microkernel      unikernel(none)
   Linux, *BSD         XNU, NT           L4, QNX, seL4    MirageOS

   fast (function calls)                 isolated (IPC, ~1-2 µs even at best)
   one fault kills all                   a fault kills one server
   unverifiable                          verifiable (seL4: ~10 kLOC, proven)
   easy to refactor globally             stable interfaces required at each boundary
```

**A boundary costs performance and buys containment.** That is the whole trade, and it recurs
at every scale in this curriculum: processes vs threads (Ch. 20), containers vs VMs
(Ch. 104 §T.9), in-kernel vs eBPF vs userspace (Ch. 89 §T.3), microservices vs a monolith.
Recognizing that these are *the same decision* at different scales is the point of studying
this debate at all — not the history.

**The microkernel position** (Mach, MINIX, L4, QNX, seL4): put only IPC, scheduling and
address-space management in the kernel; run drivers, filesystems, and the network stack as
*userspace servers*. Benefits: fault isolation (a driver crash kills a server, not the
system), formal verifiability (seL4 has a machine-checked proof of functional correctness),
and clean modularity.

**The Tanenbaum–Torvalds debate** (comp.os.minix, 1992) framed it: Tanenbaum called Linux
"obsolete" for being monolithic; Torvalds argued portability and performance in practice.

**The empirical resolution** is more interesting than either position:

- Chen & Bershad (SOSP 1993) measured that OS performance was dominated by *memory system
  behaviour*, not by structure per se.
- **Liedtke, "On µ-Kernel Construction" (SOSP 1995)** showed Mach's terrible performance was
  an *implementation* failure (huge cache footprint, ~100 µs IPC), not an inherent one. L4
  achieved IPC in ~1–2 µs — 20× faster — by ruthless minimality.
- **Härtig et al., "The Performance of µ-Kernel-Based Systems" (SOSP 1997)** ran Linux on
  L4 and measured ~5–10% overhead. Not catastrophic, but not free either.
- **seL4** (Klein et al., SOSP 2009) delivered a full functional-correctness proof —
  demonstrating the verifiability claim is real, for a ~10 kLOC kernel.

So the honest summary: microkernels *work*, cost single-digit-percent overhead when done
well, and buy genuine isolation and verifiability. Linux won anyway for reasons that are
economic and social, not technical: a monolithic kernel has **no stable internal API**, which
means the whole tree can be refactored atomically by anyone, which means it evolves faster and
absorbs hardware support faster than any competitor. Driver availability is the market;
development velocity is how you win the market.

The modern position is **hybrid convergence**: Linux has spent 20 years moving things *out*
of the kernel where isolation matters — FUSE (filesystems in userspace), UIO/VFIO (drivers in
userspace), DPDK/SPDK (network/storage stacks in userspace), gVisor (a userspace kernel),
and now **Rust** (isolation via the type system rather than address spaces). You should be
able to argue this history in an architecture review.

### T.5 The stable-ABI asymmetry as a commitment device

Linux makes two opposite promises:

| Interface | Promise | Rationale |
|---|---|---|
| Userspace ABI (syscalls, `/proc`, `/sys`, netlink, ioctl) | **never break it** | a broken userspace is a broken *product*; users cannot fix it |
| In-kernel API (`EXPORT_SYMBOL`) | **no stability whatsoever** | freedom to refactor is the competitive advantage |

This is not an accident — it is a **commitment device** in the game-theoretic sense. By
pre-committing publicly and absolutely to never breaking userspace, Linus removed the endless
negotiation that would otherwise consume every release. And by pre-committing to *never*
stabilizing the internal API, he made out-of-tree drivers structurally unsustainable, which
forces vendors to upstream — which is the single most important reason Linux has drivers for
everything.

Read `Documentation/process/stable-api-nonsense.rst`. The argument is: a stable internal API
would freeze bad designs, prevent tree-wide fixes, and (critically) remove the economic
pressure to upstream. Agree or disagree, you must be able to *state* it.

The corollary for your career: **out-of-tree code is a permanent tax.** Every rebase costs
engineering time forever. The business case for upstreaming is a maintenance-cost argument,
and you should be able to make it to a manager in one slide.

### T.6 Conway's Law and the shape of the tree

> "Organizations which design systems are constrained to produce designs which are copies of
> the communication structures of these organizations." — Melvin Conway, 1968

Linux's architecture *is* its maintainer graph. The `MAINTAINERS` file is simultaneously a
social directory and an architectural diagram. Subsystem boundaries exist where maintainer
boundaries exist. The merge window / `-rc` cycle, the subsystem-tree → `linux-next` → Linus
pipeline, and the "one patch does one thing" rule are all mechanisms for making a
10,000-contributor organization produce a coherent artifact.

Practical consequence for you: **changes that cross subsystem boundaries are
disproportionately hard to merge**, not because they're technically harder, but because they
require coordinating multiple maintainers' trees. A senior engineer designs changes to
minimize cross-tree coupling — that's an *organizational* optimization masquerading as a
technical one, and recognizing it is a real skill.

### T.7 The release model

```
   v6.N released
   ├── merge window: 2 weeks. Maintainers send pull requests. ~12–15k commits land.
   ├── -rc1 ... -rc7 : ~7 weeks. Bug fixes only. No new features.
   └── v6.(N+1) released.        Total cycle: 9–10 weeks, like clockwork since 2005.
```

- **`linux-next`** — an integration tree rebuilt daily from ~200 subsystem trees. Its purpose
  is to surface *merge conflicts and semantic conflicts* before the merge window. Getting
  your patch into `-next` is the real gate.
- **Stable/LTS** — `6.x.y` releases carry backported fixes. One LTS per year, supported
  ~2 years (extended to 6 for some). Distros and products build on these (Ch. 88).
- Nothing is time-based in the sense of "features must make it"; features slip to the next
  cycle without drama. A 9-week cycle means slipping costs little, which removes the
  incentive to merge half-finished work. **The cadence is itself a quality mechanism.**

### T.8 Kernel development is not application development

Robert Love's framing (LKD3 Ch. 2) is the best short list of what actually differs:

| Constraint | Consequence |
|---|---|
| **No libc** | `printk` not `printf`; `kmalloc` not `malloc`; a small, different string API |
| **No memory protection** | a NULL deref is an `Oops`, not a `SIGSEGV` you can catch |
| **No (easy) floating point** | FPU state isn't saved on kernel entry; use `kernel_fpu_begin()` or fixed-point |
| **Small, fixed stack** | 8 KiB on x86-32, **16 KiB on x86-64**, 16 KiB on arm64. No recursion, no big locals |
| **Concurrency everywhere** | SMP + preemption + interrupts: *every* global is shared |
| **Portability matters** | endianness, word size, alignment, page size are all variables |
| **Errors are silent-then-fatal** | no exceptions; return `-Exxx` and unwind by hand |

That stack limit is worth dwelling on: `THREAD_SIZE` is 16 KiB *total*, shared with the
`thread_info`/`pt_regs` and any interrupt that lands on it (though x86-64 and arm64 now use
separate IRQ stacks). A single 1 KiB local array in a deep call chain is a real overflow risk.
`CONFIG_VMAP_STACK` (guard pages) turns silent corruption into a clean fault, and
`scripts/checkstack.pl` finds the offenders. **Large locals are a bug, not a style issue.**

### T.9 Concurrency: the four axes

Before writing any kernel code, you must be able to answer four questions about it. This is
the single most useful checklist in this curriculum:

1. **What context am I in?** (task / softirq / hardirq / NMI)
2. **Can I sleep here?** (equivalently: can `schedule()` be called?)
3. **What can run concurrently with this code?** (another CPU, an interrupt on *this* CPU, a
   preempting task, a signal handler on the userspace side)
4. **What are the memory-ordering requirements?** (Ch. 13)

Sources of concurrency, exhaustively:

| Source | Since | Guard |
|---|---|---|
| Interrupts | always | `spin_lock_irqsave()` |
| Softirqs/tasklets | always | `spin_lock_bh()` |
| SMP (another CPU) | 2.0 | any lock |
| Kernel preemption | 2.6 | `preempt_disable()`, locks imply it |
| Sleeping/blocking | always | a lock held across a sleep must be sleepable |
| Migration to another CPU | always | `migrate_disable()`, per-CPU discipline |

Love's rule, which is worth memorizing verbatim: *"identify the data, not the code, that needs
protection."* Locks protect **data**, never code. If you cannot name the variable a lock
protects, the lock is wrong.

**Why this is stated as a rule about data.** The instinct from single-threaded programming is
to think about *critical sections* — regions of code. That instinct produces two specific
bugs, and you will see both in review:

```c
/* BUG 1: the same data protected by different locks on different paths. */
void path_a(void) { spin_lock(&a_lock); obj->count++; spin_unlock(&a_lock); }
void path_b(void) { spin_lock(&b_lock); obj->count++; spin_unlock(&b_lock); }
/* Both paths "hold a lock". Neither is protected from the other. */

/* BUG 2: the lock is released, but the data is still in use. */
struct foo *lookup(int id) {
	struct foo *f;
	spin_lock(&list_lock);
	f = find(id);
	spin_unlock(&list_lock);
	return f;          /* f can be freed by another CPU before the caller uses it */
}
/* The critical section was "correct". The DATA's protection ended too early. */
```

Bug 2 is the one that matters, because it is the shape of a huge fraction of real kernel
CVEs. The fix requires a *lifetime* mechanism, not a bigger critical section — a refcount
taken under the lock, or RCU. That is Ch. 12 and Ch. 15, and it is why they come before the
driver chapters.

So the discipline is: for every shared field, be able to complete the sentence **"`x` is
protected by `y`, and may be accessed in contexts `z`."** Write it in the struct definition:

```c
struct mydev {
	spinlock_t          lock;
	u32                 state;      /* protected by lock */
	struct list_head    pending;    /* protected by lock; also from IRQ -> irqsave */
	atomic_t            refcount;   /* self-protected */
	struct completion   done;       /* self-protected */
	const u32           caps;       /* immutable after probe -- no lock needed */
};
```

The last two rows are as important as the first: **"self-protected" and "immutable" are
legitimate answers**, and knowing when they apply is most of what keeps lock counts low.
Ch. 81 §T.2 shows how Rust turns this comment into a type the compiler checks.

---

### T.10 — What is still argued about

A curriculum that presents only settled questions teaches you to recite. These are live, and
having a position — with conditions — is what a senior interview is probing for.

**1. Is the "never break userspace" rule absolute?**
It has real costs: permanent bad interfaces (`ioctl` numbering, `/proc` text formats), and
security fixes that cannot be made because something depends on the flaw (Ch. 24 §T.7). The
counter-position: the rule's value is precisely that it is *unconditional* — a rule with
exceptions is a negotiation, and the negotiation would consume every release (§T.5). Where do
you stand, and what evidence would move you?

**2. Should Rust be in the kernel?**
See Ch. 85 §T.6 for the full treatment. The technical case is strong and the social cost is
real: a C maintainer now has code in their subsystem they cannot review. This produced public
conflict in 2025. The honest summary is that the bottleneck is *review capacity*, not the
language.

**3. Is eBPF a good idea or a Trojan horse?**
It is, in effect, a second kernel programming environment with a different safety model,
growing fast, and increasingly load-bearing. The case for: userspace-authored policy at
kernel speed, with no module and no ABI commitment. The case against: the verifier is a
large, complex attack surface that has itself had CVEs, and "we'll write a BPF program"
increasingly substitutes for fixing the interface properly. Ch. 75.

**4. Is the monolith still right at 40 M lines?**
The 1992 arguments were about performance and they were largely answered (§T.4). The modern
argument is about *reviewability* — nobody understands the whole thing, and the reference
monitor criterion fails badly (§T.2). The counter-move has not been a microkernel; it has
been shrinking what is *reachable* (seccomp), what is *trusted* (LSM, lockdown), and what is
*unsafe* (Rust). Whether that is sufficient is genuinely open.

**5. How much should be paid for speculative-execution mitigations?**
`mitigations=off` recovers most of a 3–5× syscall regression (§T.3). On a single-tenant
machine with no untrusted local code that is a defensible engineering decision; on a shared
host it is catastrophic. It is a *policy* decision requiring a stated threat model — which is
the point of Ch. 102 §T.2 and a good example of why "best practice" is not an answer.

---

### T.11 — The compressed model

If you retain nothing else from this chapter, retain this. Everything in the remaining 105
chapters is an elaboration.

```
 The kernel is a LIBRARY that runs in a PRIVILEGED CPU MODE.
 It has no main loop. It is ENTERED (trap / interrupt / kthread) and EXITED.

 It exists to MULTIPLEX, ABSTRACT, and PROTECT.
   - Protection reduces to one hardware bit and the page tables.
   - The trap is the only door, and it is narrow and total.
   - Every abstraction is a lie that leaks somewhere; mastery is knowing where.

 The BOUNDARY IS EXPENSIVE (~300 ns), so the whole design is shaped by
 avoiding it: batching, shared-memory rings, zero-copy, and mapping the
 kernel into every address space.

 Structure is HOURGLASSES: many above, many below, a narrow stable waist
 between. Find the waist and you understand the subsystem.

 The USERSPACE ABI IS FOREVER; the internal API is deliberately unstable.
 That asymmetry is a commitment device, and it is why Linux has drivers.

 CONCURRENCY IS THE DEFAULT. For every piece of shared data, answer:
   what context am I in / can I sleep / what runs concurrently / what ordering.
 Locks protect DATA, not code.
```

**Five questions to ask about any unfamiliar kernel code**, in order. This is the actual
working procedure and it does not change for the rest of your career:

1. Which of *multiplex / abstract / protect* is this serving?
2. What is the waist here — what is the narrow interface, and who is above and below it?
3. What context does this run in, and can it sleep?
4. What data is shared, and what protects it?
5. What is the failure mode, and how would I observe it?

---

## 1. Concept

### 1.1 The kernel is a library that runs in a privileged CPU mode

Strip away mystique. The Linux kernel is:

- A **big C program** (~40 million lines, but you never read more than ~2000 at a time).
- It never "runs" as a process. It has **no main loop**. It is *entered* and *exited*.
- Three ways in: **system call**, **interrupt/exception**, **kernel thread**.

```
                 ┌──────────────────────────────────────────────┐
   user mode     │  bash   nginx   systemd   your_app   ...      │
   (ring 3 /     │        ↕ libc (glibc, musl)                   │
    EL0)         └──────────────┬───────────────────────────────┘
                                │ syscall / svc / ecall  ← the ONLY door
  ─────────────────────────────────────────────────────────────────
                 ┌──────────────▼───────────────────────────────┐
   kernel mode   │  syscall entry → dispatch table              │
   (ring 0 /     │  ┌──────────┬──────────┬─────────┬────────┐  │
    EL1)         │  │ VFS      │ Net      │ MM      │ Sched  │  │
                 │  ├──────────┼──────────┼─────────┼────────┤  │
                 │  │ Block    │ Proto    │ Page    │ IRQ    │  │
                 │  │ layer    │ stack    │ alloc   │ core   │  │
                 │  └────┬─────┴────┬─────┴────┬────┴───┬────┘  │
                 │       │  Device drivers      │        │      │
                 │  ┌────▼──────────▼───────────▼────────▼───┐  │
                 │  │ arch/ : CPU-specific entry, MMU, atomics│  │
                 └──┴────────────────┬────────────────────────┴──┘
  ─────────────────────────────────────────────────────────────────
   firmware/hw   │  UEFI / U-Boot / ATF  │  CPU  MMU  IOMMU  PCIe  │
                 └──────────────────────────────────────────────┘
```

### 1.2 The three execution contexts

This single table explains 80% of kernel bugs. Memorize it.

| Context | Can sleep? | Can be preempted? | Who else can run? | Typical code |
|---|---|---|---|---|
| **Process (task) context** | ✅ yes | ✅ (if `CONFIG_PREEMPT`) | anything | syscall handlers, kernel threads, workqueues |
| **Softirq / tasklet / BH** | ❌ **no** | ❌ no (but hardirq can interrupt) | hardirqs | `NET_RX_SOFTIRQ`, timers, block completion |
| **Hardirq (interrupt) context** | ❌ **no** | ❌ no | NMI only | ISR top halves |
| **NMI context** | ❌ no | ❌ no | nothing | perf PMU, watchdog |

"Can sleep" means: may call `schedule()`, may call `kmalloc(GFP_KERNEL)`, may take a `mutex`,
may call `copy_to_user()` (which can page-fault).

> **The first rule of kernel code:** before writing any line, ask *"what context am I in?"*

Helpers: `in_task()`, `in_interrupt()`, `in_softirq()`, `in_nmi()`,
and the debugging macro `might_sleep()` (enable `CONFIG_DEBUG_ATOMIC_SLEEP`).

### 1.3 Kernel vs. user address space

On x86-64 with 4-level paging:

```
0x0000_0000_0000_0000 ┐
                      │ user virtual address space (128 TiB)
0x0000_7fff_ffff_ffff ┘
      ... non-canonical hole ...
0xffff_8000_0000_0000 ┐
0xffff_8880_0000_0000 │ direct map of all physical RAM (page_offset_base)
0xffff_c900_0000_0000 │ vmalloc / ioremap space
0xffff_ea00_0000_0000 │ vmemmap (struct page array)
0xffff_ffff_8000_0000 │ kernel text (.text/.data), KASLR-shifted
0xffff_ffff_ffff_ffff ┘
```

Key consequences:

- The kernel is mapped into **every** process's page tables (top half), so a syscall is a
  privilege transition, **not** an address-space switch — that's why syscalls are ~100 ns
  and context switches are ~1–3 µs.
- KPTI (Meltdown mitigation) breaks this: user page tables contain only a trampoline.
  Cost: extra CR3 writes + TLB flushes per syscall.
- A user pointer is **never** dereferenced directly. Always
  `copy_from_user()` / `get_user()` / `copy_to_user()`, which handle faults and
  (since v5.x) SMAP/PAN toggling.

### 1.4 Monolithic, but modular

Linux is a **monolithic** kernel: drivers run in the same address space with full privilege.
A NULL deref in a driver = `Oops` (kill the task) or `panic` (kill the box).
There is no microkernel isolation. Hence the obsession with review, `sparse`, KASAN, and now Rust.

Loadable modules (`.ko`) are ELF relocatable objects linked into the running kernel at `insmod`
time. Same privilege. "Modular" ≠ "isolated".

### 1.5 The stability contract

| Interface | Stable? | Rule |
|---|---|---|
| **Syscalls / UAPI** (`include/uapi/`) | ✅ **forever** | "We do not break userspace." — Linus |
| `/proc`, `/sys`, netlink, ioctl ABI | ✅ effectively | breaking it breaks userspace |
| **In-kernel APIs** (EXPORT_SYMBOL) | ❌ **no** | changed freely; out-of-tree code is your problem |
| `Documentation/ABI/` | described per-file | `stable/`, `testing/`, `obsolete/` |

This asymmetry is *the* defining cultural fact of Linux and drives almost every architectural
argument you will encounter. Read `Documentation/process/stable-api-nonsense.rst`.

---

## 2. Internals: the shape of the tree

```
linux/
├── arch/            CPU-specific: entry code, MMU, atomics, boot  (x86, arm64, riscv…)
├── block/           Block layer, I/O schedulers, blk-mq
├── certs/           Module signing keys
├── crypto/          Crypto API (skcipher, ahash, akcipher, AEAD)
├── Documentation/   ★ Excellent. Sphinx-built. `make htmldocs`
├── drivers/         ~60% of the tree. Everything that touches hardware
├── fs/              VFS + every filesystem
├── include/
│   ├── linux/       Internal kernel headers
│   ├── uapi/        ★ The stable userspace ABI
│   └── asm-generic/ Fallback arch implementations
├── init/            main.c — start_kernel()
├── io_uring/        io_uring (split out of fs/ in 5.19)
├── ipc/             SysV IPC, POSIX mqueue
├── kernel/          Core: sched/, locking/, rcu/, time/, trace/, bpf/, cgroup/, irq/
├── lib/             Generic data structures & helpers (rbtree, xarray, string, crc)
├── mm/              Memory management
├── net/             Networking stack
├── rust/            ★ Rust support: kernel crate, bindings, macros
├── samples/         Example code — read these
├── scripts/         Build system, checkpatch, coccinelle, gdb helpers
├── security/        LSM framework, SELinux, AppArmor, Landlock, IMA
├── sound/           ALSA
├── tools/           Userspace tools built from the tree: perf, bpftool, testing/selftests
└── virt/            KVM common code
```

**Where beginners get lost:** `drivers/` is huge, but it's shallow. `kernel/`, `mm/`, and
`fs/` are small but deep. Spend your time in the deep parts.

### 2.1 The boot path, one line per step

```
firmware (UEFI / U-Boot / ATF)
  → arch/x86/boot/  (real mode stub, decompressor)  |  arch/arm64/kernel/head.S
  → arch/*/kernel/head*.S  : set up initial page tables, enable MMU, jump to C
  → start_kernel()            init/main.c            ← READ THIS FUNCTION
      setup_arch()            arch-specific, memblock, e820/DT parsing
      build_all_zonelists()   NUMA/zone setup
      mm_core_init()          buddy allocator online
      trap_init(), init_IRQ()
      sched_init()            runqueues, init_task
      rcu_init()
      time_init(), timekeeping_init()
      kmem_cache_init()       slab online → kmalloc works
      vfs_caches_init()       dcache, inode cache, rootfs mounted (tmpfs)
      arch_call_rest_init()
        → rest_init()
            kernel_thread(kernel_init)   → PID 1
            kernel_thread(kthreadd)      → PID 2
            cpu_startup_entry(CPUHP_ONLINE)  → the idle loop, forever
  → kernel_init()
      do_initcalls()          ★ all `*_initcall()` in link order
      free_initmem()          drop __init sections
      prepare_namespace()     mount real root
      run_init_process("/sbin/init")  → execve → user mode, PID 1
```

`do_initcalls()` is the single most important mechanism to understand early. Levels:

```c
pure_initcall      0
core_initcall      1    /* kernel/ subsystems */
postcore_initcall  2
arch_initcall      3
subsys_initcall    4    /* bus types: pci, usb, i2c */
fs_initcall        5
rootfs_initcall    5s
device_initcall    6    /* == module_init() when built-in */
late_initcall      7
```

`module_init(foo)` expands to `device_initcall(foo)` when `CONFIG_FOO=y`, and to a
module load hook when `=m`. That's the trick that lets one source file be both.

---

## 3. Practice

### Lab 0.1 — Read `start_kernel()`

```bash
git clone --depth=1 https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
$EDITOR init/main.c     # jump to start_kernel()
```

Write down, in your own words, **20 lines**: one per major call, what it initializes.
Do not skip this. It is the single best 90 minutes you will spend.

### Lab 0.2 — Observe the three contexts

```bash
# Softirq/hardirq accounting per CPU
cat /proc/softirqs
cat /proc/interrupts

# Which kernel threads exist? (bracketed names in ps)
ps -eLo pid,tid,class,rtprio,comm | head -50

# Live kernel stack of a sleeping process
sudo cat /proc/$(pgrep -n sshd)/stack
```

### Lab 0.3 — Watch a syscall cross the boundary

```bash
strace -c ls                       # syscall histogram
sudo perf trace -e openat ls       # with kernel-side timing
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_openat {
    printf("%s -> %s\n", comm, str(args->filename)); }'
```

### Lab 0.4 — Prove the kernel is mapped into every process

```bash
sudo cat /proc/self/maps | tail -5      # no kernel addresses visible (by design)
sudo cat /proc/kallsyms | grep ' T start_kernel'
# kptr_restrict hides them from non-root:
cat /proc/sys/kernel/kptr_restrict
```

### Lab 0.5 — Measure the cost model yourself (T.3)

Everything in T.3 should be a number *you measured*, not a number you read. Build this:

```c
/* costs.c — build: gcc -O2 -o costs costs.c -lpthread */
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>
#include <sched.h>
#include <pthread.h>
#include <sys/syscall.h>

#define N 2000000

static inline uint64_t now_ns(void)
{
	struct timespec ts;
	clock_gettime(CLOCK_MONOTONIC, &ts);
	return ts.tv_sec * 1000000000ULL + ts.tv_nsec;
}

__attribute__((noinline)) static int nop_fn(int x) { return x + 1; }

int main(void)
{
	uint64_t t0;
	long i, acc = 0;

	/* 1. plain function call */
	t0 = now_ns();
	for (i = 0; i < N; i++) acc += nop_fn(i);
	printf("function call     : %6.2f ns\n", (double)(now_ns() - t0) / N);

	/* 2. vDSO call — looks like a syscall, never enters the kernel */
	t0 = now_ns();
	for (i = 0; i < N; i++) { struct timespec ts; clock_gettime(CLOCK_MONOTONIC, &ts); }
	printf("clock_gettime vDSO: %6.2f ns\n", (double)(now_ns() - t0) / N);

	/* 3. real syscall — getppid() is the cheapest that cannot be cached */
	t0 = now_ns();
	for (i = 0; i < N/10; i++) syscall(SYS_getppid);
	printf("syscall (getppid) : %6.2f ns\n", (double)(now_ns() - t0) / (N/10));

	/* 4. uncontended atomic */
	{
		volatile long v = 0;
		t0 = now_ns();
		for (i = 0; i < N; i++) __sync_fetch_and_add(&v, 1);
		printf("atomic add        : %6.2f ns\n", (double)(now_ns() - t0) / N);
	}
	return acc == 0;
}
```

Then measure context switches and mode switches:

```bash
# Context switch: two tasks ping-ponging on a pipe, pinned to the SAME cpu
sudo apt install -y libc6-dev sysbench lmbench    # lmbench has lat_ctx / lat_syscall
taskset -c 2 ./costs

# lmbench (the canonical microbenchmark suite):
lat_syscall null ; lat_syscall read ; lat_syscall stat /tmp
lat_ctx -s 0 2 4 8 16          # context-switch latency vs number of processes
lat_mem_rd 64M 128             # the memory hierarchy, in one command

# Prove KPTI's cost (reboot to compare):
cat /sys/devices/system/cpu/vulnerabilities/meltdown
# boot with  mitigations=off  or  nopti   in a VM and re-run lat_syscall null
```

**Expected discoveries.** The vDSO call is ~10× cheaper than the real syscall — that is the
entire reason the vDSO exists. The syscall cost roughly triples with KPTI on. Write these
numbers down; you will reuse them in every design discussion for the rest of your career.

### Lab 0.6 — Watch the mode transition in the disassembly

```bash
# Where does a syscall actually go?
objdump -d /lib/x86_64-linux-gnu/libc.so.6 | grep -A12 '<getppid@@GLIBC_2.2.5>:'
# → you will see:  mov $0x6e,%eax ; syscall ; ...

# The kernel side:
sudo grep -E ' (entry_SYSCALL_64|do_syscall_64|__x64_sys_getppid)$' /proc/kallsyms

# The vDSO — dump it out of a live process and disassemble:
cat /proc/self/maps | grep vdso
sudo dd if=/proc/self/mem bs=1 skip=$((0x...)) count=8192 of=/tmp/vdso.so 2>/dev/null
objdump -d --start-address=0 /tmp/vdso.so | grep -A20 '__vdso_clock_gettime'
# Note: NO syscall instruction on the fast path.
```

### Lab 0.7 — Prove the "no stable internal API" claim

```bash
cd linux
# How many exported symbols changed in one release?
git diff --stat v6.5..v6.6 -- '*.c' '*.h' | tail -1
git log --oneline v6.5..v6.6 --grep='EXPORT_SYMBOL' | wc -l

# Find a function that changed signature — pick any subsystem:
git log -p --follow -L :'^static int ext4_writepages':fs/ext4/inode.c | head -60

# Contrast: find a syscall whose UAPI struct changed incompatibly. (You won't.)
git log --oneline -- include/uapi/asm-generic/unistd.h | head -20
```

### Lab 0.8 — Explore every kernel↔userspace interface

You should be able to name and use all five. Do each:

```bash
# 1. syscalls
strace -c -f ls >/dev/null

# 2. /proc  — process and kernel state
cat /proc/self/status /proc/cmdline /proc/modules /proc/interrupts | head -40
ls /proc/sys/kernel/                       # sysctl namespace

# 3. /sys   — the device model (Ch. 26)
ls /sys/class/ /sys/bus/ /sys/devices/
udevadm info -a -n /dev/sda | head -40

# 4. netlink — the structured, async channel
ip -d link show                            # uses rtnetlink
sudo ss -tanp                              # uses sock_diag netlink

# 5. ioctl / device files
ls -l /dev/ | head
sudo blockdev --getsize64 /dev/sda         # BLKGETSIZE64 ioctl
```

### Lab 0.9 — Find the stack limit empirically

```bash
cd linux
grep -rn 'define THREAD_SIZE' arch/x86/include/asm/page_64_types.h arch/arm64/include/asm/memory.h
# Then, after building (Ch. 01):
./scripts/checkstack.pl < <(objdump -d vmlinux) | head -20
```
Any function above ~1024 bytes of stack is a red flag. Look at what the top offenders do.

### Lab 0.10 — Observe the boot you just read about

```bash
# The initcall trace: see every initcall and how long it took
# boot with:  initcall_debug  (add to GRUB cmdline or QEMU -append)
dmesg | grep -E 'initcall .* returned .* after' | sort -t'r' -k3 -n | tail -30

# Same data, rendered as a chart:
systemd-analyze plot > boot.svg
systemd-analyze blame | head -20
systemd-analyze critical-chain

# When did the kernel hand off to userspace?
dmesg | grep -E 'Freeing unused kernel|Run /sbin/init|Kernel command line'

# What is PID 0, 1, 2?
ps -eo pid,ppid,comm | head -5
ls /proc/1/ /proc/2/task | head
```
Correlate the `initcall_debug` output with the `do_initcalls()` level list above. You will see
`subsys_initcall` (bus registration) strictly precede `device_initcall` (drivers) — the
ordering that makes driver probing work at all (Ch. 27).

---

## 4. Mastery drills

1. **Context quiz.** For each, name the context and whether sleeping is allowed:
   `ext4_file_write_iter`, `net_rx_action`, `hrtimer_run_queues`,
   `usb_hcd_irq`, `kswapd`, `__do_page_fault`. (Grep each; look at callers.)

2. **Trace a `write(2)`** from `arch/x86/entry/entry_64.S` down to a block device
   submission. List every function. Use `perf trace` + `ftrace function_graph` to check
   yourself (Module 24 teaches how).

3. **Explain to a peer** why `printk()` is callable from NMI context but `printf()` isn't a
   thing in the kernel. (Hint: `kernel/printk/printk_ringbuffer.c`.)

4. **Find one example each** of `pure_initcall`, `subsys_initcall`, and `late_initcall` in
   the tree and explain why the author chose that level.

5. **Argue both sides of Tanenbaum–Torvalds.** Write 400 words for the microkernel position
   citing Liedtke and seL4, then 400 words for the monolithic position citing development
   velocity and driver availability. Then state which parts of the debate were settled
   *empirically* and which were settled *economically*.

6. **Policy/mechanism audit.** For each of these, decide whether Linux put policy in the
   kernel or in userspace, and whether you agree: CPU scheduling, OOM victim selection,
   I/O scheduling, TCP congestion control, security decisions (LSM), power management
   governors. Find one where the decision has been *reversed* over time and explain why.
   (Hint: `sched_ext`, pluggable congestion control, `systemd-oomd`.)

7. **The cost-model design exercise.** You are designing a new interface for a device that
   produces 1 million events/second. Using only your measured numbers from Lab 0.5, compute
   the CPU cost of: (a) one syscall per event; (b) batching 64 events per syscall;
   (c) a shared-memory ring with no syscall in the common case. Show your arithmetic. This
   is exactly the reasoning that produced `io_uring` and AF_XDP.

8. **Find a "we do not break userspace" revert.** `git log --grep="Revert" --grep="regression"
   --all-match --oneline | head -30`. Read one where a kernel change was reverted because it
   broke an application, even though the application was arguably wrong. Summarize Linus's
   reasoning.

9. **Stack budget.** Compute the worst-case stack depth for a syscall path of your choosing
   using `scripts/checkstack.pl` plus the call graph. Compare with `THREAD_SIZE`. Explain
   why `CONFIG_VMAP_STACK` exists and what it converts a silent bug into.

10. **The four-axes checklist (T.9).** Take any 50-line function from `drivers/` and answer
    all four questions about it in writing. Do this for ten different functions until it is
    automatic. This single habit prevents most kernel bugs.

---

## 5. Further reading

**Kernel documentation:**
- `Documentation/admin-guide/README.rst`
- `Documentation/process/` — **read all of it eventually**; start with
  `howto.rst`, `stable-api-nonsense.rst`, `development-process.rst`,
  `submitting-patches.rst`, `coding-style.rst`
- `Documentation/admin-guide/kernel-parameters.txt` — keep it open in a tab forever

**Books:**
- Love, *Linux Kernel Development*, 3rd ed. — **best first book.** Chapters 1–2 are exactly
  this module. 2.6-era APIs, but the conceptual framing is still the best in print.
- Bovet & Cesati, *Understanding the Linux Kernel*, 3rd ed. — dated, but the boot and MM
  chapters remain the most detailed narrative walkthroughs available.
- Corbet, Rubini & Kroah-Hartman, *Linux Device Drivers*, 3rd ed. — free online; APIs are
  stale, the *model* is correct.
- Tanenbaum & Bos, *Modern Operating Systems* — for the general OS theory in T.1–T.4.
- Arpaci-Dusseau & Arpaci-Dusseau, *Operating Systems: Three Easy Pieces* — **free**, and the
  best modern OS textbook. Read the virtualization/concurrency/persistence structure; it maps
  exactly onto T.1.

**Papers (all short, all worth it):**
- Lampson, "Hints for Computer System Design" (SOSP 1983) — **read this first**
- Saltzer, Reed & Clark, "End-to-End Arguments in System Design" (TOCS 1984)
- Levin et al., "Policy/Mechanism Separation in Hydra" (SOSP 1975)
- Conway, "How Do Committees Invent?" (Datamation, 1968)
- Liedtke, "On µ-Kernel Construction" (SOSP 1995)
- Härtig et al., "The Performance of µ-Kernel-Based Systems" (SOSP 1997)
- Klein et al., "seL4: Formal Verification of an OS Kernel" (SOSP 2009)
- Chen & Bershad, "The Impact of Operating System Structure on Memory System Performance"
  (SOSP 1993)
- Anderson, "Computer Security Technology Planning Study" (1972) — the reference monitor
- Naur, "Programming as Theory Building" (1985) — why reading code is not enough (Ch. 07)

**Ongoing:**
- LWN.net "Kernel index" — https://lwn.net/Kernel/Index/ — **the actual literature of the
  field.** A subscription is the single highest-ROI purchase in your career.
- The Tanenbaum–Torvalds debate archive (comp.os.minix, Jan–Feb 1992) — read the primary
  source, not summaries.

→ Next: [01-toolchain.md](01-toolchain.md)

# System Design for Kernel & Systems Architects

> **What this round actually tests.** Not whether you know the answer — there isn't one.
> It tests whether you can take an under-specified problem, **impose structure on it**,
> enumerate the forces, commit to a position, and state honestly what the position costs.
> At principal level it also tests whether you will *reject the premise* when the premise is
> wrong.
>
> Kernel system-design rounds differ from the web-scale kind. Nobody asks you to shard a
> database. They ask you to design an interface that will be unchangeable for twenty years,
> or a data path that must hit 10M IOPS, or a security boundary that must hold against a
> hostile local user.

---

## 1. The framework — six moves, in order

Use this every time. Saying the move names out loud is not cheating; it shows structure.

### Move 1 — Establish the requirement that actually binds

Most designs are determined by a single dominant constraint. Find it in the first three
minutes by asking:

- **What is the scale?** Ops/sec, bytes/sec, number of CPUs, number of devices. An answer
  correct at 10K IOPS is wrong at 10M.
- **What is the latency target, and is it mean or tail?** p99.9 changes everything. Mean
  latency targets permit batching; tail targets forbid it.
- **What is the failure model?** Crash-stop? Malicious local user? Malicious *device*
  (Thunderbolt, confidential computing)? Silent data corruption?
- **What is the compatibility horizon?** A UAPI is forever. An internal API is not. An
  out-of-tree module is neither.
- **Who operates this?** A fleet SRE, a device vendor, an end user? That decides where the
  policy lives.

### Move 2 — Draw the data path and mark every boundary

Literally draw it. At every arrow, name what crosses:

```
  user buffer ──syscall──► kernel ──DMA map──► device ──wire──► peer
              ^          ^                ^
              |          |                |
        copy? validate?  coherency?   ordering?  endianness?
```

Boundaries are where bugs, costs, and security failures live. Naming them all is most of
the design.

### Move 3 — Separate mechanism from policy

State which is which, explicitly. Mechanism goes in the kernel; policy goes in userspace,
in eBPF, or in a tunable. If you cannot separate them, say so and explain why — sometimes
the policy is a physical law (you cannot make the device's cache flush optional).

### Move 4 — Enumerate the alternatives and price each

Always three, never one. The shape:

| Option | Buys | Costs | Choose when |
|---|---|---|---|
| A | | | |
| B | | | |
| C (do nothing) | | | |

"Do nothing / use the existing thing" must always be on the list. Half the time it wins,
and knowing that is the senior signal.

### Move 5 — Commit, with conditions

"I would do B. I would switch to A if [measurable condition]." A design without a commitment
reads as indecision; a commitment without conditions reads as inflexibility.

### Move 6 — Name the failure modes and the observability

For each: how does it break, how do you *detect* that it broke, and what does the operator
do about it. Design that ends at "and then it works" is unfinished. Every design should
answer: **what counter would I add?**

---

## 2. Design problem: a high-throughput storage data path (10M IOPS)

> *"We have a PCIe device that can do 10 million 4 KiB IOPS. Design the software path."*

### Bind the requirement

10M IOPS is **100 ns per I/O of total CPU budget per core-equivalent**. A single syscall
with mitigations is 200–500 ns. **Therefore a per-I/O syscall is arithmetically impossible.**
Say this in the first two minutes — it eliminates most of the design space immediately and
demonstrates you reason with numbers.

Second derivation: at 4 KiB × 10M = **40 GB/s**, which exceeds a single DDR channel pair's
useful bandwidth and requires PCIe Gen5 x16. So a data *copy* is also impossible — the path
must be zero-copy end to end, and probably must be NUMA-local to the device.

### The path

```
 app ──► io_uring SQ ring (shared memory, no syscall in SQPOLL mode)
      ──► io_uring submits ──► blk-mq per-CPU software queue
      ──► hardware queue (one per CPU, matched to device SQ)
      ──► NVMe driver builds command, writes doorbell
      ──► device DMAs directly to/from the *user* pages
      ──► completion queue ──► interrupt (or polled) ──► CQ ring
```

### Decisions and their prices

| Decision | Buys | Costs |
|---|---|---|
| `io_uring` with `SQPOLL` | zero syscalls on the fast path | a burned poller core per ring; large attack surface |
| **Registered buffers** (`IORING_REGISTER_BUFFERS`) | pin + DMA-map once, not per I/O | pinned memory accounting; `RLIMIT_MEMLOCK`; complicates fork |
| `O_DIRECT` | bypass page cache — no copy, no double-buffering | app must handle alignment and its own caching; no readahead |
| **Polled completion** (`IORING_SETUP_IOPOLL`) | no interrupt, no softirq, lowest latency | burns CPU at low load; only works with `O_DIRECT` |
| **Queue-per-CPU**, `none` scheduler | no cross-CPU lock, no reordering work | no global fairness — one tenant can starve others |
| **NUMA-local** allocation and IRQ affinity | avoids cross-socket DMA and cacheline traffic | rigid placement; a scheduler migration destroys it |
| Large pages for the buffer pool | fewer TLB entries for a 40 GB/s working set | fragmentation, allocation latency |

### Where it breaks

- **Tag exhaustion.** The device has a finite queue depth; `sbitmap` handles the allocation
  but a slow device turns the tag pool into an implicit admission controller with no
  fairness. Observe: `/sys/block/*/mq/*/tags`.
- **Completion interrupt storms** if you do not poll — the `irq` mode with `nvme.poll_queues`
  split is the practical answer (poll queues for the latency-critical ring, IRQ queues for
  everything else).
- **`ksoftirqd` starvation** if completions are processed in softirq and the CPU is saturated.
- **Head-of-line blocking** in the device if a large I/O is queued ahead of small ones. If
  the target has mixed sizes, you need separate queues or a scheduler — which costs you the
  `none` decision above. Name this tension.
- **Fairness across tenants is now your problem**, because you removed every layer that
  provided it. Answer: `io_uring`'s per-ring limits, cgroup `io.max`, or accept it and
  isolate physically.

### The senior-vs-staff difference here

Senior describes the path. Staff derives "100 ns budget ⇒ no syscall ⇒ io_uring" from
arithmetic and then says **which of the removed layers provided a guarantee we now have to
reprovide**. Principal adds: "before building this, I would ask whether the application can
tolerate 5M IOPS, because the second 5M costs a burned core per ring, all fairness, and a
CVE-heavy interface."

→ Ch. 51, Ch. 63, Ch. 69, Ch. 76

---

## 3. Design problem: a new UAPI for device telemetry

> *"Devices want to export streaming telemetry to userspace. Design the interface."*

### Bind the requirement

Ask: how many devices, what rate, what record size, is loss acceptable, who consumes it
(one privileged daemon or arbitrary processes), and does it need timestamps correlated with
other kernel events? Those five answers pick the mechanism.

### The option space — know all of these and why each exists

| Mechanism | Shape | Good for | Bad for |
|---|---|---|---|
| **sysfs** | one value per file, text | configuration, rare reads | streaming (one syscall per value), structured records |
| **debugfs** | anything, **no ABI stability** | debugging, developer-only | anything a product depends on |
| **procfs** | process-related | legacy | new interfaces (don't) |
| **ioctl** on a chardev | arbitrary structs, request/response | control, batch reads | discoverability; 32/64 compat pain |
| **char device `read()`** | byte stream | simple streaming | multiple consumers, record boundaries |
| **netlink** | multicast, structured TLVs, versioned | many consumers, event broadcast, existing net-adjacent tooling | complex; text-ish overhead |
| **relay / relayfs** | per-CPU ring buffers, mmap'd | very high-rate kernel→user streaming with no syscall per record | no filtering, no structure |
| **perf ring buffer** | mmap'd per-CPU ring, sample records | correlated with perf events, existing tooling | perf-shaped only |
| **eBPF ring buffer** (`BPF_MAP_TYPE_RINGBUF`) | MPSC shared ring, mmap'd, ordered | programmable filtering *in the kernel*, no fixed ABI at all | requires BPF in your stack |
| **tracefs / trace events** | structured, filterable, existing tooling (`perf`, `bpftrace`, LTTng) | **almost always the right first answer** | not a stable ABI (though in practice heavily depended on) |

### The design

Commit: **trace events for the debugging/observability use case; a per-device chardev with
an mmap'd ring for the high-rate product use case.** Reasons:

1. Trace events cost nothing when disabled (static keys — a patched-out NOP), integrate with
   every existing tool, get filtering and triggers for free, and are the path of least
   resistance for upstreaming. If the rate is low or the consumer is a human, stop here.
2. If a product daemon must consume millions of records/sec, the syscall cost argument from
   §2 applies again — you need a shared-memory ring. A chardev gives you a capability
   (an fd), per-device permissions, `poll()` for wakeups, and `mmap()` for the ring.

### The ABI details that get you graded

- **Versioned, size-prefixed records**: every record starts with `{__u16 type; __u16 len;}`
  so a consumer can skip unknown types. This is the single most important decision — without
  it you can never add a field.
- **Explicit-width types only** (`__u32`, `__u64`), explicit padding, no holes. Run
  `pahole`; enable `CONFIG_GCC_PLUGIN_STRUCTLEAK` — leaking uninitialized padding to userspace
  is an information disclosure CVE class.
- **`__u64` for pointers**, never `void *` or `unsigned long` — 32-bit compat.
- **Timestamps**: state the clock. `CLOCK_MONOTONIC_RAW` for intervals,
  `ktime_get_ns()` in-kernel, and expose which clock it is so userspace can correlate.
- **Overflow policy must be visible.** A ring that silently drops is a lie. Include a
  dropped-record counter *in the stream* so the consumer knows its data has a hole.
- **Flags reserved and validated**: `if (arg->flags & ~KNOWN_FLAGS) return -EINVAL;` — this
  single line is what lets you add a flag in five years.
- **Per-device permissions** via the chardev's mode, plus a capability check
  (`CAP_SYS_ADMIN` is usually too coarse — prefer a specific one or file permissions).

### What would change my mind

If there are more than two independent consumers with different filtering needs → netlink
multicast or a BPF ringbuf, because a byte stream forces every consumer to read everything.
If the rate turns out to be under ~10K/sec → trace events only; the chardev is unjustified
complexity.

→ Ch. 24, Ch. 29, Ch. 30

---

## 4. Design problem: kernel or userspace?

> *"We need to do X. Should it be in the kernel?"*

### The decision procedure

```
Does it require privileged hardware/state access?
 ├─ no ──► Does the crossing cost dominate at the required rate?
 │          ├─ no ──► USERSPACE. Done.
 │          └─ yes ─► Can it be batched or amortized?
 │                     ├─ yes ─► USERSPACE + io_uring / shared ring
 │                     └─ no ──► Is it policy or mechanism?
 │                                ├─ policy ──► eBPF (userspace-authored, kernel-resident)
 │                                └─ mech ───► KERNEL
 └─ yes ─► Can the hardware be safely delegated?
            ├─ yes ─► VFIO / UIO / SPDK / DPDK (userspace driver)
            └─ no ──► KERNEL
```

### The four criteria, restated

1. **Privilege.** Does it need what only ring 0 has?
2. **Performance.** Does the boundary crossing dominate? Quantify: ~200–500 ns/syscall.
3. **Policy vs mechanism.** Policy changes faster than an ABI can. Kernel policy is
   permanent policy.
4. **Blast radius.** No fault isolation, no ABI escape, and a bug is a panic or a root.

### The cases where the obvious answer is wrong

| Situation | Naive answer | Better |
|---|---|---|
| "It's too slow in userspace" | move to kernel | batch it; measure first. `io_uring` moved several of these back |
| "We need a custom packet filter" | kernel module | eBPF/XDP — same speed, no module, no ABI risk |
| "We need a custom filesystem" | in-kernel FS | FUSE first; move in-kernel only if you measure the need. The in-kernel version is a 10× maintenance commitment |
| "We need direct device access" | kernel driver | VFIO + userspace driver, if the device can be IOMMU-isolated and the consumer is trusted |
| "We need low-latency I/O" | kernel bypass | `io_uring` + poll queues gets most of it without abandoning the kernel's protections |
| "We need our own scheduler" | patch CFS | `sched_ext` (BPF schedulers) — this is exactly why it was merged |

### The principal move

Note that the question has a hidden third option — **the interface can be in the kernel while
the decision is in userspace**. That is what eBPF, `sched_ext`, `io_uring`, FUSE, and VFIO all
are. The classic dichotomy is 1990s framing; the modern design space is "where does the
mechanism live, and who supplies the policy at runtime?"

→ Ch. 00 §T.4, Ch. 24, Ch. 75

---

## 5. Design problem: a real-time partition on a general-purpose system

> *"One core must run a control loop with a 100 µs deadline. The other 15 cores run
> everything else."*

### Bind the requirement

100 µs is achievable on stock Linux with care; 10 µs needs `PREEMPT_RT`; 1 µs needs
dedicated hardware or a co-processor. Ask which, and ask whether it is a **hard** deadline
(a miss is a failure) or firm/soft. Ask for the WCET of the loop body — if they cannot give
one, the design cannot be validated and you should say so.

### The design, layer by layer

| Layer | Action | Why |
|---|---|---|
| **Kernel config** | `CONFIG_PREEMPT_RT` | Bounds the longest non-preemptible section; converts spinlocks to `rt_mutex` with PI |
| **CPU isolation** | `isolcpus=`, better: `cpuset` + `nohz_full=` + `rcu_nocbs=` | Removes the tick (jitter source), moves RCU callbacks off the core |
| **IRQ affinity** | move all IRQs off the isolated core; `irqaffinity=` | An unrelated NIC interrupt is a 10–50 µs hit |
| **Softirq/workqueue** | `WQ_SYSFS` affinity, `kthread` affinity | Same reason |
| **Scheduling** | `SCHED_DEADLINE` with measured WCET; or `SCHED_FIFO` if aperiodic | DEADLINE gives admission control + budget enforcement |
| **RT throttling** | understand `sched_rt_runtime_us`; disable only with a watchdog | Default caps RT at 95%, which will surprise you |
| **Memory** | `mlockall(MCL_CURRENT\|MCL_FUTURE)`, prefault the stack, no `malloc` in the loop | A major fault is 100 µs — your entire budget |
| **CPU frequency** | `performance` governor, disable deep C-states via `/dev/cpu_dma_latency` | C-state exit can be 100+ µs |
| **SMT** | disable, or leave the sibling idle | The sibling steals execution resources non-deterministically |
| **NUMA** | pin memory to the local node | Remote access is 1.5–2× and variable |
| **Firmware** | check for SMIs with `hwlatdetect` | SMM is invisible to Linux and can be milliseconds. This is the one you cannot fix in software |
| **Communication** | lock-free SPSC ring in shared memory to the non-RT side | Any lock shared with a non-RT task is an inversion path |

### Validation, which is half the answer

- `cyclictest -m -p99 -i100 -l10000000` for at least 24 hours under **representative load** —
  an idle system proves nothing.
- `hwlatdetect` to separate hardware (SMI) latency from software.
- `trace-cmd`/`osnoise` and `timerlat` tracers to attribute each outlier.
- Report the **maximum**, not the average. For a hard deadline, the mean is irrelevant.
- Track the histogram over time; a regression shows up as a new tail before it shows up as a
  miss.

### Failure modes to name

- Priority inversion via a lock shared with a non-RT thread → `rt_mutex`/PI, or don't share.
- Unbounded work in an RT context (allocation, page faults, `printk`).
- A `stop_machine` (module load, CPU hotplug, some static-key updates) — which preempts
  *everything*, including your RT task, for milliseconds. Forbid those operations at runtime.
- TLB shootdown IPIs from an unrelated process on another core.

→ Ch. 103, Ch. 21, Ch. 19

---

## 6. Design problem: a secure BSP for a hostile environment

> *"Embedded device, physically accessible by an attacker, must protect a key and must
> receive signed updates."*

### The chain

```
 ROM (immutable) ──verifies──► bootloader ──verifies──► kernel+DTB
     ──verifies──► rootfs (dm-verity) ──measures──► runtime (IMA)
```

Every link must be verified by the previous one, and the root of trust must be in
**immutable** storage (mask ROM or eFuse-locked). A chain with a writable root is not a chain.

### The layers

| Concern | Mechanism |
|---|---|
| Boot integrity | Secure/verified boot; signature in ROM; anti-rollback counters in eFuse |
| Kernel integrity at rest | Signed kernel image; `CONFIG_MODULE_SIG_FORCE` |
| Rootfs integrity | **dm-verity** — Merkle tree, verified per-block on read, so tampering is detected at use, not at boot |
| Runtime measurement | IMA/EVM with a TPM PCR log, for remote attestation |
| Key protection | TPM / secure element / TrustZone; **never** a key in the kernel image or rootfs |
| Runtime hardening | KASLR, KPTI, SMEP/SMAP/PAN/PXN, `CONFIG_STRICT_KERNEL_RWX`, `CONFIG_CFI_CLANG`, `init_on_alloc/free`, hardened usercopy |
| Attack surface reduction | `CONFIG_MODULES=n` if possible; `lockdown=confidentiality`; no debugfs, no kgdb, no `/dev/mem` |
| MAC policy | SELinux or AppArmor in enforcing mode, targeted at the app, not "permissive plus hope" |
| Syscall reduction | seccomp-BPF filter per service — the largest single reduction in exploitable surface |
| Untrusted devices | IOMMU in **strict** (not lazy) mode; this is the only defence against a malicious DMA-capable peripheral |
| Update | A/B partitions, signed, with a rollback-on-failure watchdog and an anti-rollback counter |
| Physical | disable JTAG/serial console in production fuses; encrypt storage (dm-crypt) with a TPM-sealed key |

### The judgement layer

State the threat model explicitly, because the list above is a *maximum* and every item
costs something:

- If the attacker has **physical** access with unlimited time, memory encryption and
  anti-tamper matter and dm-verity alone is insufficient (they can replace the whole image if
  the ROM is not locked).
- If the threat is **remote only**, seccomp + MAC + attack-surface reduction gives most of
  the value for a fraction of the cost.
- **CFI, KASLR, and `init_on_free` each cost 1–5%**; on a battery device that is a real
  tradeoff and should be a decision, not a default.
- **IOMMU strict mode** costs significant throughput (an invalidation per unmap). Lazy mode
  is fast and leaves a window. Name the window when you choose.

→ Ch. 102, Ch. 36, Ch. 49, Ch. 66

---

## 7. Design problem: scale a subsystem to 1000 cores

> *"Our subsystem tops out at 64 cores. Make it scale to 1000."*

### The universal scaling law

Amdahl bounds you by the serial fraction; the **Universal Scalability Law** adds a
*coherency* term that makes throughput go **down** past a peak:

$$C(N) = \frac{N}{1 + \alpha(N-1) + \beta N(N-1)}$$

where $\alpha$ is contention (serialization) and $\beta$ is coherency (crosstalk). The $\beta
N^2$ term is why adding cores past the peak makes things *slower* — and it is why "just make
the critical section shorter" eventually stops working. Being able to say this, and to say
that $\beta$ is cacheline ping-pong, is a strong staff/principal signal.

### The escalation ladder

| Step | Technique | Gets you to |
|---|---|---|
| 0 | Measure. `perf c2c`, `perf lock`, `lockstat` | knowing whether it is $\alpha$ or $\beta$ |
| 1 | Shrink the critical section | 2–4× |
| 2 | Split the lock by object / hash bucket | ~10s of cores |
| 3 | Reader-writer split (`rwsem`, `seqlock`) | helps only if reads dominate **and** readers don't write the lock word |
| 4 | **RCU** — readers do not write anything | hundreds, for read-mostly |
| 5 | **Per-CPU data + fold on read** | ~linear, if the read is rare (`percpu_counter`, `percpu_rwsem`) |
| 6 | Sharding by NUMA node, with a hierarchical rollup | crosses the socket boundary |
| 7 | **Eliminate the shared state** — redesign so nothing is shared | the only true answer at 1000 |

Steps 1–3 attack $\alpha$. Steps 4–7 attack $\beta$, which is the one that actually bounds
you at high core counts.

### The specific things that bite at 1000 cores

- **Any single atomic on a hot path.** Even an uncontended `atomic_inc` on a shared line is a
  cacheline transfer — `percpu_counter` exists for exactly this.
- **IPIs.** TLB shootdown, `on_each_cpu()`, `stop_machine()` all scale with N and serialize.
- **`for_each_cpu()` loops** on a fast path become O(N).
- **Wakeup storms** — a single `wake_up()` on a queue with 1000 waiters is a thundering herd.
  Use `wake_up_nr` / exclusive waits.
- **Cross-socket memory** — at 1000 cores you have 8+ sockets; a remote access is 2–3× and a
  remote atomic far worse.
- **Timer and RCU callback offload** — `rcu_nocbs` matters at this scale.
- **The allocator** itself — SLUB per-CPU caches are fine; the buddy allocator's zone lock is
  not, hence per-CPU page lists (PCP).

### Commit

"I would spend the first week on `perf c2c` to find the top three cachelines, because at this
scale the answer is almost always a small number of specific lines, not a diffuse problem. I
would expect step 5 or 7 to be the real fix, and I would treat steps 1–3 as measurement
instruments rather than solutions."

→ Ch. 105, Ch. 16, Ch. 15, Ch. 13

---

## 8. Design problem: multi-tenant isolation

> *"Many untrusted tenants on one kernel. Guarantee isolation."*

### Start by rejecting the premise, carefully

"Guarantee" is the wrong word for a shared kernel. A shared kernel means a shared attack
surface of ~400 syscalls and a shared set of hardware resources with shared microarchitectural
state. If the requirement is *guarantee*, the answer is VMs (or Kata / gVisor / Firecracker),
not containers. If the requirement is *strong practical isolation with container economics*,
here is the design. **Saying this distinction out loud is the point of the question.**

### Resource isolation (cgroup v2)

| Resource | Control | Gap |
|---|---|---|
| CPU | `cpu.max` (bandwidth), `cpu.weight` (share) | `cpu.max` throttling causes p99 spikes at period boundaries |
| Memory | `memory.max`, `memory.high`, `memory.min` | Kernel memory counted in v2; page cache accounting is charge-on-first-touch, so a shared file is charged to whoever touched it first |
| I/O | `io.max`, `io.weight` (needs BFQ or `iocost`) | Only works for block; buffered writes are charged at writeback time to the wrong cgroup unless memcg writeback tracking is on |
| PIDs | `pids.max` | — |
| Network | tc + `net_cls`/eBPF cgroup hooks | No first-class cgroup v2 net controller |
| **Cache / memory bandwidth** | **RDT/MPAM** (`resctrl`) | This is the one people forget — LLC is otherwise entirely unisolated |

### Security isolation

- **Namespaces** for visibility (pid, mount, net, user, uts, ipc, time, cgroup).
- **User namespaces** so root-in-container is not root-on-host — and note that user
  namespaces have themselves been a CVE source, because they expose previously
  root-only code paths to unprivileged users.
- **seccomp-BPF** to reduce the syscall surface; this is the highest-leverage single control.
- **LSM** (SELinux/AppArmor) for mandatory policy.
- **`no_new_privs`**, dropped capabilities, read-only rootfs.

### The leaks you must name

1. **Shared kernel = shared bugs.** One LPE escapes everything. Mitigate with seccomp and
   kernel hardening; eliminate only with a VM boundary.
2. **Microarchitectural side channels** — LLC, TLB, branch predictors, and SMT siblings.
   Core scheduling (`prctl(PR_SCHED_CORE)`) prevents cross-tenant SMT co-residency at a
   throughput cost.
3. **Shared kernel resources without cgroup controllers**: inode/dentry cache, the page
   allocator's fragmentation state, netfilter conntrack table, file descriptor tables,
   futex hash buckets. A tenant can degrade neighbours through any of these.
4. **`/proc` and `/sys` leakage** — many files are not namespaced and leak host information
   or allow host-wide effects.
5. **Timing.** Anything shared is a covert channel. If the requirement includes "no covert
   channels," only physical separation works.

### Commit

"Containers with seccomp + user namespaces + full cgroup v2 + RDT for LLC, **for mutually
non-hostile tenants**. For hostile tenants, a microVM per tenant — the ~125 ms boot and
~5 MiB overhead of Firecracker has made the economic argument for containers-as-a-security-
boundary much weaker than it was in 2016."

→ Ch. 102, Ch. 104, Ch. 105

---

## 9. Design problem: evolve an interface without breaking anything

> *"We shipped an ioctl three years ago. We need to add two fields and change a default.
> Go."*

### The rules

1. **The userspace ABI is inviolable.** "We don't break userspace" is not a guideline. If a
   change breaks a working program, the change is wrong, no matter how wrong the original
   was.
2. **Hyrum's Law applies to everything observable**: timing, error codes, ordering, the
   contents of padding, even bugs. Someone depends on the bug.
3. Therefore: **add, never change.**

### The mechanics

| Need | Technique |
|---|---|
| Add fields | If the original struct had a `size` field → `copy_struct_from_user()`, zero-extend old structs, reject non-zero unknown tail. If it did not → new ioctl number |
| Change a default | **Never.** Add an opt-in flag. The old default is now permanent |
| Add behaviour | New flag bit, but **only if the original rejected unknown flags**. If it ignored them, that bit is unusable — you need a new ioctl |
| Deprecate | Mark in docs, add a `pr_warn_once` with the task name, keep it working. Removal requires years of no users, and often never happens |
| Fix a genuine bug userspace depends on | Keep the bug for existing callers; gate the fix behind a new flag or a new interface |

### The `copy_struct_from_user` pattern, which is the answer to most of this

```c
struct my_args_v2 {
	__u32 size;      /* set by userspace to sizeof(*args) */
	__u32 flags;
	__u64 old_field;
	__u64 new_field; /* added in v2 */
};

err = copy_struct_from_user(&args, sizeof(args), uarg, usize);
/* Returns -E2BIG if userspace sent a larger struct with NON-ZERO tail.
 * Zero-extends if userspace sent a smaller (older) struct.
 * => forward AND backward compatible in one call.               */
if (err)
	return err;
if (args.flags & ~MY_VALID_FLAGS)
	return -EINVAL;
```

This is what `clone3`, `openat2`, `sched_setattr`, and `bpf()` do. It is the single most
important UAPI idiom, and knowing it by name is a strong signal.

### The principal framing

The real lesson is about the *original* design: this problem is entirely created by not
having a size field and not rejecting unknown flags on day one. Two lines of code three
years ago would have made this a non-question. Say that — designing for the change you
cannot foresee is the actual skill being tested.

→ Ch. 24

---

## 10. Design problem: observability for a subsystem you own

> *"How would you make your subsystem debuggable in production, where you cannot attach a
> debugger?"*

### The layers, cheapest first

| Layer | Cost when off | Use |
|---|---|---|
| **Static keys / tracepoints** | a NOP — literally zero | always-available, enable on demand |
| **Counters** (`percpu_counter`, `/sys` or `/proc`) | one per-CPU increment | rates, errors, retries — the first thing anyone looks at |
| **Histograms** (trace event triggers, `bpftrace`) | zero when off | latency distributions — the mean is a lie, you need p99 |
| **debugfs state dump** | zero | full internal state on demand; no ABI commitment |
| **`WARN_ON_ONCE` + `pr_ratelimited`** | a predicted-not-taken branch | invariant violations |
| **kprobes/fprobes** | zero | anything you forgot to instrument |
| **`CONFIG_*_DEBUG` validators** | compiled out | contract checking in CI, not production |

### The rules

1. **Instrument the *boundaries*, not the middle.** Entry/exit of each layer, every error
   return, every retry, every queue depth change. That gives you a flow graph you can
   reconstruct.
2. **Every error path gets a counter, and every counter gets a name that says what to do.**
   `tx_dropped_no_buffers` is actionable; `errors` is not.
3. **Expose queue depths and latencies, not just totals.** A total tells you it is slow; a
   depth tells you where.
4. **Correlate.** Emit a request ID through every tracepoint so a slow request can be
   followed across subsystems. This is the thing most kernel subsystems do worst.
5. **Make the "what is it doing right now" dump possible** — debugfs with the full state,
   because in production the question is usually "it is stuck" not "it is slow."
6. **Never use `printk` on a hot path.** It serializes on the console and has caused more
   production outages than the bugs it was added to diagnose.

### The test

"If I get a bug report saying 'it is slow sometimes' with no reproducer, can I answer it from
what the machine already exports?" If not, the instrumentation is incomplete. Design to that
question, not to the bugs you can already reproduce.

→ Ch. 06, Ch. 50, `reference/debugging-scenarios.md`

---

## 11. Rehearsal problems

Work these to a whiteboard in 40 minutes each, using the six moves.

1. Design a driver framework for a new bus type (say, a proprietary serial bus with 200
   device types). What does the core own, what does each driver own?
2. Design the kernel side of a confidential-computing guest: the guest must not trust the
   hypervisor. What changes in DMA, in page management, in the device model?
3. You must support a device whose firmware occasionally hangs. Design the recovery path so
   that no userspace application ever sees an error it cannot retry.
4. Design a page-cache policy for a machine with 2 TiB of DRAM and 20 TiB of CXL-attached
   memory at 3× the latency.
5. A customer needs a filesystem that survives sudden power loss on cheap flash with no
   power-loss protection. What do you build or choose, and what do you refuse to promise?
6. Design the interface between a kernel driver and a userspace driver for the same device,
   such that both can exist and neither can corrupt the other.
7. 10 Gbps of small packets must be filtered against a 1M-entry blocklist with a 5 µs budget.
   Where does the code live, and what data structure?
8. Your subsystem has 60 out-of-tree vendor forks. Design the upstream interface that would
   let 50 of them delete their fork.
9. Design a mechanism to hot-patch a kernel function safely, and enumerate what makes a
   function un-patchable.
10. A fleet of 100,000 machines must be upgraded from 5.15 to 6.12. Design the rollout,
    including how you detect a regression that only appears on 1 in 1000 machines.

---

## 12. Scoring yourself

| Signal | Weak | Strong |
|---|---|---|
| Opening | Starts drawing immediately | Asks the three binding questions first |
| Numbers | Qualitative only | Derives constraints arithmetically ("100 ns ⇒ no syscall") |
| Alternatives | Presents one design | Presents three with explicit prices, including "do nothing" |
| Commitment | "It depends" | "B, and I'd switch to A if X" |
| Failure modes | Absent | Enumerated, each with a detection method |
| Observability | Absent | "Here is the counter I would add" |
| Premise | Accepts it | Reframes when the premise is wrong, and says why |
| Cost honesty | Only benefits | Names what the chosen design gives up |

The single strongest move available to you in this round is **naming what your design costs
before being asked**. It is the difference between someone selling a design and someone who
has operated one.

→ Next: [debugging-scenarios.md](debugging-scenarios.md)

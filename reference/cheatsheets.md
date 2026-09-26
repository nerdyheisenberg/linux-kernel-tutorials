# Cheat Sheets

> Dense, printable reference for things you need under pressure: in an interview, at a
> whiteboard, or at 3 a.m. with a production incident. Everything here is derived from the
> chapters; the chapter reference follows each section.

---

## 1. Latency numbers to memorize

| Operation | Time | Relative |
|---|---|---|
| L1 cache reference | 1 ns | 1× |
| Branch mispredict | 3 ns | |
| L2 cache reference | 4 ns | |
| L3 cache (local socket) | 15–20 ns | |
| **vDSO `clock_gettime`** | **20–25 ns** | |
| Mutex lock/unlock (uncontended) | 17 ns | |
| **Local DRAM** | **~100 ns** | 100× |
| Remote DRAM (1 hop) | 130–200 ns | |
| **Cacheline HITM (cross-socket)** | **200–400 ns** | |
| **Syscall (no mitigations)** | **~60 ns** | |
| **Syscall (with mitigations)** | **200–500 ns** | |
| **VM exit round trip** | **~300–500 ns** | |
| Compress 1 KB (Snappy) | 2 µs | |
| **Context switch (direct)** | **1–2 µs** | |
| Context switch (incl. cache pollution) | 5–50 µs | |
| **TLB shootdown IPI** | **1–10 µs** | |
| Send 1 KB over 10 GbE | 1 µs | |
| NVMe random read | 20–80 µs | |
| Read 1 MB sequentially from RAM | 50 µs | |
| Read 1 MB sequentially from NVMe | 150 µs | |
| **Device cache flush (NVMe)** | **20–500 µs** | |
| Datacenter round trip | 500 µs | |
| Read 1 MB from HDD | 5 ms | |
| **HDD seek** | **8–15 ms** | |
| CA → Netherlands → CA | 150 ms | |

**Derived facts to state:**
- A syscall costs ~1000 L1 hits → the argument for vDSO, `io_uring`, batching.
- HDD random/sequential ratio ~1000×; NVMe's ~2× → every seek-avoidance optimization is
  overhead on flash.
- Cacheline contention costs more than remote DRAM → contention, not latency, bounds
  scalability.

*(Ch. 51 §T.7, Ch. 102 §T.7, Ch. 105 §T.1)*

---

## 2. Choosing a synchronization primitive

```
 Can the code sleep here?
 ├─ NO (IRQ, spinlock held, preempt off)
 │   ├─ shared with hardirq on this CPU?  -> spin_lock_irqsave
 │   ├─ shared with softirq?              -> spin_lock_bh
 │   └─ otherwise                          -> spin_lock
 └─ YES
     ├─ read-mostly, pointer-reachable?    -> RCU
     ├─ read-mostly, readers may sleep?    -> SRCU
     ├─ small data, writes rare, retryable -> seqlock
     ├─ reads dominate, need blocking      -> rw_semaphore
     ├─ per-CPU data                       -> per-CPU + local_lock
     ├─ counter                            -> percpu_counter / atomic_t
     ├─ one-shot event                     -> completion
     ├─ condition wait                     -> wait_event + wake_up
     └─ otherwise                          -> mutex
```

| Primitive | Sleeps? | Owner? | Counting? | C type |
|---|---|---|---|---|
| Spinlock | busy-wait | implicit | no | `spinlock_t` |
| Mutex | yes | **yes** | no | `struct mutex` |
| Semaphore | yes | no | **yes** | `struct semaphore` |
| Completion | yes | no | signalling | `struct completion` |
| RW semaphore | yes | shared/excl | — | `struct rw_semaphore` |
| Seqlock | reader retries | writer-preferring | — | `seqlock_t` |
| RCU | readers never | — | — | — |

**Mutex vs binary semaphore:** a mutex has an **owner**, enabling PI, deadlock detection, and
"only the locker may unlock." A semaphore is a signalling counter, postable by anyone.

*(Ch. 14, Ch. 25, Ch. 81)*

---

## 3. GFP flags

| Flag | May sleep? | Reserves? | Use |
|---|---|---|---|
| `GFP_KERNEL` | **yes** | no | Normal process context |
| `GFP_ATOMIC` | no | **yes** | IRQ context, spinlock held |
| `GFP_NOWAIT` | no | no | Best-effort, no reserves |
| `GFP_NOIO` | yes | no | In the I/O path; no recursion into I/O |
| `GFP_NOFS` | yes | no | In the filesystem path |
| `GFP_DMA` / `GFP_DMA32` | — | — | Address-limited zones |
| `GFP_HIGHUSER_MOVABLE` | yes | no | Userspace pages |
| `__GFP_ZERO` | — | — | Zeroed |
| `__GFP_NOWARN` | — | — | Suppress the OOM message |
| `__GFP_NORETRY` | — | — | Fail rather than retry hard |

`kmalloc` (contiguous, size-limited) vs `vmalloc` (virtual, large, TLB cost) vs `kvmalloc`
(try then fall back).

*(Ch. 11)*

---

## 4. Memory barriers

| Barrier | Orders |
|---|---|
| `smp_mb()` | all loads+stores before ↔ all after |
| `smp_rmb()` | loads before ↔ loads after |
| `smp_wmb()` | stores before ↔ stores after |
| `smp_store_release(p, v)` | everything before ↔ this store |
| `smp_load_acquire(p)` | this load ↔ everything after |
| `smp_mb__before_atomic()` | before a non-returning RMW |
| `dma_wmb()` / `dma_rmb()` | **CPU ↔ device**, for descriptor memory |
| `mb()` / `rmb()` / `wmb()` | Including MMIO; heavier |
| `barrier()` | compiler only |

**The publication pattern:**
```c
/* writer */  init(obj); smp_store_release(&ptr, obj);
/* reader */  obj = smp_load_acquire(&ptr); if (obj) use(obj);
```
RCU spells these `rcu_assign_pointer` / `rcu_dereference`.

**Coherence ≠ ordering.** Coherence agrees on the order of writes to *one* location; barriers
constrain the relative order of accesses to *different* locations.

*(Ch. 13, Ch. 15)*

---

## 5. Reading an oops

Read in this order:

1. **The reason line** — "NULL pointer dereference, address: 0x28" → offset 40 from NULL, so
   a member of a NULL struct, not a NULL argument.
2. **`error_code`** — bit 0: 0=not-present/1=protection; bit 1: 0=read/1=**write**;
   bit 2: 0=kernel/1=user; bit 4: instruction fetch.
3. **`RIP`/`PC`** → `faddr2line module.ko func+0x2f`.
4. **`Tainted:`** — `G`=clean, `P`=proprietary, `O`=out-of-tree, `D`=previous oops,
   `W`=warning, `U`=user-forced.
5. **Context** — `Workqueue:`, `Hardware name:`, which CPU, which comm.
6. **`Code:`** — `<>` marks the faulting instruction.
7. **Call Trace** — last, and treat `?` entries as stale stack noise.

*(Ch. 06, `debugging-scenarios.md` §1)*

---

## 6. Turning silent failures loud

| Silent failure | Enable |
|---|---|
| UAF, OOB | `CONFIG_KASAN` |
| Uninitialized memory | `CONFIG_KMSAN` |
| Data race | `CONFIG_KCSAN` |
| Undefined behaviour | `CONFIG_UBSAN` |
| Lock-order inversion | `CONFIG_PROVE_LOCKING` |
| Sleeping in atomic | `CONFIG_DEBUG_ATOMIC_SLEEP` |
| RCU misuse | `CONFIG_PROVE_RCU` |
| Bad DMA usage | `CONFIG_DMA_API_DEBUG` + `iommu.strict=1` |
| Slab corruption | `slub_debug=FZP` |
| Freed-page use | `init_on_free=1`, `CONFIG_DEBUG_PAGEALLOC` |
| List corruption | `CONFIG_DEBUG_LIST` |
| Stack overflow | `CONFIG_VMAP_STACK` |
| Memory leak | `CONFIG_DEBUG_KMEMLEAK`, `/proc/allocinfo` |
| Long non-preemptible region | `irqsoff` tracer |
| Hardware/SMI latency | `hwlatdetect`, `rtla osnoise` |
| Production-safe memory bugs | **`CONFIG_KFENCE`** (~0% overhead) |

*(Ch. 06, `debugging-scenarios.md` §0)*

---

## 7. The USE method

For every resource: **U**tilization, **S**aturation, **E**rrors.

| Resource | Utilization | Saturation | Errors |
|---|---|---|---|
| CPU | `mpstat` | run queue, PSI `cpu` | MCEs |
| Memory | `MemAvailable` | swap, PSI `memory`, `allocstall` | OOM kills |
| Disk | `iostat %util` | `aqu-sz`, `await`, PSI `io` | `dmesg`, SMART |
| Network | `sar -n DEV` | retransmits, drops | `ip -s link` |
| Locks | `lock_stat` | contention count | — |
| Tags | `hctx*/tags` | full events | timeouts |

**Then:** queueing or service time? `await` high + `svctm` low = queued (fix concurrency);
service time high = fix the device or the work.

**And:** on-CPU or off-CPU? Idle CPU + slow = off-CPU analysis, not profiling.

*(Ch. 95 §T.7)*

---

## 8. Tool selection

| Question | Tool |
|---|---|
| Why is the CPU busy? | `perf record -g` + flame graph |
| Slow with an idle CPU? | `offcputime`, PSI |
| Which cacheline is contended? | `perf c2c` |
| Lock contention? | `perf lock`, `/proc/lock_stat` |
| Latency distribution? | `funclatency`, `bpftrace hist()` |
| What is it doing right now? | `echo l > sysrq-trigger`, `drgn` live |
| Why did it panic? | kdump + `crash` or `drgn` |
| Block I/O? | `biolatency`, `biosnoop`, `blktrace` |
| Longest IRQ-off region? | `irqsoff` tracer |
| RT latency attribution? | `rtla timerlat`, `rtla osnoise` |
| Memory growth? | `/proc/allocinfo`, `slabtop`, `kmemleak` |
| No tools available? | `/sys/kernel/tracing` with `cat`/`echo` |

*(Ch. 95 §T.9)*

---

## 9. Kernel command line, bring-up edition

```
earlycon=uart8250,mmio32,0x10000000,115200n8   # EXPLICIT; works pre-DT
console=ttyS0,115200
keep_bootcon                 # do not drop early output at handover
initcall_debug               # which initcall hangs
ignore_loglevel
rdinit=/bin/sh               # stop IN the initramfs
init=/bin/sh                 # bypass init entirely
root=/dev/mmcblk0p2 rootwait rootdelay=10
nokaslr                      # for debugging with fixed addresses
mitigations=off              # measure the mitigation tax
maxcpus=1                    # eliminate SMP as a variable
```

Production/RT:
```
isolcpus=3 nohz_full=3 rcu_nocbs=3 irqaffinity=0-2
crashkernel=256M panic=1 softlockup_panic=1
iommu=force intel_iommu=on iommu.strict=1
lockdown=integrity
```

*(Ch. 90, Ch. 95, Ch. 103)*

---

## 10. Debugfs and procfs map

```
/sys/kernel/debug/
  tracing/                  ftrace: events/, current_tracer, trace_pipe
  kvm/<pid>-<n>/vcpu*/      VM exit counters
  block/<dev>/hctx*/        blk-mq: tags, busy, dispatch
  clk/clk_summary           the clock tree
  pinctrl/*/pinmux-pins     pin mux state
  regulator/regulator_summary
  devices_deferred          probe failures      <- CHECK THIS FIRST
  dma-api/error_count
  kmemleak
  io_uring/
  pm_genpd/pm_genpd_summary
  dynamic_debug/control

/proc/
  cmdline  interrupts  iomem  meminfo  vmstat  slabinfo
  pressure/{cpu,memory,io}    PSI -- stall time, the metric that matters
  lock_stat                   needs CONFIG_LOCK_STAT
  allocinfo                   per-call-site allocation (6.10+)
  self/{maps,smaps,status,numa_maps,auxv,mountinfo}
  sys/kernel/{kptr_restrict,perf_event_paranoid,printk}

/sys/
  fs/cgroup/                  cgroup v2
  devices/system/cpu/vulnerabilities/
  kernel/security/{lsm,lockdown}
  firmware/devicetree/base/   the RUNTIME device tree
  block/<dev>/queue/          scheduler, nr_requests, read_ahead_kb
  kernel/btf/vmlinux          BTF, for CO-RE
```

---

## 11. Git and patch workflow

```bash
# Setup
git config --global format.signOff true
git config --global core.abbrev 12
git config --global alias.fixes "log -1 --abbrev=12 --format='Fixes: %h (\"%s\")'"

# Produce a clean series
git add -p                                  # split changes into logical commits
git rebase -i <base>
git rebase <base> --exec 'make -j$(nproc)'  # EVERY commit builds
./scripts/checkpatch.pl --strict -g <base>..HEAD

# Send
b4 prep -n my-feature -f v6.12
b4 prep --edit-cover && b4 prep --auto-to-cc
b4 send --dry-run && b4 send
b4 trailers -u                              # collect Reviewed-by into commits

# Receive / review
b4 am -o /tmp <msgid>
b4 shazam <msgid>
b4 diff <msgid>                             # v2 vs v3

# Investigate
git log -S'symbol' --oneline
git log -L :func:file.c
git bisect start bad good && git bisect run ./test.sh
git cherry -v <upstream> <mine>             # "-" = already upstream, DROP IT
```

**Trailers:** `Signed-off-by` (legal), `Reviewed-by` (I checked it), `Acked-by` (I own this
area and I'm fine), `Tested-by`, `Fixes:` (machine-read for stable), `Cc: stable@...`.

*(Ch. 86)*

---

## 12. Rust for kernel C developers

| C | Rust |
|---|---|
| `NULL` return | `Option<T>` |
| `ERR_PTR`/`IS_ERR` | `Result<T, Error>` |
| `goto err` ladder | `?` + `Drop` |
| `kmalloc(sz, GFP_KERNEL)` | `KBox::new(x, GFP_KERNEL)?` |
| `kref` | `Arc<T>` / `ARef<T>` |
| `void *drvdata` | `ForeignOwnable` |
| `__user` (sparse) | `UserSlice` (type) |
| `__iomem` (sparse) | `IoMem<SIZE>` (no `Deref`) |
| `*_ops` vtable | `#[vtable] trait` |
| "caller holds lock" (comment) | data lives *inside* `Mutex<T>` |
| `container_of` | `HasListLinks` + `offset_of!` |
| `devm_*` | ownership + `Drop`, or `Devres<T>` |

**`unsafe` unlocks exactly five things** and disables *none* of the borrow checker:
deref a raw pointer, call an `unsafe fn`, touch a mutable `static`, `unsafe impl` a trait,
read a union field.

**Every `unsafe fn` needs `# Safety`; every `unsafe {}` needs `// SAFETY:`.**

**Eliminated:** UAF, double free, OOB, data races, uninitialized use, leaked resources on
error paths. **Not eliminated:** deadlock, sleeping-in-atomic, lock ordering, DMA ownership,
memory ordering, leaks.

*(Ch. 77–85)*

---

## 13. Yocto quick reference

```bash
source oe-init-build-env
bitbake <image>
bitbake -c devshell <recipe>      # shell in the cross environment
bitbake -c menuconfig virtual/kernel
bitbake -c diffconfig virtual/kernel   # changes -> a .cfg FRAGMENT
bitbake -e <recipe> | grep -B20 '^# \$VAR'   # WHO SET THIS VARIABLE
bitbake-diffsigs <old> <new>      # why did it rebuild?
bitbake-whatchanged <image>
bitbake-layers show-appends / show-overlayed / create-layer
oe-pkgdata-util find-path /usr/bin/foo
devtool modify <recipe> / build / deploy-target / finish
runqemu <machine> <image> nographic
```

| Operator | When applied |
|---|---|
| `=` lazy, `:=` immediate, `?=` default | parse |
| `+=` `.=` | **parse order — avoid in .bbappend** |
| `:append` `:prepend` `:remove` | **after all parsing — use these** |

**In a `.bbappend` always use `:append` with a leading space.**

```bash
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"   # the required idiom
```

`DEPENDS` = build time; `RDEPENDS` = runtime. `MACHINE` = hardware, `DISTRO` = policy,
`IMAGE` = contents.

*(Ch. 96–99)*

---

## 14. Classic OS theory, condensed

**Coffman's four deadlock conditions:** mutual exclusion, hold-and-wait, no preemption,
**circular wait** ← Linux breaks this one, with lockdep enforcing it.

**Belady's anomaly:** FIFO can fault *more* with *more* frames. `1,2,3,4,1,2,5,1,2,3,4,5`
gives 9 faults with 3 frames, 10 with 4. Stack algorithms (LRU, OPT) cannot exhibit it.

**Scheduling:** SJF minimizes mean wait (needs the future); RR bounds response;
MLFQ approximates SJF without knowing job length; **EDF optimal (U ≤ 1)**, RM bound
U ≤ n(2^(1/n)−1) → ln 2.

**Priority inversion:** H blocks on L's lock; M preempts L; H waits behind M, unbounded.
Fixes: priority **inheritance** (Linux `rt_mutex`), priority **ceiling**.

**Working set** W(t,τ) = pages referenced in the last τ. Thrashing is a *phase transition*,
not gradual.

**Popek–Goldberg:** virtualizable iff sensitive ⊆ privileged. x86 had 17 sensitive-but-
unprivileged instructions.

**USL:** C(N) = N / (1 + α(N−1) + βN(N−1)). α = contention, β = coherency. The βN² term makes
throughput *decrease*.

*(`os-fundamentals.md`, Ch. 105)*

---

## 15. Security posture check

```bash
grep . /sys/devices/system/cpu/vulnerabilities/*
cat /sys/kernel/security/lsm
cat /sys/kernel/security/lockdown
grep -E 'Seccomp|NoNewPrivs|Cap' /proc/self/status
capsh --decode=$(grep CapEff /proc/self/status | cut -f2)
sysctl kernel.{kptr_restrict,dmesg_restrict,perf_event_paranoid,unprivileged_bpf_disabled}
sysctl kernel.io_uring_disabled
checksec --file=/bin/ls
kernel-hardening-checker -c /boot/config-$(uname -r)
systemd-analyze security
```

**The exploitation pipeline and what breaks each stage:** bug → primitive → defeat KASLR →
heap groom → overwrite → hijack → escalate.
`KASAN`/Rust → `FORTIFY`/hardened usercopy → `kptr_restrict` → freelist hardening →
`__ro_after_init` → SMEP/SMAP/CFI → LSM.

**Seccomp does not filter `io_uring` operations** — a sandbox must block `io_uring_setup`.

*(Ch. 102, Ch. 76 §T.7)*

---

## 16. Interview frameworks

**Answer in three layers:** mechanism → forces → alternative and its cost.

**Kernel or userspace:** privilege? → crossing cost dominates? → batchable? → policy or
mechanism? → blast radius. Third option: mechanism in kernel, policy from userspace (eBPF,
`sched_ext`, `io_uring`, FUSE, VFIO).

**API design:** does it need to exist → smallest interface → extensible (size field +
`copy_struct_from_user`) → return an **fd** → **reject unknown flags** → `*at()` form →
explicit widths, no padding → what am I committing to forever.

**Abstraction evaluation:** three users? net-negative diff? impossible-to-misuse or merely
inconvenient? what is the escape hatch?

**System design, six moves:** bind the requirement → draw the boundaries → separate
mechanism/policy → three options priced → commit with conditions → failure modes and
observability.

**The strongest single move:** name what your design costs *before* being asked.

*(Ch. 89, `system-design.md`, `interview-playbook.md`)*

---

## 17. Fifteen things to be able to say in 60 seconds

1. Four Coffman conditions; Linux breaks circular wait.
2. Exact LRU is unimplementable; Linux uses a 2Q-like active/inactive split plus refault
   distance.
3. Belady's anomaly, with the reference string.
4. Mutex has an owner; semaphore does not — hence PI and deadlock detection.
5. EDF is optimal; RM is used because EDF's overload failure is a domino effect.
6. Priority inversion, PI and PCP, Mars Pathfinder.
7. Paging beat segmentation because fixed-size units eliminate external fragmentation.
8. Working set; thrashing is a phase transition with positive feedback.
9. Inode indirect blocks: 12 direct, 4 MiB single, 4 GiB double, 4 TiB triple.
10. No hard links to directories: the graph must stay a tree.
11. Popek–Goldberg; x86 failed with 17 instructions.
12. Journalling modes: journal / ordered / writeback, and what each risks.
13. RAID-5 write hole; three fixes.
14. Context switch costs more than register save/restore because of cache and TLB pollution.
15. ACL is indexed by object, capability by subject; fds are capabilities.

*(`os-fundamentals.md` §12)*

→ Back: [../README.md](../README.md) | Next: [glossary.md](glossary.md)

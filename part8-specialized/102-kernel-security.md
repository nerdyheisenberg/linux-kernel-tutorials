# Chapter 102 — Kernel Security Architecture

> **Part 8 — Specialized domains.** These four chapters cover subjects that are referenced
> from a dozen earlier chapters but owned by none. Security is the clearest case: you have
> met `capable()` in Ch. 24, IOMMU isolation in Ch. 36, and the BPF verifier in Ch. 75, but
> nowhere has the *architecture* been laid out — what the threat model is, where the
> boundaries are, and why each mitigation costs what it costs.

---

## Theory & First Principles

### T.0 — Start here: one bug is enough

```c
/* An off-by-one in an ioctl handler in a driver nobody uses. */
if (idx > ARRAY_SIZE(tbl))       /* should be >= */
	tbl[idx] = user_value;   /* one element past the end */
```

**That is a full system compromise.** Not "a crash" — root, in every container on the box,
with the ability to disable every security mechanism above it. Because there is no privilege
boundary *inside* the kernel:

```
   USERSPACE           process A | process B | container C | VM guest
   ====================== the ONLY boundary that matters ==============
   KERNEL              drivers, filesystems, network stack, schedulers
                       ~30,000,000 lines, ALL equally privileged.

   An obscure USB driver has exactly the same power as the page allocator.
```

**This is the monolithic kernel's central security property, and it is why kernel security
looks nothing like application security.** You cannot sandbox a driver. You cannot give the
network stack fewer privileges than the scheduler. **The entire kernel is one trust domain.**

**So the strategy cannot be "have no bugs." It has to be defence in depth**, and the honest
framing is this:

> **Assume you have memory-safety bugs — because in 30 million lines of C, you do (Ch. 77
> §T.0). Then make each individual bug as hard as possible to turn into an exploit.**

**An exploit is a *chain*, and every mitigation attacks one link:**

| Step the attacker needs | The mitigation that breaks it |
|---|---|
| Find a memory bug | fuzzing (syzkaller), KASAN, Rust, static analysis |
| **Learn a kernel address** | **KASLR**, `%pK`, `kptr_restrict`, no addresses in dmesg |
| Place attacker data where the bug writes | **heap hardening**, freelist randomization, `CONFIG_SLAB_FREELIST_HARDENED` |
| Redirect control flow | **CFI**, `-fstack-protector`, shadow stacks, IBT |
| Execute data as code | **SMEP** (no executing user pages in ring 0) |
| Read/write user memory directly | **SMAP** / PAN |
| Escalate once running | seccomp, LSM, capabilities, namespaces (Ch. 93) |

**Notice the shape: no single row is sufficient, and none is reliable alone.** KASLR is
defeated by any information leak. CFI is bypassed by data-only attacks. SMEP is bypassed by
ROP. **The point of the stack is that the attacker must break *all* of them, and each one
multiplies the cost and the required bug quality.** Security economics, not absolutes.

**The most important row is the second one, and it is the least intuitive.** An *information
leak* — a few uninitialized bytes copied to userspace, a pointer printed in a log — looks
harmless and is the enabling primitive for nearly every modern exploit. **Without an address,
an attacker cannot aim.** This is why `copy_to_user` of a partially-initialized struct is
treated as a serious bug (Ch. 24 §T.0), and why "but it is only a leak" is wrong.

**Two structural observations to carry:**

1. **The attack surface is mostly drivers and parsers.** Anything that consumes
   attacker-controlled bytes — a filesystem mounted from a USB stick, a USB descriptor, a
   network protocol, an `ioctl` argument — is where the bugs are. **Trust boundaries are
   where validation belongs** (Ch. 65 §T.0), and the surface is far larger than people assume
   because *mounting an untrusted filesystem is running its parser as root*.
2. **Hardware is part of your threat model now.** Meltdown, Spectre, L1TF, MDS, and
   Retbleed broke the *architectural* guarantee that privilege separation is enforced by the
   CPU. The mitigations (KPTI, retpolines, IBRS) cost 5–30% of performance, and they are why
   a syscall went from ~100 ns to ~1 µs (Ch. 76 §T.0). **A microarchitectural optimization
   became a security vulnerability** — and the lesson generalizes: any optimization that makes
   timing depend on secret data is a channel.

```bash
grep . /sys/devices/system/cpu/vulnerabilities/*
sysctl kernel.kptr_restrict kernel.dmesg_restrict kernel.unprivileged_bpf_disabled
cat /proc/sys/kernel/randomize_va_space
grep -E 'CONFIG_(CFI|STACKPROTECTOR|FORTIFY|RANDSTRUCT|INIT_ON_ALLOC)' /boot/config-$(uname -r)
```

---

### T.1 — What is the kernel actually defending?

Start with the asset list, because every subsequent decision follows from it:

| Asset | Threat | Boundary |
|---|---|---|
| **Kernel integrity** | Code injection, ROP, data-only attacks | user→kernel (syscall, ioctl, driver) |
| **Kernel confidentiality** | Info leak → defeats KASLR → enables the above | same, plus side channels |
| **Inter-process isolation** | One process reads/writes another | address space, credentials, namespaces |
| **Host from device** | A malicious/compromised peripheral DMAs anywhere | IOMMU |
| **Host from guest** | VM escape | hypervisor |
| **Guest from host** | Confidential computing inverts the usual trust | SEV-SNP / TDX / CCA |
| **Boot-time integrity** | Persistent rootkit | Secure Boot, lockdown |

The kernel's central structural problem: **there is no fault isolation inside it.** Any
write primitive anywhere in 30 million lines is a write primitive everywhere. A userspace
process has the kernel between it and disaster; the kernel has nothing. This is the single
fact that makes kernel security qualitatively different, and it is why the strategy is
layered *mitigation* (make exploitation expensive and probabilistic) rather than
*prevention* (make bugs impossible), which nobody knows how to do at this scale.

### T.2 — The threat models, named

Be precise about which you are defending against; the mitigations differ sharply.

1. **Unprivileged local user → root.** The dominant model. Attack surface: ~400 syscalls,
   ioctls on every device node the user can open, `/proc` and `/sys`, netlink, filesystem
   parsers on mountable media, BPF.
2. **Remote unauthenticated → kernel.** Network stack parsers, and anything that processes
   attacker-controlled bytes before authentication. Much smaller surface, much higher
   severity.
3. **Container → host.** A superset of (1), plus user namespaces, which *expand* the
   unprivileged surface by letting an unprivileged user reach code that previously required
   root.
4. **Malicious peripheral.** Thunderbolt/USB4, a compromised NIC or BMC. Defeats everything
   in software because DMA bypasses the MMU — the IOMMU is the only answer.
5. **Malicious hypervisor** (confidential computing). Inverted trust: the guest must not
   trust the host's memory, its devices, or its interrupts.
6. **Physical attacker.** Cold boot, bus probing, DMA over an exposed port, JTAG. Needs
   memory encryption and fused-off debug.

A mitigation list without a threat model is cargo cult. When asked "should we enable X," the
correct first question is always "against whom?"

### T.3 — The exploitation pipeline, and where each mitigation intervenes

Understanding mitigations requires understanding the chain they break. A modern local
privilege escalation looks like:

```
 1. Find a bug            (UAF, OOB write, race, type confusion)
 2. Gain a primitive      (arbitrary read / arbitrary write / control flow)
 3. Defeat KASLR          (info leak -> learn kernel base)
 4. Groom the heap        (place a useful victim object next to the bug)
 5. Overwrite something   (cred struct, function pointer, page table)
 6. Hijack control flow   (ROP/JOP chain) OR data-only (just flip a flag)
 7. Escalate              (commit_creds(prepare_kernel_cred(0)))
 8. Persist / clean up    (hide, restore, avoid the panic)
```

| Stage | Mitigations |
|---|---|
| 1 | KASAN/syzkaller in development; `refcount_t`; Rust; `__counted_by` |
| 2 | Hardened usercopy, `CONFIG_FORTIFY_SOURCE`, bounds on `copy_to_user` |
| 3 | KASLR, FGKASLR, `kptr_restrict`, `dmesg_restrict`, removing `/proc/kallsyms` |
| 4 | Heap randomization (`CONFIG_SLAB_FREELIST_RANDOM`), cache splitting (`CONFIG_RANDOM_KMALLOC_CACHES`), separate caches for usercopy |
| 5 | `__ro_after_init`, constification, page table protections, `CONFIG_STATIC_USERMODEHELPER` |
| 6 | SMEP/SMAP/PAN/PXN, **CFI**, shadow stacks, `CONFIG_STACKPROTECTOR` |
| 7 | LSM hooks on credential change, SELinux, `CONFIG_SECURITY_LOCKDOWN` |
| 8 | IMA, audit, `dm-verity` |

The valuable insight: **mitigations are cheap where the attacker has few options and
expensive where they have many.** SMEP is nearly free and eliminates an entire technique
(ret2usr). CFI costs 1–5% and only narrows the ROP gadget set. That asymmetry is how you
prioritize under a performance budget.

### T.4 — Privilege: the three overlapping models

Linux has accumulated three access-control systems that all apply simultaneously, and a
request must pass **all** of them.

**1. DAC — the Unix model.** uid/gid, file mode bits, POSIX ACLs. Discretionary: the object
owner decides. Fundamental weakness: a compromised process inherits all of its user's
authority (the confused-deputy problem), and `root` bypasses everything.

**2. Capabilities — splitting root.** `CAP_NET_ADMIN`, `CAP_SYS_ADMIN`, ~40 others. The
theory is least privilege; the practice is that **`CAP_SYS_ADMIN` is a grab-bag that is
effectively equivalent to root** — it gates mount, many ioctls, `bpf()`, namespace
operations, and more. Granting it is not a meaningful reduction. New code should define a
*specific* capability or use a file-descriptor capability model instead. There are five
capability sets per thread (permitted, effective, inheritable, bounding, ambient) plus three
file sets, and the interaction rules are genuinely intricate — know that ambient caps
(4.3+) exist to fix the "inheritable caps don't actually inherit without a file capability"
trap.

**3. LSM — mandatory access control.** Hooks throughout the kernel at every security-relevant
decision, with a policy that the object owner *cannot* override. SELinux (type
enforcement, label-based), AppArmor (path-based, simpler), Smack, TOMOYO, plus the "minor"
LSMs that stack: Yama (ptrace restriction), Landlock (**unprivileged** sandboxing — a
process restricts itself), LoadPin, SafeSetID, BPF-LSM.

The ordering is: DAC first, then capabilities, then LSM. **LSM can only further restrict;
it can never grant.** That is a deliberate design invariant — it means adding an LSM can
never open a hole, which is what made merging a pluggable framework acceptable at all.

### T.5 — Reducing surface: seccomp

The highest-leverage single control available, because it works on the *size* of the attack
surface rather than the difficulty of each attack.

**seccomp-BPF**: attach a cBPF filter evaluated on every syscall entry. The filter sees
syscall number, architecture, instruction pointer, and the six register arguments —
**but it cannot dereference pointers**, deliberately, because dereferencing would create a
TOCTOU window between the check and the kernel's own read of the argument. That single
restriction is the most important design decision in seccomp and is a good interview
question.

Actions: `KILL_PROCESS`, `KILL_THREAD`, `TRAP` (SIGSYS), `ERRNO`, `TRACE` (to a ptracer),
`USER_NOTIF` (to a supervisor process over an fd — the modern, powerful one, enabling
userspace emulation of syscalls), `LOG`, `ALLOW`.

Two properties make it composable: filters **stack** (all must allow), and installing one
requires either `CAP_SYS_ADMIN` or `no_new_privs`, which guarantees a sandboxed process
cannot gain privilege via setuid and escape.

Chrome, systemd (`SystemCallFilter=`), Docker's default profile, Firefox, and Android all
rely on it. A typical server needs ~50 of ~400 syscalls; denying the other 350 removes
most of the historical CVE surface at close to zero cost.

### T.6 — Memory-safety mitigations

| Mitigation | Blocks | Cost |
|---|---|---|
| **SMEP** (x86) / **PXN** (ARM) | Kernel executing user pages (ret2usr) | free (hardware) |
| **SMAP** (x86) / **PAN** (ARM) | Kernel *reading/writing* user pages outside `copy_*_user` | ~free; forces `stac/clac` in accessors |
| **KASLR** | Hardcoded addresses | free; defeated by any info leak |
| **KPTI** | Meltdown; also strengthens KASLR | **5–30%** syscall cost — the expensive one |
| **`__ro_after_init`** | Overwriting post-boot-constant data (LSM hooks, syscall table) | free |
| **`CONFIG_STRICT_KERNEL_RWX`** | W^X violations in kernel text | free |
| **Stack protector** (`-fstack-protector-strong`) | Linear stack overflow | ~1% |
| **`VMAP_STACK`** | Stack overflow into neighbouring memory (guard page) | small |
| **`FORTIFY_SOURCE`** | `memcpy`/`strcpy` overflows with known sizes | compile-time mostly |
| **Hardened usercopy** | `copy_to_user` of a size exceeding the object's slab/page bounds | small |
| **`init_on_alloc` / `init_on_free`** | Uninitialized-memory info leaks; some UAF | **1–5%** |
| **`CONFIG_CFI_CLANG`** (KCFI) | Indirect-call hijack to a non-matching prototype | 1–5% |
| **Shadow stacks** (CET/BTI) | Return-address overwrite | small with hardware |
| **`SLAB_FREELIST_HARDENED`** | Freelist pointer corruption | ~free |
| **`RANDOM_KMALLOC_CACHES`** | Cross-cache heap grooming | memory |
| **`__counted_by`** | Flex-array overflows, with UBSAN bounds checking | ~free |

Two things to notice. First, **most are free and a few are expensive**; the expensive ones
(KPTI, `init_on_free`, CFI) are exactly the ones that should be a deliberate decision.
Second, **CFI only constrains the *forward* edge**; the backward edge needs shadow stacks.
Neither prevents **data-only attacks** — overwriting a `cred->uid` or a page-table entry
requires no control-flow hijack at all, which is why the strongest mitigations are the ones
that protect *data* (`__ro_after_init`, page-table protections) rather than control flow.

### T.7 — The speculative-execution tax

Spectre, Meltdown, MDS, L1TF, Retbleed, Downfall, Inception, GDS — a seven-year stream, and
the architectural lesson is more important than the individual bugs:

> **Microarchitectural state is architecturally invisible but observable through timing.
> Any optimization that makes execution depend on secret data creates a channel.**

| Class | Mechanism | Mitigation | Cost |
|---|---|---|---|
| **Meltdown** (v3) | Speculative load across a privilege check | **KPTI** — unmap kernel from user page tables | 5–30% syscall |
| **Spectre v1** | Bounds-check bypass | `array_index_nospec()`, manual, per-site | small, but requires finding every site |
| **Spectre v2** | Branch target injection | retpoline, then IBRS/eIBRS/IBPB | 0–30% depending on generation |
| **SSB** (v4) | Store-to-load forwarding | SSBD | ~few % |
| **L1TF** | L1 data leak across VMs | L1D flush on VM entry, core scheduling, PTE inversion | high for virt |
| **MDS/TAA** | Microarchitectural buffers | `VERW` on kernel exit | few % |
| **Retbleed / Inception** | Return-instruction prediction | more thunks, IBPB | few % |

Practical implications you should be able to state:
- **A syscall went from ~60 ns to ~200–500 ns.** This single number reshaped kernel API
  design — it is the reason `io_uring` and vDSO matter so much more now than in 2015.
- `mitigations=off` recovers most of it. It is a *legitimate* choice on a
  single-tenant machine with no untrusted local code, and a catastrophic one on a shared
  host. Knowing that this is a policy decision with a stated threat model — not a
  best-practice checkbox — is the senior answer.
- **SMT is the hardest case.** Many channels are cross-sibling. `nosmt` costs ~20–30% of
  throughput; **core scheduling** (`prctl(PR_SCHED_CORE)`) is the compromise, keeping only
  mutually-trusting tasks co-resident.

### T.8 — Boot and runtime integrity

```
   ROM/fuses ─verify─► FW ─verify─► bootloader ─verify─► kernel ─verify─► modules
                                                      │
                                                      ├─ dm-verity ──► rootfs (per-block, at read time)
                                                      ├─ IMA/EVM ───► measurement into TPM PCRs
                                                      └─ lockdown ──► close root→kernel paths
```

**Lockdown** (`CONFIG_SECURITY_LOCKDOWN_LSM`) deserves attention because it makes an
architectural statement: under Secure Boot, **root must not equal kernel**. Otherwise a
verified-boot chain is pointless — root simply writes to `/dev/mem` and installs whatever it
likes. Two levels:

- `integrity`: blocks unsigned modules, `/dev/mem`, `/dev/kmem`, kexec of unsigned images,
  hibernation, direct PCI/MSR access, some BPF, and ACPI table override.
- `confidentiality`: additionally blocks reading kernel memory — `/proc/kcore`, kprobes,
  most tracing, and debugfs paths that expose kernel addresses.

The cost is real: `confidentiality` breaks a lot of debugging, which is precisely the
tradeoff (see Ch. 06). This is a clean example of **security and observability being in
direct tension**, which is worth naming in an interview.

**IMA/EVM** measures files into TPM PCRs before use (appraisal can also *enforce*
signatures), producing an attestable log. It is the basis for remote attestation and for
confidential-computing launch measurement.

### T.9 — Isolating untrusted devices and untrusted hosts

**IOMMU** is the only defence against a DMA-capable peripheral, because DMA does not go
through the CPU MMU. Key decision: **strict vs lazy invalidation.**

| Mode | Behaviour | Risk |
|---|---|---|
| `iommu.strict=1` | Invalidate the IOTLB on every unmap, synchronously | none; costs significant throughput |
| lazy (default on many systems) | Batch invalidations | A freed buffer stays device-accessible for a window |
| `iommu.passthrough=1` | Identity map — no protection at all | full DMA access; only for trusted devices + performance |

For a hotpluggable external port (Thunderbolt), strict plus per-device domains plus
user-approval (`/sys/bus/thunderbolt/devices/*/authorized`) is the correct posture.

**Confidential computing** (SEV-SNP, TDX, ARM CCA) inverts trust: the guest kernel must
assume the hypervisor is hostile. Consequences that ripple through the whole kernel:
memory is encrypted with a key the host does not have; a "shared" bounce region must be
explicitly marked for any I/O (hence `swiotlb` becomes mandatory); MMIO and interrupts are
host-controlled and therefore attacker-controlled inputs, so **every driver that trusts its
device now needs hardening** (the `CONFIG_ARCH_HAS_CC_PLATFORM` / device-filter work); and
attestation must prove the launch measurement before any secret is provisioned. → Ch. 104.

### T.10 — Security as an engineering discipline

The mitigations above are the visible part. The durable part is process:

- **Fuzzing in CI.** syzkaller has found thousands of bugs; its value is not the individual
  bugs but that it converts "we hope this is fine" into a continuous measurement.
- **Sanitizers in CI** (KASAN, KCSAN, UBSAN, KMSAN) — the conversion table from
  `debugging-scenarios.md` §0 applied *before* shipping.
- **Deleting attack surface.** Removing a legacy filesystem parser or a dead driver is worth
  more than any mitigation. `CONFIG_*` minimization is a security activity.
- **Memory-safe languages.** Rust in the kernel (Part 5) eliminates classes 1–2 of the
  exploitation pipeline *by construction* for new code. This is the only intervention on
  the list that is strictly better rather than a tradeoff — which is the real argument for
  it, not performance.
- **Attack-surface-aware API design.** `openat2` rejects unknown flags; `io_uring` did not
  restrict its surface early and paid for it in CVEs. → Ch. 24.

---

## 1. Internals

### Source map

| Path | Contents |
|---|---|
| `security/` | LSM framework and all LSMs |
| `security/security.c` | Hook dispatch, LSM stacking, `lsm_static_call` infrastructure |
| `include/linux/lsm_hook_defs.h` | **The hook list** — every security decision point, ~250 hooks |
| `security/selinux/` | SELinux: AVC, policy engine, labelling |
| `security/apparmor/` | AppArmor: path-based profiles |
| `security/landlock/` | Landlock: unprivileged self-sandboxing |
| `security/lockdown/` | Lockdown LSM |
| `security/integrity/ima/` | IMA measurement and appraisal |
| `kernel/seccomp.c` | seccomp filter install and evaluation |
| `kernel/capability.c` | `capable()`, `ns_capable()`, capability sets |
| `kernel/cred.c` | `struct cred`, `prepare_creds`, `commit_creds` |
| `arch/x86/kernel/cpu/bugs.c` | **All speculative-execution mitigation selection** — read this file |
| `arch/x86/mm/pti.c` | KPTI page-table cloning |
| `include/linux/uaccess.h`, `lib/usercopy.c` | `copy_*_user`, hardened usercopy |
| `mm/usercopy.c` | `check_object_size()` — the hardened-usercopy bounds check |
| `Documentation/admin-guide/hw-vuln/` | Per-vulnerability admin documentation |
| `Documentation/security/` | LSM, credentials, self-protection |

### `struct cred` — the object every escalation targets

```c
struct cred {
	atomic_long_t	usage;
	kuid_t		uid, gid;        /* real */
	kuid_t		suid, sgid;      /* saved */
	kuid_t		euid, egid;      /* effective — what checks use */
	kuid_t		fsuid, fsgid;    /* filesystem */
	unsigned	securebits;
	kernel_cap_t	cap_inheritable, cap_permitted, cap_effective;
	kernel_cap_t	cap_bset, cap_ambient;
	void		*security;       /* LSM blob (SELinux label etc.) */
	struct user_namespace *user_ns;
	struct group_info *group_info;
	struct rcu_head	rcu;
};
```

Creds are **immutable once committed** and RCU-protected. You never modify a live `cred`;
you `prepare_creds()` (copy), modify the copy, and `commit_creds()` (publish). That
immutability is what makes lockless `capable()` checks safe from `current`.

The canonical exploit payload is `commit_creds(prepare_kernel_cred(NULL))` — which is why
`prepare_kernel_cred(NULL)` was changed in 6.2 to return `init_cred` explicitly and why
these symbols' addresses are a KASLR-leak target.

### The LSM hook mechanism

```c
/* include/linux/lsm_hook_defs.h — the authoritative list */
LSM_HOOK(int, 0, file_permission, struct file *file, int mask)
LSM_HOOK(int, 0, inode_create, struct inode *dir, struct dentry *dentry, umode_t mode)
LSM_HOOK(int, 0, task_kill, struct task_struct *p, struct kernel_siginfo *info,
	 int sig, const struct cred *cred)
LSM_HOOK(int, 0, bpf_prog_load, struct bpf_prog *prog, ...)
...
```

Call sites look like:

```c
/* fs/namei.c */
error = security_inode_permission(inode, MAY_EXEC);
if (error)
	return error;
```

Modern kernels dispatch these through **static calls** (`security/security.c`,
`lsm_static_call`), so an unused hook is a patched-out NOP rather than an indirect call
through an empty list. That matters: `file_permission` is on the hottest path in the VFS,
and an indirect call there is measurable. It is the same static-key principle as
tracepoints (Ch. 06).

**The invariant**: a non-zero return denies. With stacking, every enabled LSM is called and
**the first denial wins** — hence "LSMs can only restrict."

### seccomp evaluation path

```
 syscall entry
   └─ syscall_trace_enter()
        └─ __secure_computing()
             └─ for each filter in current->seccomp.filter (newest first):
                  bpf_prog_run_pin_on_cpu(filter->prog, &sd)
             └─ take the *most restrictive* action across all filters
```

`struct seccomp_data` is what the filter sees — and note the complete absence of any way to
follow a pointer:

```c
struct seccomp_data {
	int nr;                   /* syscall number */
	__u32 arch;               /* AUDIT_ARCH_* — MUST be checked first */
	__u64 instruction_pointer;
	__u64 args[6];            /* raw register values; pointers NOT dereferenceable */
};
```

**Always check `arch` first.** A filter that checks `nr == __NR_write` without checking
`arch` is bypassable by invoking the 32-bit ABI, where the same number means something
else. This is the classic seccomp bug.

### Observability surface

```bash
# Which mitigations are active, and what the kernel thinks of each:
grep . /sys/devices/system/cpu/vulnerabilities/*

# Which LSMs are enabled and in what order:
cat /sys/kernel/security/lsm

# Lockdown state:
cat /sys/kernel/security/lockdown

# Per-process seccomp mode and capability sets:
grep -E 'Seccomp|Cap' /proc/self/status
capsh --decode=$(grep CapEff /proc/self/status | cut -f2)

# Is KASLR active? (needs CAP_SYSLOG)
sudo grep startup_64 /proc/kallsyms

# SELinux denials:
ausearch -m avc -ts recent
# AppArmor:
dmesg | grep apparmor

# IMA measurement log:
cat /sys/kernel/security/ima/ascii_runtime_measurements

# Kernel hardening self-check:
git clone https://github.com/a13xp0p0v/kernel-hardening-checker
kernel-hardening-checker -c /boot/config-$(uname -r)
```

---

## 2. Practice

### Lab 102.1 — Audit your own kernel's security posture

```bash
#!/bin/bash
# posture.sh — what is this kernel actually protecting me from?
echo "=== Hardware vulnerabilities ==="
for f in /sys/devices/system/cpu/vulnerabilities/*; do
	printf '%-24s %s\n' "$(basename "$f")" "$(cat "$f")"
done

echo; echo "=== LSMs ==="
cat /sys/kernel/security/lsm 2>/dev/null || echo "securityfs not mounted"

echo; echo "=== Lockdown ==="
cat /sys/kernel/security/lockdown 2>/dev/null || echo "lockdown LSM not built"

echo; echo "=== Key config options ==="
CFG=/boot/config-$(uname -r)
[ -r "$CFG" ] || CFG=/proc/config.gz
for opt in CONFIG_STRICT_KERNEL_RWX CONFIG_RANDOMIZE_BASE CONFIG_PAGE_TABLE_ISOLATION \
           CONFIG_STACKPROTECTOR_STRONG CONFIG_FORTIFY_SOURCE CONFIG_HARDENED_USERCOPY \
           CONFIG_INIT_ON_ALLOC_DEFAULT_ON CONFIG_INIT_ON_FREE_DEFAULT_ON \
           CONFIG_CFI_CLANG CONFIG_SLAB_FREELIST_RANDOM CONFIG_SLAB_FREELIST_HARDENED \
           CONFIG_RANDOM_KMALLOC_CACHES CONFIG_MODULE_SIG_FORCE CONFIG_SECURITY_LOCKDOWN_LSM \
           CONFIG_BPF_UNPRIV_DEFAULT_OFF CONFIG_DEBUG_CREDENTIALS CONFIG_VMAP_STACK; do
	if zgrep -qx "$opt=y" "$CFG" 2>/dev/null; then s="ENABLED"; else s="--"; fi
	printf '%-42s %s\n' "$opt" "$s"
done

echo; echo "=== Unprivileged surface ==="
printf '%-42s %s\n' "unprivileged_userns_clone" \
	"$(cat /proc/sys/kernel/unprivileged_userns_clone 2>/dev/null || echo n/a)"
printf '%-42s %s\n' "unprivileged_bpf_disabled" \
	"$(cat /proc/sys/kernel/unprivileged_bpf_disabled 2>/dev/null || echo n/a)"
printf '%-42s %s\n' "kptr_restrict" "$(cat /proc/sys/kernel/kptr_restrict)"
printf '%-42s %s\n' "dmesg_restrict" "$(cat /proc/sys/kernel/dmesg_restrict)"
printf '%-42s %s\n' "perf_event_paranoid" "$(cat /proc/sys/kernel/perf_event_paranoid)"
printf '%-42s %s\n' "ptrace_scope" \
	"$(cat /proc/sys/kernel/yama/ptrace_scope 2>/dev/null || echo 'no yama')"
```

Run it on a distro kernel and on a kernel you built with `make defconfig`. The gap is
instructive: distros enable a great deal that `defconfig` does not.

### Lab 102.2 — Measure the mitigation tax

```c
/* syscall_cost.c — how expensive is a syscall on this machine?
 * Build: gcc -O2 -o syscall_cost syscall_cost.c
 * Compare: boot normally, then boot with mitigations=off.          */
#define _GNU_SOURCE
#include <stdio.h>
#include <time.h>
#include <unistd.h>
#include <sys/syscall.h>

#define N 2000000

static inline unsigned long long ns_now(void)
{
	struct timespec ts;
	clock_gettime(CLOCK_MONOTONIC, &ts);
	return (unsigned long long)ts.tv_sec * 1000000000ULL + ts.tv_nsec;
}

int main(void)
{
	unsigned long long t0, t1;
	volatile long r;

	/* getppid: about the cheapest real syscall — measures pure entry/exit. */
	t0 = ns_now();
	for (int i = 0; i < N; i++)
		r = syscall(SYS_getppid);
	t1 = ns_now();
	printf("getppid  : %6.1f ns/call\n", (double)(t1 - t0) / N);

	/* vDSO path for comparison: no ring transition at all. */
	struct timespec ts;
	t0 = ns_now();
	for (int i = 0; i < N; i++)
		clock_gettime(CLOCK_MONOTONIC, &ts);
	t1 = ns_now();
	printf("clock_gettime (vDSO): %6.1f ns/call\n", (double)(t1 - t0) / N);

	(void)r;
	return 0;
}
```

Expected shape: ~60–90 ns with `mitigations=off`, ~200–500 ns with full mitigations, and
~20–25 ns for the vDSO path regardless. **That ratio is the entire argument for vDSO,
`io_uring`, and batching** — you are measuring the number that reshaped kernel API design
after 2018. Cross-reference Ch. 24 §T.6 and Ch. 76.

### Lab 102.3 — Write a seccomp filter by hand

```c
/* sandbox.c — minimal seccomp-BPF sandbox.
 * Build: gcc -O2 -o sandbox sandbox.c
 * Run:   ./sandbox /bin/echo hello     (works)
 *        ./sandbox /bin/ls             (dies: needs getdents64)   */
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <linux/audit.h>
#include <linux/filter.h>
#include <linux/seccomp.h>
#include <sys/prctl.h>
#include <sys/syscall.h>

#define ALLOW(nr) \
	BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_##nr, 0, 1), \
	BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW)

int main(int argc, char **argv)
{
	struct sock_filter filter[] = {
		/* 1. Validate the architecture FIRST. Skipping this is the classic
		 *    seccomp bug: syscall numbers differ between ABIs, so a filter
		 *    written for x86_64 is trivially bypassed via the i386 entry.  */
		BPF_STMT(BPF_LD | BPF_W | BPF_ABS,
			 offsetof(struct seccomp_data, arch)),
		BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, AUDIT_ARCH_X86_64, 1, 0),
		BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL_PROCESS),

		/* 2. Load the syscall number. */
		BPF_STMT(BPF_LD | BPF_W | BPF_ABS,
			 offsetof(struct seccomp_data, nr)),

		/* 3. Allowlist. Deny-by-default is the only safe posture. */
		ALLOW(read), ALLOW(write), ALLOW(exit), ALLOW(exit_group),
		ALLOW(brk), ALLOW(mmap), ALLOW(munmap), ALLOW(mprotect),
		ALLOW(fstat), ALLOW(newfstatat), ALLOW(close),
		ALLOW(execve), ALLOW(openat), ALLOW(access), ALLOW(arch_prctl),
		ALLOW(set_tid_address), ALLOW(set_robust_list), ALLOW(rseq),
		ALLOW(prlimit64), ALLOW(getrandom), ALLOW(pread64),

		/* 4. Anything else: return ENOSYS rather than killing, so we can
		 *    observe what the program wanted. Switch to KILL_PROCESS for
		 *    production; ERRNO is a debugging aid.                        */
		BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ERRNO | (ENOSYS & SECCOMP_RET_DATA)),
	};
	struct sock_fprog prog = {
		.len = (unsigned short)(sizeof(filter) / sizeof(filter[0])),
		.filter = filter,
	};

	if (argc < 2) {
		fprintf(stderr, "usage: %s <cmd> [args...]\n", argv[0]);
		return 1;
	}

	/* no_new_privs is mandatory without CAP_SYS_ADMIN, and is the thing that
	 * guarantees a sandboxed process cannot escape via a setuid binary.     */
	if (prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0)) {
		perror("PR_SET_NO_NEW_PRIVS");
		return 1;
	}
	if (syscall(SYS_seccomp, SECCOMP_SET_MODE_FILTER, 0, &prog)) {
		perror("seccomp");
		return 1;
	}

	execvp(argv[1], &argv[1]);
	perror("execvp");
	return 1;
}
```

Then discover the real surface of a program:

```bash
# What does it actually need? Count distinct syscalls.
strace -f -c /bin/ls >/dev/null
# Or generate a filter automatically:
#   systemd-analyze syscall-filter
#   or use libseccomp's scmp_* API instead of raw cBPF for real work.
```

**The lesson to extract:** `/bin/echo` needs ~15 syscalls. A typical daemon needs ~50 of
~400. Denying the other 350 is a 7× reduction in syscall attack surface for approximately
zero runtime cost. No other single control has that ratio.

### Lab 102.4 — Build a deliberately vulnerable module and watch mitigations work

```c
/* vuln_demo.c — DELIBERATELY BUGGY. QEMU/VM ONLY. NEVER on a real system.
 *
 * Demonstrates that hardened usercopy converts a silent info leak into a
 * loud, localized kill.
 *
 * Build against a kernel with CONFIG_HARDENED_USERCOPY=y, then rebuild
 * with it off and compare.
 */
#include <linux/module.h>
#include <linux/debugfs.h>
#include <linux/slab.h>
#include <linux/uaccess.h>

static char *victim;
static struct dentry *dir;

static ssize_t leak_read(struct file *f, char __user *ubuf,
			 size_t len, loff_t *off)
{
	if (*off)
		return 0;

	/* BUG: victim is a 64-byte allocation, but we honour whatever length
	 * userspace asks for. Without hardened usercopy this silently copies
	 * adjacent slab memory to userspace -- a textbook info leak that can
	 * defeat KASLR. With CONFIG_HARDENED_USERCOPY=y, check_object_size()
	 * notices len exceeds the object's slab size and kills the task with
	 * a precise report naming the cache and the overrun.                */
	if (copy_to_user(ubuf, victim, len))
		return -EFAULT;

	*off = len;
	return len;
}

static const struct file_operations leak_fops = {
	.owner = THIS_MODULE,
	.read  = leak_read,
};

static int __init vuln_init(void)
{
	victim = kmalloc(64, GFP_KERNEL);
	if (!victim)
		return -ENOMEM;
	memset(victim, 'A', 64);

	dir = debugfs_create_dir("vuln_demo", NULL);
	debugfs_create_file("leak", 0400, dir, NULL, &leak_fops);

	pr_warn("vuln_demo: loaded. THIS MODULE IS DELIBERATELY INSECURE.\n");
	return 0;
}

static void __exit vuln_exit(void)
{
	debugfs_remove_recursive(dir);
	kfree(victim);
}

module_init(vuln_init);
module_exit(vuln_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Deliberately vulnerable module for mitigation demonstration");
```

```bash
# In the VM, as root:
insmod vuln_demo.ko
dd if=/sys/kernel/debug/vuln_demo/leak bs=64  count=1 2>/dev/null | xxd | head   # fine
dd if=/sys/kernel/debug/vuln_demo/leak bs=512 count=1 2>/dev/null | xxd | head   # boom
dmesg | tail -20
# Expect: "usercopy: Kernel memory exposure attempt detected from SLUB
#          object 'kmalloc-64' (offset 0, size 512)!"
```

Now rebuild with `CONFIG_HARDENED_USERCOPY=n` and repeat. The read *succeeds* and returns
448 bytes of whatever was next in the slab. **That is the entire point of the lab:** the bug
is identical; only the observability of its consequence changed. Then add `CONFIG_KASAN=y`
and observe that KASAN catches it too, but only if the read crosses a redzone — the two
mitigations catch overlapping but different sets.

### Lab 102.5 — Landlock: sandbox yourself, unprivileged

```c
/* landlock_demo.c — restrict this process to read-only access under /usr.
 * Build: gcc -O2 -o landlock_demo landlock_demo.c
 * Requires kernel >= 5.13 with CONFIG_SECURITY_LANDLOCK=y.
 *
 * Landlock is the first Linux MAC an *unprivileged* process may apply to
 * itself -- which makes defence-in-depth available to any application
 * without root, a systemd unit, or a container runtime.                  */
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <linux/landlock.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/prctl.h>
#include <sys/syscall.h>
#include <unistd.h>

static int landlock_create_ruleset(const struct landlock_ruleset_attr *a,
				   size_t size, __u32 flags)
{ return syscall(__NR_landlock_create_ruleset, a, size, flags); }

static int landlock_add_rule(int fd, enum landlock_rule_type t,
			     const void *attr, __u32 flags)
{ return syscall(__NR_landlock_add_rule, fd, t, attr, flags); }

static int landlock_restrict_self(int fd, __u32 flags)
{ return syscall(__NR_landlock_restrict_self, fd, flags); }

int main(int argc, char **argv)
{
	struct landlock_ruleset_attr rs = {
		/* Everything listed here is DENIED unless a rule re-grants it. */
		.handled_access_fs =
			LANDLOCK_ACCESS_FS_EXECUTE   | LANDLOCK_ACCESS_FS_WRITE_FILE |
			LANDLOCK_ACCESS_FS_READ_FILE | LANDLOCK_ACCESS_FS_READ_DIR   |
			LANDLOCK_ACCESS_FS_REMOVE_DIR| LANDLOCK_ACCESS_FS_REMOVE_FILE|
			LANDLOCK_ACCESS_FS_MAKE_REG  | LANDLOCK_ACCESS_FS_MAKE_DIR,
	};
	struct landlock_path_beneath_attr pb = {
		.allowed_access = LANDLOCK_ACCESS_FS_READ_FILE |
				  LANDLOCK_ACCESS_FS_READ_DIR  |
				  LANDLOCK_ACCESS_FS_EXECUTE,
	};
	int rfd;

	if (argc < 2) {
		fprintf(stderr, "usage: %s <cmd> [args...]\n", argv[0]);
		return 1;
	}

	rfd = landlock_create_ruleset(&rs, sizeof(rs), 0);
	if (rfd < 0) {
		perror("landlock_create_ruleset (kernel too old or disabled?)");
		return 1;
	}

	/* Re-grant read+exec beneath /usr only. */
	pb.parent_fd = open("/usr", O_PATH | O_CLOEXEC);
	if (pb.parent_fd < 0) { perror("open /usr"); return 1; }
	if (landlock_add_rule(rfd, LANDLOCK_RULE_PATH_BENEATH, &pb, 0)) {
		perror("landlock_add_rule"); return 1;
	}
	close(pb.parent_fd);

	if (prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0)) { perror("nnp"); return 1; }
	if (landlock_restrict_self(rfd, 0)) { perror("restrict_self"); return 1; }
	close(rfd);

	execvp(argv[1], &argv[1]);
	perror("execvp");
	return 1;
}
```

```bash
./landlock_demo /bin/cat /usr/share/doc/*/copyright   # allowed
./landlock_demo /bin/cat /etc/passwd                  # EACCES
./landlock_demo /bin/sh -c 'echo x > /tmp/x'          # EACCES
```

Note what this demonstrates architecturally: the restriction is **unprivileged, inherited
across `execve`, and irreversible**. That combination is what makes it usable as
defence-in-depth inside an application, which neither SELinux nor AppArmor can offer.

### Lab 102.6 — Observe LSM hook cost

```bash
# How many times is the hottest hook called, and what does it cost?
sudo bpftrace -e '
kprobe:security_file_permission { @calls = count(); }
kprobe:security_file_permission { @start[tid] = nsecs; }
kretprobe:security_file_permission /@start[tid]/ {
	@ns = hist(nsecs - @start[tid]); delete(@start[tid]);
}
interval:s:10 { exit(); }'

# Compare with SELinux enforcing vs permitted vs disabled:
#   getenforce; setenforce 0
# and with no LSM at all (boot with lsm="" -- test VM only).
```

The measurement to take away: with SELinux enforcing, `security_file_permission` is a few
hundred nanoseconds on a cache miss in the AVC and a few tens on a hit. The **AVC (access
vector cache)** is what makes SELinux viable at all — policy evaluation is expensive, so it
is cached per (source type, target type, class). Understanding that the AVC exists, and that
`avcstat` shows its hit rate, is the difference between "SELinux is slow" folklore and an
actual measurement.

---

## 3. Mastery drills

1. Build a kernel with `mitigations=off` and one with full mitigations. Measure syscall
   cost, `fork` cost, and a real workload (e.g. `redis-benchmark`). Produce a table. Then
   write the one-page memo you would give a VP deciding whether to disable mitigations on a
   single-tenant fleet — including the threat model under which the decision is correct.

2. Write a minimal LSM that logs every `execve` with the full path and the caller's
   credentials, and stacks with whatever LSM is already active. Measure the overhead on a
   kernel build. Then rewrite it as a BPF-LSM program and compare both effort and overhead.

3. Take a real daemon (nginx, redis, your own service). Derive its minimal syscall set with
   `strace -c`, write a libseccomp filter, and run it under load. Document every syscall you
   had to add after the first failure and why you did not predict it — the gap between the
   predicted and actual set is the interesting result.

4. Reproduce the hardened-usercopy lab, then extend `vuln_demo` with a use-after-free and a
   slab out-of-bounds write. For each of the three bugs, determine which of {KASAN, hardened
   usercopy, `init_on_free`, `SLAB_FREELIST_HARDENED`, `CONFIG_DEBUG_LIST`} catches it.
   Produce the coverage matrix. The point is that no single mitigation covers the space.

5. Enable lockdown in `confidentiality` mode and enumerate exactly which of your debugging
   tools stop working (kprobes, `/proc/kcore`, `perf`, BPF, kgdb, `/dev/mem`). Write the
   runbook for debugging a production issue on a locked-down machine.

6. Implement a `SECCOMP_RET_USER_NOTIF` supervisor: a parent process that intercepts
   `openat` from a child, applies its own policy, and uses
   `SECCOMP_IOCTL_NOTIF_ADDFD` to inject a file descriptor. Explain the TOCTOU hazard in
   reading the child's memory to inspect the path argument, and how
   `SECCOMP_USER_NOTIF_FLAG_CONTINUE` makes it worse.

7. Configure a system with IMA appraisal enforcing signatures, and dm-verity on the root
   filesystem. Then attempt to modify a binary and document exactly where in the boot/exec
   path the attempt fails and what the operator sees.

8. Measure IOMMU strict vs lazy vs passthrough on an NVMe device using `fio`. Quantify the
   throughput and latency cost of strict mode, then write the decision rule you would apply
   for (a) a laptop with Thunderbolt, (b) a single-tenant database server, (c) a
   multi-tenant host with SR-IOV passthrough.

9. Study three real kernel CVEs from different classes — one UAF (e.g. a `net/sched`
   qdisc), one verifier bug (BPF), one race (a `io_uring` or `tty` bug). For each: what was
   the bug, which mitigation would have blocked exploitation, which would not, and what
   structural change eliminated the class rather than the instance.

10. Take a driver from Part 2 and harden it for the confidential-computing threat model:
    the device is untrusted and may return arbitrary values from MMIO reads and DMA. List
    every place the driver currently trusts the device, and fix them. Estimate the effort
    for the whole subsystem.

11. Set up syzkaller against a subsystem of your choice with KASAN, KCSAN, and UBSAN
    enabled. Run it for 24 hours. Triage whatever it finds, and — more importantly —
    measure the *coverage* it achieved and identify which code it could not reach, and why.

12. Write a design document for a new device's UAPI under the assumption that the caller is
    hostile and unprivileged. Enumerate every input, its validation, the TOCTOU windows, the
    resource-exhaustion vectors, and the information the interface leaks (including timing).

13. Argue both sides of: "unprivileged user namespaces should be disabled by default."
    Produce the CVE evidence for the prosecution and the functionality evidence for the
    defence, then state your position and the conditions under which you would reverse it.

---

## 4. Further reading

**Kernel documentation**
- `Documentation/admin-guide/hw-vuln/` — one file per speculative-execution vulnerability,
  written by the people who implemented the mitigations. The most authoritative source.
- `Documentation/security/` — `self-protection.rst`, `credentials.rst`, `lsm.rst`,
  `landlock.rst`, `keys/`
- `Documentation/userspace-api/seccomp_filter.rst`
- `Documentation/admin-guide/kernel-parameters.txt` — every `mitigations=`, `iommu=`,
  `init_on_*`, `lsm=` option

**Papers**
- Kocher et al., "Spectre Attacks: Exploiting Speculative Execution," *IEEE S&P*, 2019
- Lipp et al., "Meltdown: Reading Kernel Memory from User Space," *USENIX Security*, 2018
- Saltzer & Schroeder, "The Protection of Information in Computer Systems," *Proc. IEEE*,
  1975 — the eight principles; still the framework
- Wright et al., "Linux Security Modules: General Security Support for the Linux Kernel,"
  *USENIX Security*, 2002 — why LSM is a hook framework and not a policy
- Chen et al., "Linux Kernel Vulnerabilities: State-of-the-Art Defenses and Open Problems,"
  *APSys*, 2011 — the categorization that still holds
- Kemerlis et al., "ret2dir: Rethinking Kernel Isolation," *USENIX Security*, 2014 — why
  SMEP alone is insufficient
- Abadi et al., "Control-Flow Integrity," *CCS*, 2005 — the original CFI formulation
- Klein et al., "seL4: Formal Verification of an OS Kernel," *SOSP*, 2009 — the other
  answer to the problem, and the reason it does not scale to Linux

**LWN**
- "The Kernel Self-Protection Project" series
- "Landlock: unprivileged access control" and the follow-ups
- "Lockdown, integrity, and confidentiality"
- The entire Spectre/Meltdown archive from January 2018 onward
- "A seccomp overview" and "Deep argument inspection for seccomp"
- "Kernel address space layout randomization" and the FGKASLR series

**Books**
- Bryant & O'Hallaron, *Computer Systems: A Programmer's Perspective* — ch. 3 for the
  machine-level foundation exploitation relies on
- Anderson, *Security Engineering*, 3rd ed. — the best book on threat modelling; ch. 1–6
- *The Shellcoder's Handbook* and Hovav Shacham's ROP paper for the offensive side (you
  cannot design defences without understanding the attack)

**Tools**
- `kernel-hardening-checker` (formerly `kconfig-hardened-check`) — audit a `.config`
- `syzkaller` — the fuzzer that finds most kernel CVEs
- `capsh`, `pscap`, `getpcaps` — capability inspection
- `audit2allow`, `avcstat`, `seinfo`, `sesearch` — SELinux
- `libseccomp` / `scmp_*` — do not hand-write cBPF in production
- `checksec` (for userspace binaries), `bpftool prog` (for BPF-LSM)

→ Next: [103-realtime.md](103-realtime.md)

# Chapter 24 — System Calls, UAPI Design, ioctl, sysfs, netlink

> **Goal:** design a userspace interface that can survive ten years of evolution without
> breaking a single existing binary — and know why every other design fails.

---

## Theory & First Principles

### T.0 — Start here: a two-line mistake that lasts forever

You are adding an ioctl. It takes a struct. You write the obvious thing:

```c
struct mydev_config {
	int  mode;
	int  timeout;
};

case MYDEV_SET_CONFIG:
	if (copy_from_user(&cfg, argp, sizeof(cfg)))
		return -EFAULT;
	return apply(&cfg);
```

It ships. **Three years later** you need to add a `priority` field, and you discover you
cannot:

| Attempt | Why it fails |
|---|---|
| Add a field to the struct | `sizeof` changes. Old binaries pass the old size; `copy_from_user` reads past their buffer. Info leak or `-EFAULT` |
| Add a new ioctl number | Works, but now there are two forever, and every future field doubles them again |
| Use a `flags` bit to mean "extended struct" | You did not reserve a `flags` field |
| Just require everyone to recompile | **You cannot.** Ch. 00 §T.5: a working binary must keep working |

**Two lines, written on day one, would have prevented all of it:**

```c
struct mydev_config {
	__u32 size;        /* ① set by userspace to sizeof(*cfg) */
	__u32 flags;       /* ② reserved; unknown bits REJECTED */
	__u32 mode;
	__u32 timeout;
	/* future fields append here, and old binaries still work */
};

err = copy_struct_from_user(&cfg, sizeof(cfg), argp, usize);
/*  -E2BIG if userspace sent a LARGER struct with a non-zero tail
 *  zero-extends if it sent a SMALLER (older) one
 *  => forward AND backward compatible, in one call                */
if (err)
	return err;
if (cfg.flags & ~MYDEV_VALID_FLAGS)
	return -EINVAL;    /* ← THE line that lets you add flags later */
```

`clone3`, `openat2`, `sched_setattr`, and `bpf()` all do exactly this. It is the single most
important UAPI idiom and it costs nothing.

**Now the harder problem, which no idiom fixes.** Consider what you have promised:

```c
/* You return -EINVAL for an unsupported mode. */
if (!supported(cfg.mode))
	return -EINVAL;
```

Somewhere, a program does `if (errno == EINVAL) fall_back_to_software()`. You later realize
`-ENOTSUPP` is more accurate and change it. **You have now broken that program.** The error
code was not documented as part of the interface; it was simply *observable*, and that was
enough.

> **Hyrum's Law.** *With a sufficient number of users, it does not matter what you promised
> in the contract: all observable behaviours of your system will be depended on by somebody.*

The observable surface is far larger than the documented one:

| You think the interface is | It also includes |
|---|---|
| The arguments and return value | Every **errno**, and their relative priority when several apply |
| | The **order** in which things happen, and what is visible in between |
| | **Timing** — a program that got away with a race now loses it |
| | The contents of **padding bytes** you never initialized |
| | **Bugs.** Somebody's workaround depends on the bug |

Real example from the tree: `readdir` returning entries in a particular order was never
promised, is not promised now, and is depended upon — so filesystems are constrained by it
anyway.

**Which gives the working rule for everything in this chapter:**

> **Add, never change.** Constrain what is observable *deliberately*, because whatever you
> expose you must keep. Reject unknown flags so that flags remain usable. Take a size field
> so that structs remain extensible. Return an fd so that lifetime is someone else's problem.

And the corollary that makes this an architecture chapter rather than a style one: **the cost
of a UAPI mistake is not the bug, it is the permanence.** A bad internal API is refactored in
an afternoon (Ch. 00 §T.5). A bad syscall is carried for twenty years by people who never met
you.

---

### T.1 An interface is a promise you cannot retract

Ch. 00 T.5 stated the asymmetry: userspace ABI is forever, in-kernel API is nothing. This
chapter is about what "forever" actually costs you at design time.

The operative law is **Hyrum's Law**:

> *With a sufficient number of users of an API, it does not matter what you promise in the
> contract: all observable behaviours of your system will be depended on by somebody.*

For the Linux kernel — with an effectively unbounded user population and binaries that are
never recompiled — this is not a caution, it is a certainty. Things that have become
de-facto ABI despite never being documented:

- The *exact* text and ordering of lines in `/proc` files (parsers break).
- The *number* of bytes `read()` returns for a given input.
- Which errno a syscall returns for a given invalid argument.
- Whether a field is zero-filled or left as garbage.
- The timing of when a `/sys` attribute becomes visible relative to a uevent.
- Undefined flag bits being ignored rather than rejected.

**The practical consequence:** every observable behaviour is a commitment. Therefore the
design discipline is **minimize what is observable** — expose the smallest surface that does
the job, reject everything undefined, and zero everything reserved. You cannot take back
what you expose, but you *can* later define something you previously rejected.

> **The asymmetry that drives all UAPI design:** *strictness can be relaxed later;
> permissiveness can never be tightened.*

### T.2 API vs ABI, and what "compatible" means

| | Changes if… | Breaks… |
|---|---|---|
| **API** (source-level) | function signatures, header constants, semantics | recompilation |
| **ABI** (binary-level) | struct layout, padding, syscall numbers, errno values, calling convention | **existing binaries** |

Linux guarantees **ABI** compatibility. Three directions matter, and people routinely conflate
the first two:

- **Backward compatibility** — a *new kernel* runs an *old binary*. **This is the guarantee.**
  Non-negotiable.
- **Forward compatibility** — an *old kernel* runs a *new binary*. Not guaranteed, but good
  design makes graceful degradation possible (feature probing rather than assuming).
- **Cross-architecture / compat** — a 32-bit binary on a 64-bit kernel. Requires explicit
  translation and is a persistent source of bugs (T.6).

### T.3 The extensibility problem, and the four known solutions

You are adding a syscall today. In three years you will need one more field. How do you add
it without breaking every existing caller *and* without the new caller silently succeeding on
an old kernel that ignores the new field?

Four mechanisms exist. Know all four, their failure modes, and which one Linux settled on.

**(1) Version field.**
```c
struct foo { __u32 version; ... };
```
*Fails* because it serializes evolution (you cannot add two independent features
concurrently), and every kernel must carry code for every historical version. Also, what does
a kernel do with `version = 7` when it knows 1–5? It cannot tell whether v7 is a superset.

**(2) Separate syscall per revision** (`stat`, `stat64`, `fstatat`, `statx`;
`clone`, `clone3`; `epoll_wait`, `epoll_pwait`, `epoll_pwait2`).
*Works*, but burns syscall numbers, duplicates code, and multiplies the compat matrix. Linux
has done this a lot and regrets most of it.

**(3) Flags word.**
```c
long sys_foo(..., unsigned int flags);
if (flags & ~FOO_VALID_MASK)
	return -EINVAL;        /* ★ reject unknown flags */
```
*Essential and cheap.* The rejection is what makes it work: an old kernel returns `-EINVAL`
for a flag it does not know, so **the caller learns the feature is absent** instead of
silently getting different behaviour. A flags word that ignores unknown bits is worse than
useless — it guarantees a future bug.

**(4) Extensible struct with a size field — the modern Linux answer.**

```c
struct clone_args {
	__aligned_u64 flags;
	__aligned_u64 pidfd;
	/* ... */
	__aligned_u64 cgroup;      /* added later */
};

/* userspace passes both the pointer AND its size */
long sys_clone3(struct clone_args __user *uargs, size_t size);
```

The kernel uses `copy_struct_from_user()`, which implements a precise three-way protocol:

```c
int copy_struct_from_user(void *dst, size_t ksize,
			  const void __user *src, size_t usize)
{
	if (usize < ksize) {
		/* OLD userspace, NEW kernel: copy what they gave, ZERO the rest.
		 * New fields must have "0 == legacy behaviour" semantics. */
		memset(dst + usize, 0, ksize - usize);
		return copy_from_user(dst, src, usize);
	}
	if (usize > ksize) {
		/* NEW userspace, OLD kernel: the tail MUST be all zeros,
		 * otherwise the caller is requesting something we cannot do. */
		if (!is_zeroed_user(src + ksize, usize - ksize))
			return -E2BIG;          /* ★ honest failure, not silent misbehaviour */
		usize = ksize;
	}
	return copy_from_user(dst, src, usize);
}
```

This is elegant, and the elegance is in the `-E2BIG` case: a new program that sets a new
field on an old kernel gets a **loud, specific error** rather than silently running without
the feature it asked for. Two design obligations follow and are enforced in review:

1. **Every new field must mean "legacy behaviour" when zero.** The whole scheme rests on it.
2. **`__u64` / `__aligned_u64` for everything**, including pointers, so the layout is
   identical on 32- and 64-bit (T.6).

Modern users: `clone3`, `openat2`, `sched_setattr`, `bpf`, `io_uring_setup`,
`landlock_create_ruleset`, `mount_setattr`, `statx` (via a mask rather than a size).
**Use this pattern for any new interface.**

### T.4 The syscall mechanism and its constraints

```
user: mov $NR, %eax; syscall
  → entry_SYSCALL_64 (arch/x86/entry/entry_64.S)
      swapgs; switch to the kernel stack; save pt_regs
  → do_syscall_64()
      nr = syscall_enter_from_user_mode()   /* audit, seccomp, ptrace, trace */
      if (nr < NR_syscalls) sys_call_table[nr](regs)
      syscall_exit_to_user_mode()           /* signals, resched, work pending */
  → sysret / iret
```

Constraints that shape every syscall API:

| Constraint | Why | Consequence |
|---|---|---|
| **≤ 6 arguments** | registers available in the syscall ABI | pack extras into a struct |
| **No structs by value** | no stack marshalling in the ABI | always pass a pointer + size |
| **`long`-sized returns** | one register | negative = errno; hence the ≤ 4095 `MAX_ERRNO` convention (Ch. 05 T.6) |
| **User pointers are hostile** | user memory can change at any moment | `copy_from_user` **once**, then use the kernel copy |
| **Must not sleep holding user pages** | faulting is allowed but the mapping can change | no assumptions across a copy |

`SYSCALL_DEFINEn()` does more than declare a function — it generates the compat wrappers,
the tracing hooks, and the argument-type metadata:

```c
SYSCALL_DEFINE3(my_op, int, fd, struct my_arg __user *, arg, unsigned int, flags)
{
	struct my_arg karg;

	if (flags & ~MY_VALID_FLAGS)
		return -EINVAL;
	if (copy_from_user(&karg, arg, sizeof(karg)))
		return -EFAULT;
	...
}
```

**The TOCTOU rule is absolute:** never read the same user memory twice and assume it is
unchanged. Another thread in the same process can rewrite it between your two reads. Copy
once into kernel memory, validate the kernel copy, use the kernel copy. A large family of
CVEs is exactly this mistake (`copy_from_user` a length, validate it, then `copy_from_user`
again using a re-read length).

### T.5 Choosing an interface family

| Family | Shape | Good for | Bad at |
|---|---|---|---|
| **syscall** | typed, 6 args, fast (~100 ns) | fundamental, frequent operations | evolving; costs a syscall number |
| **ioctl** | `(fd, cmd, arg)` — untyped | device/driver-specific ops | typing, compat, discoverability |
| **`/proc`** | text files | legacy process info, debugging | anything structured or hot |
| **`/sys`** | one value per file, text | device attributes, config | atomicity across values, bulk |
| **netlink** | structured, async, multicast | networking, subsystem config, events | simplicity; lots of boilerplate |
| **debugfs** | anything | **debugging only — NO ABI guarantee** | production interfaces |
| **eBPF** | verified programs | observability, policy, dataplane | fixed-function tasks |
| **io_uring** | shared ring, batched | high-rate async I/O | one-off operations |
| **`configfs`** | create objects via mkdir | declarative object construction | simple scalars |

Decision criteria, in order:

1. **Is it a filesystem-like object with a lifetime?** → an fd, with `read`/`write`/`ioctl`.
   Filedescriptors give you lifetime, permissions, polling, passing over `SCM_RIGHTS`, and
   `close()` cleanup for free. **"Return an fd" is almost always the right answer** — it is
   why `pidfd`, `memfd`, `userfaultfd`, `landlock`, `io_uring`, `perf_event_open`, `bpf`, and
   `timerfd`/`signalfd`/`eventfd` all do it.
2. **Is it one scalar attribute of one device?** → sysfs.
3. **Is it a stream of structured events, or config with many optional fields?** → netlink.
4. **Is it a frequent, fundamental operation?** → a syscall.
5. **Is it device-specific and irregular?** → ioctl on your device fd.
6. **Is it only for developers?** → debugfs, and say so.

### T.6 The compat problem

A 32-bit binary on a 64-bit kernel has different struct layouts. Sources of divergence:

```c
struct bad {
	int      a;      /* 4 bytes both */
	void    *p;      /* 4 on 32-bit, 8 on 64-bit  ← layout diverges */
	long     b;      /* 4 vs 8                     ← and again      */
	time_t   t;      /* 4 vs 8 (pre-y2038)         ← and again      */
	/* implicit padding differs too */
};
```
So the kernel needs a `compat_` translation path, and every such path is a chance to get it
wrong — historically a rich source of privilege-escalation bugs
(`compat_ioctl` in particular).

**The rules that eliminate the problem** rather than managing it:

1. Use **fixed-width types only**: `__u8/__u16/__u32/__u64`, `__s32`, etc. Never `long`,
   `int` (for anything that isn't genuinely an `int`), `size_t`, `time_t`, or `void *`.
2. Use `__aligned_u64` for 64-bit fields, because 32-bit x86 aligns `u64` to 4 bytes while
   x86-64 aligns to 8 — this *is* the classic compat layout bug.
3. Pass pointers as `__u64` and cast with `u64_to_user_ptr()`.
4. **Pad explicitly** and require the padding to be zero, so you can use it later.
5. Never embed a `struct timespec`; use `__s64` nanoseconds or `struct __kernel_timespec`.

```c
struct good_arg {
	__u32          fd;
	__u32          flags;
	__aligned_u64  buffer;      /* a user pointer, as u64 */
	__aligned_u64  length;
	__s64          timeout_ns;
	__u32          __reserved[4];   /* must be zero */
};
static_assert(sizeof(struct good_arg) == 48);   /* ★ pin the layout */
```
With this discipline, **the compat path is the native path** and `compat_ioctl` can be
`compat_ptr_ioctl`.

### T.7 ioctl: why it is bad and why it persists

```c
#define MY_IOC_MAGIC  'k'
#define MY_GET   _IOR(MY_IOC_MAGIC,  1, struct my_info)
#define MY_SET   _IOW(MY_IOC_MAGIC,  2, struct my_cfg)
#define MY_XCHG  _IOWR(MY_IOC_MAGIC, 3, struct my_xfer)
```
The command number encodes direction (2 bits), size (14 bits), magic (8 bits), and number
(8 bits). That encoding is the *only* type information in the interface.

**The sins:**
- **Untyped from userspace's view.** The compiler cannot check that `arg` matches `cmd`.
- **Namespace collisions** — `Documentation/userspace-api/ioctl/ioctl-number.rst` is a
  manually-maintained registry of magic numbers, which tells you everything.
- **Compat hell** — every ioctl with a struct needs a compat path (T.6).
- **Undiscoverable** — no way to enumerate what a device supports; no introspection.
- **Historically the #1 source of driver CVEs**, because each one is a bespoke parser of
  attacker-controlled data written by someone who writes one ioctl a year.

**Why it persists:** it is the only mechanism that is *local to a device* and needs no
central registration. Adding a syscall requires touching `syscall_64.tbl` on every
architecture and convincing the whole community; adding an ioctl requires nothing. That
asymmetry in *social* cost — not technical merit — is why there are tens of thousands of
ioctls.

**If you must write one, the checklist:**

```c
static long my_ioctl(struct file *f, unsigned int cmd, unsigned long arg)
{
	void __user *uarg = (void __user *)arg;
	struct my_cfg cfg;

	switch (cmd) {
	case MY_SET:
		if (copy_from_user(&cfg, uarg, sizeof(cfg)))
			return -EFAULT;
		if (cfg.flags & ~MY_VALID_FLAGS)    /* reject unknown flags */
			return -EINVAL;
		if (memchr_inv(cfg.__reserved, 0, sizeof(cfg.__reserved)))
			return -EINVAL;             /* reserved must be zero */
		if (cfg.len > MY_MAX_LEN)           /* bound EVERYTHING */
			return -EINVAL;
		return my_do_set(f->private_data, &cfg);
	default:
		return -ENOTTY;                     /* ★ ENOTTY, not EINVAL */
	}
}

static const struct file_operations fops = {
	.owner          = THIS_MODULE,
	.unlocked_ioctl = my_ioctl,
	.compat_ioctl   = compat_ptr_ioctl,   /* ★ works iff you followed T.6 */
};
```
`-ENOTTY` for an unknown command is a real convention: userspace probes with an ioctl and
treats `ENOTTY` as "not supported". Returning `-EINVAL` breaks that probe.

### T.8 sysfs: one value per file, and why

`Documentation/filesystems/sysfs.rst` states the rule:

> **"Attributes should be ASCII text files, preferably with only one value per file. It is
> noted that it may not be efficient to contain only one value per file, so it is socially
> acceptable to express an array of values of the same type."**

The rationale is compositional: one value per file means shell tools work (`cat`, `echo`,
globbing), permissions are per-attribute, uevents map to objects, and there is no parsing.

The cost is that **you cannot update two attributes atomically**. If your configuration needs
atomicity across fields, sysfs is the wrong interface — use an ioctl, netlink, or configfs.
This is the single most common sysfs design mistake.

Other hard rules:
- **One page (`PAGE_SIZE`) maximum** per `show()`.
- Use `sysfs_emit()` / `sysfs_emit_at()`, never `sprintf()` — they enforce the bound.
- Every attribute must be documented in `Documentation/ABI/` with a stability class
  (`stable/`, `testing/`, `obsolete/`, `removed/`).
- `show()` and `store()` may sleep; they run in task context with no locks held by the core.
- Attribute lifetime is tied to the kobject; use `ATTRIBUTE_GROUPS()` and the `dev_groups`
  field so the core creates and removes them at the right time (Ch. 26) — **never**
  `device_create_file()` in `probe()`, which races with userspace seeing the uevent.

### T.9 netlink: structured, extensible, asynchronous

Netlink is a socket-based protocol (`AF_NETLINK`) with a TLV (type-length-value) attribute
encoding. It is what networking uses for everything, and increasingly what other subsystems
use for configuration.

```
struct nlmsghdr { len, type, flags, seq, pid }
  └─ family header (e.g. struct ifinfomsg)
       └─ attributes: [ struct nlattr {len, type} | payload ] ...   (nested, aligned to 4)
```

What it gets right, and why it is the best *structured* interface available:

- **TLV = extensibility for free.** New attributes are just new types; old parsers skip
  unknown ones (or reject them, per the policy). This is T.3's problem solved generally.
- **Policy-driven validation.** `struct nla_policy` declares the expected type and bounds of
  each attribute, and `nla_parse_nested()` validates against it — a *declarative* parser
  instead of hand-written bounds checks. That single mechanism removed an entire class of
  driver bugs and is the strongest argument for netlink over ioctl.
- **Asynchronous and multicast** — the kernel can push events to subscribed listeners
  (`RTMGRP_LINK`, uevents over `NETLINK_KOBJECT_UEVENT`).
- **Dump semantics** — `NLM_F_DUMP` with a cursor handles arbitrarily large result sets
  without a fixed buffer.
- **Generic netlink** (`genetlink`) lets any subsystem register a family without a new
  protocol number, and **YAML specs** (`Documentation/netlink/specs/`) now generate both
  kernel parsers and userspace bindings — an actual IDL for kernel interfaces.

```c
static const struct nla_policy my_policy[MY_ATTR_MAX + 1] = {
	[MY_ATTR_NAME]  = { .type = NLA_NUL_STRING, .len = 32 },
	[MY_ATTR_VALUE] = { .type = NLA_U32 },
	[MY_ATTR_RANGE] = NLA_POLICY_RANGE(NLA_U32, 1, 100),
	[MY_ATTR_NEST]  = NLA_POLICY_NESTED(my_nested_policy),
};
```

The cost is real: netlink is verbose, the parsing boilerplate is substantial, and
misdesigned netlink APIs are as bad as bad ioctls. But for anything with many optional
fields, events, or bulk queries, it is the right tool.

### T.10 Security: every interface is attack surface

Each new syscall/ioctl is code reachable by an unprivileged attacker. The discipline:

1. **Validate everything, reject the undefined.** Unknown flags → `-EINVAL`. Non-zero
   reserved → `-EINVAL`. Out-of-range → `-EINVAL`. Never "clamp silently".
2. **Copy once** (T.4's TOCTOU rule).
3. **Integer safety** — `check_add_overflow`, `array_size`, `struct_size` (Ch. 08 T.6).
   `size` arguments are attacker-controlled.
4. **Permission check at the right time and against the right credentials.** Use
   `file->f_cred` for operations on a previously-opened fd, not `current_cred()`, or you have
   a confused-deputy bug.
5. **`capable()` vs `ns_capable()`** — a user-namespace-aware check is usually what you want;
   `capable(CAP_SYS_ADMIN)` is both over-broad and namespace-blind.
6. **LSM hook** — add one if the operation is security-relevant, so SELinux/AppArmor/Landlock
   can mediate it.
7. **Think about seccomp.** Your syscall will be filtered by argument; putting a pointer in
   argument 0 means seccomp cannot inspect what matters (seccomp cannot dereference). This is
   a genuine design constraint: **interfaces that hide everything behind a pointer are
   unfilterable**, which is a known weakness of `ioctl` and `io_uring`.
8. **Assume it will be fuzzed within days of merging** (Ch. 06 T.6) — write the syzkaller
   description yourself.

### T.11 The obligations that come with a new interface

Adding UAPI is not just code. `Documentation/process/adding-syscalls.rst` is the checklist,
and reviewers apply it:

- Header in `include/uapi/`, with fixed-width types and `static_assert` on sizes.
- Wired up in **every** architecture's syscall table (or use the generic table).
- `compat` handling, or a design that needs none (T.6).
- **`tools/testing/selftests/`** test, exercising success *and* every error path.
- **A man page**, posted to `linux-man@vger.kernel.org`.
- `Documentation/ABI/` entry for sysfs attributes.
- An `strace` decoder is appreciated.
- A real user — "someone will want this" is not sufficient; maintainers require a concrete
  in-tree or announced consumer.

That last point is policy with teeth: speculative interfaces become permanent maintenance
burdens with no one to validate them.

---

## 1. Internals

### 1.1 Source map

```
include/linux/syscalls.h             SYSCALL_DEFINEn, the prototypes
arch/x86/entry/syscalls/syscall_64.tbl   ★ the syscall number table
include/uapi/asm-generic/unistd.h    the generic table (arm64, riscv)
kernel/sys.c, kernel/sys_ni.c        misc syscalls, the ni_syscall stubs
kernel/entry/common.c                syscall_enter/exit_to_user_mode
include/linux/uaccess.h              copy_from_user, copy_struct_from_user
lib/strncpy_from_user.c, lib/usercopy.c
fs/ioctl.c                           the generic ioctl dispatch, FS_IOC_*
fs/compat_ioctl.c, fs/ioctl.c        compat_ptr_ioctl
fs/sysfs/, include/linux/sysfs.h     ★ sysfs_emit, ATTRIBUTE_GROUPS
fs/proc/, include/linux/seq_file.h   ★ seq_file — the correct way to write /proc files
net/netlink/af_netlink.c, genetlink.c
include/net/netlink.h                ★ nla_* parsing and the policy framework
Documentation/userspace-api/         ★★ the whole directory
Documentation/process/adding-syscalls.rst  ★★ the checklist
Documentation/ABI/                   ★ the sysfs contract
Documentation/netlink/specs/         the YAML IDL
Documentation/filesystems/sysfs.rst
```

### 1.2 `seq_file`: the only correct way to write a `/proc` file

Naively writing a `/proc` file with `sprintf` into a fixed buffer breaks the moment output
exceeds one page or the reader uses a small buffer. `seq_file` handles restart, partial
reads, and iteration:

```c
static void *my_start(struct seq_file *m, loff_t *pos)
{
	mutex_lock(&my_lock);                      /* held across the whole iteration */
	return seq_list_start(&my_list, *pos);
}
static void *my_next(struct seq_file *m, void *v, loff_t *pos)
{ return seq_list_next(v, &my_list, pos); }
static void my_stop(struct seq_file *m, void *v) { mutex_unlock(&my_lock); }
static int my_show(struct seq_file *m, void *v)
{
	struct my_obj *o = list_entry(v, struct my_obj, node);

	seq_printf(m, "%-16s %llu\n", o->name, o->count);
	return 0;
}
static const struct seq_operations my_sops = {
	.start = my_start, .next = my_next, .stop = my_stop, .show = my_show,
};
DEFINE_SEQ_ATTRIBUTE(my);          /* generates open/read/lseek/release */

proc_create_seq("mydriver", 0444, NULL, &my_sops);
/* single-value case: */
DEFINE_SHOW_ATTRIBUTE(my_single);
proc_create_single("mystat", 0444, NULL, my_single_show);
```

**The `start`/`stop` pair brackets the whole read**, so a lock taken in `start` is held until
`stop` — which means you must not take a lock there that a `show()` could need to drop. And
because `show()` can be called again after a buffer overflow, **it must be idempotent** — no
side effects.

---

## 2. Practice

### Lab 24.1 — Add a real syscall and wire it up

```bash
cd linux
```

`include/uapi/linux/mysys.h`:
```c
/* SPDX-License-Identifier: GPL-2.0 WITH Linux-syscall-note */
#ifndef _UAPI_LINUX_MYSYS_H
#define _UAPI_LINUX_MYSYS_H
#include <linux/types.h>

#define MYSYS_FLAG_UPPER   (1U << 0)
#define MYSYS_FLAG_REVERSE (1U << 1)
#define MYSYS_VALID_FLAGS  (MYSYS_FLAG_UPPER | MYSYS_FLAG_REVERSE)

struct mysys_args {
	__u32          flags;
	__u32          __pad;          /* must be zero */
	__aligned_u64  in;             /* user pointer, as u64 */
	__aligned_u64  out;
	__aligned_u64  len;
	__aligned_u64  __reserved[2];  /* must be zero; room to grow */
};
#endif
```

`kernel/mysys.c`:
```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/syscalls.h>
#include <linux/uaccess.h>
#include <linux/slab.h>
#include <uapi/linux/mysys.h>

SYSCALL_DEFINE2(mysys, struct mysys_args __user *, uargs, size_t, usize)
{
	struct mysys_args a;
	char *buf;
	long ret;
	size_t i;

	/* ★ T.3: the extensible-struct protocol, in one call */
	ret = copy_struct_from_user(&a, sizeof(a), uargs, usize);
	if (ret)
		return ret;                        /* -E2BIG if they set unknown fields */

	if (a.flags & ~MYSYS_VALID_FLAGS)          /* ★ reject unknown flags */
		return -EINVAL;
	if (a.__pad || a.__reserved[0] || a.__reserved[1])
		return -EINVAL;                    /* ★ reserved must be zero */
	if (!a.len || a.len > PAGE_SIZE)           /* ★ bound everything */
		return -EINVAL;

	buf = kmalloc(a.len, GFP_KERNEL);
	if (!buf)
		return -ENOMEM;

	/* ★ T.4: copy ONCE, then operate on the kernel copy */
	if (copy_from_user(buf, u64_to_user_ptr(a.in), a.len)) {
		ret = -EFAULT;
		goto out;
	}

	if (a.flags & MYSYS_FLAG_UPPER)
		for (i = 0; i < a.len; i++)
			buf[i] = (buf[i] >= 'a' && buf[i] <= 'z') ? buf[i] - 32 : buf[i];
	if (a.flags & MYSYS_FLAG_REVERSE)
		for (i = 0; i < a.len / 2; i++) {
			char t = buf[i];
			buf[i] = buf[a.len - 1 - i];
			buf[a.len - 1 - i] = t;
		}

	ret = copy_to_user(u64_to_user_ptr(a.out), buf, a.len) ? -EFAULT : (long)a.len;
out:
	kfree_sensitive(buf);
	return ret;
}
```

Wire it up:
```bash
echo 'obj-y += mysys.o' >> kernel/Makefile
# x86-64:
echo '463  common  mysys      sys_mysys' >> arch/x86/entry/syscalls/syscall_64.tbl
# generic (arm64/riscv): add to include/uapi/asm-generic/unistd.h
#   #define __NR_mysys 463
#   __SYSCALL(__NR_mysys, sys_mysys)
#   and bump __NR_syscalls
make -j$(nproc) && boot-in-qemu
```

Test both the new-on-new and new-on-old paths:
```c
/* test.c */
#include <sys/syscall.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include "mysys.h"
#define __NR_mysys 463
int main(void) {
	char in[] = "hello world", out[64] = {0};
	struct mysys_args a = { .flags = MYSYS_FLAG_UPPER | MYSYS_FLAG_REVERSE,
	                        .in = (unsigned long)in, .out = (unsigned long)out,
	                        .len = strlen(in) };
	long r = syscall(__NR_mysys, &a, sizeof(a));
	printf("ret=%ld out='%s'\n", r, out);

	/* an OLD caller: smaller struct -> kernel zero-fills */
	r = syscall(__NR_mysys, &a, 32);
	printf("small struct ret=%ld\n", r);

	/* a NEW caller with a non-zero unknown tail -> -E2BIG */
	struct { struct mysys_args base; unsigned long future; } big = { a, 1 };
	r = syscall(__NR_mysys, &big, sizeof(big));
	printf("big struct ret=%ld errno=%d (expect -1/E2BIG=7)\n", r, errno);

	/* unknown flag -> -EINVAL */
	a.flags = 0x80000000;
	r = syscall(__NR_mysys, &a, sizeof(a));
	printf("bad flag ret=%ld errno=%d (expect EINVAL=22)\n", r, errno);
	return 0;
}
```
**All four results are the point of the lab.** They are T.3 working.

### Lab 24.2 — The same functionality four ways; compare

Implement "get and set a device's mode" as (a) an ioctl, (b) a sysfs attribute,
(c) a netlink operation, (d) a `seq_file` in debugfs. Then evaluate each against:
extensibility, atomicity across multiple values, discoverability, compat cost,
shell usability, event delivery, and lines of code.

```c
/* (b) sysfs — the modern, correct form */
static ssize_t mode_show(struct device *dev, struct device_attribute *attr, char *buf)
{
	struct mydev *d = dev_get_drvdata(dev);

	return sysfs_emit(buf, "%u\n", READ_ONCE(d->mode));   /* ★ sysfs_emit, not sprintf */
}
static ssize_t mode_store(struct device *dev, struct device_attribute *attr,
			  const char *buf, size_t count)
{
	struct mydev *d = dev_get_drvdata(dev);
	unsigned int v;
	int ret = kstrtouint(buf, 0, &v);

	if (ret)
		return ret;
	if (v > MODE_MAX)
		return -EINVAL;
	guard(mutex)(&d->lock);
	d->mode = v;
	return count;
}
static DEVICE_ATTR_RW(mode);

static struct attribute *mydev_attrs[] = { &dev_attr_mode.attr, NULL };
ATTRIBUTE_GROUPS(mydev);
/* then:  .dev_groups = mydev_groups  in the driver struct — ★ never device_create_file() */
```
```bash
# And the ABI documentation you are obliged to write:
cat > Documentation/ABI/testing/sysfs-driver-mydev <<'EOF'
What:		/sys/bus/platform/devices/*/mode
Date:		September 2026
KernelVersion:	6.18
Contact:	you@example.com
Description:
		(RW) Operating mode of the device. Valid values 0-3.
		0 = off, 1 = low power, 2 = normal, 3 = turbo.
EOF
```

### Lab 24.3 — Break the compat ABI on purpose, then fix it (T.6)

```c
/* bad.h — a struct that changes layout between 32- and 64-bit */
struct bad_arg {
	int    count;
	void  *buffer;
	long   flags;
	time_t when;
};

/* good.h */
struct good_arg {
	__u32          count;
	__u32          flags;
	__aligned_u64  buffer;
	__s64          when_ns;
	__u32          __reserved[4];
};
```
```bash
cat > /tmp/layout.c <<'EOF'
#include <stdio.h>
#include <stddef.h>
#include "bad.h"
#include "good.h"
int main(void) {
	printf("bad : size=%zu  count@%zu buffer@%zu flags@%zu when@%zu\n",
	  sizeof(struct bad_arg), offsetof(struct bad_arg,count),
	  offsetof(struct bad_arg,buffer), offsetof(struct bad_arg,flags),
	  offsetof(struct bad_arg,when));
	printf("good: size=%zu  count@%zu flags@%zu buffer@%zu when@%zu\n",
	  sizeof(struct good_arg), offsetof(struct good_arg,count),
	  offsetof(struct good_arg,flags), offsetof(struct good_arg,buffer),
	  offsetof(struct good_arg,when_ns));
	return 0;
}
EOF
gcc -m64 -o /tmp/l64 /tmp/layout.c && /tmp/l64
gcc -m32 -o /tmp/l32 /tmp/layout.c && /tmp/l32   # needs gcc-multilib
diff <(/tmp/l64) <(/tmp/l32)
```
**`bad_arg` differs; `good_arg` is identical.** Also test the `__aligned_u64` trap: build a
struct with a bare `__u64` after a `__u32` and compare 32-bit x86 vs x86-64 — the padding
differs, and that is a real, shipped bug class.

```bash
# The kernel's own checks:
make headers_check 2>/dev/null || make headers_install
./scripts/checkpatch.pl --strict --file include/uapi/linux/mysys.h
```

### Lab 24.4 — Write a correct ioctl, then fuzz it

```c
static long my_ioctl(struct file *f, unsigned int cmd, unsigned long arg)
{
	struct mydev *d = f->private_data;
	void __user *uarg = (void __user *)arg;
	struct my_cfg cfg;

	switch (cmd) {
	case MY_IOC_GET: {
		struct my_info info = {};

		guard(mutex)(&d->lock);
		info.version = 1;
		info.mode    = d->mode;
		/* ★ the struct was zero-initialized: no kernel stack leak */
		return copy_to_user(uarg, &info, sizeof(info)) ? -EFAULT : 0;
	}
	case MY_IOC_SET:
		if (copy_from_user(&cfg, uarg, sizeof(cfg)))
			return -EFAULT;
		if (cfg.flags & ~MY_VALID_FLAGS)
			return -EINVAL;
		if (memchr_inv(cfg.__reserved, 0, sizeof(cfg.__reserved)))
			return -EINVAL;
		if (cfg.len > MY_MAX_LEN || cfg.mode > MODE_MAX)
			return -EINVAL;
		guard(mutex)(&d->lock);
		return my_apply(d, &cfg);
	default:
		return -ENOTTY;
	}
}
```
Now fuzz it (Ch. 06 Lab 6.9). Write the syzkaller description:
```
# sys/linux/dev_mydev.txt
resource fd_mydev[fd]
openat$mydev(fd const[AT_FDCWD], file ptr[in, string["/dev/mydev"]],
             flags const[O_RDWR], mode const[0]) fd_mydev
my_cfg {
	flags   flags[my_flags, int32]
	mode    int32[0:8]
	len     int32
	__res   array[const[0, int8], 16]
}
my_flags = MY_FLAG_A, MY_FLAG_B
ioctl$MY_IOC_SET(fd fd_mydev, cmd const[MY_IOC_SET], arg ptr[in, my_cfg])
ioctl$MY_IOC_GET(fd fd_mydev, cmd const[MY_IOC_GET], arg ptr[out, my_info])
```
```bash
# Check for info leaks specifically:
./scripts/config -e KMSAN     # uninitialized-memory sanitizer (Ch. 06 T.5)
# A missing "= {}" on `info` will now be caught as a kernel-infoleak.
```
**The uninitialized-struct infoleak is the single most common ioctl CVE.** KMSAN finds it
mechanically. Deliberately remove the `= {}` and watch it fire.

### Lab 24.5 — A `seq_file` in `/proc` and debugfs

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include <linux/debugfs.h>
#include <linux/slab.h>

struct entry { struct list_head node; char name[16]; u64 count; };
static LIST_HEAD(entries);
static DEFINE_MUTEX(entries_lock);

static void *e_start(struct seq_file *m, loff_t *pos)
	__acquires(&entries_lock)
{
	mutex_lock(&entries_lock);
	if (*pos == 0)
		seq_puts(m, "NAME             COUNT\n");
	return seq_list_start(&entries, *pos);
}
static void *e_next(struct seq_file *m, void *v, loff_t *pos)
{ return seq_list_next(v, &entries, pos); }
static void e_stop(struct seq_file *m, void *v)
	__releases(&entries_lock)
{ mutex_unlock(&entries_lock); }
static int e_show(struct seq_file *m, void *v)
{
	struct entry *e = list_entry(v, struct entry, node);

	seq_printf(m, "%-16s %llu\n", e->name, e->count);   /* ★ no side effects */
	return 0;
}
static const struct seq_operations e_sops = {
	.start = e_start, .next = e_next, .stop = e_stop, .show = e_show,
};
DEFINE_SEQ_ATTRIBUTE(e);

static int __init sf_init(void)
{
	int i;

	for (i = 0; i < 2000; i++) {          /* enough to exceed one page */
		struct entry *e = kzalloc(sizeof(*e), GFP_KERNEL);

		snprintf(e->name, sizeof(e->name), "obj%04d", i);
		e->count = i * 1337ULL;
		list_add_tail(&e->node, &entries);
	}
	proc_create_seq("mydriver", 0444, NULL, &e_sops);
	debugfs_create_file("mydriver", 0444, NULL, NULL, &e_fops);
	return 0;
}
```
```bash
cat /proc/mydriver | head
cat /proc/mydriver | wc -l                    # 2001 lines: multi-page works
dd if=/proc/mydriver bs=17 count=5 2>/dev/null # tiny reads: restart logic works
```
**Then break it on purpose**: use a fixed `sprintf` buffer instead and watch truncation at
one page. That is why `seq_file` exists.

### Lab 24.6 — Generic netlink with a validation policy

```c
// SPDX-License-Identifier: GPL-2.0
#include <net/genetlink.h>

enum { MYNL_A_UNSPEC, MYNL_A_NAME, MYNL_A_VALUE, MYNL_A_RANGE, __MYNL_A_MAX };
#define MYNL_A_MAX (__MYNL_A_MAX - 1)
enum { MYNL_C_UNSPEC, MYNL_C_SET, MYNL_C_GET, __MYNL_C_MAX };

static const struct nla_policy mynl_policy[MYNL_A_MAX + 1] = {
	[MYNL_A_NAME]  = { .type = NLA_NUL_STRING, .len = 16 },
	[MYNL_A_VALUE] = { .type = NLA_U32 },
	[MYNL_A_RANGE] = NLA_POLICY_RANGE(NLA_U32, 1, 100),   /* ★ declarative bounds */
};

static struct genl_family mynl_family;

static int mynl_set(struct sk_buff *skb, struct genl_info *info)
{
	if (!info->attrs[MYNL_A_NAME])
		return -EINVAL;
	/* The policy already validated type, length, and range. No hand-written checks. */
	pr_info("set %s = %u\n",
		nla_data(info->attrs[MYNL_A_NAME]),
		info->attrs[MYNL_A_VALUE] ? nla_get_u32(info->attrs[MYNL_A_VALUE]) : 0);
	return 0;
}

static int mynl_get(struct sk_buff *skb, struct genl_info *info)
{
	struct sk_buff *msg = genlmsg_new(NLMSG_GOODSIZE, GFP_KERNEL);
	void *hdr;

	if (!msg)
		return -ENOMEM;
	hdr = genlmsg_put(msg, info->snd_portid, info->snd_seq, &mynl_family, 0, MYNL_C_GET);
	nla_put_string(msg, MYNL_A_NAME, "device0");
	nla_put_u32(msg, MYNL_A_VALUE, 42);
	genlmsg_end(msg, hdr);
	return genlmsg_reply(msg, info);
}

static const struct genl_ops mynl_ops[] = {
	{ .cmd = MYNL_C_SET, .doit = mynl_set, .flags = GENL_ADMIN_PERM },
	{ .cmd = MYNL_C_GET, .doit = mynl_get },
};

static struct genl_family mynl_family = {
	.name = "MYNL", .version = 1,
	.maxattr = MYNL_A_MAX, .policy = mynl_policy,
	.module = THIS_MODULE, .ops = mynl_ops, .n_ops = ARRAY_SIZE(mynl_ops),
};

static int __init nl_init(void) { return genl_register_family(&mynl_family); }
static void __exit nl_exit(void) { genl_unregister_family(&mynl_family); }
```
```bash
sudo insmod mynl.ko
genl ctrl list | grep -A5 MYNL
genl -r MYNL get
# or with python:
python3 -c "
from pyroute2 import GenericNetlinkSocket
s = GenericNetlinkSocket(); s.bind('MYNL', None)
print(s.nlm_request(...))"

# Watch the policy reject bad input — no hand-written validation required:
$EDITOR Documentation/netlink/specs/     # the YAML IDL; generate bindings from a spec
```

### Lab 24.7 — Audit the syscall surface

```bash
# How many syscalls, per architecture?
wc -l arch/x86/entry/syscalls/syscall_64.tbl
grep -c '__SYSCALL' include/uapi/asm-generic/unistd.h

# Which ones have compat handlers (and therefore compat risk)?
grep -c 'compat' arch/x86/entry/syscalls/syscall_64.tbl

# The ioctl registry — read the scale of the problem
wc -l Documentation/userspace-api/ioctl/ioctl-number.rst
git grep -c 'unlocked_ioctl' -- drivers/ | wc -l

# Which recent syscalls use the extensible-struct pattern?
git grep -l 'copy_struct_from_user'
git log --oneline -S'copy_struct_from_user' | head -20

# Which UAPI headers changed last release? (ABI review targets)
git diff --stat v6.11..v6.12 -- include/uapi/ | tail -20

# Trace all syscalls a program makes, with arguments decoded:
strace -f -e trace=all -s 200 ./myprog 2>&1 | head -40
sudo perf trace -a --summary -- sleep 3
sudo bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[args->id] = count(); }'
```

### Lab 24.8 — Write the selftest and the man page (T.11)

```c
/* tools/testing/selftests/mysys/mysys_test.c */
// SPDX-License-Identifier: GPL-2.0
#include "../kselftest_harness.h"
#include <sys/syscall.h>
#include "../../../../include/uapi/linux/mysys.h"
#define __NR_mysys 463

TEST(basic_upper)
{
	char in[] = "abc", out[8] = {};
	struct mysys_args a = { .flags = MYSYS_FLAG_UPPER,
	                        .in = (unsigned long)in, .out = (unsigned long)out, .len = 3 };
	ASSERT_EQ(3, syscall(__NR_mysys, &a, sizeof(a)));
	ASSERT_STREQ("ABC", out);
}

TEST(reject_unknown_flags)
{
	struct mysys_args a = { .flags = 0x80000000, .len = 1 };
	ASSERT_EQ(-1, syscall(__NR_mysys, &a, sizeof(a)));
	ASSERT_EQ(EINVAL, errno);
}

TEST(reject_nonzero_reserved)
{
	struct mysys_args a = { .len = 1, .__reserved = { 1, 0 } };
	ASSERT_EQ(-1, syscall(__NR_mysys, &a, sizeof(a)));
	ASSERT_EQ(EINVAL, errno);
}

TEST(old_userspace_short_struct)
{
	char in[] = "x", out[4] = {};
	struct mysys_args a = { .in = (unsigned long)in, .out = (unsigned long)out, .len = 1 };
	ASSERT_EQ(1, syscall(__NR_mysys, &a, offsetof(struct mysys_args, __reserved)));
}

TEST(new_userspace_nonzero_tail)
{
	struct { struct mysys_args base; unsigned long future; } big = {};
	big.base.len = 1; big.future = 1;
	ASSERT_EQ(-1, syscall(__NR_mysys, &big, sizeof(big)));
	ASSERT_EQ(E2BIG, errno);
}

TEST_HARNESS_MAIN
```
```bash
make -C tools/testing/selftests TARGETS=mysys run_tests
```
Then write `man2/mysys.2` in the standard format (NAME / SYNOPSIS / DESCRIPTION / RETURN
VALUE / **ERRORS** / VERSIONS / CONFORMING TO / NOTES / EXAMPLES) and read
`Documentation/process/adding-syscalls.rst` to confirm you have satisfied every obligation.

---

## 3. Mastery drills

1. **Read `Documentation/process/adding-syscalls.rst`** completely. Produce a one-page
   checklist you will apply to any UAPI you write. Compare it with T.11.

2. **Study `openat2()`.** `git log --oneline -- fs/open.c | grep -i openat2`, then read
   Aleksa Sarai's cover letter on lore. Explain: why a new syscall rather than more `O_*`
   flags to `openat`? What does `RESOLVE_*` solve that `O_*` could not? How does
   `struct open_how` use T.3's pattern?

3. **Find an ABI break.** `git log --grep='Revert' --grep='ABI' --all-match --oneline | head`.
   Read three. For each, state what was observable, who depended on it, and what Hyrum's Law
   would have predicted.

4. **`statx()` vs `stat()`.** Read `include/uapi/linux/stat.h`. Explain the mask-based
   design: how does the caller request fields, how does the kernel report which it filled,
   and why is that better than a size field here?

5. **Compat archaeology.** `git log --oneline --grep='compat_ioctl' -- drivers/ | head -30`.
   Find one that was a security bug. Explain the layout divergence that caused it and which
   T.6 rule would have prevented it.

6. **Design review.** Take these three proposals and critique each: (a) a sysfs file
   containing JSON; (b) an ioctl taking a `struct` with a `void *` and a `long`;
   (c) a `/proc` file whose output format depends on a module parameter. For each, name the
   rule violated and propose a correct design.

7. **The seccomp constraint.** Explain why `ioctl` and `io_uring` are hard to filter with
   seccomp, and what that means for container security. Then read
   `Documentation/userspace-api/seccomp_filter.rst` and explain what `SECCOMP_RET_USER_NOTIF`
   offers as a partial answer.

8. **Interface archaeology.** Pick a subsystem (input, DRM, V4L2, ALSA, NVMe). Enumerate
   every userspace interface it exposes. Classify each by T.5's table. Find one you think is
   in the wrong family and argue the alternative.

9. **Netlink policy.** Read `include/net/netlink.h`'s policy macros. Write a policy for a
   message with: a required 16-byte string, an optional u32 in [1,1000], an optional nested
   attribute containing two u64s, and a binary blob of at most 4 KiB. Then explain what the
   hand-written validation would have looked like and count the lines saved.

10. **`sysfs_emit` audit.** `git grep -n 'sprintf(buf' -- drivers/ | head -30`. Each is a
    potential overflow. Convert three to `sysfs_emit()` and submit. (`git log --grep="use
    sysfs_emit"` shows hundreds of precedents — a good first-patch area.)

11. **Information leak hunt.** Find an ioctl in `drivers/` that copies a struct to userspace
    without zero-initializing it. Determine whether padding bytes leak kernel stack. Build
    with KMSAN and prove it.

12. **Design question.** You must expose a device's telemetry: 40 counters, updated at
    100 kHz, readable by an unprivileged monitoring agent at 1 Hz, plus an event stream for
    threshold crossings, plus 5 configuration values that must be set atomically as a group.
    Design the complete interface. Justify each mechanism choice against T.5, and state how
    you will extend it in three years.

13. **Write a UAPI review checklist** and use it to review a real patch series on
    lore.kernel.org that adds a new interface. Post nothing — just write the review you would
    have sent, then compare with what the actual reviewers said.

---

## 4. Further reading

**Kernel documentation:**
- `Documentation/process/adding-syscalls.rst` ★★ — **the checklist; read it before designing**
- `Documentation/process/stable-api-nonsense.rst` — the other half of the contract
- `Documentation/userspace-api/` ★★ — the whole directory, especially `ioctl/`,
  `seccomp_filter.rst`, `no_new_privs.rst`, `landlock.rst`, `netlink/`
- `Documentation/userspace-api/ioctl/ioctl-number.rst` — the magic-number registry
- `Documentation/filesystems/sysfs.rst` ★ — the one-value-per-file rule, from the source
- `Documentation/ABI/README` ★ — stability classes and how to document an attribute
- `Documentation/filesystems/seq_file.rst` ★
- `Documentation/netlink/` and `Documentation/netlink/specs/` — the YAML IDL
- `Documentation/core-api/kernel-api.rst` (user-space access section)

**Source to study as exemplars of *good* design:**
- `openat2()` / `struct open_how` — `fs/open.c`, `include/uapi/linux/openat2.h`
- `clone3()` / `struct clone_args` — `kernel/fork.c`
- `statx()` — mask-based field negotiation
- `sched_setattr()` — the original extensible-struct user
- `io_uring` — shared-ring design and feature negotiation
- `landlock` — a new security interface designed with all of T.3/T.10 in mind
- `lib/usercopy.c`, `include/linux/uaccess.h` — `copy_struct_from_user()`

**man pages:**
- `syscall(2)` — the calling conventions per architecture
- `syscalls(2)` — the full list with version history
- `ioctl(2)`, `ioctl_list(2)`, `netlink(7)`, `rtnetlink(7)`, `sysfs(5)`, `proc(5)`
- `seccomp(2)`, `seccomp_unotify(2)`
- `feature_test_macros(7)` — the userspace side of forward compatibility

**Articles & talks:**
- Hyrum Wright, "Hyrum's Law" — https://www.hyrumslaw.com
- LWN: "The extensible syscall" / "Extensible system call arguments"
- LWN: "openat2() and the quest for a safe path resolution"
- LWN: "The rapid growth of io_uring" and its security discussions
- LWN: "Netlink, and the future of kernel interfaces"
- LWN: "A tale of two ABI breaks" / "We do not break user space" retrospectives
- Michael Kerrisk, "API Design for the Linux Kernel" (LCA / Kernel Recipes talks) — **the
  best single talk on this chapter's material**, by the man-pages maintainer
- Kerrisk, *The Linux Programming Interface* — the userspace consumer's perspective on
  every interface you might design

→ Next: [25-sync-patterns.md](25-sync-patterns.md)

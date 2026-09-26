# Chapter 93 — Namespaces, cgroups v2, and Containers from Scratch

> "A container" is not a kernel object. There is no `struct container` anywhere in the tree.
> A container is a *process* with a particular combination of namespaces, cgroups, a pivoted
> root, a seccomp filter, dropped capabilities, and an LSM label. This chapter builds one
> from those primitives, which is the only way to understand what containers do and do not
> isolate — a question Ch. 104 §T.9 framed and this chapter answers concretely.

---

## Theory & First Principles

### T.0 — Start here: there is no such thing as a container

```bash
sudo unshare --pid --fork --mount-proc --net --uts --ipc --mount bash
# You are now "in a container." No Docker. No runtime. Nothing was installed.
ps aux        # you are PID 1, and you can see nothing else
ip link       # one interface: lo
hostname foo  # does not affect the host
```

**Six flags to one syscall.** There is no `struct container` anywhere in the kernel, no
container syscall, no container subsystem. **A container is a process with an unusual
combination of namespaces, cgroups, and security policy** — and Docker, podman, and Kubernetes
are userspace programs that assemble that combination for you.

**Two orthogonal mechanisms do all the work, and confusing them is the most common error:**

| | **Namespaces** | **cgroups** |
|---|---|---|
| Question | **"what can you SEE?"** | **"how much can you USE?"** |
| Kind | **isolation** — visibility | **accounting and limits** — resources |
| Examples | pid, net, mnt, uts, ipc, user, cgroup, time | cpu, memory, io, pids, hugetlb |
| If missing | you see the host's processes/network | you can starve the whole machine |

**Visibility and consumption are independent axes.** You can have perfect namespace isolation
and still be DoS'd by a neighbour's fork bomb; you can have perfect cgroup limits and still
see every process on the host. **Real isolation requires both, plus a third thing the table
does not show** — and that third thing is the point of this chapter.

**Because here is the uncomfortable fact:**

```
     VM                                   CONTAINER
  +------------+  +------------+       +-------+ +-------+ +-------+
  | guest      |  | guest      |       | proc  | | proc  | | proc  |
  | kernel     |  | kernel     |       +-------+ +-------+ +-------+
  +------------+  +------------+       +-----------------------------+
  +------------------------------+     |   ONE SHARED KERNEL         |
  |   hypervisor (~50K lines)    |     |   (~30,000,000 lines,       |
  +------------------------------+     |    ~400 syscalls)           |
                                       +-----------------------------+

   attack surface: a narrow, purpose-built interface
                   vs. the ENTIRE kernel syscall API
```

**A container escape is one kernel bug away.** Every one of ~400 syscalls, every `ioctl`,
every `/proc` and `/sys` file, every filesystem parser reachable from the container is
surface. This is not a criticism of containers — it is the trade they make: **near-zero
overhead and fast startup, in exchange for a shared TCB.** Knowing which side of that trade
you are on is the entire security conversation, and it is why gVisor, Kata, and Firecracker
exist.

**Hence the third mechanism: reduce the surface.**

```
  seccomp-bpf    -> allow only ~50 of ~400 syscalls (Docker's default profile)
  capabilities   -> drop CAP_SYS_ADMIN and the other 30+ pieces of root
  LSM (SELinux / AppArmor)  -> mandatory access control on top of DAC
  user namespaces -> be root INSIDE and uid 100000 OUTSIDE
```

**User namespaces are the most interesting and the most double-edged.** They let an
unprivileged user create a namespace in which they are root — which enables rootless
containers, a genuine security win. It also means **unprivileged users can now reach kernel
code paths that previously required root**, and a large fraction of recent container-escape
CVEs are in exactly that newly-reachable surface. **A mechanism that reduces privilege in one
direction increased attack surface in another**; distributions disagreeing about whether to
enable unprivileged user namespaces is a live and reasonable dispute.

**And one more practical consequence worth knowing**, because it bites everyone: a container
sees the *host's* `/proc/cpuinfo` and `/proc/meminfo` unless something lies to it. A JVM or a
Go runtime sizing its thread pool from `nproc` inside a container limited to 0.5 CPU will
happily create 64 threads. **Namespaces virtualize identity; they largely do not virtualize
resource *reporting*.** That gap is why `lxcfs` exists and why modern runtimes read the cgroup
files directly.

```bash
ls -l /proc/self/ns/                     # your namespaces, as inode numbers
lsns && systemd-cgls | head -30
cat /sys/fs/cgroup/cpu.max /sys/fs/cgroup/memory.max
grep Seccomp /proc/self/status
capsh --print | head
```

---

### T.1 — The two orthogonal mechanisms

| Mechanism | Controls | Answers |
|---|---|---|
| **Namespaces** | *Visibility* | "What can this process see?" |
| **cgroups** | *Resources* | "How much can this process use?" |

They are independent. You can have namespaces without cgroups (isolation without limits) or
cgroups without namespaces (limits without isolation — which is what systemd does for
ordinary services, Ch. 91 §T.6).

Add the security layer from Ch. 102 — capabilities, seccomp, LSM, `no_new_privs` — and you
have a container. Nothing else is required.

### T.2 — The eight namespaces

| Namespace | Flag | Isolates | Since |
|---|---|---|---|
| **Mount** | `CLONE_NEWNS` | The mount table | 2.4.19 (2002) |
| **UTS** | `CLONE_NEWUTS` | hostname, domainname | 2.6.19 |
| **IPC** | `CLONE_NEWIPC` | SysV IPC, POSIX message queues | 2.6.19 |
| **PID** | `CLONE_NEWPID` | Process ID numbering | 2.6.24 |
| **Network** | `CLONE_NEWNET` | Interfaces, routes, iptables, sockets, ports | 2.6.29 |
| **User** | `CLONE_NEWUSER` | uid/gid mappings, **capabilities** | 3.8 (2013) |
| **Cgroup** | `CLONE_NEWCGROUP` | The cgroup root as seen in `/proc/self/cgroup` | 4.6 |
| **Time** | `CLONE_NEWTIME` | `CLOCK_MONOTONIC`/`BOOTTIME` offsets | 5.6 |

Note the dates: the mount namespace predates the others by four years, and the user namespace
arrived a decade later. **Containers were not designed; they accreted**, which is why the
isolation has the seams it does.

**Three that deserve elaboration:**

**PID namespace** is *hierarchical* and nesting-aware. A process has a PID in its own
namespace and in every ancestor. PID 1 inside is special in exactly the ways PID 1 is always
special (Ch. 91 §T.2): it must reap orphans, and **if it dies, every process in the namespace
is killed** (`SIGKILL`). This is why a container exits when its main process exits, and why
running a shell as PID 1 in a container leaves zombies — `sh` does not reap.

**Network namespace** is the most complete isolation of the eight. A fresh netns has only
`lo`, down. Everything — interfaces, routing tables, ARP, netfilter rules, socket port space,
`/proc/net` — is per-namespace. Connecting it to the world requires a `veth` pair, a macvlan,
or moving a physical interface in.

**User namespace** is the one that changes the security model, and §T.3 is about it.

### T.3 — User namespaces: the unprivileged-container enabler and the CVE source

The mechanism:

```
 Inside the namespace          Mapping              Outside
   uid 0    (root)      ◄──────────────────────►   uid 100000
   uid 1                ◄──────────────────────►   uid 100001
   ...                                              ...
   uid 65535            ◄──────────────────────►   uid 165535
```

A process can create a user namespace **without any privilege**, and becomes **root inside
it** — with a full capability set *relative to that namespace*. That is what makes rootless
containers (Podman, `bwrap`, Flatpak) possible.

**What that root can and cannot do:**

| Can | Cannot |
|---|---|
| Create other namespaces | Access files it could not access before (the uid maps to an unprivileged outside uid) |
| Mount `tmpfs`, `proc`, `sysfs`, bind mounts | Mount arbitrary filesystems (a malicious ext4 image is a kernel attack surface) |
| Configure networking *inside* its netns | Reach the host network |
| `chroot`, `pivot_root` | Load modules, write `/dev/mem`, `reboot` |
| Set uid/gid within its mapped range | Escape its mapping |

**The security tension, which you must be able to state:** user namespaces *expand* the
unprivileged attack surface enormously. Code paths that previously required real root —
mount, network configuration, netfilter, much of the namespace machinery itself — became
reachable by any user. A large fraction of Linux LPE CVEs since 2013 are in code made
reachable by user namespaces.

Hence the mitigations:
```bash
sysctl kernel.unprivileged_userns_clone     # Debian/Ubuntu: 0 disables
sysctl user.max_user_namespaces             # 0 disables
# AppArmor's userns mediation (Ubuntu 24.04+)
```

Arguing both sides of "should unprivileged user namespaces be enabled by default" is
`reference/question-bank.md` §I.13, and it is a genuinely open question.

**`/proc/self/uid_map` rules**, which are subtle and exam-worthy:
- Writing it requires `CAP_SETUID` **in the parent** namespace, *unless* you are mapping only
  your own uid.
- Hence `newuidmap`/`newgidmap` (setuid helpers) and `/etc/subuid`/`/etc/subgid` for
  multi-uid rootless containers.
- You must write `/proc/PID/setgroups` as `deny` before writing `gid_map` unprivileged —
  otherwise you could drop a group that was restricting you.

### T.4 — Mount namespaces and propagation

The subtlest namespace, and the one that causes the most confusing bugs.

A mount namespace is a *copy* of the mount table, but mounts have **propagation types** that
govern how changes cross namespaces:

| Type | Behaviour |
|---|---|
| `shared` | Mount events propagate **both** ways between peers |
| `slave` | Events propagate **in** from the master, not out |
| `private` | No propagation |
| `unbindable` | Cannot be bind-mounted |

**The default on modern systemd systems is `shared`**, which is why:

```bash
unshare -m          # new mount namespace
mount -t tmpfs t /mnt
# ... and the HOST also sees /mnt mounted. Surprise.
```

The mount is shared, so it propagated out. Container runtimes do:

```bash
mount --make-rprivate /       # detach from the host's propagation
# or --make-rslave to receive host mounts but not send
```

`--make-rslave` is what Docker uses for `/`: the container sees new host mounts (so a mounted
USB stick appears) but its own mounts do not leak out.

**`pivot_root` versus `chroot`**, an important distinction:

- `chroot` changes the process's root directory. The old root is still **mounted** and
  reachable by a process that keeps a directory fd open — the classic `chroot` escape.
- `pivot_root` moves the *mount* of the old root out of the way so it can be unmounted. After
  `pivot_root` + `umount -l /old`, the old root is genuinely gone.

**Containers use `pivot_root`.** Anything using `chroot` alone is not a security boundary.

### T.5 — cgroups v2

v1 had a separate hierarchy per controller, which made "limit this process's CPU and memory
together" awkward and produced genuinely ambiguous semantics. v2 has **one unified
hierarchy**.

```
 /sys/fs/cgroup/                       <- the single root
 ├── cgroup.controllers                what is available here
 ├── cgroup.subtree_control            what is enabled for CHILDREN
 ├── system.slice/
 │   ├── cgroup.procs                  PIDs in this cgroup
 │   ├── cpu.max          "200000 100000"   = 2 CPUs
 │   ├── cpu.weight       "100"              = relative share
 │   ├── memory.max       "1G"               = hard limit -> OOM
 │   ├── memory.high      "768M"             = throttle point
 │   ├── memory.min       "128M"             = protected, never reclaimed
 │   ├── memory.current
 │   ├── io.max           "8:0 rbps=1048576"
 │   ├── pids.max         "512"
 │   └── nginx.service/
 └── user.slice/
```

**The two rules of v2 that trip everyone up:**

1. **No internal processes.** A cgroup may have either processes *or* enabled controllers for
   children, not both (except the root). So you cannot put a process in `system.slice` and
   also control `system.slice/nginx.service`. This exists because the semantics of "compete
   against your own children" were never well-defined in v1.

2. **Top-down enabling.** A controller must be enabled in a parent's `subtree_control` before
   a child can use it. You enable downward, one level at a time.

**The controllers you will actually use:**

| Controller | Key knobs | Notes |
|---|---|---|
| `cpu` | `cpu.max` (bandwidth), `cpu.weight` (share), `cpu.stat` | `cpu.max` throttling causes p99 spikes at period boundaries — see Ch. 06 / `debugging-scenarios.md` §5 |
| `memory` | `memory.max`, `.high`, `.low`, `.min`, `.current`, `.stat`, `.events` | `.high` throttles, `.max` kills. `.min`/`.low` are **protection**, which people forget |
| `io` | `io.max`, `io.weight`, `io.stat`, `io.pressure` | Buffered writes are charged at writeback time — needs memcg writeback tracking to be attributed correctly |
| `pids` | `pids.max` | Fork-bomb containment |
| `cpuset` | `cpuset.cpus`, `.mems`, `.cpus.partition` | The RT/NUMA tool (Ch. 103, Ch. 105) |
| `hugetlb`, `rdma`, `misc` | | |

**PSI (Pressure Stall Information)** is the thing to actually monitor:

```bash
cat /sys/fs/cgroup/system.slice/nginx.service/cpu.pressure
# some avg10=0.00 avg60=0.00 avg300=0.00 total=0
# full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

`some` = at least one task stalled; `full` = *all* tasks stalled. It measures **stall time**,
which is the thing you actually care about, rather than utilization, which is not. An
autoscaler reading PSI is measuring distress directly; one reading CPU% is inferring it.

### T.6 — What a container actually is

```
 A container =
     a process, plus:
       namespaces      (what it sees)
       cgroups         (what it may use)
       pivot_root      (its filesystem view)
       capabilities    (dropped to a minimum)
       seccomp-BPF     (syscall surface reduced)
       LSM label       (mandatory policy)
       no_new_privs    (cannot regain privilege)
       [optional] user namespace (rootless)
```

Everything a runtime does is configuring those. The **OCI runtime specification** is
literally a JSON schema describing exactly this set.

**The image is a separate concern.** An OCI image is a stack of tar layers plus a JSON
manifest; the runtime composes them with `overlayfs` into the root filesystem. Images and
runtimes are deliberately decoupled — which is why `runc`, `crun`, `youki`, `gVisor`, and
`Kata` can all run the same image with wildly different isolation properties (Ch. 104 §T.9).

### T.7 — What containers do *not* isolate

The list that matters for any security discussion, and the honest answer to "are containers
secure?"

| Not isolated | Consequence |
|---|---|
| **The kernel itself** | One LPE escapes everything. This is the whole issue |
| **Microarchitectural state** | LLC, TLB, branch predictors, SMT siblings — Ch. 105 §T.8, Ch. 102 §T.7 |
| **LLC and memory bandwidth** | No cgroup controller. `resctrl` is the only answer (Ch. 105 §T.8) |
| **The dentry/inode cache** | A tenant doing `find /` evicts everyone's metadata |
| **Page-allocator fragmentation** | One tenant's huge-page churn affects others |
| **conntrack table** | Global, and exhaustible |
| **Futex hash buckets, PID space limits** | Shared |
| **The clock** (mostly) | Time namespace covers only MONOTONIC/BOOTTIME offsets |
| **Kernel log** | `dmesg_restrict` helps; not namespaced |
| **Much of `/proc` and `/sys`** | Leaks host information; runtimes mask specific paths |
| **Timing** | Any shared resource is a covert channel |

**The one-sentence version for an interview:** *containers isolate namespaces and account
resources; they do not isolate the kernel, and therefore they are an operational boundary,
not a security boundary against hostile code. For hostile tenants, use a microVM.*

### T.8 — Container networking

```
 ┌─────────── host netns ──────────────┐
 │                                     │
 │  eth0 ──── docker0 (bridge)         │
 │              │     │                │
 │           veth0  veth2              │
 └──────────────┼─────┼────────────────┘
                │     │
     ┌──────────┴──┐  └──────────────┐
     │ container A │  │ container B  │
     │  eth0 (veth)│  │  eth0 (veth) │
     │  10.0.0.2   │  │  10.0.0.3    │
     └─────────────┘  └──────────────┘
```

The options, and what each costs:

| Mode | Mechanism | Performance | Use |
|---|---|---|---|
| **bridge** | veth pair + Linux bridge + NAT | veth is a full netdev pair; ~10–20% overhead | The default |
| **host** | No netns at all | native | Trusted, performance-critical |
| **macvlan / ipvlan** | A virtual interface on the physical NIC | near-native | Containers need real L2/L3 addresses |
| **none** | An empty netns | — | No networking wanted |
| **SR-IOV VF** | A hardware VF moved into the netns | native | High performance, needs hardware |
| **eBPF (Cilium)** | XDP/tc programs replace bridge+iptables | best software option | Large clusters |

**The `iptables` versus eBPF story is worth knowing.** Kubernetes' original `kube-proxy` used
`iptables` rules, which are evaluated **linearly** — O(n) in the number of services. At
thousands of services this became a real bottleneck, and the fixes were IPVS mode (hash
tables) and then eBPF (Cilium), which does the lookup in a BPF map (Ch. 75). It is a clean
example of Ch. 105's "measure, then fix the data structure."

---

## 1. Internals

### Source map

| Path | Contents |
|---|---|
| `kernel/nsproxy.c` | `struct nsproxy` — the per-task namespace pointers |
| `kernel/pid_namespace.c` | PID namespaces, hierarchy, PID 1 semantics |
| `kernel/user_namespace.c` | **uid/gid mapping, `cap_capable` in a userns** |
| `kernel/utsname.c`, `ipc/namespace.c` | UTS, IPC |
| `net/core/net_namespace.c` | Network namespaces |
| `fs/namespace.c` | **Mount namespaces and propagation** — the hardest file here |
| `fs/pnode.c` | Mount propagation (`shared`/`slave`/`private`) |
| `kernel/cgroup/cgroup.c` | The cgroup core |
| `kernel/cgroup/cpuset.c`, `mm/memcontrol.c`, `block/blk-cgroup.c` | Controllers |
| `kernel/sched/psi.c` | Pressure Stall Information |
| `kernel/fork.c` | `clone()` and the `CLONE_NEW*` handling |
| `kernel/nsproxy.c`: `setns()` | Joining an existing namespace |
| `Documentation/admin-guide/cgroup-v2.rst` | **The authoritative cgroup v2 document** |

### The syscalls

```c
/* Create namespaces along with a new process. */
int clone(int (*fn)(void *), void *stack, int flags, void *arg);
/* flags: CLONE_NEWNS|NEWUTS|NEWIPC|NEWPID|NEWNET|NEWUSER|NEWCGROUP|NEWTIME */

/* Create namespaces for the CALLING process (no new process). */
int unshare(int flags);
/* Note: CLONE_NEWPID with unshare affects CHILDREN, not the caller --
 * a process cannot change its own PID.                               */

/* Join an existing namespace via an fd from /proc/PID/ns/*. */
int setns(int fd, int nstype);

/* The modern, extensible form (Ch. 24 §T.4's pattern). */
int clone3(struct clone_args *args, size_t size);

/* Filesystem view. */
int pivot_root(const char *new_root, const char *put_old);
int mount(...);   /* MS_BIND, MS_REC, MS_PRIVATE, MS_SLAVE, MS_MOVE */

/* The new mount API (5.2+), which containers increasingly use. */
int fsopen(const char *fsname, unsigned int flags);
int fsconfig(int fd, unsigned int cmd, const char *key, const void *value, int aux);
int fsmount(int fd, unsigned int flags, unsigned int ms_flags);
int move_mount(int from_dfd, const char *from, int to_dfd, const char *to, unsigned);
int open_tree(int dfd, const char *filename, unsigned flags);
```

The new mount API exists because the old `mount(2)` conflates "parse options," "create a
superblock," and "attach to the tree" into one call with a `void *data` argument and no way
to report *which* option failed. The new API separates them and gives real error messages —
another instance of Ch. 24's design lessons.

### Observability

```bash
# --- Namespaces ---
lsns                                  # every namespace on the system
lsns -t net -t pid
ls -l /proc/self/ns/                  # this process's namespaces (inodes are IDs)
readlink /proc/self/ns/net            # net:[4026531840]
# Two processes share a namespace iff the inode numbers match.

nsenter -t <pid> -n -p -m -- bash     # enter an existing container's namespaces

# --- User namespace mappings ---
cat /proc/<pid>/uid_map /proc/<pid>/gid_map
cat /proc/<pid>/setgroups
grep -E 'CapInh|CapPrm|CapEff|CapBnd|CapAmb' /proc/<pid>/status
capsh --decode=$(grep CapEff /proc/<pid>/status | cut -f2)

# --- cgroups ---
cat /proc/self/cgroup                 # "0::/path" for v2
systemd-cgls                          # the tree
systemd-cgtop                         # top, per-cgroup
cat /sys/fs/cgroup/<path>/cgroup.controllers
cat /sys/fs/cgroup/<path>/memory.current /sys/fs/cgroup/<path>/memory.events
cat /sys/fs/cgroup/<path>/cpu.stat    # nr_throttled, throttled_usec  <- THIS ONE
cat /sys/fs/cgroup/<path>/{cpu,memory,io}.pressure

# --- Mount propagation (the confusing one) ---
findmnt -o TARGET,PROPAGATION,SOURCE,FSTYPE
cat /proc/self/mountinfo | awk '{print $5, $7}'    # mountpoint, optional fields

# --- Security posture of a running container ---
grep -E 'Seccomp|NoNewPrivs' /proc/<pid>/status
```

---

## 2. Practice

### Lab 93.1 — A container in 200 lines of C

```c
/* minicontainer.c — a container from first principles.
 * Build: gcc -O2 -Wall -o minicontainer minicontainer.c
 * Run:   sudo ./minicontainer /path/to/rootfs /bin/sh
 *        (or unprivileged with -u for a user namespace)
 *
 * Demonstrates: namespaces, pivot_root, cgroup v2, capability drop,
 * seccomp, and no_new_privs -- i.e. EVERY component of T.6.       */
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <fcntl.h>
#include <sched.h>
#include <signal.h>
#include <sys/mount.h>
#include <sys/wait.h>
#include <sys/syscall.h>
#include <sys/prctl.h>
#include <sys/capability.h>
#include <linux/sched.h>
#include <linux/limits.h>

#define STACK_SIZE (1024 * 1024)
#define CG_ROOT    "/sys/fs/cgroup"
#define CG_NAME    "minicontainer"

struct config {
	char *rootfs;
	char **argv;
	int   userns;
	long  mem_max;      /* bytes */
	long  pids_max;
	int   sync_pipe[2];
};

static int write_file(const char *path, const char *fmt, ...)
{
	char buf[256];
	va_list ap;
	int fd, n, r;

	va_start(ap, fmt);
	n = vsnprintf(buf, sizeof(buf), fmt, ap);
	va_end(ap);

	fd = open(path, O_WRONLY);
	if (fd < 0) return -1;
	r = write(fd, buf, n);
	close(fd);
	return r == n ? 0 : -1;
}

/* ---------------- cgroup v2 ---------------- */
static int cgroup_setup(struct config *c, pid_t pid)
{
	char path[PATH_MAX];

	/* Enable the controllers we need for children of the root. */
	write_file(CG_ROOT "/cgroup.subtree_control", "+memory +pids +cpu");

	snprintf(path, sizeof(path), CG_ROOT "/" CG_NAME);
	if (mkdir(path, 0755) && errno != EEXIST) {
		perror("mkdir cgroup");
		return -1;
	}

	snprintf(path, sizeof(path), CG_ROOT "/" CG_NAME "/memory.max");
	write_file(path, "%ld", c->mem_max);

	/* memory.high throttles BEFORE memory.max kills -- graceful
	 * degradation rather than an OOM kill (Ch. 23 §T.6).          */
	snprintf(path, sizeof(path), CG_ROOT "/" CG_NAME "/memory.high");
	write_file(path, "%ld", (long)(c->mem_max * 0.8));

	snprintf(path, sizeof(path), CG_ROOT "/" CG_NAME "/pids.max");
	write_file(path, "%ld", c->pids_max);

	/* Move the child in. THIS is what makes the limits apply. */
	snprintf(path, sizeof(path), CG_ROOT "/" CG_NAME "/cgroup.procs");
	if (write_file(path, "%d", pid)) {
		perror("cgroup.procs");
		return -1;
	}
	return 0;
}

/* ---------------- capabilities ---------------- */
static void drop_capabilities(void)
{
	/* Drop everything dangerous from the BOUNDING set, so even a
	 * setuid binary cannot regain it.                              */
	static const int keep[] = {
		CAP_CHOWN, CAP_DAC_OVERRIDE, CAP_FOWNER, CAP_FSETID,
		CAP_KILL, CAP_SETGID, CAP_SETUID, CAP_SETPCAP,
		CAP_NET_BIND_SERVICE, CAP_SYS_CHROOT, CAP_AUDIT_WRITE,
	};
	for (int cap = 0; cap <= CAP_LAST_CAP; cap++) {
		int k = 0;
		for (size_t i = 0; i < sizeof(keep)/sizeof(keep[0]); i++)
			if (keep[i] == cap) { k = 1; break; }
		if (!k)
			prctl(PR_CAPBSET_DROP, cap, 0, 0, 0);
	}
	/* Explicitly note what we dropped and why:
	 *   CAP_SYS_ADMIN   -- effectively root (Ch. 102 §T.4)
	 *   CAP_SYS_MODULE  -- load kernel modules
	 *   CAP_SYS_RAWIO   -- /dev/mem, ioperm
	 *   CAP_SYS_PTRACE  -- inspect other processes
	 *   CAP_NET_ADMIN   -- reconfigure networking
	 *   CAP_SYS_BOOT    -- reboot, kexec
	 */
}

/* ---------------- seccomp ---------------- */
static void seccomp_setup(void)
{
	/* A denylist for demonstration; a real runtime uses an allowlist
	 * (Ch. 102 §T.5). Docker's default profile denies ~44 syscalls. */
	struct sock_filter filter[] = {
		BPF_STMT(BPF_LD | BPF_W | BPF_ABS,
			 offsetof(struct seccomp_data, arch)),
		BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, AUDIT_ARCH_X86_64, 1, 0),
		BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL_PROCESS),

		BPF_STMT(BPF_LD | BPF_W | BPF_ABS,
			 offsetof(struct seccomp_data, nr)),
#define DENY(nr) \
	BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_##nr, 0, 1), \
	BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ERRNO | (EPERM & SECCOMP_RET_DATA))
		DENY(kexec_load), DENY(init_module), DENY(finit_module),
		DENY(delete_module), DENY(reboot), DENY(mount), DENY(umount2),
		DENY(pivot_root), DENY(ptrace), DENY(perf_event_open),
		DENY(bpf), DENY(iopl), DENY(ioperm),
		/* io_uring: seccomp does NOT filter io_uring OPERATIONS,
		 * so a sandbox must block ring creation itself
		 * (Ch. 76 §T.7 / Lab 76.7).                              */
		DENY(io_uring_setup),
#undef DENY
		BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
	};
	struct sock_fprog prog = {
		.len = sizeof(filter) / sizeof(filter[0]),
		.filter = filter,
	};

	if (prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0))
		perror("no_new_privs");
	if (syscall(SYS_seccomp, SECCOMP_SET_MODE_FILTER, 0, &prog))
		perror("seccomp");
}

/* ---------------- filesystem ---------------- */
static int setup_rootfs(const char *rootfs)
{
	char old[PATH_MAX];

	/* 1. Detach from the host's mount propagation. Without this, our
	 *    mounts LEAK to the host (T.4).                             */
	if (mount(NULL, "/", NULL, MS_REC | MS_PRIVATE, NULL)) {
		perror("make-rprivate");
		return -1;
	}

	/* 2. pivot_root requires new_root to be a mount point. */
	if (mount(rootfs, rootfs, NULL, MS_BIND | MS_REC, NULL)) {
		perror("bind rootfs");
		return -1;
	}

	if (chdir(rootfs)) { perror("chdir"); return -1; }

	snprintf(old, sizeof(old), "%s/.oldroot", rootfs);
	mkdir(old, 0700);

	/* 3. pivot_root, NOT chroot -- the old root must become
	 *    unmountable (T.4).                                          */
	if (syscall(SYS_pivot_root, ".", ".oldroot")) {
		perror("pivot_root");
		return -1;
	}
	if (chdir("/")) { perror("chdir /"); return -1; }

	/* 4. Mount the pseudo-filesystems. proc MUST be mounted after
	 *    entering the PID namespace or it shows the host's processes. */
	mkdir("/proc", 0555); mkdir("/sys", 0555); mkdir("/dev", 0755);
	if (mount("proc", "/proc", "proc",
		  MS_NOSUID | MS_NOEXEC | MS_NODEV, NULL))
		perror("mount proc");
	mount("sysfs", "/sys", "sysfs",
	      MS_NOSUID | MS_NOEXEC | MS_NODEV | MS_RDONLY, NULL);
	mount("tmpfs", "/dev", "tmpfs", MS_NOSUID | MS_STRICTATIME, "mode=755");

	/* 5. Unmount the old root. NOW the host filesystem is genuinely
	 *    unreachable -- this is what chroot cannot do.              */
	if (umount2("/.oldroot", MNT_DETACH))
		perror("umount old root");
	rmdir("/.oldroot");
	return 0;
}

/* ---------------- the child ---------------- */
static int child_fn(void *arg)
{
	struct config *c = arg;
	char ch;

	/* Wait for the parent to write the uid/gid maps and set up the
	 * cgroup. Without this synchronization we would run before the
	 * mappings exist and be nobody:nogroup.                         */
	close(c->sync_pipe[1]);
	if (read(c->sync_pipe[0], &ch, 1) != 0) { /* parent closed it */ }
	close(c->sync_pipe[0]);

	if (sethostname("container", 9))
		perror("sethostname");

	if (setup_rootfs(c->rootfs))
		return 1;

	/* Order matters: drop capabilities, THEN seccomp, THEN exec. */
	drop_capabilities();
	seccomp_setup();

	clearenv();
	setenv("PATH", "/usr/local/bin:/usr/bin:/bin", 1);
	setenv("HOME", "/root", 1);
	setenv("TERM", "xterm", 1);

	execvp(c->argv[0], c->argv);
	perror("execvp");
	return 127;
}

/* ---------------- the parent ---------------- */
int main(int argc, char **argv)
{
	struct config c = {
		.mem_max = 256L * 1024 * 1024,
		.pids_max = 128,
	};
	char *stack, *stack_top;
	int flags, status, opt;
	pid_t pid;
	char path[PATH_MAX];

	while ((opt = getopt(argc, argv, "um:p:")) != -1) {
		switch (opt) {
		case 'u': c.userns = 1; break;
		case 'm': c.mem_max = atol(optarg) * 1024 * 1024; break;
		case 'p': c.pids_max = atol(optarg); break;
		default:
			fprintf(stderr,
			  "usage: %s [-u] [-m MB] [-p PIDS] <rootfs> <cmd> [args]\n",
			  argv[0]);
			return 1;
		}
	}
	if (argc - optind < 2) {
		fprintf(stderr, "usage: %s [-u] <rootfs> <cmd> [args...]\n", argv[0]);
		return 1;
	}
	c.rootfs = argv[optind];
	c.argv   = &argv[optind + 1];

	if (pipe(c.sync_pipe)) { perror("pipe"); return 1; }

	stack = malloc(STACK_SIZE);
	if (!stack) { perror("malloc"); return 1; }
	stack_top = stack + STACK_SIZE;

	flags = CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWIPC |
		CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWCGROUP | SIGCHLD;
	if (c.userns)
		flags |= CLONE_NEWUSER;

	pid = clone(child_fn, stack_top, flags, &c);
	if (pid < 0) { perror("clone"); return 1; }

	printf("container pid (host view): %d\n", pid);

	/* uid/gid maps, if using a user namespace. Order matters:
	 * setgroups=deny BEFORE gid_map, or the write is refused. */
	if (c.userns) {
		snprintf(path, sizeof(path), "/proc/%d/setgroups", pid);
		write_file(path, "deny");
		snprintf(path, sizeof(path), "/proc/%d/uid_map", pid);
		write_file(path, "0 %d 1", getuid());
		snprintf(path, sizeof(path), "/proc/%d/gid_map", pid);
		write_file(path, "0 %d 1", getgid());
	}

	if (!c.userns)
		cgroup_setup(&c, pid);

	/* Release the child. */
	close(c.sync_pipe[1]);
	close(c.sync_pipe[0]);

	if (waitpid(pid, &status, 0) < 0) perror("waitpid");
	printf("container exited: %d\n", WEXITSTATUS(status));

	snprintf(path, sizeof(path), CG_ROOT "/" CG_NAME);
	rmdir(path);
	free(stack);
	return WEXITSTATUS(status);
}
```

```bash
# Build a rootfs to run in:
mkdir -p rootfs && cd rootfs
# Option A: from a container image
docker export $(docker create alpine) | tar -x
# Option B: busybox
mkdir -p bin sbin etc proc sys dev tmp root
cp $(which busybox) bin/ && chroot . /bin/busybox --install -s
cd ..

sudo ./minicontainer -m 128 -p 64 ./rootfs /bin/sh

# Inside, verify every isolation:
ps aux                       # only our processes -- PID namespace
hostname                     # "container" -- UTS namespace
ip addr                      # only lo, down -- network namespace
ls /                         # the rootfs -- mount namespace + pivot_root
cat /proc/self/cgroup        # our cgroup -- cgroup namespace
capsh --print                # reduced bounding set
grep Seccomp /proc/self/status   # mode 2 = filter
mount -t tmpfs t /mnt        # EPERM -- seccomp denied it
```

### Lab 93.2 — Namespaces one at a time

The pedagogically clearest lab: use `unshare` to add one namespace at a time and observe.

```bash
# --- UTS ---
sudo unshare --uts bash
hostname isolated
hostname                    # isolated
# In ANOTHER terminal: hostname -> unchanged. 

# --- PID ---
sudo unshare --pid --fork --mount-proc bash
ps aux                      # only bash and ps
echo $$                     # 1
#   Without --mount-proc, ps shows the HOST's processes, because
#   /proc is still the host's. The PID namespace changes numbering,
#   not what /proc shows.

# --- Network ---
sudo unshare --net bash
ip link                     # only lo, DOWN
ping 8.8.8.8                # network unreachable
exit

# --- Mount, and the propagation trap ---
sudo unshare --mount bash
mount -t tmpfs t /mnt
# In another terminal:  findmnt /mnt
#   -> on a systemd system, the HOST SEES IT. The mount propagated.
# The fix:
mount --make-rprivate /
mount -t tmpfs t /mnt       # now it does not propagate

# --- User (UNPRIVILEGED!) ---
unshare --user --map-root-user bash
id                          # uid=0(root)
cat /proc/self/uid_map      # 0 1000 1
touch /etc/probe            # Permission denied -- we are root in the
                            #   namespace, uid 1000 outside
capsh --print | head -3     # full capability set... within the namespace

# --- Combine them: the full container ---
unshare --user --map-root-user --mount --uts --ipc --pid --fork \
        --net --cgroup --mount-proc bash
```

### Lab 93.3 — cgroup v2, by hand

```bash
#!/bin/bash
# cgroup_lab.sh — feel every controller.
set -e
CG=/sys/fs/cgroup/lab
sudo mkdir -p $CG

echo "=== Available controllers ==="
cat /sys/fs/cgroup/cgroup.controllers
echo "+cpu +memory +pids +io" | sudo tee /sys/fs/cgroup/cgroup.subtree_control

# --- Memory: watch .high throttle and .max kill ---
echo "=== memory.high (throttle) vs memory.max (kill) ==="
echo "50M"  | sudo tee $CG/memory.high
echo "100M" | sudo tee $CG/memory.max

sudo bash -c "echo \$\$ > $CG/cgroup.procs; exec stress-ng --vm 1 --vm-bytes 200M -t 10s" &
SPID=$!
for i in $(seq 1 10); do
	printf 'current=%-12s high_events=%s\n' \
		"$(cat $CG/memory.current)" \
		"$(grep '^high ' $CG/memory.events | awk '{print $2}')"
	sleep 1
done
wait $SPID 2>/dev/null || echo "  -> OOM killed at memory.max"
cat $CG/memory.events

# --- CPU: watch throttling create latency spikes ---
echo; echo "=== cpu.max throttling ==="
echo "20000 100000" | sudo tee $CG/cpu.max      # 20% of one CPU
sudo bash -c "echo \$\$ > $CG/cgroup.procs; exec stress-ng --cpu 1 -t 10s" &
sleep 10
cat $CG/cpu.stat
#   nr_throttled and throttled_usec: THIS is the p99 latency source
#   from reference/debugging-scenarios.md §5.

# --- PSI: what you should actually monitor ---
echo; echo "=== Pressure ==="
cat $CG/cpu.pressure $CG/memory.pressure $CG/io.pressure

# --- pids: fork-bomb containment ---
echo; echo "=== pids.max ==="
echo 20 | sudo tee $CG/pids.max
sudo bash -c "echo \$\$ > $CG/cgroup.procs
	for i in \$(seq 1 50); do sleep 30 & done 2>&1 | tail -3"
cat $CG/pids.current $CG/pids.events

sudo rmdir $CG
```

### Lab 93.4 — Container networking with veth

```bash
#!/bin/bash
# netns_lab.sh — build container networking by hand.
set -e

# 1. Two network namespaces.
sudo ip netns add ns1
sudo ip netns add ns2

# 2. A bridge on the host.
sudo ip link add br0 type bridge
sudo ip addr add 10.10.0.1/24 dev br0
sudo ip link set br0 up

# 3. veth pairs: one end in the namespace, one on the bridge.
for i in 1 2; do
	sudo ip link add veth$i type veth peer name veth${i}-br
	sudo ip link set veth$i netns ns$i
	sudo ip link set veth${i}-br master br0
	sudo ip link set veth${i}-br up

	sudo ip netns exec ns$i ip addr add 10.10.0.1$i/24 dev veth$i
	sudo ip netns exec ns$i ip link set veth$i up
	sudo ip netns exec ns$i ip link set lo up
	sudo ip netns exec ns$i ip route add default via 10.10.0.1
done

# 4. Test.
sudo ip netns exec ns1 ping -c2 10.10.0.12          # ns1 -> ns2
sudo ip netns exec ns1 ping -c2 10.10.0.1           # ns1 -> host

# 5. NAT for outbound.
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -s 10.10.0.0/24 ! -o br0 -j MASQUERADE
sudo ip netns exec ns1 ping -c2 8.8.8.8

# 6. Measure the cost of veth.
sudo ip netns exec ns1 iperf3 -s -D
iperf3 -c 10.10.0.11 -t 10
#    Compare with host networking (iperf3 over lo) and note the delta.

# 7. Cleanup.
sudo iptables -t nat -D POSTROUTING -s 10.10.0.0/24 ! -o br0 -j MASQUERADE
sudo ip netns del ns1; sudo ip netns del ns2; sudo ip link del br0
```

### Lab 93.5 — Escape a badly-configured container

**Authorized testing only, on your own VM.** The point is to understand what each
misconfiguration costs.

```bash
# --- Escape 1: privileged container ---
docker run --rm -it --privileged alpine sh
# Inside:
fdisk -l                      # you can SEE the host disks
mkdir /host && mount /dev/sda1 /host && ls /host
#   Full host filesystem access. --privileged means "no isolation".

# --- Escape 2: CAP_SYS_ADMIN + writable cgroups (release_agent) ---
#   Historically: mount a cgroup v1 hierarchy, write release_agent,
#   trigger it, execute on the host. Mitigated in v2 and by masking,
#   but the lesson stands: CAP_SYS_ADMIN is root.

# --- Escape 3: the docker socket ---
docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock docker sh
# Inside:
docker run --rm -v /:/host --privileged alpine chroot /host sh
#   Mounting the docker socket IS granting root on the host. This is
#   extremely common in CI configurations and is always a full escape.

# --- Escape 4: host PID namespace ---
docker run --rm -it --pid=host alpine sh
ps aux                        # every host process
#   With CAP_SYS_PTRACE, you can inject into any of them.

# --- Now do it RIGHT ---
docker run --rm -it \
	--read-only \
	--cap-drop=ALL --cap-add=NET_BIND_SERVICE \
	--security-opt=no-new-privileges \
	--security-opt seccomp=/path/to/profile.json \
	--user 1000:1000 \
	--pids-limit 100 \
	--memory 256m --memory-reservation 200m \
	--cpus 0.5 \
	--tmpfs /tmp:rw,noexec,nosuid,size=64m \
	alpine sh
```

**Write up each escape as: what was misconfigured, what it granted, and the one-line fix.**
That document is directly useful to any team running containers.

### Lab 93.6 — Compare isolation boundaries

```bash
# Measure the spectrum from Ch. 104 §T.9 yourself.

# 1. Startup latency.
for rt in runc crun; do
	printf '%-20s ' "$rt"
	( time (for i in $(seq 1 20); do
		docker run --rm --runtime=$rt alpine true; done) ) 2>&1 | grep real
done
# vs a microVM:
( time (for i in $(seq 1 20); do
	docker run --rm --runtime=io.containerd.kata.v2 alpine true; done) ) 2>&1 | grep real

# 2. Memory overhead.
docker run -d --name c1 alpine sleep 300
systemd-cgtop -1 -n1 | grep c1

# 3. Syscall surface. How many syscalls can each reach?
#    container: ~350 of 400 (minus the seccomp profile)
#    gVisor:    ~60 host syscalls
#    microVM:   KVM ioctls + a handful
docker run --rm alpine sh -c 'grep Seccomp /proc/self/status'

# 4. Throughput on a syscall-heavy workload.
for rt in runc runsc kata; do
	docker run --rm --runtime=$rt alpine sh -c \
		'time (for i in $(seq 1 20000); do : ; done)' 2>&1 | tail -3
done
```

Produce the decision matrix. **The conclusion from Ch. 104 §T.9 — that microVMs changed the
economics of the container-as-security-boundary argument — should fall out of your own
numbers.**

---

## 3. Mastery drills

1. Extend `minicontainer.c` to support: a veth pair with an IP, an overlayfs root from image
   layers, a user-namespace-mapped rootfs, and `--read-only`. You will have written a
   minimal OCI runtime.

2. Implement the OCI runtime specification's `config.json` parsing and make your runtime
   usable by `docker --runtime=`. Run a real image with it.

3. Build an overlayfs-based image layering system: pull an OCI image, extract the layers, and
   compose them. Explain copy-up, whiteouts, and opaque directories.

4. Measure the cost of each namespace independently: create a process with only one namespace
   at a time and benchmark `fork`, `exec`, syscall latency, and network throughput.

5. Implement rootless containers correctly with `newuidmap`/`newgidmap` and `/etc/subuid`.
   Explain exactly why the setuid helper is necessary.

6. Build the complete mount-propagation demonstration: shared, slave, private, and
   unbindable, showing which events propagate which way. Then explain what Docker does for
   `/` and why.

7. Write a cgroup-v2 resource manager: given a set of workloads with priorities, set
   `cpu.weight`, `memory.min`/`.high`/`.max`, and `io.weight` appropriately, and drive it from
   PSI.

8. Reproduce the `cpu.max` p99 latency spike from `reference/debugging-scenarios.md` §5, then
   fix it with `cpu.weight` instead and show the difference.

9. Enumerate and test every item in §T.7's "not isolated" table. For each, write a
   demonstration of one tenant affecting another.

10. Compare `iptables`, IPVS, and eBPF (Cilium) service load balancing at 10, 1000, and 10000
    services. Reproduce the O(n) behaviour of iptables mode.

11. Build a container-escape CTF for your team: five deliberately misconfigured containers of
    increasing subtlety. Write the solutions and the fixes.

12. Write the container security standard for an organization: what is mandatory, what is
    forbidden, what requires an exception, and how it is enforced in CI. Ground every rule in
    a specific mechanism from this chapter.

---

## 4. Further reading

**Kernel documentation**
- `Documentation/admin-guide/cgroup-v2.rst` — **long, complete, and the authoritative source.**
  Read the memory and CPU controller sections in full
- `man namespaces(7)`, `user_namespaces(7)`, `pid_namespaces(7)`, `mount_namespaces(7)`,
  `cgroups(7)`, `capabilities(7)` — Kerrisk's man pages are outstanding here
- `Documentation/filesystems/sharedsubtree.txt` — mount propagation, by its designer
- `Documentation/filesystems/overlayfs.rst`
- `Documentation/accounting/psi.rst`

**Essential reading**
- Michael Kerrisk's LWN namespace series (7 parts, 2013) — **the best explanation anywhere**,
  and still accurate
- Jake Edge's and Jonathan Corbet's cgroup v2 coverage on LWN
- The OCI Runtime Specification and Image Specification — short, readable, and they *are* the
  definition of a container
- Tejun Heo's cgroup v2 design documents and the "no internal processes" rationale

**Security**
- "Understanding Docker container escapes" (Trail of Bits) and the container-escape research
  literature
- NIST SP 800-190, *Application Container Security Guide*
- The CIS Docker and Kubernetes Benchmarks
- Brad Geesaman's and Ian Coldwater's container security talks
- The user-namespace CVE history — search `cve.org` for "user namespace" and read three

**Books**
- Liz Rice, *Container Security* — the best single book; practical and accurate
- Liz Rice, "Containers from Scratch" talks — the live-coded version of Lab 93.1
- Nigel Poulton, *Docker Deep Dive* — for the tooling layer
- Chris Simmonds, *Mastering Embedded Linux Programming* — the embedded-container chapters

**Tools**
- `lsns`, `nsenter`, `unshare`, `ip netns` — learn these before any container tooling
- `systemd-cgls`, `systemd-cgtop`, `systemd-run` — the cgroup interface you already have
- `bubblewrap` (`bwrap`) — a small, auditable unprivileged sandbox; read its source
- `crun` — a C OCI runtime, ~10× smaller than `runc` and much more readable
- `podman` — rootless by default; the best system to learn on

**Cross-references**
- Ch. 91 — systemd's use of cgroups for every service
- Ch. 102 — capabilities, seccomp, LSM: the security half of a container
- Ch. 104 §T.9 — the isolation spectrum, and why microVMs changed it
- Ch. 105 §T.8 — the LLC/memory-bandwidth isolation gap that cgroups do not cover

→ Next: [94-storage-admin.md](94-storage-admin.md)

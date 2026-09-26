# Chapter 20 — Processes, Threads, `task_struct`, and fork/exec/exit

> **Goal:** know exactly what a "process" is in Linux (spoiler: it isn't a thing), be able to
> trace `fork()`, `execve()` and `exit()` through the source, and reason correctly about
> signals, credentials, and task lifetime.

---

## Theory & First Principles

### T.0 — Start here: there are no processes and no threads

Every OS textbook teaches that a process contains threads. Linux does not work that way, and
the difference is not pedantic — it explains containers, `fork`, and half the scheduler.

**There is exactly one kernel object:**

```c
struct task_struct { ... };     /* kernel/fork.c, include/linux/sched.h */
```

One per *schedulable entity*. What you call a "thread" is a `task_struct`. What you call a
"process" is **a group of `task_struct`s that happen to share things**. The sharing is
per-resource, chosen at creation:

```c
clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD, ...);
/*     ↑         ↑         ↑            ↑                ↑
       share     share     share fd     share signal     same thread group
       address   cwd/root  table        handlers         (same getpid())      */
```

| What you call it | What it actually is |
|---|---|
| `fork()` | `clone()` sharing **nothing** (everything copied, lazily) |
| `pthread_create()` | `clone()` sharing **VM + FS + FILES + SIGHAND + THREAD** |
| A container | `clone()` sharing nothing, plus **new namespaces** (Ch. 93) |
| `vfork()` | `clone(CLONE_VM \| CLONE_VFORK)` |

**So "process vs thread" is not a binary — it is a point in a space of about a dozen
independent bits.** Verify it:

```bash
ps -eLf | head                        # -L shows THREADS: note LWP vs PID
ls /proc/$$/task/                     # every thread of your shell
cat /proc/$$/status | grep -E 'Tgid|Pid|Threads'
#   Tgid = the "process" id (thread GROUP id).  Pid = this task.
#   getpid() returns TGID. gettid() returns PID. They differ for threads.
```

**Why this design and not the textbook one?** Because the textbook model forces the OS to
decide, once, what a "process" bundles. Linux refuses to decide, and gets three things for
free:

1. **Containers need no new concept.** Add namespace flags to `clone()` and you have
   isolation. There is no `struct container` anywhere in the kernel (Ch. 93 §T.6).
2. **The scheduler has one thing to schedule.** No two-level scheduler, no M:N complexity
   (`os-fundamentals.md` §1) — the model that Solaris and FreeBSD both tried and abandoned.
3. **Unusual sharing combinations are expressible**, and real code uses them — `CLONE_FILES`
   without `CLONE_VM`, `CLONE_FS` without `CLONE_THREAD`.

**Now the historical accident that everyone lives with.** `fork()` duplicates the entire
address space — and then you almost always immediately call `execve()` and throw it away:

```c
pid_t p = fork();               /* duplicate ~everything about this process */
if (p == 0) execve(prog, ...);  /* ...and immediately discard all of it */
```

This is wasteful enough that it required inventing **copy-on-write** to be tolerable
(§T.3), and COW then became the basis of half the memory manager. It also does not compose:
it interacts badly with threads (only the calling thread survives), with locks (another
thread may have held one), with buffered I/O, and with signal handlers. The 2019 paper
*"A `fork()` in the road"* (Baumann et al., HotOS) argues it should never have existed.

**We keep it because `fork` is in POSIX and POSIX is forever** — Ch. 00 §T.5 in its purest
form. Modern code uses `posix_spawn`, `vfork`, or `clone3` with `CLONE_VFORK`, and the
interpretation of `fork`'s survival is an interface-design lesson, not a performance one.

---

### T.1 A process is a bundle of resources, and Linux unbundles it

The textbook definition — "a process is a program in execution" — is useless for kernel work.
The operational definition is: **a process is a bundle of independently-virtualizable
resources**, namely

1. an **execution context** (registers, instruction pointer, stack) — a *virtual CPU*
2. an **address space** — virtual memory
3. a **file descriptor table**
4. a **filesystem context** (cwd, root, umask)
5. a **signal handler table** and signal state
6. a **credential set** (uid/gid/capabilities)
7. a **namespace set** (pid, mnt, net, uts, ipc, user, cgroup, time)
8. a **resource-limit / accounting context** (rlimits, cgroup membership)

Traditional Unix ties all eight together: `fork()` duplicates all of them, and that's that.
Threads were bolted on later as a special case that shares (2)–(8).

**Linux's defining design decision is to refuse the bundle.** There is exactly one kernel
object — `struct task_struct`, a *task* — and `clone()` takes a flag per resource saying
"share or copy":

```c
CLONE_VM        /* share the address space        (2) */
CLONE_FILES     /* share the fd table             (3) */
CLONE_FS        /* share cwd/root/umask           (4) */
CLONE_SIGHAND   /* share signal handlers          (5) */
CLONE_THREAD    /* same thread group (same TGID)      */
CLONE_NEWPID / NEWNET / NEWNS / NEWUSER / NEWUTS / NEWIPC / NEWCGROUP / NEWTIME  /* (7) */
CLONE_IO        /* share the block I/O context        */
CLONE_PARENT, CLONE_PTRACE, CLONE_SETTLS, CLONE_PIDFD, ...
```

So:

| "Concept" | What it actually is |
|---|---|
| A **process** | a task created with none of the sharing flags |
| A **thread** | a task created with `CLONE_VM\|CLONE_FILES\|CLONE_FS\|CLONE_SIGHAND\|CLONE_THREAD` |
| A **container** | a task created with the `CLONE_NEW*` namespace flags |
| A **kernel thread** | a task with `mm == NULL`, never returning to user mode |

This is **orthogonality as a design principle**, and it is the reason Linux got containers
essentially for free: namespaces were just eight more bits in an existing mechanism. Compare
Windows, where processes and threads are genuinely different kernel objects and container
support required a separate subsystem.

The cost is conceptual: *nothing in the kernel is "a process"*. `getpid()` returns the
**TGID** (thread group id); `gettid()` returns the actual `pid`. `struct task_struct` is
per-thread. Every time you read kernel code, translate "process" into "task, possibly sharing
things with other tasks."

```c
current->pid        /* the THREAD id   (what gettid() returns) */
current->tgid       /* the PROCESS id  (what getpid()  returns) */
current->group_leader /* the task whose pid == tgid */
```

### T.2 `fork()`: a historical accident that we now live with

`fork()` is unusual: it creates a new process by *duplicating* the caller, returning twice.
Its history (Ritchie & Thompson) is that it was easy to implement on a PDP-7 with swapping —
copy the image out, keep running. It was never designed.

The modern critique is worth knowing, because it is the reason the API landscape looks the
way it does. Baumann, Appavoo, Krieger & Roscoe, **"A fork() in the road" (HotOS 2019)**,
argue `fork()`:

- **Is not thread-safe** and cannot be made so. Only the calling thread survives; any lock
  held by another thread at fork time is *permanently locked* in the child. This is why
  POSIX restricts a forked child of a multithreaded program to async-signal-safe functions,
  and why `pthread_atfork()` exists and doesn't really work.
- **Is not composable.** It duplicates *everything*, including state belonging to libraries
  that never consented (buffered FILE contents get flushed twice — the classic
  `printf` before `fork` bug).
- **Is slow and getting slower.** Copying page tables is O(address space size). It defeats
  single-address-space optimizations, complicates huge pages, and is hostile to
  copy-on-write at scale.
- **Is insecure.** The child inherits everything by default — an opt-out rather than opt-in
  model. `CLOEXEC` had to be retrofitted onto every fd-creating syscall precisely because
  the default was wrong.

The modern answers, all present in Linux:

| API | Idea |
|---|---|
| `posix_spawn()` | declarative "create a process running X with this fd setup" — no duplication |
| `vfork()` | child borrows the parent's mm and blocks it until `execve`; avoids page-table copy |
| `clone3()` | extensible struct-based interface (`struct clone_args`), room to grow |
| `pidfd` | a *file descriptor* referring to a process — race-free signalling and waiting |
| `io_uring` spawn | batched, asynchronous process creation |

**Why `fork()` survives anyway:** the fork/exec split is genuinely powerful. Between `fork()`
and `execve()` the child can run arbitrary code to set up its environment — redirect fds,
change uid, join namespaces, set rlimits — using the *ordinary* API rather than a fixed set
of spawn attributes. That expressiveness is what makes shells, containers, and service
managers possible. `posix_spawn` has to encode all of that as a "file actions" DSL.

**The interview-grade summary:** `fork()` trades a clean, fast, thread-safe API for
*arbitrary programmability of the child's pre-exec state*, and that trade was worth it in
1975 and is marginal today.

### T.3 Copy-on-write: making duplication lazy

Naive `fork()` copies the entire address space. COW makes it lazy, and the theory is
elegant:

1. Both parent and child page tables point at the **same physical pages**.
2. All private, writable PTEs are marked **read-only** in both.
3. The `folio`'s mapcount/refcount records that two mappings exist.
4. A write faults; `do_wp_page()` checks: if this page has exactly one reference left, just
   make it writable again (**no copy**); otherwise allocate a new page, copy, and remap.

So the cost of `fork()` drops from O(RSS) to **O(page table size)**, and pages that are
never written are never copied. For the dominant `fork()`-then-`exec()` pattern, almost
nothing is copied at all.

Important subtleties that generate real bugs and real CVEs:

- **The page-table copy is still O(n)** in the number of mapped pages. Forking a process with
  a 100 GB mapping is slow even with COW. This is why `vfork()`/`posix_spawn` exist and why
  JVM/Redis-style "fork to snapshot" designs hurt at scale.
- **Deciding "is this page shared?"** is genuinely hard. The `mapcount`/`refcount`
  distinction (a page can have extra refcounts from GUP, the page cache, or a pinning
  driver) has caused a long series of bugs. **Dirty COW (CVE-2016-5195)** was a race in
  exactly this logic that gave arbitrary write access to read-only files. The modern fix
  (`FOLL_PIN`, `folio_maybe_dma_pinned()`, and the 6.x `mapcount` rework) is still evolving.
- **`MADV_WIPEONFORK`, `MADV_DONTFORK`** let a region opt out of inheritance — needed for
  RDMA/DMA-pinned buffers, where a COW copy would silently diverge from what hardware writes.
- COW interacts with **huge pages** (a 2 MiB COW fault copies 2 MiB), **memory cgroups**
  (who is charged?), and **NUMA** (where is the copy allocated?).

### T.4 `execve()`: the only way to change what a task runs

`execve()` replaces the address space of the *current* task, keeping its `task_struct`:

```
execve()
  → do_execve() → bprm_execve()
      alloc_bprm()            allocate struct linux_binprm, open the file
      prepare_binprm()        read the first 256 bytes, check permissions
      security_bprm_creds_*   LSM hooks; setuid/setgid/file-capability evaluation
      search_binary_handler() ★ ask each registered handler "is this yours?"
          → load_elf_binary() / load_script() / binfmt_misc / ...
              flush_old_exec()        ★ POINT OF NO RETURN: old mm destroyed
              setup_new_exec()
              set up new mm, map segments, map the interpreter (ld.so), the stack, vDSO
              create_elf_tables()     argv/envp/auxv onto the new stack
              START_THREAD()          set the new IP/SP
      (on success, never "returns" to the old program)
```

Four things to internalize:

1. **`search_binary_handler()` is a plugin dispatch.** `binfmt_elf`, `binfmt_script` (`#!`),
   `binfmt_misc` (user-registered magic → interpreter, which is how `qemu-user`, Java, Wine
   and Mono get transparent execution), and `binfmt_flat` (nommu) all register here. The
   `#!` handler recurses into `search_binary_handler()`, which is why interpreter chains
   work and why there is a recursion limit.
2. **There is a point of no return.** Before `flush_old_exec()`, failure returns an errno.
   After it, the old address space is gone and failure must kill the task
   (`force_sigsegv`). Recognizing "point of no return" structure in a state machine is a
   generalizable code-reading skill.
3. **`execve` is where privilege changes.** setuid bits, file capabilities, LSM transitions,
   and `AT_SECURE` are all evaluated here. This makes `execve()` the single most
   security-sensitive syscall in the kernel. `no_new_privs` (and hence seccomp's safety
   guarantee) is enforced right here.
4. **What survives `execve`:** the pid/tgid, the parent, open fds without `CLOSE_ON_EXEC`,
   the cwd, the namespace set, the cgroup, resource limits, and (mostly) credentials.
   What dies: the address space, threads other than the caller, signal *handlers* (reset to
   default; *dispositions* of ignored signals survive), and `mmap`ed regions.

### T.5 `exit()`, zombies, and why reaping exists

```
exit_group() → do_group_exit() → do_exit()
    exit_signals()      mark PF_EXITING; tell other threads to die
    exit_mm()           drop the mm (last thread frees it)
    exit_files(), exit_fs(), exit_sem(), exit_shm()
    exit_task_namespaces()
    exit_notify()       ★ reparent children; notify the parent with SIGCHLD
    task->exit_state = EXIT_ZOMBIE
    do_task_dead() → __schedule()  — never returns
    ... later ...
    release_task()      when the parent wait()s → free the task_struct
```

**Why zombies are necessary, not a bug:** the exit status is a *message from the child to the
parent*, and the child cannot deliver it after it is gone. So the kernel keeps the corpse —
a `task_struct` with `exit_state == EXIT_ZOMBIE` and essentially nothing else — until the
parent collects it. This is a **rendezvous problem**: information must survive until both
parties have synchronized.

Consequences:

- A zombie consumes a PID and a `task_struct` (~2 KB), nothing more. No memory, no fds.
- A parent that never `wait()`s leaks PIDs until `kernel.pid_max` is exhausted — a real DoS.
- **Orphan reparenting:** if the parent dies first, children are reparented to the nearest
  ancestor marked as a **child subreaper** (`prctl(PR_SET_CHILD_SUBREAPER)`), or else to PID 1
  in that PID namespace. Subreapers are what let `systemd --user`, container runtimes, and
  session managers reliably supervise their descendants — before them, a double-fork
  daemonized process was unsupervisable.
- **PID 1 in a namespace is special**: it is immune to signals it hasn't installed handlers
  for, and if it dies, every task in that namespace is killed. This is exactly why a
  container's init process must reap and must handle signals — the "PID 1 problem" that
  produced `tini`/`dumb-init`.

**`pidfd` fixes the PID-reuse race.** A numeric PID can be recycled between the moment you
read it and the moment you signal it — a classic TOCTOU that has produced real exploits.
`pidfd_open()` / `clone(CLONE_PIDFD)` returns an *fd* that refers to a specific task
instance; `pidfd_send_signal()` cannot hit the wrong process, and `poll()` on it reports
exit. This is the same "handle instead of a name" pattern as file descriptors themselves
(Ch. 12 T.10, solution C).

### T.6 Signals: asynchronous notification with hard semantics

Signals are the oldest IPC mechanism in Unix and the most subtly specified. The model:

- **Generation** — a signal becomes pending on a task (`sigqueue`) or on a thread group.
- **Delivery** — at the next return-to-user-mode, `get_signal()` picks one and acts.
- **Blocking** — `sigprocmask` defers but does not discard.

Properties you must know:

| Property | Consequence |
|---|---|
| **Standard signals (1–31) are not queued** | five `SIGCHLD`s while blocked = **one** delivery. Never count signals. |
| **Real-time signals (32–64) are queued** | up to `RLIMIT_SIGPENDING`; carry a `siginfo` payload |
| **Delivery is at kernel→user transition only** | a task spinning in a kernel loop with no exit point never receives anything |
| **`SIGKILL`/`SIGSTOP` cannot be caught or blocked** | but `TASK_UNINTERRUPTIBLE` tasks still can't die — hence `D` state and `TASK_KILLABLE` |
| **Thread-group signals go to any thread** that hasn't blocked them | which is why `pthread_sigmask` matters |
| **Handlers run on the user stack** (or `sigaltstack`) | the kernel builds a signal frame + a return trampoline |

**Restartable system calls** are the deep part. A signal arriving during a blocking syscall
must do something with the in-flight call. Linux uses internal pseudo-errnos:

```c
-ERESTARTSYS         /* restart if SA_RESTART, else return -EINTR */
-ERESTARTNOINTR      /* always restart (e.g. during execve setup) */
-ERESTARTNOHAND      /* restart only if no handler was invoked */
-ERESTART_RESTARTBLOCK /* restart via restart_syscall(), with adjusted arguments */
```
These **never reach userspace** — `handle_signal()` translates them. The last one is how
`nanosleep()` resumes for the *remaining* time rather than starting over: the kernel stashes
the remaining interval in `current->restart_block`.

**The `TASK_UNINTERRUPTIBLE` / `D` state problem:** a task waiting on I/O in
`TASK_UNINTERRUPTIBLE` cannot be killed, which is why an NFS server going away used to leave
unkillable processes forever. `TASK_KILLABLE` (Matthew Wilcox, 2.6.25) is the fix: sleep
uninterruptibly for everything *except* fatal signals. `wait_event_killable()` should be your
default in new code; `wait_event_interruptible()` where the caller can handle `-EINTR`; plain
`wait_event()` only when you can prove the wait is bounded.

### T.7 Credentials: a refcounted, RCU-protected immutable object

```c
struct cred {
	atomic_long_t	usage;
	kuid_t		uid, euid, suid, fsuid;
	kgid_t		gid, egid, sgid, fsgid;
	kernel_cap_t	cap_inheritable, cap_permitted, cap_effective,
			cap_bset, cap_ambient;
	struct user_namespace *user_ns;
	struct group_info *group_info;
	void		*security;      /* LSM blob */
	struct rcu_head	rcu;
};
```

The design is worth studying as a model:

- **Immutable after commit.** You never modify a live `cred`. You `prepare_creds()` (which
  copies), modify the copy, then `commit_creds()` (which RCU-swaps it in). Readers therefore
  need no lock at all — `current_cred()` under `rcu_read_lock()`.
- **`kuid_t`/`kgid_t` are opaque, `__bitwise`-style types** (Ch. 08 T.2) representing
  *kernel-internal* ids. Conversion to/from a user-namespace-relative id must go through
  `from_kuid()`/`make_kuid()`. This nominal typing is what made user namespaces implementable
  without auditing every uid comparison in the kernel by hand — **a type system change that
  enabled a feature**.
- `fsuid`/`fsgid` exist for historical NFS-daemon reasons and are what filesystem permission
  checks actually use.
- Objects capture credentials at creation: `file->f_cred` records who opened it, so later
  operations are checked against the opener's rights, not the current task's. This is
  essential to prevent confused-deputy attacks across `SCM_RIGHTS` fd passing.

### T.8 Kernel threads

A kthread is a task with `mm == NULL` that never returns to user mode. It borrows the
previous task's page tables (a "lazy TLB" optimization — no `switch_mm` needed, since it only
touches kernel addresses).

```c
struct task_struct *t;

t = kthread_run(fn, data, "mydev/%d", id);     /* create + wake */
t = kthread_create(fn, data, "name");          /* create, then kthread_bind(), wake_up_process() */
kthread_stop(t);                               /* sets should_stop, wakes, WAITS for exit */

/* Inside fn: */
static int fn(void *data)
{
	while (!kthread_should_stop()) {
		wait_event_interruptible(wq, condition || kthread_should_stop());
		if (kthread_should_stop())
			break;
		do_work();
	}
	return 0;
}
```

Rules that trip people up:
- `kthread_stop()` **blocks** until the thread returns. The thread must actually poll
  `kthread_should_stop()`, and any wait must include it in the condition, or you hang.
- A kthread that has already exited must not be `kthread_stop()`ed — hold a reference
  (`get_task_struct`) or use `kthread_stop_put()`.
- Kthreads ignore all signals by default and are not in any user namespace.
- **Prefer a workqueue** (Ch. 18) unless you genuinely need a dedicated, long-lived,
  individually-schedulable thread with its own priority/affinity. Reviewers will ask.
- `kthread_worker` gives you a kthread with a work queue attached — the middle ground, used
  where you need a dedicated thread *and* work-item semantics.

---

## 1. Internals

### 1.1 `struct task_struct` — the anatomy

It is ~7 KB and ~250 fields. You do not memorize it; you learn its *regions*.

```c
struct task_struct {
	struct thread_info      thread_info;    /* arch: flags, preempt_count, cpu */
	unsigned int            __state;        /* TASK_RUNNING / INTERRUPTIBLE / ... */
	void                    *stack;         /* the 16 KiB kernel stack */
	refcount_t              usage;
	unsigned int            flags;          /* PF_KTHREAD, PF_EXITING, PF_MEMALLOC, ... */

	/* --- scheduling (Ch. 21) --- */
	int                     prio, static_prio, normal_prio;
	const struct sched_class *sched_class;
	struct sched_entity     se;
	struct sched_rt_entity  rt;
	struct sched_dl_entity  dl;
	struct sched_statistics stats;
	cpumask_t               cpus_mask;
	int                     on_cpu, on_rq, recent_used_cpu, wake_cpu;

	/* --- memory (Ch. 22) --- */
	struct mm_struct        *mm;            /* NULL for kernel threads */
	struct mm_struct        *active_mm;     /* borrowed mm for kthreads */

	/* --- identity & relationships --- */
	pid_t                   pid, tgid;
	struct task_struct __rcu *real_parent, *parent;
	struct list_head        children, sibling;
	struct task_struct      *group_leader;
	struct pid              *thread_pid;
	struct hlist_node       pid_links[PIDTYPE_MAX];
	struct list_head        thread_node;

	/* --- credentials (T.7) --- */
	const struct cred __rcu *real_cred, *cred;
	char                    comm[TASK_COMM_LEN];   /* 16 bytes */

	/* --- resources --- */
	struct fs_struct        *fs;
	struct files_struct     *files;
	struct nsproxy          *nsproxy;
	struct signal_struct    *signal;        /* per thread-GROUP */
	struct sighand_struct __rcu *sighand;   /* handlers, shareable */
	sigset_t                blocked, real_blocked;
	struct sigpending       pending;        /* per-thread pending */

	/* --- accounting & misc --- */
	u64                     utime, stime, gtime;
	struct sched_info       sched_info;
	int                     exit_state, exit_code, exit_signal;
	struct io_context       *io_context;
	struct css_set __rcu    *cgroups;
	struct seccomp          seccomp;
	unsigned long           rcu_read_lock_nesting;   /* Ch. 15 T.7 */
	...
};
```

```bash
pahole -C task_struct vmlinux | head -60
pahole -C task_struct vmlinux | tail -20     # total size + holes
grep -n 'struct task_struct {' -A40 include/linux/sched.h
```

Per-thread vs per-process is the distinction that matters:

| Per-**thread** (`task_struct`) | Per-**process** (`signal_struct`, shared by the group) |
|---|---|
| `pid`, stack, registers, `blocked`, `pending` | `tgid`, rlimits, `shared_pending`, itimers, process CPU clocks, `tty` |

### 1.2 `current`

```c
#define current get_current()
/* x86-64: per-CPU variable, one instruction */
DECLARE_PER_CPU(struct task_struct *, current_task);
static __always_inline struct task_struct *get_current(void)
{ return this_cpu_read_stable(current_task); }
/* arm64: read from SP_EL0 */
/* older/other arches: mask the stack pointer to find thread_info at the stack base */
```

### 1.3 The fork path

```
fork() / vfork() / clone() / clone3()
  → kernel_clone(struct kernel_clone_args *)
      copy_process()                                 ★ kernel/fork.c — READ THIS
          dup_task_struct()        alloc task_struct + kernel stack (+ stack cache)
          copy_creds()
          sched_fork()             set up sched_entity, priority, place on a runqueue later
          perf_event_fork(), audit, security hooks
          copy_files()             CLONE_FILES ? get : dup_fd()
          copy_fs(), copy_sighand(), copy_signal()
          copy_mm()                ★ CLONE_VM ? get : dup_mm() → dup_mmap() → COW setup
          copy_namespaces(), copy_io(), copy_thread() (arch: set child's regs, ret = 0)
          alloc_pid()
          attach to the pid hash, the parent's children list, the thread group
      wake_up_new_task()           place on a runqueue; CFS/EEVDF "place_entity"
      /* vfork: wait_for_vfork_done() */
```

The single most important line is `copy_thread()`, which is where the child's saved registers
are set so that it "returns" from `fork()` with 0. **The child never executes the fork code
at all** — it starts at `ret_from_fork`.

```bash
$EDITOR kernel/fork.c            # copy_process()
grep -n 'ret_from_fork' arch/x86/kernel/process_64.c arch/x86/entry/entry_64.S
grep -n 'copy_thread' arch/*/kernel/process*.c | head
```

### 1.4 Where to read

```
include/linux/sched.h            ★ struct task_struct
kernel/fork.c                    ★ copy_process, dup_mm, kernel_clone
kernel/exit.c                    ★ do_exit, exit_notify, release_task, reparenting
fs/exec.c                        ★ bprm_execve, search_binary_handler, flush_old_exec
fs/binfmt_elf.c                  load_elf_binary
fs/binfmt_misc.c                 the qemu-user / Java magic
kernel/signal.c                  ★ get_signal, do_signal, restart blocks
kernel/pid.c, kernel/pid_namespace.c
kernel/cred.c                    prepare_creds/commit_creds
kernel/kthread.c
kernel/nsproxy.c
mm/memory.c                      do_wp_page() — the COW fault
Documentation/admin-guide/mm/   and  Documentation/core-api/pid_namespace.rst
```

---

## 2. Practice

### Lab 20.1 — Build a "process" from `clone()` flags yourself

```c
/* clonelab.c — gcc -O2 -o clonelab clonelab.c */
#define _GNU_SOURCE
#include <sched.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include <sys/syscall.h>

static int shared_var = 0;
static char stack[1 << 20];

static int child(void *arg)
{
	shared_var = 42;
	printf("child : pid=%d tid=%ld tgid=%d shared_var=%d\n",
	       getpid(), syscall(SYS_gettid), getpid(), shared_var);
	return 0;
}

int main(int argc, char **argv)
{
	int flags = 0;

	if (argc > 1 && atoi(argv[1]))          /* 1 = thread-like, 0 = process-like */
		flags = CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD;

	printf("parent: pid=%d tid=%ld shared_var=%d\n",
	       getpid(), syscall(SYS_gettid), shared_var);

	pid_t p = clone(child, stack + sizeof(stack), flags | SIGCHLD, NULL);
	if (p < 0) { perror("clone"); return 1; }

	sleep(1);
	printf("parent after: shared_var=%d  (%s)\n", shared_var,
	       shared_var ? "SHARED address space" : "COPIED address space");
	if (!(flags & CLONE_THREAD))
		waitpid(p, NULL, 0);
	return 0;
}
```
```bash
./clonelab 0     # process-like: shared_var stays 0 in the parent
./clonelab 1     # thread-like:  shared_var becomes 42
```
Now add flags one at a time (`CLONE_FILES` only, `CLONE_FS` only, …) and observe what becomes
shared. **This lab is T.1, made tangible in ten minutes.**

### Lab 20.2 — Watch copy-on-write happen

```bash
# Allocate 1 GiB, touch it, then fork and observe RSS
cat > /tmp/cow.c <<'EOF'
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/wait.h>
#define SZ (1UL<<30)
int main(void) {
	char *p = malloc(SZ);
	memset(p, 1, SZ);
	printf("parent pid %d; press enter to fork\n", getpid()); getchar();
	pid_t c = fork();
	if (c == 0) {
		printf("child  pid %d; enter to WRITE half\n", getpid()); getchar();
		memset(p, 2, SZ/2);
		printf("child wrote; enter to exit\n"); getchar();
		_exit(0);
	}
	wait(NULL);
	return 0;
}
EOF
gcc -O2 -o /tmp/cow /tmp/cow.c && /tmp/cow
# In another terminal, at each step:
grep -E 'VmRSS|RssAnon|RssShmem' /proc/<pid>/status
grep -E '^(Rss|Pss|Shared|Private)' /proc/<pid>/smaps_rollup
```
**Observe:** right after `fork()`, both processes show ~1 GiB RSS but `Pss` (proportional
set size) is ~0.5 GiB each — the pages are shared. After the child writes half, the child's
`Private_Dirty` jumps by 512 MiB. Then:

```bash
sudo perf stat -e page-faults,minor-faults -p <child-pid> -- sleep 5
sudo bpftrace -e 'kprobe:do_wp_page { @[comm] = count(); }'
sudo bpftrace -e 'kprobe:copy_page_range { @ = hist(arg2); }'   # page table copy cost
```

### Lab 20.3 — Measure fork vs vfork vs posix_spawn (T.2)

```c
/* spawnbench.c */
#define _GNU_SOURCE
#include <spawn.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>
#include <sys/wait.h>
extern char **environ;

static double now(void) {
	struct timespec t; clock_gettime(CLOCK_MONOTONIC, &t);
	return t.tv_sec + t.tv_nsec / 1e9;
}

int main(int argc, char **argv) {
	size_t heap_mb = argc > 1 ? atoi(argv[1]) : 0;
	int    n       = argc > 2 ? atoi(argv[2]) : 200;
	char  *argv2[] = { "/bin/true", NULL };
	double t0;
	int i;

	if (heap_mb) { char *p = malloc(heap_mb<<20); memset(p, 1, heap_mb<<20); }

	t0 = now();
	for (i = 0; i < n; i++) { pid_t c = fork(); if (!c) execv(argv2[0], argv2); wait(NULL); }
	printf("fork+exec    (%zu MB heap): %7.1f us\n", heap_mb, (now()-t0)/n*1e6);

	t0 = now();
	for (i = 0; i < n; i++) { pid_t c = vfork(); if (!c) execv(argv2[0], argv2); wait(NULL); }
	printf("vfork+exec   (%zu MB heap): %7.1f us\n", heap_mb, (now()-t0)/n*1e6);

	t0 = now();
	for (i = 0; i < n; i++) { pid_t c; posix_spawn(&c, argv2[0], NULL, NULL, argv2, environ); wait(NULL); }
	printf("posix_spawn  (%zu MB heap): %7.1f us\n", heap_mb, (now()-t0)/n*1e6);
	return 0;
}
```
```bash
gcc -O2 -o /tmp/sb /tmp/spawnbench.c
for mb in 0 64 512 2048; do /tmp/sb $mb 200; done
```
**`fork()` cost grows linearly with heap size; `vfork`/`posix_spawn` stay flat.** That is
T.2's "O(page table size)" claim, measured. This is why Redis `BGSAVE` and JVM
`Runtime.exec()` hurt on large heaps.

### Lab 20.4 — Trace fork/exec/exit through the kernel

```bash
# Static view
sudo perf trace -e 'syscalls:sys_enter_clone*,syscalls:sys_enter_execve,syscalls:sys_enter_exit_group' -a -- sleep 5

# Tracepoint view with the actual work
sudo bpftrace -e '
tracepoint:sched:sched_process_fork { printf("fork  %s(%d) -> %d\n", comm, args->parent_pid, args->child_pid); }
tracepoint:sched:sched_process_exec { printf("exec  %d -> %s\n", args->pid, str(args->filename)); }
tracepoint:sched:sched_process_exit { printf("exit  %s(%d)\n", comm, args->pid); }'

# Where is the time going inside copy_process()?
sudo bpftrace -e '
kprobe:copy_process   { @s[tid] = nsecs; }
kretprobe:copy_process /@s[tid]/ { @us = hist((nsecs-@s[tid])/1000); delete(@s[tid]); }'

# Function graph of one fork
sudo trace-cmd record -p function_graph -g kernel_clone -- bash -c 'true'
trace-cmd report | head -80
```

### Lab 20.5 — Zombies, orphans, and subreapers (T.5)

```bash
# 1. Make a zombie
cat > /tmp/zombie.c <<'EOF'
#include <unistd.h>
#include <stdio.h>
int main(void){ if(fork()==0){ printf("child %d exiting\n", getpid()); _exit(3);} sleep(60); }
EOF
gcc -o /tmp/zombie /tmp/zombie.c && /tmp/zombie &
ps -eo pid,ppid,stat,comm | grep -E 'Z|defunct'
cat /proc/<zombie-pid>/status | grep -E 'State|Name'
ls /proc/<zombie-pid>/       # note: no fd/, no maps with content

# 2. Orphan + reparent
cat > /tmp/orphan.c <<'EOF'
#include <unistd.h>
#include <stdio.h>
int main(void){ if(fork()==0){ sleep(2); printf("child %d ppid now %d\n", getpid(), getppid()); sleep(30);} _exit(0); }
EOF
gcc -o /tmp/orphan /tmp/orphan.c && /tmp/orphan
ps -eo pid,ppid,comm | grep orphan     # ppid becomes 1, or your subreaper

# 3. Become a subreaper and watch reparenting land on YOU
cat > /tmp/reaper.c <<'EOF'
#define _GNU_SOURCE
#include <sys/prctl.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>
int main(void){
	prctl(PR_SET_CHILD_SUBREAPER, 1);
	printf("subreaper %d\n", getpid());
	if (fork()==0) { if (fork()==0) { sleep(3); printf("gchild ppid=%d\n", getppid()); _exit(0);} _exit(0); }
	int st; pid_t p;
	while ((p = wait(&st)) > 0) printf("reaped %d status %d\n", p, WEXITSTATUS(st));
}
EOF
gcc -o /tmp/reaper /tmp/reaper.c && /tmp/reaper

# 4. PID exhaustion (in a VM!)
cat /proc/sys/kernel/pid_max
sudo sysctl -w kernel.pid_max=4096
# then fork-bomb-lite in a cgroup with pids.max set:
sudo mkdir /sys/fs/cgroup/forktest && echo 50 | sudo tee /sys/fs/cgroup/forktest/pids.max
```

### Lab 20.6 — `pidfd`: kill the right process, race-free (T.5)

```c
#define _GNU_SOURCE
#include <sys/syscall.h>
#include <sys/wait.h>
#include <poll.h>
#include <signal.h>
#include <stdio.h>
#include <unistd.h>

int main(void)
{
	int pidfd = -1;
	struct clone_args args = {
		.flags       = CLONE_PIDFD,
		.pidfd       = (unsigned long)&pidfd,
		.exit_signal = SIGCHLD,
	};
	pid_t p = syscall(SYS_clone3, &args, sizeof(args));

	if (p == 0) { sleep(10); _exit(7); }

	printf("child pid=%d pidfd=%d\n", p, pidfd);

	/* poll() on a pidfd reports exit — no SIGCHLD handler needed */
	struct pollfd pfd = { .fd = pidfd, .events = POLLIN };
	if (poll(&pfd, 1, 2000) == 0) {
		printf("still alive after 2s; killing via pidfd\n");
		syscall(SYS_pidfd_send_signal, pidfd, SIGKILL, NULL, 0);
	}
	siginfo_t si;
	waitid(P_PIDFD, pidfd, &si, WEXITED);
	printf("exited: code=%d status=%d\n", si.si_code, si.si_status);
	return 0;
}
```
```bash
ls -l /proc/self/fd/          # the pidfd shows as anon_inode:[pidfd]
cat /proc/<pid>/fdinfo/<pidfd>   # shows Pid: and NSpid:
```
Explain in writing the exact TOCTOU that `pidfd` eliminates, and find a real CVE caused by
PID reuse.

### Lab 20.7 — Signal semantics: prove the non-queuing rule (T.6)

```c
#define _GNU_SOURCE
#include <signal.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>

static volatile sig_atomic_t std_count, rt_count;
static void h1(int s)  { std_count++; }
static void h2(int s)  { rt_count++;  }

int main(void)
{
	sigset_t block, old;
	struct sigaction sa = { .sa_handler = h1 };
	struct sigaction sb = { .sa_handler = h2 };
	int i;

	sigaction(SIGUSR1, &sa, NULL);
	sigaction(SIGRTMIN, &sb, NULL);

	sigemptyset(&block);
	sigaddset(&block, SIGUSR1);
	sigaddset(&block, SIGRTMIN);
	sigprocmask(SIG_BLOCK, &block, &old);

	for (i = 0; i < 10; i++) { raise(SIGUSR1); raise(SIGRTMIN); }
	printf("sent 10 of each while blocked\n");

	sigprocmask(SIG_SETMASK, &old, NULL);   /* unblock: deliver everything pending */
	usleep(100000);
	printf("standard SIGUSR1 delivered: %d  (expect 1 — NOT queued)\n", std_count);
	printf("realtime SIGRTMIN delivered: %d  (expect 10 — queued)\n",  rt_count);
	return 0;
}
```
```bash
cat /proc/self/status | grep -E 'SigQ|SigPnd|SigBlk|SigIgn|SigCgt'
ulimit -i                                     # RLIMIT_SIGPENDING
sudo bpftrace -e 'tracepoint:signal:signal_generate { @[args->sig, comm] = count(); }'
sudo bpftrace -e 'tracepoint:signal:signal_deliver  { @[args->sig] = count(); }'
```

### Lab 20.8 — Restartable syscalls and `D` state (T.6)

```bash
# (a) SA_RESTART behaviour
cat > /tmp/restart.c <<'EOF'
#define _GNU_SOURCE
#include <signal.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <unistd.h>
static void h(int s) { write(2, "sig\n", 4); }
int main(int argc, char **argv) {
	struct sigaction sa = { .sa_handler = h };
	if (argc > 1) sa.sa_flags = SA_RESTART;
	sigaction(SIGALRM, &sa, NULL);
	alarm(1);
	char buf[16];
	ssize_t n = read(0, buf, sizeof(buf));
	printf("read -> %zd errno=%d (%s)\n", n, errno, strerror(errno));
}
EOF
gcc -o /tmp/restart /tmp/restart.c
/tmp/restart      # no SA_RESTART -> read returns -1 EINTR
/tmp/restart 1    # SA_RESTART    -> read is restarted transparently

# (b) nanosleep's restart_block
strace -e trace=clock_nanosleep,restart_syscall sleep 5 &
sleep 1; kill -ALRM %1

# (c) Find a D-state task and see where it's stuck
ps -eo pid,stat,wchan:30,comm | awk '$2 ~ /D/'
sudo cat /proc/<pid>/stack
sudo cat /proc/<pid>/status | grep -E 'State|SigPnd'
# Which wait primitive was used? killable or not?
git grep -n 'wait_event_killable\|wait_event_interruptible\|TASK_KILLABLE' fs/ | head
```

### Lab 20.9 — Explore `/proc/<pid>/` completely

```bash
P=$$
for f in status stat statm cmdline environ cwd exe root maps smaps_rollup \
         limits sched schedstat wchan stack syscall comm oom_score oom_score_adj \
         cgroup ns/pid ns/mnt ns/net personality io mountinfo timerslack_ns; do
  echo "=== $f"; sudo head -6 /proc/$P/$f 2>/dev/null
done
ls -l /proc/$P/fd/ /proc/$P/task/
sudo cat /proc/$P/task/*/stat | awk '{print $1, $2, $3}'
```
For each file, find the `seq_file` show function that produces it:
```bash
grep -rn '"status"\|"maps"\|"stack"\|"limits"' fs/proc/base.c | head -20
$EDITOR fs/proc/array.c          # proc_pid_status()
```

### Lab 20.10 — A kernel thread done right (T.8)

```c
// SPDX-License-Identifier: GPL-2.0
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/kthread.h>
#include <linux/wait.h>
#include <linux/sched.h>
#include <linux/delay.h>

static struct task_struct *worker;
static DECLARE_WAIT_QUEUE_HEAD(wq);
static bool have_work;
static DEFINE_SPINLOCK(lock);

static int worker_fn(void *data)
{
	pr_info("kthread start: pid=%d comm=%s mm=%p (NULL => kthread)\n",
		current->pid, current->comm, current->mm);

	while (!kthread_should_stop()) {
		wait_event_interruptible(wq, have_work || kthread_should_stop());
		if (kthread_should_stop())
			break;                   /* ★ check AFTER waking, always */

		spin_lock(&lock);
		have_work = false;
		spin_unlock(&lock);

		pr_info("doing work on cpu%d\n", smp_processor_id());
		msleep(100);                     /* legal: task context */
	}
	pr_info("kthread exiting cleanly\n");
	return 0;
}

void submit_work(void)
{
	spin_lock(&lock);
	have_work = true;
	spin_unlock(&lock);
	wake_up_interruptible(&wq);
}

static int __init kt_init(void)
{
	worker = kthread_create(worker_fn, NULL, "mydev-worker");
	if (IS_ERR(worker))
		return PTR_ERR(worker);

	kthread_bind(worker, 1);                 /* pin to CPU 1 */
	sched_set_normal(worker, 0);             /* or sched_set_fifo() for RT */
	wake_up_process(worker);

	submit_work();
	return 0;
}

static void __exit kt_exit(void)
{
	kthread_stop(worker);      /* ★ blocks until worker_fn returns */
}
module_init(kt_init); module_exit(kt_exit);
MODULE_LICENSE("GPL");
```
```bash
sudo insmod kt.ko
ps -eLo pid,tid,psr,pri,rtprio,comm | grep mydev-worker
sudo cat /proc/$(pgrep mydev-worker)/status | grep -E 'State|Cpus_allowed_list'
sudo rmmod kt
```
Now **break it deliberately**: remove `kthread_should_stop()` from the `wait_event`
condition and `rmmod` — it hangs forever. That hang is the #1 kthread bug.

### Lab 20.11 — `binfmt_misc`: run a foreign binary transparently (T.4)

```bash
sudo mount -t binfmt_misc none /proc/sys/fs/binfmt_misc 2>/dev/null
ls /proc/sys/fs/binfmt_misc/

# Register a handler for aarch64 ELF -> qemu-aarch64-static
sudo apt install -y qemu-user-static binfmt-support
cat /proc/sys/fs/binfmt_misc/qemu-aarch64

# Now an arm64 binary just... runs:
aarch64-linux-gnu-gcc -static -o /tmp/hello-arm64 hello.c
/tmp/hello-arm64

# Register your own:
echo ':mytag:M::\x7fMAGIC::/usr/local/bin/myinterp:' | sudo tee /proc/sys/fs/binfmt_misc/register
$EDITOR Documentation/admin-guide/binfmt-misc.rst
$EDITOR fs/binfmt_misc.c
```
This is the mechanism that makes cross-architecture Docker images and Yocto's `qemu-user`
SDK work (Part 7).

---

## 3. Mastery drills

1. **Read `copy_process()` end to end** (`kernel/fork.c`, ~700 lines). Produce a numbered
   list of every resource it handles and whether `CLONE_*` can share it. Cross-check against
   T.1's eight-resource list — did you find any it omits?

2. **Read the "fork() in the road" paper** (HotOS 2019). Write 500 words: which criticisms
   are fundamental, which are fixable, and what Linux has actually done about each. Then
   argue the *opposing* position using the fork/exec expressiveness argument from T.2.

3. **COW correctness.** Read `do_wp_page()` in `mm/memory.c`. Explain the exact condition
   under which it can *reuse* the page without copying. Then read the Dirty COW fix
   (`git log --oneline --grep='dirty COW'`) and explain the race.

4. **Trace an `execve` to the point of no return.** Find `flush_old_exec()` in `fs/exec.c`.
   Enumerate everything that can still fail before it, and what the kernel does about a
   failure after it. Why can't it recover?

5. **Signals in kernel code.** Find five uses of `wait_event_interruptible()` in `drivers/`.
   For each, determine whether `-ERESTARTSYS` propagates correctly to userspace and whether
   `wait_event_killable()` would be better. (Several real fixes exist:
   `git log --grep='killable' --oneline`.)

6. **Thread group signals.** Write a multithreaded program and determine, experimentally,
   which thread receives a process-directed `SIGTERM`. Then read `complete_signal()` in
   `kernel/signal.c` and explain the selection algorithm.

7. **Credentials.** Read `kernel/cred.c`. Explain why `prepare_creds()`/`commit_creds()` is
   used instead of modifying in place, and what `override_creds()`/`revert_creds()` are for
   (hint: `nfsd`, `io_uring`, and overlayfs all use them — and all three have had CVEs here).

8. **`kuid_t` and user namespaces.** Explain why making `kuid_t` a distinct type was the key
   enabler for user namespaces. Find a function that converts between `kuid_t` and `uid_t`
   and explain the mapping.

9. **PID namespaces.** Run a process in a new PID namespace (`unshare -Upf --mount-proc`).
   Inspect `/proc/<pid>/status` from *outside* and find `NSpid`. Then read
   `kernel/pid.c` and explain how one task has several `pid` numbers simultaneously.

10. **The PID 1 problem.** Explain why a container's PID 1 must reap children and handle
    signals. Demonstrate the failure mode with `docker run` (or `unshare`) using a shell as
    PID 1 that ignores `SIGTERM`. Then explain what `tini` does.

11. **Task lifetime.** `struct task_struct` is refcounted (`usage`) *and* RCU-protected
    (`->rcu`). Explain why both are needed, and find where `get_task_struct()` is required
    versus where `rcu_read_lock()` suffices. (This is Ch. 12's composition claim, in the
    single most important struct in the kernel.)

12. **Design.** You must let userspace wait for a specific task's exit *and* be notified if
    that task is replaced by a new one with the same PID. Design the interface. Compare with
    `pidfd`. What does `pidfd` guarantee that a PID cannot?

13. **kthread vs workqueue.** For each, say which you would use and why: (a) a driver polling
    a sensor every 10 ms; (b) an RT-priority audio mixing loop; (c) draining a hardware FIFO
    after an IRQ; (d) a per-CPU housekeeping task; (e) a once-per-boot firmware upload.

---

## 4. Further reading

**Kernel documentation:**
- `Documentation/admin-guide/pid_namespaces.rst`, `Documentation/core-api/pid_namespace.rst`
- `Documentation/admin-guide/binfmt-misc.rst`
- `Documentation/scheduler/` (for the `sched_entity` fields — Ch. 21)
- `Documentation/security/credentials.rst` ★ — the definitive text on T.7
- `Documentation/filesystems/proc.rst` — every `/proc/<pid>/` file, documented
- `Documentation/userspace-api/` — `no_new_privs.rst`, `seccomp_filter.rst`

**Source to read in order:**
1. `include/linux/sched.h` — `struct task_struct` (skim, then revisit)
2. `kernel/fork.c` — `copy_process()`, `dup_mm()`
3. `fs/exec.c` — `bprm_execve()`, `search_binary_handler()`
4. `kernel/exit.c` — `do_exit()`, `exit_notify()`, `release_task()`
5. `kernel/signal.c` — `get_signal()`, `complete_signal()`, restart blocks
6. `kernel/kthread.c`, `kernel/cred.c`, `kernel/pid.c`

**man pages (read these properly, they are excellent):**
- `clone(2)` ★★ — the flag table *is* T.1
- `fork(2)`, `vfork(2)`, `execve(2)`, `wait(2)`, `waitid(2)`
- `signal(7)` ★★, `signal-safety(7)`, `sigaction(2)`
- `credentials(7)`, `capabilities(7)`, `user_namespaces(7)`, `pid_namespaces(7)`
- `pidfd_open(2)`, `pidfd_send_signal(2)`, `clone3(2)`
- `prctl(2)` — `PR_SET_CHILD_SUBREAPER`, `PR_SET_NO_NEW_PRIVS`, `PR_SET_PDEATHSIG`

**Papers & books:**
- Baumann, Appavoo, Krieger, Roscoe, **"A fork() in the road" (HotOS 2019)** — read it
- Ritchie & Thompson, "The UNIX Time-Sharing System" (CACM 1974) — the origin
- Love, *Linux Kernel Development*, Ch. 3 (Process Management) and Ch. 10 (Signals... in 2e)
- Kerrisk, *The Linux Programming Interface* — **Chapters 24–28 (process creation), 20–22
  (signals), 33–35 (threads)**. This is the definitive userspace-side reference and the best
  complement to this chapter.
- Bovet & Cesati, *Understanding the Linux Kernel*, Ch. 3 and Ch. 11

**LWN:**
- "Toward race-free process signaling" / "The pidfd API"
- "Deprecating a.out" , "binfmt_misc and the F flag"
- "TASK_KILLABLE" (Wilcox, 2008)
- "Child subreapers", "The trouble with PID 1"
- "Dirty COW and the page-table walk"

→ Next: [21-scheduler.md](21-scheduler.md)

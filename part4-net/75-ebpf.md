# Chapter 75 — eBPF: the verifier, maps, program types, and CO-RE

> **Goal:** Understand the mechanism that lets untrusted code run safely in the kernel, and why that is a genuinely new capability rather than a convenience. Understand the instruction set and its deliberate restrictions, the verifier as an abstract interpreter proving safety properties and the specific analyses it performs, why bounded loops were hard and how they became possible, maps as the only durable state and how each type trades off, the program-type system as a capability model, helpers and kfuncs as the kernel's exported API, CO-RE as the solution to the kernel-version portability problem, and the security model including the debates about unprivileged access. By the end you can read `kernel/bpf/verifier.c`, write non-trivial programs, and reason about why the verifier rejected yours.

---

## Theory & First Principles

### T.0 — Start here: run your code in my kernel, and prove it is safe

```bash
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_openat {
    printf("%s -> %s\n", comm, str(args->filename)); }'
```

That one line compiles a program, loads it **into the running kernel**, attaches it to a
tracepoint, and streams results out — on a production machine, with no reboot, no module, and
no possibility of crashing the box. **Think about how outrageous that is.** Every other way
to run code in the kernel (Ch. 05) means a module that can dereference any pointer and panic
the system.

**The problem eBPF solves is a very old one**, and you should recognize it from Ch. 00 §T.1:

> The kernel must implement *mechanism*; but users keep needing *policy* the kernel authors
> did not anticipate. Every such need historically became either a new syscall, a new sysctl,
> a new module, or a rejected patch.

The three classical answers and their failure:

| Answer | Problem |
|---|---|
| Add a syscall / sysctl for each need | the interface grows forever; you must guess needs in advance (Ch. 24 §T.0) |
| Let users load kernel modules | unbounded power; one bug panics the machine; no safety argument possible |
| Do it in userspace | you cannot see kernel state, and the context switches cost more than the work |

**eBPF's answer is a fourth option: allow arbitrary user code, but only code you can *prove*
is safe before running it.**

```
   C source -> clang -> BPF bytecode -> bpf(BPF_PROG_LOAD)
                                            |
                                    +-------v--------+
                                    |  THE VERIFIER  |
                                    +----------------+
   It symbolically executes EVERY reachable path and proves:
     * the program terminates            (bounded loops only)
     * every memory access is in bounds   (tracked ranges per register)
     * no uninitialized reads             (each register's state is tracked)
     * only permitted helpers are called  (per program type)
     * pointers are not leaked to userspace
   Fail any check -> LOAD IS REJECTED. Nothing runs.
                                            |
                                    JIT to native code
                                            |
                                    attach to a hook
```

**Three consequences worth internalizing, because they explain everything about writing eBPF:**

1. **Safety is established statically, so the runtime cost is near zero.** After JIT, your
   program is native machine code with no runtime checks. **You pay at load time, not at run
   time** — a trade that is only available because the verifier does the whole proof up front.
2. **The verifier rejects safe programs.** It is *sound* (never accepts an unsafe program) but
   not *complete* (it rejects many safe ones). Most of the pain of writing eBPF is
   restructuring correct code until the verifier can follow the argument. **This is the price
   of a decidable safety proof, and it is a permanent price, not a maturity issue.**
3. **The program is not the interesting part — the hooks and maps are.** A program that
   cannot be attached anywhere is useless; maps are what let programs keep state and
   communicate with userspace.

**The map is the second half of the architecture**, and it turns eBPF from a tracing tool into
a programming model:

```
  HASH  ARRAY  PERCPU_*  LRU_HASH  RINGBUF  STACK_TRACE  SOCKMAP  PROG_ARRAY
     ^                                   ^                    ^
     |                                   |                    |
  shared with userspace           efficient event         tail calls:
  via fds; survive program        streaming to            one program
  restart if PINNED               userspace               jumps to another
```

**And the honest architectural question**, which is §T.9's argument and worth holding in mind:
eBPF makes the kernel programmable, which is wonderful — and it has also become a second,
largely undocumented, *de facto* stable ABI (Hyrum's Law, Ch. 24 §T.0). Programs attached to
tracepoints and to internal functions via `kprobe` observe kernel internals that were never
meant to be stable. The kernel's "we do not break userspace" promise is now in tension with
its freedom to refactor. BTF and CO-RE mitigate this; they do not resolve it.

```bash
sudo bpftool prog list && sudo bpftool map list
sudo bpftool prog dump xlated id 42      # the verified bytecode
sudo bpftool prog dump jited  id 42      # the native code actually running
sudo bpftool feature probe | head -30
ls /sys/kernel/btf/vmlinux               # the type information CO-RE relies on
```

---

### T.1 The problem eBPF solves

Extending the kernel has historically meant one of three things:

| Approach | Safety | Performance | Deployment |
|---|---|---|---|
| **Kernel module** | **none** — full memory access, can panic | native | reboot on bug; signing; version-tied |
| **Userspace + syscalls** | full | **poor** — context switches, copies | easy |
| **Patch the kernel** | none | native | recompile, reboot, maintain a fork |

eBPF is a fourth option:

> **Run user-supplied code in the kernel, at native speed, with a machine-checked proof that it cannot crash, cannot leak, cannot loop forever, and cannot access memory it should not.**

The proof is the whole thing. Without it, eBPF is a slow interpreter; with it, it is a safe extension mechanism. And it is a *static* proof — checked once at load time, so the program runs with no runtime checks beyond what the verifier could not eliminate.

This has turned out to matter more than networking. The original use was packet filtering (BPF, 1992); the current uses span tracing, security, scheduling, storage, and infrastructure. The reason is the property above: **you can ship kernel functionality to a running production machine without a reboot and without the risk a module carries.**

The trade is that the language is restricted in ways that make the proof possible, and those restrictions are the subject of §T.3–T.4.

### T.2 The instruction set

eBPF is a 64-bit RISC ISA with 11 registers, designed to map cleanly onto x86-64 and arm64 so the JIT is straightforward:

```c
struct bpf_insn {
	__u8	code;		/* opcode */
	__u8	dst_reg:4;
	__u8	src_reg:4;
	__s16	off;		/* signed offset */
	__s32	imm;		/* signed immediate */
};
```

Eight bytes per instruction, fixed width. The registers:

| Register | Role |
|---|---|
| `r0` | return value; also the helper return |
| `r1`–`r5` | arguments to helpers; `r1` is the context on entry |
| `r6`–`r9` | callee-saved |
| `r10` | **read-only frame pointer** |

`r10` being read-only is a deliberate restriction: the program cannot construct arbitrary stack pointers, which eliminates a whole class of verification difficulty.

The instruction classes:

```c
#define BPF_LD		0x00	/* load */
#define BPF_LDX		0x01	/* load from register-relative */
#define BPF_ST		0x02	/* store immediate */
#define BPF_STX		0x03	/* store from register */
#define BPF_ALU		0x04	/* 32-bit arithmetic */
#define BPF_JMP		0x05	/* 64-bit jumps */
#define BPF_JMP32	0x06	/* 32-bit jumps */
#define BPF_ALU64	0x07	/* 64-bit arithmetic */
```

What is deliberately absent, and why:

| Absent | Reason |
|---|---|
| Unbounded loops | termination must be provable |
| Arbitrary indirect calls | the call graph must be known |
| Floating point | no FPU state to save |
| Raw pointer arithmetic on arbitrary values | pointer provenance must be tracked |
| Signals, sleeping (mostly) | runs in atomic contexts |
| More than 512 bytes of stack | bounded resource use |
| Recursion | stack depth must be bounded |

The JIT compiles to native code (`bpf_int_jit_compile` per architecture). Interpreted execution still exists as a fallback but `net.core.bpf_jit_enable=1` is the default and interpretation can be compiled out entirely (`CONFIG_BPF_JIT_ALWAYS_ON`) — which is what a hardened kernel does, because the interpreter was a Spectre gadget.

Two extensions worth knowing:

**BPF-to-BPF calls** (4.16+) allow functions within a program, so code need not be fully inlined. The verifier analyses each function separately and tracks the combined stack depth.

**Tail calls** (`bpf_tail_call`) jump to another program from a `PROG_ARRAY` map, without returning. This gives a form of dynamic dispatch and chained processing, bounded at 33 levels. Combined with BPF-to-BPF calls it has restrictions, because the stack accounting becomes hard.

### T.3 The verifier as an abstract interpreter

`kernel/bpf/verifier.c` is ~20,000 lines and is the most security-critical code in this chapter. What it does:

**Step 1: build the control-flow graph and check it is a DAG (plus bounded loops).** `check_cfg()` does a depth-first search detecting back edges. An unbounded back edge is rejected outright.

**Step 2: simulate every reachable path, tracking the state of every register and stack slot.**

```c
struct bpf_reg_state {
	enum bpf_reg_type type;
	s32 off;
	union {
		int range;                     /* PTR_TO_PACKET */
		struct bpf_map *map_ptr;       /* PTR_TO_MAP_VALUE */
		struct btf *btf;               /* PTR_TO_BTF_ID */
		u32 mem_size;                  /* PTR_TO_MEM */
		...
	};
	u32 id;
	u32 ref_obj_id;
	/* THE value tracking: */
	struct tnum var_off;               /* known bits */
	s64 smin_value, smax_value;        /* signed range */
	u64 umin_value, umax_value;        /* unsigned range */
	s32 s32_min_value, s32_max_value;  /* 32-bit signed range */
	u32 u32_min_value, u32_max_value;
	struct bpf_reg_state *parent;
	u32 frameno;
	s32 subreg_def;
	enum bpf_reg_liveness live;
	bool precise;
};
```

Two representations are tracked simultaneously:

**Ranges** (`smin/smax/umin/umax`) — an interval. After `if (x < 100)`, the true branch knows `umax_value = 99`.

**`tnum`** (tristate number) — per-bit knowledge:

```c
struct tnum {
	u64 value;   /* the known bits' values */
	u64 mask;    /* 1 = this bit is UNKNOWN */
};
```

So `tnum{value=0x10, mask=0x0F}` means "the high bits are 0x1, the low nibble is unknown" — i.e. a value in [0x10, 0x1F]. This captures alignment and masking facts that intervals cannot: after `x &= 0xFF`, the tnum knows the high 56 bits are zero exactly, where a range would only know `umax = 255`.

The two representations refine each other (`__update_reg_bounds`, `__reg_deduce_bounds`), and that interaction is where most of the verifier's precision comes from.

**Step 3: check every memory access against the tracked state.**

```c
enum bpf_reg_type {
	NOT_INIT = 0,
	SCALAR_VALUE,
	PTR_TO_CTX,
	CONST_PTR_TO_MAP,
	PTR_TO_MAP_VALUE,
	PTR_TO_MAP_KEY,
	PTR_TO_STACK,
	PTR_TO_PACKET_META,
	PTR_TO_PACKET,
	PTR_TO_PACKET_END,
	PTR_TO_FLOW_KEYS,
	PTR_TO_SOCKET,
	PTR_TO_SOCK_COMMON,
	PTR_TO_TCP_SOCK,
	PTR_TO_TP_BUFFER,
	PTR_TO_XDP_SOCK,
	PTR_TO_BTF_ID,
	PTR_TO_MEM,
	PTR_TO_BUF,
	PTR_TO_FUNC,
	CONST_PTR_TO_DYNPTR,
};
```

**Pointer types are not interchangeable.** A `PTR_TO_PACKET` can be dereferenced only after a bounds check against `PTR_TO_PACKET_END`; a `PTR_TO_MAP_VALUE` only within the map's value size; a `PTR_TO_BTF_ID` only through `bpf_probe_read_kernel` or with BTF-verified direct access. Arithmetic that would produce an untracked pointer is rejected.

**Step 4: prune equivalent states.** Without pruning, path explosion makes verification infeasible. `states_equal()` decides whether a newly-reached state is "at least as general" as a previously-verified one at the same instruction; if so, the path need not be re-explored.

**Step 5: enforce resource limits.**

```c
#define BPF_COMPLEXITY_LIMIT_INSNS	1000000   /* verified instructions */
#define BPF_COMPLEXITY_LIMIT_STATES	64
#define MAX_BPF_STACK			512
#define MAX_CALL_FRAMES			8
#define MAX_TAIL_CALL_CNT		33
```

The million-instruction limit is on *verified* instructions, not executed ones — a program with many branches can exhaust it while being short.

### T.4 Bounded loops, and why they were hard

Until 5.3, all loops had to be unrolled (`#pragma unroll`), which limited what could be written. Bounded loops required the verifier to prove termination.

The mechanism: when a back edge is found, the verifier checks whether the loop's state is *converging* — whether the induction variable's range is monotonically shrinking toward the exit condition. If each iteration's state is provably "closer" to termination, and the bound is finite, the loop terminates.

In practice this means a loop like:

```c
	for (i = 0; i < 100; i++) { ... }
```

verifies if `i` is a scalar whose range the verifier can track. Loops with data-dependent bounds often do not:

```c
	for (i = 0; i < n; i++) { ... }    /* if n is unbounded: REJECTED */
	if (n > 100) n = 100;
	for (i = 0; i < n; i++) { ... }    /* now OK */
```

Three escape hatches exist for cases the verifier cannot handle:

**`bpf_loop()`** (5.17) — a helper taking a callback:

```c
	static long body(u32 index, void *ctx) { ... return 0; }
	bpf_loop(nr_loops, body, &ctx, 0);
```

The kernel runs the loop; the verifier checks the callback once. This moves the termination proof from the verifier to the helper, which knows the bound.

**Iterators** (`bpf_iter_num_new`, and open-coded iterators generally, 6.4+) provide a verified iteration protocol.

**`bpf_for()`** — a macro over open-coded iterators, in `libbpf`, giving natural loop syntax that verifies.

The broader lesson: **the verifier's limitations are addressed by adding verified primitives rather than by weakening the verifier.** That is the right direction and it is why eBPF's capability has grown without its safety guarantees eroding.

### T.5 Maps: the only durable state

A BPF program's stack and registers vanish when it returns. Maps are the only way to keep state, to communicate with userspace, and to communicate between programs.

```c
struct bpf_map_ops {
	int (*map_alloc_check)(union bpf_attr *attr);
	struct bpf_map *(*map_alloc)(union bpf_attr *attr);
	void (*map_release)(struct bpf_map *map, struct file *map_file);
	void (*map_free)(struct bpf_map *map);
	int (*map_get_next_key)(struct bpf_map *map, void *key, void *next_key);
	void *(*map_lookup_elem)(struct bpf_map *map, void *key);
	long (*map_update_elem)(struct bpf_map *map, void *key, void *value, u64 flags);
	long (*map_delete_elem)(struct bpf_map *map, void *key);
	int (*map_gen_lookup)(struct bpf_map *map, struct bpf_insn *insn_buf);
	...
};
```

The important types, and what each is for:

| Type | Structure | Cost | Use |
|---|---|---|---|
| `ARRAY` | contiguous, index-keyed | **~5 ns**, inlined by the JIT | counters, config |
| `PERCPU_ARRAY` | one per CPU | **fastest**, no atomics needed | per-CPU counters |
| `HASH` | chained hash, preallocated | ~20–50 ns | flow tables, lookups |
| `PERCPU_HASH` | per-CPU hash | faster, more memory | per-CPU aggregation |
| `LRU_HASH` | hash with LRU eviction | ~30–60 ns | **bounded state under attack** |
| `LPM_TRIE` | longest-prefix trie | ~100+ ns | CIDR matching, routing |
| `RINGBUF` | MPSC ring | very fast | **events to userspace** |
| `PERF_EVENT_ARRAY` | per-CPU perf buffers | fast | events (the older way) |
| `PROG_ARRAY` | program fds | — | tail calls |
| `DEVMAP`, `CPUMAP`, `XSKMAP` | redirect targets | — | Ch. 74 |
| `SOCKMAP`, `SOCKHASH` | socket fds | — | socket redirection, sk_msg |
| `STACK_TRACE` | stack IDs | — | profiling |
| `QUEUE`, `STACK` | FIFO/LIFO | — | work queues |
| `ARRAY_OF_MAPS`, `HASH_OF_MAPS` | maps of maps | — | dynamic reconfiguration |
| `STRUCT_OPS` | a struct of function pointers | — | §T.7's pluggable subsystems |
| `BLOOM_FILTER` | probabilistic set | very fast | pre-filtering before an expensive lookup |
| `CGROUP_STORAGE`, `SK_STORAGE`, `TASK_STORAGE`, `INODE_STORAGE` | **attached to an object's lifetime** | fast | per-socket/task state |

Four things worth emphasising:

**(a) Per-CPU maps eliminate atomics.** A shared counter at 20 Mpps is a contended cache line. A per-CPU counter is a plain increment, summed in userspace. **This is the single most common eBPF performance mistake** — using a `HASH` where a `PERCPU_HASH` would do.

**(b) LRU maps bound attacker-controlled state.** A `HASH` keyed by source IP can be filled by an attacker; an `LRU_HASH` evicts. Ch. 74's DDoS lab used one for exactly this.

**(c) Local storage attaches to an object's lifetime.** `BPF_MAP_TYPE_SK_STORAGE` gives per-socket state that is freed when the socket is — no leak, no cleanup code, no scanning. Same for tasks, cgroups, and inodes. This is a much better pattern than a hash keyed by pointer, and it is under-used.

**(d) Ring buffer versus perf buffer.** The perf buffer is per-CPU, so events from different CPUs can be reordered and memory is allocated per CPU. The ring buffer (5.8+) is a single MPSC buffer with ordering preserved, a reservation API that avoids a copy, and better memory efficiency. **Use `RINGBUF` for new code.**

**Map lookup inlining** is worth knowing: `map_gen_lookup` lets a map type emit inline BPF instructions for a lookup rather than a helper call, which the JIT then compiles to a few native instructions. `ARRAY` does this, which is why it is ~5 ns rather than ~30 ns.

### T.6 Program types as a capability model

A program's **type** determines everything about what it may do:

| The type determines | Via |
|---|---|
| What the context (`r1`) points to | `bpf_verifier_ops->is_valid_access` |
| Which helpers are callable | `bpf_verifier_ops->get_func_proto` |
| What return values mean | per-type checking |
| Which attach points are valid | `bpf_prog_type` → attachment |
| Whether it may sleep | `BPF_F_SLEEPABLE` and the type |

```c
enum bpf_prog_type {
	BPF_PROG_TYPE_UNSPEC,
	BPF_PROG_TYPE_SOCKET_FILTER,
	BPF_PROG_TYPE_KPROBE,
	BPF_PROG_TYPE_SCHED_CLS,        /* tc */
	BPF_PROG_TYPE_SCHED_ACT,
	BPF_PROG_TYPE_TRACEPOINT,
	BPF_PROG_TYPE_XDP,
	BPF_PROG_TYPE_PERF_EVENT,
	BPF_PROG_TYPE_CGROUP_SKB,
	BPF_PROG_TYPE_CGROUP_SOCK,
	BPF_PROG_TYPE_LWT_IN, _OUT, _XMIT, _SEG6LOCAL,
	BPF_PROG_TYPE_SOCK_OPS,
	BPF_PROG_TYPE_SK_SKB,
	BPF_PROG_TYPE_CGROUP_DEVICE,
	BPF_PROG_TYPE_SK_MSG,
	BPF_PROG_TYPE_RAW_TRACEPOINT,
	BPF_PROG_TYPE_CGROUP_SOCK_ADDR,
	BPF_PROG_TYPE_LIRC_MODE2,
	BPF_PROG_TYPE_SK_REUSEPORT,
	BPF_PROG_TYPE_FLOW_DISSECTOR,
	BPF_PROG_TYPE_CGROUP_SYSCTL,
	BPF_PROG_TYPE_CGROUP_SOCKOPT,
	BPF_PROG_TYPE_TRACING,          /* fentry/fexit/fmod_ret/iter */
	BPF_PROG_TYPE_STRUCT_OPS,       /* T.7 */
	BPF_PROG_TYPE_EXT,
	BPF_PROG_TYPE_LSM,              /* T.9 */
	BPF_PROG_TYPE_SK_LOOKUP,
	BPF_PROG_TYPE_SYSCALL,
	BPF_PROG_TYPE_NETFILTER,
};
```

**This is a capability system.** An XDP program cannot call `bpf_get_current_pid_tgid` (there is no current task in a NAPI poll); a tracing program cannot call `bpf_redirect`; an LSM program can return an error that denies an operation while a tracepoint program cannot affect anything.

The most significant recent additions:

**`fentry`/`fexit`** (BPF trampolines, 5.5) attach to any kernel function with **near-zero overhead** — a direct jump into JITed code, not a breakpoint trap like kprobes. `fexit` additionally sees the return value *and* the arguments, which kretprobe cannot. For tracing, fentry/fexit should be preferred wherever available.

**`fmod_ret`** can *change* a function's return value, for a whitelisted set of functions — error injection, and the basis of some security tooling.

**`BPF_PROG_TYPE_LSM`** attaches to LSM hooks and can deny operations. This is a genuinely significant capability: a security policy, written in BPF, loaded at runtime, that the verifier has proven safe.

**`BPF_PROG_TYPE_ITER`** implements a `seq_file` in BPF — you can write a custom `/proc`-like view of kernel data structures (all tasks, all sockets, all map entries) without a module.

### T.7 `struct_ops`: BPF implementing kernel interfaces

`BPF_MAP_TYPE_STRUCT_OPS` lets a BPF program implement a kernel operations structure. The first user was TCP congestion control:

```c
SEC("struct_ops/bbr_init")
void BPF_PROG(bbr_init, struct sock *sk) { ... }

SEC("struct_ops/bbr_main")
void BPF_PROG(bbr_main, struct sock *sk, const struct rate_sample *rs) { ... }

SEC(".struct_ops")
struct tcp_congestion_ops bbr = {
	.init		= (void *)bbr_init,
	.cong_control	= (void *)bbr_main,
	.name		= "bpf_bbr",
};
```

Load it and it appears in `net.ipv4.tcp_available_congestion_control`. **A congestion-control algorithm, loaded at runtime, verified.** Ch. 72 §T.5's pluggable interface, made pluggable from userspace.

Current users:

| Subsystem | What |
|---|---|
| `tcp_congestion_ops` | congestion control algorithms |
| `sched_ext` (`sched_ext_ops`) ★★★ | **CPU schedulers** (6.12) |
| `hid_bpf_ops` | HID device fixups |
| `bpf_qdisc_ops` (in progress) | queueing disciplines |

`sched_ext` is the most ambitious: a complete CPU scheduler written in BPF, replacing CFS/EEVDF for selected tasks. It exists because scheduler policy is workload-specific, experimentation in-tree is slow, and the alternative (out-of-tree scheduler patches) is worse. Ch. 21 §T.7 introduced it; `struct_ops` is the mechanism.

The pattern generalises: **any kernel interface that is already a struct of function pointers is a candidate.** That is a large fraction of the kernel.

### T.8 BTF and CO-RE

The portability problem: a BPF program that reads `task->pid` needs the offset of `pid` within `struct task_struct`. That offset varies by kernel version and configuration. Compiling against one kernel's headers and running on another gives wrong data — silently.

The old solution (BCC) was to **compile on the target machine at runtime**, requiring kernel headers, clang, and LLVM on every production host. That is a 100 MB dependency, several seconds of startup, and a failure mode when headers are missing.

**BTF (BPF Type Format)** is a compact debug-info format describing every type in the kernel:

```sh
ls -lh /sys/kernel/btf/vmlinux          # ~5 MB, always present if CONFIG_DEBUG_INFO_BTF=y
bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```

**CO-RE (Compile Once, Run Everywhere)** uses it:

1. The program is compiled once, with `-g` and BTF, against a generated `vmlinux.h`.
2. Field accesses are compiled into **relocations** rather than fixed offsets — the compiler emits `__builtin_preserve_access_index()` markers.
3. At load time, `libbpf` reads the *running* kernel's BTF, finds the field by name, and patches the offset into the instruction stream.

```c
	/* Source */
	pid = task->pid;

	/* Compiled: a relocation, not an offset */
	/* At load: libbpf resolves "task_struct.pid" against the running kernel */
```

Additional CO-RE facilities:

```c
	if (bpf_core_field_exists(task->new_field)) { ... }
	sz = bpf_core_field_size(task->comm);
	if (bpf_core_type_exists(struct new_struct)) { ... }
	val = BPF_CORE_READ(task, mm, owner, pid);   /* a chain of reads */
```

`BPF_CORE_READ` is worth understanding: it expands into a chain of `bpf_probe_read_kernel` calls with CO-RE relocations at each step, so `task->mm->owner->pid` works safely across versions.

For kernels without BTF, **BTFHub** provides pre-generated BTF for older distribution kernels, which `libbpf` can load externally.

**The practical result:** a single compiled `.o` file runs on kernels from 5.x to 6.x across distributions, with no clang on the target, no headers, and millisecond startup. That is what made eBPF deployable at scale, and it is why `libbpf` + CO-RE replaced BCC for production tooling.

### T.9 Helpers, kfuncs, and the exported API

BPF programs cannot call arbitrary kernel functions. Two mechanisms expose functionality:

**Helpers** — a stable, numbered ABI:

```c
static void *(*bpf_map_lookup_elem)(void *map, const void *key) = (void *)1;
static long (*bpf_map_update_elem)(void *map, const void *key,
				   const void *value, __u64 flags) = (void *)2;
/* ... ~220 of them ... */
```

They are stable UAPI: once added, a helper's signature cannot change. That stability is valuable and it is also a burden — every helper is a permanent commitment.

**kfuncs** — kernel functions exported to BPF, **without stability guarantees**:

```c
__bpf_kfunc struct task_struct *bpf_task_acquire(struct task_struct *p)
{
	return get_task_struct(p);
}

BTF_KFUNCS_START(generic_btf_ids)
BTF_ID_FLAGS(func, bpf_task_acquire, KF_ACQUIRE | KF_RET_NULL)
BTF_ID_FLAGS(func, bpf_task_release, KF_RELEASE)
BTF_KFUNCS_END(generic_btf_ids)
```

The flags encode verifier semantics:

| Flag | Meaning |
|---|---|
| `KF_ACQUIRE` | returns a reference the program **must release** |
| `KF_RELEASE` | releases a reference |
| `KF_RET_NULL` | may return NULL; the program must check |
| `KF_TRUSTED_ARGS` | arguments must be verified-trusted pointers |
| `KF_SLEEPABLE` | may sleep; only in sleepable programs |
| `KF_DESTRUCTIVE` | may crash the machine; requires `CAP_SYS_BOOT` |
| `KF_RCU` | arguments are RCU-protected |

**`KF_ACQUIRE`/`KF_RELEASE` reference tracking is a real achievement**: the verifier proves that every acquired reference is released on every path, including error paths. That is a linear-type-system property, checked statically, in a kernel written in C.

The deliberate instability of kfuncs is the interesting design decision. Helpers became a maintenance burden precisely because they were stable; kfuncs let the kernel expose functionality without a permanent commitment, on the argument that BPF programs are recompiled and redeployed more readily than userspace applications. Whether that holds is debated, and it is essentially the Hyrum's Law question of Ch. 24 §T.1 applied to a new interface.

### T.10 The security model

BPF's attack surface is large: the verifier is complex, JITs are architecture-specific, and the maps are shared memory.

**Privilege:**

| Capability | Grants |
|---|---|
| `CAP_BPF` | basic map and program operations |
| `CAP_PERFMON` | tracing programs, `bpf_probe_read_kernel` |
| `CAP_NET_ADMIN` | networking programs |
| `CAP_SYS_ADMIN` | everything, including unsafe things |

Splitting `CAP_BPF` out of `CAP_SYS_ADMIN` (5.8) was meant to allow less-privileged BPF use. In practice most useful programs need `CAP_BPF` + `CAP_PERFMON` or `CAP_NET_ADMIN`, which together approach `CAP_SYS_ADMIN` in power.

**Unprivileged BPF is disabled by default:**

```sh
sysctl kernel.unprivileged_bpf_disabled     # 2 on most distributions
```

The reason is Spectre. The verifier proves *architectural* safety — that no instruction reads out of bounds. It cannot easily prove *microarchitectural* safety: speculative execution can read out of bounds and leave a cache trace, and an attacker-supplied program is an excellent speculation gadget.

The mitigations:

| Mitigation | What |
|---|---|
| Pointer value masking | the verifier inserts masks so speculation stays in bounds |
| `nospec` insertion | speculation barriers after conditional bounds checks |
| Constant blinding (`bpf_jit_harden`) | XOR immediates with a random value, so attacker-chosen constants do not become gadgets |
| JIT address randomisation | `bpf_jit_kallsyms` off by default for unprivileged |
| Disabling the interpreter | `CONFIG_BPF_JIT_ALWAYS_ON` |
| **Disabling unprivileged BPF** | the actual answer |

The honest position: **unprivileged BPF has not been made safe against speculative attacks, and the default is to disable it.** This is a significant limitation on eBPF's original ambition — the vision of any user loading a packet filter has not survived Spectre.

**Other hardening:**

```sh
kernel.bpf_stats_enabled = 0        # run-time statistics (small overhead)
net.core.bpf_jit_harden = 0         # 1 = harden for unprivileged, 2 = for all
net.core.bpf_jit_kallsyms = 1       # expose JIT symbols to kallsyms
net.core.bpf_jit_limit               # the JIT's memory budget
```

Signed BPF programs are an active area: the ability to require that loaded programs carry a signature, so a compromised userspace cannot load arbitrary BPF. This has been discussed for years and partial mechanisms exist (BPF token, LSM hooks on `bpf()`), but a complete solution is not upstream.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `kernel/bpf/verifier.c` ★★★ | **§T.3–T.4**; ~20,000 lines, the heart |
| `kernel/bpf/core.c` ★★★ | the interpreter, program lifecycle, JIT glue |
| `kernel/bpf/syscall.c` ★★★ | the `bpf()` syscall: every command |
| `kernel/bpf/helpers.c` ★★★ | the common helpers |
| `kernel/bpf/hashtab.c`, `arraymap.c`, `lpm_trie.c`, `ringbuf.c`, `bloom_filter.c` ★★★ | §T.5 |
| `kernel/bpf/local_storage.c`, `bpf_task_storage.c`, `bpf_inode_storage.c` | §T.5(c) |
| `kernel/bpf/btf.c` ★★★ | §T.8's BTF handling |
| `kernel/bpf/trampoline.c` ★★★ | fentry/fexit; §T.6 |
| `kernel/bpf/bpf_struct_ops.c` ★★★ | §T.7 |
| `kernel/bpf/bpf_iter.c` | BPF iterators |
| `kernel/bpf/tnum.c` ★★★ | §T.3's tristate numbers; ~200 lines, read it all |
| `kernel/bpf/dispatcher.c` | the indirect-call optimisation |
| `kernel/trace/bpf_trace.c` ★★★ | tracing helpers and attach points |
| `net/core/filter.c` ★★★ | networking helpers and context access |
| `arch/x86/net/bpf_jit_comp.c` ★★★ | the x86-64 JIT |
| `include/uapi/linux/bpf.h` ★★★ | **the complete UAPI, with every helper documented** |
| `include/linux/bpf_verifier.h` ★★★ | the verifier's state structures |
| `tools/lib/bpf/` ★★★ | libbpf — read `libbpf.c`, `relo_core.c` for CO-RE |
| `Documentation/bpf/` ★★★ | `verifier.rst`, `btf.rst`, `libbpf/`, `map_*.rst`, `prog_*.rst` |

### 1.2 The verifier's main loop

```c
static int do_check(struct bpf_verifier_env *env)
{
	struct bpf_verifier_state *state = env->cur_state;
	struct bpf_insn *insns = env->prog->insnsi;
	struct bpf_reg_state *regs;
	int insn_cnt = env->prog->len;
	bool do_print_state = false;
	int prev_insn_idx = -1;

	for (;;) {
		struct bpf_insn *insn;
		u8 class;
		int err;

		env->prev_insn_idx = prev_insn_idx;
		if (env->insn_idx >= insn_cnt) {
			verbose(env, "invalid insn idx %d insn_cnt %d\n",
				env->insn_idx, insn_cnt);
			return -EFAULT;
		}

		insn = &insns[env->insn_idx];
		class = BPF_CLASS(insn->code);

		/* T.3(e): the resource limit */
		if (++env->insn_processed > BPF_COMPLEXITY_LIMIT_INSNS) {
			verbose(env, "BPF program is too large. Processed %d insn\n",
				env->insn_processed);
			return -E2BIG;
		}

		/* T.3(d): can we prune this path? */
		err = is_state_visited(env, env->insn_idx);
		if (err < 0)
			return err;
		if (err == 1) {
			/* An equivalent state was already verified. */
			if (env->log.level & BPF_LOG_LEVEL) {
				...
				verbose(env, "\nfrom %d to %d%s: safe\n", ...);
			}
			goto process_bpf_exit;
		}
		...
		regs = cur_regs(env);
		sanitize_mark_insn_seen(env);
		prev_insn_idx = env->insn_idx;

		if (class == BPF_ALU || class == BPF_ALU64) {
			err = check_alu_op(env, insn);
		} else if (class == BPF_LDX) {
			enum bpf_reg_type src_reg_type;

			/* T.3(3): check the read is in bounds for this pointer type */
			err = check_reg_arg(env, insn->src_reg, SRC_OP);
			if (err) return err;
			err = check_reg_arg(env, insn->dst_reg, DST_OP_NO_MARK);
			if (err) return err;

			src_reg_type = regs[insn->src_reg].type;

			err = check_mem_access(env, env->insn_idx, insn->src_reg,
					       insn->off, BPF_SIZE(insn->code),
					       BPF_READ, insn->dst_reg, false,
					       BPF_MODE(insn->code) == BPF_MEMSX);
			if (err) return err;
			...
		} else if (class == BPF_STX) {
			...
		} else if (class == BPF_JMP || class == BPF_JMP32) {
			u8 opcode = BPF_OP(insn->code);

			env->jmps_processed++;
			if (opcode == BPF_CALL) {
				...
				err = check_helper_call(env, insn, &env->insn_idx);
				...
			} else if (opcode == BPF_JA) {
				...
			} else if (opcode == BPF_EXIT) {
				...
			} else {
				/* A conditional jump: SPLIT the state and verify BOTH paths */
				err = check_cond_jmp_op(env, insn, &env->insn_idx);
				if (err) return err;
			}
		}
		...
		env->insn_idx++;
	}
	return 0;
}
```

### 1.3 Range and tnum tracking

```c
/* kernel/bpf/tnum.c -- T.3's per-bit knowledge; the whole file is ~200 lines */
struct tnum tnum_add(struct tnum a, struct tnum b)
{
	u64 sm, sv, sigma, chi, mu;

	sm = a.mask + b.mask;
	sv = a.value + b.value;
	sigma = sm + sv;
	chi = sigma ^ sv;              /* bits where carry made a difference */
	mu = chi | a.mask | b.mask;    /* uncertain bits propagate */
	return TNUM(sv & ~mu, mu);
}

struct tnum tnum_and(struct tnum a, struct tnum b)
{
	u64 alpha, beta, v;

	alpha = a.value | a.mask;
	beta = b.value | b.mask;
	v = a.value & b.value;
	return TNUM(v, alpha & beta & ~v);
}

/* Is `a` a subset of `b`? Used by state pruning. */
bool tnum_in(struct tnum a, struct tnum b)
{
	if (b.mask & ~a.mask)
		return false;
	b.value &= ~a.mask;
	return a.value == b.value;
}
```

And how a comparison refines the state:

```c
static void reg_set_min_max(struct bpf_reg_state *true_reg1,
			    struct bpf_reg_state *true_reg2,
			    struct bpf_reg_state *false_reg1,
			    struct bpf_reg_state *false_reg2,
			    u8 opcode, bool is_jmp32)
{
	...
	switch (opcode) {
	case BPF_JEQ:
	case BPF_JNE: {
		struct bpf_reg_state *reg = opcode == BPF_JEQ ? true_reg1 : false_reg1;
		...
		/* On the equal branch, the register IS the constant */
		if (is_jmp32) { ... } else { ___mark_reg_known(reg, val); }
		break;
	}
	case BPF_JGE:
	case BPF_JGT:
	{
		if (is_jmp32) { ... } else {
			u64 false_umax = opcode == BPF_JGT ? val    : val - 1;
			u64 true_umin  = opcode == BPF_JGT ? val + 1 : val;

			/* THE refinement: each branch learns something */
			false_reg1->umax_value = min(false_reg1->umax_value, false_umax);
			true_reg1->umin_value = max(true_reg1->umin_value, true_umin);
		}
		break;
	}
	...
	}

	/* Let the range and the tnum refine each other -- T.3 */
	reg_bounds_sync(false_reg1);
	reg_bounds_sync(true_reg1);
}
```

This is why `if (x < 100)` makes the true branch's `x` usable as an array index: the verifier *knows* `umax_value = 99`.

### 1.4 Packet-pointer verification

```c
static int check_packet_access(struct bpf_verifier_env *env, u32 regno, int off,
			       int size, bool zero_size_allowed)
{
	struct bpf_reg_state *regs = cur_regs(env);
	struct bpf_reg_state *reg = &regs[regno];
	int err;

	/* The offset must be a known constant or a bounded variable */
	if (off < 0) {
		verbose(env, "invalid access to packet, off=%d size=%d, R%d(id=%d,off=%d,r=%d)\n",
			off, size, regno, reg->id, reg->off, reg->range);
		return -EACCES;
	}

	err = reg->range < 0 ? -EINVAL :
	      __check_mem_access(env, regno, off, size, reg->range,
				 zero_size_allowed);
	if (err) {
		verbose(env, "R%d offset is outside of the packet\n", regno);
		return err;
	}
	...
	return err;
}

/* How `if (ptr + N > data_end)` grants access: */
static void find_good_pkt_pointers(struct bpf_verifier_state *vstate,
				   struct bpf_reg_state *dst_reg,
				   enum bpf_reg_type type, bool range_right_open)
{
	struct bpf_func_state *state;
	struct bpf_reg_state *reg;
	int new_range;

	if (dst_reg->off < 0 ||
	    (dst_reg->off == 0 && range_right_open))
		return;

	if (dst_reg->umax_value > MAX_PACKET_OFF ||
	    dst_reg->umax_value + dst_reg->off > MAX_PACKET_OFF)
		return;

	new_range = dst_reg->off;
	if (range_right_open)
		new_range++;

	/* Every register with the SAME id gets the new range.
	 * That is how `pkt + 14` and `pkt` are both extended. */
	bpf_for_each_reg_in_vstate(vstate, state, reg, ({
		if (reg->type == type && reg->id == dst_reg->id)
			reg->range = max(reg->range, new_range);
	}));
}
```

**`reg->id` is the mechanism**: pointers derived from the same packet pointer share an id, so a bounds check on one grants access through all of them. That is why this works:

```c
	struct iphdr *iph = data + sizeof(*eth);

	if ((void *)(iph + 1) > data_end)
		return XDP_DROP;
	/* Now BOTH `iph` and anything derived from it are usable. */
```

### 1.5 Reference tracking

```c
static int acquire_reference_state(struct bpf_verifier_env *env, int insn_idx)
{
	struct bpf_func_state *state = cur_func(env);
	int new_ofs = state->acquired_refs;
	int id, err;

	err = resize_reference_state(state, state->acquired_refs + 1);
	if (err) return err;
	id = ++env->id_gen;
	state->refs[new_ofs].id = id;
	state->refs[new_ofs].insn_idx = insn_idx;
	state->refs[new_ofs].callback_ref = state->in_callback_fn ? state->frameno : 0;
	return id;
}

/* At every exit, check that NOTHING is still held -- T.9's linear typing */
static int check_reference_leak(struct bpf_verifier_env *env, bool exception_exit)
{
	struct bpf_func_state *state = cur_func(env);
	bool refs_lingering = false;
	int i;

	if (!exception_exit && state->frameno && !state->in_callback_fn)
		return 0;

	for (i = 0; i < state->acquired_refs; i++) {
		if (!exception_exit && state->in_callback_fn &&
		    state->refs[i].callback_ref != state->frameno)
			continue;
		verbose(env, "Unreleased reference id=%d alloc_insn=%d\n",
			state->refs[i].id, state->refs[i].insn_idx);
		refs_lingering = true;
	}
	return refs_lingering ? -EINVAL : 0;
}
```

So this is rejected:

```c
	sk = bpf_sk_lookup_tcp(ctx, &tuple, sizeof(tuple), -1, 0);
	if (!sk)
		return 0;
	if (sk->state == BPF_TCP_ESTABLISHED)
		return 1;                       /* LEAK: sk not released */
	bpf_sk_release(sk);
	return 0;
```

**Every path must release.** The verifier proves it statically.

### 1.6 CO-RE relocation

```c
/* tools/lib/bpf/relo_core.c -- T.8, in libbpf */
int bpf_core_apply_relo_insn(const char *prog_name, struct bpf_insn *insn,
			     int insn_idx, const struct bpf_core_relo *relo,
			     int relo_idx, const struct btf *local_btf,
			     struct bpf_core_cand_list *cands,
			     struct bpf_core_spec *specs_scratch)
{
	struct bpf_core_spec *local_spec = &specs_scratch[0];
	struct bpf_core_spec *cand_spec = &specs_scratch[1];
	struct bpf_core_spec *targ_spec = &specs_scratch[2];
	struct bpf_core_relo_res targ_res;
	...
	/* Parse what the program WANTS: "struct task_struct . pid" */
	err = bpf_core_parse_spec(prog_name, local_btf, relo, local_spec);
	...
	/* Find it in the RUNNING kernel's BTF */
	for (i = 0, j = 0; i < cands->len; i++) {
		err = bpf_core_spec_match(local_spec, cands->cands[i].btf,
					  cands->cands[i].id, cand_spec);
		...
		err = bpf_core_calc_relo(prog_name, relo, relo_idx,
					 local_spec, cand_spec, &cand_res);
		...
	}
	...
	/* Patch the actual offset into the instruction */
	return bpf_core_patch_insn(prog_name, insn, insn_idx, relo, relo_idx,
				   &targ_res);
}

static int bpf_core_patch_insn(const char *prog_name, struct bpf_insn *insn,
			       int insn_idx, const struct bpf_core_relo *relo,
			       int relo_idx, const struct bpf_core_relo_res *res)
{
	__u32 orig_val, new_val;
	__u8 class;

	class = BPF_CLASS(insn->code);
	if (class == BPF_ALU || class == BPF_ALU64) {
		...
		orig_val = insn->imm;
		new_val = res->new_val;
		insn->imm = new_val;              /* patched */
		...
	} else if (class == BPF_LDX || class == BPF_ST || class == BPF_STX) {
		...
		orig_val = insn->off;
		new_val = res->new_val;
		insn->off = new_val;              /* THE offset, patched at load */
		...
	}
	...
}
```

**The offset is written into the instruction at load time**, by userspace, based on the running kernel's BTF. That is CO-RE in one sentence.

### 1.7 Observability

| Where | What |
|---|---|
| `bpftool prog show` ★★★ | loaded programs, type, run count, run time |
| `bpftool prog dump xlated id N` ★★★ | **the verified bytecode**, post-rewrite |
| `bpftool prog dump jited id N` ★★★ | the native code |
| `bpftool prog profile id N` ★★★ | cycles, instructions, cache misses |
| `bpftool map show`, `dump`, `update` ★★★ | |
| `bpftool btf dump file /sys/kernel/btf/vmlinux format c` ★★★ | generate `vmlinux.h` |
| `bpftool btf dump id N` | a program's BTF |
| `bpftool link show` ★★★ | attachments |
| `bpftool net show`, `cgroup tree`, `perf show` | attach points by subsystem |
| `bpftool feature probe` ★★★ | **what this kernel supports** |
| `bpftool prog tracelog` | `bpf_printk` output |
| `/sys/kernel/debug/tracing/trace_pipe` ★★★ | the same |
| `kernel.bpf_stats_enabled=1` ★★★ | per-program run count and total time |
| `/proc/sys/net/core/bpf_jit_*` | JIT tuning |
| `libbpf`'s verifier log ★★★ | `-vv` / `BPF_LOG_LEVEL2` |

`bpftool prog dump xlated` is underused and valuable: it shows the bytecode *after* the verifier's rewrites — inlined map lookups, inserted speculation barriers, patched CO-RE offsets. Comparing it against the source is how you learn what the verifier actually did.

---

## 2. Practice

### Lab 75.1 — Setup, and a first CO-RE program

```sh
sudo apt install -y clang llvm libbpf-dev libelf-dev zlib1g-dev \
     bpftool linux-tools-common linux-headers-$(uname -r) build-essential

# Does this kernel have BTF? -- T.8
ls -lh /sys/kernel/btf/vmlinux
grep CONFIG_DEBUG_INFO_BTF /boot/config-$(uname -r)

# Generate vmlinux.h
bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
wc -l vmlinux.h
grep -A15 'struct task_struct {' vmlinux.h | head -20

# What does this kernel support?
sudo bpftool feature probe | head -40
sudo bpftool feature probe | grep -c 'is available'
```

A CO-RE tracing program:

```c
// SPDX-License-Identifier: GPL-2.0
/* execsnoop.kern.c -- trace execve with CO-RE. */
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_core_read.h>
#include <bpf/bpf_tracing.h>

#define TASK_COMM_LEN	16
#define MAX_FILENAME	127

struct event {
	__u32 pid;
	__u32 ppid;
	__u32 uid;
	char comm[TASK_COMM_LEN];
	char filename[MAX_FILENAME];
};

struct {
	__uint(type, BPF_MAP_TYPE_RINGBUF);       /* T.5(d) */
	__uint(max_entries, 256 * 1024);
} events SEC(".maps");

/* For userspace to generate a matching type */
struct event _event = {};

SEC("tracepoint/syscalls/sys_enter_execve")
int trace_execve(struct trace_event_raw_sys_enter *ctx)
{
	struct task_struct *task;
	struct event *e;
	const char *filename;

	/* T.5(d): reserve, fill, submit -- no intermediate copy */
	e = bpf_ringbuf_reserve(&events, sizeof(*e), 0);
	if (!e)
		return 0;

	e->pid = bpf_get_current_pid_tgid() >> 32;
	e->uid = bpf_get_current_uid_gid() & 0xffffffff;
	bpf_get_current_comm(&e->comm, sizeof(e->comm));

	task = (struct task_struct *)bpf_get_current_task();
	/* T.8: BPF_CORE_READ chases pointers with CO-RE relocations */
	e->ppid = BPF_CORE_READ(task, real_parent, tgid);

	filename = (const char *)ctx->args[0];
	bpf_probe_read_user_str(&e->filename, sizeof(e->filename), filename);

	bpf_ringbuf_submit(e, 0);
	return 0;
}

char LICENSE[] SEC("license") = "GPL";
```

The userspace loader:

```c
// SPDX-License-Identifier: GPL-2.0
/* execsnoop.c */
#include <bpf/libbpf.h>
#include <signal.h>
#include <stdio.h>
#include <string.h>
#include <time.h>
#include <unistd.h>
#include "execsnoop.skel.h"

#define TASK_COMM_LEN	16
#define MAX_FILENAME	127

struct event {
	__u32 pid, ppid, uid;
	char comm[TASK_COMM_LEN];
	char filename[MAX_FILENAME];
};

static volatile bool stop;
static void on_sigint(int s) { stop = true; }

static int libbpf_print(enum libbpf_print_level level, const char *fmt,
			va_list args)
{
	if (level == LIBBPF_DEBUG)
		return 0;
	return vfprintf(stderr, fmt, args);
}

static int handle_event(void *ctx, void *data, size_t sz)
{
	const struct event *e = data;
	struct tm *tm;
	char ts[16];
	time_t t;

	time(&t);
	tm = localtime(&t);
	strftime(ts, sizeof(ts), "%H:%M:%S", tm);

	printf("%-9s %-7d %-7d %-6d %-16s %s\n",
	       ts, e->pid, e->ppid, e->uid, e->comm, e->filename);
	return 0;
}

int main(int argc, char **argv)
{
	struct ring_buffer *rb = NULL;
	struct execsnoop_bpf *skel;
	int err;

	libbpf_set_print(libbpf_print);
	signal(SIGINT, on_sigint);
	signal(SIGTERM, on_sigint);

	skel = execsnoop_bpf__open();
	if (!skel) { fprintf(stderr, "open failed\n"); return 1; }

	err = execsnoop_bpf__load(skel);      /* T.8: CO-RE relocation happens HERE */
	if (err) { fprintf(stderr, "load failed: %d\n", err); goto cleanup; }

	err = execsnoop_bpf__attach(skel);
	if (err) { fprintf(stderr, "attach failed: %d\n", err); goto cleanup; }

	rb = ring_buffer__new(bpf_map__fd(skel->maps.events), handle_event, NULL, NULL);
	if (!rb) { err = -1; goto cleanup; }

	printf("%-9s %-7s %-7s %-6s %-16s %s\n",
	       "TIME", "PID", "PPID", "UID", "COMM", "FILENAME");
	while (!stop) {
		err = ring_buffer__poll(rb, 100);
		if (err == -EINTR) { err = 0; break; }
		if (err < 0) break;
	}

cleanup:
	ring_buffer__free(rb);
	execsnoop_bpf__destroy(skel);
	return err != 0;
}
```

Build with a skeleton — the modern workflow:

```sh
cat > Makefile <<'EOF'
CLANG   ?= clang
BPFTOOL ?= bpftool
ARCH    := $(shell uname -m | sed 's/x86_64/x86/;s/aarch64/arm64/')
CFLAGS  := -g -O2 -Wall
BPF_CFLAGS := -g -O2 -target bpf -D__TARGET_ARCH_$(ARCH) -I.

all: execsnoop

vmlinux.h:
	$(BPFTOOL) btf dump file /sys/kernel/btf/vmlinux format c > $@

%.bpf.o: %.kern.c vmlinux.h
	$(CLANG) $(BPF_CFLAGS) -c $< -o $@

%.skel.h: %.bpf.o
	$(BPFTOOL) gen skeleton $< > $@

execsnoop: execsnoop.c execsnoop.skel.h
	$(CC) $(CFLAGS) -o $@ $< -lbpf -lelf -lz

clean:
	rm -f *.o *.skel.h execsnoop vmlinux.h
EOF

make
sudo ./execsnoop &
sleep 1
ls > /dev/null; date > /dev/null; /bin/true
sleep 2
sudo kill %1
```

Inspect it:

```sh
sudo ./execsnoop > /dev/null &
sleep 1
sudo bpftool prog show | grep -A4 trace_execve
PROGID=$(sudo bpftool prog show | grep -B1 trace_execve | grep -oP '^\d+' | head -1)
sudo bpftool prog dump xlated id $PROGID | head -30
sudo bpftool map show
sudo bpftool link show
sudo kill %1
```

Prove CO-RE works:

```sh
# The compiled object has NO fixed offsets for task_struct fields
llvm-objdump -h execsnoop.bpf.o | grep -i btf
sudo bpftool btf dump file execsnoop.bpf.o | grep -i 'task_struct\|CO-RE' | head
readelf -x .BTF.ext execsnoop.bpf.o 2>/dev/null | head -5
```

---

### Lab 75.2 — Make the verifier reject you

Understanding the verifier means reading its rejections.

```c
// SPDX-License-Identifier: GPL-2.0
/* verifier_tests.c -- deliberately broken programs. */
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>

struct {
	__uint(type, BPF_MAP_TYPE_ARRAY);
	__uint(max_entries, 16);
	__type(key, __u32);
	__type(value, __u64);
} test_map SEC(".maps");

#ifdef TEST_UNINIT
SEC("xdp")
int bad_uninit(struct xdp_md *ctx)
{
	int x;
	return x;              /* uninitialised */
}
#endif

#ifdef TEST_OOB
SEC("xdp")
int bad_oob(struct xdp_md *ctx)
{
	void *data = (void *)(long)ctx->data;
	struct ethhdr *eth = data;

	return eth->h_proto;   /* no bounds check */
}
#endif

#ifdef TEST_UNBOUNDED_LOOP
SEC("xdp")
int bad_loop(struct xdp_md *ctx)
{
	int i = 0;

	while (1) { i++; if (i == ctx->data_end) break; }   /* unbounded */
	return XDP_PASS;
}
#endif

#ifdef TEST_NULL_DEREF
SEC("xdp")
int bad_null(struct xdp_md *ctx)
{
	__u32 key = 0;
	__u64 *v = bpf_map_lookup_elem(&test_map, &key);

	return *v;             /* v may be NULL */
}
#endif

#ifdef TEST_UNBOUNDED_INDEX
SEC("xdp")
int bad_index(struct xdp_md *ctx)
{
	__u32 key = ctx->data_end - ctx->data;    /* unbounded */
	__u64 *v = bpf_map_lookup_elem(&test_map, &key);

	if (v) return *v;
	return 0;
}
#endif

#ifdef TEST_LEAK
SEC("tc")
int bad_leak(struct __sk_buff *skb)
{
	struct bpf_sock_tuple t = {};
	struct bpf_sock *sk;

	sk = bpf_sk_lookup_tcp(skb, &t, sizeof(t.ipv4), -1, 0);
	if (!sk) return 0;
	return 1;              /* LEAK: sk not released -- T.9 */
}
#endif

#ifdef TEST_STACK
SEC("xdp")
int bad_stack(struct xdp_md *ctx)
{
	char big[1024] = {};   /* > MAX_BPF_STACK (512) */

	big[0] = 1;
	return big[0];
}
#endif

#ifdef TEST_GOOD
SEC("xdp")
int good_prog(struct xdp_md *ctx)
{
	void *data = (void *)(long)ctx->data;
	void *data_end = (void *)(long)ctx->data_end;
	struct ethhdr *eth = data;
	__u32 key;
	__u64 *v;

	if ((void *)(eth + 1) > data_end)         /* bounds check */
		return XDP_DROP;

	key = eth->h_proto & 0xF;                 /* MASKED: now bounded */
	v = bpf_map_lookup_elem(&test_map, &key);
	if (!v)                                   /* NULL check */
		return XDP_PASS;
	*v += 1;
	return XDP_PASS;
}
#endif

char LICENSE[] SEC("license") = "GPL";
```

```sh
for t in UNINIT OOB UNBOUNDED_LOOP NULL_DEREF UNBOUNDED_INDEX LEAK STACK GOOD; do
  echo "=========== $t ==========="
  clang -O2 -g -target bpf -D TEST_$t -I. -c verifier_tests.c -o vt.o 2>/dev/null
  sudo bpftool prog load vt.o /sys/fs/bpf/vt 2>&1 | head -12
  sudo rm -f /sys/fs/bpf/vt
  echo
done
```

Sample rejections:

```
=========== OOB ===========
0: (61) r2 = *(u32 *)(r1 +0)
1: (61) r1 = *(u32 *)(r1 +4)
2: (bf) r3 = r2
3: (07) r3 += 14
4: (71) r0 = *(u8 *)(r2 +12)
invalid access to packet, off=12 size=1, R2(id=0,off=0,r=0)
R2 offset is outside of the packet

=========== LEAK ===========
...
Unreleased reference id=2 alloc_insn=7
```

The verbose log — §1.2:

```sh
clang -O2 -g -target bpf -D TEST_GOOD -I. -c verifier_tests.c -o vt.o
sudo bpftool prog load vt.o /sys/fs/bpf/vt --debug 2>&1 | head -50
sudo rm -f /sys/fs/bpf/vt
```

Read the state annotations: `R1=ctx(off=0,imm=0) R2_w=pkt(off=0,r=0,imm=0) R3_w=pkt_end()`. After the bounds check, `r=14` appears — the granted range of §1.4.

Bounded loops — §T.4:

```c
// SPDX-License-Identifier: GPL-2.0
/* loops.c */
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>

#ifdef LOOP_UNROLLED
SEC("xdp")
int loop_unrolled(struct xdp_md *ctx)
{
	int sum = 0, i;

#pragma unroll
	for (i = 0; i < 16; i++)
		sum += i;
	return sum & 1;
}
#endif

#ifdef LOOP_BOUNDED
SEC("xdp")
int loop_bounded(struct xdp_md *ctx)
{
	int sum = 0, i;

	for (i = 0; i < 100; i++)           /* verifier proves termination */
		sum += i;
	return sum & 1;
}
#endif

#ifdef LOOP_DATADEP
SEC("xdp")
int loop_datadep(struct xdp_md *ctx)
{
	int n = ctx->data_end - ctx->data;  /* UNBOUNDED */
	int sum = 0, i;

	for (i = 0; i < n; i++)
		sum += i;
	return sum & 1;
}
#endif

#ifdef LOOP_CLAMPED
SEC("xdp")
int loop_clamped(struct xdp_md *ctx)
{
	int n = ctx->data_end - ctx->data;
	int sum = 0, i;

	if (n > 64) n = 64;                 /* CLAMPED: now verifiable */
	if (n < 0)  n = 0;
	for (i = 0; i < n; i++)
		sum += i;
	return sum & 1;
}
#endif

#ifdef LOOP_HELPER
static long body(__u32 index, void *ctx)
{
	__u64 *sum = ctx;

	*sum += index;
	return 0;
}

SEC("xdp")
int loop_helper(struct xdp_md *ctx)
{
	__u64 sum = 0;

	bpf_loop(1000000, body, &sum, 0);   /* T.4's escape hatch */
	return sum & 1;
}
#endif

char LICENSE[] SEC("license") = "GPL";
```

```sh
for l in UNROLLED BOUNDED DATADEP CLAMPED HELPER; do
  echo -n "LOOP_$l: "
  clang -O2 -g -target bpf -D LOOP_$l -I. -c loops.c -o loops.o 2>/dev/null
  sudo bpftool prog load loops.o /sys/fs/bpf/l 2>&1 | head -3 | tr '\n' ' '
  echo
  sudo rm -f /sys/fs/bpf/l
done
```

---

### Lab 75.3 — Map types, compared

```c
// SPDX-License-Identifier: GPL-2.0
/* maps.kern.c */
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>

struct {
	__uint(type, BPF_MAP_TYPE_ARRAY);
	__uint(max_entries, 1024);
	__type(key, __u32);
	__type(value, __u64);
} m_array SEC(".maps");

struct {
	__uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
	__uint(max_entries, 1024);
	__type(key, __u32);
	__type(value, __u64);
} m_percpu_array SEC(".maps");

struct {
	__uint(type, BPF_MAP_TYPE_HASH);
	__uint(max_entries, 65536);
	__type(key, __u64);
	__type(value, __u64);
} m_hash SEC(".maps");

struct {
	__uint(type, BPF_MAP_TYPE_PERCPU_HASH);
	__uint(max_entries, 65536);
	__type(key, __u64);
	__type(value, __u64);
} m_percpu_hash SEC(".maps");

struct {
	__uint(type, BPF_MAP_TYPE_LRU_HASH);
	__uint(max_entries, 1024);
	__type(key, __u64);
	__type(value, __u64);
} m_lru SEC(".maps");

struct {
	__uint(type, BPF_MAP_TYPE_TASK_STORAGE);       /* T.5(c) */
	__uint(map_flags, BPF_F_NO_PREALLOC);
	__type(key, int);
	__type(value, __u64);
} m_task SEC(".maps");

struct {
	__uint(type, BPF_MAP_TYPE_ARRAY);
	__uint(max_entries, 1);
	__type(key, __u32);
	__type(value, __u32);
} which SEC(".maps");

SEC("tracepoint/syscalls/sys_enter_getpid")
int bench_maps(void *ctx)
{
	__u32 zero = 0, *sel, k32;
	__u64 k64 = bpf_get_current_pid_tgid(), *v, one = 1;
	struct task_struct *task;

	sel = bpf_map_lookup_elem(&which, &zero);
	if (!sel) return 0;

	k32 = k64 & 1023;
	switch (*sel) {
	case 0: break;
	case 1: v = bpf_map_lookup_elem(&m_array, &k32);        if (v) *v += 1; break;
	case 2: v = bpf_map_lookup_elem(&m_percpu_array, &k32); if (v) *v += 1; break;
	case 3:
		v = bpf_map_lookup_elem(&m_hash, &k64);
		if (v) __sync_fetch_and_add(v, 1);
		else bpf_map_update_elem(&m_hash, &k64, &one, BPF_ANY);
		break;
	case 4:
		v = bpf_map_lookup_elem(&m_percpu_hash, &k64);
		if (v) *v += 1;
		else bpf_map_update_elem(&m_percpu_hash, &k64, &one, BPF_ANY);
		break;
	case 5:
		v = bpf_map_lookup_elem(&m_lru, &k64);
		if (v) *v += 1;
		else bpf_map_update_elem(&m_lru, &k64, &one, BPF_ANY);
		break;
	case 6:
		task = (struct task_struct *)bpf_get_current_task_btf();
		v = bpf_task_storage_get(&m_task, task, 0,
					 BPF_LOCAL_STORAGE_GET_F_CREATE);
		if (v) *v += 1;
		break;
	}
	return 0;
}

char LICENSE[] SEC("license") = "GPL";
```

```sh
clang -O2 -g -target bpf -I. -c maps.kern.c -o maps.bpf.o
sudo bpftool prog load maps.bpf.o /sys/fs/bpf/maps type tracepoint
sudo bpftool prog show pinned /sys/fs/bpf/maps

PROGID=$(sudo bpftool prog show pinned /sys/fs/bpf/maps | grep -oP '^\d+')
sudo bpftool prog attach pinned /sys/fs/bpf/maps 2>/dev/null || \
  sudo bpftool prog loadall maps.bpf.o /sys/fs/bpf/m autoattach

# Enable statistics -- T.10
sudo sysctl -w kernel.bpf_stats_enabled=1

cat > getpid_bench.c <<'EOF'
#include <stdio.h>
#include <sys/syscall.h>
#include <unistd.h>
#include <time.h>
int main(void) {
	struct timespec a, b; long i, n = 2000000;
	clock_gettime(CLOCK_MONOTONIC, &a);
	for (i = 0; i < n; i++) syscall(SYS_getpid);
	clock_gettime(CLOCK_MONOTONIC, &b);
	printf("%.1f ns/call\n",
	       ((b.tv_sec-a.tv_sec)*1e9 + (b.tv_nsec-a.tv_nsec)) / n);
	return 0;
}
EOF
gcc -O2 -o getpid_bench getpid_bench.c

for sel in 0 1 2 3 4 5 6; do
  case $sel in
    0) N="no map";; 1) N="ARRAY";; 2) N="PERCPU_ARRAY";; 3) N="HASH";;
    4) N="PERCPU_HASH";; 5) N="LRU_HASH";; 6) N="TASK_STORAGE";;
  esac
  sudo bpftool map update name which key 0 0 0 0 value $sel 0 0 0 2>/dev/null
  printf "%-16s " "$N"
  ./getpid_bench
done

sudo sysctl -w kernel.bpf_stats_enabled=0
sudo bpftool prog show | grep -A3 bench_maps
sudo rm -f /sys/fs/bpf/maps /sys/fs/bpf/m*
```

**`PERCPU_ARRAY` should be fastest, `HASH` with atomics slowest.** §T.5(a).

Ring buffer versus perf buffer — §T.5(d):

```sh
sudo bpftool map show | grep -E 'ringbuf|perf_event_array'
# The RINGBUF is one shared buffer; PERF_EVENT_ARRAY is per-CPU.
sudo bpftool map dump name events 2>/dev/null | head -3
```

Local storage lifetime — §T.5(c):

```sh
# TASK_STORAGE entries vanish when the task exits: no cleanup code needed
sudo bpftool map show | grep task_storage
```

---

### Lab 75.4 — Program types and attach points

```sh
sudo bpftool feature probe | grep -A40 'program types' | head -45
```

`fentry`/`fexit` versus kprobe — §T.6:

```c
// SPDX-License-Identifier: GPL-2.0
/* attach.kern.c */
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>

struct {
	__uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
	__uint(max_entries, 8);
	__type(key, __u32);
	__type(value, __u64);
} counts SEC(".maps");

static __always_inline void bump(__u32 k)
{
	__u64 *v = bpf_map_lookup_elem(&counts, &k);

	if (v) *v += 1;
}

SEC("kprobe/do_sys_openat2")
int BPF_KPROBE(kprobe_open, int dfd, const char *filename)
{
	bump(0);
	return 0;
}

SEC("kretprobe/do_sys_openat2")
int BPF_KRETPROBE(kretprobe_open, long ret)
{
	bump(1);
	return 0;
}

/* T.6: a trampoline, not a breakpoint trap */
SEC("fentry/do_sys_openat2")
int BPF_PROG(fentry_open, int dfd, struct open_how *how)
{
	bump(2);
	return 0;
}

/* fexit sees BOTH the arguments and the return value */
SEC("fexit/do_sys_openat2")
int BPF_PROG(fexit_open, int dfd, struct open_how *how, long ret)
{
	bump(3);
	if (ret < 0)
		bump(4);
	return 0;
}

SEC("tracepoint/syscalls/sys_enter_openat")
int tp_open(struct trace_event_raw_sys_enter *ctx)
{
	bump(5);
	return 0;
}

SEC("raw_tracepoint/sys_enter")
int rawtp_enter(struct bpf_raw_tracepoint_args *ctx)
{
	bump(6);
	return 0;
}

char LICENSE[] SEC("license") = "GPL";
```

```sh
clang -O2 -g -target bpf -I. -c attach.kern.c -o attach.bpf.o
sudo bpftool gen skeleton attach.bpf.o > attach.skel.h 2>/dev/null

sudo bpftool prog loadall attach.bpf.o /sys/fs/bpf/attach autoattach 2>&1 | head -5
sudo bpftool prog show | grep -E 'kprobe_open|fentry_open|fexit_open|tp_open'
sudo bpftool link show

sudo sysctl -w kernel.bpf_stats_enabled=1
for i in $(seq 1 20000); do cat /dev/null; done 2>/dev/null
sudo bpftool prog show | grep -A2 -E 'kprobe_open|fentry_open' | grep -E 'run_time|run_cnt'
sudo bpftool map dump name counts
sudo sysctl -w kernel.bpf_stats_enabled=0
```

**fentry should be several times faster than kprobe**, because there is no trap.

Overhead measurement:

```sh
cat > openbench.c <<'EOF'
#include <fcntl.h>
#include <stdio.h>
#include <time.h>
#include <unistd.h>
int main(void) {
	struct timespec a, b; long i, n = 500000; int fd;
	clock_gettime(CLOCK_MONOTONIC, &a);
	for (i = 0; i < n; i++) { fd = open("/dev/null", O_RDONLY); close(fd); }
	clock_gettime(CLOCK_MONOTONIC, &b);
	printf("%.0f ns per open+close\n",
	       ((b.tv_sec-a.tv_sec)*1e9 + (b.tv_nsec-a.tv_nsec)) / n);
	return 0;
}
EOF
gcc -O2 -o openbench openbench.c

echo -n "baseline (no programs): "
sudo rm -rf /sys/fs/bpf/attach*
./openbench

sudo bpftool prog loadall attach.bpf.o /sys/fs/bpf/attach autoattach 2>/dev/null
echo -n "with all programs:      "
./openbench
sudo rm -rf /sys/fs/bpf/attach*
```

cgroup programs:

```c
// SPDX-License-Identifier: GPL-2.0
/* cgroup.kern.c */
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>

struct {
	__uint(type, BPF_MAP_TYPE_ARRAY);
	__uint(max_entries, 4);
	__type(key, __u32);
	__type(value, __u64);
} cg_stats SEC(".maps");

SEC("cgroup_skb/egress")
int cg_egress(struct __sk_buff *skb)
{
	__u32 k = 0;
	__u64 *v = bpf_map_lookup_elem(&cg_stats, &k);

	if (v) *v += skb->len;

	/* Block port 9999 for this cgroup */
	if (skb->protocol == bpf_htons(ETH_P_IP)) {
		struct iphdr iph;

		if (bpf_skb_load_bytes(skb, 0, &iph, sizeof(iph)) == 0 &&
		    iph.protocol == IPPROTO_TCP) {
			struct tcphdr tcph;

			if (bpf_skb_load_bytes(skb, iph.ihl * 4, &tcph,
					       sizeof(tcph)) == 0 &&
			    tcph.dest == bpf_htons(9999)) {
				k = 1;
				v = bpf_map_lookup_elem(&cg_stats, &k);
				if (v) *v += 1;
				return 0;      /* DENY */
			}
		}
	}
	return 1;                              /* allow */
}

char LICENSE[] SEC("license") = "GPL";
```

```sh
clang -O2 -g -target bpf -I. -c cgroup.kern.c -o cgroup.bpf.o
sudo mkdir -p /sys/fs/cgroup/bpftest
sudo bpftool prog load cgroup.bpf.o /sys/fs/bpf/cg type cgroup/skb
sudo bpftool cgroup attach /sys/fs/cgroup/bpftest egress pinned /sys/fs/bpf/cg
sudo bpftool cgroup tree

sudo sh -c 'echo $$ > /sys/fs/cgroup/bpftest/cgroup.procs; \
  nc -z -w1 127.0.0.1 22 2>/dev/null && echo "port 22 OK"; \
  nc -z -w1 127.0.0.1 9999 2>/dev/null && echo "port 9999 OK" || echo "port 9999 BLOCKED"'

sudo bpftool map dump name cg_stats
sudo bpftool cgroup detach /sys/fs/cgroup/bpftest egress pinned /sys/fs/bpf/cg
sudo rm -f /sys/fs/bpf/cg
sudo rmdir /sys/fs/cgroup/bpftest
```

---

### Lab 75.5 — CO-RE across kernels

```sh
# What the program asks for
sudo bpftool btf dump file execsnoop.bpf.o format raw 2>/dev/null | head -20

# What the kernel provides
sudo bpftool btf dump file /sys/kernel/btf/vmlinux format c | \
  grep -A20 'struct task_struct {' | head -25

# The offsets CO-RE resolves
cat > offsets.bt <<'EOF'
BEGIN {
	printf("offsetof(task_struct, pid) = %d\n", offsetof(struct task_struct, pid));
	printf("offsetof(task_struct, tgid) = %d\n", offsetof(struct task_struct, tgid));
	printf("offsetof(task_struct, comm) = %d\n", offsetof(struct task_struct, comm));
	printf("sizeof(struct task_struct) = %d\n", sizeof(struct task_struct));
	exit();
}
EOF
sudo bpftrace offsets.bt 2>/dev/null
```

CO-RE's conditional features — §T.8:

```c
// SPDX-License-Identifier: GPL-2.0
/* core_feat.kern.c */
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_core_read.h>
#include <bpf/bpf_tracing.h>

struct {
	__uint(type, BPF_MAP_TYPE_ARRAY);
	__uint(max_entries, 8);
	__type(key, __u32);
	__type(value, __u64);
} info SEC(".maps");

static __always_inline void put(__u32 k, __u64 v)
{
	bpf_map_update_elem(&info, &k, &v, BPF_ANY);
}

SEC("tracepoint/syscalls/sys_enter_getpid")
int probe_core(void *ctx)
{
	struct task_struct *task = (void *)bpf_get_current_task();

	/* T.8: does this field exist on the RUNNING kernel? */
	if (bpf_core_field_exists(task->pid))
		put(0, BPF_CORE_READ(task, pid));
	if (bpf_core_field_exists(task->tgid))
		put(1, BPF_CORE_READ(task, tgid));
	if (bpf_core_field_exists(task->start_boottime))
		put(2, BPF_CORE_READ(task, start_boottime));
	else if (bpf_core_field_exists(task->real_start_time))
		put(2, 0);         /* the old name, pre-5.5 */

	put(3, bpf_core_field_size(task->comm));
	put(4, bpf_core_type_exists(struct bpf_ringbuf) ? 1 : 0);

	return 0;
}

char LICENSE[] SEC("license") = "GPL";
```

```sh
clang -O2 -g -target bpf -I. -c core_feat.kern.c -o core_feat.bpf.o
sudo bpftool prog loadall core_feat.bpf.o /sys/fs/bpf/cf autoattach 2>/dev/null
getpid > /dev/null 2>&1 || /bin/true
sudo bpftool map dump name info
sudo rm -rf /sys/fs/bpf/cf*
```

Without BTF — BTFHub:

```sh
# For old kernels without CONFIG_DEBUG_INFO_BTF:
# git clone https://github.com/aquasecurity/btfhub-archive
# Then: LIBBPF_BTF_CUSTOM_PATH=/path/to/vmlinux-5.4.0-42.btf ./myprog
echo "kernel: $(uname -r)"
ls /sys/kernel/btf/ | head
ls /sys/kernel/btf/ | wc -l    # per-module BTF as well
```

The relocations, visible:

```sh
llvm-objdump -r execsnoop.bpf.o 2>/dev/null | head -20
sudo bpftool prog dump xlated id $(sudo bpftool prog show | grep trace_execve | \
  grep -oP '^\d+' | head -1) 2>/dev/null | head -20
# Compare with the compiled object: the offsets were patched at LOAD time.
```

---

### Lab 75.6 — `struct_ops`: a congestion-control algorithm

§T.7, and Ch. 72 §T.5's pluggable interface.

```c
// SPDX-License-Identifier: GPL-2.0
/* bpf_cc.kern.c -- a simple congestion control algorithm in BPF. */
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>

#define USEC_PER_SEC	1000000UL
#define TCP_INFINITE_SSTHRESH	0x7fffffff

char LICENSE[] SEC("license") = "GPL";

struct simple_state {
	__u32 acked_total;
	__u32 loss_events;
};

extern void tcp_reno_cong_avoid(struct sock *sk, __u32 ack, __u32 acked) __ksym;
extern __u32 tcp_reno_ssthresh(struct sock *sk) __ksym;
extern __u32 tcp_reno_undo_cwnd(struct sock *sk) __ksym;

SEC("struct_ops/simple_init")
void BPF_PROG(simple_init, struct sock *sk)
{
	struct simple_state *s = inet_csk_ca(sk);

	s->acked_total = 0;
	s->loss_events = 0;
}

SEC("struct_ops/simple_cong_avoid")
void BPF_PROG(simple_cong_avoid, struct sock *sk, __u32 ack, __u32 acked)
{
	struct tcp_sock *tp = tcp_sk(sk);
	struct simple_state *s = inet_csk_ca(sk);

	s->acked_total += acked;

	if (tp->snd_cwnd < tp->snd_ssthresh) {
		/* Slow start: double per RTT */
		tp->snd_cwnd += acked;
		if (tp->snd_cwnd > tp->snd_cwnd_clamp)
			tp->snd_cwnd = tp->snd_cwnd_clamp;
	} else {
		/* Congestion avoidance: +1 per RTT */
		tp->snd_cwnd_cnt += acked;
		if (tp->snd_cwnd_cnt >= tp->snd_cwnd) {
			tp->snd_cwnd_cnt -= tp->snd_cwnd;
			if (tp->snd_cwnd < tp->snd_cwnd_clamp)
				tp->snd_cwnd++;
		}
	}
}

SEC("struct_ops/simple_ssthresh")
__u32 BPF_PROG(simple_ssthresh, struct sock *sk)
{
	struct tcp_sock *tp = tcp_sk(sk);
	struct simple_state *s = inet_csk_ca(sk);

	s->loss_events++;
	return tp->snd_cwnd > 4 ? tp->snd_cwnd * 3 / 4 : 2;   /* beta = 0.75 */
}

SEC("struct_ops/simple_undo_cwnd")
__u32 BPF_PROG(simple_undo_cwnd, struct sock *sk)
{
	struct tcp_sock *tp = tcp_sk(sk);

	return tp->snd_cwnd > tp->prior_cwnd ? tp->snd_cwnd : tp->prior_cwnd;
}

/* T.7: BPF implementing a kernel operations structure */
SEC(".struct_ops")
struct tcp_congestion_ops bpf_simple = {
	.init		= (void *)simple_init,
	.ssthresh	= (void *)simple_ssthresh,
	.cong_avoid	= (void *)simple_cong_avoid,
	.undo_cwnd	= (void *)simple_undo_cwnd,
	.name		= "bpf_simple",
};
```

```sh
grep CONFIG_BPF_JIT /boot/config-$(uname -r)
sysctl net.ipv4.tcp_available_congestion_control

clang -O2 -g -target bpf -I. -c bpf_cc.kern.c -o bpf_cc.bpf.o
sudo bpftool struct_ops register bpf_cc.bpf.o 2>&1 | head -5
sudo bpftool struct_ops show
sysctl net.ipv4.tcp_available_congestion_control
```

If it registered:

```sh
sudo sysctl -w net.ipv4.tcp_congestion_control=bpf_simple
iperf3 -s -1 > /dev/null 2>&1 &
sleep 0.5; iperf3 -c 127.0.0.1 -t 5 2>/dev/null | grep receiver
ss -ti | grep -o 'bpf_simple' | head -1

sudo sysctl -w net.ipv4.tcp_congestion_control=cubic
sudo bpftool struct_ops unregister name bpf_simple 2>/dev/null
```

`sched_ext` — the more ambitious user:

```sh
grep CONFIG_SCHED_CLASS_EXT /boot/config-$(uname -r) 2>/dev/null
ls /sys/kernel/sched_ext/ 2>/dev/null
# If present: scx_simple, scx_rusty etc. from the scx repository
# git clone https://github.com/sched-ext/scx
```

---

### Lab 75.7 — Security and the LSM

```sh
sysctl kernel.unprivileged_bpf_disabled
sysctl net.core.bpf_jit_enable net.core.bpf_jit_harden net.core.bpf_jit_kallsyms
sysctl kernel.perf_event_paranoid

# Try as an unprivileged user -- T.10
sudo -u nobody bpftool prog load /dev/null /tmp/x 2>&1 | head -3
```

The JIT:

```sh
sudo bpftool prog loadall attach.bpf.o /sys/fs/bpf/j autoattach 2>/dev/null
PROGID=$(sudo bpftool prog show | grep fentry_open | grep -oP '^\d+' | head -1)

echo "=== BPF bytecode (after verifier rewrites) ==="
sudo bpftool prog dump xlated id $PROGID | head -20

echo "=== Native code ==="
sudo bpftool prog dump jited id $PROGID 2>/dev/null | head -25
sudo rm -rf /sys/fs/bpf/j*
```

Constant blinding — §T.10:

```sh
sudo sysctl -w net.core.bpf_jit_harden=2
sudo bpftool prog loadall attach.bpf.o /sys/fs/bpf/h autoattach 2>/dev/null
PROGID=$(sudo bpftool prog show | grep fentry_open | grep -oP '^\d+' | head -1)
sudo bpftool prog dump jited id $PROGID 2>/dev/null | head -20
# Constants are XORed with a random value; no attacker-chosen gadgets.
sudo sysctl -w net.core.bpf_jit_harden=0
sudo rm -rf /sys/fs/bpf/h*
```

BPF LSM — §T.6:

```c
// SPDX-License-Identifier: GPL-2.0
/* lsm.kern.c -- deny writes to a specific path. */
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>
#include <bpf/bpf_core_read.h>

char LICENSE[] SEC("license") = "GPL";

struct {
	__uint(type, BPF_MAP_TYPE_ARRAY);
	__uint(max_entries, 4);
	__type(key, __u32);
	__type(value, __u64);
} lsm_stats SEC(".maps");

static __always_inline void bump(__u32 k)
{
	__u64 *v = bpf_map_lookup_elem(&lsm_stats, &k);

	if (v) *v += 1;
}

SEC("lsm/file_open")
int BPF_PROG(deny_open, struct file *file, int ret)
{
	char name[64] = {};
	struct dentry *dentry;
	const unsigned char *dname;

	if (ret != 0)
		return ret;         /* a previous LSM already denied */

	bump(0);

	dentry = BPF_CORE_READ(file, f_path.dentry);
	dname = BPF_CORE_READ(dentry, d_name.name);
	bpf_probe_read_kernel_str(name, sizeof(name), dname);

	/* Deny opening anything named "bpf_forbidden" */
	if (name[0] == 'b' && name[1] == 'p' && name[2] == 'f' &&
	    name[3] == '_' && name[4] == 'f') {
		bump(1);
		return -EPERM;      /* T.6: BPF can DENY */
	}
	return 0;
}
```

```sh
grep CONFIG_BPF_LSM /boot/config-$(uname -r)
cat /sys/kernel/security/lsm

# BPF must be in the LSM list (kernel cmdline: lsm=...,bpf)
if grep -q bpf /sys/kernel/security/lsm; then
  clang -O2 -g -target bpf -I. -c lsm.kern.c -o lsm.bpf.o
  sudo bpftool prog loadall lsm.bpf.o /sys/fs/bpf/lsm autoattach 2>&1 | head -3
  touch /tmp/bpf_forbidden_file
  cat /tmp/bpf_forbidden_file 2>&1 | head -1
  cat /etc/hostname > /dev/null && echo "normal files still work"
  sudo bpftool map dump name lsm_stats
  sudo rm -rf /sys/fs/bpf/lsm*
else
  echo "BPF LSM not enabled; add 'lsm=...,bpf' to the kernel cmdline"
fi
```

Program statistics:

```sh
sudo sysctl -w kernel.bpf_stats_enabled=1
sudo bpftool prog loadall attach.bpf.o /sys/fs/bpf/s autoattach 2>/dev/null
for i in $(seq 1 50000); do cat /dev/null; done 2>/dev/null
sudo bpftool prog show | grep -B1 -A3 'run_time_ns' | head -20
sudo sysctl -w kernel.bpf_stats_enabled=0
sudo rm -rf /sys/fs/bpf/s*
```

---

### Lab 75.8 — A complete observability tool

Put it all together: a tool tracking TCP connection lifetimes.

```c
// SPDX-License-Identifier: GPL-2.0
/* tcplife.kern.c */
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_core_read.h>
#include <bpf/bpf_tracing.h>
#include <bpf/bpf_endian.h>

char LICENSE[] SEC("license") = "GPL";

#define AF_INET		2
#define AF_INET6	10
#define TASK_COMM_LEN	16

struct event {
	__u64 ts_us;
	__u64 span_us;
	__u64 rx_b;
	__u64 tx_b;
	__u32 pid;
	__u32 saddr;
	__u32 daddr;
	__u16 sport;
	__u16 dport;
	__u16 family;
	char comm[TASK_COMM_LEN];
};

struct {
	__uint(type, BPF_MAP_TYPE_RINGBUF);
	__uint(max_entries, 256 * 1024);
} events SEC(".maps");

/* T.5(c): per-socket storage, freed when the socket is */
struct sock_info {
	__u64 birth_us;
	__u32 pid;
	char comm[TASK_COMM_LEN];
};

struct {
	__uint(type, BPF_MAP_TYPE_SK_STORAGE);
	__uint(map_flags, BPF_F_NO_PREALLOC);
	__type(key, int);
	__type(value, struct sock_info);
} sk_info SEC(".maps");

struct event _event = {};

SEC("tracepoint/sock/inet_sock_set_state")
int trace_state(struct trace_event_raw_inet_sock_set_state *ctx)
{
	struct sock *sk = (struct sock *)ctx->skaddr;
	struct sock_info *si;
	struct event *e;
	__u64 now = bpf_ktime_get_ns() / 1000;

	if (ctx->protocol != IPPROTO_TCP)
		return 0;
	if (ctx->family != AF_INET)
		return 0;

	if (ctx->newstate == BPF_TCP_SYN_SENT ||
	    ctx->newstate == BPF_TCP_LAST_ACK) {
		si = bpf_sk_storage_get(&sk_info, sk, 0,
					BPF_LOCAL_STORAGE_GET_F_CREATE);
		if (!si)
			return 0;
		if (si->birth_us == 0) {
			si->birth_us = now;
			si->pid = bpf_get_current_pid_tgid() >> 32;
			bpf_get_current_comm(&si->comm, sizeof(si->comm));
		}
		return 0;
	}

	if (ctx->newstate != BPF_TCP_CLOSE)
		return 0;

	si = bpf_sk_storage_get(&sk_info, sk, 0, 0);
	if (!si || si->birth_us == 0)
		return 0;

	e = bpf_ringbuf_reserve(&events, sizeof(*e), 0);
	if (!e)
		return 0;

	e->ts_us = now;
	e->span_us = now - si->birth_us;
	e->pid = si->pid;
	__builtin_memcpy(e->comm, si->comm, TASK_COMM_LEN);
	e->family = ctx->family;
	e->sport = ctx->sport;
	e->dport = ctx->dport;
	__builtin_memcpy(&e->saddr, ctx->saddr, 4);
	__builtin_memcpy(&e->daddr, ctx->daddr, 4);

	{
		struct tcp_sock *tp = (struct tcp_sock *)sk;

		e->rx_b = BPF_CORE_READ(tp, bytes_received);
		e->tx_b = BPF_CORE_READ(tp, bytes_acked);
	}

	bpf_ringbuf_submit(e, 0);
	bpf_sk_storage_delete(&sk_info, sk);
	return 0;
}
```

```c
// SPDX-License-Identifier: GPL-2.0
/* tcplife.c */
#include <arpa/inet.h>
#include <bpf/libbpf.h>
#include <signal.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include "tcplife.skel.h"

#define TASK_COMM_LEN 16

struct event {
	__u64 ts_us, span_us, rx_b, tx_b;
	__u32 pid, saddr, daddr;
	__u16 sport, dport, family;
	char comm[TASK_COMM_LEN];
};

static volatile bool stop;
static void on_sigint(int s) { stop = true; }

static int handle(void *ctx, void *data, size_t sz)
{
	const struct event *e = data;
	char s[INET_ADDRSTRLEN], d[INET_ADDRSTRLEN];

	inet_ntop(AF_INET, &e->saddr, s, sizeof(s));
	inet_ntop(AF_INET, &e->daddr, d, sizeof(d));

	printf("%-7d %-16s %-15s %-6d %-15s %-6d %8llu %8llu %9.2f\n",
	       e->pid, e->comm, s, e->sport, d, e->dport,
	       e->rx_b / 1024, e->tx_b / 1024, e->span_us / 1000.0);
	return 0;
}

int main(void)
{
	struct ring_buffer *rb = NULL;
	struct tcplife_bpf *skel;
	int err;

	signal(SIGINT, on_sigint);

	skel = tcplife_bpf__open_and_load();
	if (!skel) { fprintf(stderr, "open/load failed\n"); return 1; }

	err = tcplife_bpf__attach(skel);
	if (err) { fprintf(stderr, "attach failed\n"); goto out; }

	rb = ring_buffer__new(bpf_map__fd(skel->maps.events), handle, NULL, NULL);
	if (!rb) { err = 1; goto out; }

	printf("%-7s %-16s %-15s %-6s %-15s %-6s %8s %8s %9s\n",
	       "PID", "COMM", "LADDR", "LPORT", "RADDR", "RPORT",
	       "RX_KB", "TX_KB", "MS");

	while (!stop) {
		err = ring_buffer__poll(rb, 100);
		if (err == -EINTR) { err = 0; break; }
		if (err < 0) break;
	}
out:
	ring_buffer__free(rb);
	tcplife_bpf__destroy(skel);
	return err != 0;
}
```

```sh
cat >> Makefile <<'EOF'

tcplife: tcplife.c tcplife.skel.h
	$(CC) $(CFLAGS) -o $@ $< -lbpf -lelf -lz
EOF

make tcplife
sudo ./tcplife &
sleep 1
curl -s -m 2 http://127.0.0.1:22 > /dev/null 2>&1
nc -z -w1 127.0.0.1 22 2>/dev/null
for i in 1 2 3; do (nc -z 127.0.0.1 22 2>/dev/null &); done
sleep 3
sudo kill %1
```

Inspect what it built:

```sh
sudo ./tcplife > /dev/null &
sleep 1
sudo bpftool prog show | grep -A4 trace_state
sudo bpftool map show | grep -E 'ringbuf|sk_storage'
sudo bpftool link show
sudo bpftool prog dump xlated id $(sudo bpftool prog show | grep trace_state | \
  grep -oP '^\d+' | head -1) | head -25
sudo kill %1
```

Compare with `bpftrace` — the one-liner equivalent:

```sh
sudo bpftrace -e '
tracepoint:sock:inet_sock_set_state
/args->protocol == 6 && args->newstate == 1/
{
	@start[args->skaddr] = nsecs;
}
tracepoint:sock:inet_sock_set_state
/args->protocol == 6 && args->newstate == 7 && @start[args->skaddr]/
{
	@duration_ms = hist((nsecs - @start[args->skaddr]) / 1000000);
	delete(@start[args->skaddr]);
}
interval:s:15 { print(@duration_ms); exit(); }' &
for i in $(seq 1 20); do (nc -z 127.0.0.1 22 2>/dev/null &); done
wait
```

**`bpftrace` for exploration, `libbpf` + CO-RE for production.** That is the practical division.

Cleanup:

```sh
sudo rm -rf /sys/fs/bpf/* 2>/dev/null
make clean 2>/dev/null
```

---

## 3. Mastery drills

1. State eBPF's core guarantee precisely. Then, for each of the four restrictions in §T.2, explain which part of the guarantee it enables.

2. Explain the difference between ranges and tnums, and construct a value whose constraint one captures and the other cannot.

3. `find_good_pkt_pointers` propagates a range to every register with the same `id`. Construct the code where this matters, and explain what would break without the id.

4. Explain why bounded loops were hard. Then state what `bpf_loop` moves, and to whom.

5. Compute the per-lookup cost of `ARRAY`, `PERCPU_ARRAY`, `HASH`, and `PERCPU_HASH` at 10 Mpps with one lookup per packet. Then state the cache-coherency reason for the difference.

6. `SK_STORAGE` attaches to a socket's lifetime. Write the equivalent using a plain `HASH` keyed by socket pointer, and enumerate everything that goes wrong.

7. Explain the program-type system as a capability model. For three program types, name a helper each may call that the others may not, and explain why.

8. `fentry` is faster than `kprobe`. Explain the mechanism of each, and compute the difference in terms of what the CPU does.

9. Describe the kernel-version portability problem and BCC's solution. Then explain CO-RE's, step by step, and state exactly what happens at load time.

10. Helpers are stable UAPI; kfuncs are not. Argue both positions, citing Ch. 24 §T.1, and predict which will prove correct.

11. `KF_ACQUIRE`/`KF_RELEASE` reference tracking is a linear-type property. State the invariant the verifier proves, and construct the program it must reject.

12. Unprivileged BPF is disabled by default because of Spectre. Explain why the verifier's architectural proof is insufficient, and evaluate each of §T.10's five mitigations.

13. You have written a program the verifier rejects with "R1 unbounded memory access, use 'var &= const' or 'if (var < const)'". Explain what the verifier knows, what it needs, and three ways to give it that.

---

## 4. Further reading

**Kernel documentation**

- `Documentation/bpf/verifier.rst` ★★★ — **§T.3, by the verifier's authors.** Register states, pruning, and the reasoning. Read it before `verifier.c`.
- `Documentation/bpf/btf.rst` ★★★ — §T.8's format, completely.
- `Documentation/bpf/libbpf/` ★★★ — the userspace library's API and conventions.
- `Documentation/bpf/map_*.rst` ★★★ — one document per map type; §T.5.
- `Documentation/bpf/prog_*.rst` — per program type.
- `Documentation/bpf/kfuncs.rst` ★★★ — §T.9.
- `Documentation/bpf/bpf_design_QA.rst` ★★★ — **the design rationale as Q&A.** Short and unusually candid about what eBPF will and will not do.
- `Documentation/bpf/instruction-set.rst` ★★★ — §T.2, now also an IETF standard.
- `include/uapi/linux/bpf.h` ★★★ — **every helper, with full documentation in the comments.** The authoritative helper reference.
- `man 2 bpf`, `man 7 bpf-helpers` ★★★

**Books**

- Gregg, *BPF Performance Tools* ★★★ — **the practical book.** 150+ tools, the methodology, and the BCC/bpftrace ecosystem. If you write one eBPF book on your shelf, this is it.
- Calavera & Fontana, *Linux Observability with BPF* — shorter, more introductory.
- Rice, *Learning eBPF* ★★★ — the best current introduction to the mechanism (as opposed to the tools).
- Gregg, *Systems Performance*, 2nd ed. — the broader methodology eBPF serves.

**Papers**

- McCanne & Jacobson, "The BSD Packet Filter: A New Architecture for User-level Packet Capture," USENIX 1993 ★★★ — **the origin.** Classic BPF, and the verification idea in embryo.
- Høiland-Jørgensen et al., "The eXpress Data Path," CoNEXT 2018 ★★★ — Ch. 74, but the BPF argument is here too.
- Gershuni et al., "Simple and Precise Static Analysis of Untrusted Linux Kernel Extensions," PLDI 2019 ★★★ — **PREVAIL**, an alternative verifier based on abstract interpretation with better precision. Essential for understanding what the kernel verifier does and does not prove.
- Vishwanathan et al., "Verifying the Verifier: eBPF Range Analysis Verification," CAV 2023 ★★★ — formally verifying the range-tracking logic. Found real bugs.
- Nelson et al., "Specification and verification in the field: Applying formal methods to BPF just-in-time compilers in the Linux kernel," OSDI 2020 ★★★ — **Jitterbug**, verifying the JITs.
- Jia et al., "Programmable System Call Security with eBPF," 2023
- Bhat & Shacham, "Formal Verification of the Linux Kernel eBPF Verifier Range Analysis," 2022
- Kogias & Bugnion, "Flexible Packet Scheduling with eBPF" and related sched_ext work.

**Project documentation**

- **ebpf.io** ★★★ — the landing point; the "What is eBPF?" guide is excellent.
- **The BPF and XDP Reference Guide** (Cilium docs) ★★★ — **the most complete practical reference in existence.** Covers the toolchain, every map type, every program type, and the debugging workflow.
- **libbpf-bootstrap** (`github.com/libbpf/libbpf-bootstrap`) ★★★ — **the template for new projects.** Start every libbpf application from here.
- **BCC** (`github.com/iovisor/bcc`) ★★★ — 150+ ready tools in `tools/`; read them as worked examples even if you use libbpf.
- **bpftrace** (`github.com/bpftrace/bpftrace`) ★★★ — the reference guide and the `tools/` directory.
- **BTFHub** (`github.com/aquasecurity/btfhub`) — §T.8 for old kernels.
- **sched_ext** (`github.com/sched-ext/scx`) ★★★ — §T.7's most ambitious user, with working schedulers.

**LWN**

- "A thorough introduction to eBPF" ★★★
- "BPF: the universal in-kernel virtual machine" (2014) — the original coverage
- "Bounded loops in BPF" ★★★ — §T.4
- "BPF CO-RE" and "Building BPF applications with libbpf" ★★★
- "BPF trampolines and fentry/fexit" ★★★
- "BPF struct_ops" and "BPF-based congestion control" ★★★ — §T.7
- "The BPF LSM" ★★★
- "Unprivileged BPF and Spectre" ★★★ — §T.10's honest account
- "Sleepable BPF programs"
- "BPF kfuncs and the stability question" ★★★ — §T.9's debate
- "Extending scheduling with BPF" (sched_ext) ★★★
- "The BPF ring buffer"
- The annual LSFMM+BPF summaries ★★★

**Source reading order**

1. `Documentation/bpf/verifier.rst` and `bpf_design_QA.rst`. **Do not start with `verifier.c`.**
2. `include/uapi/linux/bpf.h` ★★★ — the helpers and their documentation; a reference you will return to constantly.
3. `kernel/bpf/tnum.c` ★★★ — **~200 lines; read all of it.** §T.3's precision mechanism, in miniature.
4. `include/linux/bpf_verifier.h` ★★★ — `bpf_reg_state`, `bpf_verifier_state`, `bpf_func_state`.
5. `kernel/bpf/verifier.c`: `do_check` ★★★, `check_mem_access`, `check_cond_jmp_op`, `reg_set_min_max`, `is_state_visited`. **Budget real time; it is ~20,000 lines and the densest code in this book.**
6. `kernel/bpf/arraymap.c` and `hashtab.c` — §T.5; small and clear.
7. `kernel/bpf/ringbuf.c` — the reservation protocol.
8. `kernel/bpf/trampoline.c` ★★★ — §T.6's fentry/fexit; the code generation is elegant.
9. `kernel/bpf/bpf_struct_ops.c` — §T.7.
10. `tools/lib/bpf/relo_core.c` and `libbpf.c` ★★★ — §T.8's CO-RE, in userspace.
11. `arch/x86/net/bpf_jit_comp.c` — the JIT; read `do_jit` and the prologue/epilogue generation.

**Tools**

- `bpftool` ★★★ — **learn it thoroughly.** `prog show/dump xlated/dump jited/profile`, `map show/dump/update`, `btf dump`, `link show`, `feature probe`, `gen skeleton`, `net show`, `cgroup tree`, `struct_ops show`
- `libbpf` + **libbpf-bootstrap** ★★★ — the production workflow
- `bpftrace` ★★★ — for exploration; `bpftrace -l` to list probes, `-v` to see the generated program
- `BCC`'s `tools/` ★★★ — 150+ examples, many production-quality
- `clang -target bpf -g` and `llvm-objdump -d` ★★★
- `/sys/kernel/debug/tracing/trace_pipe` for `bpf_printk`
- `kernel.bpf_stats_enabled=1` ★★★ — per-program overhead measurement
- `bpftool prog dump xlated` ★★★ — **compare against your source to see what the verifier did**
- `veristat` (in `tools/testing/selftests/bpf/`) — verifier statistics across a corpus; useful for checking that a change did not make verification harder
- `vmtest`/`ci` from the BPF selftests — for testing across kernel versions

---

→ Next: [76-io-uring.md](76-io-uring.md)

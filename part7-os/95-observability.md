# Chapter 95 — Observability: perf, ftrace, bpftrace, Crash Dumps

> Ch. 06 set up a debugging environment and taught you to read an oops. This chapter is the
> production counterpart: the tools you use on a machine you cannot reboot, cannot attach a
> debugger to, and cannot reproduce the problem on. It is the toolbox behind
> `reference/debugging-scenarios.md`, laid out systematically.

---

## Theory & First Principles

### T.0 — Start here: the bug only happens in production

It reproduces once a day, on one machine out of four hundred, under real load. You cannot
attach a debugger — stopping the process changes the timing and takes down a production
service. You cannot add printfs — that requires a rebuild and a deploy, and you would need
twenty iterations. **`printf` debugging and `gdb` are both unavailable. What is left?**

**This is the question observability exists to answer**, and Linux gives you four
fundamentally different instruments. **Knowing which to reach for is most of the skill:**

```
  +-------------------+------------------+-------------------+------------------+
  |  COUNTERS         |  SAMPLING        |  TRACING          |  DYNAMIC PROBES  |
  |  /proc, /sys      |  perf record     |  ftrace, tracepts |  kprobe, uprobe  |
  +-------------------+------------------+-------------------+------------------+
  | "how many?"       | "where is time   | "what happened,   | "what is the     |
  |                   |  going?"         |  in what order?"  |  value of x?"    |
  +-------------------+------------------+-------------------+------------------+
  | cost: ~0          | cost: ~1%        | cost: 1-10%       | cost: varies     |
  | always on         | statistical      | every event       | you choose       |
  | no context        | misses rare      | can flood         | anywhere, even   |
  |                   | events           |                   | unexported fns   |
  +-------------------+------------------+-------------------+------------------+
```

**The decision procedure is simple and worth memorizing:**

| If you are asking | Use |
|---|---|
| "is something wrong at all?" | counters, and **PSI** |
| "which function is burning CPU?" | `perf record` → a flame graph |
| "why is this *latency* high?" (not CPU) | **off-CPU** analysis — `offcputime`, scheduler tracepoints |
| "what sequence of events led here?" | `ftrace` / tracepoints |
| "what was the argument to this specific call?" | `bpftrace` with a kprobe |

**The on-CPU / off-CPU split is the insight people are missing when they get stuck.** A flame
graph shows where CPU time goes — and a slow request that is *waiting* (on a lock, on I/O, on
a dependency) appears nowhere in it, because it is consuming no CPU at all. **Most latency
problems in real systems are off-CPU problems, and the most popular tool cannot see them.**

**Second: the observer effect is real and must be budgeted.** A tracepoint firing 10 million
times a second at 100 ns each is a full core. The kernel's answer is a layered one, and it is
good design worth noticing:

```
  static tracepoints  -> compiled in, a NOP (5 bytes) when disabled,
                         patched to a call when enabled.  Cost when off: ZERO.
  kprobes             -> patch an INT3/branch at runtime, anywhere. Higher cost.
  eBPF                -> aggregate IN THE KERNEL (a histogram in a map) and only
                         ship the summary to userspace.  <- the important one.
```

**That last line is the modern change.** `strace` copies every event to userspace and slows
the target 10–100×. `bpftrace` builds the histogram in kernel context and hands you the
finished answer. **Move the computation to the data instead of the data to the computation**
— the same reasoning as XDP (Ch. 74 §T.0) and `io_uring` (Ch. 76), applied to measurement.

**Third, the methodology, because tools without a method produce noise.** Two worth knowing by
name:

- **USE** (Gregg): for every resource, check **U**tilization, **S**aturation, **E**rrors. A
  checklist that finds bottlenecks systematically rather than by hunch.
- **PSI** (`/proc/pressure/{cpu,io,memory}`): the single best-designed metric Linux has
  added in a decade. It reports *how much time tasks were stalled waiting for a resource* —
  which is the thing you actually care about, rather than a proxy like utilization. **The
  same reasoning as CoDel measuring sojourn time instead of queue length (Ch. 73 §T.0):
  measure the harm, not a correlate of it.**

```bash
cat /proc/pressure/{cpu,io,memory}
sudo perf record -F 99 -a -g -- sleep 30 && sudo perf script | stackcollapse-perf.pl | flamegraph.pl > f.svg
sudo /usr/share/bcc/tools/offcputime -df 30 > off.stacks
sudo bpftrace -e 'kprobe:vfs_read { @[comm] = hist(arg2); }'
sudo trace-cmd record -p function_graph -g vfs_read -- sleep 2 && trace-cmd report | head -40
```

---

### T.1 — Four ways to observe, and their costs

| Method | Mechanism | Overhead when idle | Overhead when active |
|---|---|---|---|
| **Counters** | A variable incremented in the code | one add | one add |
| **Tracing** | Record an event when it happens | **zero** (static key = NOP) | 100–500 ns/event |
| **Sampling** | Interrupt periodically, record the state | zero | ~1% at 99 Hz |
| **Instrumenting** | Dynamically patch code to call a probe | zero | 1–5 µs/probe (kprobe), less with fprobe |

**The choice follows from what you are asking:**

- *"How often / how much?"* → counters.
- *"What is happening, exactly?"* → tracing.
- *"Where is time going?"* → sampling.
- *"What is this specific function doing?"* → instrumenting.

The single most important property: **tracepoints and static keys cost literally nothing when
disabled.** The instruction stream contains a `NOP` that is patched to a `JMP` when the
tracepoint is enabled (`CONFIG_JUMP_LABEL`, Ch. 06). This is why you can ship a kernel with
thousands of tracepoints and enable them selectively in production.

### T.2 — On-CPU versus off-CPU: the fork in the road

The single most useful diagnostic distinction, and the one that most often sends people down
the wrong path.

```
                   Is CPU utilization high?
                     /               \
                  YES                 NO
                   │                   │
              ON-CPU analysis     OFF-CPU analysis
              perf record -g      offcputime, wakeuptime
              flame graph         blocked-time tracing
              "who is burning     "what are we WAITING for?"
               cycles?"
```

If the system is slow and the CPU is idle, **profiling on-CPU tells you nothing** — you will
profile the idle loop. The time is going somewhere: a lock, disk I/O, a network round trip, a
page fault, a scheduler delay. Off-CPU analysis measures exactly that: the time between a
task going to sleep and being woken, with the stack that put it to sleep.

The complete picture requires both, plus the wakeup chain: *A is blocked → B was going to
wake it → B is blocked on C*. `wakeuptime` and `offwaketime` give you that chain.

### T.3 — perf: sampling and counting

`perf` does two different things and it is worth keeping them separate in your head.

**Counting** — read hardware and software counters over an interval:

```bash
perf stat -a -- sleep 10
perf stat -e cycles,instructions,cache-misses,branch-misses ./prog
perf stat -e 'syscalls:sys_enter_*' -a -- sleep 5
```

The derived metrics that matter:

| Metric | Meaning | Bad when |
|---|---|---|
| **IPC** (instructions/cycle) | How well the pipeline is filled | < 1.0 → stalled, probably on memory |
| **cache-misses / instructions** | Memory behaviour | high → working set exceeds cache, or false sharing (Ch. 105) |
| **branch-misses / branches** | Predictability | > 5% → data-dependent branching |
| **stalled-cycles-frontend/backend** | Where the stall is | frontend = I-cache/decode; backend = memory/execution |

**Sampling** — interrupt at a frequency or every N events, record the stack:

```bash
perf record -F 99 -a -g -- sleep 30       # 99 Hz, all CPUs, call graphs
perf record -e cycles:pp -g ./prog        # precise sampling (PEBS/IBS)
perf record -e cache-misses -c 10000 -g   # every 10,000 misses
perf report --stdio --sort=comm,dso,symbol
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

**Why 99 Hz and not 100:** at 100 Hz you risk locking in phase with periodic kernel work at
100 Hz or 1000 Hz and systematically over- or under-sampling it. A prime-ish offset
decorrelates.

**Call-graph methods**, which matter more than people realize:

| Method | Works with | Cost | Accuracy |
|---|---|---|---|
| `--call-graph fp` | `-fno-omit-frame-pointer` builds | cheapest | fails on optimized code without FP |
| `--call-graph dwarf` | anything with DWARF | copies stack, large data | accurate, expensive |
| `--call-graph lbr` | Intel LBR | cheap | limited depth (16–32) |

Distributions have historically built with `-fomit-frame-pointer`, which breaks `fp` stacks —
this is why Fedora and Ubuntu re-enabled frame pointers in 2023–24 after a long debate about
the ~1% performance cost versus the profiling benefit. Know that story; it is a good example
of an engineering trade made explicitly.

### T.4 — ftrace: the built-in tracer

ftrace is compiled into nearly every kernel and needs no tools beyond `cat` and `echo` —
which makes it the tool of last resort on a locked-down or minimal system.

```bash
cd /sys/kernel/debug/tracing        # or /sys/kernel/tracing

cat available_tracers
#   function function_graph wakeup wakeup_rt irqsoff preemptoff
#   preemptirqsoff osnoise timerlat hwlat blk mmiotrace nop
```

| Tracer | Answers |
|---|---|
| `function` | Which functions are being called? |
| `function_graph` | Call graph with **per-function duration** |
| `irqsoff` / `preemptoff` / `preemptirqsoff` | Longest non-preemptible region (Ch. 103) |
| `wakeup` / `wakeup_rt` | Longest scheduler wakeup latency |
| `osnoise` / `timerlat` | What is stealing CPU, attributed by source (Ch. 103 §T.9) |
| `hwlat` | SMI/firmware stalls — invisible to everything else |
| `blk` | Block layer, via `blktrace` |

**Tracepoints** are the other half — static, stable-ish instrumentation points:

```bash
ls events/                     # ~30 subsystems
ls events/sched/               # sched_switch, sched_wakeup, sched_migrate_task, ...
echo 1 > events/sched/sched_switch/enable
cat trace_pipe

# Filters are evaluated IN THE KERNEL -- much cheaper than filtering output
echo 'prev_comm == "nginx"' > events/sched/sched_switch/filter

# Triggers: act on an event
echo 'hist:key=comm:val=bytes' > events/block/block_rq_issue/trigger
cat events/block/block_rq_issue/hist

# Synthetic events: correlate two tracepoints into one
echo 'iolat u64 lat; char comm[16]' > synthetic_events
echo 'hist:keys=dev,sector:ts=common_timestamp.usecs' \
	> events/block/block_rq_issue/trigger
echo 'hist:keys=dev,sector:lat=common_timestamp.usecs-$ts:onmatch(block.block_rq_issue).trace(iolat,$lat,comm)' \
	> events/block/block_rq_complete/trigger
```

**In-kernel histogram triggers are underused.** They aggregate in the kernel with no
per-event userspace cost, which makes them usable at rates where `trace_pipe` would drown.

`trace-cmd` is the friendly front end; `kernelshark` visualizes the result.

### T.5 — eBPF: programmable observability

The modern answer (Ch. 75): attach a verified program to a tracepoint, kprobe, uprobe, or
perf event, aggregate **in the kernel**, and read summarized results.

**Why it wins over ftrace for most tasks:**

| | ftrace | eBPF |
|---|---|---|
| Aggregation | in userspace (except hist triggers) | **in the kernel** |
| Per-event cost | write to a ring buffer | run a small program, update a map |
| Arbitrary logic | no | yes |
| Cross-event correlation | synthetic events | maps, naturally |
| Userspace probes | no | yes (uprobes/USDT) |

```bash
# The one-liner that answers most questions:
bpftrace -e 'tracepoint:syscalls:sys_enter_* { @[probe] = count(); }'

# Latency distribution of a kernel function:
bpftrace -e '
kprobe:vfs_read  { @s[tid] = nsecs; }
kretprobe:vfs_read /@s[tid]/ { @us = hist((nsecs - @s[tid])/1000); delete(@s[tid]); }'

# Who is doing the I/O?
bpftrace -e 'tracepoint:block:block_rq_issue {
	@[comm, args->rwbs] = sum(args->bytes/1024); }'

# Off-CPU, with stacks:
bpftrace -e '
kprobe:finish_task_switch { @start[arg0] = nsecs; }
kprobe:try_to_wake_up /@start[arg0]/ {
	@blocked[kstack] = sum(nsecs - @start[arg0]); delete(@start[arg0]); }'
```

**Attachment points, in order of preference:**

| Type | Stability | Cost |
|---|---|---|
| **Tracepoints** | stable-ish; a documented interface | cheapest |
| **kfuncs / fentry-fexit** | needs BTF; direct attachment | very cheap |
| **kprobes** | any function; **breaks on rename/inline** | ~1–2 µs |
| **uprobes** | userspace functions | expensive (a trap per hit) |
| **USDT** | userspace static tracepoints | cheap |

**Prefer tracepoints.** A kprobe on a static function is a dependency on an implementation
detail, and it will break on the next kernel — this is Hyrum's Law (Ch. 24) applied to
observability.

**CO-RE and BTF** are what made eBPF deployable: BPF Type Format describes kernel struct
layouts, so a compiled BPF program can relocate field offsets at load time and run on a
different kernel than it was built against. Before CO-RE, every BPF tool compiled against
kernel headers on the target — which is why bcc needed clang and headers on production hosts,
and libbpf-based tools do not.

### T.6 — Crash dumps

When the machine is dead, or when you want a frozen snapshot to analyse offline.

```
 kernel panics
   │
 kdump: kexec into a PRE-LOADED crash kernel
   │      (reserved memory, crashkernel=256M, already in RAM --
   │       so it does not depend on the broken kernel's allocator)
   │
 the crash kernel boots and dumps the OLD kernel's memory
   │      /proc/vmcore is the old kernel's RAM
   │
 makedumpfile compresses and filters it
   │      (drop free pages, zero pages, user data -> often 100x smaller)
   ▼
 vmcore on disk, or over the network
```

```bash
# Setup
sudo apt install kdump-tools crash        # or kexec-tools
# cmdline: crashkernel=256M
sudo systemctl enable --now kdump-tools
cat /sys/kernel/kexec_crash_loaded        # 1 = ready

# Force a dump to test it -- TEST IT, or you will not have one when you need it
echo c | sudo tee /proc/sysrq-trigger
```

**Analysis: `crash` versus `drgn`.**

`crash` is the traditional tool: a gdb-based shell with kernel-aware commands.

```
crash /usr/lib/debug/boot/vmlinux-$(uname -r) /var/crash/vmcore

crash> bt                    # backtrace of the panicking task
crash> bt -a                 # all CPUs
crash> ps                    # all tasks
crash> log                   # the kernel log
crash> kmem -i               # memory usage
crash> kmem -s               # slab
crash> foreach bt            # every task's stack
crash> struct task_struct ffff88...
crash> dis -l <address>
crash> mod -s mymodule       # load module symbols
```

**`drgn` is better for most work**, and is the tool to learn:

```python
# drgn -c /var/crash/vmcore /usr/lib/debug/.../vmlinux
# or LIVE: sudo drgn
from drgn import *
from drgn.helpers.linux import *

# Every task, with state
for task in for_each_task(prog):
    print(task.pid.value_(), task.comm.string_().decode(),
          task_state_to_char(task))

# Every mount
for mnt in for_each_mount(prog):
    print(mount_dst(mnt).decode())

# Walk a list -- the thing that makes drgn worth it
sb = prog['super_blocks']
for s in list_for_each_entry('struct super_block', sb.address_of_(), 's_list'):
    print(s.s_id.string_().decode(), s.s_type.name.string_().decode())

# Arbitrary Python: find all tasks blocked in a given function
for task in for_each_task(prog):
    if task.__state.value_() == 2:        # TASK_UNINTERRUPTIBLE
        print(task.pid.value_(), task.comm.string_().decode())
```

**Why `drgn` wins:** it is Python, so you can script an analysis, loop over structures,
compute, and correlate. `crash` gives you fixed commands; `drgn` gives you a programming
language with kernel type awareness. **And it works on a live kernel**, so you can use the
same scripts for debugging and for production inspection.

### T.7 — The USE method, and a procedure

Brendan Gregg's USE method, which is the checklist that prevents you from guessing:

> For every resource: check **U**tilization, **S**aturation, and **E**rrors.

| Resource | Utilization | Saturation | Errors |
|---|---|---|---|
| CPU | `mpstat`, `%usr+%sys` | run queue length, PSI `cpu` | `perf stat` machine checks |
| Memory | `free`, `MemAvailable` | swapping, PSI `memory`, `allocstall` | OOM kills, `dmesg` |
| Disk | `iostat %util` | `aqu-sz`, `await`, PSI `io` | `dmesg` I/O errors, SMART |
| Network | `sar -n DEV` | `ss -ti` retransmits, drops | `ip -s link`, `netstat -s` |
| **Locks** | `lock_stat` hold time | contention count | — |
| **Tags/queues** | `hctx*/tags` | full events | timeouts |

**The second discipline: separate queueing from service time.** If `await` is high but
`svctm` is low, you are queued — the fix is concurrency or admission control. If service time
itself is high, the fix is the device or the work. Conflating them leads to the wrong fix.

**And always form a hypothesis with a predicted measurement before changing anything.**
"If this is cgroup CPU throttling, `nr_throttled` will be nonzero and removing `cpu.max` will
eliminate the tail." Then test exactly that. A change that helps for an unknown reason has
taught you nothing.

### T.8 — Continuous profiling

The modern practice: profile production **always**, at low overhead, and keep the history.

```
 target hosts                  storage/UI
 ┌──────────────┐             ┌──────────────┐
 │ perf / eBPF  │──samples──► │ Parca /      │
 │ agent @ 19Hz │             │ Pyroscope /  │
 │  ~0.5% CPU   │             │ Polar Signals│
 └──────────────┘             └──────────────┘
```

The value is **differential**: when latency regresses at 03:00, you compare this hour's flame
graph with last week's and see exactly which stack grew. Without history you are reduced to
reproducing the problem, which for a 1-in-1000 event you cannot do
(`reference/debugging-scenarios.md` §9).

The requirements that make it viable: frame pointers or LBR for cheap stacks, symbolization
(build IDs plus a symbol server), and low enough overhead that nobody argues about it (19–99
Hz sampling is under 1%).

### T.9 — Choosing the tool

| Question | Tool |
|---|---|
| Why is the CPU busy? | `perf record -g` + flame graph |
| Why is it slow with an idle CPU? | `offcputime`, PSI |
| What syscalls, how often? | `perf trace`, `bpftrace` on `raw_syscalls:sys_enter` |
| Latency distribution of X? | `funclatency`, `bpftrace hist()` |
| Which cacheline is contended? | `perf c2c` (Ch. 105) |
| Lock contention? | `perf lock`, `/proc/lock_stat` |
| Longest IRQ-disabled region? | `irqsoff` tracer |
| What is the kernel doing *right now*? | `echo l > sysrq-trigger`, `drgn` live |
| Why did it panic? | kdump + `crash`/`drgn` |
| Block I/O detail? | `blktrace`, `biosnoop`, `biolatency` |
| Network? | `tcpdump`, `ss -ti`, `tcplife`, `tcpretrans` |
| Memory growth? | `/proc/allocinfo`, `kmemleak`, `slabtop` |
| Everything, historically | continuous profiling |

### T.10 — Observability under lockdown

The tension from Ch. 102 §T.8, made concrete. With `lockdown=confidentiality`:

| Blocked | Alternative |
|---|---|
| kprobes, most BPF tracing | tracepoints (some still work), userspace-only tracing |
| `/proc/kcore`, `/dev/mem` | — |
| `perf` kernel sampling | `perf` userspace-only (`:u` modifier) |
| kgdb | — |
| Reading kernel addresses (`kptr_restrict`) | build-ID-based symbolization offline |
| kdump (unsigned kexec) | signed crash kernel |

**The practical consequence: you must decide the observability posture at design time, not
during an incident.** A locked-down fleet needs: signed kexec for kdump, tracepoint-based
(not kprobe-based) instrumentation shipped in the product, counters exported through a stable
interface, and a way to enable a debug kernel on a subset of hosts. Designing that is a real
architect deliverable, and it is a good system-design interview question.

---

## 1. Internals

### Source map

| Path | Contents |
|---|---|
| `kernel/trace/` | ftrace core, tracepoints, ring buffer |
| `kernel/trace/trace_events.c` | Tracepoint event machinery |
| `kernel/trace/ring_buffer.c` | The lockless per-CPU ring buffer |
| `kernel/trace/trace_functions_graph.c` | The function graph tracer |
| `kernel/trace/trace_osnoise.c` | `osnoise`/`timerlat` |
| `kernel/events/core.c` | The perf subsystem |
| `kernel/bpf/` | BPF core, verifier, maps |
| `kernel/kexec*.c`, `kernel/crash_core.c` | kdump |
| `include/trace/events/` | **Every tracepoint definition** — browse this |
| `Documentation/trace/` | ftrace, kprobes, uprobes, histogram triggers |
| `tools/perf/` | The perf tool itself |
| `tools/tracing/rtla/` | `rtla` |

### The interfaces

```bash
/sys/kernel/tracing/            # ftrace (debugfs symlink: /sys/kernel/debug/tracing)
  current_tracer                # which tracer
  trace, trace_pipe             # output
  events/*/*/{enable,filter,trigger,hist,format}
  set_ftrace_filter             # which functions to trace
  tracing_max_latency           # for the latency tracers
  available_events, available_filter_functions

/sys/kernel/debug/kprobes/list
/sys/kernel/debug/tracing/kprobe_events   # dynamic kprobes via text
/proc/sys/kernel/perf_event_paranoid      # 3=nothing, 2=user only, 1=+kernel, -1=all
/proc/sys/kernel/kptr_restrict
/proc/lock_stat                           # CONFIG_LOCK_STAT
/proc/pressure/{cpu,memory,io}
/proc/allocinfo                           # CONFIG_MEM_ALLOC_PROFILING (6.10+)
/sys/kernel/btf/vmlinux                   # BTF, for CO-RE
```

### A tracepoint, from definition to use

```c
/* include/trace/events/block.h */
TRACE_EVENT(block_rq_issue,
	TP_PROTO(struct request *rq),
	TP_ARGS(rq),
	TP_STRUCT__entry(
		__field(  dev_t,  dev      )
		__field(  sector_t, sector )
		__field(  unsigned int, nr_sector )
		__array(  char,   rwbs, RWBS_LEN )
		__string( cmd,    blk_rq_is_passthrough(rq) ? "PC" : "FS" )
		__array(  char,   comm, TASK_COMM_LEN )
	),
	TP_fast_assign(
		__entry->dev = rq->q->disk ? disk_devt(rq->q->disk) : 0;
		...
	),
	TP_printk("%d,%d %s %u (%s) %llu + %u [%s]", ...)
);
```

```c
/* block/blk-mq.c -- the call site */
trace_block_rq_issue(rq);
/* Expands to:
 *   if (static_branch_unlikely(&__tracepoint_block_rq_issue.key))
 *       __traceiter_block_rq_issue(NULL, rq);
 * The static branch is a patched NOP when disabled. ZERO cost. */
```

That zero-cost-when-disabled property is why the kernel can carry thousands of tracepoints,
and it is the mechanism you should copy when instrumenting your own subsystem (Ch. 89, the
observability design problem in `reference/system-design.md` §10).

---

## 2. Practice

### Lab 95.1 — Build the complete toolkit

```bash
#!/bin/bash
# observability_setup.sh
set -e

echo "=== Packages ==="
sudo apt install -y \
	linux-tools-common linux-tools-$(uname -r) \
	trace-cmd kernelshark \
	bpfcc-tools bpftrace libbpf-dev \
	crash kexec-tools makedumpfile \
	sysstat iotop htop \
	blktrace fio \
	python3-drgn drgn

echo "=== Kernel config required ==="
for opt in CONFIG_FTRACE CONFIG_FUNCTION_TRACER CONFIG_FUNCTION_GRAPH_TRACER \
           CONFIG_DYNAMIC_FTRACE CONFIG_PERF_EVENTS CONFIG_BPF_SYSCALL \
           CONFIG_BPF_JIT CONFIG_DEBUG_INFO_BTF CONFIG_KPROBES CONFIG_UPROBES \
           CONFIG_DEBUG_FS CONFIG_KEXEC CONFIG_CRASH_DUMP CONFIG_PSI \
           CONFIG_LOCK_STAT CONFIG_SCHEDSTATS; do
	printf '%-36s ' "$opt"
	zgrep -qx "$opt=y" /boot/config-$(uname -r) /proc/config.gz 2>/dev/null \
		&& echo ENABLED || echo "--"
done

echo "=== Flame graph tooling ==="
[ -d ~/FlameGraph ] || git clone --depth 1 \
	https://github.com/brendangregg/FlameGraph ~/FlameGraph
echo 'export PATH=$PATH:~/FlameGraph' >> ~/.bashrc

echo "=== Debug symbols (needed for crash/drgn) ==="
# Ubuntu: enable the ddebs repo, then:
#   sudo apt install linux-image-$(uname -r)-dbgsym
# Fedora:
#   sudo dnf debuginfo-install kernel-$(uname -r)
ls /usr/lib/debug/boot/vmlinux-$(uname -r) 2>/dev/null \
	&& echo "vmlinux with symbols: present" || echo "vmlinux: MISSING"

echo "=== Permissions ==="
sudo sysctl -w kernel.perf_event_paranoid=-1
sudo sysctl -w kernel.kptr_restrict=0
#   NOTE: these lower security. Do this on a dev machine, and understand
#   the trade before doing it in production (T.10).

echo "=== Verify each tool works ==="
sudo perf stat -a -- sleep 1 2>&1 | tail -3
sudo bpftrace -e 'BEGIN { printf("bpftrace ok\n"); exit(); }'
sudo trace-cmd record -e sched:sched_switch -- sleep 1 >/dev/null 2>&1 && \
	echo "trace-cmd ok" && rm -f trace.dat
sudo drgn -c /proc/kcore /usr/lib/debug/boot/vmlinux-$(uname -r) \
	-e 'print(prog["init_task"].comm.string_())' 2>/dev/null \
	&& echo "drgn ok" || echo "drgn: needs vmlinux with debug info"
```

### Lab 95.2 — Flame graphs, on and off CPU

```bash
# --- ON-CPU ---
sudo perf record -F 99 -a -g -- sleep 30
sudo perf script > /tmp/out.perf
~/FlameGraph/stackcollapse-perf.pl /tmp/out.perf > /tmp/out.folded
~/FlameGraph/flamegraph.pl /tmp/out.folded > /tmp/oncpu.svg

# Kernel only / user only:
sudo perf record -F 99 -a -g -e cycles:k -- sleep 30
sudo perf record -F 99 -a -g -e cycles:u -- sleep 30

# --- OFF-CPU (the one people forget) ---
sudo /usr/share/bcc/tools/offcputime -df -p $(pgrep -n myapp) 30 > /tmp/off.folded
~/FlameGraph/flamegraph.pl --color=io --title="Off-CPU" \
	/tmp/off.folded > /tmp/offcpu.svg

# --- The wakeup chain: WHO woke the blocked thread? ---
sudo /usr/share/bcc/tools/offwaketime -df -p $(pgrep -n myapp) 30 > /tmp/wake.folded
~/FlameGraph/flamegraph.pl --color=chain /tmp/wake.folded > /tmp/chain.svg

# --- DIFFERENTIAL: the most valuable form ---
# Capture a baseline, make a change, capture again:
~/FlameGraph/difffolded.pl /tmp/before.folded /tmp/after.folded \
	| ~/FlameGraph/flamegraph.pl --title="Regression" > /tmp/diff.svg
#   Red = grew, blue = shrank. This finds regressions in seconds.
```

**Read the flame graph correctly:** width is time (samples), not duration of one call;
the y-axis is stack depth, not time; and **the x-axis ordering is alphabetical, not
chronological** — a common misreading.

### Lab 95.3 — ftrace without any tools

The lab that matters on a minimal or locked-down system.

```bash
cd /sys/kernel/debug/tracing
sudo -i        # everything here needs root

# --- 1. What functions does opening a file call? ---
echo 0 > tracing_on
echo function_graph > current_tracer
echo do_sys_openat2 > set_graph_function
echo 1 > tracing_on
cat /etc/hostname > /dev/null
echo 0 > tracing_on
head -60 trace
#   -> a call graph WITH DURATIONS. The "+" and "!" markers flag
#      functions over 10us and 100us.

# --- 2. Trace one process only ---
echo > set_graph_function
echo $$ > set_ftrace_pid
echo function > current_tracer
echo 1 > tracing_on; ls /tmp > /dev/null; echo 0 > tracing_on
wc -l trace

# --- 3. Longest IRQ-disabled region (Ch. 103) ---
echo > set_ftrace_pid
echo irqsoff > current_tracer
echo 0 > tracing_max_latency
echo 1 > tracing_on
sleep 30
echo 0 > tracing_on
cat tracing_max_latency          # microseconds
head -40 trace                   # the offending code path

# --- 4. Tracepoints with in-kernel filtering ---
echo nop > current_tracer
echo 1 > events/block/block_rq_issue/enable
echo 'bytes > 65536' > events/block/block_rq_issue/filter
echo 1 > tracing_on
dd if=/dev/zero of=/tmp/big bs=1M count=100 oflag=direct 2>/dev/null
echo 0 > tracing_on
cat trace | head -20

# --- 5. In-kernel histograms: aggregate with ZERO userspace cost ---
echo 'hist:key=comm:val=bytes:sort=bytes' > events/block/block_rq_issue/trigger
sleep 20
cat events/block/block_rq_issue/hist
#   A full per-process I/O byte histogram, computed in the kernel.

# --- 6. A dynamic kprobe with no tools at all ---
echo 'p:myprobe vfs_read file=%di count=%dx' > kprobe_events
echo 1 > events/kprobes/myprobe/enable
cat trace_pipe | head -10
echo 0 > events/kprobes/myprobe/enable
echo > kprobe_events

# Cleanup
echo nop > current_tracer; echo > set_ftrace_filter
echo 0 > events/enable; echo > trace
```

### Lab 95.4 — bpftrace, ten useful one-liners

```bash
# 1. Syscall counts by process
sudo bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'

# 2. Which syscalls, system-wide
sudo bpftrace -e 'tracepoint:raw_syscalls:sys_enter {
	@[args->id] = count(); }'

# 3. Read latency distribution
sudo bpftrace -e '
kprobe:vfs_read { @s[tid] = nsecs; }
kretprobe:vfs_read /@s[tid]/ {
	@us = hist((nsecs - @s[tid]) / 1000); delete(@s[tid]); }'

# 4. Block I/O size by process and direction
sudo bpftrace -e 'tracepoint:block:block_rq_issue {
	@KB[comm, args->rwbs] = hist(args->bytes / 1024); }'

# 5. TCP retransmits, with addresses
sudo bpftrace -e 'kprobe:tcp_retransmit_skb {
	@[ntop(((struct sock *)arg0)->__sk_common.skc_daddr)] = count(); }'

# 6. Page faults by process
sudo bpftrace -e 'software:page-faults:1 { @[comm] = count(); }'

# 7. Scheduler latency (runnable -> running)
sudo bpftrace -e '
tracepoint:sched:sched_wakeup { @q[args->pid] = nsecs; }
tracepoint:sched:sched_switch /@q[args->next_pid]/ {
	@us = hist((nsecs - @q[args->next_pid]) / 1000);
	delete(@q[args->next_pid]); }'

# 8. Who is calling a specific kernel function, with stacks
sudo bpftrace -e 'kprobe:shrink_node { @[kstack] = count(); }'

# 9. Off-CPU time by stack
sudo bpftrace -e '
kprobe:finish_task_switch { @s[arg0] = nsecs; }
kprobe:try_to_wake_up /@s[arg0]/ {
	@blocked_us[kstack] = sum((nsecs - @s[arg0]) / 1000);
	delete(@s[arg0]); }'

# 10. Slow file opens, with the filename
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_openat { @s[tid] = nsecs;
	@fn[tid] = str(args->filename); }
tracepoint:syscalls:sys_exit_openat /@s[tid]/ {
	$d = (nsecs - @s[tid]) / 1000;
	if ($d > 1000) { printf("%s %s %d us\n", comm, @fn[tid], $d); }
	delete(@s[tid]); delete(@fn[tid]); }'
```

**Write ten of your own for your subsystem.** The skill is knowing which tracepoint answers
your question, and that only comes from browsing `/sys/kernel/tracing/events/`.

### Lab 95.5 — Set up and use kdump

```bash
# --- Setup ---
sudo apt install kdump-tools crash linux-image-$(uname -r)-dbgsym

# Reserve memory on the cmdline: crashkernel=256M  (512M for large systems)
sudo sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="/&crashkernel=256M /' \
	/etc/default/grub
sudo update-grub && sudo reboot

# Verify
cat /proc/cmdline | grep -o 'crashkernel=[^ ]*'
cat /sys/kernel/kexec_crash_loaded      # must be 1
cat /sys/kernel/kexec_crash_size

# Configure dump filtering -- a 512GB machine's raw dump is 512GB
sudo tee -a /etc/default/kdump-tools <<'EOF'
MAKEDUMP_ARGS="-c -d 31"
#   -c   compress
#   -d31 = 1|2|4|8|16: drop zero, cache, private cache, user, and free pages.
#          Often 100x reduction.
EOF

# --- TEST IT. This is the step everyone skips. ---
echo c | sudo tee /proc/sysrq-trigger
# The machine panics, kexecs the crash kernel, dumps, and reboots.

ls -lh /var/crash/*/

# --- Analyse with crash ---
sudo crash /usr/lib/debug/boot/vmlinux-$(uname -r) /var/crash/*/dump.*
crash> bt
crash> log | tail -40
crash> ps | head -20
crash> kmem -i
crash> foreach UN bt        # every task in TASK_UNINTERRUPTIBLE
crash> quit

# --- Analyse with drgn (better) ---
sudo drgn -c /var/crash/*/dump.* /usr/lib/debug/boot/vmlinux-$(uname -r)
```

```python
# A drgn script: find everything blocked, and what it is blocked on.
from drgn import *
from drgn.helpers.linux import *

TASK_UNINTERRUPTIBLE = 2
blocked = {}
for task in for_each_task(prog):
    if task.__state.value_() & TASK_UNINTERRUPTIBLE:
        try:
            trace = prog.stack_trace(task)
            key = str(trace[0].name) if len(trace) else "?"
        except Exception:
            key = "?"
        blocked.setdefault(key, []).append(
            (task.pid.value_(), task.comm.string_().decode()))

for fn, tasks in sorted(blocked.items(), key=lambda kv: -len(kv[1])):
    print(f"{len(tasks):4d} tasks blocked in {fn}")
    for pid, comm in tasks[:3]:
        print(f"       {pid} {comm}")
```

### Lab 95.6 — Diagnose a synthetic production problem

Have a colleague introduce one of these, and diagnose it with the tools only.

```bash
# Fault A: cgroup CPU throttling
sudo systemd-run --scope -p CPUQuota=20% stress-ng --cpu 4 -t 300s
#   Signal: cpu.stat nr_throttled, PSI cpu, latency quantized to ~100ms

# Fault B: memory pressure and reclaim
sudo systemd-run --scope -p MemoryHigh=200M stress-ng --vm 2 --vm-bytes 1G -t 300s
#   Signal: PSI memory, allocstall in /proc/vmstat, direct reclaim in stacks

# Fault C: lock contention
#   (a module that hammers a global spinlock)
#   Signal: high sys time, perf lock, IPC < 1, perf c2c

# Fault D: an I/O-bound stall
sudo systemd-run --scope -p IOReadBandwidthMax="/dev/sda 1M" \
	dd if=/dev/sda of=/dev/null bs=1M count=1000
#   Signal: PSI io, D-state tasks, iostat await high with %util low

# Fault E: a slow syscall path
#   (an LD_PRELOAD or an eBPF program adding latency)
#   Signal: perf trace, funclatency on the syscall
```

**For each, work the procedure:**
1. Classify (Ch. 06 / `debugging-scenarios.md` §0).
2. USE: which resource is saturated?
3. On-CPU or off-CPU?
4. Form a hypothesis **with a predicted measurement**.
5. Test exactly that.
6. Confirm the fix removes the symptom.

**Time yourself.** Then do it again with the tools restricted (no bpftrace, no perf — only
`/proc`, `/sys`, and ftrace), which simulates the locked-down case of §T.10.

### Lab 95.7 — Instrument your own subsystem

Apply `reference/system-design.md` §10 to real code.

```c
/* Add to a driver or subsystem you own: */

/* 1. Tracepoints at every boundary. Zero cost when off. */
#define CREATE_TRACE_POINTS
#include <trace/events/mydrv.h>
trace_mydrv_submit(req->id, req->len);
trace_mydrv_complete(req->id, err, ktime_us_delta(now, req->start));

/* 2. Per-CPU counters for rates. Name them ACTIONABLY. */
struct mydrv_stats {
	u64 submitted;
	u64 completed;
	u64 tx_dropped_no_buffers;   /* good: says what to do */
	u64 rx_dropped_ring_full;
	u64 timeouts;
	u64 resets;
};
DEFINE_PER_CPU(struct mydrv_stats, mydrv_stats);

/* 3. A debugfs full-state dump, for "it is stuck". */
static int mydrv_state_show(struct seq_file *s, void *unused)
{
	struct mydrv_dev *d = s->private;
	seq_printf(s, "ring head=%u tail=%u inflight=%u\n",
		   d->head, d->tail, d->inflight);
	seq_printf(s, "state=%s last_irq=%llu ns ago\n",
		   state_name(d->state), ktime_get_ns() - d->last_irq);
	for (i = 0; i < NUM_DESCS; i++)
		seq_printf(s, "  [%3d] %s len=%u\n", i, ...);
	return 0;
}

/* 4. WARN_ON_ONCE for invariant violations -- loud, once. */
WARN_ON_ONCE(d->inflight > NUM_DESCS);

/* 5. A request ID threaded through every tracepoint, so a slow request
 *    can be followed across subsystems. Most kernel code does this
 *    badly; doing it well is a differentiator.                        */
```

Then verify:
```bash
# Can you answer "it is slow sometimes" from what the machine exports?
sudo bpftrace -e '
tracepoint:mydrv:mydrv_submit   { @s[args->id] = nsecs; }
tracepoint:mydrv:mydrv_complete /@s[args->id]/ {
	@us = hist((nsecs - @s[args->id])/1000); delete(@s[args->id]); }'
cat /sys/kernel/debug/mydrv/state
```

---

## 3. Mastery drills

1. Produce on-CPU, off-CPU, and differential flame graphs for a real application. Identify
   one genuine optimization from each.

2. Diagnose ten synthetic faults using only `/proc`, `/sys`, and ftrace — no perf, no bpf.
   Time each. This is the locked-down scenario.

3. Set up kdump on a production-like system, force a panic, and analyse the dump with both
   `crash` and `drgn`. Write the runbook.

4. Write ten `drgn` scripts for your subsystem: dump state, find leaks, walk structures,
   correlate. Make them work on both a live kernel and a `vmcore`.

5. Build a continuous-profiling pipeline for a fleet. Measure the overhead and demonstrate
   finding a regression by comparing two time windows.

6. Instrument a subsystem you own to the standard of Lab 95.7, then have someone report a
   vague problem and verify you can answer it from the exported data alone.

7. Compare ftrace histogram triggers, bpftrace, and perf for the same measurement. Quantify
   the overhead of each at 10k, 100k, and 1M events/sec.

8. Use `perf c2c` to find and fix a false-sharing problem in real code (Ch. 105). Measure the
   improvement.

9. Build a latency-attribution tool: given a slow request, report how much time was spent in
   each subsystem. Use synthetic events or BPF maps to correlate.

10. Study the CO-RE mechanism: compile a libbpf program, examine its BTF relocations, and run
    it on three different kernel versions. Explain what relocated and why.

11. Design the observability posture for a locked-down fleet (§T.10): what is instrumented,
    what is exported, how a debug kernel is deployed, and what the incident runbook is.

12. Take a real production incident (yours or a published post-mortem) and write the
    observability gap analysis: what data would have shortened the diagnosis, and what it
    would have cost to have it.

---

## 4. Further reading

**Books — read these in this order**
- **Brendan Gregg, *Systems Performance*, 2nd ed.** — the definitive book. The USE method,
  every tool, and the methodology. If you read one book from this chapter, this is it
- **Brendan Gregg, *BPF Performance Tools*** — 150+ tools, explained. The reference for
  everything eBPF-observability
- Gregg, *Linux Performance* website (`brendangregg.com/linuxperf.html`) — the tool map
  poster; print it

**Kernel documentation**
- `Documentation/trace/ftrace.rst` — long and complete; read the histogram-trigger section
- `Documentation/trace/kprobes.rst`, `uprobetracer.rst`, `histogram.rst`
- `Documentation/trace/events.rst`
- `Documentation/admin-guide/kdump/kdump.rst`
- `Documentation/bpf/` — the BPF documentation, including CO-RE
- `tools/perf/Documentation/`

**Tools**
- `perf` — `perf help`, and the perf wiki on perf.wiki.kernel.org
- `bpftrace` — the reference guide and the one-liner tutorial in the repo
- `bcc` — `/usr/share/bcc/tools/`; each tool has a `_example.txt`
- **`drgn`** — `drgn.readthedocs.io`. **Learn this one**; it is the most capable and least
  known tool in the chapter
- `crash` — the white paper and the built-in `help`
- `trace-cmd` / `kernelshark`
- `rtla` — `timerlat`, `osnoise` (Ch. 103)
- FlameGraph — `github.com/brendangregg/FlameGraph`

**Continuous profiling**
- Parca, Pyroscope, Polar Signals — read their architecture docs
- Google's "Google-Wide Profiling" paper (2010) — the origin of the practice

**Cross-references**
- Ch. 06 — the debugging environment, oops reading, and the sanitizers
- Ch. 75 — eBPF and the verifier
- Ch. 103 — `osnoise`/`timerlat` for latency attribution
- Ch. 105 — `perf c2c` for cacheline contention
- Ch. 102 §T.8 — lockdown, and the security/observability tension
- `reference/debugging-scenarios.md` — these tools applied to worked problems

→ Next: [96-yocto-1-concepts.md](96-yocto-1-concepts.md)

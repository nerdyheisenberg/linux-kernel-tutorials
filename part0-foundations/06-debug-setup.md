# Chapter 06 — Kernel Debugging Setup: gdb, KGDB, crash, pahole

> **Goal:** when something breaks, you have five independent ways to look inside, and you
> pick the right one in seconds.

---

## Theory & First Principles

> **How to read this section.** T.0 is the concrete experience of a kernel bug and why your
> userspace instincts fail. T.1–T.3 are the method. T.4–T.9 are the machinery — how the tools
> actually work, which is what lets you choose between them.

---

### T.0 — Start here: your first kernel bug, and why `printf` debugging fails

Write a bug on purpose. This takes five minutes and it recalibrates every instinct you have.

```c
// SPDX-License-Identifier: GPL-2.0
// DANGEROUS. QEMU ONLY (Ch. 04).
#include <linux/module.h>
#include <linux/slab.h>

static int __init bug_init(void)
{
	char *p = kmalloc(32, GFP_KERNEL);
	if (!p)
		return -ENOMEM;
	strcpy(p, "hello");
	kfree(p);

	pr_info("bug: reading freed memory: %s\n", p);   // USE AFTER FREE
	p[0] = 'X';                                      // WRITE after free
	return 0;
}
module_init(bug_init);
MODULE_LICENSE("GPL");
```

Load it on a normal kernel. **It very probably prints `hello` and succeeds.** Nothing
happens. The memory is still there, still readable, still writable — `kfree` returned it to
the slab allocator's free list, and nothing zeroed it.

Now rebuild with `CONFIG_KASAN=y` and load it again:

```
BUG: KASAN: slab-use-after-free in bug_init+0x4c/0xff [bug]
Read of size 1 at addr ffff888107a3d000 by task insmod/241

Allocated by task 241:
 kmalloc_trace+0x2a/0xb0
 bug_init+0x1f/0xff [bug]        <- WHERE IT CAME FROM

Freed by task 241:
 kfree+0x8f/0x1c0
 bug_init+0x45/0xff [bug]        <- WHERE IT WAS FREED   ← the culprit
```

**Four facts that reorganize how you debug, and they are the reason this chapter exists:**

1. **Kernel bugs are silent by default.** In userspace a use-after-free usually gets you a
   `SIGSEGV` fairly soon, because the allocator returns pages to the OS and the OS unmaps
   them. In the kernel there is nobody underneath to notice. The bug *works* — for hours,
   for months — and then corrupts something unrelated, somewhere else, later. **The crash
   site is the victim, not the culprit.**
2. **Therefore the primary skill is not reading crashes; it is converting silence into
   noise.** That is one table (§T.5 and `cheatsheets.md` §6) and it is worth more than any
   amount of cleverness. "I would rebuild with KASAN and lockdep" is a *correct first answer*
   to a large fraction of kernel bugs.
3. **`printk` debugging degrades badly here.** It perturbs timing (§T.2 — the probe effect),
   it serializes on the console at 115200 baud, and for a memory bug it tells you about the
   victim. The three stacks KASAN printed — access, allocation, **free** — are information
   `printk` fundamentally cannot produce.
4. **Read the free stack first.** Not the access stack. This inverts the userspace habit of
   reading a backtrace top-down, and it is the single most useful reading order in kernel
   debugging (`debugging-scenarios.md` §4).

**Now do the complementary experiment**, because it teaches the other half:

```bash
# Two spinlocks, taken in opposite orders on two paths. No deadlock occurs.
# Boot with CONFIG_PROVE_LOCKING=y and lockdep reports it anyway --
# the FIRST time it sees both orders, before any deadlock happens.
dmesg | grep -A30 'possible circular locking dependency'
```

That is the theme of the whole chapter: **the tools do not find bugs that happened; they find
bugs that are possible.** Testing samples the interleaving space; these tools reason about
it. Ch. 25 §T.2 explains why, for concurrency, that distinction is the entire ballgame.

---

### T.1 Debugging is hypothesis testing, not guessing

Andreas Zeller (*Why Programs Fail*, 2005) formalizes debugging as the **scientific method**
applied to a program, and naming the steps is what turns flailing into engineering:

```
   1. Observe a failure                        (symptom)
   2. Form a hypothesis about the cause
   3. Derive a PREDICTION the hypothesis makes  ← the step everyone skips
   4. Design an experiment that can FALSIFY it
   5. Run it. Refine or reject.
   6. Repeat until the hypothesis is confirmed AND explains everything observed.
```

Step 3 is the discriminator. "Maybe it's a locking bug" is not a hypothesis — it makes no
prediction. "If this is a missing barrier on the publish path, then (a) it will never
reproduce on x86, (b) it will reproduce faster with more CPUs, and (c) adding `smp_wmb()`
at line 412 will eliminate it" *is* a hypothesis: three falsifiable predictions.

Zeller's chain of causality gives you the vocabulary:

```
  defect (the wrong code)
     ↓ executes
  infection (a wrong value in the program state)
     ↓ propagates through more state
  failure (observable wrong behaviour)
```

Debugging is **walking backwards along this chain**, from failure to infection to defect.
Every tool in this chapter is an instrument for observing state at some point in that chain:
`printk` and tracing observe it *during* execution, KASAN detects *infection at the moment it
occurs*, a core dump captures it *after failure*, and a debugger lets you walk it
interactively.

**Delta debugging** (Zeller & Hildebrandt, TSE 2002) automates the search: given a failing
input and a passing input, binary-search the *difference* to find the minimal change that
flips the outcome. `git bisect` is delta debugging over commits. Config bisection (Ch. 03)
is delta debugging over `.config`. `syzkaller`'s reproducer minimization is delta debugging
over syscall sequences. Recognizing that these are the same algorithm lets you apply it to
new dimensions (kernel command line, hardware config, workload parameters).

### T.2 Bohrbugs, Heisenbugs, and the probe effect

Jim Gray (*Why Do Computers Stop and What Can Be Done About It?*, 1985) named the taxonomy
that governs your strategy:

| Class | Behaviour | Strategy |
|---|---|---|
| **Bohrbug** | deterministic; reproduces every time under the same conditions | straightforward: bisect, instrument, fix |
| **Heisenbug** | disappears or changes when observed | **you cannot instrument your way in** — change strategy |
| **Mandelbug** | cause is so complex the behaviour appears chaotic | reduce the system, not the bug |
| **Schrödinbug** | works until someone reads the code and realizes it can't | audit-driven |

The **probe effect** is why Heisenbugs exist in kernel work specifically: adding a `printk`
inserts a serializing operation of ~1–10 µs, which reorders the very interleaving you are
chasing. A race with a 50 ns window becomes unreachable. Corollaries that should change your
behaviour:

- **Prefer low-overhead instrumentation for timing-sensitive bugs**: `trace_printk()`
  (~100 ns, writes to the ftrace ring buffer, no console), tracepoints, BPF, or a per-CPU
  array you dump afterwards. **Never** `printk` into a race.
- **Prefer detectors that don't change timing much**: KASAN (~2–3× slowdown but *uniform*),
  KCSAN (samples, deliberately *adds* delays to *increase* race detection — it weaponizes the
  probe effect), lockdep (changes almost nothing about timing).
- **Prefer deterministic replay** (Ch. 04 T.5) when available: record once, then observe as
  much as you like without perturbing the recorded execution. This is the *only* technique
  that fully defeats the probe effect.

### T.3 The observability hierarchy, ranked by cost and by when to reach for it

There is an optimal order, and violating it wastes days:

| Rank | Tool | Overhead | Requires reboot? | Answers |
|---|---|---|---|---|
| 1 | Read the Oops/WARN | 0 | no | *where* and often *why* — 60% of bugs end here |
| 2 | `dmesg`, existing `dev_dbg` via `dynamic_debug` | ~0 | no | what the driver thought it was doing |
| 3 | ftrace (`function_graph`, `irqsoff`, events) | 1–5% | no | call paths, latency, ordering |
| 4 | `perf`, `bpftrace`, kprobes | 1–10% | no | hot paths, arguments, histograms |
| 5 | Sanitizers (KASAN/KCSAN/UBSAN/kmemleak) | 2–5× | **yes** (rebuild) | memory & concurrency correctness |
| 6 | `gdb` on QEMU / KGDB | huge | yes | arbitrary state inspection |
| 7 | `kdump` + `crash` | 0 until it crashes | yes (config) | production postmortem |
| 8 | Fuzzing (`syzkaller`), fault injection | N/A | yes | bugs you haven't seen yet |

**The mistake juniors make is starting at rank 6.** Attaching a debugger is the *most*
expensive way to get information and usually the least informative — the kernel is a
concurrent system, so stopping it destroys exactly the timing you need. Start at rank 1 and
only descend when the current rank has been exhausted.

**The mistake seniors make is never reaching rank 8.** Proactive fuzzing and fault injection
find bugs before customers do, and that is a fundamentally better economic position.

### T.4 Shadow memory: how KASAN actually works

KASAN (Kernel Address Sanitizer, from Google's ASan) is the highest-value tool in this
chapter, and the theory is simple and beautiful.

**Idea:** maintain a *shadow* of all kernel memory recording, for each 8-byte granule, how
many of its bytes are addressable.

```
   shadow_address = (address >> 3) + KASAN_SHADOW_OFFSET
```
One shadow byte per 8 bytes of memory ⇒ **1/8 memory overhead**. The encoding:

| Shadow value | Meaning |
|---|---|
| `0` | all 8 bytes addressable |
| `1..7` | first N bytes addressable, rest are a redzone (partial granule) |
| negative (e.g. `0xFA`, `0xFB`, `0xFC`) | entirely poisoned; the *value* encodes **why** (freed, redzone, stack out-of-scope, global) |

The compiler (`-fsanitize=kernel-address`) inserts a check before every memory access:

```c
/* conceptually, before `*p = v;` */
if (__asan_load8(p)) report_error(p, size, is_write, _RET_IP_);
```

The allocator cooperates: `kmalloc` poisons **redzones** around each object and the allocator
metadata; `kfree` poisons the whole object and puts it in a **quarantine** so it is not
immediately reused — which is what makes use-after-free *detectable* rather than
*silently-works*. Quarantine size is a direct memory/detection-power trade.

Three modes, and you should know which to use:

| Mode | `CONFIG` | Overhead | Hardware | Use |
|---|---|---|---|---|
| **Generic** | `KASAN_GENERIC` | ~2–3× CPU, ~1/8 mem | any | **development; the default** |
| **Software tag-based** | `KASAN_SW_TAGS` | ~2× | arm64 (TBI) | arm64 testing |
| **Hardware tag-based** | `KASAN_HW_TAGS` | **~few %** | arm64 MTE | **production-capable** |

Tag-based modes store a 4-bit tag in the pointer's top byte and a matching tag per granule;
a mismatch faults. That's *probabilistic* (1/16 chance of a tag collision) but nearly free —
a different point on the same detection/cost curve.

**KFENCE** takes the opposite approach: instead of checking every access (expensive but
complete), it places a *small number* of allocations on guard-page-protected pages and lets
the MMU do the checking for free. Detection becomes **sampling**: probability of catching a
given bug ≈ (sampled fraction) × (number of times the bug executes). With ~0% overhead you
can run it in **production across a fleet**, and at fleet scale the sampling converges.
This is a genuinely important idea: *trade per-instance detection probability for the ability
to deploy everywhere, then recover power from scale.*

```
   KASAN:  P(detect) ≈ 1.0,   cost 200%,  deploy: dev only
   KFENCE: P(detect) ≈ 0.001, cost ~0%,   deploy: everywhere × millions of machines
```

### T.5 The sanitizer family — what each one can and cannot see

| Tool | Detects | Mechanism | Blind to |
|---|---|---|---|
| **KASAN** | OOB, UAF, double-free, invalid-free | shadow memory + redzones + quarantine | races, logic bugs, uninitialized reads |
| **KMSAN** | **uninitialized** memory reads | shadow *of initializedness*, propagated through computation | everything else; very slow |
| **KCSAN** | data races | sampled watchpoints + deliberate delays | races that never execute concurrently in your test |
| **UBSAN** | shift/overflow/bounds/alignment/`unreachable` | compiler instrumentation | anything not UB |
| **kmemleak** | leaks | conservative mark-and-sweep over kernel memory | leaks reachable from a live pointer |
| **lockdep** | lock-order inversions, IRQ-unsafe usage | graph cycle detection over lock classes | orders never exercised |
| **DEBUG_OBJECTS** | use of uninit/freed timers, work, RCU heads | per-object state tracking | non-tracked object types |
| **KFENCE** | OOB/UAF (sampled) | guard pages | most instances of any given bug |

Note the pattern: **each tool has a precisely characterized blind spot**, and the blind spots
are mostly disjoint. That is why the recommended dev config turns on *all* of them, and why
"it passed KASAN" is not the same as "it is correct." A senior engineer can state the blind
spot of each tool from memory.

`kmemleak`'s mechanism deserves a note: it is a **conservative garbage collector** that never
collects — it scans kernel memory treating any word that looks like a pointer into a tracked
allocation as a reference, then reports unreferenced allocations. "Conservative" means false
*negatives* are possible (an integer that happens to look like a pointer keeps an object
alive) but false positives are rare. Understanding it as a GC explains both its power and its
noise.

### T.6 Coverage-guided fuzzing: a genetic algorithm over the syscall interface

`syzkaller` is the reason the kernel's bug rate dropped measurably after 2016, and the theory
is worth knowing because it changes what "testing" means.

Classic random testing is hopeless against a kernel: the probability that random bytes form a
valid syscall sequence reaching deep code is effectively zero. Coverage-guided fuzzing
(AFL's insight, 2013) fixes this with a feedback loop that is literally a **genetic
algorithm**:

```
  corpus ──▶ select an input ──▶ MUTATE it ──▶ execute ──▶ measure COVERAGE (KCOV)
     ▲                                                             │
     └──────── if it reached NEW coverage, add to corpus ◀──────────┘
```

Coverage is the **fitness function**. Inputs that reach new basic blocks survive and breed;
the corpus evolves toward deeper, more interesting executions. Empirically this converts an
intractable search into a tractable one.

`syzkaller` adds two kernel-specific pieces:
- **KCOV** (`CONFIG_KCOV`) — per-task, deterministic coverage collection exported to
  userspace via a `debugfs` mmap. Per-task matters: it attributes coverage to *your*
  syscall, not to background kernel activity.
- **`syzlang` descriptions** — a declarative grammar of syscall arguments (types, ranges,
  resource dependencies like "this fd came from that `open`"). Without it, mutation would
  spend all its time producing `EINVAL`. This is the "structure-aware fuzzing" idea.

Combined with KASAN as the **oracle** (what counts as a crash), `syzbot` runs this
continuously and files bugs automatically. Practical implications for you:

1. **Assume your new syscall/ioctl will be fuzzed within weeks of merging.** Write the
   `syzlang` description yourself and test it first (Ch. 50).
2. **A syzbot report with a C reproducer is a gift** — it's a minimized, deterministic
   test case. Fixing syzbot bugs is one of the best-regarded ways to build a track record.
3. `KCOV` is also usable directly for measuring *your own* test coverage of kernel code.

**Fault injection** is the complementary technique: instead of exploring inputs, explore
*error paths*, which are the least-tested code in any kernel. `CONFIG_FAILSLAB`,
`CONFIG_FAIL_PAGE_ALLOC`, `CONFIG_FAIL_MAKE_REQUEST`, and `CONFIG_FUNCTION_ERROR_INJECTION`
make `kmalloc` etc. fail probabilistically. Most kernel bugs found by review are in error
paths; fault injection finds them mechanically.

### T.7 Bisection: binary search over a DAG

`git bisect` is binary search, but git history is a **DAG**, not a list, so the implementation
is subtler than it looks. Given a known-good commit `G` and known-bad `B`, the candidate set
is the commits reachable from `B` but not from `G`. Git picks the commit that most evenly
**bisects that set by reachability** — maximizing information gain per test, which is exactly
the information-theoretic optimum of ~log₂(N) tests.

For 10,000 commits between two releases that is ~14 builds. At 5 minutes per build+boot+test
(Ch. 04 T.1!), that's about an hour. **If your loop is 30 minutes, bisection costs a day
and you won't do it.** This is the concrete payoff of Ch. 04.

Practical technique that separates people who bisect successfully from people who give up:

```bash
git bisect start bad-sha good-sha
git bisect run ./test.sh      # script: exit 0 = good, 1 = bad, 125 = skip (won't build)
```
Writing a **reliable, automated `test.sh`** is the whole job. It must: build, boot, run the
reproducer, detect the failure *unambiguously*, and time out. `exit 125` for untestable
commits is essential — mid-series commits often don't build.

For flaky failures, run the reproducer N times and treat "any failure" as bad; note that
bisection's correctness assumes **monotonicity** (once bad, always bad), which flaky bugs
violate. If it's flaky, either make it deterministic first or accept that you may need to
re-run the bisection.

### T.8 Postmortem debugging and the information problem

A production crash gives you exactly one artifact and no second chance. The question is: how
much state must be captured for the failure to be diagnosable?

```
  panic message only        → often enough for a NULL deref with a symbolized stack
  + full dmesg ring         → the sequence of events leading up
  + register dump + stack   → the immediate cause
  + FULL memory dump (kdump)→ every data structure; the only option for corruption bugs
```

`kdump`/`kexec` works by **reserving memory at boot** (`crashkernel=256M`) for a second
kernel that is already loaded. On panic, the primary kernel `kexec`s directly into it —
no firmware, no reboot, so the old kernel's memory is intact and gets written out as
`/proc/vmcore`. Then `crash` (or `drgn`) reads it with full type information from DWARF.

The design insight worth stealing: **to diagnose a failure of system X, you need a
*separate*, already-initialized system Y that does not depend on X.** The reserved memory and
pre-loaded kernel exist because you cannot trust anything about the crashed kernel.

**`drgn`** (Meta, Omar Sandoval) is the modern alternative to `crash`: a Python library that
reads a live kernel or a vmcore using DWARF/BTF, letting you *script* structure traversal:

```python
>>> from drgn.helpers.linux import find_task, for_each_task
>>> for t in for_each_task(prog):
...     if t.mm: print(t.pid.value_(), t.comm.string_().decode())
```
Being fluent in `drgn` is a genuine differentiator in 2020s kernel work.

### T.9 Type information: DWARF and BTF

Every tool above needs to know that `((struct task_struct *)x)->pid` is at offset 0x9d8.
Two formats provide that:

- **DWARF** (`CONFIG_DEBUG_INFO`) — the full, general debug format. Enormous (a `vmlinux`
  with DWARF is 1–5 GB), consumed by gdb/crash/drgn/pahole.
- **BTF** (BPF Type Format, `CONFIG_DEBUG_INFO_BTF`) — a compact (~3–5 MB) subset containing
  just types, generated from DWARF by `pahole`. Embedded *in the running kernel* at
  `/sys/kernel/btf/vmlinux`.

BTF is what makes **BPF CO-RE** ("Compile Once, Run Everywhere") possible: a BPF program
compiled against one kernel's headers can be *relocated* at load time against the target
kernel's actual struct offsets, read from its BTF. That solves the portability problem that
made BPF tooling painful for a decade. It also means `bpftrace` can read arbitrary kernel
structures on any modern kernel with no headers installed.

```bash
ls -la /sys/kernel/btf/vmlinux
bpftool btf dump file /sys/kernel/btf/vmlinux format c | head -50
pahole -C task_struct /sys/kernel/btf/vmlinux | head -40      # works without DWARF!
```

---

### T.10 — What is still argued about

**1. How much debug instrumentation should ship in production?**
KASAN is too slow (~3× memory, ~2× CPU). But **KFENCE** samples at near-zero overhead and is
designed to ship. Lockdep is ~5–10% and catches a bug class that is otherwise found only in
the field. The fleet argument — enable expensive checks on 1% of machines — is strong and
under-used: a 5% cost on 50 machines is cheap; a week per bug is not.

**2. Is `panic_on_warn` right?**
Crashing on a `WARN_ON` turns a survivable anomaly into an outage, which sounds obviously
wrong. But it also turns an *ignored* anomaly into a `vmcore` you can analyse, and warnings
that nobody reads are worse than useless. syzkaller sets it; most production does not;
Google runs it fleet-wide on a fraction of machines. **It is a policy decision about whether
you would rather lose a machine or lose the evidence.**

**3. Does lockdep's false-positive rate undermine it?**
Annotating nested-lock and per-object-lock cases (`lockdep_set_subclass`, `mutex_lock_nested`)
is real work, and wrong annotations *hide* real bugs. The alternative — no order checking —
is strictly worse, but the annotation burden is a genuine cost and a source of subtle
mis-annotation.

**4. Security versus observability.**
`lockdown=confidentiality` disables kprobes, most BPF tracing, `/proc/kcore`, and `perf`
kernel sampling (Ch. 102 §T.8). A locked-down fleet is one you cannot debug with the tools in
this chapter. **This tension has no clean resolution** — it must be designed for up front
(Ch. 95 §T.10), not discovered during an incident.

### T.11 — The compressed model

```
 KERNEL BUGS ARE SILENT BY DEFAULT. The crash site is the VICTIM, not the
 culprit. Therefore the primary skill is CONVERTING SILENCE INTO NOISE.

 THE CONVERSION TABLE IS THE MOST VALUABLE THING IN THIS CHAPTER:
   UAF/OOB -> KASAN      uninit -> KMSAN      races -> KCSAN
   UB -> UBSAN           lock order -> lockdep
   sleep-in-atomic -> DEBUG_ATOMIC_SLEEP      DMA -> DMA_API_DEBUG + strict IOMMU
   production-safe memory bugs -> KFENCE

 KASAN WORKS BY SHADOW MEMORY: 1 shadow byte per 8 bytes, encoding how many
 are addressable. Every access is instrumented to check it. That is why it
 costs 1/8 of RAM and why it cannot see what the DEVICE does (DMA).

 DEBUGGING IS HYPOTHESIS TESTING. Form a hypothesis WITH A PREDICTED
 MEASUREMENT, then test exactly that. A fix that works for unknown reasons
 has taught you nothing.

 THESE TOOLS FIND BUGS THAT ARE POSSIBLE, not bugs that happened. lockdep
 reports an inversion it never saw deadlock. That is the entire value.
```

Five questions for any bug:

1. **Which class is it?** crash / hang / corruption / performance / latency / leak. The class
   picks the tool; reaching for `perf` on a corruption bug wastes an hour.
2. **Is it reproducible, and how cheaply?** Invest in the reproducer before the analysis.
3. **What changed?** Version, config, hardware, load. If something changed, `git bisect`
   usually beats reasoning.
4. **What is my hypothesis, and what measurement would falsify it?**
5. **Which `CONFIG_` turns this silent failure loud?**

---

## 1. The debugging toolbox, ranked by when to use it

| Situation | Tool | Cost |
|---|---|---|
| "What is this code doing?" | `ftrace function_graph` | zero setup |
| "What are the arguments?" | `bpftrace` / `kprobe` | seconds |
| "It crashed" | Oops decode + `faddr2line` | seconds |
| "It corrupted memory" | KASAN / KFENCE / SLUB_DEBUG | rebuild |
| "It deadlocked" | lockdep + `SysRq-d`/`-w` | rebuild |
| "It raced" | KCSAN | rebuild (clang) |
| "It leaked" | kmemleak | rebuild |
| "I need to single-step" | QEMU `-s -S` + gdb | VM only |
| "Production box wedged" | kdump/kexec + `crash` | prod setup |
| "Hardware-level" | JTAG / `kgdb` over serial | embedded |

---

## 2. Reading an Oops — the skill that matters most

```
BUG: kernel NULL pointer dereference, address: 0000000000000010
#PF: supervisor read access in kernel mode
#PF: error_code(0x0000) - not-present page
PGD 0 P4D 0
Oops: 0000 [#1] PREEMPT SMP NOPTI
CPU: 3 PID: 412 Comm: insmod Tainted: G           OE      6.12.0 #1
Hardware name: QEMU Standard PC (Q35 + ICH9, 2009)
RIP: 0010:hello_init+0x2f/0x80 [hello]
Code: 48 c7 c7 00 00 00 00 e8 ... <48> 8b 40 10 48 89 ...
RSP: 0018:ffffc900001b7c38 EFLAGS: 00010246
RAX: 0000000000000000 RBX: ffff888103a2c000 RCX: 0000000000000000
...
Call Trace:
 <TASK>
 do_one_initcall+0x5b/0x320
 do_init_module+0x60/0x250
 load_module+0x1f4e/0x2130
 __do_sys_finit_module+0xd5/0x140
 do_syscall_64+0x5b/0x90
 entry_SYSCALL_64_after_hwframe+0x76/0x7e
 </TASK>
```

**Decode it line by line:**

| Field | Meaning |
|---|---|
| `address: 0000000000000010` | offset 0x10 from NULL → you dereferenced `ptr->member_at_0x10` where `ptr == NULL` |
| `Oops: 0000` | bit 0 = not-present, bit 1 = write (0 → read), bit 2 = user mode, bit 4 = instruction fetch |
| `[#1]` | first oops since boot. `[#2]` etc. means you're in cascade territory |
| `PREEMPT SMP NOPTI` | kernel config flags — tells you preemption model, PTI state |
| `Tainted: G OE` | `G`=no proprietary, `O`=out-of-tree module, `E`=unsigned. Decode via `Documentation/admin-guide/tainted-kernels.rst` |
| `RIP: 0010:hello_init+0x2f/0x80 [hello]` | ★ **faulting instruction**: 0x2f bytes into an 0x80-byte function, in module `hello` |
| `Code:` | raw bytes; `<>` marks the faulting instruction |
| `Call Trace:` | stack unwind (ORC on x86 by default) |

**Step 1 — find the exact source line:**
```bash
./scripts/faddr2line hello.ko hello_init+0x2f/0x80
# hello_init+0x2f/0x80:
# hello_init at /home/you/hello/hello.c:58
```
For vmlinux symbols: `./scripts/faddr2line vmlinux do_one_initcall+0x5b/0x320`

**Step 2 — full symbolization:**
```bash
./scripts/decode_stacktrace.sh vmlinux . < oops.txt
# or from live dmesg:
dmesg | ./scripts/decode_stacktrace.sh ~/build-x86/vmlinux
```

**Step 3 — decode the instruction bytes:**
```bash
./scripts/decodecode < oops.txt      # feeds Code: line to objdump
```

**Step 4 — find the struct member at that offset:**
```bash
pahole -C hello_ctx hello.ko
# struct hello_ctx {
#     struct mutex  lock;       /*   0  32 */
#     long unsigned greetings;  /*  32   8 */
#     char *        buf;        /*  40   8 */   ← offset 0x28
# };
```
`pahole` is how you convert "address 0x10" into "you deref'd `->foo`".

### Panic vs Oops vs BUG vs WARN

| | Effect |
|---|---|
| `WARN_ON(cond)` / `WARN_ONCE` | prints stack, **continues**. Taints kernel with `W`. |
| `BUG_ON(cond)` | `Oops`, kills the task. Usually **wrong** to add — prefer graceful handling |
| `panic()` | stops the machine |
| Oops | kills the offending task; kernel limps on in an undefined state |
| `oops=panic` boot param | turn every Oops into a panic — use in CI/dev |
| `panic_on_warn=1` | turn every WARN into a panic — used by syzkaller |

> **Upstream rule:** never add `BUG_ON()` for conditions that can be handled.
> Linus: "BUG_ON() is basically always the wrong thing to do."

---

## 3. GDB on a live kernel

### 3.1 QEMU stub (the normal case)
Covered in Chapter 04. Recap:
```bash
qemu ... -s -S          # gdbserver on :1234, CPU halted
gdb vmlinux -ex 'target remote :1234'
```

Boot with `nokaslr`. Enable `CONFIG_GDB_SCRIPTS=y`, `CONFIG_DEBUG_INFO_DWARF5=y`,
`CONFIG_FRAME_POINTER=y`, `CONFIG_RANDOMIZE_BASE=n`.

### 3.2 Essential gdb-for-kernel commands

```gdb
# Kernel python helpers (scripts/gdb/linux/)
lx-symbols [modpath]       # load .ko symbols
lx-dmesg                   # kernel ring buffer
lx-ps                      # process list
lx-lsmod
lx-mounts
lx-fdtdump                 # device tree
lx-device-list-bus [bus]
lx-device-list-class
lx-clk-summary
lx-genpd-summary
lx-interruptlist
lx-timerlist
lx-configdump              # the .config baked into the kernel
lx-cmdline
lx-version

# Convenience functions
p $lx_current()                     # current task_struct
p $lx_current().pid
p $lx_per_cpu(runqueues, 2)
p $lx_task_by_pid(1)
p $lx_module("ext4")
p $lx_clk_core_lookup("uart0")

# Plain gdb, kernel-flavored
p *(struct task_struct *)0xffff888103a2c000
p ((struct task_struct *)$lx_current())->mm->mmap_base
ptype struct inode
p &((struct inode *)0)->i_size      # offsetof
x/32xg 0xffffffff82000000
info registers
bt
frame 3
info line *0xffffffff81234567
disassemble /s hello_init
watch -l ctx->greetings             # hardware watchpoint
```

### 3.3 Debugging a module
```gdb
(gdb) c                             # let it boot
# in guest: insmod /mnt/host/hello.ko
(gdb) Ctrl-C
(gdb) lx-symbols /mnt/host          # or the host path to hello.ko
(gdb) break hello_init
```

Manual alternative (if `lx-symbols` fails):
```bash
# in guest
cat /sys/module/hello/sections/.text /sys/module/hello/sections/.data \
    /sys/module/hello/sections/.bss
```
```gdb
(gdb) add-symbol-file hello.ko 0xffffffffc0000000 -s .data 0x... -s .bss 0x...
```

### 3.4 `.gdbinit` worth having
```gdb
set confirm off
set pagination off
set print pretty on
set print array-indexes on
set history save on
add-auto-load-safe-path /home/you/build-x86
define hook-stop
  info line
end
define kconnect
  target remote :1234
  lx-symbols
end
```

---

## 4. KGDB — debugging on real hardware

```
CONFIG_KGDB=y
CONFIG_KGDB_SERIAL_CONSOLE=y
CONFIG_KGDB_KDB=y          # the built-in "kdb" shell, no host needed
CONFIG_MAGIC_SYSRQ=y
CONFIG_FRAME_POINTER=y
CONFIG_KALLSYMS_ALL=y
```
Boot params: `kgdboc=ttyS0,115200 kgdbwait`
(`kgdbwait` halts at boot until a debugger attaches.)

Enter the debugger at runtime:
```bash
echo g > /proc/sysrq-trigger
```
Then on the host: `gdb vmlinux -ex 'target remote /dev/ttyUSB0'`

**kdb** is the on-target mode: `bt`, `ps`, `md` (memory display), `go`, `btc` (all CPUs).
Toggle with `echo kdb > /sys/module/kgdboc/parameters/kgdboc` variants.

For embedded boards: `kgdboc` over USB gadget serial, or `kgdb` over NETPOLL
(`kgdboe`, out-of-tree these days). JTAG (OpenOCD + gdb) is the ultimate fallback for
"it doesn't even reach `start_kernel`".

---

## 5. kdump / kexec + `crash` — production postmortem

```
CONFIG_KEXEC=y CONFIG_KEXEC_FILE=y CONFIG_CRASH_DUMP=y CONFIG_DEBUG_INFO=y
CONFIG_PROC_VMCORE=y
```
Boot param: `crashkernel=512M` (reserve RAM for the capture kernel).

```bash
sudo systemctl enable --now kdump
kdumpctl status
# force a crash
echo c | sudo tee /proc/sysrq-trigger
# after reboot:
ls /var/crash/*/vmcore
crash /usr/lib/debug/boot/vmlinux-$(uname -r) /var/crash/*/vmcore
```

Inside `crash`:
```
crash> bt                     # backtrace of panicking task
crash> bt -a                  # all CPUs
crash> ps | grep UN           # uninterruptible tasks
crash> log                    # dmesg
crash> kmem -i                # memory summary
crash> kmem -s                # slab caches
crash> files <pid>
crash> struct task_struct ffff8881...
crash> struct -o inode        # offsets
crash> dis -l hello_init
crash> mod -s hello /path/hello.ko
crash> foreach UN bt          # every blocked task's stack ← finds deadlocks
crash> dev -d                 # block device stats
crash> net
```

`makedumpfile -c -d 31` compresses and strips free/cache pages — a 128 GB box produces
a manageable dump.

Modern alternative: **`drgn`** (from Meta) — Python-based, works on live kernels *and*
core dumps, no separate crash utility:
```bash
pip install drgn
sudo drgn
>>> from drgn.helpers.linux import *
>>> for t in for_each_task(prog):
...     if t.comm.string_().decode() == 'bash': print(t.pid.value_())
>>> prog['jiffies']
>>> list_for_each_entry('struct module', prog['modules'].address_of_(), 'list')
```
`drgn` is what you'll reach for in 2026. Learn it.

---

## 6. Sanitizers & runtime checkers

| Tool | Config | Catches | Cost |
|---|---|---|---|
| **KASAN** (generic) | `CONFIG_KASAN=y` `KASAN_INLINE` | OOB, UAF, double-free | 2–3× slow, 1/8 mem |
| **KASAN** (SW tags) | arm64 | same, cheaper memory | arm64 only |
| **KASAN** (HW tags/MTE) | arm64 MTE | same, ~production-viable | needs MTE hw |
| **KFENCE** | `CONFIG_KFENCE=y` | sampled OOB/UAF | ~0% — **enable in production** |
| **KMSAN** | clang only | uninitialized memory reads | very slow |
| **KCSAN** | clang | data races | slow |
| **UBSAN** | `CONFIG_UBSAN` | shift/overflow/align/bounds UB | low |
| **kmemleak** | `CONFIG_DEBUG_KMEMLEAK` | leaks | high |
| **lockdep** | `CONFIG_PROVE_LOCKING` | lock-order inversions, wrong context | 2× |
| **DEBUG_OBJECTS** | | use-after-free of timers/work/rcu | moderate |
| **SLUB_DEBUG** | `slub_debug=FZPU` boot | redzones, poison, owner tracking | moderate |
| **DEBUG_PAGEALLOC** | | page-granular UAF | high |
| **objtool** | built in | bad control flow, missing ORC | build-time |
| **sparse** | `make C=1` | `__user`/`__iomem` misuse, endianness | build-time |
| **smatch** | external | flow analysis, error paths | build-time |
| **coccinelle** | `make coccicheck` | API misuse patterns | build-time |

Reading a KASAN report:
```
BUG: KASAN: slab-use-after-free in hello_exit+0x1a/0x60 [hello]
Read of size 8 at addr ffff888103a2c028 by task rmmod/512

Allocated by task 412:            ← WHERE IT WAS ALLOCATED
 kasan_save_stack+...
 hello_init+0x20/0x80 [hello]
Freed by task 512:                ← WHERE IT WAS FREED
 kfree+0x...
 hello_exit+0x10/0x60 [hello]
```
KASAN gives you **three** stacks: use, alloc, free. That is usually the whole bug.

lockdep report anatomy:
```
======================================================
WARNING: possible circular locking dependency detected
------------------------------------------------------
insmod/412 is trying to acquire lock:
 ffff... (&b->lock){+.+.}-{3:3}, at: foo+0x...
but task is already holding lock:
 ffff... (&a->lock){+.+.}-{3:3}, at: bar+0x...
which lock already depends on the new lock.
```
The `{+.+.}` notation: read `Documentation/locking/lockdep-design.rst`.
Position 1 = used in irq context, 2 = used with irqs enabled, 3/4 = softirq equivalents.

Useful lockdep knobs:
```bash
cat /proc/lockdep_stats
cat /proc/lock_stat            # needs CONFIG_LOCK_STAT
echo 0 > /proc/sys/kernel/lock_stat
echo w > /proc/sysrq-trigger   # blocked tasks
echo d > /proc/sysrq-trigger   # all held locks
echo l > /proc/sysrq-trigger   # backtrace all active CPUs
echo t > /proc/sysrq-trigger   # all task stacks
```

---

## 7. `pahole` — structure layout as a first-class skill

```bash
pahole -C task_struct vmlinux | head -60
pahole -C inode vmlinux
pahole --hex -C bio vmlinux
pahole -C sk_buff --reorganize vmlinux      # suggest a better layout
pahole --sizes vmlinux | sort -k2 -n | tail  # biggest structs
pahole -C ext4_inode_info --expand_types vmlinux
```

Output shows holes and cacheline boundaries:
```
struct foo {
        int      a;            /*     0     4 */
        /* XXX 4 bytes hole, try to pack */
        long     b;            /*     8     8 */
        /* --- cacheline 1 boundary (64 bytes) --- */
        ...
        /* size: 72, cachelines: 2, members: 5 */
        /* sum members: 64, holes: 1, sum holes: 4 */
        /* padding: 4 */
};
```

This is how you do real performance work on hot structures (`sk_buff`, `page`,
`task_struct`, `request`). Chapters 29 and 63 use it heavily.

Related: `__cacheline_aligned`, `____cacheline_aligned_in_smp`,
`struct_group()`, `CACHELINE_PADDING()`, `__randomize_layout`.

---

## 8. Practice

### Lab 6.1 — Deliberately crash and fully diagnose
Add `*(int *)0x10 = 1;` to `hello_init`. Boot in QEMU. Capture the Oops, run
`decode_stacktrace.sh`, `faddr2line`, and `decodecode`. Explain every field.

### Lab 6.2 — Build a KASAN kernel, plant a UAF
```c
kfree(ctx->buf);
pr_info("%c\n", ctx->buf[0]);   /* boom */
```
Read all three stacks in the report.

### Lab 6.3 — Trigger lockdep
Two mutexes, two kthreads, opposite order. Read the report. Then fix with a documented
lock ordering comment.

### Lab 6.4 — kdump end to end
On a VM with a real distro: configure `crashkernel=`, crash via sysrq, analyze in `crash`
and in `drgn`. Compare.

### Lab 6.5 — pahole a hot struct
Run `pahole -C sk_buff --reorganize`. Read `include/linux/skbuff.h` and explain why the
maintainers laid it out the way they did (hint: RX path cacheline).

### Lab 6.6 — Build the canonical debug kernel config

This config is the single most valuable artifact in this chapter. Save it and reuse it
forever.

```bash
cd linux && make O=bdbg defconfig
./scripts/config --file bdbg/.config \
  -e DEBUG_KERNEL -e DEBUG_INFO_DWARF5 -e DEBUG_INFO_BTF -e GDB_SCRIPTS \
  -e FRAME_POINTER -e DEBUG_FS -e MAGIC_SYSRQ \
  \
  -e KASAN -e KASAN_GENERIC -e KASAN_VMALLOC -e KASAN_INLINE \
  -e KFENCE --set-val KFENCE_SAMPLE_INTERVAL 100 \
  -e UBSAN -e UBSAN_BOUNDS -e UBSAN_SHIFT -e UBSAN_TRAP=n \
  -e DEBUG_KMEMLEAK --set-val DEBUG_KMEMLEAK_MEM_POOL_SIZE 16000 \
  \
  -e PROVE_LOCKING -e PROVE_RCU -e PROVE_RCU_LIST -e DEBUG_ATOMIC_SLEEP \
  -e DEBUG_SPINLOCK -e DEBUG_MUTEXES -e DEBUG_RWSEMS -e LOCK_STAT \
  -e DEBUG_LOCK_ALLOC -e DEBUG_WW_MUTEX_SLOWPATH \
  \
  -e DEBUG_OBJECTS -e DEBUG_OBJECTS_FREE -e DEBUG_OBJECTS_TIMERS \
  -e DEBUG_OBJECTS_WORK -e DEBUG_OBJECTS_RCU_HEAD -e DEBUG_OBJECTS_PERCPU_COUNTER \
  -e DEBUG_LIST -e DEBUG_PLIST -e DEBUG_SG -e DEBUG_NOTIFIERS \
  -e DEBUG_CREDENTIALS -e DEBUG_VIRTUAL -e DEBUG_PER_CPU_MAPS \
  \
  -e SLUB_DEBUG -e SLUB_DEBUG_ON -e PAGE_POISONING -e DEBUG_PAGEALLOC \
  -e DEBUG_VM -e DEBUG_VM_PGFLAGS -e PAGE_OWNER \
  \
  -e DETECT_HUNG_TASK -e WQ_WATCHDOG -e SOFTLOCKUP_DETECTOR -e HARDLOCKUP_DETECTOR \
  -e PANIC_ON_OOPS -e SCHED_DEBUG -e SCHEDSTATS \
  \
  -e FTRACE -e FUNCTION_TRACER -e FUNCTION_GRAPH_TRACER -e DYNAMIC_FTRACE \
  -e STACK_TRACER -e IRQSOFF_TRACER -e PREEMPT_TRACER -e SCHED_TRACER \
  -e KPROBES -e KPROBE_EVENTS -e UPROBES -e BPF_SYSCALL -e BPF_JIT -e DEBUG_INFO_BTF \
  -e KCOV -e KCOV_INSTRUMENT_ALL \
  \
  -e FAULT_INJECTION -e FAILSLAB -e FAIL_PAGE_ALLOC -e FAIL_MAKE_REQUEST \
  -e FAULT_INJECTION_DEBUG_FS -e FUNCTION_ERROR_INJECTION \
  \
  -e KUNIT -e KUNIT_ALL_TESTS -m KUNIT_EXAMPLE_TEST \
  -e DYNAMIC_DEBUG -e RANDOMIZE_BASE=n

make O=bdbg olddefconfig && make O=bdbg -j$(nproc)
```
Boot it and measure: it will be **2–5× slower**. That is the price of seeing everything, and
it is always worth paying during development. Keep a second, fast config for performance work.

### Lab 6.7 — Reproduce and read every sanitizer report

Write one module with a switch to trigger each defect class, then read each splat carefully.
Being able to *recognize the report type instantly* is worth more than any single tool.

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>
#include <linux/slab.h>
#include <linux/debugfs.h>

static int which;
module_param(which, int, 0644);

struct obj { int a; struct rcu_head rcu; char buf[16]; };

static int __init bug_init(void)
{
	struct obj *o;
	char *p;
	int arr[4] = {0};
	int i = 8;

	switch (which) {
	case 1:  /* slab-out-of-bounds */
		p = kmalloc(16, GFP_KERNEL);
		p[16] = 'x';
		kfree(p);
		break;
	case 2:  /* slab-use-after-free */
		p = kmalloc(16, GFP_KERNEL);
		kfree(p);
		p[0] = 'x';
		break;
	case 3:  /* double free */
		p = kmalloc(16, GFP_KERNEL);
		kfree(p);
		kfree(p);
		break;
	case 4:  /* stack-out-of-bounds (needs KASAN_STACK) */
		arr[i] = 1;
		pr_info("%d\n", arr[0]);
		break;
	case 5:  /* UBSAN: shift out of bounds */
		pr_info("%d\n", 1 << (i * 8));
		break;
	case 6:  /* memory leak (kmemleak) */
		p = kmalloc(1024, GFP_KERNEL);
		p[0] = 1;   /* keep the optimizer honest */
		break;
	case 7:  /* NULL deref → Oops */
		*(volatile int *)16 = 1;
		break;
	case 8:  /* WARN */
		WARN(1, "deliberate warning, value=%d\n", which);
		break;
	case 9:  /* sleeping in atomic (DEBUG_ATOMIC_SLEEP) */
		preempt_disable();
		msleep(10);
		preempt_enable();
		break;
	}
	return 0;
}
static void __exit bug_exit(void) { }
module_init(bug_init); module_exit(bug_exit);
MODULE_LICENSE("GPL");
```
```bash
for w in 1 2 3 4 5 6 7 8 9; do
  sudo insmod bug.ko which=$w 2>/dev/null; sudo rmmod bug 2>/dev/null
  echo "=== which=$w ==="; sudo dmesg -c | tail -40
done
# For the leak:
echo scan | sudo tee /sys/kernel/debug/kmemleak >/dev/null; sleep 6
echo scan | sudo tee /sys/kernel/debug/kmemleak >/dev/null
sudo cat /sys/kernel/debug/kmemleak | head -30
```
**Write a one-page cheat sheet** mapping each report's first line to its meaning. You will
use it for years.

### Lab 6.8 — Fault injection: exercise your error paths (T.6)

```bash
# Global: make kmalloc fail 1 in 100 times, only for tasks you mark
sudo mount -t debugfs none /sys/kernel/debug 2>/dev/null
cd /sys/kernel/debug/failslab
echo 100  | sudo tee probability      # 1 in ... actually percent*10; read the docs
echo 2    | sudo tee times            # fail this many times then stop
echo 1    | sudo tee task-filter      # only tasks with make-it-fail set
echo 0    | sudo tee verbose

# Mark a task and run it:
echo 1 | sudo tee /proc/self/make-it-fail
# (or use the bundled tool:)
sudo ./tools/testing/fault-injection/failcmd.sh -p 10 -t 1 -- ls /

# Targeted injection on one function (CONFIG_FUNCTION_ERROR_INJECTION):
cat /sys/kernel/debug/error_injection/list | head
sudo bpftrace -e 'kfunc:vfs_read { override(-ENOMEM); }'   # bpf_override_return
```
Then repeat with `FAIL_MAKE_REQUEST` while running `fio`, and with `FAIL_PAGE_ALLOC` while
running a memory-heavy workload. **Error paths you never execute are error paths that do not
work.**

### Lab 6.9 — Run syzkaller against your own driver (T.6)

```bash
# 1. Kernel: enable the fuzzing prerequisites
./scripts/config -e KCOV -e KCOV_INSTRUMENT_ALL -e KASAN -e DEBUG_FS \
                 -e DEBUG_INFO_DWARF4 -e CMDLINE_BOOL -e KALLSYMS_ALL \
                 -e FAULT_INJECTION -e FAULT_INJECTION_DEBUG_FS -e FAILSLAB
make -j$(nproc)

# 2. syzkaller
go install github.com/google/syzkaller/...@latest   # or: git clone && make
# Write a description for your ioctl in sys/linux/dev_mydev.txt:
#   resource fd_mydev[fd]
#   openat$mydev(fd const[AT_FDCWD], file ptr[in, string["/dev/mydev"]],
#                flags const[O_RDWR], mode const[0]) fd_mydev
#   ioctl$MYDEV_SET(fd fd_mydev, cmd const[MYDEV_SET], arg ptr[in, mydev_cfg])
make generate && make

# 3. Run it against a QEMU image
./bin/syz-manager -config my.cfg
# open http://localhost:56741 for coverage and crashes
```
Even a half-hour run against a driver you wrote will usually find something. **Do this before
you post the driver upstream**, not after syzbot does it for you.

### Lab 6.10 — `drgn` for live and postmortem inspection (T.8)

```bash
pip install --user drgn
sudo drgn                      # live kernel
```
```python
>>> prog['jiffies']
>>> from drgn.helpers.linux.pid import for_each_task
>>> for t in for_each_task(prog):
...     if t.__state == 2:     # TASK_UNINTERRUPTIBLE
...         print(t.pid.value_(), t.comm.string_().decode())

>>> from drgn.helpers.linux.list import list_for_each_entry
>>> for m in list_for_each_entry('struct module', prog['modules'].address_of_(), 'list'):
...     print(m.name.string_().decode())

>>> from drgn.helpers.linux.block import for_each_disk
>>> for d in for_each_disk(prog): print(d.disk_name.string_().decode())

# Postmortem on a vmcore:
# drgn -c /var/crash/.../vmcore -s vmlinux
```
Write a `drgn` script that answers a question you actually have about your driver's state.
This skill compounds.

### Lab 6.11 — Practise bisection under time pressure (T.7)

```bash
# Create an artificial regression to practise on:
cd linux && git checkout -b bisect-practice v6.6
# Pick a random commit in v6.6..v6.7 and cherry-pick a "breaking" change, or
# simply bisect a real known regression from the stable tree.

cat > /tmp/test.sh <<'EOF'
#!/bin/bash
set -e
make -j$(nproc) O=/tmp/bb >/dev/null 2>&1 || exit 125   # skip: won't build
timeout 120 ./run-qemu.sh --kernel /tmp/bb/arch/x86/boot/bzImage --test || exit 1
exit 0
EOF
chmod +x /tmp/test.sh

git bisect start v6.7 v6.6
git bisect run /tmp/test.sh
git bisect log > /tmp/bisect.log     # reproducible record — attach this to bug reports
git bisect reset
```
Time the whole thing. Then optimize your `test.sh` and do it again. **Target: full bisection
of one release in under 90 minutes.**

### Lab 6.12 — Defeat a Heisenbug with record/replay (T.2)

Take a genuine race (e.g. the module-unload sins from Ch. 05 Lab 5.D), reproduce it under
QEMU record/replay from Ch. 04 Lab 4.7, then replay under gdb and use `reverse-continue` to
walk backwards from the KASAN report to the moment of the free. Write up what you observed;
this is the single most impressive debugging demo you can give.

---

## 9. Mastery drills

1. Given only `RIP: 0010:ext4_file_write_iter+0x123/0x4a0`, produce the source line
   without running anything but `faddr2line`.
2. Write a `drgn` script that prints every task in `D` state with its kernel stack.
3. Explain the difference between `slab-out-of-bounds`, `slab-use-after-free`,
   `global-out-of-bounds`, and `stack-out-of-bounds` KASAN reports.
4. Find a `WARN_ON_ONCE` in `mm/` and explain what invariant it protects.
5. Enable `CONFIG_LOCK_STAT`, run a `fio` job, and identify the most contended lock.
6. Explain what `objtool` checks and why `CONFIG_UNWINDER_ORC` needs it.
7. **State each tool's blind spot** (T.5) from memory: KASAN, KMSAN, KCSAN, UBSAN,
   kmemleak, lockdep, KFENCE. Then construct one bug that *only* KMSAN finds and one that
   *only* KCSAN finds.
8. **Compute KFENCE's detection probability.** Given `KFENCE_SAMPLE_INTERVAL=100` (ms),
   `KFENCE_NUM_OBJECTS=255`, an allocation rate of 100k/s, and a buggy allocation executed
   1000 times/day on one machine — what is the per-machine daily detection probability?
   Across a 100,000-machine fleet? This calculation is why Google and Meta run KFENCE in
   production. Show your work.
9. **Write the hypothesis.** Take any open syzbot report (https://syzkaller.appspot.com/upstream)
   with a C reproducer. Before reading the discussion, write down a falsifiable hypothesis
   and the experiment that would test it (T.1). Then compare with the actual fix.
10. **Probe effect, demonstrated.** Write a module with a tight race between two kthreads.
    Show it reproduces with `trace_printk()` instrumentation and *stops* reproducing with
    `printk()`. Measure the difference in instrumentation cost to explain why.
11. **KCOV yourself.** Use `KCOV` directly (see `Documentation/dev-tools/kcov.rst`) to
    measure what fraction of your own driver's lines your test suite reaches. Report the
    number. Most drivers are below 40%.
12. **Read a real Oops end to end.** Find one in `dmesg` on any machine, or from a syzbot
    report, and annotate every single line: the `RIP`, the `Code:` bytes, the register dump,
    `CR2`, the "tainted" flags, the call trace, and the `---[ end trace ]---`. Do this three
    times and Oops-reading becomes automatic.
13. **kdump on real hardware.** Configure `crashkernel=`, verify with `kdumpctl status` or
    equivalent, trigger with `echo c > /proc/sysrq-trigger`, and analyze the resulting
    vmcore with both `crash` and `drgn`. Document the setup as a runbook — you will need it
    in production.

---

## 10. Further reading

**Kernel documentation:**
- `Documentation/dev-tools/` ★★ — **read the whole directory**: `kasan.rst`, `kcsan.rst`,
  `kmsan.rst`, `kfence.rst`, `kmemleak.rst`, `ubsan.rst`, `kgdb.rst`,
  `gdb-kernel-debugging.rst`, `kcov.rst`, `kunit/`, `fault-injection/`, `checkpatch.rst`,
  `sparse.rst`, `coccinelle.rst`, `testing-overview.rst`
- `Documentation/admin-guide/bug-hunting.rst` ★
- `Documentation/admin-guide/bug-bisect.rst`
- `Documentation/admin-guide/reporting-issues.rst` — what a good bug report contains
- `Documentation/admin-guide/kdump/kdump.rst`
- `Documentation/admin-guide/sysrq.rst` — memorize `l`, `t`, `w`, `m`, `c`
- `Documentation/trace/` — ftrace, kprobes, uprobes, events, histogram triggers
- `Documentation/locking/lockdep-design.rst`

**Books:**
- Zeller, *Why Programs Fail: A Guide to Systematic Debugging* — **the theory in T.1**;
  the best book on debugging ever written
- Agans, *Debugging: The 9 Indispensable Rules* — short, practical, quotable
- Gregg, *BPF Performance Tools* and *Systems Performance* (2nd ed.) — rank 3–4 tooling,
  exhaustively
- Matloff & Salzman, *The Art of Debugging with GDB, DDD, and Eclipse*

**Papers:**
- Gray, "Why Do Computers Stop and What Can Be Done About It?" (1985) — Bohrbugs/Heisenbugs
- Zeller & Hildebrandt, "Simplifying and Isolating Failure-Inducing Input" (TSE 2002) —
  delta debugging
- Serebryany, Bruening, Potapenko, Vyukov, "AddressSanitizer: A Fast Address Sanity Checker"
  (USENIX ATC 2012) — **the shadow-memory design in T.4**
- Serebryany & Iskhodzhanov, "ThreadSanitizer" (WBIA 2009)
- Stepanov & Serebryany, "MemorySanitizer: Fast Detector of Uninitialized Memory Use"
  (CGO 2015)
- Zalewski, "american fuzzy lop" technical whitepaper — coverage-guided fuzzing
- Vyukov & Konovalov, syzkaller design docs (in-repo `docs/`) — structure-aware kernel fuzzing
- Boehm, "Space Efficient Conservative Garbage Collection" (PLDI 1993) — the theory behind
  kmemleak

**Tools & sites:**
- drgn: https://drgn.readthedocs.io — **learn this**
- `crash` white paper: https://crash-utility.github.io
- syzbot dashboard: https://syzkaller.appspot.com/upstream — open bugs, with reproducers
- Brendan Gregg's site: https://brendangregg.com — flame graphs, perf, BPF
- `bcc`/`bpftrace` tool galleries: `/usr/share/bcc/tools`, `bpftrace -l`

**LWN:**
- "The kernel address sanitizer", "KFENCE: a low-overhead memory-safety error detector"
- "Finding kernel bugs with syzkaller" / "Two years of syzbot"
- "drgn: a debugger-as-a-library"
- "Reliable stack traces and ORC unwinding", "objtool: a valuable tool"

→ Next: [07-reading-code.md](07-reading-code.md)

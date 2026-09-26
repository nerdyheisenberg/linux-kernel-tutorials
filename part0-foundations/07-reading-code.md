# Chapter 07 — Reading Kernel Code Fast

> **Goal:** you can be handed an unfamiliar subsystem and produce an accurate architecture
> summary in one working day. This is *the* senior skill.

---

## Theory & First Principles

> **How to read this section.** T.0 is a worked reading, start to finish, of code you have not
> seen. T.1–T.3 are why reading is hard and what the research says. T.4–T.8 are the technique
> and the stopping rule.

---

### T.0 — Start here: read something, right now, with a method

The instinct on opening an unfamiliar kernel file is to start at line 1. That is the worst
available strategy and it is why people bounce off the tree. Here is the method, applied.

**The question:** *what happens when a process calls `dup2(oldfd, newfd)`?*

```bash
cd ~/src/linux

# ① FIND THE ENTRY POINT. Syscalls are defined by a macro, so grep for the macro.
grep -rn 'SYSCALL_DEFINE.*dup2' fs/
#   fs/file.c:SYSCALL_DEFINE2(dup2, unsigned int, oldfd, unsigned int, newfd)

# ② READ THE ENTRY FUNCTION ONLY. Do not follow any call yet. 20 lines.
sed -n '/SYSCALL_DEFINE2(dup2/,/^}/p' fs/file.c

# ③ NOW pick the ONE call that does the real work and follow it -- once.
grep -n 'do_dup2\|ksys_dup3' fs/file.c | head

# ④ CHECK THE COMMIT MESSAGE for any line you don't understand:
git log -L '/static int do_dup2/,/^}/:fs/file.c' | head -60

# ⑤ CHECK WHO ELSE CALLS IT -- callers define the real contract:
grep -rn 'do_dup2\|replace_fd' --include='*.c' . | head
```

**Five moves, in that order, every time.** Notice what you did *not* do: you did not read
`fs/file.c` (1,300 lines). You read perhaps 60 lines, chosen by the question.

**Now the observations that the rest of the chapter formalizes:**

1. **You navigated by *grammar*, not by search.** You knew syscalls live in a
   `SYSCALL_DEFINE` macro, that file-descriptor code is in `fs/file.c`, and that a `ksys_`
   prefix means "the internal entry point that both the syscall and in-kernel callers use."
   That vocabulary is §T.5's "beacons," and Ch. 02 §T.6 is where you acquire it.
2. **You followed exactly one call.** Depth-first reading of a call graph is how you lose an
   afternoon. Experts read **breadth-first and shallow**, building a map before descending
   (§T.2 — this is the measured difference between novices and experts, not a style claim).
3. **You asked `git log` why.** The code says *what*; the commit message says *why*, and the
   *why* is what Naur (§T.1) argues is the actual program. A surprising line with an
   explanatory commit message is solved; a surprising line without one is a risk.
4. **You looked at the callers.** A function's real contract is the intersection of what its
   callers assume. This is how you recover undocumented locking rules, undocumented context
   requirements, and undocumented lifetime rules — and it is the core technique of Ch. 84
   §T.10, where you must extract a C contract to write a Rust abstraction against it.
5. **You had a question before you started.** Reading without one has no stopping rule, so it
   never stops, so it never feels like progress. §T.6.

**The honest framing for a beginner:** you will never read the kernel. Nobody has. Linus has
not. What you build instead is (a) a map of where things are, (b) a vocabulary of recurring
patterns, and (c) the ability to answer a specific question in ten minutes. That is what
"knowing the kernel" actually means in practice, and it is achievable in months rather than
decades.

---

### T.1 Naur: the source code does not contain the program

Peter Naur, *"Programming as Theory Building"* (1985) — a short paper that will change how
you think about this entire curriculum:

> Programming is not the production of text; it is the building, in the programmers' minds,
> of a **theory** of how the problem and its solution relate. The program text and the
> documentation are *residues* of that theory, and they are **necessarily incomplete**.

Naur's evidence: a team that inherits a well-documented program written by someone else
consistently makes modifications that are technically correct but *architecturally wrong* —
they violate invariants that were never written down because the original authors considered
them obvious. The theory died with the team.

The consequences for kernel work are direct and practical:

1. **Reading the code is necessary but not sufficient.** The invariants — "this pointer is
   only valid under RCU", "this lock must be taken before that one", "this function may be
   called from softirq" — are largely *not in the code*. You must reconstruct the theory.
2. **The `git log` is the closest thing to a record of the theory**, because commit messages
   explain *why*. The diff shows *what*. A senior engineer reads commit messages more than
   code.
3. **A maintainer's real job is holding the theory**, which is why maintainership doesn't
   transfer by handing over a repository, and why `MAINTAINERS` is an architectural document
   (Ch. 02 T.4).
4. **Your commit messages are the only chance to transmit your theory.** This reframes
   Ch. 86's patch-writing discipline from bureaucracy into knowledge transfer.

### T.2 What the research says about how experts read code

Program comprehension has been studied empirically since the 1980s, and the findings
contradict what most people do.

**Brooks (1983) — top-down, hypothesis-driven.** Expert readers do *not* read linearly.
They form a hypothesis about what the code does from its name/context, then search for
**beacons** — recognizable idioms that confirm or refute it. A `for (;;) { ... schedule(); }`
is a beacon for "kernel thread main loop". `container_of` is a beacon for "intrusive
container callback". Expertise is largely a *library of beacons*.

**Pennington (1987) — two mental models.** Readers build a **program model** (control flow,
"what happens next") and a **domain model** ("what does this mean in terms of the problem").
Novices build only the program model and get lost. Experts build the domain model *first* —
which is why this curriculum insists you understand *why a subsystem exists* before reading
it.

**Littman et al. (1986) — systematic vs as-needed.** Programmers who read a system
*systematically* (building a complete causal model before changing anything) succeeded at
modification tasks; those who read *as-needed* (only what seemed relevant) failed — because
they missed interactions. The kicker: as-needed is faster and feels more productive, which
is why everyone does it. For a 40-million-line tree you obviously cannot be systematic about
*everything*; the resolution is: **be systematic about a bounded region, as-needed outside
it, and know precisely where your boundary is.**

**Chase & Simon (1973) / chunking.** Chess masters don't see 32 pieces; they see ~6 familiar
patterns. Same board randomized, their advantage vanishes. Applied to code: expertise is not
reading speed, it is **chunk size**. When you see

```c
rcu_read_lock();
p = rcu_dereference(gp);
if (p && refcount_inc_not_zero(&p->ref)) { ... }
rcu_read_unlock();
```
a novice reads 5 statements; you should read **one chunk**: "RCU lookup with reference
acquisition" (Ch. 12 T.3). Building chunks is what Parts 1–3 of this curriculum are for, and
it is why memorizing the *names* of patterns matters — a named chunk is a retrievable chunk.

**Cognitive load (Sweller).** Working memory holds ~4 chunks. So any reading strategy that
requires holding more than four unresolved things at once will fail. Practical technique:
**write things down immediately** — every time you resolve "what is this struct?", record it
and free the slot. This is the entire justification for the note-taking template later in
this chapter. It is not bureaucracy; it is working-memory management.

### T.3 The economics: you will never read it all

```
  Linux ≈ 40M lines.  At a generous 100 lines/hour of genuine comprehension:
     400,000 hours  ≈  200 person-years of full-time reading.
```
So *selective reading with a stopping rule* is not laziness, it is the only viable strategy.
The question is never "have I read enough?" but **"what question am I trying to answer, and
have I answered it?"**

Define the question before you open a file. Examples of good questions:
- "Under what lock is `foo->bar` modified?"
- "What is the complete set of callers of this callback?"
- "What happens if `probe()` fails halfway through?"
- "Which context does this run in?"

Examples of bad (unbounded) questions: "How does ext4 work?", "Let me understand the
scheduler."

The corollary is the **90/10 rule of subsystem reading**: in any subsystem, ~10% of the code
(the central data structure, the ops table, the main state machine, the init and teardown
paths) explains ~90% of the behaviour. The other 90% is error handling, hardware quirks,
and variants. Find the 10% first, deliberately.

### T.4 The three artifacts that carry the theory

For any kernel subsystem, the theory (T.1) lives in four places, in descending order of
reliability:

| Source | Reliability | Why |
|---|---|---|
| **The code** | 100% accurate about *what*, 0% about *why* | it is the ground truth, by definition |
| **`git log` / mailing-list threads** | very high | written at the moment of the decision, by the person who made it |
| **`Documentation/`** | high where it exists | reviewed, but may lag |
| **Comments** | variable | may be stale; kernel comments are usually good but not guaranteed |
| **Blog posts / books / LLMs** | low | almost always describe an older kernel |

The single most under-used tool is **`git log -S` / `git log -L` / `git blame`** applied as
*archaeology*. When you find code you don't understand, don't stare at it — **find the commit
that introduced it and read the message.** In the kernel, that message almost always contains
the reasoning, the benchmark, or the bug report that motivated it. And the `Link:` tag takes
you to the full mailing-list discussion, where the *rejected alternatives* are — which is
often more informative than the accepted design.

```bash
git log -S'some_exact_string' --oneline -- path/     # when was this introduced/removed?
git log -L :function_name:file.c                      # full history of one function
git log --follow -p -- path/file.c                    # across renames
git blame -w -C -C -L 100,120 file.c                  # ignore whitespace, detect moves
git log --grep='keyword' --all
git show <sha>                                        # read the FULL message, incl. Link:
```

### T.5 Beacons: the kernel-specific pattern library

This is the practical form of T.2's "expertise is a library of beacons". Learn to recognize
these instantly; each collapses many lines into one chunk.

| Beacon | Means |
|---|---|
| `container_of(ptr, type, member)` | intrusive container; `ptr` is embedded in a bigger struct |
| `struct foo_ops { ... }` + `->op()` calls | polymorphism / a subsystem contract |
| `*_register()` / `*_unregister()` pairs | a plugin registering with a core |
| `devm_*` | resource tied to device lifetime (Ch. 28) |
| `rcu_read_lock()` … `rcu_dereference()` | lockless read path (Ch. 15) |
| `refcount_inc_not_zero()` | weak→strong reference upgrade; object may be dying (Ch. 12) |
| `spin_lock_irqsave()` | this lock is also taken from hardirq context |
| `WRITE_ONCE`/`READ_ONCE` | concurrent access without a lock — read the memory model (Ch. 13) |
| `smp_store_release` / `smp_load_acquire` | publish/subscribe of an initialized object |
| `__init` / `__exit` / `__initdata` | discarded after boot / module unload |
| `__user` | userspace pointer; must use `copy_*_user` |
| `__percpu` | per-CPU; needs `this_cpu_ptr()` (Ch. 16) |
| `__must_check` / `__printf(x,y)` | compiler-enforced contract |
| `goto err_*:` ladder | hand-written destructors, reverse order (Ch. 05 T.6) |
| `guard(...)` / `__free(...)` | RAII (modern equivalent of the above) |
| `ERR_PTR` / `IS_ERR` | pointer-or-error return (Ch. 05 T.6) |
| `-EPROBE_DEFER` | driver dependency not ready yet (Ch. 27) |
| `static_branch_likely()` | runtime-patched branch; near-zero cost when disabled |
| `WARN_ON_ONCE(cond)` | an invariant the author believed could not be violated |
| `lockdep_assert_held(&l)` | **documentation of the locking contract, machine-checked** |
| `might_sleep()` | "this function may sleep" — a context contract |
| `smp_processor_id()` in non-preempt-disabled code | probably a bug |

`lockdep_assert_held()` deserves special mention: it is the closest thing the kernel has to a
*written, checkable* statement of an invariant, and in Naur's terms it is a fragment of the
theory successfully encoded into the text. When you write kernel code, adding these is how
you transmit your theory to the next reader.

### T.6 A stopping rule

You have understood a subsystem *well enough for your task* when you can answer, without
looking:

1. What is the **central data structure**, and what owns it?
2. What are the **three main paths**: initialization, the hot path, and teardown?
3. What is the **locking model**, in one sentence per lock?
4. What **contexts** can each entry point be called from?
5. What is the **userspace-visible contract** (syscall/sysfs/ioctl/netlink)?
6. Where would you **add a tracepoint** to observe it?

If you can answer those six, stop reading and start working. If you cannot answer #3, you are
not ready to modify anything.

---

### T.7 — What is still argued about

**1. Should the kernel have more comments?**
The dominant view is that comments explaining *what* rot and lie, while commit messages
explaining *why* are versioned with the code and cannot drift. The counter-view is that
`git log` archaeology has a real cost that a one-line comment would eliminate, and that
"read the commit" is a burden shifted from one author onto every future reader. Both are
true; the kernel's compromise — comments for *invariants and contracts*, commit messages for
rationale — is a reasonable settlement and is what reviewers enforce.

**2. Is kernel-doc worth the effort?**
It produces navigable API documentation and is machine-checkable (`make htmldocs` fails on
malformed entries). It also rots exactly like any comment, and the coverage is patchy enough
that its absence carries no information. The honest position: kernel-doc on a *published
interface* is valuable; on an internal static function it is usually noise.

**3. Do LLMs change how you should read code?**
They are good at summarizing an unfamiliar function and terrible at knowing whether the
summary is true — which is precisely the wrong failure mode for a kernel. The defensible use
is as a *search and orientation* tool ("where is path resolution implemented?"), followed by
reading the actual code. Using a summary as evidence in a review or a design argument is how
you lose credibility permanently. Note also the `Signed-off-by` question this raises
(Ch. 86 §T.5).

**4. How much should you read before contributing?**
The apprenticeship view says months of reading first. The empirical view is that people who
send a small patch in week one learn faster, because review is the highest-bandwidth teaching
mechanism available and reading alone gives no feedback signal. **The evidence favours the
second**, with the caveat that the patch must be small and genuinely useful.

### T.8 — The compressed model

```
 YOU WILL NEVER READ THE KERNEL. Nobody has. What you build is (a) a MAP,
 (b) a VOCABULARY of recurring patterns, (c) the ability to answer a
 SPECIFIC QUESTION in ten minutes.

 ALWAYS HAVE A QUESTION FIRST. Reading without one has no stopping rule.

 READ BREADTH-FIRST AND SHALLOW. Map before you descend. Depth-first
 following of a call graph is how experts spot novices.

 THE CODE SAYS WHAT; THE COMMIT MESSAGE SAYS WHY. Naur: the program is the
 THEORY in the author's head, and the source is a lossy projection of it.
 The three artifacts that carry the theory are the commit message, the
 mailing-list thread, and the test.

 THE REAL CONTRACT OF A FUNCTION IS THE INTERSECTION OF WHAT ITS CALLERS
 ASSUME. That is how you recover undocumented locking, context, and
 lifetime rules.
```

The five-move procedure, which does not change for the rest of your career:

1. **Find the entry point.** Grep for the macro or the registered ops struct, not the name.
2. **Read that function only.** Follow nothing yet.
3. **Follow exactly one call** — the one that does the real work.
4. **`git log -L` any line that surprises you.** A surprising line with an explanatory commit
   is solved; one without is a risk you have just found.
5. **Read the callers** to recover the contract nobody wrote down.

---

## 1. Concept: you will never read it all

40M lines. 6,000 commits per release. ~70 releases of history. Nobody reads it all.
What separates experts is a **repeatable method** for building a correct partial model fast.

The method has a name in this curriculum: **SCOPE**.

```
S — Struct        Find the central data structure.
C — Contract      Find the ops table(s). That's the API surface.
O — Ownership     Find who allocates, refcounts, and frees it.
P — Paths         Trace init, fast path, teardown.
E — Evidence      Tracepoints, debugfs, sysfs, selftests, git log.
```

---

## 2. The SCOPE method, applied

Let's actually do it for a subsystem you don't know yet: **the block layer**.

### S — Struct
```bash
ls include/linux/blk*.h
git grep -n 'struct request_queue {' include/
```
→ `include/linux/blkdev.h`. Read `struct request_queue`, `struct gendisk`,
`struct block_device`. In `include/linux/blk-mq.h`: `struct request`,
`struct blk_mq_tag_set`, `struct blk_mq_hw_ctx`. In `include/linux/blk_types.h`:
`struct bio`, `struct bio_vec`.

**Draw the ownership graph on paper.** `gendisk` → `request_queue` → `blk_mq_hw_ctx[]` →
`blk_mq_tags` → `request[]`. `block_device` → `gendisk`.

### C — Contract
```bash
git grep -n 'struct blk_mq_ops {' -A40 include/linux/blk-mq.h
git grep -n 'struct block_device_operations {' -A30 include/linux/blkdev.h
```
`blk_mq_ops` has ~15 members. That is the *entire* contract a storage driver must implement:
`queue_rq`, `commit_rqs`, `complete`, `init_hctx`, `init_request`, `timeout`, `poll`,
`map_queues`. Fifteen functions = you now know what a block driver is.

### O — Ownership
```bash
git grep -n 'blk_mq_alloc_disk\|put_disk\|del_gendisk\|blk_mq_free_tag_set'
git grep -n 'bio_alloc\|bio_put\|bio_endio'
```
Note the refcounts: `gendisk` via `kobject`, `bio` via `bi_remaining` for chained bios,
`request` via tags (a bitmap, not a refcount — interesting design choice, note it).

### P — Paths
**Init:** `null_blk` is the reference driver.
```bash
git grep -n 'blk_mq_alloc_tag_set\|blk_mq_alloc_disk\|add_disk' drivers/block/null_blk/main.c
```
**Fast path:**
```
submit_bio()                     block/blk-core.c
 → submit_bio_noacct()
 → __submit_bio() → disk->fops->submit_bio()  (bio-based drivers)
                  → blk_mq_submit_bio()        (request-based)
      → blk_mq_get_new_requests() / bio merging / plug
      → blk_mq_dispatch_rq_list()
      → q->mq_ops->queue_rq()    ← driver entry
...driver completes...
 → blk_mq_complete_request() → blk_mq_end_request() → bio_endio() → bi_end_io()
```
**Teardown:** `del_gendisk()` → `blk_mq_freeze_queue()` → `put_disk()` →
`blk_mq_free_tag_set()`.

### E — Evidence
```bash
cat include/trace/events/block.h     # every significant moment, named
ls /sys/kernel/debug/block/          # blk-mq debugfs: per-hctx state, tags, dispatch
ls /sys/block/sda/queue/             # tunables ARE the design, documented
ls Documentation/block/
ls tools/testing/selftests/... ; ls -d blktests   # external but essential
```

`/sys/block/*/queue/*` is a *specification in disguise*: `nr_requests`, `scheduler`,
`rotational`, `max_sectors_kb`, `nomerges`, `io_poll`, `write_cache`, `zoned`,
`discard_granularity`… each one names a concept you must understand.

**In four hours you now understand the block layer better than most people who "know Linux".**

---

## 3. Tools, in the order you should reach for them

### 3.1 `git grep` mastery
```bash
git grep -n 'blk_mq_ops'                        # basic
git grep -nW 'static int null_queue_rq'         # -W: whole function
git grep -n -p 'bi_end_io'                      # -p: show enclosing function header
git grep -l 'blk_mq_ops' -- drivers/            # files only
git grep -n 'EXPORT_SYMBOL.*bio_'               # what's exported
git grep -n -e 'folio' --and -e 'dirty' -- mm/  # AND
git grep -n --all-match -e A -e B
git grep -c 'kmalloc' -- drivers/ | sort -t: -k2 -n | tail   # counts per file
git grep -n 'TODO\|FIXME\|XXX' -- fs/ext4/      # where the bodies are buried
```

### 3.2 `git log` as documentation
```bash
git log --oneline -30 -- block/blk-mq.c
git log -S'blk_mq_alloc_disk' --oneline --reverse | head -1   # the introducing commit
git log -L :blk_mq_submit_bio:block/blk-mq.c | head -200      # function history
git log --grep='multiqueue' --oneline
git show --stat <sha>
git log --format='%h %ad %an %s' --date=short -- fs/iomap/
git shortlog -sn -- drivers/nvme/ | head        # who actually owns this
```

**The introducing commit is the design doc.** For `blk-mq`, `git log --reverse` on
`block/blk-mq.c` gives you Jens Axboe's original rationale.

### 3.3 `cscope` / `ctags`
```bash
make ARCH=x86 cscope tags -j$(nproc)
cscope -d
# 0: find symbol   1: global definition   2: functions called by
# 3: functions calling this   4: text string   6: egrep   7: file   8: #including
```
`3: functions calling this` is the killer feature — call-graph upward.

### 3.4 clangd + your editor
Set up per Chapter 02. Gives you: go-to-definition across macros, find-references,
type-on-hover for `void *` chains, call hierarchy.

### 3.5 Elixir (web)
https://elixir.bootlin.com/linux/v6.12/source — identifier search with
"defined in / referenced in" split, and a version selector for API archaeology.

### 3.6 `b4` — fetch patch series from lore
```bash
pipx install b4
b4 mbox -o . <message-id>            # get a whole series
b4 am -o . <message-id>              # ready-to-apply mbox
b4 shazam <message-id>               # apply directly to current branch
b4 diff <message-id>                 # interdiff between series versions
b4 ty                                # generate thank-you notes (maintainer side)
```
Reading the **mailing list discussion** around a patch is often more informative than
the code. `lore.kernel.org` has everything since 1998.

### 3.7 `bpftrace` — read the code by watching it run
```bash
# Who calls this function, and how often?
bpftrace -e 'kprobe:blk_mq_submit_bio { @[kstack] = count(); }'

# What are the arguments?
bpftrace -e 'kprobe:vfs_write { printf("%s %d bytes\n", comm, arg2); }'

# Latency distribution
bpftrace -e 'kprobe:blk_mq_submit_bio { @s[tid]=nsecs; }
             kretprobe:blk_mq_submit_bio /@s[tid]/ { @us=hist((nsecs-@s[tid])/1000); delete(@s[tid]); }'
```
This answers questions static reading can't: *which* branch is actually taken.

### 3.8 `ftrace` function graph — the X-ray
```bash
cd /sys/kernel/tracing
echo function_graph > current_tracer
echo blk_mq_submit_bio > set_graph_function
echo 5 > max_graph_depth
echo 1 > tracing_on; dd if=/dev/zero of=/tmp/f bs=4k count=1 oflag=direct; echo 0 > tracing_on
cat trace | head -100
```
Output is literally the call tree with timings. Nothing beats it for "what actually runs".

---

## 4. Reading hard C: kernel-specific patterns

### 4.1 `container_of` — the single most important macro
```c
#define container_of(ptr, type, member) ...
```
Given a pointer to a *member*, get the pointer to the *containing struct*.
This is how the kernel does polymorphism without C++ inheritance.

```c
struct my_dev {
	struct device dev;       /* "base class" */
	int my_field;
};
static void my_release(struct device *d) {
	struct my_dev *m = container_of(d, struct my_dev, dev);
	kfree(m);
}
```
When you see `void *private_data`, `->driver_data`, or an embedded struct, the
`container_of` upcast is nearby. **Trace it; it tells you the type hierarchy.**

Modern variants: `container_of_const()`, `list_entry()`, `hlist_entry()`,
`to_platform_device()`, `to_pci_dev()`, `I_BDEV()`, `EXT4_SB()` — all are `container_of`.

### 4.2 Ops tables = interfaces
```c
struct file_operations { ... };
struct inode_operations { ... };
struct blk_mq_ops { ... };
struct net_device_ops { ... };
struct dev_pm_ops { ... };
```
When you find one, you've found the subsystem's whole API. Write out the member list;
that's your study plan.

### 4.3 Macro mazes
Use the preprocessor:
```bash
make O=b fs/ext4/inode.i
grep -n 'ext4_write_begin' b/fs/ext4/inode.i | head
```
Or ask clangd to expand inline. For `SYSCALL_DEFINE3`, `DEFINE_PER_CPU`,
`TRACE_EVENT`, `module_platform_driver`, `DEFINE_SIMPLE_ATTRIBUTE`,
`__ATTR_RW`, `LIST_HEAD` — always preprocess once, then you know it forever.

### 4.4 Annotations that carry meaning
```c
__user      /* pointer into user space — sparse enforces */
__iomem     /* MMIO — must use readl/writel */
__percpu    /* per-CPU offset, not a pointer */
__rcu       /* must use rcu_dereference() */
__kernel    /* default address space */
__force     /* shut sparse up; a red flag if unexplained */
__must_check /* caller must use return value */
__acquires(lock) / __releases(lock)   /* sparse lock context */
__must_hold(lock)
__bitwise   /* strong typedef: __le32, __be16 */
__attribute__((packed)) / __packed
__aligned(n) / __cacheline_aligned
__read_mostly / __ro_after_init
__init / __exit / __initdata / __initconst / __ref
__maybe_unused / __used / __always_unused
noinline_for_stack / __noreturn / __cold
__counted_by(n)      /* flexible array with count member — 6.6+ */
```

Run `make C=1` regularly; sparse checks most of these.

### 4.5 Locking annotations in comments
Good kernel code documents lock ownership *in the struct*:
```c
struct foo {
	spinlock_t	lock;
	int		a;	/* protected by lock */
	int		b;	/* RCU-protected, see foo_get() */
	int		c;	/* immutable after init */
};
```
When it doesn't, reconstructing it **is** your job. Write it down as you go.

---

## 5. A one-day subsystem study template

Copy this into a notes file for every subsystem you learn.

```markdown
# Subsystem: <name>

## 1. One-sentence purpose

## 2. Key headers
- include/linux/...

## 3. Central data structures
| Struct | Where | Role | Lifetime/refcount |
|---|---|---|---|

## 4. Ops tables (the contract)
| Callback | Required? | Context (task/atomic) | Notes |
|---|---|---|---|

## 5. Registration API
- register_x() / unregister_x()
- Who calls it, when

## 6. Call paths
### Init
### Fast path
### Error path
### Teardown

## 7. Locking model
- Which lock protects what
- RCU usage
- Contexts

## 8. Userspace interface
- syscalls / ioctls / sysfs / debugfs / netlink / procfs

## 9. Tracepoints
- (from include/trace/events/<x>.h)

## 10. Reference implementations to read
- simplest driver/user:
- most complete:

## 11. Recent direction (last 50 commits)

## 12. Open questions
```

---

## 6. Where to find the *real* documentation

| Source | Value |
|---|---|
| `Documentation/` | Official, often excellent, sometimes stale |
| **Commit messages** | ★★★ Best design rationale in the project |
| **LWN.net** | ★★★ The journal of record. `lwn.net/Kernel/Index/` |
| `lore.kernel.org` | Every discussion ever. Searchable |
| `include/trace/events/*.h` | Data-flow documentation as code |
| `samples/`, `tools/testing/selftests/` | Working examples |
| `Documentation/ABI/` | The userspace contract, per file |
| `Documentation/devicetree/bindings/` | Hardware description schemas |
| KernelNewbies | https://kernelnewbies.org/LinuxChanges — per-release summaries |
| `MAINTAINERS` | Who to ask; also which files belong together |
| Conference talks | LPC, Kernel Recipes, FOSDEM, LSF/MM — on YouTube |

**Build the habit:** read LWN's weekly Kernel page every week. Over a year you absorb
the entire direction of the project.

---

## 7. Practice

### Lab 7.1 — SCOPE a subsystem you don't know
Pick one: `drivers/nvmem/`, `drivers/watchdog/`, `net/bridge/`, `fs/ramfs/`.
Fill in the template from §5. Time-box: 4 hours. Compare against
`Documentation/` afterwards.

### Lab 7.2 — Archaeology
Find when `folio` was introduced. Read Matthew Wilcox's cover letter on lore.
Summarize why `struct page` had to change.

### Lab 7.3 — lore + b4
```bash
b4 mbox -o /tmp $(some message-id from lore)
```
Pick a recent RFC series in a subsystem you like. Read the review comments.
Note what reviewers objected to — this teaches you upstream taste faster than anything.

### Lab 7.4 — Dynamic reading
Pick a syscall. Use `ftrace function_graph` with `set_graph_function` to print the real
call tree. Compare to what you predicted from static reading. Note every surprise.

---

## 8. Mastery drills

1. Without the internet: find how `fallocate(FALLOC_FL_PUNCH_HOLE)` reaches ext4's
   extent code. List every function.
2. Reconstruct the locking model of `struct address_space` purely from source, then check
   against `Documentation/filesystems/locking.rst`. How many did you get right?
3. Use `git log -L` to trace `__alloc_pages` over 10 years. Write a 1-page evolution story.
4. Find three places where `container_of` is used to implement what would be
   inheritance in C++. Draw the class hierarchies.
5. Subscribe to one subsystem mailing list (`linux-block`, `linux-nvme`, `rust-for-linux`)
   and read it daily for two weeks. This single habit changes your trajectory.
6. **Naur exercise.** Pick a 50-line function in `mm/` or `fs/`. Write down every invariant
   it *assumes* but does not state (locking, context, ordering, lifetime, alignment,
   nullability). Then find the commits and `Documentation/` that confirm or refute each.
   Count how many you had to reconstruct rather than read. That count is T.1 made concrete.
7. **Beacon drill.** Take 20 random functions from `drivers/` (use
   `git grep -n '^static .*(' drivers/ | shuf -n 20`). For each, spend **60 seconds only**
   and write down: context, locking, what it does. Then verify. Repeat weekly. Track your
   accuracy — this is deliberate practice for T.2's chunking.
8. **Archaeology.** Find a piece of kernel code that looks obviously wrong or redundant.
   Use `git log -S`/`git blame` to find why it's there. (There is almost always a reason —
   a hardware erratum, a userspace regression, a race.) Write up what you found. Do this
   three times; it will permanently cure you of "I'll just clean this up".
9. **Rejected alternatives.** Pick a major subsystem feature (e.g. `io_uring`, `folios`,
   `EEVDF`, `maple tree`). Find the original RFC posting on lore.kernel.org and read the
   *whole* thread. List the alternatives proposed and why each was rejected. This is the
   highest-density source of design judgement available anywhere.
10. **Apply the stopping rule (T.6).** Choose a subsystem you've never read (suggested:
    `drivers/char/hw_random/`, `fs/ramfs/`, `net/packet/`, `drivers/base/faux.c`). Answer
    all six questions in writing in under two hours. Then have someone check you against
    the source.
11. **Build your beacon glossary.** Extend the T.5 table with 20 more beacons you encounter
    over the next month. Keep it in `reference/`. This document is your externalized
    expertise.
12. **Systematic vs as-needed (T.2).** Deliberately do one task each way: fix a bug in a
    subsystem after (a) reading only what seemed relevant, and (b) building a complete model
    of one file first. Compare time taken and confidence. Littman's result should replicate.

---

## 9. Further reading

**Kernel process documentation (read these this week):**
- `Documentation/process/howto.rst` ★ (read it today)
- `Documentation/process/submitting-patches.rst` ★
- `Documentation/process/development-process.rst`
- `Documentation/process/coding-style.rst`
- `Documentation/process/maintainer-handbooks.rst` — per-subsystem expectations
- `Documentation/process/researcher-guidelines.rst`

**Papers on program comprehension (T.1, T.2) — short and genuinely useful:**
- Naur, "Programming as Theory Building" (1985) — **read this one; it is 8 pages**
- Brooks, "Towards a Theory of the Comprehension of Computer Programs" (IJMMS 1983)
- Pennington, "Stimulus Structures and Mental Representations in Expert Comprehension of
  Computer Programs" (Cognitive Psychology 1987)
- Littman, Pinto, Letovsky, Soloway, "Mental Models and Software Maintenance" (1986)
- Chase & Simon, "Perception in Chess" (Cognitive Psychology 1973) — chunking
- von Mayrhauser & Vans, "Program Comprehension During Software Maintenance and Evolution"
  (IEEE Computer 1995) — the integrated model
- Ericsson, "Deliberate Practice and Acquisition of Expert Performance" — why drill 7 works

**Books:**
- Spinellis, *Code Reading: The Open Source Perspective* — the only book on this skill;
  uses real open-source code including kernels
- Feathers, *Working Effectively with Legacy Code* — techniques for changing code you
  don't fully understand, which is every kernel change you will ever make
- Love, *Linux Kernel Development* — for narrative; then stop reading books and read the tree

**Sites and tools:**
- https://lwn.net/Kernel/Index/ — the subsystem index; your first stop for any topic
- https://lore.kernel.org — **the archive**; `b4 mbox <msgid>` pulls a whole thread locally
- https://elixir.bootlin.com — cross-referenced source, all versions
- https://kernelnewbies.org/KernelHacking
- https://syzkaller.appspot.com/upstream — real bugs with reproducers
- `scripts/get_maintainer.pl`, `git log --format='%an <%ae>' -- path | sort | uniq -c | sort -rn`

→ Next: [../part1-kernel-c/08-kernel-c-dialect.md](../part1-kernel-c/08-kernel-c-dialect.md)

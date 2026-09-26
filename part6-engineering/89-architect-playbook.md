# Chapter 89 — The Architect's Playbook: API Design and Technical Judgement

> The last chapter of Part 6, and the capstone of the whole curriculum's *judgement* thread.
> Everything before this taught mechanism. This chapter is about the decisions: what to
> build, where to put it, what to refuse, and how to be right about it in a way that
> survives contact with other people.
>
> It is also the material that principal-level interviews are actually testing, and it maps
> directly onto `reference/system-design.md` and `reference/interview-playbook.md` §7.

---

## Theory & First Principles

### T.0 — Start here: the same eight ideas, eighty-nine times

You have now read eighty-eight chapters. **The striking thing, looking back, is how few
distinct ideas there were.** The kernel is enormous; the principles are not. Read this list
slowly — in an interview, *recognizing which principle applies* is the skill actually being
tested.

**1. Separate mechanism from policy.** (Ch. 00, 21, 43, 66, 73, 87)

> Put the *ability* in the kernel and the *decision* somewhere replaceable.

dm targets plus `lvm2`. Scheduler classes. netfilter hooks plus `nft` rules. eBPF, which is
this principle taken to its limit. Even the maintainer tree is this applied to humans.

**2. Build a narrow waist.** (Ch. 00, 53, 63, 68, 84)

> Many producers above, many implementations below, one small interface between — so the cost
> is *m + n*, not *m x n*.

Syscalls. `file_operations`. `bio`. `scsi_cmnd`. And the failure modes: a waist too wide (every
filesystem must implement everything) or too narrow (nothing can express what it needs).
**Choosing the width of the waist is the single most consequential decision in any subsystem.**

**3. When a correctness obligation is routinely forgotten, move it into the primitive.**
(Ch. 05, 12, 28, 34, 52, 55, 82, and the whole of Part 5)

> If reviewers keep catching the same missing `put`/`free`/`unlock`, the API is wrong — not
> the programmers.

`devm_`. Refcounted lookup helpers. `iomap` replacing per-block mapping. RAII and `Drop`.

**4. Partition instead of locking.** (Ch. 15, 16, 54, 58, 63, 69)

> When a lock is the bottleneck, the answer is rarely a faster lock. It is to arrange the data
> so the lock is not needed.

Per-CPU counters. RCU. RCU-walk path resolution. XFS allocation groups. `blk-mq` per-CPU
queues. NVMe queue pairs. **Six subsystems, one move.**

**5. Batch the expensive notification; amortize it over cheap work.** (Ch. 46, 55, 69, 76, 104)

> Put the work in shared memory, ring one doorbell, and poll when the rate justifies it.

NAPI. `iomap` extents. NVMe doorbells. `io_uring` rings. virtqueues. The interrupt-versus-poll
crossover is a ski-rental problem (Ch. 14, 48) every single time.

**6. Validate, then apply.** (Ch. 43, 47, 61, 66, 80)

> Do everything that can fail *before* the point of no return, so the commit step cannot fail.

Journalling. dm's suspend/reload/resume. DRM atomic check/commit. `try_pin_init!`'s
reverse-order unwinding. **When you cannot make an operation atomic, make the *decision*
atomic and make everything before it reversible.**

**7. Optimizations are contracts with a hardware model, and models expire.**
(Ch. 51, 55, 57, 64, 70)

The elevator scheduler. `buffer_head`. ext4 block groups. Each was *correct*, became
*pointless*, and then became *harmful*. **Ask what model a piece of code assumes — not whether
it still compiles.**

**8. Interfaces are forever; implementations are not.** (Ch. 24, 30, 75, 86, 87, 88)

Hyrum's Law plus "we do not break userspace" means a two-line UAPI mistake outlives everyone
who made it. **Spend your design effort at the boundaries. The interior can always be
rewritten.**

---

**And the meta-skill this chapter is really about.** An architect is not someone who knows
more facts. It is someone who, shown an unfamiliar system, immediately asks:

```
  1. What is the narrow waist here, and is it the right width?
  2. What state is shared, and could it be partitioned instead?
  3. Which invariants are enforced, and which are merely documented?
  4. What hardware model does this assume, and does it still hold?
  5. Where is the point of no return, and is everything before it reversible?
  6. What is the interface, and what will it cost us in ten years?
```

**Those six questions are the real takeaway of this book. Everything else was worked
examples.**

---

### T.1 — The recurring principles, collected

Fifteen ideas appear over and over in this curriculum. Collecting them is the point of this
chapter, because an architect's value is largely pattern recognition across domains.

**1. Policy versus mechanism.** The kernel provides mechanism; policy belongs in userspace,
in eBPF, or in a tunable. Policy changes faster than an ABI can, so a kernel policy decision
is a permanent one. *(Ch. 00 §T.4, Ch. 24, Ch. 75)*

**2. The narrow waist.** A small, stable interface with many implementations below and many
users above. The syscall interface, VFS, `struct file_operations`, the driver model, `blk-mq`.
Narrow waists are what let the ecosystem evolve independently on both sides. *(Ch. 24, Ch. 53)*

**3. Validate, then apply — two-phase commit.** Check everything, commit nothing until all
checks pass. KMS atomic modesetting, V4L2 `TRY_FMT`, the clock rate-change protocol, filesystem
transactions. It converts "partial failure" from a possible state into an impossible one.
*(Ch. 43, Ch. 47)*

**4. Move the obligation into the primitive.** When a correctness requirement is routinely
forgotten, do not document it harder — make it automatic. `devm_`, `guard()`, folios,
`mmiowb()` folded into `spin_unlock`, RAII, `pin-init`'s generated unwinding. *(Ch. 05, 28,
34, 52, 80)*

**5. Hyrum's Law.** Every observable behaviour of an interface will be depended upon,
including the accidents. Therefore: constrain observability deliberately, reject unknown
flags, and never "fix" an observable behaviour. *(Ch. 24)*

**6. Optimize the operation that dominates.** Timeouts are usually cancelled, not fired — so
the timer wheel optimizes cancellation. Packets are usually dropped or forwarded, not
delivered — so XDP runs before `sk_buff` allocation. *(Ch. 19, Ch. 74)*

**7. Read sharing scales; write sharing does not.** The entire scalability discipline follows
from MESI. RCU, per-CPU, `qspinlock`, sharding. *(Ch. 15, 16, 105)*

**8. Batch to amortize.** When a boundary crossing dominates, cross less often. `io_uring`,
virtio, NAPI, `mmu_gather`, plugging, group commit. *(Ch. 63, 76, 104)*

**9. Stop, drain, free — in that order.** Teardown is the reverse of setup and it is where the
bugs are. `cancel_work_sync` before `kfree`; `free_irq` before freeing handler data; quiesce
DMA before releasing buffers. *(Ch. 25 P12, Ch. 83)*

**10. Ski rental.** When you do not know how long you will wait, spin for about as long as a
sleep would cost, then sleep. Adaptive mutexes, NAPI's budget, polling versus interrupts.
*(Ch. 14, Ch. 48)*

**11. Measure, then theorize.** At scale, the answer is usually two or three specific
cachelines or one specific lock — not a diffuse problem. `perf c2c` before hypothesis.
*(Ch. 06, Ch. 105)*

**12. Fail closed, at a boundary.** Validate at the system boundary — the syscall, the device
input — and trust internally. Validating everywhere is both slow and a sign the boundary is
in the wrong place. *(Ch. 24, Ch. 102)*

**13. Make the failure mode visible.** A ring that silently drops is a lie; export a drop
counter. A design that ends at "and then it works" is unfinished. *(Ch. 06, Ch. 50)*

**14. The ABI is forever; the implementation is not.** Design the interface as if you cannot
change it, because you cannot. Everything behind it, change freely. *(Ch. 24, Ch. 88)*

**15. Three users, or it is not an abstraction.** Two is coincidence. A one-user abstraction
encodes that user's assumptions and is wrong in ways only the second user reveals.
*(Ch. 26, Ch. 43)*

### T.2 — The API design checklist

For any new interface — syscall, ioctl, sysfs file, in-kernel API, or an internal library at
your company.

**Scope**
- [ ] Does this need to exist? What happens if we do nothing?
- [ ] Does an existing interface do this, or nearly?
- [ ] Is it at the right layer? Could it be composed from existing primitives?
- [ ] Are there at least three real users?

**Shape**
- [ ] Is it mechanism or policy? If policy, why is it in the kernel?
- [ ] What is the *smallest* interface that solves the problem?
- [ ] Is it extensible? (size field + `copy_struct_from_user`, reserved flags)
- [ ] Does it return an **fd** for anything with a lifetime? *(fds are capabilities:
      refcounted, pollable, passable, auto-closed)*
- [ ] Are unknown flags **rejected** with `-EINVAL`? *(if you ignore them, you can never use
      them)*
- [ ] Is there an `*at()` form taking a dirfd, if paths are involved?

**Types and layout**
- [ ] Explicit-width types only. No `long`, no `enum`, no pointers-as-`unsigned long`.
- [ ] Explicit padding. Run `pahole`. *(uninitialized padding copied to userspace is an info
      leak — Ch. 102)*
- [ ] 32/64-bit compat correct? Tested with a 32-bit userspace?
- [ ] Endianness specified for anything crossing a wire or a device.

**Semantics**
- [ ] What happens on partial success? Is that state observable and recoverable?
- [ ] What is the concurrency contract? Two callers at once?
- [ ] What are the error codes, and do they distinguish the cases a caller must distinguish?
- [ ] Is it interruptible? Restartable? What does it do on a signal?
- [ ] What is the resource-exhaustion behaviour, and who is charged?

**Consequences**
- [ ] **What am I committing to forever?** List it explicitly.
- [ ] What does this make impossible later?
- [ ] What is the observable behaviour I did *not* intend? (Hyrum's Law)
- [ ] What is the security surface? Who can call it, with what privilege?

**Delivery**
- [ ] Tests in `selftests/`.
- [ ] Documentation, and a man-page patch for UAPI.
- [ ] A real, named, in-tree or announced user.

### T.3 — Kernel or userspace: the decision procedure

```
 Does it need privileged hardware or state access?
 ├─ NO ──► Does the boundary-crossing cost dominate at the required rate?
 │          │          (a syscall is ~200-500 ns with mitigations)
 │          ├─ NO ──► USERSPACE. Done.
 │          └─ YES ─► Can it be batched or amortized?
 │                     ├─ YES ─► USERSPACE + io_uring / a shared ring
 │                     └─ NO ──► Policy or mechanism?
 │                                ├─ policy ──► eBPF / sched_ext
 │                                └─ mech ───► KERNEL
 └─ YES ─► Can the hardware be safely delegated (IOMMU)?
            ├─ YES ─► VFIO / userspace driver (SPDK, DPDK)
            └─ NO ──► KERNEL
```

**The move that marks a principal-level answer:** note that the dichotomy is 1990s framing.
The modern option is *mechanism in the kernel, policy supplied at runtime from userspace* —
which is what eBPF, `sched_ext`, `io_uring`, FUSE, and VFIO all are. The question is not
"where does the code live" but "where does the mechanism live, and who supplies the policy."

### T.4 — Evaluating an abstraction

Four questions, from `reference/system-design.md` §G4, expanded.

**1. Does it have at least three real users?** Two is coincidence. The second user reveals
which of your assumptions were the first user's, and the third reveals which were the
second's.

**2. Does it remove more code than it adds?** Net-negative diffs are the strongest signal an
abstraction is real. `regmap` is the canonical success: it eliminated hand-written MMIO/I²C
access code across hundreds of drivers, plus a whole class of bugs.

**3. Does it make incorrect use *impossible*, or merely inconvenient?** Type-level
guarantees beat documentation. `Pin`, `__must_check`, sparse's `__user`, `IoMem`'s absent
`Deref`, `LockedBy`. An abstraction that can be bypassed will be.

**4. What is the escape hatch, and what does it cost?** Every abstraction leaks. The ones
that survive have a clean way down a level: `regmap`'s raw accessors,
`dma_alloc_noncoherent`, `blk-mq`'s `none` scheduler. An abstraction with no escape hatch
gets forked.

**The negative signals**, equally important:
- It has exactly one user.
- It has a `void *private` that every user casts differently.
- Its callers all pass the same constant for half its parameters.
- Understanding it requires understanding all its implementations.
- It was designed before any implementation existed.

### T.5 — Evolving an interface you cannot change

You shipped it. Someone depends on it. Now you need to change it.

**The rules:**
1. The userspace ABI is inviolable. If a change breaks a working program, the change is
   wrong, however wrong the original was.
2. Hyrum's Law: timing, error codes, ordering, padding contents, and bugs are all part of the
   interface.
3. Therefore: **add, never change.**

**The mechanics:**

| Need | Technique |
|---|---|
| Add a field | If there is a size field → `copy_struct_from_user`. If not → a new ioctl/syscall number |
| Change a default | **Never.** Add an opt-in flag; the old default is permanent |
| Add behaviour | A new flag bit — but only if unknown flags were rejected originally |
| Deprecate | Document it, `pr_warn_once` with the task name, keep it working. Removal takes years and often never happens |
| Fix a bug userspace depends on | Keep the bug; gate the fix behind a new flag |

**The pattern that makes all of this possible**, and the single most important UAPI idiom:

```c
struct my_args {
	__u32 size;      /* set by userspace to sizeof(*args) */
	__u32 flags;
	__u64 field_v1;
	__u64 field_v2;  /* added later; old userspace sends a shorter struct */
};

err = copy_struct_from_user(&args, sizeof(args), uarg, usize);
/* -E2BIG if userspace sent a LARGER struct with a non-zero tail;
 * zero-extends if it sent a SMALLER (older) one.
 * => forward and backward compatible in one call.              */
if (err)
	return err;
if (args.flags & ~MY_VALID_FLAGS)
	return -EINVAL;        /* <- the line that lets you add flags later */
```

`clone3`, `openat2`, `sched_setattr`, and `bpf()` all do this. **Two lines on day one —
the size field and the flag rejection — eliminate the entire problem.** The architect's
lesson is not the technique; it is designing for the change you cannot foresee.

### T.6 — Technical debt: which kind, and what to do

Not all debt is the same, and treating it uniformly is a category error.

| Kind | Example | Action |
|---|---|---|
| **Deliberate and documented** | "We hardcoded this; issue #123 tracks generalizing it" | Fine. Track it |
| **Deliberate and undocumented** | A hack nobody wrote down | Document it *now*, before the author leaves |
| **Inadvertent** | We learned something and would do it differently | Refactor when you touch it |
| **Bit rot** | The world changed around it | Schedule it |
| **Load-bearing** | Ugly, works, subtle, nobody understands why | **Do not touch.** Test it, document it, leave it |

**Load-bearing debt is the one people get wrong.** Every subsystem has code that looks
terrible and is correct for reasons lost to history. `git log` on it usually reveals a trail
of fixes. Refactoring it re-introduces every one of those bugs. The correct response is to
*add tests* that encode the behaviour, and a comment explaining what you learned, and then
leave it alone.

**The repayment heuristic, in priority order:**
1. Debt that **blocks current work** — pay it now.
2. Debt that **causes recurring bugs** — pay it; measure the bug rate before and after.
3. Debt that **makes onboarding hard** — document first, refactor second.
4. Debt that is merely **ugly** — leave it. Aesthetics is not a business case.

### T.7 — Making and defending a decision

The structure that works, for a design review or a mailing list.

**1. State the problem, and agree on it.** Most disputes are actually disagreements about the
problem. Resolve that first, in the other person's words.

**2. State the constraints.** Scale, latency target, failure model, compatibility horizon,
who operates it. These are what make one answer right and another wrong.

**3. Present three options, priced.** Always three, always including "do nothing."

| Option | Buys | Costs | Right when |
|---|---|---|---|

**4. Commit, with conditions.** "I would do B. I would switch to A if [measurable thing]."
A recommendation without a commitment reads as indecision; a commitment without conditions
reads as inflexibility.

**5. Name what it costs.** Volunteer the downside before you are asked. This is the single
strongest credibility move available, and it is the difference between someone selling a
design and someone who has operated one.

**6. Say what would change your mind.** If nothing would, you are not having a technical
argument — and you should notice that about yourself.

### T.8 — Arguing well

**When you disagree with someone:**
- Agree on the problem before disputing the solution.
- Argue in their currency — latency, maintenance, risk, schedule — not yours.
- Name the cost precisely. "This is complex" is not an argument; "this adds a lock to a path
  every driver takes, to serve one device" is.
- Offer an alternative and be honest about *its* cost.
- **State the test that would settle it.** This converts an argument into an experiment.
- Concede promptly when you are wrong. It costs nothing and buys everything.

**When someone disagrees with you:**
- Assume they know something you do not. Frequently true.
- Ask what they are optimizing for. The disagreement is usually about constraints.
- Separate the stated objection from the underlying concern — which is usually maintenance
  burden, risk, or "I will have to support this."
- If they are the maintainer and you cannot convince them, it is their call. Disagree and
  commit, or fork, or drop it — but decide, do not litigate.

**The failure mode is arguing about taste.** The recovery is always to find the measurable
statement underneath. If there is not one, the disagreement is aesthetic and should be
resolved by whoever owns the code.

### T.9 — Judging when *not* to build

The highest-leverage architectural decisions are usually refusals.

**Do not build when:**
- **The problem will go away.** Hardware changes, upstream is already solving it, the
  workload is being retired.
- **The cost is recurring and the benefit is one-time.** An out-of-tree patch is a
  subscription, not a purchase (Ch. 88).
- **You cannot maintain it.** Shipping something nobody can support is a liability you have
  given to your successor.
- **It duplicates something you could improve instead.** Two half-good implementations is
  strictly worse than one good one.
- **The requirement is a solution in disguise.** "We need a new syscall" is almost never the
  requirement; ask what it is for.
- **Nobody has asked twice.** A feature requested once by one person is a hypothesis, not a
  requirement.

**How to refuse well** — the four kinds of no, from Ch. 87 §T.4: *not like this*, *not here*,
*not yet*, *not ever*. Saying which one you mean saves everyone months.

### T.10 — What an architect is actually for

Not "the person who designs things." The distinguishing functions:

| Function | What it means |
|---|---|
| **Choosing the constraint** | Identifying which of ten constraints actually binds, so effort goes where it matters |
| **Pricing decisions** | Making the cost of an option legible before it is chosen |
| **Saying no** | Preventing work, which is invisible and enormous |
| **Connecting domains** | Noticing that the storage problem and the networking problem are the same problem |
| **Setting the horizon** | Deciding on a five-year basis when everyone else is on a quarterly one |
| **Growing others** | Making decisions *explicable*, so the next person can make them |
| **Being wrong publicly** | Modelling how to update on evidence |

The last one matters more than it sounds. An organization where the architect is never wrong
is one where nobody says anything.

**The one-sentence version, and a good closing answer in an interview:**

> An architect's job is to make the expensive decisions cheaply — by finding the binding
> constraint, pricing the options honestly, committing with stated conditions, and refusing
> the work that should not exist.

---

## 1. Internals

### Case studies to read

The best way to learn judgement is to study decisions — including the wrong ones.

| Decision | Read for |
|---|---|
| **`io_uring`** | A successful new interface: shared-memory rings, extensibility, and the security cost of a large surface (Ch. 76) |
| **eBPF** | Kernel-resident, userspace-authored policy. A new answer to an old dichotomy (Ch. 75) |
| **`sched_ext`** | The same move applied to the scheduler; watch the arguments about *why* it was acceptable |
| **Folios** | Correcting a foundational type after 30 years, incrementally, without a flag day (Ch. 52) |
| **`blk-mq`** | Replacing a core subsystem for a hardware change (Ch. 63) |
| **Device tree** | Moving configuration out of code; a large, painful, correct migration (Ch. 32) |
| **`devm_`** | Absorbing a bug class into a primitive (Ch. 28) |
| **`PREEMPT_RT`** | A twenty-year out-of-tree effort that landed by upstreaming its dependencies first (Ch. 103) |
| **Rust** | An ongoing decision, with the social costs fully visible (Ch. 85) |
| **`devfs`** (removed) | An abstraction that did not survive; udev replaced it. Read why |
| **`bkl`** (removed) | A decade-long incremental removal of a global lock |
| **`CONFIG_PREEMPT_VOLUNTARY`, `sysfs` layout churn, `/proc` accretion** | How ABI accidents happen |

### Documentation

| Path | Contents |
|---|---|
| `Documentation/process/stable-api-nonsense.rst` | The in-kernel-ABI argument |
| `Documentation/admin-guide/sysfs-rules.rst` | What userspace may and may not assume |
| `Documentation/ABI/` | `stable/`, `testing/`, `obsolete/` — the ABI commitment levels, and a useful model for any project |
| `Documentation/process/adding-syscalls.rst` | **The syscall design checklist.** Read it even if you never add one |
| `Documentation/core-api/` | The in-kernel API reference; note which are documented and which are not |
| `Documentation/process/kernel-enforcement-statement.rst` | The GPL enforcement position |

`Documentation/ABI/`'s three-tier structure (`stable`, `testing`, `obsolete`) is worth
stealing for any project: it makes the commitment level of each interface explicit and
machine-readable, which is the single cheapest thing you can do to manage ABI risk.

---

## 2. Practice

### Lab 89.1 — Design review a real proposal

```bash
# Find a substantial RFC on lore -- a new subsystem, a new syscall, a
# major refactor. Read the entire thread before forming a view.
```

Produce the review, in the §T.7 structure:

1. **The problem, restated in your own words.** If you cannot, you do not understand it yet.
2. **The binding constraint.** Which of the stated requirements actually determines the
   answer?
3. **The API checklist (§T.2)**, worked through line by line.
4. **The three options**, priced — theirs, plus two alternatives.
5. **Your recommendation, with conditions.**
6. **What it costs**, volunteered.
7. **What would change your mind.**

Then compare with what the list actually said. **Where you and the list disagree is where
your model of the kernel's values is wrong**, and that gap is the most valuable thing you
will find.

### Lab 89.2 — Design a UAPI, then attack it

Design a new userspace interface for something you know well. Then hand it to a colleague
with this brief:

> Find every way this interface can be misused, every behaviour it accidentally exposes, and
> every future change it makes impossible.

Specific attacks to run:

| Attack | Question |
|---|---|
| **Hyrum** | What behaviour did I not intend to promise? Timing? Error ordering? Field ordering? |
| **Extensibility** | How do I add a field in three years? A flag? A whole mode? |
| **32/64** | Does the struct have holes? Run `pahole`. Does it have a `long`? |
| **Concurrency** | Two callers at once? A caller and a teardown at once? |
| **Partial failure** | What state is the world in if it fails halfway? Can the caller tell? |
| **Resource exhaustion** | Can an unprivileged caller allocate unboundedly? Who is charged? |
| **Information leak** | Does anything uninitialized reach userspace? |
| **Privilege** | What check gates this? Is `CAP_SYS_ADMIN` doing real work or is it a fig leaf? |
| **The second user** | Design a second, different consumer. Does the interface still fit? |

Then fix it. **Iterate three times.** The third iteration is usually where the design becomes
genuinely good, and noticing that is itself the lesson.

### Lab 89.3 — Write a decision record

For a real decision you or your team faces, produce a one-page record:

```markdown
# ADR-017: Buffered versus direct I/O for the ingest path

## Status
Accepted, 2025-01-15. Supersedes ADR-009.

## Context
The ingest path writes 400 MB/s of 64 KiB records. We currently use
buffered writes with periodic fsync. p99.9 latency is 180 ms, driven by
writeback stalls when the dirty ratio is hit. The requirement is p99.9
under 20 ms. We have 128 GiB of RAM and NVMe storage with power-loss
protection.

## Constraints
- Durability: an acknowledged write must survive power loss.
- p99.9 write latency < 20 ms.
- Must run on ext4 and XFS.
- The team has no O_DIRECT experience.

## Options

| Option | Buys | Costs | Right when |
|---|---|---|---|
| A. Tune dirty ratios | No code change; hours of work | Does not remove the stall, only moves it; fragile across kernel versions | The target is soft |
| B. O_DIRECT + own buffering | Removes page-cache involvement entirely; predictable | Alignment handling, we lose readahead, ~2 weeks plus a new bug class | Latency is the binding constraint |
| C. io_uring + buffered + explicit writeback | Batches syscalls, lets us pace writeback | Does not remove the dirty-ratio stall; adds a dependency on 5.10+ | Syscall cost dominates (it does not here) |
| D. Do nothing | Free | We miss the requirement | The requirement is negotiable |

## Decision
B. The binding constraint is tail latency, and only B removes the
mechanism that causes it. We will use io_uring to submit the direct
writes (so we get C's batching for free) and group-commit to amortize
the flush.

## Conditions
If measured p99.9 with B is not under 20 ms, the problem is not the page
cache and we should re-open this with new data.

## Costs we are accepting
- Alignment bugs are a new class for this team. Mitigation: a single
  wrapper layer, fuzzed.
- We lose readahead on the read path; we do not currently read this data.
- O_DIRECT semantics differ subtly between ext4 and XFS under
  concurrent writes. We will test both.

## What would change this
If the storage vendor's next firmware removes the flush cost, option A
becomes viable and B's complexity is no longer justified.
```

**Write five of these** for real decisions. The format forces the discipline; after five it
becomes how you think, which is the point.

### Lab 89.4 — Price an out-of-tree patch

For a real out-of-tree patch in a product you know:

```
 One-time costs:
   - initial development                    ___ days
   - review (internal)                      ___ days

 RECURRING costs, per kernel uprev:
   - rebase and conflict resolution         ___ hours
   - re-test                                ___ hours
   - re-review when the surrounding code changes  ___ hours
   × number of uprevs per year              ___
   × remaining product lifetime (years)     ___
   = TOTAL RECURRING                        ___ days

 Risk costs:
   - probability it silently breaks         ___%
   - cost if it does                        ___ days
   - key-person risk: who understands it?   ___

 Upstreaming cost (one-time):
   - cleanup, tests, documentation          ___ days
   - review cycles (expect 3-5, over weeks) ___ days
   - total                                  ___ days

 CROSSOVER: upstreaming pays for itself after ___ uprevs.
```

Run this for ten patches. **The result is almost always that upstreaming is cheaper than
anyone believed**, and having the number is what makes the argument to management.

### Lab 89.5 — Study a reversal

Find a kernel decision that was made, shipped, and later reversed or heavily revised.
Candidates: `devfs`, the original `sysfs` layout, `CONFIG_PREEMPT` variants, tasklets,
`ioctl` interfaces later replaced by netlink, the first attempt at a given API.

Write up:
1. What was the original reasoning? Steelman it — it was not obviously stupid at the time.
2. What information was missing?
3. What was the cost of the reversal — and who paid it?
4. **Was it knowable in advance?** Be honest; often it was not.
5. What would have made it cheaper to reverse? (This is usually the real lesson: not "make
   the right decision" but "make the decision reversible.")

Do this for three. **The recurring finding is that the expensive decisions were the
irreversible ones, not the wrong ones**, which is the most useful single idea in
architecture.

### Lab 89.6 — Argue a position you disagree with

Pick a genuine, live kernel controversy. Candidates: Rust's place in the tree; whether
`io_uring` should be on by default given its CVE record; whether the no-stable-in-kernel-ABI
policy is still right; whether the CVE flood helps or hurts; whether `sched_ext` was a good
idea.

1. Write the strongest possible case for the side you do *not* hold. Use real evidence.
2. Have someone who holds that view read it and tell you what you missed.
3. Now rebut your own case.
4. State, explicitly, what evidence would move you.

**This is the single most useful exercise in the chapter.** It reveals which of your beliefs
are held on evidence and which on habit, and the ratio is usually humbling.

### Lab 89.7 — The system design round, practised

Take the problems from `reference/system-design.md` §11 and work them to a whiteboard, out
loud, in 40 minutes, with a colleague playing the interviewer.

Grade yourself on the §12 rubric:
- Did you ask the three binding questions before designing?
- Did you derive a constraint arithmetically?
- Did you present three options with prices, including "do nothing"?
- Did you commit, with conditions?
- Did you name the failure modes and their detection?
- Did you say what counter you would add?
- Did you volunteer what your design costs, unprompted?
- Did you reframe a wrong premise?

**Do five.** The improvement between the first and the fifth is large and measurable.

---

## 3. Mastery drills

1. Design a complete new UAPI for something real, run it through the §T.2 checklist and the
   Lab 89.2 attacks, and post it as an RFC. Handle the review.

2. Write ten architecture decision records for real decisions in your current work. Revisit
   them in six months and score your own reasoning.

3. Take a subsystem and write its five-year direction: what to delete, what to abstract, what
   debt is load-bearing, what the binding constraint will be in 2030.

4. Price the full out-of-tree delta of a product you work on, using Lab 89.4. Present the
   number and an upstreaming budget request.

5. Study five kernel interfaces that survived twenty years (VFS, `file_operations`, the
   driver model, netlink, `epoll`) and five that did not. Extract the difference.

6. Argue both sides of three live controversies, to the standard of Lab 89.6.

7. Find a "requirement" in your organization that is actually a solution in disguise. Trace
   it back to the real requirement and propose a different solution.

8. Review five design proposals — kernel or internal — using the §T.7 structure. Track how
   often your recommendation matched the eventual outcome, and why when it did not.

9. Identify three pieces of load-bearing technical debt in code you own. For each: write the
   tests that encode its behaviour, and the comment explaining why it is the way it is. Do
   not refactor them.

10. Take one of the fifteen principles in §T.1 and find five instances of it in the kernel
    that this curriculum does not mention. Write them up.

11. Build the "reversibility" audit: for every significant decision your team made this year,
    classify it as reversible or not, and assess whether the irreversible ones got
    proportionate scrutiny.

12. Write your own architect's playbook — the one-page version of this chapter, in your own
    words, for your own domain. Then teach it to someone.

---

## 4. Further reading

**Kernel-specific**
- `Documentation/process/adding-syscalls.rst` — the best short API-design document in the
  tree
- `Documentation/process/stable-api-nonsense.rst`
- `Documentation/ABI/` — the three-tier commitment model
- `Documentation/admin-guide/sysfs-rules.rst`
- The LWN archives for any major interface: read the *rejected* proposals as carefully as the
  accepted ones
- The annual Maintainers Summit and Kernel Summit reports — how the expensive decisions are
  actually made

**On design**
- Lampson, "Hints for Computer System Design," *SOSP*, 1983 — **the single best paper on this
  topic.** "Keep it simple," "handle the normal case efficiently," "make it fast rather than
  general," "leave it to the client." Read it annually
- Saltzer, Reed, Clark, "End-to-End Arguments in System Design," *TOCS*, 1984 — where
  function belongs in a layered system
- Clements et al., "The Scalable Commutativity Rule," *SOSP*, 2013 — scalability as an
  **interface design** property, not an implementation one. Profound and underread
- Ousterhout, *A Philosophy of Software Design* — short, opinionated, and the "deep modules"
  idea maps exactly onto narrow waists
- Henderson, "Why Do Computers Stop and What Can Be Done About It?" (Gray, 1985) — on failure
  models and why "do nothing" is often right

**On judgement and organizations**
- Fogel, *Producing Open Source Software* — free; the governance material is directly
  applicable
- Nadia Eghbal, *Working in Public* — the economics of maintenance
- Hunt & Thomas, *The Pragmatic Programmer* — the "reversibility" and "tracer bullets"
  chapters in particular
- Will Larson, *Staff Engineer* and *An Elegant Puzzle* — the best available account of what
  the role is, outside the kernel context
- Camille Fournier, *The Manager's Path* — for the adjacent role you will be asked to
  interface with

**On being wrong well**
- Tetlock, *Superforecasting* — calibration, and stating what would change your mind
- Kahneman, *Thinking, Fast and Slow* — the biases that make architecture hard, particularly
  the planning fallacy and outcome bias

---

## Part 6 completion checkpoint

This closes the last gap against Love's *Linux Kernel Development* (his ch. 20). Confirm you
can:

- [ ] Explain why the kernel uses email, without being defensive about it
- [ ] Split a change into a clean, bisectable series where every commit builds
- [ ] Write a commit message that explains *why*, with a correct `Fixes:` tag
- [ ] Use `b4` to send a series, collect trailers, and post v2
- [ ] State what `Signed-off-by:` legally asserts
- [ ] Review a patch using the tiered checklist, distinguishing blocking from nits
- [ ] Say no in each of the four ways, and know which one you mean
- [ ] Explain the tree topology from mainline to a product kernel
- [ ] State the stable rules and the upstream-first principle, with the reasoning
- [ ] Explain the 2024 CVE change and why per-CVE triage is the wrong strategy
- [ ] Describe GKI and extract the transferable lessons
- [ ] Price an out-of-tree patch over a product lifetime
- [ ] Run the API design checklist and the kernel-or-userspace decision procedure
- [ ] Evaluate an abstraction on the four questions
- [ ] Write a decision record that commits, with conditions, and names its own cost

→ Next: [../part7-os/90-boot-flow.md](../part7-os/90-boot-flow.md)

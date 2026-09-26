# Chapter 85 — Case Studies: Binder, Nova, `rnull`, and the Upstream Story

> The last chapter of Part 5, and the one that turns technique into judgement. Four real
> Rust-in-Linux projects, each chosen because it answers a different question: *can Rust
> handle genuinely hard lifetime problems? can it handle a million-line-class driver? is it
> actually as fast? and will the community accept it?* Then the honest assessment you should
> be able to give when someone asks "should we use Rust?"

---

## Theory & First Principles

### T.0 — Start here: the only evidence that matters

Everything in Part 5 so far has been an argument. **Arguments about programming languages are
cheap and endless.** The only thing that settles them is shipped code: does it work, is it
maintainable, and what did it actually cost?

So here are the real data points, and the honest scorecard:

| Case | What it proves | What it cost |
|---|---|---|
| **Binder** (Android IPC) | a *core*, security-critical, performance-critical subsystem can be rewritten — performance parity, and a class of historical UAF CVEs structurally excluded | years of work; needed new abstractions upstream first |
| **Nova / NVIDIA GPU** | a *new, large* driver chose Rust over C from the start — the vendor's own judgement, not an experiment | pulled DRM abstractions into existence; still maturing |
| **Apple AGX (Asahi)** | a reverse-engineered GPU driver written by a small team, with **GPU firmware lifetime bugs** caught by the borrow checker rather than by debugging | out-of-tree for a long time; abstraction churn |
| **null_blk / misc / phy** | the small, boring conversions that prove the plumbing works | little; they are the on-ramp |

**The Asahi case is the most interesting technically**, and worth stating carefully because it
is the closest thing to a controlled experiment. The AGX driver shares complex object
lifetimes with GPU firmware: objects allocated by the driver are referenced by the firmware
for an indeterminate period, and freeing one early is a hard-to-debug memory corruption with a
latency of seconds. The developer's public account is that encoding those lifetimes in Rust's
type system turned a category of bug that would have been days of debugging into compile
errors. **That is the claim of Ch. 77 §T.0 meeting reality in exactly the situation it
predicts** — complex lifetimes, concurrent mutation, no good debugger.

**Now the scorecard the other way, because the costs are real and a mature reader should hold
both:**

```
  COST 1  Two toolchains, two idioms, two review cultures in one tree.
  COST 2  The abstractions are UNSTABLE. A driver written in 2023 may not
          compile in 2025 without change.
  COST 3  Architecture support is narrower than C's.
  COST 4  The C-side maintainer problem: C maintainers are not obliged to
          fix Rust callers, and Rust safety arguments can be silently
          invalidated by C changes (Ch. 84 §T.0).
  COST 5  The learning curve is genuinely steep, and the population of
          people who know BOTH kernel internals AND Rust is small.
```

**Cost 4 is the deep one**, and it is a *social* problem wearing a technical costume. It was
the substance of the public disputes in 2024–2025, and it has no purely technical solution:
it is a question about who owes what to whom in a shared codebase.

**The judgement worth forming — and you will be asked for it in an interview — is not "is
Rust good" but "where is the trade favourable?"**

| Favourable | Unfavourable |
|---|---|
| New, large, complex drivers with intricate lifetimes | small tweaks to mature C drivers |
| Code parsing untrusted input (filesystems, protocols, USB descriptors) | deeply arch-specific code, entry/exit paths, early boot |
| Subsystems with a history of memory-safety CVEs | code that must build on every architecture today |
| Teams that can invest in the abstraction layer | one-off, out-of-tree, short-lived drivers |

**And the closing frame for the whole of Part 5**, which is the thing to actually remember:

> **This is not a language debate. It is the recurring principle of this book at the largest
> possible scale: *when a correctness obligation is routinely forgotten, move it into the
> primitive.* `devm_` did it for resources (Ch. 28). Refcounted lookup helpers did it for
> lifetimes (Ch. 12). `iomap` did it for block mapping (Ch. 55). Rust does it for memory
> safety, across the board — and, like every one of those, it trades flexibility and
> familiarity for a guarantee.**

Whether that trade is worth it depends on the code, and an architect's job is to answer that
per-subsystem rather than universally.

```bash
git log --oneline -- drivers/android/ | head -20
find . -name '*.rs' -path './drivers/*' | head -30
git log --oneline --since=1.year -- rust/ | wc -l    # rate of change
```

---

### T.1 — Why these four

| Project | The question it answers | Verdict |
|---|---|---|
| **Binder** | Can Rust express the kernel's hardest object-lifetime problem? | Yes, and more safely |
| **Nova / nova-core** | Can Rust carry a large, complex, modern driver? | In progress; the flagship bet |
| **`rnull`** | Is the performance claim real? | Yes — within noise of C |
| **Android's rollout** | Does the security argument hold at scale? | Yes — the strongest evidence available |

Each is a real, merged or merging, production-intent effort. None is a toy, and none is a
rewrite of working C — which is itself the policy (§T.6).

### T.2 — Binder: the hardest lifetime problem in the kernel

**What it is.** Android's IPC mechanism. Every application, every system service, every
`Intent`, every `Binder` transaction goes through it. It is arguably the highest-value attack
surface on a billion-device platform: reachable from every unprivileged app, and historically
a rich CVE source.

**Why it is hard.** Binder maintains a *distributed reference graph across processes*:

```
 Process A                    binder driver                   Process B
 ┌──────────┐                ┌──────────────┐               ┌──────────┐
 │ handle 3 ├───────────────►│ binder_ref   │               │          │
 └──────────┘                │   ↓          │               │          │
                             │ binder_node  ├──────────────►│ object   │
 Process C                   │   ↑          │               │          │
 ┌──────────┐                │ binder_ref   │               └──────────┘
 │ handle 7 ├───────────────►│              │
 └──────────┘                └──────────────┘
```

- A node lives in one process; references to it live in others.
- References are created and destroyed by *messages*, asynchronously.
- A process can die at any moment, and every reference it held must unwind.
- Transactions carry file descriptors, memory, and further references.
- There are strong and weak references, with promotion and demotion.
- The whole thing must be safe against a hostile unprivileged caller.

**The C implementation** is ~6,000 lines with a locking scheme documented in a long comment
block, and its CVE history is exactly the shape you would predict: UAF on process death,
refcount errors under concurrent transactions, and races between transaction delivery and
teardown.

**What Rust changed.** Rust Binder was written by Alice Ryhl (Google) and merged for
Android; it is roughly comparable in size to the C version.

| C mechanism | Rust mechanism |
|---|---|
| `binder_node.refs` with manual inc/dec | `Arc<Node>` |
| Strong/weak distinction by hand | `Arc` / `Weak`-equivalent, typed |
| "Caller must hold `proc->outer_lock`" (comment) | `LockedBy<T, ProcInner>` |
| Death notification unwinding | `Drop` on the reference object |
| fd transfer ownership | `ForeignOwnable` + typed fd installation |
| Lock ordering documented in a comment block | still a comment + lockdep |

The last row matters: **lock ordering was not solved.** Binder's ordering discipline remains a
comment, and lockdep remains the enforcement. That is the honest boundary from Ch. 81 §T.10,
in a production system.

**The reported outcome:** performance parity (within measurement noise on Android's
benchmarks), and the elimination of the memory-safety CVE class. Ryhl's talks report that
several historical C CVEs simply do not compile in the Rust version.

**The lesson to extract:** the problems Rust helps with most are exactly the ones C is worst
at — complex, distributed, concurrent object lifetimes. It helps least with algorithmic
complexity and lock ordering. Binder is the best available demonstration of both halves.

### T.3 — Nova: the large-driver bet

**What it is.** A Rust driver for NVIDIA GPUs using the GSP (GPU System Processor) firmware
interface — the same interface NVIDIA's open kernel modules use. Split into `nova-core` (the
low-level device layer) and `nova-drm` (the DRM driver).

**Why it matters more than Binder for the long term.** Binder proved Rust can do hard
lifetimes. Nova is testing whether Rust can carry a *large* driver in a *complex subsystem*:

- DRM/KMS is one of the most intricate subsystems in the kernel (Ch. 47): atomic modesetting,
  the two-phase validate-then-commit discipline, `dma-buf` sharing, fences, scheduler
  integration.
- GPU drivers are the largest drivers in the tree by an order of magnitude.
- It requires abstractions for DRM, `dma-buf`, the GPU scheduler, firmware loading,
  `devcoredump`, and more — **most of which did not exist and had to be written.**

**The strategic point**, made explicitly by the DRM maintainers: Nova is not primarily about
NVIDIA. It is a forcing function to build the DRM Rust abstractions, so that *subsequent*
GPU drivers can be written in Rust. The Asahi (Apple GPU) driver, written in Rust
out-of-tree, is the existence proof that motivated it.

**What to watch, and what it will tell you:**
- Whether the DRM abstractions can be made both safe and usable at that complexity.
- Whether a C maintainer can review a Rust driver in their subsystem — the social question.
- Whether the abstraction-writing cost amortizes across the second and third driver.

As of writing, `nova-core` is merged and `nova-drm` is in progress. **Its status is the single
best indicator of kernel Rust's trajectory**, and following it is worth more than reading
another ten opinion pieces.

### T.4 — `rnull`: the performance answer

**What it is.** A Rust reimplementation of `null_blk`, the in-tree null block device used for
benchmarking the block layer (Ch. 63). It does no I/O — it completes requests immediately —
so it measures *pure block-layer and driver overhead*, which makes it the ideal instrument for
this question.

**Why it is the right experiment.** Any performance comparison between languages is
confounded by what the code does. `null_blk` does nothing, so the comparison is exactly the
per-request cost of the driver framework: the `blk_mq_ops` dispatch, the tag handling, the
completion path.

**The result**: within noise. Reported figures put Rust `rnull` at parity with C `null_blk`
on IOPS at multiple queue depths, with `.text` size in the same range. Where differences
appear they are attributable to specific inlining decisions, not to the language.

**Why this was expected, and how to explain it:**
- Rust compiles through LLVM, the same backend clang uses.
- All the abstractions in Part 5 are zero-cost: generics are monomorphized, `Guard` is a
  ZST plus a `Drop`, `IoMem` bounds are const-evaluated, `Option<&T>` is a pointer.
- The remaining costs are the same ones C pays: the atomic in `Arc::clone` is the atomic in
  `kref_get`; the bounds check you cannot elide is the bounds check C omits *incorrectly*.

**The honest caveat:** Rust makes some costs *visible* that C hides. A bounds check that Rust
inserts is one C simply did not do — and sometimes C was right (the index was provably in
range) and sometimes C was wrong (that is the OOB CVE). When you cannot prove it, use
`get()` and handle `None`; when you can, the check optimizes away.

So the accurate claim is: **"the same program is the same speed; Rust sometimes makes you
write a slightly different program, and usually that program is more correct."**

### T.5 — Android: the security evidence

The strongest empirical case for Rust anywhere, and the one to cite.

Google's Android Security team published the data:

| Finding | Figure |
|---|---|
| Memory-safety share of Android vulnerabilities, historical | ~65–76% |
| Same share by 2024, after Rust adoption in new code | **~24%** |
| Memory-safety CVEs in Android's Rust code, to date | **0** |
| New native code in Android 13 written in Rust | ~21% |

The argument they make from it is more interesting than the numbers, and it is the one to
reproduce:

> **Vulnerability density decays with code age.** New code has the highest bug density; code
> that has been in production for years has been fuzzed and audited and is comparatively
> safe. Therefore **rewriting old C in Rust has poor ROI**, while **writing new code in Rust
> has excellent ROI** — because that is where the bugs would have been.

This is why the kernel's policy (§T.6) is "new drivers, not rewrites," and being able to
justify that policy from the data rather than from taste is a strong answer.

The secondary finding is equally useful: Google reported *no measurable rollback-rate or
performance regression* from Rust components, and a **reduced** revert rate compared to
equivalent C changes — attributed to compile-time errors replacing runtime failures.

### T.6 — The upstream story, honestly

**Policy, as stated by Torvalds and the maintainers:**
- Rust is for **new** code, primarily drivers. The 30M lines of C stay.
- Rust must not break C. If a Rust abstraction constrains C's evolution, the C wins.
- A subsystem maintainer is not obliged to accept Rust, but also may not block it in an area
  they do not maintain.
- The `kernel` crate abstractions are maintained by the people who write them, with the
  relevant C maintainer having a say on the *contract*.

**The friction points, which you should be able to describe fairly:**

1. **Maintainer burden.** A C maintainer now has code in their subsystem they cannot review.
   This produced genuine, public conflict — most visibly around the DMA API abstractions in
   2025, where a maintainer objected to Rust code in their subsystem on the grounds that it
   constrained their ability to change the C API. Torvalds' resolution was roughly: Rust code
   must adapt to C changes, not the reverse, and a maintainer cannot veto Rust in areas they
   do not own. **This is the real ongoing tension and it is social, not technical.**

2. **Toolchain.** No stable Rust ABI, no stable `rustc` version until recently, `bindgen`
   fragility, and the requirement to build the whole tree with one toolchain.

3. **Abstraction lead time.** You cannot write a Rust driver until someone has written the
   subsystem abstraction, and that requires extracting an undocumented C contract (Ch. 84
   §T.10) and getting a maintainer to agree to it. This is the actual bottleneck.

4. **Two languages.** Every developer must now read both. Every refactor of a C API must
   consider Rust callers.

**The counter-argument, which is also fair:** all of these are one-time or amortizing costs,
and the memory-safety benefit is permanent and compounding.

### T.7 — The decision framework

When someone asks "should we use Rust for this?", these are the questions, in order.

**Say yes when:**
- It is **new** code, especially a new driver (the Android ROI argument).
- The **abstractions already exist** for your subsystem.
- The code handles **untrusted input** — anything reachable from an unprivileged user or the
  network.
- The problem is **lifetime- or concurrency-heavy** — that is where Rust's advantage is
  largest.
- You have, or can build, **Rust competence on the team**, including at review time.

**Say no when:**
- You are **modifying existing C**. The interop cost exceeds the benefit.
- The **abstractions do not exist** and you cannot fund writing them (budget 2–5× the driver
  itself, plus review cycles).
- The subsystem's **maintainer is hostile**. You can be technically right and still not get
  merged.
- Your **team cannot review it**. Unreviewed Rust is worse than reviewed C.
- You need a **certified toolchain** (automotive, avionics) — Ferrocene exists but changes the
  calculus.
- The driver is **small and simple**. A 300-line I²C driver has little to gain.

**The honest summary, and a good closing line for an interview:**

> Rust eliminates about two thirds of the CVE classes at zero runtime cost, and it does so by
> moving invariants from comments into types. It does not fix deadlocks, hardware
> programming, DMA ownership, or lock ordering, and it adds a real one-time cost in
> abstraction-writing and review capacity. For new drivers in subsystems that are already
> abstracted, that trade is clearly positive. Everywhere else, it is a project-management
> question rather than a technical one.

---

## 1. Internals

### Where the code is

| Project | Location |
|---|---|
| **Binder (Rust)** | `drivers/android/binder/` — in Android's tree, upstreaming in progress |
| **Nova core** | `drivers/gpu/nova-core/` |
| **Nova DRM** | `drivers/gpu/drm/nova/` |
| **`rnull`** | `drivers/block/rnull.rs` |
| **PHY drivers** | `drivers/net/phy/ax88796b_rust.rs`, `qt2025.rs` |
| **Misc sample** | `samples/rust/rust_misc_device.rs` |
| **Asahi GPU (out of tree)** | `asahilinux/linux`, `drivers/gpu/drm/asahi/` |
| **Android's Rust report data** | `security.googleblog.com` |

### `rnull` structure, as a reading guide

```rust
// drivers/block/rnull.rs, in outline

struct NullBlkDevice;

#[vtable]
impl Operations for NullBlkDevice {
    type QueueData = ();

    /// Called per request. This is the hot path being benchmarked.
    fn queue_rq(rq: ARef<mq::Request<Self>>, _is_last: bool) -> Result {
        mq::Request::end_ok(rq)      // complete immediately
            .map_err(|_e| kernel::error::code::EIO)?;
        Ok(())
    }

    fn commit_rqs(_qd: ()) {}
}

// Registration:
let tagset = Arc::pin_init(mq::TagSet::new(nr_queues, ...), GFP_KERNEL)?;
let disk = gen_disk::GenDiskBuilder::new()
    .capacity_sectors(capacity)
    .logical_block_size(block_size)?
    .rotational(false)
    .build(fmt!("rnullb{}", 0), tagset)?;
```

Read `queue_rq` against `null_blk`'s C equivalent. **The functions are the same length and do
the same thing**, which is the performance result made visible in source form. The
`ARef<Request>` is the interesting difference: the request's refcount is in the type, where C
has `blk_mq_start_request`/`blk_mq_end_request` pairing by convention.

### Binder's reference model, as a reading guide

The core insight to look for when reading Rust Binder:

```rust
// C: binder_node has a manual ref count with strong/weak, and the
//    invariant "you must hold proc->inner_lock to touch these" is a
//    comment.
//
// Rust: the relationship is in the types.

struct Node {
    // Fields protected by the owning process's lock, stated in the type:
    inner: LockedBy<NodeInner, ProcessInner>,
    // ...
}

struct NodeRef {
    node: DArc<Node>,        // a strong reference; Drop decrements
    strong_count: u32,
    weak_count: u32,
}

// Process death: dropping the Process drops its refs, which drops the
// DArc<Node>s, which decrements. The distributed unwind is `Drop`.
```

Compare with `drivers/android/binder.c`'s `binder_dec_node_nilocked` and the surrounding
comment block. **The C version's correctness argument is three paragraphs of prose; the Rust
version's is a type signature.**

---

## 2. Practice

### Lab 85.1 — Benchmark `rnull` against `null_blk`

The performance claim, measured by you.

```bash
cd $KDIR

# 1. Build both.
./scripts/config --module BLK_DEV_NULL_BLK --module RUST_NULL_BLK_DRIVER
make LLVM=1 -j$(nproc) modules

# 2. Load each and benchmark identically.
run_bench() {
	local dev=$1 name=$2
	echo "=== $name ==="
	for qd in 1 8 32 128; do
		fio --name=test --filename=$dev --ioengine=io_uring \
		    --rw=randread --bs=4k --iodepth=$qd --numjobs=$(nproc) \
		    --direct=1 --runtime=30 --time_based --group_reporting \
		    --output-format=json 2>/dev/null \
		| jq -r --arg qd "$qd" \
		    '"qd=\($qd) iops=\(.jobs[0].read.iops|floor) lat_us=\(.jobs[0].read.clat_ns.mean/1000|floor)"'
	done
}

modprobe null_blk nr_devices=1 queue_mode=2 irqmode=0 completion_nsec=0
run_bench /dev/nullb0 "C null_blk"
rmmod null_blk

insmod drivers/block/rnull.ko
run_bench /dev/rnullb0 "Rust rnull"

# 3. Compare code size.
size drivers/block/null_blk/null_blk_main.o drivers/block/rnull.o

# 4. Profile both and diff the hot paths.
sudo perf record -g -- fio --name=t --filename=/dev/rnullb0 \
	--ioengine=io_uring --rw=randread --bs=4k --iodepth=32 \
	--direct=1 --runtime=20 --time_based
sudo perf report --stdio --sort=symbol | head -30
```

**Report the numbers yourself.** Then explain any difference you find in terms of specific
code, not in terms of "Rust is slower/faster." If you find a real difference, `perf annotate`
the hot function in both and look at the generated code — that is the answer.

### Lab 85.2 — Read Rust Binder for its lifetime model

```bash
# Get the Android common kernel or the upstream series.
git clone https://android.googlesource.com/kernel/common -b android-mainline
cd common && ls drivers/android/binder/

# Read in this order:
#   rust_binder.rs   -- entry points, the misc device
#   process.rs       -- Process, ProcessInner, the lock hierarchy
#   node.rs          -- Node, NodeRef: the reference graph
#   thread.rs        -- per-thread transaction state
#   transaction.rs   -- the transaction lifecycle
#   allocation.rs    -- the shared memory allocator
```

For each, answer:

1. **What owns what?** Draw the ownership graph. Where are the `Arc`s, and what breaks a
   cycle?
2. **What does `LockedBy` protect, and by whose lock?** Compare with the C version's comments.
3. **What happens on process death?** Trace the `Drop` chain. Then find the C function that
   does the same thing by hand and compare the line counts.
4. **Where is `unsafe`?** Count it. For each block, verify the `// SAFETY:` comment.
5. **What is *not* type-enforced?** Find the lock-ordering comment and confirm that Rust did
   not solve it.

Then find a historical Binder CVE (e.g. CVE-2019-2215, the UAF in
`binder_thread_release`) and determine whether the Rust version could express it.

### Lab 85.3 — Follow Nova

```bash
cd $KDIR
ls drivers/gpu/nova-core/
$EDITOR drivers/gpu/nova-core/driver.rs
$EDITOR drivers/gpu/nova-core/gpu.rs

# What abstractions did Nova require that did not exist?
git log --oneline --no-merges -- rust/kernel/ | grep -i -E 'drm|dma|devres|firmware|auxiliary' | head -30

# Track the series:
b4 am -o /tmp $(b4 search 'nova-core' | head -1)
```

Produce a short report:
- Which new `kernel` crate abstractions landed *because of* Nova?
- What is the ratio of abstraction code to driver code so far?
- What remains unabstracted, and what is the driver doing with raw `bindings` in the interim?
- Read one of the review threads and summarize the maintainer's main concerns.

**This report is directly the kind of technical due diligence a staff/principal engineer is
asked for**, and it is a good artifact to have.

### Lab 85.4 — Reproduce the Android argument on your own code

```bash
# 1. Take a codebase you have (or a kernel subsystem) and categorize its
#    bug history.
cd $KDIR
git log --oneline --since='3 years ago' -- drivers/<subsystem>/ \
	| grep -icE 'use-after-free|uaf|double free|out of bounds|oob|overflow|null deref|race|leak' 

git log --oneline --since='3 years ago' -- drivers/<subsystem>/ | wc -l

# 2. Compute the memory-safety fraction.

# 3. Now weight by code age: which files do the fixes land in?
git log --since='3 years ago' --name-only --pretty=format: -- drivers/<subsystem>/ \
	| sort | uniq -c | sort -rn | head -20
#    Cross-reference with file creation dates:
for f in <the top files>; do
	echo "$(git log --diff-filter=A --format=%ad --date=short -- $f | tail -1) $f"
done
```

The expected finding, replicating Google's: **fixes cluster in recently-added code.** Use it
to make the "new code in Rust" argument for your own project, with your own numbers. Numbers
from your own repository are far more persuasive than Google's.

### Lab 85.5 — Write and defend a recommendation

The capstone of Part 5. Produce a two-page memo answering: *"Should we write our next driver
in Rust?"*

**Structure:**

1. **The proposal and its scope.** What driver, what subsystem, what size.
2. **Abstraction inventory.** Go through the subsystem's needs and check `rust/kernel/`
   for each. Produce the table: available / partial / missing. **This is the decisive
   section** — if half the table is "missing," the answer is probably no.
3. **Cost.** Abstraction-writing effort, review latency, team training, toolchain and CI
   work. Be specific and be honest; a memo that only lists benefits is not credible.
4. **Benefit.** Apply the Android ROI argument to your own bug history (Lab 85.4). Quantify
   what a memory-safety CVE costs you.
5. **Risk.** Maintainer reception, abstraction churn, key-person dependency, what happens if
   the abstraction you need never lands.
6. **The recommendation, with conditions.** "Yes, if X. No, if Y." Per
   `reference/system-design.md` §Move 5.
7. **What would change the answer.** Name the observable that would flip it.

Then have someone argue the other side, hard. **Both positions are defensible for most real
projects, and being able to argue either is the actual skill.**

### Lab 85.6 — Read a contentious thread

```bash
# The DMA API discussion of 2025 is the most instructive single thread
# in kernel Rust's history. Find it:
b4 search 'rust dma api'
# Or browse lore.kernel.org for the thread and Torvalds' response.
```

Read it fully and write a one-page summary covering:
- What the technical objection was.
- What the *underlying* concern was (usually not the stated one).
- How the policy question was resolved.
- What precedent it set.

Then write the email you would have sent, from the maintainer's position and from the
contributor's position. **Ch. 87 and Ch. 89 develop this skill; this is the first exercise in
it, and it is the part of kernel work that seniority is actually measured by.**

---

## 3. Mastery drills

1. Reproduce the `rnull` versus `null_blk` benchmark on three machines with different core
   counts. Report the numbers, explain every difference, and state your confidence.

2. Read Rust Binder's `node.rs` and `process.rs` completely. Draw the ownership graph. Then
   read the C `binder.c` equivalents and draw the same graph from the comments. Present both.

3. Take three historical Binder CVEs and determine, for each, whether the Rust version could
   express the bug. Show the code for the ones that could.

4. Track Nova for a month. Summarize every `kernel` crate change it caused and assess whether
   each is generally useful or Nova-specific.

5. Implement a small driver twice — once in C, once in Rust — for the same QEMU device.
   Measure: lines, development time, bugs found during development, bugs found by KASAN,
   binary size, and throughput. Publish the comparison.

6. Write the abstraction-inventory tool: given a subsystem, list which of its C APIs have
   Rust abstractions and which do not. Run it on five subsystems and rank them by readiness.

7. Compute the memory-safety fraction of the last three years of CVEs for a kernel subsystem
   of your choice. Compare with the Android figures and explain any divergence.

8. Take a merged Rust driver and review it as if it were new. Write the review. Then compare
   your review with the actual list review and note what you missed.

9. Study the Asahi GPU driver (out-of-tree Rust). Explain why it was written in Rust, what
   abstractions it invented, and why it has not upstreamed. Assess whether Nova will
   supersede its abstractions.

10. Build the case *against* Rust in the kernel as strongly as you can, using real evidence.
    Then rebut your own case. The exercise is to discover which of your beliefs are held on
    evidence and which on preference.

11. Interview (or read talks by) someone who has upstreamed Rust kernel code. Write up what
    they said was hardest and compare with what this curriculum predicted.

12. Write the six-month plan for introducing Rust into a hypothetical embedded Linux product
    team: training, first project selection, CI, review process, risk mitigation, and the
    checkpoint at which you would abandon it.

---

## 4. Further reading

**Primary sources — read these, not summaries**
- Alice Ryhl's talks and writing on Rust Binder (Kangrejos, LPC, Linux Foundation) — the
  best account of solving a hard lifetime problem in kernel Rust
- The Nova driver series on LKML/dri-devel, and Danilo Krummrich's and Dave Airlie's posts on
  the strategy
- `drivers/block/rnull.rs` and Andreas Hindborg's block-layer abstraction series
- Google Security Blog: "Eliminating Memory Safety Vulnerabilities at the Source" (2024) and
  "Memory Safe Languages in Android 13" (2022) — **the data everyone cites**
- The Rust for Linux project site and its status page

**LWN — the best continuous coverage**
- The "Rust for Linux" tag archive in full
- Kangrejos conference reports, annually — the technical design discussions
- "Coming to terms with Rust in the kernel" and the 2025 maintainer-friction coverage
- The Nova and DRM abstraction coverage
- "A pair of Rust kernel modules," "Rust in the 6.x kernels" series

**Talks worth watching**
- Miguel Ojeda's Rust-for-Linux status talks (annually, LPC)
- Alice Ryhl on Binder, and on `pin-init` and async
- Wedson Almeida Filho's early design talks (the original abstraction design)
- Asahi Lina's talks on the Apple GPU driver — the most dramatic real-world demonstration of
  Rust's advantages in a complex driver
- Any of the Kangrejos recordings

**Code, in order of reading value**
1. `drivers/block/rnull.rs` — small, complete, and benchmarkable
2. `drivers/net/phy/ax88796b_rust.rs` — a real merged driver, tiny
3. `samples/rust/rust_misc_device.rs` — the teaching example
4. `drivers/gpu/nova-core/` — the frontier
5. Rust Binder in the Android common kernel — the hardest, and the most rewarding

**Cross-references**
- Ch. 77 — the argument, and the status
- Ch. 84 — writing the abstractions Nova needed
- Ch. 89 — the architect's playbook, which formalizes the judgement in §T.7
- Ch. 102 — the security argument, in its full context
- `reference/system-design.md` §G1, §G4 — kernel-vs-userspace and abstraction evaluation

---

## Part 5 completion checkpoint

Before moving to Part 6, confirm you can:

- [ ] State the memory-safety argument with numbers, and name the one intervention that is
      zero-cost
- [ ] Explain what `unsafe` does and does not disable
- [ ] Write a `# Safety` contract and a `// SAFETY:` comment that a reviewer can check
- [ ] Explain `Send`/`Sync` and the relationship `T: Sync ⟺ &T: Send`
- [ ] Explain why `Pin` exists, using `list_head` as the example
- [ ] Explain in-place initialization and why it is a *correctness* requirement in the kernel
- [ ] Write a misc device and a platform driver in Rust, from memory
- [ ] State exactly which driver bug classes Rust eliminates and which it does not (Ch. 83 §T.9)
- [ ] Describe the three-layer architecture and why layer 2 must be correct
- [ ] Name the leak footgun and why `Drop`-dependent soundness is unsound
- [ ] Give the decision framework of §T.7, with the conditions on both sides
- [ ] Name the real bottleneck in kernel Rust adoption (abstraction lead time and maintainer
      capacity, not the language)

→ Next: [../part6-engineering/86-patch-workflow.md](../part6-engineering/86-patch-workflow.md)

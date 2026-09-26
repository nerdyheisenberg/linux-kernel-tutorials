# The Interview Playbook — senior and architect-level Linux kernel / OS roles

> **What this is.** The curriculum teaches you the material. This document teaches you to
> *perform* it: under time pressure, out loud, to a skeptical stranger who is deciding
> whether you are worth $400k. That is a genuinely different skill and it is trainable.
>
> **Who this is for.** Candidates targeting Senior (L5/E5), Staff (L6/E6), or Principal
> (L7/E7) kernel, OS, embedded-systems, or systems-architect roles. Also useful if you
> are the one conducting the interview and want a rubric.

---

## 1. What these companies are actually hiring for

There are five distinct hiring profiles, and they interview differently. Identify which one
you are in **before** you prepare, because optimizing for the wrong one is the most common
avoidable failure.

| Profile | Typical employers | What they test hardest |
|---|---|---|
| **Kernel subsystem engineer** | Red Hat, SUSE, Oracle, Meta, Google, Intel, AMD, Arm | Depth in one subsystem, upstream track record, concurrency, code reading |
| **Device driver / BSP engineer** | NXP, TI, Qualcomm, Nvidia, Bosch, Continental, Siemens, medical/industrial | Hardware interfaces, device tree, bring-up, debugging with a scope, power |
| **Systems / infrastructure engineer** | Cloud providers, HFT, storage vendors, Cloudflare, Netflix | Performance methodology, tracing, production debugging, capacity |
| **Embedded / RT architect** | Automotive, aerospace, robotics, industrial control | Determinism, safety certification, boot time, Yocto, partitioning |
| **Security engineer** | Everyone, plus dedicated firms | Attack surface, mitigations, sandboxing, exploit primitives |

**Signal:** the job description's *third* paragraph usually reveals the profile. The first
two are boilerplate. If it mentions "customer escalations," you are being hired to debug.
If it mentions "upstream," you are being hired to argue on mailing lists. If it mentions
"ASIL" or "IEC 61508," half your interview will be about determinism and process.

---

## 2. The loop, round by round

A typical senior kernel loop is 5–7 rounds. Here is what each is *really* measuring.

### Round A — Recruiter / hiring-manager screen (30 min)

Measuring: can you describe your own work at the right altitude, and are you level-matched?

The trap: describing implementation when they asked for impact, or vice versa. Calibrate:

- **Senior:** "I owned the NVMe driver's error-recovery path. I redesigned the reset state
  machine after we found a race that corrupted queues on surprise removal."
- **Staff:** "I owned storage reliability across three product lines. The reset race was one
  of five issues I found by building a fault-injection harness that we then made mandatory
  in CI, which cut field escalations by about 60%."
- **Principal:** "I made the call that we would stop carrying an out-of-tree fork and
  upstream instead. It cost two engineer-years and eighteen months, and it removed a
  recurring rebase cost that was consuming a third of the team."

Same work. Three altitudes. Match the level you are interviewing for.

### Round B — Coding (45–60 min)

Kernel interviews rarely do LeetCode. When they do, it is a filter, not a signal. What you
actually get:

| Form | Example |
|---|---|
| **Implement a kernel data structure** | Write a lock-free SPSC ring buffer. Now make it MPSC. |
| **Fix concurrency in given code** | "Here is a driver. Find the races." (Usually 3–5 planted.) |
| **Implement a classic with kernel constraints** | LRU cache, but you cannot allocate in the fast path |
| **Systems C** | Parse an ELF header; implement `memcpy` for MMIO; write a bitmap allocator |
| **Read and explain** | "Here is `__d_lookup_rcu`. Walk me through it." |

**How to perform:** narrate the four axes from Ch. 00 §T.9 *before* you write a line.
"Who calls this? What context? What lock? What can sleep?" Interviewers score this heavily
because it is exactly what they need you to do on their codebase on day one.

### Round C — Deep subsystem dive (60 min)

You pick a subsystem (or they pick from your résumé) and go to the bottom of it.

**This is the round that decides senior vs staff.** The difference is not knowing more
facts; it is being able to say *why the design is the way it is, and what the alternatives
were.* See §4.

### Round D — Debugging (60 min)

They present a symptom and you drive. Sometimes live on a machine, sometimes on paper.
See [debugging-scenarios.md](debugging-scenarios.md) for worked examples.

**What they score:** hypothesis quality, whether you gather evidence before theorizing,
whether you know which tool answers which question, and whether you say "I don't know, here
is how I would find out."

### Round E — System design / architecture (60 min)

Staff and above always. Senior sometimes. See [system-design.md](system-design.md).

### Round F — Behavioral / "architect track" (45 min)

Measuring: judgement, influence without authority, and whether you have ever been wrong in
an instructive way. §6.

### Round G — Hiring committee / cross-functional

Usually a summary round. Occasionally a "sell me on why we should build X" pitch.

---

## 3. The rubric interviewers actually use

Most large companies score on a 4-point scale across dimensions. Here is the shape of a
typical kernel rubric, translated from the internal language:

| Dimension | 1 — No hire | 2 — Junior | 3 — **Senior** | 4 — **Staff+** |
|---|---|---|---|---|
| **Depth** | Recites API names | Knows one subsystem's API | Knows one subsystem's *internals and history* | Knows the trade-offs across several, and where the bodies are buried |
| **Concurrency** | "Add a lock" | Identifies the race | Identifies the race *and* the cheapest correct fix | Redesigns so the race is unrepresentable |
| **Debugging** | Guesses | Uses printk | Picks the right tool first; forms falsifiable hypotheses | Builds the instrumentation that makes the class of bug impossible |
| **Design** | Describes an implementation | Compares two options | Articulates the forces, picks, and names the cost | Identifies the constraint everyone else missed; changes the problem |
| **Communication** | Jargon without structure | Correct but unstructured | Structured, appropriate altitude, admits uncertainty | Makes the interviewer smarter about their own system |
| **Judgement** | — | — | Knows when *not* to touch the kernel | Has killed a project for the right reason |

**The single most common reason a strong engineer gets a "senior" instead of a "staff"
score:** they answer the question asked instead of the question that matters. Staff-level
signal is saying *"before I answer — is the actual constraint latency or throughput? Because
those give opposite designs."*

---

## 4. The one technique that raises your score the most

### Answer in three layers, always

For any technical question, structure the answer:

1. **The mechanism** — what the code does. (Everyone gets here.)
2. **The forces** — why it must be that way; what constraint forced it. (Senior.)
3. **The alternative and its cost** — what else was tried, what it cost, why it lost. (Staff.)

**Worked example. Q: "Why does Linux use RCU?"**

> **Layer 1 (mechanism).** RCU lets readers access shared data with no locks and no atomics —
> `rcu_read_lock()` is a no-op or a preempt-count increment. Writers make a copy, modify it,
> swap the pointer with a release store, then wait for a grace period before freeing the old
> version.
>
> **Layer 2 (forces).** The force is read-side scalability. A rwlock still writes a shared
> cache line on every reader, so N readers on N cores cost O(N²) coherence traffic — the
> reader *is* a writer, at the cache-coherence level. RCU's whole point is that the read side
> touches nothing shared, so it scales linearly. The price is deferred reclamation, which
> means unbounded memory growth if grace periods stall, and that readers may see stale data.
>
> **Layer 3 (alternatives and cost).** The alternatives were seqlocks — cheaper still but
> readers must be pure and retryable, so you cannot dereference a pointer you read — and
> hazard pointers, which bound memory precisely but put an expensive store-load fence in the
> read path, which is exactly the cost RCU was invented to remove. Linux made the trade
> deliberately: unbounded memory in the rare stall case, in exchange for a genuinely free
> read side. Which is also why `rcu_read_lock()` sections must be short and why
> `SLAB_TYPESAFE_BY_RCU` exists for the cases where you cannot afford the grace period.

That is a 90-second answer that scores 4. The same content delivered as layer 1 only scores
2–3.

### The corollary: name the failure mode

Senior candidates describe what works. Staff candidates describe *how it breaks*. Attach a
failure mode to every mechanism you explain:

- RCU → grace-period stalls, unbounded callback backlog
- `devm_` → freed while an fd is still open
- Deferred probe → never converges, device silently missing
- Page cache → double-caching, scan pollution
- NAPI → budget exhaustion, `ksoftirqd` starvation

---

## 5. Answering when you do not know

You will be asked something you do not know. This is deliberate — they are probing the
boundary. How you handle it is worth more than the answer.

**The bad version:** bluffing, or a flat "I don't know."

**The good version, in four moves:**

1. **Bound it honestly.** "I haven't worked on the DRM scheduler directly."
2. **Reason from adjacent knowledge.** "But it is a scheduler over work that has
   dependencies expressed as fences, so I would expect it to look like a dependency-ordered
   run queue per context, with a timeout that has to force-signal fences on a hang —
   because a fence that never signals deadlocks every consumer."
3. **State what you would check.** "I'd read `drivers/gpu/drm/scheduler/sched_main.c`,
   specifically `timedout_job`, to see how reset interacts with fence signalling."
4. **Invite correction.** "Is that roughly right, or does it work differently?"

That answer demonstrates transferable reasoning, which is the thing actually being hired.
Several interviewers will tell you afterwards that it was the strongest signal in the loop.

---

## 6. The architect-track behavioral round

These questions look soft. They are not. Each one probes a specific failure mode.

| Question | What it actually probes | What a strong answer contains |
|---|---|---|
| "Tell me about a design decision you got wrong." | Do you have calibrated self-assessment, or do you narrate only wins? | A *specific* decision, the reasoning that was defensible at the time, the signal you missed, and what you changed in your process |
| "Tell me about disagreeing with a maintainer / senior person." | Can you be overruled gracefully? Can you influence without authority? | Evidence you argued with data, and either changed your mind or changed theirs — and that the relationship survived |
| "When did you decide *not* to build something?" | Judgement. Junior engineers build; senior engineers decline. | A case where the cheapest fix was organizational, not technical |
| "How do you decide kernel vs userspace?" | Do you have a framework or a reflex? | The four criteria of §7, applied to a real case |
| "Walk me through an incident you led." | Can you operate under pressure and produce durable fixes? | Timeline, hypothesis sequence, the fix, **and the systemic change** so the class cannot recur |
| "How do you onboard onto an unfamiliar subsystem?" | The Ch. 07 skill, which is the actual day-one job | Central data structure → three paths (init, fast, teardown) → locking model → git archaeology for *why* |
| "What is a technical opinion you hold that most people disagree with?" | Do you have judgement of your own, or are you a consensus machine? | Something specific and defensible, argued, not edgy |

**Prepare four stories** in advance, each usable for several questions:

1. A deep debugging win — ideally one where the first three hypotheses were wrong.
2. A design call you made that had a real cost either way.
3. A time you were wrong and it mattered.
4. A time you changed an organization's behaviour, not just its code.

Write them in **STAR + Cost** form: Situation, Task, Action, Result, **and what it cost** —
the last element is what distinguishes a staff-level story. Every real decision has a cost;
candidates who narrate costless wins read as either junior or dishonest.

---

## 7. Frameworks worth memorizing

Interviewers respond extremely well to a named, reusable framework, because it signals you
will make *repeatable* decisions rather than ad-hoc ones.

### Kernel vs userspace

Put it in the kernel only if at least one is true:

1. **It requires privilege that cannot be delegated** — direct hardware access, arbitrating
   between mutually distrustful processes.
2. **The context-switch cost dominates** — the work per syscall is smaller than the syscall.
3. **It must run in a context userspace cannot occupy** — interrupt, atomic, or during
   suspend/reclaim.
4. **It must be a shared, stable abstraction** — many consumers, one arbiter.

If none hold, it is userspace. Then cite the counter-pressure: **the kernel ABI is forever,
userspace ships weekly** — which is the whole argument for Mesa, libinput, libcamera, and
FUSE (Ch. 47 §T.9, Ch. 45 §T.9).

### Choosing a synchronization primitive

Ch. 25's decision procedure, compressed:

```
Is it read-mostly and can readers be pure?        -> seqlock
Is it read-mostly and readers deref pointers?     -> RCU
Is the critical section shorter than a switch?    -> spinlock
Can it sleep?                                     -> mutex
Is it a counter?                                  -> per-CPU, then fold
Is it a shared resource with an owner?            -> refcount + RCU lookup
Is the contention structural?                     -> partition it; do not pick a better lock
```

The last line is the staff-level one.

### Diagnosing any performance problem

The USE method (Brendan Gregg), which works on every resource:

> For every resource: **U**tilization, **S**aturation, **E**rrors.

Plus the kernel-specific refinement: **always separate queueing time from service time.**
`biolatency -Q` vs plain (Ch. 51 Lab 6) is the canonical instance, and the same split
applies to run queues, network queues, and lock acquisition.

### Evaluating a proposed API

1. What does it make *impossible*? (Good APIs remove states, not just add functions.)
2. How does it fail, and is the failure loud?
3. How does it extend in three years without breaking today's callers?
4. Can it be misused by accident? (`pm_runtime_get_sync`'s refcount-on-error is the
   canonical counter-example — Ch. 48 §T.3.)

---

## 8. Ninety-second answers to the twelve most-asked questions

Rehearse these out loud. Timed. The point is not memorization — it is having already found
the structure once, so you are not composing under pressure.

1. **Walk me through what happens on `write()` to a file.** → Ch. 51 §1.4. Nine layers;
   name the transformation at each; end at the doorbell. Mention that nothing reached the
   device.
2. **Explain RCU.** → §4 above.
3. **Spinlock vs mutex — how do you choose?** → Ski-rental: hold time vs context-switch
   cost, plus context (can you sleep?). Ch. 14 §T.2.
4. **What is a memory barrier and when do I need one?** → Ordering is not free; the compiler
   and CPU both reorder. Name the MP pattern (publish a pointer after initializing the
   object) and the SB pattern (the lost-wakeup shape). Ch. 13.
5. **How does the page cache work?** → Ch. 52 §T.3 and §T.7: XArray with tags, dirty as
   unreclaimable memory, writeback as a controller.
6. **Explain `fork()` and COW.** → Ch. 20 §T.2–T.3. Include the HotOS critique.
7. **What happens on a page fault?** → Ch. 22 §T.7's table: eight causes, one mechanism.
   "The MMU is a programmable interception point for deferring work."
8. **How would you debug a kernel panic in production?** → kdump + `crash`/`drgn`, the oops
   decode, then bisect. Ch. 06.
9. **Describe the boot sequence.** → firmware → bootloader → decompressor → `start_kernel` →
   initcalls → initramfs → init. Name one thing that goes wrong at each stage.
10. **What is `-EPROBE_DEFER` for?** → Probe ordering is undecidable without a dependency
    graph; deferral is lazy topological sort by trial and error. Ch. 27 §T.4.
11. **How do you make a driver testable?** → Ch. 50 §T.2: partition it; ~60% needs no
    hardware; extract pure functions; fault-inject the error paths.
12. **When would you use Rust in the kernel?** → New drivers with untrusted input, where the
    bug classes Rust eliminates (UAF, data races, uninitialized reads) dominate the actual
    CVE history for that subsystem. Not a rewrite argument.

---

## 9. Red flags — things that end interviews

Observed repeatedly on hiring committees:

| Behaviour | Why it is fatal |
|---|---|
| Claiming a subsystem you cannot go two levels deep in | Résumés get probed. One layer of bluff is recoverable; two is not |
| "I'd just add a lock" without asking what it protects | Signals you do not think about invariants |
| Reaching for a debugger first | Ch. 06 §T.3: it is the most expensive, least informative tool |
| Designing without asking about constraints | The constraint *is* the problem |
| Not knowing what your own code cost | "Did it help?" must have a number |
| Dismissing userspace as "not real systems work" | Half of modern OS design is the kernel/userspace boundary |
| Being unable to name a failure mode | See §4's corollary |
| Blaming a previous team | Universally scored as a no-hire signal |
| Overrunning without checkpointing | "I have two more minutes of this — useful, or shall I move on?" |

---

## 10. The thirty-day preparation plan

Assumes you have worked through the curriculum. This is *performance* training, not
learning.

**Week 1 — Foundations, out loud.**
Read [os-fundamentals.md](os-fundamentals.md) end to end. Then answer every question in
§8 above out loud, timed at 90 seconds, recorded. Listen back. You will hate it. Do it
anyway — the gap between what you know and what you can *say* is the entire problem.

**Week 2 — Depth.**
Pick your two strongest subsystems. For each, be able to draw on a whiteboard: the central
data structure, the three paths, the locking model, and two historical design changes with
the reason. Then read one *hostile* thing about each — a critique, a rejected patch series,
an LWN comment thread where someone argued the design was wrong.

**Week 3 — Breadth and the bank.**
Work [question-bank.md](question-bank.md), one section per day. Mark every question you
cannot answer in three layers (§4). Those are your list.

**Week 4 — Performance.**
Two [system-design.md](system-design.md) problems and two
[debugging-scenarios.md](debugging-scenarios.md) per day, on a whiteboard, timed, ideally
with a friend playing interviewer. Write your four behavioral stories (§6) and rehearse
them.

**The day before:** do not cram. Re-read your own four stories and the §7 frameworks. Sleep.

---

## 11. Questions to ask them

You are also evaluating. These questions are informative *and* they signal seniority:

- "What fraction of your kernel work goes upstream? What stays out-of-tree, and why?"
- "How do you test kernel changes? What does CI actually run?"
- "When was the last time you carried a revert into a release, and what was the process?"
- "How do you decide to take a stable backport versus rebasing onto a newer LTS?"
- "What is the oldest kernel you still support, and what does that cost you?"
- "Who makes the call when a customer wants a hack that would never go upstream?"
- "What is the thing about this codebase that you would warn a new senior engineer about?"

That last one, asked sincerely, produces remarkably honest answers and is the single best
predictor of what your first six months will feel like.

---

## 12. Compensation and levelling, briefly

Two facts worth knowing:

1. **Level is negotiated before the offer, not after.** If the loop is calibrated for
   senior, the committee cannot award staff no matter how well you do. Ask the recruiter
   in round A which level the loop is calibrated for, and push *then* if it is wrong.
2. **Kernel work is a specialist premium market.** Depth in a scarce subsystem (RT, storage,
   virtualization, security, an unfashionable architecture) is worth more than breadth,
   because the replacement cost is higher. Say which subsystem you own.

---

## Where to go next

| If you need | Read |
|---|---|
| The CS theory the curriculum assumes | [os-fundamentals.md](os-fundamentals.md) |
| Questions with model answers | [question-bank.md](question-bank.md) |
| Architect-level design practice | [system-design.md](system-design.md) |
| Live-debugging practice | [debugging-scenarios.md](debugging-scenarios.md) |
| To fix an actual knowledge gap | The chapter linked from the question |

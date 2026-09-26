# Capstone Projects

> Chapters teach mechanisms; drills build fluency. **Capstones prove capability.** Each of
> these is a multi-week project producing an artefact you can show, describe in an interview,
> and — in several cases — upstream.
>
> Pick based on the role you want. Do at least two: one that builds and one that operates.

---

## How to run a capstone

| Step | |
|---|---|
| **1. Write the spec first** | One page: what it does, what it does not, how you will know it works. Before any code |
| **2. Set a time box** | 3–6 weeks. A capstone that runs six months is a hobby |
| **3. Instrument from the start** | Counters, tracepoints, a state dump. Per `system-design.md` §10 |
| **4. Test the failure paths** | The working path is the easy 20%. The failure paths are the project |
| **5. Measure** | Numbers, before and after. "It's faster" is not a result |
| **6. Write it up** | Problem → design → what you rejected and why → results → what you would do differently |

**The write-up is half the value.** It is the artefact you bring to an interview, and writing
it forces you to discover what you did not understand.

---

## Capstone 1 — Ship a complete device driver, upstream

**Target role:** kernel/driver engineer, embedded platform engineer.
**Time:** 4–8 weeks. **Chapters:** 26–50, 86–89.

Write a driver for real hardware, get it reviewed on the mailing list, and get it merged.

**Scope:**
- Real hardware you own. An I²C sensor, an SPI device, a GPIO expander, an IIO device — small
  is fine and small is more likely to merge.
- Device tree binding in YAML schema form, validated with `dt_binding_check`.
- Full driver-model integration: probe/remove, `devm_`, power management, `-EPROBE_DEFER`
  handling.
- The right subsystem framework — IIO, hwmon, input, or whatever fits. Do not reinvent.
- Error paths that unwind correctly at every point.
- `CONFIG_DEBUG_*` clean: KASAN, lockdep, sparse, smatch, `W=1`.
- A `MAINTAINERS` entry.

**Deliverables:**
- A merged patch series (or, if it does not merge, the thread showing why and what you
  learned).
- The binding document.
- A test procedure someone else can follow.

**What it proves:** you can write kernel code to the standard the kernel accepts, and you can
survive review. This is the single most valuable line on a kernel CV.

**Stretch:** write it in Rust (Ch. 82–83), or write the subsystem abstraction it needs
(Ch. 84).

---

## Capstone 2 — A filesystem, from scratch

**Target role:** storage engineer, filesystem developer.
**Time:** 6–10 weeks. **Chapters:** 51–62.

Implement a working filesystem in the kernel. Not a toy that mounts — one that survives
`fsstress` and a power cut.

**Scope:**
- On-disk format: superblock, inodes, directories, an allocation scheme. Design it and
  document the layout before writing code.
- VFS integration: `super_operations`, `inode_operations`, `file_operations`,
  `address_space_operations`.
- Page cache integration, ideally via `iomap`.
- `mkfs` and `fsck` tools.
- Crash consistency: journalling or CoW. **This is the hard part and the point.**
- `xfstests` — get the generic tests passing.

**Deliverables:**
- A mountable filesystem passing the generic `xfstests` subset.
- A crash-consistency test: kill the machine mid-write, verify invariants.
- A design document explaining the format and the consistency argument.
- Performance numbers against ext4 and XFS, with an honest account of where you lose.

**What it proves:** you understand the VFS, the page cache, and — most importantly — crash
consistency, which is the thing that separates storage engineers from people who have used a
filesystem.

**Stretch:** implement reflinks, or a compression layer, or make it work on zoned storage.

---

## Capstone 3 — A high-performance network data path

**Target role:** networking engineer, performance engineer.
**Time:** 4–8 weeks. **Chapters:** 71–76, 105.

Build something that processes packets at line rate and prove it.

**Scope, pick one:**
- **An XDP-based load balancer** — consistent hashing, connection tracking in BPF maps,
  `XDP_TX`/`XDP_REDIRECT`.
- **A DDoS mitigation filter** — rate limiting, blocklists in an LPM trie, with a userspace
  control plane.
- **An AF_XDP userspace data path** — zero-copy, with a comparison against the kernel path.
- **An `io_uring`-based proxy** — multishot accept/recv, provided buffers, zero-copy send.

**Requirements regardless:**
- Measured at ≥ 10 Gbps, with p99 latency, not just throughput.
- A fair baseline: the kernel-path equivalent, tuned.
- CPU cost per packet, and where it goes (`perf`, flame graph).
- `perf c2c` clean — no false sharing on the fast path.
- Correct behaviour under overload: what gets dropped, and is it visible?

**Deliverables:**
- The code, with the userspace control plane.
- A benchmark harness anyone can re-run.
- A performance report: throughput, p50/p99/p99.9, CPU/packet, and the profile.
- An explanation of every optimization and what it bought.

**What it proves:** you can reason about per-packet cost budgets, use the modern kernel
interfaces, and measure honestly.

---

## Capstone 4 — A real-time control system

**Target role:** real-time/embedded engineer, robotics, industrial.
**Time:** 4–6 weeks. **Chapters:** 17–21, 103, 90–101.

Build a system meeting a stated hard deadline, and *prove* it meets it.

**Scope:**
- A control loop with a stated period and deadline — 1 ms is achievable, 100 µs is a real
  challenge.
- `PREEMPT_RT` kernel, fully isolated core, all the tuning from Ch. 103 §T.7.
- A correct RT application: `mlockall`, prefaulted stack, absolute-time periodic wakeup, no
  allocation in the loop, PI mutexes or lock-free communication.
- A non-RT side doing real work (logging, networking, a UI) and communicating via a lock-free
  ring.
- **Validation:** `cyclictest` under representative load for ≥ 24 hours, `hwlatdetect` to
  separate hardware latency, and `rtla timerlat`/`osnoise` to attribute every outlier.

**Deliverables:**
- The system, running.
- A latency histogram over ≥ 24 hours under load, reporting the **maximum**.
- An attribution of the worst outliers to their source.
- The configuration, documented, with a justification for each setting.
- An honest statement of what would break the guarantee.

**What it proves:** you understand that real time means *bounded*, you can decompose a latency
budget, and you validate rather than assert.

**Stretch:** an AMP design — Linux on one core, a bare-metal or Zephyr loop on another,
communicating via `rpmsg`. Compare the achievable bounds.

---

## Capstone 5 — A production embedded Linux product

**Target role:** embedded platform architect, BSP engineer.
**Time:** 6–12 weeks. **Chapters:** 88, 90–101.

Take a board from bare metal to something you could ship.

**Scope:**
- A real board. A vendor EVK is fine; a custom board is better.
- The full bring-up sequence of Ch. 101: bootloader, DRAM validation, kernel, peripherals.
- A Yocto (or Buildroot) BSP layer, properly structured — hardware in the BSP, policy in the
  distro, product in the product layer.
- Secure boot, rooted in fuses, with anti-rollback.
- A production image: no `debug-tweaks`, hardened kernel, read-only root with `dm-verity`.
- A/B updates with signed bundles and tested rollback.
- The compliance package: licence manifest, source archive, SBOM, CVE report.
- A reproducible build.
- A maintenance plan (Ch. 101 Lab 6) with the delta categorized and an upstreaming schedule.

**Deliverables:**
- A booting, updatable, hardened product image.
- The BSP layer, with every patch carrying `Upstream-Status`.
- Documentation: bring-up log, hardware notes, workarounds, runbook, maintenance plan.
- A ten-year archive, **verified by an offline rebuild**.

**What it proves:** you can take a product to ship, not just to boot. This is the whole
embedded platform role.

---

## Capstone 6 — Scale a subsystem to many cores

**Target role:** performance engineer, kernel developer, staff+ IC.
**Time:** 4–8 weeks. **Chapters:** 13–16, 105.

Find a real scalability bottleneck and fix it.

**Scope:**
- Find a workload that does not scale — a kernel subsystem, an open-source application, or
  your own code.
- **Measure first**: a scaling curve at 1…N cores, fitted to the USL. Report α and β and
  predict the peak.
- Find the cause: `perf c2c` for cachelines, `perf lock`/`lock_stat` for contention,
  `offcputime` for blocking.
- Fix it, climbing the ladder (Ch. 105 §T.6) only as far as necessary.
- Re-measure; re-fit the USL; show β moved.

**Deliverables:**
- The scaling curves, before and after, with fitted α and β.
- The `perf c2c` output identifying the specific cachelines.
- The patch.
- A write-up explaining why the fix works in MESI terms.

**What it proves:** you can do rigorous performance work — measure, attribute, fix, verify —
rather than guess-and-check.

**Stretch:** upstream the fix. Scalability patches with data are well received.

---

## Capstone 7 — A Rust kernel abstraction

**Target role:** kernel developer, Rust-for-Linux contributor.
**Time:** 4–8 weeks. **Chapters:** 77–85.

Write a safe Rust abstraction for a C kernel API that has none, and get it reviewed.

**Scope:**
- Pick a subsystem with no Rust abstraction. Check `rust/kernel/` and the mailing list first —
  someone may be working on it.
- **The hard part is extracting the C contract** (Ch. 84 §T.10). Read the implementation, read
  the callers, read the git history, then *ask the maintainer* whether your statement of the
  contract is correct.
- Write the abstraction: type invariants, `# Safety` contracts on every `unsafe fn`,
  `// SAFETY:` on every block, justified `Send`/`Sync`.
- Attack it: can safe code cause UB? Does soundness depend on `Drop` running
  (`mem::forget`)? Can a hostile call order break it?
- A driver using it, with zero `unsafe`.
- `kunit` tests.

**Deliverables:**
- The abstraction, posted for review.
- The adversarial analysis document.
- A sample driver.
- The mailing-list thread, and what the review taught you.

**What it proves:** you can work at the boundary between two languages' safety models and
write code others will trust. This is genuinely scarce.

---

## Capstone 8 — An observability platform

**Target role:** SRE, performance engineer, platform engineer.
**Time:** 4–6 weeks. **Chapters:** 06, 75, 95.

Build the tooling that answers questions about a system you cannot reproduce.

**Scope:**
- Continuous profiling: a low-overhead agent (19–99 Hz), symbolization via build IDs, and
  storage with history.
- Differential flame graphs — the feature that actually finds regressions.
- Custom eBPF tools for a subsystem you care about: latency attribution, queue depths, error
  rates.
- Crash-dump automation: kdump configured, dumps collected, `drgn` scripts producing a triage
  report automatically.
- A dashboard driven by **PSI**, not utilization.

**Deliverables:**
- The deployed system, running on something real.
- Overhead measurements proving it is affordable.
- A demonstration: introduce a regression, find it from the data alone.
- The `drgn` triage scripts.
- A runbook mapping symptoms to tools.

**What it proves:** you can build the capability to diagnose problems you cannot reproduce,
which is the defining constraint of production engineering.

---

## Capstone 9 — A security hardening programme

**Target role:** security engineer, platform architect.
**Time:** 4–8 weeks. **Chapters:** 88, 93, 99, 102.

Take a real system and harden it, with evidence.

**Scope:**
- A stated **threat model** — without one, hardening is cargo cult.
- Kernel: the full config audit (`kernel-hardening-checker`), lockdown, module signing,
  mitigations decided deliberately with measurements.
- Userspace: seccomp filters per service, systemd hardening to
  `systemd-analyze security` < 3, capability reduction, MAC policy.
- Boot: verified chain, anti-rollback, `dm-verity`.
- Supply chain: SBOM, CVE process, continuous stable adoption.
- **Measure the cost**: throughput, latency, boot time, before and after each control.

**Deliverables:**
- The hardened system.
- A threat model document with the controls mapped to threats.
- A cost table: every mitigation, what it buys, what it costs.
- Verification: tests proving each control actually applies.
- The memo for a decision-maker recommending what to enable and what to skip, with reasons.

**What it proves:** you can reason about security as engineering — threats, controls, costs,
evidence — rather than as a checklist.

---

## Capstone 10 — Kernel-or-userspace, decided with data

**Target role:** staff/principal engineer, architect.
**Time:** 3–5 weeks. **Chapters:** 00, 24, 75, 76, 89.

Take a real problem, implement it **both ways**, and write the decision.

**Scope:**
- A problem with a genuine kernel-vs-userspace question: a packet filter, a custom scheduler,
  a storage transform, a security policy engine.
- **Implement it three ways**: userspace with batching (`io_uring`), eBPF, and an in-kernel
  module.
- Measure all three: throughput, p99 latency, CPU cost, development time, lines of code.
- Evaluate against the four criteria (Ch. 89 §T.3): privilege, performance,
  policy-vs-mechanism, blast radius.
- Write the decision record (Ch. 89 Lab 3), committing with conditions.

**Deliverables:**
- Three working implementations.
- A benchmark harness and a results table.
- An architecture decision record with the recommendation, the conditions, and what it costs.
- A presentation defending it to someone who disagrees.

**What it proves:** you make architectural decisions with data rather than taste, and you can
defend them. This is precisely what a principal-level interview tests.

---

## Choosing

| If you want to be | Do |
|---|---|
| A kernel/driver developer | **1**, plus 2, 6, or 7 |
| A storage engineer | **2**, plus 6 |
| A networking engineer | **3**, plus 6 |
| An embedded platform engineer | **5**, plus 1 or 4 |
| A real-time engineer | **4**, plus 5 |
| A performance engineer | **6**, plus 8 |
| A security engineer | **9**, plus 1 |
| An SRE / production engineer | **8**, plus 6 |
| A staff/principal IC | **10**, plus any one build capstone |

**The general advice:** do one *build* capstone (1, 2, 3, 4, 5, 7) and one *operate* capstone
(6, 8, 9, 10). The combination is what makes you credible across the whole range an interview
covers — and it is what makes you actually useful.

---

## The meta-capstone

After any two capstones, do this:

**Write the post-mortem.** Not of a failure — of the project.

1. What did you predict, and what actually happened?
2. What took longest, and was it what you expected?
3. What did you get wrong, and what would have prevented it?
4. Which chapter's material turned out to matter most? Which did you never use?
5. What would you do differently, and what is the general lesson?
6. What do you still not understand?

**Question 6 is the important one.** The list of things you know you do not understand is the
most valuable output of any project, and it is the honest answer to "what are you working on
learning?" — which is a question every good interviewer asks.

---

→ Back: [README.md](README.md) | [../README.md](../README.md)

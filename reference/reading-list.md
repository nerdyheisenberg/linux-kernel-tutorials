# Reading List

> Curated, opinionated, and ordered. Not everything is worth your time; this says what is,
> why, and when to read it. Items marked **★** are the ones I would insist on.

---

## 1. If you read only five things

1. **★ McKenney, *Is Parallel Programming Hard, And, If So, What Can You Do About It?***
   Free. The kernel's concurrency bible, by the author of RCU. Chapters 2–5 (counting,
   partitioning, deferred processing) are the intellectual core of Parts 1 and 8.

2. **★ Arpaci-Dusseau & Arpaci-Dusseau, *Operating Systems: Three Easy Pieces***
   Free. The best OS textbook, by a margin. Everything in `os-fundamentals.md` derives from
   it, and it is genuinely enjoyable to read.

3. **★ Brendan Gregg, *Systems Performance*, 2nd ed.**
   The definitive performance book. The USE method, every tool, and — more importantly — the
   methodology that stops you guessing.

4. **★ Lampson, "Hints for Computer System Design," *SOSP* 1983**
   Twenty pages. The best paper on systems design judgement ever written. Read it annually;
   you will get something different each time.

5. **★ Chris Simmonds, *Mastering Embedded Linux Programming*, 3rd ed.**
   The only good book covering Part 7's material. If you work in embedded, this is
   non-negotiable.

---

## 2. Kernel fundamentals

**★ Love, *Linux Kernel Development*, 3rd ed.** — Dated (2010, pre-folios, pre-EEVDF,
pre-blk-mq) but still the clearest *explanation* of the kernel's structure. Read it for the
model, not the details. Chapters 4, 7–11, and 20 remain excellent.

**Bovet & Cesati, *Understanding the Linux Kernel*, 3rd ed.** — Very dated (2.6) but
unmatched for the x86 memory-management and scheduling internals. Use it to understand *why*
things are shaped as they are.

**Corbet, Rubini, Kroah-Hartman, *Linux Device Drivers*, 3rd ed.** — Free online. Obsolete in
its APIs; Part 2 of this curriculum supersedes it. Read chapter 1 for the philosophy and skip
the rest.

**Gorman, *Understanding the Linux Virtual Memory Manager*** — Free. Ancient (2.4) and still
the best explanation of the buddy allocator, slab, and the page-replacement machinery.

**Mauerer, *Professional Linux Kernel Architecture*** — Comprehensive, dated, useful as a
reference for subsystems nobody else explains.

**The kernel's own `Documentation/`** — Genuinely good and improving. Start with
`Documentation/process/`, `Documentation/core-api/`, `Documentation/locking/`, and
`Documentation/admin-guide/`. It is more current than any book.

---

## 3. Concurrency and memory models

**★ McKenney, *Is Parallel Programming Hard…*** — see §1.

**★ Mara Bos, *Rust Atomics and Locks*** — Free online. The best explanation of `Send`/`Sync`
and the memory model in any language. Chapters 1–4 apply directly to Ch. 13 and Ch. 81, even
if you never write Rust.

**Herlihy & Shavit, *The Art of Multiprocessor Programming*, 2nd ed.** — The theory:
linearizability, lock-free algorithms, the ABA problem. Rigorous.

**`tools/memory-model/` in the kernel tree** — LKMM, `herd7`, and litmus tests. The formal
specification of what the kernel's barriers mean.

**Preshing's blog (preshing.com)** — The clearest informal writing on memory ordering
anywhere. Read the acquire/release series.

**Papers:**
- Mellor-Crummey & Scott, "Algorithms for Scalable Synchronization," *TOCS* 1991 — the MCS
  lock, behind `qspinlock`
- Sha, Rajkumar, Lehoczky, "Priority Inheritance Protocols," *IEEE Trans. Computers* 1990
- David, Guerraoui, Trigonakis, "Everything You Always Wanted to Know About
  Synchronization," *SOSP* 2013 — excellent empirical data

---

## 4. Memory management

**★ Drepper, "What Every Programmer Should Know About Memory" (2007)** — Free, long, and
still the best explanation of cache and NUMA hardware behaviour. Sections 2, 3, and 5 are the
essential ones.

**Gorman, *Understanding the Linux VM Manager*** — see §2.

**Papers:**
- Denning, "The Working Set Model for Program Behavior," *CACM* 1968
- Belady, "A Study of Replacement Algorithms," *IBM Systems Journal* 1966
- Jiang & Zhang, "LIRS: An Efficient Low Inter-reference Recency Set Replacement Policy,"
  *SIGMETRICS* 2002 — the ideas behind Linux's scan resistance
- Johnson & Shasha, "2Q: A Low Overhead High Performance Buffer Management Replacement
  Algorithm," *VLDB* 1994

**LWN:** the folio series, the MGLRU series, and the memory-tiering/CXL coverage.

---

## 5. Storage and filesystems

**★ Brendan Gregg, *Systems Performance*, ch. 9** — the best treatment of storage performance
analysis.

**★ Pillai et al., "All File Systems Are Not Created Equal," *OSDI* 2014** — the
crash-consistency paper. Essential for anyone writing or relying on storage code.

**Papers:**
- McKusick et al., "A Fast File System for UNIX," *TOCS* 1984 — cylinder groups, and the
  origin of locality-aware allocation
- Rosenblum & Ousterhout, "The Design and Implementation of a Log-Structured File System,"
  *SOSP* 1991
- Hitz, Lau, Malcolm, "File System Design for an NFS File Server Appliance," 1994 — WAFL, the
  ancestor of every CoW filesystem
- Rodeh, Bacik, Mason, "BTRFS: The Linux B-Tree Filesystem," *TOS* 2013
- Rebello et al., "Can Applications Recover from fsync Failures?," *ATC* 2020 — they mostly
  cannot
- Bairavasundaram et al., "An Analysis of Latent Sector Errors in Disk Drives,"
  *SIGMETRICS* 2007 — why you scrub

**★ Backblaze Drive Stats** — quarterly, public, the largest real-world failure dataset
anywhere. Read the SMART analysis; it will change how you interpret SMART.

**The Linux Storage Stack Diagram** (Werner Fischer) — print it.

---

## 6. Networking

**★ Stevens, *TCP/IP Illustrated*, Vol. 1, 2nd ed.** — the protocol reference.

**Benvenuti, *Understanding Linux Network Internals*** — dated but unmatched for the
receive/transmit path structure.

**Papers:**
- Mogul & Ramakrishnan, "Eliminating Receive Livelock in an Interrupt-Driven Kernel,"
  *TOCS* 1997 — why NAPI exists
- Høiland-Jørgensen et al., "The eXpress Data Path," *CoNEXT* 2018 — XDP
- Jacobson, "Congestion Avoidance and Control," *SIGCOMM* 1988

**LWN:** the BPF and XDP coverage, and the `io_uring` archive.

---

## 7. Rust

**★ *The Rustonomicon*** — Free. **Read it completely before writing any `unsafe`
abstraction.** The "Implementing Vec" chapter is the best worked example of building a safe
abstraction over unsafe code that exists.

**Klabnik & Nichols, *The Rust Programming Language*** — Free. The book. Chapters 4, 10, 15,
16 are the load-bearing ones for kernel work.

**Blandy, Orendorff, Tindall, *Programming Rust*, 2nd ed.** — The best systems-oriented Rust
book, for people who already think in pointers.

**Gjengset, *Rust for Rustaceans*** — The intermediate book. Chapters on types, interface
design, and unsafe map directly onto Ch. 84.

**★ Mara Bos, *Rust Atomics and Locks*** — see §3.

**Papers:**
- Jung et al., "RustBelt: Securing the Foundations of the Rust Programming Language,"
  *POPL* 2018 — the soundness proof
- Jung et al., "Stacked Borrows: An Aliasing Model for Rust," *POPL* 2020
- Astrauskas et al., "How Do Programmers Use Unsafe Rust?," *OOPSLA* 2020

**★ Google Security Blog, "Eliminating Memory Safety Vulnerabilities at the Source" (2024)** —
the Android data and, more importantly, the *new-code-not-rewrites* argument.

---

## 8. Security

**★ Anderson, *Security Engineering*, 3rd ed.** — Free online. The best book on threat
modelling. Chapters 1–6 are essential regardless of your domain.

**Saltzer & Schroeder, "The Protection of Information in Computer Systems," 1975** — the eight
principles. Still the framework.

**Papers:**
- Kocher et al., "Spectre Attacks," *IEEE S&P* 2019
- Lipp et al., "Meltdown," *USENIX Security* 2018
- Wright et al., "Linux Security Modules," *USENIX Security* 2002 — why LSM is a hook
  framework, not a policy
- Kemerlis et al., "ret2dir: Rethinking Kernel Isolation," *USENIX Security* 2014
- Abadi et al., "Control-Flow Integrity," *CCS* 2005
- Klein et al., "seL4: Formal Verification of an OS Kernel," *SOSP* 2009 — the other answer

**`Documentation/admin-guide/hw-vuln/`** — one file per speculative-execution vulnerability,
by the people who wrote the mitigations. The most authoritative source.

**Tools to read about:** `kernel-hardening-checker`, `syzkaller`.

---

## 9. Real-time

**★ Buttazzo, *Hard Real-Time Computing Systems*, 3rd ed.** — the standard text. Chapters 4
(periodic scheduling) and 7 (resource access protocols).

**Liu, *Real-Time Systems*** — more formal, excellent on schedulability analysis.

**Kopetz, *Real-Time Systems: Design Principles for Distributed Embedded Applications*** —
the architecture view, including time-triggered design.

**Papers:**
- Liu & Layland, "Scheduling Algorithms for Multiprogramming in a Hard-Real-Time
  Environment," *JACM* 1973 — RM and EDF
- Abeni & Buttazzo, "Integrating Multimedia Applications in Hard Real-Time Systems,"
  *RTSS* 1998 — the CBS behind `SCHED_DEADLINE`
- ★ Reghenzani, Massari, Fornaciari, "The Real-Time Linux Kernel: A Survey on PREEMPT_RT,"
  *ACM Computing Surveys* 2019
- de Oliveira et al., "Demystifying the Real-Time Linux Scheduling Latency," *ECRTS* 2020 —
  the model behind the `osnoise` tracer

---

## 10. Virtualization

**★ Bugnion, Nieh, Tsafrir, *Hardware and Software Support for Virtualization*** — the best
single book; concise and current.

**Papers:**
- Popek & Goldberg, "Formal Requirements for Virtualizable Third Generation Architectures,"
  *CACM* 1974
- Barham et al., "Xen and the Art of Virtualization," *SOSP* 2003
- Kivity et al., "kvm: the Linux Virtual Machine Monitor," 2007 — short and very readable
- Ben-Yehuda et al., "The Turtles Project," *OSDI* 2010 — nested virtualization
- Russell, "virtio: Towards a De-Facto Standard for Virtual I/O Devices," *OSR* 2008
- ★ Agache et al., "Firecracker: Lightweight Virtualization for Serverless Applications,"
  *NSDI* 2020 — **read this one**; the clearest statement of the modern isolation trade
- Young et al., "The True Cost of Containing: A gVisor Case Study," *HotCloud* 2019

**Code:** `cloud-hypervisor`, `firecracker`, and `kvmtool` — all far smaller than QEMU and
therefore readable.

---

## 11. Scalability

**★ Boyd-Wickizer et al., "An Analysis of Linux Scalability to Many Cores," *OSDI* 2010** —
*the* paper on this topic.

**★ Clements et al., "The Scalable Commutativity Rule," *SOSP* 2013** — a profound result:
whenever interface operations commute, they can be implemented to scale. This makes
scalability an **interface design** property. Underread; read it.

**Gunther, *Guerrilla Capacity Planning*** — the Universal Scalability Law, and how to fit it
to real data.

**Dashti et al., "Traffic Management: A Holistic Approach to Memory Placement on NUMA
Systems," *ASPLOS* 2013** — why congestion matters, not just latency.

**Lameter, "NUMA: An Overview," *ACM Queue* 2013.**

---

## 12. Containers and userspace

**★ Michael Kerrisk, *The Linux Programming Interface*** — the userspace reference. Enormous
and worth owning. Chapters 24–28 (processes), 37 (daemons), 41–42 (shared libraries).

**★ Kerrisk's LWN namespace series (7 parts, 2013)** — the best explanation of namespaces
anywhere, and still accurate.

**Liz Rice, *Container Security*** — the best single book; practical and accurate. Her
"Containers from Scratch" talks are the live-coded version.

**★ Drepper, "How To Write Shared Libraries"** — free PDF. The definitive document on dynamic
linking, PLT/GOT, and symbol versioning.

**Levine, *Linkers and Loaders*** — the textbook; free drafts online.

**Bryant & O'Hallaron, *Computer Systems: A Programmer's Perspective*, ch. 7** — the best
pedagogical treatment of linking.

---

## 13. Embedded Linux and build systems

**★ Chris Simmonds, *Mastering Embedded Linux Programming*, 3rd ed.** — see §1. Chapters 2–6
and 9–11 cover most of Part 7.

**Streif, *Embedded Linux Systems with the Yocto Project*** — the best Yocto book; dated in
details, sound on concepts.

**Salvador & Angolini, *Embedded Linux Development Using Yocto Project*** — more current and
more practical.

**★ The Buildroot user manual** — genuinely good, and short enough to read completely in an
afternoon. Do that.

**★ Bootlin's training materials** — free, comprehensive, better than most books. The embedded
Linux, kernel, and Yocto courses in particular.

**Vasquez & Simmonds, *Mastering Embedded Linux Development*.**

**Liberal de los Ríos, *Linux Driver Development for Embedded Processors*.**

---

## 14. Engineering, process, and judgement

**★ Lampson, "Hints for Computer System Design"** — see §1.

**Saltzer, Reed, Clark, "End-to-End Arguments in System Design," *TOCS* 1984** — where
function belongs in a layered system.

**Ousterhout, *A Philosophy of Software Design*** — short and opinionated. "Deep modules" maps
exactly onto narrow waists.

**Fogel, *Producing Open Source Software*** — free. The best book on running a project; the
governance material applies directly to maintainership.

**Nadia Eghbal, *Working in Public*** — the modern analysis of maintainer burnout and the
economics of open source. Read it if you plan to maintain anything.

**Will Larson, *Staff Engineer*** and ***An Elegant Puzzle*** — the best account of the
senior/staff/principal role outside the kernel context.

**★ `Documentation/process/` in the kernel tree** — `submitting-patches.rst`,
`development-process.rst`, `management-style.rst`, `stable-api-nonsense.rst`. All short, all
essential.

**Tetlock, *Superforecasting*** — calibration, and the discipline of stating what would change
your mind.

---

## 15. Continuous sources

**★ LWN.net** — a subscription is the single best professional investment available to a
kernel engineer. Read it weekly. The "Kernel index" pages and the annual conference coverage
(Kernel Summit, Maintainers Summit, LPC, Kangrejos) are where you learn what the community is
actually worried about.

**lore.kernel.org** — the mailing-list archive. Learn its search syntax. Reading review
threads in a subsystem you care about, weekly, teaches more about kernel engineering than any
book.

**`git log` on the kernel tree** — underrated. `git log --oneline -20 <file>` before touching
anything.

**The kernel's release notes and `Documentation/` changes** — `git log v6.11..v6.12 --
Documentation/` after each release.

**Conference talks:** Linux Plumbers, Embedded Open Source Summit, Kangrejos, SCALE, and
FOSDEM. Most are recorded and free.

**Blogs worth following:** Brendan Gregg (performance), Paul McKenney (RCU and concurrency),
Jake Edge and Jonathan Corbet (LWN), Mara Bos (Rust), Thomas Petazzoni and the Bootlin blog
(embedded), and the Rust for Linux project.

---

## 16. A reading order

If you are starting from zero and want a path:

| Phase | Read |
|---|---|
| **Foundations** | *Three Easy Pieces* (all), Love *LKD* ch. 1–11, `Documentation/process/` |
| **Concurrency** | McKenney ch. 1–5, Drepper's memory paper §2–5, *Rust Atomics and Locks* ch. 1–4 |
| **Performance** | Gregg *Systems Performance* ch. 1–6, then ch. 9 |
| **Storage** | *OSTEP* persistence chapters, "All File Systems Are Not Created Equal" |
| **Design** | Lampson's "Hints," the end-to-end paper, Ousterhout |
| **Specialization** | Buttazzo (RT), Bugnion (virt), Anderson (security), Boyd-Wickizer (scale) |
| **Embedded** | Simmonds (all), Buildroot manual, Bootlin materials |
| **Ongoing** | LWN weekly, lore for your subsystem, one paper per week |

**One paper a week for a year is 52 papers.** That is more systems literature than most
engineers read in a career, and it compounds.

→ Back: [../README.md](../README.md) | Next: [../labs/README.md](../labs/README.md)

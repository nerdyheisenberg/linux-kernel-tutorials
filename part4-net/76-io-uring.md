# Chapter 76 — io_uring

> The last chapter of Part 4, and the one that closes a circle. Ch. 24 established that the
> syscall is the kernel's narrow waist and that it costs 200–500 ns with mitigations.
> Ch. 63 showed a block layer capable of millions of IOPS. Ch. 74 showed XDP removing
> per-packet allocation. `io_uring` is the general answer to the same pressure: **if the
> boundary crossing is the cost, stop crossing the boundary.**

---

## Theory & First Principles

### T.0 — Start here: count the syscalls in a modern server

A proxy handling 1,000,000 requests per second, doing the simplest possible thing per request:

```
  epoll_wait  (amortized)   |
  read()                    |  ~4 syscalls per request
  write()                   |
  (timer / bookkeeping)     |

  4,000,000 syscalls/second x ~1 us each (WITH Spectre/Meltdown mitigations)
  = 4 CPU-seconds per second of pure transition overhead.
```

**Four entire cores doing nothing but crossing the user/kernel boundary.** And the syscall got
*worse*, not better: KPTI, retpolines, and IBRS turned a ~100 ns operation into a ~1 µs one.
An overhead everyone had treated as negligible for thirty years became the dominant cost.

**Two separate problems, and it is important to see that they are separate:**

| Problem | Cost |
|---|---|
| **Transition overhead** — one boundary crossing per operation | ~1 µs each, unavoidable per-syscall |
| **`epoll` is readiness-based, not completion-based** | you are told "you may now read" — then you still have to *do* the read, which itself blocks for files |

The second is the one people miss. `epoll` never worked for regular files: a file is *always*
"ready," and the `read()` can still block for 80 µs on I/O. Which is why every async framework
before 2019 had a thread pool bolted onto the side for file I/O. And Linux AIO (`io_submit`)
solved neither cleanly — it required `O_DIRECT`, silently blocked otherwise, and supported
almost nothing beyond read/write.

**`io_uring`'s answer is two shared ring buffers in memory mapped by both the kernel and your
process:**

```
   userspace                                          kernel
   ---------                                          ------
   write SQE -> [ SUBMISSION QUEUE ]  -- shared mmap -->  consumes
                 (you own the tail,                       (owns the head)
                  kernel owns the head)

   read  CQE <- [ COMPLETION QUEUE ]  <- shared mmap ---  produces

   Submit N operations: N stores to memory, then ONE io_uring_enter().
   Or, with SQPOLL: a kernel thread polls the ring -- ZERO syscalls.
```

**You have seen this exact structure before, twice.** It is NVMe's submission/completion queue
pair with a doorbell (Ch. 69 §T.0), and it is virtio's virtqueue (Ch. 104). **The same answer
appears whenever two agents that cannot cheaply call each other must exchange work:**

> **Put the work in shared memory. Make the expensive notification optional and batched. Let
> the consumer poll when the rate is high enough to justify it.**

NAPI (Ch. 46) is the same idea applied to interrupts; the interrupt-versus-poll crossover is
the same ski-rental calculation (Ch. 14 §T.0, Ch. 48).

**Three things `io_uring` adds beyond batching, which is why it became a general interface and
not a networking one:**

1. **It is completion-based and universal.** Over 50 operation types — `read`, `write`,
   `accept`, `connect`, `openat`, `statx`, `fsync`, `madvise`, `send`, `recv`, `timeout`.
   Buffered file I/O works properly, which Linux AIO never managed.
2. **Operations can be *linked*.** `IOSQE_IO_LINK` makes `accept -> read -> write` a chain
   submitted as one unit, with no userspace round trip between the steps. **Batching the
   *dependency graph*, not just the calls.**
3. **Registered files and buffers** amortize the per-call work of resolving an fd and pinning
   user memory — the same "pay once, not per operation" move again.

**And the cost, stated plainly, because it is significant.** A shared-memory interface with
50+ opcodes and asynchronous execution in kernel worker threads is an enormous attack surface,
and `io_uring` has been the source of a long run of serious vulnerabilities — enough that
Google disabled it on ChromeOS and Android, and several distributions gate it behind
`kernel.io_uring_disabled`. **Performance interfaces that bypass careful serial checking trade
auditability for speed**, and that is the honest summary of this chapter and, arguably, of
Parts 3 and 4 together.

```bash
cat /proc/sys/kernel/io_uring_disabled
sudo fio --ioengine=io_uring --sqthread_poll=1 --name=t --rw=randread \
         --bs=4k --iodepth=128 --filename=/dev/nvme0n1 --runtime=10
ls /sys/kernel/debug/tracing/events/io_uring/
sudo bpftrace -e 'tracepoint:io_uring:io_uring_submit_sqe { @[args->opcode] = count(); }'
```

---

### T.1 — The problem, stated arithmetically

Linux's asynchronous I/O story was, for twenty years, a series of partial answers:

| Mechanism | Works for | Fails for |
|---|---|---|
| Blocking syscalls + threads | everything | context-switch cost, memory per thread |
| `select`/`poll` | sockets, pipes | O(n) per call, rebuilds the set each time |
| `epoll` | sockets, pipes, at scale | **regular files are always "ready"** — useless for file I/O |
| POSIX AIO (glibc) | nothing, really | implemented with a thread pool in userspace |
| **Linux AIO** (`io_submit`) | `O_DIRECT` file I/O | silently blocks for buffered I/O, metadata, or when the request queue is full; awkward API; only ~3 operations |
| `sendfile`/`splice` | specific copy patterns | not general |

The deeper problem is that `epoll` implements **readiness** notification, not **completion**.
"The fd is readable" still requires a `read()` syscall afterwards. So the minimum cost of
handling one network event is two syscalls: `epoll_wait` (amortized) plus `read`. At
500 ns each and a target of 1M events/sec, you have spent half a core on boundary crossings
before doing any work.

`io_uring` (Jens Axboe, 5.1, 2019) changes the model to **completion** notification over a
**shared-memory ring**, so the syscall becomes optional.

### T.2 — The architecture: two rings in shared memory

```
                      mmap'd, shared between user and kernel
   ┌──────────────────────────────────────────────────────────────┐
   │  Submission Queue (SQ)              Completion Queue (CQ)    │
   │  ┌────────────────────┐             ┌──────────────────────┐ │
   │  │ head (kernel)      │             │ head (user)          │ │
   │  │ tail (user)   ─────┼──┐       ┌──┼─ tail (kernel)       │ │
   │  │ ring[] -> indices  │  │       │  │ cqes[]               │ │
   │  └────────────────────┘  │       │  └──────────────────────┘ │
   │  ┌────────────────────┐  │       │                           │
   │  │ SQE array          │◄─┘       │                           │
   │  │ [op, fd, off, len, │          │                           │
   │  │  flags, user_data] │          │                           │
   │  └────────────────────┘          │                           │
   └───────────────────────────────────┼───────────────────────────┘
                                       │
   user: write SQE, advance SQ tail ───┘
   kernel: consume, execute, write CQE, advance CQ tail
```

Each ring is a **single-producer, single-consumer** ring with the producer and consumer on
opposite sides of the privilege boundary. That is the crucial structural property:

- The SQ is produced by userspace and consumed by the kernel.
- The CQ is produced by the kernel and consumed by userspace.
- Because each index has exactly one writer, no locks are needed — only **memory barriers**
  (`smp_store_release` on the tail, `smp_load_acquire` on the head) to order the data writes
  against the index publication. This is exactly the publication problem of Ch. 13 §T.5,
  applied across the syscall boundary.

The SQ has one level of indirection (an array of indices into an SQE array) so userspace can
prepare SQEs out of order and submit them in a chosen order — useful when building a batch
incrementally.

**Three modes of operation**, in increasing aggressiveness:

| Mode | Syscalls per N operations | Cost |
|---|---|---|
| Default | 1 `io_uring_enter` per batch | amortized syscall |
| `IORING_SETUP_SQPOLL` | **0** — a kernel thread polls the SQ | one burned kernel thread per ring |
| `IORING_SETUP_IOPOLL` | 0 for completions — busy-poll the device | a burned core; `O_DIRECT` only |

The first mode alone is usually the win: batching 32 operations into one syscall turns
500 ns/op of overhead into 16 ns/op.

### T.3 — What actually happens to a request

```
 io_uring_enter()
   └─ io_submit_sqes()
        └─ for each SQE:
             io_init_req()        -- allocate io_kiocb from a cached pool
             io_issue_sqe()
               │
               ├─ Try NON-BLOCKING first (IOCB_NOWAIT).
               │  If it completes -> write CQE inline. NO context switch at all.
               │  (page cache hit, ready socket, ...)
               │
               ├─ Returns -EAGAIN and the fd is pollable
               │  -> arm poll internally; when ready, retry from the
               │     poll callback. This is "IOPOLL-free async":
               │     no thread, no blocking.
               │
               └─ Returns -EAGAIN and the op genuinely must block
                  (buffered read that misses, getdents, statx on a cold
                   inode, anything not pollable)
                  -> punt to io-wq: a bounded per-ring kernel worker pool.
```

**This three-tier fallback is the design's core insight.** Linux AIO's fatal flaw was that
there was no tier 3 — a buffered read that missed the page cache silently blocked the
submitting thread, which destroyed the "asynchronous" promise. `io_uring` guarantees the
submitter never blocks by having a worker pool as a backstop, while making the fast path
(tier 1) take no worker at all.

`struct io_kiocb` is the per-request object. Requests are allocated from a per-ring cache
(`io_alloc_cache`) to keep submission allocation-free in the common case.

### T.4 — Feature surface

`io_uring` is no longer an I/O interface; it is **a general asynchronous syscall interface**.
~60 opcodes as of 6.12:

| Category | Ops |
|---|---|
| File I/O | `READ`, `WRITE`, `READV`, `WRITEV`, `READ_FIXED`, `WRITE_FIXED`, `FSYNC`, `FALLOCATE`, `FADVISE` |
| Network | `SEND`, `RECV`, `SENDMSG`, `RECVMSG`, `ACCEPT`, `CONNECT`, `SHUTDOWN`, `SOCKET`, `SEND_ZC`, `SENDMSG_ZC` |
| Polling | `POLL_ADD`, `POLL_REMOVE`, `POLL_UPDATE` |
| Filesystem | `OPENAT`, `OPENAT2`, `CLOSE`, `STATX`, `RENAMEAT`, `UNLINKAT`, `MKDIRAT`, `LINKAT`, `SYMLINKAT`, `GETDENTS` |
| Timers | `TIMEOUT`, `TIMEOUT_REMOVE`, `LINK_TIMEOUT` |
| Control | `ASYNC_CANCEL`, `FILES_UPDATE`, `PROVIDE_BUFFERS`, `MSG_RING` |
| Zero-copy / splice | `SPLICE`, `TEE`, `SEND_ZC` |
| Misc | `NOP`, `EPOLL_CTL`, `URING_CMD`, `FIXED_FD_INSTALL`, `WAITID`, `FUTEX_WAIT/WAKE` |

**`IORING_OP_URING_CMD` deserves special attention.** It is a generic passthrough: a driver
implements `->uring_cmd()` and gets an asynchronous, batched command channel for free. NVMe
uses it for passthrough commands, bypassing the block layer entirely and reaching the device
with a user-supplied command. It is effectively "async ioctl done right," and it is how
`io_uring` is spreading beyond I/O.

### T.5 — Chaining and ordering

Because requests are submitted as a batch, ordering must be expressible:

| Flag | Meaning |
|---|---|
| (none) | Fully independent, may complete in any order, executed concurrently |
| `IOSQE_IO_LINK` | The next SQE starts only after this one completes **successfully**; a failure cancels the rest of the chain |
| `IOSQE_IO_HARDLINK` | Same, but continues even on failure |
| `IOSQE_IO_DRAIN` | Wait for all *previously* submitted requests to complete first |
| `IOSQE_ASYNC` | Skip the non-blocking attempt; go straight to io-wq |
| `IOSQE_CQE_SKIP_SUCCESS` | Do not post a CQE on success — reduces CQ pressure for fire-and-forget |

Linking lets you express `open → read → close` as one submission with one syscall and, on a
cache hit, zero context switches. That composition is what turns `io_uring` from "faster
read" into "a way to move whole operation graphs into the kernel."

**The important limitation:** links are sequential, not a general DAG. Expressing "A and B,
then C" requires either two chains plus userspace joining, or `MSG_RING` between rings.

### T.6 — Registration: amortizing the per-call setup

A normal `read(fd, buf, len)` does work every single call that could be done once:

| Per-call cost | Removed by |
|---|---|
| `fdget()` — look up fd, take a reference, validate | **Registered files** (`IORING_REGISTER_FILES`): refer to a table index; the reference is held for the ring's lifetime |
| `get_user_pages()` — pin and translate the buffer; `dma_map` it | **Registered buffers** (`IORING_REGISTER_BUFFERS` + `READ_FIXED`/`WRITE_FIXED`): pin and map once |
| Buffer selection by userspace | **Provided buffers** (`PROVIDE_BUFFERS`, and ring-mapped buffers): give the kernel a pool; it picks one at completion time and tells you which |
| Ring creation per thread | Shared work queues (`IORING_SETUP_ATTACH_WQ`) |

**Provided buffers** solve a real networking problem: with `recv`, you must commit a buffer
per in-flight request, so 100k idle connections need 100k buffers. With provided buffers you
hand the kernel a pool of, say, 1024 and it consumes one only when data actually arrives.
Memory scales with *active* connections, not with *open* ones. That is a qualitative
improvement, not a micro-optimization.

Registered buffers matter more as you go faster: at 10M IOPS (Ch. 51, `system-design.md` §2),
`get_user_pages` per I/O is not affordable.

### T.7 — The costs, stated honestly

This is where a senior answer separates itself from enthusiasm.

**1. Security surface.** `io_uring` has produced a large number of CVEs — enough that Google
disabled it by default in Android and ChromeOS and restricted it on production servers. The
reasons are structural, not incidental:

- It performs operations **on behalf of** a task from a different context (io-wq workers),
  so every `current`-relative check (credentials, namespaces, `cgroup`, LSM labels, audit
  context) must be explicitly propagated. Any place that is forgotten is a bug.
- It reaches ~60 opcodes across a dozen subsystems, many of which had never been called from
  a kernel worker before.
- The ring memory is shared with userspace, so every read of a shared index is
  attacker-controlled and needs `READ_ONCE` plus validation — a classic double-fetch
  (TOCTOU) surface.
- Reference counting across async completion is genuinely hard, and UAFs have resulted.

Controls: `io_uring_disabled` sysctl (0=on, 1=restricted to a group, 2=off),
`io_uring_group`, seccomp filtering of `io_uring_setup`, and LSM hooks
(`security_uring_override_creds`, `security_uring_sqpoll`, `security_uring_cmd`).

**A crucial subtlety for anyone doing sandboxing:** seccomp filters syscalls, and
`io_uring` operations **are not syscalls**. A seccomp policy that blocks `openat` does not
block `IORING_OP_OPENAT`. Any sandbox must block `io_uring_setup` itself, or it has a hole.
This is one of the best "do you actually understand the security model" questions available.

**2. Complexity.** Correct use requires understanding barriers, completion ordering,
cancellation, buffer lifetime, and the fact that an SQE's contents must remain stable until
consumed. `liburing` exists because the raw interface is easy to misuse — and you should use
it.

**3. It is not always faster.** For low request rates the syscall was never the bottleneck,
and `io_uring` adds setup cost and memory. `SQPOLL` burns a core. Measure.

**4. Resource accounting.** Registered buffers are pinned memory against `RLIMIT_MEMLOCK`;
io-wq workers are tasks; rings are memory. In a multi-tenant environment these need bounding.

### T.8 — Where it sits in the design space

`io_uring` is one instance of a pattern that now appears repeatedly:

| Mechanism | Shared-memory ring between user and kernel |
|---|---|
| `io_uring` | General async syscalls |
| AF_XDP | Zero-copy packet rings (→ Ch. 74) |
| `perf` ring buffer | Event streaming (→ Ch. 06) |
| BPF ringbuf | Programmable event streaming (→ Ch. 75) |
| `vhost-user`/virtio | Guest↔host I/O (→ Ch. 104) |
| `relayfs` | High-rate kernel→user streaming |

The unifying principle: **when the crossing dominates, replace the crossing with shared
memory plus a producer/consumer protocol, and make the notification optional.** Every one of
these arrived independently at rings with acquire/release index publication, and being able
to state that generalization is a strong architectural answer.

It also reopens a design question from Ch. 00 §T.4. The classic argument for putting code in
the kernel was "the syscall boundary costs too much." `io_uring` weakens that argument
substantially, which is why some things that were heading in-kernel (userspace NVMe drivers,
kernel-bypass networking) now have viable in-kernel-with-fast-path answers.

---

## 1. Internals

### Source map

| Path | Contents |
|---|---|
| `io_uring/io_uring.c` | Core: setup, `io_uring_enter`, submission, completion, ring management |
| `io_uring/io_uring.h` | Internal helpers, CQE posting, `io_kiocb` handling |
| `io_uring/sqpoll.c` | The SQ polling kernel thread |
| `io_uring/io-wq.c` | The bounded/unbounded worker pool — the blocking backstop |
| `io_uring/rw.c` | Read/write ops, including the `-EAGAIN` retry machinery |
| `io_uring/net.c` | Socket operations, including zero-copy send |
| `io_uring/poll.c` | Internal poll arming — tier 2 of §T.3 |
| `io_uring/timeout.c` | Timeouts and linked timeouts |
| `io_uring/rsrc.c` | Registered files and buffers |
| `io_uring/kbuf.c` | Provided/ring-mapped buffers |
| `io_uring/uring_cmd.c` | `IORING_OP_URING_CMD` passthrough |
| `io_uring/cancel.c` | Cancellation |
| `io_uring/msg_ring.c` | Ring-to-ring messaging |
| `include/uapi/linux/io_uring.h` | **The ABI** — SQE/CQE layout, all opcodes and flags |
| `include/linux/io_uring_types.h` | `io_ring_ctx`, `io_kiocb` |
| `Documentation/` + `man io_uring(7)` | Plus the `liburing` man pages, which are better |

### The SQE and CQE

```c
struct io_uring_sqe {
	__u8	opcode;        /* IORING_OP_*                              */
	__u8	flags;         /* IOSQE_*                                  */
	__u16	ioprio;
	__s32	fd;            /* or an index if IOSQE_FIXED_FILE          */
	union { __u64 off; __u64 addr2; __u32 cmd_op; };
	union { __u64 addr; __u64 splice_off_in; };
	__u32	len;
	union { __kernel_rwf_t rw_flags; __u32 fsync_flags;
		__u32 msg_flags;    /* ... one per op family ... */ };
	__u64	user_data;     /* OPAQUE: echoed back in the CQE. This is
				* how you correlate. Put a pointer here. */
	union { __u16 buf_index; __u16 buf_group; };
	__u16	personality;   /* registered credentials to run as          */
	...
};

struct io_uring_cqe {
	__u64	user_data;     /* exactly what you put in the SQE           */
	__s32	res;           /* >= 0 result, or -errno                    */
	__u32	flags;         /* IORING_CQE_F_MORE, F_BUFFER, F_NOTIF...   */
	__u64	big_cqe[];     /* if IORING_SETUP_CQE32                     */
};
```

Two ABI-design observations worth making (→ Ch. 24 §T.4):

- `user_data` is **opaque to the kernel**, which is what lets userspace put a pointer or a
  tagged index there and correlate completions without any kernel-side bookkeeping. A
  deceptively important design choice.
- The heavy use of unions keeps the SQE at exactly 64 bytes — **one cacheline**. With a
  128-byte `SQE128` variant for `URING_CMD`. That is deliberate: submission is a streaming
  write, and one cacheline per entry is the difference between one and two cache misses per
  submission.

### Barrier discipline — the part people get wrong

```c
/* Userspace submission, conceptually (liburing does this for you): */
unsigned tail = *sq->ktail;                  /* our own value            */
unsigned index = tail & sq->ring_mask;
sqe = &sq->sqes[index];
fill_in(sqe);                                /* (1) write the SQE data   */
sq->array[index] = index;
smp_store_release(sq->ktail, tail + 1);      /* (2) publish the tail     */
/* (2) MUST be a release store so the kernel cannot observe the new tail
 * before the SQE contents. Identical to rcu_assign_pointer's job.      */

/* Kernel consumption: */
tail = smp_load_acquire(&ctx->rings->sq.tail);   /* acquire load        */
while (head != tail) { ...consume... }
```

And the mirror image on the completion side. If you were asked "explain publication
ordering" in Ch. 13, this is the same problem with a privilege boundary in the middle —
which adds the requirement that the kernel must treat every index it reads as hostile
(`READ_ONCE` + masking, never trusting it to be in range).

### io-wq: the blocking backstop

```c
/* Two pools per ring: */
enum { IO_WQ_ACCT_BOUND, IO_WQ_ACCT_UNBOUND, IO_WQ_ACCT_NR };
```

- **Bounded** workers serve operations with a predictable completion time (file I/O). The
  default limit is based on CPU count.
- **Unbounded** workers serve operations that may block indefinitely (network without poll).
  The default limit is `RLIMIT_NPROC`.

Both are tunable with `IORING_REGISTER_IOWQ_MAX_WORKERS`. Workers are created lazily and
exit when idle. They are visible as `iou-wrk-<pid>` in `ps`, and `iou-sqp-<pid>` for the
SQPOLL thread — knowing those names is a practical debugging fact.

The reason the bounded/unbounded split exists: without it, a few blocked network operations
would consume every worker and starve file I/O, a classic head-of-line blocking problem.

### Observability surface

```bash
# The ring's internal state -- extremely useful, often unknown:
ls /sys/kernel/debug/io_uring/            # per-ring dirs (CONFIG_IO_URING + debugfs)
cat /proc/<pid>/fdinfo/<uring_fd>
#   -> SqMask, SqHead, SqTail, CqHead, CqTail, SQEs, CQEs,
#      SqThread, SqThreadCpu, UserFiles, UserBufs, and the in-flight ops.
#   This is the first thing to read when a ring is "stuck".

# Workers and the SQPOLL thread:
ps -eLo pid,tid,comm | grep -E 'iou-wrk|iou-sqp'

# Tracepoints -- a complete set:
sudo ls /sys/kernel/debug/tracing/events/io_uring/
sudo trace-cmd record -e io_uring:io_uring_submit_req \
                      -e io_uring:io_uring_complete \
                      -e io_uring:io_uring_queue_async_work \
                      -e io_uring:io_uring_poll_arm -- ./app

# How often are we falling back to a worker? (the key health metric)
sudo bpftrace -e '
tracepoint:io_uring:io_uring_submit_req  { @submitted = count(); }
tracepoint:io_uring:io_uring_queue_async_work { @punted_to_wq = count(); }
tracepoint:io_uring:io_uring_poll_arm    { @polled = count(); }
interval:s:10 { exit(); }'

# System-wide policy:
sysctl kernel.io_uring_disabled kernel.io_uring_group
```

**The "punted to io-wq" ratio is the metric that matters.** A high ratio means you are
paying for a thread pool with extra steps — which usually means buffered I/O missing the
page cache, and the fix is `O_DIRECT` plus registered buffers, or accepting it.

---

## 2. Practice

### Lab 76.1 — Raw `io_uring`, no library

Write it once without `liburing` so the ring mechanics are concrete. Then never do it again.

```c
/* raw_uring.c — cat(1) using io_uring with no library.
 * Build: gcc -O2 -Wall -o raw_uring raw_uring.c
 * Run:   ./raw_uring /etc/hostname                                    */
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <sys/mman.h>
#include <sys/syscall.h>
#include <sys/uio.h>
#include <linux/io_uring.h>

#define QD 8

struct ring {
	int fd;
	unsigned *sq_head, *sq_tail, *sq_mask, *sq_array;
	struct io_uring_sqe *sqes;
	unsigned *cq_head, *cq_tail, *cq_mask;
	struct io_uring_cqe *cqes;
	void *sq_ptr, *cq_ptr;
	size_t sq_sz, cq_sz;
};

static int io_uring_setup_(unsigned entries, struct io_uring_params *p)
{ return syscall(__NR_io_uring_setup, entries, p); }

static int io_uring_enter_(int fd, unsigned to_submit,
			   unsigned min_complete, unsigned flags)
{ return syscall(__NR_io_uring_enter, fd, to_submit, min_complete, flags,
		 NULL, 0); }

static int setup_ring(struct ring *r)
{
	struct io_uring_params p;
	void *sq, *cq;
	size_t sring_sz, cring_sz;

	memset(&p, 0, sizeof(p));
	r->fd = io_uring_setup_(QD, &p);
	if (r->fd < 0) { perror("io_uring_setup"); return -1; }

	sring_sz = p.sq_off.array + p.sq_entries * sizeof(unsigned);
	cring_sz = p.cq_off.cqes  + p.cq_entries * sizeof(struct io_uring_cqe);

	/* Since 5.4, SINGLE_MMAP lets both rings share one mapping. */
	if (p.features & IORING_FEAT_SINGLE_MMAP) {
		if (cring_sz > sring_sz) sring_sz = cring_sz;
		cring_sz = sring_sz;
	}

	sq = mmap(NULL, sring_sz, PROT_READ | PROT_WRITE,
		  MAP_SHARED | MAP_POPULATE, r->fd, IORING_OFF_SQ_RING);
	if (sq == MAP_FAILED) { perror("mmap sq"); return -1; }

	cq = (p.features & IORING_FEAT_SINGLE_MMAP) ? sq :
	     mmap(NULL, cring_sz, PROT_READ | PROT_WRITE,
		  MAP_SHARED | MAP_POPULATE, r->fd, IORING_OFF_CQ_RING);
	if (cq == MAP_FAILED) { perror("mmap cq"); return -1; }

	r->sq_ptr = sq; r->cq_ptr = cq;
	r->sq_sz = sring_sz; r->cq_sz = cring_sz;

	/* The kernel tells us the offset of every field. Never hardcode. */
	r->sq_head  = sq + p.sq_off.head;
	r->sq_tail  = sq + p.sq_off.tail;
	r->sq_mask  = sq + p.sq_off.ring_mask;
	r->sq_array = sq + p.sq_off.array;
	r->cq_head  = cq + p.cq_off.head;
	r->cq_tail  = cq + p.cq_off.tail;
	r->cq_mask  = cq + p.cq_off.ring_mask;
	r->cqes     = cq + p.cq_off.cqes;

	r->sqes = mmap(NULL, p.sq_entries * sizeof(struct io_uring_sqe),
		       PROT_READ | PROT_WRITE, MAP_SHARED | MAP_POPULATE,
		       r->fd, IORING_OFF_SQES);
	if (r->sqes == MAP_FAILED) { perror("mmap sqes"); return -1; }
	return 0;
}

int main(int argc, char **argv)
{
	struct ring r;
	struct io_uring_sqe *sqe;
	struct io_uring_cqe *cqe;
	struct iovec iov;
	unsigned tail, index, head;
	char *buf;
	int fd, ret;
	off_t size;

	if (argc < 2) { fprintf(stderr, "usage: %s <file>\n", argv[0]); return 1; }
	if (setup_ring(&r)) return 1;

	fd = open(argv[1], O_RDONLY);
	if (fd < 0) { perror("open"); return 1; }
	size = lseek(fd, 0, SEEK_END);
	lseek(fd, 0, SEEK_SET);
	if (size <= 0) size = 4096;

	buf = malloc(size);
	iov.iov_base = buf;
	iov.iov_len  = size;

	/* ---- SUBMIT ---- */
	tail  = *r.sq_tail;              /* we are the only writer of tail  */
	index = tail & *r.sq_mask;
	sqe   = &r.sqes[index];
	memset(sqe, 0, sizeof(*sqe));
	sqe->opcode    = IORING_OP_READV;
	sqe->fd        = fd;
	sqe->addr      = (unsigned long)&iov;
	sqe->len       = 1;              /* one iovec                       */
	sqe->off       = 0;
	sqe->user_data = 0xdeadbeef;     /* echoed back; our correlation tag */

	r.sq_array[index] = index;

	/* RELEASE store: everything above must be visible to the kernel
	 * before it can observe the new tail. __atomic_store_n with
	 * __ATOMIC_RELEASE is the portable spelling of smp_store_release. */
	__atomic_store_n(r.sq_tail, tail + 1, __ATOMIC_RELEASE);

	/* One syscall for the whole batch. Ask to wait for 1 completion. */
	ret = io_uring_enter_(r.fd, 1, 1, IORING_ENTER_GETEVENTS);
	if (ret < 0) { perror("io_uring_enter"); return 1; }

	/* ---- REAP ---- */
	head = *r.cq_head;
	/* ACQUIRE load: do not read the CQE until we have seen the tail. */
	if (head == __atomic_load_n(r.cq_tail, __ATOMIC_ACQUIRE)) {
		fprintf(stderr, "no completion?\n");
		return 1;
	}
	cqe = &r.cqes[head & *r.cq_mask];

	if (cqe->res < 0) {
		fprintf(stderr, "read failed: %s\n", strerror(-cqe->res));
	} else {
		fprintf(stderr, "[user_data=0x%llx res=%d]\n",
			(unsigned long long)cqe->user_data, cqe->res);
		fwrite(buf, 1, cqe->res, stdout);
	}

	/* Tell the kernel we consumed it. */
	__atomic_store_n(r.cq_head, head + 1, __ATOMIC_RELEASE);

	free(buf);
	close(fd);
	close(r.fd);
	return 0;
}
```

Every detail matters: the offsets come from the kernel (never hardcode them), the release
store publishes the SQE, the acquire load orders the CQE read, and `user_data` is how you
know which request completed.

### Lab 76.2 — The same thing with `liburing`

```c
/* liburing_cat.c — Build: gcc -O2 -o liburing_cat liburing_cat.c -luring */
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <liburing.h>

int main(int argc, char **argv)
{
	struct io_uring ring;
	struct io_uring_sqe *sqe;
	struct io_uring_cqe *cqe;
	char buf[65536];
	int fd, ret;
	off_t off = 0;

	if (argc < 2) { fprintf(stderr, "usage: %s <file>\n", argv[0]); return 1; }

	if ((ret = io_uring_queue_init(8, &ring, 0)) < 0) {
		fprintf(stderr, "queue_init: %s\n", strerror(-ret)); return 1;
	}
	if ((fd = open(argv[1], O_RDONLY)) < 0) { perror("open"); return 1; }

	for (;;) {
		sqe = io_uring_get_sqe(&ring);
		io_uring_prep_read(sqe, fd, buf, sizeof(buf), off);
		io_uring_sqe_set_data64(sqe, off);

		io_uring_submit(&ring);

		if ((ret = io_uring_wait_cqe(&ring, &cqe)) < 0) {
			fprintf(stderr, "wait_cqe: %s\n", strerror(-ret)); break;
		}
		if (cqe->res < 0) {
			fprintf(stderr, "read: %s\n", strerror(-cqe->res));
			io_uring_cqe_seen(&ring, cqe); break;
		}
		if (cqe->res == 0) { io_uring_cqe_seen(&ring, cqe); break; }

		fwrite(buf, 1, cqe->res, stdout);
		off += cqe->res;
		io_uring_cqe_seen(&ring, cqe);
	}

	close(fd);
	io_uring_queue_exit(&ring);
	return 0;
}
```

150 lines became 40, and all the barrier subtleties are gone. **Use `liburing`.** The raw lab
exists so you know what it is doing, not so you will repeat it.

### Lab 76.3 — Benchmark against the alternatives

```c
/* bench.c — read a file N times with four mechanisms. Compare.
 * Build: gcc -O2 -o bench bench.c -luring -lpthread
 * Run:   ./bench /path/to/1GB_file                                    */
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <time.h>
#include <liburing.h>

#define BS     (4 * 1024)
#define DEPTH  64
#define NREQ   200000

static double now(void)
{
	struct timespec t;
	clock_gettime(CLOCK_MONOTONIC, &t);
	return t.tv_sec + t.tv_nsec / 1e9;
}

/* 1. Plain pread: one syscall per request. The baseline. */
static double bench_pread(int fd, off_t fsize)
{
	char *buf = aligned_alloc(4096, BS);
	double t0 = now();
	for (int i = 0; i < NREQ; i++) {
		off_t off = ((off_t)random() % (fsize / BS)) * BS;
		if (pread(fd, buf, BS, off) < 0) { perror("pread"); break; }
	}
	double t = now() - t0;
	free(buf);
	return t;
}

/* 2. io_uring, batched submission. One syscall per DEPTH requests. */
static double bench_uring(int fd, off_t fsize, unsigned flags, int fixed)
{
	struct io_uring ring;
	struct io_uring_params p;
	char *bufs[DEPTH];
	struct iovec iov[DEPTH];
	int inflight = 0, done = 0, ret;
	double t0;

	memset(&p, 0, sizeof(p));
	p.flags = flags;
	if ((ret = io_uring_queue_init_params(DEPTH, &ring, &p)) < 0) {
		fprintf(stderr, "init: %s\n", strerror(-ret));
		return -1;
	}

	for (int i = 0; i < DEPTH; i++) {
		bufs[i] = aligned_alloc(4096, BS);
		iov[i].iov_base = bufs[i];
		iov[i].iov_len  = BS;
	}
	if (fixed) {
		if (io_uring_register_buffers(&ring, iov, DEPTH) < 0)
			fprintf(stderr, "register_buffers failed (RLIMIT_MEMLOCK?)\n");
		if (io_uring_register_files(&ring, &fd, 1) < 0)
			fprintf(stderr, "register_files failed\n");
	}

	t0 = now();
	while (done < NREQ) {
		struct io_uring_cqe *cqe;
		unsigned head, n = 0;

		/* Fill the pipeline. */
		while (inflight < DEPTH && done + inflight < NREQ) {
			struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
			if (!sqe) break;
			off_t off = ((off_t)random() % (fsize / BS)) * BS;
			int slot = inflight;
			if (fixed) {
				io_uring_prep_read_fixed(sqe, 0, bufs[slot],
							 BS, off, slot);
				sqe->flags |= IOSQE_FIXED_FILE;
			} else {
				io_uring_prep_read(sqe, fd, bufs[slot], BS, off);
			}
			inflight++;
		}
		io_uring_submit_and_wait(&ring, 1);

		io_uring_for_each_cqe(&ring, head, cqe) {
			if (cqe->res < 0)
				fprintf(stderr, "cqe: %s\n", strerror(-cqe->res));
			n++;
		}
		io_uring_cq_advance(&ring, n);
		inflight -= n;
		done += n;
	}
	double t = now() - t0;

	for (int i = 0; i < DEPTH; i++) free(bufs[i]);
	io_uring_queue_exit(&ring);
	return t;
}

int main(int argc, char **argv)
{
	int fd, fdd;
	off_t fsize;
	double t;

	if (argc < 2) { fprintf(stderr, "usage: %s <big file>\n", argv[0]); return 1; }

	fd = open(argv[1], O_RDONLY);
	if (fd < 0) { perror("open"); return 1; }
	fsize = lseek(fd, 0, SEEK_END);
	fdd = open(argv[1], O_RDONLY | O_DIRECT);
	if (fdd < 0) { perror("open O_DIRECT"); fdd = fd; }

	printf("file %s, %.1f GiB, %d requests of %d bytes, depth %d\n\n",
	       argv[1], fsize / 1073741824.0, NREQ, BS, DEPTH);

	t = bench_pread(fdd, fsize);
	printf("%-36s %7.3f s  %9.0f IOPS\n", "pread (1 syscall/req)", t, NREQ / t);

	t = bench_uring(fdd, fsize, 0, 0);
	printf("%-36s %7.3f s  %9.0f IOPS\n", "io_uring batched", t, NREQ / t);

	t = bench_uring(fdd, fsize, 0, 1);
	printf("%-36s %7.3f s  %9.0f IOPS\n", "io_uring + registered buf/file",
	       t, NREQ / t);

	t = bench_uring(fdd, fsize, IORING_SETUP_SQPOLL, 1);
	printf("%-36s %7.3f s  %9.0f IOPS\n", "io_uring SQPOLL + registered",
	       t, NREQ / t);

	/* IOPOLL requires O_DIRECT and a polling-capable device. */
	t = bench_uring(fdd, fsize, IORING_SETUP_IOPOLL, 1);
	printf("%-36s %7.3f s  %9.0f IOPS\n", "io_uring IOPOLL + registered",
	       t, NREQ / t);

	return 0;
}
```

```bash
# Make a test file and drop caches between runs.
fallocate -l 4G /tmp/testfile
sync; echo 3 | sudo tee /proc/sys/vm/drop_caches
ulimit -l unlimited          # registered buffers need MEMLOCK headroom
./bench /tmp/testfile

# Count the syscalls each actually makes -- the mechanism, not just the result:
strace -c -f ./bench /tmp/testfile 2>&1 | tail -20
```

Attribute each improvement to its mechanism: batching removes syscalls, registration removes
`fdget` and `get_user_pages`, SQPOLL removes the last syscall, IOPOLL removes the interrupt.
**Four independent wins, and you should be able to say which one gave you what.**

### Lab 76.4 — A linked chain: open → read → close

```c
/* chain.c — three operations, one submission, one syscall.
 * Build: gcc -O2 -o chain chain.c -luring                             */
#include <stdio.h>
#include <string.h>
#include <fcntl.h>
#include <liburing.h>

int main(int argc, char **argv)
{
	struct io_uring ring;
	struct io_uring_sqe *sqe;
	struct io_uring_cqe *cqe;
	static char buf[4096];
	int ret;

	if (argc < 2) { fprintf(stderr, "usage: %s <file>\n", argv[0]); return 1; }
	io_uring_queue_init(8, &ring, 0);

	/* Register a 1-slot file table; the open will install into slot 0
	 * and the read will refer to it -- WITHOUT the fd ever being
	 * visible to userspace. No fd table churn at all.                 */
	int fds[1] = { -1 };
	io_uring_register_files(&ring, fds, 1);

	/* 1. openat -> direct into fixed slot 0 */
	sqe = io_uring_get_sqe(&ring);
	io_uring_prep_openat_direct(sqe, AT_FDCWD, argv[1], O_RDONLY, 0, 0);
	sqe->flags |= IOSQE_IO_LINK;      /* next runs only if this succeeds */
	io_uring_sqe_set_data64(sqe, 1);

	/* 2. read from fixed slot 0 */
	sqe = io_uring_get_sqe(&ring);
	io_uring_prep_read(sqe, 0, buf, sizeof(buf), 0);
	sqe->flags |= IOSQE_FIXED_FILE | IOSQE_IO_LINK;
	io_uring_sqe_set_data64(sqe, 2);

	/* 3. close the fixed slot */
	sqe = io_uring_get_sqe(&ring);
	io_uring_prep_close_direct(sqe, 0);
	io_uring_sqe_set_data64(sqe, 3);

	/* ONE syscall for all three. */
	ret = io_uring_submit(&ring);
	printf("submitted %d SQEs in 1 syscall\n", ret);

	for (int i = 0; i < 3; i++) {
		io_uring_wait_cqe(&ring, &cqe);
		printf("  op %llu -> res %d%s\n",
		       (unsigned long long)io_uring_cqe_get_data64(cqe),
		       cqe->res,
		       cqe->res == -ECANCELED ? "  (chain cancelled)" : "");
		if (io_uring_cqe_get_data64(cqe) == 2 && cqe->res > 0)
			fwrite(buf, 1, cqe->res, stdout);
		io_uring_cqe_seen(&ring, cqe);
	}

	io_uring_queue_exit(&ring);
	return 0;
}
```

Now run it on a nonexistent file and observe `-ENOENT` on op 1 and `-ECANCELED` on ops 2 and
3 — the chain semantics working. Then count syscalls with `strace -c` and compare with the
`open`/`read`/`close` equivalent: three syscalls become one, and on a page-cache hit, zero
context switches.

### Lab 76.5 — An echo server with provided buffers

```c
/* echo.c — io_uring echo server using multishot accept and a ring-mapped
 * buffer pool, so memory scales with ACTIVE connections, not open ones.
 * Build: gcc -O2 -o echo echo.c -luring
 * Run:   ./echo 8080   then:  nc localhost 8080                      */
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <liburing.h>

#define QD        256
#define NBUFS     64
#define BUFSZ     2048
#define BGID      1

enum { OP_ACCEPT = 1, OP_RECV, OP_SEND };
#define MKDATA(op, fd)  (((__u64)(op) << 32) | (unsigned)(fd))
#define DATA_OP(d)      ((unsigned)((d) >> 32))
#define DATA_FD(d)      ((int)((d) & 0xffffffff))

static struct io_uring ring;
static struct io_uring_buf_ring *br;
static char *bufbase;

static void add_buf(int idx)
{
	io_uring_buf_ring_add(br, bufbase + idx * BUFSZ, BUFSZ, idx,
			      io_uring_buf_ring_mask(NBUFS), 0);
}

static void post_accept(int lfd)
{
	struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
	/* MULTISHOT: one SQE produces a CQE per incoming connection,
	 * forever. No re-arming. */
	io_uring_prep_multishot_accept(sqe, lfd, NULL, NULL, 0);
	io_uring_sqe_set_data64(sqe, MKDATA(OP_ACCEPT, lfd));
}

static void post_recv(int fd)
{
	struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
	io_uring_prep_recv_multishot(sqe, fd, NULL, 0, 0);
	/* Do not name a buffer: the kernel picks one from group BGID when
	 * data actually arrives, and tells us which in cqe->flags.       */
	sqe->flags     |= IOSQE_BUFFER_SELECT;
	sqe->buf_group  = BGID;
	io_uring_sqe_set_data64(sqe, MKDATA(OP_RECV, fd));
}

int main(int argc, char **argv)
{
	int port = (argc > 1) ? atoi(argv[1]) : 8080;
	struct sockaddr_in addr = {
		.sin_family = AF_INET, .sin_port = htons(port),
		.sin_addr.s_addr = htonl(INADDR_ANY),
	};
	int lfd, one = 1, ret;

	io_uring_queue_init(QD, &ring, 0);

	br = io_uring_setup_buf_ring(&ring, NBUFS, BGID, 0, &ret);
	if (!br) { fprintf(stderr, "buf_ring: %s\n", strerror(-ret)); return 1; }
	bufbase = malloc((size_t)NBUFS * BUFSZ);
	for (int i = 0; i < NBUFS; i++) add_buf(i);
	io_uring_buf_ring_advance(br, NBUFS);

	lfd = socket(AF_INET, SOCK_STREAM, 0);
	setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &one, sizeof(one));
	if (bind(lfd, (struct sockaddr *)&addr, sizeof(addr))) { perror("bind"); return 1; }
	listen(lfd, 512);
	printf("echo server on port %d\n", port);

	post_accept(lfd);
	io_uring_submit(&ring);

	for (;;) {
		struct io_uring_cqe *cqe;
		unsigned head, count = 0;

		if ((ret = io_uring_wait_cqe(&ring, &cqe)) < 0) {
			fprintf(stderr, "wait: %s\n", strerror(-ret)); break;
		}
		io_uring_for_each_cqe(&ring, head, cqe) {
			__u64 d  = io_uring_cqe_get_data64(cqe);
			unsigned op = DATA_OP(d);
			int fd = DATA_FD(d);
			count++;

			if (op == OP_ACCEPT) {
				if (cqe->res >= 0) post_recv(cqe->res);
				if (!(cqe->flags & IORING_CQE_F_MORE))
					post_accept(lfd);     /* re-arm */
			} else if (op == OP_RECV) {
				if (cqe->res <= 0) { close(fd); continue; }
				if (!(cqe->flags & IORING_CQE_F_BUFFER)) continue;
				int bid = cqe->flags >> IORING_CQE_BUFFER_SHIFT;

				struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
				io_uring_prep_send(sqe, fd, bufbase + bid * BUFSZ,
						   cqe->res, 0);
				io_uring_sqe_set_data64(sqe, MKDATA(OP_SEND, fd));
				/* NB: the buffer is recycled only after the send
				 * completes -- ownership is the subtle part. */
				sqe->buf_index = bid;

				if (!(cqe->flags & IORING_CQE_F_MORE))
					post_recv(fd);
			} else { /* OP_SEND */
				add_buf(cqe->buf_index);
				io_uring_buf_ring_advance(br, 1);
			}
		}
		io_uring_cq_advance(&ring, count);
		io_uring_submit(&ring);
	}
	return 0;
}
```

Three modern features are on display: **multishot accept** (one SQE, unbounded CQEs),
**multishot recv**, and **ring-mapped provided buffers**. Benchmark against an `epoll` echo
server with `wrk` or `rust_echo_bench` at 10k connections, and compare both throughput and
RSS. The RSS difference is the provided-buffer point from §T.6.

### Lab 76.6 — Observe the io-wq fallback

```bash
# Buffered reads that MISS the page cache must be punted to a worker.
# Direct reads are handled inline. Watch the difference.

sudo bpftrace -e '
tracepoint:io_uring:io_uring_submit_req       { @submit = count(); }
tracepoint:io_uring:io_uring_queue_async_work { @punt   = count(); }
tracepoint:io_uring:io_uring_poll_arm         { @poll   = count(); }
tracepoint:io_uring:io_uring_complete         { @done   = count(); }
interval:s:5 { print(@submit); print(@punt); print(@poll); print(@done);
               clear(@submit); clear(@punt); clear(@poll); clear(@done); }' &

# (a) Cold buffered reads -> high punt ratio
sync; echo 3 | sudo tee /proc/sys/vm/drop_caches
./bench /tmp/testfile          # with the O_DIRECT open replaced by buffered

# (b) Warm page cache -> near-zero punt ratio (completed inline)
./bench /tmp/testfile

# (c) O_DIRECT -> handled by the block layer async, no worker
# Watch the worker threads appear and disappear:
watch -n1 'ps -eLo comm | grep -c iou-wrk'
```

**The takeaway:** `io_uring` is not magic for buffered I/O that misses — it is a thread pool
with a better interface. The wins that are structural (no syscall, no `fdget`, no
`get_user_pages`, no wakeup) apply regardless; the win that is *not* structural is
"asynchronous," and that one depends entirely on whether tier 1 or tier 2 of §T.3 handles
your request.

### Lab 76.7 — The security boundary

```c
/* sandbox_hole.c — demonstrate that seccomp does NOT filter io_uring ops.
 * Build: gcc -O2 -o sandbox_hole sandbox_hole.c -luring
 *
 * Install a seccomp filter denying openat, then open a file anyway
 * via IORING_OP_OPENAT. This is a real and frequently-missed hole.   */
#define _GNU_SOURCE
#include <stdio.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <linux/audit.h>
#include <linux/filter.h>
#include <linux/seccomp.h>
#include <sys/prctl.h>
#include <sys/syscall.h>
#include <liburing.h>

static void deny_openat(void)
{
	struct sock_filter f[] = {
		BPF_STMT(BPF_LD | BPF_W | BPF_ABS,
			 offsetof(struct seccomp_data, arch)),
		BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, AUDIT_ARCH_X86_64, 1, 0),
		BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL_PROCESS),
		BPF_STMT(BPF_LD | BPF_W | BPF_ABS,
			 offsetof(struct seccomp_data, nr)),
		BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_openat, 0, 1),
		BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ERRNO | EPERM),
		BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
	};
	struct sock_fprog p = { .len = sizeof(f) / sizeof(f[0]), .filter = f };

	prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);
	if (syscall(SYS_seccomp, SECCOMP_SET_MODE_FILTER, 0, &p))
		perror("seccomp");
}

int main(void)
{
	struct io_uring ring;
	struct io_uring_sqe *sqe;
	struct io_uring_cqe *cqe;

	/* Set the ring up BEFORE the filter -- as a real sandboxed process
	 * would if io_uring_setup were allowed.                          */
	if (io_uring_queue_init(4, &ring, 0) < 0) {
		perror("queue_init"); return 1;
	}

	deny_openat();

	printf("direct openat():     %s\n",
	       open("/etc/passwd", O_RDONLY) < 0 ? strerror(errno) : "SUCCEEDED");

	sqe = io_uring_get_sqe(&ring);
	io_uring_prep_openat(sqe, AT_FDCWD, "/etc/passwd", O_RDONLY, 0);
	io_uring_submit(&ring);
	io_uring_wait_cqe(&ring, &cqe);

	printf("IORING_OP_OPENAT:    %s\n",
	       cqe->res < 0 ? strerror(-cqe->res) : "SUCCEEDED -- seccomp bypassed");

	io_uring_cqe_seen(&ring, cqe);
	io_uring_queue_exit(&ring);
	return 0;
}
```

Then verify the correct mitigations:

```bash
# 1. Block ring creation entirely (the only complete fix for a sandbox):
#    add __NR_io_uring_setup, __NR_io_uring_enter, __NR_io_uring_register
#    to the seccomp denylist.
# 2. System-wide policy:
sysctl kernel.io_uring_disabled          # 0=on 1=group-restricted 2=off
sudo sysctl -w kernel.io_uring_disabled=2
./sandbox_hole                            # io_uring_queue_init now fails
sudo sysctl -w kernel.io_uring_disabled=0
# 3. Or restrict to a group:
sudo sysctl -w kernel.io_uring_group=$(getent group uring | cut -d: -f3)
```

**This lab is the most interview-relevant one in the chapter.** "How does `io_uring` interact
with seccomp?" is a question that cleanly separates people who have thought about the
security model from people who have only read the performance numbers.

---

## 3. Mastery drills

1. Implement a complete `cp(1)` with `io_uring`: queue depth management, back-pressure when
   the SQ is full, correct short-read/short-write handling, and `O_DIRECT` with alignment.
   Benchmark against `cp`, `dd`, and `copy_file_range` across three storage types.

2. Measure the four optimizations independently — batching, registered files, registered
   buffers, SQPOLL — and attribute a precise ns/op saving to each. Explain why the savings
   are not additive.

3. Build an HTTP server on `io_uring` with multishot accept, multishot recv, ring-mapped
   buffers, and `SEND_ZC`. Benchmark against an `epoll` server at 100, 10k, and 100k
   connections, measuring throughput, p99 latency, and RSS. Explain each curve's shape.

4. Instrument the io-wq punt ratio for a realistic workload. Find the conditions under which
   it exceeds 50%, and propose and test three changes that reduce it.

5. Read `io_uring/rw.c` and trace the `-EAGAIN` retry path in the source. Then write a
   program that deliberately triggers each of the three tiers in §T.3, and verify with
   tracepoints that each took the path you predicted.

6. Implement request cancellation correctly: submit a long-running operation, cancel it with
   `IORING_OP_ASYNC_CANCEL`, and handle every possible outcome (`-ENOENT`, `-EALREADY`,
   success, and the race where it completes first). Document the state machine.

7. Study `IORING_OP_URING_CMD` and NVMe passthrough. Write a program that issues an NVMe
   Identify command through `io_uring` and compare its latency with the equivalent ioctl.
   Explain what layers you bypassed.

8. Write an `io_uring` backend for an existing event-loop library (libev, libevent, or your
   own). Document every place the readiness model and the completion model do not map onto
   each other, and how you resolved it.

9. Determine empirically the queue depth at which `io_uring` stops improving on your
   hardware, for both NVMe and network. Explain the limit in each case (device queue depth,
   CPU, or ring size).

10. Audit `io_uring`'s credential handling: read `io_uring/io_uring.c` for `override_creds`
    and `personality`, and explain how a request submitted by task A and executed by an io-wq
    worker gets A's credentials, cgroup, and LSM label. Identify what would break if one were
    missed.

11. Compare `io_uring`'s ring design against AF_XDP's, the perf ring buffer's, and BPF's
    ringbuf. Produce a table of: SPSC vs MPSC, barrier discipline, wakeup mechanism, and
    buffer ownership. Explain why they differ where they differ.

12. Build the threat model for enabling `io_uring` on a multi-tenant host. Enumerate the
    attack surface, the available controls, the performance cost of each control, and write
    the recommendation with the conditions that would reverse it.

13. Take a real application that uses a thread pool for I/O (a database, a web server, a
    proxy) and port its hot path to `io_uring`. Measure the change in throughput, p99
    latency, CPU utilization, and thread count. Then write up what the port cost in
    complexity — this is the number that decides real adoption.

---

## 4. Further reading

**Kernel documentation and specs**
- `include/uapi/linux/io_uring.h` — the ABI; read it end to end once
- Axboe, "Efficient IO with io_uring" — the original design document, the single best
  starting point
- `man io_uring(7)`, `io_uring_setup(2)`, `io_uring_enter(2)`, `io_uring_register(2)`
- The `liburing` man pages (`io_uring_prep_*`, `io_uring_submit`, ...) — better than the
  kernel's
- `Documentation/filesystems/` for the interaction with buffered I/O and `IOCB_NOWAIT`

**Papers and talks**
- Didona et al., "Understanding Modern Storage APIs: A Systematic Study of libaio, SPDK,
  and io_uring," *SYSTOR*, 2022 — the careful comparative measurement
- Ren et al., "From Dynamic Loading to Extensible Transformation: An Infrastructure for
  Dynamic Library Transformation" — less directly relevant, but see the io_uring
  security literature generally
- Axboe's Kernel Recipes and SNIA talks, 2019–2024 — the design rationale year by year
- Corbet's LWN coverage of each major addition (multishot, provided buffers, zero-copy send,
  `URING_CMD`, `FUTEX` ops)

**LWN**
- "Ringing in a new asynchronous I/O API" (Jan 2019) — the introduction
- "The rapid growth of io_uring" (Jan 2020) — the feature explosion
- "io_uring and security" and "Sandboxing and io_uring" — the seccomp interaction
- "Zero-copy network transmission with io_uring"
- "io_uring and the shifting boundary of the kernel" — the architectural argument

**Books and code**
- `liburing`'s `examples/` and `test/` directories — the best real code to read;
  `test/` in particular documents every edge case by exercising it
- Shuveb Hussain, *Lord of the io_uring* — a free online guide, good for the practical side
- The `fio` `io_uring` engine (`engines/io_uring.c`) — production-quality usage with every
  option exercised
- `rustix`/`tokio-uring`/`glommio` — read `glommio` if you want to see a thread-per-core
  runtime built entirely on `io_uring`

**Tools**
- `fio` with `ioengine=io_uring` and `io_uring_cmd` — the reference benchmark
- `liburing`'s `t/io_uring` micro-benchmark — what Axboe uses to report IOPS records
- `strace -c` — to verify syscall counts actually dropped
- `/proc/<pid>/fdinfo/<fd>` and the `io_uring` tracepoints — the debugging surface

---

## Part 4 completion checkpoint

Networking and the modern I/O interface are done. Confirm you can:

- [ ] Explain `sk_buff` headroom/tailroom and why the layered stack is O(1) per layer
- [ ] Trace a packet from NIC interrupt through NAPI, the protocol stack, to a socket
- [ ] Explain why NAPI exists (receive livelock) and what its budget is for
- [ ] Place netfilter, tc, XDP, and eBPF on one pipeline diagram with their relative costs
- [ ] Explain what the BPF verifier proves and what it cannot
- [ ] Draw the `io_uring` SQ/CQ rings and state the barrier on each index
- [ ] Explain the three-tier submission path and why Linux AIO lacked tier 3
- [ ] Name four independent optimizations `io_uring` offers and what each removes
- [ ] Explain why seccomp does not filter `io_uring` operations, and what does
- [ ] Generalize: name five other kernel interfaces that use the same shared-ring pattern

→ Next: [../part5-rust/77-rust-foundations.md](../part5-rust/77-rust-foundations.md)

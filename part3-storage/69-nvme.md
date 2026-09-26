# Chapter 69 — NVMe: PCIe, NVMe-oF, zoned namespaces, and passthrough

> **Goal:** Understand the first storage protocol designed for flash rather than adapted to it. Understand why SCSI-over-AHCI was the bottleneck and what NVMe removed, the submission/completion queue pair as the fundamental abstraction and why it maps so well onto blk-mq, doorbells and phase tags as a lockless host–device protocol, namespaces as the new LUN, the admin/IO command split, interrupt coalescing and polling, NVMe over Fabrics as the same command set over RDMA/TCP/FC, multipath done in the protocol rather than bolted on, ZNS and the end of the FTL, and the passthrough interface that lets userspace drive the device directly. By the end you can read `drivers/nvme/`, interpret `nvme-cli` output fluently, and reason about where a microsecond goes.

---

## Theory & First Principles

### T.0 — Start here: what one command costs

AHCI — the SATA host interface, designed in 2004 for spinning disks — issues one command like
this:

```
  1. take the port lock             (ONE port, ONE queue: all CPUs contend)
  2. build a command in the slot list (32 slots total, for the whole device)
  3. write an MMIO register         <- an UNCACHED write across PCIe, ~1 us
  4. the device DMAs the command and executes it
  5. ONE interrupt line, serviced by ONE CPU
  6. read MMIO status registers to learn what completed  <- ~1 us EACH
```

**Roughly 6 µs of CPU and two-plus uncached MMIO reads per command, behind a single lock and a
single queue.** Against a 10 ms seek that is 0.06% overhead — free. Against an 80 µs NAND
read it is **7%**, and the single queue caps you near 200K IOPS however many cores you have
(Ch. 63 §T.0).

**NVMe's answer is not "the same thing, faster." It is a different shape:**

```
   CPU0             CPU1             CPU2             CPU3
    |                |                |                |
  +-SQ0--CQ0-+    +-SQ1--CQ1-+    +-SQ2--CQ2-+    +-SQ3--CQ3-+
  | submission|    |          |    |          |    |          |
  | + completion   |   ...    |    |   ...    |    |   ...    |
  | queue PAIR,    |          |    |          |    |          |
  | in HOST DRAM   |          |    |          |    |          |
  +-----+-----+    +----+-----+    +----+-----+    +----+-----+
        |               |               |               |
        +---------------+-------+-------+---------------+
                                v
                        NVMe controller

   up to 65,535 queue pairs x 65,536 commands deep
   each queue has its OWN MSI-X vector -> the interrupt lands on the owning CPU
```

**Five decisions, each undoing one line of the AHCI cost list:**

| Decision | What it kills |
|---|---|
| **Queues live in host DRAM**, not device registers | the uncached MMIO reads — the driver polls a *cached* memory location |
| **One queue pair per CPU** | the port lock; no shared state on the fast path (Ch. 16 §T.0) |
| **Per-queue MSI-X vector** | cross-CPU interrupts and IPIs; completion runs cache-warm on the submitting CPU (Ch. 38) |
| **One 32-bit doorbell write** to submit | the multi-register handshake — and **one doorbell can announce many commands** |
| **A fixed 64-byte command format** | parsing and translation; no SCSI mid-layer at all |

**Result: about 2 µs and one uncached write per command, scaling linearly with cores.**

**The doorbell deserves its own paragraph, because it is the reusable idea.** You place N
commands into your queue in ordinary cacheable memory — costing nothing but stores — and then
do *one* expensive uncached write to say "the tail moved." **Amortize the expensive
notification over a batch of cheap work.** That is precisely `io_uring` (Ch. 76), virtio
(Ch. 104), NAPI (Ch. 46), and `iomap` (Ch. 55 §T.0). Once you see the pattern you will find it
in nearly every high-performance interface designed since 2010.

**Two things NVMe deliberately gave up, and one it had to get back:**

- It gave up **transport independence** — then rebuilt it as **NVMe-oF** (over RDMA, TCP,
  Fibre Channel), rediscovering exactly the property that made SCSI durable (Ch. 68 §T.0).
- It gave up **rich error recovery**; the escalation ladder is far shallower and the answer to
  a misbehaving controller is largely "reset it."
- It kept and extended **the device as a managed resource**: namespaces, multiple controllers,
  admin-versus-I/O queue separation, and ZNS.

**ZNS is the honest closing thought for Part 3.** Ch. 60 §T.0 showed the FTL as a hidden
log-structured filesystem inside the device, duplicating and fighting the one above it. Zoned
Namespaces remove the lie: the host sees zones that must be written sequentially and reset in
bulk, and takes responsibility for garbage collection itself. **Fewer layers, a more honest
interface, more work for the host** — the recurring trade of this entire part.

```bash
sudo nvme list && sudo nvme id-ctrl /dev/nvme0 | head -30
sudo nvme smart-log /dev/nvme0        # wear, media errors, throttling
ls /sys/block/nvme0n1/mq/             # one directory per hardware queue
cat /proc/interrupts | grep nvme      # one vector per CPU
sudo nvme zns report-zones /dev/nvme0n1   # if it is a ZNS device
```

---

### T.1 What AHCI cost

SATA SSDs in 2010 were fast enough that the *protocol* became the bottleneck. AHCI, designed in 2004 for rotating disks, imposed:

| AHCI property | Consequence |
|---|---|
| **One command queue** | every CPU contends for it |
| **32 commands deep** | insufficient for a device with internal parallelism |
| **One interrupt** | all completions land on one CPU |
| **Uncacheable register reads per command** | ~2000 ns each, and AHCI needed several |
| **SCSI/ATA translation** | Ch. 68 §T.6's layer, on every command |
| **Command set assumes seeks** | ordering, elevators, and a legacy of rotating-media semantics |

Measured: AHCI's per-command overhead was around 6 µs, against a device latency of 50 µs. As device latency fell toward 10 µs, the protocol became the majority of the cost.

NVMe (2011) was designed against that list:

| NVMe property | Effect |
|---|---|
| **Up to 65,535 queues** | one per CPU; no contention |
| **64K commands per queue** | arbitrary depth |
| **MSI-X, one vector per queue** | completions on the submitting CPU |
| **Zero register reads in the I/O path** | only a doorbell *write* |
| **Native command set** | no translation |
| **Designed for parallel, non-seeking media** | no ordering assumptions |

Per-command overhead dropped to well under 1 µs. The protocol stopped being the bottleneck.

The design lesson generalises: **AHCI's problem was not that it was slow, but that its structure assumed a serial device.** Adding queues to AHCI would not have helped, because the register-per-command model and the single interrupt were structural. A new protocol was required.

### T.2 Queue pairs

The central abstraction:

```
     HOST                              DEVICE
  ┌─────────────┐                  ┌─────────────┐
  │ Submission  │  64-byte entries │             │
  │ Queue (SQ)  │ ───────────────► │  fetches    │
  │ tail ptr    │                  │  commands   │
  └─────────────┘                  │             │
        │ doorbell write           │             │
        ▼                          │             │
  [SQ tail doorbell register]      │             │
                                   │             │
  ┌─────────────┐  16-byte entries │             │
  │ Completion  │ ◄─────────────── │  posts      │
  │ Queue (CQ)  │                  │  completions│
  │ head ptr    │                  │  + MSI-X    │
  └─────────────┘                  └─────────────┘
        │ doorbell write
        ▼
  [CQ head doorbell register]
```

Both queues are **circular buffers in host memory**. The device DMAs commands out of the SQ and DMAs completions into the CQ. The only MMIO in the I/O path is a doorbell **write** — and writes are posted (Ch. 34 §T.4), so they do not stall the CPU.

Three properties make this fast:

**(a) No register reads.** The host never reads a device register during I/O. Reading MMIO costs ~2000 ns because it must round-trip across PCIe uncached; writing costs ~100 ns because it is posted. AHCI required reads; NVMe requires none.

**(b) The phase tag.** How does the host know a completion entry is new without reading a device register? Each CQ entry has a **phase bit**, and the device flips the expected value every time it wraps the queue:

```c
static inline bool nvme_cqe_pending(struct nvme_queue *nvmeq)
{
	struct nvme_completion *hcqe = &nvmeq->cqes[nvmeq->cq_head];

	return (le16_to_cpu(READ_ONCE(hcqe->status)) & 1) == nvmeq->cq_phase;
}
```

The host polls its own memory — a cached read, a few nanoseconds — to detect completions. **This is the single most important micro-design decision in NVMe**, and it is what makes polled I/O (§T.6) viable.

**(c) One queue pair per CPU.** The submission path touches only per-CPU memory plus one doorbell write. This is exactly blk-mq's model (Ch. 63 §T.5), and the correspondence is not accidental — blk-mq and NVMe were designed in the same period by overlapping people, for the same reason.

The mapping is direct:

```
blk_mq_hw_ctx  <->  NVMe queue pair
blk-mq tag     <->  NVMe command identifier
queue_rq()     <->  write an SQE, ring the doorbell
interrupt      <->  drain the CQ, complete requests
```

A 64-byte submission entry:

```c
struct nvme_rw_command {
	__u8			opcode;
	__u8			flags;
	__u16			command_id;
	__le32			nsid;
	__le32			cdw2, cdw3;
	__le64			metadata;
	union nvme_data_ptr	dptr;     /* PRP or SGL */
	__le64			slba;
	__le16			length;
	__le16			control;
	__le32			dsmgmt;
	__le32			reftag;
	__le16			apptag, appmask;
};
```

Compare with a SCSI CDB (Ch. 68 §T.3): the NVMe entry carries the data pointer, the command ID, and the namespace inline. There is no separate scatter-gather descriptor to fetch, and no sense-data round trip on error.

### T.3 PRPs and SGLs

NVMe describes data buffers two ways.

**PRP (Physical Region Page)** is the original and the simpler:

```
PRP1: a 64-bit address (may have a page offset)
PRP2: - unused if the transfer fits in one page
      - a second page address if it fits in two
      - a pointer to a PRP LIST if more
PRP list: an array of page addresses, all page-aligned
```

The constraint that shapes everything: **every entry after the first must be page-aligned and page-sized.** So a scattered buffer whose fragments are not page-aligned cannot be described. The block layer enforces this with `virt_boundary_mask`:

```c
	blk_queue_virt_boundary(q, NVME_CTRL_PAGE_SIZE - 1);
```

which tells the block layer "do not merge two bio segments unless the first ends on a page boundary and the second begins on one." That is why NVMe devices report `virt_boundary_mask` and why some scattered I/O gets split where it would not on SCSI.

**SGL (Scatter Gather List)** removes the restriction: arbitrary address/length descriptors, chainable. NVMe 1.1+ supports it, and it is mandatory for NVMe-oF. The kernel uses SGLs when the controller supports them and the transfer is large enough to benefit:

```c
static inline bool nvme_pci_use_sgls(struct nvme_dev *dev, struct request *req,
				     int nseg)
{
	...
	if (!nvme_ctrl_sgl_supported(&dev->ctrl))
		return false;
	if (!sgl_threshold)
		return false;
	if (nvme_req(req)->flags & NVME_REQ_USERCMD)
		return false;
	return nvme_pci_avg_seg_size(req, nseg) >= sgl_threshold;
}
```

`sgl_threshold` is a module parameter (default 32 KiB average segment size). Small scattered transfers use PRPs (compact); large ones use SGLs (fewer descriptors).

### T.4 Namespaces, and the admin/IO split

A **namespace** is a logical block range — the NVMe equivalent of a SCSI LUN, but with more structure:

```
Controller (nvme0)
  ├─ Namespace 1 (nvme0n1)   — 1 TB, 512-byte blocks, no metadata
  ├─ Namespace 2 (nvme0n2)   — 500 GB, 4K blocks, 8-byte PI
  └─ Namespace 3 (nvme0n3)   — zoned
```

Each namespace has independent:

| Property | Note |
|---|---|
| Size | from the controller's capacity pool |
| **LBA format** | block size and metadata size, chosen from a list the device supports |
| Protection information | T10 DIF, 0/1/2/3 |
| Type | conventional, ZNS, key-value, computational |
| Features | thin provisioning, atomic write sizes |

**The LBA format choice matters.** Most NVMe devices ship formatted with 512-byte logical blocks for compatibility, but support 4096 natively:

```sh
nvme id-ns /dev/nvme0n1 -H | grep -A20 'LBA Format'
```

```
LBA Format  0 : Metadata Size: 0   bytes - Data Size: 512 bytes - Relative Performance: 0x2 Good
LBA Format  1 : Metadata Size: 0   bytes - Data Size: 4096 bytes - Relative Performance: 0 Best (in use)
```

`Relative Performance` is the device telling you which it prefers. Reformatting to 4096 typically improves performance measurably (fewer commands per byte, no internal read-modify-write) and costs all the data.

**The admin/IO command split** is a clean separation:

| Queue | Commands |
|---|---|
| **Admin (queue 0)** | Identify, Create/Delete Queue, Get/Set Features, Format, Firmware, Get Log Page, Namespace Management, Sanitize |
| **I/O (queues 1–N)** | Read, Write, Flush, Write Zeroes, Dataset Management, Compare, Write Uncorrectable, Zone Management |

There is exactly one admin queue, and everything about controller state goes through it. The I/O queues carry only data commands. This means:

- The I/O path has a small, fixed command set that can be optimised hard.
- Management operations cannot interfere with I/O queue behaviour.
- The device can implement the two paths with different hardware.

Compare with SCSI, where INQUIRY, MODE SENSE, and READ all travel the same path and share the same queue.

`Identify` is the discovery mechanism and is worth knowing:

| CNS value | Returns |
|---|---|
| 0x00 | Identify Namespace |
| 0x01 | **Identify Controller** — capabilities, limits, features |
| 0x02 | Active namespace list |
| 0x03 | Namespace identification descriptors (UUID, NGUID, EUI-64) |
| 0x04 | I/O Command Set specific (ZNS, KV) |
| 0x13 | Allocated namespace list |

The Identify Controller structure is several hundred fields and is the authoritative description of what the device can do. `nvme id-ctrl -H` renders it readably, and reading it for your device is the first step in any NVMe investigation.

### T.5 Error handling, and how it differs

NVMe's completion entry carries the status inline:

```c
struct nvme_completion {
	union nvme_result {
		__le16	u16;
		__le32	u32;
		__le64	u64;
	} result;
	__le16	sq_head;
	__le16	sq_id;
	__u16	command_id;
	__le16	status;      /* phase bit + status code type + status code */
};
```

```
bit  0      : phase tag
bits 1-8    : status code
bits 9-10   : status code type
bit  14     : more (additional info in a log page)
bit  15     : do not retry
```

Status code types:

| SCT | Meaning |
|---|---|
| 0x0 | Generic (invalid opcode, invalid field, data transfer error, ...) |
| 0x1 | Command Specific (invalid queue ID, invalid format, ...) |
| 0x2 | **Media and Data Integrity** (unrecovered read error, end-to-end guard check) |
| 0x3 | Path Related (internal path error, controller path error) |
| 0x7 | Vendor Specific |

**No separate REQUEST SENSE round trip.** SCSI requires a second command to learn why the first failed (Ch. 68 §T.4); NVMe puts the reason in the completion. One fewer round trip on every error.

The **DNR (Do Not Retry) bit** is a protocol-level statement that retrying is pointless — the host does not have to guess. Combined with the status code type, the host's decision is straightforward:

```c
static inline enum nvme_disposition nvme_decide_disposition(struct request *req)
{
	if (likely(nvme_req(req)->status == 0))
		return COMPLETE;

	if (blk_noretry_request(req) ||
	    (nvme_req(req)->status & NVME_STATUS_DNR) ||
	    nvme_req(req)->retries >= nvme_max_retries)
		return COMPLETE;

	if (req->cmd_flags & REQ_NVME_MPATH) {
		if (nvme_is_path_error(nvme_req(req)->status) ||
		    blk_queue_dying(req->q))
			return FAILOVER;           /* T.8 */
	} else {
		if (blk_queue_dying(req->q))
			return COMPLETE;
	}

	return RETRY;
}
```

Three outcomes, decided from the status word alone. Compare with `scsi_check_sense`'s long switch (Ch. 68 §1.5).

**Error recovery** escalates much more simply than SCSI's ladder:

```
1. Command timeout (default 30 s, tunable per-command-type)
2. Abort the command (an admin command)
3. Controller reset (CC.EN 0 → 1; recreates all queues)
4. Subsystem reset
5. Mark the controller dead
```

`nvme_timeout()` implements this, and there is a specific subtlety: if the controller is not responding, the *abort* will also time out, so the timeout handler checks whether the controller is healthy first and jumps straight to reset if not.

The **AEN (Asynchronous Event Notification)** mechanism replaces SCSI's UNIT ATTENTION: the host posts an Async Event Request command that the device completes when something happens (namespace changed, SMART threshold crossed, firmware activation needed). It is a clean event channel rather than error-piggybacking.

### T.6 Interrupts, coalescing, and polling

At a million IOPS, one interrupt per completion is 1M interrupts per second — untenable. Three mechanisms:

**(a) Natural batching.** The interrupt handler drains the entire CQ, so a burst of completions costs one interrupt:

```c
static irqreturn_t nvme_irq(int irq, void *data)
{
	struct nvme_queue *nvmeq = data;
	DEFINE_IO_COMP_BATCH(iob);

	if (nvme_poll_cq(nvmeq, &iob)) {
		if (!rq_list_empty(&iob.req_list))
			nvme_pci_complete_batch(&iob);   /* Ch. 63 T.7 */
		return IRQ_HANDLED;
	}
	return IRQ_NONE;
}
```

`nvme_pci_complete_batch` completes many requests with one pass — Ch. 65 §T.6's batched completion.

**(b) Interrupt coalescing.** The device can delay an interrupt until N completions or T microseconds:

```sh
nvme set-feature /dev/nvme0 -f 8 -v 0x0810    # threshold 8, time 16*100us
nvme get-feature /dev/nvme0 -f 8 -H
```

This trades latency for CPU. It is rarely worth enabling on Linux because (a) already handles bursts, and it hurts low-queue-depth latency badly. Most deployments leave it off.

**(c) Polling.** For the lowest latency, avoid interrupts entirely:

```sh
nvme_core.io_poll_queues=4        # or the poll_queues module parameter
cat /sys/block/nvme0n1/queue/io_poll
```

The driver dedicates queues with **no interrupt vector**; completions are found by spinning on the phase tag (§T.2(b)). With `io_uring`'s `IORING_SETUP_IOPOLL`, this gives sub-10 µs round trips.

The queue maps (Ch. 63 §T.5):

```c
enum {
	NVMEQ_TYPE_DEFAULT,	/* reads and writes */
	NVMEQ_TYPE_READ,	/* dedicated read queues */
	NVMEQ_TYPE_POLL,	/* no interrupt */
};

module_param(write_queues, uint, 0644);
module_param(poll_queues, uint, 0644);
```

Dedicated write queues exist so that a write flood cannot fill the queues a latency-sensitive reader needs — isolation at the queue level rather than at the scheduler level (Ch. 64).

**Hybrid polling** was an attempt to get polling's latency with less CPU: sleep for most of the expected latency, then poll. It was measured to be rarely better than either pure approach and was removed in 6.3. The lesson: an adaptive mechanism must beat both endpoints, not sit between them.

### T.7 NVMe over Fabrics

The command set is transport-independent. NVMe-oF carries it over:

| Transport | Note |
|---|---|
| **RDMA** (RoCE, iWARP, InfiniBand) | lowest latency; the queue pair maps onto an RDMA QP |
| **TCP** | works anywhere; ~20–50 µs added latency |
| **FC** | for existing Fibre Channel fabrics |

The abstraction that makes it work: **the SQ/CQ pair is already a message-passing interface.** Over PCIe the messages are DMA'd; over RDMA they are RDMA sends; over TCP they are PDUs on a socket. The upper driver, the namespace model, and the command set are identical.

```
nvme_core  (namespaces, controllers, multipath, ioctl)
     |
  +--+---------+---------+---------+
  |            |         |         |
 pci         rdma       tcp        fc          <- transports
```

`drivers/nvme/host/core.c` is transport-independent; `pci.c`, `rdma.c`, `tcp.c`, `fc.c` implement `struct nvme_ctrl_ops`.

Discovery:

```sh
nvme discover -t tcp -a 192.168.1.10 -s 4420
nvme connect -t tcp -a 192.168.1.10 -s 4420 -n nqn.2024-01.com.example:target1
nvme list-subsys
nvme disconnect -n nqn.2024-01.com.example:target1
```

The **NQN (NVMe Qualified Name)** identifies subsystems and hosts, in the same role as an iSCSI IQN. `/etc/nvme/hostnqn` holds the host's.

The target side is in-kernel (`drivers/nvme/target/`), which is notable: unlike iSCSI's history of userspace targets, NVMe-oF's reference target ships in the kernel and is configured through configfs:

```sh
mkdir /sys/kernel/config/nvmet/subsystems/nqn.example
echo 1 > /sys/kernel/config/nvmet/subsystems/nqn.example/attr_allow_any_host
mkdir /sys/kernel/config/nvmet/subsystems/nqn.example/namespaces/1
echo -n /dev/nvme0n1 > .../namespaces/1/device_path
echo 1 > .../namespaces/1/enable
```

**NVMe-oF TCP versus iSCSI** is the practical comparison: both carry a block protocol over TCP, but NVMe-oF's queue model means many parallel queues rather than iSCSI's typically-one session, and there is no SCSI translation. Measured, NVMe-oF/TCP typically achieves 2–4× the IOPS of iSCSI on the same hardware.

### T.8 Multipath, done in the protocol

SCSI multipath (Ch. 68 §T.8) is bolted on: the kernel sees several block devices, userspace identifies them as the same LUN via VPD 0x83, and dm-multipath unifies them.

NVMe builds it in. A namespace can be attached to multiple controllers in the same **subsystem**, and the subsystem NQN plus the namespace identifier (NGUID/UUID) establishes identity at the protocol level. The kernel creates **one** block device with several paths:

```
nvme-subsys0
  ├─ nvme0 (controller, path A)
  ├─ nvme1 (controller, path B)
  └─ nvme0n1  <- ONE block device, two paths
```

```sh
nvme list-subsys
ls /sys/class/nvme-subsystem/nvme-subsys0/
cat /sys/block/nvme0n1/multipath 2>/dev/null
```

**ANA (Asymmetric Namespace Access)** is the standardised path-state protocol — the NVMe counterpart of SCSI's ALUA, but specified in the base standard rather than as an optional vendor-handled extension:

| ANA state | Meaning |
|---|---|
| optimized | use this path |
| non-optimized | works, but prefer optimized |
| inaccessible | do not use |
| persistent loss | permanently gone |
| change | in transition |

```sh
nvme ana-log /dev/nvme0
cat /sys/block/nvme0n1/ana_state 2>/dev/null
```

I/O policies:

```sh
cat /sys/class/nvme-subsystem/nvme-subsys0/iopolicy
# numa (default) | round-robin | queue-depth
echo round-robin > /sys/class/nvme-subsystem/nvme-subsys0/iopolicy
```

`numa` picks the path whose controller is closest in NUMA terms — a policy that only makes sense because the multipath layer knows about the transport.

Native multipath can be disabled (`nvme_core.multipath=0`) to use dm-multipath instead, which some storage vendors still require for their management tooling. The native path is faster (no extra dm layer) and is the default.

### T.9 ZNS and the end of the FTL

Chapter 60 §T.5 introduced zoned storage. NVMe ZNS is its most fully-developed form:

```
Namespace divided into ZONES
Each zone: must be written SEQUENTIALLY from its write pointer
           must be RESET before rewriting
Limited number of zones may be OPEN simultaneously
```

Zone states:

```
EMPTY -> IMPLICITLY OPENED -> CLOSED -> FULL
      -> EXPLICITLY OPENED ->
FULL  -> (reset) -> EMPTY
```

The `max_open_zones` and `max_active_zones` limits are the interesting constraint: each open zone requires device resources (a write buffer, a portion of the mapping table), so the device caps how many can be open. The host must manage this — which is exactly the information the FTL never had.

What ZNS removes from the device:

| Removed | Consequence |
|---|---|
| The L2P mapping table | **less DRAM on the drive** (a real cost saving) |
| Garbage collection | **no write amplification from GC** |
| Over-provisioning | **7–28 % more usable capacity** |
| Unpredictable GC latency | **consistent tail latency** |

And what it demands of the host: a log-structured filesystem (F2FS, btrfs) or an application that manages zones directly (RocksDB with a zoned backend, SPDK).

Zone Append is ZNS's cleverest command: rather than specifying an LBA, it says "append this to this zone, and tell me where it landed." The device returns the assigned LBA in the completion. This removes the need for the host to serialise writes to a zone — multiple threads can append concurrently and the device assigns positions. Linux exposes it as `REQ_OP_ZONE_APPEND` (Ch. 63 §T.3).

**Why it matters conceptually:** ZNS is the clearest case in Part 3 of the end-to-end argument (Ch. 00 §T.4). The FTL was doing log-structuring blindly, guessing at what the host meant. ZNS moves that work to the host, which knows which data is hot, which blocks are free, and when a region will be rewritten wholesale. The result is better on every metric except "works with existing software."

### T.10 Passthrough and userspace drivers

NVMe provides a clean path for userspace to issue arbitrary commands:

```c
struct nvme_passthru_cmd {
	__u8	opcode;
	__u8	flags;
	__u16	rsvd1;
	__u32	nsid;
	__u32	cdw2, cdw3;
	__u64	metadata;
	__u64	addr;
	__u32	metadata_len;
	__u32	data_len;
	__u32	cdw10, cdw11, cdw12, cdw13, cdw14, cdw15;
	__u32	timeout_ms;
	__u32	result;
};

#define NVME_IOCTL_ADMIN_CMD	_IOWR('N', 0x41, struct nvme_passthru_cmd)
#define NVME_IOCTL_IO_CMD	_IOWR('N', 0x43, struct nvme_passthru_cmd)
#define NVME_IOCTL_ADMIN64_CMD	_IOWR('N', 0x47, struct nvme_passthru_cmd64)
#define NVME_IOCTL_IO64_CMD	_IOWR('N', 0x48, struct nvme_passthru_cmd64)
```

That is how `nvme-cli` works, and it is why `nvme-cli` can support new device features without kernel changes — the same escape-hatch principle as SCSI's `ATA PASS-THROUGH` (Ch. 68 §T.6).

Beyond ioctls, three approaches to bypassing more of the stack:

**(a) `io_uring` passthrough (`IORING_OP_URING_CMD`)** — submit NVMe commands asynchronously through io_uring, with the same batching and polling as normal I/O. This gives near-SPDK latency without leaving the kernel, and is the current best answer for applications that need raw device access.

**(b) `/dev/ngX` (NVMe generic)** — a character device per namespace supporting only passthrough, for applications that want commands but not a block device.

**(c) SPDK / VFIO** — take the device away from the kernel entirely, map its registers into userspace with VFIO (Ch. 36), and poll. Achieves the lowest possible latency and highest IOPS per core, at the cost of: the device is unusable by anything else, no filesystem, no page cache, and you have written a device driver.

The trend worth noting: **`io_uring` passthrough has closed most of the gap to SPDK.** The measured difference is now small enough that the operational cost of a userspace driver is rarely justified except at extreme scale.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `drivers/nvme/host/core.c` ★★★ | transport-independent: namespaces, controllers, ioctls |
| `drivers/nvme/host/pci.c` ★★★ | **the PCIe transport**; queues, doorbells, PRPs, interrupts |
| `drivers/nvme/host/nvme.h` ★★★ | `nvme_ctrl`, `nvme_ns`, `nvme_ctrl_ops` |
| `drivers/nvme/host/multipath.c` ★★★ | §T.8 |
| `drivers/nvme/host/zns.c` | §T.9 |
| `drivers/nvme/host/ioctl.c` ★★★ | §T.10's passthrough, including io_uring cmd |
| `drivers/nvme/host/tcp.c`, `rdma.c`, `fc.c` | §T.7's transports |
| `drivers/nvme/host/fault_inject.c` | error injection |
| `drivers/nvme/target/` ★★★ | the in-kernel NVMe-oF target |
| `include/linux/nvme.h` ★★★ | **the complete protocol definition** |
| `include/uapi/linux/nvme_ioctl.h` | §T.10 |
| `block/blk-zoned.c` | zone management (Ch. 60 §T.5) |
| `Documentation/nvme/` | feature documentation |

`include/linux/nvme.h` deserves special mention: it is a complete, readable transcription of the NVMe specification's data structures. Reading it is a faster way to learn the protocol than reading the specification.

### 1.2 Queue structures

```c
struct nvme_queue {
	struct nvme_dev *dev;
	spinlock_t sq_lock;
	void *sq_cmds;                   /* the submission queue */
	struct nvme_completion *cqes;    /* the completion queue */
	dma_addr_t sq_dma_addr;
	dma_addr_t cq_dma_addr;
	u32 __iomem *q_db;               /* THE doorbell (T.2) */
	u32 q_depth;
	u16 cq_vector;
	u16 sq_tail;
	u16 last_sq_tail;
	u16 cq_head;
	u16 qid;
	u8 cq_phase;                     /* T.2(b) */
	u8 sqes;
	unsigned long flags;
#define NVMEQ_ENABLED		0
#define NVMEQ_SQ_CMB		1
#define NVMEQ_DELETE_ERROR	2
#define NVMEQ_POLLED		3
	__le32 *dbbuf_sq_db;             /* shadow doorbell (virtualisation) */
	__le32 *dbbuf_cq_db;
	__le32 *dbbuf_sq_ei;
	__le32 *dbbuf_cq_ei;
	struct completion delete_done;
};

struct nvme_iod {                        /* per-request driver data */
	struct nvme_request req;
	struct nvme_command cmd;
	bool aborted;
	s8 nr_allocations;
	unsigned int dma_len;
	dma_addr_t first_dma;
	dma_addr_t meta_dma;
	struct sg_table sgt;
	union nvme_descriptor list[NVME_MAX_NR_ALLOCATIONS];
};
```

`nvme_iod` is the `cmd_size` allocation of Ch. 65 §T.4 — preallocated per tag, so the I/O path never allocates.

The **shadow doorbell** (`dbbuf_*`) is a virtualisation optimisation: in a VM, an MMIO doorbell write causes a VM exit (~2 µs). The shadow doorbell buffer lets the guest write to memory instead, with the hypervisor polling, eliminating the exit. It is negotiated via an admin command and is why NVMe in VMs is much faster than it used to be.

### 1.3 Submission

```c
static blk_status_t nvme_queue_rq(struct blk_mq_hw_ctx *hctx,
				  const struct blk_mq_queue_data *bd)
{
	struct nvme_queue *nvmeq = hctx->driver_data;
	struct nvme_dev *dev = nvmeq->dev;
	struct request *req = bd->rq;
	struct nvme_iod *iod = blk_mq_rq_to_pdu(req);
	blk_status_t ret;

	if (unlikely(!test_bit(NVMEQ_ENABLED, &nvmeq->flags)))
		return BLK_STS_IOERR;

	if (unlikely(!nvme_check_ready(&dev->ctrl, req, true)))
		return nvme_fail_nonready_command(&dev->ctrl, req);

	ret = nvme_prep_rq(dev, req);        /* build the 64-byte SQE */
	if (unlikely(ret))
		return ret;

	spin_lock(&nvmeq->sq_lock);
	nvme_sq_copy_cmd(nvmeq, &iod->cmd);
	nvme_write_sq_db(nvmeq, bd->last);   /* Ch. 65 T.4: bd->last */
	spin_unlock(&nvmeq->sq_lock);
	return BLK_STS_OK;
}

static inline void nvme_write_sq_db(struct nvme_queue *nvmeq, bool write_sq)
{
	if (!write_sq) {
		/* Not the last of the batch: defer the doorbell. */
		u16 next_tail = nvmeq->sq_tail + 1;

		if (next_tail == nvmeq->q_depth)
			next_tail = 0;
		if (next_tail != nvmeq->last_sq_tail)
			return;
	}

	if (nvme_dbbuf_update_and_check_event(nvmeq->sq_tail,
			nvmeq->dbbuf_sq_db, nvmeq->dbbuf_sq_ei))
		writel(nvmeq->sq_tail, nvmeq->q_db);   /* THE doorbell */
	nvmeq->last_sq_tail = nvmeq->sq_tail;
}
```

**One `writel` per batch, and nothing else touches the device.** That is §T.1's entire improvement, visible in eight lines.

Building the command:

```c
static blk_status_t nvme_setup_rw(struct nvme_ns *ns, struct request *req,
				  struct nvme_command *cmnd, enum nvme_opcode op)
{
	u16 control = 0;
	u32 dsmgmt = 0;

	if (req->cmd_flags & REQ_FUA)
		control |= NVME_RW_FUA;         /* Ch. 61 T.3 */
	if (req->cmd_flags & (REQ_FAILFAST_DEV | REQ_RAHEAD))
		control |= NVME_RW_LR;
	if (req->cmd_flags & REQ_RAHEAD)
		dsmgmt |= NVME_RW_DSM_FREQ_PREFETCH;

	cmnd->rw.opcode = op;
	cmnd->rw.flags = 0;
	cmnd->rw.nsid = cpu_to_le32(ns->head->ns_id);
	cmnd->rw.cdw2 = 0;
	cmnd->rw.cdw3 = 0;
	cmnd->rw.metadata = 0;
	cmnd->rw.slba = cpu_to_le64(nvme_sect_to_lba(ns, blk_rq_pos(req)));
	cmnd->rw.length = cpu_to_le16((blk_rq_bytes(req) >> ns->head->lba_shift) - 1);
	cmnd->rw.reftag = 0;
	cmnd->rw.apptag = 0;
	cmnd->rw.appmask = 0;

	if (ns->head->ms) {
		/* Protection information (T10 DIF) handling */
		...
	}

	cmnd->rw.control = cpu_to_le16(control);
	cmnd->rw.dsmgmt = cpu_to_le32(dsmgmt);
	return 0;
}
```

Note `length` is **blocks minus one** — a zero-length transfer is impossible, so the encoding uses the space. A classic protocol economy, and a classic off-by-one hazard.

### 1.4 Completion

```c
static inline int nvme_poll_cq(struct nvme_queue *nvmeq,
			       struct io_comp_batch *iob)
{
	int found = 0;

	while (nvme_cqe_pending(nvmeq)) {       /* T.2(b): a CACHED read */
		found++;
		/* Read the CQE before the phase tag of the next one. */
		dma_rmb();
		nvme_handle_cqe(nvmeq, iob, nvmeq->cq_head);
		nvme_update_cq_head(nvmeq);
	}

	if (found)
		nvme_ring_cq_doorbell(nvmeq);
	return found;
}

static inline void nvme_handle_cqe(struct nvme_queue *nvmeq,
				   struct io_comp_batch *iob, u16 idx)
{
	struct nvme_completion *cqe = &nvmeq->cqes[idx];
	__u16 command_id = READ_ONCE(cqe->command_id);
	struct request *req;

	if (unlikely(nvme_is_aen_req(nvmeq->qid, command_id))) {
		nvme_complete_async_event(&nvmeq->dev->ctrl,
				cqe->status, &cqe->result);   /* T.5's AEN */
		return;
	}

	req = nvme_find_rq(nvme_queue_tagset(nvmeq), cqe);
	if (unlikely(!req)) {
		dev_warn(nvmeq->dev->ctrl.device,
			"invalid id %d completed on queue %d\n",
			command_id, le16_to_cpu(cqe->sq_id));
		return;
	}

	trace_nvme_sq(req, cqe->sq_head, nvmeq->sq_tail);
	if (!nvme_try_complete_req(req, cqe->status, cqe->result) &&
	    !blk_mq_add_to_batch(req, iob, nvme_req(req)->status,
					nvme_pci_complete_batch))
		nvme_pci_complete_rq(req);
}
```

`dma_rmb()` between reading the CQE and checking the next phase tag is essential: without it, the CPU could read the next entry's phase tag before the current entry's contents, seeing a "valid" entry with stale data. This is Ch. 12's memory ordering, in a place where getting it wrong produces corruption that is nearly impossible to debug.

### 1.5 Timeout and reset

```c
static enum blk_eh_timer_return nvme_timeout(struct request *req)
{
	struct nvme_iod *iod = blk_mq_rq_to_pdu(req);
	struct nvme_queue *nvmeq = req->mq_hctx->driver_data;
	struct nvme_dev *dev = nvmeq->dev;
	struct request *abort_req;
	struct nvme_command cmd = { };
	u32 csts = readl(dev->bar + NVME_REG_CSTS);

	/* Polled queues have no interrupt; check the CQ ourselves. */
	if (req->mq_hctx->type == HCTX_TYPE_POLL) {
		nvme_poll(req->mq_hctx, NULL);
		goto check_ready;
	}

	/* Did it complete while we were getting here? */
	if (nvme_poll_cq(nvmeq, NULL)) {
		dev_warn(dev->ctrl.device,
			 "I/O tag %d (%04x) QID %d timeout, completion polled\n",
			 req->tag, nvme_cid(req), nvmeq->qid);
		return BLK_EH_DONE;
	}

	/* T.5: if the controller is sick, do not bother with an abort. */
	if (dev->ctrl.state != NVME_CTRL_LIVE || csts & NVME_CSTS_CFS) {
		dev_warn(dev->ctrl.device,
			 "I/O tag %d (%04x) QID %d timeout, reset controller\n",
			 req->tag, nvme_cid(req), nvmeq->qid);
		nvme_req(req)->flags |= NVME_REQ_CANCELLED;
		goto disable;
	}

	if (!nvmeq->qid || iod->aborted) {
		/* Admin queue, or the abort itself timed out: escalate. */
		dev_warn(dev->ctrl.device,
			 "I/O tag %d (%04x) QID %d timeout, reset controller\n",
			 req->tag, nvme_cid(req), nvmeq->qid);
		nvme_req(req)->flags |= NVME_REQ_CANCELLED;
		goto disable;
	}

	iod->aborted = true;
	cmd.abort.opcode = nvme_admin_abort_cmd;
	cmd.abort.cid = nvme_cid(req);
	cmd.abort.sqid = cpu_to_le16(nvmeq->qid);
	...
	abort_req = blk_mq_alloc_request(dev->ctrl.admin_q, nvme_req_op(&cmd),
					 BLK_MQ_REQ_NOWAIT);
	...
	return BLK_EH_RESET_TIMER;       /* give the abort time to work */

disable:
	...
	nvme_dev_disable(dev, false);
	...
	return BLK_EH_DONE;
}
```

**`nvme_poll_cq()` at the top is the defensive check**: a completion may have been posted but the interrupt lost. Polling before declaring a timeout avoids spurious resets, and lost interrupts are real (a known hazard with some virtualisation stacks and some firmware).

### 1.6 Observability

| Where | What |
|---|---|
| `nvme list`, `nvme list-subsys` ★★★ | devices and subsystems |
| `nvme id-ctrl -H /dev/nvme0` ★★★ | **the controller's capabilities, in full** |
| `nvme id-ns -H /dev/nvme0n1` ★★★ | namespace geometry, LBA formats |
| `nvme smart-log /dev/nvme0` ★★★ | health, wear, temperature, errors |
| `nvme error-log /dev/nvme0` ★★★ | the last N errors with their commands |
| `nvme fw-log`, `nvme effects-log`, `nvme telemetry-log` | |
| `nvme get-feature -f N -H` ★★★ | queue counts, power state, coalescing |
| `nvme zns report-zones` | §T.9 |
| `nvme ana-log` | §T.8 |
| `/sys/class/nvme/nvme0/` ★★★ | `model`, `serial`, `firmware_rev`, `cntlid`, `state`, `queue_count`, `sqsize` |
| `/sys/class/nvme-subsystem/` | §T.8 |
| `/sys/block/nvme0n1/queue/` | the block-layer view |
| `/sys/kernel/debug/block/nvme0n1/` ★★★ | per-hctx state (Ch. 63) |
| `trace-cmd record -e nvme:\*` ★★★ | `nvme_setup_cmd`, `nvme_complete_rq`, `nvme_sq` |
| `/sys/kernel/debug/nvme*/fault_inject/` | error injection |
| `lspci -vvv -s $(basename ...)` | link width and speed |

The `nvme:nvme_setup_cmd` tracepoint prints the decoded command, which makes it far more useful than a raw hex dump:

```
nvme_setup_cmd: nvme0: qid=3, cmdid=1234, nsid=1, flags=0x0, meta=0x0,
                cmd=(nvme_cmd_read slba=12345, len=7, ctrl=0x0, dsmgmt=0, reftag=0)
```

---

## 2. Practice

### Lab 69.1 — Identify your device completely

```sh
sudo apt install -y nvme-cli
sudo nvme list
sudo nvme list-subsys
lspci | grep -i nvme
```

The controller:

```sh
sudo nvme id-ctrl -H /dev/nvme0 | head -80
```

The fields that matter:

```sh
sudo nvme id-ctrl /dev/nvme0 -H | grep -iE \
  'mdts|cntlid|ver|oacs|acl|aerl|frmw|lpa|elpe|npss|sqes|cqes|nn|oncs|fuses|vwc|awun|awupf|sgls|subnqn'
```

| Field | Meaning |
|---|---|
| `mdts` | **Maximum Data Transfer Size**, as `2^mdts × page size` → `max_hw_sectors` |
| `vwc` | **Volatile Write Cache** present → Ch. 61's flush requirement |
| `awun`/`awupf` | **atomic write sizes** — how large a write is guaranteed atomic |
| `oncs` | optional commands: write zeroes, dataset management, compare, reservations |
| `sgls` | SGL support (§T.3) |
| `npss` | number of power states |
| `nn` | maximum namespaces |

Verify the derivation:

```sh
MDTS=$(sudo nvme id-ctrl /dev/nvme0 | grep -oP 'mdts\s*:\s*\K\d+')
echo "mdts=$MDTS -> max transfer = $((4096 * (1 << MDTS) / 1024)) KiB"
cat /sys/block/nvme0n1/queue/max_hw_sectors_kb
```

The namespace:

```sh
sudo nvme id-ns -H /dev/nvme0n1 | head -60
sudo nvme id-ns /dev/nvme0n1 -H | grep -A20 'LBA Format'
```

**Check whether you are running at 512 or 4096** (§T.4):

```sh
cat /sys/block/nvme0n1/queue/logical_block_size
sudo nvme id-ns /dev/nvme0n1 | grep -E 'flbas|lbaf'
```

If format 1 is "Best" and format 0 is "in use", reformatting is a free performance improvement:

```sh
# DESTROYS ALL DATA
# sudo nvme format /dev/nvme0n1 --lbaf=1 --force
```

Namespace identifiers — §T.8's identity:

```sh
sudo nvme id-ns /dev/nvme0n1 -H | grep -iE 'nguid|eui64'
sudo nvme ns-descs /dev/nvme0n1 -H
cat /sys/block/nvme0n1/{nguid,uuid,eui} 2>/dev/null
cat /sys/block/nvme0n1/wwid
```

Features:

```sh
for f in 1 2 4 5 6 7 8 9 0x0b 0x0c; do
  echo -n "feature 0x$(printf %02x $f): "
  sudo nvme get-feature /dev/nvme0 -f $f -H 2>/dev/null | head -2 | tail -1
done
# 7 = number of queues; 8 = interrupt coalescing (T.6)
```

PCIe link:

```sh
PCI=$(basename $(readlink /sys/class/nvme/nvme0/device))
sudo lspci -vvv -s $PCI | grep -iE 'LnkCap:|LnkSta:|MaxPayload|MaxReadReq'
# Check that LnkSta matches LnkCap -- a device running at x2 instead of x4
# is a common and easily-missed problem.
```

---

### Lab 69.2 — Queues and the blk-mq mapping

```sh
cat /sys/class/nvme/nvme0/queue_count
ls /sys/block/nvme0n1/mq/ | wc -l
nproc

for q in /sys/block/nvme0n1/mq/*/; do
  printf "hctx%-3s cpus: %s\n" $(basename $q) "$(cat $q/cpu_list)"
done | head -10
```

**One hardware queue per CPU** — §T.2(c) and Ch. 63 §T.5, on real hardware.

```sh
for q in /sys/kernel/debug/block/nvme0n1/hctx*/; do
  echo "=== $(basename $q) ==="
  sudo cat $q/type
  sudo cat $q/tags | head -6
done 2>/dev/null | head -30
```

Interrupt affinity:

```sh
grep nvme /proc/interrupts | head -10
for irq in $(grep -oP '^\s*\K\d+(?=:.*nvme)' /proc/interrupts | head -5); do
  echo "irq $irq -> cpu $(cat /proc/irq/$irq/smp_affinity_list)"
done
```

Watch completions land on the submitting CPU:

```sh
sudo bpftrace -e '
tracepoint:nvme:nvme_setup_cmd  { @submit[cpu] = count(); }
tracepoint:nvme:nvme_complete_rq { @complete[cpu] = count(); }
interval:s:10 { print(@submit); print(@complete); exit(); }' &

sudo fio --name=t --filename=/dev/nvme0n1 --direct=1 --rw=randread --bs=4k \
         --iodepth=32 --ioengine=libaio --runtime=10 --time_based \
         --numjobs=$(nproc) --readonly > /dev/null 2>&1
wait
```

The histograms should match closely — that is §T.2(c)'s payoff.

Queue count tuning:

```sh
sudo nvme get-feature /dev/nvme0 -f 7 -H
cat /sys/module/nvme/parameters/* 2>/dev/null
# write_queues, poll_queues, io_queue_depth, sgl_threshold, use_threaded_interrupts

# Reload with dedicated write and poll queues (T.6)
sudo umount /mnt/nvme 2>/dev/null
sudo modprobe -r nvme 2>/dev/null   # only if not the root device!
sudo modprobe nvme write_queues=4 poll_queues=4
for q in /sys/kernel/debug/block/nvme0n1/hctx*/; do
  echo "$(basename $q): $(sudo cat $q/type 2>/dev/null)"
done | sort -u
```

Scalability:

```sh
for n in 1 2 4 8 $(nproc); do
  echo -n "$n jobs: "
  sudo fio --name=t --filename=/dev/nvme0n1 --direct=1 --rw=randread --bs=4k \
           --iodepth=32 --ioengine=libaio --runtime=8 --time_based \
           --numjobs=$n --readonly --group_reporting 2>/dev/null | \
    grep -oP 'IOPS=\K[^,]+'
done
```

Near-linear scaling is the expected result and is what §T.1 bought.

---

### Lab 69.3 — Watch commands on the wire

```sh
sudo trace-cmd record -e nvme:nvme_setup_cmd -e nvme:nvme_complete_rq -- \
  sudo dd if=/dev/nvme0n1 of=/dev/null bs=4k count=10 iflag=direct 2>/dev/null
sudo trace-cmd report | head -20
```

```
nvme_setup_cmd: nvme0: disk=nvme0n1, qid=5, cmdid=4106, nsid=1, flags=0x0,
                meta=0x0, cmd=(nvme_cmd_read slba=0, len=7, ctrl=0x0, dsmgmt=0, reftag=0)
nvme_complete_rq: nvme0: disk=nvme0n1, qid=5, cmdid=4106, res=0x0, retries=0,
                  flags=0x0, status=0x0
```

Note `len=7` for an 8-block read — §1.3's zero-based encoding.

Map every block-layer operation to its NVMe command:

```sh
sudo mkfs.ext4 -qF /dev/nvme0n1 2>/dev/null   # only on a scratch device!
sudo mkdir -p /mnt/nvme && sudo mount /dev/nvme0n1 /mnt/nvme

sudo trace-cmd record -e nvme:nvme_setup_cmd -- sudo sh -c '
  echo test > /mnt/nvme/f
  sync
  fallocate -p -o 0 -l 1M /mnt/nvme/f 2>/dev/null
  fstrim /mnt/nvme' 2>/dev/null
sudo trace-cmd report | grep -oP 'cmd=\(\K\w+' | sort | uniq -c
```

```
     12 nvme_cmd_write
      3 nvme_cmd_flush           <- Ch. 61's fsync
      1 nvme_cmd_dsm             <- discard
      8 nvme_cmd_read
```

FUA and flush:

```sh
sudo trace-cmd record -e nvme:nvme_setup_cmd -- \
  sudo dd if=/dev/zero of=/mnt/nvme/x bs=4k count=10 oflag=direct,dsync 2>/dev/null
sudo trace-cmd report | grep -oP 'ctrl=0x\K\w+|cmd=\(\K\w+' | paste - - | sort | uniq -c
# ctrl bit 0x8000 is FUA (Ch. 61 T.3)
```

Latency decomposition:

```sh
sudo bpftrace -e '
tracepoint:nvme:nvme_setup_cmd { @start[args->cid, args->qid] = nsecs; }
tracepoint:nvme:nvme_complete_rq /@start[args->cid, args->qid]/ {
	@device_ns = hist(nsecs - @start[args->cid, args->qid]);
	delete(@start[args->cid, args->qid]);
}
tracepoint:block:block_rq_issue { @blk_start[args->dev, args->sector] = nsecs; }
tracepoint:block:block_rq_complete /@blk_start[args->dev, args->sector]/ {
	@blk_ns = hist(nsecs - @blk_start[args->dev, args->sector]);
	delete(@blk_start[args->dev, args->sector]);
}
interval:s:10 { print(@device_ns); print(@blk_ns); exit(); }' &

sudo fio --name=t --filename=/dev/nvme0n1 --direct=1 --rw=randread --bs=4k \
         --iodepth=1 --runtime=10 --time_based --readonly > /dev/null 2>&1
wait
```

---

### Lab 69.4 — Polling versus interrupts

```sh
cat /sys/block/nvme0n1/queue/io_poll
cat /sys/module/nvme/parameters/poll_queues
```

If poll queues are not configured:

```sh
sudo umount /mnt/nvme 2>/dev/null
sudo modprobe -r nvme && sudo modprobe nvme poll_queues=4
cat /sys/block/nvme0n1/queue/io_poll
for q in /sys/kernel/debug/block/nvme0n1/hctx*/; do
  sudo cat $q/type
done | sort | uniq -c
```

The comparison:

```sh
for mode in "" "--hipri=1"; do
  echo "=== ${mode:-interrupt-driven} ==="
  sudo fio --name=t --filename=/dev/nvme0n1 --direct=1 --rw=randread --bs=4k \
           --iodepth=1 --ioengine=io_uring $mode --runtime=10 --time_based \
           --readonly 2>/dev/null | grep -E 'clat.*avg|99.00th|IOPS|cpu.*usr'
done
```

Polling should show noticeably lower latency and much higher CPU. That is the trade.

Interrupt count:

```sh
BEFORE=$(grep nvme /proc/interrupts | awk '{for(i=2;i<=NF-2;i++) s+=$i} END {print s}')
sudo fio --name=t --filename=/dev/nvme0n1 --direct=1 --rw=randread --bs=4k \
         --iodepth=32 --ioengine=libaio --runtime=10 --time_based \
         --readonly > /dev/null 2>&1
AFTER=$(grep nvme /proc/interrupts | awk '{for(i=2;i<=NF-2;i++) s+=$i} END {print s}')
echo "interrupts: $((AFTER-BEFORE))"
# Compare with the IOPS: the ratio shows natural batching (T.6(a)).
```

Interrupt coalescing — §T.6(b):

```sh
sudo nvme get-feature /dev/nvme0 -f 8 -H

# threshold=8 completions, time=16 (in 100us units)
sudo nvme set-feature /dev/nvme0 -f 8 -v 0x1008
sudo nvme get-feature /dev/nvme0 -f 8 -H

for qd in 1 32; do
  echo -n "coalescing on, iodepth=$qd: "
  sudo fio --name=t --filename=/dev/nvme0n1 --direct=1 --rw=randread --bs=4k \
           --iodepth=$qd --ioengine=libaio --runtime=8 --time_based \
           --readonly 2>/dev/null | grep -oP 'clat.*avg=\K[0-9.]+'
done

sudo nvme set-feature /dev/nvme0 -f 8 -v 0      # off
for qd in 1 32; do
  echo -n "coalescing off, iodepth=$qd: "
  sudo fio --name=t --filename=/dev/nvme0n1 --direct=1 --rw=randread --bs=4k \
           --iodepth=$qd --ioengine=libaio --runtime=8 --time_based \
           --readonly 2>/dev/null | grep -oP 'clat.*avg=\K[0-9.]+'
done
```

Coalescing hurts low-queue-depth latency badly and helps almost nothing at high depth — which is why it stays off.

The full latency matrix:

```sh
for eng in psync libaio io_uring; do
  for qd in 1 8 64; do
    printf "%-10s qd=%-3s " $eng $qd
    sudo fio --name=t --filename=/dev/nvme0n1 --direct=1 --rw=randread --bs=4k \
             --iodepth=$qd --ioengine=$eng --runtime=6 --time_based \
             --readonly 2>/dev/null | grep -oP 'IOPS=\K[^,]+|clat.*avg=\K[0-9.]+' | \
      tr '\n' ' '
    echo
  done
done
```

---

### Lab 69.5 — SMART, errors, and wear

```sh
sudo nvme smart-log /dev/nvme0
```

```
critical_warning                    : 0
temperature                         : 42 C
available_spare                     : 100%
available_spare_threshold           : 10%
percentage_used                     : 3%
data_units_read                     : 45,234,112
data_units_written                  : 31,004,238
host_read_commands                  : 892,341,223
host_write_commands                 : 412,883,491
controller_busy_time                : 1,234
power_cycles                        : 421
power_on_hours                      : 8,432
unsafe_shutdowns                    : 12
media_errors                        : 0
num_err_log_entries                 : 0
```

The fields that matter:

```sh
cat <<'EOF'
percentage_used         the device's own wear estimate; >100% = past rated endurance
available_spare         remaining spare blocks; below the threshold = failing
media_errors            ** unrecovered errors -- should be 0 **
num_err_log_entries     check the error log if non-zero
unsafe_shutdowns        power losses; relevant to Ch. 61's durability
data_units_written      512,000 bytes each -> compute total writes
critical_warning        bit 0: spare low, 1: temp, 2: reliability, 3: read-only, 4: volatile mem
EOF
```

Compute write amplification and endurance:

```sh
DUW=$(sudo nvme smart-log /dev/nvme0 | grep -oP 'data_units_written\s*:\s*\K[\d,]+' | tr -d ',')
TBW=$(echo "scale=2; $DUW * 512000 / 1000000000000" | bc)
PCT=$(sudo nvme smart-log /dev/nvme0 | grep -oP 'percentage_used\s*:\s*\K\d+')
echo "host writes: ${TBW} TB, wear: ${PCT}%"
[ "$PCT" -gt 0 ] && echo "implied endurance: $(echo "scale=1; $TBW * 100 / $PCT" | bc) TBW"
```

The error log:

```sh
sudo nvme error-log /dev/nvme0 -e 16 | head -40
# Each entry: error count, sqid, cmdid, status field, LBA, namespace
```

Vendor-specific and telemetry:

```sh
sudo nvme get-log /dev/nvme0 --log-id=0xca --log-len=512 -b 2>/dev/null | hexdump -C | head
sudo nvme intel smart-log-add /dev/nvme0 2>/dev/null
sudo nvme wdc smart-log-add /dev/nvme0 2>/dev/null
sudo nvme telemetry-log /dev/nvme0 -o /tmp/telemetry.bin 2>/dev/null && \
  ls -lh /tmp/telemetry.bin
sudo nvme effects-log /dev/nvme0 -H | head -20
```

Self-test:

```sh
sudo nvme device-self-test /dev/nvme0 -s 1      # short
sleep 60
sudo nvme self-test-log /dev/nvme0
# sudo nvme device-self-test /dev/nvme0 -s 2    # extended (hours)
```

Fault injection:

```sh
ls /sys/kernel/debug/nvme0/fault_inject/ 2>/dev/null
echo 100 | sudo tee /sys/kernel/debug/nvme0/fault_inject/probability 2>/dev/null
echo 10  | sudo tee /sys/kernel/debug/nvme0/fault_inject/times 2>/dev/null
sudo dd if=/dev/nvme0n1 of=/dev/null bs=4k count=50 iflag=direct 2>&1 | tail -2
dmesg | tail -10
echo 0 | sudo tee /sys/kernel/debug/nvme0/fault_inject/probability 2>/dev/null
```

Monitoring:

```sh
sudo nvme monitor 2>/dev/null &
# Or with smartd:
grep -i nvme /etc/smartd.conf
sudo smartctl -a /dev/nvme0 | head -30
```

---

### Lab 69.6 — Passthrough

```sh
# The commands nvme-cli issues are ioctls (T.10)
sudo strace -e ioctl nvme id-ctrl /dev/nvme0 2>&1 | grep -c NVME
sudo trace-cmd record -e nvme:nvme_setup_cmd -- sudo nvme smart-log /dev/nvme0 > /dev/null
sudo trace-cmd report | grep -oP 'cmd=\(\K\w+'
```

Write your own:

```c
// SPDX-License-Identifier: GPL-2.0
/* nvmepass.c -- issue NVMe commands directly (T.10). */
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ioctl.h>
#include <unistd.h>
#include <linux/nvme_ioctl.h>

#define NVME_ADMIN_IDENTIFY	0x06
#define NVME_ADMIN_GET_LOG_PAGE	0x02
#define NVME_CMD_READ		0x02
#define NVME_CMD_WRITE		0x01

static int identify(int fd, __u32 nsid, __u32 cns, void *data)
{
	struct nvme_passthru_cmd cmd = {
		.opcode   = NVME_ADMIN_IDENTIFY,
		.nsid     = nsid,
		.addr     = (__u64)(uintptr_t)data,
		.data_len = 4096,
		.cdw10    = cns,
	};

	return ioctl(fd, NVME_IOCTL_ADMIN_CMD, &cmd);
}

static int read_blocks(int fd, __u64 slba, __u16 nblocks, void *data,
		       size_t len)
{
	struct nvme_passthru_cmd cmd = {
		.opcode   = NVME_CMD_READ,
		.nsid     = 1,
		.addr     = (__u64)(uintptr_t)data,
		.data_len = len,
		.cdw10    = slba & 0xffffffff,
		.cdw11    = slba >> 32,
		.cdw12    = nblocks - 1,          /* zero-based (§1.3) */
	};

	return ioctl(fd, NVME_IOCTL_IO_CMD, &cmd);
}

static int smart_log(int fd, void *data)
{
	struct nvme_passthru_cmd cmd = {
		.opcode   = NVME_ADMIN_GET_LOG_PAGE,
		.nsid     = 0xffffffff,
		.addr     = (__u64)(uintptr_t)data,
		.data_len = 512,
		.cdw10    = 0x02 | (((512 / 4) - 1) << 16),   /* LID 2, NUMD */
	};

	return ouch(ioctl(fd, NVME_IOCTL_ADMIN_CMD, &cmd));
}

int main(int argc, char **argv)
{
	int fd = open(argv[1], O_RDONLY);
	void *buf;
	unsigned char *b;

	if (fd < 0) { perror("open"); return 1; }
	if (posix_memalign(&buf, 4096, 4096)) return 1;
	b = buf;

	/* Identify Controller (CNS=1) */
	memset(buf, 0, 4096);
	if (identify(fd, 0, 1, buf)) { perror("identify"); return 1; }
	printf("VID:   0x%04x\n", b[0] | (b[1] << 8));
	printf("SN:    %.20s\n", b + 4);
	printf("MN:    %.40s\n", b + 24);
	printf("FR:    %.8s\n",  b + 64);
	printf("MDTS:  %u (max transfer = %u KiB)\n", b[77], (4 << b[77]));
	printf("VWC:   0x%02x %s\n", b[525],
	       (b[525] & 1) ? "(volatile write cache present)" : "");

	/* Read LBA 0 */
	memset(buf, 0, 4096);
	if (read_blocks(fd, 0, 8, buf, 4096) == 0) {
		printf("\nLBA 0 (first 64 bytes):\n");
		for (int i = 0; i < 64; i++) {
			printf("%02x ", b[i]);
			if ((i + 1) % 16 == 0) printf("\n");
		}
	}

	close(fd);
	free(buf);
	return 0;
}
```

```sh
gcc -O2 -o nvmepass nvmepass.c
sudo ./nvmepass /dev/nvme0
```

`io_uring` passthrough — §T.10(a):

```sh
sudo apt install -y liburing-dev
# fio supports it directly:
sudo fio --name=t --filename=/dev/ng0n1 --direct=1 --rw=randread --bs=4k \
         --iodepth=32 --ioengine=io_uring_cmd --cmd_type=nvme \
         --runtime=10 --time_based 2>/dev/null | grep -E 'IOPS|clat.*avg'

# Compare with the block path
sudo fio --name=t --filename=/dev/nvme0n1 --direct=1 --rw=randread --bs=4k \
         --iodepth=32 --ioengine=io_uring --runtime=10 --time_based \
         --readonly 2>/dev/null | grep -E 'IOPS|clat.*avg'
```

The generic character device:

```sh
ls /dev/ng*
sudo nvme id-ctrl /dev/ng0n1 | head -5
# Passthrough only -- no block device, no filesystem, no page cache.
```

---

### Lab 69.7 — NVMe over Fabrics

Set up a target and connect to it, on one machine.

```sh
sudo modprobe nvmet nvmet-tcp
sudo mount -t configfs none /sys/kernel/config 2>/dev/null

NQN=nqn.2024-01.io.lab:target1
cd /sys/kernel/config/nvmet/subsystems
sudo mkdir -p $NQN && cd $NQN
echo 1 | sudo tee attr_allow_any_host > /dev/null

# Back it with a file or a device
sudo dd if=/dev/zero of=/tmp/nvmet.img bs=1M count=1024 2>/dev/null
LOOP=$(sudo losetup -f --show /tmp/nvmet.img)

sudo mkdir -p namespaces/1 && cd namespaces/1
echo -n $LOOP | sudo tee device_path > /dev/null
echo 1 | sudo tee enable > /dev/null
cat device_nguid device_uuid

# The port
cd /sys/kernel/config/nvmet/ports
sudo mkdir -p 1 && cd 1
echo 127.0.0.1 | sudo tee addr_traddr > /dev/null
echo tcp       | sudo tee addr_trtype > /dev/null
echo 4420      | sudo tee addr_trsvcid > /dev/null
echo ipv4      | sudo tee addr_adrfam > /dev/null
sudo ln -sf /sys/kernel/config/nvmet/subsystems/$NQN subsystems/$NQN

ss -tlnp | grep 4420
```

Connect:

```sh
sudo modprobe nvme-tcp
sudo nvme discover -t tcp -a 127.0.0.1 -s 4420
sudo nvme connect -t tcp -a 127.0.0.1 -s 4420 -n $NQN
sudo nvme list
sudo nvme list-subsys
```

The same command set over a different transport (§T.7):

```sh
FABDEV=$(sudo nvme list | grep -i linux | tail -1 | awk '{print $1}')
sudo nvme id-ctrl $FABDEV -H | grep -iE 'subnqn|mn |sgls'
cat /sys/class/nvme/nvme1/transport 2>/dev/null
cat /sys/class/nvme/nvme1/address 2>/dev/null
```

Performance:

```sh
for dev in /dev/nvme0n1 $FABDEV; do
  echo "=== $dev ==="
  sudo fio --name=t --filename=$dev --direct=1 --rw=randread --bs=4k \
           --iodepth=32 --ioengine=libaio --runtime=8 --time_based \
           --readonly 2>/dev/null | grep -E 'IOPS|clat.*avg'
done
```

Tune the fabric:

```sh
sudo nvme disconnect -n $NQN
sudo nvme connect -t tcp -a 127.0.0.1 -s 4420 -n $NQN \
     -i 8 -Q 128 --keep-alive-tmo=5 --ctrl-loss-tmo=60
# -i: number of I/O queues; -Q: queue size
cat /sys/class/nvme/nvme1/queue_count
```

Watch the protocol:

```sh
sudo tcpdump -i lo -w /tmp/nvmetcp.pcap port 4420 -c 200 &
sudo dd if=$FABDEV of=/dev/null bs=4k count=20 iflag=direct 2>/dev/null
sleep 1; sudo pkill tcpdump
sudo tcpdump -r /tmp/nvmetcp.pcap 2>/dev/null | head -10
# Wireshark decodes NVMe/TCP if you have a recent version.
```

Compare with iSCSI, if available:

```sh
# Set up an iSCSI target on the same backing store and benchmark both.
# NVMe-oF/TCP typically achieves 2-4x the IOPS (T.7).
```

Cleanup:

```sh
sudo nvme disconnect -n $NQN
cd /sys/kernel/config/nvmet
sudo rm -f ports/1/subsystems/*
sudo rmdir ports/1
sudo rmdir subsystems/$NQN/namespaces/1
sudo rmdir subsystems/$NQN
sudo losetup -d $LOOP
```

---

### Lab 69.8 — Zoned namespaces

Without ZNS hardware, emulate it:

```sh
sudo modprobe null_blk nr_devices=0
cd /sys/kernel/config/nullb
sudo mkdir zns0 && cd zns0
echo 1    | sudo tee zoned > /dev/null
echo 64   | sudo tee zone_size > /dev/null        # 64 MiB zones
echo 0    | sudo tee zone_nr_conv > /dev/null
echo 14   | sudo tee zone_max_open > /dev/null
echo 14   | sudo tee zone_max_active > /dev/null
echo 4096 | sudo tee size > /dev/null
echo 1    | sudo tee memory_backed > /dev/null
echo 1    | sudo tee power > /dev/null
cd /

ZDEV=/dev/nullb0
sudo blkzone report $ZDEV | head -6
cat /sys/block/nullb0/queue/{zoned,nr_zones,chunk_sectors,max_open_zones,max_active_zones}
```

Real ZNS hardware:

```sh
sudo nvme list | grep -i zns
sudo nvme zns id-ctrl /dev/nvme0 2>/dev/null
sudo nvme zns id-ns /dev/nvme0n1 -H 2>/dev/null
sudo nvme zns report-zones /dev/nvme0n1 -d 5 2>/dev/null
```

The sequential-write constraint:

```sh
sudo blkzone report -o 0 -c 1 $ZDEV

sudo dd if=/dev/zero of=$ZDEV bs=1M count=10 oflag=direct 2>&1 | tail -1
sudo blkzone report -o 0 -c 1 $ZDEV        # wptr advanced

# Writing behind the pointer: REFUSED
sudo dd if=/dev/zero of=$ZDEV bs=1M count=1 seek=2 oflag=direct 2>&1 | tail -2

sudo blkzone reset -o 0 -c 1 $ZDEV
sudo blkzone report -o 0 -c 1 $ZDEV        # wptr back to 0
```

Zone management:

```sh
sudo blkzone open   -o 0 -c 1 $ZDEV
sudo blkzone report -o 0 -c 1 $ZDEV
sudo blkzone close  -o 0 -c 1 $ZDEV
sudo blkzone finish -o 0 -c 1 $ZDEV
sudo blkzone report -o 0 -c 1 $ZDEV        # FULL
sudo blkzone reset  -o 0 -c 1 $ZDEV
```

Which filesystems work (Ch. 60 §T.5):

```sh
for fs in ext4 xfs btrfs f2fs; do
  echo -n "$fs: "
  sudo mkfs.$fs -f $ZDEV > /dev/null 2>&1 || sudo mkfs.$fs -F $ZDEV > /dev/null 2>&1
  [ $? -eq 0 ] && echo "OK" || echo "REFUSED"
done

sudo mkfs.f2fs -f -m $ZDEV 2>&1 | tail -3
sudo mkdir -p /mnt/zns && sudo mount $ZDEV /mnt/zns
sudo dd if=/dev/urandom of=/mnt/zns/data bs=1M count=500 2>/dev/null
sync
sudo blkzone report $ZDEV | head -8
df -h /mnt/zns
```

Zone Append — §T.9:

```sh
sudo umount /mnt/zns
sudo fio --name=zapp --filename=$ZDEV --direct=1 --zonemode=zbd \
         --rw=write --bs=64k --iodepth=8 --ioengine=libaio \
         --max_open_zones=8 --runtime=10 --time_based 2>/dev/null | \
  grep -E 'IOPS|BW='

sudo bpftrace -e '
tracepoint:block:block_rq_issue /strcontains(str(args->rwbs), "ZA")/ {
	@zone_append = count();
}
tracepoint:block:block_rq_issue { @all = count(); }
interval:s:10 { print(@zone_append); print(@all); exit(); }' &
sudo fio --name=t --filename=$ZDEV --direct=1 --zonemode=zbd --rw=write \
         --bs=64k --iodepth=4 --runtime=8 --time_based > /dev/null 2>&1
wait
```

The open-zone limit — the constraint that shapes host design:

```sh
cat /sys/block/nullb0/queue/max_open_zones
# Try to open more than the limit:
for z in $(seq 0 20); do
  sudo blkzone open -o $((z * 131072)) -c 1 $ZDEV 2>&1 | tail -1
done
sudo blkzone report $ZDEV | grep -c 'zcond: 2\|zcond: 3'   # open zones
```

---

## 3. Mastery drills

1. Enumerate AHCI's six structural problems (§T.1). For each, state whether adding queues to AHCI would have fixed it, and why NVMe's design does.

2. Explain the phase tag mechanism. Then compute the cost of the alternative (reading a device register to check for completions) at 1M IOPS.

3. `dma_rmb()` sits between reading a CQE and checking the next phase tag. Construct the corruption that occurs without it, on a machine with a weakly-ordered memory model.

4. Explain PRPs' page-alignment constraint and derive `virt_boundary_mask` from it. Then construct a user buffer that must be split under PRPs but not under SGLs.

5. Why does `sgl_threshold` exist? Compute the descriptor count for a 1 MiB transfer of 4 KiB fragments under both PRPs and SGLs.

6. The admin/IO queue split is a design decision, not an accident. State three things it enables, and identify the SCSI problem it avoids.

7. NVMe's `length` field is blocks-minus-one. Compute the maximum transfer size this permits, and state what happens if a driver forgets the bias.

8. Compare NVMe's error reporting with SCSI's. Count the round trips for a failed read in each, and state what the DNR bit replaces.

9. Interrupt coalescing hurts at low queue depth and barely helps at high depth. Derive both results, and state when it would be worth enabling.

10. NVMe multipath is in the protocol; SCSI's is bolted on. Enumerate everything this simplifies, and identify what is lost.

11. ZNS removes the FTL. For each of the four things it removes, state what the host must now do and what information the host has that the device did not.

12. `io_uring` passthrough has closed most of the gap to SPDK. Enumerate what SPDK still bypasses, and state the workload where it would still be worth the operational cost.

13. You are given an NVMe device rated for 1M IOPS that delivers 150K under your workload. Give the ordered diagnostic procedure using this chapter's tools and Chapters 63–65's, and the seven most likely causes.

---

## 4. Further reading

**Specifications**

The NVMe specifications are freely available from `nvmexpress.org` and are **unusually well written** — clearer than most kernel documentation:

- **NVM Express Base Specification** ★★★ — the queue model, admin commands, error handling. Chapters 3 (queues) and 5 (admin commands) are the core.
- **NVM Command Set Specification** ★★★ — Read, Write, Flush, Dataset Management, Write Zeroes.
- **Zoned Namespace Command Set Specification** ★★★ — §T.9.
- **NVMe over Fabrics Specification** ★★★ — §T.7.
- **NVMe Management Interface (NVMe-MI)** — out-of-band management.
- **Key Value Command Set** — the KV namespace type.

Reading the Base Specification's queue chapter alongside `drivers/nvme/host/pci.c` is the fastest way to understand both.

**Kernel documentation**

- `Documentation/nvme/` — feature-specific notes.
- `Documentation/block/zoned.rst` ★★★ — Ch. 60 §T.5.
- `Documentation/ABI/stable/sysfs-block` and `sysfs-class-nvme`
- `include/linux/nvme.h` ★★★ — **the protocol, transcribed.** Often clearer than the specification for finding a specific field.
- `man 1 nvme` and the per-subcommand pages ★★★ — `nvme-id-ctrl`, `nvme-id-ns`, `nvme-smart-log`, `nvme-connect`, `nvme-zns`.

**Papers and presentations**

- Bjørling, Gonzalez, Bonnet, "LightNVM: The Linux Open-Channel SSD Subsystem," FAST 2017 — the predecessor to ZNS; useful for understanding why ZNS took the shape it did.
- Bjørling et al., "ZNS: Avoiding the Block Interface Tax for Flash-based SSDs," USENIX ATC 2021 ★★★ — **§T.9's argument, with measurements.**
- Yang, Minturn, Hady, "When Poll is Better than Interrupt," FAST 2012 ★★★ — §T.6's polling case.
- Koo et al., "Modernizing File System through In-Storage Indexing," OSDI 2021 — computational storage, the next step.
- Didona et al., "Understanding Modern Storage APIs: A systematic study of libaio, SPDK, and io_uring," SYSTOR 2022 ★★★ — **§T.10's comparison, rigorously.** Essential if you are choosing between them.
- Jens Axboe's io_uring papers and presentations ★★★
- Intel/Samsung NVMe architecture presentations from the Storage Developer Conference (SNIA) — many are excellent and freely available.

**LWN**

- "NVMe: the new storage interface" and the original merge coverage ★★★
- "Zoned namespaces" and the ZNS series ★★★
- "NVMe passthrough and io_uring" ★★★ — §T.10(a)
- "Multipathing for NVMe" ★★★ — the native-versus-dm debate, including the vendor pushback
- "The removal of hybrid polling" (2023) — §T.6's lesson
- "NVMe over TCP"
- "Block-layer support for zone append"
- "Persistent memory and NVMe" discussions
- "Discard, TRIM, and deallocate" — the Dataset Management command's semantics

**Source reading order**

1. The NVMe Base Specification, chapter 3 (queues). Genuinely readable; start here.
2. `include/linux/nvme.h` ★★★ — the structures.
3. `drivers/nvme/host/pci.c` ★★★ — **read the whole file.** `nvme_queue_rq`, `nvme_poll_cq`, `nvme_handle_cqe`, `nvme_setup_prps`, `nvme_timeout`, `nvme_setup_io_queues`. ~3500 lines and the clearest complete blk-mq driver in the tree.
4. `drivers/nvme/host/core.c`: `nvme_setup_cmd`, `nvme_complete_rq`, `nvme_decide_disposition` ★★★, `nvme_init_identify`.
5. `drivers/nvme/host/multipath.c` — §T.8; small and clear.
6. `drivers/nvme/host/zns.c` — §T.9.
7. `drivers/nvme/host/ioctl.c` — §T.10, including `nvme_uring_cmd`.
8. `drivers/nvme/host/tcp.c` — §T.7; shows how the same core works over a socket.
9. `drivers/nvme/target/` — the target side, if you need it.

**Tools**

- `nvme-cli` ★★★ — **learn it thoroughly.** `id-ctrl -H`, `id-ns -H`, `smart-log`, `error-log`, `get-feature`, `get-log`, `zns report-zones`, `ana-log`, `connect`/`discover`, `format`, `sanitize`.
- `fio` ★★★ — `--ioengine=io_uring --hipri`, `--ioengine=io_uring_cmd --cmd_type=nvme`, `--zonemode=zbd`
- `trace-cmd record -e nvme:\*` ★★★ — the decoded-command tracepoints
- `/sys/kernel/debug/block/nvme0n1/` ★★★ — per-hctx state (Ch. 63)
- `null_blk` with `zoned=1` ★★★ — experiment with ZNS without hardware
- `nvmetcli` and configfs — §T.7's target
- `blkzone` and `libzbd` ★★★
- `lspci -vvv` — **always check `LnkSta` against `LnkCap`**
- `bpftrace` on `nvme:nvme_setup_cmd` / `nvme_complete_rq` for latency histograms
- `SPDK`'s `perf` tool — for the upper bound on what the hardware can do

---

→ Next: [70-mtd-flash.md](70-mtd-flash.md)

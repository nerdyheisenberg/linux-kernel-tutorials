# Chapter 46 — Network device drivers: netdev, NAPI, XDP, ethtool

> **Goal:** Understand why a network interface is *not* a character device, why the driver model here is push-based in both directions rather than request/response, how NAPI resolved the interrupt-versus-polling dilemma with a hybrid that is provably better than either, how `sk_buff` became both the subsystem's greatest asset and its greatest cost, why XDP exists as a second datapath below the stack, and why `ethtool` is the largest configuration surface of any driver class. By the end you can write a working NIC driver's datapath from memory, reason about ring buffers and DMA lifetime without hand-waving, and diagnose packet loss to the exact counter that explains it.

> **Note on scope.** This is the *driver* chapter. Part 4 (Ch. 71–76) covers the stack itself: sockets, TCP, qdiscs, netfilter, eBPF/XDP in depth. Here we build everything from the wire up to `netif_receive_skb()` and back down from `ndo_start_xmit()`.

---

## Theory & First Principles

### T.0 — Start here: 14.88 million packets per second

A 10 Gbps link, minimum-size (64-byte) frames. Do the arithmetic, because it dictates
everything:

```
  64-byte frame + 20 bytes of preamble/gap = 84 bytes = 672 bits
  10,000,000,000 / 672  =  14,880,952 packets/sec

  1 second / 14.88M  =  67 NANOSECONDS PER PACKET.
```

Now price the obvious implementation against that budget (Ch. 00 §T.3):

| Per packet | Cost | Budget used |
|---|---|---|
| One interrupt | ~1–5 µs | **7400%** — impossible by a factor of 70 |
| One `sk_buff` allocation | ~100 ns | **150%** — impossible on its own |
| One `read()` syscall | ~300 ns | 450% |
| One cache miss | ~80 ns | 120% |

**Every naive choice is off by one to two orders of magnitude.** A NIC driver is not "a
character device for packets" — it lives in a regime where a single cache miss per packet is
unaffordable, and the entire architecture follows from that.

**What the budget forces, in order:**

**1. Stop interrupting. (NAPI.)** Interrupt on the first packet, then *disable the interrupt*
and poll until the ring drains. Interrupt-driven at low load (low latency), polled at high
load (high throughput), switching automatically. This is not an optimization — without it you
get **receive livelock**: past a threshold, interrupt processing consumes all CPU and
*preempts the very code that would drain the queue*, so goodput falls to **zero** as offered
load rises (Mogul & Ramakrishnan, 1997).

**2. Stop allocating per packet.** DMA rings are preallocated at open time; buffers are
recycled, not freed. Page pools amortize what remains.

**3. Stop using one queue.** One ring means one lock and one interrupt — a global
serialization point. Multi-queue NICs give **one ring, one MSI-X vector, one CPU** per queue
(Ch. 38 §T.0), with RSS hashing flows across them so packets of one connection stay on one
CPU:

```
   packet -> [RSS hash on the 5-tuple] -> queue 3 -> MSI-X vector 3 -> CPU 3
                                                       ↑
              no shared state, no lock, no cacheline bouncing (Ch. 105)
```

**4. Push work into hardware.** Checksum offload, TSO/GSO (hand the NIC a 64 KiB buffer and
let it emit 45 packets), GRO (coalesce received packets *before* the stack sees them). The
stack then processes one large object instead of 45 small ones, and the per-packet cost is
amortized ~45×.

**So the driver's actual job** is not "send and receive." It is:

```
  - manage DMA descriptor rings (Ch. 35), with correct ownership and barriers
  - implement NAPI poll with a budget
  - map queues to CPUs and interrupts
  - advertise offload capabilities honestly
  - never allocate on the fast path
  - keep per-queue state per-CPU (Ch. 16)
```

**And the thing to carry forward:** this is the first chapter where the *cost model* fully
determines the architecture, with no room for taste. Ch. 63 (blk-mq) and Ch. 76 (`io_uring`)
reach the same conclusions from the same arithmetic — **when the per-operation budget drops
below the cost of a syscall or an allocation, the design space collapses to one shape.**

```bash
ethtool -l eth0                 # how many queues
ethtool -k eth0 | head -20      # offloads
ethtool -S eth0 | head -20      # per-queue stats, drops
cat /proc/interrupts | grep eth0        # one line per queue?
cat /proc/net/softnet_stat              # col 3 = times the NAPI budget was exhausted
```

---

### T.1 Why a NIC is not a character device

Chapter 29 argued that "everything is a file" is an economic reuse argument, and it holds remarkably well — until you reach networking, where Linux deliberately declines to use it. There is no `/dev/eth0`. Understanding *why* is the entry point to this entire subsystem.

A character device is **pull on receive**: data exists in the device, userspace calls `read()`, the driver delivers. A network interface is **push in both directions**:

| | Char device | Network interface |
|---|---|---|
| Who initiates receive | userspace (`read()`) | **the wire** — packets arrive unsolicited |
| Can you decline data? | yes, just don't read | **no** — refusing means dropping, which is a protocol event |
| Is there a "current position"? | yes | no; there is no stream, only framed units |
| Who is the consumer? | the process that opened it | the **stack**, which may fan out to many sockets, or to a bridge, or to nobody |
| Unit of work | bytes | **frames**, indivisible, with deadlines |
| Multiplexing | one fd per opener | one interface, thousands of flows |

The decisive line is the third and fourth. A packet arriving on the wire is not addressed to a file descriptor; it is addressed to a *protocol endpoint* that the driver cannot identify without parsing headers it has no business parsing. So there must be a demultiplexer between the device and userspace, and that demultiplexer — the network stack — is the driver's real client. The driver's interface is therefore **to the stack, not to userspace**, and it looks nothing like `file_operations`.

The result is `struct net_device` and `struct net_device_ops`: a driver registers an interface and the stack calls it. The user-visible names (`eth0`, `enp3s0`) live in a *namespace* (`struct net`), not in a filesystem, which is what makes network namespaces (Ch. 71) possible in a way that would be very awkward if interfaces were files.

> **The general lesson:** "everything is a file" works when the device has a stream or a seekable store and one consumer. It fails when the device produces unsolicited, framed, multi-consumer data with hard timing. Recognising which situation you are in is a design skill, not a rule lookup.

### T.2 The interrupt/polling dilemma, and NAPI as its resolution

This is the single most important piece of theory in the chapter, and it has a named paper behind it.

**Pure interrupt-driven receive.** Each frame raises an IRQ; the handler moves it to the stack. Latency is minimal at low rates. But as the arrival rate $\lambda$ grows, per-packet interrupt overhead $c_{\text{irq}}$ (on the order of 1–5 µs including cache pollution and pipeline effects) dominates, and beyond $\lambda > 1/c_{\text{irq}}$ the CPU does nothing but take interrupts. Worse: interrupt handlers preempt the softirq/process context that would *consume* the queued packets, so throughput does not plateau — **it collapses to zero**. This is **receive livelock**, formalised by Mogul & Ramakrishnan (ACM TOCS, 1997), and it is the same failure we met in Ch. 17 §T.1.

**Pure polling.** The CPU checks the ring on a timer. Overhead is bounded and independent of $\lambda$, so no livelock. But at low rates you burn CPU checking an empty ring, and latency is bounded below by the poll period.

Each is optimal in exactly the regime where the other fails. NAPI ("New API", Jamal Hadi Salim, Robert Olsson, Alexey Kuznetsov, 2001–2003) is the hybrid:

```
IRQ fires
  └─ disable device RX interrupts
  └─ napi_schedule()                     /* raise NET_RX_SOFTIRQ */
        ...
NET_RX_SOFTIRQ runs poll(napi, budget)
  └─ process up to `budget` packets from the ring
  ├─ if we used the whole budget: return budget   /* stay in polled mode */
  └─ if the ring went empty: napi_complete_done() /* re-enable IRQs */
                                                   /* back to interrupt mode */
```

The properties this achieves are worth stating precisely, because they are why every high-performance I/O subsystem since has copied the shape (blk-mq, io_uring completion, NVMe):

1. **Interrupt coalescing is automatic and load-adaptive.** Under load, one interrupt amortises over up to `budget` packets. The coalescing ratio rises with load *with no configuration*. At low load, you get one interrupt per packet — minimum latency.
2. **Livelock is structurally impossible.** While polling, receive interrupts are off, so the polling context cannot be preempted by more arrivals. The consumer always makes progress.
3. **Fairness across devices.** `net_rx_action()` runs a round-robin over the poll list with a global budget (`netdev_budget`, default 300) and a time limit (`netdev_budget_usecs`, default 2000 µs). No single device can starve others — and, critically, the softirq loop itself yields to `ksoftirqd` when it exceeds these, so networking cannot starve the *scheduler* (Ch. 18 §T.1).
4. **Backpressure becomes a drop at the NIC**, which is the correct place: the ring fills, the NIC drops and counts it, and the counter tells you exactly what happened.

The budget is the tuning knob that trades latency for throughput, and the fact that it is *per-poll* rather than *per-second* is what makes it self-regulating.

Modern refinements, each solving a residual problem:

| Mechanism | Problem solved |
|---|---|
| `NAPI_POLL_WEIGHT` (64) per device | bounds one device's share within the global budget |
| **Busy polling** (`SO_BUSY_POLL`, `napi_busy_loop`) | softirq scheduling latency matters for µs-scale RPC; let the *application* poll the ring directly |
| **Deferred IRQ / `gro_flush_timeout`, `napi_defer_hard_irqs`** | after a poll ends, delay re-enabling IRQs briefly so a burst is caught by another poll instead of an interrupt |
| **Threaded NAPI** (`/sys/class/net/*/threaded`, 5.12) | run `poll()` in a kthread instead of softirq, so it is schedulable, can be pinned, and is visible to the scheduler and to PSI |

Threaded NAPI is a notable admission: softirq's "runs in whatever context it lands in" (Ch. 18) is a poor fit for a workload that can consume an entire core, and making it a thread gives it a place in the scheduler's model.

### T.3 Rings and descriptors: the shared-memory contract

A NIC does not receive packets into arbitrary memory. Driver and device share **descriptor rings** in DMA-coherent memory (Ch. 35):

```
   RX ring (N descriptors, power of 2)
   ┌────┬────┬────┬────┬────┬────┬────┬────┐
   │ D0 │ D1 │ D2 │ D3 │ D4 │ D5 │ D6 │ D7 │
   └────┴────┴────┴────┴────┴────┴────┴────┘
          ▲                   ▲
          │                   │
      next_to_clean       next_to_use
      (driver reads       (driver refills
       completed here)     buffers here)
```

Each descriptor holds a **DMA address of a data buffer**, a length, and status/ownership bits. The invariant is an ownership protocol exactly like Ch. 35 §T.4's move semantics:

> A descriptor is owned by either the driver or the device, never both. Ownership is transferred by writing an ownership bit (or by advancing a tail pointer), and **the transfer must be ordered after every write to the buffer it describes**.

That last clause is where drivers get it wrong, and it is a direct application of Ch. 13's memory model. The correct RX refill sequence is:

```c
	/* 1. allocate + DMA-map a buffer */
	/* 2. write the descriptor's address and length fields */
	dma_wmb();                     /* descriptor writes visible before ... */
	/* 3. set the OWN bit / advance tail */
	writel(tail, ring->tail_reg);  /* MMIO doorbell (Ch. 34) */
```

and the receive sequence:

```c
	status = READ_ONCE(desc->status);
	if (!(status & DESC_OWN_DRIVER))
		break;
	dma_rmb();                     /* do not read length/data before status */
	len = desc->len;
	dma_sync_single_for_cpu(dev, dma, len, DMA_FROM_DEVICE);
```

`dma_wmb()`/`dma_rmb()` are the cheap variants that order accesses to *coherent DMA memory* only — weaker and faster than `wmb()`/`rmb()`, which also order MMIO. Using the wrong one is either a correctness bug (too weak) or a measurable throughput loss (too strong, on a path executed 10 million times per second).

Two more design facts follow from the ring structure:

- **Rings are power-of-two sized** so the wrap is a mask, not a branch or a division — Ch. 10 §T.4's kfifo argument, in the hottest loop in the kernel.
- **The ring size is a latency/loss trade.** A larger ring absorbs bigger bursts (fewer drops) but adds queuing delay and holds more memory. This is one face of **bufferbloat**: the naive "make the buffer bigger" fix converts loss into latency, and for TCP, loss is a *signal* while latency is a *cost*. The modern answer is small rings plus BQL (§T.6).

### T.4 `sk_buff`: the universal packet container, and what it costs

`struct sk_buff` is the packet representation from the driver to the socket. It carries the data pointers, the protocol metadata, the routing decision, the checksum state, the timestamps, the netfilter state, and the destructor. It is ~232 bytes on x86-64, plus `skb_shared_info` at the end of the data buffer.

Its central mechanism is the **headroom/tailroom pointer set**:

```
    head        data            tail             end
     │           │               │                │
     ▼           ▼               ▼                ▼
     ┌───────────┬───────────────┬────────────────┐
     │ headroom  │  packet data  │    tailroom    │
     └───────────┴───────────────┴────────────────┘
        skb_push/pull moves `data`;  skb_put/trim moves `tail`
```

This is what makes protocol layering cheap: adding an IP header is `skb_push(skb, 20)` — a pointer decrement, no copy — provided the driver reserved headroom. Hence `NET_SKB_PAD` (64 bytes) and the universal driver idiom `skb_reserve(skb, NET_IP_ALIGN)`.

`NET_IP_ALIGN` (2 on most architectures) deserves a paragraph because it looks like a hack and is actually a considered trade. Ethernet headers are 14 bytes, so a DMA-aligned buffer puts the IP header at offset 14 — misaligned by 2 for 32-bit fields. On architectures that trap unaligned access, every IP header field read is a fault. Reserving 2 bytes makes the IP header 4-byte aligned at the cost of making the *DMA write* misaligned, which modern x86 doesn't care about (hence `NET_IP_ALIGN == 0` there) but which matters on ARM. The macro encodes a per-architecture answer to "which misalignment is cheaper."

The costs of `sk_buff` are real and much-discussed:

| Cost | Consequence |
|---|---|
| Large and metadata-heavy | allocating+initialising one is a significant fraction of small-packet cost |
| Allocated per packet | at 14.88 Mpps (10 GbE line rate, 64 B frames) that is 14.88 M allocations/sec |
| Cache-unfriendly | the struct spans several cache lines and nearly all are touched |
| Coupled to the whole stack | you cannot use a lighter container without bypassing the stack |

Drivers mitigate with:

- **Page-based buffers and page pool** (`include/net/page_pool.h`) — recycle DMA-mapped pages instead of alloc/map per packet. `page_pool` keeps pages DMA-mapped across uses, which removes the IOMMU map cost (Ch. 36) from the hot path. This is now the standard for new drivers.
- **`build_skb()`** — receive into a page, then wrap an `sk_buff` around it without copying.
- **Copybreak** — for small frames, copy into a fresh small skb and recycle the big buffer, because copying 128 bytes is cheaper than losing a 4 KiB page from the pool.
- **GRO** (§T.5) — amortise per-skb cost over many segments.

XDP (§T.7) exists precisely because for some workloads the right answer is "never build an skb at all."

### T.5 Offloads: moving work into silicon, and the correctness obligations

A modern NIC can do a surprising amount of protocol work. Each offload is a *contract* with specific obligations, and getting the contract wrong causes corruption that appears far away.

**Checksum offload.** Two independent directions:

- *RX:* the NIC verifies checksums and reports via `skb->ip_summed`:
  - `CHECKSUM_UNNECESSARY` — "I verified it; trust me." Requires the NIC to actually understand the protocol.
  - `CHECKSUM_COMPLETE` — "here is the 16-bit ones-complement sum of the whole frame from offset X." Strictly better, because the stack can *adjust* it as headers are pulled, making it valid for encapsulated protocols the NIC has never heard of. This is a lovely piece of design: an offload that composes with unknown future protocols.
  - `CHECKSUM_NONE` — the stack does it.
- *TX:* `CHECKSUM_PARTIAL` means "the pseudo-header sum is already in place at `skb->csum_offset`; NIC, finish it." The driver must tell the NIC where via `skb_checksum_start_offset()`.

**Segmentation offload (TSO/GSO).** The stack hands down a 64 KiB "super-packet"; the NIC (TSO) or the stack just before the driver (GSO) splits it into MSS-sized segments, replicating and adjusting headers. The win is enormous — per-packet stack cost is amortised over ~45 segments — and it is why `GSO` exists as a software fallback: the *stack* gets the benefit of large packets even when the hardware cannot segment, deferring the split to the last possible moment.

**Receive aggregation (GRO/LRO).** The inverse: merge consecutive segments of one flow into one large skb before the stack sees them. The critical distinction:

> **LRO is lossy; GRO is not.** LRO (hardware) may merge packets in ways that cannot be undone, which breaks forwarding — you cannot re-emit what you received. GRO defines strict merge criteria such that `skb_gso_segment()` reproduces the original packets exactly. That is why LRO must be disabled when forwarding and GRO need not be.

GRO's reversibility requirement is a good example of a *design constraint derived from a use case*: because Linux boxes route, any receive-side aggregation must be invertible.

**RSS / RPS / RFS / XPS** — steering, and the theory is queueing:

| Mechanism | Where | What it does |
|---|---|---|
| **RSS** | hardware | hash the flow tuple, index an indirection table, choose an RX queue → one queue per CPU |
| **RPS** | software | same idea when hardware cannot; hash in the driver, IPI the target CPU |
| **RFS** | software | steer to the CPU where the *application* is running, using a flow→CPU table updated by `recvmsg` |
| **aRFS** | hardware | program the NIC's flow steering to do what RFS does |
| **XPS** | software TX | choose the TX queue based on the sending CPU, so completion interrupts land locally |

The unifying goal is **flow-to-core affinity**, and the reason is not load balancing — it is that a TCP connection's state (socket, congestion control, receive queue) is hot in exactly one core's cache, and processing its packets anywhere else costs cache misses *plus* lock contention on the socket. RFS is the interesting one because it optimises for the *consumer's* location rather than the producer's, which is the correct objective and requires information only the socket layer has.

A crucial invariant: **per-flow ordering must be preserved.** All these mechanisms hash on the flow tuple precisely so that a given flow always lands in one queue on one core. Reordering within a TCP flow triggers spurious retransmits and wrecks throughput — so "spread packets over cores" must never mean "spread a flow over cores."

### T.6 Transmit: queueing, backpressure, and completion

The TX path has a different shape from RX because the driver is the *consumer*, not the producer:

```
   stack → qdisc (Ch. 74) → ndo_start_xmit() → TX ring → NIC → wire
                                                  │
                                                  └─ TX completion IRQ → free skbs
```

Three obligations that define a correct TX path:

**(a) `ndo_start_xmit()` must never block and must not fail for "ring full".** Its return values are `NETDEV_TX_OK` and `NETDEV_TX_BUSY`, and `NETDEV_TX_BUSY` is *strongly discouraged* — it makes the qdic requeue, which costs more than avoiding the situation. Instead the driver must implement **flow control against itself**:

```c
	/* after enqueueing: if we can't fit another worst-case packet, stop. */
	if (unlikely(tx_ring_space(ring) < MAX_SKB_FRAGS + 2))
		netif_stop_queue(ndev);

	/* in the completion handler, after freeing: */
	if (netif_queue_stopped(ndev) && tx_ring_space(ring) >= THRESH)
		netif_wake_queue(ndev);
```

And the classic race: between the space check and `netif_stop_queue()`, the completion handler may free everything and check "is the queue stopped?" — finding it not yet stopped — leaving the queue stopped forever with a full ring. The standard fix is the `smp_mb()` + re-check idiom, or `netif_txq_maybe_stop()` / `__netif_txq_completed_wake()` helpers (5.17+) that encapsulate it. This is literally Ch. 25's P3 wait/wakeup pattern (an SB litmus test) in the busiest code in the kernel, and every driver that hand-rolls it eventually gets it wrong — which is why the helpers now exist.

**(b) The skb must be freed only after the NIC has read it.** `dev_kfree_skb_any()` in the completion path, never in `ndo_start_xmit()` after a successful enqueue. Freeing early is a use-after-free that the device performs via DMA — undetectable by KASAN, visible only as corrupted packets on the wire.

**(c) Byte Queue Limits (BQL).** A TX ring of 4096 descriptors at 1 Gbps holds up to 49 ms of data. TCP's control loop sees that as RTT, and its congestion window inflates to fill it: **bufferbloat**. BQL (Tom Herbert, 2011) bounds the ring not by descriptors but by **bytes**, with a limit adapted at runtime to just cover the completion latency:

```c
	netdev_tx_sent_queue(txq, skb->len);        /* in xmit */
	netdev_tx_completed_queue(txq, pkts, bytes); /* in completion */
```

Two function calls, and the driver gets automatic, self-tuning queue-depth control that keeps the ring just full enough to never starve the link and no fuller. It is one of the highest value-per-line changes a driver can make, and omitting it is a real bug in a modern driver.

The deeper principle, worth carrying beyond networking:

> **A queue's correct depth is determined by the service-time distribution, not by the memory you can afford.** Sizing buffers by available memory converts throughput problems into latency problems and hides them.

### T.7 XDP: a second datapath below the stack

Some workloads — DDoS filtering, L4 load balancing, packet forwarding — need to make a decision about a packet using a few header fields and then drop, forward, or redirect it. Running such a packet through `sk_buff` allocation, the routing lookup, and netfilter costs orders of magnitude more than the decision itself.

**XDP (eXpress Data Path)** puts an eBPF program at the earliest possible point: in the driver's receive routine, on the raw DMA buffer, *before* any `sk_buff` exists.

```c
	/* inside the NAPI poll, per packet: */
	xdp_init_buff(&xdp, frame_sz, &rq->xdp_rxq);
	xdp_prepare_buff(&xdp, hard_start, headroom, len, true);
	act = bpf_prog_run_xdp(prog, &xdp);
	switch (act) {
	case XDP_PASS:     break;                 /* build skb, continue normally */
	case XDP_DROP:     recycle_buffer(); continue;
	case XDP_TX:       transmit_back_out(&xdp); continue;
	case XDP_REDIRECT: xdp_do_redirect(dev, &xdp, prog); continue;
	case XDP_ABORTED:  trace_xdp_exception(...); recycle_buffer(); continue;
	}
```

The performance is startling — `XDP_DROP` at ~25–100 Mpps per core — and the reason is entirely structural: no allocation, no metadata initialisation, no per-packet cache misses beyond the header itself.

The design constraints XDP accepts in exchange are the interesting part:

| Constraint | Why |
|---|---|
| Runs in the driver, so **every driver must add support** | there is no generic fast path; `XDP_GENERIC` exists as a slow fallback for testing only |
| Buffer must be **one page, writable, with 256 B headroom** | so the program can `bpf_xdp_adjust_head()` to add/remove encapsulation |
| No access to stack state (sockets, routes, conntrack) | it runs before any of that exists; use BPF maps or `bpf_fib_lookup()` |
| Multi-buffer (jumbo/GRO) support came later and is opt-in | the single-page assumption was baked into the original ABI |
| Program is verified, bounded, non-sleeping | it runs in the NAPI poll; an unbounded program would be a livelock |

`AF_XDP` builds on this: `XDP_REDIRECT` into a userspace-mapped UMEM ring gives a zero-copy kernel-bypass socket *without* taking the NIC away from the kernel (unlike DPDK). That combination — bypass for the flows you choose, normal stack for everything else — is why AF_XDP largely displaced full-bypass frameworks for general use.

For the driver author the obligations are: allocate RX buffers as single writable pages with headroom (page_pool makes this natural), register an `xdp_rxq_info` per queue, run the program in the poll loop, and implement `ndo_xdp_xmit()` for redirect targets.

### T.8 `ethtool`: why one driver class has a hundred configuration operations

`struct ethtool_ops` has over 80 members. No other driver class has anything comparable. The reason is not sprawl; it is that a NIC has more *legitimately configurable, standardised* state than any other device:

| Category | Examples |
|---|---|
| Link | speed, duplex, autoneg, media type, forced modes, link mode bitmaps |
| Ring/queue | ring sizes, channel counts, queue-to-IRQ mapping |
| Coalescing | rx-usecs, rx-frames, adaptive coalescing |
| Offloads | ~30 feature flags via `netdev_features_t` |
| Flow control | pause frames, PFC |
| Steering | RSS hash key, indirection table, ntuple filters |
| Diagnostics | self-test, register dump, EEPROM, SFP module info, cable diagnostics |
| Statistics | arbitrary driver-defined named counters |
| Time | PTP/PHC capabilities, timestamping modes |
| Power | WoL, EEE |
| Firmware | version reporting, flashing |

The important architectural detail: **`ethtool` is a stable ABI, and it is now netlink-based.** The original ioctl interface (`SIOCETHTOOL`) used fixed structs and hit the extensibility wall of Ch. 24 §T.3 — every new field needed a new command. The netlink interface (`ETHTOOL_MSG_*`, 5.6+) uses attributes, so new fields are additive and old tools keep working. New features go to netlink only. This is the chapter's clearest instance of "declarative, attribute-based interfaces outlive struct-based ones."

Two obligations that drivers routinely get wrong:

- **`ndo_get_stats64` counters must be monotonic and must mean what `Documentation/networking/statistics.rst` says.** `rx_dropped` vs `rx_missed_errors` vs `rx_fifo_errors` are *different events*, and conflating them makes the counters useless for exactly the diagnosis they exist for (Lab 46.6).
- **`netdev_features_t` has three fields**: `features` (what we can do), `hw_features` (what the user may toggle), and `wanted_features`. The core calls `ndo_fix_features()` to let the driver enforce dependencies (e.g. TSO requires TX checksum), then `ndo_set_features()` to apply. A driver that ignores the dependency logic produces a NIC that silently corrupts packets when a user disables checksum offload but leaves TSO on.

### T.9 The lifecycle: when is a netdev allowed to receive?

Registration order matters more here than in most subsystems because the interface becomes globally visible the instant it registers:

```
probe():
  alloc_etherdev_mq(sizeof(priv), n_tx, n_rx)   /* or alloc_netdev_mqs */
  SET_NETDEV_DEV(ndev, &pdev->dev)              /* parent for sysfs (Ch. 26) */
  ndev->netdev_ops  = &my_netdev_ops;
  ndev->ethtool_ops = &my_ethtool_ops;
  set features, MTU limits, MAC address
  netif_napi_add(ndev, &priv->napi, my_poll)
  register_netdev(ndev)          <-- VISIBLE TO USERSPACE FROM HERE
  
ndo_open():                       /* ip link set up */
  allocate rings, DMA buffers
  request_irq()
  napi_enable()
  netif_start_queue() / netif_tx_start_all_queues()
  phy_start()                     /* link may now come up */

ndo_stop():                       /* the exact reverse */
  netif_tx_disable()
  napi_disable()                  /* waits for in-flight polls */
  free_irq()
  free rings, unmap DMA
```

Three rules, each learned the hard way by the community:

1. **Allocate rings in `ndo_open`, not `probe`.** An interface that is administratively down should consume no DMA memory. On a machine with 64 interfaces this is the difference between gigabytes and nothing.
2. **`napi_disable()` before `free_irq()` before freeing buffers.** `napi_disable()` blocks until any running poll finishes and prevents new ones; this is Ch. 25 P12 (stop–drain–free) and getting the order wrong is a use-after-free in softirq context, which is about as bad as it gets.
3. **`register_netdev()` is the point of no return.** Between it and a complete `ndo_open`, userspace can already call every operation. Everything the ops touch must be initialised *before* registration.

`netif_carrier_on/off` is separate from admin up/down, and the separation matters: administrative state is policy (the user asked for it), carrier is physical fact (the cable). `ip link` shows both (`UP` vs `LOWER_UP`), and a driver that conflates them produces the classic "interface is up but nothing works, and no counter says why."

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `net/core/dev.c` | the core: `netif_receive_skb`, `net_rx_action`, `dev_queue_xmit`, registration |
| `net/core/skbuff.c` | `sk_buff` alloc/free/clone/fragment |
| `net/core/gro.c`, `net/core/gso.c` | aggregation and segmentation |
| `net/core/page_pool.c` | recycled DMA-mapped page allocator |
| `net/core/xdp.c` | XDP infrastructure, `xdp_do_redirect`, `xdp_frame` |
| `net/core/net-sysfs.c` | `/sys/class/net/*` |
| `net/ethtool/` | netlink ethtool (`ioctl.c` holds the legacy path) |
| `include/linux/netdevice.h` | `net_device`, `net_device_ops`, NAPI |
| `include/linux/skbuff.h` | the packet container |
| `include/net/xdp.h`, `include/net/page_pool/*.h` | driver-facing XDP/page-pool APIs |
| `drivers/net/ethernet/` | the drivers |
| `drivers/net/phy/` | PHY library, MDIO bus |
| `drivers/net/virtio_net.c` | **read this first** — complete, modern, no hardware needed |
| `Documentation/networking/` | extensive; see §4 |

### 1.2 `struct net_device_ops` — the mandatory core

```c
struct net_device_ops {
	int  (*ndo_init)(struct net_device *dev);
	void (*ndo_uninit)(struct net_device *dev);
	int  (*ndo_open)(struct net_device *dev);
	int  (*ndo_stop)(struct net_device *dev);
	netdev_tx_t (*ndo_start_xmit)(struct sk_buff *skb,
				      struct net_device *dev);
	u16  (*ndo_select_queue)(struct net_device *dev, struct sk_buff *skb,
				 struct net_device *sb_dev);
	void (*ndo_set_rx_mode)(struct net_device *dev);
	int  (*ndo_set_mac_address)(struct net_device *dev, void *addr);
	int  (*ndo_eth_ioctl)(struct net_device *dev, struct ifreq *ifr, int cmd);
	int  (*ndo_change_mtu)(struct net_device *dev, int new_mtu);
	void (*ndo_tx_timeout)(struct net_device *dev, unsigned int txqueue);
	void (*ndo_get_stats64)(struct net_device *dev,
				struct rtnl_link_stats64 *storage);
	int  (*ndo_bpf)(struct net_device *dev, struct netdev_bpf *bpf);
	int  (*ndo_xdp_xmit)(struct net_device *dev, int n,
			     struct xdp_frame **xdp, u32 flags);
	int  (*ndo_set_features)(struct net_device *dev,
				 netdev_features_t features);
	netdev_features_t (*ndo_fix_features)(struct net_device *dev,
					      netdev_features_t features);
	/* ... 60 more ... */
};
```

Only `ndo_open`, `ndo_stop`, and `ndo_start_xmit` are truly mandatory for an Ethernet device; `ether_setup()` fills in sensible defaults for the rest.

### 1.3 NAPI in `struct napi_struct`

```c
struct napi_struct {
	struct list_head	poll_list;
	unsigned long		state;      /* NAPI_STATE_SCHED, _NPSVC, ... */
	int			weight;
	int			(*poll)(struct napi_struct *, int budget);
	struct net_device	*dev;
	struct gro_node		gro;        /* GRO hash of in-progress flows */
	struct hrtimer		timer;      /* gro_flush_timeout */
	struct task_struct	*thread;    /* threaded NAPI */
};
```

`NAPI_STATE_SCHED` is the "exactly one poller" bit, set by `napi_schedule()` with an atomic test-and-set and cleared by `napi_complete_done()`. That single bit is what makes the whole scheme lock-free and race-free; read `napi_schedule_prep()` and `napi_complete_done()` together and you have the core of NAPI in 60 lines.

### 1.4 The receive path, end to end

```
NIC DMA writes frame into RX buffer, sets descriptor status, raises MSI-X (Ch. 38)
 └─ my_msix_rx_handler()
      napi_schedule_irqoff(&q->napi)            /* sets SCHED, adds to per-CPU list */
      → raise NET_RX_SOFTIRQ
 └─ net_rx_action()                              [softirq or ksoftirqd or NAPI thread]
      loop over poll_list within netdev_budget / netdev_budget_usecs
        my_poll(napi, 64)
          for each completed descriptor:
            dma_rmb()
            dma_sync_single_for_cpu()
            [XDP program, if attached]
            skb = build_skb(page_addr, truesize)
            skb_reserve(skb, headroom)
            skb_put(skb, len)
            skb->protocol = eth_type_trans(skb, ndev)
            skb->ip_summed = CHECKSUM_UNNECESSARY   /* if HW verified */
            napi_gro_receive(napi, skb)            /* → netif_receive_skb */
            refill descriptor with a fresh page
          if (done < budget && napi_complete_done(napi, done))
            re-enable RX interrupts
 └─ netif_receive_skb_core → ptype dispatch → ip_rcv → ... → socket
```

### 1.5 Where the counters live

```sh
/sys/class/net/eth0/statistics/*        # the rtnl_link_stats64 set
ip -s -s link show eth0                 # same, formatted, with detail
ethtool -S eth0                         # driver-private named counters
/proc/net/softnet_stat                  # per-CPU: processed, dropped, time_squeeze
```

`softnet_stat` column 1 is packets processed, column 2 is **dropped because the backlog was full** (RPS/`netif_rx` path), column 3 is `time_squeeze` — how often `net_rx_action` ran out of budget with work remaining. A rising column 3 means your budget is too small or one device is hogging; this is the counter almost nobody knows about and it answers real questions.

---

## 2. Practice

### Lab 46.1 — Observe NAPI, budget, and coalescing on a live system

```sh
IF=$(ip -o -4 route show default | awk '{print $5}')
ethtool -i  $IF        # driver, firmware, bus
ethtool     $IF        # link modes, speed, duplex
ethtool -g  $IF        # ring sizes: current vs max
ethtool -c  $IF        # interrupt coalescing
ethtool -l  $IF        # channel (queue) counts
ethtool -k  $IF        # offload features
ethtool -S  $IF | head -40
```

Now watch NAPI adapt:

```sh
# Baseline
cat /proc/net/softnet_stat | head -4
watch -d -n1 'grep -E "eth|enp|ens" /proc/interrupts'

# Generate load in another terminal
iperf3 -s &                    # on the other machine
iperf3 -c <peer> -P 8 -t 30

# During load, compare interrupts/sec to packets/sec
```

You should observe interrupts/second far below packets/second, and the *ratio rising with load* — §T.2's automatic coalescing, measured. Compute the coalescing factor at idle and at line rate.

Then:

```sh
cat /proc/sys/net/core/netdev_budget          # 300
cat /proc/sys/net/core/netdev_budget_usecs    # 2000
awk '{print $3}' /proc/net/softnet_stat       # time_squeeze per CPU
```

If `time_squeeze` climbs during your test, raise `netdev_budget` and measure the effect on throughput and on `ksoftirqd` CPU time. Explain the direction of both changes.

Finally, enable threaded NAPI and re-measure:

```sh
echo 1 | sudo tee /sys/class/net/$IF/threaded
ps -eo pid,comm,psr | grep napi
```

Now the poll loop is a schedulable thread you can pin and account. Compare `top` output and latency before/after.

---

### Lab 46.2 — A complete virtual network driver

A loopback-style netdev with a real NAPI receive path fed by a timer. It exercises every structural element of §T.9 without hardware.

`vnet.c`:

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>
#include <linux/netdevice.h>
#include <linux/etherdevice.h>
#include <linux/ip.h>
#include <linux/skbuff.h>
#include <linux/ethtool.h>

#define VNET_NAPI_WEIGHT	64
#define VNET_RX_QUEUE_LEN	256

struct vnet_priv {
	struct net_device	*ndev;
	struct napi_struct	napi;
	struct sk_buff_head	rx_queue;	/* our fake "ring" */
	struct u64_stats_sync	syncp;
	u64			rx_packets, rx_bytes;
	u64			tx_packets, tx_bytes;
	u64			rx_dropped_noskb;
};

/* ---------------- receive: NAPI poll ---------------- */

static int vnet_poll(struct napi_struct *napi, int budget)
{
	struct vnet_priv *priv = container_of(napi, struct vnet_priv, napi);
	struct sk_buff *skb;
	int done = 0;

	while (done < budget) {
		skb = skb_dequeue(&priv->rx_queue);
		if (!skb)
			break;			/* ring empty */

		skb->protocol = eth_type_trans(skb, priv->ndev);
		skb->ip_summed = CHECKSUM_UNNECESSARY;

		u64_stats_update_begin(&priv->syncp);
		priv->rx_packets++;
		priv->rx_bytes += skb->len;
		u64_stats_update_end(&priv->syncp);

		napi_gro_receive(napi, skb);
		done++;
	}

	/* Ring drained before exhausting budget -> leave polled mode.
	 * napi_complete_done() can return false if we were re-scheduled. */
	if (done < budget && napi_complete_done(napi, done)) {
		/* here a real driver re-enables RX interrupts */
	}

	return done;
}

/* ---------------- transmit ---------------- */

static netdev_tx_t vnet_start_xmit(struct sk_buff *skb,
				   struct net_device *ndev)
{
	struct vnet_priv *priv = netdev_priv(ndev);
	struct sk_buff *rx;

	u64_stats_update_begin(&priv->syncp);
	priv->tx_packets++;
	priv->tx_bytes += skb->len;
	u64_stats_update_end(&priv->syncp);

	/* BQL bookkeeping: we "complete" immediately, but do it properly
	 * so the pattern is visible (T.6). */
	netdev_sent_queue(ndev, skb->len);

	/* Loop the frame back as a received one. */
	rx = skb_copy(skb, GFP_ATOMIC);
	if (!rx) {
		priv->rx_dropped_noskb++;
	} else if (skb_queue_len(&priv->rx_queue) >= VNET_RX_QUEUE_LEN) {
		/* ring full: drop and count, exactly as hardware would */
		dev_kfree_skb_any(rx);
		ndev->stats.rx_dropped++;
	} else {
		skb_queue_tail(&priv->rx_queue, rx);
		napi_schedule(&priv->napi);
	}

	netdev_completed_queue(ndev, 1, skb->len);
	dev_kfree_skb_any(skb);		/* our "device" is done with it */
	return NETDEV_TX_OK;
}

/* ---------------- lifecycle ---------------- */

static int vnet_open(struct net_device *ndev)
{
	struct vnet_priv *priv = netdev_priv(ndev);

	skb_queue_head_init(&priv->rx_queue);
	napi_enable(&priv->napi);
	netif_start_queue(ndev);
	netif_carrier_on(ndev);		/* "cable plugged in" */
	netdev_info(ndev, "opened\n");
	return 0;
}

static int vnet_stop(struct net_device *ndev)
{
	struct vnet_priv *priv = netdev_priv(ndev);

	netif_carrier_off(ndev);
	netif_tx_disable(ndev);		/* stop xmit, wait for in-flight */
	napi_disable(&priv->napi);	/* drain polls: T.9 rule 2 */
	skb_queue_purge(&priv->rx_queue);
	netdev_reset_queue(ndev);	/* reset BQL */
	netdev_info(ndev, "stopped\n");
	return 0;
}

static void vnet_get_stats64(struct net_device *ndev,
			     struct rtnl_link_stats64 *s)
{
	struct vnet_priv *priv = netdev_priv(ndev);
	unsigned int start;

	do {
		start = u64_stats_fetch_begin(&priv->syncp);
		s->rx_packets = priv->rx_packets;
		s->rx_bytes   = priv->rx_bytes;
		s->tx_packets = priv->tx_packets;
		s->tx_bytes   = priv->tx_bytes;
	} while (u64_stats_fetch_retry(&priv->syncp, start));

	s->rx_dropped = ndev->stats.rx_dropped + priv->rx_dropped_noskb;
}

static void vnet_tx_timeout(struct net_device *ndev, unsigned int txq)
{
	netdev_err(ndev, "tx timeout on queue %u - resetting\n", txq);
	netif_wake_queue(ndev);
}

static const struct net_device_ops vnet_netdev_ops = {
	.ndo_open		= vnet_open,
	.ndo_stop		= vnet_stop,
	.ndo_start_xmit		= vnet_start_xmit,
	.ndo_get_stats64	= vnet_get_stats64,
	.ndo_tx_timeout		= vnet_tx_timeout,
	.ndo_set_mac_address	= eth_mac_addr,
	.ndo_validate_addr	= eth_validate_addr,
};

/* ---------------- ethtool ---------------- */

static void vnet_get_drvinfo(struct net_device *ndev,
			     struct ethtool_drvinfo *info)
{
	strscpy(info->driver, "vnet", sizeof(info->driver));
	strscpy(info->version, "1.0", sizeof(info->version));
	strscpy(info->bus_info, "virtual", sizeof(info->bus_info));
}

static const char vnet_stat_names[][ETH_GSTRING_LEN] = {
	"rx_dropped_noskb",
};

static int vnet_get_sset_count(struct net_device *ndev, int sset)
{
	return sset == ETH_SS_STATS ? ARRAY_SIZE(vnet_stat_names) : -EOPNOTSUPP;
}

static void vnet_get_strings(struct net_device *ndev, u32 sset, u8 *data)
{
	if (sset == ETH_SS_STATS)
		memcpy(data, vnet_stat_names, sizeof(vnet_stat_names));
}

static void vnet_get_ethtool_stats(struct net_device *ndev,
				   struct ethtool_stats *stats, u64 *data)
{
	struct vnet_priv *priv = netdev_priv(ndev);

	data[0] = priv->rx_dropped_noskb;
}

static int vnet_get_link_ksettings(struct net_device *ndev,
				   struct ethtool_link_ksettings *cmd)
{
	cmd->base.speed = SPEED_10000;
	cmd->base.duplex = DUPLEX_FULL;
	cmd->base.port = PORT_OTHER;
	cmd->base.autoneg = AUTONEG_DISABLE;
	return 0;
}

static const struct ethtool_ops vnet_ethtool_ops = {
	.get_drvinfo		= vnet_get_drvinfo,
	.get_link		= ethtool_op_get_link,
	.get_link_ksettings	= vnet_get_link_ksettings,
	.get_sset_count		= vnet_get_sset_count,
	.get_strings		= vnet_get_strings,
	.get_ethtool_stats	= vnet_get_ethtool_stats,
};

/* ---------------- module ---------------- */

static struct net_device *vnet_dev;

static int __init vnet_init(void)
{
	struct vnet_priv *priv;
	int ret;

	vnet_dev = alloc_etherdev(sizeof(*priv));
	if (!vnet_dev)
		return -ENOMEM;

	priv = netdev_priv(vnet_dev);
	priv->ndev = vnet_dev;
	u64_stats_init(&priv->syncp);
	skb_queue_head_init(&priv->rx_queue);

	vnet_dev->netdev_ops  = &vnet_netdev_ops;
	vnet_dev->ethtool_ops = &vnet_ethtool_ops;
	vnet_dev->watchdog_timeo = 5 * HZ;
	vnet_dev->min_mtu = ETH_MIN_MTU;
	vnet_dev->max_mtu = 9000;

	vnet_dev->features |= NETIF_F_SG | NETIF_F_HW_CSUM |
			      NETIF_F_RXCSUM | NETIF_F_TSO | NETIF_F_GRO;
	vnet_dev->hw_features = vnet_dev->features;

	eth_hw_addr_random(vnet_dev);
	strscpy(vnet_dev->name, "vnet%d", IFNAMSIZ);

	netif_napi_add(vnet_dev, &priv->napi, vnet_poll);

	/* Everything above must be done BEFORE this line (T.9 rule 3). */
	ret = register_netdev(vnet_dev);
	if (ret) {
		netif_napi_del(&priv->napi);
		free_netdev(vnet_dev);
		return ret;
	}
	netif_carrier_off(vnet_dev);	/* down until ndo_open */
	pr_info("vnet: registered %s\n", vnet_dev->name);
	return 0;
}

static void __exit vnet_exit(void)
{
	struct vnet_priv *priv = netdev_priv(vnet_dev);

	unregister_netdev(vnet_dev);	/* calls ndo_stop if up */
	netif_napi_del(&priv->napi);
	free_netdev(vnet_dev);
}

module_init(vnet_init);
module_exit(vnet_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Teaching network driver with NAPI, BQL and ethtool");
```

Build and exercise it:

```sh
sudo insmod vnet.ko
ip link show vnet0
sudo ip addr add 10.99.0.1/24 dev vnet0
sudo ip link set vnet0 up
ping -c3 -I vnet0 10.99.0.2        # ARP goes out, loops back
ip -s link show vnet0
ethtool -i vnet0
ethtool -S vnet0
ethtool -k vnet0 | head
cat /sys/class/net/vnet0/queues/tx-0/byte_queue_limits/limit   # BQL
```

Deliberate experiments:

1. **Break the drain order.** Move `napi_disable()` *after* `skb_queue_purge()` in `vnet_stop`. Run with `CONFIG_DEBUG_KOBJECT`/KASAN and a busy ping; observe the use-after-free. Restore the correct order and explain the invariant in one sentence (Ch. 25 P12).
2. **Remove `napi_complete_done()`.** The device stays in polled mode forever: `ksoftirqd` pins a core. Confirm with `top`.
3. **Return `budget` unconditionally from `vnet_poll`.** Same symptom, different cause — trace which.
4. **Remove `netif_carrier_on()`.** `ip link` shows `UP` but not `LOWER_UP`, routes do not install, and nothing works with no error anywhere (§T.9). This is a real bug class; experiencing it once makes it a five-second diagnosis forever.
5. **Delete the `netdev_sent_queue`/`netdev_completed_queue` pair** and confirm the BQL sysfs limit stops adapting.

---

### Lab 46.3 — Read `virtio_net`, then instrument it

`drivers/net/virtio_net.c` is the best driver to learn from: complete, modern, multi-queue, XDP-capable, page-pool-based, and you can run it in QEMU (Ch. 04) and modify it freely.

```sh
# In your QEMU VM from Ch. 04:
ethtool -i eth0            # driver: virtio_net
ethtool -l eth0            # combined channels = vCPUs
```

Reading assignment, in this order, and write one paragraph on each:

1. `virtnet_poll()` — find the budget accounting and the `napi_complete_done()` call. Note how it handles *both* RX and TX completion in one poll.
2. `start_xmit()` — find the `netif_stop_subqueue` / wake logic and the exact barrier used. Compare to §T.6's race.
3. `receive_buf()` → `receive_mergeable()` — three different buffer strategies (small, big, mergeable) and why mergeable wins.
4. `virtnet_xdp_*` — where the XDP program runs and what it does with the page.
5. `virtnet_set_features()` / `virtnet_fix_features()` — the offload dependency logic of §T.8.

Then instrument it without editing code:

```sh
sudo bpftrace -e 'kprobe:start_xmit { @tx = count(); }
                  kprobe:virtnet_poll { @poll = count(); }
                  interval:s:1 { print(@tx); print(@poll); clear(@tx); clear(@poll); }'
```

Run `iperf3` and watch the packets-per-poll ratio rise with load. That number *is* §T.2.

---

### Lab 46.4 — XDP from zero

```sh
sudo apt install clang llvm libbpf-dev bpftool linux-headers-$(uname -r)
```

`xdp_count.c`:

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <linux/in.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>

struct {
	__uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
	__uint(max_entries, 4);
	__type(key, __u32);
	__type(value, __u64);
} counters SEC(".maps");

#define C_TOTAL 0
#define C_IPV4  1
#define C_ICMP  2
#define C_DROP  3

static __always_inline void bump(__u32 k)
{
	__u64 *v = bpf_map_lookup_elem(&counters, &k);

	if (v)
		(*v)++;
}

SEC("xdp")
int xdp_prog(struct xdp_md *ctx)
{
	void *data = (void *)(long)ctx->data;
	void *end  = (void *)(long)ctx->data_end;
	struct ethhdr *eth = data;
	struct iphdr *ip;

	bump(C_TOTAL);

	/* Every access must be bounds-checked or the verifier rejects it. */
	if ((void *)(eth + 1) > end)
		return XDP_PASS;
	if (eth->h_proto != bpf_htons(ETH_P_IP))
		return XDP_PASS;

	ip = (void *)(eth + 1);
	if ((void *)(ip + 1) > end)
		return XDP_PASS;

	bump(C_IPV4);

	if (ip->protocol == IPPROTO_ICMP) {
		bump(C_ICMP);
		bump(C_DROP);
		return XDP_DROP;      /* silently eat all ICMP */
	}
	return XDP_PASS;
}

char _license[] SEC("license") = "GPL";
```

```sh
clang -O2 -g -target bpf -c xdp_count.c -o xdp_count.o
sudo ip link set dev eth0 xdpgeneric obj xdp_count.o sec xdp
# or, on a driver with native support:
sudo ip link set dev eth0 xdp obj xdp_count.o sec xdp

ip link show eth0 | grep -o 'xdp[a-z]*'
ping -c3 <peer>                          # fails: dropped in the driver
sudo bpftool map dump name counters

sudo ip link set dev eth0 xdp off
```

Now the measurements that make the point:

1. Compare `xdpgeneric` (runs after skb allocation) with native `xdp`. Measure drop rate with `pktgen` or a traffic generator. Expect roughly an order of magnitude.
2. Add `bpf_printk()` and watch `/sys/kernel/debug/tracing/trace_pipe`; then remove it and re-measure. Explain the cost.
3. Try `XDP_TX` (reflect the packet) — you must swap MAC addresses first; the verifier will force you to bounds-check again.
4. Write a program that uses `bpf_xdp_adjust_head()` to prepend a header, and observe what happens if the driver did not reserve 256 bytes of headroom.

---

### Lab 46.5 — Offloads: prove each one matters

```sh
IF=eth0
ethtool -k $IF | grep -E 'tcp-segmentation|generic-receive|checksum|scatter'

# Baseline throughput and CPU
iperf3 -c <peer> -t 20 & mpstat 1 20

# Turn off one at a time and re-measure
sudo ethtool -K $IF tso off
sudo ethtool -K $IF gso off
sudo ethtool -K $IF gro off
sudo ethtool -K $IF tx off rx off      # checksums
```

Record throughput, CPU%, and interrupts/sec for each configuration. Then explain, from §T.5:

1. Why disabling TSO costs more CPU than disabling GSO.
2. Why GRO off raises CPU on the *receiver* far more than on the sender.
3. Why `ethtool -K $IF tx off` may also silently disable TSO (dependency logic in `ndo_fix_features`), and verify it with `ethtool -k` before and after.

Then observe GSO directly:

```sh
sudo bpftrace -e 'kprobe:__skb_gso_segment { @ = hist(arg0); }'   # crude but illustrative
sudo tcpdump -i $IF -n -c 20 'tcp' | head       # note lengths > MTU with TSO on
```

Seeing a 64 KB "packet" in `tcpdump` on a 1500-MTU link is many engineers' first encounter with the fact that the capture point is *above* segmentation.

---

### Lab 46.6 — Find the drop

The skill this lab teaches is worth more than the rest of the chapter: **every dropped packet has exactly one counter that explains it.** Build the map.

```sh
IF=eth0

# Layer by layer, outside in:
ethtool -S $IF | grep -Ei 'drop|err|miss|fifo|nobuf|discard'   # NIC/driver
ip -s -s link show $IF                                          # netdev summary
cat /proc/net/softnet_stat                                      # backlog / budget
nstat -az | grep -Ei 'drop|error|overflow|prune|collapse'       # stack
ss -tim                                                         # per-socket
tc -s qdisc show dev $IF                                        # qdisc drops
```

| Symptom | Counter | Meaning | Fix |
|---|---|---|---|
| NIC had no descriptor | `rx_missed_errors`, `rx_no_buffer_count` | ring exhausted; driver too slow to refill | bigger ring, more queues, NAPI budget |
| Backlog full | `/proc/net/softnet_stat` col 2 | RPS/`netif_rx` backlog overflow | `netdev_max_backlog`, RPS tuning |
| Ran out of budget | col 3 `time_squeeze` | poll loop hit budget with work left | `netdev_budget`, threaded NAPI |
| Socket buffer full | `ss -tim` shows rcv drops; `nstat TcpExtTCPRcvQDrop` | application too slow | bigger `rcvbuf`, faster app |
| Qdisc dropped | `tc -s qdisc` `dropped N` | egress queue full / policed | qdisc choice, BQL, rate limits |
| No route / martian | `nstat IpInAddrErrors` | routing | config |

Force each one deliberately:

```sh
# Ring exhaustion
sudo ethtool -G $IF rx 128
# then hammer with small packets from a fast sender; watch rx_missed_errors

# Backlog overflow
sudo sysctl -w net.core.netdev_max_backlog=16
# enable RPS and flood

# time_squeeze
sudo sysctl -w net.core.netdev_budget=8
```

Restore every setting afterwards. Write down, for each, the *single command* that would have identified it in production.

---

### Lab 46.7 — A software netdev you can route through: `veth` and namespaces

Before Part 4, build the topology you will use there:

```sh
sudo ip netns add ns1
sudo ip link add veth0 type veth peer name veth1
sudo ip link set veth1 netns ns1

sudo ip addr add 10.0.0.1/24 dev veth0
sudo ip link set veth0 up
sudo ip netns exec ns1 ip addr add 10.0.0.2/24 dev veth1
sudo ip netns exec ns1 ip link set veth1 up
sudo ip netns exec ns1 ip link set lo up

ping -c3 10.0.0.2
sudo ip netns exec ns1 ethtool -i veth1
sudo ip netns exec ns1 ethtool -S veth1
```

Then read `drivers/net/veth.c` — 1700 lines, and it implements NAPI, GRO, XDP, and multi-queue. It is the best example of a netdev whose "hardware" is another netdev, and the XDP support there is the clearest in the tree.

Exercises: attach the Lab 46.4 XDP program to `veth1`; measure `ping` before and after; then use `XDP_REDIRECT` to bounce traffic from `veth0` to another interface and observe that the packet never enters the stack.

---

## 3. Mastery drills

1. Derive the throughput curve of a pure interrupt-driven receive path as a function of arrival rate, including the term that produces livelock. Then show formally why NAPI's "disable IRQs while polling" makes the collapse impossible regardless of $\lambda$.

2. `napi_complete_done()` can return `false`. Enumerate every circumstance in which it does, and show what a driver that ignores the return value does wrong in each.

3. Write the exact barrier-annotated pseudocode for an RX descriptor ring where the device writes status last, and prove that replacing `dma_rmb()` with a compiler barrier is incorrect on arm64 but happens to work on x86-64. Name the reordering that would occur.

4. The `netif_stop_queue`/`netif_wake_queue` race (§T.6) is an SB litmus test. Write it as one (Ch. 13), identify the two stores and two loads, and show which barrier placement makes it safe. Then find `netif_txq_maybe_stop()` in the tree and check your answer.

5. BQL adapts the byte limit at runtime. Read `net/core/dynamic_queue_limits.c` and describe its control law. Is it proportional, integral, or something else? What is its convergence behaviour under a step change in link rate?

6. LRO is irreversible and GRO is not. State precisely the merge conditions GRO enforces to guarantee `skb_gso_segment()` reproduces the originals, and give a packet sequence where a *naive* merger would be irreversible.

7. `CHECKSUM_COMPLETE` lets the stack adjust the checksum as headers are pulled. Derive the arithmetic identity that makes this possible for ones-complement sums, and explain why it fails for CRC32 (as used by, e.g., SCTP).

8. RSS hashes the 4-tuple to a queue. Show that this preserves per-flow ordering but not per-*host* ordering, and describe a workload where the latter matters. Then explain why IP fragments break RSS and what NICs do about it.

9. Compute the memory cost of a 64-queue NIC with 4096-descriptor rings using 4 KiB page buffers. Now redo it with page_pool recycling and explain what changed and what did not.

10. XDP requires a single writable page with 256 B headroom. Enumerate three driver optimisations that this constraint forbids, and estimate the cost of each.

11. `AF_XDP` gives kernel-bypass performance without taking the NIC from the kernel. Explain the mechanism that makes this possible, and state what the kernel gives up in terms of isolation compared to a normal socket.

12. `ndo_get_stats64` uses `u64_stats_sync`. Explain why a plain `u64` read is insufficient on 32-bit, why a spinlock would be too expensive, and how the seqcount variant achieves correctness (relate to Ch. 14 §T.9 and Ch. 16).

13. Design the full teardown sequence for a driver being unbound while (a) a NAPI poll is running, (b) an XDP program is attached, (c) an AF_XDP socket has the queue bound, and (d) a packet is in flight to DMA. Name the primitive that handles each and the order that is forced.

---

## 4. Further reading

**Kernel documentation** (`Documentation/networking/`)

- `napi.rst` ★★★ — the definitive modern description: states, threaded NAPI, deferred IRQs, budget.
- `driver.rst` ★★★ — the TX-stop race, locking rules, the "do not do this" list. Short and essential.
- `netdev-features.rst` ★★★ — `features`/`hw_features`/`wanted_features` and the dependency rules of §T.8.
- `segmentation-offloads.rst`, `checksum-offloads.rst` ★★★ — the offload contracts, precisely stated.
- `scaling.rst` ★★★ — RSS/RPS/RFS/aRFS/XPS, including the reasoning about cache locality.
- `statistics.rst` ★★★ — what every standard counter means; read before writing `ndo_get_stats64`.
- `page_pool.rst`, `af_xdp.rst`, `xdp-rx-metadata.rst` ★★★
- `ethtool-netlink.rst` ★★ — the modern configuration ABI.
- `kapi.rst`, `netdevices.rst` ★★ — API reference.
- `Documentation/bpf/` — the verifier, maps, and the XDP program model.

**Papers**

- J. C. Mogul and K. K. Ramakrishnan, "Eliminating Receive Livelock in an Interrupt-Driven Kernel," *ACM TOCS* 15(3), 1997 — **read this one**. It is the theoretical basis for NAPI and for `blk-mq`'s and NVMe's completion designs.
- J. H. Salim, R. Olsson, A. Kuznetsov, "Beyond Softnet," Annual Linux Showcase, 2001 — the NAPI paper.
- T. Høiland-Jørgensen et al., "The eXpress Data Path: Fast Programmable Packet Processing in the Operating System Kernel," CoNEXT 2018 — the XDP paper, with measurements.
- M. Karlsson and B. Töpel, "The Path to DPDK Speeds for AF_XDP," Linux Plumbers 2018.
- J. Gettys and K. Nichols, "Bufferbloat: Dark Buffers in the Internet," *CACM* 55(1), 2012 — the problem BQL and fq_codel solve.
- L. Rizzo, "netmap: A Novel Framework for Fast Packet I/O," USENIX ATC 2012 — the bypass design XDP was responding to.
- S. Han et al., "PacketShader" and L. Rizzo's work more broadly, for the 10 GbE-era cost analyses that motivated all of this.

**Source worth reading end to end**

- `drivers/net/virtio_net.c` ★★★ — the best complete modern driver to learn from; runs in QEMU.
- `drivers/net/veth.c` ★★★ — NAPI, GRO, XDP, multi-queue, no hardware.
- `drivers/net/ethernet/intel/igb/` ★★ — a classic, very readable real NIC driver.
- `drivers/net/ethernet/mellanox/mlx5/core/en_rx.c` ★★ — state of the art; page_pool, XDP, striding RQ. Hard but instructive.
- `net/core/dev.c`: `net_rx_action()`, `napi_schedule_prep()`, `napi_complete_done()`, `__dev_queue_xmit()`.
- `net/core/page_pool.c` — small, and it explains itself.
- `net/core/dynamic_queue_limits.c` — BQL's control law in 200 lines.

**Books**

- Rami Rosen, *Linux Kernel Networking: Implementation and Theory* (Apress, 2014) — dated on XDP but excellent on the core structures.
- Christian Benvenuti, *Understanding Linux Network Internals* (O'Reilly, 2005) — very dated, still the clearest explanation of the netdev layer's design.
- W. Richard Stevens, *TCP/IP Illustrated, Vol. 1* — for the protocol knowledge this chapter assumes.

**LWN**

- "The rapidly changing world of Linux networking" (annual roundups)
- "Bufferbloat" and "Byte queue limits" (2011) — BQL introduced and argued
- "A JIT for network packet filtering", "Accelerating networking with AF_XDP" (2018)
- "Threaded NAPI" and "Deferring network interrupts" (2020–2021)
- "The page pool" series

**Tools**

- `ethtool` (`-i -g -c -l -k -S -s -K -G -L -C -T -m --show-fec`), `ip -s -s link`, `nstat`, `ss -tim`, `tc -s`
- `bpftool prog/map/net`, `xdp-loader`/`xdp-filter` from `xdp-tools`
- `pktgen` (`samples/pktgen/`), `trafgen`, `iperf3`, `netperf`, `TRex`
- `perf top -e cycles:k`, `perf record -e net:*`, `bpftrace`
- `tcpdump`/`tshark` — and remember the capture point is above segmentation
- `dropwatch`, and `perf record -e skb:kfree_skb` (which since 6.0 reports a *drop reason* enum — the single best addition to network debugging in years; see `include/net/dropreason.h`)

---

→ Next: [47-drm-v4l2.md](47-drm-v4l2.md)

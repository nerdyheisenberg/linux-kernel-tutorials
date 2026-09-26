# Chapter 71 — `sk_buff`, the network device layer, and the RX/TX paths

> **Goal:** Understand the data structure that every packet in Linux lives in, and the two paths it travels. Understand why `sk_buff` looks the way it does and what each of its many pointers is for, headroom and tailroom as the mechanism that makes encapsulation cheap, cloning versus copying and the shared-data model, `struct net_device` and the driver contract, NAPI as the answer to receive livelock, GRO and GSO as the inverse operations that let the stack process fewer, larger packets, the transmit path's queueing discipline and BQL, and XDP as the hook before any of it. By the end you can read `net/core/`, trace a packet from the wire to a socket, and reason about where a microsecond goes.

---

## Theory & First Principles

### T.0 — Start here: one buffer, seven layers, zero copies

A packet arrives. On its way up it must be examined by the driver, Ethernet, IP, netfilter,
TCP, and finally a socket — each of which wants to strip its own header and hand the rest
upward. The obvious implementation copies:

```c
eth_payload = malloc(len - 14); memcpy(...);   /* strip Ethernet */
ip_payload  = malloc(len - 34); memcpy(...);   /* strip IP       */
tcp_payload = malloc(len - 54); memcpy(...);   /* strip TCP      */
```

At **14.88 Mpps** (10 GbE, minimum-size frames) you have **67 nanoseconds per packet** —
about 200 CPU cycles. One `malloc` costs more than that. Three copies of 1500 bytes cost far
more. **The copying implementation is off by two orders of magnitude before it does anything
useful.**

**So the `sk_buff` never copies. It moves pointers.**

```
   +----------+--------+------------------------+----------+
   | headroom | headers|        payload         | tailroom |
   +----------+--------+------------------------+----------+
   ^          ^                                 ^          ^
   head       data                              tail       end

   skb_pull(skb, 14)  -> data += 14   (strip a header going UP)
   skb_push(skb, 20)  -> data -= 20   (prepend a header going DOWN)
   skb_put (skb, n)   -> tail += n    (append payload)
```

**Headroom is the key design decision and is worth understanding as a principle.** Drivers
allocate space *before* the data — typically `NET_SKB_PAD` (64 bytes) — for headers that do
not exist yet. When the packet later needs a VLAN tag, an IP header, a tunnel encapsulation,
or a VXLAN wrapper, they are written into space that was reserved at allocation time.

> **Reserve room for the future at allocation time, so that later prepends are free.** Get
> the headroom wrong and every encapsulated packet triggers a reallocation and a full copy —
> which is exactly the symptom people see as "tunnels are slow."

**Second, the `sk_buff` is also a *control block*, and this is what makes it big.** It carries
not just the data but the packet's entire processing state: which device, which protocol, the
checksum status, the connection-tracking entry, the routing decision, GSO segmentation
parameters, timestamps, and a 48-byte opaque `cb[]` scratch area that each layer reinterprets.

```
  struct sk_buff: ~232 bytes of metadata for a possibly-64-byte packet.
```

**That ratio is the central tension of the Linux network stack**, and the whole of Ch. 74
(XDP) is the argument that for some workloads you should make a decision *before* an
`sk_buff` ever exists. Allocating and initializing 232 bytes, touching several cachelines, is
already a large fraction of your 67 ns budget.

**Third, notice how differently this reads from Part 3.** Storage and networking look
superficially similar — both move buffers between a device and a process — but:

| | Storage | Networking |
|---|---|---|
| Who initiates? | **you do** (a read/write) | **the other end does**; packets arrive unbidden |
| Can you apply backpressure? | yes — don't submit | **no** — you can only drop |
| Is the peer trusted? | mostly | **never** — every byte is attacker-controlled |
| Ordering | you control it | reordering is normal and must be handled |
| Failure | an error code | silence, indistinguishable from slowness (Ch. 62 §T.0) |

**"You cannot apply backpressure, you can only drop"** is the one to internalize. It is why
the stack has queue disciplines, why NAPI polls rather than interrupts (a receive livelock
otherwise — Ch. 17, Ch. 46), and why every buffer in the receive path must be preallocated:
you cannot fail to allocate on the receive path, so you must never need to.

```bash
ip -s -s link show                       # per-device counters, including drops
ethtool -S eth0 | grep -Ei 'drop|error|miss'
cat /proc/net/softnet_stat               # per-CPU backlog and squeeze counts
sudo /usr/share/bcc/tools/tcpdrop        # where packets are dropped, with stacks
```

---

### T.1 What makes networking different from storage

Part 3's storage stack had a comfortable property: **the initiator is the host.** An application asks for data; the kernel fetches it. Everything is demand-driven and back-pressure flows naturally — if you stop asking, nothing happens.

Networking inverts this:

| | Storage | Networking |
|---|---|---|
| Who initiates? | **the host** | **the remote end** |
| Can you refuse work? | yes, just do not submit | **no — packets arrive regardless** |
| Back-pressure | natural | must be constructed |
| Units | fixed-size blocks | **variable-length packets** |
| Failure | an error return | **silent drops, by design** |
| Ordering | mostly irrelevant | matters enormously |
| Latency tolerance | milliseconds | **microseconds** |

The second row is the decisive one. A storage device cannot force you to do work; a network adapter can. At 10 Gbit/s with 64-byte frames, packets arrive at **14.88 million per second** — one every 67 ns. On a 3 GHz core that is about 200 cycles per packet for *everything*: DMA completion, protocol parsing, routing, filtering, socket delivery.

That budget is the source of nearly every design decision in this chapter. If handling one packet costs more than the inter-arrival time, the system enters **receive livelock** (§T.6): it spends all its time taking interrupts and drops everything. The mechanisms below — NAPI, GRO, batching, XDP — all exist to spend fewer cycles per packet or to process more bytes per unit of work.

### T.2 `sk_buff`: one structure, every packet

```c
struct sk_buff {
	union {
		struct {
			struct sk_buff	*next;
			struct sk_buff	*prev;
			union {
				struct net_device *dev;
				unsigned long	 dev_scratch;
			};
		};
		struct rb_node	rbnode;      /* for TCP out-of-order queues */
		struct list_head list;
		struct llist_node ll_node;
	};

	struct sock	*sk;
	union {
		ktime_t		tstamp;
		u64		skb_mstamp_ns;
	};

	char		cb[48] __aligned(8);   /* per-layer scratch space */

	union {
		struct {
			unsigned long	_skb_refdst;
			void		(*destructor)(struct sk_buff *skb);
		};
		struct list_head	tcp_tsorted_anchor;
	};

	unsigned int	len,          /* total data length, incl. fragments */
			data_len;     /* length in FRAGMENTS only */
	__u16		mac_len,
			hdr_len;
	__u16		queue_mapping;
	__u8		__cloned_offset[0];
	__u8		cloned:1, nohdr:1, fclone:2, peeked:1,
			head_frag:1, pfmemalloc:1, pp_recycle:1;

	__u32		headers_start[0];
	__u8		__pkt_type_offset[0];
	__u8		pkt_type:3, ignore_df:1, dst_pending_confirm:1,
			ip_summed:2, ooo_okay:1;
	__u8		l4_hash:1, sw_hash:1, wifi_acked_valid:1, ...;
	__be16		protocol;
	__u16		transport_header;   /* OFFSETS, not pointers */
	__u16		network_header;
	__u16		mac_header;
	__u32		headers_end[0];

	sk_buff_data_t	tail;
	sk_buff_data_t	end;
	unsigned char	*head,        /* the allocated buffer */
			*data;        /* the current protocol's header */
	unsigned int	truesize;     /* total memory charged */
	refcount_t	users;
};
```

It is a large structure — around 230 bytes — and its size is a perennial complaint, because at 14.88 Mpps you are touching 3.4 GB/s of metadata before looking at any packet data.

The layout worth understanding:

```
     head            data            tail            end
      |               |               |               |
      v               v               v               v
      +---------------+---------------+---------------+
      |   HEADROOM    |     DATA      |   TAILROOM    |
      +---------------+---------------+---------------+
      <-------------------- truesize -------------------->
```

**Headroom exists so that adding headers is free.** A packet received on Ethernet has its IP header at some offset; when it is forwarded and re-encapsulated (VLAN, VXLAN, IPsec), the new header is written into the headroom by moving `data` *backwards*:

```c
static inline void *skb_push(struct sk_buff *skb, unsigned int len)
{
	skb->data -= len;
	skb->len  += len;
	if (unlikely(skb->data < skb->head))
		skb_under_panic(skb, len, __builtin_return_address(0));
	return skb->data;
}
```

No copy, no allocation. `NET_SKB_PAD` (typically 64 bytes) is the default headroom every driver should reserve, and drivers that do not reserve enough force a reallocation on every encapsulation — a classic and easily-missed performance bug.

`skb_put` grows the data at the tail; `skb_pull` removes a header from the front; `skb_reserve` adjusts headroom before any data exists. Those four operations are most of what packet processing does.

**Header offsets, not pointers.** `transport_header`, `network_header`, and `mac_header` are 16-bit *offsets from `head`*, not pointers. This is so the structure survives being copied and so it is smaller. Accessors hide it:

```c
static inline struct iphdr *ip_hdr(const struct sk_buff *skb)
{
	return (struct iphdr *)skb_network_header(skb);
}
```

**`cb[48]` is per-layer scratch.** Each layer may use it while it owns the skb, and must not assume anything survives being passed on. TCP uses it for `struct tcp_skb_cb` (sequence numbers, flags); the bridge uses it; qdiscs use it. It is a union by convention rather than by type, which is fragile and has caused real bugs, but it avoids allocating per-layer state.

### T.3 Non-linear skbs and the shared-data model

A packet's data need not be contiguous. Large packets — anything built from pages by GRO, or sent with `sendfile`/`splice` — use fragments:

```c
struct skb_shared_info {
	__u8		flags;
	__u8		nr_frags;
	__u8		tx_flags;
	unsigned short	gso_size;        /* T.8 */
	unsigned short	gso_segs;
	struct sk_buff	*frag_list;
	struct skb_shared_hwtstamps hwtstamps;
	unsigned int	gso_type;
	u32		tskey;
	atomic_t	dataref;         /* how many skbs share this data */
	unsigned int	xdp_frags_size;
	void		*destructor_arg;
	skb_frag_t	frags[MAX_SKB_FRAGS];   /* 17 on x86-64 */
};

#define skb_shinfo(SKB)	((struct skb_shared_info *)(skb_end_pointer(SKB)))
```

`skb_shared_info` lives **at the end of the linear buffer**, after `end`. So one allocation holds the linear data and the fragment array.

The lengths then mean:

| Field | Meaning |
|---|---|
| `skb->len` | **total** bytes: linear + fragments + frag_list |
| `skb->data_len` | bytes **not** in the linear area |
| `skb_headlen(skb)` | `len - data_len` — the linear part |

A function that assumes `skb->len` bytes are readable from `skb->data` is wrong for any non-linear skb. The correct idiom is `skb_header_pointer()` or `pskb_may_pull()`:

```c
	if (!pskb_may_pull(skb, sizeof(struct tcphdr)))
		goto drop;
	th = tcp_hdr(skb);
```

`pskb_may_pull` ensures the first N bytes are linear, copying from fragments if necessary. **Forgetting it is the most common source of out-of-bounds reads in networking code**, and it is why fuzzing the network stack finds so many bugs.

**Cloning versus copying** is the other half of the model:

| Operation | `sk_buff` | data |
|---|---|---|
| `skb_get()` | refcount++ | shared |
| `skb_clone()` | **new struct** | **shared** (`dataref`++) |
| `skb_copy()` | new struct | **copied** |
| `pskb_copy()` | new struct | linear copied, frags shared |
| `skb_share_check()` | clone if shared | |

Cloning gives you a private `sk_buff` (so you can modify `data`, `len`, the header offsets) while sharing the payload. `tcpdump` clones every packet; TCP clones packets it retransmits. Writing to shared data requires `skb_cow()` or `skb_ensure_writable()` first — and forgetting *that* corrupts other users' packets, which is a spectacularly confusing bug.

`skb->cloned` and `skb_shared_info->dataref` together encode the state, and `skb_cloned()` / `skb_shared()` are the predicates.

### T.4 Allocation: where the cycles go

At 14.88 Mpps, allocating and freeing an skb per packet is a significant fraction of the budget. Four mechanisms:

**(a) `fclone`.** `alloc_skb_fclone()` allocates two skbs adjacently, because TCP nearly always clones the packet it sends (one for transmission, one for the retransmit queue). One allocation instead of two.

**(b) `build_skb()`.** A driver that has already received data into a page can wrap it rather than allocating and copying:

```c
	skb = build_skb(data, frag_size);
	skb_reserve(skb, NET_SKB_PAD);
	skb_put(skb, len);
```

This is the standard modern driver pattern: DMA into a page, then `build_skb` around it.

**(c) Page pool.** `net/core/page_pool.c` provides a per-queue, DMA-mapped page cache so drivers need not map and unmap on every packet. DMA mapping is expensive (Ch. 35), especially with an IOMMU, and the page pool amortises it to nearly zero. `skb->pp_recycle` marks an skb whose pages should return to the pool rather than the page allocator.

**(d) NAPI skb cache.** `napi_alloc_skb()` and the per-CPU `napi_alloc_cache` keep a small batch of skbs ready, and `napi_consume_skb()` returns them, avoiding the slab allocator in the common case.

Together these turn per-packet allocation from a major cost into a minor one — but only for drivers that use them. A driver using plain `netdev_alloc_skb` plus `dma_map_single` per packet leaves most of the performance on the table.

### T.5 `net_device` and the driver contract

```c
struct net_device {
	char			name[IFNAMSIZ];
	unsigned long		state;
	struct list_head	dev_list;
	netdev_features_t	features;        /* what the DRIVER supports */
	netdev_features_t	hw_features;     /* what can be toggled */
	netdev_features_t	wanted_features;
	netdev_features_t	vlan_features;
	int			ifindex;
	const struct net_device_ops *netdev_ops;   /* THE contract */
	const struct ethtool_ops *ethtool_ops;
	const struct header_ops	*header_ops;
	unsigned int		flags;
	unsigned int		mtu, min_mtu, max_mtu;
	unsigned short		type;
	unsigned char		addr_len;
	unsigned char		dev_addr[MAX_ADDR_LEN];
	struct netdev_rx_queue	*_rx;
	unsigned int		num_rx_queues, real_num_rx_queues;
	struct netdev_queue	*_tx;
	unsigned int		num_tx_queues, real_num_tx_queues;
	struct Qdisc __rcu	*qdisc;
	unsigned long		tx_queue_len;
	struct net		*nd_net;             /* the network namespace */
	...
};

struct net_device_ops {
	int  (*ndo_init)(struct net_device *dev);
	void (*ndo_uninit)(struct net_device *dev);
	int  (*ndo_open)(struct net_device *dev);
	int  (*ndo_stop)(struct net_device *dev);
	netdev_tx_t (*ndo_start_xmit)(struct sk_buff *skb,
				      struct net_device *dev);   /* THE one */
	u16  (*ndo_select_queue)(struct net_device *dev, struct sk_buff *skb, ...);
	void (*ndo_set_rx_mode)(struct net_device *dev);
	int  (*ndo_set_mac_address)(struct net_device *dev, void *addr);
	int  (*ndo_do_ioctl)(struct net_device *dev, struct ifreq *ifr, int cmd);
	int  (*ndo_change_mtu)(struct net_device *dev, int new_mtu);
	void (*ndo_tx_timeout)(struct net_device *dev, unsigned int txqueue);
	void (*ndo_get_stats64)(struct net_device *dev,
				struct rtnl_link_stats64 *storage);
	int  (*ndo_bpf)(struct net_device *dev, struct netdev_bpf *bpf);
	int  (*ndo_xdp_xmit)(struct net_device *dev, int n,
			     struct xdp_frame **xdp, u32 flags);
	...
};
```

`ndo_start_xmit` is the heart, and its contract is strict:

| Rule | Why |
|---|---|
| Must not sleep | called in softirq context |
| Returns `NETDEV_TX_OK` or `NETDEV_TX_BUSY` | busy is a bug in modern drivers; stop the queue instead |
| **Owns the skb after returning OK** | must free it (usually at TX completion) |
| Must handle any skb the features advertise | if you claim `NETIF_F_SG`, handle fragments |

**Features are a contract.** `dev->features` tells the stack what the hardware can do; the stack then hands the driver packets it must handle:

| Feature | Meaning |
|---|---|
| `NETIF_F_SG` | scatter-gather DMA — non-linear skbs are acceptable |
| `NETIF_F_IP_CSUM` / `IPV6_CSUM` / `HW_CSUM` | compute the L4 checksum |
| `NETIF_F_TSO` / `TSO6` | **accept a 64 KB skb and segment it** (§T.8) |
| `NETIF_F_GSO_*` | specific offload types |
| `NETIF_F_GRO` / `LRO` | receive aggregation (§T.8) |
| `NETIF_F_RXCSUM` | verify checksums on receive |
| `NETIF_F_HW_VLAN_CTAG_TX/RX` | VLAN tag insertion/removal |
| `NETIF_F_NTUPLE` | flow steering |
| `NETIF_F_RXHASH` | compute a flow hash for RPS/RSS |

Advertising a feature the hardware does not correctly implement produces corruption that appears only under specific traffic — which is why `ethtool -K` exists and why "turn off offloads and see if it fixes it" is a standard diagnostic.

### T.6 NAPI: the answer to receive livelock

**The problem.** One interrupt per packet means that at high rates the CPU does nothing but take interrupts. Worse, interrupt handling has priority over softirq processing, so the system receives packets into the ring, takes interrupts, and never gets to *process* them. Throughput collapses to zero while the CPU is 100 % busy. Mogul and Ramakrishnan named this **receive livelock** in 1996 and it is the canonical failure mode of interrupt-driven I/O.

**The insight:** under load, you do not need interrupts. You need to know *when* to start polling and *when* to stop.

NAPI:

```
packet arrives -> interrupt
  -> driver DISABLES the interrupt
  -> schedules NAPI poll (napi_schedule)
  -> softirq context: poll(napi, budget)
       - process up to `budget` packets
       - if fewer than budget were available:
             ring is empty -> napi_complete_done() -> RE-ENABLE interrupt
       - else:
             stay in polling mode; the softirq will call us again
```

So at low rates you get one interrupt per packet (low latency); at high rates you get one interrupt per *batch* and then pure polling (high throughput). **The system adapts automatically between the two regimes**, which is the whole design.

```c
static int my_poll(struct napi_struct *napi, int budget)
{
	struct my_queue *q = container_of(napi, struct my_queue, napi);
	int work_done = 0;

	while (work_done < budget) {
		struct sk_buff *skb = my_get_next_packet(q);

		if (!skb)
			break;
		napi_gro_receive(napi, skb);     /* T.8 */
		work_done++;
	}

	if (work_done < budget) {
		/* We drained the ring. Go back to interrupts. */
		if (napi_complete_done(napi, work_done))
			my_enable_irq(q);
	}
	return work_done;
}
```

The `budget` matters: `net.core.netdev_budget` (default 300) bounds the total work per softirq invocation across all NAPI instances, and `netdev_budget_usecs` (default 2000) bounds the time. Exceeding either defers to `ksoftirqd`, which is visible as `time_squeeze` in `/proc/net/softnet_stat` — the standard signal that the system is receive-bound.

Three refinements worth knowing:

**Interrupt coalescing** (`ethtool -c`) delays the interrupt in hardware, trading latency for fewer interrupts. NAPI already provides most of this benefit adaptively, so aggressive coalescing usually just adds latency. `adaptive-rx` lets the driver tune it dynamically.

**Busy polling** (`SO_BUSY_POLL`, `net.core.busy_poll`) lets a socket read call poll the device directly rather than sleeping, eliminating the interrupt and the wakeup — sub-10 µs latency at the cost of burning a core.

**Threaded NAPI** (`/sys/class/net/*/threaded`) runs NAPI in a kernel thread rather than softirq context, which makes it schedulable, cgroup-accountable, and priority-adjustable. It is increasingly the right default for latency-sensitive workloads because softirq processing is otherwise invisible to the scheduler.

### T.7 The receive path, end to end

```
NIC DMAs a packet into a ring buffer, raises an interrupt
 └─ driver ISR: disable IRQ, napi_schedule()
     └─ NET_RX_SOFTIRQ: net_rx_action()
         └─ driver poll():
             ├─ build_skb() around the received page
             ├─ set skb->protocol, checksum status, hash
             ├─ XDP program runs HERE (if attached)  -- T.10
             └─ napi_gro_receive(napi, skb)
                 ├─ GRO: try to merge with a pending flow  -- T.8
                 └─ netif_receive_skb() when flushed
                     ├─ tcpdump / AF_PACKET taps (ptype_all)
                     ├─ tc ingress (clsact) -- Ch. 73
                     ├─ netfilter NF_INET_PRE_ROUTING via the protocol handler
                     └─ deliver to the protocol handler by skb->protocol
                         └─ ip_rcv()
                             ├─ validate the IP header
                             ├─ NF_INET_PRE_ROUTING
                             └─ ip_rcv_finish()
                                 └─ routing decision (fib_lookup)
                                     ├─ local -> ip_local_deliver()
                                     │   ├─ defragment
                                     │   ├─ NF_INET_LOCAL_IN
                                     │   └─ tcp_v4_rcv() / udp_rcv()
                                     │       └─ socket lookup -> receive queue
                                     │           └─ wake the reader
                                     └─ forward -> ip_forward()
```

`netif_receive_skb()` is the entry to the protocol-independent stack, and it is where every tap and hook lives. `__netif_receive_skb_core()` is the function to read.

**RPS and RFS** address the case where the hardware cannot steer flows:

| Mechanism | What it does |
|---|---|
| **RSS** | hardware hashes the flow and picks a queue → a CPU. Free, but hardware-dependent. |
| **RPS** | software: hash the flow, IPI the target CPU, queue there. Costs an IPI. |
| **RFS** | software: steer to the CPU where the *application* runs. Better cache behaviour. |
| **aRFS** | ask the hardware to steer, based on where the application runs. Best of both. |

RFS's insight is that the packet's data will be touched by the application, so delivering it to that CPU avoids a cache-line transfer of the whole payload. `rps_sock_flow_table` records where each flow's socket was last read from.

```sh
echo f > /sys/class/net/eth0/queues/rx-0/rps_cpus
echo 32768 > /proc/sys/net/core/rps_sock_flow_entries
echo 2048 > /sys/class/net/eth0/queues/rx-0/rps_flow_cnt
```

### T.8 GRO and GSO: process fewer, larger packets

The per-packet cost is roughly fixed regardless of size. So if you can turn ten 1500-byte packets into one 15000-byte skb, you pay the stack's per-packet cost once instead of ten times. That is the entire idea, applied in both directions.

**GRO (Generic Receive Offload)** merges received packets:

```c
struct packet_offload {
	__be16			type;
	u16			priority;
	struct offload_callbacks callbacks;
	struct list_head	list;
};

struct offload_callbacks {
	struct sk_buff *(*gso_segment)(struct sk_buff *skb,
				       netdev_features_t features);
	struct sk_buff *(*gro_receive)(struct list_head *head,
				       struct sk_buff *skb);
	int (*gro_complete)(struct sk_buff *skb, int nhoff);
};
```

Each protocol provides `gro_receive`, which decides whether an incoming packet can be merged with a pending one. TCP's checks are strict: same 4-tuple, consecutive sequence numbers, compatible flags, same TCP options, no urgent data. If they match, the payload is appended as a fragment and the header counts updated.

**The critical property: GRO is reversible.** The merged skb records `gso_size` and `gso_segs`, so if it must be forwarded, GSO (below) can split it back into the original packets. That is what makes GRO safe to enable on a router.

**LRO (Large Receive Offload)** is the hardware equivalent and is **not** reversible — the hardware discards information. Enabling LRO on a forwarding machine violates the end-to-end principle: you cannot reproduce the original packets, so you may emit packets that differ from what arrived. **LRO must be disabled on any machine that forwards or bridges**, and the kernel enforces this by disabling it automatically when forwarding is enabled.

**GSO (Generic Segmentation Offload)** is the transmit side. The stack builds one large skb (up to 64 KB) and either:

- **TSO/USO**: hands it to hardware that segments it, if `NETIF_F_TSO` is set, or
- **GSO**: segments it in software just before `ndo_start_xmit`.

Either way, the expensive per-packet work (routing lookup, netfilter traversal, qdisc enqueue) happens **once per 64 KB rather than once per 1500 bytes** — a 40× reduction. That is the single largest throughput optimisation in the Linux network stack.

`skb_shinfo(skb)->gso_size` is the segment size; `gso_type` says what kind (TCPV4, TCPV6, UDP_L4, GRE, UDP_TUNNEL, ...). `skb_gso_segment()` does the software split.

Measured: disabling GSO and GRO typically costs 50–80 % of throughput on a 10 Gbit link. Lab 71.5 measures it.

### T.9 The transmit path and queueing

```
sendmsg()
 └─ tcp_sendmsg() -> builds skbs, may wait for window/memory
     └─ tcp_write_xmit() -> tcp_transmit_skb()
         └─ ip_queue_xmit()
             ├─ routing lookup (or cached dst)
             ├─ build the IP header
             ├─ NF_INET_LOCAL_OUT, NF_INET_POST_ROUTING
             └─ ip_finish_output() -> neighbour resolution (ARP)
                 └─ dev_queue_xmit(skb)
                     ├─ netdev_pick_tx() -> select a TX queue
                     ├─ tc egress (clsact)   -- Ch. 73
                     ├─ qdisc enqueue        -- Ch. 73
                     └─ __qdisc_run() -> sch_direct_xmit()
                         ├─ validate_xmit_skb(): GSO, checksum, linearize
                         └─ netdev_start_xmit() -> ndo_start_xmit()
                             └─ driver: build descriptors, DMA map, ring doorbell
```

Three mechanisms deserve attention.

**Multiqueue and queue selection.** `netdev_pick_tx()` chooses a TX queue, usually by `skb->hash` (so a flow always uses the same queue, preserving ordering) or by `XPS` (Transmit Packet Steering), which maps CPUs to queues so a CPU always uses its own queue and avoids lock contention:

```sh
echo f > /sys/class/net/eth0/queues/tx-0/xps_cpus
```

**The stop-queue race** is the classic driver bug, and it is a memory-ordering problem:

```c
	/* WRONG */
	if (tx_ring_full(ring))
		netif_stop_queue(dev);
	/* the TX completion handler may have just freed everything here,
	 * seen a stopped queue, and woken it -- but we stop it AFTER.
	 * Result: a permanently stopped queue. */

	/* RIGHT */
	netif_stop_queue(dev);
	smp_mb();                      /* pair with the completion path */
	if (tx_ring_has_space(ring))
		netif_wake_queue(dev);
```

This is a store-buffer litmus test (Ch. 12 §T.5) in a driver, and getting it wrong produces a device that stops transmitting under load and recovers only on a TX timeout. `Documentation/networking/driver.rst` documents the correct pattern.

**BQL (Byte Queue Limits)** addresses bufferbloat *inside the driver*. A large TX ring can hold hundreds of milliseconds of packets at low link rates, and those packets are past the qdisc, so no scheduling or AQM applies to them. BQL limits the *bytes* (not packets) in flight to the hardware, adapting to the observed drain rate:

```c
	netdev_tx_sent_queue(txq, skb->len);        /* on transmit */
	netdev_tx_completed_queue(txq, pkts, bytes); /* on completion */
```

```sh
cat /sys/class/net/eth0/queues/tx-0/byte_queue_limits/limit
cat /sys/class/net/eth0/queues/tx-0/byte_queue_limits/inflight
```

BQL typically reduces latency under load by an order of magnitude on a congested link, at negligible throughput cost. A driver that does not implement it is leaving that on the table.

### T.10 XDP: before any of it

XDP (eXpress Data Path) runs an eBPF program **in the driver's receive path, before an skb is allocated**:

```c
enum xdp_action {
	XDP_ABORTED = 0,
	XDP_DROP,        /* free the page; never allocate an skb */
	XDP_PASS,        /* continue to the normal stack */
	XDP_TX,          /* transmit back out the same interface */
	XDP_REDIRECT,    /* to another interface, a CPU, or AF_XDP */
};
```

Because no skb exists yet, `XDP_DROP` costs almost nothing — measured at 20–30 million packets per second per core, against roughly 1–2 Mpps for an iptables drop. That makes it the right tool for DDoS mitigation and for load balancing.

Modes:

| Mode | Where | Speed |
|---|---|---|
| **Native** | in the driver, on the raw page | fastest |
| **Offloaded** | on the NIC (Netronome) | fastest, limited programs |
| **Generic** (`xdpgeneric`) | after skb allocation, in `netif_receive_skb` | **slow**; for testing only |

Generic mode exists so you can develop without driver support, but it defeats the entire purpose — an skb has already been allocated. Benchmarks run in generic mode are meaningless.

Chapter 74 covers XDP and AF_XDP properly. What matters here is where it sits: **before GRO, before the skb, before every hook in §T.7.**

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `include/linux/skbuff.h` ★★★ | §T.2 and §T.3; the accessors are the API |
| `net/core/skbuff.c` ★★★ | allocation, cloning, copying, fragment handling |
| `net/core/dev.c` ★★★ | **the core**: `netif_receive_skb`, `dev_queue_xmit`, NAPI, `net_rx_action` |
| `net/core/gro.c` ★★★ | §T.8's GRO engine |
| `net/core/gso.c` | `skb_gso_segment` |
| `net/core/page_pool.c` ★★★ | §T.4(c) |
| `net/core/net-sysfs.c` | `/sys/class/net/*` |
| `net/core/rtnetlink.c` | the configuration interface |
| `net/core/dev_addr_lists.c` | MAC address lists |
| `net/core/netpoll.c` | netconsole, kgdboe |
| `include/linux/netdevice.h` ★★★ | §T.5 |
| `drivers/net/virtio_net.c` ★★★ | a readable, complete modern driver |
| `drivers/net/ethernet/intel/igb/`, `ixgbe/` ★★★ | production reference drivers |
| `drivers/net/veth.c`, `dummy.c`, `loopback.c` | trivial drivers worth reading |
| `Documentation/networking/` ★★★ | `driver.rst`, `napi.rst`, `scaling.rst`, `segmentation-offloads.rst`, `skbuff.rst` |

### 1.2 skb allocation and the NAPI path

```c
struct sk_buff *__alloc_skb(unsigned int size, gfp_t gfp_mask, int flags,
			    int node)
{
	struct kmem_cache *cache;
	struct sk_buff *skb;
	bool pfmemalloc;
	u8 *data;

	cache = (flags & SKB_ALLOC_FCLONE)
		? net_hotdata.skbuff_fclone_cache : net_hotdata.skbuff_cache;
	...
	skb = kmem_cache_alloc_node(cache, gfp_mask & ~GFP_DMA, node);
	if (unlikely(!skb))
		return NULL;
	prefetchw(skb);

	/* The linear buffer, PLUS room for skb_shared_info at the end. */
	size = SKB_DATA_ALIGN(size);
	size += SKB_DATA_ALIGN(sizeof(struct skb_shared_info));
	osize = kmalloc_size_roundup(size);
	data = kmalloc_reserve(&osize, gfp_mask, node, &pfmemalloc);
	if (unlikely(!data))
		goto nodata;
	size = SKB_WITH_OVERHEAD(osize);

	__build_skb_around(skb, data, osize);
	skb->pfmemalloc = pfmemalloc;
	...
	return skb;
nodata:
	kmem_cache_free(cache, skb);
	return NULL;
}
```

**Two allocations**: the `sk_buff` from a slab cache, and the data buffer. `build_skb()` skips the second when the driver already has the data:

```c
struct sk_buff *build_skb(void *data, unsigned int frag_size)
{
	struct sk_buff *skb = __build_skb(data, frag_size);

	if (likely(skb && frag_size)) {
		skb->head_frag = 1;
		skb_propagate_pfmemalloc(virt_to_head_page(data), skb);
	}
	return skb;
}
```

And the per-CPU NAPI cache avoids even the slab:

```c
struct sk_buff *napi_alloc_skb(struct napi_struct *napi, unsigned int len)
{
	struct napi_alloc_cache *nc;
	struct sk_buff *skb;
	...
	if (len <= SKB_WITH_OVERHEAD(1024) || len > SKB_WITH_OVERHEAD(PAGE_SIZE) ||
	    (gfp_mask & (__GFP_DIRECT_RECLAIM | GFP_DMA))) {
		skb = __napi_alloc_skb(napi, len, gfp_mask | __GFP_NOWARN);
		...
	}

	nc = this_cpu_ptr(&napi_alloc_cache);
	...
	data = page_frag_alloc(&nc->page, len, gfp_mask);
	...
	skb = __napi_build_skb(data, len);
	...
}
```

### 1.3 NAPI's core loop

```c
static __latent_entropy void net_rx_action(struct softirq_action *h)
{
	struct softnet_data *sd = this_cpu_ptr(&softnet_data);
	unsigned long time_limit = jiffies +
		usecs_to_jiffies(READ_ONCE(net_hotdata.netdev_budget_usecs));
	int budget = READ_ONCE(net_hotdata.netdev_budget);
	LIST_HEAD(list);
	LIST_HEAD(repoll);

start:
	sd->in_net_rx_action = true;
	local_irq_disable();
	list_splice_init(&sd->poll_list, &list);
	local_irq_enable();

	for (;;) {
		struct napi_struct *n;

		skb_defer_free_flush(sd);

		if (list_empty(&list)) {
			if (list_empty(&repoll)) { ... goto end; }
			break;
		}

		n = list_first_entry(&list, struct napi_struct, poll_list);
		budget -= napi_poll(n, &repoll);

		/* T.6: bounded by BOTH work and time. */
		if (unlikely(budget <= 0 ||
			     time_after_eq(jiffies, time_limit))) {
			sd->time_squeeze++;          /* the signal to watch */
			break;
		}
	}
	...
	if (!list_empty(&list) || !list_empty(&repoll))
		__raise_softirq_irqoff(NET_RX_SOFTIRQ);   /* -> ksoftirqd */
	...
}

static int __napi_poll(struct napi_struct *n, bool *repoll)
{
	int work, weight = n->weight;

	weight = n->weight;
	work = 0;
	if (test_bit(NAPI_STATE_SCHED, &n->state)) {
		work = n->poll(n, weight);
		trace_napi_poll(n, work, weight);
		...
	}

	if (unlikely(work > weight))
		netdev_err_once(n->dev, "NAPI poll function %pS returned %d, exceeding its budget of %d.\n",
				n->poll, work, weight);

	if (likely(work < weight))
		return work;     /* the driver called napi_complete_done */
	...
	*repoll = true;
	return work;
}
```

`sd->time_squeeze` is the counter to watch: it counts times the budget or time limit was hit, meaning the CPU could not keep up and deferred to `ksoftirqd`.

### 1.4 `netif_receive_skb`

```c
static int __netif_receive_skb_core(struct sk_buff **pskb, bool pfmemalloc,
				    struct packet_type **ppt_prev)
{
	struct packet_type *ptype, *pt_prev;
	struct sk_buff *skb = *pskb;
	struct net_device *orig_dev;
	bool deliver_exact = false;
	int ret = NET_RX_DROP;
	__be16 type;

	net_timestamp_check(!READ_ONCE(net_hotdata.tstamp_prequeue), skb);
	orig_dev = skb->dev;
	skb_reset_network_header(skb);
	...
another_round:
	skb->skb_iif = skb->dev->ifindex;
	__this_cpu_inc(softnet_data.processed);

	if (static_branch_unlikely(&generic_xdp_needed_key)) {
		/* T.10: generic XDP -- AFTER the skb exists */
		ret2 = do_xdp_generic(rcu_dereference(skb->dev->xdp_prog), &skb);
		...
	}
	...
	/* Every packet, to every tap: tcpdump lives here. */
	list_for_each_entry_rcu(ptype, &net_hotdata.ptype_all, list) {
		if (pt_prev)
			ret = deliver_skb(skb, pt_prev, orig_dev);
		pt_prev = ptype;
	}
	...
	/* tc ingress -- Ch. 73 */
	skb = sch_handle_ingress(skb, &pt_prev, &ret, orig_dev, &another_round);
	if (!skb) goto out;
	...
	if (skb_vlan_tag_present(skb)) { ... }
	...
	/* Deliver to the protocol handler, by skb->protocol */
	type = skb->protocol;
	if (likely(!deliver_exact))
		deliver_ptype_list_skb(skb, &pt_prev, orig_dev, type,
				       &ptype_base[ntohs(type) & PTYPE_HASH_MASK]);
	...
}
```

Note the ordering: taps first (so `tcpdump` sees everything), then tc ingress, then the protocol handler. Anything dropped by tc is invisible to the protocol stack but visible to `tcpdump` — which is why `tcpdump` showing a packet that the application never receives is a normal and important diagnostic.

### 1.5 GRO

```c
static enum gro_result dev_gro_receive(struct napi_struct *napi,
				       struct sk_buff *skb)
{
	u32 bucket = skb_get_hash_raw(skb) & (GRO_HASH_BUCKETS - 1);
	struct gro_list *gro_list = &napi->gro_hash[bucket];
	struct list_head *head = &net_hotdata.offload_base;
	struct packet_offload *ptype;
	__be16 type = skb->protocol;
	struct sk_buff *pp = NULL;
	enum gro_result ret;
	int same_flow;

	if (netif_elide_gro(skb->dev))
		goto normal;

	gro_list_prepare(&gro_list->list, skb);

	list_for_each_entry_rcu(ptype, head, list) {
		if (ptype->type == type && ptype->callbacks.gro_receive)
			goto found_ptype;
	}
	goto normal;

found_ptype:
	skb_set_network_header(skb, skb_gro_offset(skb));
	skb_reset_mac_len(skb);
	BUILD_BUG_ON(sizeof_field(struct napi_gro_cb, zeroed) != sizeof(u32));
	...
	pp = INDIRECT_CALL_INET(ptype->callbacks.gro_receive,
				ipv6_gro_receive, inet_gro_receive,
				&gro_list->list, skb);
	...
	same_flow = NAPI_GRO_CB(skb)->same_flow;
	ret = NAPI_GRO_CB(skb)->free ? GRO_MERGED_FREE : GRO_MERGED;

	if (pp) {
		skb_list_del_init(pp);
		napi_gro_complete(napi, pp);    /* flush this one */
		gro_list->count--;
	}
	if (same_flow)
		goto ok;
	if (NAPI_GRO_CB(skb)->flush)
		goto normal;
	...
}
```

And TCP's merge decision:

```c
struct sk_buff *tcp_gro_receive(struct list_head *head, struct sk_buff *skb)
{
	struct sk_buff *pp = NULL;
	struct tcphdr *th2;
	unsigned int thlen;
	unsigned int flags;
	...
	th = tcp_hdr(skb);
	thlen = th->doff * 4;
	...
	flags = tcp_flag_word(th);

	list_for_each_entry(p, head, list) {
		if (!NAPI_GRO_CB(p)->same_flow)
			continue;

		th2 = tcp_hdr(p);

		/* The four-tuple and header must match exactly. */
		if (*(u32 *)&th->source ^ *(u32 *)&th2->source) {
			NAPI_GRO_CB(p)->same_flow = 0;
			continue;
		}
		goto found;
	}
	goto out_check_final;

found:
	...
	/* Sequence numbers must be CONSECUTIVE. */
	flush = (u16)((ntohl(*(__be32 *)th) ^ ntohl(*(__be32 *)th2)) |
		      (__force unsigned int)(th->ack_seq ^ th2->ack_seq));
	...
	if (flush || skb_gro_receive(p, skb)) {
		mss = 1;
		goto out_check_final;
	}
	...
out_check_final:
	/* Flush if: small segment, FIN/SYN/RST, or the queue is full. */
	flush = len < mss;
	flush |= (__force int)(flags & (TCP_FLAG_URG | TCP_FLAG_PSH |
					TCP_FLAG_RST | TCP_FLAG_SYN |
					TCP_FLAG_FIN));
	...
}
```

**A FIN, SYN, RST, PSH, or URG flushes immediately** — GRO must not delay a connection-state-changing packet, and must not delay data the sender asked to be pushed.

### 1.6 Transmit

```c
static int __dev_queue_xmit(struct sk_buff *skb, struct net_device *sb_dev)
{
	struct net_device *dev = skb->dev;
	struct netdev_queue *txq = NULL;
	struct Qdisc *q;
	int rc = -ENOMEM;
	bool again = false;

	skb_reset_mac_header(skb);
	skb_assert_len(skb);
	...
	rcu_read_lock_bh();
	skb_update_prio(skb);

	qdisc_pkt_len_init(skb);
	tcx_set_ingress(skb, false);
#ifdef CONFIG_NET_EGRESS
	if (static_branch_unlikely(&egress_needed_key)) {
		if (nf_skip_egress(skb, true))
			goto no_lock_out;
		skb = sch_handle_egress(skb, &rc, dev);     /* tc egress */
		if (!skb) goto out;
		...
	}
#endif
	txq = netdev_core_pick_tx(dev, skb, sb_dev);    /* T.9 */
	q = rcu_dereference_bh(txq->qdisc);

	trace_net_dev_queue(skb);
	if (q->enqueue) {
		rc = __dev_xmit_skb(skb, q, dev, txq);      /* qdisc path */
		goto out;
	}

	/* No qdisc (e.g. loopback, or NETIF_F_LLTX): direct transmit. */
	if (dev->flags & IFF_UP) {
		int cpu = smp_processor_id();

		if (READ_ONCE(txq->xmit_lock_owner) != cpu) {
			...
			HARD_TX_LOCK(dev, txq, cpu);
			if (!netif_xmit_stopped(txq)) {
				...
				skb = dev_hard_start_xmit(skb, dev, txq, &rc);
				...
			}
			HARD_TX_UNLOCK(dev, txq);
			...
		}
	}
	...
}

struct sk_buff *dev_hard_start_xmit(struct sk_buff *first,
				    struct net_device *dev,
				    struct netdev_queue *txq, int *ret)
{
	struct sk_buff *skb = first;
	int rc = NETDEV_TX_OK;

	while (skb) {
		struct sk_buff *next = skb->next;

		skb_mark_not_on_list(skb);
		rc = xmit_one(skb, dev, txq, next != NULL);
		if (unlikely(!dev_xmit_complete(rc))) {
			skb->next = next;
			goto out;
		}

		skb = next;
		if (netif_tx_queue_stopped(txq) && skb) {
			rc = NETDEV_TX_BUSY;
			break;
		}
	}
out:
	*ret = rc;
	return skb;
}
```

`validate_xmit_skb()` (called from `xmit_one`'s path) is where GSO segmentation, checksum computation, and linearisation happen if the device cannot handle what the stack built:

```c
static struct sk_buff *validate_xmit_skb(struct sk_buff *skb,
					 struct net_device *dev, bool *again)
{
	netdev_features_t features;

	features = netif_skb_features(skb);
	skb = validate_xmit_vlan(skb, features);
	if (unlikely(!skb)) goto out_null;
	...
	if (netif_needs_gso(skb, features)) {
		struct sk_buff *segs;

		segs = skb_gso_segment(skb, features);   /* T.8: software GSO */
		...
	} else {
		if (skb_needs_linearize(skb, features) &&
		    __skb_linearize(skb))
			goto out_kfree_skb;
		...
		if (skb->ip_summed == CHECKSUM_PARTIAL) {
			if (skb->encapsulation)
				skb_set_inner_transport_header(skb, ...);
			if (skb_csum_hwoffload_help(skb, features))
				goto out_kfree_skb;
		}
	}
	...
}
```

### 1.7 Observability

| Where | What |
|---|---|
| `ip -s link` ★★★ | per-interface packet/byte/error/drop counters |
| `ethtool -S eth0` ★★★ | **driver-specific counters** — the real diagnostic |
| `ethtool -k eth0` ★★★ | offload feature state |
| `ethtool -g eth0` | ring sizes |
| `ethtool -c eth0` | interrupt coalescing |
| `ethtool -l eth0` | queue counts |
| `/proc/net/softnet_stat` ★★★ | **per-CPU: processed, dropped, `time_squeeze`** |
| `/proc/net/dev` | the same as `ip -s link` |
| `/sys/class/net/eth0/queues/{rx,tx}-N/` ★★★ | RPS, XPS, BQL |
| `/sys/class/net/eth0/threaded` | threaded NAPI |
| `tcpdump`, `tshark` ★★★ | |
| `trace-cmd record -e net:\* -e napi:\* -e skb:\*` ★★★ | |
| `dropwatch`, `perf trace -e skb:kfree_skb` ★★★ | **where packets are dropped** |
| `bpftrace` on `kfree_skb_reason` ★★★ | the drop reason enum |
| `ss -i` | per-socket detail |
| `nstat`, `netstat -s` ★★★ | protocol counters |

**`/proc/net/softnet_stat` is the first thing to read** on a receive performance problem. Column 1 is packets processed, column 2 is dropped (the backlog was full), column 3 is `time_squeeze`.

**`kfree_skb_reason`** (5.17+) is a substantial improvement: every drop site passes a reason code, so you can ask *why* packets are being dropped rather than just counting:

```sh
sudo bpftrace -e 'tracepoint:skb:kfree_skb { @[args->reason] = count(); }'
```

---

## 2. Practice

### Lab 71.1 — Anatomy of an skb

```sh
sudo apt install -y bpftrace linux-tools-common iproute2 ethtool tcpdump
```

Watch skbs being created and destroyed:

```sh
sudo bpftrace -e '
kprobe:__alloc_skb  { @alloc = count(); }
kprobe:build_skb    { @build = count(); }
kprobe:napi_alloc_skb { @napi = count(); }
kprobe:kfree_skb    { @free = count(); }
kprobe:consume_skb  { @consume = count(); }
interval:s:5 { print(@alloc); print(@build); print(@napi);
               print(@free); print(@consume);
               clear(@alloc); clear(@build); clear(@napi);
               clear(@free); clear(@consume); }' &

ping -c 20 -i 0.1 127.0.0.1 > /dev/null
iperf3 -s -1 > /dev/null 2>&1 &
sleep 1
iperf3 -c 127.0.0.1 -t 5 > /dev/null 2>&1
```

Inspect the fields:

```sh
sudo bpftrace -e '
kprobe:ip_rcv {
	$skb = (struct sk_buff *)arg0;
	printf("len=%-6d data_len=%-6d truesize=%-6d headroom=%-4d cloned=%d users=%d\n",
	       $skb->len, $skb->data_len, $skb->truesize,
	       $skb->data - $skb->head,
	       $skb->cloned,
	       $skb->users.refs.counter);
}' &
ping -c 5 8.8.8.8 2>/dev/null || ping -c 5 127.0.0.1
```

Headroom — §T.2:

```sh
sudo bpftrace -e '
kprobe:netif_receive_skb {
	$skb = (struct sk_buff *)arg0;
	@headroom = hist($skb->data - $skb->head);
	@tailroom = hist($skb->end - $skb->tail);
}
interval:s:10 { print(@headroom); print(@tailroom); exit(); }' &
ping -c 50 -i 0.05 127.0.0.1 > /dev/null
wait
```

Linear versus non-linear — §T.3:

```sh
sudo bpftrace -e '
kprobe:tcp_v4_rcv {
	$skb = (struct sk_buff *)arg0;
	if ($skb->data_len > 0) { @nonlinear = count(); @frag_bytes = hist($skb->data_len); }
	else { @linear = count(); }
}
interval:s:10 { print(@linear); print(@nonlinear); print(@frag_bytes); exit(); }' &

iperf3 -s -1 > /dev/null 2>&1 &
sleep 1
iperf3 -c 127.0.0.1 -t 8 > /dev/null 2>&1
wait
```

Cloning:

```sh
sudo bpftrace -e '
kprobe:skb_clone { @clone = count(); }
kprobe:skb_copy  { @copy = count(); }
kprobe:pskb_expand_head { @expand = count(); }
kprobe:__pskb_pull_tail { @pull_tail = count(); }
interval:s:5 { print(@clone); print(@copy); print(@expand); print(@pull_tail);
               clear(@clone); clear(@copy); clear(@expand); clear(@pull_tail); }' &

# tcpdump clones EVERY packet
sudo tcpdump -i lo -c 1000 -w /dev/null 2>/dev/null &
iperf3 -s -1 > /dev/null 2>&1 &
sleep 1; iperf3 -c 127.0.0.1 -t 5 > /dev/null 2>&1
```

Size distribution:

```sh
sudo bpftrace -e '
tracepoint:net:net_dev_xmit { @tx_len = hist(args->len); }
tracepoint:net:netif_receive_skb { @rx_len = hist(args->len); }
interval:s:15 { print(@tx_len); print(@rx_len); exit(); }' &
iperf3 -s -1 > /dev/null 2>&1 &
sleep 1; iperf3 -c 127.0.0.1 -t 12 > /dev/null 2>&1
wait
```

**Look for sizes far above the MTU** — those are GSO/GRO aggregates (§T.8).

---

### Lab 71.2 — Trace a packet end to end

```sh
# Set up an isolated pair
sudo ip netns add ns1
sudo ip link add veth0 type veth peer name veth1
sudo ip link set veth1 netns ns1
sudo ip addr add 10.99.0.1/24 dev veth0
sudo ip link set veth0 up
sudo ip netns exec ns1 ip addr add 10.99.0.2/24 dev veth1
sudo ip netns exec ns1 ip link set veth1 up
sudo ip netns exec ns1 ip link set lo up

ping -c 3 10.99.0.2
```

The full receive path — §T.7:

```sh
sudo trace-cmd record -e net:\* -e napi:\* -e skb:\* -- ping -c 3 10.99.0.2 > /dev/null
sudo trace-cmd report | head -40
```

With a stack trace at each stage:

```sh
sudo bpftrace -e '
kprobe:netif_receive_skb,
kprobe:ip_rcv,
kprobe:ip_rcv_finish,
kprobe:ip_local_deliver,
kprobe:icmp_rcv,
kprobe:__dev_queue_xmit,
kprobe:dev_hard_start_xmit
{
	printf("%-24s comm=%s\n", probe, comm);
}' &
ping -c 2 10.99.0.2 > /dev/null
```

Time it:

```sh
sudo bpftrace -e '
kprobe:netif_receive_skb { @start[arg0] = nsecs; }
kprobe:ip_rcv /@start[arg0]/ { @to_ip = hist(nsecs - @start[arg0]); }
kprobe:tcp_v4_rcv /@start[arg0]/ { @to_tcp = hist(nsecs - @start[arg0]); }
kprobe:tcp_queue_rcv /@start[arg0]/ {
	@to_socket = hist(nsecs - @start[arg0]); delete(@start[arg0]);
}
interval:s:10 { print(@to_ip); print(@to_tcp); print(@to_socket); exit(); }' &

iperf3 -s -1 > /dev/null 2>&1 &
sleep 1; iperf3 -c 127.0.0.1 -t 8 > /dev/null 2>&1
wait
```

The transmit path:

```sh
sudo trace-cmd record -e net:net_dev_queue -e net:net_dev_start_xmit \
                      -e net:net_dev_xmit -e qdisc:\* -- \
  ping -c 3 10.99.0.2 > /dev/null
sudo trace-cmd report | head -30
```

**Where packets are dropped** — the single most useful networking diagnostic:

```sh
sudo bpftrace -e '
tracepoint:skb:kfree_skb {
	@drops[args->reason, ksym(args->location)] = count();
}
interval:s:10 { print(@drops); clear(@drops); }' &

# Generate some drops
sudo ip netns exec ns1 iptables -A INPUT -p icmp -j DROP 2>/dev/null
ping -c 5 -W 1 10.99.0.2 > /dev/null 2>&1
ping -c 3 -W 1 10.99.0.99 > /dev/null 2>&1    # no such host
nc -z -w1 10.99.0.2 12345 2>/dev/null          # closed port
```

```sh
# The reason codes:
grep -A80 'enum skb_drop_reason' /usr/src/linux*/include/net/dropreason-core.h 2>/dev/null | head -50
# Or: bpftrace -lv 'tracepoint:skb:kfree_skb'
```

`dropwatch`:

```sh
sudo apt install -y dropwatch
sudo dropwatch -l kas <<'EOF'
start
EOF
```

---

### Lab 71.3 — NAPI

```sh
IFACE=$(ip -o -4 route show default | awk '{print $5}' | head -1)
echo "using $IFACE"

cat /proc/net/softnet_stat
echo "cols: processed dropped time_squeeze 0 0 0 0 0 cpu_collision received_rps flow_limit_count"
```

Decode it:

```sh
awk '{printf "cpu%-3d processed=%-12d dropped=%-8d time_squeeze=%-8d\n",
      NR-1, strtonum("0x"$1), strtonum("0x"$2), strtonum("0x"$3)}' /proc/net/softnet_stat
```

**`time_squeeze` non-zero means the budget was exhausted** — the CPU could not keep up.

Watch NAPI poll:

```sh
sudo bpftrace -e '
tracepoint:napi:napi_poll {
	@work = hist(args->work);
	@by_dev[str(args->dev_name)] = count();
}
interval:s:10 { print(@work); print(@by_dev); exit(); }' &

iperf3 -s -1 > /dev/null 2>&1 &
sleep 1; iperf3 -c 127.0.0.1 -t 8 -P 4 > /dev/null 2>&1
wait
```

A poll returning exactly `budget` means the ring was not drained — the system is in sustained polling mode.

Budget tuning:

```sh
cat /proc/sys/net/core/netdev_budget
cat /proc/sys/net/core/netdev_budget_usecs
cat /proc/sys/net/core/netdev_max_backlog

for b in 64 300 1000; do
  echo $b | sudo tee /proc/sys/net/core/netdev_budget > /dev/null
  BEFORE=$(awk '{s+=strtonum("0x"$3)} END {print s}' /proc/net/softnet_stat)
  iperf3 -s -1 > /dev/null 2>&1 &
  sleep 0.5; iperf3 -c 127.0.0.1 -t 5 -P 8 2>/dev/null | grep -E 'receiver'
  AFTER=$(awk '{s+=strtonum("0x"$3)} END {print s}' /proc/net/softnet_stat)
  echo "  budget=$b time_squeeze delta=$((AFTER-BEFORE))"
done
echo 300 | sudo tee /proc/sys/net/core/netdev_budget > /dev/null
```

Threaded NAPI — §T.6:

```sh
cat /sys/class/net/$IFACE/threaded 2>/dev/null
echo 1 | sudo tee /sys/class/net/$IFACE/threaded 2>/dev/null
ps aux | grep -i 'napi' | head
# Now NAPI is a schedulable thread:
pgrep -f "napi/$IFACE" | while read p; do
  echo "pid $p: $(cat /proc/$p/comm) prio=$(ps -o ni= -p $p)"
done
echo 0 | sudo tee /sys/class/net/$IFACE/threaded 2>/dev/null
```

Interrupt coalescing:

```sh
sudo ethtool -c $IFACE 2>/dev/null
# Fewer interrupts, more latency:
# sudo ethtool -C $IFACE rx-usecs 100 rx-frames 64
# Adaptive:
# sudo ethtool -C $IFACE adaptive-rx on
```

Interrupt counts:

```sh
grep -i $IFACE /proc/interrupts | head
BEFORE=$(grep -i $IFACE /proc/interrupts | awk '{for(i=2;i<=NF-2;i++) s+=$i} END {print s}')
iperf3 -s -1 > /dev/null 2>&1 &
sleep 0.5; iperf3 -c 127.0.0.1 -t 5 > /dev/null 2>&1
AFTER=$(grep -i $IFACE /proc/interrupts | awk '{for(i=2;i<=NF-2;i++) s+=$i} END {print s}')
echo "interrupts: $((AFTER-BEFORE))"
```

Busy polling:

```sh
cat /proc/sys/net/core/busy_poll
cat /proc/sys/net/core/busy_read
echo 50 | sudo tee /proc/sys/net/core/busy_read > /dev/null
# Then measure latency with sockperf or a ping-pong program.
echo 0 | sudo tee /proc/sys/net/core/busy_read > /dev/null
```

---

### Lab 71.4 — GRO and GSO

```sh
sudo ethtool -k $IFACE | grep -E 'generic-receive-offload|generic-segmentation|tcp-segmentation|large-receive'
```

Watch aggregation happen:

```sh
sudo bpftrace -e '
kprobe:napi_gro_receive { @gro_in = count(); }
kprobe:netif_receive_skb { @to_stack = count(); }
kprobe:tcp_gro_receive { @tcp_gro = count(); }
kprobe:skb_gso_segment { @gso_seg = count(); }
interval:s:5 { print(@gro_in); print(@to_stack); print(@tcp_gro); print(@gso_seg);
               clear(@gro_in); clear(@to_stack); clear(@tcp_gro); clear(@gso_seg); }' &

iperf3 -s -1 > /dev/null 2>&1 &
sleep 1; iperf3 -c 127.0.0.1 -t 8 > /dev/null 2>&1
```

`gro_in` ≫ `to_stack` is GRO working.

Packet sizes above and below GRO:

```sh
sudo bpftrace -e '
kprobe:napi_gro_receive {
	$skb = (struct sk_buff *)arg1;
	@before_gro = hist($skb->len);
}
kprobe:tcp_v4_rcv {
	$skb = (struct sk_buff *)arg0;
	@after_gro = hist($skb->len);
	if ((uint64)skb_shinfo_gso_size($skb) > 0) { @gso = count(); }
}
interval:s:12 { print(@before_gro); print(@after_gso); exit(); }' 2>/dev/null &

# Simpler:
sudo bpftrace -e '
kprobe:napi_gro_receive { $s = (struct sk_buff *)arg1; @wire = hist($s->len); }
kprobe:tcp_v4_rcv       { $s = (struct sk_buff *)arg0; @stack = hist($s->len); }
interval:s:12 { print(@wire); print(@stack); exit(); }' &
iperf3 -s -1 > /dev/null 2>&1 &
sleep 1; iperf3 -c 127.0.0.1 -t 10 > /dev/null 2>&1
wait
```

**Measure the cost** — §T.8's claim:

```sh
# A veth pair gives a clean measurement
sudo ip netns exec ns1 iperf3 -s -D 2>/dev/null
sleep 1

for setting in "on" "off"; do
  sudo ethtool -K veth0 gro $setting gso $setting tso $setting 2>/dev/null
  sudo ip netns exec ns1 ethtool -K veth1 gro $setting gso $setting tso $setting 2>/dev/null
  echo -n "gro/gso/tso $setting: "
  iperf3 -c 10.99.0.2 -t 5 2>/dev/null | grep receiver | awk '{print $7, $8}'
done
sudo ethtool -K veth0 gro on gso on tso on 2>/dev/null
sudo ip netns exec ns1 ethtool -K veth1 gro on gso on tso on 2>/dev/null
```

GSO segment counts:

```sh
sudo bpftrace -e '
kprobe:skb_gso_segment {
	$skb = (struct sk_buff *)arg0;
	printf("gso: len=%d\n", $skb->len);
	@gso_len = hist($skb->len);
}
kretprobe:skb_gso_segment { @segmented = count(); }
interval:s:10 { print(@gso_len); print(@segmented); exit(); }' &

sudo ethtool -K veth0 tso off 2>/dev/null    # force software GSO
iperf3 -c 10.99.0.2 -t 8 > /dev/null 2>&1
wait
sudo ethtool -K veth0 tso on 2>/dev/null
```

Why LRO is dangerous — §T.8:

```sh
sudo ethtool -k $IFACE | grep large-receive
# Enable forwarding and watch LRO be disabled automatically:
sudo sysctl -w net.ipv4.ip_forward=1
sudo ethtool -K $IFACE lro on 2>&1 | tail -2
sudo ethtool -k $IFACE | grep large-receive
sudo sysctl -w net.ipv4.ip_forward=0
```

Per-feature effect:

```sh
for feat in gro gso tso sg rx-checksumming tx-checksumming; do
  sudo ethtool -K veth0 $feat off 2>/dev/null || continue
  echo -n "$feat off: "
  iperf3 -c 10.99.0.2 -t 4 2>/dev/null | grep receiver | awk '{print $7, $8}'
  sudo ethtool -K veth0 $feat on 2>/dev/null
done
```

---

### Lab 71.5 — Multiqueue, RSS, RPS, and XPS

```sh
sudo ethtool -l $IFACE 2>/dev/null
ls /sys/class/net/$IFACE/queues/
nproc
```

Where do packets land?

```sh
sudo bpftrace -e '
kprobe:netif_receive_skb { @rx_cpu[cpu] = count(); }
kprobe:__dev_queue_xmit  { @tx_cpu[cpu] = count(); }
interval:s:10 { print(@rx_cpu); print(@tx_cpu); exit(); }' &

iperf3 -s -1 > /dev/null 2>&1 &
sleep 1; iperf3 -c 127.0.0.1 -t 8 -P 8 > /dev/null 2>&1
wait
```

RPS — §T.7:

```sh
for q in /sys/class/net/$IFACE/queues/rx-*/; do
  echo "$(basename $q): rps_cpus=$(cat $q/rps_cpus) rps_flow_cnt=$(cat $q/rps_flow_cnt)"
done

# Spread across all CPUs
MASK=$(printf '%x' $(( (1 << $(nproc)) - 1 )))
for q in /sys/class/net/veth0/queues/rx-*/; do
  echo $MASK | sudo tee $q/rps_cpus > /dev/null
done

echo 32768 | sudo tee /proc/sys/net/core/rps_sock_flow_entries > /dev/null
for q in /sys/class/net/veth0/queues/rx-*/; do
  echo 2048 | sudo tee $q/rps_flow_cnt > /dev/null
done

sudo bpftrace -e '
kprobe:netif_receive_skb { @rx_cpu[cpu] = count(); }
kprobe:enqueue_to_backlog { @rps_enqueued = count(); }
interval:s:10 { print(@rx_cpu); print(@rps_enqueued); exit(); }' &
sudo ip netns exec ns1 iperf3 -s -D 2>/dev/null
sleep 1; iperf3 -c 10.99.0.2 -t 8 -P 4 > /dev/null 2>&1
wait

# Check the RPS counter
awk '{printf "cpu%-3d received_rps=%d\n", NR-1, strtonum("0x"$10)}' /proc/net/softnet_stat
```

XPS — §T.9:

```sh
for q in /sys/class/net/$IFACE/queues/tx-*/; do
  echo "$(basename $q): xps_cpus=$(cat $q/xps_cpus 2>/dev/null)"
done

# Typical: CPU N -> queue N
i=0
for q in /sys/class/net/$IFACE/queues/tx-*/; do
  printf '%x' $((1 << i)) | sudo tee $q/xps_cpus > /dev/null 2>&1
  i=$((i+1))
done
```

Flow hashing:

```sh
sudo ethtool -n $IFACE rx-flow-hash tcp4 2>/dev/null
sudo ethtool -x $IFACE 2>/dev/null | head -10     # the indirection table

sudo bpftrace -e '
kprobe:netif_receive_skb {
	$skb = (struct sk_buff *)arg0;
	@queue[$skb->queue_mapping] = count();
	@hash_set[$skb->l4_hash] = count();
}
interval:s:10 { print(@queue); print(@hash_set); exit(); }' &
iperf3 -c 10.99.0.2 -t 8 -P 8 > /dev/null 2>&1
wait
```

---

### Lab 71.6 — BQL and the transmit queue

```sh
for q in /sys/class/net/$IFACE/queues/tx-*/byte_queue_limits/; do
  [ -d "$q" ] || continue
  echo "=== $(dirname $(dirname $q) | xargs basename) ==="
  for f in limit limit_max limit_min inflight hold_time; do
    printf "  %-12s %s\n" $f "$(cat $q/$f 2>/dev/null)"
  done
done
```

Watch it adapt:

```sh
Q=/sys/class/net/$IFACE/queues/tx-0/byte_queue_limits
[ -d "$Q" ] && {
  for i in $(seq 1 15); do
    echo "limit=$(cat $Q/limit) inflight=$(cat $Q/inflight)"
    sleep 1
  done &
  iperf3 -s -1 > /dev/null 2>&1 &
  sleep 1; iperf3 -c 127.0.0.1 -t 12 > /dev/null 2>&1
  wait
}
```

BQL's effect on latency — §T.9:

```sh
# Create a slow link to make queueing visible
sudo tc qdisc add dev veth0 root netem rate 10mbit 2>/dev/null

# Baseline latency
ping -c 10 -i 0.2 10.99.0.2 | tail -2

# Under load
iperf3 -c 10.99.0.2 -t 15 > /dev/null 2>&1 &
sleep 2
ping -c 10 -i 0.2 10.99.0.2 | tail -2
wait

sudo tc qdisc del dev veth0 root 2>/dev/null
```

The stop-queue race — §T.9's classic bug:

```sh
sudo bpftrace -e '
kprobe:netif_tx_stop_queue { @stop = count(); }
kprobe:netif_tx_wake_queue { @wake = count(); }
kprobe:dev_watchdog        { @watchdog = count(); }
interval:s:5 { print(@stop); print(@wake); print(@watchdog);
               clear(@stop); clear(@wake); clear(@watchdog); }' &

iperf3 -s -1 > /dev/null 2>&1 &
sleep 1; iperf3 -c 127.0.0.1 -t 8 -P 8 > /dev/null 2>&1
```

Stops and wakes should balance. A `dev_watchdog` firing means a queue was stopped and never woken — the bug.

TX queue length and drops:

```sh
ip link show $IFACE | grep qlen
ip -s link show $IFACE
sudo tc -s qdisc show dev $IFACE
```

---

### Lab 71.7 — Write a network driver

```c
// SPDX-License-Identifier: GPL-2.0
/* loopnet.c -- a virtual NIC that loops TX back to RX.
 * Demonstrates the T.5 driver contract and T.6's NAPI. */
#include <linux/module.h>
#include <linux/netdevice.h>
#include <linux/etherdevice.h>
#include <linux/skbuff.h>
#include <linux/ip.h>

#define LOOPNET_RING_SIZE	256
#define LOOPNET_NAPI_WEIGHT	64

struct loopnet_priv {
	struct net_device	*dev;
	struct napi_struct	napi;
	struct sk_buff		*rx_ring[LOOPNET_RING_SIZE];
	unsigned int		rx_head, rx_tail;
	spinlock_t		rx_lock;
	struct u64_stats_sync	syncp;
	u64			rx_packets, rx_bytes;
	u64			tx_packets, tx_bytes;
	u64			rx_dropped;
};

static struct net_device *loopnet_dev;

static int loopnet_open(struct net_device *dev)
{
	struct loopnet_priv *priv = netdev_priv(dev);

	napi_enable(&priv->napi);
	netif_start_queue(dev);
	netif_carrier_on(dev);
	return 0;
}

static int loopnet_stop(struct net_device *dev)
{
	struct loopnet_priv *priv = netdev_priv(dev);
	int i;

	netif_carrier_off(dev);
	netif_stop_queue(dev);
	napi_disable(&priv->napi);

	/* Drain the ring */
	for (i = 0; i < LOOPNET_RING_SIZE; i++) {
		if (priv->rx_ring[i]) {
			dev_kfree_skb(priv->rx_ring[i]);
			priv->rx_ring[i] = NULL;
		}
	}
	priv->rx_head = priv->rx_tail = 0;
	return 0;
}

static netdev_tx_t loopnet_xmit(struct sk_buff *skb, struct net_device *dev)
{
	struct loopnet_priv *priv = netdev_priv(dev);
	unsigned int next;
	unsigned long flags;

	/* T.5: we own the skb now and must dispose of it. */
	u64_stats_update_begin(&priv->syncp);
	priv->tx_packets++;
	priv->tx_bytes += skb->len;
	u64_stats_update_end(&priv->syncp);

	spin_lock_irqsave(&priv->rx_lock, flags);
	next = (priv->rx_tail + 1) % LOOPNET_RING_SIZE;
	if (next == priv->rx_head) {
		/* Ring full. T.9: stop the queue, do not return BUSY. */
		netif_stop_queue(dev);
		spin_unlock_irqrestore(&priv->rx_lock, flags);
		u64_stats_update_begin(&priv->syncp);
		priv->rx_dropped++;
		u64_stats_update_end(&priv->syncp);
		dev_kfree_skb_any(skb);
		return NETDEV_TX_OK;
	}

	/* Loop it back: it becomes an RX packet. */
	priv->rx_ring[priv->rx_tail] = skb;
	priv->rx_tail = next;
	spin_unlock_irqrestore(&priv->rx_lock, flags);

	/* T.6: schedule NAPI rather than processing inline. */
	napi_schedule(&priv->napi);
	return NETDEV_TX_OK;
}

static int loopnet_poll(struct napi_struct *napi, int budget)
{
	struct loopnet_priv *priv =
		container_of(napi, struct loopnet_priv, napi);
	struct net_device *dev = priv->dev;
	int work_done = 0;
	unsigned long flags;

	while (work_done < budget) {
		struct sk_buff *skb;

		spin_lock_irqsave(&priv->rx_lock, flags);
		if (priv->rx_head == priv->rx_tail) {
			spin_unlock_irqrestore(&priv->rx_lock, flags);
			break;
		}
		skb = priv->rx_ring[priv->rx_head];
		priv->rx_ring[priv->rx_head] = NULL;
		priv->rx_head = (priv->rx_head + 1) % LOOPNET_RING_SIZE;
		spin_unlock_irqrestore(&priv->rx_lock, flags);

		if (!skb)
			break;

		/* Turn a TX skb into an RX skb (T.2). */
		skb_orphan(skb);
		skb->dev = dev;
		skb->protocol = eth_type_trans(skb, dev);
		skb->ip_summed = CHECKSUM_UNNECESSARY;
		skb->pkt_type = PACKET_HOST;

		u64_stats_update_begin(&priv->syncp);
		priv->rx_packets++;
		priv->rx_bytes += skb->len;
		u64_stats_update_end(&priv->syncp);

		napi_gro_receive(napi, skb);    /* T.8 */
		work_done++;
	}

	/* T.9: we made room; wake the queue. */
	if (netif_queue_stopped(dev)) {
		smp_mb();
		netif_wake_queue(dev);
	}

	if (work_done < budget)
		napi_complete_done(napi, work_done);   /* T.6 */

	return work_done;
}

static void loopnet_get_stats64(struct net_device *dev,
				struct rtnl_link_stats64 *stats)
{
	struct loopnet_priv *priv = netdev_priv(dev);
	unsigned int start;

	do {
		start = u64_stats_fetch_begin(&priv->syncp);
		stats->rx_packets = priv->rx_packets;
		stats->rx_bytes   = priv->rx_bytes;
		stats->tx_packets = priv->tx_packets;
		stats->tx_bytes   = priv->tx_bytes;
		stats->rx_dropped = priv->rx_dropped;
	} while (u64_stats_fetch_retry(&priv->syncp, start));
}

static int loopnet_change_mtu(struct net_device *dev, int new_mtu)
{
	dev->mtu = new_mtu;
	return 0;
}

static const struct net_device_ops loopnet_ops = {
	.ndo_open		= loopnet_open,
	.ndo_stop		= loopnet_stop,
	.ndo_start_xmit		= loopnet_xmit,
	.ndo_get_stats64	= loopnet_get_stats64,
	.ndo_change_mtu		= loopnet_change_mtu,
	.ndo_set_mac_address	= eth_mac_addr,
	.ndo_validate_addr	= eth_validate_addr,
};

static void loopnet_setup(struct net_device *dev)
{
	ether_setup(dev);
	dev->netdev_ops = &loopnet_ops;
	dev->needs_free_netdev = true;

	dev->flags |= IFF_NOARP;
	dev->flags &= ~IFF_MULTICAST;
	dev->min_mtu = 68;
	dev->max_mtu = 65535;

	/* T.5: the feature contract. Claim only what we handle. */
	dev->features |= NETIF_F_SG | NETIF_F_FRAGLIST | NETIF_F_HIGHDMA |
			 NETIF_F_HW_CSUM | NETIF_F_TSO | NETIF_F_TSO6 |
			 NETIF_F_GSO_ROBUST | NETIF_F_LLTX;
	dev->hw_features = dev->features;
	dev->vlan_features = dev->features;

	/* T.2: reserve headroom so encapsulation is free. */
	dev->needed_headroom = NET_SKB_PAD;

	eth_hw_addr_random(dev);
}

static int __init loopnet_init(void)
{
	struct loopnet_priv *priv;
	int ret;

	loopnet_dev = alloc_netdev(sizeof(*priv), "loopnet%d",
				   NET_NAME_UNKNOWN, loopnet_setup);
	if (!loopnet_dev)
		return -ENOMEM;

	priv = netdev_priv(loopnet_dev);
	priv->dev = loopnet_dev;
	spin_lock_init(&priv->rx_lock);
	u64_stats_init(&priv->syncp);

	netif_napi_add(loopnet_dev, &priv->napi, loopnet_poll);

	ret = register_netdev(loopnet_dev);
	if (ret) {
		netif_napi_del(&priv->napi);
		free_netdev(loopnet_dev);
		return ret;
	}

	pr_info("loopnet: registered %s\n", loopnet_dev->name);
	return 0;
}

static void __exit loopnet_exit(void)
{
	struct loopnet_priv *priv = netdev_priv(loopnet_dev);

	unregister_netdev(loopnet_dev);
	netif_napi_del(&priv->napi);
	/* needs_free_netdev handles free_netdev */
}

module_init(loopnet_init);
module_exit(loopnet_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Loopback network device with NAPI");
```

```sh
cat > Makefile <<'EOF'
obj-m += loopnet.o
all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
EOF
make && sudo insmod loopnet.ko

ip link show loopnet0
sudo ip addr add 10.98.0.1/24 dev loopnet0
sudo ip link set loopnet0 up
ip -s link show loopnet0
sudo ethtool -k loopnet0 | head -15
```

Test it:

```sh
ping -c 3 -I loopnet0 10.98.0.1 2>/dev/null
ip -s link show loopnet0

sudo bpftrace -e '
tracepoint:napi:napi_poll /str(args->dev_name) == "loopnet0"/ {
	@work = hist(args->work);
}
interval:s:8 { print(@work); exit(); }' &
ping -c 50 -i 0.02 -I loopnet0 10.98.0.1 > /dev/null 2>&1
wait
```

```sh
sudo tcpdump -i loopnet0 -c 5 2>/dev/null &
ping -c 5 -I loopnet0 10.98.0.1 > /dev/null 2>&1
wait

sudo rmmod loopnet
```

Exercises:

1. Add `ethtool_ops` with `get_drvinfo`, `get_link`, `get_ringparam`, and per-queue stats.
2. Add multiqueue support (`alloc_etherdev_mq`) and implement `ndo_select_queue`.
3. Add BQL (`netdev_tx_sent_queue` / `netdev_tx_completed_queue`).
4. Add XDP support (`ndo_bpf`) and handle `XDP_DROP`/`XDP_PASS`.
5. Use a page pool and `build_skb` instead of passing skbs through.

---

### Lab 71.8 — Measure where the cycles go

```sh
sudo apt install -y linux-tools-$(uname -r) 2>/dev/null

sudo perf record -a -g -F 999 -- timeout 10 sh -c '
  iperf3 -s -1 > /dev/null 2>&1 &
  sleep 0.5
  iperf3 -c 127.0.0.1 -t 8 -P 4 > /dev/null 2>&1'
sudo perf report --stdio --sort symbol 2>/dev/null | head -40
```

Filter to the network stack:

```sh
sudo perf report --stdio 2>/dev/null | \
  grep -iE 'skb|napi|netif|tcp_|ip_|dev_|gro|gso' | head -25
```

Per-packet cost:

```sh
sudo bpftrace -e '
kprobe:netif_receive_skb { @t[tid] = nsecs; }
kretprobe:netif_receive_skb /@t[tid]/ {
	@receive_ns = hist(nsecs - @t[tid]); delete(@t[tid]);
}
kprobe:__dev_queue_xmit { @x[tid] = nsecs; }
kretprobe:__dev_queue_xmit /@x[tid]/ {
	@xmit_ns = hist(nsecs - @x[tid]); delete(@x[tid]);
}
interval:s:12 { print(@receive_ns); print(@xmit_ns); exit(); }' &

iperf3 -s -1 > /dev/null 2>&1 &
sleep 1; iperf3 -c 127.0.0.1 -t 10 -P 4 > /dev/null 2>&1
wait
```

Allocation cost:

```sh
sudo bpftrace -e '
kprobe:__alloc_skb { @a[tid] = nsecs; }
kretprobe:__alloc_skb /@a[tid]/ {
	@alloc_ns = hist(nsecs - @a[tid]); delete(@a[tid]);
}
kprobe:kfree_skb  { @free = count(); }
kprobe:__alloc_skb { @alloc = count(); }
interval:s:10 { print(@alloc_ns); print(@alloc); print(@free); exit(); }' &
iperf3 -s -1 > /dev/null 2>&1 &
sleep 1; iperf3 -c 127.0.0.1 -t 8 > /dev/null 2>&1
wait
```

Softirq time — the cost that is invisible to `top`:

```sh
cat /proc/softirqs | head -2
grep -E 'NET_RX|NET_TX' /proc/softirqs

BEFORE=$(grep NET_RX /proc/softirqs | awk '{for(i=2;i<=NF;i++) s+=$i} END {print s}')
iperf3 -s -1 > /dev/null 2>&1 &
sleep 0.5; iperf3 -c 127.0.0.1 -t 5 -P 4 > /dev/null 2>&1
AFTER=$(grep NET_RX /proc/softirqs | awk '{for(i=2;i<=NF;i++) s+=$i} END {print s}')
echo "NET_RX softirqs: $((AFTER-BEFORE))"

# Time spent in softirq
sudo bpftrace -e '
tracepoint:irq:softirq_entry /args->vec == 3/ { @s[cpu] = nsecs; }
tracepoint:irq:softirq_exit /args->vec == 3 && @s[cpu]/ {
	@net_rx_us = hist((nsecs - @s[cpu]) / 1000); delete(@s[cpu]);
}
interval:s:10 { print(@net_rx_us); exit(); }' &
iperf3 -s -1 > /dev/null 2>&1 &
sleep 1; iperf3 -c 127.0.0.1 -t 8 -P 4 > /dev/null 2>&1
wait
```

`mpstat` shows it as `%soft`:

```sh
mpstat -P ALL 1 5 2>/dev/null | head -20
```

A full-stack picture:

```sh
cat > netdiag.sh <<'EOF'
#!/bin/bash
IF=${1:-eth0}
echo "=== Interface ==="
ip -s link show $IF | tail -4
echo "=== Driver counters (errors/drops only) ==="
ethtool -S $IF 2>/dev/null | grep -iE 'err|drop|miss|fail|discard|no_buf' | grep -v ': 0$'
echo "=== Offloads ==="
ethtool -k $IF 2>/dev/null | grep -E 'gro|gso|tso|sg|checksum' | head
echo "=== Rings ==="
ethtool -g $IF 2>/dev/null | tail -5
echo "=== Queues ==="
ethtool -l $IF 2>/dev/null | tail -4
echo "=== softnet ==="
awk '{printf "cpu%-3d processed=%-10d dropped=%-6d squeeze=%-6d rps=%d\n",
      NR-1, strtonum("0x"$1), strtonum("0x"$2), strtonum("0x"$3), strtonum("0x"$10)}' \
  /proc/net/softnet_stat
echo "=== Protocol counters (non-zero) ==="
nstat -az 2>/dev/null | grep -iE 'drop|error|retrans|overflow|prune|collapse' | \
  awk '$2 != 0' | head -15
EOF
chmod +x netdiag.sh
sudo ./netdiag.sh $IFACE
```

Cleanup:

```sh
sudo ip link del veth0 2>/dev/null
sudo ip netns del ns1 2>/dev/null
```

---

## 3. Mastery drills

1. Compute the per-packet cycle budget at 10, 25, and 100 Gbit/s with 64-byte frames on a 3 GHz core. Then state which of §T.1's mechanisms each rate makes mandatory.

2. Explain receive livelock precisely, including why the CPU can be 100 % busy at zero throughput. Then show how NAPI's two-regime design eliminates it.

3. Draw the `sk_buff` buffer layout and state what each of `head`, `data`, `tail`, `end`, `len`, `data_len`, and `truesize` means. Then compute each after a `skb_pull` of 20 bytes on a non-linear skb.

4. Why does `NET_SKB_PAD` exist? Construct the performance bug that occurs when a driver reserves too little headroom, and trace what the kernel does instead.

5. Explain the difference between `skb_clone`, `skb_copy`, and `pskb_copy`. For each, state who may write to what, and construct the corruption from writing to cloned data.

6. `pskb_may_pull` is required before reading a header. Construct the out-of-bounds read that occurs without it, and explain why it is invisible in testing on a typical driver.

7. GRO is reversible; LRO is not. State precisely what information LRO destroys, construct the forwarding bug it causes, and explain how the kernel defends against it.

8. Derive GSO's throughput benefit: count the per-packet operations avoided when sending 64 KB as one skb versus 44 × 1500-byte packets.

9. Explain the TX stop-queue race as a memory-ordering problem. Write both interleavings, identify the litmus test it corresponds to, and give the correct barrier placement.

10. BQL limits bytes, not packets. Explain why bytes is the right unit, and compute the queueing delay a 4096-descriptor ring introduces at 1 Mbit/s and at 10 Gbit/s.

11. Compare RSS, RPS, RFS, and aRFS. For each, state what it costs, what it buys, and the workload where it is the right choice.

12. XDP_DROP is ~20× faster than an iptables drop. Enumerate everything XDP skips, and state the one thing it cannot do that the stack can.

13. You are given a server dropping packets at 2 Gbit/s on a 10 Gbit NIC, with one CPU at 100 % `%soft`. Give the ordered diagnostic procedure using this chapter's tools and the seven most likely causes.

---

## 4. Further reading

**Kernel documentation**

- `Documentation/networking/skbuff.rst` ★★★ — §T.2 and §T.3.
- `Documentation/networking/napi.rst` ★★★ — **§T.6, normatively.** Read this before writing any driver.
- `Documentation/networking/driver.rst` ★★★ — the `ndo_start_xmit` contract and **the stop-queue race** of §T.9, documented explicitly.
- `Documentation/networking/scaling.rst` ★★★ — **RSS, RPS, RFS, aRFS, XPS in full.** The best single document on §T.7's steering mechanisms.
- `Documentation/networking/segmentation-offloads.rst` ★★★ — §T.8.
- `Documentation/networking/checksum-offloads.rst` ★★★ — the `ip_summed` state machine, which is subtler than it looks.
- `Documentation/networking/netdev-features.rst` ★★★ — §T.5's contract.
- `Documentation/networking/kapi.rst`, `netdevices.rst`
- `Documentation/networking/page_pool.rst` — §T.4(c).
- `Documentation/networking/multiqueue.rst`, `xps.rst`
- `Documentation/networking/af_xdp.rst`, `xdp-rx-metadata.rst` — Ch. 74.
- `Documentation/networking/statistics.rst` — where each counter comes from.

**Papers**

- Mogul & Ramakrishnan, "Eliminating Receive Livelock in an Interrupt-Driven Kernel," USENIX 1996 ★★★ — **§T.6's problem and solution.** The paper NAPI implements. Short and essential.
- Druschel & Banga, "Lazy Receiver Processing (LRP): A Network Subsystem Architecture for Server Systems," OSDI 1996 — a related approach.
- Rizzo, "netmap: a novel framework for fast packet I/O," USENIX ATC 2012 ★★★ — the kernel-bypass argument that motivated XDP.
- Høiland-Jørgensen et al., "The eXpress Data Path: Fast Programmable Packet Processing in the Operating System Kernel," CoNEXT 2018 ★★★ — §T.10; Ch. 74's foundation.
- Cardwell et al., "BBR: Congestion-Based Congestion Control," ACM Queue 2016 — Ch. 72, but relevant to why pacing and BQL matter.
- Gettys & Nichols, "Bufferbloat: Dark Buffers in the Internet," ACM Queue 2011 ★★★ — **§T.9's motivation.** Read this to understand why BQL and fq_codel exist.
- Jacobson & Karels, "Congestion Avoidance and Control," SIGCOMM 1988 — Ch. 72.

**Books**

- Rosen, *Linux Kernel Networking: Implementation and Theory* ★★★ — dated (3.x era) but structurally still the best book on this material.
- Benvenuti, *Understanding Linux Network Internals* — much older (2.6), excellent on the receive path's design rationale.
- Stevens, *TCP/IP Illustrated, Volume 1* ★★★ — the protocols themselves; Ch. 72's companion.
- Gregg, *Systems Performance*, 2nd ed., Chapter 10 ★★★ — the networking methodology and tooling.

**LWN**

- "The 'kernel address sanitizer' and networking" and the skb fuzzing coverage
- "JLS2009: Generic receive offload" ★★★ — §T.8's introduction, by its author
- "Byte queue limits" ★★★ — §T.9
- "Receive packet steering" and "Receive flow steering" ★★★
- "The rapidly changing world of XDP" and the XDP series ★★★
- "Threaded NAPI"
- "A netlink-based interface for network statistics"
- "kfree_skb_reason(): why was this packet dropped?" ★★★
- "Page pool and the networking stack"
- "Toward a smaller sk_buff" — the recurring effort and why it is hard
- The annual netdev conference summaries ★★★ — where this subsystem is designed

**Source reading order**

1. `Documentation/networking/napi.rst` and `driver.rst` first.
2. `include/linux/skbuff.h` ★★★ — **read the accessors, not just the struct.** `skb_put`, `skb_push`, `skb_pull`, `skb_reserve`, `skb_headlen`, `pskb_may_pull`, `skb_header_pointer`.
3. `drivers/net/veth.c` ★★★ — ~1800 lines; a complete, modern, readable driver with NAPI, GRO, and XDP.
4. `drivers/net/virtio_net.c` ★★★ — a real driver with multiqueue, page pool, XDP, and all the offloads.
5. `net/core/dev.c`: `__netif_receive_skb_core` ★★★, `net_rx_action`, `__dev_queue_xmit`, `dev_hard_start_xmit`, `validate_xmit_skb`.
6. `net/core/skbuff.c`: `__alloc_skb`, `build_skb`, `skb_clone`, `pskb_expand_head`, `__pskb_pull_tail`.
7. `net/core/gro.c` and `net/ipv4/tcp_offload.c`: `tcp_gro_receive` ★★★ — §T.8.
8. `drivers/net/ethernet/intel/igb/igb_main.c` — a production driver's `igb_poll` and `igb_xmit_frame`; larger but complete.

**Tools**

- `ethtool` ★★★ — `-S` (driver stats), `-k`/`-K` (features), `-g`/`-G` (rings), `-c`/`-C` (coalescing), `-l`/`-L` (queues), `-x`/`-X` (RSS), `-n`/`-N` (flow steering)
- `/proc/net/softnet_stat` ★★★ — **the first place to look**
- `ip -s link`, `ip -s -s link` ★★★
- `nstat -az` ★★★ — protocol counters; far better than `netstat -s`
- `bpftrace` on `skb:kfree_skb` with reasons ★★★, `napi:napi_poll`, `net:net_dev_xmit`
- `dropwatch` ★★★
- `perf record -a -g` with `-e net:*` and `-e skb:*`
- `tcpdump -i any -nn` ★★★ — and remember it sees packets tc may later drop
- `iperf3`, `netperf`, `sockperf`, `pktgen` (`samples/pktgen/`) ★★★
- `trafgen`/`netsniff-ng` — high-rate packet generation
- `veth` + network namespaces ★★★ — the laboratory; every experiment in this chapter runs without hardware

---

→ Next: [72-protocol-stack.md](72-protocol-stack.md)

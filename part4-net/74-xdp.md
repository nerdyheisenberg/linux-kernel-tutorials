# Chapter 74 — XDP, AF_XDP, and high-performance networking

> **Goal:** Understand the fast path that bypasses most of the stack, and the arguments for and against bypassing it entirely. Understand why kernel bypass appeared and what XDP was designed to reclaim, the `xdp_buff` and its constraints, the five actions and what each costs, the three attach modes and why generic mode is a trap, redirect maps as the composition mechanism, AF_XDP's zero-copy socket with its four rings and the UMEM, how XDP and AF_XDP compose into a programmable data plane, the metadata problem and the hints interface, and an honest comparison with DPDK. By the end you can write and attach XDP programs, build an AF_XDP application, and reason about where the remaining nanoseconds go.

---

## Theory & First Principles

### T.0 — Start here: drop a packet in 30 nanoseconds

```c
SEC("xdp")
int drop_all(struct xdp_md *ctx)
{
	return XDP_DROP;
}
```

Attach that to a NIC and it drops **tens of millions of packets per second per core**. The
equivalent iptables rule manages perhaps 1–2 Mpps. **Same machine, same NIC, ~20× difference.**
The question worth answering is exactly *where* that factor of 20 comes from, because the
answer is a general lesson about layering.

```
   packet arrives at the NIC, DMA'd into a page
     |
     +---> XDP program runs HERE  <-- ~30 ns. No sk_buff exists yet.
     |                                 You have a raw pointer to bytes.
     |
     v  (only if XDP_PASS)
   build an sk_buff                <-- ~232 bytes alloc'd + initialized,
     |                                 several cachelines touched (Ch. 71 §T.0)
     v
   GRO -> tc ingress -> netfilter PREROUTING -> routing -> conntrack
     |                                                        ^
     v                                                        |
   TCP/UDP -> socket -> wake process              lots of state, lots of
                                                  cachelines, lots of branches
```

**XDP's entire performance story is "run before the expensive part."** It is not faster code
doing the same work; it is a decision point moved earlier, to before the `sk_buff` is
allocated and before any of the stack's generality has been paid for.

> **The cheapest work is work you decide not to do. Move the decision as early as the
> information allows.**

You have seen this exact move three times already: the page fault as the earliest point where
the kernel can intervene (Ch. 22 §T.0), netfilter's PRE_ROUTING placement (Ch. 73 §T.0), and
`iomap` asking the mapping question once per extent instead of once per block (Ch. 55 §T.0).

**But XDP is not "kernel bypass," and the difference matters.** DPDK and similar frameworks
take the NIC away from the kernel entirely:

| | **Kernel bypass (DPDK)** | **XDP** |
|---|---|---|
| Who owns the NIC | userspace; the kernel cannot use it | the kernel |
| Can you still use `ssh`, `tcpdump`, the TCP stack? | **no** | **yes** — `XDP_PASS` keeps everything |
| CPU cost when idle | a core spinning at 100% | zero |
| Safety | a bug corrupts arbitrary memory | **verified** before load (Ch. 75) |
| Deployment | rebuild the application around a framework | attach a program to a running system |

**XDP's real claim is not raw speed — DPDK is faster. It is that you get most of the speed
without giving up the operating system.** That is a *programmability* argument, and it is the
reason XDP won in places bypass never did (Cloudflare's DDoS mitigation, Facebook's Katran
load balancer, Cilium's service routing).

**The five verdicts are worth memorizing**, because they define what XDP can and cannot be:

| Verdict | Meaning | Typical use |
|---|---|---|
| `XDP_DROP` | free the page immediately | DDoS filtering |
| `XDP_PASS` | continue into the normal stack | everything you do not care about |
| `XDP_TX` | bounce it back out the same NIC | load balancing, reflection |
| `XDP_REDIRECT` | send to another NIC, a CPU, or an `AF_XDP` socket | routing, userspace fast path |
| `XDP_ABORTED` | a bug; drop and raise a tracepoint | debugging |

**And the costs, which are real and should temper the enthusiasm:** you see one packet at a
time, before GRO, with no fragment reassembly and no connection state except what you build
yourself in maps; headroom is limited; and "native" mode requires driver support (generic
mode works everywhere and is slow enough to defeat the purpose). **XDP is the right tool for
stateless or cheaply-stateful per-packet decisions, and the wrong tool for anything that needs
the stack's semantics.**

```bash
sudo bpftool net show && sudo bpftool prog list
ip -s link show dev eth0                 # XDP counters appear here
sudo xdp-loader status
sudo ethtool -S eth0 | grep -i xdp
```

---

### T.1 Why bypass appeared

Chapter 71 §T.1 established the budget: 67 ns per packet at 14.88 Mpps on 10 GbE. A full trip through the Linux stack — skb allocation, GRO, netfilter, routing, socket lookup, copy to userspace — costs several microseconds. For a router or a load balancer that only needs to look at a header and make a decision, almost all of that is wasted.

So around 2010, userspace frameworks appeared that took the NIC away from the kernel entirely:

| Framework | Approach |
|---|---|
| **netmap** (Rizzo, 2012) | a kernel module exposing the NIC rings to userspace via `mmap` |
| **DPDK** (Intel, 2010) | a complete userspace driver set, poll-mode, huge pages |
| **PF_RING ZC**, **Snabb**, **VPP** | variations on the theme |

They achieved 10–40× the packet rate. The cost:

| Cost | Detail |
|---|---|
| **The NIC is gone** | no `ip addr`, no `tcpdump`, no kernel TCP on that interface |
| **You write a driver** | and maintain it against firmware changes |
| **No kernel services** | no routing table, no netfilter, no sockets |
| **Poll-mode burns a core** | 100 % CPU whether traffic exists or not |
| **Security is yours** | no kernel isolation between the app and the device |
| **Deployment is invasive** | huge pages, IOMMU configuration, device binding |

XDP's premise (Høiland-Jørgensen et al., CoNEXT 2018):

> **Run programmable packet processing inside the kernel driver, before the skb exists, and keep everything else.**

You get most of bypass's speed *and* keep the kernel's networking stack for the packets you pass through. The interface stays usable; `tcpdump` still works; the routing table is still there. That composability is the argument, and it has largely won for the use cases it fits.

### T.2 `xdp_buff`: what you get before the skb

```c
struct xdp_buff {
	void *data;             /* the packet's start */
	void *data_end;         /* one past the end */
	void *data_meta;        /* metadata area, GROWS DOWN from data */
	void *data_hard_start;  /* the start of the usable buffer */
	struct xdp_rxq_info *rxq;
	struct xdp_txq_info *txq;
	u32 frame_sz;
	u32 flags;
};

/* The userspace-visible view, given to the BPF program */
struct xdp_md {
	__u32 data;
	__u32 data_end;
	__u32 data_meta;
	__u32 ingress_ifindex;
	__u32 rx_queue_index;
	__u32 egress_ifindex;
};
```

This is **a pointer to a DMA buffer**, not an skb. There is no allocation, no metadata parsing, no protocol dispatch. The program sees raw bytes.

The constraints that follow:

| Constraint | Why |
|---|---|
| **Every access must be bounds-checked** | the verifier requires `ptr + len <= data_end` before any dereference |
| **The packet is normally linear** | multi-buffer XDP exists but most programs assume one buffer |
| **Headroom is limited** | typically 256 bytes (`XDP_PACKET_HEADROOM`); encapsulation must fit |
| **No sleeping, no locks** | it runs in the driver's NAPI poll |
| **Bounded execution** | the verifier proves termination (Ch. 75) |

The bounds-checking discipline is the thing that surprises people coming from C:

```c
	struct ethhdr *eth = data;

	if ((void *)(eth + 1) > data_end)      /* MANDATORY, before any access */
		return XDP_DROP;
	if (eth->h_proto != bpf_htons(ETH_P_IP))
		return XDP_PASS;

	struct iphdr *iph = (void *)(eth + 1);
	if ((void *)(iph + 1) > data_end)      /* AGAIN, for each header */
		return XDP_DROP;
```

Omitting a check produces a verifier rejection, not a crash — which is the point. **The verifier converts a class of kernel memory-safety bugs into compile-time errors**, and that is much of why XDP is deployable where a kernel module would not be.

`data_meta` is a small area *before* `data` that an XDP program can claim (`bpf_xdp_adjust_meta`) and that survives into the skb, readable by a later tc-bpf program. It is how XDP passes computed state forward — a classification result, a hash, a decision — without re-parsing.

### T.3 The five actions, and what each costs

```c
enum xdp_action {
	XDP_ABORTED = 0,   /* error; a tracepoint fires; the packet is dropped */
	XDP_DROP,          /* free the page; never allocate an skb */
	XDP_PASS,          /* continue into the normal stack */
	XDP_TX,            /* transmit back out the SAME interface */
	XDP_REDIRECT,      /* to another interface, CPU, or AF_XDP socket */
};
```

The measured cost, roughly, per core on modern hardware:

| Action | Packets/second |
|---|---|
| `XDP_DROP` | **20–30 million** |
| `XDP_TX` | 10–15 million |
| `XDP_REDIRECT` (to another NIC) | 10–15 million |
| `XDP_REDIRECT` (to AF_XDP) | 15–25 million |
| `XDP_PASS` | ~2–3 million (the stack's rate) |
| iptables DROP | ~1–2 million |

`XDP_DROP`'s speed comes from what it *does not do*: no skb allocation, no page-pool return through the normal path, no protocol dispatch. The page goes straight back to the driver's ring.

`XDP_TX` is the basis of in-kernel load balancers and NAT: rewrite the headers, recompute the checksum incrementally, send it back. Facebook's Katran and Cloudflare's L4 layer both work this way.

`XDP_REDIRECT` is the composition primitive and is covered in §T.5.

`XDP_ABORTED` deserves a note: it means the program errored (a helper failed, a bounds check the verifier could not eliminate at runtime). It drops the packet *and* fires the `xdp:xdp_exception` tracepoint, so it is observable. A program returning `XDP_ABORTED` in production is a bug; monitoring for that tracepoint is standard practice.

### T.4 Three attach modes

```c
#define XDP_FLAGS_SKB_MODE	(1U << 1)   /* generic */
#define XDP_FLAGS_DRV_MODE	(1U << 2)   /* native */
#define XDP_FLAGS_HW_MODE	(1U << 3)   /* offloaded */
```

| Mode | Where it runs | Speed | Requires |
|---|---|---|---|
| **Native** (`xdpdrv`) | in the driver's NAPI poll, on the DMA page | **full** | driver support |
| **Offloaded** (`xdpoffload`) | on the NIC | **full, zero host CPU** | Netronome/Agilio |
| **Generic** (`xdpgeneric`) | in `netif_receive_skb`, **after the skb exists** | **slow** | nothing |

Generic mode exists so you can develop and test without driver support. It is **not** a performance mode: the skb has already been allocated, which is the dominant cost XDP exists to avoid. Benchmarks run in generic mode are meaningless, and "XDP is only 2× faster than iptables" claims almost always trace to generic mode.

```sh
ip link set dev eth0 xdpdrv obj prog.o sec xdp      # native, fails if unsupported
ip link set dev eth0 xdpgeneric obj prog.o sec xdp  # generic
ip link set dev eth0 xdp obj prog.o sec xdp         # try native, fall back to generic
ip link show dev eth0 | grep -o 'xdp[a-z]*'
```

Driver support is now broad — ixgbe, i40e, ice, mlx5, mlx4, bnxt, nfp, virtio_net, veth, tun, and more — but checking is the first step in any XDP work.

`veth` support matters particularly: it means XDP programs can be tested and used in container networking without physical hardware, and it is how Cilium implements much of its datapath.

### T.5 Redirect maps: the composition mechanism

`XDP_REDIRECT` alone is not useful; it needs a target, and the target comes from a map. This indirection is what makes XDP composable.

```c
	bpf_redirect_map(&map, key, flags);
```

| Map type | Target | Use |
|---|---|---|
| `BPF_MAP_TYPE_DEVMAP` | a network device | forwarding, bridging |
| `BPF_MAP_TYPE_DEVMAP_HASH` | device by ifindex | sparse ifindex spaces |
| `BPF_MAP_TYPE_CPUMAP` | **another CPU** | steer to a CPU for stack processing |
| `BPF_MAP_TYPE_XSKMAP` | an AF_XDP socket | zero-copy to userspace |

**CPUMAP is the underappreciated one.** It lets XDP decide which CPU will do the expensive stack processing, based on a programmable classification rather than the NIC's RSS hash. So:

- A NIC with poor RSS (or none) can still spread load.
- Traffic can be steered by criteria the hardware cannot express (a tunnel's inner header, an application-level field).
- You can isolate a tenant's traffic onto specific cores.

The redirected packet is queued to the target CPU, which then builds an skb and passes it to the stack — so you get the stack's full functionality with a programmable steering decision.

**DEVMAP entries can carry their own XDP program** (since 5.8), which runs on the egress device after redirect. This enables a two-stage pipeline: classify on ingress, transform on egress.

Redirect is **batched**: `xdp_do_flush()` is called at the end of the NAPI poll, so a burst of redirects to the same target is one bulk operation. This is where much of redirect's throughput comes from, and it is why a driver that forgets to call `xdp_do_flush` has correct-but-slow XDP.

### T.6 AF_XDP: zero-copy to userspace

XDP can process packets in the kernel, but some applications need them in userspace — a userspace TCP stack, a packet capture system, a custom protocol. AF_XDP provides that path without copying.

The structure is four rings plus a shared memory region:

```
        UMEM (a user-allocated region, registered with the kernel)
        ┌──────────────────────────────────────────────────┐
        │  chunk 0 │ chunk 1 │ chunk 2 │ ... │ chunk N      │
        └──────────────────────────────────────────────────┘
              ▲                                   ▲
              │                                   │
   ┌──────────┴─────────┐              ┌──────────┴─────────┐
   │   FILL ring        │              │  COMPLETION ring   │
   │ (user -> kernel):  │              │ (kernel -> user):  │
   │ "here are empty    │              │ "I finished with   │
   │  chunks for RX"    │              │  these TX chunks"  │
   └────────────────────┘              └────────────────────┘

   ┌────────────────────┐              ┌────────────────────┐
   │   RX ring          │              │   TX ring          │
   │ (kernel -> user):  │              │ (user -> kernel):  │
   │ "here are received │              │ "please transmit   │
   │  packets"          │              │  these chunks"     │
   └────────────────────┘              └────────────────────┘
```

The rings hold **descriptors** (offsets into the UMEM), not data. The data never moves:

```c
struct xdp_desc {
	__u64 addr;    /* offset into the UMEM */
	__u32 len;
	__u32 options;
};
```

The ownership protocol is the important part, and it is a four-ring dance:

**Receive:**
1. Userspace puts empty chunk addresses on the **FILL** ring.
2. The kernel takes one, DMAs a packet into it.
3. The kernel puts the descriptor on the **RX** ring.
4. Userspace consumes it, processes the packet, and returns the chunk to the **FILL** ring.

**Transmit:**
1. Userspace writes packet data into a chunk and puts the descriptor on the **TX** ring.
2. Userspace calls `sendto()` to kick the kernel (or relies on a busy-polling driver).
3. The kernel transmits.
4. The kernel puts the address on the **COMPLETION** ring.
5. Userspace reclaims the chunk.

**If userspace does not refill the FILL ring, receive stops.** That is the flow control: the application's rate of returning buffers is the rate at which it receives. There is no queue to overflow and no hidden buffering — a genuinely clean design.

Two modes:

| Mode | Copy | Requires |
|---|---|---|
| **Zero-copy** (`XDP_ZEROCOPY`) | none; the NIC DMAs into the UMEM | driver support |
| **Copy** (`XDP_COPY`) | one copy from the driver's buffer | works anywhere |

Zero-copy requires the driver to allocate its RX buffers from the UMEM — which means the driver must know about AF_XDP. Support exists in i40e, ice, ixgbe, mlx5, stmmac, and others.

**`XDP_USE_NEED_WAKEUP`** is a significant optimisation: rather than the application syscalling on every batch, the kernel sets a flag when it needs a kick. The application checks the flag and only syscalls when necessary. In the common case of sustained traffic, that is zero syscalls.

The routing of packets to an AF_XDP socket goes through XDP:

```c
SEC("xdp")
int xdp_sock_prog(struct xdp_md *ctx)
{
	int index = ctx->rx_queue_index;

	if (bpf_map_lookup_elem(&xsks_map, &index))
		return bpf_redirect_map(&xsks_map, index, 0);
	return XDP_PASS;
}
```

**So you choose, per packet, what goes to userspace and what goes to the kernel stack.** That is the composability argument again: a single interface can carry both a userspace fast path and ordinary kernel traffic.

### T.7 The metadata problem

An skb carries a great deal the NIC computed: the RX hash, the checksum result, the VLAN tag, a hardware timestamp. XDP runs before the skb exists, so none of it is available in a portable way.

This was a real limitation for years. The **XDP hints** interface (6.8+) addresses it with kfuncs:

```c
	__u32 hash;
	enum xdp_rss_hash_type type;
	__u64 timestamp;

	if (bpf_xdp_metadata_rx_hash(ctx, &hash, &type) == 0) { ... }
	if (bpf_xdp_metadata_rx_timestamp(ctx, &timestamp) == 0) { ... }
	if (bpf_xdp_metadata_rx_vlan_tag(ctx, &proto, &tci) == 0) { ... }
```

These are **kfuncs, not helpers** — the driver implements them, and the BPF program calls them if available. The design choice is interesting: rather than defining a fixed metadata structure that every driver must fill (which would cost time for programs that do not need it), the metadata is fetched on demand, per field, by a driver-specific function that the verifier inlines.

This is a good example of a general principle: **when the cost of providing information exceeds the cost of fetching it on demand, provide an accessor rather than a structure.**

The application then stores what it wants in `data_meta` for the tc program or the AF_XDP consumer downstream.

### T.8 XDP versus DPDK, honestly

| | **XDP** | **DPDK** |
|---|---|---|
| Peak packets/sec/core | 20–30 M (drop) | 30–50 M |
| Latency floor | ~1 µs | ~0.5 µs |
| CPU when idle | **zero** (interrupt-driven) | **100 %** (poll mode) |
| Interface usable by the kernel | **yes** | no |
| `tcpdump`, `ip`, routing table | **yes** | no |
| Driver maintenance | **kernel's** | DPDK's |
| Safety | **verified, isolated** | full memory access |
| Deployment | load a program | bind the device, huge pages, IOMMU |
| Programming model | restricted C, verified | unrestricted C |
| Partial offload | **yes** — pass what you do not handle | all or nothing |

The honest summary:

**DPDK is faster and XDP is more deployable**, and the gap has narrowed enough that deployability usually wins. The exceptions are real: a dedicated packet-processing appliance where the machine does nothing else, or a workload needing the absolute latency floor, is still DPDK territory.

But note the third row. DPDK's poll-mode driver burns a core continuously. On a machine doing other work, or one where power matters, that is a significant cost XDP does not have.

And the fourth and fifth rows are why Cloudflare, Facebook, and Cilium all chose XDP: they needed to *add* fast packet processing to machines that were still ordinary Linux servers.

**Where each is used in production:**

| System | Approach | What it does |
|---|---|---|
| **Cilium** | XDP + tc-bpf | Kubernetes networking, replacing kube-proxy |
| **Katran** (Meta) | XDP | L4 load balancer, millions of pps per host |
| **Cloudflare L4Drop** | XDP | DDoS mitigation at the edge |
| **Calico** | XDP + tc-bpf | container networking |
| **VPP, OVS-DPDK** | DPDK | virtual switching in NFV |
| **Seastar, ScyllaDB** | DPDK (optional) | userspace networking for a database |

### T.9 The remaining costs

Even at XDP's speed, some costs remain and they are worth knowing:

**(a) DMA and cache misses.** The packet arrives in memory the CPU has not touched. The first access is a cache miss to DRAM (~80 ns), which dominates everything else at these rates. Drivers issue prefetches; `DDIO`/`DCA` on Intel makes the NIC write directly into L3. On a system where the NIC and the CPU are on different NUMA nodes, this cost doubles and is the single most common XDP performance problem.

**(b) The page pool.** XDP requires that RX buffers be recyclable without going through the page allocator. `net/core/page_pool.c` provides this, and a driver that does not use it will be slower.

**(c) The verifier's limits.** Programs are bounded (1 million instructions, 512 bytes of stack, bounded loops). Complex processing must be split across tail calls or done differently.

**(d) Map lookups.** A hash lookup is ~20–50 ns. At 67 ns per packet, two lookups is your entire budget. `BPF_MAP_TYPE_ARRAY` and per-CPU maps are much cheaper; LPM tries are more expensive. **Map choice is a first-order performance decision.**

**(e) Batching.** `xdp_do_flush()` amortises the redirect. A program that redirects one packet at a time without the flush is far slower.

**(f) Multi-buffer.** Jumbo frames and GRO'd packets may span buffers. `xdp.frags` support exists but programs must opt in and handle it, and many do not.

### T.10 When XDP is the wrong answer

Being fair, because XDP is over-applied:

| Situation | Better answer |
|---|---|
| Stateful firewalling | netfilter + conntrack (Ch. 73) |
| Anything needing TCP state | the kernel stack |
| Complex policy with many rules | nftables sets |
| Fewer than ~1 Mpps | the ordinary stack is fine |
| Packets you will pass to the stack anyway | tc-bpf (the skb exists; you get metadata) |
| Traffic shaping | tc qdiscs |
| Application-layer processing | userspace, normally |

**XDP is for high-rate, header-only, stateless-or-simply-stateful decisions made early.** Drop, redirect, encapsulate, load-balance, count. If your decision needs the payload, the connection state, or a large rule set, the cost of XDP's restrictions exceeds its benefit.

The most common mistake is using XDP where tc-bpf would do: tc-bpf runs after skb allocation, which means the metadata is there, the packet is linearised, and you can use `bpf_skb_*` helpers that XDP lacks. For anything that will reach the stack anyway, tc-bpf costs almost nothing extra and is far more capable.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `net/core/filter.c` ★★★ | the XDP helpers, `bpf_redirect_map`, verifier context handling |
| `net/core/xdp.c` ★★★ | `xdp_rxq_info`, `xdp_frame`, redirect infrastructure |
| `net/core/dev.c` | `do_xdp_generic`, `generic_xdp_tx` |
| `kernel/bpf/devmap.c` ★★★ | §T.5's DEVMAP |
| `kernel/bpf/cpumap.c` ★★★ | §T.5's CPUMAP |
| `net/xdp/xsk.c` ★★★ | §T.6's AF_XDP socket |
| `net/xdp/xsk_queue.h` ★★★ | the ring implementation |
| `net/xdp/xsk_buff_pool.c` | UMEM management |
| `net/xdp/xskmap.c` | the XSKMAP |
| `include/net/xdp.h`, `include/net/xdp_sock.h` ★★★ | |
| `include/uapi/linux/if_xdp.h` ★★★ | **the AF_XDP userspace ABI** |
| `drivers/net/veth.c` ★★★ | a readable native-XDP implementation |
| `drivers/net/ethernet/intel/i40e/i40e_xsk.c` ★★★ | zero-copy AF_XDP in a real driver |
| `samples/bpf/xdp*` ★★★ | many worked examples |
| `tools/testing/selftests/bpf/` | the test suite, full of small programs |
| `Documentation/networking/af_xdp.rst` ★★★ | |
| `Documentation/networking/xdp-rx-metadata.rst` ★★★ | §T.7 |

### 1.2 A driver's XDP path

```c
/* drivers/net/veth.c -- one of the clearest implementations */
static struct sk_buff *veth_xdp_rcv_skb(struct veth_rq *rq,
					struct sk_buff *skb,
					struct veth_xdp_tx_bq *bq,
					struct veth_stats *stats)
{
	void *orig_data, *orig_data_end;
	struct bpf_prog *xdp_prog;
	struct veth_xdp_buff vxbuf;
	struct xdp_buff *xdp = &vxbuf.xdp;
	u32 act, metalen;
	int off;

	skb_prepare_for_gro(skb);

	rcu_read_lock();
	xdp_prog = rcu_dereference(rq->xdp_prog);
	if (unlikely(!xdp_prog)) { rcu_read_unlock(); goto out; }

	__skb_push(skb, skb->data - skb_mac_header(skb));
	if (veth_convert_skb_to_xdp_buff(rq, xdp, &skb))
		goto drop;

	orig_data = xdp->data;
	orig_data_end = xdp->data_end;

	act = bpf_prog_run_xdp(xdp_prog, xdp);          /* THE program */

	switch (act) {
	case XDP_PASS:
		break;
	case XDP_TX:
		veth_xdp_get(xdp);
		consume_skb(skb);
		xdp->rxq->mem = rq->xdp_mem;
		if (unlikely(veth_xdp_tx(rq, xdp, bq) < 0)) {
			trace_xdp_exception(rq->dev, xdp_prog, act);
			stats->rx_drops++;
			goto err_xdp;
		}
		stats->xdp_tx++;
		rcu_read_unlock();
		goto xdp_xmit;
	case XDP_REDIRECT:
		veth_xdp_get(xdp);
		consume_skb(skb);
		xdp->rxq->mem = rq->xdp_mem;
		if (xdp_do_redirect(rq->dev, xdp, xdp_prog)) {  /* T.5 */
			stats->rx_drops++;
			goto err_xdp;
		}
		stats->xdp_redirect++;
		rcu_read_unlock();
		goto xdp_xmit;
	default:
		bpf_warn_invalid_xdp_action(rq->dev, xdp_prog, act);
		fallthrough;
	case XDP_ABORTED:
		trace_xdp_exception(rq->dev, xdp_prog, act);   /* observable */
		fallthrough;
	case XDP_DROP:
		stats->xdp_drops++;
		goto xdp_drop;
	}
	rcu_read_unlock();

	/* The program may have adjusted data/data_end -- fix up the skb */
	off = orig_data - xdp->data;
	if (off > 0)
		__skb_push(skb, off);
	else if (off < 0)
		__skb_pull(skb, -off);
	...
	metalen = xdp->data - xdp->data_meta;
	if (metalen)
		skb_metadata_set(skb, metalen);   /* T.2: data_meta survives */
out:
	return skb;
	...
}
```

Note the fix-up after `XDP_PASS`: the program may have pushed or pulled headers, so the skb's pointers must be recomputed. Drivers that get this wrong produce corrupted packets only when a program adjusts the headers, which is why it is a recurring bug class.

### 1.3 Redirect and the flush

```c
int xdp_do_redirect(struct net_device *dev, struct xdp_buff *xdp,
		    struct bpf_prog *xdp_prog)
{
	struct bpf_redirect_info *ri = bpf_net_ctx_get_ri();
	enum bpf_map_type map_type = ri->map_type;

	if (map_type == BPF_MAP_TYPE_XSKMAP)
		return __xdp_do_redirect_xsk(ri, dev, xdp, xdp_prog);

	return __xdp_do_redirect_frame(ri, dev, xdp_convert_buff_to_frame(xdp),
				       xdp_prog);
}

static __always_inline int
__xdp_do_redirect_frame(struct bpf_redirect_info *ri, struct net_device *dev,
			struct xdp_frame *xdpf, struct bpf_prog *xdp_prog)
{
	enum bpf_map_type map_type = ri->map_type;
	void *fwd = ri->tgt_value;
	u32 map_id = ri->map_id;
	int err;

	ri->map_id = 0;
	ri->map_type = BPF_MAP_TYPE_UNSPEC;

	if (unlikely(!xdpf)) { err = -EOVERFLOW; goto err; }

	switch (map_type) {
	case BPF_MAP_TYPE_DEVMAP:
		fallthrough;
	case BPF_MAP_TYPE_DEVMAP_HASH:
		map = READ_ONCE(ri->map);
		if (unlikely(map)) {
			WRITE_ONCE(ri->map, NULL);
			err = dev_map_enqueue_multi(xdpf, dev, map,
						    ri->flags & BPF_F_EXCLUDE_INGRESS);
		} else {
			err = dev_map_enqueue(fwd, xdpf, dev);   /* QUEUED, not sent */
		}
		break;
	case BPF_MAP_TYPE_CPUMAP:
		err = cpu_map_enqueue(fwd, xdpf, dev);
		break;
	case BPF_MAP_TYPE_XSKMAP:
		err = __xsk_map_redirect(fwd, xdp);
		break;
	...
	}
	...
}

/* Called at the END of the NAPI poll -- T.5's batching */
void xdp_do_flush(void)
{
	__dev_flush();
	__cpu_map_flush();
	__xsk_map_flush();
}
```

`dev_map_enqueue` puts the frame on a per-CPU bulk queue; `__dev_flush` transmits the whole batch with one `ndo_xdp_xmit` call. **That batching is where redirect's throughput comes from.**

```c
static void bq_xmit_all(struct xdp_dev_bulk_queue *bq, u32 flags)
{
	struct net_device *dev = bq->dev;
	unsigned int cnt = bq->count;
	int sent = 0, err = 0;
	int to_send = cnt;
	int i;

	if (unlikely(!cnt))
		return;

	for (i = 0; i < cnt; i++) {
		struct xdp_frame *xdpf = bq->q[i];
		prefetch(xdpf);
	}
	...
	/* ONE call for the whole batch */
	sent = dev->netdev_ops->ndo_xdp_xmit(dev, to_send, bq->q, flags);
	...
	trace_xdp_devmap_xmit(bq->dev_rx, dev, sent, cnt - sent, err);
	bq->count = 0;
}
```

### 1.4 CPUMAP

```c
static int cpu_map_kthread_run(void *data)
{
	struct bpf_cpu_map_entry *rcpu = data;
	unsigned long last_qs = jiffies;

	complete(&rcpu->kthread_running);
	set_current_state(TASK_INTERRUPTIBLE);

	while (!kthread_should_stop() || !__ptr_ring_empty(rcpu->queue)) {
		struct xdp_cpumap_stats stats = {};
		unsigned int drops = 0, sched = 0;
		void *frames[CPUMAP_BATCH];
		void *skbs[CPUMAP_BATCH];
		int i, n, m, nframes, xdp_n;

		/* Wait for work */
		if (__ptr_ring_empty(rcpu->queue)) {
			set_current_state(TASK_INTERRUPTIBLE);
			if (__ptr_ring_empty(rcpu->queue)) {
				schedule();
				sched = 1;
				last_qs = jiffies;
			} else {
				__set_current_state(TASK_RUNNING);
			}
		}
		...
		n = __ptr_ring_consume_batched(rcpu->queue, frames, CPUMAP_BATCH);
		for (i = 0, xdp_n = 0; i < n; i++) {
			void *f = frames[i];
			struct page *page;
			...
			page = virt_to_page(f);
			prefetchw(page);     /* the destination CPU warms its cache */
		}

		/* An OPTIONAL second XDP program on the destination CPU */
		nframes = cpu_map_bpf_prog_run(rcpu, frames, xdp_n, &stats, &list);
		...
		/* NOW build skbs, on the DESTINATION CPU */
		m = kmem_cache_alloc_bulk(net_hotdata.skbuff_cache, gfp,
					  nframes, skbs);
		...
		for (i = 0; i < nframes; i++) {
			struct xdp_frame *xdpf = frames[i];
			struct sk_buff *skb = skbs[i];

			skb = __xdp_build_skb_from_frame(xdpf, skb, xdpf->dev_rx);
			...
			napi_gro_receive(&rcpu->napi, skb);   /* into the stack */
		}
		...
	}
	...
}
```

**The skb is allocated on the destination CPU**, which means the allocation, the GRO, and the stack processing all happen where the data will be consumed. That cache locality is much of CPUMAP's value beyond the steering itself.

### 1.5 AF_XDP's rings

```c
/* include/uapi/linux/if_xdp.h -- the USERSPACE ABI */
struct xdp_ring_offset {
	__u64 producer;
	__u64 consumer;
	__u64 desc;
	__u64 flags;
};

struct xdp_mmap_offsets {
	struct xdp_ring_offset rx;
	struct xdp_ring_offset tx;
	struct xdp_ring_offset fr;   /* fill */
	struct xdp_ring_offset cr;   /* completion */
};

struct xdp_umem_reg {
	__u64 addr;          /* the userspace region */
	__u64 len;
	__u32 chunk_size;
	__u32 headroom;
	__u32 flags;
	__u32 tx_metadata_len;
};

#define XDP_SHARED_UMEM		(1 << 0)
#define XDP_COPY		(1 << 1)
#define XDP_ZEROCOPY		(1 << 2)
#define XDP_USE_NEED_WAKEUP	(1 << 3)   /* T.6's syscall elimination */
#define XDP_USE_SG		(1 << 4)
```

The ring is a lock-free single-producer/single-consumer queue:

```c
/* net/xdp/xsk_queue.h */
struct xsk_queue {
	u32 ring_mask;
	u32 nentries;
	u32 cached_prod;
	u32 cached_cons;
	struct xdp_ring *ring;
	u64 invalid_descs;
	u64 queue_empty_descs;
	size_t ring_vmalloc_size;
};

static inline bool xskq_cons_read_desc(struct xsk_queue *q,
				       struct xdp_desc *desc,
				       struct xsk_buff_pool *pool)
{
	if (q->cached_cons != q->cached_prod) {
		struct xdp_rxtx_ring *ring = (struct xdp_rxtx_ring *)q->ring;
		u32 idx = q->cached_cons & q->ring_mask;

		*desc = ring->desc[idx];
		if (xskq_cons_is_valid_desc(q, desc, pool))
			return true;

		q->cached_cons++;
	}
	return false;
}

static inline void __xskq_cons_release(struct xsk_queue *q)
{
	smp_store_release(&q->ring->consumer, q->cached_cons);  /* RELEASE */
}

static inline void __xskq_cons_peek(struct xsk_queue *q)
{
	q->cached_prod = smp_load_acquire(&q->ring->producer);  /* ACQUIRE */
}
```

**`smp_store_release` / `smp_load_acquire` on the producer and consumer indices** is the entire synchronisation. Ch. 12's acquire/release pairing, in a userspace-visible ABI — and getting it wrong in an application produces exactly the torn-read bugs that chapter describes.

The `cached_prod`/`cached_cons` are a performance detail worth understanding: rather than reading the shared index on every descriptor, the code caches it and only re-reads when the cache is exhausted. That turns a shared-cache-line read per packet into one per batch.

### 1.6 `XDP_USE_NEED_WAKEUP`

```c
static inline void xsk_set_rx_need_wakeup(struct xsk_buff_pool *pool)
{
	if (pool->cached_need_wakeup & XDP_WAKEUP_RX)
		return;
	pool->fq->ring->flags |= XDP_RING_NEED_WAKEUP;
	pool->cached_need_wakeup |= XDP_WAKEUP_RX;
}

static inline bool xsk_uses_need_wakeup(struct xsk_buff_pool *pool)
{
	return pool->uses_need_wakeup;
}
```

And the application side:

```c
	if (xsk_ring_prod__needs_wakeup(&xsk->fq)) {
		recvfrom(xsk_socket__fd(xsk->xsk), NULL, 0, MSG_DONTWAIT, NULL, NULL);
	}
	/* Otherwise: NO SYSCALL AT ALL */
```

Under sustained load the flag is never set, so the application runs entirely in userspace, reading and writing shared memory. **Zero syscalls per packet** is what makes AF_XDP competitive with DPDK.

### 1.7 Observability

| Where | What |
|---|---|
| `ip link show` ★★★ | `xdp`/`xdpgeneric`/`xdpdrv`/`xdpoffload` and the program ID |
| `bpftool prog show` ★★★ | loaded programs, their type, and run statistics |
| `bpftool prog profile` | instructions, cycles, cache misses per run |
| `bpftool map dump` ★★★ | map contents |
| `bpftool net show` ★★★ | **what is attached where** |
| `bpftool prog tracelog` | `bpf_printk` output |
| `trace-cmd record -e xdp:\*` ★★★ | `xdp_exception`, `xdp_redirect`, `xdp_devmap_xmit`, `xdp_cpumap_enqueue` |
| `ethtool -S` ★★★ | driver XDP counters (`rx_xdp_drop`, `rx_xdp_tx`, `rx_xdp_redirect`) |
| `/proc/net/xdp_stats` (some drivers) | |
| `xdp-loader`, `xdp-bench`, `xdp-filter` (xdp-tools) ★★★ | |
| `bpftrace` on `xdp:xdp_exception` ★★★ | **monitor this in production** |
| `perf record -e cycles -a` with the program symbol | |

`bpftool prog profile` is worth highlighting:

```sh
sudo bpftool prog profile id 42 duration 10 cycles instructions llc_misses
```

```
      4839201 run_cnt
     87105618 cycles              (18.0 cycles/run)
    121980023 instructions        (25.2 insns/run)
       241960 llc_misses          (0.05 misses/run)
```

**18 cycles per packet** for a simple program. That is the number to aim for, and it makes the cost model concrete.

---

## 2. Practice

### Lab 74.1 — Setup and a first program

```sh
sudo apt install -y clang llvm libbpf-dev libelf-dev linux-headers-$(uname -r) \
     bpftool linux-tools-common linux-tools-$(uname -r) iproute2 tcpdump

# Check driver support -- T.4
for i in $(ls /sys/class/net/); do
  echo -n "$i: "
  sudo ip link set dev $i xdpdrv off 2>&1 | grep -q 'not supported' && \
    echo "no native XDP" || echo "native XDP likely"
done

# A veth pair always supports native XDP
sudo ip netns add xdpns
sudo ip link add veth0 type veth peer name veth1
sudo ip link set veth1 netns xdpns
sudo ip addr add 10.77.0.1/24 dev veth0
sudo ip link set veth0 up
sudo ip netns exec xdpns ip addr add 10.77.0.2/24 dev veth1
sudo ip netns exec xdpns ip link set veth1 up
sudo ip netns exec xdpns ip link set lo up
ping -c 2 10.77.0.2
```

The minimal program:

```c
// SPDX-License-Identifier: GPL-2.0
/* xdp_pass.c */
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

SEC("xdp")
int xdp_pass_prog(struct xdp_md *ctx)
{
	return XDP_PASS;
}

char _license[] SEC("license") = "GPL";
```

```sh
clang -O2 -g -Wall -target bpf -c xdp_pass.c -o xdp_pass.o
llvm-objdump -d xdp_pass.o

sudo ip link set dev veth0 xdpdrv obj xdp_pass.o sec xdp
ip link show veth0
sudo bpftool prog show
sudo bpftool net show
ping -c 2 10.77.0.2

sudo ip link set dev veth0 xdpdrv off
```

Counting, with bounds checks — §T.2:

```c
// SPDX-License-Identifier: GPL-2.0
/* xdp_count.c */
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <linux/ipv6.h>
#include <linux/tcp.h>
#include <linux/udp.h>
#include <linux/in.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>

struct {
	__uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
	__uint(max_entries, 256);
	__type(key, __u32);
	__type(value, __u64);
} proto_count SEC(".maps");

struct {
	__uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
	__uint(max_entries, 8);
	__type(key, __u32);
	__type(value, __u64);
} stats SEC(".maps");

#define STAT_TOTAL   0
#define STAT_IPV4    1
#define STAT_IPV6    2
#define STAT_OTHER   3
#define STAT_BYTES   4

static __always_inline void bump(void *map, __u32 key, __u64 n)
{
	__u64 *v = bpf_map_lookup_elem(map, &key);

	if (v)
		*v += n;
}

SEC("xdp")
int xdp_count_prog(struct xdp_md *ctx)
{
	void *data = (void *)(long)ctx->data;
	void *data_end = (void *)(long)ctx->data_end;
	struct ethhdr *eth = data;
	__u32 proto = 0;

	bump(&stats, STAT_TOTAL, 1);
	bump(&stats, STAT_BYTES, data_end - data);

	/* T.2: MANDATORY bounds check before EVERY access */
	if ((void *)(eth + 1) > data_end)
		return XDP_DROP;

	if (eth->h_proto == bpf_htons(ETH_P_IP)) {
		struct iphdr *iph = (void *)(eth + 1);

		if ((void *)(iph + 1) > data_end)
			return XDP_DROP;
		proto = iph->protocol;
		bump(&stats, STAT_IPV4, 1);
	} else if (eth->h_proto == bpf_htons(ETH_P_IPV6)) {
		struct ipv6hdr *ip6 = (void *)(eth + 1);

		if ((void *)(ip6 + 1) > data_end)
			return XDP_DROP;
		proto = ip6->nexthdr;
		bump(&stats, STAT_IPV6, 1);
	} else {
		bump(&stats, STAT_OTHER, 1);
		return XDP_PASS;
	}

	bump(&proto_count, proto, 1);
	return XDP_PASS;
}

char _license[] SEC("license") = "GPL";
```

```sh
clang -O2 -g -target bpf -c xdp_count.c -o xdp_count.o
sudo ip link set dev veth0 xdpdrv obj xdp_count.o sec xdp

ping -c 5 10.77.0.2 > /dev/null
sudo ip netns exec xdpns nc -z -w1 10.77.0.1 22 2>/dev/null

sudo bpftool map dump name stats
sudo bpftool map dump name proto_count | grep -B1 -A2 '"value"' | head -20
```

Remove a bounds check and see the verifier reject it:

```c
	/* if ((void *)(eth + 1) > data_end) return XDP_DROP; */
	if (eth->h_proto == bpf_htons(ETH_P_IP)) { ... }
```

```sh
clang -O2 -target bpf -c xdp_bad.c -o xdp_bad.o
sudo ip link set dev veth0 xdpdrv obj xdp_bad.o sec xdp 2>&1 | head -20
# "invalid access to packet, off=12 size=2, R1(id=0,off=0,r=0)"
```

**The verifier catches it at load time.** §T.2's point.

---

### Lab 74.2 — The five actions, measured

```c
// SPDX-License-Identifier: GPL-2.0
/* xdp_actions.c -- selectable action, for benchmarking. */
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>

struct {
	__uint(type, BPF_MAP_TYPE_ARRAY);
	__uint(max_entries, 1);
	__type(key, __u32);
	__type(value, __u32);
} action_map SEC(".maps");

struct {
	__uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
	__uint(max_entries, 5);
	__type(key, __u32);
	__type(value, __u64);
} action_count SEC(".maps");

static __always_inline void swap_mac(struct ethhdr *eth)
{
	__u8 tmp[ETH_ALEN];

	__builtin_memcpy(tmp, eth->h_source, ETH_ALEN);
	__builtin_memcpy(eth->h_source, eth->h_dest, ETH_ALEN);
	__builtin_memcpy(eth->h_dest, tmp, ETH_ALEN);
}

SEC("xdp")
int xdp_action_prog(struct xdp_md *ctx)
{
	void *data = (void *)(long)ctx->data;
	void *data_end = (void *)(long)ctx->data_end;
	struct ethhdr *eth = data;
	__u32 key = 0, *action, act;
	__u64 *cnt;

	if ((void *)(eth + 1) > data_end)
		return XDP_DROP;

	action = bpf_map_lookup_elem(&action_map, &key);
	act = action ? *action : XDP_PASS;

	if (act == XDP_TX)
		swap_mac(eth);

	cnt = bpf_map_lookup_elem(&action_count, &act);
	if (cnt)
		*cnt += 1;

	return act;
}

char _license[] SEC("license") = "GPL";
```

```sh
clang -O2 -g -target bpf -c xdp_actions.c -o xdp_actions.o
sudo ip link set dev veth0 xdpdrv obj xdp_actions.o sec xdp

set_action() {
  sudo bpftool map update name action_map key 0 0 0 0 value $1 0 0 0
}

# 1 = DROP
set_action 1
ping -c 3 -W 1 10.77.0.2 2>&1 | tail -2    # 100% loss

# 2 = PASS
set_action 2
ping -c 3 10.77.0.2 2>&1 | tail -2         # works

sudo bpftool map dump name action_count
```

Measure the rates — §T.3:

```sh
# pktgen from the kernel samples generates load
ls /usr/src/linux*/samples/pktgen/ 2>/dev/null || \
  git clone --depth 1 https://github.com/torvalds/linux.git /tmp/linux 2>/dev/null

# Or use xdp-tools' generator
git clone --depth 1 https://github.com/xdp-project/xdp-tools.git 2>/dev/null
cd xdp-tools && ./configure && make 2>/dev/null
sudo ./xdp-bench/xdp-bench drop veth0 2>/dev/null | head -10
```

A simpler measurement:

```sh
sudo bpftool prog show | grep xdp_action
PROGID=$(sudo bpftool prog show | grep -B0 xdp_action | grep -oP '^\d+')

for act in 1 2; do
  set_action $act
  echo "=== action $act ($([ $act -eq 1 ] && echo DROP || echo PASS)) ==="
  sudo bpftool prog profile id $PROGID duration 5 cycles instructions 2>/dev/null
done
```

`XDP_TX` — bounce packets back:

```sh
set_action 3
sudo tcpdump -i veth0 -c 5 -nn 2>/dev/null &
ping -c 3 -W 1 10.77.0.2 > /dev/null 2>&1
wait
# ICMP echo requests come back with the MACs swapped.
sudo bpftool map dump name action_count
```

`XDP_ABORTED` and its tracepoint:

```sh
set_action 0
sudo bpftrace -e 'tracepoint:xdp:xdp_exception {
	printf("XDP_ABORTED on ifindex %d, prog %d, act %d\n",
	       args->ifindex, args->prog_id, args->act);
}' &
ping -c 3 -W 1 10.77.0.2 > /dev/null 2>&1
sleep 2
set_action 2
```

Native versus generic — §T.4:

```sh
sudo ip link set dev veth0 xdpdrv off
for mode in xdpdrv xdpgeneric; do
  sudo ip link set dev veth0 $mode obj xdp_actions.o sec xdp 2>/dev/null || continue
  set_action 1
  echo "=== $mode ==="
  PROGID=$(sudo bpftool prog show | grep xdp_action | grep -oP '^\d+' | tail -1)
  sudo bpftool prog profile id $PROGID duration 5 cycles 2>/dev/null | head -4
  sudo ip link set dev veth0 $mode off
done
```

**Generic mode is dramatically slower**, because the skb was already built.

---

### Lab 74.3 — Redirect and maps

Set up a three-namespace topology to forward through:

```sh
sudo ip link set dev veth0 xdpdrv off 2>/dev/null
sudo ip netns del xdpns 2>/dev/null
sudo ip link del veth0 2>/dev/null

sudo ip netns add a
sudo ip netns add b
sudo ip link add va type veth peer name va-br
sudo ip link add vb type veth peer name vb-br
sudo ip link set va netns a
sudo ip link set vb netns b
sudo ip link set va-br up
sudo ip link set vb-br up

sudo ip netns exec a ip addr add 10.88.0.1/24 dev va
sudo ip netns exec a ip link set va up
sudo ip netns exec a ip link set lo up
sudo ip netns exec b ip addr add 10.88.0.2/24 dev vb
sudo ip netns exec b ip link set vb up
sudo ip netns exec b ip link set lo up

# No bridge, no routing: they cannot reach each other
sudo ip netns exec a ping -c 1 -W 1 10.88.0.2 2>&1 | tail -1
```

Now bridge them in XDP — §T.5:

```c
// SPDX-License-Identifier: GPL-2.0
/* xdp_redirect.c */
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <bpf/bpf_helpers.h>

struct {
	__uint(type, BPF_MAP_TYPE_DEVMAP);
	__uint(max_entries, 64);
	__type(key, __u32);
	__type(value, __u32);
} tx_port SEC(".maps");

struct {
	__uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
	__uint(max_entries, 4);
	__type(key, __u32);
	__type(value, __u64);
} redirect_stats SEC(".maps");

SEC("xdp")
int xdp_redirect_prog(struct xdp_md *ctx)
{
	void *data = (void *)(long)ctx->data;
	void *data_end = (void *)(long)ctx->data_end;
	struct ethhdr *eth = data;
	__u32 key = ctx->ingress_ifindex;
	__u64 *cnt;
	__u32 zero = 0;

	if ((void *)(eth + 1) > data_end)
		return XDP_DROP;

	cnt = bpf_map_lookup_elem(&redirect_stats, &zero);
	if (cnt)
		*cnt += 1;

	/* T.5: the map lookup names the target */
	return bpf_redirect_map(&tx_port, key, 0);
}

char _license[] SEC("license") = "GPL";
```

```sh
clang -O2 -g -target bpf -c xdp_redirect.c -o xdp_redirect.o

IFA=$(cat /sys/class/net/va-br/ifindex)
IFB=$(cat /sys/class/net/vb-br/ifindex)
echo "va-br=$IFA vb-br=$IFB"

sudo ip link set dev va-br xdpdrv obj xdp_redirect.o sec xdp
sudo ip link set dev vb-br xdpdrv obj xdp_redirect.o sec xdp

# Map each ingress ifindex to the OTHER device
sudo bpftool map update name tx_port key hex $(printf '%08x' $IFA | sed 's/\(..\)/\1 /g' | \
     awk '{print $4, $3, $2, $1}') value hex $(printf '%08x' $IFB | \
     sed 's/\(..\)/\1 /g' | awk '{print $4, $3, $2, $1}') 2>/dev/null

# Simpler with a helper
cat > setmap.sh <<'EOF'
#!/bin/bash
le32() { printf '%08x' $1 | sed 's/../& /g' | awk '{print $4,$3,$2,$1}'; }
sudo bpftool map update name tx_port key $(le32 $1) value $(le32 $2)
EOF
chmod +x setmap.sh
./setmap.sh $IFA $IFB
./setmap.sh $IFB $IFA
sudo bpftool map dump name tx_port

sudo ip netns exec a ping -c 3 10.88.0.2
```

**An XDP bridge in 30 lines.**

Watch the redirect:

```sh
sudo bpftrace -e '
tracepoint:xdp:xdp_redirect     { @ok = count(); }
tracepoint:xdp:xdp_redirect_err { @err = count(); }
tracepoint:xdp:xdp_devmap_xmit  { @xmit_batch = hist(args->sent); }
interval:s:5 { print(@ok); print(@err); print(@xmit_batch);
               clear(@ok); clear(@err); clear(@xmit_batch); }' &

sudo ip netns exec a ping -c 20 -i 0.05 10.88.0.2 > /dev/null
```

`@xmit_batch` shows §T.5's batching in action.

CPUMAP — steering to a CPU:

```c
// SPDX-License-Identifier: GPL-2.0
/* xdp_cpumap.c */
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>

struct {
	__uint(type, BPF_MAP_TYPE_CPUMAP);
	__uint(max_entries, 64);
	__type(key, __u32);
	__type(value, __u32);
} cpu_map SEC(".maps");

struct {
	__uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
	__uint(max_entries, 64);
	__type(key, __u32);
	__type(value, __u64);
} cpu_count SEC(".maps");

SEC("xdp")
int xdp_cpumap_prog(struct xdp_md *ctx)
{
	void *data = (void *)(long)ctx->data;
	void *data_end = (void *)(long)ctx->data_end;
	struct ethhdr *eth = data;
	struct iphdr *iph;
	__u32 cpu;
	__u64 *cnt;

	if ((void *)(eth + 1) > data_end)
		return XDP_PASS;
	if (eth->h_proto != bpf_htons(ETH_P_IP))
		return XDP_PASS;

	iph = (void *)(eth + 1);
	if ((void *)(iph + 1) > data_end)
		return XDP_PASS;

	/* OUR hash, not the NIC's -- T.5's point */
	cpu = (bpf_ntohl(iph->saddr) ^ bpf_ntohl(iph->daddr)) % 4;

	cnt = bpf_map_lookup_elem(&cpu_count, &cpu);
	if (cnt)
		*cnt += 1;

	return bpf_redirect_map(&cpu_map, cpu, XDP_PASS);
}

char _license[] SEC("license") = "GPL";
```

```sh
clang -O2 -g -target bpf -c xdp_cpumap.c -o xdp_cpumap.o
sudo ip link set dev va-br xdpdrv off 2>/dev/null
sudo ip link set dev va-br xdpdrv obj xdp_cpumap.o sec xdp

# Populate the CPUMAP: value = the queue size for each CPU's kthread
for cpu in 0 1 2 3; do
  ./setmap.sh $cpu 192 2>/dev/null
done
sudo bpftool map dump name cpu_map 2>/dev/null

sudo bpftrace -e '
tracepoint:xdp:xdp_cpumap_enqueue {
	@to_cpu[args->to_cpu] = count();
}
tracepoint:xdp:xdp_cpumap_kthread { @kthread[cpu] = count(); }
interval:s:8 { print(@to_cpu); print(@kthread); exit(); }' &
sudo ip netns exec a ping -c 40 -i 0.05 10.88.0.1 > /dev/null 2>&1
wait
```

---

### Lab 74.4 — An XDP load balancer

The Katran-style pattern — §T.3's `XDP_TX`.

```c
// SPDX-License-Identifier: GPL-2.0
/* xdp_lb.c -- a minimal L4 load balancer using IP-in-IP encapsulation. */
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <linux/tcp.h>
#include <linux/in.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>

#define MAX_BACKENDS 16

struct backend {
	__u32 ip;
	__u8  mac[ETH_ALEN];
	__u16 pad;
};

struct {
	__uint(type, BPF_MAP_TYPE_ARRAY);
	__uint(max_entries, MAX_BACKENDS);
	__type(key, __u32);
	__type(value, struct backend);
} backends SEC(".maps");

struct {
	__uint(type, BPF_MAP_TYPE_ARRAY);
	__uint(max_entries, 1);
	__type(key, __u32);
	__type(value, __u32);
} config SEC(".maps");          /* [0] = number of active backends */

struct {
	__uint(type, BPF_MAP_TYPE_PERCPU_HASH);
	__uint(max_entries, 65536);
	__type(key, __u64);          /* the flow hash */
	__type(value, __u32);        /* the chosen backend */
} flow_table SEC(".maps");

struct {
	__uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
	__uint(max_entries, MAX_BACKENDS);
	__type(key, __u32);
	__type(value, __u64);
} backend_stats SEC(".maps");

static __always_inline __u16 csum_fold(__u32 csum)
{
	csum = (csum & 0xffff) + (csum >> 16);
	csum = (csum & 0xffff) + (csum >> 16);
	return (__u16)~csum;
}

static __always_inline __u32 csum_add(__u32 csum, __u32 addend)
{
	csum += addend;
	return csum + (csum < addend);
}

SEC("xdp")
int xdp_lb_prog(struct xdp_md *ctx)
{
	void *data = (void *)(long)ctx->data;
	void *data_end = (void *)(long)ctx->data_end;
	struct ethhdr *eth = data;
	struct iphdr *iph;
	struct tcphdr *tcph;
	struct backend *be;
	__u32 key = 0, *nbackends, idx, *cached;
	__u64 flow;
	__u64 *cnt;
	__u32 csum;
	int i;

	if ((void *)(eth + 1) > data_end)
		return XDP_DROP;
	if (eth->h_proto != bpf_htons(ETH_P_IP))
		return XDP_PASS;

	iph = (void *)(eth + 1);
	if ((void *)(iph + 1) > data_end)
		return XDP_DROP;
	if (iph->protocol != IPPROTO_TCP)
		return XDP_PASS;

	tcph = (void *)iph + iph->ihl * 4;
	if ((void *)(tcph + 1) > data_end)
		return XDP_DROP;

	/* Only load-balance our VIP's port */
	if (tcph->dest != bpf_htons(8080))
		return XDP_PASS;

	nbackends = bpf_map_lookup_elem(&config, &key);
	if (!nbackends || *nbackends == 0)
		return XDP_PASS;

	/* Flow affinity: the same connection always goes to the same backend */
	flow = ((__u64)iph->saddr << 32) | ((__u64)tcph->source << 16) |
	       bpf_ntohs(tcph->dest);

	cached = bpf_map_lookup_elem(&flow_table, &flow);
	if (cached) {
		idx = *cached;
	} else {
		idx = (bpf_ntohl(iph->saddr) ^ bpf_ntohs(tcph->source)) % *nbackends;
		bpf_map_update_elem(&flow_table, &flow, &idx, BPF_ANY);
	}

	be = bpf_map_lookup_elem(&backends, &idx);
	if (!be || be->ip == 0)
		return XDP_PASS;

	/* Rewrite the destination, fixing the checksum INCREMENTALLY */
	csum = ~((__u32)iph->check) & 0xffff;
	csum = csum_add(csum, ~bpf_ntohl(iph->daddr) & 0xffff);
	csum = csum_add(csum, (~bpf_ntohl(iph->daddr) >> 16) & 0xffff);
	iph->daddr = be->ip;
	csum = csum_add(csum, bpf_ntohl(iph->daddr) & 0xffff);
	csum = csum_add(csum, (bpf_ntohl(iph->daddr) >> 16) & 0xffff);
	iph->check = csum_fold(csum);

	/* Rewrite the L2 destination */
	__builtin_memcpy(eth->h_dest, be->mac, ETH_ALEN);

	cnt = bpf_map_lookup_elem(&backend_stats, &idx);
	if (cnt)
		*cnt += 1;

	return XDP_TX;          /* T.3: back out the same interface */
}

char _license[] SEC("license") = "GPL";
```

```sh
clang -O2 -g -target bpf -c xdp_lb.c -o xdp_lb.o
sudo ip link set dev va-br xdpdrv off 2>/dev/null
sudo ip link set dev va-br xdpdrv obj xdp_lb.o sec xdp

# Populate: 2 backends
sudo bpftool map update name config key 0 0 0 0 value 2 0 0 0
sudo bpftool map dump name config

sudo bpftool prog show
sudo bpftool map show

sudo bpftrace -e '
tracepoint:xdp:xdp_exception { @abort = count(); }
interval:s:5 { print(@abort); clear(@abort); }' &

sudo ip netns exec a nc -z -w1 10.88.0.100 8080 2>/dev/null
sudo bpftool map dump name backend_stats
sudo bpftool map dump name flow_table | head
```

**Flow affinity via a map** is the key correctness property: a TCP connection must always reach the same backend, or it breaks. Katran solves this more elegantly with consistent hashing (Maglev), so adding or removing a backend disturbs few flows.

---

### Lab 74.5 — AF_XDP

```c
// SPDX-License-Identifier: GPL-2.0
/* afxdp.c -- a minimal AF_XDP receiver demonstrating T.6's four rings. */
#define _GNU_SOURCE
#include <errno.h>
#include <getopt.h>
#include <net/if.h>
#include <poll.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/resource.h>
#include <arpa/inet.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <linux/udp.h>
#include <bpf/libbpf.h>
#include <xdp/xsk.h>

#define NUM_FRAMES	4096
#define FRAME_SIZE	XSK_UMEM__DEFAULT_FRAME_SIZE
#define RX_BATCH_SIZE	64

struct xsk_umem_info {
	struct xsk_ring_prod fq;    /* T.6: FILL */
	struct xsk_ring_cons cq;    /* T.6: COMPLETION */
	struct xsk_umem *umem;
	void *buffer;
};

struct xsk_socket_info {
	struct xsk_ring_cons rx;    /* T.6: RX */
	struct xsk_ring_prod tx;    /* T.6: TX */
	struct xsk_umem_info *umem;
	struct xsk_socket *xsk;
	__u64 umem_frame_addr[NUM_FRAMES];
	__u32 umem_frame_free;
	unsigned long rx_packets, rx_bytes;
};

static volatile int stop;
static void on_sigint(int s) { stop = 1; }

static struct xsk_umem_info *configure_umem(void *buffer, __u64 size)
{
	struct xsk_umem_info *umem = calloc(1, sizeof(*umem));
	int ret;

	if (!umem) return NULL;
	ret = xsk_umem__create(&umem->umem, buffer, size, &umem->fq, &umem->cq, NULL);
	if (ret) { errno = -ret; return NULL; }
	umem->buffer = buffer;
	return umem;
}

static __u64 alloc_frame(struct xsk_socket_info *xsk)
{
	__u64 frame;

	if (xsk->umem_frame_free == 0)
		return -1;
	frame = xsk->umem_frame_addr[--xsk->umem_frame_free];
	xsk->umem_frame_addr[xsk->umem_frame_free] = -1;
	return frame;
}

static void free_frame(struct xsk_socket_info *xsk, __u64 frame)
{
	xsk->umem_frame_addr[xsk->umem_frame_free++] = frame;
}

static struct xsk_socket_info *configure_socket(struct xsk_umem_info *umem,
						const char *ifname, int queue)
{
	struct xsk_socket_config cfg = {
		.rx_size = XSK_RING_CONS__DEFAULT_NUM_DESCS,
		.tx_size = XSK_RING_PROD__DEFAULT_NUM_DESCS,
		.libbpf_flags = XSK_LIBBPF_FLAGS__INHIBIT_PROG_LOAD,
		.xdp_flags = XDP_FLAGS_DRV_MODE,
		.bind_flags = XDP_USE_NEED_WAKEUP,   /* T.6's optimisation */
	};
	struct xsk_socket_info *xsk = calloc(1, sizeof(*xsk));
	__u32 idx, i;
	int ret;

	if (!xsk) return NULL;
	xsk->umem = umem;

	ret = xsk_socket__create(&xsk->xsk, ifname, queue, umem->umem,
				 &xsk->rx, &xsk->tx, &cfg);
	if (ret) { errno = -ret; free(xsk); return NULL; }

	for (i = 0; i < NUM_FRAMES; i++)
		xsk->umem_frame_addr[i] = i * FRAME_SIZE;
	xsk->umem_frame_free = NUM_FRAMES;

	/* T.6: prime the FILL ring with empty chunks */
	ret = xsk_ring_prod__reserve(&xsk->umem->fq,
				     XSK_RING_PROD__DEFAULT_NUM_DESCS, &idx);
	for (i = 0; i < XSK_RING_PROD__DEFAULT_NUM_DESCS; i++)
		*xsk_ring_prod__fill_addr(&xsk->umem->fq, idx++) = alloc_frame(xsk);
	xsk_ring_prod__submit(&xsk->umem->fq,
			      XSK_RING_PROD__DEFAULT_NUM_DESCS);
	return xsk;
}

static void handle_packet(struct xsk_socket_info *xsk, __u64 addr, __u32 len)
{
	__u8 *pkt = xsk_umem__get_data(xsk->umem->buffer, addr);
	struct ethhdr *eth = (void *)pkt;
	struct iphdr *iph;
	char src[INET_ADDRSTRLEN], dst[INET_ADDRSTRLEN];

	xsk->rx_packets++;
	xsk->rx_bytes += len;

	if (len < sizeof(*eth) + sizeof(*iph))
		return;
	if (eth->h_proto != htons(ETH_P_IP))
		return;

	iph = (void *)(eth + 1);
	inet_ntop(AF_INET, &iph->saddr, src, sizeof(src));
	inet_ntop(AF_INET, &iph->daddr, dst, sizeof(dst));

	if (xsk->rx_packets <= 10 || xsk->rx_packets % 10000 == 0)
		printf("pkt %-8lu %s -> %s proto=%d len=%u\n",
		       xsk->rx_packets, src, dst, iph->protocol, len);
}

static void rx_loop(struct xsk_socket_info *xsk)
{
	struct pollfd fds = { .fd = xsk_socket__fd(xsk->xsk), .events = POLLIN };

	while (!stop) {
		unsigned int rcvd, i, stock;
		__u32 idx_rx = 0, idx_fq = 0;
		int ret;

		/* T.6: only syscall if the kernel asked us to */
		if (xsk_ring_prod__needs_wakeup(&xsk->umem->fq)) {
			ret = poll(&fds, 1, 1000);
			if (ret <= 0) continue;
		}

		rcvd = xsk_ring_cons__peek(&xsk->rx, RX_BATCH_SIZE, &idx_rx);
		if (!rcvd)
			continue;

		/* Refill the FILL ring with as many as we consumed */
		stock = xsk_prod_nb_free(&xsk->umem->fq, xsk->umem_frame_free);
		if (stock > 0) {
			ret = xsk_ring_prod__reserve(&xsk->umem->fq, rcvd, &idx_fq);
			while (ret != rcvd) {
				if (ret < 0) return;
				if (xsk_ring_prod__needs_wakeup(&xsk->umem->fq))
					recvfrom(xsk_socket__fd(xsk->xsk), NULL, 0,
						 MSG_DONTWAIT, NULL, NULL);
				ret = xsk_ring_prod__reserve(&xsk->umem->fq, rcvd, &idx_fq);
			}
		}

		for (i = 0; i < rcvd; i++) {
			const struct xdp_desc *d =
				xsk_ring_cons__rx_desc(&xsk->rx, idx_rx++);

			handle_packet(xsk, d->addr, d->len);
			/* Return the chunk to the FILL ring -- T.6's flow control */
			*xsk_ring_prod__fill_addr(&xsk->umem->fq, idx_fq++) =
				xsk_umem__extract_addr(d->addr);
		}

		xsk_ring_prod__submit(&xsk->umem->fq, rcvd);
		xsk_ring_cons__release(&xsk->rx, rcvd);
	}
}

int main(int argc, char **argv)
{
	struct rlimit rl = { RLIM_INFINITY, RLIM_INFINITY };
	struct xsk_umem_info *umem;
	struct xsk_socket_info *xsk;
	void *packet_buffer;
	__u64 buffer_size = NUM_FRAMES * FRAME_SIZE;
	const char *ifname = argc > 1 ? argv[1] : "veth0";
	int queue = argc > 2 ? atoi(argv[2]) : 0;

	setrlimit(RLIMIT_MEMLOCK, &rl);
	signal(SIGINT, on_sigint);

	if (posix_memalign(&packet_buffer, getpagesize(), buffer_size)) {
		perror("posix_memalign"); return 1;
	}
	umem = configure_umem(packet_buffer, buffer_size);
	if (!umem) { perror("umem"); return 1; }

	xsk = configure_socket(umem, ifname, queue);
	if (!xsk) { perror("socket"); return 1; }

	printf("AF_XDP on %s queue %d; %d frames of %d bytes\n",
	       ifname, queue, NUM_FRAMES, FRAME_SIZE);
	rx_loop(xsk);

	printf("\nreceived %lu packets, %lu bytes\n",
	       xsk->rx_packets, xsk->rx_bytes);
	xsk_socket__delete(xsk->xsk);
	xsk_umem__delete(umem->umem);
	free(packet_buffer);
	return 0;
}
```

The XDP program that routes to it:

```c
// SPDX-License-Identifier: GPL-2.0
/* xdp_xsk.c */
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

struct {
	__uint(type, BPF_MAP_TYPE_XSKMAP);
	__uint(max_entries, 64);
	__type(key, __u32);
	__type(value, __u32);
} xsks_map SEC(".maps");

struct {
	__uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
	__uint(max_entries, 2);
	__type(key, __u32);
	__type(value, __u64);
} xsk_stats SEC(".maps");

SEC("xdp")
int xdp_xsk_prog(struct xdp_md *ctx)
{
	__u32 index = ctx->rx_queue_index;
	__u32 k;
	__u64 *v;

	if (bpf_map_lookup_elem(&xsks_map, &index)) {
		k = 0;
		v = bpf_map_lookup_elem(&xsk_stats, &k);
		if (v) *v += 1;
		return bpf_redirect_map(&xsks_map, index, 0);
	}

	k = 1;
	v = bpf_map_lookup_elem(&xsk_stats, &k);
	if (v) *v += 1;
	return XDP_PASS;       /* T.6: not for us -> the kernel stack */
}

char _license[] SEC("license") = "GPL";
```

```sh
sudo apt install -y libxdp-dev libbpf-dev
clang -O2 -g -target bpf -c xdp_xsk.c -o xdp_xsk.o
gcc -O2 -o afxdp afxdp.c -lxdp -lbpf

sudo ip link set dev va-br xdpdrv off 2>/dev/null
sudo ip link set dev va-br xdpdrv obj xdp_xsk.o sec xdp

# Run the receiver
sudo ./afxdp va-br 0 &
sleep 2

sudo ip netns exec a ping -c 10 -i 0.1 10.88.0.2 > /dev/null 2>&1
sudo bpftool map dump name xsk_stats
sleep 2
sudo kill -INT %1
```

**Packets went to userspace with no copy and (mostly) no syscall.**

Verify zero-copy versus copy mode:

```sh
sudo bpftrace -e '
kprobe:xsk_rcv_zc       { @zerocopy = count(); }
kprobe:__xsk_rcv        { @copy = count(); }
kprobe:xsk_generic_rcv  { @generic = count(); }
interval:s:5 { print(@zerocopy); print(@copy); print(@generic);
               clear(@zerocopy); clear(@copy); clear(@generic); }' &

sudo ./afxdp va-br 0 > /dev/null &
sudo ip netns exec a ping -c 20 -i 0.05 10.88.0.2 > /dev/null 2>&1
sudo kill -INT %2 2>/dev/null
```

veth does copy mode; a real NIC with support does zero-copy.

---

### Lab 74.6 — XDP hints and metadata

```c
// SPDX-License-Identifier: GPL-2.0
/* xdp_meta.c -- T.7's hints, plus data_meta for downstream tc. */
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>

struct meta_info {
	__u32 hash;
	__u32 classification;
} __attribute__((aligned(4)));

/* T.7: kfuncs, implemented by the driver */
extern int bpf_xdp_metadata_rx_hash(const struct xdp_md *ctx, __u32 *hash,
				    enum xdp_rss_hash_type *rss_type) __ksym __weak;
extern int bpf_xdp_metadata_rx_timestamp(const struct xdp_md *ctx,
					 __u64 *timestamp) __ksym __weak;

struct {
	__uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
	__uint(max_entries, 4);
	__type(key, __u32);
	__type(value, __u64);
} meta_stats SEC(".maps");

SEC("xdp")
int xdp_meta_prog(struct xdp_md *ctx)
{
	void *data, *data_meta, *data_end;
	struct meta_info *meta;
	struct ethhdr *eth;
	__u32 hash = 0, key;
	__u64 ts = 0, *cnt;
	enum xdp_rss_hash_type rss_type = 0;
	int ret;

	/* Claim metadata space BEFORE reading data pointers */
	ret = bpf_xdp_adjust_meta(ctx, -(int)sizeof(*meta));
	if (ret < 0)
		return XDP_PASS;

	data = (void *)(long)ctx->data;
	data_meta = (void *)(long)ctx->data_meta;
	data_end = (void *)(long)ctx->data_end;

	meta = data_meta;
	if ((void *)(meta + 1) > data)
		return XDP_PASS;

	eth = data;
	if ((void *)(eth + 1) > data_end)
		return XDP_PASS;

	/* T.7: ask the driver, if it can answer */
	if (bpf_xdp_metadata_rx_hash) {
		if (bpf_xdp_metadata_rx_hash(ctx, &hash, &rss_type) == 0) {
			key = 0;
			cnt = bpf_map_lookup_elem(&meta_stats, &key);
			if (cnt) *cnt += 1;
		}
	}
	if (bpf_xdp_metadata_rx_timestamp) {
		if (bpf_xdp_metadata_rx_timestamp(ctx, &ts) == 0) {
			key = 1;
			cnt = bpf_map_lookup_elem(&meta_stats, &key);
			if (cnt) *cnt += 1;
		}
	}

	meta->hash = hash;
	meta->classification = (eth->h_proto == bpf_htons(ETH_P_IP)) ? 1 : 0;

	return XDP_PASS;      /* data_meta survives into the skb */
}

char _license[] SEC("license") = "GPL";
```

And the tc program that reads it:

```c
// SPDX-License-Identifier: GPL-2.0
/* tc_readmeta.c */
#include <linux/bpf.h>
#include <linux/pkt_cls.h>
#include <bpf/bpf_helpers.h>

struct meta_info {
	__u32 hash;
	__u32 classification;
} __attribute__((aligned(4)));

SEC("tc")
int tc_read_meta(struct __sk_buff *skb)
{
	void *data = (void *)(long)skb->data;
	void *data_meta = (void *)(long)skb->data_meta;
	struct meta_info *meta = data_meta;

	if ((void *)(meta + 1) > data)
		return TC_ACT_OK;

	bpf_printk("tc sees XDP metadata: hash=%u class=%u",
		   meta->hash, meta->classification);
	return TC_ACT_OK;
}

char _license[] SEC("license") = "GPL";
```

```sh
clang -O2 -g -target bpf -c xdp_meta.c -o xdp_meta.o
clang -O2 -g -target bpf -c tc_readmeta.c -o tc_readmeta.o

sudo ip link set dev va-br xdpdrv off 2>/dev/null
sudo ip link set dev va-br xdpdrv obj xdp_meta.o sec xdp
sudo tc qdisc add dev va-br clsact 2>/dev/null
sudo tc filter add dev va-br ingress bpf da obj tc_readmeta.o sec tc

sudo ip netns exec a ping -c 3 10.88.0.2 > /dev/null 2>&1
sudo cat /sys/kernel/debug/tracing/trace_pipe | head -5 &
sleep 3
sudo kill %1 2>/dev/null

sudo bpftool map dump name meta_stats
# veth does not implement the hints kfuncs; on mlx5/ice it will.
```

---

### Lab 74.7 — Measure the costs

§T.9's list, quantified.

```sh
# Program cost
sudo ip link set dev va-br xdpdrv off 2>/dev/null
sudo ip link set dev va-br xdpdrv obj xdp_count.o sec xdp
PROGID=$(sudo bpftool prog show | grep xdp_count | grep -oP '^\d+' | tail -1)

sudo ip netns exec a ping -f -c 5000 10.88.0.1 > /dev/null 2>&1 &
sudo bpftool prog profile id $PROGID duration 5 cycles instructions llc_misses 2>/dev/null
wait
```

Map lookup cost — §T.9(d):

```c
// SPDX-License-Identifier: GPL-2.0
/* xdp_maptest.c -- compare map types. */
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>

struct {
	__uint(type, BPF_MAP_TYPE_ARRAY);
	__uint(max_entries, 1024);
	__type(key, __u32);
	__type(value, __u64);
} array_map SEC(".maps");

struct {
	__uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
	__uint(max_entries, 1024);
	__type(key, __u32);
	__type(value, __u64);
} percpu_map SEC(".maps");

struct {
	__uint(type, BPF_MAP_TYPE_HASH);
	__uint(max_entries, 65536);
	__type(key, __u32);
	__type(value, __u64);
} hash_map SEC(".maps");

struct lpm_key { __u32 prefixlen; __u32 addr; };

struct {
	__uint(type, BPF_MAP_TYPE_LPM_TRIE);
	__uint(max_entries, 1024);
	__type(key, struct lpm_key);
	__type(value, __u64);
	__uint(map_flags, BPF_F_NO_PREALLOC);
} lpm_map SEC(".maps");

struct {
	__uint(type, BPF_MAP_TYPE_ARRAY);
	__uint(max_entries, 1);
	__type(key, __u32);
	__type(value, __u32);
} which SEC(".maps");

SEC("xdp")
int xdp_maptest(struct xdp_md *ctx)
{
	void *data = (void *)(long)ctx->data;
	void *data_end = (void *)(long)ctx->data_end;
	struct ethhdr *eth = data;
	struct iphdr *iph;
	__u32 zero = 0, *sel, k;
	__u64 *v;
	struct lpm_key lk;

	if ((void *)(eth + 1) > data_end) return XDP_DROP;
	if (eth->h_proto != bpf_htons(ETH_P_IP)) return XDP_PASS;
	iph = (void *)(eth + 1);
	if ((void *)(iph + 1) > data_end) return XDP_DROP;

	sel = bpf_map_lookup_elem(&which, &zero);
	if (!sel) return XDP_PASS;

	k = bpf_ntohl(iph->saddr) & 1023;
	switch (*sel) {
	case 0: break;                                    /* no lookup */
	case 1: v = bpf_map_lookup_elem(&array_map, &k);  if (v) *v += 1; break;
	case 2: v = bpf_map_lookup_elem(&percpu_map, &k); if (v) *v += 1; break;
	case 3: v = bpf_map_lookup_elem(&hash_map, &k);   if (v) *v += 1; break;
	case 4:
		lk.prefixlen = 32; lk.addr = iph->saddr;
		v = bpf_map_lookup_elem(&lpm_map, &lk);
		break;
	}
	return XDP_DROP;
}

char _license[] SEC("license") = "GPL";
```

```sh
clang -O2 -g -target bpf -c xdp_maptest.c -o xdp_maptest.o
sudo ip link set dev va-br xdpdrv off 2>/dev/null
sudo ip link set dev va-br xdpdrv obj xdp_maptest.o sec xdp
PROGID=$(sudo bpftool prog show | grep xdp_maptest | grep -oP '^\d+' | tail -1)

for k in $(seq 0 1023); do
  sudo bpftool map update name hash_map key hex $(printf '%08x' $k | \
    sed 's/../& /g' | awk '{print $4,$3,$2,$1}') value 0 0 0 0 0 0 0 0 2>/dev/null
done

for sel in 0 1 2 3 4; do
  case $sel in
    0) N="no lookup";; 1) N="array";; 2) N="percpu array";;
    3) N="hash";; 4) N="lpm trie";;
  esac
  sudo bpftool map update name which key 0 0 0 0 value $sel 0 0 0
  sudo ip netns exec a ping -f -c 3000 10.88.0.1 > /dev/null 2>&1 &
  printf "%-16s " "$N"
  sudo bpftool prog profile id $PROGID duration 3 cycles 2>/dev/null | \
    grep -oP '\d+ cycles' | head -1
  wait
done
```

**Map choice is a first-order decision.** §T.9(d), measured.

NUMA — §T.9(a):

```sh
numactl -H 2>/dev/null | head -5
for i in /sys/class/net/*/device/numa_node; do
  [ -f "$i" ] && echo "$(basename $(dirname $(dirname $i))): NUMA node $(cat $i)"
done
# A NIC on node 1 with the XDP program running on a node-0 CPU pays
# a remote-memory penalty on every packet.
```

Tracepoints:

```sh
sudo bpftrace -l 'tracepoint:xdp:*'
sudo bpftrace -e '
tracepoint:xdp:xdp_exception      { @exception[args->act] = count(); }
tracepoint:xdp:xdp_redirect       { @redirect = count(); }
tracepoint:xdp:xdp_redirect_err   { @redirect_err[args->err] = count(); }
tracepoint:xdp:xdp_devmap_xmit    { @xmit = hist(args->sent); @drops = sum(args->drops); }
tracepoint:xdp:xdp_cpumap_enqueue { @cpumap_enq = count(); }
interval:s:10 { print(@exception); print(@redirect); print(@redirect_err);
                print(@xmit); print(@drops); exit(); }' &
sudo ip netns exec a ping -f -c 5000 10.88.0.1 > /dev/null 2>&1
wait
```

---

### Lab 74.8 — xdp-tools and a realistic filter

```sh
git clone --depth 1 https://github.com/xdp-project/xdp-tools.git 2>/dev/null
cd xdp-tools && ./configure && make -j$(nproc) 2>&1 | tail -3
sudo make install 2>/dev/null

xdp-loader status
sudo xdp-loader load -m skb va-br xdp-filter/xdpfilt_alw_all.o 2>/dev/null
xdp-loader status
sudo xdp-loader unload va-br --all 2>/dev/null
```

`xdp-filter` — a production-quality filter:

```sh
sudo xdp-filter load va-br -m native -f ipv4,tcp,udp 2>/dev/null
sudo xdp-filter status

sudo xdp-filter ip 10.88.0.1 -m src 2>/dev/null
sudo ip netns exec a ping -c 3 -W 1 10.88.0.2 2>&1 | tail -2
sudo xdp-filter status
sudo xdp-filter ip 10.88.0.1 -m src -r 2>/dev/null

sudo xdp-filter port 8080 -m dst 2>/dev/null
sudo xdp-filter status
sudo xdp-filter unload va-br 2>/dev/null
```

`xdp-bench` — the benchmark harness:

```sh
sudo xdp-bench drop va-br -i 1 2>/dev/null | head -8 &
sudo ip netns exec a ping -f -c 10000 10.88.0.1 > /dev/null 2>&1
sleep 2; sudo kill %1 2>/dev/null

# The other modes:
# xdp-bench pass, tx, redirect, redirect-cpu, redirect-map
```

A complete DDoS filter combining everything:

```c
// SPDX-License-Identifier: GPL-2.0
/* xdp_ddos.c -- rate-limit per source, block a set, pass the rest. */
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <linux/tcp.h>
#include <linux/udp.h>
#include <linux/in.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>

#define RATE_WINDOW_NS	1000000000ULL      /* 1 second */
#define MAX_PPS		1000

struct rate_state {
	__u64 window_start;
	__u64 count;
};

struct {
	__uint(type, BPF_MAP_TYPE_LRU_HASH);
	__uint(max_entries, 1000000);
	__type(key, __u32);                  /* source IP */
	__type(value, struct rate_state);
} rate_limit SEC(".maps");

struct lpm_key { __u32 prefixlen; __u8 addr[4]; };

struct {
	__uint(type, BPF_MAP_TYPE_LPM_TRIE);
	__uint(max_entries, 10000);
	__type(key, struct lpm_key);
	__type(value, __u8);
	__uint(map_flags, BPF_F_NO_PREALLOC);
} blocklist SEC(".maps");

struct {
	__uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
	__uint(max_entries, 8);
	__type(key, __u32);
	__type(value, __u64);
} ddos_stats SEC(".maps");

#define S_TOTAL 0
#define S_BLOCKED 1
#define S_RATELIMITED 2
#define S_PASSED 3
#define S_MALFORMED 4

static __always_inline void bump(__u32 k)
{
	__u64 *v = bpf_map_lookup_elem(&ddos_stats, &k);

	if (v) *v += 1;
}

SEC("xdp")
int xdp_ddos_prog(struct xdp_md *ctx)
{
	void *data = (void *)(long)ctx->data;
	void *data_end = (void *)(long)ctx->data_end;
	struct ethhdr *eth = data;
	struct iphdr *iph;
	struct lpm_key lk;
	struct rate_state *rs, init = {};
	__u64 now;
	__u32 saddr;

	bump(S_TOTAL);

	if ((void *)(eth + 1) > data_end) { bump(S_MALFORMED); return XDP_DROP; }
	if (eth->h_proto != bpf_htons(ETH_P_IP)) { bump(S_PASSED); return XDP_PASS; }

	iph = (void *)(eth + 1);
	if ((void *)(iph + 1) > data_end) { bump(S_MALFORMED); return XDP_DROP; }
	if (iph->ihl < 5) { bump(S_MALFORMED); return XDP_DROP; }

	saddr = iph->saddr;

	/* 1. The block list: an LPM trie, so CIDRs work */
	lk.prefixlen = 32;
	__builtin_memcpy(lk.addr, &saddr, 4);
	if (bpf_map_lookup_elem(&blocklist, &lk)) {
		bump(S_BLOCKED);
		return XDP_DROP;
	}

	/* 2. Per-source rate limiting, in an LRU hash so it cannot be filled */
	now = bpf_ktime_get_ns();
	rs = bpf_map_lookup_elem(&rate_limit, &saddr);
	if (!rs) {
		init.window_start = now;
		init.count = 1;
		bpf_map_update_elem(&rate_limit, &saddr, &init, BPF_ANY);
	} else {
		if (now - rs->window_start > RATE_WINDOW_NS) {
			rs->window_start = now;
			rs->count = 1;
		} else {
			rs->count++;
			if (rs->count > MAX_PPS) {
				bump(S_RATELIMITED);
				return XDP_DROP;
			}
		}
	}

	bump(S_PASSED);
	return XDP_PASS;
}

char _license[] SEC("license") = "GPL";
```

```sh
cd ..
clang -O2 -g -target bpf -c xdp_ddos.c -o xdp_ddos.o
sudo ip link set dev va-br xdpdrv off 2>/dev/null
sudo ip link set dev va-br xdpdrv obj xdp_ddos.o sec xdp

# Block a /24
sudo bpftool map update name blocklist \
     key hex 18 00 00 00 0a 58 00 00 value 1 2>/dev/null
sudo bpftool map dump name blocklist

echo "=== normal traffic ==="
sudo ip netns exec a ping -c 3 10.88.0.2 > /dev/null 2>&1
sudo bpftool map dump name ddos_stats

echo "=== flood (should be rate limited) ==="
sudo ip netns exec a ping -f -c 5000 10.88.0.2 > /dev/null 2>&1
sudo bpftool map dump name ddos_stats

sudo bpftool map show
sudo bpftool prog show | grep -A3 xdp_ddos
```

Cleanup:

```sh
sudo ip link set dev va-br xdpdrv off 2>/dev/null
sudo ip link set dev vb-br xdpdrv off 2>/dev/null
sudo tc qdisc del dev va-br clsact 2>/dev/null
sudo ip netns del a b 2>/dev/null
sudo ip link del va-br 2>/dev/null
sudo ip link del vb-br 2>/dev/null
```

---

## 3. Mastery drills

1. Enumerate the six costs of kernel bypass from §T.1. For each, state whether XDP pays it, avoids it, or partially pays it.

2. Explain why the verifier requires a bounds check before every packet access. Construct the kernel memory-safety bug this prevents, and state what a kernel module would have had to do instead.

3. Rank the five XDP actions by cost, and for each explain what work it avoids relative to `XDP_PASS`.

4. Generic mode is "not a performance mode." Enumerate exactly what has already happened by the time a generic-mode program runs, and compute the fraction of XDP's benefit it forfeits.

5. `bpf_redirect_map` requires a map. Explain why redirect was designed with this indirection rather than taking an ifindex directly, and name three capabilities it enables.

6. CPUMAP steers to a CPU. State three things it can do that RSS cannot, and explain why the skb is built on the destination CPU.

7. Draw AF_XDP's four rings and trace a packet through receive and transmit, naming who owns the chunk at each step. Then state what happens if the application stops refilling.

8. `XDP_USE_NEED_WAKEUP` eliminates syscalls. Explain the mechanism, and construct the traffic pattern where it saves nothing.

9. `smp_store_release`/`smp_load_acquire` synchronise the AF_XDP rings. Write the torn-read bug that occurs if an application uses plain loads and stores, and say why the compiler alone cannot prevent it.

10. XDP hints are kfuncs rather than a fixed metadata structure. Argue both designs, and state the general principle that decided it.

11. Compare XDP and DPDK on §T.8's ten dimensions. Then construct the deployment where DPDK is still correct, and the one where XDP clearly wins.

12. §T.10 lists seven situations where XDP is the wrong answer. For each, name the right mechanism and state what XDP would cost you.

13. You have an XDP program achieving 3 Mpps where you expected 20 Mpps. Give the ordered diagnostic procedure and the seven most likely causes, in order.

---

## 4. Further reading

**Kernel documentation**

- `Documentation/networking/af_xdp.rst` ★★★ — **§T.6, completely.** The ring protocol, the UMEM, the modes, and the performance considerations. Read it before writing any AF_XDP code.
- `Documentation/networking/xdp-rx-metadata.rst` ★★★ — §T.7.
- `Documentation/bpf/` ★★★ — particularly `verifier.rst`, `map_*.rst`, `prog_*.rst` (Ch. 75's material).
- `Documentation/networking/filter.rst` — classic BPF, the ancestry.
- `Documentation/networking/napi.rst` — Ch. 71, but XDP lives inside NAPI.
- `samples/bpf/xdp*_user.c` and `xdp*_kern.c` ★★★ — many worked examples; `xdp_redirect_map`, `xdp_router_ipv4`, `xdpsock` are the most instructive.
- `tools/testing/selftests/bpf/prog_tests/xdp*.c` — the tests, which double as specification.

**Papers**

- Høiland-Jørgensen, Brouer, Borkmann, Fastabend, Herbert, Ahern, Miller, "The eXpress Data Path: Fast Programmable Packet Processing in the Operating System Kernel," CoNEXT 2018 ★★★ — **the XDP paper.** The design, the rationale, and the measurements. Read it.
- Rizzo, "netmap: a novel framework for fast packet I/O," USENIX ATC 2012 ★★★ — the kernel-bypass argument that XDP responds to.
- Karlsson & Töpel, "The Path to DPDK Speeds for AF_XDP," Linux Plumbers 2018 ★★★ — §T.6's design and its performance journey.
- Vieira et al., "Fast Packet Processing with eBPF and XDP: Concepts, Code, Challenges, and Applications," ACM Computing Surveys 2020 — a thorough survey.
- Eran et al., "NICA: An Infrastructure for Inline Acceleration of Network Applications," USENIX ATC 2019 — the offload direction.
- Miano et al., "Creating Complex Network Services with eBPF: Experience and Lessons Learned," HPSR 2018 — what is hard in practice.
- Katran's design posts from Meta engineering ★★★ — a real XDP load balancer, including the Maglev consistent-hashing scheme this chapter's lab simplifies.

**Project documentation**

- **xdp-project.net** and the **xdp-tools** repository ★★★ — `xdp-filter`, `xdp-bench`, `xdp-loader`, `xdp-trafficgen`, and `libxdp`. The best practical resource.
- **The XDP tutorial** (`github.com/xdp-project/xdp-tutorial`) ★★★ — **the best way to learn XDP hands-on.** Progressive, well-commented, and maintained. Work through it.
- **ebpf.io** ★★★ — the broader eBPF documentation (Ch. 75).
- **Cilium's documentation** ★★★ — particularly the "eBPF Datapath" section, which explains a production XDP+tc architecture in detail.
- **BPF and XDP Reference Guide** (Cilium docs) ★★★ — the most complete practical reference for the BPF side.

**LWN**

- "The rapidly-changing world of XDP" ★★★
- "Accelerating networking with AF_XDP" ★★★
- "XDP: a new fast path for packet processing"
- "Zero-copy networking" and the AF_XDP zero-copy coverage
- "XDP metadata and hardware hints"
- "Multi-buffer XDP"
- "BPF at Facebook" and "BPF at Netflix" — production experience reports
- "A thorough introduction to eBPF" ★★★
- The LSFMM and netdev conference coverage ★★★ — XDP is designed at netdev

**Source reading order**

1. The XDP paper (Høiland-Jørgensen 2018) and the XDP tutorial. **Do not start with the code.**
2. `include/net/xdp.h` and `include/uapi/linux/bpf.h`'s `xdp_md` / `xdp_action` ★★★
3. `drivers/net/veth.c`: `veth_xdp_rcv_skb`, `veth_xdp_rcv_one` ★★★ — the clearest driver implementation.
4. `net/core/filter.c`: `bpf_xdp_redirect_map`, `xdp_do_redirect`, and the XDP helper definitions ★★★
5. `kernel/bpf/devmap.c`: `dev_map_enqueue`, `bq_xmit_all` ★★★ — §T.5's batching.
6. `kernel/bpf/cpumap.c`: `cpu_map_kthread_run` ★★★ — §T.5's CPU steering.
7. `include/uapi/linux/if_xdp.h` ★★★ — the AF_XDP ABI; short and complete.
8. `net/xdp/xsk_queue.h` ★★★ — **the ring implementation.** ~400 lines, and the memory ordering is the lesson.
9. `net/xdp/xsk.c`: `xsk_rcv_zc`, `xsk_generic_rcv`, `xsk_bind`.
10. `drivers/net/ethernet/intel/i40e/i40e_xsk.c` — zero-copy in a real driver.
11. `samples/bpf/xdpsock_user.c` — the reference AF_XDP application.

**Tools**

- **xdp-tools** ★★★ — `xdp-loader`, `xdp-filter`, `xdp-bench`, `xdp-trafficgen`, `libxdp`
- `bpftool` ★★★ — `prog show`, `prog profile` ★★★, `map dump`, `net show`, `prog tracelog`
- `clang -target bpf` and `llvm-objdump -d` ★★★ — read the generated bytecode
- `ip link set dev X xdpdrv/xdpgeneric/xdpoffload` ★★★
- `bpftrace` on `tracepoint:xdp:*` ★★★ — **monitor `xdp_exception` in production**
- `ethtool -S` for driver XDP counters ★★★
- `pktgen` (`samples/pktgen/`) and `xdp-trafficgen` ★★★ — load generation
- `TRex`, `MoonGen` — for serious packet generation
- `perf record -e cycles:k -a` with the BPF program's symbol
- `veth` + network namespaces ★★★ — **every lab in this chapter runs without hardware**
- `libxdp` / `libbpf` ★★★ — the userspace libraries; use them rather than raw syscalls

---

→ Next: [75-ebpf.md](75-ebpf.md)

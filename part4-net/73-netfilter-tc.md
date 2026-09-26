# Chapter 73 — Netfilter, nftables, conntrack, and traffic control

> **Goal:** Understand the two systems that decide what happens to a packet beyond routing: netfilter, which filters and transforms, and traffic control, which shapes and schedules. Understand the five hooks and why they sit where they do, the table/chain/priority model and how iptables and nftables both map onto it, connection tracking as the stateful layer that makes NAT and stateful filtering possible and its scaling limits, NAT as a conntrack consumer, nftables' bytecode VM and why it replaced iptables' linear rule matching, the qdisc hierarchy with classful and classless disciplines, the bufferbloat problem and CoDel/FQ-CoDel/CAKE as its solution, and tc's filter/action model including eBPF. By the end you can read `net/netfilter/` and `net/sched/`, write nftables and tc rules with intent, and diagnose why a packet did not arrive.

---

## Theory & First Principles

### T.0 — Start here: one rule, and everything it implies

```bash
sudo nft add rule inet filter input tcp dport 22 accept
sudo tc qdisc add dev eth0 root fq_codel
```

Two commands, two completely different subsystems, and people conflate them constantly.
Separate them first, because the distinction is the chapter:

| | **netfilter / nftables** | **tc (traffic control)** |
|---|---|---|
| Question | **"should this packet exist?"** | **"when does this packet go out?"** |
| Verbs | accept, drop, reject, NAT, mark, log | queue, delay, drop-to-signal, prioritize, shape |
| Operates on | individual packets and **connections** | **queues** of packets |
| Cares about | identity, policy, security | time, rate, fairness, latency |

**Policy versus scheduling.** Ch. 00 §T.1's mechanism/policy split shows up here as two
entirely separate subsystems, deliberately not merged, because "is this allowed" and "in what
order" are independent questions that compose.

**Start with netfilter's structure**, which is five hook points in the packet's path:

```
                          +-> LOCAL_IN -> [local process] -> LOCAL_OUT -+
                          |                                            |
   NIC -> PRE_ROUTING ->[routing decision]                       [routing]
                          |                                            |
                          +-> FORWARD --------------------------> POST_ROUTING -> NIC
```

**The hooks are placed where they are because of the routing decision, not arbitrarily.**
PRE_ROUTING is the only place you can DNAT (you must change the destination *before* the
kernel decides where to send it); POST_ROUTING is the only place you can SNAT (you must
change the source *after* the kernel has picked an outgoing interface). **The interception
points are dictated by the data dependencies of the thing being intercepted** — a completely
general principle for designing any hook system (compare Ch. 22 §T.0: the page fault is where
it is because that is the only point at which the information exists).

**Second: connection tracking is what makes a firewall usable, and it is expensive.**

```
  stateless:  "allow TCP dport 80 out" AND "allow TCP sport 80 in"
              -> anything from source port 80 gets in. Trivially bypassed.

  stateful:   "allow NEW out; allow ESTABLISHED,RELATED in"
              -> conntrack remembers the flow. Only real replies get in.
```

The cost is a hash table entry per flow, with a fixed maximum. **`nf_conntrack: table full,
dropping packet` is one of the most common production networking failures**, and the
interesting thing about it is that it is the *stateful* design's inherent failure mode:
keeping state means having a finite amount of it, means having an exhaustion attack, means
needing eviction policy and timeouts. Ch. 62 §T.0's stateless-vs-stateful trade again,
exactly.

**Third: the single best thing in `tc` is `codel`, and the idea generalizes far beyond
networking.** Bufferbloat is the observation that adding *more* buffer makes latency worse:

```
  a 1 GB buffer on a 1 Mbit link = 8000 seconds of queued data.
  TCP only learns to slow down when packets are DROPPED.
  A huge buffer therefore DELAYS the congestion signal instead of sending it.
  -> throughput unchanged, latency catastrophic.
```

`codel`'s fix is not a queue-length limit but a **sojourn-time** limit: measure how long each
packet *waited*, and if the minimum wait over an interval exceeds 5 ms, start dropping.

> **Measure the thing you actually care about (delay), not a proxy for it (queue length).**

A queue-length threshold is wrong because the right length depends on the link rate, which
varies. Time is the invariant. **This is one of the most transferable ideas in the book** —
the same reasoning applies to thread pools, request queues, and any admission control system
you will ever build.

```bash
sudo nft list ruleset
sudo conntrack -S && sysctl net.netfilter.nf_conntrack_max
tc -s qdisc show dev eth0        # backlog, drops, ce_marks
sudo tc qdisc replace dev eth0 root netem delay 100ms loss 1%   # emulate a bad link
```

---

### T.1 Two orthogonal systems

They are frequently confused, and separating them is the first step:

| | **netfilter** | **traffic control (tc)** |
|---|---|---|
| Question | *whether* and *how modified* | *when* and *at what rate* |
| Operations | accept, drop, NAT, mark, log, queue to userspace | queue, shape, prioritise, drop for congestion |
| Placement | five hooks around the routing decision | at the device's transmit (and ingress) point |
| Statefulness | conntrack gives it flow state | mostly flow-agnostic, with per-flow queueing exceptions |
| Userspace tool | `iptables`/`nftables` | `tc` |
| Kernel | `net/netfilter/` | `net/sched/` |

They interact through `skb->mark` (netfilter sets it, tc filters on it) and `skb->priority`, and both can call eBPF. But conceptually: **netfilter decides the packet's fate; tc decides its timing.**

### T.2 The five hooks

Netfilter's mechanism is minimal: five points in the packet path where registered functions run.

```
                             ┌──────────────┐
                             │   LOCAL_IN   │──► local socket
                             └──────▲───────┘
                                    │
   ┌───────────┐   ┌────────┐   ┌───┴────┐   ┌─────────┐   ┌─────────────┐
──►│PRE_ROUTING│──►│ routing│──►│FORWARD │──►│ routing │──►│POST_ROUTING │──►
   └───────────┘   │decision│   └────────┘   │ decision│   └──────▲──────┘
                   └────────┘                └─────────┘          │
                                             ┌───────────┐        │
                        local socket ───────►│LOCAL_OUT  │────────┘
                                             └───────────┘
```

```c
enum nf_inet_hooks {
	NF_INET_PRE_ROUTING,   /* every incoming packet, before routing */
	NF_INET_LOCAL_IN,      /* destined for this host */
	NF_INET_FORWARD,       /* being routed through */
	NF_INET_LOCAL_OUT,     /* generated locally */
	NF_INET_POST_ROUTING,  /* every outgoing packet, after routing */
	NF_INET_NUMHOOKS
};
```

**The placement relative to routing is the whole design.** PRE_ROUTING runs before the routing decision, so DNAT there changes where the packet is routed. POST_ROUTING runs after, so SNAT there does not disturb routing. That is why DNAT must be in PRE_ROUTING (or LOCAL_OUT) and SNAT in POST_ROUTING — not convention, but necessity.

Verdicts:

```c
#define NF_DROP   0   /* free the packet; no reply */
#define NF_ACCEPT 1   /* continue to the next hook function */
#define NF_STOLEN 2   /* I took it; do not free it, do not continue */
#define NF_QUEUE  3   /* hand to userspace via nfnetlink_queue */
#define NF_REPEAT 4   /* call this hook again */
#define NF_STOP   5   /* deprecated */
```

`NF_STOLEN` is how `NFQUEUE` and some tunnel implementations work: the packet leaves the pipeline entirely and its fate becomes someone else's problem.

**Priorities** order the functions at each hook, and the numbers encode the architecture:

```c
enum nf_ip_hook_priorities {
	NF_IP_PRI_RAW_BEFORE_DEFRAG = -450,
	NF_IP_PRI_CONNTRACK_DEFRAG = -400,
	NF_IP_PRI_RAW = -300,
	NF_IP_PRI_SELINUX_FIRST = -225,
	NF_IP_PRI_CONNTRACK = -200,          /* conntrack runs EARLY */
	NF_IP_PRI_MANGLE = -150,
	NF_IP_PRI_NAT_DST = -100,            /* DNAT */
	NF_IP_PRI_FILTER = 0,                /* the filter table */
	NF_IP_PRI_SECURITY = 50,
	NF_IP_PRI_NAT_SRC = 100,             /* SNAT */
	NF_IP_PRI_SELINUX_LAST = 225,
	NF_IP_PRI_CONNTRACK_HELPER = 300,
	NF_IP_PRI_CONNTRACK_CONFIRM = INT_MAX,   /* conntrack commits LAST */
};
```

Reading this list tells you the order of everything: defrag, then conntrack lookup, then mangle, then DNAT, then filter, then SNAT, then helpers, then conntrack confirmation. **The iptables "table" abstraction is nothing more than a name for a priority**, which is why nftables discarded it.

### T.3 Connection tracking

Stateful filtering ("allow replies to connections I started") and NAT both require the kernel to remember flows. Conntrack does.

```c
struct nf_conn {
	struct nf_conntrack ct_general;
	spinlock_t lock;
	u32 timeout;
	struct nf_conntrack_zone zone;
	struct nf_conntrack_tuple_hash tuplehash[IP_CT_DIR_MAX];  /* BOTH directions */
	unsigned long status;
	possible_net_t ct_net;
	struct hlist_node nat_bysource;
	struct nf_conn *master;          /* for expectations (FTP data, etc.) */
	u_int32_t mark, secmark;
	struct nf_ct_ext *ext;           /* NAT, helper, timeout, accounting */
	union nf_conntrack_proto proto;  /* TCP state, or other protocol state */
};

struct nf_conntrack_tuple {
	struct nf_conntrack_man src;
	struct {
		union nf_inet_addr u3;
		union { __be16 all; struct { __be16 port; } tcp, udp; ... } u;
		u_int8_t protonum;
		u_int8_t dir;
	} dst;
};
```

**Two tuples per connection** — original and reply — is the key design. The reply tuple is what NAT rewrites: if a packet is SNATted from 10.0.0.1:1234 to 203.0.113.1:5678, the reply tuple becomes "from the server to 203.0.113.1:5678", so the reply matches and is un-NATted automatically.

The packet's relation to a connection:

```c
enum ip_conntrack_info {
	IP_CT_ESTABLISHED,           /* in the original direction */
	IP_CT_RELATED,               /* related to an existing one (ICMP error, FTP data) */
	IP_CT_NEW,                   /* starts a new connection */
	IP_CT_IS_REPLY,              /* +3 for reply-direction variants */
	IP_CT_ESTABLISHED_REPLY = IP_CT_ESTABLISHED + IP_CT_IS_REPLY,
	IP_CT_RELATED_REPLY = IP_CT_RELATED + IP_CT_IS_REPLY,
	IP_CT_NUMBER,
};
```

`ct state` in nftables (`ctstate` in iptables) matches on this, and "accept established,related" is the foundation of every stateful ruleset.

Three things about conntrack worth internalising:

**(a) TCP state tracking is independent of the endpoints'.** Conntrack maintains its own TCP state machine with its own timeouts:

```sh
net.netfilter.nf_conntrack_tcp_timeout_established = 432000   # 5 DAYS
net.netfilter.nf_conntrack_tcp_timeout_syn_sent = 120
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 120
net.netfilter.nf_conntrack_tcp_timeout_close_wait = 60
net.netfilter.nf_conntrack_udp_timeout = 30
net.netfilter.nf_conntrack_udp_timeout_stream = 120
```

The 5-day established timeout is a frequent operational surprise: a firewall or NAT that drops entries earlier than the endpoints expect causes connections to break silently, which is why TCP keepalives exist and why `tcp_keepalive_time` should be *below* the NAT's timeout.

**(b) The table has a hard limit.**

```sh
net.netfilter.nf_conntrack_max = 262144
net.netfilter.nf_conntrack_buckets = 65536
```

When full, new connections are dropped and `nf_conntrack: table full, dropping packet` appears in the log. On a busy server or a NAT gateway this is a real limit, and it is the main argument for `notrack` on high-volume flows that do not need state.

**(c) The confirmation race.** A conntrack entry is created on the first packet but **not inserted into the hash table until the packet successfully leaves** (`NF_IP_PRI_CONNTRACK_CONFIRM` at POST_ROUTING). This means a dropped packet leaves no entry, and it means two simultaneous packets of the same new flow can both create entries, with one losing the race and being dropped (`insert_failed` in the statistics).

**Helpers** handle protocols that embed addresses in their payload (FTP's PORT command, SIP, TFTP). A helper parses the payload, creates an *expectation*, and a matching future connection becomes `RELATED`. Helpers are also a security liability — an attacker who can control payload content can create expectations for arbitrary ports — which is why automatic helper assignment was disabled by default (`nf_conntrack_helper=0`) and helpers must now be assigned explicitly by rule.

### T.4 NAT

NAT is implemented **on top of** conntrack, not alongside it. The rules:

| Type | Hook | Changes |
|---|---|---|
| **SNAT / MASQUERADE** | POST_ROUTING | source address (and port) |
| **DNAT / REDIRECT** | PRE_ROUTING or LOCAL_OUT | destination address (and port) |

**NAT is decided once per connection.** The first packet traverses the NAT rules; the resulting mapping is stored in the conntrack entry's reply tuple; every subsequent packet is translated by lookup, not by rule evaluation. This is why:

- NAT rules only ever match `ct state new`.
- Changing a NAT rule does not affect existing connections.
- NAT is cheap after the first packet.

`MASQUERADE` versus `SNAT`: the former picks the source address from the outgoing interface at packet time (correct for dynamic addresses, slightly more expensive, and it flushes mappings when the interface goes down); the latter uses a fixed address. Use `SNAT` when the address is static.

**Port allocation** is where NAT gets interesting at scale. `nf_nat_l4proto_unique_tuple()` must find a free source port, and with many connections to the same destination this becomes a search. The `--random-fully` option (and nftables' `fully-random`) randomises allocation to avoid the birthday-collision problem that caused observable connection failures at scale in container environments — a real and well-documented issue in Kubernetes.

The hard limit: **one NAT source address supports at most ~64K concurrent connections per destination 4-tuple**. Beyond that you need more addresses. This is the arithmetic behind "our NAT gateway is dropping connections" in any large deployment.

### T.5 iptables versus nftables

iptables' problems, structurally:

| Problem | Detail |
|---|---|
| **Linear evaluation** | a rule set of N rules costs O(N) per packet |
| **Separate binaries** | `iptables`, `ip6tables`, `arptables`, `ebtables` — four implementations of the same idea |
| **Atomic replacement only** | adding one rule rewrites the whole table |
| **Fixed match structure** | each match is a kernel module with a fixed binary layout |
| **No sets** | matching 1000 addresses means 1000 rules |
| **Table/chain rigidity** | tables are hardcoded, hooks are implicit |

nftables replaces the matching engine with a **small bytecode VM**:

```c
struct nft_expr_ops {
	void (*eval)(const struct nft_expr *expr, struct nft_regs *regs,
		     const struct nft_pktinfo *pkt);
	...
};

struct nft_regs {
	union {
		u32			data[20];
		struct nft_verdict	verdict;
	};
};
```

A rule is a sequence of expressions operating on registers. `nft add rule ip filter input tcp dport 22 accept` becomes:

```
[ meta load l4proto => reg 1 ]
[ cmp eq reg 1 0x00000006 ]          # TCP
[ payload load 2b @ transport header + 2 => reg 1 ]
[ cmp eq reg 1 0x00001600 ]          # port 22
[ immediate reg 0 accept ]
```

What this buys:

**(a) Sets and maps, with O(1) or O(log n) lookup.**

```
nft add set ip filter blocked { type ipv4_addr\; flags interval\; }
nft add element ip filter blocked { 192.0.2.0/24, 198.51.100.7 }
nft add rule ip filter input ip saddr @blocked drop
```

One rule, any number of addresses. Set implementations are chosen by the kernel: `nft_set_hash` (exact match), `nft_set_rbtree` (intervals), `nft_set_pipapo` (the "PIle PAcket POlicies" algorithm, for concatenated intervals — genuinely clever, based on a cross-product bitmap technique).

**Maps** go further — a set that yields a value:

```
nft add rule ip nat postrouting snat to ip saddr map { 10.0.1.0/24 : 203.0.113.1, \
                                                       10.0.2.0/24 : 203.0.113.2 }
```

One rule replacing a chain of per-subnet rules, with a single lookup.

**(b) Atomic, incremental updates** via a netlink transaction. Add a rule without rewriting the table; add a batch of changes that all apply or none do.

**(c) One implementation for all families**: `ip`, `ip6`, `inet` (both), `arp`, `bridge`, `netdev`.

**(d) Verdict maps** turn a chain of comparisons into one lookup:

```
nft add rule ip filter input meta l4proto vmap { tcp : jump tcp-chain, \
                                                  udp : jump udp-chain }
```

The `inet` family deserves emphasis: one rule set covering IPv4 and IPv6, written once. Maintaining parallel `iptables` and `ip6tables` rule sets is a well-known source of security holes where the IPv6 set was forgotten.

`iptables-nft` provides the iptables command syntax over the nftables kernel backend, which is how distributions migrated without breaking scripts. It is a translation layer, and mixing `iptables-legacy` and `iptables-nft` on one system produces confusing results — a common operational trap.

### T.6 Traffic control: the qdisc model

Every network device has a **root qdisc**. Packets are enqueued into it and dequeued when the device can transmit.

```c
struct Qdisc_ops {
	struct Qdisc_ops	*next;
	const struct Qdisc_class_ops *cl_ops;
	char			id[IFNAMSIZ];
	int			priv_size;
	unsigned int		static_flags;

	int (*enqueue)(struct sk_buff *skb, struct Qdisc *sch,
		       struct sk_buff **to_free);
	struct sk_buff *(*dequeue)(struct Qdisc *);
	struct sk_buff *(*peek)(struct Qdisc *);

	int (*init)(struct Qdisc *sch, struct nlattr *arg,
		    struct netlink_ext_ack *extack);
	void (*reset)(struct Qdisc *);
	void (*destroy)(struct Qdisc *);
	int (*change)(struct Qdisc *, struct nlattr *arg,
		      struct netlink_ext_ack *extack);
	...
};
```

Two kinds:

| | **Classless** | **Classful** |
|---|---|---|
| Structure | one queue | a tree of classes, each with its own qdisc |
| Configuration | parameters only | classes + filters to assign packets |
| Examples | `pfifo_fast`, `fq_codel`, `fq`, `sfq`, `codel`, `netem` | `htb`, `hfsc`, `cbq`, `prio`, `drr`, `qfq` |

Classful qdiscs use **filters** to classify packets into classes:

```
                    root qdisc (htb)
                          |
                    class 1:1 (rate 100mbit)
                    /           \
          class 1:10           class 1:20
          (rate 60mbit)        (rate 40mbit)
              |                     |
          fq_codel              fq_codel
```

```sh
tc qdisc add dev eth0 root handle 1: htb default 20
tc class add dev eth0 parent 1: classid 1:1 htb rate 100mbit
tc class add dev eth0 parent 1:1 classid 1:10 htb rate 60mbit ceil 100mbit
tc class add dev eth0 parent 1:1 classid 1:20 htb rate 40mbit ceil 100mbit
tc qdisc add dev eth0 parent 1:10 handle 10: fq_codel
tc qdisc add dev eth0 parent 1:20 handle 20: fq_codel
tc filter add dev eth0 parent 1: protocol ip prio 1 u32 \
   match ip dport 22 0xffff flowid 1:10
```

**HTB's `rate` versus `ceil`** is the mechanism that makes hierarchies useful: a class is *guaranteed* `rate` and may *borrow* up to `ceil` from its parent when the parent has spare capacity. This gives both isolation (you always get your rate) and efficiency (unused capacity is not wasted).

`handle` and `classid` notation is `major:minor`, where qdiscs have minor 0 and classes have non-zero minors. `1:` means qdisc 1; `1:10` means class 10 under qdisc 1.

### T.7 Bufferbloat, and the AQM answer

**The problem.** A router with a large buffer does not drop packets under congestion; it queues them. TCP's loss-based congestion control (Ch. 72 §T.5) therefore keeps increasing its window until the buffer *does* overflow — which can be hundreds of milliseconds later. Result: a fully-utilised link with a full buffer and a round-trip time inflated by the buffer's drain time.

The consequence is familiar: a large download makes video calls unusable, DNS queries take seconds, and ping times go from 20 ms to 2000 ms. Jim Gettys named it **bufferbloat** in 2010.

Why the obvious fixes fail:

| Fix | Why it fails |
|---|---|
| Smaller buffers | you then drop during legitimate bursts and lose throughput |
| RED (Random Early Detection) | requires tuning against the link rate and traffic mix; nobody tunes it correctly |
| Just use a faster link | the buffer scales too |

**CoDel** (Controlled Delay, Nichols & Jacobson 2012) is the answer, and its design is worth understanding because it is parameterless in the ways that matter:

```
Track the MINIMUM queue delay over a sliding interval (100 ms).
If the minimum delay exceeds target (5 ms) for a full interval:
    enter dropping state; drop one packet
    schedule the next drop at interval/sqrt(count)  -- drop faster if it persists
If the delay falls below target: exit dropping state
```

The insight: **the minimum delay over an interval distinguishes a standing queue from a burst.** A burst raises the delay briefly but the minimum returns to near zero; a standing queue keeps even the minimum high. So CoDel drops only when there is persistent queueing — exactly the condition that needs a congestion signal — and never punishes bursts.

`target` (5 ms) and `interval` (100 ms) are not link-rate-dependent, which is why CoDel works without tuning where RED did not.

**FQ-CoDel** adds fair queueing: hash flows into buckets, run CoDel on each, and dequeue round-robin with priority for new flows. This gives:

- A bulk transfer cannot delay an interactive flow — they are in different buckets.
- A new flow (the first packets of a DNS query or a TCP handshake) gets immediate service.
- Each flow gets its own CoDel instance, so each sees the right signal.

**FQ-CoDel is the default qdisc on modern Linux** (`net.core.default_qdisc`), and it is close to free: the fairness comes from hashing, not from per-flow state.

**CAKE** (Common Applications Kept Enhanced) goes further for the home-router case: integrated shaping (so you do not need HTB separately), per-host fairness in addition to per-flow (so one host with 50 connections does not dominate), DiffServ handling, ACK filtering, and overhead compensation for DSL/cable encapsulation. For a home gateway it is a single-line configuration replacing an elaborate HTB tree.

**`fq`** (Fair Queue) is different again: designed for hosts rather than routers, it provides per-flow queueing *and pacing*, which is what BBR (Ch. 72 §T.5) requires. On a server, `fq` is usually the right choice; on a router, `fq_codel` or `cake`.

### T.8 tc filters and actions

The classification model is `filter → action`:

```c
struct tcf_proto_ops {
	struct list_head	head;
	char			kind[IFNAMSIZ];
	int (*classify)(struct sk_buff *, const struct tcf_proto *,
			struct tcf_result *);
	int (*init)(struct tcf_proto *);
	void (*destroy)(struct tcf_proto *, bool, struct netlink_ext_ack *);
	void *(*get)(struct tcf_proto *, u32 handle);
	int (*change)(struct net *net, struct sk_buff *, struct tcf_proto *, ...);
	int (*delete)(struct tcf_proto *, void *, bool *, bool, ...);
	...
};
```

Filter types:

| Filter | Matches |
|---|---|
| `u32` | arbitrary bits at arbitrary offsets — powerful, unreadable |
| `flower` ★★★ | structured L2–L4 fields; **hardware-offloadable** |
| `bpf` | an eBPF program |
| `matchall` | everything |
| `fw` | `skb->mark` (set by netfilter) |
| `route`, `basic`, `cgroup`, `flow` | various |

`flower` deserves emphasis: because it expresses matches as structured fields rather than bit offsets, it can be **translated to hardware**. `tc filter add ... skip_sw` offloads the rule to the NIC, and a modern SmartNIC can do millions of flows in hardware. This is the mechanism behind Open vSwitch's hardware offload path.

Actions:

| Action | Effect |
|---|---|
| `drop`, `pass`, `pipe`, `continue`, `reclassify` | control flow |
| `mirred` | **mirror or redirect** to another interface |
| `police` | rate limit, with a token bucket |
| `pedit` | edit packet bytes |
| `nat` | stateless NAT |
| `vlan`, `mpls`, `tunnel_key` | encapsulation |
| `bpf` | run an eBPF program |
| `ct` | connection tracking from tc |
| `gact` | generic (drop/pass with probability) |

**`clsact`** is the modern attachment point, and it matters:

```sh
tc qdisc add dev eth0 clsact
tc filter add dev eth0 ingress bpf da obj prog.o sec ingress
tc filter add dev eth0 egress  bpf da obj prog.o sec egress
```

`clsact` is a pseudo-qdisc providing hooks at both ingress and egress without queueing anything. It replaced `ingress` (which had only one direction) and is where eBPF programs attach for packet processing that needs more than XDP's raw-page view — because at this point the skb exists and the full metadata is available.

**Ingress policing versus shaping**: you cannot *shape* ingress traffic — the packets have already consumed the link. You can only drop them (policing) and hope the sender backs off. `ifb` (Intermediate Functional Block) is the workaround: redirect ingress traffic to a virtual device and shape it there, which works but adds a queue and a redirect. For genuine ingress control, shape at the sender or use ECN.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `net/netfilter/core.c` ★★★ | §T.2's hook registration and traversal |
| `net/netfilter/nf_conntrack_core.c` ★★★ | §T.3 |
| `net/netfilter/nf_conntrack_proto_tcp.c` ★★★ | conntrack's TCP state machine |
| `net/netfilter/nf_nat_core.c` ★★★ | §T.4 |
| `net/netfilter/nf_tables_api.c` ★★★ | §T.5's netlink API and transactions |
| `net/netfilter/nf_tables_core.c` ★★★ | **the bytecode VM** |
| `net/netfilter/nft_*.c` | the expressions: `nft_cmp`, `nft_payload`, `nft_meta`, `nft_lookup` |
| `net/netfilter/nft_set_hash.c`, `nft_set_rbtree.c`, `nft_set_pipapo.c` ★★★ | set implementations |
| `net/ipv4/netfilter/ip_tables.c` | the legacy engine |
| `net/netfilter/xt_*.c` | iptables matches and targets |
| `net/sched/sch_api.c` ★★★ | §T.6's qdisc framework |
| `net/sched/sch_generic.c` ★★★ | `__qdisc_run`, `sch_direct_xmit` |
| `net/sched/sch_fq_codel.c` ★★★ | §T.7 |
| `net/sched/sch_codel.c`, `sch_cake.c`, `sch_fq.c` ★★★ | |
| `net/sched/sch_htb.c` ★★★ | the classful reference |
| `net/sched/sch_netem.c` | the test tool |
| `net/sched/cls_flower.c` ★★★, `cls_bpf.c`, `cls_u32.c` | §T.8 |
| `net/sched/act_*.c` | actions |
| `include/net/netfilter/nf_conntrack.h`, `include/net/sch_generic.h` ★★★ | |
| `Documentation/networking/nf_conntrack-sysctl.rst` ★★★ | |

### 1.2 Hook traversal

```c
int nf_hook_slow(struct sk_buff *skb, struct nf_hook_state *state,
		 const struct nf_hook_entries *e, unsigned int s)
{
	unsigned int verdict;
	int ret;

	for (; s < e->num_hook_entries; s++) {
		verdict = nf_hook_entry_hookfn(&e->hooks[s], skb, state);
		switch (verdict & NF_VERDICT_MASK) {
		case NF_ACCEPT:
			break;                      /* continue to the next */
		case NF_DROP:
			kfree_skb_reason(skb,
				SKB_DROP_REASON_NETFILTER_DROP);
			ret = NF_DROP_GETERR(verdict);
			if (ret == 0)
				ret = -EPERM;
			return ret;
		case NF_QUEUE:
			ret = nf_queue(skb, state, s, verdict);
			if (ret == 1)
				continue;
			return ret;
		case NF_STOLEN:
			return NF_STOLEN;           /* someone else owns it now */
		default:
			WARN_ON_ONCE(1);
			return 0;
		}
	}
	return 1;
}
```

And the fast path, which matters because most packets traverse hooks with no rules:

```c
static inline int nf_hook(u_int8_t pf, unsigned int hook, struct net *net,
			  struct sock *sk, struct sk_buff *skb,
			  struct net_device *indev, struct net_device *outdev,
			  int (*okfn)(struct net *, struct sock *, struct sk_buff *))
{
	struct nf_hook_entries *hook_head = NULL;
	int ret = 1;

	/* A static key: ZERO cost when netfilter is not in use. */
	if (__builtin_constant_p(pf) &&
	    __builtin_constant_p(hook) &&
	    !static_key_false(&nf_hooks_needed[pf][hook]))
		return 1;
	...
	rcu_read_lock();
	switch (pf) {
	case NFPROTO_IPV4:
		hook_head = rcu_dereference(net->nf.hooks_ipv4[hook]);
		break;
	...
	}

	if (hook_head) {
		struct nf_hook_state state;

		nf_hook_state_init(&state, hook, pf, indev, outdev, sk, net, okfn);
		ret = nf_hook_slow(skb, &state, hook_head, 0);
	}
	rcu_read_unlock();
	return ret;
}
```

`static_key_false` (Ch. 13) means a kernel with netfilter compiled in but no rules loaded pays **nothing** — the branch is patched out. That is why "netfilter costs performance even when unused" is false on modern kernels.

### 1.3 Conntrack lookup and confirmation

```c
unsigned int
nf_conntrack_in(struct sk_buff *skb, const struct nf_hook_state *state)
{
	enum ip_conntrack_info ctinfo;
	struct nf_conn *ct, *tmpl;
	u_int8_t protonum;
	int dataoff, ret;

	tmpl = nf_ct_get(skb, &ctinfo);
	if (tmpl || ctinfo == IP_CT_UNTRACKED) {
		/* Already tracked, or explicitly NOTRACKed */
		if ((tmpl && !nf_ct_is_template(tmpl)) || ctinfo == IP_CT_UNTRACKED)
			return NF_ACCEPT;
		skb->_nfct = 0;
	}

	dataoff = get_l4proto(skb, skb_network_offset(skb), state->pf, &protonum);
	...
repeat:
	ret = resolve_normal_ct(tmpl, skb, dataoff, protonum, state);
	if (ret < 0) { ... goto out; }

	ct = nf_ct_get(skb, &ctinfo);
	...
	ret = nf_conntrack_handle_packet(ct, skb, dataoff, ctinfo, state);
	if (ret <= 0) {
		/* The protocol tracker rejected it (e.g. an invalid TCP state) */
		nf_ct_put(ct);
		skb->_nfct = 0;
		...
		return -ret;
	}

	if (ctinfo == IP_CT_ESTABLISHED_REPLY &&
	    !test_and_set_bit(IPS_SEEN_REPLY_BIT, &ct->status))
		nf_conntrack_event_cache(IPCT_REPLY, ct);
out:
	...
	return ret;
}

/* T.3(c): the entry is inserted only when the packet SURVIVES. */
int __nf_conntrack_confirm(struct sk_buff *skb)
{
	const struct nf_conntrack_zone *zone;
	unsigned int hash, reply_hash;
	struct nf_conntrack_tuple_hash *h;
	struct nf_conn *ct;
	...
	ct = nf_ct_get(skb, &ctinfo);
	if (!ct || CTINFO2DIR(ctinfo) != IP_CT_DIR_ORIGINAL)
		return NF_ACCEPT;
	...
	local_bh_disable();
	do {
		sequence = read_seqcount_begin(&nf_conntrack_generation);
		hash = *(unsigned long *)&ct->tuplehash[IP_CT_DIR_REPLY].hnnode.pprev;
		hash = scale_hash(hash);
		reply_hash = hash_conntrack(net, &ct->tuplehash[IP_CT_DIR_REPLY].tuple,
					    nf_ct_zone_id(...));
	} while (nf_conntrack_double_lock(net, hash, reply_hash, sequence));
	...
	/* Did someone else insert the same tuple while we were not looking? */
	hlist_nulls_for_each_entry(h, n, &nf_conntrack_hash[hash], hnnode) {
		if (nf_ct_key_equal(h, &ct->tuplehash[IP_CT_DIR_ORIGINAL].tuple,
				    zone, net))
			goto out;
	}
	...
	__nf_conntrack_hash_insert(ct, hash, reply_hash);
	...
	return NF_ACCEPT;

out:
	nf_ct_add_to_dying_list(ct);
	ret = nf_ct_resolve_clash(skb, h, reply_hash);
	...
	NF_CT_STAT_INC(net, insert_failed);      /* the race of T.3(c) */
	...
	return NF_DROP;
}
```

### 1.4 nftables' VM

```c
void nft_do_chain(struct nft_pktinfo *pkt, void *priv)
{
	const struct nft_chain *chain = priv, *basechain = chain;
	const struct nft_rule_dp *rule, *last_rule;
	struct nft_regs regs = {};
	unsigned int stackptr = 0;
	struct nft_jumpstack jumpstack[NFT_JUMP_STACK_SIZE];
	bool genbit = READ_ONCE(net->nft.gencursor);
	struct nft_rule_blob *blob;
	struct nft_traceinfo info;

	...
do_chain:
	...
	rule = (struct nft_rule_dp *)blob->data;
	last_rule = (void *)blob->data + blob->size;
next_rule:
	regs.verdict.code = NFT_CONTINUE;
	for (; rule < last_rule; rule = nft_rule_next(rule)) {
		nft_rule_dp_for_each_expr(expr, last, rule) {
			/* Indirect-call optimisation for the hot expressions */
			if (expr->ops == &nft_cmp_fast_ops) {
				if (nft_payload_fast_eval(...))
					...
			} else if (expr->ops == &nft_bitwise_fast_ops) {
				nft_bitwise_fast_eval(expr, &regs);
			} else if (expr->ops != &nft_payload_fast_ops ||
				   !nft_payload_fast_eval(expr, &regs, pkt)) {
				expr_call_ops_eval(expr, &regs, pkt);   /* THE VM */
			}

			if (regs.verdict.code != NFT_CONTINUE)
				break;
		}

		switch (regs.verdict.code) {
		case NFT_BREAK:
			regs.verdict.code = NFT_CONTINUE;
			continue;
		case NFT_CONTINUE:
			nft_trace_packet(&info, chain, rule, NFT_TRACETYPE_RULE);
			continue;
		}
		break;
	}
	...
	switch (regs.verdict.code & NF_VERDICT_MASK) {
	case NF_ACCEPT:
	case NF_QUEUE:
	case NF_STOLEN:
		return regs.verdict.code;
	case NF_DROP:
		return NF_DROP;
	}

	switch (regs.verdict.code) {
	case NFT_JUMP:
		if (WARN_ON_ONCE(stackptr >= NFT_JUMP_STACK_SIZE))
			return NF_DROP;
		jumpstack[stackptr].chain = chain;
		jumpstack[stackptr].rule = nft_rule_next(rule);
		jumpstack[stackptr].last_rule = last_rule;
		stackptr++;
		fallthrough;
	case NFT_GOTO:
		chain = regs.verdict.chain;
		goto do_chain;
	case NFT_CONTINUE:
	case NFT_RETURN:
		break;
	}
	...
}
```

The `nft_cmp_fast_ops` / `nft_payload_fast_ops` special-casing is worth noting: the two most common expressions are inlined to avoid an indirect call per rule per packet, which is a measurable win given retpolines.

A set lookup:

```c
void nft_lookup_eval(const struct nft_expr *expr, struct nft_regs *regs,
		     const struct nft_pktinfo *pkt)
{
	const struct nft_lookup *priv = nft_expr_priv(expr);
	const struct nft_set *set = priv->set;
	const struct nft_set_ext *ext = NULL;
	const struct net *net = nft_net(pkt);
	bool found;

	found = nft_set_do_lookup(net, set, &regs->data[priv->sreg], &ext) ^
		priv->invert;
	if (!found) {
		ext = nft_set_catchall_lookup(net, set);
		if (!ext) { regs->verdict.code = NFT_BREAK; return; }
	}

	if (ext) {
		if (priv->dreg_set)
			nft_data_copy(&regs->data[priv->dreg],
				      nft_set_ext_data(ext), set->dlen);   /* MAP */
		nft_set_elem_update_expr(ext, regs, pkt);
	}
}
```

One expression, whether the set has 1 element or 10 million.

### 1.5 The qdisc run loop

```c
void __qdisc_run(struct Qdisc *q)
{
	int quota = READ_ONCE(net_hotdata.dev_tx_weight);
	int packets;

	while (qdisc_restart(q, &packets)) {
		quota -= packets;
		if (quota <= 0) {
			if (q->flags & TCQ_F_NOLOCK)
				set_bit(__QDISC_STATE_MISSED, &q->state);
			else
				__netif_schedule(q);    /* defer to softirq */
			break;
		}
	}
}

static inline bool qdisc_restart(struct Qdisc *q, int *packets)
{
	spinlock_t *root_lock = NULL;
	struct netdev_queue *txq;
	struct net_device *dev;
	struct sk_buff *skb;
	bool validate;

	skb = dequeue_skb(q, &validate, packets);    /* the qdisc's ->dequeue */
	if (unlikely(!skb))
		return false;

	if (!(q->flags & TCQ_F_NOLOCK))
		root_lock = qdisc_lock(q);

	dev = qdisc_dev(q);
	txq = skb_get_tx_queue(dev, skb);

	return sch_direct_xmit(skb, q, dev, txq, root_lock, validate);
}
```

The `dev_tx_weight` quota bounds how much one qdisc run does before deferring — the same batching-with-a-bound discipline as NAPI (Ch. 71 §T.6).

### 1.6 FQ-CoDel

```c
struct fq_codel_sched_data {
	struct tcf_proto __rcu *filter_list;
	struct tcf_block *block;
	struct fq_codel_flow *flows;       /* the flow table */
	u32		*backlogs;
	u32		flows_cnt;          /* 1024 by default */
	u32		quantum;
	u32		drop_batch_size;
	u32		memory_limit;
	struct codel_params cparams;
	struct codel_stats cstats;
	u32		memory_usage;
	u32		drop_overmemory;
	u32		drop_overlimit;
	u32		new_flow_count;

	struct list_head new_flows;        /* T.7: new flows get priority */
	struct list_head old_flows;
};

static struct sk_buff *fq_codel_dequeue(struct Qdisc *sch)
{
	struct fq_codel_sched_data *q = qdisc_priv(sch);
	struct sk_buff *skb;
	struct fq_codel_flow *flow;
	struct list_head *head;

begin:
	/* NEW flows first -- a new connection's first packets are latency-critical */
	head = &q->new_flows;
	if (list_empty(head)) {
		head = &q->old_flows;
		if (list_empty(head))
			return NULL;
	}
	flow = list_first_entry(head, struct fq_codel_flow, flowchain);

	if (flow->deficit <= 0) {
		flow->deficit += q->quantum;      /* DRR */
		list_move_tail(&flow->flowchain, &q->old_flows);
		goto begin;
	}

	/* THE CoDel algorithm, per flow */
	skb = codel_dequeue(sch, &sch->qstats.backlog, &q->cparams,
			    &flow->cvars, &q->cstats, qdisc_pkt_len,
			    codel_get_enqueue_time, drop_func, dequeue_func);

	if (!skb) {
		if ((head == &q->new_flows) && !list_empty(&q->old_flows))
			list_move_tail(&flow->flowchain, &q->old_flows);
		else
			list_del_init(&flow->flowchain);
		goto begin;
	}
	qdisc_bstats_update(sch, skb);
	flow->deficit -= qdisc_pkt_len(skb);
	...
	return skb;
}
```

And CoDel itself:

```c
static struct sk_buff *codel_dequeue(void *ctx, u32 *backlog,
				     struct codel_params *params,
				     struct codel_vars *vars,
				     struct codel_stats *stats, ...)
{
	struct sk_buff *skb = dequeue_func(ctx, vars);
	codel_time_t now;
	bool drop;

	if (!skb) { vars->dropping = false; return skb; }
	now = codel_get_time();
	drop = codel_should_drop(skb, ctx, vars, params, stats, ...);

	if (vars->dropping) {
		if (!drop) {
			vars->dropping = false;      /* delay fell below target */
		} else if (codel_time_after_eq(now, vars->drop_next)) {
			/* T.7: drop FASTER if the queue persists */
			while (vars->dropping &&
			       codel_time_after_eq(now, vars->drop_next)) {
				vars->count++;
				drop_func(skb, ctx);
				...
				skb = dequeue_func(ctx, vars);
				if (!codel_should_drop(skb, ctx, vars, params, stats, ...)) {
					vars->dropping = false;
				} else {
					/* interval / sqrt(count) */
					codel_Newton_step(vars);
					vars->drop_next =
						codel_control_law(vars->drop_next,
								  params->interval,
								  vars->rec_inv_sqrt);
				}
			}
		}
	} else if (drop) {
		...
		vars->dropping = true;
		vars->count = (vars->count > 2 && ...) ? vars->count - 2 : 1;
		codel_Newton_step(vars);
		vars->drop_next = codel_control_law(now, params->interval,
						    vars->rec_inv_sqrt);
	}
	return skb;
}

static bool codel_should_drop(const struct sk_buff *skb, void *ctx, ...)
{
	...
	vars->ldelay = now - codel_get_enqueue_time(skb);
	...
	if (codel_time_before(vars->ldelay, params->target) ||
	    *backlog <= params->mtu) {
		/* Below target: the MINIMUM has not been above target */
		vars->first_above_time = 0;
		return false;
	}

	if (vars->first_above_time == 0) {
		/* First time above target: start the interval timer */
		vars->first_above_time = now + params->interval;
	} else if (codel_time_after(now, vars->first_above_time)) {
		ok_to_drop = true;      /* above target for a FULL interval */
	}
	return ok_to_drop;
}
```

**`first_above_time` is the minimum-tracking mechanism of §T.7**: the moment delay drops below target, it resets, so only persistent queueing triggers dropping.

### 1.7 Observability

| Where | What |
|---|---|
| `nft list ruleset` ★★★ | the whole rule set |
| `nft -a list ruleset` | with handles, for deletion |
| `nft monitor` ★★★ | live rule-set changes |
| `nft list counters`, `list sets` | |
| `iptables -L -v -n --line-numbers` ★★★ | with packet/byte counts |
| `iptables -t nat -L -v -n`, `-t mangle`, `-t raw` | |
| `conntrack -L`, `-E`, `-S` ★★★ | list, event-monitor, statistics |
| `/proc/net/nf_conntrack` | the raw table |
| `/proc/sys/net/netfilter/*` ★★★ | timeouts and limits |
| `cat /proc/sys/net/netfilter/nf_conntrack_count` ★★★ | |
| `tc -s qdisc show dev eth0` ★★★ | **per-qdisc statistics including drops and overlimits** |
| `tc -s class show dev eth0` | per-class |
| `tc -s filter show dev eth0` | per-filter hit counts |
| `tc -j -s qdisc show` | JSON, for scripting |
| `trace-cmd record -e netfilter:\* -e qdisc:\*` | |
| `nft ... meta nftrace set 1` + `nft monitor trace` ★★★ | **per-packet rule tracing** |
| `iptables -t raw -A PREROUTING -j TRACE` | the iptables equivalent |
| `bpftrace` on `nf_hook_slow`, `nft_do_chain`, `__qdisc_run` | |

`nft monitor trace` is the most useful debugging tool here — it shows, for a selected packet, every rule it matched and the verdict at each step.

---

## 2. Practice

### Lab 73.1 — The five hooks

```sh
sudo apt install -y nftables iptables conntrack iproute2 bpfcc-tools

# Watch every hook traversal -- T.2
sudo bpftrace -e '
kprobe:nf_hook_slow {
	$state = (struct nf_hook_state *)arg1;
	@hook[$state->hook == 0 ? "PRE_ROUTING" :
	      $state->hook == 1 ? "LOCAL_IN" :
	      $state->hook == 2 ? "FORWARD" :
	      $state->hook == 3 ? "LOCAL_OUT" : "POST_ROUTING"] = count();
}
interval:s:5 { print(@hook); clear(@hook); }' &

ping -c 5 127.0.0.1 > /dev/null
curl -s -m 2 http://127.0.0.1:22 > /dev/null 2>&1
```

Set up a forwarding topology:

```sh
sudo ip netns add cl
sudo ip netns add srv
sudo ip link add v-cl type veth peer name v-gw-cl
sudo ip link add v-srv type veth peer name v-gw-srv
sudo ip link set v-cl netns cl
sudo ip link set v-srv netns srv

sudo ip addr add 10.10.0.1/24 dev v-gw-cl
sudo ip addr add 10.20.0.1/24 dev v-gw-srv
sudo ip link set v-gw-cl up
sudo ip link set v-gw-srv up
sudo sysctl -w net.ipv4.ip_forward=1

sudo ip netns exec cl ip addr add 10.10.0.2/24 dev v-cl
sudo ip netns exec cl ip link set v-cl up
sudo ip netns exec cl ip link set lo up
sudo ip netns exec cl ip route add default via 10.10.0.1

sudo ip netns exec srv ip addr add 10.20.0.2/24 dev v-srv
sudo ip netns exec srv ip link set v-srv up
sudo ip netns exec srv ip link set lo up
sudo ip netns exec srv ip route add default via 10.20.0.1

sudo ip netns exec cl ping -c 2 10.20.0.2
```

Now the hook order becomes visible:

```sh
sudo bpftrace -e '
kprobe:nf_hook_slow {
	$state = (struct nf_hook_state *)arg1;
	printf("%-14s in=%s out=%s\n",
	       $state->hook == 0 ? "PRE_ROUTING" :
	       $state->hook == 1 ? "LOCAL_IN" :
	       $state->hook == 2 ? "FORWARD" :
	       $state->hook == 3 ? "LOCAL_OUT" : "POST_ROUTING",
	       $state->in ? $state->in->name : "-",
	       $state->out ? $state->out->name : "-");
}' &
sudo ip netns exec cl ping -c 2 10.20.0.2 > /dev/null
sleep 1
```

A forwarded packet traverses PRE_ROUTING → FORWARD → POST_ROUTING; a local one PRE_ROUTING → LOCAL_IN.

Register a hook yourself:

```c
// SPDX-License-Identifier: GPL-2.0
/* hookdemo.c -- count packets at every hook. */
#include <linux/module.h>
#include <linux/netfilter.h>
#include <linux/netfilter_ipv4.h>
#include <linux/ip.h>
#include <linux/skbuff.h>

static atomic_t counts[NF_INET_NUMHOOKS];
static const char *names[] = {
	"PRE_ROUTING", "LOCAL_IN", "FORWARD", "LOCAL_OUT", "POST_ROUTING"
};

static unsigned int demo_hook(void *priv, struct sk_buff *skb,
			      const struct nf_hook_state *state)
{
	struct iphdr *iph;

	atomic_inc(&counts[state->hook]);

	if (skb->protocol == htons(ETH_P_IP)) {
		iph = ip_hdr(skb);
		if (iph->protocol == IPPROTO_ICMP)
			pr_info("hookdemo: %-13s %pI4 -> %pI4 in=%s out=%s\n",
				names[state->hook], &iph->saddr, &iph->daddr,
				state->in ? state->in->name : "-",
				state->out ? state->out->name : "-");
	}
	return NF_ACCEPT;
}

static struct nf_hook_ops demo_ops[NF_INET_NUMHOOKS];

static int __init demo_init(void)
{
	int i, ret;

	for (i = 0; i < NF_INET_NUMHOOKS; i++) {
		demo_ops[i].hook     = demo_hook;
		demo_ops[i].pf       = NFPROTO_IPV4;
		demo_ops[i].hooknum  = i;
		demo_ops[i].priority = NF_IP_PRI_FIRST;
		ret = nf_register_net_hook(&init_net, &demo_ops[i]);
		if (ret) {
			while (--i >= 0)
				nf_unregister_net_hook(&init_net, &demo_ops[i]);
			return ret;
		}
	}
	pr_info("hookdemo: registered\n");
	return 0;
}

static void __exit demo_exit(void)
{
	int i;

	for (i = 0; i < NF_INET_NUMHOOKS; i++) {
		nf_unregister_net_hook(&init_net, &demo_ops[i]);
		pr_info("hookdemo: %-13s %d packets\n",
			names[i], atomic_read(&counts[i]));
	}
}

module_init(demo_init);
module_exit(demo_exit);
MODULE_LICENSE("GPL");
```

```sh
cat > Makefile <<'EOF'
obj-m += hookdemo.o
all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
EOF
make && sudo insmod hookdemo.ko
sudo ip netns exec cl ping -c 2 10.20.0.2 > /dev/null
dmesg | tail -10
sudo rmmod hookdemo
dmesg | tail -6
```

---

### Lab 73.2 — Connection tracking

```sh
sudo modprobe nf_conntrack
sysctl net.netfilter.nf_conntrack_max net.netfilter.nf_conntrack_count
sysctl net.netfilter.nf_conntrack_buckets

# Conntrack is only active if something asks for it:
sudo nft add table inet filter
sudo nft add chain inet filter input '{ type filter hook input priority 0; }'
sudo nft add rule inet filter input ct state established,related accept

conntrack -S
conntrack -L 2>/dev/null | head
```

Watch a connection's lifecycle — §T.3:

```sh
sudo conntrack -E -e ALL 2>/dev/null &
sleep 1
sudo ip netns exec srv python3 -m http.server 8080 > /dev/null 2>&1 &
sleep 1
sudo ip netns exec cl curl -s -m 3 http://10.20.0.2:8080/ > /dev/null
sleep 3
kill %1 2>/dev/null
```

```
    [NEW] tcp 6 120 SYN_SENT src=10.10.0.2 dst=10.20.0.2 sport=45678 dport=8080 [UNREPLIED] ...
 [UPDATE] tcp 6 60 SYN_RECV src=10.10.0.2 ...
 [UPDATE] tcp 6 432000 ESTABLISHED src=10.10.0.2 ... [ASSURED]
 [UPDATE] tcp 6 120 FIN_WAIT ...
 [UPDATE] tcp 6 30 LAST_ACK ...
 [UPDATE] tcp 6 120 TIME_WAIT ...
[DESTROY] tcp 6 src=10.10.0.2 ...
```

The timeouts — §T.3(a):

```sh
sysctl -a 2>/dev/null | grep nf_conntrack_tcp_timeout
sysctl -a 2>/dev/null | grep nf_conntrack_udp_timeout
sysctl net.netfilter.nf_conntrack_tcp_timeout_established
# 432000 = 5 days
```

Demonstrate the trap:

```sh
sudo sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=10
sudo ip netns exec cl bash -c '
  exec 3<>/dev/tcp/10.20.0.2/8080
  echo "connection open"
  sleep 15
  echo -e "GET / HTTP/1.0\r\n\r" >&3
  timeout 3 cat <&3 | head -2 || echo "CONNECTION BROKEN"
' 2>/dev/null
sudo sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=432000
```

Fill the table — §T.3(b):

```sh
sudo sysctl -w net.netfilter.nf_conntrack_max=128
dmesg -C
sudo ip netns exec cl bash -c '
for i in $(seq 1 300); do
  (timeout 2 nc -w1 10.20.0.2 8080 < /dev/null &) 2>/dev/null
done; sleep 3'
dmesg | grep -i conntrack | tail -3
conntrack -S | head -3
sudo sysctl -w net.netfilter.nf_conntrack_max=262144
```

The confirmation race — §T.3(c):

```sh
conntrack -S
# insert_failed is the race; drop is the table being full

sudo bpftrace -e '
kprobe:__nf_conntrack_confirm { @confirm = count(); }
kprobe:nf_conntrack_alloc     { @alloc = count(); }
kprobe:nf_ct_resolve_clash    { @clash = count(); }
kprobe:nf_conntrack_in        { @in = count(); }
interval:s:5 { print(@in); print(@alloc); print(@confirm); print(@clash);
               clear(@in); clear(@alloc); clear(@confirm); clear(@clash); }' &

sudo ip netns exec cl bash -c 'for i in $(seq 1 200); do
  (nc -z -w1 10.20.0.2 8080 2>/dev/null &); done; sleep 3'
```

`notrack` for high-volume flows:

```sh
sudo nft add table inet raw
sudo nft add chain inet raw prerouting '{ type filter hook prerouting priority -300; }'
sudo nft add rule inet raw prerouting ip daddr 10.20.0.2 udp dport 53 notrack

sudo bpftrace -e 'kprobe:nf_conntrack_alloc { @ = count(); }
                  interval:s:5 { print(@); clear(@); }' &
# DNS traffic now bypasses conntrack entirely.
```

Helpers — §T.3:

```sh
sysctl net.netfilter.nf_conntrack_helper
lsmod | grep nf_conntrack_
# Explicit assignment (the modern, safe way):
sudo nft add table inet cthelper
sudo nft 'add ct helper inet cthelper ftp-standard { type "ftp" protocol tcp; }' 2>/dev/null
```

---

### Lab 73.3 — NAT

```sh
# MASQUERADE the client network -- T.4
sudo nft add table ip nat
sudo nft add chain ip nat postrouting '{ type nat hook postrouting priority 100; }'
sudo nft add chain ip nat prerouting '{ type nat hook prerouting priority -100; }'
sudo nft add rule ip nat postrouting ip saddr 10.10.0.0/24 oif v-gw-srv masquerade

sudo ip netns exec srv ip route del default 2>/dev/null
# The server now has no route back to 10.10.0.0/24 -- NAT makes it work anyway
sudo ip netns exec cl ping -c 2 10.20.0.2

conntrack -L 2>/dev/null | grep -i 10.10.0.2
```

```
icmp 1 29 src=10.10.0.2 dst=10.20.0.2 type=8 code=0 id=1234
         src=10.20.0.2 dst=10.20.0.1 type=0 code=0 id=1234 mark=0 use=1
         ^^^ the REPLY tuple shows the translation (T.3's two tuples)
```

Watch NAT happen exactly once:

```sh
sudo bpftrace -e '
kprobe:nf_nat_packet          { @nat_packet = count(); }
kprobe:nf_nat_setup_info      { @setup = count(); }
kprobe:nf_nat_alloc_null_binding { @null_bind = count(); }
interval:s:5 { print(@setup); print(@nat_packet);
               clear(@setup); clear(@nat_packet); }' &

sudo ip netns exec srv python3 -m http.server 8080 > /dev/null 2>&1 &
sleep 1
sudo ip netns exec cl curl -s -m 3 http://10.20.0.2:8080/ > /dev/null
```

**`setup` fires once per connection; `nat_packet` fires per packet.** §T.4's key property.

DNAT — port forwarding:

```sh
sudo nft add rule ip nat prerouting iif v-gw-cl tcp dport 9999 \
     dnat to 10.20.0.2:8080

sudo ip netns exec cl curl -s -m 3 http://10.10.0.1:9999/ | head -3
conntrack -L 2>/dev/null | grep 9999
```

Why DNAT must be in PRE_ROUTING — §T.2:

```sh
# Try it in POST_ROUTING: it cannot work, because routing already happened
sudo nft add rule ip nat postrouting tcp dport 9998 dnat to 10.20.0.2:8080 2>&1 | tail -2
```

Port exhaustion — §T.4:

```sh
sudo nft flush table ip nat 2>/dev/null
sudo nft add chain ip nat postrouting '{ type nat hook postrouting priority 100; }'
sudo nft add rule ip nat postrouting ip saddr 10.10.0.0/24 \
     snat to 10.20.0.1 comment "fixed source"

sudo bpftrace -e '
kretprobe:nf_nat_l4proto_unique_tuple { @tries = count(); }
kprobe:nf_nat_used_tuple { @used = count(); }
interval:s:5 { print(@tries); print(@used); clear(@tries); clear(@used); }' &

sudo ip netns exec cl bash -c 'for i in $(seq 1 500); do
  (nc -z -w1 10.20.0.2 8080 2>/dev/null &); done; sleep 3'
conntrack -C
```

Randomised port allocation:

```sh
sudo nft flush chain ip nat postrouting
sudo nft add rule ip nat postrouting ip saddr 10.10.0.0/24 \
     snat to 10.20.0.1 fully-random
conntrack -L 2>/dev/null | grep -oP 'sport=\K\d+' | head -20
# Without fully-random the ports are sequential; with it they are scattered.
```

---

### Lab 73.4 — nftables versus iptables

```sh
# Linear evaluation cost -- T.5
sudo iptables -F
for i in $(seq 1 2000); do
  sudo iptables -A INPUT -s 192.0.2.$((i % 254 + 1)) -j ACCEPT
done
sudo iptables -L INPUT -n | wc -l

sudo bpftrace -e '
kprobe:ipt_do_table { @t[tid] = nsecs; }
kretprobe:ipt_do_table /@t[tid]/ { @ns = hist(nsecs - @t[tid]); delete(@t[tid]); }
interval:s:10 { print(@ns); exit(); }' &
ping -c 100 -i 0.02 127.0.0.1 > /dev/null 2>&1
wait
sudo iptables -F
```

Now with an nftables set:

```sh
sudo nft add table ip perftest
sudo nft add chain ip perftest input '{ type filter hook input priority 0; }'
sudo nft add set ip perftest allowed '{ type ipv4_addr; }'
for i in $(seq 1 254); do
  sudo nft add element ip perftest allowed "{ 192.0.2.$i }" 2>/dev/null
done
sudo nft add rule ip perftest input ip saddr @allowed accept

sudo bpftrace -e '
kprobe:nft_do_chain { @t[tid] = nsecs; }
kretprobe:nft_do_chain /@t[tid]/ { @ns = hist(nsecs - @t[tid]); delete(@t[tid]); }
interval:s:10 { print(@ns); exit(); }' &
ping -c 100 -i 0.02 127.0.0.1 > /dev/null 2>&1
wait
```

See the bytecode — §T.5:

```sh
sudo nft --debug=netlink add rule ip perftest input tcp dport 22 accept 2>&1 | head -15
```

```
ip perftest input
  [ meta load l4proto => reg 1 ]
  [ cmp eq reg 1 0x00000006 ]
  [ payload load 2b @ transport header + 2 => reg 1 ]
  [ cmp eq reg 1 0x00001600 ]
  [ immediate reg 0 accept ]
```

Set types — §T.5:

```sh
# Interval set (rbtree)
sudo nft add set ip perftest nets '{ type ipv4_addr; flags interval; }'
sudo nft add element ip perftest nets '{ 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 }'
sudo nft list set ip perftest nets

# Concatenated set (pipapo)
sudo nft add set ip perftest svc '{ type ipv4_addr . inet_service; }'
sudo nft add element ip perftest svc '{ 10.20.0.2 . 8080, 10.20.0.2 . 443 }'
sudo nft add rule ip perftest input ip daddr . tcp dport @svc accept
sudo nft list set ip perftest svc

# A MAP: one lookup replacing a chain of rules
sudo nft add map ip perftest portmap '{ type inet_service : verdict; }'
sudo nft add element ip perftest portmap '{ 22 : accept, 80 : accept, 443 : accept, 23 : drop }'
sudo nft add rule ip perftest input tcp dport vmap @portmap
sudo nft list map ip perftest portmap
```

The `inet` family — one rule set for v4 and v6:

```sh
sudo nft add table inet unified
sudo nft add chain inet unified input '{ type filter hook input priority 0; policy drop; }'
sudo nft add rule inet unified input ct state established,related accept
sudo nft add rule inet unified input iif lo accept
sudo nft add rule inet unified input tcp dport 22 accept
sudo nft add rule inet unified input icmp type echo-request accept
sudo nft add rule inet unified input icmpv6 type { echo-request, nd-neighbor-solicit, \
     nd-neighbor-advert, nd-router-advert } accept
sudo nft list table inet unified
```

Atomic updates:

```sh
cat > /tmp/ruleset.nft <<'EOF'
flush table inet unified
table inet unified {
    chain input {
        type filter hook input priority 0; policy drop;
        ct state established,related accept
        iif lo accept
        tcp dport { 22, 80, 443 } accept
        icmp type echo-request limit rate 10/second accept
        counter comment "dropped"
    }
}
EOF
sudo nft -f /tmp/ruleset.nft      # ATOMIC: all or nothing
sudo nft list table inet unified
```

Rule tracing — §1.7's best tool:

```sh
sudo nft add rule inet unified input tcp dport 22 meta nftrace set 1
sudo nft monitor trace &
sleep 1
nc -z -w1 127.0.0.1 22 2>/dev/null
sleep 2
kill %1 2>/dev/null
```

Cleanup:

```sh
sudo nft flush ruleset
```

---

### Lab 73.5 — Qdiscs

```sh
IFACE=v-gw-cl
tc qdisc show dev $IFACE
sysctl net.core.default_qdisc
```

Compare disciplines — §T.7:

```sh
sudo ip netns exec srv iperf3 -s -D 2>/dev/null
sleep 1

for q in pfifo_fast fq_codel fq cake sfq; do
  sudo tc qdisc del dev $IFACE root 2>/dev/null
  sudo tc qdisc add dev $IFACE root $q 2>/dev/null || { echo "$q unavailable"; continue; }
  sudo tc qdisc add dev $IFACE root netem rate 50mbit 2>/dev/null
  sudo tc qdisc del dev $IFACE root 2>/dev/null
  sudo tc qdisc add dev $IFACE root handle 1: netem rate 50mbit delay 20ms
  sudo tc qdisc add dev $IFACE parent 1: handle 2: $q 2>/dev/null || continue

  echo "=== $q ==="
  sudo ip netns exec cl iperf3 -c 10.20.0.2 -t 10 > /dev/null 2>&1 &
  sleep 3
  echo -n "  latency under load: "
  sudo ip netns exec cl ping -c 5 -i 0.2 10.20.0.2 2>/dev/null | tail -1
  wait
  sudo tc -s qdisc show dev $IFACE | grep -A2 "$q" | head -3
done
```

**Bufferbloat, demonstrated** — §T.7:

```sh
sudo tc qdisc del dev $IFACE root 2>/dev/null

# A huge buffer: the bufferbloat case
sudo tc qdisc add dev $IFACE root handle 1: netem rate 10mbit limit 10000
echo "=== 10000-packet buffer (bufferbloat) ==="
echo -n "idle latency: "
sudo ip netns exec cl ping -c 3 10.20.0.2 2>/dev/null | tail -1
sudo ip netns exec cl iperf3 -c 10.20.0.2 -t 15 > /dev/null 2>&1 &
sleep 4
echo -n "loaded latency: "
sudo ip netns exec cl ping -c 5 -i 0.3 10.20.0.2 2>/dev/null | tail -1
wait

# Now with fq_codel managing it
sudo tc qdisc del dev $IFACE root 2>/dev/null
sudo tc qdisc add dev $IFACE root handle 1: netem rate 10mbit limit 10000
sudo tc qdisc add dev $IFACE parent 1: handle 2: fq_codel
echo "=== same buffer + fq_codel ==="
sudo ip netns exec cl iperf3 -c 10.20.0.2 -t 15 > /dev/null 2>&1 &
sleep 4
echo -n "loaded latency: "
sudo ip netns exec cl ping -c 5 -i 0.3 10.20.0.2 2>/dev/null | tail -1
wait
sudo tc -s qdisc show dev $IFACE | grep -A3 fq_codel
```

**Two orders of magnitude in latency, same buffer.** That is CoDel.

CoDel's parameters:

```sh
sudo tc qdisc del dev $IFACE root 2>/dev/null
for target in 1ms 5ms 50ms; do
  sudo tc qdisc del dev $IFACE root 2>/dev/null
  sudo tc qdisc add dev $IFACE root handle 1: netem rate 10mbit limit 5000
  sudo tc qdisc add dev $IFACE parent 1: handle 2: fq_codel target $target interval 100ms
  echo "=== target=$target ==="
  sudo ip netns exec cl iperf3 -c 10.20.0.2 -t 10 2>/dev/null | grep receiver | awk '{print "  throughput:", $7, $8}'
  sudo ip netns exec cl iperf3 -c 10.20.0.2 -t 12 > /dev/null 2>&1 &
  sleep 4
  echo -n "  latency: "
  sudo ip netns exec cl ping -c 4 -i 0.3 10.20.0.2 2>/dev/null | tail -1
  wait
done
```

Fair queueing:

```sh
sudo tc qdisc del dev $IFACE root 2>/dev/null
sudo tc qdisc add dev $IFACE root handle 1: netem rate 20mbit
sudo tc qdisc add dev $IFACE parent 1: handle 2: fq_codel

# One greedy flow plus one modest flow
sudo ip netns exec cl iperf3 -c 10.20.0.2 -t 12 -P 8 > /dev/null 2>&1 &
sleep 2
echo "single flow's share with 8 competing:"
sudo ip netns exec cl iperf3 -c 10.20.0.2 -t 6 -p 5201 2>/dev/null | grep receiver
wait
sudo tc -s qdisc show dev $IFACE | grep -A4 fq_codel
```

---

### Lab 73.6 — Classful shaping with HTB

```sh
sudo tc qdisc del dev $IFACE root 2>/dev/null

# The T.6 hierarchy
sudo tc qdisc add dev $IFACE root handle 1: htb default 30
sudo tc class add dev $IFACE parent 1:  classid 1:1  htb rate 100mbit
sudo tc class add dev $IFACE parent 1:1 classid 1:10 htb rate 50mbit ceil 100mbit prio 0
sudo tc class add dev $IFACE parent 1:1 classid 1:20 htb rate 30mbit ceil 100mbit prio 1
sudo tc class add dev $IFACE parent 1:1 classid 1:30 htb rate 20mbit ceil 100mbit prio 2

for c in 10 20 30; do
  sudo tc qdisc add dev $IFACE parent 1:$c handle $c: fq_codel
done

sudo tc class show dev $IFACE
```

Classify with filters:

```sh
# Interactive (SSH) to the high-priority class
sudo tc filter add dev $IFACE parent 1: protocol ip prio 1 \
     flower ip_proto tcp dst_port 22 classid 1:10
# Bulk (iperf) to the low class
sudo tc filter add dev $IFACE parent 1: protocol ip prio 2 \
     flower ip_proto tcp dst_port 5201 classid 1:30
sudo tc filter show dev $IFACE
```

Test the isolation:

```sh
sudo ip netns exec cl iperf3 -c 10.20.0.2 -t 15 > /dev/null 2>&1 &
sleep 3
sudo tc -s class show dev $IFACE | grep -A3 '1:30'
sudo tc -s class show dev $IFACE | grep -A3 '1:10'
wait
```

Borrowing — §T.6's `rate` vs `ceil`:

```sh
# With only one flow, it should borrow up to ceil
sudo tc -s class show dev $IFACE | grep -E 'class htb 1:(10|20|30)' -A2
sudo ip netns exec cl iperf3 -c 10.20.0.2 -t 8 2>/dev/null | grep receiver
# ~100mbit (ceil), not 20mbit (rate), because nothing else is competing.
```

Classify by mark, set from netfilter — §T.1's interaction:

```sh
sudo nft add table inet mangle
sudo nft add chain inet mangle postrouting '{ type filter hook postrouting priority -150; }'
sudo nft add rule inet mangle postrouting tcp dport 5201 meta mark set 0x10
sudo nft add rule inet mangle postrouting tcp dport 22 meta mark set 0x1

sudo tc filter del dev $IFACE parent 1: 2>/dev/null
sudo tc filter add dev $IFACE parent 1: protocol ip prio 1 handle 0x1  fw classid 1:10
sudo tc filter add dev $IFACE parent 1: protocol ip prio 2 handle 0x10 fw classid 1:30

sudo ip netns exec cl iperf3 -c 10.20.0.2 -t 8 > /dev/null 2>&1 &
sleep 3
sudo tc -s filter show dev $IFACE
wait
```

CAKE — the single-line alternative:

```sh
sudo tc qdisc del dev $IFACE root 2>/dev/null
sudo tc qdisc add dev $IFACE root cake bandwidth 100mbit 2>/dev/null && {
  sudo tc -s qdisc show dev $IFACE
  sudo ip netns exec cl iperf3 -c 10.20.0.2 -t 10 > /dev/null 2>&1 &
  sleep 3
  echo -n "latency under load with cake: "
  sudo ip netns exec cl ping -c 4 -i 0.3 10.20.0.2 2>/dev/null | tail -1
  wait
  sudo tc -s qdisc show dev $IFACE | head -12
}
```

---

### Lab 73.7 — tc filters, actions, and eBPF

```sh
sudo tc qdisc del dev $IFACE root 2>/dev/null
sudo tc qdisc add dev $IFACE clsact         # T.8's modern hook
tc qdisc show dev $IFACE
```

Policing on ingress — §T.8:

```sh
sudo tc filter add dev $IFACE ingress protocol ip prio 1 \
     flower ip_proto tcp \
     action police rate 10mbit burst 100k conform-exceed drop

sudo ip netns exec cl iperf3 -c 10.20.0.2 -t 8 -R 2>/dev/null | grep receiver
sudo tc -s filter show dev $IFACE ingress
```

Mirroring:

```sh
sudo ip link add mirror0 type dummy
sudo ip link set mirror0 up
sudo tc filter add dev $IFACE ingress protocol ip prio 2 \
     matchall action mirred egress mirror dev mirror0

sudo tcpdump -i mirror0 -c 5 -nn 2>/dev/null &
sudo ip netns exec cl ping -c 6 10.20.0.2 > /dev/null 2>&1
wait
```

`flower` and hardware offload — §T.8:

```sh
sudo tc filter del dev $IFACE ingress 2>/dev/null
sudo tc filter add dev $IFACE ingress protocol ip prio 1 \
     flower src_ip 10.10.0.2 dst_ip 10.20.0.2 ip_proto tcp dst_port 8080 \
     action drop
sudo tc filter show dev $IFACE ingress

# On real hardware that supports it:
# tc filter add dev eth0 ingress protocol ip prio 1 skip_sw flower ... action drop
# ^^ skip_sw means "hardware only"; it fails if the NIC cannot do it.
```

eBPF at tc — §T.8, and Ch. 75's subject:

```c
// SPDX-License-Identifier: GPL-2.0
/* tc_count.c -- count and classify packets at tc. */
#include <linux/bpf.h>
#include <linux/pkt_cls.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <linux/tcp.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>

struct {
	__uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
	__uint(max_entries, 8);
	__type(key, __u32);
	__type(value, __u64);
} counts SEC(".maps");

SEC("tc")
int classify(struct __sk_buff *skb)
{
	void *data = (void *)(long)skb->data;
	void *data_end = (void *)(long)skb->data_end;
	struct ethhdr *eth = data;
	struct iphdr *iph;
	struct tcphdr *tcph;
	__u32 key;
	__u64 *val;

	if ((void *)(eth + 1) > data_end)
		return TC_ACT_OK;
	if (eth->h_proto != bpf_htons(ETH_P_IP))
		return TC_ACT_OK;

	iph = (void *)(eth + 1);
	if ((void *)(iph + 1) > data_end)
		return TC_ACT_OK;

	key = iph->protocol;
	val = bpf_map_lookup_elem(&counts, &key);
	if (val)
		__sync_fetch_and_add(val, 1);

	if (iph->protocol == IPPROTO_TCP) {
		tcph = (void *)iph + iph->ihl * 4;
		if ((void *)(tcph + 1) > data_end)
			return TC_ACT_OK;

		/* Classify SSH into 1:10, everything else into 1:30 */
		if (tcph->dest == bpf_htons(22)) {
			skb->priority = 0x10010;
			return TC_ACT_OK;
		}
		/* Drop telnet */
		if (tcph->dest == bpf_htons(23))
			return TC_ACT_SHOT;
	}
	return TC_ACT_OK;
}

char _license[] SEC("license") = "GPL";
```

```sh
sudo apt install -y clang llvm libbpf-dev linux-headers-$(uname -r)
clang -O2 -g -target bpf -c tc_count.c -o tc_count.o \
      -I/usr/include/$(uname -m)-linux-gnu

sudo tc filter del dev $IFACE ingress 2>/dev/null
sudo tc filter add dev $IFACE ingress bpf da obj tc_count.o sec tc
sudo tc filter show dev $IFACE ingress

sudo ip netns exec cl ping -c 5 10.20.0.2 > /dev/null 2>&1
sudo ip netns exec cl nc -z -w1 10.20.0.2 23 2>/dev/null && echo "telnet allowed" || echo "telnet DROPPED"

sudo bpftool map dump name counts 2>/dev/null | head
sudo bpftool prog show 2>/dev/null | grep -A2 classify
```

The full pipeline:

```sh
sudo bpftrace -e '
kprobe:nf_hook_slow { @netfilter = count(); }
kprobe:__qdisc_run  { @qdisc_run = count(); }
kprobe:tcf_classify { @tc_classify = count(); }
kprobe:nft_do_chain { @nft = count(); }
interval:s:5 { print(@netfilter); print(@nft); print(@tc_classify); print(@qdisc_run);
               clear(@netfilter); clear(@nft); clear(@tc_classify); clear(@qdisc_run); }' &
sudo ip netns exec cl iperf3 -c 10.20.0.2 -t 8 > /dev/null 2>&1
```

---

### Lab 73.8 — Diagnose "the packet did not arrive"

The methodical procedure.

```sh
sudo nft flush ruleset
sudo tc qdisc del dev $IFACE root 2>/dev/null
sudo tc qdisc del dev $IFACE clsact 2>/dev/null

# Introduce a problem
sudo nft add table inet filter
sudo nft add chain inet filter forward '{ type filter hook forward priority 0; policy accept; }'
sudo nft add rule inet filter forward tcp dport 8080 ct state new drop

sudo ip netns exec srv python3 -m http.server 8080 > /dev/null 2>&1 &
sleep 1
sudo ip netns exec cl curl -s -m 3 http://10.20.0.2:8080/ > /dev/null 2>&1 && \
  echo "WORKS" || echo "BROKEN"
```

**Step 1: does the packet leave the source?**

```sh
sudo ip netns exec cl tcpdump -i v-cl -c 5 -nn 'port 8080' 2>/dev/null &
sudo ip netns exec cl curl -s -m 2 http://10.20.0.2:8080/ > /dev/null 2>&1
wait
```

**Step 2: does it arrive at the gateway?**

```sh
sudo tcpdump -i v-gw-cl -c 5 -nn 'port 8080' 2>/dev/null &
sudo ip netns exec cl curl -s -m 2 http://10.20.0.2:8080/ > /dev/null 2>&1
wait
```

**Step 3: does it leave the gateway?**

```sh
sudo tcpdump -i v-gw-srv -c 5 -nn 'port 8080' 2>/dev/null &
sudo ip netns exec cl curl -s -m 2 http://10.20.0.2:8080/ > /dev/null 2>&1
wait
# Nothing: it was dropped between ingress and egress on the gateway.
```

**Step 4: where exactly?**

```sh
sudo bpftrace -e '
tracepoint:skb:kfree_skb {
	@[args->reason, ksym(args->location)] = count();
}
interval:s:5 { print(@); clear(@); }' &
sudo ip netns exec cl curl -s -m 2 http://10.20.0.2:8080/ > /dev/null 2>&1
sleep 2
```

`SKB_DROP_REASON_NETFILTER_DROP` at `nf_hook_slow` — netfilter did it.

**Step 5: which rule?**

```sh
sudo nft list ruleset -a
sudo nft add rule inet filter forward tcp dport 8080 meta nftrace set 1
sudo nft monitor trace &
sleep 1
sudo ip netns exec cl curl -s -m 2 http://10.20.0.2:8080/ > /dev/null 2>&1
sleep 2
kill %1 2>/dev/null
```

Add counters to everything — the production approach:

```sh
sudo nft flush ruleset
cat > /tmp/diag.nft <<'EOF'
table inet filter {
    chain forward {
        type filter hook forward priority 0; policy accept;
        counter comment "all forwarded"
        ct state established,related counter accept comment "established"
        ct state new counter comment "new connections"
        tcp dport 8080 counter drop comment "the rule under suspicion"
    }
}
EOF
sudo nft -f /tmp/diag.nft
sudo ip netns exec cl curl -s -m 2 http://10.20.0.2:8080/ > /dev/null 2>&1
sudo nft list table inet filter
```

The complete diagnostic script:

```sh
cat > netdiag.sh <<'EOF'
#!/bin/bash
IF=${1:-eth0}
echo "=== Routing ==="
ip route show
ip rule show
echo "=== Forwarding ==="
sysctl -n net.ipv4.ip_forward net.ipv4.conf.all.rp_filter
echo "=== nftables ==="
nft list ruleset 2>/dev/null | head -40
echo "=== iptables (if any) ==="
iptables -L -v -n 2>/dev/null | head -20
iptables -t nat -L -v -n 2>/dev/null | head -15
echo "=== conntrack ==="
echo "count: $(cat /proc/sys/net/netfilter/nf_conntrack_count 2>/dev/null) / $(cat /proc/sys/net/netfilter/nf_conntrack_max 2>/dev/null)"
conntrack -S 2>/dev/null | head -3
echo "=== tc ==="
tc -s qdisc show dev $IF 2>/dev/null
tc -s filter show dev $IF 2>/dev/null | head -10
tc -s filter show dev $IF ingress 2>/dev/null | head -10
echo "=== Interface drops ==="
ip -s link show $IF | tail -4
echo "=== Drop reasons (5s sample) ==="
timeout 5 bpftrace -e 'tracepoint:skb:kfree_skb { @[args->reason] = count(); }' 2>/dev/null | tail -10
EOF
chmod +x netdiag.sh
sudo ./netdiag.sh $IFACE
```

Cleanup:

```sh
sudo nft flush ruleset
sudo tc qdisc del dev $IFACE root 2>/dev/null
sudo tc qdisc del dev $IFACE clsact 2>/dev/null
sudo ip link del mirror0 2>/dev/null
sudo ip netns del cl srv 2>/dev/null
```

---

## 3. Mastery drills

1. State the distinction between netfilter and tc in one sentence each. Then classify: rate limiting, DNAT, packet mirroring, connection tracking, priority queueing, and logging.

2. Explain why DNAT must occur in PRE_ROUTING and SNAT in POST_ROUTING. Construct the failure that results from swapping them.

3. Order the netfilter priorities from §T.2 and explain what each stage must see or not see. Then map each iptables table onto a priority.

4. Conntrack stores two tuples per connection. Explain how NAT uses the reply tuple, and trace a packet's translation in both directions.

5. Explain the conntrack confirmation race. Construct the two-packet interleaving that produces an `insert_failed`, and state why the design accepts it.

6. Compute the maximum concurrent connections a single NAT source address supports to one destination. Then compute it for 4 addresses and for a destination range of 256 addresses.

7. Derive iptables' O(N) matching cost and nftables' set-lookup cost. For a 10,000-address block list, compute the per-packet work for each.

8. Explain CoDel's use of the *minimum* delay over an interval. Construct a burst and a standing queue, and show that CoDel distinguishes them.

9. Explain why `target` and `interval` need no tuning, and construct the link where RED would require retuning but CoDel would not.

10. Compare `fq_codel`, `fq`, and `cake`. For each, state the workload it was designed for and the one where it is the wrong choice.

11. HTB has both `rate` and `ceil`. Prove that `rate` alone gives isolation without efficiency, and `ceil` alone gives efficiency without isolation.

12. `flower` can be offloaded to hardware and `u32` cannot. Explain the structural reason, and state what this implies about designing a match language.

13. You are told that a service is unreachable through a Linux gateway. Give the ordered diagnostic procedure from this chapter, with the specific command at each step and what each result rules out.

---

## 4. Further reading

**Kernel documentation**

- `Documentation/networking/nf_conntrack-sysctl.rst` ★★★ — §T.3's tunables, all of them.
- `Documentation/networking/netfilter-sysctl.rst`
- `Documentation/networking/sch_cake.rst` ★★★ — §T.7's CAKE, by its authors; an excellent design document.
- `Documentation/networking/tc-actions-env-rules.rst`
- `Documentation/networking/filter.rst` — classic BPF, background for Ch. 75.
- `man 8 nft` ★★★ — **long and worth reading completely.** The syntax reference and the semantic explanation both.
- `man 8 tc`, `man 8 tc-htb`, `man 8 tc-fq_codel` ★★★, `man 8 tc-cake` ★★★, `man 8 tc-netem` ★★★, `man 8 tc-flower`, `man 8 tc-bpf`
- `man 8 conntrack`, `man 8 iptables-extensions`

**Project documentation**

- **wiki.nftables.org** ★★★ — **the nftables documentation.** "Quick reference", "Moving from iptables to nftables", "Sets", "Maps and Verdict maps", "Performing Network Address Translation". Read the migration guide even if you know iptables well; the model differs.
- **netfilter.org/documentation/** — the packet-flow diagrams ★★★ (the "Netfilter packet flow" diagram is the single most useful reference in this chapter; print it).
- **lartc.org** — the Linux Advanced Routing & Traffic Control HOWTO. Dated but the tc concepts are unchanged and the explanations are good.
- **bufferbloat.net** ★★★ — the project that produced CoDel, FQ-CoDel, and CAKE. The wiki and the mailing list archives are the primary source for §T.7.

**Papers**

- Nichols & Jacobson, "Controlling Queue Delay," ACM Queue 2012 ★★★ — **CoDel.** Short, clear, and one of the best systems papers of the decade. Read it.
- Hoeiland-Joergensen, McKenney, Taht, Gettys, Dumazet, "The Flow Queue CoDel Packet Scheduler and Active Queue Management Algorithm," RFC 8290 ★★★ — FQ-CoDel, normatively.
- Gettys & Nichols, "Bufferbloat: Dark Buffers in the Internet," ACM Queue 2011 ★★★ — the problem statement.
- Floyd & Jacobson, "Random Early Detection Gateways for Congestion Avoidance," IEEE/ACM ToN 1993 — RED, and why it needed tuning.
- Høiland-Jørgensen et al., "Piece of CAKE: A Comprehensive Queue Management Solution for Home Gateways," 2018 ★★★
- Pfaff et al., "The Design and Implementation of Open vSwitch," NSDI 2015 ★★★ — the flow-caching architecture and why `flower` matters.
- Stefano Brivio's work on `nft_set_pipapo` and the "PIle PAcket POlicies" algorithm — the set implementation of §T.5.
- Demers, Keshav, Shenker, "Analysis and Simulation of a Fair Queueing Algorithm," SIGCOMM 1989 — fair queueing's origin.

**RFCs**

- RFC 8290 — FQ-CoDel ★★★
- RFC 8289 — CoDel
- RFC 7567 — AQM recommendations (IETF's position on why you need AQM)
- RFC 3168 — ECN
- RFC 2475, 2474 — DiffServ, for DSCP marking
- RFC 5382, 4787 — NAT behavioural requirements for TCP and UDP ★★★ — what a well-behaved NAT must do

**LWN**

- "The return of nftables" ★★★ and the nftables merge coverage
- "nftables: a new packet filtering engine" ★★★
- "Controlling queue delay" and the bufferbloat series ★★★
- "Network transmit queue limits" (BQL, Ch. 71)
- "CAKE: a new qdisc for home routers"
- "Flow-based queueing and the FQ scheduler"
- "Connection tracking scalability" and the conntrack lock-contention work
- "bpfilter" — the attempt to reimplement iptables with BPF, and what happened to it
- "tc-bpf and the eBPF traffic control classifier"
- "Hardware offload of tc rules"

**Source reading order**

1. The netfilter.org packet-flow diagram ★★★, then `Documentation/networking/nf_conntrack-sysctl.rst`.
2. `include/linux/netfilter.h` and `include/uapi/linux/netfilter_ipv4.h` ★★★ — the hooks and priorities.
3. `net/netfilter/core.c`: `nf_hook_slow`, `nf_register_net_hook` ★★★ — short and complete.
4. `net/netfilter/nf_conntrack_core.c`: `nf_conntrack_in`, `resolve_normal_ct`, `__nf_conntrack_confirm` ★★★
5. `net/netfilter/nf_conntrack_proto_tcp.c`: the state table ★★★ — a readable, complete TCP state machine.
6. `net/netfilter/nf_tables_core.c`: `nft_do_chain` ★★★ — **the VM, in ~200 lines.**
7. `net/netfilter/nft_cmp.c`, `nft_payload.c`, `nft_lookup.c` — the expressions; each is tiny.
8. `net/sched/sch_generic.c`: `__qdisc_run`, `qdisc_restart`, `sch_direct_xmit` ★★★
9. `include/net/codel.h` and `net/sched/sch_fq_codel.c` ★★★ — **read the CoDel paper first**, then this; the correspondence is exact.
10. `net/sched/sch_htb.c` — the classful reference; the borrowing logic is the interesting part.
11. `net/sched/cls_flower.c` — §T.8's offloadable classifier.

**Tools**

- `nft` ★★★ — `list ruleset`, `-a` for handles, `--debug=netlink` for bytecode, `monitor trace` ★★★
- `conntrack` ★★★ — `-L`, `-E`, `-S`, `-D`, `-F`
- `tc -s` ★★★ — **always use `-s`**; the statistics are the diagnostic
- `tc -j -s qdisc show` — JSON for scripting
- `netem` ★★★ — `delay`, `loss`, `reorder`, `duplicate`, `corrupt`, `rate`, `limit`; the experimental apparatus for every lab here
- `flent` ★★★ — the bufferbloat project's test harness; `flent rrul` is the standard bufferbloat test
- `bpftrace` on `nf_hook_slow`, `nft_do_chain`, `tcf_classify`, `__qdisc_run`
- `nft ... meta nftrace set 1` + `nft monitor trace` ★★★ — **the best netfilter debugging tool**
- `iptables -t raw -A PREROUTING -j TRACE` + `dmesg` — the legacy equivalent
- `bpftool prog`/`map` — for tc-bpf programs
- Network namespaces + `veth` ★★★ — every lab in this chapter runs on one machine

---

→ Next: [74-xdp.md](74-xdp.md)

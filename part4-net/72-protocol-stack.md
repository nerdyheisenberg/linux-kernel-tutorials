# Chapter 72 — Protocol stack: IP, TCP, UDP, and sockets

> **Goal:** Understand the protocols themselves as implemented, not as described in RFCs. Understand the socket layer as a generic dispatcher over protocol families, why `struct sock` is separate from `struct socket` and what each owns, IP's routing and fragmentation paths, the neighbour subsystem, TCP's state machine and the three intertwined control loops (flow, congestion, loss recovery) with their modern algorithms, why socket memory accounting exists and what happens when it runs out, UDP's deliberate minimalism and the GRO/GSO work that made it fast, and how a connection is established, maintained, and torn down in code. By the end you can read `net/ipv4/`, interpret `ss -i` and `nstat` fluently, and diagnose a latency or throughput problem from first principles.

---

## Theory & First Principles

### T.0 — Start here: why `close()` does not close the connection

```c
int fd = accept(...);
/* ... serve the request ... */
close(fd);       /* the file descriptor is gone. The CONNECTION is not. */
```

```bash
ss -tan | grep TIME-WAIT | wc -l     # often thousands, on a busy server
```

Those connections have no file descriptor, no process, and no socket in any process's table —
and yet the kernel is holding state for each of them, for 60 seconds (`2*MSL`). Why?

**Because a TCP connection is not an object your process owns. It is a distributed agreement
between two machines, and one of them has not finished.** Specifically: if the final ACK is
lost, the peer will retransmit its FIN, and something must be present to answer it.
Additionally, a delayed packet from this connection could otherwise be delivered to a *new*
connection reusing the same 4-tuple.

**This forces the stack's core structural decision: two objects, two lifetimes.**

```
  struct socket        <- the VFS/file view. Lives exactly as long as the fd.
       |                  This is what read/write/poll operate on (Ch. 53).
       v
  struct sock          <- the PROTOCOL view. Lives as long as the PROTOCOL
                          says it must -- which may be long after the fd is
                          gone (TIME-WAIT) or before it exists (a SYN queue
                          entry for a connection not yet accept()ed).
```

**Whenever two things have genuinely different lifetimes, they must be two objects.** You saw
exactly this in Ch. 29 §T.0 (the device's two lifetimes), Ch. 53 §T.0 (`inode` vs `dentry` vs
`file`), and Ch. 12 (refcounted lookup). Merging them produces either a leak or a
use-after-free; there is no third outcome.

**Now the second big idea: where does protocol processing actually run?** Not where you think:

```
  packet arrives -> IRQ -> NAPI poll (softirq context, no process!)
                             |
                             v
                    IP -> netfilter -> TCP receive
                             |
                     +-------+--------+
                     |                |
              socket is LOCKED    socket is free
              by a process?           |
                     |                v
                     v          process immediately,
              put on the        append to receive queue,
              BACKLOG queue     wake the waiter
                     |
              processed when the process releases the lock
```

**Most TCP processing happens in softirq context, on whatever CPU the NIC interrupt landed
on, with no process context at all** (Ch. 17 §T.0). That fact explains a great deal: why you
cannot sleep there, why `sk_backlog` exists, why RPS/RFS (steering packets to the CPU where
the application runs) gives large wins, and why a single busy socket can be a scalability
bottleneck.

**Third, the thing that makes TCP hard is not the state machine — it is that every buffer is
attacker-controlled and unbounded.** The receive path must work when:

- packets arrive out of order (hold them in an out-of-order queue — which is itself a DoS
  surface, and has been exploited),
- the peer sends nothing and never closes (keepalives, timeouts),
- the peer opens a million half-connections (SYN cookies: **encode the connection state in
  the sequence number you send back, and keep no state at all** — an elegant and completely
  general defence worth remembering),
- or the peer advertises a tiny window forever (the zero-window probe).

**Every one of those is a rule that exists because someone exploited its absence.** Reading
the TCP receive path as a security artefact rather than a protocol implementation is the
right frame for this chapter.

```bash
ss -tani                                  # per-connection cwnd, rtt, retrans
nstat -az | grep -Ei 'TcpExt|Retrans|Drop'
cat /proc/net/sockstat
sudo /usr/share/bcc/tools/tcpretrans
sudo /usr/share/bcc/tools/tcplife          # connection lifetimes
```

---

### T.1 Two objects, two lifetimes

Every socket is really two structures:

```c
struct socket {                  /* the VFS-facing object */
	socket_state		state;
	short			type;
	unsigned long		flags;
	struct file		*file;
	struct sock		*sk;         /* -> the protocol object */
	const struct proto_ops	*ops;
	struct socket_wq	wq;
};

struct sock {                    /* the protocol-facing object */
	struct sock_common	__sk_common;
	socket_lock_t		sk_lock;
	atomic_t		sk_drops;
	int			sk_rcvlowat;
	struct sk_buff_head	sk_error_queue;
	struct sk_buff_head	sk_receive_queue;
	struct {
		atomic_t	rmem_alloc;
		int		len;
		struct sk_buff	*head, *tail;
	} sk_backlog;
	int			sk_forward_alloc;
	...
	int			sk_rcvbuf, sk_sndbuf;
	int			sk_wmem_queued;
	refcount_t		sk_wmem_alloc;
	struct sk_buff_head	sk_write_queue;
	struct proto		*sk_prot;
	void			(*sk_state_change)(struct sock *sk);
	void			(*sk_data_ready)(struct sock *sk);
	void			(*sk_write_space)(struct sock *sk);
	void			(*sk_error_report)(struct sock *sk);
	void			(*sk_destruct)(struct sock *sk);
	...
};
```

The split exists because **their lifetimes differ**. A `struct socket` dies when the last file descriptor closes. A `struct sock` may outlive it — a TCP connection in `TIME_WAIT` or one still draining its send queue has no file descriptor but must still exist, still respond to incoming segments, and still hold its port. Conversely, a `sock` exists before any `socket` does when it is an accept-queue child.

The operation vectors mirror the split:

| Vector | Level | Examples |
|---|---|---|
| `struct proto_ops` | socket layer | `bind`, `connect`, `accept`, `sendmsg`, `poll` |
| `struct proto` | protocol | `close`, `connect`, `sendmsg`, `recvmsg`, `hash`, `get_port` |

`inet_stream_ops` (proto_ops) calls into `tcp_prot` (proto). The first is generic to AF_INET stream sockets; the second is TCP-specific. A new protocol in the same family reuses the first.

**`sock_common` is the hashable prefix**, deliberately placed first so that lookup structures can be shared:

```c
struct sock_common {
	union {
		__addrpair	skc_addrpair;     /* daddr + rcv_saddr, one 64-bit compare */
		struct { __be32 skc_daddr; __be32 skc_rcv_saddr; };
	};
	union { unsigned int skc_hash; __u16 skc_u16hashes[2]; };
	union {
		__portpair	skc_portpair;     /* dport + num, one 32-bit compare */
		struct { __be16 skc_dport; __u16 skc_num; };
	};
	unsigned short		skc_family;
	volatile unsigned char	skc_state;
	unsigned char		skc_reuse:4, skc_reuseport:1, skc_ipv6only:1, skc_net_refcnt:1;
	int			skc_bound_dev_if;
	...
};
```

The `__addrpair` and `__portpair` unions let the 4-tuple comparison be **two loads and two compares** instead of four of each. At millions of lookups per second that is a real saving, and it is why those fields are adjacent and in that order.

### T.2 Socket locking: the backlog

A socket is touched from two contexts that cannot simply share a spinlock:

- **Process context**: `sendmsg`, `recvmsg`, `setsockopt` — may sleep.
- **Softirq context**: packet arrival — must not sleep.

Linux's solution is a two-level lock:

```c
void lock_sock_nested(struct sock *sk, int subclass)
{
	spin_lock_bh(&sk->sk_lock.slock);
	if (sock_owned_by_user_nocheck(sk))
		__lock_sock(sk);                /* sleep until the owner releases */
	sk->sk_lock.owned = 1;
	spin_unlock_bh(&sk->sk_lock.slock); /* the SPINLOCK is released here */
	mutex_acquire(&sk->sk_lock.dep_map, subclass, 0, _RET_IP_);
}
```

**The spinlock is held only briefly; the `owned` flag is the real lock.** So process context can sleep while "holding" the socket lock, because it holds only a flag.

Then what happens when a packet arrives for a socket a process owns?

```c
int tcp_v4_rcv(struct sk_buff *skb)
{
	...
	bh_lock_sock_nested(sk);
	...
	if (!sock_owned_by_user(sk)) {
		ret = tcp_v4_do_rcv(sk, skb);          /* process it now */
	} else {
		if (tcp_add_backlog(sk, skb, &drop_reason))  /* QUEUE it */
			goto discard_and_relse;
	}
	bh_unlock_sock(sk);
	...
}
```

It goes on the **backlog queue**, and the process drains it when it releases the lock:

```c
void release_sock(struct sock *sk)
{
	spin_lock_bh(&sk->sk_lock.slock);
	if (sk->sk_backlog.tail)
		__release_sock(sk);        /* process the backlog HERE */
	...
	sk->sk_lock.owned = 0;
	if (waitqueue_active(&sk->sk_lock.wq))
		wake_up(&sk->sk_lock.wq);
	spin_unlock_bh(&sk->sk_lock.slock);
}
```

Two consequences worth knowing:

**(a) The backlog is bounded** by `sk_rcvbuf`. If it overflows, packets are dropped — visible as `TCPBacklogDrop` in `nstat`. A process that holds a socket lock for a long time (a slow `recvmsg` copying to a swapped-out page, say) causes drops on a connection that is otherwise healthy.

**(b) Protocol processing can happen in process context.** `tcp_v4_do_rcv` may run from `release_sock`, on the application's CPU. This is sometimes good (cache locality with the reader) and sometimes bad (latency added to an unrelated syscall).

### T.3 The IP layer: routing as the central operation

```
Receive:  ip_rcv -> ip_rcv_finish -> [ROUTE] -> ip_local_deliver | ip_forward
Transmit: ip_queue_xmit -> [ROUTE cached in sk] -> ip_output -> ip_finish_output
```

The routing decision produces a `struct dst_entry`, which is the answer to "what do I do with this packet":

```c
struct dst_entry {
	struct net_device	*dev;
	struct  dst_ops		*ops;
	unsigned long		_metrics;
	unsigned long           expires;
	void			*__pad1;
	int			(*input)(struct sk_buff *);       /* what to do on RX */
	int			(*output)(struct net *net, struct sock *sk,
					  struct sk_buff *skb);   /* what to do on TX */
	unsigned short		flags;
	short			obsolete;
	unsigned short		header_len;
	unsigned short		trailer_len;
	...
};
```

`dst->input` and `dst->output` are **function pointers chosen by the routing decision**. A local packet gets `ip_local_deliver`; a forwarded one gets `ip_forward`; a blackholed one gets `dst_discard`. The routing lookup is thus not "find the next hop" but "decide what function processes this packet" — a small but important reframing, and the reason the same structure serves both directions.

The routing table is a **LC-trie** (Level-Compressed trie, `fib_trie.c`), which gives longest-prefix-match in O(prefix length) with good cache behaviour. It replaced a hash-based scheme in 2.6.13 and scales to hundreds of thousands of routes.

Caching: Linux once had a per-flow route cache, removed in 3.6 because it was a DoS vector (an attacker could fill it) and because `fib_trie` lookups became fast enough. What remains is:

- **Per-socket `dst` caching** (`sk->sk_dst_cache`) — a connected socket looks up once.
- **`fnhe` (FIB nexthop exceptions)** — for PMTU and redirects, which are per-destination.

**Fragmentation** is the other IP-layer concern, and the modern position is that it should not happen:

| Direction | Mechanism |
|---|---|
| Transmit | **PMTU discovery**: set DF, learn the path MTU from ICMP, never fragment |
| Receive | reassembly, with a memory budget and a timeout |

Reassembly is a security-sensitive operation: an attacker can send first fragments of many packets and never complete them, consuming memory. Hence `net.ipv4.ipfrag_high_thresh`, `ipfrag_time`, and the rehashing added after the FragmentSmack DoS (2018). **IPv6 forbids router fragmentation entirely**, which is the right design.

PMTU discovery's failure mode — "PMTU black holes", where ICMP is filtered and packets silently vanish — is common enough that `net.ipv4.tcp_mtu_probing` exists to detect it by probing.

### T.4 Neighbour resolution

Before a packet can be transmitted on a link, the next hop's link-layer address is needed. The **neighbour subsystem** (`net/core/neighbour.c`) is protocol-independent; ARP (IPv4) and NDISC (IPv6) are its users.

The state machine is the interesting part:

```
        NONE
          |
       INCOMPLETE  (probe sent, no reply yet; packets queued)
          |
        REACHABLE  (confirmed; valid for base_reachable_time)
          |
         STALE     (timer expired; still usable, will confirm on next use)
        /     \
    DELAY     PROBE   (re-confirming)
       |         |
    REACHABLE  FAILED
```

The key design decision: **STALE entries are still used.** A stale entry sends the packet immediately and triggers a background confirmation, rather than stalling traffic to re-probe. This is why ARP timeouts do not produce latency spikes in the common case.

**Confirmation without probing**: if TCP receives an ACK, it knows the neighbour is reachable and calls `dst_confirm_neigh()`. So a busy connection never needs an ARP probe at all — the higher layer's evidence is used. This is a small, elegant cross-layer optimisation.

The garbage-collection thresholds matter operationally:

```sh
net.ipv4.neigh.default.gc_thresh1 = 128    # below this, never GC
net.ipv4.neigh.default.gc_thresh2 = 512    # soft limit; GC after 5s
net.ipv4.neigh.default.gc_thresh3 = 1024   # hard limit; GC immediately
```

A host on a large flat network (a /16 with thousands of hosts) will exceed `gc_thresh3` and start thrashing, producing intermittent "neighbour table overflow" messages and packet loss. Raising the thresholds is a standard large-network tuning step.

### T.5 TCP: three control loops

TCP's complexity comes from three concurrent control loops with different purposes, which are frequently conflated:

| Loop | Question | Signal | Mechanism |
|---|---|---|---|
| **Flow control** | can the *receiver* accept more? | the advertised window | `rwnd` |
| **Congestion control** | can the *network* carry more? | loss, delay, ECN | `cwnd` |
| **Loss recovery** | what needs retransmitting? | duplicate ACKs, SACK, timers | fast retransmit, RACK, RTO |

The amount in flight is bounded by `min(cwnd, rwnd)`. Confusing the two produces wrong diagnoses: a receiver-limited connection and a congestion-limited one look similar in throughput but have completely different fixes.

**Flow control** is straightforward but has one subtlety: **window auto-tuning**. Linux grows `sk_rcvbuf` dynamically based on the measured bandwidth-delay product:

```sh
net.ipv4.tcp_rmem = 4096 131072 6291456    # min default max
net.ipv4.tcp_wmem = 4096 16384 4194304
net.ipv4.tcp_moderate_rcvbuf = 1
```

Setting `SO_RCVBUF` explicitly **disables** auto-tuning, which is almost always a mistake — applications that "tune" their buffers usually make throughput worse on high-BDP paths. The default max (6 MB) is the real limit, and raising *that* is the correct action for long fat networks.

**Congestion control** is pluggable:

```c
struct tcp_congestion_ops {
	u32 (*ssthresh)(struct sock *sk);
	void (*cong_avoid)(struct sock *sk, u32 ack, u32 acked);
	void (*set_state)(struct sock *sk, u8 new_state);
	void (*cwnd_event)(struct sock *sk, enum tcp_ca_event ev);
	void (*in_ack_event)(struct sock *sk, u32 flags);
	void (*pkts_acked)(struct sock *sk, const struct ack_sample *sample);
	u32 (*min_tso_segs)(struct sock *sk);
	void (*cong_control)(struct sock *sk, u32 ack, int flag,
			     const struct rate_sample *rs);
	u32 (*undo_cwnd)(struct sock *sk);
	u32 (*sndbuf_expand)(struct sock *sk);
	...
	char name[TCP_CA_NAME_MAX];
	struct module *owner;
};
```

The algorithms, and what each assumes:

| Algorithm | Signal | Assumption |
|---|---|---|
| **Reno** | loss | loss means congestion |
| **CUBIC** (default) | loss | same, but a cubic growth function tuned for high BDP |
| **BBR** | **bandwidth and RTT** | **loss does not mean congestion**; model the bottleneck |
| **DCTCP** | **ECN marks** | the network tells you, precisely; datacentre only |
| **Vegas** | delay | RTT increase means queueing |

**BBR is the significant departure.** Loss-based algorithms fill the bottleneck queue until it overflows, which is how they detect congestion — so they *cause* bufferbloat by design. BBR instead estimates the bottleneck bandwidth and the minimum RTT, and paces at that rate, keeping queues nearly empty. On paths with deep buffers or non-congestive loss (wireless), it is dramatically better. It is also more aggressive against loss-based flows sharing a bottleneck, which is a real and debated fairness concern.

The `cong_control` callback (as opposed to `cong_avoid`) gives an algorithm full control of `cwnd` and pacing rate, which BBR needs. Its addition was what made BBR expressible.

**Loss recovery** has changed substantially and the modern form is worth knowing:

| Mechanism | What |
|---|---|
| Fast retransmit | 3 duplicate ACKs → retransmit, do not wait for RTO |
| **SACK** | the receiver reports exactly which ranges arrived |
| **FACK/RACK** | **time-based**: a segment is lost if a later one was ACKed more than RTT/4 ago |
| **TLP** (Tail Loss Probe) | send a probe before the RTO fires, to avoid an RTO on tail loss |
| **F-RTO** | detect and undo spurious RTOs |
| **DSACK** | the receiver reports duplicates, letting the sender detect spurious retransmits |

**RACK-TLP replaced the dupack-counting heuristics** and is now the default. The insight: with SACK, you know the exact set of delivered segments and when each was ACKed; a segment sent before a segment that was ACKed, and not itself ACKed, is probably lost. Time-based reasoning is more robust than counting, especially with reordering.

The RTO calculation is Jacobson's, unchanged since 1988:

```c
	/* srtt = 7/8 srtt + 1/8 rtt;  mdev = 3/4 mdev + 1/4 |rtt - srtt| */
	rto = srtt + max(4 * mdev, TCP_ATO_MIN)
	rto = clamp(rto, TCP_RTO_MIN /* 200ms */, TCP_RTO_MAX /* 120s */)
```

`TCP_RTO_MIN` of 200 ms is a datacentre problem: on a 100 µs RTT network, an RTO costs 2000× the RTT. `tcp_rto_min_us` (per-route, via `ip route ... rto_min`) exists to fix this.

### T.6 The TCP state machine, and where connections go to die

```
                    CLOSED
                   /      \
         (connect)/        \(listen)
                 /          \
            SYN_SENT       LISTEN
                 |            |
                 |     (SYN)  |
                 |        SYN_RECV
                  \          /
                   ESTABLISHED
                   /         \
          (close) /           \ (FIN received)
                 /             \
           FIN_WAIT_1      CLOSE_WAIT
             /     \            |
    FIN_WAIT_2   CLOSING    LAST_ACK
          |          |          |
      TIME_WAIT <----+          |
          |                     |
        CLOSED <----------------+
```

Two states cause most operational confusion:

**`TIME_WAIT`** (2×MSL, 60 s in Linux) exists for two reasons: to absorb delayed duplicates from the old connection so they do not corrupt a new one with the same 4-tuple, and to ensure the final ACK is retransmittable. It is held by the *active closer*.

A server that closes connections accumulates `TIME_WAIT` sockets. The traditional responses are mostly wrong:

| Response | Verdict |
|---|---|
| `SO_REUSEADDR` | lets you *bind*; does not reduce TIME_WAIT |
| `tcp_tw_reuse` | **safe**: reuses TIME_WAIT for *outgoing* connections, guarded by timestamps |
| `tcp_tw_recycle` | **removed in 4.12** — it broke NAT badly and should never have existed |
| `SO_LINGER` with timeout 0 | sends RST instead of FIN; avoids TIME_WAIT by **discarding unsent data** |
| Have the *client* close first | correct, if you control the protocol |
| More ephemeral ports / more IPs | correct, and usually sufficient |

**`CLOSE_WAIT`** is different and is almost always an application bug: it means the peer sent FIN and **your application has not called `close()`**. Accumulating `CLOSE_WAIT` sockets is a file-descriptor leak with a specific cause.

**SYN flood handling** uses `SYN cookies`: when the SYN queue overflows, encode the connection state into the ISN and validate it in the returning ACK, so no state is held until the handshake completes. The cost is losing TCP options that cannot be encoded (window scale is partially preserved via a timestamp trick; SACK permitted is lost in some configurations). `net.ipv4.tcp_syncookies=1` (the default) means "use them only when the queue overflows", which is correct.

The two-queue model matters for `listen()` tuning:

| Queue | Holds | Sized by |
|---|---|---|
| **SYN queue** | half-open connections | `tcp_max_syn_backlog` |
| **accept queue** | established, waiting for `accept()` | `listen()`'s backlog, capped by `somaxconn` |

Overflow of the second (`ListenOverflows`, `ListenDrops` in `nstat`) means the application is not calling `accept()` fast enough — a different problem from a SYN flood.

### T.7 Socket memory accounting

Every socket has budgets, and understanding them is how you diagnose "the connection stalled for no reason."

```c
	sk->sk_rcvbuf          /* receive budget */
	sk->sk_sndbuf          /* send budget */
	sk->sk_wmem_queued     /* bytes queued for transmission */
	sk->sk_forward_alloc   /* pre-charged memory, to avoid per-skb accounting */
	atomic_read(&sk->sk_rmem_alloc)   /* bytes in the receive queue */
```

Plus a **global** budget:

```sh
net.ipv4.tcp_mem = 380697 507598 761394    # in PAGES: low, pressure, high
net.ipv4.udp_mem = 761394 1015193 1522788
```

The three values are a hysteresis band:

| Below `low` | no pressure; allocate freely |
| Between `low` and `pressure` | enter pressure mode; shrink buffers |
| Above `high` | **refuse allocations**; drop packets |

`tcp_memory_pressure` being set is visible in `nstat` as `TCPMemoryPressures` and means the system-wide TCP memory budget is exhausted — every connection's buffers shrink. On a machine with many connections this is the usual cause of unexplained throughput collapse.

`sk_forward_alloc` is a small optimisation with a large effect: memory is charged in `SK_MEM_QUANTUM` (page-sized) chunks rather than per skb, so the atomic operations on the global counter happen rarely.

**Receive-side collapsing** is the last-resort mechanism: when the receive queue is full of small skbs with high `truesize` overhead, `tcp_collapse()` merges them into fewer, denser skbs. It shows as `TCPRcvCollapsed` and it is expensive — a sign that the sender is sending small segments or that GRO is not working.

**`truesize` versus `len`** is the subtlety: accounting uses `truesize` (the actual memory consumed, including the skb and any page fragments), not the payload length. A 100-byte packet in a 2 KB page fragment charges ~2 KB. An attacker sending many small packets can exhaust a receive buffer with very little data — which is why `tcp_rmem` and `truesize` both exist and why the ratio matters.

### T.8 UDP: deliberate minimalism, and the work to make it fast

UDP is 8 bytes of header and no state. That was the point: applications that need something other than TCP's semantics build on it.

The receive path is short:

```
udp_rcv -> __udp4_lib_rcv
  -> checksum validation
  -> socket lookup (hash on dest port, or the 4-tuple for connected sockets)
  -> __udp_queue_rcv_skb -> sock_queue_rcv_skb
  -> sk_data_ready -> wake the reader
```

The problems, and their solutions:

**(a) Socket lookup cost.** UDP's hash was originally on the destination port only, so a server with one socket receiving from many peers hashed everything to one bucket. `udp_table` now has a second hash on the 4-tuple (`udp_table.hash2`) for connected sockets, making lookup O(1).

**(b) Per-packet overhead.** UDP had none of TCP's aggregation, so a QUIC server doing 1.2 million small packets per second spent everything in the stack. **UDP GRO and GSO** (`UDP_SEGMENT`, `UDP_GRO` socket options) fixed this: the kernel can coalesce received UDP datagrams of equal size into one skb and segment a large buffer into many datagrams on send.

This was driven almost entirely by QUIC. `sendmsg` with `UDP_SEGMENT` set writes 64 KB and the kernel (or the NIC) splits it into MTU-sized datagrams — the same 40× reduction in per-packet work that TSO gives TCP.

**(c) `recvmmsg`/`sendmmsg`** amortise the syscall over many datagrams.

**(d) `SO_REUSEPORT`** lets many sockets bind the same port, with the kernel hashing flows across them — one socket per core, no lock contention. Plus `SO_REUSEPORT` BPF programs, which let you choose the socket programmatically (Ch. 75).

The checksum rules are a perennial source of bugs:

| | IPv4 | IPv6 |
|---|---|---|
| Checksum optional? | **yes** (0 means "not computed") | **no** |
| Zero checksum means | not computed | **invalid** (0xFFFF is used for a true zero) |

And `sk_buff`'s `ip_summed` states interact with this:

| State | Meaning |
|---|---|
| `CHECKSUM_NONE` | not verified; software must check |
| `CHECKSUM_UNNECESSARY` | hardware verified it |
| `CHECKSUM_COMPLETE` | hardware computed a sum over the whole packet; `skb->csum` holds it |
| `CHECKSUM_PARTIAL` | (TX) hardware will compute it; `csum_start`/`csum_offset` say where |

`CHECKSUM_COMPLETE` is the useful one: a single checksum over everything can be adjusted arithmetically as headers are stripped, so each layer's checksum is derivable without re-reading the data.

### T.9 The data path from `write()` to the wire

```
send(fd, buf, len)
 └─ sock_sendmsg -> inet_sendmsg -> tcp_sendmsg
     ├─ lock_sock()
     ├─ for each chunk:
     │   ├─ get an skb from sk_write_queue (or allocate one)
     │   ├─ sk_stream_wait_memory() if sk_wmem_queued >= sk_sndbuf
     │   │    -> the application BLOCKS here (or gets EAGAIN)
     │   ├─ copy data in (or reference user pages with MSG_ZEROCOPY)
     │   └─ tcp_push() if appropriate
     └─ release_sock()

tcp_push -> __tcp_push_pending_frames -> tcp_write_xmit
 ├─ for each skb while cwnd and rwnd allow:
 │   ├─ tcp_nagle_test() -- may hold back a small segment
 │   ├─ tcp_tso_segs() -- how many MSS to send as one GSO skb
 │   ├─ tcp_pacing check -- BBR paces; fq enforces
 │   └─ tcp_transmit_skb()
 │       ├─ CLONE the skb (the original stays in the retransmit queue)
 │       ├─ build the TCP header
 │       └─ ip_queue_xmit() -> ... -> dev_queue_xmit()
 └─ the original skb stays in sk_write_queue until ACKed
```

Three points:

**The retransmit queue is the write queue.** Segments stay in `sk_write_queue` (an rbtree in modern kernels, for fast SACK processing) until acknowledged. `tcp_transmit_skb` *clones* — so the data is shared and the transmitted copy can be freed independently.

**`sk_wmem_queued` versus `sk_wmem_alloc`.** The first counts bytes in the write queue (charged against `sk_sndbuf`); the second counts skbs currently in the transmit path (charged against the socket for lifetime purposes). A socket cannot be freed while `sk_wmem_alloc` is non-zero.

**Nagle and `TCP_NODELAY`.** Nagle's algorithm holds a small segment until the previous one is ACKed, to avoid a stream of tiny packets. It interacts badly with delayed ACKs (the receiver waits up to 40 ms to ACK; the sender waits for the ACK) producing the classic 40 ms stall. `TCP_NODELAY` disables it and is correct for request/response protocols. `TCP_CORK` is the opposite: hold everything until explicitly uncorked or 200 ms pass — useful for building a response from several writes.

**Pacing** deserves mention: rather than sending a window's worth of packets back to back, the sender spaces them at the estimated bottleneck rate. This reduces queueing and loss substantially. `fq` (the qdisc) or `tcp_pacing` (internal, via `sk->sk_pacing_rate`) implements it, and BBR requires it.

### T.10 Zero-copy and the modern socket API

Copying data between userspace and kernel is a significant cost at high rates. Four mechanisms:

| Mechanism | Direction | How |
|---|---|---|
| **`sendfile`/`splice`** | TX | move pages between a file and a socket without userspace |
| **`MSG_ZEROCOPY`** | TX | pin user pages, notify completion via the error queue |
| **`SO_ZEROCOPY` + `TCP_ZEROCOPY_RECEIVE`** | RX | `mmap` received data directly |
| **`AF_XDP`** | both | bypass the stack entirely (Ch. 74) |

`MSG_ZEROCOPY` has a subtlety worth understanding: the kernel pins the user's pages and transmits from them, so **the application must not modify the buffer until notified**. Notification arrives on the socket's error queue (`MSG_ERRQUEUE`), asynchronously. This makes it awkward to use and only worthwhile above ~10 KB per send, but for bulk transfer it eliminates a memcpy of the entire data volume.

`TCP_ZEROCOPY_RECEIVE` requires page-aligned, MSS-aligned data, which in practice means the sender must cooperate. It is used by a small number of high-throughput applications.

`io_uring` (Ch. 76) provides all of these with a better interface, and is increasingly the right answer for new code.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `net/socket.c` ★★★ | the syscall layer: `socket`, `bind`, `connect`, `sendmsg` |
| `net/core/sock.c` ★★★ | `struct sock` lifecycle, memory accounting, `setsockopt` |
| `net/ipv4/af_inet.c` ★★★ | `inet_stream_ops`, `inet_dgram_ops`, protocol registration |
| `net/ipv4/ip_input.c`, `ip_output.c`, `ip_forward.c` ★★★ | §T.3 |
| `net/ipv4/route.c`, `fib_trie.c` ★★★ | routing |
| `net/ipv4/ip_fragment.c` | reassembly |
| `net/core/neighbour.c`, `net/ipv4/arp.c` ★★★ | §T.4 |
| `net/ipv4/tcp.c` ★★★ | `tcp_sendmsg`, `tcp_recvmsg`, `setsockopt` |
| `net/ipv4/tcp_input.c` ★★★ | **the largest and most important file**: ACK processing, loss detection |
| `net/ipv4/tcp_output.c` ★★★ | `tcp_write_xmit`, `tcp_transmit_skb`, Nagle, TSO |
| `net/ipv4/tcp_ipv4.c` | `tcp_v4_rcv`, socket lookup |
| `net/ipv4/tcp_timer.c` | RTO, keepalive, TLP |
| `net/ipv4/tcp_cong.c`, `tcp_cubic.c`, `tcp_bbr.c` ★★★ | §T.5 |
| `net/ipv4/tcp_recovery.c` | RACK |
| `net/ipv4/tcp_minisocks.c` | TIME_WAIT, SYN_RECV |
| `net/ipv4/udp.c` ★★★ | §T.8 |
| `net/ipv4/udp_offload.c` | UDP GRO/GSO |
| `net/ipv4/inet_hashtables.c` ★★★ | socket lookup |
| `include/net/sock.h`, `tcp.h`, `inet_sock.h` ★★★ | |
| `Documentation/networking/ip-sysctl.rst` ★★★ | **every sysctl, documented** |

### 1.2 `tcp_sock`

```c
struct tcp_sock {
	struct inet_connection_sock	inet_conn;
	u16	tcp_header_len;
	u16	gso_segs;
	__be32	pred_flags;
	u64	bytes_received;
	u32	segs_in, data_segs_in;
	u32	rcv_nxt;          /* what we want next */
	u32	copied_seq;       /* what the app has read */
	u32	rcv_wup;          /* rcv_nxt at the last window update */
	u32	snd_nxt;          /* what we will send next */
	u32	snd_una;          /* first unacknowledged byte */
	u32	snd_sml;
	u32	rcv_tstamp, lsndtime;
	...
	u32	snd_wl1, snd_wnd, max_window;
	u32	mss_cache;        /* the current effective MSS */
	u32	window_clamp, rcv_ssthresh;
	...
	/* RTT estimation (T.5) */
	u32	srtt_us;          /* smoothed RTT, in 1/8 us */
	u32	mdev_us, mdev_max_us, rttvar_us, rtt_seq;
	struct minmax rtt_min;

	u32	packets_out;      /* in flight */
	u32	retrans_out;
	u32	max_packets_out;
	...
	u32	snd_ssthresh;
	u32	snd_cwnd;         /* THE congestion window */
	u32	snd_cwnd_cnt;
	u32	snd_cwnd_clamp;
	u32	snd_cwnd_used, snd_cwnd_stamp;
	u32	prior_cwnd;
	u32	prr_delivered, prr_out;
	u32	delivered, delivered_ce;
	u32	lost, app_limited;
	u64	first_tx_mstamp, delivered_mstamp;
	u32	rate_delivered, rate_interval_us;

	u32	rcv_wnd;
	u32	write_seq, notsent_lowat, pushed_seq;
	u32	lost_out, sacked_out;
	struct hrtimer	pacing_timer;
	struct hrtimer	compressed_ack_timer;
	struct sk_buff	*lost_skb_hint, *retransmit_skb_hint;
	struct rb_root	out_of_order_queue;
	...
	u64	bytes_sent, bytes_acked, bytes_retrans;
	...
	u32	rcv_ooopack;
	u32	rcv_rtt_last_tsecr;
	struct { u32 rtt_us, seq, time; } rcv_rtt_est;
	struct { u32 space, seq, time; } rcvq_space;   /* auto-tuning (T.5) */
	...
};
```

Reading this structure carefully is most of understanding Linux TCP. Note the grouping: receive-side sequence numbers, send-side sequence numbers, RTT estimation, congestion control state, rate measurement, and auto-tuning state are each contiguous — deliberate, for cache behaviour on the ACK-processing path.

### 1.3 `tcp_sendmsg`

```c
int tcp_sendmsg_locked(struct sock *sk, struct msghdr *msg, size_t size)
{
	struct tcp_sock *tp = tcp_sk(sk);
	struct sk_buff *skb;
	int flags, err, copied = 0;
	int mss_now = 0, size_goal, copied_syn = 0;
	...
	flags = msg->msg_flags;
	...
	if (unlikely(flags & MSG_FASTOPEN || inet_test_bit(DEFER_CONNECT, sk)) &&
	    !tp->repair) {
		err = tcp_sendmsg_fastopen(sk, msg, &copied_syn, size, uarg);
		...
	}
	...
	/* Wait for a connection if necessary */
	if (((1 << sk->sk_state) & ~(TCPF_ESTABLISHED | TCPF_CLOSE_WAIT)) &&
	    !tcp_passive_fastopen(sk)) {
		err = sk_stream_wait_connect(sk, &timeo);
		if (err != 0) goto do_error;
	}
	...
	mss_now = tcp_send_mss(sk, &size_goal, flags);

	while (msg_data_left(msg)) {
		int copy = 0;

		skb = tcp_write_queue_tail(sk);
		if (skb)
			copy = size_goal - skb->len;

		if (copy <= 0 || !tcp_skb_can_collapse_to(skb)) {
new_segment:
			/* T.9: BLOCK HERE if the send buffer is full */
			if (!sk_stream_memory_free(sk))
				goto wait_for_space;

			skb = tcp_stream_alloc_skb(sk, sk->sk_allocation,
						   first_skb);
			if (!skb) goto wait_for_space;
			...
			skb_entail(sk, skb);
			copy = size_goal;
			...
		}
		if (copy > msg_data_left(msg))
			copy = msg_data_left(msg);

		if (zc == 0) {
			/* The normal path: COPY into the skb */
			err = skb_copy_to_page_nocache(sk, &msg->msg_iter, skb,
						       pfrag->page, pfrag->offset, copy);
			...
		} else if (zc == MSG_ZEROCOPY) {
			/* T.10: reference the user's pages */
			err = skb_zerocopy_iter_stream(sk, skb, msg, copy, uarg);
			...
		}
		...
		tp->write_seq += copy;
		TCP_SKB_CB(skb)->end_seq += copy;
		...
		copied += copy;
		if (!msg_data_left(msg)) {
			if (unlikely(flags & MSG_EOR))
				TCP_SKB_CB(skb)->eor = 1;
			goto out;
		}
		...
		if (forced_push(tp)) {
			tcp_mark_push(tp, skb);
			__tcp_push_pending_frames(sk, mss_now, TCP_NAGLE_PUSH);
		} else if (skb == tcp_send_head(sk)) {
			tcp_push_one(sk, mss_now);
		}
		continue;

wait_for_space:
		set_bit(SOCK_NOSPACE, &sk->sk_socket->flags);
		tcp_remove_empty_skb(sk);
		if (copied)
			tcp_push(sk, flags & ~MSG_MORE, mss_now,
				 TCP_NAGLE_PUSH, size_goal);

		err = sk_stream_wait_memory(sk, &timeo);   /* SLEEP */
		if (err != 0) goto do_error;
		mss_now = tcp_send_mss(sk, &size_goal, flags);
	}
	...
}
```

`wait_for_space` is where a blocking `send()` blocks, and `sk_stream_memory_free()` is the predicate — §T.7's accounting, made visible.

### 1.4 `tcp_ack`: the heart of the input path

```c
static int tcp_ack(struct sock *sk, const struct sk_buff *skb, int flag)
{
	struct inet_connection_sock *icsk = inet_csk(sk);
	struct tcp_sock *tp = tcp_sk(sk);
	struct tcp_sacktag_state sack_state;
	struct rate_sample rs = { .prior_delivered = 0 };
	u32 prior_snd_una = tp->snd_una;
	u32 ack_seq = TCP_SKB_CB(skb)->seq;
	u32 ack = TCP_SKB_CB(skb)->ack_seq;
	int num_dupack = 0;
	...
	/* Old ACK? Ignore. */
	if (before(ack, prior_snd_una)) { ... goto old_ack; }

	/* ACK for data we never sent? */
	if (after(ack, tp->snd_nxt)) goto invalid_ack;
	...
	if (flag & FLAG_UPDATE_TS_RECENT)
		tcp_replace_ts_recent(tp, TCP_SKB_CB(skb)->seq);

	if ((flag & (FLAG_SLOWPATH | FLAG_SND_UNA_ADVANCED)) == FLAG_SND_UNA_ADVANCED) {
		/* THE FAST PATH: in-order ACK advancing snd_una */
		tcp_update_wl(tp, ack_seq);
		tcp_snd_una_update(tp, ack);
		flag |= FLAG_WIN_UPDATE;
		...
	} else {
		...
		if (TCP_SKB_CB(skb)->sacked)
			flag |= tcp_sacktag_write_queue(sk, skb, prior_snd_una,
							&sack_state);
		if (tcp_ecn_rcv_ecn_echo(tp, tcp_hdr(skb)))
			flag |= FLAG_ECE;
		...
	}
	...
	/* Remove acknowledged data from the write queue */
	flag |= tcp_clean_rtx_queue(sk, skb, prior_fack, prior_snd_una,
				    &sack_state, flag & FLAG_ECE);
	...
	tcp_rack_update_reo_wnd(sk, &rs);           /* T.5: RACK */

	if (tcp_ack_is_dubious(sk, flag)) {
		if (!(flag & (FLAG_SND_UNA_ADVANCED | FLAG_NOT_DUP))) {
			num_dupack = 1;
			if (!(flag & FLAG_DATA))
				num_dupack = max_t(u16, 1, skb_shinfo(skb)->gso_segs);
		}
		tcp_fastretrans_alert(sk, prior_snd_una, num_dupack, &flag,
				      &rexmit);           /* LOSS RECOVERY */
	}
	...
	delivered = tcp_newly_delivered(sk, prior_delivered, flag);
	lost = tp->lost - lost;
	rs.is_ack_delayed = !!(flag & FLAG_ACK_MAYBE_DELAYED);
	tcp_rate_gen(sk, delivered, lost, is_sack_reneg, sack_state.rate);
	tcp_cong_control(sk, ack, delivered, flag, sack_state.rate);  /* CC */
	tcp_xmit_recovery(sk, rexmit);
	return 1;
	...
}
```

**Every one of §T.5's three loops appears here**: the window update (flow), `tcp_cong_control` (congestion), `tcp_fastretrans_alert` and RACK (loss recovery). Reading this function carefully is the single most valuable thing in the chapter.

### 1.5 Congestion control: CUBIC and BBR

```c
/* net/ipv4/tcp_cubic.c -- the default */
static void bictcp_cong_avoid(struct sock *sk, u32 ack, u32 acked)
{
	struct tcp_sock *tp = tcp_sk(sk);
	struct bictcp *ca = inet_csk_ca(sk);

	if (!tcp_is_cwnd_limited(sk))
		return;

	if (tcp_in_slow_start(tp)) {
		acked = tcp_slow_start(tp, acked);
		if (!acked) return;
	}
	bictcp_update(ca, tcp_snd_cwnd(tp), acked);
	tcp_cong_avoid_ai(tp, ca->cnt, acked);
}

static u32 bictcp_recalc_ssthresh(struct sock *sk)
{
	const struct tcp_sock *tp = tcp_sk(sk);
	struct bictcp *ca = inet_csk_ca(sk);

	ca->epoch_start = 0;

	/* Wmax and fast convergence */
	if (tcp_snd_cwnd(tp) < ca->last_max_cwnd && fast_convergence)
		ca->last_max_cwnd = (tcp_snd_cwnd(tp) * (BICTCP_BETA_SCALE + beta))
			/ (2 * BICTCP_BETA_SCALE);
	else
		ca->last_max_cwnd = tcp_snd_cwnd(tp);

	/* beta = 717/1024 = 0.7 -- less aggressive than Reno's 0.5 */
	return max((tcp_snd_cwnd(tp) * beta) / BICTCP_BETA_SCALE, 2U);
}
```

BBR's different shape:

```c
/* net/ipv4/tcp_bbr.c */
static void bbr_main(struct sock *sk, const struct rate_sample *rs)
{
	struct bbr *bbr = inet_csk_ca(sk);
	u32 bw;

	bbr_update_model(sk, rs);       /* estimate bandwidth and min RTT */

	bw = bbr_bw(sk);
	bbr_set_pacing_rate(sk, bw, bbr->pacing_gain);   /* PACE, not burst */
	bbr_set_cwnd(sk, rs, rs->acked_sacked, bw, bbr->cwnd_gain);
}

static void bbr_update_bw(struct sock *sk, const struct rate_sample *rs)
{
	struct tcp_sock *tp = tcp_sk(sk);
	struct bbr *bbr = inet_csk_ca(sk);
	u64 bw;
	...
	/* Delivery rate = delivered bytes / elapsed time */
	bw = div64_long((u64)rs->delivered * BW_UNIT, rs->interval_us);
	...
	if (!rs->is_app_limited || bw >= bbr_max_bw(sk)) {
		/* A WINDOWED MAX filter -- take the max over 10 round trips */
		minmax_running_max(&bbr->bw, bbr_bw_rtts, bbr->rtt_cnt, bw);
	}
}
```

**BBR uses a windowed max of delivery rate and a windowed min of RTT.** Neither loss nor ECN appears. That is the departure of §T.5: congestion is modelled, not inferred from loss.

### 1.6 Socket lookup

```c
struct sock *__inet_lookup_established(struct net *net,
				       struct inet_hashinfo *hashinfo,
				       const __be32 saddr, const __be16 sport,
				       const __be32 daddr, const u16 hnum,
				       const int dif, const int sdif)
{
	INET_ADDR_COOKIE(acookie, saddr, daddr);
	const __portpair ports = INET_COMBINED_PORTS(sport, hnum);
	struct sock *sk;
	const struct hlist_nulls_node *node;
	unsigned int hash = inet_ehashfn(net, daddr, hnum, saddr, sport);
	unsigned int slot = hash & hashinfo->ehash_mask;
	struct inet_ehash_bucket *head = &hashinfo->ehash[slot];

begin:
	sk_nulls_for_each_rcu(sk, node, &head->chain) {
		if (sk->sk_hash != hash)
			continue;
		if (likely(inet_match(net, sk, acookie, ports, dif, sdif))) {
			if (unlikely(!refcount_inc_not_zero(&sk->sk_refcnt)))
				goto out;
			if (unlikely(!inet_match(net, sk, acookie, ports, dif, sdif))) {
				sock_gen_put(sk);
				goto begin;
			}
			goto found;
		}
	}
	/* A NULLS list: if we ended on the wrong marker, we moved buckets. */
	if (get_nulls_value(node) != slot)
		goto begin;
out:
	sk = NULL;
found:
	return sk;
}
```

Two techniques worth noting:

**`hlist_nulls`** encodes the bucket index in the list terminator, so an RCU reader that walks off the end can detect that the entry it was following was moved to a different bucket. Without it, a reader could silently traverse into another bucket's chain.

**`INET_ADDR_COOKIE` and `INET_COMBINED_PORTS`** pack the addresses and ports so `inet_match` is two 64-bit comparisons, per §T.1.

### 1.7 Observability

| Where | What |
|---|---|
| `ss -tanip` ★★★ | sockets with process, timer, and internal state |
| `ss -i` ★★★ | **`cwnd`, `rtt`, `ssthresh`, `retrans`, `bytes_acked`, pacing rate** |
| `ss -m` ★★★ | socket memory: `skmem:(r,rb,t,tb,f,w,o,bl,d)` |
| `ss -e` | detailed flags, inode, uid |
| `nstat -az` ★★★ | **every protocol counter**; far better than `netstat -s` |
| `nstat` (no args) | deltas since last call |
| `/proc/net/{tcp,tcp6,udp,udp6}` | the raw tables |
| `/proc/net/sockstat` ★★★ | memory in use, orphans, TIME_WAIT count |
| `/proc/net/netstat`, `snmp` | the source of `nstat` |
| `sysctl net.ipv4.*` ★★★ | |
| `ip route get <addr>` ★★★ | the routing decision for one destination |
| `ip -s neigh`, `ip neigh show` | §T.4 |
| `trace-cmd record -e tcp:\* -e sock:\*` ★★★ | |
| `tcplife`, `tcpretrans`, `tcpconnlat`, `tcptop` (bcc) ★★★ | |
| `ss -i` + `tcp_probe`-style bpftrace | per-connection cwnd over time |

`ss -tinme` is the single most informative socket command:

```
ESTAB  0  0  10.0.0.1:22  10.0.0.2:51234
     skmem:(r0,rb131072,t0,tb87040,f4096,w0,o0,bl0,d0)
     cubic wscale:7,7 rto:204 rtt:3.75/1.5 ato:40 mss:1448 pmtu:1500
     rcvmss:536 advmss:1448 cwnd:10 bytes_sent:4521 bytes_acked:4522
     bytes_received:2108 segs_out:23 segs_in:21 data_segs_out:12
     data_segs_in:9 send 30.9Mbps lastsnd:44 lastrcv:44 lastack:44
     pacing_rate 61.8Mbps delivery_rate 12.1Mbps delivered:13 app_limited
     busy:36ms rcv_space:14480 rcv_ssthresh:64088 minrtt:3.06
```

Every field maps to a `tcp_sock` member from §1.2. `app_limited` is particularly useful: it means the connection was not sending as fast as it could, so throughput is application-limited, not network-limited.

---

## 2. Practice

### Lab 72.1 — Socket internals

```sh
sudo apt install -y iproute2 bpfcc-tools iperf3 netcat-openbsd socat

# Watch socket creation and destruction
sudo bpftrace -e '
kprobe:sk_alloc { @alloc = count(); }
kprobe:__sk_free, kprobe:sk_destruct { @free = count(); }
kprobe:inet_create { @inet = count(); }
kprobe:tcp_close { @tcp_close = count(); }
interval:s:5 { print(@alloc); print(@free); print(@inet); print(@tcp_close);
               clear(@alloc); clear(@free); clear(@inet); clear(@tcp_close); }' &

for i in $(seq 1 100); do nc -z 127.0.0.1 22 2>/dev/null; done
```

`struct sock` outliving `struct socket` — §T.1:

```sh
# Create a connection, close it, watch TIME_WAIT
(printf 'GET / HTTP/1.0\r\n\r\n'; sleep 0.2) | nc 127.0.0.1 22 > /dev/null 2>&1
ss -tan state time-wait | head -5
ss -tan | awk '{print $1}' | sort | uniq -c

cat /proc/net/sockstat
```

Socket memory — §T.7:

```sh
iperf3 -s -1 > /dev/null 2>&1 &
sleep 0.5
iperf3 -c 127.0.0.1 -t 10 > /dev/null 2>&1 &
sleep 2
ss -tim dst 127.0.0.1 | head -6
```

```
skmem:(r0,rb131072,t0,tb2626560,f1792,w1074944,o0,bl0,d0)
       ^  ^         ^  ^         ^     ^        ^  ^   ^
       |  rcvbuf    |  sndbuf    fwd   wmem_q   |  |   drops
       rmem_alloc   wmem_alloc                  |  backlog
                                           optmem
```

Watch the buffers auto-tune:

```sh
sysctl net.ipv4.tcp_rmem net.ipv4.tcp_wmem net.ipv4.tcp_moderate_rcvbuf

iperf3 -s -1 > /dev/null 2>&1 &
sleep 0.5; iperf3 -c 127.0.0.1 -t 15 > /dev/null 2>&1 &
for i in $(seq 1 12); do
  ss -tim dst 127.0.0.1 2>/dev/null | grep -oP 'rb\K\d+|tb\K\d+' | tr '\n' ' '
  echo
  sleep 1
done
wait
```

Disable auto-tuning and see it stop:

```sh
sudo sysctl -w net.ipv4.tcp_moderate_rcvbuf=0
# ... repeat ...
sudo sysctl -w net.ipv4.tcp_moderate_rcvbuf=1
```

The socket backlog — §T.2:

```sh
sudo bpftrace -e '
kprobe:tcp_add_backlog { @backlog = count(); }
kprobe:__release_sock  { @release_drain = count(); }
kprobe:sk_backlog_rcv  { @backlog_rcv = count(); }
interval:s:5 { print(@backlog); print(@backlog_rcv);
               clear(@backlog); clear(@backlog_rcv); }' &

iperf3 -s -1 > /dev/null 2>&1 &
sleep 0.5; iperf3 -c 127.0.0.1 -t 8 -P 4 > /dev/null 2>&1
nstat -az | grep -i backlog
```

---

### Lab 72.2 — The IP layer and routing

```sh
ip route show
ip route show table all | head -20
ip rule show

# The routing DECISION for one destination -- T.3
ip route get 8.8.8.8
ip route get 127.0.0.1
ip route get 8.8.8.8 from 10.0.0.1 2>/dev/null
```

Watch lookups:

```sh
sudo bpftrace -e '
kprobe:ip_route_output_key_hash { @output_lookup = count(); }
kprobe:ip_route_input_noref     { @input_lookup = count(); }
kprobe:fib_table_lookup         { @fib = count(); }
interval:s:5 { print(@output_lookup); print(@input_lookup); print(@fib);
               clear(@output_lookup); clear(@input_lookup); clear(@fib); }' &

for i in $(seq 1 200); do ping -c 1 -W 1 127.0.0.1 > /dev/null 2>&1; done
# Connected sockets cache the dst, so repeated sends do NOT re-lookup:
iperf3 -s -1 > /dev/null 2>&1 &
sleep 0.5; iperf3 -c 127.0.0.1 -t 5 > /dev/null 2>&1
```

Set up a routed topology:

```sh
sudo ip netns add r1
sudo ip netns add h1
sudo ip netns add h2

sudo ip link add v-h1 type veth peer name v-r1a
sudo ip link add v-h2 type veth peer name v-r1b
sudo ip link set v-h1 netns h1
sudo ip link set v-r1a netns r1
sudo ip link set v-h2 netns h2
sudo ip link set v-r1b netns r1

sudo ip netns exec h1 ip addr add 10.1.0.2/24 dev v-h1
sudo ip netns exec h1 ip link set v-h1 up
sudo ip netns exec h1 ip link set lo up
sudo ip netns exec h1 ip route add default via 10.1.0.1

sudo ip netns exec h2 ip addr add 10.2.0.2/24 dev v-h2
sudo ip netns exec h2 ip link set v-h2 up
sudo ip netns exec h2 ip link set lo up
sudo ip netns exec h2 ip route add default via 10.2.0.1

sudo ip netns exec r1 ip addr add 10.1.0.1/24 dev v-r1a
sudo ip netns exec r1 ip addr add 10.2.0.1/24 dev v-r1b
sudo ip netns exec r1 ip link set v-r1a up
sudo ip netns exec r1 ip link set v-r1b up
sudo ip netns exec r1 ip link set lo up
sudo ip netns exec r1 sysctl -w net.ipv4.ip_forward=1

sudo ip netns exec h1 ping -c 3 10.2.0.2
```

Watch forwarding:

```sh
sudo ip netns exec r1 bpftrace -e '
kprobe:ip_forward       { @forward = count(); }
kprobe:ip_local_deliver { @local = count(); }
kprobe:ip_rcv           { @rcv = count(); }
interval:s:5 { print(@rcv); print(@forward); print(@local);
               clear(@rcv); clear(@forward); clear(@local); }' &
sudo ip netns exec h1 ping -c 10 -i 0.2 10.2.0.2 > /dev/null
```

PMTU and fragmentation — §T.3:

```sh
sudo ip netns exec r1 ip link set v-r1b mtu 1000
sudo ip netns exec h1 ping -c 2 -s 1400 -M do 10.2.0.2 2>&1 | tail -3
# "Frag needed and DF set" -- PMTU discovery

sudo ip netns exec h1 ip route get 10.2.0.2
sudo ip netns exec h1 ping -c 2 -s 1400 10.2.0.2 2>&1 | tail -2  # fragments

sudo ip netns exec h1 bpftrace -e '
kprobe:ip_fragment      { @frag = count(); }
kprobe:ip_defrag        { @defrag = count(); }
kprobe:ip_do_fragment   { @do_frag = count(); }
interval:s:5 { print(@frag); print(@do_frag); print(@defrag);
               clear(@frag); clear(@do_frag); clear(@defrag); }' &
sudo ip netns exec h1 ping -c 10 -s 2000 10.2.0.2 > /dev/null 2>&1

sudo ip netns exec h1 nstat -az | grep -i frag
```

```sh
sudo ip netns exec r1 ip link set v-r1b mtu 1500
```

---

### Lab 72.3 — Neighbour resolution

```sh
ip neigh show
ip -s neigh show

sysctl net.ipv4.neigh.default.gc_thresh1 \
       net.ipv4.neigh.default.gc_thresh2 \
       net.ipv4.neigh.default.gc_thresh3
sysctl net.ipv4.neigh.default.base_reachable_time_ms
```

Watch the state machine — §T.4:

```sh
sudo ip netns exec h1 ip neigh flush all
sudo ip netns exec h1 ip neigh show

sudo ip netns exec h1 bpftrace -e '
kprobe:neigh_resolve_output  { @resolve = count(); }
kprobe:__neigh_event_send    { @event_send = count(); }
kprobe:neigh_timer_handler   { @timer = count(); }
kprobe:arp_send_dst          { @arp_sent = count(); }
interval:s:3 { print(@resolve); print(@event_send); print(@arp_sent);
               clear(@resolve); clear(@event_send); clear(@arp_sent); }' &

sudo ip netns exec h1 ping -c 1 10.1.0.1 > /dev/null
sudo ip netns exec h1 ip neigh show
sleep 2
sudo ip netns exec h1 ip neigh show      # REACHABLE
```

Watch it go STALE and be used anyway:

```sh
sudo ip netns exec h1 sysctl -w net.ipv4.neigh.v-h1.base_reachable_time_ms=3000
sudo ip netns exec h1 ping -c 1 10.1.0.1 > /dev/null
for i in $(seq 1 8); do
  echo -n "t=$i: "
  sudo ip netns exec h1 ip neigh show 10.1.0.1
  sleep 1
done
# Then use it while STALE -- no stall
time sudo ip netns exec h1 ping -c 1 10.1.0.1 | tail -2
sudo ip netns exec h1 ip neigh show 10.1.0.1     # DELAY, then REACHABLE
```

Confirmation from TCP — §T.4:

```sh
sudo ip netns exec h1 bpftrace -e '
kprobe:neigh_update { @update = count(); }
kprobe:dst_confirm_neigh { @confirm = count(); }
interval:s:5 { print(@update); print(@confirm); clear(@update); clear(@confirm); }' &

sudo ip netns exec h2 iperf3 -s -D 2>/dev/null
sleep 1
sudo ip netns exec h1 iperf3 -c 10.2.0.2 -t 8 > /dev/null 2>&1
# Many confirmations, no ARP probes: TCP's ACKs prove reachability.
```

Table overflow:

```sh
sudo sysctl -w net.ipv4.neigh.default.gc_thresh3=64
# On a large network this would now thrash. Watch:
dmesg | grep -i 'neighbour table overflow' | tail -3
sudo sysctl -w net.ipv4.neigh.default.gc_thresh3=1024
```

---

### Lab 72.4 — TCP's three control loops

```sh
sysctl net.ipv4.tcp_congestion_control
sysctl net.ipv4.tcp_available_congestion_control
ls /lib/modules/$(uname -r)/kernel/net/ipv4/tcp_*.ko 2>/dev/null | head
```

Observe cwnd, rwnd, and RTT together — §T.5:

```sh
sudo tc qdisc add dev v-r1a root netem delay 20ms 2>/dev/null || \
  sudo ip netns exec r1 tc qdisc add dev v-r1a root netem delay 20ms

sudo ip netns exec h2 iperf3 -s -D 2>/dev/null
sleep 1
sudo ip netns exec h1 iperf3 -c 10.2.0.2 -t 20 > /dev/null 2>&1 &

for i in $(seq 1 18); do
  sudo ip netns exec h1 ss -ti dst 10.2.0.2 2>/dev/null | \
    grep -oP 'cwnd:\K\d+|rtt:\K[\d.]+|ssthresh:\K\d+|retrans:\K\S+|bytes_acked:\K\d+' | \
    tr '\n' ' '
  echo
  sleep 1
done
wait
```

Plot cwnd over time:

```sh
sudo bpftrace -e '
kprobe:tcp_ack {
	$sk = (struct sock *)arg0;
	$tp = (struct tcp_sock *)$sk;
	@cwnd = hist($tp->snd_cwnd);
	@ssthresh = hist($tp->snd_ssthresh);
	@srtt_us = hist($tp->srtt_us >> 3);
}
interval:s:15 { print(@cwnd); print(@srtt_us); exit(); }' &

sudo ip netns exec h1 iperf3 -c 10.2.0.2 -t 12 > /dev/null 2>&1
wait
```

Compare congestion-control algorithms — §T.5:

```sh
sudo modprobe tcp_bbr 2>/dev/null
sudo ip netns exec r1 tc qdisc del dev v-r1a root 2>/dev/null
sudo ip netns exec r1 tc qdisc add dev v-r1a root netem delay 30ms loss 0.5% 2>/dev/null

for cc in cubic reno bbr; do
  sudo ip netns exec h1 sysctl -w net.ipv4.tcp_congestion_control=$cc > /dev/null 2>&1 || continue
  echo -n "$cc: "
  sudo ip netns exec h1 iperf3 -c 10.2.0.2 -t 10 2>/dev/null | \
    grep receiver | awk '{print $7, $8}'
done
sudo ip netns exec h1 sysctl -w net.ipv4.tcp_congestion_control=cubic > /dev/null
```

Under a deep buffer — where BBR shines:

```sh
sudo ip netns exec r1 tc qdisc del dev v-r1a root 2>/dev/null
sudo ip netns exec r1 tc qdisc add dev v-r1a root netem \
     rate 10mbit delay 20ms limit 1000 2>/dev/null   # a huge buffer

for cc in cubic bbr; do
  sudo ip netns exec h1 sysctl -w net.ipv4.tcp_congestion_control=$cc > /dev/null
  sudo ip netns exec h1 iperf3 -c 10.2.0.2 -t 12 > /dev/null 2>&1 &
  sleep 4
  echo -n "$cc latency under load: "
  sudo ip netns exec h1 ping -c 5 -i 0.2 10.2.0.2 2>/dev/null | tail -1
  wait
done
```

**CUBIC fills the buffer and latency explodes; BBR keeps it nearly empty.** That is §T.5's central difference.

Per-connection congestion control:

```sh
cat > setcc.c <<'EOF'
#define _GNU_SOURCE
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <unistd.h>
int main(int argc, char **argv) {
	int fd = socket(AF_INET, SOCK_STREAM, 0);
	char cc[16];
	socklen_t len = sizeof(cc);

	getsockopt(fd, IPPROTO_TCP, TCP_CONGESTION, cc, &len);
	printf("default: %s\n", cc);
	if (argc > 1) {
		if (setsockopt(fd, IPPROTO_TCP, TCP_CONGESTION, argv[1], strlen(argv[1])))
			perror("setsockopt");
		len = sizeof(cc);
		getsockopt(fd, IPPROTO_TCP, TCP_CONGESTION, cc, &len);
		printf("now:     %s\n", cc);
	}
	return 0;
}
EOF
gcc -o setcc setcc.c && ./setcc bbr
```

---

### Lab 72.5 — Loss recovery

```sh
sudo ip netns exec r1 tc qdisc del dev v-r1a root 2>/dev/null
sudo ip netns exec r1 tc qdisc add dev v-r1a root netem delay 10ms loss 2% 2>/dev/null

sudo ip netns exec h1 nstat -az > /tmp/before.nstat
sudo ip netns exec h1 iperf3 -c 10.2.0.2 -t 15 2>/dev/null | grep -E 'sender|receiver'
sudo ip netns exec h1 nstat -az > /tmp/after.nstat

diff <(sort /tmp/before.nstat) <(sort /tmp/after.nstat) 2>/dev/null | head
sudo ip netns exec h1 nstat 2>/dev/null | grep -iE 'retrans|loss|recovery|sack|dsack|timeout|spurious'
```

Watch the mechanisms — §T.5:

```sh
sudo bpftrace -e '
kprobe:tcp_retransmit_skb { @retrans = count(); }
kprobe:tcp_enter_recovery { @enter_recovery = count(); }
kprobe:tcp_enter_loss     { @rto_loss = count(); }
kprobe:tcp_rack_mark_lost { @rack = count(); }
kprobe:tcp_send_loss_probe { @tlp = count(); }
kprobe:tcp_sacktag_write_queue { @sack = count(); }
kprobe:tcp_try_undo_loss  { @undo = count(); }
interval:s:5 { print(@retrans); print(@enter_recovery); print(@rto_loss);
               print(@rack); print(@tlp); print(@sack); print(@undo);
               clear(@retrans); clear(@enter_recovery); clear(@rto_loss);
               clear(@rack); clear(@tlp); clear(@sack); clear(@undo); }' &

sudo ip netns exec h1 iperf3 -c 10.2.0.2 -t 12 > /dev/null 2>&1
```

Per-connection retransmits:

```sh
sudo tcpretrans-bpfcc 2>/dev/null | head -20 &
sudo ip netns exec h1 iperf3 -c 10.2.0.2 -t 8 > /dev/null 2>&1
```

RACK versus the old heuristics:

```sh
sysctl net.ipv4.tcp_recovery      # 1 = RACK enabled
sudo sysctl -w net.ipv4.tcp_recovery=0
echo -n "RACK off: "
sudo ip netns exec h1 iperf3 -c 10.2.0.2 -t 10 2>/dev/null | grep receiver | awk '{print $7,$8}'
sudo sysctl -w net.ipv4.tcp_recovery=1
echo -n "RACK on:  "
sudo ip netns exec h1 iperf3 -c 10.2.0.2 -t 10 2>/dev/null | grep receiver | awk '{print $7,$8}'
```

Reordering, which is what RACK handles well:

```sh
sudo ip netns exec r1 tc qdisc del dev v-r1a root 2>/dev/null
sudo ip netns exec r1 tc qdisc add dev v-r1a root netem \
     delay 10ms reorder 25% 50% 2>/dev/null

sudo ip netns exec h1 nstat -az | grep -i reorder
sudo ip netns exec h1 iperf3 -c 10.2.0.2 -t 10 2>/dev/null | grep receiver
sudo ip netns exec h1 nstat 2>/dev/null | grep -iE 'reorder|dsack|spurious'
```

RTO minimum — §T.5:

```sh
sysctl net.ipv4.tcp_rto_min_us 2>/dev/null
ip route show | head -1
# Per-route:
# sudo ip route change <prefix> rto_min 5ms

sudo bpftrace -e '
kprobe:tcp_retransmit_timer {
	$sk = (struct sock *)arg0;
	$icsk = (struct inet_connection_sock *)$sk;
	@rto_ms = hist($icsk->icsk_rto * 1000 / 250);
}
interval:s:12 { print(@rto_ms); exit(); }' &
sudo ip netns exec h1 iperf3 -c 10.2.0.2 -t 10 > /dev/null 2>&1
wait
```

---

### Lab 72.6 — TCP state machine and connection lifecycle

```sh
sudo ip netns exec r1 tc qdisc del dev v-r1a root 2>/dev/null

# Watch every state transition -- T.6
sudo trace-cmd record -e tcp:tcp_set_state -e sock:inet_sock_set_state -- \
  sh -c '(printf "GET /\r\n\r\n"; sleep 0.3) | nc 127.0.0.1 22 > /dev/null 2>&1'
sudo trace-cmd report | grep -oP 'oldstate=\K\w+|newstate=\K\w+' | paste - - | head -20
```

```sh
sudo bpftrace -e '
tracepoint:sock:inet_sock_set_state /args->protocol == 6/ {
	printf("%-10s %s -> %s  %d -> %d\n", comm,
	       args->oldstate == 1 ? "ESTAB" :
	       args->oldstate == 2 ? "SYN_SENT" :
	       args->oldstate == 3 ? "SYN_RECV" :
	       args->oldstate == 4 ? "FIN_WAIT1" :
	       args->oldstate == 5 ? "FIN_WAIT2" :
	       args->oldstate == 6 ? "TIME_WAIT" :
	       args->oldstate == 7 ? "CLOSE" :
	       args->oldstate == 8 ? "CLOSE_WAIT" :
	       args->oldstate == 9 ? "LAST_ACK" :
	       args->oldstate == 10 ? "LISTEN" : "CLOSING",
	       args->newstate == 1 ? "ESTAB" :
	       args->newstate == 2 ? "SYN_SENT" :
	       args->newstate == 3 ? "SYN_RECV" :
	       args->newstate == 4 ? "FIN_WAIT1" :
	       args->newstate == 5 ? "FIN_WAIT2" :
	       args->newstate == 6 ? "TIME_WAIT" :
	       args->newstate == 7 ? "CLOSE" :
	       args->newstate == 8 ? "CLOSE_WAIT" :
	       args->newstate == 9 ? "LAST_ACK" :
	       args->newstate == 10 ? "LISTEN" : "CLOSING",
	       args->sport, args->dport);
}' &
sleep 1
(printf 'GET /\r\n\r\n'; sleep 0.3) | nc 127.0.0.1 22 > /dev/null 2>&1
sleep 2
```

TIME_WAIT — §T.6:

```sh
# Generate many
for i in $(seq 1 300); do (nc -z 127.0.0.1 22 2>/dev/null &) ; done
sleep 2
ss -tan state time-wait | wc -l
cat /proc/net/sockstat | grep -i tcp

sysctl net.ipv4.tcp_fin_timeout
sysctl net.ipv4.tcp_tw_reuse
sysctl net.ipv4.tcp_max_tw_buckets
sysctl net.ipv4.ip_local_port_range

# tcp_tw_reuse: safe, guarded by timestamps
sudo sysctl -w net.ipv4.tcp_tw_reuse=1
nstat -az | grep -i timewait
```

CLOSE_WAIT — the application bug:

```c
cat > closewait.c <<'EOF'
/* Accept a connection, never close it. */
#define _GNU_SOURCE
#include <arpa/inet.h>
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <unistd.h>
int main(void) {
	int s = socket(AF_INET, SOCK_STREAM, 0), c;
	struct sockaddr_in a = { .sin_family = AF_INET,
				 .sin_port = htons(19999),
				 .sin_addr.s_addr = htonl(INADDR_LOOPBACK) };
	int one = 1;
	setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &one, sizeof(one));
	bind(s, (struct sockaddr *)&a, sizeof(a));
	listen(s, 10);
	while ((c = accept(s, NULL, NULL)) >= 0) {
		char buf[16];
		read(c, buf, sizeof(buf));
		/* DELIBERATELY never close(c) */
		printf("accepted fd %d, leaking it\n", c);
	}
	return 0;
}
EOF
gcc -o closewait closewait.c && ./closewait &
sleep 1
for i in $(seq 1 10); do (echo hi | nc -q0 127.0.0.1 19999 &) ; done
sleep 2
ss -tan state close-wait | head
ls /proc/$(pgrep closewait)/fd | wc -l
kill %1 2>/dev/null
```

The two queues — §T.6:

```sh
sysctl net.core.somaxconn net.ipv4.tcp_max_syn_backlog
ss -ltn        # Recv-Q = current accept queue, Send-Q = max

# Overflow the accept queue
python3 -c "
import socket, time
s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(('127.0.0.1', 19998)); s.listen(2)   # backlog of 2
time.sleep(20)" &
sleep 1
for i in $(seq 1 20); do (nc -w1 127.0.0.1 19998 &) ; done 2>/dev/null
sleep 2
ss -ltn sport = :19998
nstat -az | grep -iE 'listen|syncookie'
wait 2>/dev/null
```

SYN cookies:

```sh
sysctl net.ipv4.tcp_syncookies
nstat -az | grep -i syncookie
```

---

### Lab 72.7 — UDP

```sh
# The receive path -- T.8
sudo bpftrace -e '
kprobe:udp_rcv          { @rcv = count(); }
kprobe:__udp_enqueue_schedule_skb { @enqueue = count(); }
kprobe:udp_queue_rcv_skb { @queue = count(); }
kretprobe:__udp4_lib_lookup /retval == 0/ { @no_socket = count(); }
interval:s:5 { print(@rcv); print(@enqueue); print(@no_socket);
               clear(@rcv); clear(@enqueue); clear(@no_socket); }' &

# Generate UDP traffic
iperf3 -s -1 > /dev/null 2>&1 &
sleep 0.5; iperf3 -c 127.0.0.1 -u -b 500M -t 8 > /dev/null 2>&1
```

Drops and their causes:

```sh
nstat -az | grep -iE 'udp'
cat /proc/net/udp | head -3
ss -uam | head -10

# Overflow the receive buffer
cat > udpslow.c <<'EOF'
#define _GNU_SOURCE
#include <arpa/inet.h>
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <unistd.h>
int main(void) {
	int s = socket(AF_INET, SOCK_DGRAM, 0);
	struct sockaddr_in a = { .sin_family = AF_INET, .sin_port = htons(19997),
				 .sin_addr.s_addr = htonl(INADDR_LOOPBACK) };
	int rcv = 4096;
	char buf[2048];

	setsockopt(s, SOL_SOCKET, SO_RCVBUF, &rcv, sizeof(rcv));
	bind(s, (struct sockaddr *)&a, sizeof(a));
	for (;;) { recv(s, buf, sizeof(buf), 0); usleep(10000); }  /* SLOW */
	return 0;
}
EOF
gcc -o udpslow udpslow.c && ./udpslow &
sleep 1
for i in $(seq 1 5000); do echo "packet $i"; done | \
  socat - UDP-SENDTO:127.0.0.1:19997 2>/dev/null
sleep 1
ss -uam sport = :19997
nstat -az | grep -i 'UdpRcvbufErrors\|UdpInErrors'
kill %1 2>/dev/null
```

UDP GRO and GSO — §T.8:

```sh
sudo bpftrace -e '
kprobe:udp_gro_receive { @udp_gro = count(); }
kprobe:__udp_gso_segment { @udp_gso = count(); }
kprobe:udp_rcv { @plain = count(); }
interval:s:8 { print(@udp_gro); print(@udp_gso); print(@plain);
               clear(@udp_gro); clear(@udp_gso); clear(@plain); }' &

iperf3 -s -1 > /dev/null 2>&1 &
sleep 0.5; iperf3 -c 127.0.0.1 -u -b 1G -l 1400 -t 6 > /dev/null 2>&1
```

Use `UDP_SEGMENT` directly:

```c
cat > udpgso.c <<'EOF'
#define _GNU_SOURCE
#include <arpa/inet.h>
#include <netinet/udp.h>
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <unistd.h>

#ifndef UDP_SEGMENT
#define UDP_SEGMENT 103
#endif

int main(void) {
	int s = socket(AF_INET, SOCK_DGRAM, 0);
	struct sockaddr_in a = { .sin_family = AF_INET, .sin_port = htons(19996),
				 .sin_addr.s_addr = htonl(INADDR_LOOPBACK) };
	char buf[60000];
	int seg = 1400;

	memset(buf, 'A', sizeof(buf));
	connect(s, (struct sockaddr *)&a, sizeof(a));

	/* T.8: ONE sendmsg -> many datagrams */
	if (setsockopt(s, SOL_UDP, UDP_SEGMENT, &seg, sizeof(seg)))
		perror("UDP_SEGMENT");

	for (int i = 0; i < 1000; i++)
		if (send(s, buf, sizeof(buf), 0) < 0) { perror("send"); break; }

	printf("sent 1000 x 60000 bytes as %d-byte segments\n", seg);
	return 0;
}
EOF
gcc -o udpgso udpgso.c

sudo bpftrace -e '
kprobe:__udp_gso_segment { @gso = count(); }
tracepoint:net:net_dev_start_xmit { @packets = count(); }
kprobe:udp_sendmsg { @sendmsg = count(); }
interval:s:6 { print(@sendmsg); print(@gso); print(@packets); exit(); }' &
./udpgso
wait
```

**1000 `sendmsg` calls producing ~43000 packets** — §T.8's 40× amortisation.

`SO_REUSEPORT`:

```sh
cat > reuseport.c <<'EOF'
#define _GNU_SOURCE
#include <arpa/inet.h>
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <unistd.h>
int main(int argc, char **argv) {
	int n = argc > 1 ? atoi(argv[1]) : 4, i;
	struct sockaddr_in a = { .sin_family = AF_INET, .sin_port = htons(19995),
				 .sin_addr.s_addr = htonl(INADDR_LOOPBACK) };
	for (i = 0; i < n; i++) {
		if (fork() == 0) {
			int s = socket(AF_INET, SOCK_DGRAM, 0), one = 1;
			char buf[2048];
			long count = 0;
			setsockopt(s, SOL_SOCKET, SO_REUSEPORT, &one, sizeof(one));
			if (bind(s, (struct sockaddr *)&a, sizeof(a))) { perror("bind"); return 1; }
			for (;;) {
				if (recv(s, buf, sizeof(buf), 0) > 0 && ++count % 1000 == 0)
					printf("worker %d: %ld packets\n", i, count);
			}
		}
	}
	sleep(20);
	return 0;
}
EOF
gcc -o reuseport reuseport.c && ./reuseport 4 &
sleep 1
ss -uln sport = :19995
for i in $(seq 1 10000); do echo x; done | socat - UDP-SENDTO:127.0.0.1:19995 2>/dev/null
sleep 2
kill %1 2>/dev/null
```

---

### Lab 72.8 — Diagnose a real problem

Build a scenario and diagnose it methodically.

```sh
sudo ip netns exec r1 tc qdisc del dev v-r1a root 2>/dev/null
sudo ip netns exec r1 tc qdisc add dev v-r1a root netem \
     rate 100mbit delay 50ms 2>/dev/null

sudo ip netns exec h2 iperf3 -s -D 2>/dev/null
sleep 1
echo "=== baseline ==="
sudo ip netns exec h1 iperf3 -c 10.2.0.2 -t 10 2>/dev/null | grep receiver
```

**Step 1: is it receiver-limited, sender-limited, or congestion-limited?**

```sh
sudo ip netns exec h1 iperf3 -c 10.2.0.2 -t 20 > /dev/null 2>&1 &
sleep 3
sudo ip netns exec h1 ss -tinme dst 10.2.0.2
```

Look for:

| Sign | Meaning |
|---|---|
| `cwnd` ≪ what BDP requires | congestion-limited |
| `rwnd` (from `ss -i` on the receiver) small | receiver-limited |
| `app_limited` present | **sender-limited**: the application is not writing fast enough |
| high `retrans` | loss |
| `rb` (rcvbuf) at its maximum | **buffer-limited** — raise `tcp_rmem` max |

```sh
wait
BDP=$(echo "100 * 1000000 * 0.05 / 8" | bc)
echo "BDP = $BDP bytes ($(echo "$BDP/1448" | bc) segments)"
sysctl net.ipv4.tcp_rmem net.ipv4.tcp_wmem
```

**Step 2: raise the buffers if the BDP demands it.**

```sh
sudo ip netns exec h1 sysctl -w "net.ipv4.tcp_wmem=4096 16384 16777216" > /dev/null
sudo ip netns exec h2 sysctl -w "net.ipv4.tcp_rmem=4096 131072 16777216" > /dev/null
echo "=== with larger buffers ==="
sudo ip netns exec h1 iperf3 -c 10.2.0.2 -t 10 2>/dev/null | grep receiver
```

**Step 3: check the counters.**

```sh
sudo ip netns exec h1 nstat -az > /tmp/b
sudo ip netns exec h1 iperf3 -c 10.2.0.2 -t 10 > /dev/null 2>&1
sudo ip netns exec h1 nstat 2>/dev/null | awk '$2 != 0' | \
  grep -iE 'retrans|loss|drop|error|overflow|prune|collapse|pressure|backlog'
```

A diagnostic script:

```sh
cat > tcpdiag.sh <<'EOF'
#!/bin/bash
DST=${1:-}
echo "=== Sockets ==="
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn
echo "=== Detail ==="
ss -tinme ${DST:+dst $DST} 2>/dev/null | head -20
echo "=== Socket memory ==="
cat /proc/net/sockstat
echo "=== TCP memory limits ==="
sysctl -n net.ipv4.tcp_mem net.ipv4.tcp_rmem net.ipv4.tcp_wmem
echo "=== Non-zero error counters ==="
nstat -az 2>/dev/null | awk '$2 != 0' | \
  grep -iE 'retrans|loss|drop|error|overflow|prune|collapse|pressure|backlog|reset|timeout|syncookie|reorder|dsack' | head -25
echo "=== Congestion control ==="
sysctl -n net.ipv4.tcp_congestion_control
echo "=== Queue disciplines ==="
tc -s qdisc show 2>/dev/null | grep -A2 'qdisc' | head -20
EOF
chmod +x tcpdiag.sh
sudo ip netns exec h1 ./tcpdiag.sh 10.2.0.2
```

Connection latency:

```sh
sudo tcpconnlat-bpfcc 2>/dev/null | head -10 &
for i in $(seq 1 5); do sudo ip netns exec h1 nc -z -w2 10.2.0.2 5201; done
```

Connection lifetimes:

```sh
sudo tcplife-bpfcc 2>/dev/null | head -15 &
sudo ip netns exec h1 iperf3 -c 10.2.0.2 -t 3 > /dev/null 2>&1
for i in $(seq 1 5); do (nc -z 127.0.0.1 22 2>/dev/null &); done
sleep 3
```

Cleanup:

```sh
sudo ip netns del h1 h2 r1 2>/dev/null
```

---

## 3. Mastery drills

1. State why `struct socket` and `struct sock` are separate. Construct a situation where each exists without the other, and name the field that keeps the second alive.

2. Explain the socket lock's two levels. Construct the deadlock that a single spinlock would cause, and the drop that the backlog's bound causes.

3. `dst->input` and `dst->output` are chosen by the routing lookup. Enumerate the functions each can be set to, and explain why framing routing this way is better than "find the next hop."

4. Explain why STALE neighbour entries are used without re-probing, and construct the latency problem that would occur otherwise.

5. State TCP's three control loops precisely. For a connection achieving 10 Mbit/s on a 1 Gbit/s path with 50 ms RTT, give the diagnostic that distinguishes which loop is limiting.

6. Compute the BDP for 10 Gbit/s at 100 ms RTT. Then state what `tcp_rmem`, `tcp_wmem`, and window scaling must each be for that path to saturate.

7. Explain why BBR is not loss-based, and construct two paths: one where it dramatically beats CUBIC, and one where it is unfair to CUBIC.

8. Explain RACK's rule precisely. Construct a reordering scenario where dupack-counting retransmits spuriously and RACK does not.

9. `TIME_WAIT` exists for two reasons. State both, then evaluate each of §T.6's six mitigations against both.

10. Distinguish `TIME_WAIT` accumulation from `CLOSE_WAIT` accumulation: the cause, the diagnosis, and the fix for each.

11. Socket memory is accounted by `truesize`, not `len`. Construct the attack this defends against, and compute the ratio for a 64-byte packet in a page fragment.

12. UDP GSO gives QUIC what TSO gives TCP. Derive the benefit by counting per-packet operations, and state what QUIC gives up by living in userspace.

13. You are given a service where p99 latency is 210 ms and p50 is 2 ms, on a 1 ms-RTT network. Give the ordered diagnostic procedure and the five most likely causes.

---

## 4. Further reading

**Kernel documentation**

- `Documentation/networking/ip-sysctl.rst` ★★★ — **every sysctl with its semantics.** The single most useful document for this chapter; read it completely once.
- `Documentation/networking/tcp_ao.rst`, `tcp-thin.rst`, `bbr.rst` (where present)
- `Documentation/networking/segmentation-offloads.rst` ★★★ — §T.8's UDP GSO.
- `Documentation/networking/msg_zerocopy.rst` ★★★ — §T.10.
- `Documentation/networking/scaling.rst` — Ch. 71, but relevant to `SO_REUSEPORT`.
- `Documentation/networking/nf_conntrack-sysctl.rst` — Ch. 73.
- `man 7 tcp` ★★★, `man 7 udp`, `man 7 ip`, `man 7 socket` ★★★ — **unusually good**; `man 7 tcp` documents most of the sysctls and socket options with their real behaviour.

**RFCs**

- RFC 9293 — **TCP**, the 2022 consolidation replacing 793 and its many updates. Readable and authoritative.
- RFC 5681 — TCP Congestion Control (the Reno baseline)
- RFC 8312 — CUBIC
- RFC 6298 — Computing TCP's RTO (Jacobson's algorithm, formalised)
- RFC 2018, 6675 — SACK
- RFC 8985 ★★★ — **RACK-TLP**, §T.5's modern loss recovery
- RFC 3168 — ECN; RFC 8257 — DCTCP
- RFC 7323 — window scaling and timestamps
- RFC 1191, 4821 — PMTU discovery and packetization-layer PMTUD
- RFC 768 — UDP (three pages; read it for the contrast)

**Papers**

- Jacobson & Karels, "Congestion Avoidance and Control," SIGCOMM 1988 ★★★ — **the foundational paper.** Slow start, congestion avoidance, and the RTO estimator, all still in the code.
- Cardwell, Cheng, Gunn, Yeganeh, Jacobson, "BBR: Congestion-Based Congestion Control," ACM Queue 2016 ★★★ — §T.5's departure, by its authors.
- Ha, Rhee, Xu, "CUBIC: A New TCP-Friendly High-Speed TCP Variant," SIGOPS OSR 2008 — the default algorithm.
- Cheng, Cardwell, Dukkipati, Jha, "RACK-TLP: A Time-Based Efficient Loss Detection for TCP," and the associated IETF work ★★★
- Alizadeh et al., "Data Center TCP (DCTCP)," SIGCOMM 2010 ★★★ — ECN used properly.
- Dukkipati et al., "An Argument for Increasing TCP's Initial Congestion Window," CCR 2010 — why `initcwnd` is 10.
- Mittal et al., "TIMELY: RTT-based Congestion Control for the Datacenter," SIGCOMM 2015
- Radhakrishnan et al., "TCP Fast Open," CoNEXT 2011
- Nichols & Jacobson, "Controlling Queue Delay" (CoDel), ACM Queue 2012 ★★★ — Ch. 73, but the bufferbloat argument belongs here too.

**Books**

- Stevens, Fenner, Rudoff, *UNIX Network Programming, Vol. 1*, 3rd ed. ★★★ — the socket API, definitively.
- Stevens, *TCP/IP Illustrated, Volume 1*, 2nd ed. (Fall & Stevens) ★★★ — **the protocols, with packet traces.** The best single book on TCP.
- Wright & Stevens, *TCP/IP Illustrated, Volume 2* — the BSD implementation; dated but the reasoning transfers.
- Rosen, *Linux Kernel Networking* ★★★ — Ch. 71's recommendation, with substantial TCP/IP coverage.
- Marek Majkowski's Cloudflare blog posts on Linux networking ★★★ — consistently excellent, particularly on socket queues, `SO_REUSEPORT`, and the accept path.

**LWN**

- "TCP small queues" ★★★ and the bufferbloat series
- "BBR congestion control" ★★★
- "RACK: a time-based loss detection algorithm"
- "TCP Fast Open"
- "The rise of BBRv2/v3" and the fairness debates
- "MSG_ZEROCOPY" ★★★ and "Zero-copy networking"
- "UDP GSO and GRO" ★★★ — the QUIC-driven work
- "The end of tcp_tw_recycle" ★★★ — §T.6's removed option and why
- "Removing the routing cache" (2012)
- "Network namespaces" and the container networking series
- "TCP authentication option (TCP-AO)"

**Source reading order**

1. `man 7 tcp` and `Documentation/networking/ip-sysctl.rst` first.
2. `include/linux/tcp.h`'s `struct tcp_sock` ★★★ — **read every field and know what it means.**
3. `net/ipv4/tcp.c`: `tcp_sendmsg_locked` ★★★, `tcp_recvmsg_locked`, `tcp_setsockopt`.
4. `net/ipv4/tcp_output.c`: `tcp_write_xmit` ★★★, `tcp_transmit_skb`, `tcp_nagle_test`, `tcp_tso_segs`.
5. `net/ipv4/tcp_input.c`: `tcp_ack` ★★★, `tcp_clean_rtx_queue`, `tcp_fastretrans_alert`, `tcp_rcv_established`. **The largest and hardest file in networking; budget real time.**
6. `net/ipv4/tcp_cong.c` and `tcp_cubic.c`, then `tcp_bbr.c` ★★★ — the contrast is the lesson.
7. `net/ipv4/tcp_recovery.c` — RACK; short and clear.
8. `net/ipv4/udp.c`: `udp_sendmsg`, `__udp4_lib_rcv`, `__udp_enqueue_schedule_skb`.
9. `net/core/sock.c`: `sock_alloc_send_pskb`, `__sk_mem_raise_allocated` ★★★ — §T.7.
10. `net/ipv4/inet_hashtables.c`: `__inet_lookup_established` — §1.6.

**Tools**

- `ss -tinme` ★★★ — **learn every field it prints**; it is `tcp_sock` made visible
- `nstat -az` ★★★ — and `nstat` alone for deltas
- `/proc/net/sockstat` ★★★
- `ip route get`, `ip -s neigh` ★★★
- `tcplife`, `tcpretrans`, `tcpconnlat`, `tcpdrop`, `tcpsynbl`, `tcptop` (bcc) ★★★
- `bpftrace` on `tcp:*` and `sock:inet_sock_set_state` ★★★
- `netem` via `tc` ★★★ — **the experimental apparatus**: `delay`, `loss`, `reorder`, `duplicate`, `corrupt`, `rate`
- `iperf3` (with `-u` for UDP, `-P` for parallel), `netperf`, `sockperf`, `wrk`
- `tcpdump -i any -nn 'tcp[tcpflags] & (tcp-syn|tcp-fin|tcp-rst) != 0'` ★★★
- Wireshark's TCP stream graphs (`Statistics → TCP Stream Graphs → Time Sequence`) ★★★ — the fastest way to see a congestion-control problem
- Network namespaces + `veth` + `netem` ★★★ — every experiment in this chapter runs on one machine

---

→ Next: [73-netfilter-tc.md](73-netfilter-tc.md)

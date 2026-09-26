# Chapter 10 — Core Data Structures II: XArray, Maple Tree, IDR, kfifo, Bitmaps

> **Goal:** you use the modern kernel containers correctly, including their locking and
> RCU semantics, and you know why the radix tree and IDR are legacy.

---

## Theory & First Principles

### T.0 — Start here: why a hash table is the wrong answer here

The page cache must map `(file, offset)` → `struct page`. A file may be 1 TiB; the pages
present may be three. What structure?

**Your first instinct is a hash table.** It is the right instinct in userspace and the wrong
one here. Work out why, because the reasons generalize:

| Requirement | Hash table |
|---|---|
| Point lookup | ✅ O(1) |
| **Range query** — "every page in \[4096, 8192)", needed by `readahead`, `truncate`, writeback | ❌ **O(n)**: you must probe every key in the range, or scan the whole table |
| **Ordered iteration** — writeback must walk pages in file order for sequential I/O | ❌ none |
| **Resize under load** | ❌ rehashing the world, with readers running |
| **Memory when sparse** | ❌ you must size for the worst case |
| **Lockless readers** (RCU) | ⚠️ possible but resizing makes it genuinely hard |

The range query is what disqualifies it. **Every real workload on the page cache is a range
operation** — readahead pulls 32 consecutive pages, `truncate` drops a suffix, writeback
walks in order to produce large sequential I/Os. A structure with O(1) point lookup and no
ordering is optimized for the operation that matters least.

**Second instinct: a balanced tree.** An rbtree gives you ordering and O(log n) ranges. That
was in fact the answer for years. But:

- Every node is a separate allocation with three pointers plus a colour bit — for a
  *pointer-sized* value, the metadata exceeds the payload.
- A lookup is a **dependent load chain** through scattered memory: ~20 cache misses for a
  million entries, and the hardware cannot prefetch any of them (Ch. 09 §T.3).
- Rebalancing rotates nodes, which is hostile to lockless readers.

**The answer the kernel actually uses is a radix trie with a very wide fan-out — the
XArray** — and the reasoning is entirely about the cost model from Ch. 00 §T.3:

```
 Tree depth for 2^32 entries:

   binary tree     (fan-out 2)   ->  32 levels  ->  ~32 cache misses
   B-tree          (fan-out 16)  ->   8 levels  ->   ~8 cache misses
   XArray          (fan-out 64)  ->   6 levels  ->   ~6 cache misses
                                        ↑
                     64 pointers = 512 bytes = 8 cache lines per node,
                     scanned linearly and PREFETCHABLE
```

The insight is the one from Ch. 09 §T.3 applied to trees: **you are not minimizing
comparisons, you are minimizing dependent cache misses.** A wide node costs more
comparisons per level — but those comparisons are on prefetched, contiguous memory at ~0.25
cycles each, while each *level* costs a ~200-cycle dependent miss. Trading 64 cheap
comparisons for one expensive miss is an enormous win, and it is why B-trees beat binary
trees on disk *and* why wide radix tries beat rbtrees in memory. **Same principle, three
orders of magnitude apart.**

And the radix trie gives, for free, the things the hash table could not: keys are stored in
order (so ranges and ordered iteration are natural), sparse regions cost nothing (an absent
subtree is one NULL pointer), there is no resize operation at all, and — because insertion
never rotates existing nodes — **lockless RCU readers are straightforward** (Ch. 15).

**The general lesson, which is the reason this chapter exists:** the right data structure is
determined by *the operation mix and the memory hierarchy*, not by asymptotic complexity.
Ask "what queries does this actually serve, and how many dependent cache misses does each
cost?" before you ask about big-O.

```bash
cd ~/src/linux
grep -n 'XA_CHUNK_SHIFT' include/linux/xarray.h    # the fan-out, and why
grep -n 'struct xarray\|i_pages' include/linux/fs.h | head
```

---

### T.1 The problem: mapping an integer key to a pointer

An enormous fraction of kernel work is *"given an integer, find the object"*:

| Mapping | Key range | Density |
|---|---|---|
| page cache: file offset → folio | 0 .. 2⁶³ | sparse, but clustered |
| fd table: fd → `struct file *` | 0 .. ~1M | **dense** from 0 |
| PID → `struct task_struct *` | 0 .. 4M | sparse |
| IRQ virq → `irq_desc` | 0 .. ~10k | moderately dense |
| memcg / cgroup IDs, device minors, inode numbers | varies | varies |
| VMA: address range → `vm_area_struct` | 0 .. 2⁴⁷ | **ranges**, not points |

Three candidate structures, three different failure modes:

- **Array**: O(1), but memory ∝ *max key*, so hopeless for sparse keys.
- **Hash table**: O(1) expected, but **no ordered traversal, no "next key" query, no range
  query**, and it needs resizing.
- **Balanced tree (rbtree)**: ordered and range-capable, but O(log n) with a **cache miss per
  level** (T.3 of Ch. 09) — for 1M entries that's ~20 dependent misses ≈ 4 µs.

The kernel's answer is a fourth option: a **radix trie** (XArray) for point lookups by
integer, and a **B-tree** (maple tree) for ranges. Both are chosen specifically to minimize
*cache lines touched*, not comparisons.

### T.2 Radix tries, and why the XArray is 64-ary

A **trie** (Fredkin, 1960) indexes by *consuming bits of the key*, not by comparing keys.
A radix-2^k trie consumes `k` bits per level:

```
  key = 0x1234, chunk = 6 bits
  level 0: bits 0-5    → slot 0x34 & 0x3f
  level 1: bits 6-11   → ...
  level 2: bits 12-17  → ...
```

Properties that matter:

- **Lookup is O(height) = O(word_bits / k)** — *independent of the number of entries*.
  For 64-bit keys with k=6 that is at most **11 levels**, always.
- **No comparisons, no balancing, no rotations.** The path is determined by the key alone,
  so there is no rebalancing on insert/delete and therefore **lock-free readers are easy**
  (crucial for RCU).
- **Memory is proportional to the number of *populated* subtrees**, not the key range.
  Sparse keys cost little; the trie is *path-compressed* by simply not allocating empty
  subtrees.

**Why k = 6 (64 slots)?** `XA_CHUNK_SHIFT = 6`, so a node holds 64 pointers = **512 bytes =
8 cache lines**. The trade:

| k | slots/node | node size | height for 2⁶⁴ | wasted memory on sparse keys |
|---|---|---|---|---|
| 4 | 16 | 128 B (2 lines) | 16 | low |
| **6** | **64** | **512 B** | **11** | moderate |
| 8 | 256 | 2 KiB | 8 | high |

k=6 was chosen empirically as the knee: short enough trees, node size that fits comfortably
in L1/L2, and acceptable waste. (On `CONFIG_BASE_SMALL` kernels it drops to 4.) Recognizing
that this is a *tunable point on a memory/height/cache curve* — not a magic constant — is the
theoretical takeaway.

**Comparison with an rbtree for 1M entries:**

| | rbtree | XArray |
|---|---|---|
| Levels traversed | ~20 | ~4 (for keys < 2²⁴) |
| Comparisons | ~20 | 0 |
| Dependent cache misses | ~20 | ~4 |
| Rebalancing on insert | yes | **no** |
| RCU-safe lookup | hard | **natural** |

That is why the page cache moved from a radix tree to the XArray and *never* to an rbtree,
and why the VMA tree moved from an rbtree to the maple tree.

### T.3 Tagged pointers: encoding types in the low bits

The XArray stores `void *` values, but it must also store: internal nodes, "this slot is a
sibling of a multi-order entry", small integers, and error codes — all in the same word.
It does this with **pointer tagging**, exploiting that kernel pointers are at least 4-byte
aligned so the low 2 bits are always zero:

```
  bit 0 = 1          → "internal entry" (a node pointer, a sibling entry, or a retry marker)
  bit 1 = 1, bit 0=0 → "value entry": the remaining 62 bits are a plain integer
  both 0             → an ordinary pointer
```

```c
xa_mk_value(v)     /* pack an unsigned long (< LONG_MAX) as an entry */
xa_is_value(e)     /* test */
xa_to_value(e)     /* unpack */
xa_is_err(e) / xa_err(e)   /* ERR_PTR-style errors, as in Ch. 05 T.6 */
```

This lets the page cache store *shadow entries* (eviction bookkeeping) and swap entries in
the same slots as folio pointers, with no extra allocation and no separate structure. It is
the same technique as `ERR_PTR` (Ch. 05), rbtree colour bits (Ch. 09 T.5), and the
`work_struct->data` field (Ch. 18) — **the kernel steals unused bits everywhere, and you
should learn to recognize it as a deliberate, reusable technique** rather than as a hack.

Separately, the XArray supports **marks** (formerly "tags"): up to 3 bits per entry,
summarized upward through the tree so that "find the next entry with mark 0 set" is fast
without scanning. The page cache uses these for `PAGECACHE_TAG_DIRTY`, `TAG_WRITEBACK`, and
`TAG_TOWRITE` — which is how writeback finds dirty pages in a 4 GB file without examining
every page. The summarization-up-the-tree idea is the same as a segment tree's.

### T.4 The maple tree: B-trees, ranges, and RCU

The XArray maps *points*. Some problems are inherently about *ranges*: a VMA covers
`[start, end)`, and the question is "which VMA contains address X?"

The **maple tree** (Liam Howlett & Matthew Wilcox, merged 6.1) is an **RCU-safe B-tree**
storing non-overlapping ranges. Two theoretical reasons it replaced the VMA rbtree:

**(a) B-trees are cache-optimal; binary trees are not.** This is the core result. A binary
tree node holds one key and two pointers — a cache line fetch delivers ~4 keys' worth of
bytes but you use 1. A B-tree node is *sized to the cache line*, so each miss delivers
~10–16 keys. The number of cache misses drops from `log₂(n)` to `log_b(n)` where `b` is the
branching factor:

```
  n = 1,000,000
  rbtree  : log₂(10⁶) ≈ 20 cache misses
  B-tree  : log₁₆(10⁶) ≈ 5 cache misses      ← 4× fewer
```
You do *more* comparisons (within a node, but those are in registers/L1) and *fewer* memory
stalls. Since a stall is ~100× a comparison, the B-tree wins decisively. This is the same
argument that makes B-trees the universal choice for on-disk indexes (Bayer & McCreight,
1970) — the memory hierarchy just moved up a level.

**(b) RCU-safe updates.** The maple tree does **copy-on-write of nodes** during modification,
publishing the new node with `rcu_assign_pointer()`. Readers walking the old node see a
consistent old state; no reader ever observes a partially-updated node. An rbtree cannot
easily do this because rotations mutate several nodes' pointers in place, and a concurrent
reader can be led into a cycle or miss a subtree.

The payoff, measured: the VMA conversion removed the `mmap_lock` from many read paths
(`lock_vma_under_rcu()`, enabling **per-VMA locking** for page faults in 6.4+) — one of the
largest MM scalability wins in years. **The data structure change *enabled* the locking
change.** That is the Ch. 14 T.9 lesson in action: the winning move is not a better lock, it
is a structure where readers don't write.

### T.5 kfifo: why power-of-two sizes and unsigned wraparound are correct

`kfifo` is a single-producer/single-consumer ring buffer. Its implementation looks too simple
to be right, and the reason it *is* right is a nice piece of modular arithmetic.

```c
struct __kfifo {
	unsigned int in;       /* producer index — MONOTONICALLY INCREASING, never masked */
	unsigned int out;      /* consumer index — same */
	unsigned int mask;     /* size - 1, size is a power of two */
	void *data;
};

/* number of elements present */
len  = fifo->in - fifo->out;
/* free space */
free = fifo->mask + 1 - (fifo->in - fifo->out);
/* physical slot */
slot = index & fifo->mask;
```

The key decision: **`in` and `out` are never wrapped**; they increment forever and wrap only
at `UINT_MAX`. Two consequences:

1. **`in - out` is always correct**, even across the `UINT_MAX` wrap, by modular arithmetic —
   exactly the same reasoning as `time_after()` in Ch. 19 T.8. Unsigned subtraction in C is
   defined to wrap, so `in - out` gives the true distance as long as the true occupancy is
   less than 2³². Since occupancy ≤ size ≤ 2³¹, this always holds.
2. **`in == out` unambiguously means empty**, and `in - out == size` means full. If you
   wrapped the indices instead, `in == out` would be ambiguous between empty and full, and
   you'd need either a wasted slot or an extra flag.

Power-of-two size makes `index & mask` replace a modulo (which would be a ~20-cycle division).

**Lock-freedom for SPSC**: the producer only writes `in` and reads `out`; the consumer only
writes `out` and reads `in`. Each index has exactly one writer, so **no atomic RMW is needed
at all** — just ordered stores. The required ordering (which `kfifo` provides with
`smp_wmb()`/`smp_rmb()`) is:

```
producer:  write data ; smp_wmb() ; WRITE_ONCE(in, in + n)      ← publish (release)
consumer:  READ_ONCE(in) ; smp_rmb() ; read data ; ... ; WRITE_ONCE(out, out + n)
```
This is precisely the **MP litmus test** from Ch. 13 T.3. If you understand why `kfifo`
needs `smp_wmb()`, you understand the memory model.

For multiple producers or consumers you need external locking — `kfifo` is documented as
lockless **only** for one-producer/one-consumer.

### T.6 Bitmaps: sets as machine words

A bitmap represents a set over a bounded universe as an array of `unsigned long`. The
operations map to single instructions:

| Operation | Instruction | Kernel API |
|---|---|---|
| test/set/clear one bit | `bt`/`bts`/`btr`, or and/or/andn | `test_bit`, `set_bit`, `clear_bit` |
| find first set | `bsf`/`tzcnt` (x86), `rbit`+`clz` (arm64) | `find_first_bit`, `ffs` |
| find first zero | invert + `bsf` | `find_first_zero_bit` |
| population count | `popcnt` | `bitmap_weight`, `hweight_long` |
| set union/intersection/difference | word-wise or/and/andn | `bitmap_or/and/andnot` |

So a set of up to 64 elements costs **one word and one instruction per operation** —
unbeatable by any pointer structure. For `n` elements it's `n/64` words with excellent
locality and vectorizable bulk operations.

This is why `cpumask` (a bitmap over CPUs), page flags, `nodemask`, IRQ affinity, and every
allocation bitmap in the kernel work this way. The theoretical point: **when the universe is
bounded and small, the optimal data structure is usually "bits in a word", and it beats
everything asymptotically better.**

Note the atomic/non-atomic split, which mirrors Ch. 13 T.11: `set_bit()` is atomic (and
unordered); `__set_bit()` is not atomic at all; `test_and_set_bit()` is atomic and fully
ordered; `clear_bit_unlock()`/`test_and_set_bit_lock()` provide release/acquire.

### T.7 Hash tables: chaining, resizing, and `rhashtable`

Classical theory in one paragraph: with `n` items in `m` buckets and a good hash, chain length
is ~`n/m` (the **load factor** α). Expected lookup is O(1 + α). Two collision strategies:
**chaining** (a list per bucket — what Linux uses) and **open addressing** (probe other
slots — better cache behaviour but deletion is painful and it cannot exceed α = 1).

The kernel uses chaining with `hlist` (Ch. 09 T.4) because deletion given an element must be
O(1) and must not disturb other entries — a hard requirement when objects are destroyed
independently.

**The hard problem is resizing.** A static `DEFINE_HASHTABLE` is fine when you know the size.
When you don't (connection tables, netlink sockets, nftables sets), you must grow — and a
naive rehash requires stopping all readers, which is unacceptable on a hot path.

`rhashtable` (`lib/rhashtable.c`, based on Triplett, McKenney & Walpole,
*"Resizable, Scalable, Concurrent Hash Tables via Relativistic Programming"*, USENIX ATC 2011)
solves this with **incremental, RCU-safe resizing**:

- Two tables coexist: the old and the new. Readers check the new table, then the old.
- Buckets are migrated incrementally, each under its own per-bucket lock.
- The relativistic-programming trick: because entries are moved in an order that guarantees a
  concurrent reader can never *miss* an entry (it may transiently see it in either table), no
  reader ever needs to block or retry.

**Use `rhashtable` rather than rolling your own resizable hash.** Reviewers will ask. It also
gives you `rhltable` (allowing duplicate keys) and automatic shrink.

```c
static const struct rhashtable_params params = {
	.key_len     = sizeof(u32),
	.key_offset  = offsetof(struct my_obj, key),
	.head_offset = offsetof(struct my_obj, node),   /* struct rhash_head node; */
	.automatic_shrinking = true,
};
rhashtable_init(&ht, &params);
rhashtable_lookup_fast(&ht, &key, params);     /* call under rcu_read_lock() */
rhashtable_insert_fast(&ht, &obj->node, params);
rhashtable_remove_fast(&ht, &obj->node, params);
rhashtable_destroy(&ht);
```

**Hash function choice** matters for more than distribution — it's a security property. A
weak, attacker-predictable hash lets a remote attacker force all keys into one bucket,
turning O(1) into O(n): a **hash-flooding DoS** (Crosby & Wallach, USENIX Security 2003).
Linux's defences: `siphash`/`half_siphash` for anything attacker-influenced (keyed, with a
boot-time random key), `jhash` with a random seed for internal tables, and
`hash_32`/`hash_64` (Fibonacci hashing, multiplying by 2⁶⁴/φ) for trusted integer keys.

> **Rule:** if the key can be influenced by an unprivileged local or remote party, use
> `siphash` with a secret key. `jhash` alone is not sufficient.

### T.8 IDR/IDA: allocating small dense integers

Handing an ID to userspace (a file descriptor, an IPC key, a device minor) needs
"give me the smallest unused non-negative integer". That is set-complement search, and the
efficient implementation is a **bitmap over a radix trie** — which is exactly what `IDA`
(ID Allocator) is: an XArray whose values are bitmaps of 1024 IDs each.

Note the Ch. 16 T.3 caveat: "smallest available" is a **non-commutative** specification, so it
is inherently unscalable across CPUs. Where you don't need smallest-available, use
`xa_alloc_cyclic()` (which hands out increasing IDs and wraps) — it reduces contention *and*
avoids ID-reuse confusion in userspace, which is a real source of bugs.

```c
DEFINE_IDA(my_ida);
id = ida_alloc_range(&my_ida, 1, 255, GFP_KERNEL);   /* returns -ENOSPC when exhausted */
ida_free(&my_ida, id);
ida_destroy(&my_ida);
```
`IDR` (ID→pointer) is now a thin compatibility layer over the XArray; **new code should use
the XArray directly** (`xa_alloc`, `xa_load`), and there is an ongoing conversion effort —
another good first-patch area.

### T.9 The full selection table

| You need | Use | Why (theory) |
|---|---|---|
| Set over a small bounded universe | **bitmap** / `cpumask` | one instruction per op (T.6) |
| Integer key → pointer, any density | **XArray** | O(height), no rebalance, RCU-natural (T.2) |
| Integer key → pointer, need "smallest free" | **IDA** / `xa_alloc` | bitmap over trie (T.8) |
| Arbitrary key → pointer, fixed size | `DEFINE_HASHTABLE` + `hlist` | O(1+α), cheap (T.7) |
| Arbitrary key → pointer, must resize, hot | **rhashtable** | RCU-safe incremental resize (T.7) |
| **Ranges** → pointer, ordered, RCU readers | **maple tree** | B-tree cache behaviour + CoW (T.4) |
| Ordered, needs augmentation (interval/max) | **augmented rbtree** | Ch. 09 T.5 |
| SPSC byte/record stream | **kfifo** | lock-free by construction (T.5) |
| Small (< ~100) unordered collection | **array or list_head** | cache beats asymptotics (Ch. 09 T.3) |
| LRU / arbitrary membership, O(1) removal | **list_head** | intrusive, no allocation (Ch. 09 T.1) |

---

## 1. XArray — the sparse array of pointers

`include/linux/xarray.h`, `lib/xarray.c`. Introduced 4.20 by Matthew Wilcox as a clean
API over the old radix tree. **This is the kernel's default "map unsigned long → pointer".**

Users: the page cache (`address_space::i_pages`), `idr`/`ida`, block layer tags,
`vmalloc`, IOMMU, DRM, many drivers.

### 1.1 Mental model

A sparse array indexed by `unsigned long`, storing `void *`. Internally a 64-ary
radix trie. Key properties:

- **Lockless reads** via RCU (`xa_load` under `rcu_read_lock()`).
- **Built-in spinlock** (`xa->xa_lock`) for writes.
- Can store **values** as well as pointers: `xa_mk_value(n)` packs an integer
  (up to `LONG_MAX/2`) into the pointer slot, tagged by bit 0.
- Three **marks** (tags) per entry, bit-indexed for fast "find next marked entry"
  — this is how the page cache finds dirty/writeback pages in O(log n).
- Supports multi-index entries (one entry covering 2^n indices) — how the page cache
  stores large folios.

### 1.2 API

```c
DEFINE_XARRAY(my_xa);                       /* static */
DEFINE_XARRAY_FLAGS(my_xa, XA_FLAGS_ALLOC); /* allocating (IDR-like) */
DEFINE_XARRAY_ALLOC(xa);   DEFINE_XARRAY_ALLOC1(xa);  /* ids from 0 / from 1 */
xa_init(&xa);  xa_init_flags(&xa, XA_FLAGS_LOCK_IRQ);
xa_destroy(&xa);

/* Simple (take/drop the internal lock for you) */
void *xa_load(&xa, index);                        /* NULL if absent */
void *xa_store(&xa, index, entry, gfp);           /* returns OLD entry or ERR_PTR */
void *xa_erase(&xa, index);
void *xa_cmpxchg(&xa, index, old, new, gfp);
int   xa_insert(&xa, index, entry, gfp);          /* -EBUSY if occupied */
int   xa_reserve(&xa, index, gfp);
void *xa_store_range(&xa, first, last, entry, gfp);

/* Allocating (replaces IDR) */
int xa_alloc(&xa, &id, entry, XA_LIMIT(0, 1024), gfp);
int xa_alloc_cyclic(&xa, &id, entry, limit, &next, gfp);   /* avoids ID reuse */

/* Marks */
void xa_set_mark(&xa, index, XA_MARK_0);
void xa_clear_mark(&xa, index, XA_MARK_1);
bool xa_get_mark(&xa, index, XA_MARK_2);
bool xa_marked(&xa, XA_MARK_0);                   /* any entry marked? */

/* Search / iterate */
void *xa_find(&xa, &index, max, filter);
void *xa_find_after(&xa, &index, max, filter);
xa_for_each(&xa, index, entry) { ... }
xa_for_each_start(&xa, index, entry, start)
xa_for_each_range(&xa, index, entry, start, last)
xa_for_each_marked(&xa, index, entry, XA_MARK_0)
unsigned int xa_extract(&xa, dst[], start, max, n, filter);

/* Values (integers in pointer slots) */
void *e = xa_mk_value(42);
if (xa_is_value(e)) n = xa_to_value(e);
xa_is_err(e); xa_err(e);           /* error encoding */
xa_is_zero(e);                     /* XA_ZERO_ENTRY: reserved slot */

/* Explicit locking (when you need atomicity across ops) */
xa_lock(&xa);      /* or xa_lock_irq / xa_lock_bh / xa_lock_irqsave */
  __xa_store(&xa, index, entry, gfp);
  __xa_erase(&xa, index);
  __xa_alloc(&xa, &id, entry, limit, gfp);
  __xa_set_mark(&xa, index, XA_MARK_0);
xa_unlock(&xa);

/* Advanced: xa_state for efficient multi-op sequences */
XA_STATE(xas, &xa, index);
XA_STATE_ORDER(xas, &xa, index, order);   /* multi-index entries */
rcu_read_lock();
xas_for_each(&xas, entry, max) { ... }
rcu_read_unlock();

xas_lock(&xas);
do {
	xas_store(&xas, entry);
} while (xas_nomem(&xas, GFP_KERNEL));   /* ★ the retry-on-OOM idiom */
xas_unlock(&xas);
if (xas_error(&xas)) ...
```

### 1.3 The `xas_nomem()` idiom

XArray may need to allocate internal nodes. In contexts where you hold a spinlock, you
can't `GFP_KERNEL`. The `xas_` API lets you drop the lock, allocate, and retry:

```c
	XA_STATE(xas, &mapping->i_pages, index);

	do {
		xas_lock_irq(&xas);
		xas_store(&xas, folio);
		if (xas_error(&xas))
			goto unlock;
		/* ... more work under the lock ... */
unlock:
		xas_unlock_irq(&xas);
	} while (xas_nomem(&xas, gfp));
```
`xas_nomem()` returns true if it successfully preallocated a node, meaning "retry".
Read `mm/filemap.c:__filemap_add_folio()` — the canonical example.

### 1.4 Example: replacing a list+hash with XArray

```c
static DEFINE_XARRAY_ALLOC(sessions);

static int session_create(struct session *s)
{
	u32 id;
	int ret = xa_alloc(&sessions, &id, s, XA_LIMIT(1, 65535), GFP_KERNEL);

	if (ret)
		return ret;
	s->id = id;
	return 0;
}

static struct session *session_get(u32 id)
{
	struct session *s;

	rcu_read_lock();
	s = xa_load(&sessions, id);
	if (s && !refcount_inc_not_zero(&s->ref))
		s = NULL;
	rcu_read_unlock();
	return s;
}

static void session_destroy(struct session *s)
{
	xa_erase(&sessions, s->id);
	/* then drop the refcount; free via call_rcu if readers are lockless */
}
```

---

## 2. Maple Tree — RCU-safe range storage

`include/linux/maple_tree.h`, `lib/maple_tree.c`. Merged 6.1 (Liam Howlett, Matthew Wilcox).

A **B-tree** (not a trie) storing **non-overlapping ranges** → pointers, with
**RCU-safe lockless reads**. Replaced the VMA rbtree + linked list in `mm/`.

### Why it exists
- rbtree = 3 pointers/node, terrible cache behavior, not RCU-safe for lookups.
- B-tree nodes hold many keys per cache line → far fewer cache misses.
- Ranges are first-class: `mtree_store_range(mt, first, last, ptr, gfp)`.
- Readers need no locks at all.

### API
```c
DEFINE_MTREE(mt);
mt_init(&mt);  mt_init_flags(&mt, MT_FLAGS_LOCK_EXTERN | MT_FLAGS_ALLOC_RANGE);
mtree_destroy(&mt);

void *mtree_load(&mt, index);
int   mtree_store(&mt, index, entry, gfp);
int   mtree_store_range(&mt, first, last, entry, gfp);
int   mtree_insert(&mt, index, entry, gfp);
int   mtree_insert_range(&mt, first, last, entry, gfp);
void *mtree_erase(&mt, index);
int   mtree_alloc_range(&mt, &startp, entry, size, min, max, gfp);  /* find a gap */
int   mtree_alloc_rrange(...);                                      /* from the top */

/* State-based (efficient) */
MA_STATE(mas, &mt, first, last);
mas_walk(&mas);
mas_find(&mas, max);
mas_next(&mas, max);  mas_prev(&mas, min);
mas_store_gfp(&mas, entry, GFP_KERNEL);
mas_empty_area(&mas, min, max, size);
mas_for_each(&mas, entry, max) { ... }

/* VMA-specific wrappers in mm/ */
VMA_ITERATOR(vmi, mm, addr);
for_each_vma(vmi, vma) { ... }
for_each_vma_range(vmi, vma, end) { ... }
vma_iter_store(&vmi, vma);
find_vma(mm, addr);        /* now maple-tree backed */
```

### Where to read it
`mm/mmap.c`, `mm/vma.c` (split out in 6.12) — the VMA management code is now entirely
maple-tree based. Also `lib/test_maple_tree.c` for behavior examples.

**When to choose maple tree:** you index by non-overlapping *ranges*, need RCU reads,
and have many entries. Otherwise XArray.

---

## 3. IDR / IDA — small-integer ID allocation

`include/linux/idr.h`. IDR is now a thin wrapper over XArray; **new code should use
`xa_alloc()` directly**. IDA (ID Allocator, no pointer) is still very much current.

### IDA — "give me a free small integer"
```c
static DEFINE_IDA(my_ida);

int id = ida_alloc(&my_ida, GFP_KERNEL);
int id = ida_alloc_range(&my_ida, 0, 255, GFP_KERNEL);
int id = ida_alloc_max(&my_ida, 255, GFP_KERNEL);
int id = ida_alloc_min(&my_ida, 1, GFP_KERNEL);
if (id < 0) return id;
...
ida_free(&my_ida, id);
ida_destroy(&my_ida);
bool used = ida_exists(&my_ida, id);
```

This is how nearly every driver allocates minor numbers, instance indices, etc.:
```c
	dev->index = ida_alloc(&drv_ida, GFP_KERNEL);
	dev_set_name(&dev->dev, "mydev%d", dev->index);
```

### IDR (legacy)
```c
static DEFINE_IDR(my_idr);
idr_alloc(&my_idr, ptr, start, end, GFP_KERNEL);
idr_find(&my_idr, id);
idr_remove(&my_idr, id);
idr_for_each_entry(&my_idr, entry, id) { ... }
idr_destroy(&my_idr);
```
IDR requires *external* locking for writes (unlike XArray's built-in lock).
If you see IDR in new code, flag it in review.

---

## 4. kfifo — lock-free ring buffer

`include/linux/kfifo.h`. A power-of-two circular buffer with **lock-free
single-producer/single-consumer** semantics (via `smp_wmb`/`smp_rmb` on the indices).

```c
/* Fixed-size, typed, on the stack or in a struct */
DECLARE_KFIFO(my_fifo, struct event, 128);      /* in a struct */
DEFINE_KFIFO(my_fifo, struct event, 128);       /* static */
INIT_KFIFO(my_fifo);

/* Dynamic */
struct kfifo fifo;
kfifo_alloc(&fifo, 4096, GFP_KERNEL);   /* bytes; rounded to power of 2 */
kfifo_init(&fifo, buf, size);            /* use existing buffer */
kfifo_free(&fifo);

/* Operations */
kfifo_put(&fifo, val);                   /* single element; 0 if full */
kfifo_get(&fifo, &val);                  /* 0 if empty */
kfifo_in(&fifo, buf, n);                 /* returns elements copied */
kfifo_out(&fifo, buf, n);
kfifo_out_peek(&fifo, buf, n);
kfifo_in_spinlocked(&fifo, buf, n, &lock);
kfifo_out_spinlocked(&fifo, buf, n, &lock);
kfifo_from_user(&fifo, ubuf, len, &copied);
kfifo_to_user(&fifo, ubuf, len, &copied);

kfifo_len(&fifo);  kfifo_avail(&fifo);  kfifo_size(&fifo);
kfifo_is_empty(&fifo);  kfifo_is_full(&fifo);
kfifo_reset(&fifo);  kfifo_reset_out(&fifo);
kfifo_skip(&fifo);

/* Record-based (variable-length records) */
DEFINE_KFIFO_REC_1(fifo, 1024);    /* 1-byte length prefix */
DEFINE_KFIFO_REC_2(fifo, 4096);    /* 2-byte length prefix */
```

Classic driver usage: IRQ handler does `kfifo_in()`, read() does `kfifo_to_user()`,
with a waitqueue for blocking. See `drivers/char/` and many IIO drivers.

**Caveat:** lock-free only for exactly one producer and one consumer. With more, use the
`_spinlocked` variants or your own lock.

---

## 5. Bitmaps

`include/linux/bitmap.h`, `include/linux/bitops.h`. Arrays of `unsigned long`.

```c
DECLARE_BITMAP(my_bits, 256);           /* unsigned long my_bits[4] on 64-bit */
unsigned long *b = bitmap_alloc(nbits, GFP_KERNEL);
bitmap_zalloc(nbits, gfp);  bitmap_free(b);

/* Atomic single-bit ops (lib/bitops) */
set_bit(nr, addr);          clear_bit(nr, addr);       change_bit(nr, addr);
test_bit(nr, addr);
test_and_set_bit(nr, addr); test_and_clear_bit(nr, addr);
test_and_set_bit_lock(nr, addr);  clear_bit_unlock(nr, addr);   /* with acquire/release */

/* Non-atomic (faster; use when you hold a lock) */
__set_bit  __clear_bit  __change_bit  __test_and_set_bit

/* Bitmap-wide */
bitmap_zero(dst, nbits);  bitmap_fill(dst, nbits);  bitmap_copy(dst, src, nbits);
bitmap_and/or/xor/andnot(dst, s1, s2, nbits);
bitmap_equal / bitmap_empty / bitmap_full / bitmap_subset / bitmap_intersects
bitmap_weight(src, nbits);              /* popcount */
bitmap_set(dst, start, len);  bitmap_clear(dst, start, len);
bitmap_shift_left / bitmap_shift_right
bitmap_find_next_zero_area(map, size, start, nr, align_mask);
bitmap_parse / bitmap_parselist / bitmap_print_to_pagebuf   /* "0-3,7" strings */

/* Iterate */
for_each_set_bit(bit, addr, nbits) { ... }
for_each_clear_bit(bit, addr, nbits)
for_each_set_bit_from(bit, addr, nbits)
find_first_bit / find_next_bit / find_first_zero_bit / find_next_zero_bit
find_last_bit / find_next_and_bit

/* Single-word bit ops */
ffs(x)      /* find first set, 1-based, 0 if none */
__ffs(x)    /* 0-based, undefined if x==0 */
fls(x) fls64(x)  __fls(x)
ffz(x)
hweight8/16/32/64(x)     /* popcount */
```

Special-purpose bitmap types:
- `cpumask_t` — `include/linux/cpumask.h`. **Always** use the cpumask API
  (`for_each_cpu`, `cpumask_set_cpu`, `cpumask_next`), never raw bitmap ops.
- `nodemask_t` — NUMA nodes.
- `sbitmap` — `include/linux/sbitmap.h`. **Scalable bitmap**: sharded with per-CPU
  allocation hints and a "sbitmap_queue" for waiters. This is how blk-mq allocates
  request tags across many CPUs without cacheline ping-pong. Study it in Chapter 63.

---

## 6. Other containers you'll meet

| Structure | Header | Purpose |
|---|---|---|
| `timerqueue` | `linux/timerqueue.h` | rbtree_cached of expiry times — hrtimer's backing store |
| `genpool` | `linux/genalloc.h` | general-purpose allocator for special memory (SRAM, device memory) |
| `flex_proportions` | `linux/flex_proportions.h` | per-entity fraction estimation (writeback bandwidth) |
| `percpu_counter` | `linux/percpu_counter.h` | approximate counter, exact on demand (fs free blocks) |
| `percpu_ref` | `linux/percpu-refcount.h` | refcount that is per-CPU while "alive", atomic when killing |
| `objpool`, `mempool` | `linux/mempool.h` | reserve pool guaranteeing forward progress under memory pressure |
| `bloom filter` | `lib/bloom_filter` (BPF map) | probabilistic membership |
| `min-heap` | `linux/min_heap.h` | array heap (used by perf, bpf) |
| `lru_cache` | `lib/lru_cache.c` | fixed-size LRU (DRBD) |
| `circ_buf` | `linux/circ_buf.h` | simple circular buffer macros (tty) |
| `assoc_array` | `linux/assoc_array.h` | RCU-safe associative array (keyrings) |
| `radix tree` | legacy | do not use in new code — use XArray |

`percpu_counter` and `percpu_ref` are so important for scalability that they get their
own treatment in Chapter 16.

---

## 7. Practice

### Lab 10.1 — XArray-backed object store
Rewrite Lab 9.1's registry using only an XArray with `XA_FLAGS_ALLOC`. Add marks:
`XA_MARK_0` = "dirty". Implement `xa_for_each_marked` to flush dirty objects.
Compare LOC and lookup performance with the hashtable version.

### Lab 10.2 — kfifo + IRQ
Write a module with a timer that "produces" events into a `kfifo` from softirq context
and a char device `read()` that blocks on a waitqueue and drains via `kfifo_to_user()`.
This is the skeleton of 80% of real char drivers.

### Lab 10.3 — sbitmap
Read `lib/sbitmap.c`. Explain: what is `sb->map[]`, what is `alloc_hint`,
what does `sbitmap_queue_wake_up()` do, and why is `SB_NR_TO_INDEX` sharded per-CPU.
Then find where blk-mq calls it (`blk_mq_get_tag`).

### Lab 10.4 — Maple tree
Read `lib/test_maple_tree.c`. Write a small module that stores 3 non-overlapping ranges
and does lookups. Then read `find_vma()` in `mm/mmap.c` and trace how it uses `mas_walk`.

### Lab 10.5 — Bitmap CPU affinity
Write a module that prints, for each online CPU, its NUMA node and sibling mask,
using only the cpumask API.

---

## 8. Extended practice

### Lab 10.A — XArray vs rbtree vs hash, at scale (T.2)

Extend the Ch. 09 Lab 9.A harness with an XArray and a maple tree, and sweep `n` over four
orders of magnitude.

```c
#include <linux/xarray.h>
#include <linux/maple_tree.h>

static DEFINE_XARRAY(xa);
static DEFINE_MTREE(mt);

/* insert */
for (i = 0; i < n; i++)
	xa_store(&xa, i, &arr[i], GFP_KERNEL);
for (i = 0; i < n; i++)
	mtree_store_range(&mt, i * 10, i * 10 + 9, &arr[i], GFP_KERNEL);

/* lookup — note: XArray lookups may be done under RCU with no lock */
t0 = ktime_get_ns();
rcu_read_lock();
for (i = 0; i < M; i++) {
	e = xa_load(&xa, keys[i]);
	if (e) sum += ((struct elem *)e)->val;
}
rcu_read_unlock();
pr_info("n=%d xarray: %llu ns/lookup\n", n, (ktime_get_ns() - t0) / M);

t0 = ktime_get_ns();
rcu_read_lock();
for (i = 0; i < M; i++) {
	e = mtree_load(&mt, keys[i] * 10 + 5);   /* a point inside the range */
	if (e) sum += ((struct elem *)e)->val;
}
rcu_read_unlock();
pr_info("n=%d maple : %llu ns/lookup\n", n, (ktime_get_ns() - t0) / M);
```
```bash
for n in 100 1000 10000 100000 1000000; do sudo insmod ds2.ko n=$n; sudo rmmod ds2; done
dmesg | grep ds2
sudo perf stat -e cache-misses,instructions -- sh -c 'sudo insmod ds2.ko n=1000000; sudo rmmod ds2'
```
**Expected:** the XArray's lookup time is nearly *flat* in `n` (it depends on key magnitude,
not count), while the rbtree grows logarithmically. Verify this; it is the clearest possible
demonstration of T.2.

### Lab 10.B — Sparse vs dense keys: measure the memory (T.2)

```c
static int __init sparse_init(void)
{
	static DEFINE_XARRAY(dense);
	static DEFINE_XARRAY(sparse);
	int i;

	for (i = 0; i < 10000; i++)
		xa_store(&dense, i, (void *)1UL, GFP_KERNEL);
	for (i = 0; i < 10000; i++)
		xa_store(&sparse, (unsigned long)i * 1000000, (void *)1UL, GFP_KERNEL);
	return 0;
}
```
```bash
grep -E 'radix_tree_node|xarray' /proc/slabinfo    # before and after
sudo slabtop -o | grep -i radix
```
Explain the difference using the trie structure: dense keys share interior nodes; sparse keys
each need their own path. Then compute the theoretical node count and compare.

### Lab 10.C — Tagged entries and marks (T.3)

```c
static int __init xatag_init(void)
{
	static DEFINE_XARRAY(xa);
	void *e;
	unsigned long idx;

	/* Value entries: no allocation at all */
	xa_store(&xa, 1, xa_mk_value(42), GFP_KERNEL);
	xa_store(&xa, 2, kzalloc(8, GFP_KERNEL), GFP_KERNEL);

	e = xa_load(&xa, 1);
	pr_info("idx1: is_value=%d value=%lu\n", xa_is_value(e), xa_to_value(e));
	e = xa_load(&xa, 2);
	pr_info("idx2: is_value=%d ptr=%px\n", xa_is_value(e), e);

	pr_info("max value entry = %lu\n", LONG_MAX);

	/* Marks: find entries without scanning */
	xa_set_mark(&xa, 2, XA_MARK_0);
	xa_for_each_marked(&xa, idx, e, XA_MARK_0)
		pr_info("marked entry at %lu\n", idx);

	xa_destroy(&xa);
	return 0;
}
```
Then find the real users:
```bash
grep -rn 'PAGECACHE_TAG_DIRTY\|PAGECACHE_TAG_WRITEBACK\|PAGECACHE_TAG_TOWRITE' include/linux/ mm/ | head
$EDITOR mm/page-writeback.c    # look at write_cache_pages(): tag_pages_for_writeback()
```
Explain how `tag_pages_for_writeback()` + `TOWRITE` prevent livelock when a file is being
continuously dirtied during writeback. (This is a *real* algorithmic use of marks, not just
bookkeeping.)

### Lab 10.D — Prove kfifo's modular arithmetic (T.5)

```c
static int __init fifo_math_init(void)
{
	unsigned int in, out, size = 8, mask = size - 1;

	/* Normal case */
	in = 5; out = 2;
	pr_info("len=%u free=%u slot(in)=%u\n", in - out, mask + 1 - (in - out), in & mask);

	/* Across the UINT_MAX wrap */
	in = 2; out = UINT_MAX - 2;          /* in has wrapped, out has not */
	pr_info("wrapped: len=%u (should be 5)\n", in - out);

	/* Why in==out is unambiguous */
	in = out = 100;
	pr_info("empty: len=%u\n", in - out);
	in = out + size;
	pr_info("full : len=%u (== size %u)\n", in - out, size);
	return 0;
}
```
Then write a **deliberately broken** version that masks the indices on store and demonstrate
the empty/full ambiguity. Finally, find the barriers:
```bash
grep -n 'smp_wmb\|smp_rmb\|smp_store_release\|smp_load_acquire' lib/kfifo.c include/linux/kfifo.h
```
For each, say which litmus test from Ch. 13 it is solving.

### Lab 10.E — A working SPSC kfifo between a kthread and userspace

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>
#include <linux/kfifo.h>
#include <linux/kthread.h>
#include <linux/miscdevice.h>
#include <linux/poll.h>
#include <linux/delay.h>

struct sample { u64 ts; u32 val; };

static DEFINE_KFIFO(fifo, struct sample, 256);
static DECLARE_WAIT_QUEUE_HEAD(rq);
static struct task_struct *producer;

static int produce(void *unused)
{
	u32 i = 0;

	while (!kthread_should_stop()) {
		struct sample s = { .ts = ktime_get_ns(), .val = i++ };

		if (!kfifo_put(&fifo, s))
			pr_warn_ratelimited("fifo full, dropping\n");
		wake_up_interruptible(&rq);
		msleep(10);
	}
	return 0;
}

static ssize_t f_read(struct file *f, char __user *ubuf, size_t len, loff_t *off)
{
	unsigned int copied;
	int ret;

	if (kfifo_is_empty(&fifo)) {
		if (f->f_flags & O_NONBLOCK)
			return -EAGAIN;
		ret = wait_event_interruptible(rq, !kfifo_is_empty(&fifo));
		if (ret)
			return ret;
	}
	ret = kfifo_to_user(&fifo, ubuf, len, &copied);
	return ret ? ret : copied;
}

static __poll_t f_poll(struct file *f, poll_table *wait)
{
	poll_wait(f, &rq, wait);
	return kfifo_is_empty(&fifo) ? 0 : (EPOLLIN | EPOLLRDNORM);
}

static const struct file_operations fops = {
	.owner = THIS_MODULE, .read = f_read, .poll = f_poll, .llseek = noop_llseek,
};
static struct miscdevice md = { MISC_DYNAMIC_MINOR, "kfifo_demo", &fops };

static int __init k_init(void)
{
	int ret = misc_register(&md);

	if (ret)
		return ret;
	producer = kthread_run(produce, NULL, "kfifo_prod");
	if (IS_ERR(producer)) {
		misc_deregister(&md);
		return PTR_ERR(producer);
	}
	return 0;
}
static void __exit k_exit(void)
{
	kthread_stop(producer);     /* ★ Ch. 05 T.4: stop the producer BEFORE deregistering */
	misc_deregister(&md);
}
module_init(k_init); module_exit(k_exit);
MODULE_LICENSE("GPL");
```
```bash
sudo insmod kfifo_demo.ko
sudo hexdump -C -n 96 /dev/kfifo_demo
sudo cat /dev/kfifo_demo | hexdump -C | head
```
This is a complete, idiomatic producer/consumer driver in 60 lines. Study the teardown order.

### Lab 10.F — `rhashtable` with resize under load (T.7)

```c
struct entry { u32 key; struct rhash_head node; struct rcu_head rcu; u32 val; };

static const struct rhashtable_params params = {
	.key_len             = sizeof(u32),
	.key_offset          = offsetof(struct entry, key),
	.head_offset         = offsetof(struct entry, node),
	.automatic_shrinking = true,
};
static struct rhashtable ht;

/* insert 1M entries from several kthreads while others look up concurrently */
```
```bash
# Watch it grow:
grep -i rhashtable /proc/slabinfo
sudo bpftrace -e 'kprobe:rhashtable_rehash_alloc { @rehashes = count(); }'
# Read the algorithm:
$EDITOR lib/rhashtable.c        # rhashtable_rehash_chain(), the two-table walk in lookup
```
Verify with `CONFIG_PROVE_RCU=y` that a lookup during a rehash never misses an entry.

### Lab 10.G — Hash flooding, demonstrated (T.7)

In **userspace first** (safer), build a hash table with `jhash` and a fixed seed, then
construct 10,000 keys that all collide into one bucket (brute-force search). Measure lookup
time vs. random keys. Then repeat with `siphash` and a secret key and show the attack fails.

```bash
grep -rn 'siphash\|get_random_once\|jhash' net/ipv4/inet_hashtables.c | head
$EDITOR include/linux/siphash.h        # read the header comment: when to use which
$EDITOR Documentation/security/siphash.rst
```
Then find a kernel hash table you believe uses a weak hash on attacker-controlled input, and
check whether it's actually exploitable. (Several real CVEs and hardening patches exist here.)

### Lab 10.H — `sbitmap`: scalable tag allocation

```c
#include <linux/sbitmap.h>

struct sbitmap_queue sbq;

sbitmap_queue_init_node(&sbq, 256 /*depth*/, -1 /*shift: auto*/, false /*round_robin*/,
			GFP_KERNEL, NUMA_NO_NODE);
tag = sbitmap_queue_get(&sbq, &cpu);     /* -1 if none free */
sbitmap_queue_clear(&sbq, tag, cpu);
sbitmap_queue_free(&sbq);
```
Benchmark a plain `DECLARE_BITMAP` + spinlock against `sbitmap` with N kthreads contending.
Predict the result from Ch. 16 T.1 *before* running it, then verify.
```bash
sudo cat /sys/kernel/debug/block/nvme0n1/hctx0/tags     # sbitmap state in blk-mq
$EDITOR lib/sbitmap.c   # note: per-CPU caching + word-level spreading to avoid one hot line
```

---

## 9. Mastery drills

1. Explain the XArray "value entry" encoding. Why is bit 0 used? What's the max value?
   Read `xa_mk_value()`.
2. What are the three XArray marks used for in the page cache? Find them
   (`PAGECACHE_TAG_DIRTY`, `PAGECACHE_TAG_WRITEBACK`, `PAGECACHE_TAG_TOWRITE`).
   Trace how `filemap_fdatawrite_range()` uses them.
3. Explain precisely how the maple tree achieves RCU-safe reads during a node split.
   (Hint: it never modifies a live node; it builds replacements and swaps them.)
4. Why is `ida_alloc()` better than a hand-rolled bitmap + spinlock? Read `lib/idr.c`.
5. `kfifo` claims lock-free SPSC. Find the memory barriers in
   `include/linux/kfifo.h`/`lib/kfifo.c` and explain what each one orders.
6. `sbitmap` vs plain bitmap: design an experiment (many CPUs contending for tags) that
   demonstrates the difference. Predict the result before measuring.
7. **Derive the branching factor.** Given `XA_CHUNK_SHIFT = 6`, compute node size, tree
   height for keys up to 2³², and the number of cache lines touched per lookup. Repeat for
   shift 4 and 8 and argue which you'd pick for (a) the page cache, (b) a 1000-entry IRQ
   table, (c) a 64-bit sparse ID space.
8. **B-tree vs binary tree (T.4).** For n = 10⁶, compute the expected number of cache misses
   for an rbtree and for a B-tree with 16 keys/node. Then find the actual maple tree node
   size (`MAPLE_NODE_SLOTS` in `include/linux/maple_tree.h`) and redo the calculation with
   real numbers.
9. **Read the maple tree conversion series.** `git log --oneline --grep="maple tree" -- mm/ |
   head -40`. Read the cover letter on lore.kernel.org. List the measured improvements and
   the follow-on change it enabled (per-VMA locking). Write 400 words on why the data
   structure change was the *prerequisite* for the locking change.
10. **Find the tagged-pointer instances.** Locate five distinct places in the kernel that
    steal low bits of a pointer (`ERR_PTR`, rbtree colour, `work_struct->data`, XArray
    internal entries, `hlist_nulls`, `list_head` poison, `dentry` flags, ...). For each, say
    how many bits and what guarantees they're free.
11. **IDR → XArray conversion.** Find a driver still using the IDR API and convert it to the
    XArray. Verify behaviour is unchanged. (`git log --grep="convert.*IDR.*XArray" --oneline`
    for precedents — this is an actively welcomed cleanup.)
12. **Choose correctly.** For each, pick a structure from the T.9 table and justify in two
    sentences: (a) 64 hardware queues, need "any free one"; (b) file offset → folio for a
    100 GB sparse file; (c) a driver's list of open contexts, ≤ 8; (d) 5M TCP flows with
    100k inserts/sec and attacker-controlled keys; (e) "which memory region contains this
    address" over 200k regions; (f) a trace buffer written by one CPU and read by userspace.

---

## 10. Further reading

**Kernel documentation (unusually good in this area):**
- `Documentation/core-api/xarray.rst` ★★ (excellent; read fully)
- `Documentation/core-api/maple_tree.rst`
- `Documentation/core-api/idr.rst`
- `Documentation/core-api/kernel-api.rst` (bitmaps, kfifo sections)
- `Documentation/core-api/assoc_array.rst`
- `Documentation/security/siphash.rst` ★ — when to use siphash vs jhash
- `include/linux/xarray.h` — the header comment is a full tutorial
- `lib/test_xarray.c`, `lib/test_maple_tree.c`, `lib/test_bitmap.c`, `lib/test_rhashtable.c`
  — **executable specifications**; the fastest way to learn each API

**Papers:**
- Fredkin, "Trie Memory" (CACM 1960) — the original
- Bayer & McCreight, "Organization and Maintenance of Large Ordered Indices" (1972) — B-trees
- Morrison, "PATRICIA — Practical Algorithm To Retrieve Information Coded In Alphanumeric"
  (JACM 1968) — path compression
- Triplett, McKenney & Walpole, "Resizable, Scalable, Concurrent Hash Tables via Relativistic
  Programming" (USENIX ATC 2011) — **the rhashtable algorithm**
- Crosby & Wallach, "Denial of Service via Algorithmic Complexity Attacks"
  (USENIX Security 2003) — hash flooding (T.7)
- Aumasson & Bernstein, "SipHash: a fast short-input PRF" (2012)
- Carter & Wegman, "Universal Classes of Hash Functions" (1979) — why keyed hashing works
- Leis, Kemper & Neumann, "The Adaptive Radix Tree" (ICDE 2013) — the state of the art in
  radix tries; useful context for judging the XArray's design choices
- Rao & Ross, "Making B+-Trees Cache Conscious in Main Memory" (SIGMOD 2000) — T.4's
  cache-miss argument, quantified

**LWN:**
- "The XArray data structure" (Wilcox) and the whole radix-tree → XArray conversion series
- "Introducing maple trees" / "The maple tree, a modern data structure"
- "Per-VMA locking" — the payoff described in T.4
- "A block layer introduction part 1" (sbitmap context)
- "Denial of service via hash collisions" and the siphash adoption series

→ Next: [11-memory-apis.md](11-memory-apis.md)

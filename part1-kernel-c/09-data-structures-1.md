# Chapter 09 — Core Data Structures I: Lists, hlists, rbtree

> **Goal:** you can pick the right container for a kernel problem and implement it with
> correct locking and RCU semantics without looking anything up.

---

## Theory & First Principles

> **How to read this section.** T.0 is the concrete problem that makes kernel containers look
> the way they do. T.1–T.4 are the design theory. T.5–T.9 are the individual structures and
> when to use each.

---

### T.0 — Start here: the problem with `std::list`

Suppose you are writing a driver. A request object needs to be on several lists at once:

```
 struct request
   ├─ on the device's PENDING list
   ├─ on an LRU list for reclaim
   ├─ in a HASH TABLE indexed by tag
   └─ on a per-CPU COMPLETION list
```

Write it the way userspace would:

```c
/* The userspace instinct: the container owns nodes that POINT AT your object. */
struct list_node { void *data; struct list_node *next, *prev; };

int submit(struct request *r)
{
	struct list_node *n = malloc(sizeof(*n));   /* ① ALLOCATION */
	if (!n)
		return -ENOMEM;                     /* ② INSERTION CAN FAIL */
	n->data = r;
	list_push(&dev->pending, n);
	return 0;
}

void complete(struct request *r)
{
	/* ③ To remove r, SEARCH the list for the node pointing at it. O(n). */
	/* ④ ...and repeat for the LRU list, the hash table, the per-CPU list. */
}
```

**Four problems, and in a kernel each one is disqualifying:**

① **It allocates.** You cannot allocate in an interrupt handler with `GFP_KERNEL`, you may
not be able to allocate at all under memory pressure, and the allocation is exactly when you
do not want to fail — you are trying to *free* memory.

② **Insertion can fail**, so every call site needs an error path. Error paths are where bugs
live (Ch. 06 §T.0). And what do you do when adding to the *third* list fails, having already
added to two?

⁢ **Removal is O(n) per list**, and you have four lists. Teardown of one object becomes a
linear scan of everything.

⁣ **The object does not know it is on those lists.** When it is freed, nothing prevents a
stale node pointing at it — a use-after-free waiting to happen.

**Now the kernel's answer.** Put the link *inside* the object:

```c
struct request {
	int tag;
	struct list_head pending;     /* the links live IN the object */
	struct list_head lru;
	struct hlist_node hash;
};

void submit(struct request *r)          /* returns void -- IT CANNOT FAIL */
{
	list_add(&r->pending, &dev->pending);
}

void complete(struct request *r)
{
	list_del(&r->pending);          /* O(1). No search. */
	list_del(&r->lru);              /* O(1). */
	hlist_del(&r->hash);            /* O(1). */
}
```

**All four problems vanish simultaneously.** No allocation, so no failure, so no error path.
O(1) removal from every list given only the object. And the object carries its own links, so
its lifetime and its membership are the same question.

**The cost — and there is always a cost:** `struct request` must now *know* about every
container it can join. The container is no longer a black box, and you cannot put a
third-party type in a kernel list without modifying it. In a general-purpose library that is
fatal. In a kernel it is free, **because the kernel is the library** — you own both sides.

That trade is the subject of §T.1, and the mechanism that makes it type-safe — recovering
`struct request *` from a `struct list_head *` — is `container_of` from Ch. 08 §T.3. Every
intrusive structure in the kernel is this same idea: `rb_node`, `hlist_node`, `work_struct`,
`hrtimer`, `kref`, `kobject`, `llist_node`.

**See it in real code right now:**

```bash
cd ~/src/linux
grep -n 'struct list_head\|struct hlist_node' include/linux/sched.h | head -20
#   ^ task_struct is on a dozen lists at once. Count them.
```

---

### T.1 Intrusive vs. non-intrusive containers

Every container library makes one foundational choice: does the container **own storage for
its nodes**, or does the *element* carry the node inside itself?

```c
/* NON-INTRUSIVE (C++ std::list, glib GList, most userspace libraries) */
struct list_node { void *data; struct list_node *next, *prev; };
/* the container allocates a node per insertion; node->data points to your object */

/* INTRUSIVE (Linux) */
struct my_obj {
	int data;
	struct list_head node;      /* the link lives INSIDE the element */
};
```

The differences are not stylistic. In a kernel they are decisive:

| Property | Non-intrusive | **Intrusive (Linux)** |
|---|---|---|
| Allocation on insert | **yes** — a node must be allocated | **none** |
| Can insertion fail? | **yes** (`-ENOMEM`) | **no** — it cannot fail |
| Usable in atomic/NMI context? | no (allocation may sleep or fail) | **yes** |
| Cache behaviour | node and object are in different cache lines | link and data share a line |
| Indirections to reach data | 2 (node → data) | **1** (the node *is* in the data) |
| Element in multiple containers | one node per container, each allocated | just embed multiple `list_head`s |
| Type safety | `void *` | `container_of` with `__same_type` assert |
| Removal given only the element | needs a search, or a back-pointer | **O(1)** — you already have the link |

The two that matter most:

1. **Insertion cannot fail.** This is enormous. A non-intrusive container forces an error path
   into every insertion site — and error paths are where bugs live (Ch. 06 T.6). With
   intrusive containers, `list_add()` returns `void`. Whole classes of failure handling
   disappear.
2. **O(1) removal from an arbitrary element.** Given a `struct my_obj *`, you can unlink it
   from every list it is on without searching any of them. This is exactly what you need when
   an object is being destroyed and is simultaneously on an LRU list, a hash chain, a
   per-device list, and a dirty list — which is the normal situation in a kernel.

The cost: the element's definition must know about the containers it may join, so the
container is not a black box. In a kernel that is acceptable — the kernel *is* the library.

**This is the same embedding-based structural subtyping as Ch. 08 T.3.** `struct list_head`
is a base class; `container_of` is the downcast. Once you see it that way, `rb_node`,
`hlist_node`, `work_struct`, `hrtimer`, `kref`, and `kobject` are all the same idea.

### T.2 Why a *circular*, *doubly-linked* list with a sentinel head

Linux's list is circular and headed by a sentinel node that is *not* part of any element:

```
  head ⇄ A ⇄ B ⇄ C ⇄ (back to head)
```

Three design choices, each eliminating a branch:

**(a) Doubly-linked.** Enables O(1) removal given only the node (`n->prev->next = n->next`).
A singly-linked list would require a traversal or a back-pointer. (`hlist` gives back most of
the memory — see T.4.)

**(b) Circular.** There is no `NULL` terminator, so no end-of-list test in the pointer
manipulation code.

**(c) Sentinel head.** The head is a real `struct list_head` with `next`/`prev`, so **the
empty list is not a special case**: an empty list is simply `head.next == head.prev == &head`.

The payoff is visible in the implementation, which is completely branch-free:

```c
static inline void __list_add(struct list_head *new,
			      struct list_head *prev, struct list_head *next)
{
	next->prev = new;
	new->next = next;
	new->prev = prev;
	WRITE_ONCE(prev->next, new);
}

static inline void __list_del(struct list_head *prev, struct list_head *next)
{
	next->prev = prev;
	WRITE_ONCE(prev->next, next);
}
```

**Four stores, zero branches, for insert or delete, regardless of position — including
inserting into an empty list or deleting the only element.** A `NULL`-terminated,
non-sentinel list needs 4–6 conditionals to handle head/tail/empty/single cases, and every
one of those conditionals is a place to write a bug. This is a small, perfect example of
**choosing a representation that makes special cases disappear** — Lampson's "handle normal
and worst case separately" inverted into "make them the same case".

(The `WRITE_ONCE` on `prev->next` is not decoration: it is what makes `list_for_each_entry_rcu`
safe. The store that publishes the new node must not be torn or reordered by the compiler —
Ch. 13, Ch. 15.)

### T.3 Complexity is not the cost model: cache misses are

Textbook analysis says a linked list has O(1) insert and O(n) search. On modern hardware that
is misleading, because the constants differ by two orders of magnitude:

| Operation | Cycles (L1 hit) | Cycles (cache miss) |
|---|---|---|
| Pointer chase, next node in L1 | ~4 | — |
| Pointer chase, next node in DRAM | — | **~200–300** |
| Linear scan of an array, prefetched | ~0.25/element (SIMD/prefetch) | — |

A linked-list traversal is a **dependent load chain**: the address of node *n+1* is not known
until node *n*'s load returns, so the hardware **cannot prefetch** and cannot overlap the
misses. An array scan has independent addresses, so the prefetcher hides the latency and you
get memory-level parallelism.

Empirically: **for up to a few hundred elements, a linear scan of a flat array usually beats a
linked list, a tree, and often a hash table** — despite worse asymptotics. This is why the
kernel:

- Uses arrays and bitmaps for small, bounded sets (`cpumask`, fd tables, `kfifo`).
- Uses B-tree-like structures (the maple tree, Ch. 10) instead of rbtrees in new code —
  B-trees do more comparisons but **far fewer cache misses**, because one node fills a
  cache line.
- Cares intensely about where `list_head` sits within a struct (`pahole`, Ch. 16 T.2).

**The rule:** choose by *number of cache lines touched*, not by big-O. State this in review
and you will be right more often than the person quoting asymptotics.

### T.4 `hlist`: paying a pointer for a hash table

A hash table with 1M buckets using `struct list_head` heads costs 16 MB of *heads alone*
(two pointers each), most of them empty. `hlist` halves it:

```c
struct hlist_head { struct hlist_node *first; };                  /* ONE pointer */
struct hlist_node { struct hlist_node *next, **pprev; };          /* two, but see below */
```

The trick is `pprev`: instead of pointing to the previous *node*, it points to **the previous
node's `next` pointer** (or to the head's `first` pointer for the first node). So removal is
still O(1) and still uniform:

```c
*(node->pprev) = node->next;
if (node->next) node->next->pprev = node->pprev;
```
No special case for "first element", because the head's `first` field is addressed exactly
like any node's `next` field. This is the **pointer-to-pointer idiom**, and it is the same
trick Linus famously cites as an example of "good taste" in programming. Recognize it; it
appears throughout the tree.

Cost: no O(1) backward traversal and no `list_for_each_prev`. Hash chains never need that, so
it's free in practice.

`hlist_nulls` is a further refinement for RCU hash tables where an element can **move between
buckets** while a reader traverses it: the terminator encodes the bucket number, so a reader
that walks off the end of a chain can detect it ended in the *wrong* bucket and restart.
Without it, a lockless reader could silently miss an element that was rehashed. This is the
socket-lookup fast path (Ch. 15 drill 6).

### T.5 Red-black trees: the invariants, and why RB rather than AVL

An rbtree is a binary search tree with five invariants:

1. Every node is red or black.
2. The root is black.
3. All leaves (NIL) are black.
4. **A red node's children are both black** (no two reds in a row).
5. **Every path from a node to its descendant leaves contains the same number of black
   nodes** (the *black height*).

From 4 and 5: the longest root-to-leaf path is at most **twice** the shortest, so
height ≤ 2·log₂(n+1) — hence O(log n) for search, insert, delete.

**Why red-black and not AVL?** Both are O(log n). The difference is in the *constants* and
*which operation you favour*:

| | AVL | **Red-black** |
|---|---|---|
| Height bound | ~1.44 log n (tighter) | ~2 log n |
| **Lookups** | **faster** (shallower) | slightly slower |
| **Rotations per insert** | ≤ 2 | ≤ 2 |
| **Rotations per delete** | **O(log n)** | **≤ 3** |
| Rebalancing after update | may cascade to the root | **O(1) amortized** |

The decisive row is **deletion**. AVL deletion can require rotations all the way to the root;
rbtree deletion requires at most three rotations (plus O(log n) recolourings, which are cheap
— just a bit flip, no pointer writes). For kernel workloads — scheduler runqueues, VMA trees,
timer queues, epoll sets — **insert and delete dominate lookups**, and every rotation is a
cache-line write. So the kernel trades a slightly taller tree for a bounded update cost.

Two Linux-specific engineering notes:

- **The rbtree API is not a container; it is a toolkit.** Unlike `list_head`, Linux's rbtree
  makes *you* write the search loop and call `rb_link_node()` + `rb_insert_color()`. This is
  deliberate: comparison is type-specific, and inlining the comparison into your loop avoids
  an indirect call per comparison (Ch. 08 T.4). It also lets you do "find or insert" in a
  single descent. Verbose, but fast.
- **The colour is stored in the low bits of the parent pointer** (`rb_parent_color`), because
  nodes are at least 4-byte aligned so the low bits are free. Another instance of pointer
  tagging (Ch. 05 T.6).
- **`rb_root_cached`** adds a cached pointer to the leftmost node, making "find minimum" O(1)
  instead of O(log n). Essential for anything that repeatedly asks "what's next?" — the CFS
  runqueue, hrtimer bases, and deadline scheduling all use it.

**Augmented rbtrees** extend this further: each node caches a summary of its subtree
(e.g. the maximum endpoint), maintained through rotations by a callback. That gives you
**interval trees** — "find all intervals overlapping [a,b]" in O(log n + k) — used for
`mmap` region lookups, `userfaultfd`, and the memory notifier (`mmu_interval_notifier`).
The theory is standard (CLRS Ch. 14); the kernel's contribution is doing it with callbacks
so the augmentation is generic.

### T.6 Choosing a structure: a decision procedure

```
How many elements, and is the count bounded?
├─ ≤ ~64 booleans/flags        → bitmap (DECLARE_BITMAP, Ch. 10)
├─ small (< ~100), fixed set   → flat array + linear scan   ← usually wins (T.3)
└─ unbounded
   ├─ Iterate all, no ordering needed, O(1) insert/remove given the element
   │     → list_head  (or hlist if you need many heads / it's a hash chain)
   ├─ Look up by an arbitrary key
   │     → hash table: hlist + your own hashing, or rhashtable (resizable, RCU-safe)
   ├─ Look up by an INTEGER key, densely or sparsely
   │     → XArray (Ch. 10)  — better than both a hash table and an rbtree for this
   ├─ Need ORDERED traversal, or "next/previous key", or range queries
   │     ├─ mostly static, cache-sensitive, RCU readers → maple tree (Ch. 10)
   │     └─ frequent insert/delete, need augmentation    → rbtree / augmented rbtree
   ├─ Priority queue with frequent "get min"
   │     → rb_root_cached, or plist if priorities are few and O(1) matters
   ├─ Producer/consumer byte or record stream
   │     → kfifo (Ch. 10)
   └─ Need an ID for a userspace handle
         → IDA/IDR or xa_alloc (Ch. 10)
```

Then ask the Ch. 14/15/16 questions: who else touches it, under what lock, and can readers be
made not to write?

---

## 1. Concept: intrusive containers

Userspace containers *own* your data (`std::list<T>` stores `T`). Kernel containers are
**intrusive**: the node is embedded *in* your struct.

```c
struct my_obj {
	int			id;
	struct list_head	list;    /* the node lives INSIDE the object */
	char			name[32];
};
```

Why:
- **Zero allocation** for insertion — no separate node object, no allocation failure path.
- **O(1) removal given the element** — you don't need to search for it.
- An object can be on **many lists at once** (just embed multiple nodes).
- Deterministic memory behavior, critical in atomic contexts.

Cost: you need `container_of` to get back from the node to the object, and type safety
is by convention.

---

## 2. `struct list_head` — circular doubly-linked list

`include/linux/list.h`. This is the most-used data structure in the kernel.

```c
struct list_head {
	struct list_head *next, *prev;
};
```

### 2.1 Declaration and init

```c
LIST_HEAD(my_list);                       /* static/global */
static LIST_HEAD(driver_list);

struct my_ctx {
	struct list_head head;
	spinlock_t       lock;
};
INIT_LIST_HEAD(&ctx->head);               /* dynamic */
```

The head is a *sentinel*: an empty list is a node pointing to itself. This eliminates
every NULL check in the implementation — insertion and deletion are branch-free.

### 2.2 The complete API

```c
/* Insert */
list_add(new, head);              /* at head (stack/LIFO) */
list_add_tail(new, head);         /* at tail (queue/FIFO)  ← most common */

/* Remove */
list_del(entry);                  /* poisons entry->next/prev with LIST_POISON1/2 */
list_del_init(entry);             /* re-inits so list_empty(entry) is true */

/* Move */
list_move(entry, head);
list_move_tail(entry, head);

/* Replace / rotate / splice */
list_replace(old, new);  list_replace_init(old, new);
list_rotate_left(head);
list_splice(list, head);          /* merge `list` into `head` */
list_splice_tail(list, head);
list_splice_init(list, head);     /* and re-init `list` */
list_cut_position(new, head, entry);
list_bulk_move_tail(head, first, last);

/* Test */
list_empty(head);
list_empty_careful(head);         /* safe-ish against concurrent list_del_init */
list_is_singular(head);
list_is_first(entry, head);  list_is_last(entry, head);
list_is_head(entry, head);

/* Access */
list_entry(ptr, type, member);              /* == container_of */
list_first_entry(head, type, member);       /* UNSAFE if empty */
list_first_entry_or_null(head, type, member);
list_last_entry(head, type, member);
list_next_entry(pos, member);
list_prev_entry(pos, member);

/* Iterate */
list_for_each(pos, head)                            /* pos is list_head* */
list_for_each_prev(pos, head)
list_for_each_safe(pos, n, head)                    /* safe against removal of pos */
list_for_each_entry(pos, head, member)              /* ★ the one you'll use */
list_for_each_entry_safe(pos, n, head, member)      /* ★ when deleting */
list_for_each_entry_reverse(pos, head, member)
list_for_each_entry_continue(pos, head, member)
list_for_each_entry_from(pos, head, member)
list_for_each_entry_safe_reverse(...)

/* Sort */
list_sort(priv, head, cmp);        /* lib/list_sort.c — stable merge sort, no alloc */
```

### 2.3 Canonical usage

```c
struct my_obj {
	int			id;
	struct list_head	node;
};

static LIST_HEAD(objs);
static DEFINE_SPINLOCK(objs_lock);

static int add_obj(int id)
{
	struct my_obj *o = kzalloc(sizeof(*o), GFP_KERNEL);

	if (!o)
		return -ENOMEM;
	o->id = id;
	INIT_LIST_HEAD(&o->node);

	spin_lock(&objs_lock);
	list_add_tail(&o->node, &objs);
	spin_unlock(&objs_lock);
	return 0;
}

static void drain_objs(void)
{
	struct my_obj *o, *tmp;
	LIST_HEAD(dead);

	/* Move everything to a private list under the lock... */
	spin_lock(&objs_lock);
	list_splice_init(&objs, &dead);
	spin_unlock(&objs_lock);

	/* ...then free outside the lock. Classic kernel pattern. */
	list_for_each_entry_safe(o, tmp, &dead, node) {
		list_del(&o->node);
		kfree(o);
	}
}
```

**Why `_safe`:** `list_for_each_entry` computes `pos = list_next_entry(pos, member)`
*after* the body. If the body freed `pos`, that's a use-after-free. `_safe` caches the
next pointer first.

**Why `list_del` before `kfree`:** not strictly required if the list is being destroyed,
but doing it keeps `CONFIG_DEBUG_LIST` happy and documents intent.

### 2.4 `CONFIG_DEBUG_LIST`

Turns on `__list_add_valid()` / `__list_del_entry_valid()`, which detect:
- double-add (`list_add` on an already-linked entry)
- double-del (`prev`/`next` are `LIST_POISON`)
- corrupted `prev`/`next` cross-links

```
list_add corruption. prev->next should be next (ffff...), but was ffff.... (prev=ffff...)
```
**Always on in your dev kernel.**

### 2.5 RCU list variants

```c
#include <linux/rculist.h>

list_add_rcu(new, head);
list_add_tail_rcu(new, head);
list_del_rcu(entry);                  /* does NOT poison ->next (readers may be there) */
list_replace_rcu(old, new);
list_for_each_entry_rcu(pos, head, member)   /* inside rcu_read_lock() */
list_entry_rcu(ptr, type, member);

/* Typical pattern */
rcu_read_lock();
list_for_each_entry_rcu(o, &objs, node)
	if (o->id == want) { found = o; break; }
rcu_read_unlock();

/* Removal */
spin_lock(&objs_lock);
list_del_rcu(&o->node);
spin_unlock(&objs_lock);
call_rcu(&o->rcu, obj_free_rcu);      /* or synchronize_rcu(); kfree(o); */
```

**Critical detail:** `list_del_rcu()` poisons only `prev`, not `next`, so a concurrent
reader mid-traversal can still walk forward out of the removed node. That is the whole
trick. Chapter 15 covers RCU properly.

---

## 3. `struct hlist_head` — hash-table list

`include/linux/list.h` (bottom half of the file).

```c
struct hlist_head { struct hlist_node *first; };
struct hlist_node { struct hlist_node *next, **pprev; };
```

**Why it exists:** a hash table needs an array of N heads. `list_head` heads are
2 pointers each; `hlist_head` is 1. For a 1M-bucket table that's 8 MB saved.

The trick is `pprev`: a pointer *to the pointer that points at me*. This makes
`hlist_del` O(1) without a `prev` node pointer:
```c
*(node->pprev) = node->next;
if (node->next) node->next->pprev = node->pprev;
```

### API
```c
HLIST_HEAD(name);  INIT_HLIST_HEAD(&h);  INIT_HLIST_NODE(&n);
hlist_add_head(n, h);
hlist_add_before(n, next);  hlist_add_behind(n, prev);
hlist_del(n);  hlist_del_init(n);
hlist_empty(h);  hlist_unhashed(n);
hlist_entry(ptr, type, member);
hlist_for_each_entry(pos, head, member)
hlist_for_each_entry_safe(pos, n, head, member)
/* RCU: hlist_add_head_rcu, hlist_del_rcu, hlist_for_each_entry_rcu */
```

### Hash tables
```c
#include <linux/hashtable.h>

DEFINE_HASHTABLE(my_table, 10);       /* 2^10 = 1024 buckets, static */
DECLARE_HASHTABLE(t, 8);              /* in a struct */
hash_init(t);

hash_add(my_table, &obj->hnode, key);
hash_del(&obj->hnode);
hash_for_each_possible(my_table, obj, hnode, key) { ... }   /* ★ bucket scan */
hash_for_each(my_table, bkt, obj, hnode) { ... }            /* whole table */
hash_for_each_safe(...)
hash_empty(t);
/* RCU variants: hash_add_rcu, hash_del_rcu, hash_for_each_possible_rcu */

/* Hash functions */
hash_32(val, bits);  hash_64(val, bits);  hash_ptr(p, bits);
jhash(data, len, initval);  jhash2(u32 *k, len, initval);
full_name_hash(salt, name, len);   /* for strings */
siphash(data, len, &key);          /* ★ when input is attacker-controlled */
```

> **Security note:** for any hash table keyed by untrusted input (network, filenames),
> use `siphash` with a per-boot random key, otherwise you enable hash-flooding DoS.
> See `include/linux/siphash.h` and `net_get_random_once()`.

For dynamically sized, lockless hash tables: **`rhashtable`**
(`include/linux/rhashtable.h`) — resizable, RCU-safe, used by netfilter, nft, IPC.
It's the right answer when the element count varies by orders of magnitude.

```c
static const struct rhashtable_params params = {
	.key_len     = sizeof(u32),
	.key_offset  = offsetof(struct my_obj, id),
	.head_offset = offsetof(struct my_obj, rhead),
	.automatic_shrinking = true,
};
rhashtable_init(&ht, &params);
rhashtable_lookup_fast(&ht, &key, params);
rhashtable_insert_fast(&ht, &obj->rhead, params);
rhashtable_remove_fast(&ht, &obj->rhead, params);
rhashtable_free_and_destroy(&ht, free_fn, arg);
```

---

## 4. Red-black trees — `struct rb_root`

`include/linux/rbtree.h`, `lib/rbtree.c`. Self-balancing BST: O(log n) insert/delete/search,
in-order traversal, no rebalancing allocation.

Used by: CFS/EEVDF runqueues (historically), `hrtimer` queues, ext4 extent status,
`epoll`, `i_mmap` (interval tree), deadline scheduler, memory cgroup soft-limit tree.

```c
struct rb_node {
	unsigned long  __rb_parent_color;   /* parent pointer + color in low bit */
	struct rb_node *rb_right, *rb_left;
} __attribute__((aligned(sizeof(long))));

struct rb_root { struct rb_node *rb_node; };
struct rb_root_cached { struct rb_root rb_root; struct rb_node *rb_leftmost; };
```

`rb_root_cached` caches the leftmost node — O(1) "get minimum". Use it whenever you need
a priority queue (which is most of the time).

### 4.1 The mandatory boilerplate

Unlike lists, rbtree does **not** provide search/insert — you write them, because the
comparison is yours.

```c
struct my_obj {
	struct rb_node	node;
	u64		key;
	/* payload */
};

/* SEARCH */
static struct my_obj *my_search(struct rb_root *root, u64 key)
{
	struct rb_node *n = root->rb_node;

	while (n) {
		struct my_obj *o = rb_entry(n, struct my_obj, node);

		if (key < o->key)
			n = n->rb_left;
		else if (key > o->key)
			n = n->rb_right;
		else
			return o;
	}
	return NULL;
}

/* INSERT */
static struct my_obj *my_insert(struct rb_root *root, struct my_obj *new)
{
	struct rb_node **link = &root->rb_node, *parent = NULL;

	while (*link) {
		struct my_obj *o = rb_entry(*link, struct my_obj, node);

		parent = *link;
		if (new->key < o->key)
			link = &(*link)->rb_left;
		else if (new->key > o->key)
			link = &(*link)->rb_right;
		else
			return o;             /* duplicate */
	}

	rb_link_node(&new->node, parent, link);
	rb_insert_color(&new->node, root);
	return NULL;
}

/* DELETE */
rb_erase(&obj->node, root);
RB_CLEAR_NODE(&obj->node);

/* TRAVERSE */
for (n = rb_first(root); n; n = rb_next(n)) { ... }
rb_last(root);  rb_prev(n);
rbtree_postorder_for_each_entry_safe(pos, n, root, node)   /* for destroying the tree */
```

Cached variant:
```c
struct rb_root_cached root = RB_ROOT_CACHED;
rb_insert_color_cached(&new->node, &root, leftmost_bool);
rb_erase_cached(&obj->node, &root);
rb_first_cached(&root);         /* O(1) */
```

### 4.2 Augmented rbtrees & interval trees

For "find all intervals overlapping [a,b]" — used by `i_mmap`, `mmu_notifier`, DRM MM:
```c
#include <linux/interval_tree.h>
struct interval_tree_node it;    /* .start, .last */
interval_tree_insert(&it, &root);
interval_tree_iter_first(&root, start, last);
interval_tree_iter_next(node, start, last);
```
Under the hood: an *augmented* rbtree where each node caches the max endpoint of its
subtree (`lib/interval_tree.c`, `include/linux/rbtree_augmented.h`).

### 4.3 When NOT to use rbtree

Since ~5.17, many rbtree users have migrated to the **maple tree** (Chapter 10) because:
- rbtree nodes are 24 bytes embedded per object and cache-hostile (pointer chasing)
- maple tree is a B-tree: better cache locality, RCU-safe reads, range-native

`mm/mmap.c` VMAs moved from rbtree → maple tree in 6.1. For *new* range-indexed code,
evaluate maple tree first.

---

## 5. Other list-ish structures worth knowing

| Structure | Header | Use |
|---|---|---|
| `llist` | `linux/llist.h` | **lockless** singly-linked LIFO via cmpxchg. Great for "producer in IRQ, consumer in workqueue". |
| `plist` | `linux/plist.h` | priority-sorted list, O(1) for the highest priority. Used by rt-mutex, futex. |
| `klist` | `linux/klist.h` | refcounted, node-safe list — used by the driver model so iteration is safe against removal. |
| `hlist_nulls` | `linux/list_nulls.h` | for lockless hash lookups where an item may move between buckets (TCP socket hash). The "nulls" value encodes the bucket. |
| `hlist_bl` | `linux/list_bl.h` | bit-spinlock in the head pointer — used by the dcache to save memory. |
| `rhashtable` | `linux/rhashtable.h` | resizable RCU hash table |
| `xarray` / `maple tree` | Chapter 10 | |

`llist` example (a genuinely useful pattern):
```c
static LLIST_HEAD(pending);

/* IRQ context — no locks at all */
llist_add(&item->llnode, &pending);
schedule_work(&drain_work);

/* Worker */
struct llist_node *batch = llist_del_all(&pending);
struct my_obj *o, *n;
llist_for_each_entry_safe(o, n, batch, llnode)
	process(o);
```

---

## 6. Choosing a container — decision table

| Need | Use |
|---|---|
| Ordered collection, iterate all, few elements | `list_head` |
| FIFO queue | `list_head` + `list_add_tail` |
| Lockless multi-producer queue | `llist` |
| Lookup by key, fixed-ish size | `DEFINE_HASHTABLE` + `hlist` |
| Lookup by key, wildly varying size | `rhashtable` |
| Sorted, need min/max, ordered traversal | `rb_root_cached` |
| Sparse index → pointer (id → object) | **XArray** (Ch. 10) |
| Allocate small integer IDs | **IDA/IDR** (Ch. 10) |
| Non-overlapping ranges | **maple tree** (Ch. 10) |
| Overlapping ranges | interval tree |
| Bounded ring of bytes/records | `kfifo` (Ch. 10) |
| Priority queue with O(1) top | `plist` or `rb_root_cached` |
| Bit set | `bitmap` (Ch. 10) |

---

## 7. Practice

### Lab 9.1 — Build an object registry module
Write a module that:
- keeps objects in a `list_head` protected by a spinlock,
- also indexes them in a `DEFINE_HASHTABLE` by id,
- also keeps them sorted by a `u64` timestamp in an `rb_root_cached`,
- exposes add/remove/dump via debugfs.

Deliberately do the wrong thing once (iterate with the non-`_safe` macro while freeing)
and observe the KASAN report.

### Lab 9.2 — Measure
Insert 100k objects. Time lookup by list scan vs hash vs rbtree. Use `ktime_get_ns()`.
Plot. Explain the crossover points in terms of cache lines.

### Lab 9.3 — Read real users
- `list_head`: `drivers/base/core.c` (`devices_kset`), `kernel/workqueue.c`
- `hlist`: `fs/dcache.c` (dentry hash), `net/ipv4/inet_hashtables.c`
- `rbtree`: `kernel/time/hrtimer.c` (timerqueue), `fs/ext4/extents_status.c`
- `llist`: `kernel/sched/core.c` (`ttwu_queue`), `mm/vmalloc.c`
- `plist`: `kernel/locking/rtmutex.c`
- `rhashtable`: `net/netfilter/nf_tables_api.c`

Pick two; write a paragraph on why that structure was chosen.

### Lab 9.4 — Convert list → RCU
Take Lab 9.1's list, make lookups lockless with `list_for_each_entry_rcu` and
`call_rcu`-based free. Prove correctness reasoning in comments. (Revisit after Ch. 15.)

---

## 8. Extended practice

### Lab 9.A — Measure T.3: array vs list vs rbtree vs hash

The single most useful experiment in this chapter. Build a module that stores N elements and
performs M random lookups in each structure, then plot.

```c
// SPDX-License-Identifier: GPL-2.0
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/slab.h>
#include <linux/list.h>
#include <linux/rbtree.h>
#include <linux/hashtable.h>
#include <linux/random.h>
#include <linux/ktime.h>
#include <linux/vmalloc.h>

static int n = 1000;
module_param(n, int, 0644);

struct elem {
	u32              key;
	u32              val;
	struct list_head lnode;
	struct rb_node   rnode;
	struct hlist_node hnode;
};

static LIST_HEAD(the_list);
static struct rb_root the_tree = RB_ROOT;
static DEFINE_HASHTABLE(the_hash, 10);

static void tree_insert(struct elem *e)
{
	struct rb_node **link = &the_tree.rb_node, *parent = NULL;

	while (*link) {
		struct elem *this = rb_entry(*link, struct elem, rnode);

		parent = *link;
		link = (e->key < this->key) ? &(*link)->rb_left : &(*link)->rb_right;
	}
	rb_link_node(&e->rnode, parent, link);
	rb_insert_color(&e->rnode, &the_tree);
}

static struct elem *tree_find(u32 key)
{
	struct rb_node *node = the_tree.rb_node;

	while (node) {
		struct elem *e = rb_entry(node, struct elem, rnode);

		if (key < e->key)      node = node->rb_left;
		else if (key > e->key) node = node->rb_right;
		else                   return e;
	}
	return NULL;
}

static int __init ds_bench_init(void)
{
	struct elem *arr, *e;
	u32 *keys;
	u64 t0, sum = 0;
	int i, j, M = 100000;

	arr  = vzalloc(n * sizeof(*arr));
	keys = vmalloc(M * sizeof(*keys));
	if (!arr || !keys)
		return -ENOMEM;

	for (i = 0; i < n; i++) {
		arr[i].key = i;
		arr[i].val = i * 7;
		list_add_tail(&arr[i].lnode, &the_list);
		tree_insert(&arr[i]);
		hash_add(the_hash, &arr[i].hnode, arr[i].key);
	}
	for (i = 0; i < M; i++)
		keys[i] = get_random_u32() % n;

	/* 1. array linear scan */
	t0 = ktime_get_ns();
	for (i = 0; i < M; i++)
		for (j = 0; j < n; j++)
			if (arr[j].key == keys[i]) { sum += arr[j].val; break; }
	pr_info("n=%5d array-scan : %6llu ns/lookup\n", n, (ktime_get_ns() - t0) / M);

	/* 2. array direct index (the best case, for reference) */
	t0 = ktime_get_ns();
	for (i = 0; i < M; i++)
		sum += arr[keys[i]].val;
	pr_info("n=%5d array-index: %6llu ns/lookup\n", n, (ktime_get_ns() - t0) / M);

	/* 3. list traversal */
	t0 = ktime_get_ns();
	for (i = 0; i < M; i++)
		list_for_each_entry(e, &the_list, lnode)
			if (e->key == keys[i]) { sum += e->val; break; }
	pr_info("n=%5d list-walk  : %6llu ns/lookup\n", n, (ktime_get_ns() - t0) / M);

	/* 4. rbtree */
	t0 = ktime_get_ns();
	for (i = 0; i < M; i++) {
		e = tree_find(keys[i]);
		if (e) sum += e->val;
	}
	pr_info("n=%5d rbtree     : %6llu ns/lookup\n", n, (ktime_get_ns() - t0) / M);

	/* 5. hash table */
	t0 = ktime_get_ns();
	for (i = 0; i < M; i++)
		hash_for_each_possible(the_hash, e, hnode, keys[i])
			if (e->key == keys[i]) { sum += e->val; break; }
	pr_info("n=%5d hashtable  : %6llu ns/lookup\n", n, (ktime_get_ns() - t0) / M);

	pr_info("checksum %llu\n", sum);
	vfree(keys);
	/* NOTE: arr is intentionally leaked here for brevity; free it in exit in real code */
	return 0;
}
static void __exit ds_bench_exit(void) { }
module_init(ds_bench_init); module_exit(ds_bench_exit);
MODULE_LICENSE("GPL");
```
```bash
for n in 8 16 32 64 128 256 1024 8192 65536; do
  sudo insmod ds_bench.ko n=$n; sudo rmmod ds_bench
done
dmesg | grep ds_bench
```
**Find the crossover point** where the tree/hash beats the linear scan on *your* machine.
It is usually far higher than people expect (often n ≈ 64–256). Plot it. This graph is your
evidence next time someone insists on a tree for a 20-element list.

### Lab 9.B — Watch the cache misses (T.3)

```bash
sudo perf stat -e cache-misses,cache-references,L1-dcache-load-misses,LLC-load-misses \
   -- bash -c 'sudo insmod ds_bench.ko n=65536; sudo rmmod ds_bench'

# Even better — per-structure. Wrap each loop in its own module param and run separately.
sudo perf record -e cache-misses -g -- <workload>
sudo perf report --stdio | head -30
```
Correlate cache-miss counts with the timings from Lab 9.A. The list should show ~1 miss per
element; the array scan far fewer per element.

### Lab 9.C — Implement an augmented rbtree (interval tree) (T.5)

```c
#include <linux/interval_tree.h>
/* or roll your own with: */
#include <linux/rbtree_augmented.h>

struct region {
	struct rb_node rb;
	unsigned long start, last;
	unsigned long __subtree_last;   /* the augmentation */
	const char *name;
};

#define START(n) ((n)->start)
#define LAST(n)  ((n)->last)

INTERVAL_TREE_DEFINE(struct region, rb, unsigned long, __subtree_last,
		     START, LAST, static, region_tree);

/* Now you have: region_tree_insert/remove/iter_first/iter_next */
```
Insert 1000 random intervals, then query "which regions overlap [a,b]" and verify against a
brute-force linear scan. Then read the real users:
```bash
git grep -l INTERVAL_TREE_DEFINE
$EDITOR mm/interval_tree.c include/linux/interval_tree_generic.h
```

### Lab 9.D — Prove `list_del()` poisoning earns its keep

```c
static void poison_demo(void)
{
	struct elem *e = kzalloc(sizeof(*e), GFP_KERNEL);

	list_add(&e->lnode, &the_list);
	list_del(&e->lnode);
	pr_info("after del: next=%px prev=%px\n", e->lnode.next, e->lnode.prev);
	list_del(&e->lnode);      /* double delete → Oops at LIST_POISON1 */
}
```
```bash
grep -n 'LIST_POISON' include/linux/poison.h
# Build with CONFIG_DEBUG_LIST=y and compare the diagnostic quality:
#  without: Oops at 0xdead000000000100  (you must know what that means)
#  with:    "list_del corruption, ... prev is LIST_POISON2"  (it tells you)
```
Then explain in writing: why is poisoning *better* than setting `NULL`? (Because `NULL`
deref could be confused with dozens of other bugs; a fault at `0xdead000000000100` is
unambiguously a double-`list_del`.) This is **designing your failure modes to be
self-identifying** — a technique worth reusing.

### Lab 9.E — RCU list traversal, correctly (preview of Ch. 15)

```c
/* Writer (under a lock) */
spin_lock(&lock);
list_add_rcu(&e->lnode, &the_list);
spin_unlock(&lock);

/* Reader — no lock at all */
rcu_read_lock();
list_for_each_entry_rcu(e, &the_list, lnode)
	if (e->key == key) { found = e->val; break; }
rcu_read_unlock();

/* Removal */
spin_lock(&lock);
list_del_rcu(&e->lnode);
spin_unlock(&lock);
kfree_rcu(e, rcu);
```
Build with `CONFIG_PROVE_RCU_LIST=y`, then deliberately use `list_for_each_entry()` instead
of the `_rcu` variant inside `rcu_read_lock()` and read the lockdep splat. Then explain:
why does `list_del_rcu()` poison only `prev` and leave `next` intact?

### Lab 9.F — `list_sort()` and allocation-free sorting

```c
#include <linux/list_sort.h>

static int cmp(void *priv, const struct list_head *a, const struct list_head *b)
{
	struct elem *ea = list_entry(a, struct elem, lnode);
	struct elem *eb = list_entry(b, struct elem, lnode);

	return (ea->key > eb->key) - (ea->key < eb->key);   /* branch-free 3-way compare */
}

list_sort(NULL, &the_list, cmp);
```
Then read `lib/list_sort.c`. It is a bottom-up merge sort that uses **no extra memory** and
is carefully tuned for cache behaviour (the comment explains why it merges in a specific
size order). It is one of the most elegant files in the kernel — 250 lines, worth a full
read.

---

## 9. Mastery drills

1. Explain `hlist_node::pprev` precisely. Draw the pointer diagram for a 3-element bucket
   and show `hlist_del` on the middle element.
2. Why does `list_del()` poison with `LIST_POISON1` (0x100) rather than NULL? What does
   that buy you in an Oops? (Hint: the faulting address tells you the bug class.)
3. `rb_node` stores the parent pointer and color in one `unsigned long`. Show the bit
   arithmetic in `rb_set_parent_color()`. Why is the struct `aligned(sizeof(long))`?
4. Find a place in the tree that embeds **three** different list nodes in one struct.
   Explain each list's purpose and lock.
5. `list_empty_careful()` — read the implementation and explain exactly which race it
   handles and which it does not.
6. Design: you need a per-CPU free list with occasional cross-CPU stealing. Which
   structures, which locks? Compare to how SLUB does it (`mm/slub.c`).
7. **Prove the branch-free claim (T.2).** Write a conventional `NULL`-terminated,
   non-circular doubly-linked list insert/delete. Count the branches. Then compile both
   yours and the kernel's and diff the generated assembly.
8. **AVL vs RB (T.5).** Write (in userspace) an AVL and an RB tree. Instrument rotation
   counts for 1M random inserts and 1M random deletes. Confirm the deletion asymmetry.
   Then explain why the kernel's workloads make that the deciding factor.
9. **Find the rbtree users.** `git grep -l 'rb_insert_color' | wc -l`. Pick five from
   different subsystems and, for each, state what is keyed on what and whether
   `rb_root_cached` is used (and whether it should be).
10. **Non-intrusive cost.** Write a version of the Lab 9.A benchmark using a non-intrusive
    list (allocate a node per insert). Measure insert throughput and memory. Quantify the
    T.1 argument.
11. **Multiple membership.** Find `struct page`/`struct folio`, `struct inode`, and
    `struct sock`. For each, list every list/tree/hash node embedded in it and name the
    container and lock for each. (`struct inode` has ~6. This exercise is the single best
    way to internalize T.1.)
12. **Design an LRU.** Using only `list_head` and a hash table, design an LRU cache with
    O(1) lookup, O(1) promotion, and O(1) eviction. Then compare with how the kernel's page
    reclaim does it (`mm/vmscan.c`, and why it uses *two* lists plus referenced bits rather
    than strict LRU — Ch. 23).

---

## 10. Further reading

**Kernel source & docs:**
- `include/linux/list.h` — **read the whole file** (it is ~1100 lines and mostly comments)
- `include/linux/rbtree.h`, `rbtree_augmented.h`, `lib/rbtree.c`
- `Documentation/core-api/rbtree.rst` ★ — the official tutorial, with the augmented example
- `include/linux/interval_tree_generic.h`, `mm/interval_tree.c`
- `include/linux/plist.h` + `Documentation/locking/pi-futex.rst` — priority lists
- `lib/list_sort.c` — a beautiful, allocation-free merge sort
- `Documentation/RCU/listRCU.rst`, `include/linux/rculist.h`, `include/linux/list_nulls.h`
- `include/linux/hashtable.h`, `include/linux/jhash.h`, `include/linux/hash.h`

**Algorithms:**
- Cormen, Leiserson, Rivest & Stein (CLRS), Ch. 13 (red-black trees) and Ch. 14
  (augmenting data structures — **this is exactly Linux's interval tree**)
- Sedgewick, "Left-leaning Red-Black Trees" (2008) — and why Linux does *not* use LLRB
- Guibas & Sedgewick, "A Dichromatic Framework for Balanced Trees" (FOCS 1978) — the
  original red-black paper
- Bayer, "Symmetric binary B-Trees" (1972) — the ancestor of red-black trees

**Cache-aware data structure design (T.3):**
- Drepper, "What Every Programmer Should Know About Memory" (2007) — §3 and §6
- Chilimbi, Hill & Larus, "Cache-Conscious Structure Layout" (PLDI 1999)
- Prokopec et al. and the broader "data-oriented design" literature
- Mike Acton, "Data-Oriented Design and C++" (CppCon 2014) — polemical, correct

**LWN:**
- "Trees I: Radix trees", "Trees II: red-black trees" (Corbet) — the classic intro pair
- "The maple tree" / "Replacing the VMA rbtree"
- "Linked lists, RCU, and the kernel"
- Torvalds on "good taste" in the linked-list delete (TED 2016 talk, ~14:00) — the `pprev`
  idiom in T.4

→ Next: [10-data-structures-2.md](10-data-structures-2.md)

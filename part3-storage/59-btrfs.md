# Chapter 59 — Btrfs internals: CoW, B-trees, subvolumes, RAID

> **Goal:** Understand the filesystem that treats copy-on-write not as a feature but as the foundation, and see what becomes possible — and what becomes hard — when nothing is ever overwritten in place. Understand the single unified B-tree design, how CoW makes crash consistency automatic and snapshots free, the extent back-reference problem that is btrfs's central complexity, checksums that actually verify data, the chunk/device-tree indirection that makes multi-device RAID a filesystem concern, and the `ENOSPC` and fragmentation problems that are CoW's unavoidable bill. By the end you can read `fs/btrfs/`, use `btrfs inspect-internal` fluently, and explain precisely which btrfs problems are bugs and which are consequences of the design.

---

## Theory & First Principles

### T.0 — Start here: take a snapshot of a 10 TB filesystem instantly

```bash
btrfs subvolume snapshot /data /data/.snap-$(date +%s)
# Returns in milliseconds. 10 TB. No copying.
```

That is not a trick, and it is not incremental magic. It falls directly out of **one rule**:

> **Never overwrite anything. Ever.** To change a block, write a *new* block, then update the
> pointer to it — which itself means writing a new version of the block containing that
> pointer, and so on up to the root.

```
   BEFORE                          AFTER modifying leaf L

        ROOT                       ROOT'          ROOT (old, still valid!)
       /    \                     /    \         /    \
      A      B                   A'     B  <----+      B
     / \    / \                 / \                   / \
    L   M  N   O               L'  M  (M, N, O unchanged and SHARED)
```

Modifying one leaf rewrites the path to the root — **O(log n) blocks** — and leaves the old
tree completely intact. So:

- **A snapshot is a copy of the root pointer.** That is all. Hence: instant, and O(1).
- **The final root update is a single atomic write**, so a crash leaves you either fully at
   the old tree or fully at the new one. **No journal is needed for consistency.**
- **Checksums are natural**, because every block is written fresh: the parent stores the
   child's checksum, so the tree is a Merkle tree and silent corruption is *detectable*, not
   just survivable.

**This is the third answer to Ch. 56 §T.0's atomicity problem**, and it is the most elegant.

**One more consequence, which is why btrfs is a volume manager and not just a filesystem.**
If blocks are already reference-counted and shared (that is what snapshots require), then:

| Feature | Falls out of CoW + refcounting |
|---|---|
| Snapshots | share the whole tree |
| **Reflinks** (`cp --reflink`) | share *some* extents between two files |
| **Deduplication** | point two extents at one block, bump the refcount |
| **Send/receive** | diff two trees by generation number |
| **Multi-device, RAID, rebalance** | the allocator already indirects through a chunk map |

**Which is why btrfs absorbed the layers below it.** Ch. 66–67 keep LVM/dm/MD as separate
layers under ext4/XFS; btrfs argues those layers exist only because the filesystem could not
see through them — and that a filesystem that knows about its devices can rebuild only the
blocks that are *in use*, and can repair a bad block from a mirror because it knows the
checksum failed. **A RAID layer below the filesystem cannot do either.** That is a real
end-to-end argument (Ch. 62 §T.1), not a marketing one.

**And now the costs, stated honestly, because they are severe and specific:**

| Cost | Why it follows from CoW |
|---|---|
| **Random-overwrite fragmentation** | a database rewriting 8 KiB pages in place scatters them; a 1 GiB DB file can reach 100,000 extents |
| **Write amplification** | changing one block writes log(n) blocks |
| **Free-space accounting is hard** | `df` cannot tell you what a delete will free, because blocks are shared |
| **ENOSPC is genuinely difficult** | freeing space may first *require* space to write the new tree |
| **RAID5/6 write hole** | unresolved for years; Ch. 67 §T.7 |

The database case is the one to internalize: the correct answer is `chattr +C` (disable CoW
for that file), which **turns off the very property that makes btrfs btrfs** — losing
checksums and snapshot correctness for that file. **A design whose central idea is wrong for
an important workload has to offer an escape hatch, and the escape hatch is an admission.**

```bash
btrfs filesystem usage /data           # the honest space accounting
btrfs subvolume list /data
filefrag -v /data/postgres/base/...    # watch fragmentation happen
btrfs scrub start /data                # verify every checksum
btrfs device stats /data               # per-device error counters
```

---

### T.1 One idea, taken all the way

Btrfs (Chris Mason, Oracle, 2007) rests on a single decision:

> **Never overwrite live data. Write a new copy, then atomically switch a pointer.**

Everything else follows. Ext4 and XFS use CoW nowhere (ext4) or only for shared blocks (XFS §T.7); btrfs uses it for **every block of every structure, including its own metadata.**

What falls out, for free:

| Property | Why it is free |
|---|---|
| **Crash consistency** | the old tree is intact until the superblock's root pointer is updated; a crash reverts to the previous consistent state |
| **Snapshots** | share the tree root; both trees now CoW independently |
| **Reflinks / dedup** | the extent refcounting is already there |
| **Checksums with self-healing** | the block moved anyway; write the checksum with it |
| **Online resize (both directions)** | relocate extents; the back-references let you find every pointer |
| **Multi-device without LVM** | the chunk indirection is needed for relocation anyway |

What it costs, and these are not small:

| Cost | Why it is unavoidable |
|---|---|
| **Fragmentation** | overwriting a 4 KiB block in the middle of a 1 GiB file allocates elsewhere |
| **Write amplification** | changing one block dirties its parent, grandparent, … to the root |
| **`ENOSPC` complexity** | *freeing* space may require *allocating* space |
| **Metadata volume** | back-references for every extent reference |
| **RAID5/6 write hole** | the parity update is not CoW-atomic |

**Btrfs is the most intellectually interesting filesystem in Linux and the most operationally contentious.** Both statements are true and both follow from §T.1.

### T.2 Copy-on-write, precisely

Consider a B-tree where a leaf must change:

```
BEFORE                          AFTER
      root(A)                        root(A')      root(A) still valid
      /    \                        /     \        until the superblock
   n1(B)  n2(C)                 n1(B')  n2(C)      is updated
   /  \     / \                 /   \    /  \
 L1   L2  L3  L4              L1  L2'  L3  L4
              ^                    ^
         modify L2            new block
```

Modifying L2 requires:

1. Allocate a new block, write the modified leaf → `L2'`.
2. Its parent `n1` must now point at `L2'`, so `n1` is CoW'd → `n1'`.
3. The root must point at `n1'`, so the root is CoW'd → `A'`.
4. **Finally**, write the superblock pointing at `A'`.

Step 4 is the atomic commit point. Before it, the filesystem is entirely the old tree; after it, entirely the new. **There is no partial state.** That is the entire crash-consistency story — no journal, no `fsck` after a crash, no replay.

Two immediate consequences:

**(a) Write amplification is O(tree depth).** Changing one leaf writes ~4 blocks. Btrfs mitigates this by batching: a transaction accumulates many changes and CoWs each path once, so the roots are shared across many modifications. Still, metadata write volume is materially higher than ext4's.

**(b) Old blocks cannot be freed immediately.** The old `L2` is still referenced by the old tree, which snapshots or in-flight readers may need. Freeing is deferred and refcounted — which is §T.5's problem.

The superblock update itself must be atomic and durable. Btrfs writes **multiple superblock copies** (at 64 KiB, 64 MiB, 256 GiB, 1 PiB on each device), each with a generation number and checksum, and issues a FLUSH before and a FUA for the superblock. Recovery picks the highest generation with a valid checksum. This is a careful piece of engineering and the point at which btrfs depends on the device telling the truth about flushes (Ch. 51 §T.4).

### T.3 One B-tree implementation, many trees

Like XFS, btrfs has one B-tree implementation. Unlike XFS, **every tree has the same key type**:

```c
struct btrfs_key {
	__u64 objectid;
	__u8  type;
	__u64 offset;
};
```

A 17-byte key, sorted lexicographically by `(objectid, type, offset)`. The meaning of each field depends on `type`, and this uniformity is the design's elegance: one tree implementation, one key comparison, one set of search/insert/delete routines for everything.

The trees:

| Tree | Holds |
|---|---|
| **ROOT_TREE** | the roots of all other trees — the top of the whole structure |
| **FS_TREE** (one per subvolume) | inodes, directory entries, file extents, xattrs |
| **EXTENT_TREE** | allocated extents and their **back-references** (§T.5) |
| **CHUNK_TREE** | logical → physical mapping (§T.7) |
| **DEV_TREE** | per-device extent allocation |
| **CSUM_TREE** | data checksums (§T.6) |
| **LOG_TREE** | the fsync fast path (§T.4) |
| **UUID_TREE** | subvolume UUID → id |
| **QUOTA_TREE** | qgroup accounting |
| **FREE_SPACE_TREE** | the free-space cache (v2) |
| **BLOCK_GROUP_TREE** | block group items (separated in 6.1 for mount speed) |

Within the FS_TREE, the key encodes everything:

| Item | objectid | type | offset |
|---|---|---|---|
| inode | inode number | `INODE_ITEM` | 0 |
| dir entry | parent inode | `DIR_ITEM` | hash(name) |
| dir index | parent inode | `DIR_INDEX` | sequence number |
| inode backref | inode number | `INODE_REF` | parent inode |
| file extent | inode number | `EXTENT_DATA` | file offset |
| xattr | inode number | `XATTR_ITEM` | hash(name) |

**The sort order does real work.** All items for one inode are contiguous (same objectid), sorted by type, then by offset. Reading a file's metadata is one sequential range scan. `readdir` walks `DIR_INDEX` items in sequence order (giving stable, creation-ordered offsets — unlike ext4's hash order, Ch. 57 §T.4), while `DIR_ITEM` gives hash-ordered lookup. **Two indexes over the same data, in the same tree, distinguished only by the type byte.** That is what the uniform key buys.

Nodes and leaves differ:

```c
struct btrfs_header {
	__u8 csum[BTRFS_CSUM_SIZE];
	__u8 fsid[BTRFS_FSID_SIZE];
	__le64 bytenr;            /* self-identifying: T.6 */
	__le64 flags;
	__u8 chunk_tree_uuid[BTRFS_UUID_SIZE];
	__le64 generation;
	__le64 owner;
	__le32 nritems;
	__u8 level;               /* 0 = leaf */
};

struct btrfs_item {           /* leaves: key + where the data lives */
	struct btrfs_disk_key key;
	__le32 offset;            /* from the end of the leaf */
	__le32 size;
};

struct btrfs_key_ptr {        /* nodes: key + child pointer */
	struct btrfs_disk_key key;
	__le64 blockptr;
	__le64 generation;
};
```

Leaves store items growing from the front and variable-length data growing from the back — so items of wildly different sizes coexist in one leaf. A small file's entire contents can be **inlined** into the leaf as an `EXTENT_DATA` item with `type = INLINE`, costing no data block at all.

`generation` in the key pointer is a nice detail: it records the transaction that wrote the child, letting `btrfs send` and scrub skip subtrees unchanged since a given generation. **Incremental send is O(changes), not O(filesystem size)**, purely because of this field.

### T.4 Transactions and the log tree

Btrfs batches changes into **transactions**, committed every 30 seconds (`commit=`) or on demand. A commit CoWs all dirty tree paths, writes them, flushes, then writes the superblocks.

But `fsync()` cannot wait up to 30 seconds. Forcing a full transaction commit per `fsync` would be catastrophic — it writes all dirty metadata for the whole filesystem, not just the one file.

Hence the **log tree**: a small, separate tree recording just enough to replay the fsync'd operations.

```
fsync(fd)
 └─ btrfs_log_inode()
     ├─ write the inode item, its extents, and its dir entries to the log tree
     ├─ write the log tree blocks
     ├─ FLUSH
     └─ update the log root in the superblock
        (NOT a full transaction commit)

On mount after a crash:
 └─ replay the log tree into the FS tree, then discard it
```

This is **a journal bolted onto a CoW filesystem**, and it exists purely as an `fsync` optimisation. It has historically been the most bug-prone part of btrfs, because it duplicates state that must be kept consistent with the main tree — the classic problem of having two representations of the same thing.

`btrfs_log_inode` also has a subtle obligation: if a file's *parent directory* changed in a way that matters (the file was renamed, or a hard link added), the log must capture enough to reconstruct the namespace. The rules here are intricate and a recurring source of "file disappeared after a crash" bugs.

**Mount options worth knowing:**

- `commit=N` — transaction interval. Lower means less data at risk and more write amplification.
- `notreelog` — disable the log tree; every `fsync` becomes a full commit. Slow but removes a bug class.
- `flushoncommit` — force data out on every commit.

### T.5 Back-references: btrfs's central complexity

Here is the hardest part of btrfs, and it is worth understanding properly.

Because extents can be shared (snapshots, reflinks) and because relocation (resize, balance, defrag) must update every pointer to an extent, btrfs must be able to answer:

> **Given physical extent E, who references it?**

XFS answers this with `rmapbt` (Ch. 58 §T.7), a simple physical → owner map. Btrfs's answer is more complicated, because of snapshots.

Consider: a subvolume is snapshotted. The snapshot shares the entire tree. Every extent in the original is now referenced by *two* trees — but no per-extent record was written, because the whole point of a snapshot is that it is O(1). So a back-reference cannot enumerate individual referrers.

Btrfs's solution is **two kinds of back-reference**:

| Kind | Records | Used when |
|---|---|---|
| **Normal** (`EXTENT_DATA_REF`, `TREE_BLOCK_REF`) | `(root, objectid, offset)` — the exact referrer | the extent was allocated by this tree |
| **Shared** (`SHARED_DATA_REF`, `SHARED_BLOCK_REF`) | `(parent block)` — only the immediate parent | the reference was inherited via snapshot |

A shared back-reference says "some tree block at address P points at me," without saying which subvolume. To find the actual owners you must walk *up* from P, and there may be several paths. This is why:

- `btrfs inspect-internal logical-resolve` can be slow and can return many paths.
- Quota groups (qgroups) are expensive: correctly attributing a shared extent requires exactly this walk.
- The `FIEMAP` `SHARED` flag is expensive to compute on btrfs.
- Balance and device removal are slow: every extent must be resolved and every referrer updated.

The extent tree records, per extent:

```c
struct btrfs_extent_item {
	__le64 refs;                  /* total reference count */
	__le64 generation;
	__le64 flags;                 /* DATA or TREE_BLOCK */
	/* followed by inline back-references */
};

struct btrfs_extent_inline_ref {
	__u8 type;                    /* which of the four kinds */
	__le64 offset;
};
```

Small reference counts are stored **inline in the extent item**; when there are too many, they spill into separate items. Same small-case optimisation as everywhere else, but here it matters enormously because the common case (one referrer) must be cheap.

**This is the price of making snapshots O(1).** XFS's reflink is O(extents) to create because it writes a refcount record per extent; btrfs's snapshot is O(1) because it writes nothing. Btrfs pays later, during relocation and quota accounting. It is a genuine trade, not a mistake, but it is the source of most of btrfs's performance surprises.

### T.6 Checksums: the thing ext4 and XFS do not do

**Btrfs checksums every data block and every metadata block, and verifies on every read.**

```c
struct btrfs_header {
	__u8 csum[BTRFS_CSUM_SIZE];   /* covers the rest of the block */
	__u8 fsid[BTRFS_FSID_SIZE];   /* right filesystem? */
	__le64 bytenr;                /* right LOCATION? */
	...
	__le64 generation;            /* right VERSION? */
};
```

Four checks, not one:

| Field | Catches |
|---|---|
| `csum` | bit rot, torn writes, bad cables, bad RAM en route |
| `fsid` | a block from a *different* filesystem |
| `bytenr` | a **misdirected write** — the block is valid but in the wrong place |
| `generation` | a **lost write** — a valid but stale block (the write never reached the disk) |

Misdirected and lost writes are real and are exactly what a plain checksum misses: the data is internally consistent, just wrong. Only self-identifying metadata catches them. (XFS v5 does this too, for metadata; ext4's `metadata_csum` seeds by inode and UUID, which catches relocation but not staleness.)

Data checksums live in the **CSUM_TREE**, keyed by logical address — separate from the data, so a single bad region cannot corrupt both. Default algorithm is CRC32c; `mkfs.btrfs --csum` offers xxhash64 (fast, 64-bit), blake2b, and sha256 (cryptographic).

Storage cost: 4 bytes per 4 KiB = **0.1 %**. Utterly negligible. There is no good reason not to have this, which makes its absence from ext4 and XFS a genuine gap rather than a reasonable trade.

**And with redundancy, btrfs self-heals.** On a RAID1 filesystem:

```
read block -> checksum fails -> read the other copy -> checksum passes
           -> return the good data
           -> REWRITE the bad copy
           -> log "csum error at logical X, fixed"
```

This is the capability that makes btrfs and ZFS categorically different. A single-device ext4 with bit rot returns garbage silently forever. Btrfs on RAID1 detects it, returns correct data, and repairs the disk.

Caveat worth knowing: `nodatacow` files (see §T.9) have **no checksums**, because there is no atomic way to update data and checksum in place. So the databases and VM images most likely to be set `nodatacow` for performance are exactly the files that lose integrity checking. This tension is real and has no clean resolution.

### T.7 Chunks: why btrfs subsumes the volume manager

Btrfs has three address spaces:

```
file offset  --(FS tree, EXTENT_DATA)-->  logical address
logical      --(CHUNK tree)-->            (device, physical) × N copies
```

A **chunk** is a contiguous logical range (typically 1 GiB for data, 256 MiB for metadata) mapped to one or more physical ranges according to a profile:

| Profile | Copies | Stripes | Space efficiency | Survives |
|---|---|---|---|---|
| `single` | 1 | 1 | 100 % | nothing |
| `dup` | 2 | 1 | 50 % | bad sectors on one device |
| `raid0` | 1 | N | 100 % | nothing |
| `raid1` | 2 | 1 | 50 % | 1 device |
| `raid1c3`/`raid1c4` | 3/4 | 1 | 33 %/25 % | 2/3 devices |
| `raid10` | 2 | N | 50 % | 1 device per mirror |
| `raid5`/`raid6` | 1 | N−1/N−2 | high | 1/2 devices — **but see §T.9** |

Three things make this different from MD RAID beneath a filesystem:

**(a) Data and metadata can have different profiles.** `mkfs.btrfs -d raid0 -m raid1` gives fast data and redundant metadata — a sensible configuration that is impossible with a block-level RAID.

**(b) Allocation is per-chunk, so devices can be different sizes.** Btrfs allocates the next chunk wherever there is room, honouring the profile. Adding a 2 TB disk to a pool of 1 TB disks *works*, which MD RAID cannot do.

**(c) Only allocated chunks are replicated/scrubbed.** A scrub reads only live data, not the whole device. On a half-empty array this halves the scrub time.

And the chunk indirection is what makes **online reshaping** possible:

```sh
btrfs balance start -dconvert=raid1 -mconvert=raid1 /mnt
btrfs device add /dev/sdc /mnt
btrfs device remove /dev/sdb /mnt     # relocates chunks off sdb first
btrfs filesystem resize -10G /mnt     # SHRINK -- XFS cannot do this
```

Every one of these is "relocate chunks, update the chunk tree, update the back-references." The shrink is notable: Ch. 58 §T.10 listed it as XFS's most-cited limitation, and btrfs has it because the chunk indirection plus back-references make relocation tractable.

There is a bootstrap problem: to read the chunk tree you must map its logical address, which requires the chunk tree. Btrfs solves it with a **system chunk array embedded in the superblock** — enough of the chunk tree to find the rest. The same shape as XFS's AGFL (Ch. 58 §1.2): a small reserve breaking a circular dependency.

### T.8 Subvolumes and snapshots

A **subvolume** is an independent FS_TREE with its own root in the ROOT_TREE. It is a namespace, not a partition: subvolumes share the filesystem's free space entirely.

```sh
btrfs subvolume create /mnt/sv
btrfs subvolume snapshot /mnt/sv /mnt/snap      # writable, O(1)
btrfs subvolume snapshot -r /mnt/sv /mnt/ro     # read-only
```

A snapshot copies **one tree root**. Both trees now point at the same blocks; both CoW independently. Creation is O(1) regardless of size — a 10 TB subvolume snapshots instantly.

Deletion is *not* O(1): it must walk the tree, decrement refcounts, and free what reaches zero. This is done in the background by the cleaner thread, which is why `btrfs subvolume delete` returns immediately but space appears slowly.

Subvolumes are the foundation for:

- **`btrfs send`/`receive`** — serialise a read-only snapshot, or the *difference* between two snapshots. Incremental send uses the `generation` field (§T.3) to skip unchanged subtrees, so it is O(changes). This makes btrfs an excellent backup and replication substrate.
- **Snapshot-based rollback** — the pattern `snapper`, `timeshift`, and openSUSE's transactional updates use: snapshot before an update, roll back by changing the default subvolume and rebooting.
- **Per-subvolume mount options** — `compress`, `nodatacow` can differ per subvolume.
- **Container images** — snapshot-per-layer, which is what the btrfs docker/podman driver does.

**Quota groups (qgroups)** track per-subvolume usage, and they are expensive for §T.5's reason: correctly attributing a shared extent requires resolving its back-references. Enabling qgroups on a large filesystem with many snapshots can make commits dramatically slower. This is the single most common cause of "btrfs got slow" reports. `squota` (6.7+) trades exactness for speed and is a reasonable alternative.

### T.9 The costs, honestly

**(a) Fragmentation.** In-place overwrite is impossible. A database file or VM image receiving random 4 KiB writes accumulates extents without bound — tens of thousands of extents on a file that ext4 would keep in one.

Mitigations, all partial:
- `chattr +C` (`nodatacow`) on the file or directory: overwrite in place. **But this disables checksums and CoW for that file**, so snapshots of it become expensive and integrity checking is gone.
- `autodefrag` mount option: detect random writes and rewrite the region. Helps, costs I/O, and interacts badly with snapshots (defragmenting a shared extent *unshares* it, which can massively increase space usage).
- Periodic `btrfs filesystem defragment`. Same unsharing hazard.

This is CoW's bill, and it is why btrfs's reputation for database and VM workloads is poor. It is not a bug.

**(b) `ENOSPC`.** Btrfs's hardest operational problem, and the reason is genuinely interesting: **freeing space may require allocating space.** Deleting a file modifies the extent tree, which CoWs blocks, which requires allocation. If metadata chunks are full and no unallocated space remains, you cannot delete anything.

Compounding it:
- Space is allocated in **chunks**, so `df` shows unallocated space that cannot be used for metadata if all chunks are data chunks.
- `btrfs filesystem df` and `btrfs filesystem usage` show the truth; plain `df` is misleading.
- **Global reserve** (`GlobalReserve` in `btrfs fi df`) is a metadata reserve specifically to break this deadlock — the same pattern as the AGFL and the system chunk array.

Modern btrfs handles this far better than it did in 2014, but the failure mode still exists:

```sh
btrfs filesystem usage /mnt         # the real picture
btrfs balance start -dusage=10 /mnt # compact sparse data chunks
```

**(c) RAID5/6 write hole.** Parity RAID updates parity in place — it is not CoW. A crash during a stripe update can leave data and parity inconsistent, and a subsequent device failure then reconstructs *wrong data*. Btrfs's RAID5/6 has long-standing warnings in the documentation and is **not recommended for production**. `raid1c3`/`raid1c4` for metadata with `raid5` data is a partial mitigation; the honest answer is to use RAID10 or raid1c3, or put btrfs on MD RAID6.

**(d) Metadata volume.** Back-references, checksums, and CoW's amplification mean btrfs writes substantially more metadata than ext4 or XFS. On small files this can dominate.

**(e) Performance variance.** Transaction commits, the cleaner thread, background qgroup accounting, and balance operations all produce latency spikes. For workloads needing predictable p99, this matters.

### T.10 Where btrfs is the right answer

Being fair, after §T.9:

| Use case | Why btrfs |
|---|---|
| **Workstation / laptop root** | snapshots before updates, rollback, compression, checksums |
| **Backup target** | `send`/`receive` incremental replication is excellent |
| **Container image storage** | snapshot-per-layer is exactly the model |
| **Any data you care about on a single machine** | it is the only mainline filesystem that detects bit rot |
| **Multi-device with mismatched disks** | nothing else handles this well |
| **NAS with RAID1/RAID10** | checksums + self-healing + online reshaping |

| Use case | Prefer something else |
|---|---|
| Database (PostgreSQL, MySQL) | fragmentation; use XFS, or btrfs with `nodatacow` and accept the losses |
| VM image store | same; or use raw devices |
| Very large single filesystem, metadata-heavy | XFS |
| RAID5/6 | MD RAID beneath XFS, or ZFS |
| Predictable p99 latency | XFS |

`compress=zstd` deserves a specific mention: transparent compression is a genuine win on typical workstation data (source trees, logs, documents), often 30–50 % space saved *and* faster reads because less I/O happens. `compress-force=zstd:1` is a reasonable default for a laptop.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `fs/btrfs/ctree.h` ★★★ | every on-disk structure and key type |
| `fs/btrfs/ctree.c` ★★★ | the B-tree: `btrfs_search_slot`, `btrfs_cow_block`, split/merge |
| `fs/btrfs/extent-tree.c` ★★★ | allocation, back-references, `ENOSPC` |
| `fs/btrfs/backref.c` ★★★ | §T.5's resolution — the hardest file |
| `fs/btrfs/volumes.c` ★★★ | §T.7's chunks, device management, RAID |
| `fs/btrfs/transaction.c` | §T.4 |
| `fs/btrfs/tree-log.c` ★★★ | the log tree; historically the most bug-prone file |
| `fs/btrfs/disk-io.c` | superblock read/write, tree root management, verification |
| `fs/btrfs/inode.c` | the VFS interface, the CoW write path |
| `fs/btrfs/file.c` | `write_iter`, prealloc, `nodatacow` |
| `fs/btrfs/file-item.c` | §T.6's checksums |
| `fs/btrfs/scrub.c` | background verification and repair |
| `fs/btrfs/relocation.c` | balance, shrink, device removal |
| `fs/btrfs/send.c`, `qgroup.c`, `compression.c`, `raid56.c` | as named |
| `fs/btrfs/free-space-tree.c` | the v2 space cache |
| `Documentation/filesystems/btrfs.rst` | brief; the wiki is the real documentation |

### 1.2 The key, and what it does

```c
struct btrfs_key {
	__u64 objectid;
	__u8  type;
	__u64 offset;
};

/* Selected types -- the full list is in ctree.h */
#define BTRFS_INODE_ITEM_KEY		1
#define BTRFS_INODE_REF_KEY		12
#define BTRFS_INODE_EXTREF_KEY		13
#define BTRFS_XATTR_ITEM_KEY		24
#define BTRFS_ORPHAN_ITEM_KEY		48
#define BTRFS_DIR_LOG_ITEM_KEY		60
#define BTRFS_DIR_ITEM_KEY		84
#define BTRFS_DIR_INDEX_KEY		96
#define BTRFS_EXTENT_DATA_KEY		108
#define BTRFS_EXTENT_CSUM_KEY		128
#define BTRFS_ROOT_ITEM_KEY		132
#define BTRFS_EXTENT_ITEM_KEY		168
#define BTRFS_METADATA_ITEM_KEY		169
#define BTRFS_EXTENT_DATA_REF_KEY	178
#define BTRFS_SHARED_DATA_REF_KEY	184
#define BTRFS_BLOCK_GROUP_ITEM_KEY	192
#define BTRFS_DEV_EXTENT_KEY		204
#define BTRFS_CHUNK_ITEM_KEY		228
```

The type values are chosen so that the *sort order* groups related items usefully: for one inode, `INODE_ITEM`(1) sorts before `INODE_REF`(12) before `XATTR_ITEM`(24) before `DIR_ITEM`(84) before `EXTENT_DATA`(108). Reading a file's metadata is one contiguous scan; reading a directory's entries is a contiguous range. **The numeric values of the type constants are part of the design.**

### 1.3 CoW in code

```c
int btrfs_cow_block(struct btrfs_trans_handle *trans,
		    struct btrfs_root *root, struct extent_buffer *buf,
		    struct extent_buffer *parent, int parent_slot,
		    struct extent_buffer **cow_ret,
		    enum btrfs_lock_nesting nest)
{
	u64 search_start;
	int ret;
	...
	/* Already CoW'd in THIS transaction? Then it is private to us
	 * and can be modified in place -- the key optimisation. */
	if (btrfs_header_generation(buf) == trans->transid &&
	    !btrfs_header_flag(buf, BTRFS_HEADER_FLAG_WRITTEN) &&
	    !(root->root_key.objectid != BTRFS_TREE_RELOC_OBJECTID &&
	      btrfs_header_flag(buf, BTRFS_HEADER_FLAG_RELOC))) {
		*cow_ret = buf;
		return 0;
	}

	search_start = round_down(buf->start, SZ_1G);
	ret = __btrfs_cow_block(trans, root, buf, parent, parent_slot,
				cow_ret, search_start, 0, nest);
	return ret;
}
```

The generation check is what makes CoW affordable: **a block CoW'd earlier in the same transaction is modified in place.** So a transaction touching a leaf 1000 times CoWs it once. This is the same coalescing insight as XFS's CIL (Ch. 58 §T.6), applied to blocks rather than log items.

The commit:

```c
int btrfs_commit_transaction(struct btrfs_trans_handle *trans)
{
	...
	/* 1. Everyone joins or waits */
	wait_for_commit(cur_trans, TRANS_STATE_COMMIT_START);

	/* 2. Flush delayed items, delayed refs, delalloc */
	btrfs_run_delayed_items(trans);
	btrfs_run_delayed_refs(trans, U64_MAX);
	btrfs_start_delalloc_flush(fs_info);

	/* 3. CoW and write every dirty tree */
	ret = commit_cowonly_roots(trans);

	/* 4. Write the tree blocks and WAIT */
	ret = btrfs_write_and_wait_transaction(trans);

	/* 5. THE atomic commit point: write the superblocks with FUA */
	ret = write_all_supers(fs_info, 0);

	/* 6. Only now can the old blocks be freed */
	btrfs_finish_extent_commit(trans);
	...
}
```

Step 5 is §T.2's atomic switch. Everything before it is reversible; nothing after it is.

### 1.4 Back-reference resolution

```c
/* Given a logical address, find every inode/offset that references it. */
int iterate_extent_inodes(struct btrfs_backref_walk_ctx *ctx,
			  bool search_commit_root,
			  iterate_extent_inodes_t *iterate, void *user_ctx)
{
	struct ulist *refs;
	struct ulist *roots;
	struct ulist_node *ref_node;
	struct ulist_node *root_node;
	...
	/* Find all leaves referencing this extent... */
	ret = btrfs_find_all_leafs(ctx);
	...
	while ((ref_node = ulist_next(ctx->refs, &ref_uiter))) {
		/* ...then, for each leaf, find every ROOT that reaches it.
		 * This is the expensive part: a shared ref names only the
		 * parent block, so we must walk UP, possibly many paths. */
		ret = btrfs_find_all_roots(ctx);
		...
		while ((root_node = ulist_next(ctx->roots, &uiter))) {
			ret = iterate_leaf_refs(ctx->fs_info, ref_node->aux,
						root_node->val, ...);
		}
	}
	...
}
```

The nested walk — leaves, then roots per leaf — is §T.5's cost made explicit. This function backs `logical-resolve`, `FIEMAP`'s `SHARED` flag, qgroup accounting, scrub's error reporting, and relocation. It is why all of those are slow.

### 1.5 Checksum verification

```c
static int check_data_csum(struct inode *inode, struct btrfs_bio *bbio,
			   u32 bio_offset, struct page *page, u32 pgoff)
{
	struct btrfs_fs_info *fs_info = inode_to_fs_info(inode);
	SHASH_DESC_ON_STACK(shash, fs_info->csum_shash);
	u8 csum[BTRFS_CSUM_SIZE];
	u8 *csum_expected;
	...
	csum_expected = bbio->csum + (bio_offset >> fs_info->sectorsize_bits) *
				     fs_info->csum_size;

	kaddr = kmap_local_page(page) + pgoff;
	shash->tfm = fs_info->csum_shash;
	crypto_shash_digest(shash, kaddr, fs_info->sectorsize, csum);
	kunmap_local(kaddr);

	if (memcmp(csum, csum_expected, fs_info->csum_size))
		goto zeroit;
	return 0;

zeroit:
	btrfs_print_data_csum_error(BTRFS_I(inode), start, csum,
				    csum_expected, bbio->mirror_num);
	/* The bio layer will now RETRY on the next mirror (T.6). */
	return -EIO;
}
```

The retry-on-next-mirror logic lives in `fs/btrfs/bio.c`'s `btrfs_check_read_bio` / repair path: on a checksum failure it reads each remaining mirror, verifies, returns the good data, and writes it back over the bad copy. Self-healing in about 100 lines, made possible entirely by having a checksum and a second copy.

### 1.6 Observability surface

| Where | What |
|---|---|
| `btrfs filesystem usage MNT` ★★★ | the **only** correct view of space |
| `btrfs filesystem df MNT` | per-profile allocation |
| `btrfs filesystem show` | devices |
| `btrfs device stats MNT` ★★★ | per-device error counters (write/read/flush/corruption/generation) |
| `btrfs subvolume list -t MNT` | subvolumes and snapshots |
| `btrfs inspect-internal dump-super -f DEV` ★★★ | the superblock |
| `btrfs inspect-internal dump-tree DEV` ★★★ | **every tree, item by item** |
| `btrfs inspect-internal logical-resolve` | §T.5's resolution, from the command line |
| `btrfs inspect-internal inode-resolve` | inode → path |
| `btrfs scrub start/status MNT` ★★★ | verify all checksums |
| `btrfs balance status MNT` | relocation progress |
| `btrfs check --readonly DEV` | offline consistency check |
| `compsize` / `btrfs filesystem du` | compression ratio, shared/exclusive usage |
| `/sys/fs/btrfs/UUID/` | allocation, features, tunables |
| `trace-cmd record -e btrfs:\*` ★★★ | ~200 tracepoints |

`btrfs inspect-internal dump-tree` is remarkable: it prints the *entire* on-disk structure in human-readable form. Nothing equivalent exists for ext4 or XFS. Lab 59.2 uses it heavily.

---

## 2. Practice

### Lab 59.1 — CoW, observed

```sh
sudo apt install -y btrfs-progs
sudo modprobe scsi_debug dev_size_mb=4096
DEV=$(lsblk -ndo NAME,MODEL | awk '/scsi_debug/{print "/dev/"$1}')

sudo mkfs.btrfs -f $DEV
sudo mkdir -p /mnt/btr && sudo mount $DEV /mnt/btr
sudo btrfs filesystem usage /mnt/btr
```

Prove that overwriting allocates new blocks:

```sh
sudo dd if=/dev/urandom of=/mnt/btr/f bs=1M count=10 2>/dev/null
sync
echo "--- original ---"
sudo filefrag -v /mnt/btr/f | head -5

sudo dd if=/dev/urandom of=/mnt/btr/f bs=4k count=1 seek=500 conv=notrunc 2>/dev/null
sync
echo "--- after overwriting one block in the middle ---"
sudo filefrag -v /mnt/btr/f | head -8
```

The block moved. On ext4 or XFS it would not have. **That is §T.1, in one command.**

Watch the amplification:

```sh
sudo trace-cmd record -e btrfs:btrfs_cow_block -e block:block_rq_issue -- \
  sudo sh -c 'dd if=/dev/zero of=/mnt/btr/g bs=4k count=1 conv=notrunc 2>/dev/null; sync'
sudo trace-cmd report | grep -c btrfs_cow_block
# More than one: the whole path to the root was CoW'd.
sudo trace-cmd report | grep btrfs_cow_block | head
```

Generation numbers advancing:

```sh
for i in 1 2 3; do
  sudo sh -c "echo $i > /mnt/btr/gen; sync"
  echo -n "commit $i: generation="
  sudo btrfs inspect-internal dump-super $DEV | grep '^generation'
done
```

The superblock copies:

```sh
sudo btrfs inspect-internal dump-super -a -f $DEV | grep -E '^(superblock|generation|bytenr)'
# Multiple copies at 64K, 64M, 256G -- T.2
```

Fragmentation, the cost:

```sh
sudo dd if=/dev/zero of=/mnt/btr/db bs=1M count=200 2>/dev/null
sync
echo -n "initial extents: "; sudo filefrag /mnt/btr/db

# Random 4 KiB overwrites, like a database
sudo python3 -c "
import os, random
fd = os.open('/mnt/btr/db', os.O_RDWR)
for _ in range(5000):
    os.pwrite(fd, b'x'*4096, random.randrange(0, 200*1024*1024, 4096))
os.fsync(fd); os.close(fd)"
echo -n "after 5000 random writes: "; sudo filefrag /mnt/btr/db
```

Thousands of extents. Now the mitigations:

```sh
# nodatacow
sudo touch /mnt/btr/db2
sudo chattr +C /mnt/btr/db2
lsattr /mnt/btr/db2
sudo dd if=/dev/zero of=/mnt/btr/db2 bs=1M count=200 2>/dev/null
sync
sudo python3 -c "
import os, random
fd = os.open('/mnt/btr/db2', os.O_RDWR)
for _ in range(5000):
    os.pwrite(fd, b'x'*4096, random.randrange(0, 200*1024*1024, 4096))
os.fsync(fd); os.close(fd)"
echo -n "nodatacow extents: "; sudo filefrag /mnt/btr/db2

# But: no checksums
sudo btrfs inspect-internal dump-tree -t 7 $DEV 2>/dev/null | grep -c EXTENT_CSUM
# Compare the csum items for db vs db2.

# autodefrag
sudo mount -o remount,autodefrag /mnt/btr
# repeat; then:
sudo btrfs filesystem defragment -r -v /mnt/btr/db
sudo filefrag /mnt/btr/db
```

---

### Lab 59.2 — Read the trees

```sh
sudo umount /mnt/btr
sudo btrfs inspect-internal dump-super -f $DEV | head -40
```

Every tree, by number:

```sh
sudo btrfs inspect-internal dump-tree -t root  $DEV | head -40   # ROOT_TREE
sudo btrfs inspect-internal dump-tree -t chunk $DEV | head -40   # T.7
sudo btrfs inspect-internal dump-tree -t extent $DEV | head -60  # T.5
sudo btrfs inspect-internal dump-tree -t fs    $DEV | head -80   # FS_TREE
sudo btrfs inspect-internal dump-tree -t csum  $DEV | head -20   # T.6
sudo btrfs inspect-internal dump-tree -t dev   $DEV | head -20
```

Find one file's items and see §T.3's sort order:

```sh
sudo mount $DEV /mnt/btr
sudo sh -c 'echo "hello btrfs" > /mnt/btr/readme'
sudo setfattr -n user.note -v "an xattr" /mnt/btr/readme
sync
INO=$(stat -c %i /mnt/btr/readme)
sudo umount /mnt/btr

sudo btrfs inspect-internal dump-tree -t fs $DEV | grep -A3 "key ($INO "
```

```
item N key (257 INODE_ITEM 0) itemoff ... itemsize 160
item N+1 key (257 INODE_REF 256) ...
item N+2 key (257 XATTR_ITEM 0x...) ...
item N+3 key (257 EXTENT_DATA 0) itemoff ... itemsize 25
        inline extent data size 12 ...
```

Note: all items for inode 257, in type order, contiguous. And the file content is **inline** — no data block at all, because it is 12 bytes.

The two directory indexes:

```sh
sudo mount $DEV /mnt/btr
sudo mkdir /mnt/btr/d
sudo sh -c 'for i in a b c d e; do : > /mnt/btr/d/$i; done'
sync
DINO=$(stat -c %i /mnt/btr/d)
sudo umount /mnt/btr

sudo btrfs inspect-internal dump-tree -t fs $DEV | grep "($DINO DIR_ITEM"
sudo btrfs inspect-internal dump-tree -t fs $DEV | grep "($DINO DIR_INDEX"
```

`DIR_ITEM` is keyed by name hash (for lookup); `DIR_INDEX` by sequence (for stable `readdir` order). **Two indexes over the same data, in one tree, distinguished by the type byte.** Compare ext4's htree, which has only the hash index and therefore hash-ordered `readdir` (Ch. 57 §T.4).

Confirm the ordering difference:

```sh
sudo mount $DEV /mnt/btr
sudo sh -c 'cd /mnt/btr/d && for i in $(seq 1 20); do : > f$i; done'
ls -f /mnt/btr/d                # creation order, unlike ext4
```

---

### Lab 59.3 — Snapshots

```sh
sudo btrfs subvolume create /mnt/btr/sv
sudo dd if=/dev/urandom of=/mnt/btr/sv/data bs=1M count=500 2>/dev/null
sync
sudo btrfs filesystem usage /mnt/btr | grep -E 'Used|Free'

# O(1) snapshot
time sudo btrfs subvolume snapshot /mnt/btr/sv /mnt/btr/snap1
sudo btrfs filesystem usage /mnt/btr | grep -E 'Used|Free'   # unchanged

sudo btrfs subvolume list -t /mnt/btr
```

Now diverge them:

```sh
sudo dd if=/dev/urandom of=/mnt/btr/snap1/data bs=1M count=10 seek=100 conv=notrunc 2>/dev/null
sync
sudo btrfs filesystem usage /mnt/btr | grep Used    # +10 MiB, not +500

sudo btrfs filesystem du -s /mnt/btr/sv/data /mnt/btr/snap1/data
#      Total   Exclusive  Set shared  Filename
```

`Exclusive` is the space freed by deleting this file alone — computed via §T.5's back-reference walk.

Snapshot chains, and the cost:

```sh
for i in $(seq 1 50); do
  sudo btrfs subvolume snapshot /mnt/btr/sv /mnt/btr/s$i > /dev/null
  sudo dd if=/dev/urandom of=/mnt/btr/sv/data bs=4k count=100 \
          seek=$((i*100)) conv=notrunc 2>/dev/null
done
sync
sudo btrfs filesystem usage /mnt/btr
sudo btrfs subvolume list /mnt/btr | wc -l

# Deletion is NOT O(1)
time sudo btrfs subvolume delete /mnt/btr/s1     # returns fast
sudo btrfs filesystem usage /mnt/btr | grep Used # space returns slowly
sleep 10
sudo btrfs filesystem usage /mnt/btr | grep Used
```

`send`/`receive`:

```sh
sudo btrfs subvolume snapshot -r /mnt/btr/sv /mnt/btr/base
sudo btrfs send /mnt/btr/base > /tmp/full.send
ls -lh /tmp/full.send

sudo dd if=/dev/urandom of=/mnt/btr/sv/data bs=1M count=5 seek=200 conv=notrunc 2>/dev/null
sync
sudo btrfs subvolume snapshot -r /mnt/btr/sv /mnt/btr/inc

# Incremental: only the difference (T.3's generation field)
sudo btrfs send -p /mnt/btr/base /mnt/btr/inc > /tmp/inc.send
ls -lh /tmp/full.send /tmp/inc.send
```

The incremental stream is a few MiB against a 500 MiB full stream. **That is O(changes), and it is why btrfs is an excellent backup substrate.**

Receive it:

```sh
sudo mkfs.btrfs -f $DEV2 && sudo mount $DEV2 /mnt/btr2
sudo btrfs receive /mnt/btr2 < /tmp/full.send
sudo btrfs receive /mnt/btr2 < /tmp/inc.send
sudo btrfs subvolume list /mnt/btr2
diff <(sudo md5sum /mnt/btr/inc/data | cut -d' ' -f1) \
     <(sudo md5sum /mnt/btr2/inc/data | cut -d' ' -f1) && echo "identical"
```

Qgroups, and their cost:

```sh
sudo btrfs quota enable /mnt/btr
sudo btrfs quota rescan -w /mnt/btr
sudo btrfs qgroup show -re /mnt/btr

# Measure the commit slowdown
sudo btrfs quota disable /mnt/btr
echo -n "qgroups off: "
sudo /usr/bin/time -f '%e s' sh -c 'for i in $(seq 1 3000); do : > /mnt/btr/q$i; done; sync'
sudo rm -f /mnt/btr/q*

sudo btrfs quota enable /mnt/btr && sudo btrfs quota rescan -w /mnt/btr
echo -n "qgroups on:  "
sudo /usr/bin/time -f '%e s' sh -c 'for i in $(seq 1 3000); do : > /mnt/btr/q$i; done; sync'
```

---

### Lab 59.4 — Checksums catch what nothing else does

This is the lab that distinguishes btrfs.

```sh
sudo umount /mnt/btr
sudo mkfs.btrfs -f $DEV && sudo mount $DEV /mnt/btr

sudo sh -c 'echo "IMPORTANT DATA THAT MUST NOT CHANGE" > /mnt/btr/critical'
sync
md5sum /mnt/btr/critical

# Find its physical location
sudo filefrag -v /mnt/btr/critical
PHYS=$(sudo filefrag -v /mnt/btr/critical | awk '/^ *0:/{print $4}' | tr -d '.')
sudo umount /mnt/btr

# Corrupt it
sudo dd if=/dev/urandom of=$DEV bs=4096 seek=$PHYS count=1 conv=notrunc 2>/dev/null

sudo mount $DEV /mnt/btr
cat /mnt/btr/critical
```

```
cat: /mnt/btr/critical: Input/output error
```

```sh
dmesg | tail -5
# BTRFS warning: csum failed root 5 ino 257 off 0 csum 0x... expected csum 0x...
sudo btrfs device stats /mnt/btr
#   corruption_errs 1
```

**Compare Ch. 57 Lab 57.7 and Ch. 58 Lab 58.7, where the same corruption was returned silently as garbage.** This is the single most important practical difference in Part 3.

Now self-healing:

```sh
sudo umount /mnt/btr
sudo mkfs.btrfs -f -d raid1 -m raid1 $DEV $DEV2
sudo mount $DEV /mnt/btr

sudo sh -c 'echo "REDUNDANT IMPORTANT DATA" > /mnt/btr/critical'
sync
sudo umount /mnt/btr

# Corrupt ONE copy
sudo mount $DEV /mnt/btr
PHYS=$(sudo filefrag -v /mnt/btr/critical | awk '/^ *0:/{print $4}' | tr -d '.')
sudo umount /mnt/btr
sudo dd if=/dev/urandom of=$DEV bs=4096 seek=$PHYS count=1 conv=notrunc 2>/dev/null

sudo mount $DEV /mnt/btr
cat /mnt/btr/critical          # CORRECT DATA
dmesg | tail -5
# "csum failed ... " then
# "read error corrected: ino 257 off 0 (dev /dev/... sector ...)"
sudo btrfs device stats /mnt/btr
```

**The data was returned correctly and the bad copy was repaired.** That is the capability.

Scrub:

```sh
sudo btrfs scrub start -B /mnt/btr
sudo btrfs scrub status -d /mnt/btr
# error summary: csum=1
#   corrected: 1, uncorrectable: 0, unverified: 0
```

Set up a periodic scrub, as you would in production:

```sh
systemctl list-unit-files 'btrfs-scrub*'
sudo systemctl enable --now btrfs-scrub@-.timer 2>/dev/null
```

Checksum algorithms:

```sh
for csum in crc32c xxhash blake2 sha256; do
  sudo umount /mnt/btr 2>/dev/null
  sudo mkfs.btrfs -f --csum $csum $DEV > /dev/null
  sudo mount $DEV /mnt/btr
  echo -n "$csum: "
  sudo dd if=/dev/zero of=/mnt/btr/bench bs=1M count=500 conv=fsync 2>&1 | tail -1
done
```

crc32c has hardware acceleration on x86 and ARM and is essentially free; sha256 is measurably slower but cryptographically strong (useful with `btrfs send` for verified replication).

---

### Lab 59.5 — Chunks, profiles, and multi-device

```sh
# Three devices of DIFFERENT sizes -- impossible with MD RAID
sudo modprobe -r scsi_debug
sudo modprobe scsi_debug dev_size_mb=1024 num_tgts=3
DEVS=$(lsblk -ndo NAME,MODEL | awk '/scsi_debug/{print "/dev/"$1}')
echo $DEVS

sudo mkfs.btrfs -f -d raid1 -m raid1 $DEVS
sudo mount $(echo $DEVS | cut -d' ' -f1) /mnt/btr
sudo btrfs filesystem show /mnt/btr
sudo btrfs filesystem usage /mnt/btr
```

Different profiles for data and metadata:

```sh
sudo umount /mnt/btr
sudo mkfs.btrfs -f -d raid0 -m raid1 $DEVS      # fast data, safe metadata
sudo mount $(echo $DEVS | cut -d' ' -f1) /mnt/btr
sudo btrfs filesystem df /mnt/btr
# Data, RAID0: ...
# Metadata, RAID1: ...
```

**This is impossible with block-level RAID**, where the whole device has one profile.

The chunk tree:

```sh
sudo dd if=/dev/zero of=/mnt/btr/x bs=1M count=500 2>/dev/null; sync
sudo umount /mnt/btr
sudo btrfs inspect-internal dump-tree -t chunk $(echo $DEVS | cut -d' ' -f1) | head -40
# CHUNK_ITEM: logical range -> (device, offset) x N stripes
```

Online reshaping — the whole point of §T.7:

```sh
sudo mount $(echo $DEVS | cut -d' ' -f1) /mnt/btr
sudo dd if=/dev/urandom of=/mnt/btr/data bs=1M count=300 2>/dev/null; sync

# Convert the profile, online, with data in place
sudo btrfs balance start -dconvert=raid1 -mconvert=raid1 /mnt/btr
sudo btrfs filesystem df /mnt/btr

# Remove a device, online
sudo btrfs device remove $(echo $DEVS | cut -d' ' -f3) /mnt/btr
sudo btrfs filesystem show /mnt/btr

# Add it back
sudo btrfs device add -f $(echo $DEVS | cut -d' ' -f3) /mnt/btr
sudo btrfs balance start -dusage=100 /mnt/btr

# SHRINK -- XFS cannot do this (Ch. 58 T.10)
sudo btrfs filesystem resize 1:-200M /mnt/btr
sudo btrfs filesystem usage /mnt/btr
```

Device failure and degraded mount:

```sh
sudo umount /mnt/btr
# Simulate a dead device
sudo dd if=/dev/zero of=$(echo $DEVS | cut -d' ' -f2) bs=1M count=100 2>/dev/null

sudo mount -o degraded $(echo $DEVS | cut -d' ' -f1) /mnt/btr
sudo btrfs filesystem show /mnt/btr
md5sum /mnt/btr/data           # still readable from the mirror

sudo btrfs device stats /mnt/btr
# Replace it:
sudo btrfs replace start -f 2 $(echo $DEVS | cut -d' ' -f3) /mnt/btr
sudo btrfs replace status /mnt/btr
```

---

### Lab 59.6 — `ENOSPC`, the hard way

```sh
sudo umount /mnt/btr 2>/dev/null
sudo modprobe -r scsi_debug; sudo modprobe scsi_debug dev_size_mb=1024
DEV=$(lsblk -ndo NAME,MODEL | awk '/scsi_debug/{print "/dev/"$1}')
sudo mkfs.btrfs -f $DEV && sudo mount $DEV /mnt/btr

# Fill with many small files to consume metadata chunks
sudo sh -c 'for i in $(seq 1 50000); do echo x > /mnt/btr/f$i; done' 2>&1 | tail -2
sync

df -h /mnt/btr                                 # misleading
sudo btrfs filesystem usage /mnt/btr           # the truth
sudo btrfs filesystem df /mnt/btr
```

```
Overall:
    Device size:                   1.00GiB
    Device allocated:              1.00GiB      <- ALL chunks allocated
    Device unallocated:            1.00MiB      <- nothing left for new chunks
    Used:                        756.00MiB
    Free (estimated):             58.00MiB      (min: 58.00MiB)
    Data ratio:                       1.00
    Metadata ratio:                   1.00
    Global reserve:               16.00MiB      (used: 0.00B)   <- T.9
```

The deadlock shape:

```sh
# Now try to delete -- which requires metadata allocation
sudo sh -c 'for i in $(seq 1 10000); do rm /mnt/btr/f$i; done' 2>&1 | tail -3
# On an older kernel this could fail with ENOSPC. Modern kernels use
# the global reserve to make progress.
sudo btrfs filesystem usage /mnt/btr
```

Fix it by compacting sparse chunks:

```sh
sudo btrfs balance start -dusage=10 /mnt/btr
sudo btrfs filesystem usage /mnt/btr
sudo btrfs balance start -dusage=50 -musage=50 /mnt/btr
sudo btrfs filesystem usage /mnt/btr
# "Device unallocated" recovers: half-empty chunks were consolidated.
```

Watch it happen:

```sh
sudo trace-cmd record -e btrfs:btrfs_reserve_extent\* -e btrfs:btrfs_space_reservation \
                      -e btrfs:btrfs_trigger_flush -e btrfs:btrfs_flush_space \
  -- sudo sh -c 'dd if=/dev/zero of=/mnt/btr/big bs=1M count=500 2>/dev/null'
sudo trace-cmd report | head -30
```

And the metadata-exhaustion scenario:

```sh
# Force metadata exhaustion specifically
sudo mkfs.btrfs -f -m single -d single $DEV && sudo mount $DEV /mnt/btr
sudo dd if=/dev/zero of=/mnt/btr/hog bs=1M count=900 2>/dev/null   # eat data chunks
sync
sudo btrfs filesystem usage /mnt/btr
sudo sh -c 'for i in $(seq 1 100000); do : > /mnt/btr/m$i; done' 2>&1 | tail -2
# ENOSPC: no unallocated space left to create a metadata chunk.
```

---

### Lab 59.7 — Compression

```sh
sudo umount /mnt/btr
sudo mkfs.btrfs -f $DEV
sudo mount -o compress=zstd:3 $DEV /mnt/btr

sudo cp -r /usr/share/doc /mnt/btr/ 2>/dev/null
sudo cp -r /usr/include /mnt/btr/ 2>/dev/null
sync

sudo apt install -y btrfs-compsize 2>/dev/null || \
  sudo apt install -y compsize
sudo compsize /mnt/btr
```

```
Type       Perc     Disk Usage   Uncompressed Referenced
TOTAL       38%      112M         293M         293M
none       100%       18M          18M          18M
zstd        32%       94M         275M         275M
```

Compare algorithms and levels:

```sh
for algo in no zlib:1 zlib:9 lzo zstd:1 zstd:3 zstd:15; do
  sudo umount /mnt/btr 2>/dev/null
  sudo mkfs.btrfs -f $DEV > /dev/null
  sudo mount -o compress-force=$algo $DEV /mnt/btr 2>/dev/null || \
    sudo mount $DEV /mnt/btr
  START=$(date +%s.%N)
  sudo cp -r /usr/share/doc /mnt/btr/ 2>/dev/null
  sync
  END=$(date +%s.%N)
  printf "%-10s %6.1fs  " $algo $(echo "$END - $START" | bc)
  sudo compsize /mnt/btr 2>/dev/null | awk '/TOTAL/{print $2, $3}'
done
```

`zstd:1` is usually the sweet spot: near-`lzo` speed with much better ratios.

Per-file and per-directory control:

```sh
sudo mkdir /mnt/btr/nocomp
sudo chattr +m /mnt/btr/nocomp       # no compression here
sudo mkdir /mnt/btr/comp
sudo btrfs property set /mnt/btr/comp compression zstd
sudo btrfs property get /mnt/btr/comp
```

Compression can make reads *faster*:

```sh
sudo mount -o compress-force=zstd:1 $DEV /mnt/btr
sudo cp -r /usr/share/doc /mnt/btr/d1 2>/dev/null; sync
sudo umount /mnt/btr
sudo mount -o compress=no $DEV /mnt/btr
sudo cp -r /usr/share/doc /mnt/btr/d2 2>/dev/null; sync

echo 3 | sudo tee /proc/sys/vm/drop_caches > /dev/null
echo -n "compressed read:   "; /usr/bin/time -f '%e s' sudo tar cf /dev/null /mnt/btr/d1 2>&1 | tail -1
echo 3 | sudo tee /proc/sys/vm/drop_caches > /dev/null
echo -n "uncompressed read: "; /usr/bin/time -f '%e s' sudo tar cf /dev/null /mnt/btr/d2 2>&1 | tail -1
```

On a slow device, less I/O beats more CPU.

---

### Lab 59.8 — Back-references, resolved

```sh
sudo mkfs.btrfs -f $DEV && sudo mount $DEV /mnt/btr
sudo btrfs subvolume create /mnt/btr/sv
sudo dd if=/dev/urandom of=/mnt/btr/sv/shared bs=1M count=50 2>/dev/null
sync

# Create sharing three ways
sudo btrfs subvolume snapshot /mnt/btr/sv /mnt/btr/snap
sudo cp --reflink=always /mnt/btr/sv/shared /mnt/btr/sv/reflink
sync

PHYS=$(sudo filefrag -v /mnt/btr/sv/shared | awk '/^ *0:/{print $4}' | tr -d '.')
LOGICAL=$((PHYS * 4096))

# Who references this extent? (T.5)
time sudo btrfs inspect-internal logical-resolve -v $LOGICAL /mnt/btr
```

```
inode 257 offset 0 root 256
inode 257 offset 0 root 257
inode 258 offset 0 root 256
```

Three referrers found by walking back-references. See them in the extent tree:

```sh
sudo umount /mnt/btr
sudo btrfs inspect-internal dump-tree -t extent $DEV | grep -A8 "EXTENT_ITEM" | head -30
# extent refs 3 gen 8 flags DATA
#   ref#0: extent data backref root 256 objectid 257 offset 0 count 1
#   ref#1: shared data backref parent 30605312 count 1        <- T.5's shared kind
```

**The shared backref names only the parent block, not a subvolume.** That is why resolution requires the upward walk.

Measure the cost as sharing grows:

```sh
sudo mount $DEV /mnt/btr
for n in 1 10 50 200; do
  sudo btrfs subvolume delete /mnt/btr/s* 2>/dev/null
  for i in $(seq 1 $n); do
    sudo btrfs subvolume snapshot /mnt/btr/sv /mnt/btr/s$i > /dev/null 2>&1
  done
  sync
  echo -n "$n snapshots, logical-resolve: "
  /usr/bin/time -f '%e s' sudo btrfs inspect-internal logical-resolve \
    $LOGICAL /mnt/btr 2>&1 | tail -1
done
```

The time grows with sharing. **That is the bill for O(1) snapshots.**

Inode → path, the other direction:

```sh
sudo btrfs inspect-internal inode-resolve 257 /mnt/btr
# Fast: INODE_REF items make this a direct lookup.
```

Where else this cost appears:

```sh
# FIEMAP's SHARED flag
sudo filefrag -v /mnt/btr/sv/shared | head -5     # note "shared" in flags

# Scrub error reporting: rmap-equivalent, needed to name the file
# Balance: every referrer must be updated
sudo trace-cmd record -e btrfs:btrfs_\* -- sudo btrfs balance start -dusage=100 /mnt/btr
sudo trace-cmd report | wc -l
```

---

## 3. Mastery drills

1. Trace, block by block, what btrfs writes when you change one byte in the middle of a 1 GiB file. Count blocks. Then do the same for ext4 and XFS.

2. The generation check in `btrfs_cow_block` makes CoW affordable. State the invariant it relies on and construct the bug that occurs if it is wrong.

3. Btrfs's key is `(objectid, type, offset)` and the *numeric values* of the type constants matter. Give three places where the sort order does real work, and one where a different assignment would break something.

4. A directory has both `DIR_ITEM` and `DIR_INDEX` items. Explain why both, and what ext4 gives up by having only the hash index (Ch. 57 §T.4).

5. Explain precisely why a snapshot cannot write a back-reference per extent, and why that forces the shared-backref design. Then compute the cost of `logical-resolve` as a function of snapshot count.

6. XFS's `rmapbt` gives O(log n) physical→owner. Btrfs's back-references do not, in general. State exactly what btrfs gains in exchange.

7. Btrfs checksums catch four distinct failure classes. Name each, and for each state whether ext4's `metadata_csum` and XFS's v5 metadata catch it.

8. `nodatacow` disables checksums. Explain why it must, and design a scheme that keeps checksums with in-place overwrite. What does your scheme require from the device?

9. Explain the btrfs `ENOSPC` deadlock: why freeing space can require allocating space. Name the three mechanisms btrfs uses to break it, and find the analogous mechanism in XFS and in the page allocator.

10. The RAID5/6 write hole: construct the exact sequence of events producing wrong data, and explain why CoW does not prevent it. Then state what would.

11. `btrfs send -p` is O(changes). Explain the mechanism precisely and state what would break it.

12. For each of ext4, XFS, and btrfs, state the single design decision that most determines its behaviour, and derive three consequences from each.

13. A user reports that their btrfs filesystem "got slow after a few months." Give the ordered diagnostic procedure and the five most likely causes, in order of probability.

---

## 4. Further reading

**Documentation**

- The **btrfs documentation** at `btrfs.readthedocs.io` ★★★ — the official documentation, actively maintained, far better than the old wiki. Read: "Btrfs design," "Trees," "On-disk format," "Resize," "Balance," "Scrub," "Compression," "Qgroups," and — importantly — **"Status"**, which honestly documents which features are production-ready and which are not.
- `Documentation/filesystems/btrfs.rst` — brief kernel-side overview.
- `man 8 btrfs` and its subcommand pages ★★★ — unusually good manual pages.
- `man 5 btrfs` — mount options.
- The btrfs wiki's "Gotchas" page ★★★ — **read this before deploying btrfs anywhere.** An honest catalogue of surprises.

**Papers and primary sources**

- Rodeh, Bacik, Mason, "BTRFS: The Linux B-tree Filesystem," ACM TOS 2013 ★★★ — **the design paper. Read it.**
- Rodeh, "B-trees, Shadowing, and Clones," ACM TOS 2008 ★★★ — the theoretical foundation: CoW B-trees and how cloning works. The paper btrfs is built on.
- Bonwick & Moore, "ZFS: The Last Word in File Systems" — the design btrfs responds to; the similarities and differences are instructive.
- Hitz, Lau, Malcolm, "File System Design for an NFS File Server Appliance," USENIX 1994 — NetApp's WAFL; the first production CoW filesystem and the ancestor of both ZFS and btrfs.
- Zhang et al., "ViewBox: Integrating Local File Systems with Cloud Storage Services," FAST 2014 — uses btrfs checksums; a good demonstration of what data integrity enables.
- Pillai et al., "All File Systems Are Not Created Equal," OSDI 2014 ★★★ — again; btrfs's guarantees versus ext4's and XFS's, empirically.

**LWN**

- "A short history of btrfs" (Valerie Aurora, 2009) ★★★ — **the single best introduction to btrfs's design.** Explains the B-tree, CoW, and back-references clearly.
- "Btrfs: Working with multiple devices" and the multi-device series
- "Btrfs send/receive" coverage
- "The Btrfs filesystem: An introduction" and the many status updates
- "Btrfs and ENOSPC" threads ★★★ — §T.9's problem, as experienced
- "Filesystem quotas and btrfs qgroups"
- "Btrfs RAID 5/6 status" coverage ★★★ — the honest assessment
- "Squota: simple quotas for btrfs" (2023)
- The recurring "should Btrfs be the default?" debates — worth reading for the arguments on both sides

**Source reading order**

1. The Rodeh 2008 paper (CoW B-trees), then the 2013 btrfs paper. Do not start with the code.
2. `fs/btrfs/ctree.h` ★★★ — the structures and, importantly, the key type constants.
3. `fs/btrfs/ctree.c`: `btrfs_search_slot`, `btrfs_cow_block`, `__btrfs_cow_block` ★★★ — the heart.
4. `fs/btrfs/transaction.c`: `btrfs_commit_transaction` — §T.4's commit sequence.
5. `fs/btrfs/extent-tree.c`: `btrfs_alloc_reserved_file_extent`, `__btrfs_free_extent`, the reservation code.
6. `fs/btrfs/backref.c` ★★★ — §T.5. Hard; read the paper first.
7. `fs/btrfs/volumes.c`: `btrfs_map_block` — §T.7's three address spaces.
8. `fs/btrfs/file-item.c` and `fs/btrfs/bio.c` — §T.6, including the repair path.
9. `fs/btrfs/tree-log.c` — after Ch. 61.

**Tools**

- `btrfs filesystem usage` ★★★ — **use this, not `df`.** Learn to read every line.
- `btrfs inspect-internal dump-tree` ★★★ — no other filesystem has anything this good. Use it constantly while learning.
- `btrfs inspect-internal dump-super -f`, `logical-resolve`, `inode-resolve`
- `btrfs device stats` ★★★ — check this periodically in production
- `btrfs scrub start -B` ★★★ — and schedule it
- `btrfs balance start -dusage=N` — the `ENOSPC` remedy
- `btrfs send`/`receive` ★★★ — for backups; combine with `btrbk` or `snapper`
- `compsize` — compression ratios
- `btrfs check --readonly` — offline check; **do not use `--repair` casually**, it can make things worse
- `btrfs restore` — extract data from a filesystem too damaged to mount
- `trace-cmd record -e btrfs:\*`
- `snapper`, `timeshift`, `btrbk` — the snapshot-management tooling that makes btrfs practical

---

→ Next: [60-f2fs-zfs.md](60-f2fs-zfs.md)

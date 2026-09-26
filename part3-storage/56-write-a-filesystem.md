# Chapter 56 — Writing a filesystem from scratch (ramfs → simplefs)

> **Goal:** Build three filesystems of increasing seriousness — a memory-backed one on `libfs`, a disk-backed one with a real on-disk format, and the beginnings of a crash-safe one — and in doing so discover *why* every field in the VFS interfaces exists. You will design an on-disk layout, write a `mkfs`, implement `fill_super`, `lookup`, `create`, `unlink`, `readdir`, and the full I/O path, write an `fsck`, and then deliberately corrupt your filesystem to learn why hostile-input validation is the single largest source of filesystem CVEs. By the end, reading any real filesystem's source is a matter of recognising choices you have already had to make.

---

## Theory & First Principles

### T.0 — Start here: design the on-disk format before writing any code

You have a block device: an array of 4 KiB blocks, numbered 0 to N. Nothing else. Build a
filesystem.

**Do not start with code. Start with the layout**, because the layout is the part you cannot
change later:

```
  block 0       1         2..K           K+1..M              M+1..N
  +---------+---------+-------------+------------------+----------------+
  | SUPER   | block   | inode       | inode table      | data blocks    |
  | BLOCK   | bitmap  | bitmap      | (fixed-size      |                |
  |         |         |             |  inode records)  |                |
  +---------+---------+-------------+------------------+----------------+
   magic,     which     which         owner, mode,       file contents
   sizes,     blocks    inodes        size, timestamps,  and directory
   counts     are free  are free      block pointers     entries
```

**Four questions the layout must answer**, and every real filesystem is a different set of
answers:

| Question | Cheap answer | Consequence |
|---|---|---|
| **How do I find a file's blocks?** | direct + indirect pointers | Ch. 57; simple, but O(depth) and poor for large files |
| | **extents** (start, length) | ext4/XFS/btrfs; far better for big contiguous files |
| **How do I know what is free?** | a bitmap | one bit per block; 32 MiB for a 1 TiB device |
| | a **B-tree of free extents** | XFS; finds a contiguous run of exactly the right size |
| **How do directories work?** | a linear list of (name, inode) | O(n) lookup — fatal past ~10,000 entries |
| | a **hash tree** or **B-tree** | ext4 htree, XFS/btrfs B-tree |
| **What survives a crash?** | *(nothing — run fsck)* | O(filesystem size); hours on a large volume |
| | a **journal** or **copy-on-write** | Ch. 61 |

**Now the decision that determines everything else**, and it is worth making explicitly
before you write a line:

```
   OVERWRITE IN PLACE                    COPY ON WRITE
   (ext4, XFS)                           (btrfs, ZFS)
   +--------------------------+          +---------------------------+
   | Update data where it is. |          | NEVER overwrite. Write a  |
   | Fast, predictable, low   |          | new copy, then atomically |
   | write amplification.     |          | swap the ROOT pointer.    |
   |                          |          |                           |
   | A crash mid-update       |          | Snapshots are free.       |
   | leaves TORN state, so    |          | Checksums are natural.    |
   | you need a JOURNAL.      |          | Fragmentation is severe   |
   |                          |          | for random overwrite.     |
   +--------------------------+          +---------------------------+
```

Everything downstream follows: whether you need a journal, whether snapshots are cheap or
impossible, what your write amplification is, and whether a database will perform well on you
(Ch. 59 §T.6).

**And one non-obvious constraint that catches every first-time filesystem author:**

> **The only atomic unit the device gives you is a single sector write, and even that is not
> guaranteed on all hardware.** Every multi-block operation — "allocate a block, link it into
> the inode, update the bitmap, update the free count" — is four separate writes that can be
> interrupted at any point.

There are exactly three ways out, and every filesystem in existence picks one (Ch. 61):

1. **Journal** the intent, then do the work. Replay on mount.
2. **Order the writes** so that every reachable intermediate state is consistent (soft
   updates — elegant, and complex enough that almost nobody does it).
3. **Copy-on-write**: build the new state entirely, then flip one pointer.

**The practical advice for this chapter:** write the mount/unmount path and `mkfs` first, get
an empty filesystem that mounts, then add lookup, then read, then write, then — last —
crash consistency. And test with `fsstress` plus a power-cut simulation from day one, because
retrofitting consistency onto a design that did not anticipate it does not work.

```bash
mkfs.ext4 -b 4096 -I 256 /tmp/img && dumpe2fs /tmp/img | head -40
xfs_db -r -c 'sb 0' -c print /tmp/xfs.img
hexdump -C /tmp/img | head -20        # look at the actual superblock bytes
```

---

### T.1 What a filesystem actually is

Strip away the implementation and a filesystem is **three maps plus a durability story**:

| Map | From | To | Realised as |
|---|---|---|---|
| **Namespace** | `(directory, name)` | inode number | directory entries |
| **Inode** | inode number | metadata + block map | the inode table |
| **Block map** | `(inode, file offset)` | device block | direct/indirect blocks, extents, B-tree |
| **Allocator** | — | free space | bitmaps, B-trees, log |

Every filesystem in Part 3 is a different set of answers to those four questions plus a different answer to "what happens if the power fails mid-update?"

ext2 answers: linked list of entries; fixed inode table; direct+indirect blocks; bitmaps; **nothing** (hence `fsck`).
ext4: htree; fixed inode table; extents; bitmaps; journal.
XFS: B+tree; dynamic inodes in an allocation-group B+tree; extent B+tree; free-space B+trees; journal.
btrfs: one B-tree for everything; CoW; **the tree itself is the durability story**.

Building a filesystem is choosing a point in that space. Doing it once makes the rest legible.

### T.2 Why start with `libfs`

The VFS interface is large. Implementing it from zero means writing a lot of code before anything works, and most of that code is identical across filesystems. `fs/libfs.c` provides the identical parts:

| Helper | What it does |
|---|---|
| `simple_lookup` | for filesystems fully in the dcache: return a negative dentry on miss |
| `simple_dir_operations` | `readdir` from the dcache's child list |
| `simple_statfs` | plausible `statfs` values |
| `simple_link`/`simple_unlink`/`simple_rmdir`/`simple_rename` | dcache-only namespace operations |
| `simple_get_link` | symlinks whose target is in `inode->i_link` |
| `simple_write_begin`/`simple_write_end` | page-cache-only writes |
| `generic_file_read_iter`/`generic_file_write_iter` | the full buffered I/O path |
| `dcache_dir_open`/`dcache_readdir` | directory iteration entirely from the dcache |
| `simple_fill_super` | build a root directory with a fixed set of files |
| `alloc_anon_inode` | an inode with no filesystem behind it |

The insight `libfs` encodes: **for a purely in-memory filesystem, the dcache and page cache *are* the filesystem.** There is no separate representation. `ramfs` is essentially nothing but glue, which is why it is 300 lines.

That gives a ladder:

1. **ramfs-alike** — no persistence; `libfs` does almost everything. You learn the VFS shape.
2. **simplefs** — a real on-disk format; you write superblock/inode/bitmap/directory code. You learn why real filesystems are complicated.
3. **simplefs + journal** — you learn Ch. 61's problem by hitting it.

### T.3 Designing an on-disk format is designing a protocol with your future self

On-disk format decisions are permanent in a way in-memory ones are not: existing filesystems must keep working forever. This is Ch. 24's ABI problem with a much longer horizon — ext2 images from 1993 still mount.

The non-negotiable rules:

**(a) A magic number, at a fixed offset.** So `blkid`, `mount`, and `fsck` can identify the format without a hint. Not at offset 0 — that is the boot sector on partitioned media; ext2 puts the superblock at byte 1024 for exactly this reason.

**(b) Explicit endianness.** Use `__le32`/`__le64`, never `u32`. A filesystem image is data interchanged between machines. Use `sparse` (`make C=1`) to check.

**(c) Explicit sizes, explicit padding.** `__u32` not `unsigned long`. Pad to alignment explicitly; never rely on the compiler's layout. `_Static_assert(sizeof(struct x) == N)` every on-disk structure.

**(d) Version and feature flags — three sets.** ext2 got this exactly right and everyone copied it:

```c
	__le32 s_feature_compat;    /* mount RW even if unknown */
	__le32 s_feature_incompat;  /* refuse to mount if unknown */
	__le32 s_feature_ro_compat; /* mount read-only if unknown */
```

Three categories because there are three kinds of change: cosmetic additions, format changes that break readers, and changes that break *writers* but not readers. Without this taxonomy you can never add a feature without breaking old kernels. **Design this in from day one** — you cannot add it later, because old kernels will not know to check it.

**(e) Reserved space.** Leave padding in the superblock and inode. You will want it.

**(f) Checksums, or a plan for them.** Retrofitting checksums requires a feature flag and a format change. Ext4 did it (`metadata_csum`); it was a multi-year effort.

**(g) 64-bit everything, including time.** 32-bit timestamps expire in 2038. 32-bit block numbers cap you at 16 TiB with 4 KiB blocks. Both limits were reached by filesystems designed to be "big enough."

### T.4 The inode number is a promise

`ino_t` is exposed to userspace via `stat`, and applications rely on three properties:

1. **Unique within the filesystem at any instant.** `(st_dev, st_ino)` identifies a file; `find`'s hardlink detection, `tar`, `rsync`, and `cp -a` all depend on it.
2. **Stable for the file's lifetime.** It must not change under rename.
3. **Reusable only after the file is gone.** And reuse is a real hazard for NFS (§T.8).

Two design approaches:

- **Positional** (ext2/ext4): the inode number *is* the index into a fixed table. Lookup is O(1) arithmetic. Cost: the inode count is fixed at `mkfs` time, and you can run out of inodes with free space remaining.
- **Allocated** (XFS, btrfs): inode numbers come from a counter or encode a location; the table is dynamic. Cost: a lookup structure is needed.

`simplefs` uses positional because the arithmetic is instructive:

```c
	block = sb->inode_table_start + ino / inodes_per_block;
	offset = (ino % inodes_per_block) * sizeof(struct simplefs_inode);
```

Inode 0 is conventionally invalid (so `ino == 0` can mean "none"); inode 1 is often the bad-blocks inode; the root is inode 2 by ext2 convention. Following convention costs nothing and helps tooling.

### T.5 The `fill_super` contract and the hostile-input problem

`fill_super` is the most security-critical function in a filesystem.

It reads data from a block device that **may be entirely attacker-controlled**. On a desktop with automounting, inserting a USB stick runs your `fill_super` on hostile input, in the kernel, with full privileges. The overwhelming majority of filesystem CVEs are of this form: a crafted image causing an out-of-bounds read, an integer overflow in a size computation, or an infinite loop.

Therefore: **every single field read from disk must be validated before use.** Not "probably fine" — validated.

```c
static int simplefs_check_sb(struct super_block *sb,
			     struct simplefs_sb_info *disk)
{
	uint32_t nr_blocks = le32_to_cpu(disk->nr_blocks);
	uint32_t nr_inodes = le32_to_cpu(disk->nr_inodes);
	uint32_t ibmap = le32_to_cpu(disk->nr_ibitmap_blocks);
	uint32_t bbmap = le32_to_cpu(disk->nr_bbitmap_blocks);
	uint64_t dev_blocks;

	if (le32_to_cpu(disk->magic) != SIMPLEFS_MAGIC)
		return -EINVAL;

	/* 1. Nothing may be zero that must not be. */
	if (!nr_blocks || !nr_inodes)
		return -EINVAL;

	/* 2. Nothing may exceed the device. */
	dev_blocks = i_size_read(sb->s_bdev->bd_inode) >> SIMPLEFS_BLOCK_SHIFT;
	if (nr_blocks > dev_blocks)
		return -EINVAL;

	/* 3. Derived quantities must not overflow. Check BEFORE computing. */
	if (nr_inodes > SIMPLEFS_MAX_INODES)
		return -EINVAL;
	if (ibmap > SIMPLEFS_MAX_BITMAP_BLOCKS ||
	    bbmap > SIMPLEFS_MAX_BITMAP_BLOCKS)
		return -EINVAL;

	/* 4. The layout must be self-consistent and fit. */
	if (check_add_overflow(ibmap, bbmap, &tmp) ||
	    1 + tmp + DIV_ROUND_UP(nr_inodes, SIMPLEFS_INODES_PER_BLOCK)
	        > nr_blocks)
		return -EINVAL;

	/* 5. The bitmaps must be large enough for what they describe. */
	if (ibmap * SIMPLEFS_BLOCK_SIZE * 8 < nr_inodes)
		return -EINVAL;
	if (bbmap * SIMPLEFS_BLOCK_SIZE * 8 < nr_blocks)
		return -EINVAL;

	return 0;
}
```

Five classes of check, and they generalise to every filesystem:

| Class | Question |
|---|---|
| Magic | is this even my format? |
| Non-zero | would a zero here cause a div-by-zero or an empty loop that never terminates? |
| Bounds | does this exceed the device / a sane maximum? |
| Overflow | does computing a derived value wrap? Use `check_*_overflow()`. |
| Consistency | do the pieces fit together and not overlap? |

The same discipline applies to inodes (`i_blocks` vs `i_size`, block pointers within range, mode bits sane), directory entries (name length, record length, offset within the block, no infinite loop), and everything else read from disk.

**The rule to internalise: a filesystem driver is a parser for a hostile binary format.** Treat it with the paranoia you would apply to an image decoder.

### T.6 Directory entries: the variable-length record problem

A directory is a sequence of `(name, inode)` pairs. Names are variable length. Three encodings:

**(a) Fixed-size slots** (`simplefs` at first): `struct { __le32 ino; char name[MAX]; }`. Trivial to parse, trivial to make safe, wasteful, and imposes a hard name-length limit.

**(b) Variable-length with an explicit record length** (ext2/ext4):

```c
struct ext2_dir_entry_2 {
	__le32	inode;
	__le16	rec_len;      /* distance to the next entry */
	__u8	name_len;
	__u8	file_type;
	char	name[];
};
```

`rec_len` ≥ the actual entry size, so deleting an entry just extends the *previous* entry's `rec_len` over it — deletion without compaction. Elegant. Also the source of a long list of CVEs, because parsing requires:

```c
static bool ext2_check_dir_entry(struct ext2_dir_entry_2 *de,
				 unsigned offset, unsigned blocksize)
{
	unsigned rlen = le16_to_cpu(de->rec_len);

	if (rlen < EXT2_DIR_REC_LEN(1))       return false;  /* too short */
	if (rlen % 4 != 0)                    return false;  /* misaligned */
	if (rlen < EXT2_DIR_REC_LEN(de->name_len)) return false; /* name doesn't fit */
	if (offset + rlen > blocksize)        return false;  /* runs past the block */
	if (le32_to_cpu(de->inode) > max_ino) return false;
	return true;
}
```

Miss the `rlen < minimum` check and `rec_len == 0` gives an infinite loop. Miss the `offset + rlen > blocksize` check and you read past the buffer. **Both have been real CVEs, more than once.**

**(c) An index** (ext4 htree, XFS B+tree, btrfs B-tree). Required for large directories: linear scan is O(n) per lookup, so creating n files is O(n²). This is very easy to demonstrate (Lab 56.6) and is why `dir_index` is on by default.

Two additional constraints:

- **`readdir` offsets must be stable.** `getdents` returns a cookie (`d_off`) that must still make sense after entries are added or removed, because NFS and `rewinddir` depend on it. Linear directories use the byte offset; hashed directories use the hash, which is why ext4 htree needs careful handling of hash collisions.
- **`.` and `..` must exist**, and `..` must be updated on rename. `i_nlink` of a directory counts: itself (`.`), its entry in the parent, and one per subdirectory's `..`. Getting `nlink` accounting wrong is a classic bug — `fsck` catches it.

### T.7 Block allocation: the decision that determines performance

The allocator decides where data lands, and therefore whether reads are sequential.

**Bitmaps** are the simplest: one bit per block. 4 KiB block, 4 KiB bitmap block → 32768 blocks (128 MiB) per bitmap block. Fast to scan with `find_next_zero_bit`, trivial to make crash-safe-ish, and they scale poorly to very large filesystems (a 100 TiB filesystem needs 3 GiB of bitmap).

Four policies layered on top, all of which real filesystems implement:

| Policy | Why |
|---|---|
| **Locality**: allocate near the inode | keeps a file's data near its metadata |
| **Goal block**: allocate near the previous block of this file | contiguity |
| **Preallocation**: reserve a run per file | prevents interleaving of concurrent writers |
| **Delayed allocation** (Ch. 55 §T.8) | decide with full information |

Without locality and goal blocks, concurrent writers interleave and every file ends up fragmented. Lab 56.7 demonstrates this in a few lines.

Ext2's **block groups** are the canonical locality mechanism: divide the device into groups, each with its own bitmaps and inode table, and try to keep a directory's inodes and their data in one group. It costs metadata duplication and buys locality. XFS's **allocation groups** are the same idea applied also to parallelism — separate groups can be allocated from concurrently with separate locks.

### T.8 `export_operations`: why NFS forces you to think about identity

If your filesystem can be NFS-exported, it must be able to answer: **"here is an opaque handle I gave you an hour ago; give me back the file."**

```c
struct export_operations {
	int (*encode_fh)(struct inode *, __u32 *fh, int *max_len,
			 struct inode *parent);
	struct dentry *(*fh_to_dentry)(struct super_block *, struct fid *,
				       int fh_len, int fh_type);
	struct dentry *(*fh_to_parent)(struct super_block *, struct fid *,
				       int fh_len, int fh_type);
	int (*get_name)(struct dentry *parent, char *name, struct dentry *child);
	struct dentry *(*get_parent)(struct dentry *child);
	...
};
```

This is harder than it looks, and the difficulties are instructive:

**(a) The handle must survive a reboot and a cache flush.** It cannot be a pointer. It is typically `(inode number, generation)`.

**(b) The generation number exists because inode numbers are reused.** If inode 42 is deleted and reallocated, an old handle must *fail*, not silently open the wrong file. So every inode has a generation counter, incremented on reuse, stored on disk, and included in the handle. **If you do not have a generation field in your on-disk inode, you cannot safely support NFS** — and you cannot add one later without a format change. Put it in from the start.

**(c) You must be able to go from an inode to a dentry without a path.** The dcache is built top-down; NFS needs bottom-up. `d_obtain_alias()` creates a **disconnected dentry**, and the VFS reconnects it lazily via `get_parent()`. This is why `..` must be findable from the inode — for directories, you need the parent, which means either storing it or having `..` in the directory data.

**(d) `get_name()` requires reverse lookup**: given a parent and a child inode, find the name. That is a linear scan of the directory.

The general lesson: **supporting NFS export forces a filesystem to have a persistent, verifiable, path-independent notion of file identity.** Filesystems that did not design for it (early FUSE, some network filesystems) had to retrofit it painfully. It is a good test of whether your on-disk format is well designed.

### T.9 Mount options, and the modern API

The old `->mount()` took an opaque `char *data` string that each filesystem parsed by hand — with `strsep` and `simple_strtoul`, in the kernel, with no common error reporting. Every filesystem had its own bugs.

The new **filesystem context API** (Ch. 53 §T.6) replaces it:

```c
enum simplefs_param { Opt_debug, Opt_maxinodes, Opt_errors };

static const struct fs_parameter_spec simplefs_fs_parameters[] = {
	fsparam_flag  ("debug",     Opt_debug),
	fsparam_u32   ("maxinodes", Opt_maxinodes),
	fsparam_enum  ("errors",    Opt_errors, simplefs_param_errors),
	{}
};

static int simplefs_parse_param(struct fs_context *fc,
				struct fs_parameter *param)
{
	struct simplefs_fs_context *ctx = fc->fs_private;
	struct fs_parse_result result;
	int opt = fs_parse(fc, simplefs_fs_parameters, param, &result);

	if (opt < 0)
		return opt;

	switch (opt) {
	case Opt_debug:
		ctx->debug = true;
		break;
	case Opt_maxinodes:
		if (result.uint_32 > SIMPLEFS_MAX_INODES)
			return invalfc(fc, "maxinodes too large (max %u)",
				       SIMPLEFS_MAX_INODES);
		ctx->max_inodes = result.uint_32;
		break;
	}
	return 0;
}
```

Three real gains: declarative specification (typed, validated centrally), **structured error messages** that reach userspace via `fsconfig`'s log instead of `dmesg`, and options parsed *before* any I/O happens, so a bad option fails cleanly.

`errorfc()`/`invalfc()`/`warnfc()` put messages in the context log. Compare the old world where a mount failure produced `-EINVAL` and, if you were lucky, something in `dmesg` that might be from a different mount attempt.

### T.10 `fsck` is part of the design, not an afterthought

If your filesystem has no journal (and even if it does), you need a checker. Writing it is a design exercise because it forces you to state your invariants explicitly:

| Invariant | Check |
|---|---|
| Every allocated block is referenced exactly once | build a reference bitmap; compare to the on-disk bitmap |
| Every referenced block is allocated | same pass |
| `i_nlink` equals the number of directory entries pointing at the inode | count while walking the tree |
| Every inode with `i_nlink > 0` is reachable from the root | tree walk; orphans go to `lost+found` |
| Every directory has `.` and `..`, and `..` points at the real parent | per-directory check |
| The directory tree is a tree (no cycles) | mark-and-check during the walk |
| `i_size` is consistent with the block map | per-inode |

Writing these down *is* writing the specification of your format. Every check that fails to fire on real corruption is a specification you did not state. Ext2's `e2fsck` runs five passes corresponding roughly to those rows; that structure is not arbitrary.

And the deeper lesson: `fsck` time is O(filesystem size) — it is why journalling exists (Ch. 61), and why at multi-terabyte sizes, filesystems that require `fsck` after a crash became unusable.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `fs/libfs.c` ★★★ | every `simple_*` helper; read this before writing anything |
| `fs/ramfs/` ★★★ | ~300 lines; the minimal filesystem |
| `fs/minix/` ★★★ | ~3000 lines; the minimal *disk* filesystem — the best model for simplefs |
| `fs/ext2/` ★★★ | ~10000 lines; production quality, no journal; the reference for §T.3–T.7 |
| `fs/romfs/`, `fs/cramfs/` | read-only; even simpler |
| `fs/fs_context.c`, `fs/fs_parser.c` | §T.9 |
| `fs/exportfs/expfs.c` | §T.8's reconnection logic |
| `fs/inode.c` | `new_inode`, `iget_locked`, `iput` |
| `fs/super.c` | `get_tree_bdev`, `sget_fc`, `deactivate_super` |
| `Documentation/filesystems/vfs.rst` ★★★ | the contract for every method |
| `Documentation/filesystems/locking.rst` ★★★ | what is held when each is called |
| `Documentation/filesystems/mount_api.rst` ★★★ | §T.9 |
| `Documentation/filesystems/porting.rst` | the change log; read the last few years before starting |

### 1.2 The on-disk format

```c
// SPDX-License-Identifier: GPL-2.0
/* simplefs.h -- shared between the kernel module and mkfs. */
#ifndef _SIMPLEFS_H
#define _SIMPLEFS_H

#define SIMPLEFS_MAGIC		0x53465321	/* "SFS!" */
#define SIMPLEFS_BLOCK_SHIFT	12
#define SIMPLEFS_BLOCK_SIZE	(1 << SIMPLEFS_BLOCK_SHIFT)
#define SIMPLEFS_MAX_FILENAME	60
#define SIMPLEFS_N_DIRECT	12
#define SIMPLEFS_ROOT_INO	2
#define SIMPLEFS_MAX_INODES	(1U << 24)
#define SIMPLEFS_MAX_BITMAP_BLOCKS 4096

/* Feature flags -- three sets, per T.3(d). Design these in from day one. */
#define SIMPLEFS_FEATURE_COMPAT_SUPP		0
#define SIMPLEFS_FEATURE_INCOMPAT_SUPP		0
#define SIMPLEFS_FEATURE_RO_COMPAT_SUPP		0

struct simplefs_super_block {
	__le32	magic;
	__le32	nr_blocks;
	__le32	nr_inodes;
	__le32	nr_free_blocks;
	__le32	nr_free_inodes;
	__le32	nr_ibitmap_blocks;
	__le32	nr_bbitmap_blocks;
	__le32	inode_table_start;	/* block number */
	__le32	data_start;		/* block number */
	__le32	feature_compat;
	__le32	feature_incompat;
	__le32	feature_ro_compat;
	__le32	state;			/* CLEAN / ERROR */
	__le32	mtime;
	__le32	wtime;
	__le32	csum;
	__u8	uuid[16];
	__u8	reserved[SIMPLEFS_BLOCK_SIZE - 80];   /* T.3(e) */
};
_Static_assert(sizeof(struct simplefs_super_block) == SIMPLEFS_BLOCK_SIZE, "");

struct simplefs_inode {
	__le16	mode;
	__le16	nlink;
	__le32	uid;
	__le32	gid;
	__le64	size;
	__le64	atime;			/* 64-bit: T.3(g) */
	__le64	mtime;
	__le64	ctime;
	__le32	blocks;			/* in 512-byte units, like stat */
	__le32	generation;		/* T.8(b) -- REQUIRED for NFS */
	__le32	flags;
	__le32	block[SIMPLEFS_N_DIRECT + 1];	/* [N_DIRECT] = single indirect */
	__u8	reserved[16];
};
_Static_assert(sizeof(struct simplefs_inode) == 128, "");

#define SIMPLEFS_INODES_PER_BLOCK \
	(SIMPLEFS_BLOCK_SIZE / sizeof(struct simplefs_inode))

struct simplefs_dir_entry {
	__le32	ino;			/* 0 = free slot */
	char	name[SIMPLEFS_MAX_FILENAME];
};
_Static_assert(sizeof(struct simplefs_dir_entry) == 64, "");

#define SIMPLEFS_ENTRIES_PER_BLOCK \
	(SIMPLEFS_BLOCK_SIZE / sizeof(struct simplefs_dir_entry))

/*
 * Layout:
 *   block 0                : superblock
 *   1 .. 1+ib              : inode bitmap
 *   .. +bb                 : block bitmap
 *   .. inode_table_start   : inode table
 *   .. data_start          : data blocks
 */
#endif
```

Fixed-size directory entries (§T.6(a)) are chosen deliberately: it makes the first version safe and simple, and Drill 5 asks you to convert to variable-length and discover the parsing hazards yourself.

### 1.3 `fill_super`

```c
// SPDX-License-Identifier: GPL-2.0
static int simplefs_fill_super(struct super_block *sb, struct fs_context *fc)
{
	struct simplefs_sb_info *sbi;
	struct simplefs_super_block *disk;
	struct buffer_head *bh;
	struct inode *root;
	int ret;

	if (!sb_set_blocksize(sb, SIMPLEFS_BLOCK_SIZE))
		return invalfc(fc, "device does not support %d-byte blocks",
			       SIMPLEFS_BLOCK_SIZE);

	bh = sb_bread(sb, 0);
	if (!bh)
		return -EIO;
	disk = (struct simplefs_super_block *)bh->b_data;

	ret = simplefs_check_sb(sb, disk, fc);     /* T.5: validate EVERYTHING */
	if (ret)
		goto out_bh;

	sbi = kzalloc(sizeof(*sbi), GFP_KERNEL);
	if (!sbi) { ret = -ENOMEM; goto out_bh; }

	sbi->nr_blocks	      = le32_to_cpu(disk->nr_blocks);
	sbi->nr_inodes	      = le32_to_cpu(disk->nr_inodes);
	sbi->nr_ibitmap_blocks = le32_to_cpu(disk->nr_ibitmap_blocks);
	sbi->nr_bbitmap_blocks = le32_to_cpu(disk->nr_bbitmap_blocks);
	sbi->inode_table_start = le32_to_cpu(disk->inode_table_start);
	sbi->data_start	      = le32_to_cpu(disk->data_start);
	mutex_init(&sbi->alloc_lock);

	/* Unknown incompat features: refuse. Unknown ro_compat: read-only. */
	if (le32_to_cpu(disk->feature_incompat) & ~SIMPLEFS_FEATURE_INCOMPAT_SUPP) {
		ret = invalfc(fc, "unsupported incompat features 0x%x",
			      le32_to_cpu(disk->feature_incompat));
		goto out_sbi;
	}
	if ((le32_to_cpu(disk->feature_ro_compat) & ~SIMPLEFS_FEATURE_RO_COMPAT_SUPP)
	    && !sb_rdonly(sb)) {
		warnfc(fc, "unsupported ro_compat features; mounting read-only");
		sb->s_flags |= SB_RDONLY;
	}

	ret = simplefs_read_bitmaps(sb, sbi);
	if (ret)
		goto out_sbi;

	sb->s_magic	= SIMPLEFS_MAGIC;
	sb->s_fs_info	= sbi;
	sb->s_op	= &simplefs_super_ops;
	sb->s_export_op	= &simplefs_export_ops;
	sb->s_maxbytes	= SIMPLEFS_MAX_FILESIZE;
	sb->s_time_gran	= 1;
	sb->s_time_min	= S64_MIN;
	sb->s_time_max	= S64_MAX;
	super_set_uuid(sb, disk->uuid, sizeof(disk->uuid));

	root = simplefs_iget(sb, SIMPLEFS_ROOT_INO);
	if (IS_ERR(root)) { ret = PTR_ERR(root); goto out_bitmaps; }
	if (!S_ISDIR(root->i_mode)) {
		iput(root);
		ret = invalfc(fc, "root inode is not a directory");
		goto out_bitmaps;
	}

	sb->s_root = d_make_root(root);        /* consumes `root` on failure */
	if (!sb->s_root) { ret = -ENOMEM; goto out_bitmaps; }

	brelse(bh);
	return 0;

out_bitmaps:
	simplefs_free_bitmaps(sbi);
out_sbi:
	kfree(sbi);
out_bh:
	brelse(bh);
	return ret;
}
```

Note the goto ladder (Ch. 05) — `fill_super` is exactly the kind of function it exists for.

### 1.4 `iget`: the inode-instantiation protocol

```c
struct inode *simplefs_iget(struct super_block *sb, unsigned long ino)
{
	struct simplefs_sb_info *sbi = SIMPLEFS_SB(sb);
	struct simplefs_inode_info *si;
	struct simplefs_inode *di;
	struct buffer_head *bh;
	struct inode *inode;
	uint32_t block, offset;
	int i;

	if (ino >= sbi->nr_inodes || ino < SIMPLEFS_ROOT_INO)
		return ERR_PTR(-EINVAL);          /* T.5 again */

	inode = iget_locked(sb, ino);
	if (!inode)
		return ERR_PTR(-ENOMEM);
	if (!(inode->i_state & I_NEW))
		return inode;                     /* already cached */

	si = SIMPLEFS_I(inode);
	block = sbi->inode_table_start + ino / SIMPLEFS_INODES_PER_BLOCK;
	offset = ino % SIMPLEFS_INODES_PER_BLOCK;

	bh = sb_bread(sb, block);
	if (!bh) { iget_failed(inode); return ERR_PTR(-EIO); }
	di = (struct simplefs_inode *)bh->b_data + offset;

	inode->i_mode = le16_to_cpu(di->mode);
	i_uid_write(inode, le32_to_cpu(di->uid));
	i_gid_write(inode, le32_to_cpu(di->gid));
	set_nlink(inode, le16_to_cpu(di->nlink));
	inode->i_size = le64_to_cpu(di->size);
	inode->i_blocks = le32_to_cpu(di->blocks);
	inode->i_generation = le32_to_cpu(di->generation);
	inode_set_atime(inode, le64_to_cpu(di->atime), 0);
	inode_set_mtime(inode, le64_to_cpu(di->mtime), 0);
	inode_set_ctime(inode, le64_to_cpu(di->ctime), 0);

	for (i = 0; i <= SIMPLEFS_N_DIRECT; i++) {
		uint32_t b = le32_to_cpu(di->block[i]);

		if (b && (b < sbi->data_start || b >= sbi->nr_blocks)) {
			brelse(bh);
			simplefs_error(sb, "inode %lu: bad block %u", ino, b);
			iget_failed(inode);
			return ERR_PTR(-EFSCORRUPTED);
		}
		si->block[i] = b;
	}
	brelse(bh);

	/* Sanity: size must be consistent with the block map. */
	if (inode->i_size > SIMPLEFS_MAX_FILESIZE) {
		simplefs_error(sb, "inode %lu: size %lld too large",
			       ino, inode->i_size);
		iget_failed(inode);
		return ERR_PTR(-EFSCORRUPTED);
	}

	switch (inode->i_mode & S_IFMT) {
	case S_IFREG:
		inode->i_op  = &simplefs_file_inode_ops;
		inode->i_fop = &simplefs_file_ops;
		inode->i_mapping->a_ops = &simplefs_aops;
		break;
	case S_IFDIR:
		inode->i_op  = &simplefs_dir_inode_ops;
		inode->i_fop = &simplefs_dir_ops;
		break;
	case S_IFLNK:
		inode->i_op = &simplefs_symlink_inode_ops;
		inode_nohighmem(inode);
		inode->i_mapping->a_ops = &simplefs_aops;
		break;
	default:
		init_special_inode(inode, inode->i_mode,
				   le32_to_cpu(di->block[0]));   /* rdev */
		break;
	}

	unlock_new_inode(inode);
	return inode;
}
```

`iget_locked` + `I_NEW` + `unlock_new_inode` is a **mandatory protocol**, and it is Ch. 25's P5 (Lazy Init): two concurrent lookups of the same inode number must get the same `struct inode`, and the second must wait for the first to finish populating it. `iget_failed()` on the error path is essential — forgetting it leaves a permanently-`I_NEW` inode that wedges every future lookup.

### 1.5 `lookup` and `create`

```c
static struct dentry *simplefs_lookup(struct inode *dir, struct dentry *dentry,
				      unsigned int flags)
{
	struct super_block *sb = dir->i_sb;
	struct inode *inode = NULL;
	uint32_t ino;

	if (dentry->d_name.len > SIMPLEFS_MAX_FILENAME)
		return ERR_PTR(-ENAMETOOLONG);

	ino = simplefs_find_entry(dir, &dentry->d_name);
	if (ino) {
		inode = simplefs_iget(sb, ino);
		if (IS_ERR(inode))
			return ERR_CAST(inode);
	}
	/* d_splice_alias handles inode == NULL by making a negative dentry. */
	return d_splice_alias(inode, dentry);
}

static int simplefs_create(struct mnt_idmap *idmap, struct inode *dir,
			   struct dentry *dentry, umode_t mode, bool excl)
{
	return simplefs_mknod(idmap, dir, dentry, mode | S_IFREG, 0);
}

static int simplefs_mknod(struct mnt_idmap *idmap, struct inode *dir,
			  struct dentry *dentry, umode_t mode, dev_t rdev)
{
	struct super_block *sb = dir->i_sb;
	struct inode *inode;
	uint32_t ino;
	int ret;

	if (dentry->d_name.len > SIMPLEFS_MAX_FILENAME)
		return -ENAMETOOLONG;

	ino = simplefs_alloc_inode_nr(sb);
	if (!ino)
		return -ENOSPC;

	inode = simplefs_new_inode(sb, dir, ino, mode, rdev);
	if (IS_ERR(inode)) { ret = PTR_ERR(inode); goto free_ino; }

	ret = simplefs_add_entry(dir, &dentry->d_name, ino);
	if (ret)
		goto drop_inode;

	inode_set_mtime_to_ts(dir, inode_set_ctime_current(dir));
	mark_inode_dirty(dir);
	d_instantiate_new(dentry, inode);
	return 0;

drop_inode:
	clear_nlink(inode);
	mark_inode_dirty(inode);
	unlock_new_inode(inode);
	iput(inode);
free_ino:
	simplefs_free_inode_nr(sb, ino);
	return ret;
}
```

The error path is where correctness lives. `clear_nlink` + `iput` is what causes `->evict_inode` to free the inode's resources. Without it, you leak an inode on every failed create — invisible until `fsck` finds it.

Note also: this sequence is **not atomic**. A crash between `simplefs_new_inode` and `simplefs_add_entry` leaves an allocated, unreferenced inode. That is precisely the orphan `fsck` looks for, and precisely what a journal (Ch. 61) exists to prevent.

### 1.6 `readdir`

```c
static int simplefs_readdir(struct file *file, struct dir_context *ctx)
{
	struct inode *dir = file_inode(file);
	struct super_block *sb = dir->i_sb;
	struct buffer_head *bh;
	struct simplefs_dir_entry *de;
	uint32_t blk_idx, i, phys;

	if (!dir_emit_dots(file, ctx))          /* handles "." and ".." */
		return 0;

	if (ctx->pos < 2)
		return 0;

	/* ctx->pos is (2 + entry index). Stable across add/remove because
	 * entries never move -- T.6's readdir-offset requirement. */
	for (i = ctx->pos - 2; i < dir->i_size / sizeof(*de); i++) {
		blk_idx = i / SIMPLEFS_ENTRIES_PER_BLOCK;
		phys = simplefs_bmap(dir, blk_idx);
		if (!phys) { ctx->pos = i + 2 + SIMPLEFS_ENTRIES_PER_BLOCK; continue; }

		bh = sb_bread(sb, phys);
		if (!bh)
			return -EIO;
		de = (struct simplefs_dir_entry *)bh->b_data
		     + (i % SIMPLEFS_ENTRIES_PER_BLOCK);

		if (le32_to_cpu(de->ino)) {
			size_t len = strnlen(de->name, SIMPLEFS_MAX_FILENAME);

			if (!dir_emit(ctx, de->name, len,
				      le32_to_cpu(de->ino), DT_UNKNOWN)) {
				brelse(bh);
				return 0;         /* userspace buffer full */
			}
		}
		brelse(bh);
		ctx->pos = i + 3;
	}
	return 0;
}
```

`strnlen` rather than `strlen` — the name from disk may not be NUL-terminated (§T.5). `dir_emit` returning false is not an error: it means the userspace buffer is full and `getdents` will be called again from `ctx->pos`.

### 1.7 Everything else, briefly

```c
static const struct super_operations simplefs_super_ops = {
	.alloc_inode	= simplefs_alloc_inode,
	.free_inode	= simplefs_free_inode,
	.write_inode	= simplefs_write_inode,
	.evict_inode	= simplefs_evict_inode,
	.put_super	= simplefs_put_super,
	.sync_fs	= simplefs_sync_fs,
	.statfs		= simplefs_statfs,
	.show_options	= simplefs_show_options,
};

static const struct inode_operations simplefs_dir_inode_ops = {
	.lookup		= simplefs_lookup,
	.create		= simplefs_create,
	.mkdir		= simplefs_mkdir,
	.rmdir		= simplefs_rmdir,
	.unlink		= simplefs_unlink,
	.link		= simplefs_link,
	.symlink	= simplefs_symlink,
	.mknod		= simplefs_mknod,
	.rename		= simplefs_rename,
	.getattr	= simplefs_getattr,
	.setattr	= simplefs_setattr,
};

static const struct file_operations simplefs_file_ops = {
	.llseek		= generic_file_llseek,
	.read_iter	= generic_file_read_iter,
	.write_iter	= generic_file_write_iter,
	.mmap		= generic_file_mmap,
	.fsync		= generic_file_fsync,
	.splice_read	= filemap_splice_read,
	.splice_write	= iter_file_splice_write,
};

static const struct address_space_operations simplefs_aops = {
	.read_folio	= simplefs_read_folio,
	.readahead	= simplefs_readahead,
	.writepages	= simplefs_writepages,
	.write_begin	= simplefs_write_begin,
	.write_end	= simplefs_write_end,
	.bmap		= simplefs_bmap_aop,
	.dirty_folio	= block_dirty_folio,
	.invalidate_folio = block_invalidate_folio,
	.migrate_folio	= buffer_migrate_folio,
};
```

The `file_operations` are entirely generic — that is `libfs` and `mm/filemap.c` doing the work. Your filesystem supplies only `->get_block` (or `iomap_ops`, per Ch. 55) and everything above it is free.

---

## 2. Practice

### Lab 56.1 — A minimal in-memory filesystem

```c
// SPDX-License-Identifier: GPL-2.0
/* tinyfs.c -- ~150 lines, fully functional, entirely on libfs. */
#include <linux/fs.h>
#include <linux/fs_context.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/pagemap.h>
#include <linux/seq_file.h>
#include <linux/statfs.h>

#define TINYFS_MAGIC 0x54494e59	/* "TINY" */

static const struct inode_operations tinyfs_dir_inode_ops;

static struct inode *tinyfs_get_inode(struct super_block *sb,
				      const struct inode *dir, umode_t mode)
{
	struct inode *inode = new_inode(sb);

	if (!inode)
		return NULL;

	inode->i_ino = get_next_ino();
	inode_init_owner(&nop_mnt_idmap, inode, dir, mode);
	simple_inode_init_ts(inode);
	inode->i_mapping->a_ops = &ram_aops;
	mapping_set_gfp_mask(inode->i_mapping, GFP_HIGHUSER);
	mapping_set_unevictable(inode->i_mapping);

	switch (mode & S_IFMT) {
	case S_IFREG:
		inode->i_op  = &simple_file_inode_operations;
		inode->i_fop = &generic_ro_fops;
		inode->i_fop = &simple_file_operations;
		break;
	case S_IFDIR:
		inode->i_op  = &tinyfs_dir_inode_ops;
		inode->i_fop = &simple_dir_operations;
		inc_nlink(inode);        /* for "." */
		break;
	case S_IFLNK:
		inode->i_op = &page_symlink_inode_operations;
		inode_nohighmem(inode);
		break;
	default:
		init_special_inode(inode, mode, 0);
		break;
	}
	return inode;
}

static int tinyfs_mknod(struct mnt_idmap *idmap, struct inode *dir,
			struct dentry *dentry, umode_t mode, dev_t dev)
{
	struct inode *inode = tinyfs_get_inode(dir->i_sb, dir, mode);

	if (!inode)
		return -ENOSPC;
	d_instantiate(dentry, inode);
	dget(dentry);                    /* the dcache holds the only ref */
	inode_set_mtime_to_ts(dir, inode_set_ctime_current(dir));
	return 0;
}

static int tinyfs_create(struct mnt_idmap *idmap, struct inode *dir,
			 struct dentry *dentry, umode_t mode, bool excl)
{
	return tinyfs_mknod(idmap, dir, dentry, mode | S_IFREG, 0);
}

static int tinyfs_mkdir(struct mnt_idmap *idmap, struct inode *dir,
			struct dentry *dentry, umode_t mode)
{
	int ret = tinyfs_mknod(idmap, dir, dentry, mode | S_IFDIR, 0);

	if (!ret)
		inc_nlink(dir);          /* the new dir's ".." */
	return ret;
}

static int tinyfs_symlink(struct mnt_idmap *idmap, struct inode *dir,
			  struct dentry *dentry, const char *symname)
{
	struct inode *inode = tinyfs_get_inode(dir->i_sb, dir, S_IFLNK | 0777);
	int ret;

	if (!inode)
		return -ENOSPC;
	ret = page_symlink(inode, symname, strlen(symname) + 1);
	if (ret) { iput(inode); return ret; }
	d_instantiate(dentry, inode);
	dget(dentry);
	return 0;
}

static const struct inode_operations tinyfs_dir_inode_ops = {
	.lookup		= simple_lookup,
	.create		= tinyfs_create,
	.mkdir		= tinyfs_mkdir,
	.rmdir		= simple_rmdir,
	.unlink		= simple_unlink,
	.link		= simple_link,
	.symlink	= tinyfs_symlink,
	.mknod		= tinyfs_mknod,
	.rename		= simple_rename,
};

static const struct super_operations tinyfs_super_ops = {
	.statfs		= simple_statfs,
	.drop_inode	= generic_delete_inode,
};

static int tinyfs_fill_super(struct super_block *sb, struct fs_context *fc)
{
	struct inode *root;

	sb->s_maxbytes		= MAX_LFS_FILESIZE;
	sb->s_blocksize		= PAGE_SIZE;
	sb->s_blocksize_bits	= PAGE_SHIFT;
	sb->s_magic		= TINYFS_MAGIC;
	sb->s_op		= &tinyfs_super_ops;
	sb->s_time_gran		= 1;

	root = tinyfs_get_inode(sb, NULL, S_IFDIR | 0755);
	sb->s_root = d_make_root(root);
	return sb->s_root ? 0 : -ENOMEM;
}

static int tinyfs_get_tree(struct fs_context *fc)
{
	return get_tree_nodev(fc, tinyfs_fill_super);
}

static const struct fs_context_operations tinyfs_context_ops = {
	.get_tree = tinyfs_get_tree,
};

static int tinyfs_init_fs_context(struct fs_context *fc)
{
	fc->ops = &tinyfs_context_ops;
	return 0;
}

static struct file_system_type tinyfs_type = {
	.owner			= THIS_MODULE,
	.name			= "tinyfs",
	.init_fs_context	= tinyfs_init_fs_context,
	.kill_sb		= kill_litter_super,
	.fs_flags		= FS_USERNS_MOUNT,
};
MODULE_ALIAS_FS("tinyfs");

static int __init tinyfs_init(void) { return register_filesystem(&tinyfs_type); }
static void __exit tinyfs_exit(void) { unregister_filesystem(&tinyfs_type); }
module_init(tinyfs_init);
module_exit(tinyfs_exit);
MODULE_LICENSE("GPL");
```

```sh
cat > Makefile <<'EOF'
obj-m += tinyfs.o
all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
EOF
make
sudo insmod tinyfs.ko
sudo mkdir -p /mnt/tiny
sudo mount -t tinyfs none /mnt/tiny

sudo sh -c 'echo hello > /mnt/tiny/file'
cat /mnt/tiny/file
sudo mkdir -p /mnt/tiny/a/b/c
sudo ln -s file /mnt/tiny/link
ls -lR /mnt/tiny
stat -f /mnt/tiny

# It behaves like a real filesystem:
sudo dd if=/dev/zero of=/mnt/tiny/big bs=1M count=100
free -m                       # memory consumed: this is tmpfs-like
sudo rm /mnt/tiny/big
free -m

sudo umount /mnt/tiny && sudo rmmod tinyfs
```

**Exercises:**
- Add `->statfs` reporting real numbers by tracking allocation.
- Add mount options via §T.9 (`size=`, `mode=`).
- Run `xfstests`' generic quick group against it and count failures. Each failure is a VFS contract you have not met.
- Compare your module line-for-line with `fs/ramfs/inode.c`. Note what they did differently and work out why.

---

### Lab 56.2 — `mkfs.simplefs`

```c
// SPDX-License-Identifier: GPL-2.0
/* mkfs.c -- userspace; shares simplefs.h with the module. */
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ioctl.h>
#include <sys/stat.h>
#include <time.h>
#include <unistd.h>
#include <linux/fs.h>
#include "simplefs.h"

static void set_bit_le(uint8_t *map, uint32_t bit)
{
	map[bit / 8] |= 1 << (bit % 8);
}

int main(int argc, char **argv)
{
	int fd;
	uint64_t dev_bytes;
	uint32_t nr_blocks, nr_inodes, ib, bb, itbl, data, i;
	struct simplefs_super_block sb = {0};
	uint8_t *buf;
	struct simplefs_inode *itab;
	struct simplefs_dir_entry *de;

	if (argc != 2) { fprintf(stderr, "usage: mkfs.simplefs DEV\n"); return 1; }

	fd = open(argv[1], O_RDWR);
	if (fd < 0) { perror("open"); return 1; }

	if (ioctl(fd, BLKGETSIZE64, &dev_bytes) < 0) {
		struct stat st;
		if (fstat(fd, &st) < 0) { perror("stat"); return 1; }
		dev_bytes = st.st_size;
	}
	nr_blocks = dev_bytes / SIMPLEFS_BLOCK_SIZE;
	if (nr_blocks < 16) { fprintf(stderr, "device too small\n"); return 1; }

	/* One inode per 4 data blocks -- a common heuristic. */
	nr_inodes = nr_blocks / 4;
	if (nr_inodes < 16) nr_inodes = 16;

	ib   = (nr_inodes + SIMPLEFS_BLOCK_SIZE * 8 - 1) / (SIMPLEFS_BLOCK_SIZE * 8);
	bb   = (nr_blocks + SIMPLEFS_BLOCK_SIZE * 8 - 1) / (SIMPLEFS_BLOCK_SIZE * 8);
	itbl = 1 + ib + bb;
	data = itbl + (nr_inodes + SIMPLEFS_INODES_PER_BLOCK - 1)
		      / SIMPLEFS_INODES_PER_BLOCK;

	if (data >= nr_blocks) { fprintf(stderr, "device too small\n"); return 1; }

	buf = calloc(1, SIMPLEFS_BLOCK_SIZE);

	/* --- superblock --- */
	sb.magic	     = htole32(SIMPLEFS_MAGIC);
	sb.nr_blocks	     = htole32(nr_blocks);
	sb.nr_inodes	     = htole32(nr_inodes);
	sb.nr_free_blocks    = htole32(nr_blocks - data - 1);
	sb.nr_free_inodes    = htole32(nr_inodes - SIMPLEFS_ROOT_INO - 1);
	sb.nr_ibitmap_blocks = htole32(ib);
	sb.nr_bbitmap_blocks = htole32(bb);
	sb.inode_table_start = htole32(itbl);
	sb.data_start	     = htole32(data);
	sb.state	     = htole32(1);          /* CLEAN */
	sb.mtime = sb.wtime  = htole32(time(NULL));
	{
		int rf = open("/dev/urandom", O_RDONLY);
		read(rf, sb.uuid, sizeof(sb.uuid)); close(rf);
	}
	pwrite(fd, &sb, sizeof(sb), 0);

	/* --- inode bitmap: mark 0..ROOT_INO used --- */
	memset(buf, 0, SIMPLEFS_BLOCK_SIZE);
	for (i = 0; i <= SIMPLEFS_ROOT_INO; i++)
		set_bit_le(buf, i);
	pwrite(fd, buf, SIMPLEFS_BLOCK_SIZE, (off_t)1 * SIMPLEFS_BLOCK_SIZE);
	memset(buf, 0, SIMPLEFS_BLOCK_SIZE);
	for (i = 1; i < ib; i++)
		pwrite(fd, buf, SIMPLEFS_BLOCK_SIZE, (off_t)(1 + i) * SIMPLEFS_BLOCK_SIZE);

	/* --- block bitmap: metadata + the root's first data block --- */
	memset(buf, 0, SIMPLEFS_BLOCK_SIZE);
	for (i = 0; i <= data; i++)
		set_bit_le(buf, i);
	pwrite(fd, buf, SIMPLEFS_BLOCK_SIZE, (off_t)(1 + ib) * SIMPLEFS_BLOCK_SIZE);
	memset(buf, 0, SIMPLEFS_BLOCK_SIZE);
	for (i = 1; i < bb; i++)
		pwrite(fd, buf, SIMPLEFS_BLOCK_SIZE,
		       (off_t)(1 + ib + i) * SIMPLEFS_BLOCK_SIZE);

	/* --- inode table: the root directory --- */
	memset(buf, 0, SIMPLEFS_BLOCK_SIZE);
	itab = (struct simplefs_inode *)buf;
	itab[SIMPLEFS_ROOT_INO].mode  = htole16(S_IFDIR | 0755);
	itab[SIMPLEFS_ROOT_INO].nlink = htole16(2);        /* "." and ".." */
	itab[SIMPLEFS_ROOT_INO].size  = htole64(2 * sizeof(struct simplefs_dir_entry));
	itab[SIMPLEFS_ROOT_INO].blocks = htole32(SIMPLEFS_BLOCK_SIZE / 512);
	itab[SIMPLEFS_ROOT_INO].atime =
	itab[SIMPLEFS_ROOT_INO].mtime =
	itab[SIMPLEFS_ROOT_INO].ctime = htole64(time(NULL));
	itab[SIMPLEFS_ROOT_INO].generation = htole32(1);
	itab[SIMPLEFS_ROOT_INO].block[0] = htole32(data);
	pwrite(fd, buf, SIMPLEFS_BLOCK_SIZE, (off_t)itbl * SIMPLEFS_BLOCK_SIZE);

	/* --- the root directory's data block: "." and ".." --- */
	memset(buf, 0, SIMPLEFS_BLOCK_SIZE);
	de = (struct simplefs_dir_entry *)buf;
	de[0].ino = htole32(SIMPLEFS_ROOT_INO); strcpy(de[0].name, ".");
	de[1].ino = htole32(SIMPLEFS_ROOT_INO); strcpy(de[1].name, "..");
	pwrite(fd, buf, SIMPLEFS_BLOCK_SIZE, (off_t)data * SIMPLEFS_BLOCK_SIZE);

	fsync(fd);
	close(fd);
	free(buf);

	printf("simplefs on %s:\n", argv[1]);
	printf("  blocks      %u (%.1f MiB)\n", nr_blocks,
	       nr_blocks * SIMPLEFS_BLOCK_SIZE / 1048576.0);
	printf("  inodes      %u\n", nr_inodes);
	printf("  ibitmap     %u block(s) at 1\n", ib);
	printf("  bbitmap     %u block(s) at %u\n", bb, 1 + ib);
	printf("  inode table at %u\n", itbl);
	printf("  data        at %u\n", data);
	return 0;
}
```

```sh
gcc -O2 -Wall -o mkfs.simplefs mkfs.c
dd if=/dev/zero of=/tmp/sfs.img bs=1M count=64
./mkfs.simplefs /tmp/sfs.img
hexdump -C /tmp/sfs.img | head -8       # the superblock
hexdump -C -s $((4096*15)) -n 256 /tmp/sfs.img   # adjust to your data_start
```

Verifying the layout by hand with `hexdump` before writing any kernel code is the fastest way to catch format bugs.

---

### Lab 56.3 — Build and mount simplefs

Assemble the module from §1.2–§1.7 (this is the bulk of the chapter's work; implement `simplefs_find_entry`, `add_entry`, `remove_entry`, `alloc_block`, `free_block`, `get_block`, and `write_inode` yourself).

```sh
make
sudo insmod simplefs.ko
sudo losetup -fP --show /tmp/sfs.img     # -> /dev/loopN
sudo mkdir -p /mnt/sfs
sudo mount -t simplefs /dev/loop0 /mnt/sfs

sudo sh -c 'echo "persistent!" > /mnt/sfs/hello'
cat /mnt/sfs/hello
sudo mkdir /mnt/sfs/dir
sudo cp /etc/hostname /mnt/sfs/dir/
ls -lR /mnt/sfs
df -h /mnt/sfs

# The real test: does it survive?
sudo umount /mnt/sfs
sudo mount -t simplefs /dev/loop0 /mnt/sfs
cat /mnt/sfs/hello              # "persistent!"
ls -lR /mnt/sfs
```

Then the correctness tests:

```sh
# Basic POSIX behaviour
cd /mnt/sfs
sudo touch a; sudo ln a b; stat -c '%h %i' a b     # nlink 2, same inode
sudo rm a; stat -c '%h' b                          # nlink 1
sudo mkdir d; stat -c '%h' d                       # 2 (. and parent entry)
sudo mkdir d/e; stat -c '%h' d                     # 3 (+ e's "..")
sudo rmdir d/e d

# Large-ish file (tests indirect blocks)
sudo dd if=/dev/urandom of=big bs=4k count=1000
md5sum big > /tmp/sum1
sudo umount /mnt/sfs && sudo mount -t simplefs /dev/loop0 /mnt/sfs
md5sum -c /tmp/sum1

# Fill it up
sudo dd if=/dev/zero of=fill bs=1M count=100 2>&1 | tail -2   # ENOSPC
df -h /mnt/sfs
sudo rm fill
df -h /mnt/sfs                  # space returned?
```

Run the filesystem test suite:

```sh
git clone https://git.kernel.org/pub/scm/fs/xfs/xfstests-dev.git
cd xfstests-dev && make
# Configure for simplefs and run the generic group:
sudo ./check -g generic/quick 2>&1 | tail -40
```

Expect many failures at first. **Each failure names a contract you have not met**, and fixing them in order is the single best way to learn the VFS.

---

### Lab 56.4 — Attack your own filesystem

This is the most important lab in the chapter.

```c
// SPDX-License-Identifier: GPL-2.0
/* corrupt.c -- targeted corruption of specific superblock fields. */
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include "simplefs.h"

struct attack { const char *name; void (*apply)(struct simplefs_super_block *); };

static void a_nr_blocks_huge(struct simplefs_super_block *s)
	{ s->nr_blocks = htole32(0xFFFFFFFF); }
static void a_nr_inodes_zero(struct simplefs_super_block *s)
	{ s->nr_inodes = htole32(0); }
static void a_bitmap_huge(struct simplefs_super_block *s)
	{ s->nr_ibitmap_blocks = htole32(0x7FFFFFFF); }
static void a_itbl_past_end(struct simplefs_super_block *s)
	{ s->inode_table_start = htole32(le32toh(s->nr_blocks) + 1000); }
static void a_data_before_itbl(struct simplefs_super_block *s)
	{ s->data_start = htole32(0); }
static void a_overlap(struct simplefs_super_block *s)
	{ s->inode_table_start = htole32(1); }
static void a_free_gt_total(struct simplefs_super_block *s)
	{ s->nr_free_blocks = htole32(le32toh(s->nr_blocks) * 2); }

static struct attack attacks[] = {
	{ "nr_blocks = UINT32_MAX",      a_nr_blocks_huge },
	{ "nr_inodes = 0",               a_nr_inodes_zero },
	{ "ibitmap_blocks = INT32_MAX",  a_bitmap_huge },
	{ "inode_table past device end", a_itbl_past_end },
	{ "data_start = 0",              a_data_before_itbl },
	{ "inode_table overlaps bitmap", a_overlap },
	{ "free_blocks > nr_blocks",     a_free_gt_total },
	{ NULL, NULL }
};

int main(int argc, char **argv)
{
	int which = atoi(argv[2]), fd;
	struct simplefs_super_block sb;

	fd = open(argv[1], O_RDWR);
	pread(fd, &sb, sizeof(sb), 0);
	printf("applying: %s\n", attacks[which].name);
	attacks[which].apply(&sb);
	pwrite(fd, &sb, sizeof(sb), 0);
	fsync(fd);
	close(fd);
	return 0;
}
```

```sh
gcc -o corrupt corrupt.c
for i in 0 1 2 3 4 5 6; do
  cp /tmp/sfs.img /tmp/bad.img
  ./corrupt /tmp/bad.img $i
  sudo losetup -f --show /tmp/bad.img
  echo "--- attack $i ---"
  sudo mount -t simplefs /dev/loop1 /mnt/sfs 2>&1
  sudo umount /mnt/sfs 2>/dev/null
  sudo losetup -d /dev/loop1
  dmesg | tail -3
done
```

**Every one must fail cleanly with a diagnostic, not oops, not hang, not corrupt memory.** Run the whole sweep under KASAN:

```sh
# Kernel built with CONFIG_KASAN=y, CONFIG_UBSAN=y, CONFIG_KFENCE=y
dmesg -w | grep -E 'KASAN|UBSAN|BUG|Oops' &
# ... repeat the sweep ...
```

Now go further: fuzz it.

```sh
# Random single-bit flips over the metadata region
for trial in $(seq 1 500); do
  cp /tmp/sfs.img /tmp/fuzz.img
  OFF=$((RANDOM % (4096 * 20)))
  BIT=$((RANDOM % 8))
  python3 -c "
import sys
f=open('/tmp/fuzz.img','r+b'); f.seek($OFF)
b=f.read(1)[0]; f.seek($OFF); f.write(bytes([b ^ (1 << $BIT)])); f.close()"
  L=$(sudo losetup -f --show /tmp/fuzz.img)
  timeout 10 sudo mount -t simplefs $L /mnt/sfs 2>/dev/null && {
      timeout 10 sudo find /mnt/sfs -type f -exec cat {} \; > /dev/null 2>&1
      sudo umount /mnt/sfs
  }
  sudo losetup -d $L
  dmesg | grep -qE 'KASAN|BUG|Oops' && { echo "CRASH at offset $OFF bit $BIT"; break; }
done
```

For serious work, use the kernel's own harness:

```sh
# syzkaller has a dedicated filesystem-image fuzzing mode
# See tools/testing/ and the syzkaller docs for syz_mount_image.
```

**This lab is the entire point of §T.5.** Filesystem fuzzing routinely finds bugs in *production* filesystems; it will certainly find bugs in yours. Fix every one and re-run.

---

### Lab 56.5 — `fsck.simplefs`

```c
// SPDX-License-Identifier: GPL-2.0
/* fsck.c -- checks the invariants of T.10. */
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include "simplefs.h"

static int errors, fixes;
static uint8_t *seen_block, *seen_inode;
static uint16_t *nlink_count;
static struct simplefs_super_block sb;
static int fd, do_fix;

#define ERR(fmt, ...) do { errors++; printf("ERROR: " fmt "\n", ##__VA_ARGS__); } while (0)
#define FIX(fmt, ...) do { fixes++;  printf("FIXED: " fmt "\n", ##__VA_ARGS__); } while (0)

static void read_block(uint32_t b, void *buf)
	{ pread(fd, buf, SIMPLEFS_BLOCK_SIZE, (off_t)b * SIMPLEFS_BLOCK_SIZE); }

static void read_inode(uint32_t ino, struct simplefs_inode *out)
{
	uint8_t buf[SIMPLEFS_BLOCK_SIZE];
	uint32_t blk = le32toh(sb.inode_table_start) + ino / SIMPLEFS_INODES_PER_BLOCK;

	read_block(blk, buf);
	memcpy(out, (struct simplefs_inode *)buf + ino % SIMPLEFS_INODES_PER_BLOCK,
	       sizeof(*out));
}

/* Pass 1: block accounting */
static void check_inode_blocks(uint32_t ino)
{
	struct simplefs_inode di;
	uint32_t i, b, data = le32toh(sb.data_start), nb = le32toh(sb.nr_blocks);

	read_inode(ino, &di);
	if (!le16toh(di.nlink))
		return;

	for (i = 0; i <= SIMPLEFS_N_DIRECT; i++) {
		b = le32toh(di.block[i]);
		if (!b) continue;
		if (b < data || b >= nb) {
			ERR("inode %u: block[%u] = %u out of range [%u, %u)",
			    ino, i, b, data, nb);
			continue;
		}
		if (seen_block[b]) {
			ERR("block %u multiply claimed (inode %u and %u)",
			    b, seen_block[b], ino);
		}
		seen_block[b] = 1;
		/* TODO: walk the indirect block too */
	}
}

/* Pass 2: directory tree walk, nlink counting, cycle detection */
static void walk_dir(uint32_t ino, uint32_t parent, int depth)
{
	struct simplefs_inode di;
	uint8_t buf[SIMPLEFS_BLOCK_SIZE];
	struct simplefs_dir_entry *de;
	uint32_t i, j, b;
	int saw_dot = 0, saw_dotdot = 0;

	if (depth > 64) { ERR("directory nesting > 64 at inode %u (cycle?)", ino); return; }
	if (seen_inode[ino]) { ERR("inode %u reachable twice (cycle)", ino); return; }
	seen_inode[ino] = 1;

	read_inode(ino, &di);
	for (i = 0; i < SIMPLEFS_N_DIRECT; i++) {
		b = le32toh(di.block[i]);
		if (!b) continue;
		read_block(b, buf);
		de = (struct simplefs_dir_entry *)buf;
		for (j = 0; j < SIMPLEFS_ENTRIES_PER_BLOCK; j++) {
			uint32_t child = le32toh(de[j].ino);
			struct simplefs_inode cdi;
			char name[SIMPLEFS_MAX_FILENAME + 1];

			if (!child) continue;
			if (child >= le32toh(sb.nr_inodes)) {
				ERR("dir %u: entry points to bad inode %u", ino, child);
				continue;
			}
			memcpy(name, de[j].name, SIMPLEFS_MAX_FILENAME);
			name[SIMPLEFS_MAX_FILENAME] = 0;

			if (!strcmp(name, ".")) {
				saw_dot = 1;
				if (child != ino) ERR("dir %u: '.' -> %u", ino, child);
				nlink_count[ino]++;
				continue;
			}
			if (!strcmp(name, "..")) {
				saw_dotdot = 1;
				if (child != parent)
					ERR("dir %u: '..' -> %u, expected %u",
					    ino, child, parent);
				nlink_count[parent]++;
				continue;
			}
			nlink_count[child]++;
			read_inode(child, &cdi);
			if (S_ISDIR(le16toh(cdi.mode)))
				walk_dir(child, ino, depth + 1);
			else
				seen_inode[child] = 1;
		}
	}
	if (!saw_dot)    ERR("dir %u: missing '.'", ino);
	if (!saw_dotdot) ERR("dir %u: missing '..'", ino);
}

int main(int argc, char **argv)
{
	uint32_t i, nb, ni, bm_base, im_base;
	uint8_t *bitmap;

	do_fix = (argc > 2 && !strcmp(argv[2], "-y"));
	fd = open(argv[1], do_fix ? O_RDWR : O_RDONLY);
	pread(fd, &sb, sizeof(sb), 0);

	if (le32toh(sb.magic) != SIMPLEFS_MAGIC) {
		printf("not a simplefs filesystem\n"); return 8;
	}
	nb = le32toh(sb.nr_blocks);
	ni = le32toh(sb.nr_inodes);
	printf("simplefs: %u blocks, %u inodes\n", nb, ni);

	seen_block  = calloc(nb, 1);
	seen_inode  = calloc(ni, 1);
	nlink_count = calloc(ni, sizeof(*nlink_count));

	printf("Pass 1: checking blocks and sizes\n");
	for (i = 0; i <= le32toh(sb.data_start); i++)
		seen_block[i] = 1;                      /* metadata */
	for (i = SIMPLEFS_ROOT_INO; i < ni; i++)
		check_inode_blocks(i);

	printf("Pass 2: checking directory structure\n");
	walk_dir(SIMPLEFS_ROOT_INO, SIMPLEFS_ROOT_INO, 0);

	printf("Pass 3: checking link counts\n");
	for (i = SIMPLEFS_ROOT_INO; i < ni; i++) {
		struct simplefs_inode di;

		read_inode(i, &di);
		if (!le16toh(di.nlink) && !nlink_count[i])
			continue;
		if (le16toh(di.nlink) != nlink_count[i])
			ERR("inode %u: nlink is %u, should be %u",
			    i, le16toh(di.nlink), nlink_count[i]);
		if (le16toh(di.nlink) && !seen_inode[i])
			ERR("inode %u: orphaned (nlink %u, unreachable)",
			    i, le16toh(di.nlink));
	}

	printf("Pass 4: checking the block bitmap\n");
	bitmap = calloc(le32toh(sb.nr_bbitmap_blocks), SIMPLEFS_BLOCK_SIZE);
	bm_base = 1 + le32toh(sb.nr_ibitmap_blocks);
	for (i = 0; i < le32toh(sb.nr_bbitmap_blocks); i++)
		read_block(bm_base + i, bitmap + i * SIMPLEFS_BLOCK_SIZE);
	for (i = 0; i < nb; i++) {
		int on_disk = (bitmap[i / 8] >> (i % 8)) & 1;

		if (on_disk != seen_block[i])
			ERR("block %u: bitmap says %s, tree walk says %s",
			    i, on_disk ? "used" : "free",
			    seen_block[i] ? "used" : "free");
	}

	printf("\n%d error(s), %d fix(es)\n", errors, fixes);
	return errors ? 4 : 0;
}
```

```sh
gcc -O2 -o fsck.simplefs fsck.c
./fsck.simplefs /tmp/sfs.img                 # clean
# Now break something and see it caught:
./corrupt /tmp/sfs.img 6
./fsck.simplefs /tmp/sfs.img
```

**Exercises:** add indirect-block walking; add `lost+found` reconnection for orphans; add a `-y` repair mode for `nlink` mismatches; write a `-n` (check only) mode; then **use `fsck` after every crash test in Lab 56.6**.

---

### Lab 56.6 — Crash consistency, or the absence of it

```sh
# Set up a device you can "power fail"
sudo dmsetup create flaky --table \
  "0 $(sudo blockdev --getsz /dev/loop0) linear /dev/loop0 0"
sudo mount -t simplefs /dev/mapper/flaky /mnt/sfs

# Write, then cut power mid-operation
sudo sh -c 'for i in $(seq 1 200); do echo data > /mnt/sfs/f$i; done' &
WRITER=$!
sleep 0.2
sudo dmsetup suspend flaky
sudo dmsetup load flaky --table \
  "0 $(sudo blockdev --getsz /dev/loop0) error"
sudo dmsetup resume flaky        # all further I/O fails: a "power cut"
kill $WRITER 2>/dev/null
sudo umount -f /mnt/sfs 2>/dev/null

# Restore and check
sudo dmsetup suspend flaky
sudo dmsetup load flaky --table \
  "0 $(sudo blockdev --getsz /dev/loop0) linear /dev/loop0 0"
sudo dmsetup resume flaky
./fsck.simplefs /tmp/sfs.img
```

Expect errors: orphaned inodes, bitmap mismatches, wrong `nlink`. **This is §1.5's non-atomicity, realised.**

Now use `dm-log-writes` to replay every possible crash point:

```sh
# Record every write with its flush/FUA flags
sudo dmsetup create logwrites --table \
  "0 $(sudo blockdev --getsz /dev/loop0) log-writes /dev/loop0 /dev/loop1"
sudo mount -t simplefs /dev/mapper/logwrites /mnt/sfs
sudo sh -c 'echo hello > /mnt/sfs/x; sync'
sudo umount /mnt/sfs

# Replay to each flush point and fsck
for mark in $(sudo dmsetup message logwrites 0 mark list 2>/dev/null); do
  sudo dmsetup message logwrites 0 "replay $mark"
  ./fsck.simplefs /tmp/sfs.img | tail -1
done
```

Then implement enough ordering to fix the worst cases:

```c
/* Minimal ordering: write the data, flush, THEN the metadata that
 * references it. Not a journal, but it eliminates one class of bug. */
static int simplefs_write_inode(struct inode *inode,
				struct writeback_control *wbc)
{
	...
	if (wbc->sync_mode == WB_SYNC_ALL) {
		sync_dirty_buffer(bh);
		if (buffer_req(bh) && !buffer_uptodate(bh))
			return -EIO;
	}
	...
}
```

Measure: how much does correct ordering cost? Then note that a journal costs less *and* gives more — which is Ch. 61.

---

### Lab 56.7 — Allocation policy, measured

```c
/* Version A: naive -- first free block in the bitmap. */
static uint32_t alloc_block_naive(struct super_block *sb)
{
	struct simplefs_sb_info *sbi = SIMPLEFS_SB(sb);
	uint32_t b = find_next_zero_bit(sbi->bbitmap, sbi->nr_blocks,
					sbi->data_start);
	if (b >= sbi->nr_blocks) return 0;
	set_bit(b, sbi->bbitmap);
	return b;
}

/* Version B: goal-directed -- start from the file's last block. */
static uint32_t alloc_block_goal(struct super_block *sb, uint32_t goal)
{
	struct simplefs_sb_info *sbi = SIMPLEFS_SB(sb);
	uint32_t b;

	if (goal >= sbi->data_start && goal < sbi->nr_blocks) {
		b = find_next_zero_bit(sbi->bbitmap, sbi->nr_blocks, goal);
		if (b < sbi->nr_blocks) goto got;
	}
	b = find_next_zero_bit(sbi->bbitmap, sbi->nr_blocks, sbi->data_start);
	if (b >= sbi->nr_blocks) return 0;
got:
	set_bit(b, sbi->bbitmap);
	return b;
}
```

Demonstrate the difference:

```sh
# Interleaved concurrent writers -- the worst case for a naive allocator
sudo mount -t simplefs /dev/loop0 /mnt/sfs
for i in 1 2 3 4; do
  ( for j in $(seq 1 500); do
      sudo dd if=/dev/zero of=/mnt/sfs/w$i bs=4k count=1 seek=$j \
              conv=notrunc 2>/dev/null
    done ) &
done
wait
sync
for i in 1 2 3 4; do filefrag /mnt/sfs/w$i; done
```

With the naive allocator the four files interleave block-for-block and every one is maximally fragmented. With goal-directed allocation they stay mostly contiguous. Then measure the read cost:

```sh
echo 3 | sudo tee /proc/sys/vm/drop_caches
time cat /mnt/sfs/w1 > /dev/null
sudo bpftrace -e 'tracepoint:block:block_bio_queue {
	@sz = hist(args->nr_sector * 512); }' &
cat /mnt/sfs/w1 > /dev/null
```

Then add preallocation (reserve 8 blocks per file on first allocation) and measure again. You have now independently derived three of the four policies in §T.7.

---

### Lab 56.8 — NFS export, and why generation numbers matter

```c
static struct dentry *simplefs_fh_to_dentry(struct super_block *sb,
					    struct fid *fid, int len, int type)
{
	return generic_fh_to_dentry(sb, fid, len, type, simplefs_nfs_get_inode);
}

static struct inode *simplefs_nfs_get_inode(struct super_block *sb,
					    u64 ino, u32 generation)
{
	struct inode *inode;

	if (ino < SIMPLEFS_ROOT_INO || ino >= SIMPLEFS_SB(sb)->nr_inodes)
		return ERR_PTR(-ESTALE);

	inode = simplefs_iget(sb, ino);
	if (IS_ERR(inode))
		return ERR_CAST(inode);

	/* THE check. Without it, a stale handle opens a DIFFERENT file. */
	if (generation && inode->i_generation != generation) {
		iput(inode);
		return ERR_PTR(-ESTALE);
	}
	return inode;
}

static struct dentry *simplefs_get_parent(struct dentry *child)
{
	struct qstr dotdot = QSTR_INIT("..", 2);
	uint32_t ino = simplefs_find_entry(d_inode(child), &dotdot);

	if (!ino)
		return ERR_PTR(-ENOENT);
	return d_obtain_alias(simplefs_iget(child->d_sb, ino));
}

static const struct export_operations simplefs_export_ops = {
	.encode_fh	= generic_encode_ino32_fh,
	.fh_to_dentry	= simplefs_fh_to_dentry,
	.fh_to_parent	= simplefs_fh_to_parent,
	.get_parent	= simplefs_get_parent,
};
```

Demonstrate why the generation check is essential:

```sh
sudo apt install -y nfs-kernel-server
echo "/mnt/sfs 127.0.0.1(rw,sync,no_subtree_check,fsid=42)" | \
  sudo tee -a /etc/exports
sudo exportfs -ra
sudo mkdir -p /mnt/nfs && sudo mount -t nfs 127.0.0.1:/mnt/sfs /mnt/nfs

# Open a file over NFS, then delete and recreate the underlying inode
sudo sh -c 'echo original > /mnt/sfs/target'
exec 8< /mnt/nfs/target                # the client holds a handle
sudo rm /mnt/sfs/target
sudo sh -c 'echo IMPOSTOR > /mnt/sfs/other'   # likely reuses the inode number

cat <&8                                # MUST be ESTALE, not "IMPOSTOR"
exec 8<&-
```

If you omitted the generation check, this reads the wrong file's data — a **cross-file information disclosure over the network**. Remove the check, rebuild, and see it happen. Then put it back.

Also test the disconnected-dentry path:

```sh
# Access a deep file by handle with nothing cached above it
sudo umount /mnt/sfs && sudo mount -t simplefs /dev/loop0 /mnt/sfs
# NFS will call fh_to_dentry and then reconnect via get_parent()
sudo bpftrace -e 'kprobe:simplefs_get_parent { printf("get_parent called\n"); }' &
cat /mnt/nfs/dir/deep/file
```

---

## 3. Mastery drills

1. Give the §T.1 four-map answer for ext2, ext4, XFS, and btrfs. Then place `simplefs` in that table and identify the two choices that most limit it.

2. `ramfs` is 300 lines; `ext2` is 10,000. Account for the 9,700-line difference by category, and identify which categories exist *only* because of persistence.

3. Design a feature-flag addition to `simplefs` that adds extents. Which of the three flag sets does it belong in, and why? Now do the same for adding a checksum, and for extending timestamps to nanoseconds.

4. Positional versus allocated inode numbers (§T.4): compute, for a 1 TiB filesystem, the space wasted by a positional scheme that provisions one inode per 4 KiB, and the cost of running out of inodes with 40 % free space.

5. Convert `simplefs` to variable-length directory entries. Enumerate every validation check the parser needs, and for each, write the crafted image that exploits its absence.

6. Prove that a linear directory makes creating n files O(n²). Measure it on `simplefs` with n = 1000, 5000, 20000. Then sketch the minimal hash index that fixes it and state what it does to `readdir` offset stability.

7. `d_splice_alias` versus `d_add` versus `d_instantiate` versus `d_instantiate_new`: state precisely when each is correct, and what breaks if you use the wrong one.

8. `iget_locked`/`I_NEW`/`unlock_new_inode` is Ch. 25's P5. Write the race that occurs if two lookups of the same inode number proceed concurrently without it.

9. §1.5's create is not atomic. Enumerate every crash point and the resulting inconsistency, then state the minimal write-ordering discipline that reduces the set to "leaked inode only."

10. Write the complete list of invariants `fsck.simplefs` should check (aim for 15+). For each, name the bug in the kernel module that would violate it.

11. Argue whether `simplefs` should support `fallocate`. If yes, which of Ch. 55's states must the format represent, and what format change does that require?

12. The generation number cannot be added after release without a format change. Enumerate three other fields with this property and justify each.

13. You are given a filesystem image that mounts but on which `ls` hangs forever. Give the ordered diagnostic procedure using only this chapter's tools, and the three most likely causes.

---

## 4. Further reading

**Kernel documentation**

- `Documentation/filesystems/vfs.rst` ★★★ — every method's contract. Read completely before starting.
- `Documentation/filesystems/locking.rst` ★★★ — what is held when each method is called. **Violating this is the most common new-filesystem bug.**
- `Documentation/filesystems/porting.rst` ★★★ — read the last three years' entries; the interfaces move.
- `Documentation/filesystems/mount_api.rst` ★★★ — §T.9.
- `Documentation/filesystems/ext2.rst`, `ext4/` ★★★ — on-disk format documentation worth imitating.
- `Documentation/filesystems/nfs/exporting.rst` ★★★ — §T.8, normatively.
- `Documentation/filesystems/directory-locking.rst` — needed for `rename`.
- `Documentation/filesystems/api-summary.rst`
- `Documentation/filesystems/idmappings.rst` — needed if you accept `struct mnt_idmap`.

**Source to read, in order**

1. `fs/libfs.c` ★★★ — before writing anything.
2. `fs/ramfs/inode.c` ★★★ — 300 lines; Lab 56.1's reference.
3. `fs/minix/` ★★★ — the best model for `simplefs`: a real on-disk filesystem small enough to read entirely in an afternoon.
4. `fs/ext2/` ★★★ — production quality without a journal; the reference for §T.3–T.7. Read `super.c` first, then `inode.c`, then `dir.c`, then `balloc.c`/`ialloc.c`.
5. `fs/exportfs/expfs.c` — §T.8's reconnection.
6. `fs/jbd2/` — after Ch. 61.

**Books and papers**

- Love, *Linux Kernel Development* (3rd ed.), Ch. 13 "The Virtual Filesystem" — the conceptual overview.
- Bovet & Cesati, *Understanding the Linux Kernel*, Ch. 12 and 18 — dated but the ext2 layout material remains excellent.
- Arpaci-Dusseau, *Operating Systems: Three Easy Pieces*, Ch. 39–43 ★★★ — free online; the clearest treatment of file-system implementation and crash consistency that exists. **Read chapters 40 and 42 before Lab 56.6.**
- McKusick et al., "A Fast File System for UNIX," ACM TOCS 1984 ★★★ — cylinder groups, i.e. §T.7's locality argument, in its original form.
- Card, Ts'o, Tweedie, "Design and Implementation of the Second Extended Filesystem," 1994 — `simplefs` is essentially a simplified ext2.
- Rosenblum & Ousterhout, "The Design and Implementation of a Log-Structured File System," 1992 ★★★ — background for Ch. 60.

**LWN**

- "Creating a Linux filesystem" / the various filesystem-writing tutorials
- "The new mount API" series ★★★
- "Filesystem fuzzing" and the syzkaller filesystem-image coverage ★★★ — Lab 56.4's motivation
- "Fixing the filesystem-image trust problem" — the ongoing debate about whether untrusted images should be mountable at all
- "The trouble with `d_splice_alias()`"
- "Idmapped mounts" — if your filesystem must handle them

**Other implementations to learn from**

- The `simplefs` teaching filesystem by Jim Huang / National Cheng Kung University (`sysprog21/simplefs` on GitHub) — an actively maintained pedagogical filesystem very close to this chapter's design.
- `psankar/simplefs` — an earlier, simpler variant with a good commit history to read chronologically.
- The Linux Kernel Labs filesystem exercises (`linux-kernel-labs.github.io`) ★★★ — guided, with solutions.

**Tools**

- `xfstests` ★★★ — the real conformance suite. Getting `generic/quick` to pass is a serious achievement and the right target.
- `dm-log-writes`, `dm-flakey`, `dm-error` ★★★ — Lab 56.6.
- `syzkaller` with `syz_mount_image` ★★★ — Lab 56.4, done properly.
- KASAN + UBSAN + KFENCE + lockdep + `CONFIG_DEBUG_ATOMIC_SLEEP` — run *everything* under these.
- `hexdump -C`, `xxd`, `od` — you will live in these while debugging the format.
- `blkid`, `file`, `losetup`, `dmsetup`
- `debugfs` (e2fsprogs) — study how it lets you inspect ext2 by hand, then write the equivalent for your format. It is the most useful debugging tool you can build.

---

→ Next: [57-ext4.md](57-ext4.md)

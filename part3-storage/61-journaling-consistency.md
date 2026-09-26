# Chapter 61 — Journaling and crash consistency (JBD2, `fsync`, barriers, FUA)

> **Goal:** Understand the problem every storage system must solve — how to make a multi-block update atomic on hardware that only guarantees single-sector atomicity — and every mechanism invented to solve it. Understand write-ahead logging and why it works, JBD2's transaction machinery in detail, ordering versus durability as two distinct requirements, `REQ_PREFLUSH`/`REQ_FUA` as the only ordering primitives Linux exposes, what `fsync` actually guarantees and the four ways applications get it wrong, why "fsyncgate" happened, and how to *test* crash consistency rather than hope. By the end you can reason about durability from the application's `write()` down to the NAND cell, and you can build a test harness that proves your reasoning.

---

## Theory & First Principles

### T.0 — Start here: cut the power in the middle of `rename()`

```c
rename("/data/new", "/data/live");   /* POSIX says: ATOMIC. */
```

POSIX promises that after this, `/data/live` is either the old file or the new one — never
neither, never both. But look at what the filesystem must actually do:

```
  1. remove the directory entry "new"       -> write block P
  2. add/replace the directory entry "live" -> write block Q
  3. decrement the link count of the old    -> write block R (inode table)
  4. update the free bitmap if it hit zero  -> write block S
  5. update the free-block counter          -> write block T (superblock)
```

**Five independent block writes. The device guarantees atomicity for exactly one sector.**
Cut power after step 1 and the file has *no name*. After step 2 and the link count is wrong —
the inode will never be freed, or will be freed while still referenced. **Every intermediate
state is corruption.**

```
   what the user was promised:   [old state] ------------> [new state]
   what the hardware provides:   [old] -> [?] -> [?] -> [?] -> [new]
                                          ^^^^^^^^^^^^^^^^
                                          THE ATOMICITY GAP
```

**The journal closes the gap by adding one level of indirection** — write down what you *are
about to do*, then do it:

```
  JOURNAL (a contiguous area on the device)
  +--------------------------------------------------+
  | [descriptor] P' Q' R' S' T' [COMMIT BLOCK]        |
  +--------------------------------------------------+
                                 ^
                                 |  ONE sector. Its presence, and its
                                 |  checksum matching, is the atomic
                                 |  decision point for the whole group.

  Crash BEFORE the commit block lands -> replay nothing. Old state. Valid.
  Crash AFTER  it lands               -> replay P'..T'. New state. Valid.
```

**That is the whole trick, and it is worth naming because you will meet it everywhere:**

> **Reduce an N-write atomic operation to a 1-write atomic operation by writing the intent
> first, and letting a single atomic marker decide whether the intent counts.**

Database WAL, `fsync`-then-`rename` in userspace, two-phase commit across machines, the
validate-then-apply pattern of Ch. 43 and Ch. 89, and btrfs's atomic root-pointer swap
(Ch. 59 §T.0) are all this same move. **Idempotent replay is the required companion
property:** replay must be safe to run any number of times, because a crash *during* replay
is normal.

**Now the part people get wrong — what the journal does and does not protect.**

| ext4 mode | Metadata | Data | A crash can leave |
|---|---|---|---|
| `data=journal` | journalled | **also journalled** | nothing inconsistent; ~2x write cost |
| `data=ordered` *(default)* | journalled | written **before** the metadata commit | correct metadata, possibly a stale or short file |
| `data=writeback` | journalled | unordered | metadata pointing at blocks containing **someone else's deleted data** |

`data=writeback` is fast and is an information-disclosure hazard. `data=ordered` costs an
ordering constraint and eliminates it. **This is a security decision disguised as a
performance tunable**, and it is the reason for the default.

**And the barrier nobody thinks about.** The journal only works if the commit block reaches
stable media *after* the journal data. The device's write cache will happily reorder them. So
the filesystem must issue a **cache flush** (`REQ_PREFLUSH` / `REQ_FUA`) — which costs
hundreds of microseconds and is exactly what makes `fsync` slow. Disable it (`nobarrier`, or
a device that lies about flushing) and you get a filesystem that is fast and **silently not
crash-safe**. Cheap consumer SSDs and some virtualization stacks have shipped exactly this
bug.

```bash
cat /proc/fs/jbd2/sda1-8/info          # commit counts, average commit time
tune2fs -l /dev/sda1 | grep -i journal
sudo dmesg | grep -i 'recovering journal'
sudo bpftrace -e 'kprobe:jbd2_journal_commit_transaction { @ = count(); }'
```

---

### T.1 The atomicity gap

A single file creation requires modifying, at minimum:

1. The inode bitmap (mark an inode used)
2. The inode table (write the new inode)
3. The directory data block (add the entry)
4. The directory's inode (mtime, possibly size)
5. The block bitmap and superblock counters

Five blocks, in five different places. The hardware guarantees **exactly one thing**: a single sector write is atomic — it either happens completely or not at all. (And even this is a convention rather than a standard; it holds because drives have enough residual power to finish a sector.)

So there is a gap:

> **The filesystem needs multi-block atomicity. The hardware provides single-sector atomicity.**

Every mechanism in this chapter bridges that gap. There are only four approaches:

| Approach | Used by | Mechanism |
|---|---|---|
| **Check and repair afterwards** | ext2, FFS | `fsck` reconstructs consistency from redundancy |
| **Careful ordering** | Soft updates (FreeBSD FFS) | order writes so any prefix is recoverable |
| **Write-ahead logging** | ext3/4, XFS, JBD2 | write intentions first, then do the work |
| **Copy-on-write** | btrfs, ZFS, LFS/F2FS | never overwrite; switch one pointer atomically |

The first fails at scale: `fsck` time is O(filesystem size), and at 100 TiB it is hours. The second is brilliant and almost nobody uses it — Ganger and Patt's soft updates require a dependency-tracking scheme so intricate that only one production implementation exists, and it still needs a background `fsck` for space accounting. The third and fourth dominate, and Part 3 has now covered both.

### T.2 Write-ahead logging

The idea, from database systems (System R, ARIES):

> **Before modifying anything in place, write a durable record of what you intend to do. Then, if you crash, either the record is complete (redo it) or it is not (ignore it).**

The protocol:

```
1. Write the journal descriptor + the new contents of all affected blocks
2. FLUSH                              <- the journal must be durable first
3. Write the commit record
4. FLUSH                              <- the commit must be durable
   ---- the transaction is now COMMITTED ----
5. Checkpoint: write the blocks to their real locations
6. Free the journal space
```

Recovery:

```
scan the journal from the last checkpoint
for each transaction:
    if it has a valid commit record: REPLAY it (write the blocks to their real places)
    else: STOP -- this transaction was in flight when we crashed; discard
```

Four properties make this work:

**(a) Replay must be idempotent.** Recovery may run repeatedly (crash during recovery). Physical logging — "block 1234 contains these 4096 bytes" — is idempotent by construction. Logical logging ("increment the free count") is not, and requires sequence numbers. JBD2 uses physical logging for exactly this reason.

**(b) The commit record's atomicity is the linchpin.** It is one sector, so the hardware guarantees it. The entire scheme reduces multi-block atomicity to single-sector atomicity. **That reduction is the whole idea.**

**(c) Both flushes are mandatory.** Without flush 2, the commit record could reach the platter before the data it commits — and recovery would replay garbage. Without flush 4, the checkpoint writes could beat the commit record, so a crash leaves the real locations half-updated with no journal record to fix them. Miss either and you have a filesystem that is *usually* fine and occasionally destroys itself.

**(d) Writes happen twice.** That is the cost: journal write + checkpoint write. Which is why ext3/4 default to journalling only *metadata* (`data=ordered`, Ch. 57 §T.6) — metadata is small, data is large.

### T.3 Ordering and durability are different requirements

This distinction is the most commonly confused thing in the chapter, and getting it right makes everything else clear.

| Requirement | Meaning | Primitive |
|---|---|---|
| **Ordering** | "A must reach stable storage before B" | `REQ_PREFLUSH` |
| **Durability** | "A must be on stable storage *now*, before I return" | `REQ_FUA`, or flush + wait |

A journal needs *ordering* (journal before commit, commit before checkpoint). `fsync` needs *durability* (the data is safe before the syscall returns). They are related but distinct, and the mechanisms differ.

Linux exposes exactly two flags:

```c
#define REQ_PREFLUSH  (1ULL << __REQ_PREFLUSH)  /* flush the cache BEFORE this */
#define REQ_FUA       (1ULL << __REQ_FUA)       /* THIS write goes to media */
```

| Flag | Semantics |
|---|---|
| `REQ_PREFLUSH` | all writes completed before this one are on stable media before this one starts |
| `REQ_FUA` | this write is on stable media before it is reported complete |
| both | a full barrier plus a durable write |

Note what is **absent**: there is no way to say "order these two writes relative to each other without flushing everything." The old `WRITE_BARRIER` did try to provide that and was removed in 2010 because it was unimplementable efficiently — it required draining the queue, which serialised everything.

So Linux's model is: **the filesystem issues writes, waits for their completions, then issues a flush.** Ordering is achieved by *waiting*, not by a barrier. The block layer's `blk-flush` machinery (`block/blk-flush.c`) handles the flush itself, including merging concurrent flush requests from different callers into one device flush — an important optimisation, since a flush is expensive and many callers often want one simultaneously.

If a device has no volatile cache (or reports none), `REQ_PREFLUSH` and `REQ_FUA` are no-ops. If it has a cache but no FUA support, FUA is emulated as write + flush.

### T.4 What `fsync` guarantees, precisely

POSIX says `fsync(fd)` transfers "all modified in-core data" for the file to the storage device. What that actually means in Linux:

**It DOES guarantee** (on a correctly functioning device and filesystem):
- The file's data blocks are on stable media
- The file's metadata needed to *read that data back* is on stable media (size, block pointers)
- A device cache flush was issued and completed

**It does NOT guarantee:**

| Trap | Reality |
|---|---|
| **The file's directory entry is durable** | `fsync` the *directory* too, after creating a file |
| **Other files are unaffected** | it says nothing about them |
| **Ordering between files** | `fsync(a); fsync(b)` does not mean a hit disk before b from a crash-observer's perspective... actually it does, but `write(a); write(b); fsync(b)` says nothing about a |
| **An error will be reported** | see §T.5 |
| **`O_DIRECT` implies durability** | it does not (Ch. 55 §T.6) |
| **The device is telling the truth** | see §T.6 |

The directory trap is the most common real bug:

```c
	/* WRONG: the file may not exist after a crash */
	fd = open("data.new", O_WRONLY|O_CREAT|O_TRUNC, 0644);
	write(fd, buf, len);
	fsync(fd);
	close(fd);
	rename("data.new", "data");

	/* RIGHT */
	fd = open("data.new", O_WRONLY|O_CREAT|O_TRUNC, 0644);
	write(fd, buf, len);
	fsync(fd);                       /* the DATA is durable */
	close(fd);
	rename("data.new", "data");
	dirfd = open(".", O_RDONLY|O_DIRECTORY);
	fsync(dirfd);                    /* the RENAME is durable */
	close(dirfd);
```

Without the directory `fsync`, a crash can leave the rename un-committed — the data is safe but unreachable, or the old file is still there.

**`fdatasync` versus `fsync`:** `fdatasync` omits metadata not needed to read the data back. Specifically it skips `mtime`/`atime` updates but *not* a size change. So for overwriting an existing region of a file, `fdatasync` avoids a metadata journal transaction entirely and is substantially faster. For appending, it must still update the size, so the gain is smaller. Databases use `fdatasync` for exactly this reason.

**`sync_file_range`** is often suggested as a faster alternative. **It is not a durability primitive.** It initiates or waits for writeback but issues **no cache flush**, so the data may be in the device's volatile cache. It is useful for pacing writeback, and using it as a substitute for `fsync` is a data-loss bug. The man page now says so in unusually blunt terms.

### T.5 fsyncgate: error reporting

In 2018, PostgreSQL discovered that their 20-year-old durability strategy was broken, and the discovery reshaped kernel error reporting.

The pattern:

```c
	write(fd, buf, len);          /* into the page cache */
	...
	if (fsync(fd) != 0)           /* error: retry */
		fsync(fd);            /* <- returns 0. The data is GONE. */
```

What happened:

1. Writeback of a dirty page failed (I/O error).
2. The kernel marked the error on the `address_space`, **cleared the page's dirty bit, and discarded the page** — because keeping infinitely many failed dirty pages would exhaust memory.
3. `fsync` returned the error — **once**. The error flag was cleared on read.
4. A second `fsync` returned success, because there were no dirty pages left.
5. PostgreSQL concluded the data was durable. It was not. It was gone.

Worse: which process got the error was arbitrary. If a *different* process (a monitoring script running `sync`) consumed the error first, PostgreSQL's `fsync` returned 0 having never seen it.

The fixes (4.13–4.16):

```c
struct address_space {
	...
	errseq_t	wb_err;      /* an error sequence counter */
};

struct file {
	...
	errseq_t	f_wb_err;    /* what THIS fd has already seen */
};
```

`errseq_t` is a clever little type: a 32-bit value packing an error code with a counter and a "seen" flag. Each `struct file` remembers the sequence value it last observed. `filemap_check_wb_err()` reports an error if the mapping's counter has advanced past what this fd has seen. So:

- **Every fd open at the time of the error sees it**, exactly once each.
- An fd opened *after* the error does not see it (correctly — it was never told to write that data).
- `sync` from an unrelated process no longer steals the error.

Three lessons worth internalising:

1. **`fsync` returning an error means your data may be gone, not "retry."** There is nothing to retry; the page has been discarded.
2. The only safe response to an `fsync` error is to **re-write the data from application memory**, or to treat the database as corrupt and recover from WAL/backup.
3. **Error reporting is part of a durability interface's specification**, not an implementation detail. An interface that can report an error only once, to an arbitrary caller, is broken even if every individual operation is correct.

PostgreSQL's response was to `panic()` on `fsync` failure rather than retry. That is the correct response, and it looks drastic only if you have not understood the above.

### T.6 Does the device tell the truth?

Every guarantee above rests on one assumption: **when the device reports a flush complete, the data really is on non-volatile media.**

Devices that lie, and why:

| Device | The lie | Why |
|---|---|---|
| Cheap USB sticks | ignore FLUSH entirely | benchmarks look better |
| Some consumer SSDs | ack FLUSH before the NAND write completes | same |
| USB-SATA bridges | do not translate FLUSH/FUA | protocol translation gaps |
| Virtual disks with `cache=unsafe` | explicitly discard flushes | speed for throwaway VMs |
| RAID controllers with a dead BBU | cache is volatile but reports write-back safe | battery failure not detected |
| Hardware RAID, misconfigured | data in a volatile controller cache | default is often wrong |

A **power-loss-protected** enterprise SSD legitimately acks flushes immediately, because it has capacitors holding enough charge to write its cache to NAND on power loss. This is a real and correct optimisation, and it is why enterprise SSDs have vastly better `fsync` latency than consumer ones — not because they are faster, but because they do not need to wait.

Check:

```sh
cat /sys/block/sda/queue/write_cache        # "write back" or "write through"
hdparm -W /dev/sda
smartctl -a /dev/sda | grep -i 'power.loss\|capacitor'
nvme id-ctrl /dev/nvme0 | grep -i vwc       # Volatile Write Cache
```

The only way to *know* is to test with actual power loss (Lab 61.7), or to use `diskchecker.pl` against a second machine.

Related: the `nobarrier`/`nobarriers` mount option disables flushes. It is a **data-destroying option** unless the storage is genuinely power-loss-protected. It exists because of RAID controllers with BBUs, and it has been removed or deprecated in ext4 and XFS precisely because it was so widely misused. If you find it in a configuration, treat it as a bug until proven otherwise.

### T.7 JBD2: the machinery

JBD2 (Journaling Block Device, version 2) is ext4's journal, and a standalone layer — OCFS2 also uses it.

**Objects:**

```c
journal_t     /* the journal: log device, head/tail, transaction lists */
transaction_t /* a group of atomic updates, in one of several states */
handle_t      /* ONE filesystem operation's participation in a transaction */
journal_head  /* per-buffer journalling state */
```

**The key idea: many filesystem operations share one transaction.** Each operation gets a `handle_t`; the transaction commits when it has accumulated enough or a timeout elapses (5 s by default). So 1000 file creations become one journal write, not 1000. This is the same batching insight as XFS's CIL (Ch. 58 §T.6), though less aggressive — XFS coalesces repeated changes to the same object; JBD2 batches whole transactions.

**Transaction states:**

```
T_RUNNING    -> accepting new handles
T_LOCKED     -> no new handles; waiting for existing ones to finish
T_FLUSH      -> writing the journal blocks
T_COMMIT     -> writing the commit record
T_COMMIT_DFLUSH / T_COMMIT_JFLUSH -> the flushes
T_FINISHED   -> committed; may be checkpointed
```

**The filesystem's protocol:**

```c
	handle_t *handle;
	int err;

	/* 1. Reserve credits -- how many blocks might we dirty? */
	handle = ext4_journal_start(inode, EXT4_HT_WRITE_PAGE,
				    ext4_writepage_trans_blocks(inode));
	if (IS_ERR(handle))
		return PTR_ERR(handle);

	/* 2. Declare intent BEFORE modifying -- this is write-ahead */
	err = ext4_journal_get_write_access(handle, bh);
	if (err)
		goto out;

	/* 3. Modify */
	modify_the_buffer(bh);

	/* 4. Hand it to the journal */
	err = ext4_handle_dirty_metadata(handle, inode, bh);

out:
	/* 5. Done. The transaction commits later, with others. */
	ext4_journal_stop(handle);
```

Step 1 is the part people get wrong. **Credits must be reserved up front** because a transaction cannot fail partway — there must be guaranteed journal space for everything it will dirty. Under-reserving causes `JBD2: Journal transaction ... aborted` or, worse, silent corruption. Over-reserving wastes journal space and forces early commits. `ext4_writepage_trans_blocks()` and friends encode the worst-case block counts for each operation type, and they are genuinely hard to get right.

Step 2 before step 3 is write-ahead logging made literal: the journal captures the *old* content (for the undo/escape logic) and registers the buffer before it changes.

**Revoke records** solve a subtle and important problem. Suppose:

1. Block 500 is a directory block. It is journalled.
2. The directory is deleted; block 500 becomes a free data block.
3. A file writes user data to block 500.
4. Crash.
5. Recovery replays the journal and **writes the old directory content over the user's data.**

The revoke record says "do not replay block 500 past this point." Every block that transitions from metadata to data must be revoked. Forgetting a revoke is a corruption bug, and it is the kind that only appears under crash testing.

**Escaping** solves another: if a data block happens to begin with the JBD2 magic number (`0xc03b3998`), recovery would misinterpret it as a descriptor. JBD2 zeroes the first four bytes and sets an escape flag in the tag. A nice example of an in-band signalling problem with the standard solution.

**`data=ordered` implementation:** data pages are written and waited for *before* the metadata transaction commits. This is done by the `t_inode_list` — inodes whose data must be flushed at commit time. That flush-and-wait is why `data=ordered` costs what it does, and why `data=writeback` is faster and less safe (Ch. 57 §T.6).

### T.8 The checksum problem

If a journal write is torn (partially written) across a power loss, recovery replaying it writes garbage over good metadata. **A corrupt journal is worse than no journal.**

`journal_checksum` and `journal_async_commit` address this:

- Each descriptor block and each commit block carries a CRC32c.
- The commit block's checksum covers the whole transaction.
- Recovery validates before replaying; a transaction with a bad checksum is discarded, along with everything after it.

This also enables an optimisation: **with checksums, the flush between the journal blocks and the commit record can be skipped**, because a commit record whose checksum does not match the journal contents is detectably invalid. `journal_async_commit` does this and measurably improves `fsync` latency — one fewer flush per commit.

Note the structure of the argument: *a checksum lets you replace an ordering constraint with a validity check.* That is a general and powerful technique, and it appears again in ZFS (the uberblock array) and in btrfs (superblock generations).

### T.9 Application-level crash consistency

Pillai et al. (OSDI 2014) tested real applications — SQLite, PostgreSQL, LevelDB, Git, Mercurial, HDFS, ZooKeeper, VMware — against crash simulation. **They found vulnerabilities in every single one.**

The recurring mistakes:

| Mistake | Consequence |
|---|---|
| Assuming `rename` is atomic *and* ordered with respect to the data | zero-length files (Ch. 57 §T.6) |
| Not `fsync`-ing the parent directory | the file does not exist after a crash |
| Assuming ordered writes to a single file | a filesystem may reorder within a file |
| Assuming `fsync(a)` orders `a` before unrelated `b` | it does not |
| Treating `fsync` failure as retryable | §T.5 |
| Assuming a torn write cannot happen mid-block | it can, beyond 512 bytes |
| Assuming `O_DIRECT` is durable | it is not |
| Assuming appends are atomic | they are not, beyond a sector |

The safe patterns, in full:

**Atomic file replacement:**

```c
	int fd = open(tmp, O_WRONLY|O_CREAT|O_TRUNC, 0644);
	write_all(fd, data, len);
	fsync(fd);                  /* data durable */
	close(fd);
	rename(tmp, final);         /* atomic namespace change */
	int dfd = open(dir, O_RDONLY|O_DIRECTORY);
	fsync(dfd);                 /* rename durable */
	close(dfd);
```

**Durable append (write-ahead log):**

```c
	/* Preallocate to avoid metadata updates on every append */
	fallocate(fd, 0, 0, LOG_SIZE);
	...
	pwrite(fd, record, reclen, offset);
	fdatasync(fd);              /* no size change -> no metadata txn */
```

**Checksummed records** — because torn writes happen:

```c
	struct record {
		uint32_t magic;
		uint32_t len;
		uint32_t crc;      /* over the payload */
		char payload[];
	};
	/* On recovery: scan forward; stop at the first record whose crc
	 * does not match. That is where the crash happened. */
```

The last one is important and under-used: **if you checksum your records, you do not need the filesystem to guarantee much at all.** You need only that a completed `fsync` means the bytes are durable, and you detect everything else yourself. This is the end-to-end argument (Ch. 00 §T.4) applied to durability, and it is why SQLite and PostgreSQL WAL formats are checksummed.

### T.10 Testing crash consistency

Hoping is not a strategy. Three tools, in increasing order of rigour:

**(a) `dm-flakey`** — a device-mapper target that drops or corrupts writes after a configurable interval. Crude but catches gross errors.

**(b) `dm-log-writes`** ★★★ — records **every write, with its flags**, to a log device. You can then replay to any point — particularly to each flush/FUA boundary — and check the filesystem at each. This lets you enumerate *every crash point the device could have exposed* rather than sampling randomly.

```sh
dmsetup create logwrites --table "0 $SZ log-writes $DATA_DEV $LOG_DEV"
# ... run the workload ...
dmsetup message logwrites 0 mark mypoint
# Replay:
src/log-writes/replay-log --log $LOG_DEV --replay $DATA_DEV --end-mark mypoint
fsck / mount / check invariants
```

**(c) `CrashMonkey`/`ALICE`** — systematic exploration of reorderings. ALICE (from the Pillai paper) models what each filesystem may legally reorder and enumerates the resulting states.

And the ground truth: **actually cutting power.** A relay, a script, and a thousand cycles will find things no simulation does — particularly device-level lying (§T.6).

`xfstests` includes a substantial crash-consistency group (`generic/45[0-9]`, `generic/47[0-9]`, and the `dm-log-writes`-based tests). Running them against your filesystem or your application's storage layer is the practical minimum.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `fs/jbd2/journal.c` ★★★ | journal lifecycle, `jbd2_journal_start`, superblock |
| `fs/jbd2/transaction.c` ★★★ | `start_this_handle`, `jbd2_journal_get_write_access`, `jbd2_journal_stop` |
| `fs/jbd2/commit.c` ★★★ | `jbd2_journal_commit_transaction` — §T.2's protocol |
| `fs/jbd2/recovery.c` ★★★ | the three-pass recovery |
| `fs/jbd2/revoke.c` | §T.7's revoke records |
| `fs/jbd2/checkpoint.c` | writing committed blocks to their real locations |
| `include/linux/jbd2.h` ★★★ | every structure |
| `fs/ext4/ext4_jbd2.c` | ext4's wrapper and credit calculations |
| `block/blk-flush.c` ★★★ | §T.3's flush machinery, including merging |
| `fs/sync.c` | `sync`, `syncfs`, `fsync`, `sync_file_range` |
| `mm/filemap.c` | `filemap_write_and_wait_range`, `errseq` handling |
| `lib/errseq.c` ★★★ | §T.5's error tracking — 200 lines, read it all |
| `fs/xfs/xfs_log.c`, `xfs_log_cil.c` | XFS's very different approach (Ch. 58 §T.6) |
| `Documentation/filesystems/journalling.rst` | the JBD2 API |

### 1.2 The on-disk journal format

```c
#define JBD2_MAGIC_NUMBER 0xc03b3998U

typedef struct journal_header_s {
	__be32		h_magic;
	__be32		h_blocktype;
	__be32		h_sequence;
} journal_header_t;

#define JBD2_DESCRIPTOR_BLOCK	1
#define JBD2_COMMIT_BLOCK	2
#define JBD2_SUPERBLOCK_V1	3
#define JBD2_SUPERBLOCK_V2	4
#define JBD2_REVOKE_BLOCK	5

typedef struct journal_block_tag_s {
	__be32		t_blocknr;	/* where this block really lives */
	__be16		t_checksum;
	__be16		t_flags;
	__be32		t_blocknr_high;
} journal_block_tag_t;

#define JBD2_FLAG_ESCAPE	1	/* §T.7: the data began with the magic */
#define JBD2_FLAG_SAME_UUID	2
#define JBD2_FLAG_DELETED	4
#define JBD2_FLAG_LAST_TAG	8
```

The layout of one transaction on disk:

```
[descriptor block: tags naming the real locations of the following blocks]
[data block 1]
[data block 2]
...
[revoke block(s)]
[commit block: sequence, checksum, timestamp]
```

Note that the descriptor precedes its data — so recovery reads the descriptor, knows how many blocks follow and where each belongs, and can validate before writing anything.

### 1.3 Starting a handle

```c
static int start_this_handle(journal_t *journal, handle_t *handle, gfp_t gfp_mask)
{
	transaction_t	*transaction, *new_transaction = NULL;
	int		blocks = handle->h_total_credits;
	int		rsv_blocks = 0;
	...
repeat:
	read_lock(&journal->j_state_lock);
	BUG_ON(journal->j_flags & JBD2_UNMOUNT);
	if (is_journal_aborted(journal) ||
	    (journal->j_errno != 0 && !(journal->j_flags & JBD2_ACK_ERR))) {
		read_unlock(&journal->j_state_lock);
		jbd2_journal_free_transaction(new_transaction);
		return -EROFS;
	}

	/* The transaction may be locked (committing). Wait for the next. */
	if (journal->j_barrier_count) {
		read_unlock(&journal->j_state_lock);
		wait_event(journal->j_wait_transaction_locked,
			   journal->j_barrier_count == 0);
		goto repeat;
	}

	if (!journal->j_running_transaction) {
		...
		goto alloc_transaction;
	}

	transaction = journal->j_running_transaction;

	if (!handle->h_reserved) {
		/* Is there room in this transaction for our credits? */
		if (atomic_read(&transaction->t_outstanding_credits) + blocks >
		    journal->j_max_transaction_buffers) {
			/* No: wait for it to commit, then join the next. */
			...
			wait_transaction_locked(journal);
			goto repeat;
		}
	}
	...
	atomic_add(blocks, &transaction->t_outstanding_credits);
	handle->h_transaction = transaction;
	...
}
```

The credit accounting is visible here: a handle joins the running transaction only if its worst-case block count fits. **A transaction never runs out of journal space mid-way**, because every participant reserved up front. That invariant is what makes the whole scheme safe.

### 1.4 The commit: §T.2, in code

```c
void jbd2_journal_commit_transaction(journal_t *journal)
{
	...
	/* PHASE 1: close the transaction to new handles */
	write_lock(&journal->j_state_lock);
	commit_transaction->t_state = T_LOCKED;
	...
	/* Wait for outstanding handles to finish */
	while (atomic_read(&commit_transaction->t_updates)) {
		...
		schedule();
	}

	/* PHASE 2: data=ordered -- flush DATA before committing METADATA */
	err = journal_submit_data_buffers(journal, commit_transaction);
	...
	err = journal_finish_inode_data_buffers(journal, commit_transaction);

	commit_transaction->t_state = T_COMMIT;
	...
	/* PHASE 3: write the descriptor blocks and the metadata */
	while (commit_transaction->t_buffers) {
		...
		descriptor = jbd2_journal_get_descriptor_buffer(...);
		...
		jbd2_file_log_bh(&io_bufs, wbuf[bufs]);
		...
		submit_bh(REQ_OP_WRITE | JBD2_JOURNAL_REQ_FLAGS, bh);
	}

	/* PHASE 4: WAIT for all journal writes to complete */
	while (!list_empty(&io_bufs)) {
		...
		wait_on_buffer(bh);
		...
	}

	/* PHASE 5: the commit record -- with PREFLUSH and FUA if needed */
	if (commit_transaction->t_need_data_flush &&
	    (journal->j_fs_dev != journal->j_dev) &&
	    (journal->j_flags & JBD2_BARRIER))
		blkdev_issue_flush(journal->j_fs_dev);

	if (!jbd2_has_feature_async_commit(journal))
		write_flags |= REQ_PREFLUSH | REQ_FUA;    /* T.3 */

	err = journal_submit_commit_record(journal, commit_transaction, &cbh, crc32_sum);

	/* PHASE 6: the transaction is COMMITTED. Checkpointing may proceed. */
	commit_transaction->t_state = T_FINISHED;
	...
}
```

The `if (!jbd2_has_feature_async_commit(...))` is §T.8's optimisation: **with checksums, the PREFLUSH+FUA on the commit record can be dropped**, because an inconsistent commit record is detectable.

### 1.5 Recovery

```c
int jbd2_journal_recover(journal_t *journal)
{
	int err, err2;
	journal_superblock_t *sb = journal->j_superblock;
	struct recovery_info info;

	memset(&info, 0, sizeof(info));

	if (!sb->s_start) {
		jbd2_debug(1, "No recovery required, last transaction %d\n",
			   be32_to_cpu(sb->s_sequence));
		journal->j_transaction_sequence = be32_to_cpu(sb->s_sequence) + 1;
		return 0;
	}

	err  = do_one_pass(journal, &info, PASS_SCAN);    /* find the end */
	if (!err)
		err = do_one_pass(journal, &info, PASS_REVOKE);  /* collect revokes */
	if (!err)
		err = do_one_pass(journal, &info, PASS_REPLAY);  /* write the blocks */

	jbd2_debug(1, "JBD2: recovery, exit status %d, "
		   "recovered transactions %u to %u\n", err,
		   info.start_transaction, info.end_transaction);
	...
	jbd2_journal_clear_revoke(journal);
	err2 = sync_blockdev(journal->j_fs_dev);
	...
}
```

**Three passes, and the order is forced:**

1. **SCAN** — find the last valid commit record. You cannot replay before knowing where the journal ends.
2. **REVOKE** — collect *all* revoke records from the whole journal. A block revoked in transaction 50 must not be replayed from transaction 10, so revokes must be known before any replay.
3. **REPLAY** — write the blocks, skipping revoked ones.

The revoke pass existing separately is the clearest evidence of why revokes are subtle: you cannot handle them inline.

### 1.6 `errseq`: §T.5's fix

```c
/* lib/errseq.c -- 32 bits: [error code][counter][SEEN bit] */
#define ERRSEQ_SHIFT		ilog2(MAX_ERRNO + 1)
#define ERRSEQ_SEEN		(1 << (ERRSEQ_SHIFT - 1))

errseq_t errseq_set(errseq_t *eseq, int err)
{
	errseq_t cur, old;

	old = READ_ONCE(*eseq);
	for (;;) {
		errseq_t new;

		cur = old;
		new = (cur & ~(MAX_ERRNO | ERRSEQ_SEEN)) | -err;

		/* Only bump the counter if the old value was SEEN. */
		if (cur & ERRSEQ_SEEN)
			new += ERRSEQ_CTR_INC;
		...
		old = cmpxchg(eseq, cur, new);
		if (likely(old == cur) || old == new)
			break;
	}
	return new;
}

int errseq_check_and_advance(errseq_t *eseq, errseq_t *since)
{
	errseq_t old, new;
	int err = 0;

	if (likely(*eseq == *since))
		return 0;                    /* the fast path: nothing new */

	old = READ_ONCE(*eseq);
	if (old != *since) {
		new = old | ERRSEQ_SEEN;
		if (old != new)
			cmpxchg(eseq, old, new);
		*since = new;                /* remember what WE have seen */
		err = -(old & MAX_ERRNO);
	}
	return err;
}
```

Each `struct file` carries `f_wb_err`. An error set after the file was opened will be reported to that file exactly once, regardless of how many other processes also see it. 200 lines, and it fixed a 20-year-old class of silent data loss.

### 1.7 Observability

| Where | What |
|---|---|
| `dumpe2fs -h DEV \| grep -i journal` | journal size, features, state |
| `debugfs -R "logdump -O -S" DEV` ★★★ | **dump the journal contents** |
| `debugfs -R "logdump -a" DEV` | with block tags |
| `/proc/fs/jbd2/DEV/info` ★★★ | transaction statistics, average commit time |
| `trace-cmd record -e jbd2:\*` ★★★ | every transaction start/commit/checkpoint |
| `trace-cmd record -e block:block_rq_issue` | look for `F`/`FUA` in `rwbs` |
| `/sys/block/DEV/queue/write_cache` | `write back` or `write through` |
| `/sys/fs/ext4/DEV/journal_task_ioprio` | journal thread priority |
| `xfs_logprint` | XFS's equivalent |

`/proc/fs/jbd2/DEV/info` is unusually useful:

```
27 transactions (25 requested), each up to 8192 blocks
average:
  0ms waiting for transaction
  0ms request delay
  4527ms running transaction
  0ms transaction was being locked
  0ms flushing data (in ordered mode)
  1ms logging transaction
  1232us average transaction commit time
  29 handles per transaction
  2 blocks per transaction
  3 logged blocks per transaction
```

Every line is diagnostic. High "waiting for transaction" means the journal is too small; high "flushing data" means `data=ordered` is the bottleneck.

---

## 2. Practice

### Lab 61.1 — Watch a journal transaction

```sh
sudo modprobe scsi_debug dev_size_mb=2048
DEV=$(lsblk -ndo NAME,MODEL | awk '/scsi_debug/{print "/dev/"$1}')
sudo mkfs.ext4 -F $DEV && sudo mkdir -p /mnt/j && sudo mount $DEV /mnt/j

sudo dumpe2fs -h $DEV 2>/dev/null | grep -i journal
sudo cat /proc/fs/jbd2/$(basename $DEV)-8/info 2>/dev/null || \
  sudo cat /proc/fs/jbd2/*/info
```

Trace one file creation end to end:

```sh
sudo trace-cmd record -e jbd2:\* -e block:block_rq_issue -- \
  sudo sh -c 'echo hello > /mnt/j/file; sync'
sudo trace-cmd report | head -40
```

```
jbd2_handle_start:   dev 8,16 tid 3 type 1 line_no 4913 requested_blocks 10
jbd2_handle_stats:   dev 8,16 tid 3 ... dirtied_blocks 3
jbd2_start_commit:   dev 8,16 transaction 3 sync 0
jbd2_commit_locking: dev 8,16 transaction 3
jbd2_commit_flushing:dev 8,16 transaction 3
jbd2_commit_logging: dev 8,16 transaction 3
block_rq_issue:      8,16 WS 4096 () 68624 + 8      <- journal blocks
block_rq_issue:      8,16 FWFS 4096 () 68632 + 8    <- COMMIT: F = flush, FUA
jbd2_end_commit:     dev 8,16 transaction 3 head 2
jbd2_checkpoint:     dev 8,16 result 0
```

**`FWFS` is the commit record**: `F`=PREFLUSH, `W`=write, `FS`=FUA+sync. §T.3's two primitives, visible.

Dump the journal:

```sh
sudo umount /mnt/j
sudo debugfs -R "logdump -O -S" $DEV 2>/dev/null | head -60
```

```
Journal starts at block 1, transaction 3
Found expected sequence 3, type 1 (descriptor block) at block 1
Dumping descriptor block, sequence 3, at block 1:
  FS block 129 logged at journal block 2 (flags 0x2)
  FS block 1 logged at journal block 3 (flags 0x2)
  FS block 641 logged at journal block 4 (flags 0xa)
Found expected sequence 3, type 2 (commit block) at block 5
```

Descriptor → data blocks → commit. §1.2's layout, on your disk.

Batching:

```sh
sudo mount $DEV /mnt/j
sudo trace-cmd record -e jbd2:jbd2_handle_start -e jbd2:jbd2_start_commit -- \
  sudo sh -c 'for i in $(seq 1 1000); do : > /mnt/j/f$i; done; sync'
echo -n "handles: "; sudo trace-cmd report | grep -c handle_start
echo -n "commits: "; sudo trace-cmd report | grep -c start_commit
```

Roughly 1000 handles, a handful of commits. **That is §T.7's batching**, and it is why ext4 can create thousands of files per second.

---

### Lab 61.2 — Journal modes, measured

```sh
for mode in ordered writeback journal; do
  sudo umount /mnt/j 2>/dev/null
  sudo mkfs.ext4 -qF $DEV
  if [ $mode = journal ]; then
    sudo tune2fs -o journal_data $DEV > /dev/null
  fi
  sudo mount -o data=$mode $DEV /mnt/j 2>/dev/null || sudo mount $DEV /mnt/j
  echo "=== data=$mode ==="
  mount | grep /mnt/j

  D=$(basename $DEV)
  B=$(awk "/$D /{print \$10}" /proc/diskstats)
  sudo dd if=/dev/zero of=/mnt/j/x bs=1M count=200 conv=fsync 2>&1 | tail -1
  A=$(awk "/$D /{print \$10}" /proc/diskstats)
  echo "  device sectors written: $((A-B)) (data was $((200*2048)))"
  echo "  write amplification: $(echo "scale=2; ($A-$B)/(200*2048)" | bc)"
done
```

`data=journal` writes ~2×. That is §T.2(d)'s cost, measured.

Now the safety difference — `data=writeback` exposing stale data:

```sh
sudo umount /mnt/j
sudo mkfs.ext4 -qF $DEV && sudo mount $DEV /mnt/j

# Write a "secret" then delete it, leaving the blocks free-but-dirty
sudo sh -c 'python3 -c "print(\"SECRET\"*10000)" > /mnt/j/secret'
sync
sudo rm /mnt/j/secret

sudo umount /mnt/j && sudo mount -o data=writeback $DEV /mnt/j
# Now extend a new file into those blocks and crash before the data is written
sudo sh -c 'dd if=/dev/zero of=/mnt/j/newfile bs=1M count=20 2>/dev/null'
# (crash here, via dm-flakey, then examine newfile's contents)
```

Lab 61.6 does this properly with `dm-log-writes`.

`fsync` latency by mode:

```sh
cat > fsyncbench.c <<'EOF'
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>

static double now(void) {
	struct timespec ts; clock_gettime(CLOCK_MONOTONIC, &ts);
	return ts.tv_sec + ts.tv_nsec / 1e9;
}

int main(int argc, char **argv) {
	int fd, i, n = 2000, mode = argc > 2 ? atoi(argv[2]) : 0;
	char buf[4096] = {0};
	double t0, *lat;

	fd = open(argv[1], O_WRONLY|O_CREAT|O_TRUNC, 0644);
	if (fd < 0) { perror("open"); return 1; }
	fallocate(fd, 0, 0, (off_t)n * 4096);
	lat = malloc(n * sizeof(double));

	for (i = 0; i < n; i++) {
		t0 = now();
		pwrite(fd, buf, 4096, (off_t)i * 4096);
		switch (mode) {
		case 0: fsync(fd); break;
		case 1: fdatasync(fd); break;
		case 2: sync_file_range(fd, (off_t)i*4096, 4096,
					SYNC_FILE_RANGE_WRITE|SYNC_FILE_RANGE_WAIT_AFTER);
			break;   /* NOT DURABLE -- T.4 */
		case 3: break;   /* nothing */
		}
		lat[i] = (now() - t0) * 1e6;
	}
	/* p50 / p99 */
	{
		int cmp(const void *a, const void *b) {
			double x = *(const double*)a, y = *(const double*)b;
			return x < y ? -1 : x > y;
		}
		qsort(lat, n, sizeof(double), cmp);
		printf("  p50=%.0fus p99=%.0fus max=%.0fus\n",
		       lat[n/2], lat[n*99/100], lat[n-1]);
	}
	close(fd);
	return 0;
}
EOF
gcc -O2 -o fsyncbench fsyncbench.c

for m in 0 1 2 3; do
  case $m in 0) N=fsync;; 1) N=fdatasync;; 2) N=sync_file_range;; 3) N=nothing;; esac
  echo -n "$N: "
  sudo ./fsyncbench /mnt/j/bench $m
done
```

`sync_file_range` looks fast because **it is not durable**. That is the trap.

---

### Lab 61.3 — Prove the flush is what costs

```sh
# With the device cache enabled
echo "write back" | sudo tee /sys/block/$(basename $DEV)/queue/write_cache
sudo umount /mnt/j; sudo mount $DEV /mnt/j
echo -n "write_cache=write back: "; sudo ./fsyncbench /mnt/j/b 0

# Claim there is no cache: flushes become no-ops (UNSAFE on real hardware)
echo "write through" | sudo tee /sys/block/$(basename $DEV)/queue/write_cache
sudo umount /mnt/j; sudo mount $DEV /mnt/j
echo -n "write_cache=write through: "; sudo ./fsyncbench /mnt/j/b 0
```

Count the flushes:

```sh
echo "write back" | sudo tee /sys/block/$(basename $DEV)/queue/write_cache
sudo bpftrace -e '
tracepoint:block:block_rq_issue {
	$rw = str(args->rwbs);
	if (strcontains($rw, "F")) { @flush = count(); }
	@all = count();
}
interval:s:5 { printf("flushes=%d total=%d\n", @flush, @all);
               clear(@flush); clear(@all); }' &

sudo ./fsyncbench /mnt/j/b 0
```

Flush merging (§T.3):

```sh
# Many processes fsync-ing simultaneously
sudo bpftrace -e '
kprobe:blkdev_issue_flush { @issue_flush = count(); }
kprobe:blk_insert_flush   { @insert = count(); }
tracepoint:block:block_rq_issue /strcontains(str(args->rwbs), "F")/ {
	@device_flushes = count();
}
interval:s:3 { print(@insert); print(@device_flushes);
               clear(@insert); clear(@device_flushes); }' &

for i in 1 2 3 4 5 6 7 8; do
  ( sudo ./fsyncbench /mnt/j/p$i 0 > /dev/null ) &
done
wait
```

`@device_flushes` should be substantially less than `@insert`: `blk-flush` merged concurrent requests into shared device flushes.

`journal_async_commit` (§T.8):

```sh
for feat in "" "journal_async_commit"; do
  sudo umount /mnt/j 2>/dev/null
  sudo mkfs.ext4 -qF -O metadata_csum $DEV
  [ -n "$feat" ] && sudo tune2fs -O $feat $DEV > /dev/null 2>&1
  sudo mount $DEV /mnt/j
  echo -n "${feat:-default}: "
  sudo ./fsyncbench /mnt/j/b 0
  # Count flushes per commit:
  sudo trace-cmd record -e block:block_rq_issue -o /tmp/f.dat -- \
    sudo sh -c 'for i in $(seq 1 100); do echo x > /mnt/j/a$i; sync; done' 2>/dev/null
  echo -n "  flushes: "
  sudo trace-cmd report -i /tmp/f.dat 2>/dev/null | grep -c 'F.*4096\|FWFS\|FUA'
done
```

One fewer flush per commit, because a checksum replaced an ordering constraint.

---

### Lab 61.4 — The four `fsync` traps

```c
// SPDX-License-Identifier: GPL-2.0
/* traps.c -- each function demonstrates one durability mistake. */
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

/* TRAP 1: no directory fsync. The file may not exist after a crash. */
static void trap1_no_dir_fsync(const char *dir)
{
	char tmp[256], fin[256];
	int fd;

	snprintf(tmp, sizeof(tmp), "%s/t1.tmp", dir);
	snprintf(fin, sizeof(fin), "%s/t1", dir);
	fd = open(tmp, O_WRONLY|O_CREAT|O_TRUNC, 0644);
	write(fd, "DATA", 4);
	fsync(fd);                    /* data durable... */
	close(fd);
	rename(tmp, fin);             /* ...but the rename is NOT */
}

static void correct1(const char *dir)
{
	char tmp[256], fin[256];
	int fd, dfd;

	snprintf(tmp, sizeof(tmp), "%s/c1.tmp", dir);
	snprintf(fin, sizeof(fin), "%s/c1", dir);
	fd = open(tmp, O_WRONLY|O_CREAT|O_TRUNC, 0644);
	write(fd, "DATA", 4);
	fsync(fd);
	close(fd);
	rename(tmp, fin);
	dfd = open(dir, O_RDONLY|O_DIRECTORY);
	fsync(dfd);                   /* THE FIX */
	close(dfd);
}

/* TRAP 2: fsync one file, assume another is ordered. It is not. */
static void trap2_cross_file(const char *dir)
{
	char a[256], b[256];
	int fa, fb;

	snprintf(a, sizeof(a), "%s/t2a", dir);
	snprintf(b, sizeof(b), "%s/t2b", dir);
	fa = open(a, O_WRONLY|O_CREAT|O_TRUNC, 0644);
	fb = open(b, O_WRONLY|O_CREAT|O_TRUNC, 0644);
	write(fa, "FIRST", 5);
	write(fb, "SECOND", 6);
	fsync(fb);                    /* says NOTHING about `a` */
	close(fa); close(fb);
}

/* TRAP 3: treating fsync failure as retryable. It is not (T.5). */
static void trap3_retry(int fd)
{
	if (fsync(fd) != 0) {
		perror("fsync");
		if (fsync(fd) == 0)
			printf("  second fsync 'succeeded' -- DATA MAY BE GONE\n");
	}
}

/* TRAP 4: O_DIRECT is not durable (Ch. 55 T.6). */
static void trap4_odirect(const char *dir)
{
	char p[256];
	void *buf;
	int fd;

	snprintf(p, sizeof(p), "%s/t4", dir);
	posix_memalign(&buf, 4096, 4096);
	memset(buf, 'D', 4096);
	fd = open(p, O_WRONLY|O_CREAT|O_TRUNC|O_DIRECT, 0644);
	pwrite(fd, buf, 4096, 0);     /* on the device, maybe in its cache */
	close(fd);                    /* NO fsync -- not durable */
}

int main(int argc, char **argv)
{
	const char *dir = argc > 1 ? argv[1] : "/mnt/j";

	trap1_no_dir_fsync(dir);
	correct1(dir);
	trap2_cross_file(dir);
	trap4_odirect(dir);
	printf("done; now crash and see what survived\n");
	return 0;
}
```

Observe the difference in what each emits:

```sh
gcc -O2 -o traps traps.c
sudo trace-cmd record -e block:block_rq_issue -e ext4:ext4_sync_file\* -- \
  sudo ./traps /mnt/j
sudo trace-cmd report | grep -E 'sync_file|F' | head -30
# correct1 produces an EXTRA sync_file for the directory inode.
```

Now demonstrate trap 3 with real I/O errors:

```sh
# dm-error: everything fails
sudo umount /mnt/j
SZ=$(sudo blockdev --getsz $DEV)
sudo dmsetup create flaky --table "0 $SZ linear $DEV 0"
sudo mkfs.ext4 -qF /dev/mapper/flaky
sudo mount /dev/mapper/flaky /mnt/j

cat > errtest.c <<'EOF'
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
int main(int argc, char **argv) {
	int fd = open(argv[1], O_WRONLY|O_CREAT|O_TRUNC, 0644);
	char buf[4096]; memset(buf, 'X', sizeof(buf));
	write(fd, buf, sizeof(buf));
	printf("first  fsync: %d (%s)\n", fsync(fd),
	       fsync(fd) ? strerror(errno) : "ok");
	printf("second fsync: %d\n", fsync(fd));
	printf("third  fsync: %d\n", fsync(fd));
	return 0;
}
EOF
gcc -o errtest errtest.c

sudo ./errtest /mnt/j/err &
sleep 0.2
# Make the device fail
sudo dmsetup suspend flaky
sudo dmsetup load flaky --table "0 $SZ error"
sudo dmsetup resume flaky
wait
```

The first `fsync` returns `-EIO`; subsequent ones return 0. **The data is gone, and the second call says everything is fine.** That is fsyncgate.

Now prove `errseq` works for multiple fds:

```c
	/* Two fds to the same file; BOTH should see the error exactly once. */
	int fd1 = open(path, O_WRONLY);
	int fd2 = open(path, O_WRONLY);
	write(fd1, buf, 4096);
	/* ... trigger the error ... */
	printf("fd1: %d\n", fsync(fd1));   /* -EIO */
	printf("fd2: %d\n", fsync(fd2));   /* -EIO, not 0 (post-4.13) */
```

---

### Lab 61.5 — Recovery, observed

```sh
sudo dmsetup remove flaky 2>/dev/null
sudo mkfs.ext4 -qF $DEV && sudo mount $DEV /mnt/j

# Build up a workload, then cut power mid-transaction
sudo sh -c 'for i in $(seq 1 500); do echo data > /mnt/j/f$i; done' &
WORKER=$!
sleep 0.3
# Simulate power loss: freeze the device with no unmount
sudo dmsetup create pl --table "0 $SZ linear $DEV 0" 2>/dev/null
kill -9 $WORKER 2>/dev/null
sudo umount -l /mnt/j 2>/dev/null

# Examine the journal BEFORE recovery
sudo dumpe2fs -h $DEV 2>/dev/null | grep -iE 'state|journal'
sudo debugfs -R "logdump -O -S" $DEV 2>/dev/null | tail -30

# Mount: recovery runs
sudo trace-cmd record -e jbd2:\* -e ext4:\* -- sudo mount $DEV /mnt/j
dmesg | tail -5
sudo trace-cmd report | grep -iE 'recover|replay' | head
ls /mnt/j | wc -l
```

Then the three-pass structure:

```sh
sudo bpftrace -e '
kprobe:do_one_pass { printf("recovery pass %d\n", arg2); }
kprobe:jbd2_journal_recover { printf("--- recovery start ---\n"); }
kretprobe:jbd2_journal_recover { printf("--- recovery done: %d ---\n", retval); }' &
sudo umount /mnt/j && sudo mount $DEV /mnt/j
```

Revoke records — make one:

```sh
sudo mkfs.ext4 -qF $DEV && sudo mount $DEV /mnt/j
sudo trace-cmd record -e jbd2:\* -- sudo sh -c '
  mkdir /mnt/j/d
  for i in $(seq 1 200); do : > /mnt/j/d/f$i; done
  sync
  rm -rf /mnt/j/d                 # directory blocks become free
  sync
  dd if=/dev/urandom of=/mnt/j/reuse bs=1M count=10 2>/dev/null  # reuse them
  sync'
sudo umount /mnt/j
sudo debugfs -R "logdump -O -S" $DEV 2>/dev/null | grep -i revoke | head
```

Journal size and its effect:

```sh
for jsz in 1024 4096 32768 131072; do
  sudo umount /mnt/j 2>/dev/null
  sudo mkfs.ext4 -qF -J size=$((jsz/1024)) $DEV 2>/dev/null || continue
  sudo mount $DEV /mnt/j
  echo -n "journal ${jsz}KB: "
  sudo /usr/bin/time -f '%e s' sh -c \
    'sudo sh -c "for i in \$(seq 1 20000); do : > /mnt/j/f\$i; done; sync"' 2>&1 | tail -1
  sudo cat /proc/fs/jbd2/*/info 2>/dev/null | grep -E 'waiting|commit time'
done
```

An external journal device:

```sh
sudo mke2fs -O journal_dev $DEV2
sudo mkfs.ext4 -qF -J device=$DEV2 $DEV
sudo mount $DEV /mnt/j
sudo ./fsyncbench /mnt/j/b 0
# Journal writes go to a separate device: no seek contention with data.
sudo trace-cmd record -e block:block_rq_issue -- sudo ./fsyncbench /mnt/j/b 0
sudo trace-cmd report | awk '{print $1}' | sort | uniq -c
# Two devices in the trace.
```

---

### Lab 61.6 — `dm-log-writes`: enumerate every crash point

This is the rigorous way to test crash consistency.

```sh
sudo modprobe -r scsi_debug
sudo modprobe scsi_debug dev_size_mb=2048 num_tgts=2
DEVS=($(lsblk -ndo NAME,MODEL | awk '/scsi_debug/{print "/dev/"$1}'))
DATA=${DEVS[0]}; LOG=${DEVS[1]}

SZ=$(sudo blockdev --getsz $DATA)
sudo dmsetup create lw --table "0 $SZ log-writes $DATA $LOG"

sudo mkfs.ext4 -qF /dev/mapper/lw
sudo mkdir -p /mnt/lw && sudo mount /dev/mapper/lw /mnt/lw
sudo dmsetup message lw 0 mark mkfs_done
```

Run the workload, marking points of interest:

```sh
sudo ./traps /mnt/lw
sudo dmsetup message lw 0 mark traps_done

sudo sh -c 'echo "v1" > /mnt/lw/config; sync'
sudo dmsetup message lw 0 mark v1

sudo sh -c 'echo "v2" > /mnt/lw/config.tmp; mv /mnt/lw/config.tmp /mnt/lw/config'
sudo dmsetup message lw 0 mark v2_no_fsync

sudo umount /mnt/lw
```

Now replay to each point and check:

```sh
# Get the xfstests replay tool
git clone https://git.kernel.org/pub/scm/fs/xfs/xfstests-dev.git
cd xfstests-dev && make -C src/log-writes
REPLAY=./src/log-writes/replay-log

# Count the flush/FUA points -- these are the crash points that matter
sudo $REPLAY --log $LOG --num-entries | head

for mark in mkfs_done traps_done v1 v2_no_fsync; do
  echo "=== replaying to $mark ==="
  sudo $REPLAY --log $LOG --replay $DATA --end-mark $mark
  sudo e2fsck -fn $DATA 2>&1 | tail -3
  sudo mount $DATA /mnt/lw 2>/dev/null && {
    echo -n "  files: "; ls /mnt/lw | tr '\n' ' '; echo
    echo -n "  config: "; sudo cat /mnt/lw/config 2>/dev/null || echo "(missing)"
    echo -n "  t1 (no dir fsync): "; sudo cat /mnt/lw/t1 2>/dev/null || echo "(MISSING)"
    echo -n "  c1 (with dir fsync): "; sudo cat /mnt/lw/c1 2>/dev/null || echo "(MISSING)"
    sudo umount /mnt/lw
  }
done
```

Then the exhaustive version — replay to **every flush point**:

```sh
N=$(sudo $REPLAY --log $LOG --num-entries 2>/dev/null | grep -oP '\d+' | head -1)
FAILURES=0
for i in $(seq 1 100 $N); do
  sudo $REPLAY --log $LOG --replay $DATA --limit $i > /dev/null 2>&1
  if ! sudo e2fsck -fn $DATA > /dev/null 2>&1; then
    echo "INCONSISTENT at entry $i"
    FAILURES=$((FAILURES+1))
  fi
done
echo "$FAILURES inconsistent states out of $((N/100)) checked"
```

**A correct journalling filesystem should produce zero.** Any failure is a real bug. Run this against your own filesystem from Ch. 56 and compare.

`dm-flakey`, for the cruder version:

```sh
sudo dmsetup remove lw
# Drop writes after 5 seconds of operation
sudo dmsetup create fl --table \
  "0 $SZ flakey $DATA 0 5 5 1 drop_writes"
sudo mkfs.ext4 -qF /dev/mapper/fl
sudo mount /dev/mapper/fl /mnt/lw
sudo sh -c 'for i in $(seq 1 1000); do echo x > /mnt/lw/f$i; done'
sudo umount /mnt/lw
sudo e2fsck -fn $DATA
```

---

### Lab 61.7 — Does your device tell the truth?

```sh
# What it claims
cat /sys/block/$(basename $DEV)/queue/write_cache
cat /sys/block/$(basename $DEV)/queue/fua
sudo hdparm -W $DEV 2>/dev/null
sudo sdparm --get=WCE $DEV 2>/dev/null
sudo nvme id-ctrl /dev/nvme0 2>/dev/null | grep -iE 'vwc|awun|awupf'
sudo smartctl -a $DEV 2>/dev/null | grep -iE 'power.loss|capacitor|Power_Loss'
```

Measure whether flushes actually cost anything:

```sh
sudo mkfs.ext4 -qF $DEV && sudo mount $DEV /mnt/j

# If fsync latency << device write latency, the device is lying
# (or it is power-loss protected)
sudo ./fsyncbench /mnt/j/b 0

# Compare with a raw device flush
cat > flushbench.c <<'EOF'
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <time.h>
#include <unistd.h>
int main(int argc, char **argv) {
	int fd = open(argv[1], O_WRONLY|O_DIRECT), i;
	void *buf; struct timespec a, b;
	posix_memalign(&buf, 4096, 4096); memset(buf, 0, 4096);
	clock_gettime(CLOCK_MONOTONIC, &a);
	for (i = 0; i < 1000; i++) { pwrite(fd, buf, 4096, i*4096); fdatasync(fd); }
	clock_gettime(CLOCK_MONOTONIC, &b);
	printf("raw write+flush: %.0f us\n",
	       ((b.tv_sec-a.tv_sec)*1e6 + (b.tv_nsec-a.tv_nsec)/1e3)/1000);
	return 0;
}
EOF
gcc -O2 -o flushbench flushbench.c
sudo ./flushbench $DEV
```

Reference numbers:

| Device | Honest write+flush |
|---|---|
| 7200 RPM HDD | 8–15 ms |
| Consumer SATA SSD | 200 µs – 2 ms |
| Consumer NVMe | 50–500 µs |
| **Enterprise NVMe (PLP)** | **10–50 µs** |
| **A lying device** | **< 10 µs on any of the above** |

If a consumer SSD reports 5 µs per write+flush, it is not flushing.

The real test — actual power loss:

```sh
# diskchecker.pl (from Brad Fitzpatrick), the standard tool.
# On a SECOND machine:
#   ./diskchecker.pl -l
# On the machine under test:
#   ./diskchecker.pl -s <server> create /mnt/j/testfile 500
# Then CUT POWER PHYSICALLY (pull the plug; do not use the OS).
# After reboot:
#   ./diskchecker.pl -s <server> verify /mnt/j/testfile
```

There is no substitute. Simulation cannot detect a device that lies about flushes.

Atomic write size:

```sh
# How large a write is atomic?
sudo nvme id-ns /dev/nvme0n1 2>/dev/null | grep -iE 'nawun|nabo|npwg'
cat /sys/block/$(basename $DEV)/queue/atomic_write_unit_max_bytes 2>/dev/null
cat /sys/block/$(basename $DEV)/queue/physical_block_size

# Torn-write detection: write a pattern, crash, check for mixed content
cat > tornwrite.c <<'EOF'
/* Write alternating 'A'-filled and 'B'-filled 16 KiB blocks with fdatasync.
 * After a crash, a block containing BOTH A and B is a torn write. */
EOF
```

---

### Lab 61.8 — Build a crash-safe application store

Put it all together.

```c
// SPDX-License-Identifier: GPL-2.0
/* safestore.c -- a minimal crash-safe key-value store.
 * Demonstrates every correct pattern from T.9. */
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <zlib.h>            /* crc32 */

#define LOG_MAGIC 0x5AFE5701
#define LOG_SIZE  (64UL << 20)

struct record {
	uint32_t magic;
	uint32_t crc;            /* over everything after this field */
	uint32_t klen, vlen;
	uint64_t seq;
	char     data[];         /* key then value */
} __attribute__((packed));

struct store {
	int   log_fd;
	int   dir_fd;
	off_t log_off;
	uint64_t seq;
};

static int fsync_dir(const char *path)
{
	int fd = open(path, O_RDONLY | O_DIRECTORY);
	int ret;

	if (fd < 0)
		return -1;
	ret = fsync(fd);         /* T.4's directory trap */
	close(fd);
	return ret;
}

static struct store *store_open(const char *dir)
{
	struct store *s = calloc(1, sizeof(*s));
	char path[512];

	snprintf(path, sizeof(path), "%s/wal", dir);
	s->log_fd = open(path, O_RDWR | O_CREAT, 0644);
	if (s->log_fd < 0) { free(s); return NULL; }

	/* Preallocate: no metadata updates on append -> fdatasync suffices */
	if (fallocate(s->log_fd, 0, 0, LOG_SIZE) && errno != EOPNOTSUPP) {
		perror("fallocate");
	}
	fsync(s->log_fd);
	fsync_dir(dir);          /* the WAL file itself must be durable */

	return s;
}

static int store_put(struct store *s, const char *k, const char *v)
{
	size_t klen = strlen(k), vlen = strlen(v);
	size_t total = sizeof(struct record) + klen + vlen;
	struct record *r = calloc(1, total);
	ssize_t n;

	r->magic = LOG_MAGIC;
	r->klen = klen;
	r->vlen = vlen;
	r->seq = ++s->seq;
	memcpy(r->data, k, klen);
	memcpy(r->data + klen, v, vlen);

	/* T.9: checksum the record, so a torn write is DETECTABLE */
	r->crc = crc32(0, (const Bytef *)&r->klen,
		       total - offsetof(struct record, klen));

	n = pwrite(s->log_fd, r, total, s->log_off);
	free(r);
	if (n != (ssize_t)total)
		return -1;

	/* No size change (preallocated) -> fdatasync, not fsync */
	if (fdatasync(s->log_fd) != 0) {
		/* T.5: an fsync error is NOT retryable. Fail loudly. */
		fprintf(stderr, "FATAL: fdatasync failed: %s\n", strerror(errno));
		fprintf(stderr, "Data may be lost. Recover from the last checkpoint.\n");
		abort();
	}

	s->log_off += total;
	return 0;
}

/* Recovery: scan forward, stop at the first invalid record. */
static int store_recover(struct store *s)
{
	off_t off = 0;
	struct record hdr;
	int recovered = 0;

	while (pread(s->log_fd, &hdr, sizeof(hdr), off) == sizeof(hdr)) {
		size_t total;
		struct record *r;
		uint32_t crc;

		if (hdr.magic != LOG_MAGIC)
			break;                        /* end of the log */

		total = sizeof(hdr) + hdr.klen + hdr.vlen;
		if (hdr.klen > 4096 || hdr.vlen > (1 << 20))
			break;                        /* implausible: corrupt */

		r = malloc(total);
		if (pread(s->log_fd, r, total, off) != (ssize_t)total) {
			free(r); break;
		}
		crc = crc32(0, (const Bytef *)&r->klen,
			    total - offsetof(struct record, klen));
		if (crc != r->crc) {
			printf("torn/corrupt record at %ld: stopping\n", (long)off);
			free(r);
			break;                        /* THE crash point */
		}
		printf("recovered seq %lu: %.*s = %.*s\n",
		       (unsigned long)r->seq, r->klen, r->data,
		       r->vlen, r->data + r->klen);
		s->seq = r->seq;
		off += total;
		recovered++;
		free(r);
	}
	s->log_off = off;
	return recovered;
}

/* Checkpoint: write a snapshot atomically, then truncate the log. */
static int store_checkpoint(struct store *s, const char *dir)
{
	char tmp[512], fin[512];
	int fd;

	snprintf(tmp, sizeof(tmp), "%s/snapshot.tmp", dir);
	snprintf(fin, sizeof(fin), "%s/snapshot", dir);

	fd = open(tmp, O_WRONLY | O_CREAT | O_TRUNC, 0644);
	if (fd < 0) return -1;
	/* ... write the in-memory state, checksummed ... */
	if (fsync(fd) != 0) { close(fd); return -1; }    /* data durable */
	close(fd);

	if (rename(tmp, fin) != 0) return -1;            /* atomic swap */
	if (fsync_dir(dir) != 0) return -1;              /* rename durable */

	/* ONLY NOW is it safe to discard the log. */
	ftruncate(s->log_fd, 0);
	fallocate(s->log_fd, 0, 0, LOG_SIZE);
	fdatasync(s->log_fd);
	s->log_off = 0;
	return 0;
}

int main(int argc, char **argv)
{
	const char *dir = argc > 1 ? argv[1] : "/mnt/j";
	struct store *s = store_open(dir);
	char k[64], v[64];
	int i;

	printf("recovering...\n");
	printf("%d records recovered\n", store_recover(s));

	for (i = 0; i < 100; i++) {
		snprintf(k, sizeof(k), "key%d", i);
		snprintf(v, sizeof(v), "value%d", i);
		store_put(s, k, v);
	}
	printf("wrote 100 records\n");
	return 0;
}
```

```sh
gcc -O2 -o safestore safestore.c -lz
sudo ./safestore /mnt/lw
```

Now test it with `dm-log-writes`:

```sh
sudo dmsetup message lw 0 mark before
sudo ./safestore /mnt/lw
sudo dmsetup message lw 0 mark after
sudo umount /mnt/lw

# Replay to every point between and verify recovery always works
for i in $(seq 1 50 $N); do
  sudo $REPLAY --log $LOG --replay $DATA --limit $i > /dev/null 2>&1
  sudo mount $DATA /mnt/lw 2>/dev/null || continue
  OUT=$(sudo ./safestore /mnt/lw 2>&1 | grep -c recovered)
  sudo umount /mnt/lw
  # Every replay point must produce a consistent recovery.
done
```

**If your recovery is correct, every single crash point yields a valid prefix of the writes.** That is the property to aim for, and it is achievable with the patterns above.

---

## 3. Mastery drills

1. State the atomicity gap precisely. Then show that write-ahead logging reduces multi-block atomicity to single-sector atomicity, and identify the exact sector that carries the reduction.

2. Enumerate the four approaches in §T.1. For each, state its recovery-time complexity and its steady-state write amplification.

3. Both flushes in §T.2 are mandatory. For each, construct the exact crash interleaving that corrupts the filesystem if it is omitted.

4. `REQ_PREFLUSH` and `REQ_FUA` are the only primitives. Show that "order A before B without flushing everything" cannot be expressed, and explain why `WRITE_BARRIER` was removed.

5. JBD2 uses physical logging. State the idempotence argument, then construct a logical log record that is not idempotent and the corruption it causes on double replay.

6. Revoke records: construct the full five-step corruption scenario, then explain why the revoke pass must precede the replay pass rather than being handled inline.

7. Credits must be reserved up front. Construct the failure that occurs if a transaction under-reserves, and explain why "just extend the transaction" is not an option.

8. `journal_async_commit` removes a flush. State the argument precisely, then construct the corruption that would occur without the checksum.

9. Reconstruct fsyncgate: the pattern, the kernel behaviour, the data loss, and the fix. Then state why `errseq_t`'s per-file tracking is necessary rather than a simple sticky flag.

10. §T.9 lists eight application mistakes. For each, write the minimal program that exhibits it and the minimal fix.

11. A device that lies about flushes is indistinguishable from an honest one except under power loss. Design a test that detects lying **without** cutting power, and state the assumption it requires.

12. `sync_file_range` is not a durability primitive. Explain exactly what it does, when it is useful, and construct the data-loss scenario from misusing it.

13. You are given a storage stack: application → ext4 → dm-crypt → MD RAID5 → SATA SSDs with volatile cache. Trace `fsync` through every layer and identify every point where durability could be lost.

---

## 4. Further reading

**Kernel documentation**

- `Documentation/filesystems/journalling.rst` ★★★ — the JBD2 API.
- `Documentation/block/writeback_cache_control.rst` ★★★ — **§T.3, normatively.** Short and essential.
- `Documentation/filesystems/ext4/journal.rst` ★★★ — the on-disk journal format in full.
- `Documentation/filesystems/xfs/xfs-delayed-logging-design.rst` — the contrast (Ch. 58 §T.6).
- `man 2 fsync`, `man 2 fdatasync`, `man 2 sync_file_range` ★★★ — read the BUGS and NOTES sections carefully; they were rewritten after fsyncgate.
- `man 2 open` (O_DIRECT, O_SYNC, O_DSYNC sections)
- `man 5 ext4` — the `data=`, `barrier=`, `journal_checksum` options.

**Papers**

- Mohan et al., "ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging," ACM TODS 1992 ★★★ — **the foundational WAL paper.** Long, but §T.2 is its first few sections.
- Tweedie, "Journaling the Linux ext2fs Filesystem," LinuxExpo 1998 ★★★ — ext3's design, by its author.
- Prabhakaran, Arpaci-Dusseau, Arpaci-Dusseau, "Analysis and Evolution of Journaling File Systems," USENIX 2005 ★★★ — **semantic block-level analysis**: reverse-engineers what each journalling filesystem actually does. A brilliant methodology paper.
- Ganger & Patt, "Metadata Update Performance in File Systems," OSDI 1994 — soft updates, §T.1's road not taken.
- Pillai, Chidambaram, Alagappan, Al-Kiswany, Arpaci-Dusseau, Arpaci-Dusseau, "All File Systems Are Not Created Equal: On the Complexity of Crafting Crash-Consistent Applications," OSDI 2014 ★★★ — **§T.9 in full, with the ALICE tool.** The single most important paper for anyone writing code that calls `fsync`.
- Chidambaram et al., "Optimistic Crash Consistency," SOSP 2013 — can you get consistency without ordering? (Partly.)
- Zheng et al., "Torturing Databases for Fun and Profit," OSDI 2014 ★★★ — systematic power-fault injection against real databases. Found bugs in all of them.
- Bornholt et al., "Specifying and Checking File System Crash-Consistency Models," ASPLOS 2016 — formalises what each filesystem guarantees.
- Arpaci-Dusseau, *OSTEP*, Chapters 42 ("Crash Consistency: FSCK and Journaling") ★★★ — free, and the clearest possible introduction. **Read this first if any of §T.1–T.2 is unclear.**

**LWN**

- "PostgreSQL's fsync() surprise" (2018) ★★★ — **fsyncgate, covered thoroughly.** Read this.
- "Improved block-layer error handling" and the `errseq_t` coverage ★★★
- "Barriers and journaling filesystems" (2008) ★★★ — the WRITE_BARRIER era
- "The end of block barriers" (2010) ★★★ — why they were removed, and what replaced them
- "Ensuring data reaches disk" (2011) ★★★ — a superb practical guide, still accurate
- "Filesystem crash-consistency testing" and the CrashMonkey coverage
- "Atomic writes" and the recent `RWF_ATOMIC` work
- "Toward reliable power-fail testing"

**Blog posts and primary sources worth reading**

- The PostgreSQL `fsync` mailing-list thread (2018) — the original discovery, in real time.
- Craig Ringer's "PostgreSQL's handling of fsync() errors is unsafe" — the summary that started it.
- Dan Luu, "Files are hard" ★★★ — an accessible, well-researched survey of everything that goes wrong.
- The `diskchecker.pl` source and README — §T.6's test, and the reasoning behind it.

**Source reading order**

1. OSTEP Chapter 42, then the Tweedie paper.
2. `include/linux/jbd2.h` ★★★ — the structures.
3. `fs/jbd2/transaction.c`: `start_this_handle`, `jbd2_journal_get_write_access`, `jbd2_journal_stop`.
4. `fs/jbd2/commit.c`: `jbd2_journal_commit_transaction` ★★★ — read it top to bottom; the phase comments are excellent.
5. `fs/jbd2/recovery.c`: `do_one_pass` — all three passes.
6. `fs/jbd2/revoke.c` — the header comment explains §T.7's problem better than most papers.
7. `block/blk-flush.c` ★★★ — the header comment is a complete explanation of §T.3.
8. `lib/errseq.c` ★★★ — 200 lines, read all of it.

**Tools**

- `dm-log-writes` + `replay-log` ★★★ — **the right way to test crash consistency.** In `xfstests-dev/src/log-writes/`.
- `dm-flakey`, `dm-error`, `dm-dust` — cruder fault injection
- `debugfs -R "logdump"` ★★★, `xfs_logprint`
- `/proc/fs/jbd2/DEV/info` ★★★
- `trace-cmd record -e jbd2:\*` and `-e block:\*` ★★★
- `diskchecker.pl` ★★★ — the power-loss test
- `CrashMonkey` / `ALICE` — systematic exploration
- `xfstests` crash-consistency groups: `./check -g recoveryloop` and `-g log`
- `fio` with `--fsync=1`, `--fdatasync=1`, `--sync=1` to isolate each cost
- `blktrace`/`blkparse` — to see flush and FUA flags on every request

---

→ Next: [62-network-stacking-fs.md](62-network-stacking-fs.md)

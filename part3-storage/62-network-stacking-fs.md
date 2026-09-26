# Chapter 62 — Network and stacking filesystems: NFS, SMB, overlayfs, FUSE

> **Goal:** Understand what breaks when the filesystem is not local. Understand why caching is impossible to get both fast and correct over a network, NFS's close-to-open consistency as an engineering compromise rather than a specification, statefulness as the defining difference between NFSv3 and NFSv4, why file handles must outlive the server, overlayfs's copy-up model and its three hard problems, FUSE as a protocol rather than a filesystem and the deadlock classes it creates, and why "just put a filesystem in userspace" costs what it does. By the end you can debug an NFS stale handle, explain an overlayfs whiteout, and reason about FUSE performance from first principles.

---

## Theory & First Principles

### T.0 — Start here: the same code, one impossible assumption

```bash
mount -t nfs server:/export /mnt
cat /mnt/file       # identical syscalls, identical VFS path (Ch. 53)
```

The application cannot tell. The VFS barely can. **But one assumption that every local
filesystem relies on is now false:**

> **A local filesystem's storage either answers or the machine is dead. A network
> filesystem's storage can be *unreachable but fine*, *reachable but slow*, or *answering
> with stale data* — and it cannot distinguish these from each other.**

That single change cascades through every design decision:

| Local assumption | On a network | Consequence |
|---|---|---|
| Operations complete in bounded time | may hang **forever** | `hard` vs `soft` mounts; `D`-state processes that even `kill -9` cannot touch |
| An operation happens exactly once | a reply can be lost *after* the work was done | the client retries — so operations must be **idempotent**, or a reply cache must dedupe them |
| One machine sees all changes | many clients, each with a page cache | **cache coherence** becomes a distributed problem |
| `fsync` means durable | durable *where*? | the server may still have it in RAM |
| The kernel enforces permissions | the *client's* kernel asserts "uid 1000" | classic NFS simply trusts it — Ch. 93 |

**Take the retry problem concretely**, because it is the cleanest illustration:

```
  client: MKDIR /foo  ------------->  server: creates /foo, replies OK
                                              (the reply is LOST)
  client: timeout. retry.
  client: MKDIR /foo  ------------->  server: EEXIST
  client: "mkdir failed"  -- but it SUCCEEDED.
```

NFSv2/v3 made most operations idempotent (`WRITE` at an explicit offset is; `MKDIR` and
`REMOVE` are not) and added a **duplicate request cache** on the server. **Idempotence is not
a nicety in a distributed system; it is the price of being allowed to retry** — the same
lesson as journal replay in Ch. 61 §T.0.

**And the coherence problem has no free answer.** NFSv3 chose *close-to-open* consistency:
flush on `close()`, revalidate on `open()`. Between those points, clients may disagree.

```
  client A: write(fd, ...); close(fd);     <- now visible
  client B: open(); read();                <- sees A's data

  but:  A writes, B reads, A writes again, B reads again -> B may see anything
```

That is **not** POSIX semantics. It was chosen because real POSIX coherence requires a
distributed locking protocol on every read — which is what NFSv4 delegations and SMB oplocks
provide, at the cost of state on the server and a recovery protocol when a client dies.
**Statelessness bought NFSv3 trivial server restart and cost it correctness; NFSv4 made the
opposite trade and had to invent lease recovery.** There is no third option.

**The organizing principle for this chapter is the end-to-end argument** (Saltzer, Reed &
Clark, 1984):

> A function can only be completely and correctly implemented with the knowledge of the
> application at the endpoints. Implementing it in the communication system is either
> impossible, or merely a performance optimization.

The network cannot promise your data is durable — only the application, by doing an explicit
`fsync` and checking its result, can know. This is the same argument that made ZFS put
checksums at the filesystem level (Ch. 60 §T.0) instead of trusting the disk, and the same
argument you will apply to stacking filesystems (overlayfs, eCryptfs) later in this chapter.

```bash
nfsstat -c                       # client RPC counts, retransmissions
mount | grep nfs                 # hard/soft, actimeo, vers
cat /proc/self/mountstats        # per-operation latency histograms
sudo rpcdebug -m nfs -s all      # verbose client tracing
```

---

### T.1 What changes when the filesystem is remote

Every assumption from Chapters 51–61 breaks:

| Local assumption | Remote reality |
|---|---|
| A lookup costs a memory access | a lookup costs a round trip (0.1–100 ms) |
| The dcache is authoritative once populated | another client may have changed it |
| `i_size` is the truth | the truth is on the server |
| A file's data is stable while we hold the inode | another client can write it |
| An operation either happens or does not | **it may have happened but the reply was lost** |
| Locks are held by threads we can see | the lock holder may have crashed, or be unreachable |
| The filesystem is always available | the network partitions |
| Permissions are checked against local uids | the uid namespaces differ |

The last three are the genuinely hard ones, and the fifth is the deepest: **on a network, you cannot distinguish "the operation failed" from "the operation succeeded and the reply was lost."** That single fact shapes everything about NFS's design.

### T.2 The caching dilemma

Without caching, every `read()` is a round trip and performance is unusable. With caching, a client can return stale data. There is no third option; there is only a choice of where on the spectrum to sit.

The consistency models, from strongest to weakest:

| Model | Guarantee | Cost |
|---|---|---|
| **Strict / linearizable** | every read sees the latest write, globally | a round trip per operation, or a distributed lock protocol |
| **Sequential** | all clients see operations in one consistent order | synchronous invalidation |
| **Close-to-open** | a client opening a file sees everything written by any client that *closed* it first | flush on close, revalidate on open |
| **Eventual / timeout-based** | caches expire after N seconds | cheap, sometimes wrong |
| **None** | whatever is cached | fastest, often wrong |

**NFS chose close-to-open (CTO)**, and it is worth understanding as an engineering decision rather than a limitation.

The observation: most files are not concurrently shared. A file is typically written by one process, closed, then read by another. If you guarantee that sequence works, you cover the overwhelming majority of real usage — and you can cache aggressively in between.

The contract:

```
Client A:  open() ... write() ... close()     <- close FLUSHES to the server
Client B:                                 open() ... read()
                                          ^-- open REVALIDATES (GETATTR)
           B sees everything A wrote.
```

What CTO does **not** guarantee:

- Two clients writing the same file concurrently: undefined interleaving.
- A client reading a file another client has open and is writing: may see stale data indefinitely.
- `mmap` coherence across clients: essentially none.
- Append-atomicity across clients: none (`O_APPEND` is client-side).

The classic failure: a shared log file written by two hosts. Both cache, both compute offsets locally, both write. Data is lost. **Not a bug; CTO never promised otherwise.**

Implementation: the client caches attributes with a timeout (`acregmin`/`acregmax`, default 3–60 s) and uses the file's `ctime`/`mtime`/`change` attribute to decide whether cached data is valid. `GETATTR` on open is the revalidation. If the attributes match what was cached, the page cache is kept; if not, it is invalidated.

`nocto` disables the revalidation (faster, less correct). `actimeo=0` forces revalidation on every access (correct-ish, very slow). `noac` also disables attribute caching entirely.

### T.3 Idempotence and the duplicate request problem

Because a lost reply is indistinguishable from a lost request, the client must retransmit. So **every operation may execute more than once.**

For idempotent operations this is fine:

| Idempotent | Not idempotent |
|---|---|
| `GETATTR`, `LOOKUP`, `READ` | `CREATE` with `O_EXCL` |
| `WRITE` **at an explicit offset** | `REMOVE`, `RMDIR` |
| `SETATTR` (absolute values) | `RENAME` |
| `READDIR` | `LINK`, `MKDIR` |

NFSv2/v3's design deliberately makes as much idempotent as possible — which is why NFS `WRITE` takes an explicit offset rather than using an implicit file position, and why there is no server-side `O_APPEND`. **The protocol was shaped by the retransmission requirement.**

For the non-idempotent remainder, servers keep a **Duplicate Request Cache (DRC)**: a hash of recent `(xid, client, procedure)` → reply. A retransmission hits the cache and gets the original reply rather than re-executing. The DRC is best-effort — it is bounded in size and lost on server restart — which is why `rm` over NFS can return `ENOENT` for a file it successfully removed (the reply was lost, the retry found it gone, the DRC had evicted the entry).

This is a genuine, unavoidable artefact. `rm -f` exists partly because of it.

NFSv4 improves on this with **sessions and exactly-once semantics (EOS)**: a per-session reply cache with explicit slot management, where the client guarantees at most one outstanding request per slot, so the server knows exactly how much to remember. This is the correct solution, and it required making the protocol stateful.

### T.4 File handles: identity without paths

A client refers to files by **file handle** — an opaque byte string the server issued. It must satisfy:

1. **Persistent across server reboot.** So it cannot contain memory addresses.
2. **Independent of the path.** Rename must not invalidate it.
3. **Verifiable.** A handle for a deleted-and-recreated file must fail.
4. **Bounded size** (64 bytes in v3, 128 in v4).

The Linux server's construction:

```
fsid (which exported filesystem) | inode number | generation number | parent info
```

which is exactly Ch. 56 §T.8's `export_operations`. The generation number is what makes requirement 3 work: reuse of an inode number bumps the generation, and an old handle then fails with **`ESTALE`**.

`ESTALE` is the characteristic NFS error and it means precisely: *"the file this handle referred to no longer exists."* Causes, in order of frequency:

| Cause | Fix |
|---|---|
| The file was deleted by another client | expected; the application must handle it |
| The exported filesystem was remounted with a different `fsid` | pin `fsid=` in `/etc/exports` |
| The server's filesystem was `fsck`'d and inodes renumbered | re-mount the client |
| The export was removed and re-added | pin `fsid=` |
| A subtree-checked export and the file moved | use `no_subtree_check` |

The `subtree_check` option deserves a note because it is a classic wrong-default: it encodes the file's position within the export in the handle so the server can verify the file is still under the exported subtree. But that makes the handle path-dependent, breaking requirement 2 — so renaming a file causes `ESTALE`. `no_subtree_check` is now the default and is correct for essentially all uses.

### T.5 NFSv3 versus NFSv4: statelessness was the mistake

NFSv2/v3 were deliberately **stateless**: the server keeps no per-client state, so a server reboot is invisible — clients just retry and everything works. This was a celebrated design decision in 1985.

It has three consequences that turned out to be fatal:

**(a) Locking requires a separate, stateful protocol.** NLM (Network Lock Manager) plus NSM (Network Status Monitor) sit alongside NFS to provide locks, with their own ports, their own crash-recovery protocol, and a well-earned reputation for fragility. Locks that persist after a client crashes require the server to notice — hence NSM, which is a heartbeat protocol that frequently does not work.

**(b) Multiple ports and RPC.** portmapper/rpcbind (111), mountd, nfsd (2049), lockd, statd — each on a different, often dynamic, port. **Impossible to firewall sensibly, impossible to NAT.**

**(c) No open state means no delegations.** The server cannot know that only one client has a file open, so it cannot grant that client permission to cache aggressively.

**NFSv4 is stateful and fixes all three:**

| | v3 | v4 |
|---|---|---|
| Ports | many, dynamic | **2049 only** |
| Locking | separate NLM/NSM | **integrated** |
| Mount protocol | separate mountd | integrated (`PUTROOTFH`) |
| State | none | open/lock state with a lease |
| Recovery | implicit (retry) | **explicit grace period** |
| Compound ops | no | **yes** — many operations per round trip |
| Delegations | no | yes |
| Security | AUTH_SYS (uid/gid on the wire) | **RPCSEC_GSS / Kerberos** |
| ACLs | no | yes |
| Identity | numeric uid/gid | **`user@domain` strings** (idmapd) |

Three v4 features worth understanding:

**COMPOUND** lets a client send `PUTFH, LOOKUP, LOOKUP, LOOKUP, GETFH, GETATTR` in one round trip. Path resolution over a high-latency link goes from N round trips to one. This is the single biggest performance improvement in v4.

**Leases and the grace period.** Client state is held under a lease (typically 90 s) which the client renews. After a server reboot, the server enters a **grace period** during which it accepts only *reclaim* operations from clients recovering their previous state, refusing new locks. Once the grace period expires, state that was not reclaimed is discarded. This is a clean, explicit crash-recovery protocol — the thing v3 lacked.

**Delegations.** If only one client has a file open, the server can grant a **read** or **write delegation**: "you may cache this aggressively; I promise to tell you if anyone else wants it." The client then operates locally at full speed. When another client requests conflicting access, the server **recalls** the delegation via a callback, and the holder must flush and return it.

Delegations require a callback channel — the server must be able to contact the client. In v4.0 this was a separate connection (a NAT and firewall disaster); **v4.1 introduced sessions, which multiplex the callback over the existing connection.** That alone makes v4.1 the minimum version worth deploying.

### T.6 Stacking: overlayfs

Overlayfs presents a union of two (or more) directory trees:

```
          merged view (what you see)
              ↑
    ┌─────────┴─────────┐
  upper (read-write)   lower (read-only, possibly several)
```

Rules:

- A file present in upper shadows the same name in lower.
- Directories are **merged**: the union of entries from all layers.
- Reading a file that exists only in lower reads it directly from lower — no copy.
- **Writing** to a lower file triggers **copy-up**: the whole file is copied to upper, then modified there.
- Deleting a lower file creates a **whiteout** in upper.

This is the mechanism behind every container image system. A Docker image's layers are lower dirs; the container's writable layer is upper.

Three hard problems, and the solutions are instructive:

**(a) Whiteouts.** How do you represent "this file, which exists in lower, has been deleted"? Overlayfs uses a **character device with major 0, minor 0** in upper. It cannot be a regular file (that would be a file, not an absence) and cannot be a special overlayfs-only type (upper must be an ordinary filesystem). A 0/0 char device is otherwise meaningless, so it is available as a sentinel.

For directories, an **opaque** marker (`trusted.overlay.opaque="y"` xattr) means "do not merge with lower; this directory replaces it entirely."

**(b) Copy-up granularity.** Copy-up is **whole-file**. Appending one byte to a 10 GiB file in lower copies 10 GiB. There is no partial copy-up, because tracking which ranges came from where would require per-range metadata that upper cannot hold.

Mitigations:
- **Metadata-only copy-up** (`metacopy=on`): if only metadata changes (chmod, chown), copy only the metadata and record the origin in `trusted.overlay.redirect`. Data stays in lower.
- **Reflink copy-up**: if upper and lower are the same filesystem and it supports reflinks, copy-up is O(1). This is why container storage on XFS-with-reflink or btrfs is much better than on ext4.

**(c) `readdir` and inode numbers.** A merged directory's entries come from multiple filesystems with independent inode-number spaces. Two different files can have the same inode number. `find -inum`, hardlink detection in `tar`, and `rsync -H` all break.

`xino=on` fixes this by stealing high bits of the inode number to encode the layer, producing unique numbers — provided there are spare bits. On 32-bit inode numbers there are not, and `xino=auto` falls back.

Also: `rename` of a directory from lower to upper is not possible atomically, so overlayfs returns `EXDEV` and expects userspace to copy. `redirect_dir=on` makes it work by leaving a redirect xattr, at the cost of making upper non-portable.

### T.7 FUSE: a protocol, not a filesystem

FUSE lets a userspace process implement a filesystem. The structure:

```
Application
    ↓ syscall
VFS
    ↓ ->lookup, ->read, ...
fuse kernel module
    ↓ writes a request to /dev/fuse
    ↓ ─────────────────────────────►  userspace daemon
    ↓ ◄─────────────────────────────  reply
    ↓
returns to the application
```

Each operation is a **round trip through userspace**: at minimum two context switches and two copies. A cached local `stat()` costs ~1 µs; a FUSE `stat()` costs 10–50 µs even with a trivial handler. That ratio is FUSE's fundamental cost and no amount of optimisation removes it.

Mitigations, all partial:

| Mechanism | Effect |
|---|---|
| Attribute caching (`entry_timeout`, `attr_timeout`) | avoid the round trip for repeated `stat` |
| `FOPEN_KEEP_CACHE` | keep the page cache across opens |
| `writeback_cache` | let the kernel buffer writes rather than passing each through |
| `max_read`/`max_write`, `max_pages` | fewer, larger round trips |
| `splice_read`/`splice_write` | avoid one copy |
| **`FUSE_PASSTHROUGH`** (6.9+) | for stacking filesystems, let the kernel do I/O directly on a backing fd, bypassing userspace entirely for reads/writes |
| **virtiofs** | FUSE over a virtio transport, with DAX; for VMs |

`FUSE_PASSTHROUGH` is the significant recent development: a FUSE filesystem that is mostly a pass-through (as most container and sandboxing filesystems are) can register a backing file descriptor, and the kernel then routes `read`/`write` straight to it. The userspace daemon still handles the namespace, but the data path is native speed.

**The deadlock problem** is FUSE's deepest issue and worth understanding properly:

```
Process A: write() to a FUSE file
  -> the kernel allocates memory
  -> memory is low -> direct reclaim
  -> reclaim tries to write back a dirty FUSE page
  -> which requires the FUSE daemon to service the request
  -> but the FUSE daemon is blocked allocating memory
  -> DEADLOCK
```

Mitigations: FUSE pages are accounted against a global limit (`/sys/fs/fuse/connections/*/max_background`), the daemon should set `PR_SET_IO_FLUSHER` (5.6+) so reclaim treats it specially, and it should `mlockall()` its working set. **A FUSE daemon that allocates memory in its request path is a latent deadlock**, and this is the single most common serious FUSE bug.

Related: a FUSE daemon that hangs makes every process touching the mount unkillable in `D` state. `fusermount -uz` (lazy unmount) and `/sys/fs/fuse/connections/N/abort` are the escape hatches:

```sh
echo 1 > /sys/fs/fuse/connections/42/abort   # fail all pending requests
```

**Security.** Unprivileged FUSE mounts (`user_allow_other`, and unprivileged mounts in a user namespace) mean an unprivileged user can present arbitrary filesystem behaviour to a privileged reader. This has produced real vulnerabilities: a FUSE filesystem can block indefinitely in the middle of a `read()` issued by a setuid program, or return different data on successive reads to win a TOCTOU race. Mounting untrusted FUSE filesystems where privileged code will read them is a genuine hazard.

### T.8 The stacking problem in general

Stacking filesystems (overlayfs, ecryptfs, and historically unionfs) all face the same structural issues:

**(a) Which credentials?** When overlayfs copies a file up, whose permissions apply — the caller's, or the mounter's? Overlayfs records the mounter's credentials at mount time and uses them for internal operations, with `override_creds`. Getting this wrong is a privilege-escalation vulnerability, and overlayfs has had several.

**(b) Double caching.** A file read through overlayfs may be cached in both the upper/lower filesystem's page cache and, conceptually, the overlay's. Overlayfs avoids this by not having its own `address_space` for regular files — it uses the underlying file's. That is why `ovl_open` returns a file pointing at the real file, and why `mmap` through overlayfs works at native speed.

**(c) Change notification.** `inotify` on an overlay file does not see changes made directly to the underlying file. Nor should it, arguably, but it surprises people. Similarly, changing a lower layer while it is mounted is **explicitly unsupported** — the documentation says so, and doing it produces incoherent results.

**(d) Nesting.** Overlay on overlay works but multiplies the cost and can hit the 500-layer limits in container systems. Docker's default of squashing layers exists for this reason.

### T.9 Choosing

| Need | Use | Why |
|---|---|---|
| Unix-to-Unix file sharing | **NFSv4.1+** | integrated locking, one port, sessions, delegations |
| Windows interop | **SMB3** | native; also has good Linux client support |
| Container image layers | **overlayfs** | it is what the ecosystem assumes |
| VM filesystem passthrough | **virtiofs** | FUSE protocol, virtio transport, DAX |
| Cloud storage as a filesystem | FUSE (s3fs, rclone) | with clear-eyed expectations about semantics |
| Encryption | **fscrypt** or **dm-crypt**, not ecryptfs | ecryptfs is effectively unmaintained |
| Prototyping a filesystem | **FUSE** ★★★ | vastly easier than kernel code; port later if needed |
| Distributed, scale-out | CephFS, GlusterFS, Lustre | beyond this chapter |

And the honest warnings:

- **Do not run a database over NFS** unless you have verified the locking works and `sync` is honoured. The failure modes are silent.
- **Do not assume POSIX semantics over any network filesystem.** Test what you actually rely on.
- **`mmap` over NFS** is coherent only within one client.
- **FUSE performance is a protocol property**, not an implementation one. If you need native speed, you need kernel code or `FUSE_PASSTHROUGH`.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `fs/nfs/` ★★★ | the client: `inode.c`, `dir.c`, `file.c`, `write.c`, `nfs4state.c` |
| `fs/nfs/nfs4proc.c` | v4 operations and COMPOUND construction |
| `fs/nfsd/` ★★★ | the server: `nfssvc.c`, `nfs4state.c`, `vfs.c`, `nfsfh.c` |
| `fs/nfsd/nfsfh.c` ★★★ | §T.4's file handles |
| `fs/lockd/` | NLM (v3 locking) |
| `net/sunrpc/` | the RPC layer, `RPCSEC_GSS`, transports |
| `fs/smb/client/` | cifs/SMB3 client (formerly `fs/cifs/`) |
| `fs/smb/server/` | ksmbd, the in-kernel SMB server |
| `fs/overlayfs/` ★★★ | `super.c`, `dir.c`, `copy_up.c`, `inode.c`, `readdir.c` |
| `fs/fuse/` ★★★ | `dev.c` (the protocol), `dir.c`, `file.c`, `inode.c` |
| `fs/fuse/virtio_fs.c` | virtiofs |
| `fs/fuse/passthrough.c` | §T.7's `FUSE_PASSTHROUGH` |
| `include/uapi/linux/fuse.h` ★★★ | the complete FUSE protocol |
| `Documentation/filesystems/overlayfs.rst` ★★★ | authoritative |
| `Documentation/filesystems/nfs/` | exporting, idmapper, RPC cache |

### 1.2 NFS client caching

```c
struct nfs_inode {
	__u64			fileid;
	struct nfs_fh		fh;               /* T.4 */
	unsigned long		flags;
	unsigned long		cache_validity;   /* what needs revalidating */
	unsigned long		read_cache_jiffies;
	unsigned long		attrtimeo;        /* adaptive, T.2 */
	unsigned long		attrtimeo_timestamp;
	unsigned long		attr_gencount;
	struct nfs4_change_info	cinfo;
	...
	struct nfs_open_context *nfs4_open_context;
	struct nfs_delegation __rcu *delegation;   /* T.5 */
	...
};

/* Bits in cache_validity */
#define NFS_INO_INVALID_DATA	 BIT(1)
#define NFS_INO_INVALID_ATIME	 BIT(2)
#define NFS_INO_INVALID_ACCESS	 BIT(3)
#define NFS_INO_INVALID_ACL	 BIT(4)
#define NFS_INO_REVAL_FORCED	 BIT(5)
#define NFS_INO_INVALID_SIZE	 BIT(8)
#define NFS_INO_INVALID_MODE	 BIT(10)
#define NFS_INO_INVALID_CHANGE	 BIT(13)
```

Close-to-open in code:

```c
int nfs_revalidate_inode(struct inode *inode, unsigned long flags)
{
	if (!nfs_check_cache_invalid(inode, flags))
		return NFS_STALE(inode) ? -ESTALE : 0;
	return __nfs_revalidate_inode(NFS_SERVER(inode), inode);
}

/* On open: */
static int nfs_file_open(struct inode *inode, struct file *filp)
{
	...
	res = nfs_check_flags(filp->f_flags);
	if (res < 0) return res;
	return nfs_open(inode, filp);       /* -> revalidation */
}

/* On close: */
static int nfs_file_release(struct inode *inode, struct file *filp)
{
	...
	nfs_file_clear_open_context(filp);   /* -> flushes dirty data */
	...
}
```

The **adaptive attribute timeout** is a nice detail: `attrtimeo` starts at `acregmin` and doubles (up to `acregmax`) each time revalidation finds the attributes unchanged, resetting to the minimum when a change is found. A file nobody is modifying gets progressively cheaper to revalidate; a file being actively changed is checked often.

### 1.3 File handle construction

```c
/* fs/nfsd/nfsfh.h */
struct knfsd_fh {
	unsigned int	fh_size;
	union {
		char			fh_raw[NFS4_FHSIZE];
		struct {
			u8	fh_version;	/* == 1 */
			u8	fh_auth_type;
			u8	fh_fsid_type;
			u8	fh_fileid_type;
			u32	fh_fsid[];	/* then the file id */
		};
	};
};

__be32 fh_compose(struct svc_fh *fhp, struct svc_export *exp,
		  struct dentry *dentry, struct svc_fh *ref_fh)
{
	...
	if (fhp->fh_handle.fh_fileid_type != FILEID_ROOT) {
		int maxsize = ...;

		/* This calls the filesystem's ->encode_fh (Ch. 56 T.8) */
		fhp->fh_handle.fh_fileid_type =
			exportfs_encode_fh(dentry,
					   (struct fid *)fhp->fh_handle.fh_fileid,
					   &maxsize,
					   fhp->fh_maxsize > maxsize);
	}
	...
}
```

And the reverse:

```c
static __be32 nfsd_set_fh_dentry(struct svc_rqst *rqstp, struct svc_fh *fhp)
{
	...
	exp = rqst_exp_find(rqstp, fh->fh_fsid_type, fh->fh_fsid);
	...
	dentry = exportfs_decode_fh_raw(exp->ex_path.mnt, fid,
					data_left, fileid_type,
					nfsd_match_parent, exp);
	if (IS_ERR_OR_NULL(dentry)) {
		trace_nfsd_set_fh_dentry_badhandle(rqstp, fhp,
				dentry ? PTR_ERR(dentry) : -ESTALE);
		switch (PTR_ERR(dentry)) {
		case -ENOMEM:
		case -ETIMEDOUT:
			break;
		default:
			dentry = ERR_PTR(-ESTALE);     /* T.4 */
		}
	}
	...
}
```

`exportfs_decode_fh` checks the generation number (Ch. 56 §T.8(b)), and a mismatch produces `ESTALE`.

### 1.4 Overlayfs copy-up

```c
static int ovl_copy_up_one(struct dentry *parent, struct dentry *dentry,
			   int flags)
{
	struct ovl_copy_up_ctx ctx = {
		.parent = parent,
		.dentry = dentry,
		.workdir = ovl_workdir(dentry),
	};
	...
	ovl_path_lower(dentry, &ctx.lowerpath);
	err = vfs_getattr(&ctx.lowerpath, &ctx.stat,
			  STATX_BASIC_STATS, AT_STATX_SYNC_AS_STAT);
	...
	/* metacopy: copy only metadata if the data is unchanged (T.6(b)) */
	if (!S_ISREG(ctx.stat.mode))
		ctx.metacopy = false;
	else if (!ctx.origin && ovl_need_meta_copy_up(dentry, ctx.stat.mode, flags))
		ctx.metacopy = true;

	if (parent) {
		ovl_do_check_copy_up(ofs, ctx.lowerpath.dentry);
		err = ovl_copy_up_start(dentry, flags);
		...
		err = ovl_do_copy_up(&ctx);
		...
	}
	...
}

static int ovl_copy_up_data(struct ovl_copy_up_ctx *c, const struct path *temp)
{
	...
	/* Try reflink first: O(1) if upper and lower share a filesystem
	 * that supports it (T.6(b)). */
	error = do_clone_file_range(old_file, 0, new_file, 0, len, 0);
	if (error == len)
		goto out_fput;
	...
	/* Otherwise: copy_file_range, then splice, then a plain loop. */
	error = do_copy_file_range(old_file, old_pos, new_file, new_pos, len, 0);
	...
}
```

Whiteouts:

```c
int ovl_do_whiteout(struct ovl_fs *ofs, struct inode *dir,
		    struct dentry *dentry)
{
	/* T.6(a): a character device with major 0, minor 0 */
	int err = ovl_do_mknod(ofs, dir, dentry, S_IFCHR | 0, WHITEOUT_DEV);

	pr_debug("whiteout(%pd2) = %i\n", dentry, err);
	return err;
}

static bool ovl_is_whiteout(struct dentry *dentry)
{
	struct inode *inode = dentry->d_inode;

	return inode && IS_WHITEOUT(inode);
}

#define WHITEOUT_DEV 0
static inline bool is_whiteout_inode(struct inode *inode)
{
	return S_ISCHR(inode->i_mode) && inode->i_rdev == WHITEOUT_DEV;
}
```

### 1.5 The FUSE protocol

```c
/* include/uapi/linux/fuse.h */
struct fuse_in_header {
	uint32_t	len;
	uint32_t	opcode;
	uint64_t	unique;      /* matches the reply */
	uint64_t	nodeid;      /* which object */
	uint32_t	uid, gid, pid;
	uint16_t	total_extlen;
	uint16_t	padding;
};

struct fuse_out_header {
	uint32_t	len;
	int32_t		error;
	uint64_t	unique;
};

enum fuse_opcode {
	FUSE_LOOKUP		= 1,
	FUSE_FORGET		= 2,
	FUSE_GETATTR		= 3,
	FUSE_SETATTR		= 4,
	FUSE_READLINK		= 5,
	FUSE_SYMLINK		= 6,
	FUSE_MKNOD		= 8,
	FUSE_MKDIR		= 9,
	FUSE_UNLINK		= 10,
	...
	FUSE_OPEN		= 14,
	FUSE_READ		= 15,
	FUSE_WRITE		= 16,
	...
	FUSE_INIT		= 26,
	...
	FUSE_READDIRPLUS	= 44,
	FUSE_COPY_FILE_RANGE	= 47,
	...
};
```

The request loop:

```c
static void fuse_request_send(struct fuse_mount *fm, struct fuse_req *req)
{
	struct fuse_iqueue *fiq = &fm->fc->iq;

	__set_bit(FR_ISREPLY, &req->flags);
	if (!test_bit(FR_WAITING, &req->flags)) {
		__set_bit(FR_WAITING, &req->flags);
		atomic_inc(&fm->fc->num_waiting);
	}
	queue_request_and_unlock(fiq, req);
	/* The daemon reads it from /dev/fuse, handles it, writes the reply */
	request_wait_answer(req);       /* <- the round trip */
}
```

`request_wait_answer` is where the cost lives, and where the deadlock of §T.7 occurs if the daemon cannot make progress.

Attribute caching:

```c
void fuse_change_attributes(struct inode *inode, struct fuse_attr *attr,
			    struct fuse_statx *sx,
			    u64 attr_valid, u64 attr_version)
{
	...
	fi->attr_version = atomic64_inc_return(&fc->attr_version);
	fi->i_time = attr_valid;      /* the timeout the DAEMON chose */
	...
}
```

The daemon chooses the cache lifetime per reply. A filesystem backing immutable content can say "cache forever"; one backing a live remote source says "do not cache." **The kernel does not guess.**

### 1.6 Observability

| Where | What |
|---|---|
| `nfsstat -c` / `-s` ★★★ | per-operation client/server counters |
| `mountstats` / `nfsiostat` ★★★ | per-mount latency by operation |
| `/proc/self/mountstats` ★★★ | the raw data behind the above |
| `rpcdebug -m nfs -s all` | verbose client tracing (noisy) |
| `trace-cmd record -e nfs:\* -e nfs4:\* -e sunrpc:\*` ★★★ | modern, much better than rpcdebug |
| `/proc/fs/nfsd/*` | server exports, threads, state |
| `exportfs -v`, `showmount -e` | export configuration |
| `tcpdump -i any port 2049 -w nfs.pcap` + Wireshark ★★★ | the protocol itself |
| `trace-cmd record -e fuse:\*` | FUSE requests |
| `/sys/fs/fuse/connections/N/` ★★★ | `waiting`, `abort`, `max_background` |
| `cat /proc/PID/stack` | where a hung FUSE-using process is stuck |
| `mount -t overlay ... -o index=on,xino=on` then `getfattr -d -m - FILE` ★★★ | overlayfs xattrs |
| `smbstatus`, `smbclient`, `cifsiostat` | SMB |

---

## 2. Practice

### Lab 62.1 — NFS: set it up and watch it work

```sh
sudo apt install -y nfs-kernel-server nfs-common
sudo mkdir -p /srv/export /mnt/nfs
sudo chmod 777 /srv/export

echo "/srv/export 127.0.0.1(rw,sync,no_subtree_check,fsid=42)" | \
  sudo tee /etc/exports
sudo exportfs -ra
sudo exportfs -v
showmount -e 127.0.0.1
```

Mount v3 and v4 side by side:

```sh
sudo mkdir -p /mnt/nfs3 /mnt/nfs4
sudo mount -t nfs -o vers=3 127.0.0.1:/srv/export /mnt/nfs3
sudo mount -t nfs -o vers=4.2 127.0.0.1:/srv/export /mnt/nfs4
mount | grep nfs
```

Count round trips — §T.5's COMPOUND, measured:

```sh
sudo mkdir -p /srv/export/a/b/c/d/e
sudo touch /srv/export/a/b/c/d/e/file

for M in /mnt/nfs3 /mnt/nfs4; do
  sudo umount $M && sudo mount -t nfs -o vers=$( [ $M = /mnt/nfs3 ] && echo 3 || echo 4.2 ) \
       127.0.0.1:/srv/export $M
  echo "=== $M ==="
  sudo nfsstat -c -z > /dev/null 2>&1    # zero the counters
  sudo stat $M/a/b/c/d/e/file > /dev/null
  sudo nfsstat -c | head -20
done
```

Packet-level:

```sh
sudo tcpdump -i lo -w /tmp/nfs4.pcap port 2049 &
TCPD=$!
sudo umount /mnt/nfs4 && sudo mount -t nfs -o vers=4.2 127.0.0.1:/srv/export /mnt/nfs4
sudo stat /mnt/nfs4/a/b/c/d/e/file > /dev/null
sleep 1; sudo kill $TCPD

sudo tcpdump -r /tmp/nfs4.pcap -A 2>/dev/null | grep -c 'PUTFH\|LOOKUP' 
# Or open in Wireshark: one COMPOUND containing several LOOKUPs.
```

Per-operation latency:

```sh
sudo apt install -y nfs-common
nfsiostat 1 5 /mnt/nfs4
mountstats --nfs /mnt/nfs4 | head -40
```

Tracing:

```sh
sudo trace-cmd record -e nfs4:\* -e sunrpc:rpc_task_begin -e sunrpc:rpc_task_end -- \
  sudo sh -c 'echo test > /mnt/nfs4/x; cat /mnt/nfs4/x'
sudo trace-cmd report | head -40
```

---

### Lab 62.2 — Close-to-open, demonstrated and broken

```sh
# Two "clients" via two mounts of the same export
sudo mkdir -p /mnt/c1 /mnt/c2
sudo mount -t nfs -o vers=4.2 127.0.0.1:/srv/export /mnt/c1
sudo mount -t nfs -o vers=4.2 127.0.0.1:/srv/export /mnt/c2
```

CTO works:

```sh
sudo sh -c 'echo "version 1" > /mnt/c1/shared'     # write + close
sudo cat /mnt/c2/shared                            # open -> revalidate
# "version 1"

sudo sh -c 'echo "version 2" > /mnt/c1/shared'
sudo cat /mnt/c2/shared
# "version 2"  -- CTO delivered
```

Now break it — concurrent access, which CTO does not cover:

```c
// SPDX-License-Identifier: GPL-2.0
/* cto.c: hold a file open on one mount while another writes it. */
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

int main(int argc, char **argv)
{
	int fd = open(argv[1], O_RDONLY);
	char buf[256];
	int i;

	for (i = 0; i < 20; i++) {
		ssize_t n = pread(fd, buf, sizeof(buf) - 1, 0);

		if (n > 0) { buf[n] = 0; printf("%2d: %s", i, buf); }
		fflush(stdout);
		sleep(1);
	}
	close(fd);
	return 0;
}
```

```sh
gcc -O2 -o cto cto.c
sudo sh -c 'echo "initial" > /mnt/c1/live'
sudo ./cto /mnt/c2/live &        # holds it OPEN
sleep 2
for i in 1 2 3 4 5; do
  sudo sh -c "echo 'update $i' > /mnt/c1/live"
  sleep 2
done
wait
```

The reader may see stale data for `acregmax` seconds or longer. **This is correct CTO behaviour.**

Tune the caching and observe:

```sh
for opts in "actimeo=0" "acregmin=1,acregmax=1" "" "nocto"; do
  sudo umount /mnt/c2
  sudo mount -t nfs -o vers=4.2,${opts:-defaults} 127.0.0.1:/srv/export /mnt/c2
  echo "=== ${opts:-default} ==="
  sudo sh -c 'echo A > /mnt/c1/t'
  sudo cat /mnt/c2/t
  sudo sh -c 'echo B > /mnt/c1/t'
  sudo cat /mnt/c2/t        # does it see B immediately?
  # And the cost:
  echo -n "  1000 stats: "
  /usr/bin/time -f '%e s' sudo sh -c 'for i in $(seq 1 1000); do stat /mnt/c2/t > /dev/null; done' 2>&1 | tail -1
done
```

`actimeo=0` is correct and slow; the default is fast and eventually correct. **That is the dilemma of §T.2, priced.**

The shared-log failure:

```c
	/* Two clients appending to the same file over NFS. */
	int fd = open(path, O_WRONLY | O_APPEND);
	for (i = 0; i < 1000; i++)
		write(fd, line, len);     /* O_APPEND is CLIENT-side */
```

```sh
sudo sh -c '> /mnt/c1/log'
( sudo sh -c 'for i in $(seq 1 500); do echo "client1 line $i" >> /mnt/c1/log; done' ) &
( sudo sh -c 'for i in $(seq 1 500); do echo "client2 line $i" >> /mnt/c2/log; done' ) &
wait
wc -l /srv/export/log        # expect 1000; you will get fewer
grep -c client1 /srv/export/log
grep -c client2 /srv/export/log
```

Lines are lost. **`O_APPEND` does not work across NFS clients.**

---

### Lab 62.3 — `ESTALE` and file handles

```sh
sudo mount -t nfs -o vers=4.2 127.0.0.1:/srv/export /mnt/nfs

# Hold a handle while the file is deleted server-side
sudo sh -c 'echo data > /srv/export/victim'
exec 9< /mnt/nfs/victim
sudo rm /srv/export/victim
sudo sh -c 'echo other > /srv/export/other'     # may reuse the inode
sudo cat <&9 2>&1                                # ESTALE (or the old data, cached)
exec 9<&-
```

Force the generation-number check:

```sh
# Fill and reuse inodes aggressively
sudo sh -c 'for i in $(seq 1 1000); do echo x > /srv/export/f$i; done'
sudo sh -c 'echo target > /srv/export/target'
INO=$(stat -c %i /srv/export/target)
exec 9< /mnt/nfs/target
sudo rm /srv/export/target
sudo sh -c 'for i in $(seq 1 1000); do echo y > /srv/export/g$i; done'
# Find whether the inode was reused:
sudo find /srv/export -inum $INO
sudo cat <&9 2>&1 | head -1      # ESTALE, not the new file's contents
exec 9<&-
```

**Without the generation number this would return another file's data.** Ch. 56 §T.8(b), over a network.

`subtree_check`, the wrong default:

```sh
echo "/srv/export 127.0.0.1(rw,sync,subtree_check,fsid=43)" | sudo tee /etc/exports
sudo exportfs -ra
sudo umount /mnt/nfs && sudo mount -t nfs 127.0.0.1:/srv/export /mnt/nfs

sudo mkdir -p /srv/export/d1 /srv/export/d2
sudo sh -c 'echo moving > /srv/export/d1/file'
exec 9< /mnt/nfs/d1/file
sudo mv /srv/export/d1/file /srv/export/d2/file
sudo cat <&9 2>&1                # ESTALE with subtree_check
exec 9<&-

# Restore the sane setting
echo "/srv/export 127.0.0.1(rw,sync,no_subtree_check,fsid=42)" | sudo tee /etc/exports
sudo exportfs -ra
```

`fsid` stability:

```sh
# Change the fsid and watch every handle go stale
sudo umount /mnt/nfs
echo "/srv/export 127.0.0.1(rw,sync,no_subtree_check,fsid=99)" | sudo tee /etc/exports
sudo exportfs -ra
sudo mount -t nfs 127.0.0.1:/srv/export /mnt/nfs
ls /mnt/nfs                       # works: new handles
# But a client that had cached handles from fsid=42 gets ESTALE.
```

---

### Lab 62.4 — Delegations and state (NFSv4)

```sh
sudo umount /mnt/nfs 2>/dev/null
sudo mount -t nfs -o vers=4.2 127.0.0.1:/srv/export /mnt/nfs
cat /proc/fs/nfsd/versions
ls /proc/fs/nfsd/
```

Watch a delegation being granted and recalled:

```sh
sudo trace-cmd record -e nfsd:\* -e nfs4:\* -o /tmp/deleg.dat -- sh -c '
  # Client 1 opens a file exclusively -> delegation granted
  sudo sh -c "echo content > /srv/export/deleg"
  exec 8< /mnt/nfs/deleg
  cat <&8 > /dev/null
  sleep 1
  # Client 2 writes it -> the delegation must be RECALLED
  sudo sh -c "echo changed > /srv/export/deleg"
  sleep 1
  exec 8<&-
'
sudo trace-cmd report -i /tmp/deleg.dat | grep -iE 'deleg|recall' | head -20
```

Server-side state:

```sh
sudo cat /proc/fs/nfsd/clients/*/info 2>/dev/null
sudo cat /proc/fs/nfsd/clients/*/states 2>/dev/null | head -20
```

The grace period after a server restart:

```sh
sudo systemctl restart nfs-server
dmesg | tail -5 | grep -i grace
cat /proc/fs/nfsd/nfsv4gracetime
cat /proc/fs/nfsd/nfsv4leasetime

# During the grace period, NEW locks are refused but reclaims succeed
sudo systemctl restart nfs-server &
sleep 0.5
sudo flock -w 1 /mnt/nfs/lockfile -c 'echo got the lock' 2>&1
# May fail with "Resource temporarily unavailable" during grace.
sleep 45
sudo flock -w 1 /mnt/nfs/lockfile -c 'echo got the lock'
```

Locking, v3 versus v4:

```sh
for v in 3 4.2; do
  sudo umount /mnt/nfs
  sudo mount -t nfs -o vers=$v 127.0.0.1:/srv/export /mnt/nfs
  echo "=== v$v ==="
  ss -tlnp | grep -E '2049|lockd|statd' | head
  sudo rpcinfo -p 2>/dev/null | grep -iE 'nlock|status|nfs|mount'
done
# v3 needs several ports; v4 needs only 2049 (T.5).
```

---

### Lab 62.5 — Overlayfs: copy-up, whiteouts, and their costs

```sh
sudo mkdir -p /tmp/ovl/{lower,upper,work,merged}
sudo sh -c 'echo "from lower" > /tmp/ovl/lower/a'
sudo sh -c 'echo "also lower" > /tmp/ovl/lower/b'
sudo mkdir -p /tmp/ovl/lower/dir
sudo sh -c 'echo "in dir" > /tmp/ovl/lower/dir/c'
sudo dd if=/dev/zero of=/tmp/ovl/lower/big bs=1M count=200 2>/dev/null

sudo mount -t overlay overlay \
  -o lowerdir=/tmp/ovl/lower,upperdir=/tmp/ovl/upper,workdir=/tmp/ovl/work \
  /tmp/ovl/merged

ls -l /tmp/ovl/merged
ls -l /tmp/ovl/upper          # empty: nothing copied yet
```

Copy-up:

```sh
sudo cat /tmp/ovl/merged/a     # read: NO copy-up
ls -l /tmp/ovl/upper           # still empty

sudo sh -c 'echo "modified" >> /tmp/ovl/merged/a'   # write: copy-up
ls -l /tmp/ovl/upper           # `a` is now here
cat /tmp/ovl/lower/a           # unchanged
cat /tmp/ovl/merged/a          # the merged view
```

**The whole-file cost** (§T.6(b)):

```sh
sudo trace-cmd record -e overlayfs:\* -- \
  sudo sh -c 'echo x >> /tmp/ovl/merged/big'
sudo trace-cmd report | head -10

ls -lh /tmp/ovl/upper/big      # 200 MB copied for ONE BYTE
```

Time it:

```sh
sudo umount /tmp/ovl/merged
sudo rm -rf /tmp/ovl/upper/* /tmp/ovl/work/*
sudo mount -t overlay overlay \
  -o lowerdir=/tmp/ovl/lower,upperdir=/tmp/ovl/upper,workdir=/tmp/ovl/work \
  /tmp/ovl/merged
echo -n "copy-up of 200MB for 1 byte: "
sudo /usr/bin/time -f '%e s' sh -c 'echo x >> /tmp/ovl/merged/big' 2>&1 | tail -1
```

Now with reflink (XFS or btrfs as upper and lower):

```sh
sudo umount /tmp/ovl/merged
sudo modprobe scsi_debug dev_size_mb=2048
DEV=$(lsblk -ndo NAME,MODEL | awk '/scsi_debug/{print "/dev/"$1}')
sudo mkfs.xfs -f -m reflink=1 $DEV
sudo mkdir -p /mnt/ovlfs && sudo mount $DEV /mnt/ovlfs
sudo mkdir -p /mnt/ovlfs/{lower,upper,work,merged}
sudo dd if=/dev/zero of=/mnt/ovlfs/lower/big bs=1M count=200 2>/dev/null
sudo mount -t overlay overlay \
  -o lowerdir=/mnt/ovlfs/lower,upperdir=/mnt/ovlfs/upper,workdir=/mnt/ovlfs/work \
  /mnt/ovlfs/merged
echo -n "copy-up with reflink: "
sudo /usr/bin/time -f '%e s' sh -c 'echo x >> /mnt/ovlfs/merged/big' 2>&1 | tail -1
df -h /mnt/ovlfs               # space barely changed
```

**This is why container storage on reflink-capable filesystems matters.**

Whiteouts (§T.6(a)):

```sh
sudo rm /tmp/ovl/merged/b
ls /tmp/ovl/merged             # `b` is gone
ls /tmp/ovl/lower              # `b` is still there
sudo ls -l /tmp/ovl/upper/b
# crw-r--r-- 1 root root 0, 0 ... b        <- T.6(a): a 0/0 char device
stat -c '%F %t,%T' /tmp/ovl/upper/b
```

Opaque directories:

```sh
sudo rm -rf /tmp/ovl/merged/dir
sudo mkdir /tmp/ovl/merged/dir
sudo sh -c 'echo "new" > /tmp/ovl/merged/dir/new'
ls /tmp/ovl/merged/dir         # only `new`, not the lower `c`
sudo getfattr -d -m - /tmp/ovl/upper/dir
# trusted.overlay.opaque="y"
```

`metacopy` (§T.6(b)):

```sh
sudo umount /tmp/ovl/merged
sudo rm -rf /tmp/ovl/upper/* /tmp/ovl/work/*
sudo mount -t overlay overlay \
  -o lowerdir=/tmp/ovl/lower,upperdir=/tmp/ovl/upper,workdir=/tmp/ovl/work,metacopy=on \
  /tmp/ovl/merged

sudo chmod 600 /tmp/ovl/merged/big     # metadata only
ls -l /tmp/ovl/upper/big               # size 0 or small!
sudo getfattr -d -m - /tmp/ovl/upper/big
# trusted.overlay.metacopy  and  trusted.overlay.redirect
du -sh /tmp/ovl/upper                  # tiny
```

Inode numbers and `xino` (§T.6(c)):

```sh
sudo umount /tmp/ovl/merged
sudo mount -t overlay overlay \
  -o lowerdir=/tmp/ovl/lower,upperdir=/tmp/ovl/upper,workdir=/tmp/ovl/work,xino=off \
  /tmp/ovl/merged
stat -c '%i %n' /tmp/ovl/merged/* 2>/dev/null | sort -n | head
# Possible collisions between layers.

sudo umount /tmp/ovl/merged
sudo mount -t overlay overlay \
  -o lowerdir=/tmp/ovl/lower,upperdir=/tmp/ovl/upper,workdir=/tmp/ovl/work,xino=on \
  /tmp/ovl/merged
stat -c '%i %n' /tmp/ovl/merged/* 2>/dev/null | sort -n | head
# High bits encode the layer: unique.
```

Multi-layer, as containers use it:

```sh
sudo mkdir -p /tmp/ovl/l{1,2,3}
sudo sh -c 'echo "layer1" > /tmp/ovl/l1/f; echo "l1 only" > /tmp/ovl/l1/x'
sudo sh -c 'echo "layer2" > /tmp/ovl/l2/f; echo "l2 only" > /tmp/ovl/l2/y'
sudo sh -c 'echo "layer3" > /tmp/ovl/l3/f'
sudo umount /tmp/ovl/merged
sudo rm -rf /tmp/ovl/upper/* /tmp/ovl/work/*
sudo mount -t overlay overlay \
  -o lowerdir=/tmp/ovl/l3:/tmp/ovl/l2:/tmp/ovl/l1,upperdir=/tmp/ovl/upper,workdir=/tmp/ovl/work \
  /tmp/ovl/merged
cat /tmp/ovl/merged/f          # "layer3": leftmost wins
ls /tmp/ovl/merged             # f, x, y: directories merge
```

See the real thing:

```sh
docker run --rm -d --name t alpine sleep 300 2>/dev/null && \
  mount | grep overlay | tr ',' '\n' | head -20
docker stop t 2>/dev/null
```

---

### Lab 62.6 — Write a FUSE filesystem, then measure the protocol cost

```c
// SPDX-License-Identifier: GPL-2.0
/* hellofs.c -- a minimal FUSE filesystem. */
#define FUSE_USE_VERSION 31
#include <errno.h>
#include <fuse3/fuse.h>
#include <stdio.h>
#include <string.h>
#include <time.h>

static const char *content = "hello from userspace\n";
static long lookups, getattrs, reads, opens;

static int hf_getattr(const char *path, struct stat *st,
		      struct fuse_file_info *fi)
{
	__atomic_fetch_add(&getattrs, 1, __ATOMIC_RELAXED);
	memset(st, 0, sizeof(*st));
	if (!strcmp(path, "/")) {
		st->st_mode = S_IFDIR | 0755;
		st->st_nlink = 2;
	} else if (!strcmp(path, "/hello")) {
		st->st_mode = S_IFREG | 0444;
		st->st_nlink = 1;
		st->st_size = strlen(content);
	} else if (!strcmp(path, "/stats")) {
		st->st_mode = S_IFREG | 0444;
		st->st_nlink = 1;
		st->st_size = 256;
	} else {
		return -ENOENT;
	}
	return 0;
}

static int hf_readdir(const char *path, void *buf, fuse_fill_dir_t filler,
		      off_t off, struct fuse_file_info *fi,
		      enum fuse_readdir_flags flags)
{
	if (strcmp(path, "/"))
		return -ENOENT;
	filler(buf, ".", NULL, 0, 0);
	filler(buf, "..", NULL, 0, 0);
	filler(buf, "hello", NULL, 0, 0);
	filler(buf, "stats", NULL, 0, 0);
	return 0;
}

static int hf_open(const char *path, struct fuse_file_info *fi)
{
	__atomic_fetch_add(&opens, 1, __ATOMIC_RELAXED);
	if (strcmp(path, "/hello") && strcmp(path, "/stats"))
		return -ENOENT;
	if ((fi->flags & O_ACCMODE) != O_RDONLY)
		return -EACCES;
	return 0;
}

static int hf_read(const char *path, char *buf, size_t size, off_t off,
		   struct fuse_file_info *fi)
{
	char tmp[256];
	const char *src;
	size_t len;

	__atomic_fetch_add(&reads, 1, __ATOMIC_RELAXED);

	if (!strcmp(path, "/stats")) {
		snprintf(tmp, sizeof(tmp),
			 "getattr=%ld open=%ld read=%ld\n",
			 getattrs, opens, reads);
		src = tmp;
	} else {
		src = content;
	}
	len = strlen(src);
	if (off >= (off_t)len)
		return 0;
	if (off + size > len)
		size = len - off;
	memcpy(buf, src + off, size);
	return size;
}

static const struct fuse_operations hf_ops = {
	.getattr = hf_getattr,
	.readdir = hf_readdir,
	.open    = hf_open,
	.read    = hf_read,
};

int main(int argc, char *argv[])
{
	return fuse_main(argc, argv, &hf_ops, NULL);
}
```

```sh
sudo apt install -y libfuse3-dev fuse3
gcc -O2 -o hellofs hellofs.c $(pkg-config fuse3 --cflags --libs)
mkdir -p /tmp/fusemnt
./hellofs -f /tmp/fusemnt &        # -f: foreground, so you can see it
sleep 1

cat /tmp/fusemnt/hello
ls -l /tmp/fusemnt
cat /tmp/fusemnt/stats
cat /tmp/fusemnt/stats             # the counters advance
```

Watch the protocol:

```sh
sudo trace-cmd record -e fuse:\* -- cat /tmp/fusemnt/hello
sudo trace-cmd report | head -20
```

Measure the cost:

```c
// SPDX-License-Identifier: GPL-2.0
/* statbench.c */
#define _GNU_SOURCE
#include <stdio.h>
#include <sys/stat.h>
#include <time.h>
int main(int argc, char **argv) {
	struct stat st; struct timespec a, b; int i, n = 100000;
	stat(argv[1], &st);
	clock_gettime(CLOCK_MONOTONIC, &a);
	for (i = 0; i < n; i++) stat(argv[1], &st);
	clock_gettime(CLOCK_MONOTONIC, &b);
	printf("%-30s %.2f us/stat\n", argv[1],
	       ((b.tv_sec-a.tv_sec)*1e9 + (b.tv_nsec-a.tv_nsec)) / n / 1000);
	return 0;
}
```

```sh
gcc -O2 -o statbench statbench.c
./statbench /tmp/fusemnt/hello
./statbench /etc/hostname
./statbench /mnt/nfs/a
```

Typical: local ~0.5 µs, FUSE ~15 µs, NFS ~40 µs. **That 30× is the protocol, not the code.**

Tune the caching:

```sh
kill %1; sleep 1
./hellofs -f -o entry_timeout=60,attr_timeout=60 /tmp/fusemnt &
sleep 1
./statbench /tmp/fusemnt/hello     # much faster now: the kernel caches
sudo trace-cmd record -e fuse:\* -- ./statbench /tmp/fusemnt/hello
sudo trace-cmd report | wc -l       # far fewer requests
```

Connection state and the escape hatch:

```sh
ls /sys/fs/fuse/connections/
for c in /sys/fs/fuse/connections/*/; do
  echo "=== $c ==="
  cat $c/waiting $c/max_background $c/congestion_threshold 2>/dev/null
done

# Simulate a hung daemon
kill -STOP %1
timeout 5 cat /tmp/fusemnt/hello   # hangs
CONN=$(ls /sys/fs/fuse/connections/ | head -1)
sudo cat /sys/fs/fuse/connections/$CONN/waiting   # pending requests
# Free the stuck processes:
echo 1 | sudo tee /sys/fs/fuse/connections/$CONN/abort
kill -CONT %1
fusermount3 -u /tmp/fusemnt
```

Throughput with a real FUSE filesystem:

```sh
sudo apt install -y sshfs bindfs
mkdir -p /tmp/bindsrc /tmp/bindmnt
dd if=/dev/urandom of=/tmp/bindsrc/data bs=1M count=500 2>/dev/null
bindfs /tmp/bindsrc /tmp/bindmnt

for p in /tmp/bindsrc/data /tmp/bindmnt/data; do
  echo 3 | sudo tee /proc/sys/vm/drop_caches > /dev/null
  echo -n "$p: "
  dd if=$p of=/dev/null bs=1M 2>&1 | tail -1
done

# With larger requests
fusermount3 -u /tmp/bindmnt
bindfs -o big_writes,max_read=1048576 /tmp/bindsrc /tmp/bindmnt 2>/dev/null || \
  bindfs /tmp/bindsrc /tmp/bindmnt
echo 3 | sudo tee /proc/sys/vm/drop_caches > /dev/null
dd if=/tmp/bindmnt/data of=/dev/null bs=1M 2>&1 | tail -1
fusermount3 -u /tmp/bindmnt
```

---

### Lab 62.7 — SMB and ksmbd

```sh
sudo apt install -y samba cifs-utils
sudo mkdir -p /srv/smb && sudo chmod 777 /srv/smb

sudo tee -a /etc/samba/smb.conf > /dev/null <<'EOF'
[testshare]
   path = /srv/smb
   browseable = yes
   read only = no
   guest ok = yes
   force user = nobody
EOF
sudo systemctl restart smbd
smbclient -L localhost -N 2>/dev/null | head

sudo mkdir -p /mnt/smb
sudo mount -t cifs //127.0.0.1/testshare /mnt/smb \
  -o guest,vers=3.1.1,uid=$(id -u),gid=$(id -g)
mount | grep cifs

echo "smb test" > /mnt/smb/f
cat /srv/smb/f
```

SMB3 features NFS lacks:

```sh
# Server-side copy (copy_file_range offloaded to the server)
dd if=/dev/urandom of=/mnt/smb/src bs=1M count=100 2>/dev/null
sudo trace-cmd record -e block:block_rq_issue -e cifs:\* -- \
  cp --reflink=auto /mnt/smb/src /mnt/smb/dst 2>/dev/null
sudo trace-cmd report | grep -ci 'copychunk\|duplicate_extents'

# Leases (SMB's delegations)
mount | grep cifs
cat /proc/fs/cifs/Stats 2>/dev/null | head -20
cat /proc/fs/cifs/DebugData 2>/dev/null | head -30
```

ksmbd, the in-kernel server:

```sh
sudo apt install -y ksmbd-tools 2>/dev/null
sudo modprobe ksmbd 2>/dev/null && lsmod | grep ksmbd
# ksmbd exists because userspace Samba could not reach line rate for
# SMB3 multichannel and RDMA. The tradeoff: a network server in the kernel.
```

Compare protocols:

```sh
for M in /mnt/nfs4 /mnt/smb; do
  [ -d $M ] || continue
  echo "=== $M ==="
  ./statbench $M/f 2>/dev/null
  echo -n "  create 500 files: "
  sudo /usr/bin/time -f '%e s' sh -c "for i in \$(seq 1 500); do : > $M/c\$i; done" 2>&1 | tail -1
  sudo rm -f $M/c* 2>/dev/null
done
```

---

### Lab 62.8 — Failure modes

Network partition:

```sh
sudo mount -t nfs -o vers=4.2,hard 127.0.0.1:/srv/export /mnt/nfs
cat /mnt/nfs/a

# Block the traffic
sudo iptables -I INPUT -p tcp --dport 2049 -j DROP
timeout 20 cat /mnt/nfs/a        # hangs (hard mount)
cat /proc/$(pgrep -f 'cat /mnt/nfs')/stack 2>/dev/null | head
dmesg | tail -3                  # "server not responding"
sudo iptables -D INPUT -p tcp --dport 2049 -j DROP
dmesg | tail -2                  # "server OK"
```

`hard` versus `soft`:

```sh
for opt in hard soft; do
  sudo umount -f /mnt/nfs 2>/dev/null
  sudo mount -t nfs -o vers=4.2,$opt,timeo=20,retrans=2 127.0.0.1:/srv/export /mnt/nfs
  sudo iptables -I INPUT -p tcp --dport 2049 -j DROP
  echo -n "$opt: "
  timeout 30 cat /mnt/nfs/a 2>&1 | head -1
  sudo iptables -D INPUT -p tcp --dport 2049 -j DROP
done
```

**`soft` can return `EIO` on a write that actually succeeded.** That is why `hard` is the default and why `soft` should be used only for read-only mounts where hanging is worse than wrong answers.

`softreval` and `nconnect`:

```sh
sudo umount /mnt/nfs
sudo mount -t nfs -o vers=4.2,nconnect=4 127.0.0.1:/srv/export /mnt/nfs
ss -tn | grep 2049 | wc -l       # 4 connections
mount | grep nfs
```

Overlayfs: modifying the lower layer while mounted (unsupported, and here is why):

```sh
sudo mount -t overlay overlay \
  -o lowerdir=/tmp/ovl/lower,upperdir=/tmp/ovl/upper,workdir=/tmp/ovl/work \
  /tmp/ovl/merged
cat /tmp/ovl/merged/a
sudo sh -c 'echo "changed behind your back" > /tmp/ovl/lower/a'
cat /tmp/ovl/merged/a            # may be the OLD content: incoherent
```

FUSE deadlock potential:

```sh
# A daemon that allocates in its request path under memory pressure
# is a latent deadlock (T.7). The mitigation:
grep -r PR_SET_IO_FLUSHER /usr/include/linux/prctl.h
# A well-behaved daemon calls:
#   prctl(PR_SET_IO_FLUSHER, 1, 0, 0, 0);
#   mlockall(MCL_CURRENT | MCL_FUTURE);
```

Clean up:

```sh
sudo umount /tmp/ovl/merged /mnt/nfs /mnt/nfs3 /mnt/nfs4 /mnt/c1 /mnt/c2 /mnt/smb 2>/dev/null
fusermount3 -u /tmp/fusemnt 2>/dev/null
sudo exportfs -ua
```

---

## 3. Mastery drills

1. §T.1 lists eight broken assumptions. For each, name the specific NFS mechanism that copes with it, or state that none does.

2. Prove that "the operation failed" and "the operation succeeded and the reply was lost" are indistinguishable over an unreliable network. Then show how the DRC narrows the gap and why it cannot close it.

3. NFS `WRITE` takes an explicit offset. Show that an implicit-position `WRITE` cannot be made idempotent, and name two other protocol decisions with the same cause.

4. State close-to-open precisely as a pre/post-condition pair. Then construct three programs that are correct locally and incorrect under CTO.

5. The adaptive `attrtimeo` doubles on unchanged revalidation. Derive the expected number of `GETATTR` calls for a file modified every N seconds and accessed every M seconds.

6. A file handle must satisfy four properties (§T.4). For each, construct the failure that occurs if it does not, and name the field in the Linux handle that provides it.

7. `subtree_check` breaks property 2. Explain what it was trying to achieve, and design an alternative that achieves it without path dependence.

8. NFSv3's statelessness has three consequences. For each, explain how v4 fixes it and what the fix costs.

9. Delegations require a callback channel. Explain why v4.0's separate connection was a problem and how sessions solve it.

10. Overlayfs copy-up is whole-file. Explain why partial copy-up would require metadata upper cannot hold, then design the metadata and say what it would break.

11. A whiteout is a 0/0 char device. Enumerate every alternative encoding and state why each fails.

12. Derive FUSE's minimum per-operation cost from first principles: count context switches, copies, and scheduler interactions. Then explain precisely what `FUSE_PASSTHROUGH` eliminates.

13. Construct the FUSE reclaim deadlock as a precise sequence, then state all three mitigations and what each assumes.

---

## 4. Further reading

**Kernel documentation**

- `Documentation/filesystems/overlayfs.rst` ★★★ — **authoritative and unusually complete.** Covers whiteouts, opaque dirs, metacopy, xino, `redirect_dir`, and the "do not modify lower" rule. Read it entirely.
- `Documentation/filesystems/nfs/exporting.rst` ★★★ — §T.4 and Ch. 56 §T.8.
- `Documentation/filesystems/nfs/client-identifier.rst`, `rpc-cache.rst`, `idmapper.rst`
- `Documentation/filesystems/fuse.rst` ★★★ and `fuse-io.rst`, `fuse-passthrough.rst`
- `Documentation/filesystems/cifs/` and the ksmbd documentation
- `man 5 nfs` ★★★ — **every mount option, precisely documented.** One of the best man pages in Linux.
- `man 5 exports` ★★★, `man 5 nfsd`
- `man 8 mount.cifs`, `man 8 fusermount3`

**RFCs and specifications**

- RFC 1813 — NFSv3 ★★★ (short, readable, and the idempotence reasoning is visible throughout)
- RFC 7530 — NFSv4.0
- RFC 8881 — **NFSv4.1** ★★★ (sessions, EOS, pNFS; the version that matters)
- RFC 7862 — NFSv4.2 (server-side copy, sparse files, labelled NFS)
- MS-SMB2 — the SMB3 specification (Microsoft publishes it; it is thorough)

**Papers**

- Sandberg, Goldberg, Kleiman, Walsh, Lyon, "Design and Implementation of the Sun Network Filesystem," USENIX 1985 ★★★ — **the original.** The statelessness argument in its own words; read it knowing how the story ends.
- Howard et al., "Scale and Performance in a Distributed File System," ACM TOCS 1988 ★★★ — **AFS.** Whole-file caching and callbacks; the road not taken, and the origin of the delegation idea.
- Kistler & Satyanarayanan, "Disconnected Operation in the Coda File System," SOSP 1991 ★★★ — what if the network is *expected* to fail?
- Pawlowski et al., "The NFS Version 4 Protocol," 2000 — the v4 design rationale.
- Gibson et al., "A Cost-Effective, High-Bandwidth Storage Architecture," ASPLOS 1998 — the pNFS ancestry.
- Vangoor, Tarasov, Zadok, "To FUSE or Not to FUSE: Performance of User-Space File Systems," FAST 2017 ★★★ — **§T.7 measured rigorously.** Quantifies FUSE's overhead across 45 workloads. Essential if you are considering FUSE for anything performance-sensitive.
- Zadok & Nieh, "FiST: A Language for Stackable File Systems," USENIX 2000 — §T.8's problems, generally.

**LWN**

- "Unioning file systems" series and the overlayfs merge coverage ★★★
- "Overlayfs and the return of the union mount" / "Overlayfs issues"
- "Filesystems in user space" and the FUSE coverage
- "FUSE passthrough mode" (2023–2024) ★★★
- "Virtio-fs: a shared file system for virtual machines" ★★★
- "The trouble with FUSE" and the FUSE deadlock threads
- "NFS: the new version" and the v4.1/v4.2 coverage
- "ksmbd: a kernel SMB server" ★★★ — including the security debate about network servers in the kernel
- "Unprivileged filesystem mounts" ★★★ — §T.7's security discussion
- "Idmapped mounts" — relevant to all of these

**Source reading order**

1. `Documentation/filesystems/overlayfs.rst` ★★★, then `fs/overlayfs/copy_up.c` and `dir.c`.
2. `include/uapi/linux/fuse.h` ★★★ — the whole protocol is here, and it is short.
3. `fs/fuse/dev.c` — `fuse_dev_do_read`, `fuse_dev_do_write`, `request_wait_answer`.
4. `fs/fuse/file.c` — the caching decisions.
5. RFC 1813, then `fs/nfs/inode.c`: `nfs_revalidate_inode`, `nfs_update_inode`, the attrtimeo logic.
6. `fs/nfsd/nfsfh.c` ★★★ — §T.4.
7. RFC 8881 §2 (sessions), then `fs/nfs/nfs4state.c` and `fs/nfsd/nfs4state.c`.
8. `net/sunrpc/` — the RPC layer, if you need to debug transport issues.

**Tools**

- `nfsstat -c -s` ★★★, `nfsiostat`, `mountstats` ★★★
- `/proc/self/mountstats` — the raw per-operation latency data
- `trace-cmd record -e nfs:\* -e nfs4:\* -e sunrpc:\*` ★★★ — far better than `rpcdebug`
- `tcpdump port 2049` + **Wireshark** ★★★ — Wireshark's NFS dissector is excellent; use it
- `exportfs -v`, `showmount -e`, `/proc/fs/nfsd/*`
- `/sys/fs/fuse/connections/*/` ★★★ — `waiting`, `abort`, `max_background`
- `trace-cmd record -e fuse:\*`, `strace -f` on the daemon
- `getfattr -d -m -` ★★★ for overlayfs xattrs
- `smbstatus`, `smbclient`, `/proc/fs/cifs/{Stats,DebugData}`
- `bindfs`, `sshfs`, `rclone mount` — FUSE filesystems to experiment with
- `unshare -m` and `nsenter` — for testing mount-namespace interactions

---

→ Next: [63-block-layer-1.md](63-block-layer-1.md)

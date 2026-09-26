# Chapter 29 — Character Devices from Scratch

> **Goal:** build a complete, correct character driver — `open`/`read`/`write`/`ioctl`/
> `poll`/`mmap`/`llseek` — with correct lifetime, correct concurrency, and correct blocking
> semantics. This is the chapter where Parts 0 and 1 become a real driver.

---

## Theory & First Principles

### T.0 — Start here: `cat /dev/urandom` — what makes that work?

```bash
ls -l /dev/urandom /dev/null /dev/tty
# crw-rw-rw- 1 root root  1, 9 ... /dev/urandom
# crw-rw-rw- 1 root root  1, 3 ... /dev/null
# crw--w---- 1 root tty   5, 0 ... /dev/tty
#  ^                      ^  ^
#  c = CHARACTER device    major, minor
```

There is **no file** at `/dev/urandom`. There are no bytes on disk. What exists is an inode
with a type (`c`) and a pair of numbers, and those numbers are a key into a kernel table:

```
 open("/dev/urandom")
   │
   ├─ VFS resolves the path, finds an inode with S_IFCHR and dev = (1,9)
   │
   ├─ chrdev_open() looks up major 1 in the character-device table
   │     -> finds struct cdev -> its struct file_operations
   │
   ├─ REPLACES file->f_op with the driver's fops        ← the whole trick
   │
   └─ calls fops->open()

 read(fd, buf, 4096)
   └─ vfs_read() -> file->f_op->read_iter()   -> the DRIVER's function
```

**That pointer swap is the entire mechanism.** After `open`, the file descriptor's operations
table points at your driver, and every subsequent `read`/`write`/`mmap`/`ioctl`/`poll` is a
direct call into your code. A character device is *a driver wearing a file's clothes.*

**What the disguise buys you** is the point of the chapter, and it is a lot:

| You implement | You get for free |
|---|---|
| `read_iter`/`write_iter` | `cat`, `dd`, shell redirection, `read(2)` from any language |
| `poll` | `select`, `poll`, `epoll`, and every event loop ever written |
| `mmap` | zero-copy access from userspace |
| *(nothing)* | permissions, ownership, ACLs, SELinux labels — it is an inode |
| *(nothing)* | passing over `SCM_RIGHTS`, inheritance across `fork`, close-on-exec |
| *(nothing)* | refcounting, and automatic cleanup when the process dies |

That is the narrow-waist payoff from Ch. 00 §T.3b, cashed in. **You did not design a security
model, a permission system, an event-notification protocol, or a lifetime scheme** — you
inherited all four by being a file.

**Now the hard part, and it is genuinely hard.** Consider:

```
  t0  userspace: fd = open("/dev/mydev")
  t1  someone:   echo 0000:00:1f.6 > /sys/bus/pci/drivers/mydrv/unbind
  t2  the driver's remove() runs, frees everything
  t3  userspace: read(fd, ...)        <- calls into a driver that is GONE
```

**An open file descriptor outlives the device.** This is not an edge case — it is what
happens every time a USB device is unplugged while a program has it open, and it is the
single most common source of oopses in character drivers.

And note what it means for the previous chapter: **`devm_` is *wrong* here.** `devm_` ties
lifetime to the device, and the per-open state must outlive the device. The two lifetimes are
genuinely different and must be represented separately:

```
  DEVICE lifetime  ─────────────────┐          hardware resources: devm_ is right
                                     │
  SHARED object    ─────────────────┼───────────┐   refcounted (kref): outlives both
                                     │           │
  OPEN FILE        ─────────────────────────────┘   holds a reference
                                     ↑
                              unbind happens here; operations after it
                              must return -ENODEV, cleanly
```

**Getting this right is §T.6 and it is the real content of the chapter** — everything before
it is plumbing. It is also exactly the problem Rust's `Devres<T>` solves by returning an
`Option` from `try_access()` (Ch. 83 §T.4), which is worth noticing now so that it lands
later.

---

### T.1 "Everything is a file" — what the abstraction actually buys

Unix's central design decision is that devices are accessed through the **file** abstraction.
This is not just aesthetics; it is an *economic* argument about interface reuse:

| You get for free | Because it is a file |
|---|---|
| A namespace with permissions | filesystem paths, uid/gid/mode, ACLs, SELinux labels |
| A lifetime handle | `open()`/`close()`, refcounted by the VFS |
| Inheritance and passing | `fork()`, `exec` with `CLOEXEC`, `SCM_RIGHTS` over sockets |
| Multiplexing | `select`/`poll`/`epoll`/`io_uring` work on any fd |
| Redirection and composition | shell pipelines, `dd`, `cat`, `strace` |
| Async notification | `SIGIO`/`fasync`, `eventfd` integration |
| Memory mapping | `mmap()` |
| Auditing and sandboxing | seccomp, LSM hooks, `landlock` — all fd-aware |

**This is why Ch. 24 T.5's rule is "return an fd".** A new interface that is an fd
automatically inherits a decade of infrastructure; one that is not must reinvent every row of
that table.

The cost is that the file API is *narrow*: a byte stream with a position, plus an untyped
escape hatch (`ioctl`). Everything a device wants to say must be encoded into that. That
tension — a universal but impoverished interface — is the defining characteristic of
character-device design, and `ioctl`'s existence and its problems (Ch. 24 T.7) follow
directly from it.

### T.2 Device numbers: a two-level namespace with a historical shape

```
dev_t = 32 bits = major (12 bits) : minor (20 bits)
```

- **major** — historically "which driver", now "which registration".
- **minor** — historically "which instance", now entirely driver-defined.

```c
dev_t devt = MKDEV(major, minor);
unsigned int maj = MAJOR(devt), min = MINOR(devt);
```

This numbering is **ABI** (Ch. 24 T.1): `mknod` and static `/dev` layouts depend on it, and
`Documentation/admin-guide/devices.txt` is the historical registry. Three consequences:

1. **Never hardcode a major.** Use `alloc_chrdev_region()` for a dynamic allocation.
   Static majors are for legacy devices only and require registry coordination.
2. **The number is not the identity.** Userspace should find devices by *name* (via udev) or
   by *sysfs path*, not by number. `/dev/disk/by-uuid/` exists for this reason.
3. **devtmpfs + udev create the node**, not the kernel (Ch. 26 T.5). Your driver's job is to
   register a `dev_t` and let the uevent carry `MAJOR=`/`MINOR=`.

```bash
cat /proc/devices                       # allocated majors, char and block
ls -l /dev/ | head -20                  # the c/b prefix, major, minor
stat -c '%t:%T %n' /dev/null /dev/urandom
cat /sys/class/misc/*/dev
```

### T.3 The three objects, and why there are three

```
   struct file_operations   ← the VTABLE      (one per DRIVER, const, in .rodata)
   struct cdev              ← the REGISTRATION (one per device, owns a kobject)
   struct file              ← the OPEN INSTANCE (one per open(), per process)
      └── ->private_data     ← your per-open state
```

This separation is not arbitrary; each object has a distinct lifetime:

- `file_operations` lives as long as the **module**. It must be `const` and is shared.
- `cdev` lives as long as the **device is registered**. It is kobject-refcounted, which is
  what keeps the *device object* alive while an fd is open.
- `struct file` lives from `open()` to the last `close()` of that descriptor (accounting for
  `dup()` and `fork()` — hence `f_count`).

**`->private_data` is the hook for per-open state**, and getting it right is most of what
makes a char driver correct:

```c
static int my_open(struct inode *inode, struct file *file)
{
	/* Recover the device object from the inode's cdev. */
	struct my_dev *d = container_of(inode->i_cdev, struct my_dev, cdev);
	struct my_ctx *ctx;

	if (!my_dev_get(d))                     /* ★ take a reference (Ch. 12) */
		return -ENODEV;

	ctx = kzalloc(sizeof(*ctx), GFP_KERNEL);   /* ★ per-OPEN, not per-device */
	if (!ctx) {
		my_dev_put(d);
		return -ENOMEM;
	}
	ctx->dev = d;
	file->private_data = ctx;

	stream_open(inode, file);               /* if there is no meaningful position */
	return 0;
}

static int my_release(struct inode *inode, struct file *file)
{
	struct my_ctx *ctx = file->private_data;

	my_dev_put(ctx->dev);
	kfree(ctx);
	return 0;
}
```

**Per-device vs per-open state is a design decision you must make explicitly.** A serial port
has per-device state (baud rate) and per-open state (read buffer position). Conflating them
produces drivers that break when opened twice — a classic bug that only appears in
production.

`.owner = THIS_MODULE` in `file_operations` is what prevents `rmmod` while an fd is open
(Ch. 05 T.4): the VFS takes a module reference on `open()`. **Omitting it is a
use-after-free of module text.**

### T.4 Copying to and from userspace: the safety boundary

```c
copy_to_user(void __user *to, const void *from, unsigned long n);    /* returns BYTES NOT copied */
copy_from_user(void *to, const void __user *from, unsigned long n);
get_user(val, ptr);   put_user(val, ptr);                            /* return 0 or -EFAULT */
```

Three facts that trip people up:

1. **They return the number of bytes *not* copied**, not an errno. `if (copy_to_user(...))
   return -EFAULT;` is the idiom because non-zero means failure.
2. **They can sleep** (the user page may need to be faulted in). So they may not be called
   with a spinlock held, in interrupt context, or inside `rcu_read_lock()`.
3. **They can partially fail.** A partial copy is a real outcome (the user buffer spans a
   valid and an invalid page). Some APIs (`read`) legitimately return the partial count;
   others must treat it as total failure.

The `__user` annotation is enforced by sparse (Ch. 08 T.2) and dereferencing such a pointer
directly is a catastrophic bug — on a modern CPU, SMAP/PAN will fault, but the *logic* bug
(trusting user memory) remains.

**The TOCTOU rule (Ch. 24 T.4) applies absolutely**: copy once into kernel memory, validate
the kernel copy, use the kernel copy. Never re-read.

The modern preferred form for `read`/`write` is **iterators** (`read_iter`/`write_iter` with
`struct iov_iter`), which handle `readv`/`writev`/`preadv2`/`io_uring`/direct-I/O uniformly:

```c
static ssize_t my_read_iter(struct kiocb *iocb, struct iov_iter *to)
{
	struct my_ctx *ctx = iocb->ki_filp->private_data;
	size_t n = min(iov_iter_count(to), ctx->len);

	return copy_to_iter(ctx->buf, n, to);     /* handles all vectored forms */
}
```
Simple drivers may still use `.read`/`.write`; anything that may be used with `io_uring` or
`readv` should use the iter forms.

### T.5 Blocking, non-blocking, and `poll()`: one state machine, three views

A device that is not always ready must support three modes, and they must be *consistent*:

| Mode | Behaviour when not ready |
|---|---|
| Blocking (default) | sleep until ready, or until a signal arrives |
| `O_NONBLOCK` | return `-EAGAIN` immediately |
| `poll()`/`select()`/`epoll` | report not-ready; wake the poller when it becomes ready |

The *same* readiness predicate must drive all three, or you get the classic bugs: `poll()`
says readable but `read()` blocks; or `poll()` never fires because the wakeup path forgot the
wait queue.

```c
static ssize_t my_read(struct file *f, char __user *ubuf, size_t len, loff_t *off)
{
	struct my_ctx *ctx = f->private_data;
	struct my_dev *d = ctx->dev;
	int ret;

	/* ★ ONE predicate: data_available(d) */
	if (!data_available(d)) {
		if (f->f_flags & O_NONBLOCK)
			return -EAGAIN;
		ret = wait_event_interruptible(d->readq, data_available(d));
		if (ret)
			return ret;                /* -ERESTARTSYS: the VFS handles it */
	}

	guard(mutex)(&d->lock);
	return my_copy_out(d, ubuf, len);
}

static __poll_t my_poll(struct file *f, poll_table *wait)
{
	struct my_dev *d = ((struct my_ctx *)f->private_data)->dev;
	__poll_t mask = 0;

	poll_wait(f, &d->readq, wait);         /* ★ register; does NOT sleep */
	poll_wait(f, &d->writeq, wait);

	if (data_available(d))
		mask |= EPOLLIN  | EPOLLRDNORM;
	if (space_available(d))
		mask |= EPOLLOUT | EPOLLWRNORM;
	if (d->dead)
		mask |= EPOLLHUP;
	if (d->error)
		mask |= EPOLLERR;
	return mask;
}

/* The producer side — Ch. 25 P3: establish the condition, THEN wake. */
static void my_data_arrived(struct my_dev *d)
{
	kfifo_in(&d->fifo, buf, n);
	wake_up_interruptible(&d->readq);
	kill_fasync(&d->fasync, SIGIO, POLL_IN);   /* optional SIGIO support */
}
```

**`poll_wait()` does not sleep.** It registers the wait queue with the poll table and returns.
The *caller* (`do_poll`) sleeps once, on all registered queues. A `->poll` that sleeps is a
bug. And a `->poll` that fails to `poll_wait()` on a queue it reports readiness from will
hang forever — the lost-wakeup pattern (Ch. 25 P3) in fd clothing.

`EPOLLET` (edge-triggered) users depend on the wakeup being sent on every *transition*; a
driver that only wakes "sometimes" works with level-triggered `poll()` and hangs with
`epoll`+`EPOLLET`. Test both.

### T.6 `llseek`: the position semantics you must declare

Every `struct file` has `f_pos`. If your device has no meaningful position, saying so is
mandatory — because the default behaviour is to *allow* seeking, and a partially-implemented
position is worse than none:

```c
.llseek = no_llseek,        /* deprecated; returns -ESPIPE (now the default if unset) */
.llseek = noop_llseek,      /* accept seeks, ignore them (legacy compat) */
.llseek = default_llseek,   /* generic, uses i_size */
.llseek = generic_file_llseek,
.llseek = fixed_size_llseek,
```
Since 6.x, **leaving `.llseek` unset means "not seekable"** and `no_llseek` was removed
tree-wide. The corresponding `open()`-side declaration is:

```c
stream_open(inode, file);   /* ★ no position at all: f_pos is not used, and
                             * concurrent read/write from multiple threads is allowed */
nonseekable_open(inode, file);
```

`stream_open()` matters for concurrency: without it, the VFS serializes `read()`/`write()`
on `f_pos_lock` for shared file descriptors, which silently serializes a multi-threaded
application on your device. Pipes, ttys, and sockets are all stream-open. **If your device is
a stream, say so.**

### T.7 Concurrency in a char driver: the four questions, applied

Ch. 00 T.9's checklist, specialized:

1. **Two processes `open()` the same device.** Two `struct file`s, two `private_data`s, one
   device. Per-device state needs a lock.
2. **One process, two threads, one fd.** They share `struct file` *and* `private_data`. If
   your per-open state is mutable, it needs its own lock — a fact people consistently miss.
3. **`read()` racing with the IRQ handler.** Classic producer/consumer; use `kfifo` +
   `spin_lock_irqsave` or a lockless ring (Ch. 25 P6).
4. **`read()` racing with unbind/rmmod.** This is the lifetime problem (Ch. 28 T.3) and it is
   the one that produces UAFs. Solved by: `.owner = THIS_MODULE` for module text, a refcount
   on the device object for device memory, and a `dead` flag checked under the lock so
   in-flight operations fail cleanly with `-ENODEV`.

The complete removal protocol (Ch. 25 P12 for char devices):

```c
static void my_remove(struct platform_device *pdev)
{
	struct my_dev *d = platform_get_drvdata(pdev);

	/* 1. STOP: no new opens, no new hardware activity */
	cdev_del(&d->cdev);              /* removes from the cdev map; open() now fails */
	device_destroy(my_class, d->devt);
	my_hw_disable(d);

	/* 2. DRAIN: existing opens must stop and get out */
	scoped_guard(mutex, &d->lock)
		d->dead = true;
	wake_up_all(&d->readq);          /* ★ kick anyone blocked in wait_event */
	wake_up_all(&d->writeq);
	free_irq(d->irq, d);
	cancel_work_sync(&d->work);

	/* 3. FREE: the last fd close drops the last reference */
	my_dev_put(d);                   /* the DRIVER's reference; fds hold their own */
}
```

Every blocking wait must therefore include the death condition:

```c
ret = wait_event_interruptible(d->readq, data_available(d) || READ_ONCE(d->dead));
if (ret)
	return ret;
if (READ_ONCE(d->dead))
	return -ENODEV;
```
**A `wait_event` that does not check for teardown is an unkillable task after unbind.**

### T.8 `mmap`: handing the device's memory to userspace

```c
static int my_mmap(struct file *f, struct vm_area_struct *vma)
{
	struct my_dev *d = ((struct my_ctx *)f->private_data)->dev;
	unsigned long size = vma->vm_end - vma->vm_start;

	if (size > d->buf_size)
		return -EINVAL;
	if (vma->vm_pgoff)                       /* ★ validate the offset */
		return -EINVAL;

	/* For DMA-coherent memory: */
	return dma_mmap_coherent(d->dev, vma, d->cpu_addr, d->dma_handle, size);

	/* For MMIO registers: */
	vm_flags_set(vma, VM_IO | VM_DONTEXPAND | VM_DONTDUMP);
	vma->vm_page_prot = pgprot_noncached(vma->vm_page_prot);
	return io_remap_pfn_range(vma, vma->vm_start, d->phys >> PAGE_SHIFT,
				  size, vma->vm_page_prot);

	/* For kernel-allocated normal memory, page by page: */
	return remap_vmalloc_range(vma, d->vbuf, 0);
}
```

Three obligations, each a real CVE class when missed:

1. **Validate `vm_pgoff` and the length.** An unvalidated offset is an arbitrary-physical-
   memory read/write primitive. This is the classic `/dev/mem`-style vulnerability replicated
   in device drivers.
2. **Set the right `vm_flags` and `pgprot`.** MMIO must be uncached (`pgprot_noncached` or
   `pgprot_writecombine`) or the CPU will cache device registers. `VM_IO | VM_DONTEXPAND |
   VM_DONTDUMP` prevents the mapping from being expanded, forked oddly, or dumped into a core
   file.
3. **Handle the lifetime.** The mapping can outlive `close()` (a process can `mmap`, `close`,
   and keep using the mapping). If the memory can go away, you need `vm_ops->close` and a
   refcount, or `->fault` with a liveness check.

### T.9 Where char devices fit: `misc`, `cdev`, and `class`

Three registration levels, and choosing correctly saves a lot of code:

| Mechanism | Use when | Cost |
|---|---|---|
| **`misc_register()`** | one or a few devices, no per-device major needed | ~5 lines; shares major 10 (Ch. 30) |
| **`alloc_chrdev_region()` + `cdev_add()`** | many minors, or you need a whole major | ~30 lines |
| **A subsystem's `class`** (`input`, `hwmon`, `iio`, `tty`) | your device fits an existing category | least code, most conventions |

**The rule: if a subsystem exists for your device type, use it.** A temperature sensor should
be `hwmon` or `iio`, not a bespoke char device with a custom ioctl — because then `sensors`,
`lm-sensors`, and every monitoring tool work with no extra code, and you inherit a reviewed,
documented ABI. Writing a raw char device when a subsystem exists is the most common reason a
driver is rejected upstream.

---

## 1. Internals

### 1.1 The path from `open()` to your `->open`

```
open("/dev/mydev", O_RDWR)
  → do_sys_openat2() → path_openat() → do_dentry_open()
       inode->i_mode says S_IFCHR
    → chrdev_open()                           fs/char_dev.c
         kobj_lookup(cdev_map, inode->i_rdev) ★ the dev_t → cdev lookup
         inode->i_cdev = cdev                  (cached on the inode)
         cdev_get(cdev)                        ★ takes a kobject reference
         file->f_op = fops_get(cdev->ops)      ★ takes the MODULE reference (.owner)
         file->f_op->open(inode, file)         → YOUR ->open
```

`cdev_map` is a `kobj_map` — a hash of major → a list of (minor-range → kobject) entries.
`cdev_get()` is what keeps the device object alive for the duration of the open, and
`fops_get()` is what keeps the module alive.

### 1.2 `struct file_operations` — the ones that matter

```c
struct file_operations {
	struct module *owner;                     /* ★ ALWAYS THIS_MODULE */
	loff_t (*llseek)(struct file *, loff_t, int);
	ssize_t (*read)(struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *);
	ssize_t (*read_iter)(struct kiocb *, struct iov_iter *);    /* ★ preferred */
	ssize_t (*write_iter)(struct kiocb *, struct iov_iter *);
	__poll_t (*poll)(struct file *, struct poll_table_struct *);
	long (*unlocked_ioctl)(struct file *, unsigned int, unsigned long);
	long (*compat_ioctl)(struct file *, unsigned int, unsigned long);
	int (*mmap)(struct file *, struct vm_area_struct *);
	int (*open)(struct inode *, struct file *);
	int (*release)(struct inode *, struct file *);     /* LAST close only */
	int (*flush)(struct file *, fl_owner_t id);        /* EVERY close() */
	int (*fsync)(struct file *, loff_t, loff_t, int datasync);
	int (*fasync)(int, struct file *, int);            /* SIGIO */
	long (*fallocate)(struct file *, int, loff_t, loff_t);
	unsigned long (*get_unmapped_area)(...);
	int (*uring_cmd)(struct io_uring_cmd *, unsigned int);  /* io_uring passthrough */
};
```

**`release` vs `flush`:** `flush` runs on *every* `close()` of a descriptor; `release` runs
only when the **last** reference to the `struct file` drops (after `dup`/`fork` are accounted
for). Cleanup belongs in `release`; per-descriptor semantics (like flushing buffered data)
belong in `flush`.

### 1.3 Source map

```
fs/char_dev.c            ★ register_chrdev_region, cdev_add, chrdev_open
include/linux/cdev.h
include/linux/fs.h       ★★ struct file, struct file_operations, struct inode
fs/read_write.c          vfs_read/vfs_write, the iter plumbing
fs/select.c              ★ do_poll, poll_wait, the poll table
lib/iov_iter.c           copy_to_iter/copy_from_iter
drivers/char/            ★ real examples: mem.c, random.c, misc.c, hpet.c
samples/                 in-tree examples
Documentation/filesystems/vfs.rst   ★ the file_operations contract
Documentation/driver-api/basics.rst
```

---

## 2. Practice

### Lab 29.1 — A complete, correct character driver

This is the reference implementation for this chapter. Every line is load-bearing.

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * scullfifo — a complete character device.
 *
 * Locking and lifetime model (Ch. 25 §3):
 *   Object graph:
 *     scull_dev (kref)  <- one per minor
 *       ├── cdev          (kobject-refcounted; keeps scull_dev alive while open)
 *       └── kfifo         (protected by dev->lock)
 *     scull_ctx           <- one per open(); holds a reference on scull_dev
 *
 *   Locks:
 *     dev->lock (mutex)   - protects fifo, dead, nr_readers. Task context only.
 *
 *   Patterns:
 *     P2  refcounted lookup: open() takes a reference, release() drops it.
 *     P3  wait/wakeup: wait_event_interruptible with a death check.
 *     P6  producer/consumer via kfifo.
 *     P12 teardown: cdev_del -> dead=true -> wake_up_all -> kref_put.
 */
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/fs.h>
#include <linux/kfifo.h>
#include <linux/kref.h>
#include <linux/module.h>
#include <linux/mutex.h>
#include <linux/poll.h>
#include <linux/slab.h>
#include <linux/uaccess.h>
#include <linux/uio.h>

#define SCULL_NDEV   4
#define SCULL_FIFO   4096

#define SCULL_IOC_MAGIC   'S'
#define SCULL_IOC_RESET   _IO(SCULL_IOC_MAGIC, 0)
#define SCULL_IOC_GETINFO _IOR(SCULL_IOC_MAGIC, 1, struct scull_info)
#define SCULL_IOC_MAXNR   1

struct scull_info {
	__u32 fifo_size;
	__u32 fifo_used;
	__u32 nr_opens;
	__u32 __reserved[5];        /* ★ Ch. 24 T.3: room to grow, must be zero */
};

struct scull_dev {
	struct cdev            cdev;
	struct device         *dev;
	struct kref            ref;
	struct mutex           lock;
	DECLARE_KFIFO(fifo, char, SCULL_FIFO);
	wait_queue_head_t      readq, writeq;
	struct fasync_struct  *fasync;
	bool                   dead;
	unsigned int           nr_opens;
	int                    index;
};

struct scull_ctx {
	struct scull_dev *dev;
	u64               bytes_read, bytes_written;   /* per-OPEN state (T.3) */
};

static dev_t           scull_devt;
static struct class   *scull_class;
static struct scull_dev *scull_devs;

/* ---------------- lifetime ---------------- */

static void scull_release_dev(struct kref *ref)
{
	struct scull_dev *d = container_of(ref, struct scull_dev, ref);

	pr_info("scull%d: last reference dropped, freeing\n", d->index);
	mutex_destroy(&d->lock);
	/* the array itself is freed in module_exit */
}

static bool scull_get(struct scull_dev *d) { return kref_get_unless_zero(&d->ref); }
static void scull_put(struct scull_dev *d) { kref_put(&d->ref, scull_release_dev); }

/* ---------------- readiness predicates (T.5: ONE source of truth) -------- */

static bool scull_readable(struct scull_dev *d)
{
	guard(mutex)(&d->lock);
	return !kfifo_is_empty(&d->fifo) || d->dead;
}
static bool scull_writable(struct scull_dev *d)
{
	guard(mutex)(&d->lock);
	return !kfifo_is_full(&d->fifo) || d->dead;
}

/* ---------------- file operations ---------------- */

static int scull_open(struct inode *inode, struct file *file)
{
	struct scull_dev *d = container_of(inode->i_cdev, struct scull_dev, cdev);
	struct scull_ctx *ctx;

	if (!scull_get(d))                      /* ★ P2: may be dying */
		return -ENODEV;

	ctx = kzalloc(sizeof(*ctx), GFP_KERNEL);
	if (!ctx) {
		scull_put(d);
		return -ENOMEM;
	}
	ctx->dev = d;
	file->private_data = ctx;

	scoped_guard(mutex, &d->lock) {
		if (d->dead) {
			kfree(ctx);
			scull_put(d);
			return -ENODEV;
		}
		d->nr_opens++;
	}

	stream_open(inode, file);               /* ★ T.6: no position, allow concurrency */
	dev_dbg(d->dev, "opened (now %u)\n", d->nr_opens);
	return 0;
}

static int scull_release(struct inode *inode, struct file *file)
{
	struct scull_ctx *ctx = file->private_data;
	struct scull_dev *d = ctx->dev;

	scoped_guard(mutex, &d->lock)
		d->nr_opens--;

	dev_dbg(d->dev, "closed after r=%llu w=%llu\n", ctx->bytes_read, ctx->bytes_written);
	kfree(ctx);
	scull_put(d);
	return 0;
}

static ssize_t scull_read(struct file *file, char __user *ubuf, size_t len, loff_t *off)
{
	struct scull_ctx *ctx = file->private_data;
	struct scull_dev *d = ctx->dev;
	unsigned int copied = 0;
	int ret;

	if (!scull_readable(d)) {
		if (file->f_flags & O_NONBLOCK)
			return -EAGAIN;
		/* ★ P3: the death condition is part of the predicate */
		ret = wait_event_interruptible(d->readq, scull_readable(d));
		if (ret)
			return ret;                       /* -ERESTARTSYS */
	}

	guard(mutex)(&d->lock);
	if (d->dead)
		return -ENODEV;

	ret = kfifo_to_user(&d->fifo, ubuf, len, &copied);
	if (ret)
		return ret;

	if (copied) {
		ctx->bytes_read += copied;
		wake_up_interruptible(&d->writeq);        /* space freed */
		kill_fasync(&d->fasync, SIGIO, POLL_OUT);
	}
	return copied;
}

static ssize_t scull_write(struct file *file, const char __user *ubuf,
			   size_t len, loff_t *off)
{
	struct scull_ctx *ctx = file->private_data;
	struct scull_dev *d = ctx->dev;
	unsigned int copied = 0;
	int ret;

	if (!scull_writable(d)) {
		if (file->f_flags & O_NONBLOCK)
			return -EAGAIN;
		ret = wait_event_interruptible(d->writeq, scull_writable(d));
		if (ret)
			return ret;
	}

	guard(mutex)(&d->lock);
	if (d->dead)
		return -ENODEV;

	ret = kfifo_from_user(&d->fifo, ubuf, len, &copied);
	if (ret)
		return ret;

	if (copied) {
		ctx->bytes_written += copied;
		wake_up_interruptible(&d->readq);         /* ★ P3: condition THEN wake */
		kill_fasync(&d->fasync, SIGIO, POLL_IN);
	}
	return copied;
}

static __poll_t scull_poll(struct file *file, poll_table *wait)
{
	struct scull_ctx *ctx = file->private_data;
	struct scull_dev *d = ctx->dev;
	__poll_t mask = 0;

	poll_wait(file, &d->readq, wait);        /* ★ register on BOTH queues */
	poll_wait(file, &d->writeq, wait);

	guard(mutex)(&d->lock);
	if (d->dead)
		return EPOLLHUP | EPOLLERR;
	if (!kfifo_is_empty(&d->fifo))
		mask |= EPOLLIN  | EPOLLRDNORM;
	if (!kfifo_is_full(&d->fifo))
		mask |= EPOLLOUT | EPOLLWRNORM;
	return mask;
}

static long scull_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
{
	struct scull_ctx *ctx = file->private_data;
	struct scull_dev *d = ctx->dev;
	void __user *uarg = (void __user *)arg;

	if (_IOC_TYPE(cmd) != SCULL_IOC_MAGIC || _IOC_NR(cmd) > SCULL_IOC_MAXNR)
		return -ENOTTY;                  /* ★ Ch. 24 T.7: ENOTTY, not EINVAL */

	switch (cmd) {
	case SCULL_IOC_RESET: {
		guard(mutex)(&d->lock);
		if (d->dead)
			return -ENODEV;
		kfifo_reset(&d->fifo);
		wake_up_interruptible(&d->writeq);
		return 0;
	}
	case SCULL_IOC_GETINFO: {
		struct scull_info info = {};      /* ★ zero-init: no stack infoleak */

		scoped_guard(mutex, &d->lock) {
			if (d->dead)
				return -ENODEV;
			info.fifo_size = kfifo_size(&d->fifo);
			info.fifo_used = kfifo_len(&d->fifo);
			info.nr_opens  = d->nr_opens;
		}
		return copy_to_user(uarg, &info, sizeof(info)) ? -EFAULT : 0;
	}
	default:
		return -ENOTTY;
	}
}

static int scull_fasync(int fd, struct file *file, int mode)
{
	struct scull_ctx *ctx = file->private_data;

	return fasync_helper(fd, file, mode, &ctx->dev->fasync);
}

static const struct file_operations scull_fops = {
	.owner          = THIS_MODULE,       /* ★ T.3: blocks rmmod while open */
	.open           = scull_open,
	.release        = scull_release,
	.read           = scull_read,
	.write          = scull_write,
	.poll           = scull_poll,
	.unlocked_ioctl = scull_ioctl,
	.compat_ioctl   = compat_ptr_ioctl,  /* ★ safe: our struct is fixed-width */
	.fasync         = scull_fasync,
	/* .llseek intentionally unset: not seekable (T.6) */
};

/* ---------------- module init/exit ---------------- */

static char *scull_devnode(const struct device *dev, umode_t *mode)
{
	if (mode)
		*mode = 0666;                /* world-writable, for the lab only */
	return NULL;
}

static int __init scull_init(void)
{
	int i, ret;

	scull_devs = kcalloc(SCULL_NDEV, sizeof(*scull_devs), GFP_KERNEL);
	if (!scull_devs)
		return -ENOMEM;

	ret = alloc_chrdev_region(&scull_devt, 0, SCULL_NDEV, "scullfifo");
	if (ret)
		goto err_free;
	pr_info("major %d allocated\n", MAJOR(scull_devt));

	scull_class = class_create("scullfifo");
	if (IS_ERR(scull_class)) {
		ret = PTR_ERR(scull_class);
		goto err_region;
	}
	scull_class->devnode = scull_devnode;

	for (i = 0; i < SCULL_NDEV; i++) {
		struct scull_dev *d = &scull_devs[i];
		dev_t devt = MKDEV(MAJOR(scull_devt), MINOR(scull_devt) + i);

		d->index = i;
		kref_init(&d->ref);
		mutex_init(&d->lock);
		INIT_KFIFO(d->fifo);
		init_waitqueue_head(&d->readq);
		init_waitqueue_head(&d->writeq);

		cdev_init(&d->cdev, &scull_fops);
		d->cdev.owner = THIS_MODULE;
		ret = cdev_add(&d->cdev, devt, 1);
		if (ret)
			goto err_devs;

		d->dev = device_create(scull_class, NULL, devt, d, "scull%d", i);
		if (IS_ERR(d->dev)) {
			ret = PTR_ERR(d->dev);
			cdev_del(&d->cdev);
			goto err_devs;
		}
	}
	return 0;

err_devs:
	while (--i >= 0) {
		device_destroy(scull_class, scull_devs[i].dev->devt);
		cdev_del(&scull_devs[i].cdev);
	}
	class_destroy(scull_class);
err_region:
	unregister_chrdev_region(scull_devt, SCULL_NDEV);
err_free:
	kfree(scull_devs);
	return ret;
}

static void __exit scull_exit(void)
{
	int i;

	for (i = 0; i < SCULL_NDEV; i++) {
		struct scull_dev *d = &scull_devs[i];

		/* ★ P12: STOP */
		device_destroy(scull_class, MKDEV(MAJOR(scull_devt), MINOR(scull_devt) + i));
		cdev_del(&d->cdev);            /* no new open() can find us */

		/* ★ P12: DRAIN — wake everyone blocked, they will see dead and bail */
		scoped_guard(mutex, &d->lock)
			d->dead = true;
		wake_up_all(&d->readq);
		wake_up_all(&d->writeq);

		/* ★ P12: FREE — our reference; open fds hold their own */
		scull_put(d);
	}
	class_destroy(scull_class);
	unregister_chrdev_region(scull_devt, SCULL_NDEV);
	kfree(scull_devs);
}

module_init(scull_init);
module_exit(scull_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("A complete character device example");
```

```bash
sudo insmod scullfifo.ko
ls -l /dev/scull*
cat /proc/devices | grep scull
ls /sys/class/scullfifo/

echo "hello world" > /dev/scull0
cat /dev/scull0
# blocking read in one terminal:
cat /dev/scull1 &
echo "ping" > /dev/scull1
# non-blocking:
dd if=/dev/scull2 iflag=nonblock bs=1 count=1 2>&1 | tail -2   # → EAGAIN
```

### Lab 29.2 — Test every path from userspace

```c
/* sculltest.c — gcc -O2 -o sculltest sculltest.c -lpthread */
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <poll.h>
#include <pthread.h>
#include <signal.h>
#include <stdio.h>
#include <string.h>
#include <sys/ioctl.h>
#include <unistd.h>

#define SCULL_IOC_MAGIC 'S'
#define SCULL_IOC_RESET   _IO(SCULL_IOC_MAGIC, 0)
struct scull_info { unsigned fifo_size, fifo_used, nr_opens, __r[5]; };
#define SCULL_IOC_GETINFO _IOR(SCULL_IOC_MAGIC, 1, struct scull_info)

static void *writer(void *arg)
{
	int fd = open("/dev/scull0", O_WRONLY);
	sleep(1);
	write(fd, "delayed data", 12);
	close(fd);
	return NULL;
}

int main(void)
{
	int fd = open("/dev/scull0", O_RDWR);
	char buf[64];
	struct scull_info info;
	pthread_t t;

	/* 1. ioctl */
	ioctl(fd, SCULL_IOC_RESET);
	ioctl(fd, SCULL_IOC_GETINFO, &info);
	printf("size=%u used=%u opens=%u\n", info.fifo_size, info.fifo_used, info.nr_opens);

	/* 2. unknown ioctl must give ENOTTY */
	printf("bad ioctl: %d errno=%d (want ENOTTY=%d)\n",
	       ioctl(fd, _IO(SCULL_IOC_MAGIC, 99)), errno, ENOTTY);

	/* 3. O_NONBLOCK read on an empty fifo */
	fcntl(fd, F_SETFL, O_NONBLOCK);
	printf("nonblock read: %zd errno=%d (want EAGAIN=%d)\n",
	       read(fd, buf, sizeof(buf)), errno, EAGAIN);
	fcntl(fd, F_SETFL, 0);

	/* 4. poll() must report not-ready, then ready */
	struct pollfd p = { .fd = fd, .events = POLLIN };
	printf("poll (empty): %d\n", poll(&p, 1, 100));
	pthread_create(&t, NULL, writer, NULL);
	printf("poll (waiting): %d revents=0x%x\n", poll(&p, 1, 5000), p.revents);
	printf("read: %zd '%.*s'\n", read(fd, buf, sizeof(buf)), 12, buf);
	pthread_join(t, NULL);

	/* 5. seek must fail (T.6) */
	printf("lseek: %ld errno=%d (want ESPIPE=%d)\n",
	       (long)lseek(fd, 0, SEEK_SET), errno, ESPIPE);

	/* 6. blocking read interrupted by a signal */
	signal(SIGALRM, (void (*)(int))1);   /* SIG_IGN-ish; we want EINTR */
	struct sigaction sa = { .sa_handler = (void (*)(int))+[](int){} };
	alarm(1);
	printf("interrupted read: %zd errno=%d (want EINTR=%d)\n",
	       read(fd, buf, sizeof(buf)), errno, EINTR);

	close(fd);
	return 0;
}
```
```bash
gcc -O2 -o /tmp/sculltest /tmp/sculltest.c -lpthread && /tmp/sculltest
strace -e trace=openat,read,write,ioctl,poll,lseek /tmp/sculltest 2>&1 | tail -20
```

### Lab 29.3 — Prove the `.owner` and refcount protections

```bash
# 1. .owner blocks rmmod
exec 3< /dev/scull0
sudo rmmod scullfifo            # → "Module scullfifo is in use"
lsmod | grep scull              # "Used by" count is 1
exec 3<&-
sudo rmmod scullfifo            # now it works

# 2. Remove .owner from scull_fops, rebuild, and repeat with KASAN
./scripts/config -e KASAN
exec 3< /dev/scull0
sudo rmmod scullfifo            # succeeds!
head -c 1 /dev/fd/3             # → jump into unmapped module text
dmesg | tail -30

# 3. The dead-flag path: block a reader, then unload
cat /dev/scull0 &               # blocks in wait_event_interruptible
sudo rmmod scullfifo            # wake_up_all + dead -> the cat exits with an error
wait; dmesg | tail -5
```

### Lab 29.4 — Multi-threaded on one fd (T.7 case 2)

```c
/* Two threads, ONE fd, both reading. Without stream_open() the VFS
 * serializes them on f_pos_lock; with it, they run concurrently. */
static void *rd(void *a) {
	int fd = *(int *)a; char b[16];
	for (int i = 0; i < 1000; i++) read(fd, b, sizeof(b));
	return NULL;
}
int main(void) {
	int fd = open("/dev/scull0", O_RDWR | O_NONBLOCK);
	pthread_t t[4];
	for (int i = 0; i < 4; i++) pthread_create(&t[i], NULL, rd, &fd);
	for (int i = 0; i < 4; i++) pthread_join(t[i], NULL);
}
```
Run with and without `stream_open()` in `scull_open()` and measure with
`perf stat -e context-switches`. Then look for `f_pos_lock` contention:
```bash
sudo bpftrace -e 'kprobe:__fdget_pos { @[comm] = count(); }'
grep -n 'f_pos_lock\|FMODE_STREAM' fs/file.c include/linux/fs.h
```

### Lab 29.5 — Add `mmap` and validate it (T.8)

```c
static int scull_mmap(struct file *file, struct vm_area_struct *vma)
{
	struct scull_dev *d = ((struct scull_ctx *)file->private_data)->dev;
	unsigned long size = vma->vm_end - vma->vm_start;

	if (vma->vm_pgoff)                          /* ★ no offset allowed */
		return -EINVAL;
	if (size > SCULL_MMAP_SIZE)                 /* ★ bound the length */
		return -EINVAL;

	vm_flags_set(vma, VM_DONTEXPAND | VM_DONTDUMP);
	return remap_vmalloc_range(vma, d->mmap_buf, 0);
}
```
```c
/* userspace */
char *p = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
strcpy(p, "written via mmap");
munmap(p, 4096);
```
**Then attack your own code:** try `mmap(..., offset = 0x100000)`, `length = 1 GiB`, and
`PROT_EXEC`. Each must be rejected. Build with KASAN and confirm no out-of-bounds mapping is
possible. Compare with `drivers/char/mem.c`'s checks.

### Lab 29.6 — `read_iter` / `write_iter` and `readv`

Convert `scull_read`/`scull_write` to the iter forms:
```c
static ssize_t scull_read_iter(struct kiocb *iocb, struct iov_iter *to)
{
	struct scull_ctx *ctx = iocb->ki_filp->private_data;
	struct scull_dev *d = ctx->dev;
	char tmp[256];
	unsigned int n;

	if (!scull_readable(d)) {
		if (iocb->ki_flags & IOCB_NOWAIT)
			return -EAGAIN;
		if (wait_event_interruptible(d->readq, scull_readable(d)))
			return -ERESTARTSYS;
	}
	guard(mutex)(&d->lock);
	n = kfifo_out(&d->fifo, tmp, min(sizeof(tmp), iov_iter_count(to)));
	return n ? copy_to_iter(tmp, n, to) : 0;
}
```
```bash
# Now readv/writev and preadv2(RWF_NOWAIT) work:
cat > /tmp/iov.c <<'EOF'
#include <sys/uio.h>
#include <fcntl.h>
#include <stdio.h>
int main(void) {
	int fd = open("/dev/scull0", O_RDWR);
	char a[8], b[8];
	struct iovec iv[2] = { { a, 8 }, { b, 8 } };
	write(fd, "0123456789abcdef", 16);
	printf("readv -> %zd  a='%.8s' b='%.8s'\n", readv(fd, iv, 2), a, b);
}
EOF
gcc -O2 -o /tmp/iov /tmp/iov.c && /tmp/iov
```

### Lab 29.7 — Fuzz it

```bash
# Build with the full sanitizer set (Ch. 06 Lab 6.6), then:
# 1. Trinity or syzkaller against /dev/scull*
# 2. Or a quick homemade fuzzer:
cat > /tmp/fuzz.sh <<'EOF'
for i in $(seq 1 100000); do
  dd if=/dev/urandom of=/dev/scull0 bs=$((RANDOM % 8192 + 1)) count=1 2>/dev/null
  dd if=/dev/scull0 of=/dev/null bs=$((RANDOM % 8192 + 1)) count=1 2>/dev/null
done
EOF
for i in 1 2 3 4; do bash /tmp/fuzz.sh & done
# Meanwhile:
while true; do sudo rmmod scullfifo 2>/dev/null; sudo insmod scullfifo.ko; done
dmesg -w
```
**The acceptance criterion is zero splats under concurrent I/O and module churn.**

---

## 3. Mastery drills

1. **Read `fs/char_dev.c`** completely. Explain `kobj_map`, and trace `open()` from
   `chrdev_open()` to your `->open`. Where exactly is the module reference taken?

2. **Per-device vs per-open.** For each of: a serial port, a GPIO chip, a random number
   generator, a video capture device — list which state is per-device and which per-open.
   Then find a real driver that gets it wrong.

3. **`release` vs `flush`.** Write a test that `dup()`s an fd and closes both. Instrument
   both callbacks and show `flush` runs twice, `release` once. Explain what belongs in each.

4. **Poll correctness.** Deliberately remove one `poll_wait()` from Lab 29.1 and show that an
   `epoll`-based reader hangs while a `select`-based one may not. Explain why.

5. **Edge-triggered.** Write an `EPOLLET` reader for your device. Find a wakeup path that
   works level-triggered but drops events edge-triggered. This is a real, subtle bug class.

6. **`stream_open()`.** Read `fs/file.c`'s `__fdget_pos()` and `include/linux/fs.h`'s
   `FMODE_STREAM`. Explain exactly what serialization it removes and measure the difference.

7. **Signal handling.** Explain the full path from `SIGINT` during a blocking `read()` to
   userspace seeing `EINTR` — including `-ERESTARTSYS`, `SA_RESTART`, and where the
   translation happens (Ch. 20 T.6).

8. **mmap lifetime.** Construct the case where a process `mmap`s, then `close()`s, then the
   driver is unbound. What keeps the mapping valid? Implement `vm_ops->open`/`->close` with a
   refcount and verify under KASAN.

9. **Choose the subsystem.** For each device, decide char-device vs an existing subsystem and
   justify: a 3-axis accelerometer, a watchdog timer, an LED, a CAN controller, a hardware
   RNG, a fingerprint reader, an FPGA bitstream loader. (Answers exist for all seven —
   find them.)

10. **Read a real driver.** `drivers/char/hw_random/core.c` or `drivers/char/misc.c` or
    `drivers/char/random.c`. Write its model comment (Ch. 25 §3) from the source alone.

11. **The `uring_cmd` frontier.** Read `include/linux/fs.h`'s `->uring_cmd` and a user
    (`drivers/nvme/host/ioctl.c`). Explain how it lets a char device expose async,
    batched operations without `ioctl`'s round-trip cost — and what new concurrency
    obligations it creates.

---

## 4. Further reading

**Kernel documentation:**
- `Documentation/filesystems/vfs.rst` ★★ — the `file_operations` contract, method by method
- `Documentation/driver-api/basics.rst`
- `Documentation/admin-guide/devices.txt` — the historical major/minor registry
- `Documentation/filesystems/locking.rst` — **which VFS locks are held when your method is
  called**; essential and frequently overlooked
- `Documentation/core-api/kernel-api.rst` (uaccess section)

**Source to read:**
- `fs/char_dev.c`, `include/linux/cdev.h`
- `drivers/char/misc.c` — the simplest complete registration (Ch. 30)
- `drivers/char/mem.c` — `/dev/mem`, `/dev/null`, `/dev/zero`: mmap and llseek done properly
- `drivers/char/random.c` — a serious char device: per-CPU state, poll, ioctl, entropy
- `samples/` and `tools/testing/selftests/` for test patterns

**Books:**
- Corbet, Rubini & Kroah-Hartman, *Linux Device Drivers* 3e, **Ch. 3 (char drivers), Ch. 5
  (concurrency), Ch. 6 (advanced char operations), Ch. 15 (mmap)** — the `scull` example this
  chapter descends from. APIs are dated; the structure is exactly right.
- Kerrisk, *The Linux Programming Interface*, Ch. 63 (alternative I/O models) — the userspace
  side of `poll`/`epoll`/`SIGIO`
- Venkateswaran, *Essential Linux Device Drivers* — broader subsystem coverage

**LWN:**
- "The iov_iter interface" / "Fixing read_iter and write_iter"
- "stream_open() and concurrent reads"
- "The trouble with no_llseek"
- "io_uring passthrough and uring_cmd"

→ Next: [30-misc-faux-debugfs.md](30-misc-faux-debugfs.md)

# Chapter 30 — misc Devices, the `faux` Bus, and debugfs

> **Goal:** register a character device in five lines instead of fifty, know where
> hardware-less "devices" belong now that abusing the platform bus is deprecated, and use
> debugfs correctly — including understanding why it is the one interface with *no* ABI
> guarantee, and why that is a feature.

---

## Theory & First Principles

### T.0 — Start here: you need a `/dev` node. Which of five ways?

Ch. 29 showed the full character-device path: allocate a region, set up a `cdev`, create a
class, create a device. That is ~40 lines of boilerplate and four failure points. For a
driver that just needs one device node, it is absurd.

The kernel offers five ways to expose something to userspace, and choosing wrong is a
reviewable mistake:

```c
/* ① full cdev        */  alloc_chrdev_region(); cdev_init(); cdev_add(); class_create();
/* ② miscdevice       */  misc_register(&(struct miscdevice){ .name = "mydev", .fops = &f });
/* ⁢ sysfs attribute  */  DEVICE_ATTR_RW(value);
/* ⁣ debugfs          */  debugfs_create_file("state", 0444, dir, d, &fops);
/* ⁤ faux device      */  faux_device_create("mything", NULL, &faux_ops);
```

**The decision is not about convenience. It is about what you are promising:**

| | Purpose | ABI stability | Rule |
|---|---|---|---|
| **full cdev** ① | many minors, or you need the class | **stable, forever** | only when you truly need many devices |
| **miscdevice** ② | one device node, dynamic minor | **stable, forever** | the default for a single node |
| **sysfs** ⁢ | configuration and status | **stable, forever** | **one value per file**, text, no exceptions |
| **debugfs** ⁣ | developer diagnostics | **none whatsoever** | anything you like; **nothing may depend on it** |
| **faux** ⁤ | a driver with no real device | n/a | replaces the old "fake platform device" abuse |

**The column that matters is the middle one.** Three of these are permanent commitments under
Ch. 24's rules; one is explicitly not. Getting that wrong in either direction is a real cost:

- Put a product-critical interface in **debugfs** and you have shipped something that may
  vanish, and that is not mounted on hardened systems (Ch. 102 §T.8).
- Put a debugging dump in **sysfs** and you have promised its exact text format forever —
  Hyrum's Law (Ch. 24 §T.0) applies to every byte of it.

**The sysfs "one value per file" rule deserves a sentence of its own**, because it is the
most-violated rule in the kernel and reviewers will reject you for it:

```
  GOOD:  /sys/class/mydev/mydev0/temperature   ->  "42000\n"
  BAD:   /sys/class/mydev/mydev0/status        ->  "temp=42 fan=on errors=3\n"
```

The bad version is an *unversioned, unparseable, permanent* mini-protocol. The good version
composes with `cat`, with shell globs, with `udev` rules, and with every monitoring tool ever
written — which is the whole point of the narrow waist (Ch. 00 §T.3b).

**And `miscdevice` itself is a small, elegant lesson in resource management.** Character
device major numbers were a fixed 8-bit namespace — 256 of them, hand-allocated by a human
maintaining a registry file. `misc` is one major (10) shared by everyone, sub-allocated
dynamically by minor. **A scarce, centrally-administered resource replaced by a dynamically
allocated one** — the same move as dynamic port allocation, dynamic device numbers, and
`/dev` itself becoming `devtmpfs`.

```bash
cat /proc/devices | head -20                 # the major-number registry
ls -l /dev/ | awk '$5 == "10," {print}'       # everything sharing major 10
mount | grep debugfs                          # is it even mounted?
ls /sys/class/misc/
```

---

### T.1 The major-number scarcity problem, and `misc` as a shared registry

Ch. 29 T.2 established that `dev_t` is 12 bits of major and 20 of minor. 4096 majors sounds
like plenty until you remember that **every driver that wants a char device wants one**, and
that for decades they were statically assigned by a human maintaining
`Documentation/admin-guide/devices.txt`.

The observation that produced `misc`: **the overwhelming majority of char drivers need
exactly one device node.** Allocating a whole major (1M minors) for one node wastes the
namespace by a factor of a million.

`misc` is therefore a **second-level allocator**: it owns major **10** and sub-allocates
minors to drivers that need one node.

```c
#include <linux/miscdevice.h>

static struct miscdevice my_misc = {
	.minor = MISC_DYNAMIC_MINOR,     /* ★ let the core pick */
	.name  = "mydev",                /* → /dev/mydev */
	.fops  = &my_fops,
	.mode  = 0666,                   /* optional: the node's permissions */
};

ret = misc_register(&my_misc);       /* creates the device, the node, the uevent */
misc_deregister(&my_misc);
/* or, managed (Ch. 28): */
ret = devm_misc_register(dev, &my_misc);
```

That replaces `alloc_chrdev_region` + `cdev_init` + `cdev_add` + `class_create` +
`device_create` + all four error paths. The trade-off is precise:

| | `misc` | `cdev` + own major |
|---|---|---|
| Lines of code | ~5 | ~40 |
| Device nodes | **1** (or a few) | up to 1M minors |
| Major number | shared (10) | your own |
| Class | `/sys/class/misc/` | your own class |
| Parent device | `.parent` field, optional | you choose |

**Rule: if you need one node, use `misc`. If you need many minors of the same kind, use
`cdev`. If a subsystem exists, use the subsystem** (Ch. 29 T.9).

A subtlety worth knowing: `misc_register()` does **not** take a reference on anything for you
and the `struct miscdevice` must stay alive until `misc_deregister()`. Embedding it in a
`devm_kzalloc`'d struct is the classic Ch. 28 T.3 bug — an open fd outlives unbind and
`file->private_data` (which points at your container) dangles.

```bash
ls /sys/class/misc/
cat /proc/misc                      # every allocated misc minor
grep -w 10 /proc/devices            # "10 misc"
ls -l /dev/ | awk '$5 == "10," '    # every misc node
```

### T.2 The pseudo-device problem, and why `platform_device` was abused for it

Consider a driver with **no hardware at all**: `zram`, `null_blk`, a test module, a software
crypto engine, a loopback device, an in-kernel unit test. It still wants:

- a `struct device` (for sysfs attributes, `dev_err()`, device links, PM callbacks),
- the probe/remove lifecycle,
- a place in `/sys/devices/`.

But it has no bus, no address, and nothing to enumerate. For two decades the answer was
`platform_device_register_simple()` — **abusing the platform bus** (Ch. 31), which exists to
represent *real, non-discoverable hardware described by firmware*.

The abuse caused real problems:

- Platform devices carry `struct resource` arrays, DMA configuration, IRQ mapping, OF/ACPI
  node pointers — none of which a pseudo-device has, all of which the core tries to set up.
- `of_platform` and `fw_devlink` (Ch. 27 T.5) walk platform devices looking for firmware
  nodes and get confused by ones that have none.
- `/sys/bus/platform/devices/` became a mix of "the UART at 0x10000000" and
  "the zram software device", which is meaningless as a taxonomy (Ch. 26 T.3).
- Platform-bus PM and DMA paths ran unnecessarily.

**The `faux` bus (6.14+, Greg Kroah-Hartman)** exists to say the quiet part out loud:

```c
#include <linux/device/faux.h>

static int my_faux_probe(struct faux_device *fdev)
{
	dev_info(&fdev->dev, "probed\n");
	return 0;
}
static void my_faux_remove(struct faux_device *fdev) { }

static const struct faux_device_ops my_faux_ops = {
	.probe  = my_faux_probe,
	.remove = my_faux_remove,
};

static struct faux_device *fdev;

fdev = faux_device_create("mypseudo", NULL, &my_faux_ops);
/* or with sysfs groups: faux_device_create_with_groups(...) */
...
faux_device_destroy(fdev);
```

It gives you a `struct device` in `/sys/devices/faux/` with attributes and probe/remove, and
**nothing else** — no resources, no IRQ mapping, no firmware node, no DMA setup. The name is
deliberately unflattering: it is a *fake* device, and calling it that prevents the category
error.

The design lesson generalizes: **when a mechanism is being used for a purpose it was not
designed for, the fix is usually a new, smaller, honestly-named mechanism** — not more
special cases in the old one. There is an ongoing conversion of pseudo-devices from platform
to faux; it is an excellent first-patch area.

```bash
ls /sys/devices/faux/ 2>/dev/null
ls /sys/bus/platform/devices/ | head -30     # count how many are NOT real hardware
git log --oneline --grep='faux' | head -20
$EDITOR drivers/base/faux.c                  # ~250 lines; read it completely
```

### T.3 debugfs: the value of an *explicitly* unstable interface

Ch. 24 T.1 established that every observable behaviour becomes ABI (Hyrum's Law). debugfs is
the deliberate exception:

> **`Documentation/filesystems/debugfs.rst`: "There are no rules. There is no ABI stability
> guarantee. Anything in debugfs can change or disappear at any time."**

Why is an interface with no guarantees *valuable*? Because the guarantee is what makes
interfaces expensive. Having a place where you can:

- expose whatever internal state is useful today,
- delete it next release when the implementation changes,
- not write ABI documentation,
- not support it for a decade,

means developers actually expose useful state instead of exposing nothing. **The alternative
to debugfs is not a better-designed sysfs file; it is no visibility at all.**

This is a general architectural principle: **provide a clearly-marked unstable tier so that
the stable tier can stay small.** The same reasoning produces `/sys/kernel/debug` vs `/sys`,
`tracefs` vs `perf_event_open`, `-rc` kernels vs releases, and (in userspace) semver's
`0.x` versions and "experimental" feature flags.

The rule is enforced socially and structurally: debugfs is mounted at
`/sys/kernel/debug` with mode `0700` owned by root, is compiled out entirely with
`CONFIG_DEBUG_FS=n`, and is disabled by `lockdown` mode. **A distribution shipping a tool
that depends on a debugfs file is doing something the kernel explicitly refuses to support**
— and when it breaks, the answer is "yes, that is what debugfs means."

(In practice, a few debugfs files *have* become de-facto ABI anyway — `tracefs` was split out
of debugfs precisely because tracing tools depended on it. That split is the correct response:
*if something becomes ABI, promote it to a real interface with real guarantees.*)

### T.4 The debugfs API, and the one rule that matters

```c
#include <linux/debugfs.h>

struct dentry *dir = debugfs_create_dir("mydriver", NULL);

/* Scalars — no code required at all */
debugfs_create_u32("counter",   0644, dir, &priv->counter);
debugfs_create_u64("bytes",     0444, dir, &priv->bytes);
debugfs_create_x32("flags",     0644, dir, &priv->flags);       /* hex */
debugfs_create_bool("enabled",  0644, dir, &priv->enabled);
debugfs_create_ulong("jiffies_at_probe", 0444, dir, &priv->t0);
debugfs_create_atomic_t("refs", 0444, dir, &priv->refs);
debugfs_create_size_t("bufsz",  0444, dir, &priv->bufsz);

/* A blob of memory */
priv->blob.data = priv->regs_snapshot;
priv->blob.size = sizeof(priv->regs_snapshot);
debugfs_create_blob("regs", 0444, dir, &priv->blob);

/* A register dump, formatted by the core */
static const struct debugfs_reg32 my_regs[] = {
	{ .name = "CTRL",   .offset = 0x00 },
	{ .name = "STATUS", .offset = 0x04 },
};
priv->regset.regs = my_regs;
priv->regset.nregs = ARRAY_SIZE(my_regs);
priv->regset.base = priv->mmio;
debugfs_create_regset32("registers", 0444, dir, &priv->regset);

/* Arbitrary logic: a seq_file (Ch. 24 §1.2) */
static int state_show(struct seq_file *m, void *v)
{
	struct my_priv *p = m->private;

	seq_printf(m, "state=%d queued=%u errors=%llu\n", p->state, p->queued, p->errors);
	return 0;
}
DEFINE_SHOW_ATTRIBUTE(state);
debugfs_create_file("state", 0444, dir, priv, &state_fops);

/* A simple settable value with custom get/set */
DEFINE_DEBUGFS_ATTRIBUTE(my_fops, my_get, my_set, "%llu\n");
debugfs_create_file_unsafe("threshold", 0644, dir, priv, &my_fops);

debugfs_remove_recursive(dir);
/* or managed: */
struct dentry *d = debugfs_create_dir("mydriver", NULL);
devm_add_action_or_reset(dev, (void (*)(void *))debugfs_remove_recursive, d);
```

**The one rule:** *never check the return value for errors.* Every `debugfs_create_*`
returns an `ERR_PTR` on failure, but the API is explicitly designed so that passing that
error pointer to subsequent calls is harmless, and `debugfs_remove()` accepts it. Drivers
must **not** fail probe because debugfs failed — debugfs may be compiled out entirely, in
which case every function is an inline no-op.

```c
/* ★ WRONG — breaks on CONFIG_DEBUG_FS=n and adds pointless error paths */
dir = debugfs_create_dir("mydriver", NULL);
if (IS_ERR(dir))
	return PTR_ERR(dir);

/* ★ RIGHT */
dir = debugfs_create_dir("mydriver", NULL);
debugfs_create_u32("counter", 0644, dir, &priv->counter);
```

There is one genuine hazard: **debugfs files can be open while your driver unbinds.**
`debugfs_remove()` waits for active references (the same `kernfs` mechanism as Ch. 26 T.4),
but a file whose `->private` points at `devm_` memory is a UAF if the file survives.
`debugfs_file_get()`/`debugfs_file_put()` and the `_unsafe` naming exist to make this
explicit: `debugfs_create_file()` wraps your fops with reference protection;
`debugfs_create_file_unsafe()` does not, and requires you to handle it.

### T.5 Choosing between debugfs, sysfs, procfs, and tracefs

| | Stability | Permissions | Format | Use for |
|---|---|---|---|---|
| **sysfs** | **permanent ABI** | per-attribute | one value, text | device attributes, configuration |
| **procfs** | permanent ABI | per-file | text, often tabular | legacy; **do not add new files** |
| **debugfs** | **none** | root-only by default | anything | internal state, developer tools |
| **tracefs** | mostly stable | root | tracing protocol | ftrace/events |
| **configfs** | ABI | per-item | mkdir-created objects | *creating* kernel objects from userspace |

Two rules maintainers enforce:

1. **Do not add new files to `/proc`.** It is a process filesystem that accumulated global
   state for historical reasons. New global state goes to sysfs (if stable) or debugfs (if
   not). New per-process state can go in `/proc/<pid>/` with justification.
2. **Do not put configuration in debugfs.** If users need it in production, it is ABI and it
   belongs in sysfs — with documentation, a stability class, and the maintenance commitment
   that implies. Putting a knob in debugfs to avoid the ABI discussion is a known
   anti-pattern and reviewers call it out.

**configfs** deserves a mention because it fills a real gap: sysfs lets you configure objects
the kernel created; configfs lets userspace **create** them with `mkdir`. That is how
`null_blk`, NVMe targets, USB gadgets, and `dm` targets are configured:

```bash
sudo mount -t configfs none /sys/kernel/config
sudo mkdir /sys/kernel/config/nullb/disk0
echo 1024 | sudo tee /sys/kernel/config/nullb/disk0/size
echo 1    | sudo tee /sys/kernel/config/nullb/disk0/power   # instantiate
ls /dev/nullb0
```

---

## 1. Internals

### 1.1 Source map

```
drivers/char/misc.c          ★ ~300 lines; read it completely
include/linux/miscdevice.h
drivers/base/faux.c          ★ (6.14+) ~250 lines; the pseudo-device bus
include/linux/device/faux.h
fs/debugfs/inode.c, file.c   ★ the debugfs implementation
include/linux/debugfs.h      ★ note the CONFIG_DEBUG_FS=n inline stubs
fs/tracefs/                  the promoted-out-of-debugfs sibling
fs/configfs/                 userspace-created objects
Documentation/filesystems/debugfs.rst   ★★ short, mandatory
Documentation/filesystems/configfs.rst
Documentation/filesystems/sysfs.rst     (the contrast)
```

### 1.2 `misc_register()`, annotated

```c
int misc_register(struct miscdevice *misc)
{
	...
	if (misc->minor == MISC_DYNAMIC_MINOR) {
		int i = ida_alloc_max(&misc_minors_ida, DYNAMIC_MINORS - 1, GFP_KERNEL);
		misc->minor = DYNAMIC_MINORS - i - 1;     /* allocated downward from 255 */
	}
	dev = MKDEV(MISC_MAJOR, misc->minor);         /* MISC_MAJOR == 10 */

	misc->this_device = device_create_with_groups(&misc_class, misc->parent, dev,
						      misc, misc->groups, "%s", misc->name);
	...
}
```

Note `device_create_with_groups()` — the attributes are created **before** the `KOBJ_ADD`
uevent (Ch. 26 T.5), so `misc->groups` is the correct way to add sysfs attributes to a misc
device. `misc->parent` places it correctly in the `/sys/devices/` topology, which matters for
PM ordering (Ch. 26 T.1) — **set it if you have a real parent device.**

The single shared `file_operations` for major 10 is `misc_fops`, whose `->open` looks up the
minor in a list and **replaces `file->f_op` with yours**, then calls your `->open`. That
`f_op` substitution is a neat trick worth recognizing; it is how one major serves hundreds of
unrelated drivers.

---

## 2. Practice

### Lab 30.1 — The same driver, three ways

Take Lab 29.1's `scullfifo` and re-register it as (a) misc, (b) faux + misc, and compare.

```c
// SPDX-License-Identifier: GPL-2.0
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/miscdevice.h>
#include <linux/fs.h>
#include <linux/kfifo.h>
#include <linux/mutex.h>
#include <linux/poll.h>
#include <linux/slab.h>
#include <linux/uaccess.h>

struct mini {
	struct miscdevice miscdev;        /* ★ embedded, so container_of works */
	struct mutex      lock;
	DECLARE_KFIFO(fifo, char, 1024);
	wait_queue_head_t readq;
	u32               reads, writes;  /* exported via debugfs below */
};
static struct mini *g;                /* ★ NOT devm_: outlives unbind (Ch. 28 T.3) */

static int mini_open(struct inode *i, struct file *f)
{
	/* misc gives us this for free: */
	f->private_data = container_of(f->private_data, struct mini, miscdev);
	stream_open(i, f);
	return 0;
}

static ssize_t mini_read(struct file *f, char __user *ub, size_t n, loff_t *o)
{
	struct mini *m = f->private_data;
	unsigned int copied;
	int ret;

	if (kfifo_is_empty(&m->fifo)) {
		if (f->f_flags & O_NONBLOCK)
			return -EAGAIN;
		ret = wait_event_interruptible(m->readq, !kfifo_is_empty(&m->fifo));
		if (ret)
			return ret;
	}
	guard(mutex)(&m->lock);
	ret = kfifo_to_user(&m->fifo, ub, n, &copied);
	if (!ret)
		m->reads++;
	return ret ? ret : copied;
}

static ssize_t mini_write(struct file *f, const char __user *ub, size_t n, loff_t *o)
{
	struct mini *m = f->private_data;
	unsigned int copied;
	int ret;

	scoped_guard(mutex, &m->lock) {
		ret = kfifo_from_user(&m->fifo, ub, n, &copied);
		if (!ret)
			m->writes++;
	}
	if (!ret && copied)
		wake_up_interruptible(&m->readq);
	return ret ? ret : copied;
}

static __poll_t mini_poll(struct file *f, poll_table *w)
{
	struct mini *m = f->private_data;

	poll_wait(f, &m->readq, w);
	return kfifo_is_empty(&m->fifo) ? EPOLLOUT : (EPOLLIN | EPOLLRDNORM | EPOLLOUT);
}

static const struct file_operations mini_fops = {
	.owner   = THIS_MODULE,
	.open    = mini_open,
	.read    = mini_read,
	.write   = mini_write,
	.poll    = mini_poll,
};

/* sysfs attributes, created BEFORE the uevent via .groups (Ch. 26 T.5) */
static ssize_t used_show(struct device *dev, struct device_attribute *a, char *buf)
{
	struct mini *m = dev_get_drvdata(dev);

	return sysfs_emit(buf, "%u\n", kfifo_len(&m->fifo));
}
static DEVICE_ATTR_RO(used);
static struct attribute *mini_attrs[] = { &dev_attr_used.attr, NULL };
ATTRIBUTE_GROUPS(mini);

static int __init mini_init(void)
{
	int ret;

	g = kzalloc(sizeof(*g), GFP_KERNEL);
	if (!g)
		return -ENOMEM;
	mutex_init(&g->lock);
	INIT_KFIFO(g->fifo);
	init_waitqueue_head(&g->readq);

	g->miscdev.minor  = MISC_DYNAMIC_MINOR;
	g->miscdev.name   = "mini";
	g->miscdev.fops   = &mini_fops;
	g->miscdev.mode   = 0666;
	g->miscdev.groups = mini_groups;        /* ★ */

	ret = misc_register(&g->miscdev);       /* ← FIVE lines replaced forty */
	if (ret) {
		kfree(g);
		return ret;
	}
	dev_set_drvdata(g->miscdev.this_device, g);
	pr_info("registered as minor %d\n", g->miscdev.minor);
	return 0;
}

static void __exit mini_exit(void)
{
	misc_deregister(&g->miscdev);
	mutex_destroy(&g->lock);
	kfree(g);
}
module_init(mini_init); module_exit(mini_exit);
MODULE_LICENSE("GPL");
```
```bash
sudo insmod mini.ko
cat /proc/misc | grep mini
ls -l /dev/mini
ls /sys/class/misc/mini/
cat /sys/class/misc/mini/used
echo hello > /dev/mini && cat /dev/mini
# Compare the line counts:
wc -l scullfifo.c mini.c
```

### Lab 30.2 — Add debugfs to it (T.4)

```c
#include <linux/debugfs.h>

static struct dentry *mini_dbg;

static int mini_state_show(struct seq_file *s, void *v)
{
	struct mini *m = s->private;

	guard(mutex)(&m->lock);
	seq_printf(s, "fifo_size   %u\n", kfifo_size(&m->fifo));
	seq_printf(s, "fifo_used   %u\n", kfifo_len(&m->fifo));
	seq_printf(s, "reads       %u\n", m->reads);
	seq_printf(s, "writes      %u\n", m->writes);
	seq_printf(s, "waiters     %d\n", waitqueue_active(&m->readq));
	return 0;
}
DEFINE_SHOW_ATTRIBUTE(mini_state);

/* A settable knob with custom get/set */
static u64 injected_delay_us;
static int delay_get(void *data, u64 *val) { *val = injected_delay_us; return 0; }
static int delay_set(void *data, u64 val)
{
	if (val > 1000000)
		return -EINVAL;
	injected_delay_us = val;
	return 0;
}
DEFINE_DEBUGFS_ATTRIBUTE(delay_fops, delay_get, delay_set, "%llu\n");

static void mini_debugfs_init(struct mini *m)
{
	/* ★ NO error checking. Ever. */
	mini_dbg = debugfs_create_dir("mini", NULL);
	debugfs_create_u32("reads",  0444, mini_dbg, &m->reads);
	debugfs_create_u32("writes", 0444, mini_dbg, &m->writes);
	debugfs_create_file("state", 0444, mini_dbg, m, &mini_state_fops);
	debugfs_create_file_unsafe("inject_delay_us", 0644, mini_dbg, NULL, &delay_fops);
}
/* teardown: debugfs_remove_recursive(mini_dbg); */
```
```bash
sudo insmod mini.ko
sudo ls /sys/kernel/debug/mini/
sudo cat /sys/kernel/debug/mini/state
echo hi > /dev/mini; sudo cat /sys/kernel/debug/mini/writes
echo 500 | sudo tee /sys/kernel/debug/mini/inject_delay_us

# Prove it compiles away entirely:
./scripts/config -d DEBUG_FS && make M=$PWD modules
nm mini.ko | grep -i debugfs        # → nothing
size mini.ko                        # smaller
```

### Lab 30.3 — A `faux` device (T.2)

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>
#include <linux/device/faux.h>

struct fauxpriv { u32 counter; };

static ssize_t counter_show(struct device *dev, struct device_attribute *a, char *buf)
{
	struct fauxpriv *p = dev_get_drvdata(dev);

	return sysfs_emit(buf, "%u\n", p->counter++);
}
static DEVICE_ATTR_RO(counter);
static struct attribute *fx_attrs[] = { &dev_attr_counter.attr, NULL };
ATTRIBUTE_GROUPS(fx);

static int fx_probe(struct faux_device *fdev)
{
	struct fauxpriv *p = devm_kzalloc(&fdev->dev, sizeof(*p), GFP_KERNEL);

	if (!p)
		return -ENOMEM;
	dev_set_drvdata(&fdev->dev, p);
	dev_info(&fdev->dev, "faux device probed — no hardware here\n");
	return 0;
}
static void fx_remove(struct faux_device *fdev)
{
	dev_info(&fdev->dev, "faux device removed\n");
}

static const struct faux_device_ops fx_ops = { .probe = fx_probe, .remove = fx_remove };
static struct faux_device *fx;

static int __init fx_init(void)
{
	fx = faux_device_create_with_groups("mypseudo", NULL, &fx_ops, fx_groups);
	return fx ? 0 : -ENODEV;
}
static void __exit fx_exit(void) { faux_device_destroy(fx); }
module_init(fx_init); module_exit(fx_exit);
MODULE_LICENSE("GPL");
```
```bash
sudo insmod fauxdemo.ko
ls /sys/devices/faux/
cat /sys/devices/faux/mypseudo/counter
dmesg | tail -2

# Contrast with the old way:
# platform_device_register_simple("mypseudo", -1, NULL, 0);
# → lands in /sys/bus/platform/devices/ among REAL hardware
ls /sys/bus/platform/devices/ | head -20
# For each, decide: is it real hardware, or should it be faux?
```

### Lab 30.4 — Find platform-bus abuse in the tree (T.2)

```bash
# Drivers creating a platform device for themselves with no firmware node:
git grep -n 'platform_device_register_simple\|platform_device_alloc' -- drivers/ | head -30

# Cross-reference: does it have an of_match_table or acpi_match_table?
# If not, it is almost certainly a pseudo-device and a faux candidate.
for f in $(git grep -l 'platform_device_register_simple' -- drivers/ | head -20); do
  if ! grep -q 'of_device_id\|acpi_device_id' "$f"; then echo "FAUX CANDIDATE: $f"; fi
done

git log --oneline --grep='convert.*faux\|faux bus' | head
$EDITOR drivers/base/faux.c
```
Pick one and write the conversion patch. This is a genuinely welcome contribution right now.

### Lab 30.5 — The debugfs lifetime hazard (T.4)

```c
/* ★ BROKEN: debugfs file points at devm_ memory */
static int bad_probe(struct platform_device *pdev)
{
	struct priv *p = devm_kzalloc(&pdev->dev, sizeof(*p), GFP_KERNEL);

	p->dbg = debugfs_create_dir(dev_name(&pdev->dev), NULL);
	debugfs_create_u32("value", 0444, p->dbg, &p->value);   /* points INTO p */
	return 0;                                               /* no removal! */
}
/* unbind → devres frees p → the debugfs file still points at freed memory */
```
```bash
./scripts/config -e KASAN -e DEBUG_FS
sudo insmod baddbg.ko
sudo cat /sys/kernel/debug/baddbg.0/value      # fine
echo baddbg.0 | sudo tee /sys/bus/platform/drivers/baddbg/unbind
sudo cat /sys/kernel/debug/baddbg.0/value      # ★ KASAN use-after-free
dmesg | grep -A25 KASAN
```
Fix it three ways and compare: (a) `debugfs_remove_recursive()` in `remove()`,
(b) `devm_add_action_or_reset()`, (c) don't use `devm_` for the exported state. Explain which
is best and why.

### Lab 30.6 — Explore real debugfs

```bash
sudo ls /sys/kernel/debug/
sudo ls /sys/kernel/debug/tracing/            # → actually tracefs, bind-mounted
sudo cat /sys/kernel/debug/devices_deferred   # Ch. 27
sudo ls /sys/kernel/debug/block/*/            # blk-mq internals (Ch. 63)
sudo ls /sys/kernel/debug/dri/0/              # DRM state (Ch. 47)
sudo cat /sys/kernel/debug/gpio               # every GPIO, its direction and value
sudo cat /sys/kernel/debug/clk/clk_summary    # ★ the entire clock tree (Ch. 43)
sudo cat /sys/kernel/debug/pinctrl/*/pinmux-pins | head
sudo cat /sys/kernel/debug/regmap/*/registers | head
sudo cat /sys/kernel/debug/usb/devices | head -30
sudo ls /sys/kernel/debug/kvm/
sudo cat /sys/kernel/debug/sched/debug | head -40
sudo ls /sys/kernel/debug/dynamic_debug/

# How much of the kernel exposes debugfs?
git grep -l 'debugfs_create' -- drivers/ | wc -l
```
**`clk_summary`, `gpio`, `pinmux-pins`, and `regmap/*/registers` are the four debugfs files
you will use most in embedded bring-up** (Ch. 101). Learn them now.

### Lab 30.7 — configfs: create an object from userspace (T.5)

```bash
sudo modprobe null_blk nr_devices=0
sudo mount -t configfs none /sys/kernel/config 2>/dev/null
ls /sys/kernel/config/nullb/

sudo mkdir /sys/kernel/config/nullb/lab0
ls /sys/kernel/config/nullb/lab0/            # attributes appeared with the mkdir
echo 512  | sudo tee /sys/kernel/config/nullb/lab0/size        # MiB
echo 4096 | sudo tee /sys/kernel/config/nullb/lab0/blocksize
echo 4    | sudo tee /sys/kernel/config/nullb/lab0/submit_queues
echo 1    | sudo tee /sys/kernel/config/nullb/lab0/power       # ★ instantiate
lsblk | grep nullb
sudo fio --name=t --filename=/dev/nullb0 --rw=randread --bs=4k --runtime=5 --time_based

echo 0 | sudo tee /sys/kernel/config/nullb/lab0/power
sudo rmdir /sys/kernel/config/nullb/lab0
```
Compare with sysfs: **you created a kernel object with `mkdir`.** Read
`drivers/block/null_blk/main.c`'s configfs code and `Documentation/filesystems/configfs.rst`,
then explain when configfs is the right answer (hint: when the *set* of objects is defined by
userspace policy, not by hardware).

---

## 3. Mastery drills

1. **Read `drivers/char/misc.c`** completely (~300 lines). Explain the `f_op` substitution in
   `misc_open()` and why it works. What happens if two drivers request the same static minor?

2. **Read `drivers/base/faux.c`** completely. List everything a `faux_device` does *not* have
   compared to a `platform_device`, and explain why each absence is correct.

3. **Count the abuse.** On your machine, list every `/sys/bus/platform/devices/` entry and
   classify each as real hardware or pseudo-device. Report the ratio.

4. **The stability argument.** Write 400 words for a colleague who says "just put it in sysfs,
   debugfs might not be mounted." Cover: what the ABI commitment costs, what tracefs's split
   from debugfs teaches, and when promotion to sysfs is correct.

5. **Find a debugfs-as-ABI violation.** Search for userspace tools that read
   `/sys/kernel/debug` (e.g. some GPU tools, some tracing wrappers). What happens when the
   file changes? Is there a stable alternative they should use?

6. **`_unsafe` semantics.** Read `fs/debugfs/file.c`. Explain exactly what
   `debugfs_create_file()` wraps that `debugfs_create_file_unsafe()` does not, and construct
   the race the wrapper prevents.

7. **misc minor exhaustion.** `misc` has 256 minors (fewer, since some are static). Compute
   how many are currently used on your system. What happens when it runs out, and what should
   a driver do about it?

8. **Design question.** You have a driver with: 3 configuration values users must set in
   production, 40 internal counters useful only to you, a register dump for debugging, and a
   way to inject faults for testing. Place each in the right interface and justify.

9. **Convert a driver.** Find a driver using the full `cdev` dance for a single node. Convert
   it to `misc`. Measure the diff. Verify `/dev` node, permissions, and sysfs attributes are
   unchanged (this is ABI — Ch. 24 T.1).

10. **The `.parent` field.** Find a misc driver that sets `miscdevice.parent` and one that
    does not. Show the difference in `/sys/devices/` placement and explain the PM-ordering
    consequence (Ch. 26 T.1).

---

## 4. Further reading

**Kernel documentation:**
- `Documentation/filesystems/debugfs.rst` ★★ — short; read it before using debugfs
- `Documentation/filesystems/configfs.rst` ★
- `Documentation/filesystems/sysfs.rst` — the contrast
- `Documentation/admin-guide/devices.txt` — the static major/minor registry
- `Documentation/driver-api/driver-model/platform.rst` — what platform *is* for (Ch. 31)

**Source:**
- `drivers/char/misc.c` ★, `include/linux/miscdevice.h`
- `drivers/base/faux.c` ★ (6.14+), `include/linux/device/faux.h`
- `fs/debugfs/inode.c`, `fs/debugfs/file.c`
- `drivers/block/null_blk/main.c` — an exemplary configfs user
- `drivers/clk/clk.c` (`clk_summary`), `drivers/gpio/gpiolib.c` (the `gpio` debugfs file) —
  examples of debugfs done well

**LWN:**
- "The faux bus" / "A new bus for fake devices" (2025)
- "debugfs and the stable-ABI question"
- "Moving tracing out of debugfs" — the tracefs split, and what it teaches
- "configfs: a filesystem for creating kernel objects"

→ Next: [31-platform-devices.md](31-platform-devices.md)

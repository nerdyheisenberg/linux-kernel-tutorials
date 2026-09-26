# Chapter 26 — Driver Model Foundations: kobject, kset, sysfs, uevent

> **Goal:** understand the object model that every driver in Linux is built on — why it
> exists, what `kobject` really is, how sysfs is generated from it, and how userspace learns
> that a device appeared.

---

## Theory & First Principles

### T.0 — Start here: why not just a list of drivers?

A kernel has drivers and hardware. The naive design is a list:

```c
struct driver *drivers[] = { &e1000_driver, &nvme_driver, &i2c_gpio_driver, ... };
for (i = 0; drivers[i]; i++)
	drivers[i]->probe();     /* "is your hardware there? if so, claim it" */
```

That is literally how Linux worked before 2.5. Now ask the questions a real system asks, and
watch it fall apart:

| Question | The list cannot answer |
|---|---|
| "Suspend the machine." | In **what order**? A USB mouse must suspend before the USB controller, which must suspend before the PCI bridge it is behind. The list has no notion of *behind* |
| "Turn off power to this device." | Which clock and which regulator feed it? Is anything else using them? |
| "A device was hot-unplugged." | Which other devices just disappeared with it? |
| "Show me the hardware." | There is no representation to show — `/sys` cannot exist |
| "This USB device moved to another port." | The list has no identity for a device, only for a driver |
| "Load the module for whatever this is." | Nothing maps a device ID to a module name |

**Every one of those questions is really the same question: *what is the structure?*** The
list has drivers but no *devices*, and no relationships between them.

**The device model is the answer: make the topology a first-class data structure.**

```
                      /sys/devices/
                            │
                    pci0000:00                     <- a bus controller
                            │
                    0000:00:14.0                   <- a USB host controller
                            │
                    usb1 ────┬──── 1-1          <- a hub, and a device on it
                            │        │
                            │        └─ 1-1:1.0    <- an interface
                            └──── 1-2
```

Now every question above has a mechanical answer. Suspend order is a **post-order traversal**
of that tree. Hot-unplug removes a subtree. `/sys` *is* the tree. Module autoloading is a
lookup keyed on the device's ID, emitted as a uevent. **One data structure, six problems.**

**Three objects and one operation** make the whole thing work:

```c
struct bus_type  { ... int (*match)(struct device *, struct device_driver *); ... };
struct device         { struct device *parent; struct bus_type *bus; ... };
struct device_driver  { struct bus_type *bus;  int (*probe)(struct device *); ... };
```

> **Devices and drivers are registered independently, in any order. The *bus* matches them.**
> When a match happens, `probe()` runs. That is the entire protocol.

That indirection is what makes hotplug and modular drivers possible: a device can appear
before its driver is loaded, or after, and neither has to know about the other. `§T.3` works
through the matching; Ch. 27 works through what `probe` must do.

**Look at the real thing right now** — it makes the abstraction concrete in a way no diagram
does:

```bash
ls /sys/devices/                              # the roots
find /sys/devices -name 'driver' | head -5    # bound devices -> their drivers
ls -l /sys/bus/pci/devices/ | head            # symlinks INTO the tree
ls /sys/bus/                                  # every bus type registered

# The tree, as a tree:
find /sys/devices/pci0000:00 -maxdepth 3 -name 'uevent' | sed 's|/uevent||' | head -20

# What a device tells userspace about itself (this is what udev reads):
cat /sys/bus/pci/devices/*/uevent | head -20
```

**The one thing to carry into §T.1:** the device model is not about drivers. It is about
making *structure* explicit so that generic code — power management, hotplug, sysfs, device
links, autoloading — can be written once instead of once per driver.

---

### T.1 The problem the device model was invented to solve

Before Linux 2.5 there was **no unified representation of a device**. Each bus had its own
private list: `pci_devices`, a USB tree, SCSI hosts, platform devices hardcoded in board
files. Nothing knew the relationship between them.

That was tolerable until **power management** arrived, and PM turned out to be the forcing
function. To suspend a laptop you must answer a question no one could answer:

> *In what order do I suspend these devices?*

You cannot suspend a USB host controller before the USB mouse hanging off it (the mouse's
suspend needs the bus to still work). You cannot resume a disk before the SATA controller.
You cannot power-gate an I²C bus before the PMIC on it is quiesced. **Correct suspend/resume
requires a complete, ordered, parent→child topology of every device in the system.**

Patrick Mochel's device model (2.5, ~2002) built exactly that. Every other benefit —
sysfs, udev, hotplug, `/dev` auto-creation, driver binding, runtime PM, device links,
deferred probe — came out of having the topology. This is worth remembering as a design
lesson: **a seemingly boring bookkeeping structure unlocked a decade of features**, because
the topology was the missing shared abstraction.

```bash
# The topology, today:
ls /sys/devices/                          # the roots
find /sys/devices -maxdepth 4 -name 'power' | head
cat /sys/power/pm_print_times             # then: dmesg | grep 'PM:' after a suspend
```

### T.2 `kobject`: a base class in C

Ch. 08 T.3 established embedding + `container_of` as the kernel's object model. `kobject` is
that model's canonical base class — the thing you embed when your object needs
**identity, lifetime, and a place in a hierarchy**.

```c
struct kobject {
	const char              *name;       /* the sysfs directory name */
	struct list_head        entry;       /* membership in a kset */
	struct kobject          *parent;     /* ★ the hierarchy edge */
	struct kset             *kset;       /* the containing set (also default parent) */
	const struct kobj_type  *ktype;      /* ★ the vtable */
	struct kernfs_node      *sd;         /* the sysfs directory node */
	struct kref             kref;        /* ★ lifetime */
	unsigned int state_initialized:1, state_in_sysfs:1,
		     state_add_uevent_sent:1, state_remove_uevent_sent:1,
		     uevent_suppress:1;
};
```

Exactly four responsibilities, and it is worth naming them precisely because people assume it
does more:

1. **Reference counting** (`kref`) — Ch. 12's pattern, embedded.
2. **A name** — which becomes a sysfs directory name.
3. **A parent pointer** — which becomes the sysfs directory nesting. *This is the topology
   from T.1.*
4. **A type** (`ktype`) — the vtable carrying `release()`, the attribute set, and the sysfs
   ops.

```c
struct kobj_type {
	void (*release)(struct kobject *kobj);            /* ★ MANDATORY */
	const struct sysfs_ops *sysfs_ops;                /* how to read/write attributes */
	const struct attribute_group **default_groups;    /* attributes created automatically */
	const struct kobj_ns_type_operations *(*child_ns_type)(const struct kobject *);
	const void *(*namespace)(const struct kobject *); /* network-namespace awareness */
	void (*get_ownership)(const struct kobject *, kuid_t *, kgid_t *);
};
```

**The `release()` rule is absolute and is the #1 kobject bug:**

> When the last reference is dropped, `ktype->release()` is called and **it must free the
> containing structure**. The kobject core will *not* free it for you, and you must *never*
> `kfree()` a kobject-containing object directly.

Violating it produces the loudest warning in the kernel:

```
kobject: 'foo' (00000000deadbeef): does not have a release() function,
it is broken and must be fixed. See Documentation/core-api/kobject.rst.
```

Why so strict? Because a kobject's references are held by things you do not control —
an open sysfs file, a udev process, a symlink from another device. Freeing while any of them
holds a reference is a UAF reachable from unprivileged userspace (`cat` a sysfs file during
device removal). The design forces the free through the refcount.

### T.3 Two orthogonal taxonomies over one object set

This is the structural insight that makes `/sys` legible, and almost nobody explains it.

The same set of devices is organized in **two independent ways**:

| Taxonomy | Question it answers | sysfs representation |
|---|---|---|
| **Hierarchy** (parent/child) | *How is this physically connected?* | directory **nesting** under `/sys/devices/` |
| **Classification** (class/bus/driver) | *What kind of thing is this? Who drives it?* | **symlinks** under `/sys/class/`, `/sys/bus/` |

```
/sys/devices/pci0000:00/0000:00:1f.2/ata1/host0/target0:0:0/0:0:0:0/block/sda
   └── the PHYSICAL path: PCI root → AHCI controller → ATA port → SCSI → block device

/sys/class/block/sda      -> ../../devices/pci0000:00/.../block/sda     (symlink)
/sys/bus/pci/devices/0000:00:1f.2 -> ../../../devices/pci0000:00/0000:00:1f.2
/sys/bus/pci/drivers/ahci/0000:00:1f.2 -> ../../../../devices/pci0000:00/0000:00:1f.2
```

**There is exactly one real directory per object** (under `/sys/devices/`), and every other
view is a symlink into it. That is why `/sys` can present many orthogonal classifications
without duplicating state — a filesystem-level implementation of *one object, many indices*
(cf. Ch. 20 T.1's resource unbundling, and Ch. 09 T.1's "one object on many lists").

Three registries, each a different index over the same devices:

- **`/sys/bus/<bus>/devices/`** — devices *on* that bus (the discovery/matching axis)
- **`/sys/bus/<bus>/drivers/<drv>/`** — devices *bound to* that driver
- **`/sys/class/<class>/`** — devices offering that *functional interface*
  (`net`, `block`, `tty`, `input`, `hwmon`, `leds`, `thermal`)

The bus/class distinction matters and is frequently confused: **a bus is how you found it; a
class is what it does.** A network card can be on PCI, USB, or platform — three buses, one
class. Userspace almost always wants the class.

### T.4 sysfs: a filesystem projection of an object graph

sysfs is not a filesystem that stores things; it is a **live, generated view** of the kobject
graph. The mapping is mechanical:

| Kernel object | sysfs artifact |
|---|---|
| `kobject` | a directory |
| `kobject->parent` | the directory's location |
| `struct attribute` | a file in that directory |
| `struct bin_attribute` | a binary file (firmware, EEPROM, `config` space) |
| `kobject` → `kobject` reference | a symlink |
| `kset` | a directory whose members are kobjects, plus a uevent namespace |

Reads and writes are dispatched through `ktype->sysfs_ops` to your `show()`/`store()`:

```c
static ssize_t mode_show(struct device *dev, struct device_attribute *attr, char *buf)
{
	struct mydev *d = dev_get_drvdata(dev);

	return sysfs_emit(buf, "%u\n", READ_ONCE(d->mode));
}

static ssize_t mode_store(struct device *dev, struct device_attribute *attr,
			  const char *buf, size_t count)
{
	struct mydev *d = dev_get_drvdata(dev);
	unsigned int v;
	int ret = kstrtouint(buf, 0, &v);

	if (ret)
		return ret;
	if (v > MODE_MAX)
		return -EINVAL;
	guard(mutex)(&d->lock);
	d->mode = v;
	return count;                     /* ★ return the byte count, not 0 */
}
static DEVICE_ATTR_RW(mode);
```

The rules (Ch. 24 T.8 established the *why*; these are the mechanics):

- **One value per file.** No atomicity across files — if you need it, sysfs is wrong.
- **≤ `PAGE_SIZE` per `show()`.** Use `sysfs_emit()`/`sysfs_emit_at()`, never `sprintf()`.
- **`store()` returns the count consumed**, or a negative errno. Returning 0 makes the writer
  loop forever.
- Both may sleep; both run in task context with no core-held locks.
- **Every attribute must be documented in `Documentation/ABI/`** — it is permanent ABI.
- The file's existence is itself ABI: userspace probes for features by `stat()`ing
  attributes.

**`kernfs`** is the backing implementation (split out of sysfs so cgroupfs could reuse it).
It solves a hard problem: an attribute file can be open while the underlying kobject is
removed. `kernfs` handles this with **active references** — `kernfs_get_active()` /
`kernfs_put_active()` around every callback, and `kernfs_remove()` *waits* for active
references to drain. That is the Ch. 25 P12 quiescence pattern, implemented once in the VFS
layer so drivers never have to think about it. It is also why `sysfs_remove_file()` from
within a `store()` handler on the same file deadlocks (there is a `sysfs_break_active_protection()`
escape hatch for exactly that case — `/sys/bus/pci/devices/*/remove` uses it).

### T.5 uevent: notification, and the race it creates

When a kobject appears, moves, or disappears, the kernel broadcasts a **uevent** — a set of
`KEY=value` strings — over a netlink socket (`NETLINK_KOBJECT_UEVENT`, multicast group 1).

```
ACTION=add
DEVPATH=/devices/pci0000:00/0000:00:1f.2/ata1/host0/.../block/sda
SUBSYSTEM=block
DEVNAME=sda
DEVTYPE=disk
MAJOR=8
MINOR=0
SEQNUM=1234
```

`udev` listens, applies rules, and creates `/dev` nodes, sets permissions, runs programs, and
maintains `/dev/disk/by-*` symlinks. **`/dev` is entirely a userspace construct**; the kernel
only announces `MAJOR`/`MINOR` (and provides `devtmpfs` as a bootstrap so a minimal system
has nodes before udev starts).

Actions: `add`, `remove`, `change`, `move`, `online`, `offline`, `bind`, `unbind`.
`bind`/`unbind` (3.x+) are separate from `add`/`remove` precisely because a device can exist
without a driver — which is the T.1 topology again: presence and binding are independent
facts.

**The race that shapes the API.** The `add` uevent is sent when the device is added to sysfs.
If your driver then creates attributes in `probe()`, udev may already have run its rules and
**missed them**:

```c
/* ★ BROKEN — races with udev */
static int my_probe(struct platform_device *pdev)
{
	...
	device_create_file(&pdev->dev, &dev_attr_mode);   /* too late! */
}

/* ★ CORRECT — the core creates them BEFORE sending the uevent */
static struct attribute *mydev_attrs[] = { &dev_attr_mode.attr, NULL };
ATTRIBUTE_GROUPS(mydev);

static struct platform_driver my_driver = {
	.driver = {
		.name       = "mydev",
		.dev_groups = mydev_groups,      /* ★ */
	},
	.probe = my_probe,
};
```

`device_add()` creates `dev_groups`, *then* sends `KOBJ_ADD`. So the attributes are guaranteed
visible to the uevent consumer. **`device_create_file()` in `probe()` is a review rejection**,
and there is an ongoing tree-wide conversion away from it — a good first-patch area.

For attributes whose *existence* is conditional, `attribute_group->is_visible()` lets you hide
individual attributes while still using the group mechanism.

If you genuinely must change something after the fact, announce it:
```c
sysfs_notify(&dev->kobj, NULL, "status");     /* wakes poll()ers on that attribute */
kobject_uevent(&dev->kobj, KOBJ_CHANGE);      /* tells udev to re-run rules */
```
`sysfs_notify()` + `poll()`/`select()` on a sysfs file with `POLLPRI` is a legitimate,
widely-used event mechanism for low-rate state changes (thermal trips, cable detect, LED
triggers).

### T.6 `kset`, and what it adds over a bare parent pointer

A `kset` is a kobject that is also a **container** of kobjects, with an associated
`uevent_ops`:

```c
struct kset {
	struct list_head        list;       /* its members */
	spinlock_t              list_lock;
	struct kobject          kobj;       /* it IS a kobject (embedding again) */
	const struct kset_uevent_ops *uevent_ops;   /* ★ filter/name/add env for members */
};
```

It provides three things a plain parent pointer does not:

1. **An enumerable set** — you can iterate the members.
2. **A default parent** — adding a kobject to a kset sets its parent if unset.
3. **A uevent policy for the whole set** — `->filter()` (suppress events),
   `->name()` (override `SUBSYSTEM=`), `->uevent()` (add environment variables).

That third point is the real reason `kset` exists: **uevent policy is a property of a
collection, not of an individual object**. `bus_type` and `class` each own a kset and use
`uevent_ops` to add subsystem-specific variables (`MODALIAS=`, `DEVTYPE=`, `PCI_ID=`) that
udev and module autoloading depend on.

```bash
ls /sys/kernel/                # ksets not tied to devices: mm, slab, debug, ...
cat /sys/class/block/sda/uevent        # the environment the kernel would send
udevadm info -q all -n /dev/sda
udevadm monitor --kernel --property    # watch raw kernel uevents live
```

### T.7 The layering: `kobject` → `device` → bus-specific

Almost no driver uses `kobject` directly. The layering is:

```
        struct kobject            identity, refcount, parent, sysfs
              ▲ embedded in
        struct device             + bus, driver, class, PM, DMA, fwnode, devres
              ▲ embedded in
   struct pci_dev / platform_device / usb_interface / i2c_client / spi_device ...
              ▲ embedded in
        your driver's private struct (or pointed to by drvdata)
```

```c
struct device {
	struct kobject           kobj;
	struct device           *parent;          /* ★ the topology (T.1) */
	struct device_private   *p;
	const char              *init_name;
	const struct device_type *type;
	const struct bus_type   *bus;             /* ★ how it is discovered/matched */
	struct device_driver    *driver;          /* ★ what is bound to it (may be NULL) */
	const struct class      *class;           /* ★ what interface it offers */
	void                    *platform_data, *driver_data;
	struct dev_pm_info       power;           /* runtime PM state (Ch. 48) */
	struct dev_pm_domain    *pm_domain;
	struct device_node      *of_node;         /* device tree (Ch. 32) */
	struct fwnode_handle    *fwnode;          /* ★ unified DT/ACPI/swnode handle */
	u64                     *dma_mask;
	const struct dma_map_ops *dma_ops;        /* Ch. 35 */
	struct list_head         devres_head;     /* ★ Ch. 28 */
	struct list_head         links;           /* device links (Ch. 27) */
	dev_t                    devt;            /* for /dev node creation */
	void   (*release)(struct device *dev);    /* ★ still mandatory */
};
```

**When to use a bare `kobject`:** almost never. The guidance in
`Documentation/core-api/kobject.rst` is explicit — if you want a directory in sysfs for
something that is *not* a device, use `kobject_create_and_add()`; if it *is* a device, use
`struct device`. Writing a custom `ktype` is a strong signal you are doing something unusual.

### T.8 Naming is ABI

A device's sysfs name and path become things userspace scripts depend on (Ch. 24 T.1,
Hyrum's Law). Consequences:

- **Enumeration order is not stable.** `eth0`/`eth1` depend on probe order, which depends on
  PCI enumeration, link order, and parallelism. This is why systemd introduced
  **predictable network interface names** (`enp3s0` = PCI bus 3 slot 0) derived from the
  *topology* — the T.1 topology solving a naming problem.
- `/dev/sda` is likewise unstable; `/dev/disk/by-uuid/` and `by-path/` exist for this reason.
- **Renaming a device is an ABI change.** `device_rename()` exists but is documented as a bad
  idea (it breaks udev rules and open paths).
- Pick a naming scheme at design time that encodes the *stable* property (topology, serial,
  UUID), not the *discovery order*.

---

## 1. Internals

### 1.1 Source map

```
lib/kobject.c                 ★ kobject_init, kobject_add, kobject_put, the release path
lib/kobject_uevent.c          ★ uevent generation and netlink broadcast
include/linux/kobject.h       ★ read the whole header
drivers/base/core.c           ★★ device_add, device_del, device_register, dev_groups
drivers/base/bus.c            bus_type, driver binding (Ch. 27)
drivers/base/class.c          class registration, /sys/class
drivers/base/devres.c         managed resources (Ch. 28)
drivers/base/dd.c             ★ driver binding, deferred probe (Ch. 27)
drivers/base/power/           PM core, suspend ordering (Ch. 48)
fs/sysfs/                     the filesystem layer
fs/kernfs/                    ★ the backing store; read dir.c for active references
include/linux/device.h        ★★ struct device, DEVICE_ATTR_*, dev_err/dev_dbg
include/linux/sysfs.h         attribute macros, sysfs_emit
Documentation/core-api/kobject.rst          ★★ mandatory
Documentation/filesystems/sysfs.rst         ★★ mandatory
Documentation/driver-api/driver-model/      ★ overview, binding, device, driver, bus, class
Documentation/ABI/                          the contract
```

### 1.2 `device_add()` — the order matters

```
device_register(dev) = device_initialize(dev) + device_add(dev)

device_add():
  1. kobject_add()                     create the sysfs directory
  2. device_create_file(dev, uevent)   the "uevent" attribute
  3. device_add_class_symlinks()       /sys/class/<c>/<name> -> ...
  4. device_add_attrs()                ★ class/type/dev_groups attributes
  5. device_create_sys_dev_entry()     /sys/dev/{char,block}/MAJ:MIN
  6. devtmpfs_create_node()            the /dev node (if CONFIG_DEVTMPFS)
  7. kobject_uevent(KOBJ_ADD)          ★ ONLY NOW is userspace told
  8. bus_probe_device()                → try to bind a driver (Ch. 27)
```

Step 4 before step 7 is the guarantee from T.5. Step 8 *after* step 7 is why `bind`/`unbind`
uevents are separate from `add`/`remove`.

`device_del()` runs it in reverse, with `KOBJ_REMOVE` sent *before* the sysfs directory is
torn down, so udev can still read attributes while handling the removal.

---

## 2. Practice

### Lab 26.1 — A bare kobject with attributes, from scratch

```c
// SPDX-License-Identifier: GPL-2.0
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/kobject.h>
#include <linux/sysfs.h>
#include <linux/slab.h>

struct my_obj {
	struct kobject kobj;
	int            value;
	char           label[32];
};
#define to_my_obj(x) container_of(x, struct my_obj, kobj)

/* Our own attribute type, so show/store get the right object type. */
struct my_attribute {
	struct attribute attr;
	ssize_t (*show)(struct my_obj *o, struct my_attribute *a, char *buf);
	ssize_t (*store)(struct my_obj *o, struct my_attribute *a, const char *buf, size_t n);
};
#define to_my_attr(x) container_of(x, struct my_attribute, attr)

/* The sysfs_ops bridge: kobject+attribute -> my_obj+my_attribute */
static ssize_t my_attr_show(struct kobject *kobj, struct attribute *attr, char *buf)
{
	struct my_attribute *a = to_my_attr(attr);

	return a->show ? a->show(to_my_obj(kobj), a, buf) : -EIO;
}
static ssize_t my_attr_store(struct kobject *kobj, struct attribute *attr,
			     const char *buf, size_t n)
{
	struct my_attribute *a = to_my_attr(attr);

	return a->store ? a->store(to_my_obj(kobj), a, buf, n) : -EIO;
}
static const struct sysfs_ops my_sysfs_ops = {
	.show = my_attr_show, .store = my_attr_store,
};

static ssize_t value_show(struct my_obj *o, struct my_attribute *a, char *buf)
{ return sysfs_emit(buf, "%d\n", o->value); }

static ssize_t value_store(struct my_obj *o, struct my_attribute *a,
			   const char *buf, size_t n)
{
	int ret = kstrtoint(buf, 0, &o->value);

	return ret ? ret : n;
}

static ssize_t label_show(struct my_obj *o, struct my_attribute *a, char *buf)
{ return sysfs_emit(buf, "%s\n", o->label); }

static struct my_attribute value_attr = __ATTR(value, 0664, value_show, value_store);
static struct my_attribute label_attr = __ATTR_RO(label);

static struct attribute *my_attrs[] = {
	&value_attr.attr, &label_attr.attr, NULL,
};
ATTRIBUTE_GROUPS(my);

/* ★ THE MANDATORY RELEASE FUNCTION */
static void my_release(struct kobject *kobj)
{
	struct my_obj *o = to_my_obj(kobj);

	pr_info("releasing '%s' — freeing now\n", kobject_name(kobj));
	kfree(o);
}

static const struct kobj_type my_ktype = {
	.release        = my_release,
	.sysfs_ops      = &my_sysfs_ops,
	.default_groups = my_groups,
};

static struct kset *my_kset;
static struct my_obj *obj_a, *obj_b;

static struct my_obj *my_obj_create(const char *name, int val)
{
	struct my_obj *o = kzalloc(sizeof(*o), GFP_KERNEL);
	int ret;

	if (!o)
		return NULL;
	o->value = val;
	strscpy(o->label, name, sizeof(o->label));
	o->kobj.kset = my_kset;                  /* sets the default parent too */

	ret = kobject_init_and_add(&o->kobj, &my_ktype, NULL, "%s", name);
	if (ret) {
		kobject_put(&o->kobj);           /* ★ put, NOT kfree — release() frees */
		return NULL;
	}
	kobject_uevent(&o->kobj, KOBJ_ADD);      /* ★ announce AFTER attributes exist */
	return o;
}

static int __init kobj_init(void)
{
	my_kset = kset_create_and_add("my_objects", NULL, kernel_kobj);
	if (!my_kset)
		return -ENOMEM;
	obj_a = my_obj_create("alpha", 1);
	obj_b = my_obj_create("beta", 2);
	if (!obj_a || !obj_b) {
		kset_unregister(my_kset);
		return -ENOMEM;
	}
	return 0;
}

static void __exit kobj_exit(void)
{
	kobject_put(&obj_a->kobj);       /* release() will kfree */
	kobject_put(&obj_b->kobj);
	kset_unregister(my_kset);
}
module_init(kobj_init); module_exit(kobj_exit);
MODULE_LICENSE("GPL");
```
```bash
sudo insmod kobjdemo.ko
tree /sys/kernel/my_objects/
cat /sys/kernel/my_objects/alpha/value
echo 42 | sudo tee /sys/kernel/my_objects/alpha/value
cat /sys/kernel/my_objects/alpha/value
sudo udevadm monitor --kernel &          # watch the KOBJ_ADD arrive
sudo rmmod kobjdemo && dmesg | tail -4
```

### Lab 26.2 — Trigger the "no release function" warning

Delete `.release` from `my_ktype`, rebuild, and load:
```bash
sudo insmod kobjdemo.ko
dmesg | grep -A5 'does not have a release'
```
Then do the *other* half: keep `release()` but `kfree(o)` directly instead of
`kobject_put()`, build with KASAN, and `cat` an attribute during unload:
```bash
( while true; do cat /sys/kernel/my_objects/alpha/value >/dev/null 2>&1; done ) &
sudo rmmod kobjdemo
dmesg | grep -A20 KASAN
```
**This is the exact UAF the rule exists to prevent**, and it is reachable by an unprivileged
`cat`.

### Lab 26.3 — Map the two taxonomies (T.3)

```bash
# Follow one device through BOTH views
DEV=$(ls /sys/class/net | grep -v lo | head -1)
echo "class view : /sys/class/net/$DEV"
readlink -f /sys/class/net/$DEV
echo "bus view   :"; readlink -f /sys/class/net/$DEV/device
echo "driver     :"; readlink -f /sys/class/net/$DEV/device/driver

# Walk UP the physical topology
P=$(readlink -f /sys/class/net/$DEV)
while [ "$P" != "/sys/devices" ] && [ -n "$P" ]; do
  echo "  $P   [$(cat $P/subsystem/../subsystem 2>/dev/null; basename $(readlink -f $P/subsystem 2>/dev/null) 2>/dev/null)]"
  P=$(dirname $P)
done

# Every index over the same objects:
ls /sys/bus/                          # discovery axis
ls /sys/class/                        # functional axis
ls /sys/devices/                      # the ONE real hierarchy
ls -l /sys/bus/pci/devices/ | head    # all symlinks
ls -l /sys/bus/pci/drivers/*/ | head -20

# Count: how many symlinks vs real dirs?
find /sys/class -maxdepth 2 -type l | wc -l
find /sys/devices -maxdepth 6 -type d | wc -l
```

### Lab 26.4 — Watch uevents and the udev pipeline

```bash
# Raw kernel uevents (before udev processes them)
sudo udevadm monitor --kernel --property &

# udev's view (after rules)
sudo udevadm monitor --udev --property &

# Now generate events:
sudo modprobe -r usb_storage; sudo modprobe usb_storage
# or plug a USB device, or:
echo change | sudo tee /sys/class/net/$DEV/uevent

# Replay an event for one device
sudo udevadm trigger --action=change --subsystem-match=net

# What environment does the kernel provide?
cat /sys/class/net/$DEV/uevent
udevadm info -q all -p /sys/class/net/$DEV

# Which udev rules matched?
sudo udevadm test /sys/class/net/$DEV 2>&1 | head -40

# Predictable naming in action (T.8):
udevadm info -q property /sys/class/net/$DEV | grep -E 'ID_NET_NAME|ID_PATH'
```

### Lab 26.5 — `dev_groups` vs `device_create_file` (T.5)

Write two versions of a platform driver, one using each, and prove the race:

```bash
# With device_create_file() in probe():
cat > /etc/udev/rules.d/99-test.rules <<'EOF'
SUBSYSTEM=="platform", ACTION=="add", KERNEL=="mydev*", \
  RUN+="/bin/sh -c 'ls /sys%p/ > /tmp/udev-saw-$kernel.txt 2>&1'"
EOF
sudo udevadm control --reload
sudo insmod mydev_bad.ko
cat /tmp/udev-saw-*.txt          # the 'mode' attribute is often MISSING

sudo rmmod mydev_bad; rm -f /tmp/udev-saw-*
sudo insmod mydev_good.ko        # uses .dev_groups
cat /tmp/udev-saw-*.txt          # 'mode' is ALWAYS present
```

### Lab 26.6 — `sysfs_notify()` + `poll()` as an event channel

```c
/* kernel side: when state changes */
d->status = NEW_STATUS;
sysfs_notify(&d->dev->kobj, NULL, "status");
```
```c
/* userspace: block until it changes */
#include <poll.h>
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>
int main(int argc, char **argv) {
	int fd = open(argv[1], O_RDONLY);
	char buf[64];
	struct pollfd p = { .fd = fd, .events = POLLPRI | POLLERR };

	read(fd, buf, sizeof(buf));                 /* ★ must read once first */
	while (1) {
		poll(&p, 1, -1);
		lseek(fd, 0, SEEK_SET);                 /* ★ and rewind each time */
		int n = read(fd, buf, sizeof(buf));
		buf[n] = 0;
		printf("changed: %s", buf);
	}
}
```
```bash
gcc -O2 -o /tmp/sysfspoll /tmp/sysfspoll.c
/tmp/sysfspoll /sys/class/power_supply/BAT0/status   # unplug/replug your laptop
/tmp/sysfspoll /sys/class/net/eth0/carrier           # unplug the cable
```
This is how `libgpiod` (pre-chardev), thermal daemons, and battery monitors worked. Note the
two non-obvious requirements: an initial `read()` and an `lseek()` before each re-read.

### Lab 26.7 — kernfs active references (T.4)

```bash
# Hold an attribute open while removing the device
exec 3< /sys/bus/pci/devices/0000:00:1f.2/vendor
# In another shell:
echo 1 | sudo tee /sys/bus/pci/devices/0000:00:1f.2/remove
# The removal blocks until the fd is closed? Or does it? Investigate:
exec 3<&-

$EDITOR fs/kernfs/dir.c          # kernfs_get_active / kernfs_drain
grep -n 'sysfs_break_active_protection\|sysfs_unbreak' fs/sysfs/file.c drivers/pci/pci-sysfs.c
```
Explain in writing why `/sys/bus/pci/devices/*/remove` needs
`sysfs_break_active_protection()` — it is a store handler that removes the very file it is
running in (a self-deadlock the framework otherwise prevents).

### Lab 26.8 — Explore the whole model

```bash
# Every bus, with its device and driver counts
for b in /sys/bus/*/; do
  printf "%-16s devices=%-4s drivers=%s\n" "$(basename $b)" \
    "$(ls $b/devices 2>/dev/null | wc -l)" "$(ls $b/drivers 2>/dev/null | wc -l)"
done | sort -k2 -t= -rn | head -20

# Every class
for c in /sys/class/*/; do
  printf "%-20s %s\n" "$(basename $c)" "$(ls $c | wc -l)"
done | sort -k2 -rn | head -20

# Devices with NO driver bound (T.3: presence != binding)
for d in /sys/bus/pci/devices/*/; do
  [ -e "$d/driver" ] || echo "unbound: $(basename $d) $(cat $d/class)"
done

# The full tree, rendered
ls /sys/devices/
find /sys/devices/platform -maxdepth 2 | head -30
lsblk -O 2>/dev/null | head -5
lspci -tv | head -20
lsusb -t
```

---

## 3. Mastery drills

1. **Read `Documentation/core-api/kobject.rst`** completely (it is ~400 lines and written by
   Greg KH). Then read `lib/kobject.c`. Explain exactly what happens between
   `kobject_put()` and your `release()` being called.

2. **The PM origin story.** Find the device-model merge in the git history
   (`git log --oneline --reverse -- drivers/base/core.c | head -20`). Then read
   `drivers/base/power/main.c` and explain how the parent pointer is used to order suspend
   and resume. Why must resume be the *reverse* order?

3. **Two taxonomies.** For a USB webcam, write out its full `/sys/devices/` path and every
   `/sys/bus/` and `/sys/class/` symlink pointing at it. Explain what each answers.

4. **Release-function audit.** `git grep -l 'kobj_type' -- drivers/ | head -20`. For each,
   verify `release()` exists and actually frees the container. Find one that leaks or is
   suspicious.

5. **Attribute race.** `git grep -n 'device_create_file' -- drivers/ | wc -l`. Pick one, and
   determine whether it races with udev. Convert it to `dev_groups` and submit the patch.
   (`git log --grep="use dev_groups"` for precedents.)

6. **`is_visible()`.** Find three drivers using `attribute_group->is_visible()`. Explain
   what each is conditionalizing on and why a separate group would not work.

7. **Design question.** You need to expose 200 counters per device, updated at 1 MHz,
   readable at 1 Hz. Explain why sysfs is wrong (cite T.4's rules), and choose an alternative
   with justification (Ch. 24 T.5).

8. **kernfs.** Read `fs/kernfs/dir.c`'s active-reference logic. Explain how it implements the
   Ch. 25 P12 quiescence pattern, and what would break without it.

9. **Naming as ABI.** Read systemd's predictable-network-interface-names documentation.
   Explain how it uses the T.1 topology, and what stable property each scheme
   (`en[oPsx]`) encodes. Then explain why `/dev/sda` is unstable and what `by-path` fixes.

10. **Write the model comment** (Ch. 25 §3) for `drivers/base/core.c`'s device lifecycle:
    what refcounts exist, who holds them, and what the teardown order is.

---

## 4. Further reading

**Kernel documentation (all mandatory for this Part):**
- `Documentation/core-api/kobject.rst` ★★
- `Documentation/filesystems/sysfs.rst` ★★
- `Documentation/driver-api/driver-model/` ★★ — `overview.rst`, `device.rst`, `driver.rst`,
  `bus.rst`, `class.rst`, `binding.rst`, `porting.rst`
- `Documentation/ABI/README` and the `stable/`/`testing/` trees
- `Documentation/admin-guide/sysfs-rules.rst` ★ — **what userspace may and may not assume**;
  read this before designing any sysfs interface

**Source:**
- `lib/kobject.c`, `lib/kobject_uevent.c`, `include/linux/kobject.h`
- `drivers/base/core.c` — `device_add()`, `device_del()`
- `fs/kernfs/dir.c` — active references
- `samples/kobject/` ★ — in-tree examples of exactly Lab 26.1

**Books & articles:**
- Corbet, Rubini & Kroah-Hartman, *Linux Device Drivers* 3e, **Ch. 14 "The Linux Device
  Model"** — still the best narrative explanation; APIs have moved but the model has not
- Greg Kroah-Hartman, "The Linux Kernel Driver Model" (in *Linux Symposium* proceedings) and
  his "udev — A Userspace Implementation of devfs" paper
- Patrick Mochel, "The Linux Kernel Device Model" (OLS 2002) — **the original design paper**
- LWN: "The zen of kobjects", "Sysfs rules", "A new udev", "Device model conversion" series

→ Next: [27-bus-probe-binding.md](27-bus-probe-binding.md)

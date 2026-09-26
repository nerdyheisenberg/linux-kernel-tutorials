# Chapter 27 — Bus, Device, Driver Binding: probe, match, deferred probe, device links

> **Goal:** know exactly how a driver gets attached to a device, why `-EPROBE_DEFER` exists,
> why probe ordering is unsolvable without dependency information, and how device links fixed
> it.

---

## Theory & First Principles

### T.0 — Start here: probe order is not dependency order

Your driver needs a clock. You write the obvious probe:

```c
static int my_probe(struct platform_device *pdev)
{
	struct clk *clk = devm_clk_get(&pdev->dev, "core");
	if (IS_ERR(clk)) {
		dev_err(&pdev->dev, "no clock!\n");     /* ← and here you are stuck */
		return PTR_ERR(clk);
	}
	...
}
```

It works on your desk and fails on a colleague's board with `no clock!`, even though the
device tree clearly has the clock. **The clock controller driver had not probed yet.**

Why would it not? Because **probe order is determined by link order and module load order,
not by dependencies.** Devices are discovered by bus enumeration; drivers register when their
module loads. Neither knows anything about your clock.

**The obvious fixes are all wrong**, and the reasons are instructive:

| Fix | Why it fails |
|---|---|
| Make the clock driver link earlier (`obj-y` order) | Works until someone reorders a Makefile. Couples build files to runtime behaviour (Ch. 03 §T.6) |
| Compute a topological sort at boot | Dependencies are not all knowable statically — they come from DT phandles, ACPI, discovery, and firmware |
| Retry in a loop with a sleep | Unbounded wait, no failure detection, and you are blocking the boot |
| `late_initcall` for anything with dependencies | Only two levels of ordering exist, and this just moves the collision |

**Linux's answer is to make failure normal and retry a protocol:**

```c
static int my_probe(struct platform_device *pdev)
{
	struct clk *clk = devm_clk_get(&pdev->dev, "core");
	if (IS_ERR(clk))
		return dev_err_probe(&pdev->dev, PTR_ERR(clk), "getting core clock\n");
	/*  ↑ if the error is -EPROBE_DEFER, log NOTHING and return it;
	 *    the core puts the device on a list and retries later.
	 *    Otherwise log properly. One helper, both cases.              */
	...
}
```

> **`-EPROBE_DEFER` means "my dependency is not here *yet*."** The driver core moves the
> device to a deferred list and retries after each successful probe elsewhere, because a
> success may have been the thing you were waiting for.

**This is a genuinely interesting design choice and worth understanding as such.** The
alternative — compute the dependency graph and sort it — is what a static system would do.
Linux instead treats the system as **dynamic and partially unknown**, and converges by
retrying. It is eventual consistency applied to device initialization: no global knowledge is
required, it handles dependencies discovered at runtime, and it works identically for modules
loaded an hour after boot.

The cost is that failure is now *ambiguous*: `-EPROBE_DEFER` forever looks exactly like a
missing dependency. Hence the diagnostic surface, which is the first thing to check when a
device does not appear:

```bash
cat /sys/kernel/debug/devices_deferred      # WHAT is waiting, and on what
dmesg | grep -i 'deferred\|probe'

# Force a re-probe of one device, by hand:
echo 0000:00:1f.6 > /sys/bus/pci/drivers/e1000e/unbind
echo 0000:00:1f.6 > /sys/bus/pci/drivers/e1000e/bind
```

**And a second, subtler point** that §T.5 develops: `dev_err_probe()` exists because people
kept logging an error on the defer path, producing a log full of alarming messages about a
system that was working perfectly. **When a correctness obligation is routinely forgotten,
move it into the primitive** (Ch. 89 §T.1 #4) — that principle, which you will meet in
`devm_` next chapter and in folios and in Rust, appears here first in a two-line helper.

---

### T.1 Binding is a matchmaking problem

Ch. 26 gave us a set of devices. Now: **which code drives which device?**

The system has, at any moment, a set of **devices** (discovered by buses) and a set of
**drivers** (registered by modules). Binding is a bipartite matching:

```
   devices ──────┐              ┌────── drivers
   0000:00:1f.2  │   match()    │  ahci
   i2c-0-0x48    ├──────────────┤  lm75
   /soc/uart@... │              │  8250_of
   usb 1-2:1.0   │              │  usb-storage
                 └──────────────┘
```

The crucial property: **both sets change at runtime, independently.** A device can appear
(hotplug) before or after its driver is loaded. Therefore binding cannot be a one-time pass —
it must be an **event-driven, symmetric** process:

```c
/* drivers/base/bus.c — the two halves */
bus_probe_device(dev)   /* a DEVICE appeared: try every driver on its bus */
driver_attach(drv)      /* a DRIVER appeared: try every device on its bus */
```

Every bus implements exactly one interesting function:

```c
struct bus_type {
	const char *name;
	int  (*match)(struct device *dev, const struct device_driver *drv);  /* ★ */
	int  (*probe)(struct device *dev);
	void (*remove)(struct device *dev);
	int  (*uevent)(const struct device *dev, struct kobj_uevent_env *env);
	const struct dev_pm_ops *pm;
	const struct attribute_group **dev_groups, **drv_groups, **bus_groups;
	...
};
```

`match()` is the whole bus abstraction. Everything else — enumeration, power management,
DMA — is bus-specific detail hanging off it.

### T.2 The four matching mechanisms, and the module-autoload problem

How does `match()` decide? Four schemes, layered historically:

| Mechanism | Key | Buses |
|---|---|---|
| **Name** | a literal string | platform (legacy) |
| **ID table** | vendor/device/class numbers | PCI, USB, PCMCIA, virtio |
| **Device tree `compatible`** | `"vendor,model"` strings | all DT platforms (Ch. 32) |
| **ACPI `_HID`/`_CID`** | ACPI hardware ids | x86/ARM ACPI (Ch. 33) |

```c
/* PCI */
static const struct pci_device_id my_pci_ids[] = {
	{ PCI_DEVICE(0x1234, 0x5678) },
	{ PCI_DEVICE_CLASS(PCI_CLASS_STORAGE_EXPRESS, 0xffffff) },
	{ }
};
MODULE_DEVICE_TABLE(pci, my_pci_ids);      /* ★ */

/* Device tree */
static const struct of_device_id my_of_ids[] = {
	{ .compatible = "acme,widget-v2", .data = &v2_config },
	{ .compatible = "acme,widget",    .data = &v1_config },
	{ }
};
MODULE_DEVICE_TABLE(of, my_of_ids);

/* ACPI */
static const struct acpi_device_id my_acpi_ids[] = {
	{ "ACME0001", (kernel_ulong_t)&v1_config }, { }
};
MODULE_DEVICE_TABLE(acpi, my_acpi_ids);
```

**`MODULE_DEVICE_TABLE()` is the mechanism that makes modular drivers usable at all**, and
its design is worth understanding because it solves a genuine bootstrapping problem:

> *To know which module to load, you must read the module's device table. To read its table,
> you must load it.*

The resolution: `modpost` (Ch. 03) extracts the table at **build time** and emits it as
`MODULE_ALIAS` strings into the `.modinfo` section. `depmod` collects those into
`/lib/modules/*/modules.alias`. The kernel emits a `MODALIAS=` uevent variable describing the
device; udev/`kmod` looks it up and `modprobe`s the match. **Static extraction breaks the
cycle** — the same technique as `pkg-config` files or Java's `META-INF/services`.

```bash
modinfo -F alias e1000e | head
grep '^alias pci:v00008086' /lib/modules/$(uname -r)/modules.alias | head -3
cat /sys/bus/pci/devices/0000:00:1f.2/modalias
# and the whole chain:
udevadm info -q property /sys/bus/pci/devices/0000:00:1f.2 | grep MODALIAS
```

`match()` also carries **configuration data**: `of_device_get_match_data()` /
`device_get_match_data()` returns the `.data` from the matching entry, which is how one
driver supports twenty SoC variants without `if (of_machine_is_compatible(...))` chains.

### T.3 `probe()`: the contract

```c
static int my_probe(struct platform_device *pdev)
{
	/* Return 0        : bound successfully. remove() will be called eventually. */
	/* Return -ENODEV  : not my device after all (rare). */
	/* Return -EPROBE_DEFER : dependencies not ready; RETRY ME LATER. */
	/* Return other <0 : real failure. remove() will NOT be called —
	 *                   probe() must undo everything itself.        */
}
```

The last rule is the one that causes bugs: **on probe failure, `remove()` is not called.**
`probe()` owns its own error unwinding — which is exactly the Ch. 05 T.6 `goto` ladder, and
exactly why `devm_` (Ch. 28) exists.

**`probe()` runs in task context and may sleep.** It may not assume: that interrupts are
enabled at the device, that other devices are probed, that userspace exists, or that it will
run on any particular CPU. It *may* be running concurrently with other probes
(`CONFIG_ASYNC_PROBE` / `PROBE_PREFER_ASYNCHRONOUS`), so it must not touch global state
unlocked.

The canonical skeleton, which you should be able to write from memory:

```c
static int my_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	struct my_priv *priv;
	int irq, ret;

	priv = devm_kzalloc(dev, sizeof(*priv), GFP_KERNEL);
	if (!priv)
		return -ENOMEM;
	priv->dev = dev;
	platform_set_drvdata(pdev, priv);

	priv->cfg = device_get_match_data(dev);            /* per-variant config (T.2) */

	priv->base = devm_platform_ioremap_resource(pdev, 0);
	if (IS_ERR(priv->base))
		return PTR_ERR(priv->base);

	priv->clk = devm_clk_get_enabled(dev, NULL);       /* may return -EPROBE_DEFER */
	if (IS_ERR(priv->clk))
		return dev_err_probe(dev, PTR_ERR(priv->clk), "failed to get clock\n");

	priv->vdd = devm_regulator_get(dev, "vdd");        /* may return -EPROBE_DEFER */
	if (IS_ERR(priv->vdd))
		return dev_err_probe(dev, PTR_ERR(priv->vdd), "failed to get vdd\n");

	irq = platform_get_irq(pdev, 0);                   /* may return -EPROBE_DEFER */
	if (irq < 0)
		return irq;

	ret = my_hw_init(priv);                            /* ★ hardware ready BEFORE irq */
	if (ret)
		return ret;

	ret = devm_request_threaded_irq(dev, irq, NULL, my_thread_fn,
					IRQF_ONESHOT, dev_name(dev), priv);
	if (ret)
		return dev_err_probe(dev, ret, "failed to request irq\n");

	return devm_my_register_subsystem(dev, priv);      /* publish to userspace LAST */
}
```

**`dev_err_probe()` is not optional style.** It (a) returns the error unchanged so you can
`return` it directly, (b) prints nothing for `-EPROBE_DEFER` (which would otherwise spam the
log on every retry), and (c) records the deferral reason for
`/sys/kernel/debug/devices_deferred`. Use it in every probe error path.

### T.4 Why probe ordering is unsolvable, and `-EPROBE_DEFER`

Here is the fundamental problem, and it is genuinely hard.

An I²C touchscreen needs: a GPIO for reset, a regulator for power, an interrupt from a GPIO
controller, and a clock. Each of those is provided by *another device* with its own driver.
So probing has a **dependency graph**, and you must probe in topological order.

But the kernel **does not know the graph** at probe time:

- Drivers are in modules that load in arbitrary order (or are built in, in link order).
- Devices appear asynchronously (hotplug, deferred bus scans).
- The dependency is expressed in the *device tree*, as a phandle — but only the *driver*
  knows how to interpret `clocks = <&clk 3>` as "I need the device behind clk".
- Cycles are possible in principle (device A's regulator is on a bus clocked by device B,
  which is powered by A's regulator).

Attempts that failed:
- **Initcall levels** (Ch. 00) — too coarse; only orders subsystems, not instances.
- **Link order** — works only for built-in, breaks with modules, and is invisible/fragile.
- **A static dependency list** — nobody can maintain it; it duplicates the DT.

**The solution is dynamic and beautifully simple: let the driver discover the dependency and
say "not yet".**

```c
priv->clk = devm_clk_get(dev, NULL);
if (IS_ERR(priv->clk))
	return dev_err_probe(dev, PTR_ERR(priv->clk), "no clock\n");
	/* if the clock provider hasn't probed, clk_get returns -EPROBE_DEFER */
```

The core catches `-EPROBE_DEFER`, **undoes the partial probe** (this is why `devm_` matters —
it makes the undo automatic), puts the device on a **deferred list**, and retries it whenever
*any* driver successfully probes (on the theory that a new successful probe may have provided
the missing resource), and once more at `late_initcall` / after modules settle.

This is **lazy topological sort by trial and error**: instead of computing the order, repeatedly
attempt and retry until a fixed point. It converges because each successful probe strictly
reduces the unsatisfied-dependency count, and it requires no global knowledge. The cost is
repeated partial probes — which is exactly why probe functions must be cheap up to the point
of the first possible deferral, and why `dev_err_probe()` suppresses the log spam.

```bash
cat /sys/kernel/debug/devices_deferred          # ★ devices still waiting, and why
dmesg | grep -i 'deferred probe'
# boot with: initcall_debug  and look for repeated probe attempts
sudo bpftrace -e 'kprobe:driver_deferred_probe_add { @[str(arg0)] = count(); }'
```

**When probing never converges** you get a silently missing device — the most common
bring-up failure (Ch. 101). `/sys/kernel/debug/devices_deferred` is the first place to look,
and it tells you the *last* error string, which is why `dev_err_probe()`'s message matters.

### T.5 Device links: making the graph explicit

Deferred probe solves *ordering* but leaves three problems:

1. **Suspend/resume order** — the PM core orders by parent/child (Ch. 26 T.1), but a
   dependency on a regulator on a *different* branch of the tree is invisible. You can
   suspend the regulator before its consumer.
2. **Runtime PM** — a consumer that is active must keep its supplier powered.
3. **Unbind order** — unbinding a supplier while a consumer holds its resources is a UAF.

**Device links** (4.10+, Rafael Wysocki) add explicit consumer→supplier edges to the device
model, turning the implicit tree into a proper DAG:

```c
struct device_link *link;

link = device_link_add(consumer, supplier,
		       DL_FLAG_STATELESS |          /* no PM/probe implications */
		       DL_FLAG_PM_RUNTIME |         /* supplier stays on while consumer is */
		       DL_FLAG_RPM_ACTIVE |
		       DL_FLAG_AUTOREMOVE_CONSUMER); /* dropped when consumer unbinds */
```

Once the link exists the core guarantees:
- The supplier probes **before** the consumer (probe ordering, without deferral).
- The supplier suspends **after** the consumer, and resumes **before** it.
- Unbinding the supplier **first unbinds the consumer**.
- Runtime-PM reference counting propagates.

And critically, **`fw_devlink`** (5.12+) creates these links **automatically** by parsing the
device tree / ACPI *before* any driver probes: it walks every phandle reference
(`clocks`, `regulators`, `gpios`, `interrupts`, `dmas`, `phys`, …) and creates a link for
each. So the dependency graph that was previously unknowable is now derived from the firmware
description.

```bash
cat /sys/kernel/debug/device_component/*  2>/dev/null
ls -l /sys/devices/platform/*/consumer:*  /sys/devices/platform/*/supplier:*  2>/dev/null | head
# boot param:
#   fw_devlink=on|permissive|rpm|off
dmesg | grep -i fw_devlink
```

`fw_devlink` dramatically reduced deferred-probe churn — many devices now probe first time.
It also **surfaces broken device trees loudly** (a missing provider now produces a clear
"supplier not found" instead of an infinite deferral), which caused a wave of DT fixes when
it was enabled by default. That is the mark of a good mechanism: it converts silent
misbehaviour into a loud, specific error.

### T.6 The full bind/unbind lifecycle

```
   device_add()  or  driver_register()
        │
        ▼
   bus_for_each_drv / bus_for_each_dev
        │
        ▼
   driver_match_device()  →  bus->match(dev, drv)
        │ match
        ▼
   really_probe(dev, drv)
        ├─ dev->driver = drv
        ├─ devres_open_group()              ★ Ch. 28: start a resource group
        ├─ pinctrl default state applied
        ├─ dma_configure()
        ├─ bus->probe() or drv->probe()
        │     ├─ 0            → SUCCESS: driver_bound(), KOBJ_BIND uevent,
        │     │                  sysfs links drv<->dev created
        │     ├─ -EPROBE_DEFER→ devres_release_group(), dev->driver = NULL,
        │     │                  driver_deferred_probe_add()
        │     └─ other <0     → devres_release_group(), dev->driver = NULL, done
        ▼
   ... device is bound and operating ...
        │
   device_release_driver()  (rmmod, unbind, hotplug removal)
        ├─ KOBJ_UNBIND uevent
        ├─ device_links_unbind_consumers()  ★ unbind everyone who depends on us first
        ├─ drv->remove(dev)                 ★ must return void (6.11+); cannot fail
        ├─ devres_release_all()             ★ Ch. 28: automatic cleanup, reverse order
        ├─ dev->driver = NULL
        └─ pinctrl sleep state, dma_deconfigure
```

**`remove()` returns `void`** (converted tree-wide, completing in 6.11). The reason is
principled: there is no caller who can handle a failure. If a device is being physically
removed, refusing is not an option. A driver that "fails" to remove leaves the core in an
inconsistent state. So the API was changed to make the impossible inexpressible — the same
philosophy as Ch. 22 T.5's folio and Ch. 24 T.3's flag rejection.

**Manual bind/unbind from userspace** is a critical debugging and VFIO workflow:

```bash
# Unbind a device from its driver
echo 0000:03:00.0 | sudo tee /sys/bus/pci/drivers/nvme/unbind
# Bind it to another (e.g. for VFIO passthrough, Ch. 36)
echo 10de 1c03 | sudo tee /sys/bus/pci/drivers/vfio-pci/new_id
echo 0000:03:00.0 | sudo tee /sys/bus/pci/drivers/vfio-pci/bind
# Prevent auto-binding entirely
echo 0 | sudo tee /sys/bus/pci/drivers_autoprobe
# Override the match for one device
echo vfio-pci | sudo tee /sys/bus/pci/devices/0000:03:00.0/driver_override
```
`driver_override` is worth knowing: it bypasses `match()` entirely for one device. It is how
`vfio-pci` binding is done cleanly, and it exists because `new_id` had a race and polluted the
driver's id table globally.

### T.7 Async probe and the boot-time argument

Probing is often dominated by waiting (firmware download, PHY reset, disk spin-up), so
serializing it wastes wall-clock time. `PROBE_PREFER_ASYNCHRONOUS` lets the core probe
devices in parallel kthreads:

```c
static struct platform_driver my_driver = {
	.driver = {
		.name = "mydev",
		.probe_type = PROBE_PREFER_ASYNCHRONOUS,   /* ★ opt in */
	},
};
```
```bash
# Globally:  driver_async_probe=*   or  driver_async_probe=nvme,mmcblk
cat /sys/module/*/parameters/async_probe 2>/dev/null
systemd-analyze blame | head
dmesg | grep -E 'async|probe.*took'
```

**The obligation this creates:** your `probe()` must be safe against concurrent execution
with *other* probes, including other instances of *your own* driver. Global state in a driver
(a static array indexed by instance number, a shared regmap, a singleton) becomes a race.
This is a real source of bugs in drivers that were written when probe was serial.

---

## 1. Internals

### 1.1 Source map

```
drivers/base/dd.c        ★★ really_probe, driver_probe_device, deferred probe, async probe
drivers/base/bus.c       ★ bus_type registration, bus_probe_device, driver_attach
drivers/base/driver.c    driver_register, driver_find
drivers/base/core.c      device_add/del, device links (device_link_add)
drivers/base/platform.c  ★ the platform bus — read this first, it is the simplest
drivers/base/property.c  ★ fwnode: the DT/ACPI/swnode abstraction
drivers/base/component.c the "aggregate driver" pattern (DRM, ASoC)
drivers/base/faux.c      (6.14+) the minimal bus for pseudo-devices
include/linux/device/bus.h, driver.h, class.h
include/linux/mod_devicetable.h   ★ every *_device_id struct
scripts/mod/file2alias.c ★ MODULE_DEVICE_TABLE -> MODULE_ALIAS extraction (T.2)
Documentation/driver-api/driver-model/binding.rst ★
Documentation/driver-api/device_link.rst          ★★
```

### 1.2 `fwnode`: one API for DT, ACPI, and software nodes

A driver should not care whether its properties came from a device tree, ACPI, or a
board file. `fwnode` is that abstraction (Ch. 32/33 go deeper):

```c
/* Prefer these — they work on DT, ACPI, and swnodes alike */
device_property_read_u32(dev, "acme,max-speed", &speed);
device_property_read_string(dev, "label", &label);
device_property_present(dev, "acme,inverted");
device_property_count_u32(dev, "acme,channels");
device_property_read_u32_array(dev, "acme,channels", vals, n);

fwnode_for_each_child_node(dev_fwnode(dev), child) { ... }
fwnode_handle_put(child);

/* Avoid these unless you genuinely need DT-only behaviour: */
of_property_read_u32(dev->of_node, ...);
```

**Software nodes** (`swnode`) let you attach the same property model to devices created from
C code — which is how the same driver can serve a DT-described SoC and a
statically-instantiated PC platform device. This is the modern replacement for
`platform_data`.

---

## 2. Practice

### Lab 27.1 — A minimal bus, device, and driver from scratch

The fastest way to understand binding is to implement it.

```c
// SPDX-License-Identifier: GPL-2.0
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/device.h>
#include <linux/slab.h>

/* ---- the device type ---- */
struct acme_device {
	struct device dev;
	const char   *model;
	u32           revision;
};
#define to_acme_dev(d) container_of(d, struct acme_device, dev)

struct acme_driver {
	struct device_driver driver;
	const char *const   *models;      /* NULL-terminated list we can drive */
	int  (*probe)(struct acme_device *adev);
	void (*remove)(struct acme_device *adev);
};
#define to_acme_drv(d) container_of(d, struct acme_driver, driver)

/* ---- the bus ---- */
static int acme_match(struct device *dev, const struct device_driver *drv)
{
	struct acme_device *adev = to_acme_dev(dev);
	const struct acme_driver *adrv = to_acme_drv(drv);
	const char *const *m;

	for (m = adrv->models; *m; m++)
		if (!strcmp(*m, adev->model)) {
			pr_info("match: %s <-> %s\n", adev->model, drv->name);
			return 1;
		}
	return 0;
}

static int acme_probe(struct device *dev)
{
	struct acme_driver *adrv = to_acme_drv(dev->driver);

	return adrv->probe ? adrv->probe(to_acme_dev(dev)) : 0;
}

static void acme_remove(struct device *dev)
{
	struct acme_driver *adrv = to_acme_drv(dev->driver);

	if (adrv->remove)
		adrv->remove(to_acme_dev(dev));
}

static int acme_uevent(const struct device *dev, struct kobj_uevent_env *env)
{
	const struct acme_device *adev = to_acme_dev((struct device *)dev);

	/* ★ this is what makes module autoloading work (T.2) */
	return add_uevent_var(env, "MODALIAS=acme:%s", adev->model);
}

static ssize_t model_show(struct device *dev, struct device_attribute *a, char *buf)
{ return sysfs_emit(buf, "%s\n", to_acme_dev(dev)->model); }
static DEVICE_ATTR_RO(model);

static ssize_t modalias_show(struct device *dev, struct device_attribute *a, char *buf)
{ return sysfs_emit(buf, "acme:%s\n", to_acme_dev(dev)->model); }
static DEVICE_ATTR_RO(modalias);

static struct attribute *acme_dev_attrs[] = {
	&dev_attr_model.attr, &dev_attr_modalias.attr, NULL,
};
ATTRIBUTE_GROUPS(acme_dev);

static const struct bus_type acme_bus_type = {
	.name       = "acme",
	.match      = acme_match,
	.probe      = acme_probe,
	.remove     = acme_remove,
	.uevent     = acme_uevent,
	.dev_groups = acme_dev_groups,        /* ★ Ch. 26 T.5 */
};

/* ---- device creation ---- */
static void acme_dev_release(struct device *dev)
{
	pr_info("releasing %s\n", dev_name(dev));
	kfree(to_acme_dev(dev));              /* ★ Ch. 26 T.2 */
}

static struct acme_device *acme_device_create(const char *model, u32 rev, int id)
{
	struct acme_device *adev = kzalloc(sizeof(*adev), GFP_KERNEL);
	int ret;

	if (!adev)
		return ERR_PTR(-ENOMEM);
	adev->model    = model;
	adev->revision = rev;
	adev->dev.bus     = &acme_bus_type;
	adev->dev.release = acme_dev_release;
	dev_set_name(&adev->dev, "acme%d", id);

	ret = device_register(&adev->dev);     /* ★ triggers matching immediately */
	if (ret) {
		put_device(&adev->dev);        /* release() frees it */
		return ERR_PTR(ret);
	}
	return adev;
}

/* ---- a driver for it ---- */
static const char *const widget_models[] = { "widget-v1", "widget-v2", NULL };

static int widget_probe(struct acme_device *adev)
{
	dev_info(&adev->dev, "probed: model=%s rev=%u\n", adev->model, adev->revision);
	return 0;
}
static void widget_remove(struct acme_device *adev)
{
	dev_info(&adev->dev, "removed\n");
}

static struct acme_driver widget_driver = {
	.driver = { .name = "acme-widget", .owner = THIS_MODULE, .bus = &acme_bus_type },
	.models = widget_models,
	.probe  = widget_probe,
	.remove = widget_remove,
};

static struct acme_device *d0, *d1, *d2;

static int __init acme_init(void)
{
	int ret = bus_register(&acme_bus_type);

	if (ret)
		return ret;

	/* Create a device BEFORE the driver exists... */
	d0 = acme_device_create("widget-v1", 1, 0);
	d2 = acme_device_create("gizmo-v9", 9, 2);      /* nothing will drive this */

	ret = driver_register(&widget_driver.driver);   /* ...now d0 binds */
	if (ret) {
		bus_unregister(&acme_bus_type);
		return ret;
	}

	/* ...and one AFTER: it binds immediately. Both directions work. */
	d1 = acme_device_create("widget-v2", 2, 1);
	return 0;
}

static void __exit acme_exit(void)
{
	device_unregister(&d0->dev);
	device_unregister(&d1->dev);
	device_unregister(&d2->dev);
	driver_unregister(&widget_driver.driver);
	bus_unregister(&acme_bus_type);
}
module_init(acme_init); module_exit(acme_exit);
MODULE_LICENSE("GPL");
```
```bash
sudo insmod acmebus.ko
dmesg | tail -20
tree /sys/bus/acme/
cat /sys/bus/acme/devices/acme0/model
ls -l /sys/bus/acme/drivers/acme-widget/     # bound devices as symlinks
ls /sys/bus/acme/devices/acme2/driver 2>&1   # gizmo: no driver — presence != binding

# Manual unbind/bind:
echo acme0 | sudo tee /sys/bus/acme/drivers/acme-widget/unbind
echo acme0 | sudo tee /sys/bus/acme/drivers/acme-widget/bind
```
**This ~180-line module contains the entire binding model.** Study it until every line is
obvious.

### Lab 27.2 — Force and observe deferred probe (T.4)

```c
/* Deliberately defer a fixed number of times */
static int defer_count = 3;
module_param(defer_count, int, 0644);

static int my_probe(struct platform_device *pdev)
{
	static int attempts;

	if (attempts++ < defer_count)
		return dev_err_probe(&pdev->dev, -EPROBE_DEFER,
				     "waiting for imaginary resource (attempt %d)\n",
				     attempts);
	dev_info(&pdev->dev, "finally probed after %d attempts\n", attempts);
	return 0;
}
```
```bash
sudo insmod deferdemo.ko defer_count=3
cat /sys/kernel/debug/devices_deferred        # ★ shows the device and the reason
dmesg | grep -i defer

# Watch the retry trigger: probing ANOTHER driver kicks the list
sudo modprobe some_other_driver
cat /sys/kernel/debug/devices_deferred

# Trace it
sudo bpftrace -e '
kprobe:driver_deferred_probe_add    { @added  = count(); }
kprobe:driver_deferred_probe_trigger{ @trig   = count(); }
kprobe:really_probe                 { @probes = count(); }'
```
Now the important experiment: **make it never converge** (`defer_count=999999`) and see how
the failure presents. This is what a bring-up failure looks like, and
`/sys/kernel/debug/devices_deferred` is the only place it is visible.

### Lab 27.3 — A real dependency chain in QEMU/DT

On an arm64 QEMU `virt` machine or a Raspberry Pi, find a genuine chain:

```bash
# Who is deferred, and why?
cat /sys/kernel/debug/devices_deferred

# The device tree dependency edges fw_devlink parsed:
ls -l /sys/devices/platform/*/ | grep -E 'consumer:|supplier:' | head -20
for d in /sys/devices/platform/*/; do
  c=$(ls -d $d/consumer:* 2>/dev/null | wc -l)
  s=$(ls -d $d/supplier:* 2>/dev/null | wc -l)
  [ "$c$s" != "00" ] && echo "$(basename $d): consumers=$c suppliers=$s"
done

# Turn fw_devlink off and compare deferred-probe counts
# boot with: fw_devlink=off   vs   fw_devlink=on
dmesg | grep -ci 'deferred probe'

# Which DT properties create links?
grep -n 'fwnode_link\|parse_prop\|supplier' drivers/base/property.c drivers/of/property.c | head -30
$EDITOR drivers/of/property.c       # look at the `of_supplier_bindings[]` table
```

### Lab 27.4 — Module autoloading end to end (T.2)

```bash
# 1. What alias does a driver advertise?
modinfo -F alias nvme | head
modinfo -F alias i2c-designware-platform

# 2. Where does it come from?
git grep -n 'MODULE_DEVICE_TABLE' drivers/nvme/host/pci.c
$EDITOR scripts/mod/file2alias.c       # find do_pci_entry() / do_of_entry()

# 3. The generated table
grep -c . /lib/modules/$(uname -r)/modules.alias
grep '^alias of:N\*T\*Csnps,dw-apb-uart' /lib/modules/$(uname -r)/modules.alias

# 4. The device's advertised modalias
cat /sys/bus/pci/devices/*/modalias | head -3
cat /sys/bus/platform/devices/*/modalias 2>/dev/null | head -3

# 5. Resolve it manually, as udev would:
MA=$(cat /sys/bus/pci/devices/0000:00:1f.2/modalias)
/sbin/modprobe -R "$MA"

# 6. Break it: build a module WITHOUT MODULE_DEVICE_TABLE and show it never autoloads
```

### Lab 27.5 — Device links, created by hand

```c
static int consumer_probe(struct platform_device *pdev)
{
	struct device *supplier = bus_find_device_by_name(&platform_bus_type, NULL, "supplier.0");
	struct device_link *link;

	if (!supplier)
		return -EPROBE_DEFER;

	link = device_link_add(&pdev->dev, supplier,
			       DL_FLAG_AUTOREMOVE_CONSUMER | DL_FLAG_PM_RUNTIME);
	put_device(supplier);
	if (!link)
		return -EINVAL;

	dev_info(&pdev->dev, "linked to supplier\n");
	return 0;
}
```
```bash
sudo insmod supplier.ko && sudo insmod consumer.ko
ls -l /sys/devices/platform/supplier.0/consumer:*
ls -l /sys/devices/platform/consumer.0/supplier:*
cat /sys/devices/platform/consumer.0/supplier:*/status

# ★ Now prove the ordering guarantee: unbind the SUPPLIER
echo supplier.0 | sudo tee /sys/bus/platform/drivers/supplier/unbind
dmesg | tail -5      # the CONSUMER is unbound first, automatically

# And the suspend ordering:
echo 1 | sudo tee /sys/power/pm_print_times
systemctl suspend      # (in a VM: echo mem > /sys/power/state)
dmesg | grep -E 'PM: .*(supplier|consumer)'
```

### Lab 27.6 — Async probe and the concurrency it demands (T.7)

```c
static DEFINE_MUTEX(global_lock);
static int instance_count;              /* ← shared state: a race under async probe */

static int my_probe(struct platform_device *pdev)
{
	int id;

	if (async_unsafe) {
		id = instance_count++;          /* RACY */
	} else {
		guard(mutex)(&global_lock);
		id = instance_count++;          /* correct */
	}
	msleep(200);                            /* simulate slow hardware init */
	dev_info(&pdev->dev, "instance %d on cpu%d\n", id, smp_processor_id());
	return 0;
}

static struct platform_driver my_driver = {
	.driver = { .name = "asyncdemo", .probe_type = PROBE_PREFER_ASYNCHRONOUS },
	.probe  = my_probe,
};
```
```bash
# Register 8 devices; time the total probe
time sudo insmod asyncdemo.ko n=8 async=0    # ~1.6 s serial
time sudo insmod asyncdemo.ko n=8 async=1    # ~0.2 s parallel
dmesg | grep instance                        # note the CPU numbers and ordering

# Catch the race:
./scripts/config -e KCSAN
sudo insmod asyncdemo.ko n=32 async=1 async_unsafe=1
dmesg | grep -A20 KCSAN

# System-wide:
cat /sys/module/*/parameters/async_probe 2>/dev/null
systemd-analyze blame | head -10
```

### Lab 27.7 — `driver_override` and VFIO-style rebinding

```bash
# Pick a device you can safely detach (a spare NIC, a USB controller in a VM)
D=0000:00:03.0
cat /sys/bus/pci/devices/$D/driver/../../drivers/*/module 2>/dev/null
readlink /sys/bus/pci/devices/$D/driver

# Unbind
echo $D | sudo tee /sys/bus/pci/devices/$D/driver/unbind

# Override the match and rebind to vfio-pci
sudo modprobe vfio-pci
echo vfio-pci | sudo tee /sys/bus/pci/devices/$D/driver_override
echo $D | sudo tee /sys/bus/pci/drivers_probe
readlink /sys/bus/pci/devices/$D/driver     # now vfio-pci

# Restore
echo "" | sudo tee /sys/bus/pci/devices/$D/driver_override
echo $D | sudo tee /sys/bus/pci/drivers/vfio-pci/unbind
echo $D | sudo tee /sys/bus/pci/drivers_probe

# Disable autoprobe entirely (useful during bring-up)
echo 0 | sudo tee /sys/bus/pci/drivers_autoprobe
```

### Lab 27.8 — Trace the whole bind path

```bash
sudo trace-cmd record -p function_graph -g really_probe -- \
     sh -c 'sudo modprobe -r mydrv; sudo modprobe mydrv'
trace-cmd report | head -100

sudo bpftrace -e '
kprobe:really_probe   { @s[tid] = nsecs; printf("probe start: %s\n", str(((struct device *)arg0)->kobj.name)); }
kretprobe:really_probe /@s[tid]/ { @us = hist((nsecs-@s[tid])/1000); delete(@s[tid]); }
kprobe:driver_deferred_probe_add { printf("  DEFERRED\n"); }'

# Per-driver probe time (boot optimization)
dmesg | grep -E 'probe of .* took' | sort -t' ' -k5 -rn | head -20
# (needs initcall_debug, or CONFIG_DEBUG_DRIVER)
```

---

## 3. Mastery drills

1. **Read `drivers/base/dd.c`** end to end (~1200 lines). Write out the complete state
   machine for `really_probe()` including every error path. Cross-check with T.6.

2. **Implement a bus.** Extend Lab 27.1: add `-EPROBE_DEFER` support, an id table with
   `.data` per entry, and `MODULE_DEVICE_TABLE`-style autoloading via a custom
   `file2alias` entry. (The last part requires patching `scripts/mod/file2alias.c` — do it.)

3. **The convergence argument.** Prove informally that deferred probe terminates. What
   assumption does the proof require? Construct a device tree where it does *not* converge
   and explain what the user sees.

4. **`fw_devlink` archaeology.** `git log --oneline --grep='fw_devlink' | head -40`. Read the
   series that enabled it by default. What broke? How were the breakages classified into
   "kernel bug" vs "DT bug"? Why is `fw_devlink=permissive` needed?

5. **Device link flags.** Read `Documentation/driver-api/device_link.rst`. Explain each
   `DL_FLAG_*` and construct a scenario requiring each. When is `DL_FLAG_STATELESS` correct?

6. **`remove()` returns void.** Find the conversion series
   (`git log --oneline --grep='make .*remove.*void'`). Read the cover letter. Summarize the
   argument in three sentences, and relate it to Ch. 24 T.3's "make the impossible
   inexpressible".

7. **Probe ordering without DT.** On x86 with ACPI, how is the dependency problem solved?
   Read `drivers/acpi/scan.c` and compare with the DT path. What does `_DEP` do?

8. **Async-probe safety audit.** Pick three drivers with `PROBE_PREFER_ASYNCHRONOUS`. Verify
   each is actually safe against concurrent probes of its own instances. Look for static
   variables, global lists without locks, and singleton initialization.

9. **The `component` framework.** Read `drivers/base/component.c` and a DRM driver that uses
   it. Explain the problem it solves (an "aggregate" device assembled from N independent
   devices that must all be present) and why deferred probe alone is insufficient.

10. **Design question.** You are bringing up an SoC with: a PMIC on I²C providing 6
    regulators, a clock controller, a pin controller, an ethernet MAC needing a clock +
    regulator + PHY on MDIO, and the MDIO bus needing a GPIO reset from the pin controller.
    Draw the dependency DAG. Explain, for each edge, how `fw_devlink` discovers it, and what
    happens at boot if the PMIC driver is a module on a root filesystem that needs the
    ethernet.

---

## 4. Further reading

**Kernel documentation:**
- `Documentation/driver-api/driver-model/binding.rst` ★
- `Documentation/driver-api/device_link.rst` ★★
- `Documentation/driver-api/driver-model/platform.rst`
- `Documentation/driver-api/infrastructure.rst`
- `Documentation/admin-guide/kernel-parameters.txt` — `fw_devlink=`, `driver_async_probe=`,
  `deferred_probe_timeout=`
- `Documentation/ABI/testing/sysfs-bus-pci` — `driver_override`, `bind`, `unbind`, `new_id`

**Source, in reading order:**
1. `drivers/base/platform.c` — the simplest complete bus
2. `drivers/base/dd.c` — binding and deferred probe
3. `drivers/base/bus.c` — registration and iteration
4. `drivers/base/core.c` — `device_link_add()` and friends
5. `drivers/of/property.c` — `fw_devlink`'s DT parsing
6. `scripts/mod/file2alias.c` — the autoload bootstrap

**LWN:**
- "Deferred probing" / "The deferred probe problem"
- "Device links" (Corbet) and "fw_devlink: solving the probe-ordering problem"
- "Asynchronous probing" / "Speeding up boot with async probe"
- "The platform device problem" and the `faux` bus introduction (6.14)
- "Driver core: the road to removing `remove()`'s return value"

→ Next: [28-devres.md](28-devres.md)

# Chapter 31 — Platform Devices and the Platform Bus

> **Goal:** understand the deepest division in device drivers — **discoverable vs
> non-discoverable hardware** — and write platform drivers that work identically on a device
> tree, on ACPI, and on a board file.

---

## Theory & First Principles

### T.0 — Start here: a device that cannot be found

Plug a PCI card in and the kernel finds it without being told anything:

```bash
lspci -nn | head -3
#  00:1f.6 Ethernet controller [0200]: Intel I219-LM [8086:15bb]
#                                                     ^^^^^^^^^^
#  The card ANSWERS when asked. Vendor 0x8086, device 0x15bb.
```

The kernel walks the bus, reads a configuration register from each slot, gets an ID, and
looks it up. **The hardware describes itself.** Ch. 37.

Now look at a typical ARM SoC. There is a UART at physical address `0x02020000`. There is no
bus to walk, no configuration space, no ID register. If you read `0x02020000` and there is
nothing there, you do not get a "not present" answer — **you get a bus fault, or a hang.**

```
 PCI / USB / PCIe                    SoC internal peripherals
 ┌───────────────────┐              ┌─────────────────────────┐
 │ "Who is there?"    │              │ There is nobody to ask.  │
 │  -> 8086:15bb      │              │ Probing is DANGEROUS.    │
 │  -> 1af4:1000      │              │ You must be TOLD.        │
 └───────────────────┘              └─────────────────────────┘
   DISCOVERABLE                          NON-DISCOVERABLE
```

**So where does the information come from?** Historically, from C code:

```c
/* arch/arm/mach-omap2/board-n8x0.c -- the old way, and it was a disaster */
static struct resource uart_resources[] = {
	{ .start = 0x02020000, .end = 0x02020fff, .flags = IORESOURCE_MEM },
	{ .start = 72,          .end = 72,        .flags = IORESOURCE_IRQ },
};
static struct platform_device uart_device = {
	.name = "omap-uart", .id = 0,
	.num_resources = ARRAY_SIZE(uart_resources),
	.resource = uart_resources,
};
```

One of these files **per board**. By 2011 `arch/arm/` was over a million lines of such board
files, most of it near-identical, and a new board revision meant a new kernel. Torvalds
objected publicly and memorably, and the result was the move to device tree (Ch. 32).

**But notice what did *not* change.** Whether the description comes from a board file, a
device tree, or ACPI, the kernel still needs an *object* to hang the driver on — something
with a name, resources, a place in the `/sys` tree, power-management callbacks, and a
`probe()` to call. That object is `struct platform_device`, and this is the distinction worth
getting straight because people conflate them constantly:

> **`platform_device` is the *software model* for a non-discoverable device. Device tree and
> ACPI are *data sources* that populate it.** They answer different questions: "what kind of
> kernel object is this?" versus "how did we find out it exists?"

The platform bus is therefore a slightly odd thing: a **pseudo-bus** with no hardware behind
it, whose `match()` compares strings rather than probing anything (§T.4). It exists so that
non-discoverable devices get the same driver-model machinery — probe, remove, suspend,
resume, sysfs, deferred probe — as real buses, written once instead of once per SoC.

```bash
ls /sys/bus/platform/devices/ | head -20       # everything non-discoverable
ls /sys/bus/platform/drivers/ | head -10
cat /sys/bus/platform/devices/*/modalias | head -5   # how autoloading matches

# The resources the kernel was TOLD about, for one device:
cat /proc/iomem | head -20
cat /proc/interrupts | head
```

---

### T.1 The fundamental division: can the hardware describe itself?

Every bus in Linux falls on one side of a single line:

| | **Discoverable** | **Non-discoverable** |
|---|---|---|
| Examples | PCI/PCIe, USB, Thunderbolt, virtio, MMC/SD, SCSI | memory-mapped SoC blocks, I²C, SPI, most GPIO/clock/regulator |
| How the kernel finds a device | **asks the bus**: scan config space, request descriptors | **is told** by firmware or board code |
| Identity | vendor/device ID read from the device | a string in a firmware description |
| Resources (address, IRQ) | reported by the device / assigned by the bus | stated in the description |
| Hotplug | native | rare/none |

**This distinction is architectural, not incidental.** PCI has a configuration space at a
known location with a defined enumeration protocol, so the kernel can walk the bus and learn
everything. An I²C EEPROM at address 0x50 on bus 2 has **no such protocol** — reading from
address 0x50 might return data, might hang the bus, might reprogram a PMIC and switch the
board off. You cannot probe blindly; you must be told.

And a memory-mapped UART at physical address `0x1c020000` is worse: there is no bus to scan at
all. Nothing distinguishes that address from unmapped space except a human's knowledge of the
SoC.

So non-discoverable hardware requires an **external description**, and the platform bus is
the kernel's representation of "devices somebody told us about."

> **The platform bus is not a bus.** There is no wire, no protocol, no enumeration. It is a
> *registry* of devices whose existence was asserted by firmware or board code. Understanding
> this prevents a long list of category errors — including the pseudo-device abuse of
> Ch. 30 T.2.

### T.2 Three generations of "somebody told us"

The history is worth knowing because you will read code from all three eras.

**Generation 1 — board files (pre-2011).** A C file per board, in `arch/arm/mach-*/`:

```c
static struct resource smc91x_resources[] = {
	[0] = { .start = 0x08000300, .end = 0x080003ff, .flags = IORESOURCE_MEM },
	[1] = { .start = IRQ_EINT9,  .end = IRQ_EINT9,  .flags = IORESOURCE_IRQ },
};
static struct platform_device smc91x_device = {
	.name = "smc91x", .id = -1,
	.num_resources = ARRAY_SIZE(smc91x_resources),
	.resource = smc91x_resources,
};
static struct platform_device *devices[] __initdata = { &smc91x_device, ... };
platform_add_devices(devices, ARRAY_SIZE(devices));
```

This worked, but it meant **one kernel binary per board**. `arch/arm/` grew to ~1500 board
files and several million lines of nearly-identical data. Linus famously objected to the
churn in 2011 ("this whole ARM thing is a f*cking pain in the ass"), and that objection is
what forced the move to device tree.

**Generation 2 — device tree (2011→).** The hardware description moves *out* of the kernel
into a separate binary (`.dtb`) passed by the bootloader:

```dts
uart0: serial@1c020000 {
	compatible = "snps,dw-apb-uart";
	reg = <0x1c020000 0x1000>;
	interrupts = <GIC_SPI 32 IRQ_TYPE_LEVEL_HIGH>;
	clocks = <&ccu CLK_BUS_UART0>;
	clock-names = "apb";
	resets = <&ccu RST_BUS_UART0>;
	reg-shift = <2>;
	reg-io-width = <4>;
	status = "okay";
};
```

Now **one kernel image boots a thousand boards**, and adding a board is a data change, not a
code change. `of_platform_populate()` walks the tree and creates a `platform_device` per node
(Ch. 32).

**Generation 3 — ACPI on ARM, and software nodes.** Servers use ACPI; some x86 platform
devices come from ACPI `_HID`s; and pure-software instantiation uses `swnode`. The unifying
abstraction is **`fwnode`** (Ch. 27 §1.2), so a well-written driver never asks *which*
description it came from.

```bash
# Generation 1 remnants:
ls arch/arm/mach-*/board-*.c 2>/dev/null | wc -l
# Generation 2:
ls arch/arm64/boot/dts/*/ | head
ls /proc/device-tree/ 2>/dev/null | head
# Generation 3:
ls /sys/firmware/acpi/tables/ 2>/dev/null
git grep -l 'software_node' -- drivers/ | head
```

### T.3 `struct resource`: describing "where the hardware is"

```c
struct resource {
	resource_size_t start, end;
	const char     *name;
	unsigned long   flags;      /* IORESOURCE_MEM | _IO | _IRQ | _DMA | _REG | _BUS */
	unsigned long   desc;
	struct resource *parent, *sibling, *child;   /* ★ a TREE, not a list */
};
```

Two things are non-obvious and both matter:

**(a) Resources form a tree, and that tree is the system's address-space allocator.**
`/proc/iomem` and `/proc/ioports` are that tree, printed. `request_mem_region()` inserts a
node and **fails if it overlaps an existing claim** — which is how the kernel detects two
drivers claiming the same registers.

```bash
cat /proc/iomem | head -30
cat /proc/ioports | head -20
# Note the nesting — that IS the resource tree
```

**(b) Requesting is separate from mapping.** `request_mem_region()` claims the range
(conflict detection); `ioremap()` creates the virtual mapping. The modern managed call does
both:

```c
/* The one-liner you will use 95% of the time */
base = devm_platform_ioremap_resource(pdev, 0);
if (IS_ERR(base))
	return PTR_ERR(base);

/* By name — ★ prefer this; index ordering is fragile */
base = devm_platform_ioremap_resource_byname(pdev, "regs");

/* The long form, when you need the resource itself */
struct resource *res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
if (!res)
	return -ENODEV;
priv->phys_base = res->start;
priv->size = resource_size(res);
base = devm_ioremap_resource(dev, res);
```

IRQs have their own accessors because they involve `irq_domain` translation (Ch. 17 T.6):

```c
irq = platform_get_irq(pdev, 0);               /* logs its own error; may return -EPROBE_DEFER */
if (irq < 0)
	return irq;                            /* ★ propagate, don't mangle */

irq = platform_get_irq_byname(pdev, "rx");     /* ★ prefer by name */
irq = platform_get_irq_optional(pdev, 1);      /* -ENXIO if absent, no error log */
```

**Never treat 0 as a valid IRQ.** virq 0 is reserved. And `platform_get_irq()` can return
`-EPROBE_DEFER` when the interrupt controller has not probed yet — returning it unchanged is
what makes Ch. 27 T.4 work.

### T.4 The driver must not know where its description came from

This is the central discipline of modern platform-driver writing, and it is what makes one
driver serve a DT SoC, an ACPI server, and a PC platform device.

```c
/* ★ Firmware-agnostic — works on DT, ACPI, and swnode */
device_property_read_u32(dev, "acme,max-speed", &speed);
device_property_read_string(dev, "label", &label);
device_property_present(dev, "acme,inverted");
device_property_count_u32(dev, "acme,channels");
device_property_read_u32_array(dev, "acme,channels", vals, n);
device_property_match_string(dev, "mode-names", "fast");

fwnode_for_each_available_child_node(dev_fwnode(dev), child) {
	u32 reg;
	fwnode_property_read_u32(child, "reg", &reg);
	...
}

/* ★ Per-variant configuration, from the match entry (Ch. 27 T.2) */
const struct my_soc_data *cfg = device_get_match_data(dev);

/* ✗ DT-only — use only when the property is genuinely DT-specific */
of_property_read_u32(dev->of_node, "acme,max-speed", &speed);
```

The same applies to every resource type — the subsystem accessors are already
firmware-agnostic:

```c
clk    = devm_clk_get_enabled(dev, "apb");
reg    = devm_regulator_get(dev, "vdd");
gpiod  = devm_gpiod_get(dev, "reset", GPIOD_OUT_HIGH);
reset  = devm_reset_control_get_exclusive(dev, NULL);
phy    = devm_phy_get(dev, "usb2");
dma    = dma_request_chan(dev, "tx");
```

Each of these resolves through `fwnode`, works on DT and ACPI, and returns `-EPROBE_DEFER`
when the provider is not ready.

**`platform_data` is legacy.** It was a `void *` pointing at a board-file struct — untyped,
board-specific, and impossible to validate. New code uses properties (from firmware) or
software nodes (from code). Seeing `pdev->dev.platform_data` in new code is a review flag.

### T.5 `of_platform_populate()`: how DT nodes become devices

```
Device Tree                          Linux device model
────────────────────────────────────────────────────────────
/ {
    soc {                            (a "simple-bus" — populated recursively)
        uart@1c020000 { ... }    →   platform_device "1c020000.serial"
        i2c@1c2ac00  { ... }     →   platform_device "1c2ac00.i2c"
            └ rtc@68 { ... }     →   i2c_client (created by the I2C CORE, not here)
        spi@1c68000  { ... }     →   platform_device "1c68000.spi"
            └ flash@0 { ... }    →   spi_device (created by the SPI core)
    };
};
```

The crucial rule: **only nodes whose parent is a "simple bus" become platform devices.**
A child of an I²C controller node is *not* a platform device — the I²C core creates an
`i2c_client` for it when the controller driver registers its adapter. The same for SPI, MDIO,
and USB.

`of_platform_populate()` recurses into nodes marked `compatible = "simple-bus"`,
`"simple-mfd"`, `"isa"`, or `"arm,amba-bus"`. A node under a real bus controller stops the
recursion, because that controller's driver owns the enumeration of its children.

This is why a DT node sometimes "doesn't become a device": its parent is not a simple bus, or
it has `status = "disabled"`, or nothing claimed it.

```bash
ls /sys/bus/platform/devices/ | head -30
# Every platform device from DT is named "<address>.<node-name>"
cat /sys/bus/platform/devices/*/of_node/compatible 2>/dev/null | tr '\0' '\n' | sort -u | head
# The DT itself:
ls /proc/device-tree/soc/ 2>/dev/null | head -20
dtc -I fs -O dts /proc/device-tree 2>/dev/null | head -60
```

### T.6 The `platform_driver` shape

```c
static const struct of_device_id my_of_match[] = {
	{ .compatible = "acme,widget-v2", .data = &widget_v2_data },
	{ .compatible = "acme,widget",    .data = &widget_v1_data },
	{ }
};
MODULE_DEVICE_TABLE(of, my_of_match);        /* ★ autoloading (Ch. 27 T.2) */

static const struct acpi_device_id my_acpi_match[] = {
	{ "ACME0002", (kernel_ulong_t)&widget_v2_data },
	{ }
};
MODULE_DEVICE_TABLE(acpi, my_acpi_match);

static const struct platform_device_id my_ids[] = {      /* name-based, legacy/swnode */
	{ "acme-widget", (kernel_ulong_t)&widget_v1_data },
	{ }
};
MODULE_DEVICE_TABLE(platform, my_ids);

static struct platform_driver my_driver = {
	.driver = {
		.name            = "acme-widget",
		.of_match_table  = my_of_match,
		.acpi_match_table = my_acpi_match,
		.pm              = pm_ptr(&my_pm_ops),
		.dev_groups      = my_groups,               /* ★ Ch. 26 T.5 */
		.probe_type      = PROBE_PREFER_ASYNCHRONOUS,
	},
	.id_table = my_ids,
	.probe    = my_probe,
	.remove   = my_remove,        /* returns void (Ch. 27 T.6) */
	.shutdown = my_shutdown,      /* ★ called on reboot/poweroff — quiesce DMA! */
};
module_platform_driver(my_driver);    /* generates module_init/exit */
```

`platform_match()` tries, in order: OF `compatible`, ACPI `_HID`, `id_table` name, then
`driver->name`. Knowing the order explains otherwise-baffling binding behaviour.

**`.shutdown` is frequently omitted and frequently should not be.** On reboot, a device still
performing DMA will corrupt memory in the *next* kernel (or in the bootloader). Anything with
active DMA or a hardware watchdog needs a `shutdown` that quiesces it. This is a classic
kexec/kdump failure mode.

### T.7 Complete anatomy of a modern platform probe

Every line here exists for a reason established in an earlier chapter:

```c
static int my_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	struct my_priv *priv;
	int irq, ret;

	priv = devm_kzalloc(dev, sizeof(*priv), GFP_KERNEL);   /* Ch. 28 */
	if (!priv)
		return -ENOMEM;
	priv->dev = dev;
	platform_set_drvdata(pdev, priv);
	mutex_init(&priv->lock);

	priv->cfg = device_get_match_data(dev);                /* Ch. 27 T.2 */
	if (!priv->cfg)
		return -ENODEV;

	/* --- resources, in dependency order --- */
	priv->base = devm_platform_ioremap_resource_byname(pdev, "regs");   /* T.3 */
	if (IS_ERR(priv->base))
		return PTR_ERR(priv->base);

	priv->clk = devm_clk_get_enabled(dev, "apb");          /* may -EPROBE_DEFER */
	if (IS_ERR(priv->clk))
		return dev_err_probe(dev, PTR_ERR(priv->clk), "no apb clock\n");

	priv->vdd = devm_regulator_get_enable(dev, "vdd");
	if (IS_ERR(priv->vdd))
		return dev_err_probe(dev, PTR_ERR(priv->vdd), "no vdd supply\n");

	priv->rst = devm_reset_control_get_optional_exclusive(dev, NULL);
	if (IS_ERR(priv->rst))
		return dev_err_probe(dev, PTR_ERR(priv->rst), "no reset\n");

	priv->reset_gpio = devm_gpiod_get_optional(dev, "reset", GPIOD_OUT_HIGH);
	if (IS_ERR(priv->reset_gpio))
		return dev_err_probe(dev, PTR_ERR(priv->reset_gpio), "no reset gpio\n");

	/* --- optional properties, with sane defaults --- */
	if (device_property_read_u32(dev, "acme,max-speed", &priv->max_speed))
		priv->max_speed = priv->cfg->default_speed;
	priv->inverted = device_property_read_bool(dev, "acme,inverted");

	/* --- DMA capability, before any DMA allocation --- */
	ret = dma_set_mask_and_coherent(dev, DMA_BIT_MASK(priv->cfg->dma_bits));
	if (ret)
		return dev_err_probe(dev, ret, "no suitable DMA mask\n");

	/* --- bring the hardware up --- */
	reset_control_deassert(priv->rst);
	gpiod_set_value_cansleep(priv->reset_gpio, 0);
	ret = my_hw_init(priv);                                 /* ★ BEFORE request_irq */
	if (ret)
		return dev_err_probe(dev, ret, "hw init failed\n");

	/* --- interrupts, only once the hardware is quiesced (Ch. 17 §3.1) --- */
	irq = platform_get_irq(pdev, 0);
	if (irq < 0)
		return irq;
	ret = devm_request_threaded_irq(dev, irq, NULL, my_thread_fn,
					IRQF_ONESHOT, dev_name(dev), priv);
	if (ret)
		return dev_err_probe(dev, ret, "irq request failed\n");

	/* --- runtime PM (Ch. 48) --- */
	pm_runtime_set_active(dev);
	devm_pm_runtime_enable(dev);

	/* --- publish to userspace LAST --- */
	return devm_my_register(dev, priv);
}
```

Read the ordering constraints in that function; there are five, and each is a bug if
violated:
1. `platform_set_drvdata()` before anything that could call back into you.
2. Resource acquisition before hardware access.
3. `dma_set_mask()` before any DMA allocation.
4. Hardware init **before** `request_irq` (the handler can fire immediately).
5. Userspace registration **last** (a `/dev` node implies the device works).

### T.8 AMBA and other near-relatives

ARM's AMBA/PrimeCell peripherals have a wrinkle worth knowing: they carry **ID registers** at
a fixed offset (`0xfe0`–`0xffc`) — so they are *partially* discoverable. `amba_device` is a
separate bus that reads those IDs and matches on them:

```c
static const struct amba_id pl011_ids[] = {
	{ .id = 0x00041011, .mask = 0x000fffff, .data = &vendor_arm },
	{ 0, 0 },
};
```
You still need the DT to tell you the *address*, but once mapped, the device identifies
itself. It is the one clean counterexample to T.1's binary division, and it shows the
distinction is really about *how much* self-description exists.

Similar hybrid buses: `mdio` (PHYs have ID registers), `sdio`, `hid`.

---

## 1. Internals

### 1.1 Source map

```
drivers/base/platform.c        ★★ the whole platform bus; ~1600 lines, very readable
include/linux/platform_device.h
drivers/of/platform.c          ★ of_platform_populate, of_device_alloc
drivers/of/address.c           of_address_to_resource, ranges translation
drivers/of/irq.c               of_irq_get, irq_of_parse_and_map
drivers/acpi/scan.c            ACPI → platform_device creation
drivers/base/property.c        ★ the fwnode unification (T.4)
drivers/base/swnode.c          software nodes
kernel/resource.c              ★ the resource TREE, /proc/iomem
drivers/amba/bus.c             the partially-discoverable case (T.8)
Documentation/driver-api/driver-model/platform.rst ★
Documentation/devicetree/       (Ch. 32)
```

### 1.2 `platform_match()` — the four-way fallback

```c
static int platform_match(struct device *dev, const struct device_driver *drv)
{
	struct platform_device *pdev = to_platform_device(dev);
	struct platform_driver *pdrv = to_platform_driver(drv);

	if (pdev->driver_override)                     /* 0. explicit override wins */
		return !strcmp(pdev->driver_override, drv->name);

	if (of_driver_match_device(dev, drv))          /* 1. OF compatible */
		return 1;
	if (acpi_driver_match_device(dev, drv))        /* 2. ACPI _HID/_CID */
		return 1;
	if (pdrv->id_table)                            /* 3. platform_device_id name */
		return platform_match_id(pdrv->id_table, pdev) != NULL;

	return (strcmp(pdev->name, drv->name) == 0);   /* 4. bare name */
}
```

### 1.3 Creating platform devices from code (when it is legitimate)

```c
/* Modern, property-based — the correct way when you must instantiate from code */
static const struct property_entry my_props[] = {
	PROPERTY_ENTRY_U32("acme,max-speed", 400000),
	PROPERTY_ENTRY_BOOL("acme,inverted"),
	PROPERTY_ENTRY_STRING("label", "front-panel"),
	{ }
};
static const struct platform_device_info my_info = {
	.name       = "acme-widget",
	.id         = PLATFORM_DEVID_AUTO,
	.res        = my_resources,
	.num_res    = ARRAY_SIZE(my_resources),
	.properties = my_props,            /* ★ becomes a software node */
	.dma_mask   = DMA_BIT_MASK(32),
};

pdev = platform_device_register_full(&my_info);
...
platform_device_unregister(pdev);
```

This is legitimate for **MFD sub-devices** (one chip exposing a regulator + an RTC + a
watchdog, instantiated by the parent driver) and for x86 platform quirk drivers. It is
**not** legitimate for pseudo-devices — those belong on the `faux` bus (Ch. 30 T.2).

---

## 2. Practice

### Lab 31.1 — A complete platform driver with a device tree overlay

This is the canonical exercise of the chapter: a driver plus a DT description, bound together
at runtime.

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * acme-widget — a platform driver demonstrating every modern idiom.
 */
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

#include <linux/clk.h>
#include <linux/delay.h>
#include <linux/gpio/consumer.h>
#include <linux/io.h>
#include <linux/iopoll.h>
#include <linux/mod_devicetable.h>
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/property.h>
#include <linux/pm_runtime.h>
#include <linux/regulator/consumer.h>
#include <linux/reset.h>

#define WIDGET_CTRL     0x00
#define  CTRL_ENABLE    BIT(0)
#define  CTRL_RESET     BIT(1)
#define WIDGET_STATUS   0x04
#define  STATUS_READY   BIT(0)
#define WIDGET_SPEED    0x08
#define WIDGET_ID       0x0c

struct widget_soc_data {
	const char *name;
	u32         max_speed;
	u32         dma_bits;
	bool        has_reset;
};

static const struct widget_soc_data widget_v1 = {
	.name = "v1", .max_speed = 100000, .dma_bits = 32, .has_reset = false,
};
static const struct widget_soc_data widget_v2 = {
	.name = "v2", .max_speed = 400000, .dma_bits = 36, .has_reset = true,
};

struct widget {
	struct device               *dev;
	void __iomem                *base;
	struct clk                  *clk;
	struct regulator            *vdd;
	struct reset_control        *rst;
	struct gpio_desc            *enable_gpio;
	const struct widget_soc_data *cfg;
	u32                          speed;
	bool                         inverted;
	int                          irq;
	u64                          irq_count;
};

static irqreturn_t widget_isr(int irq, void *data)
{
	struct widget *w = data;
	u32 st = readl(w->base + WIDGET_STATUS);

	if (!st)
		return IRQ_NONE;
	writel(st, w->base + WIDGET_STATUS);      /* W1C ack */
	w->irq_count++;
	return IRQ_HANDLED;
}

static ssize_t speed_show(struct device *dev, struct device_attribute *a, char *buf)
{
	struct widget *w = dev_get_drvdata(dev);

	return sysfs_emit(buf, "%u\n", w->speed);
}
static ssize_t speed_store(struct device *dev, struct device_attribute *a,
			   const char *buf, size_t n)
{
	struct widget *w = dev_get_drvdata(dev);
	u32 v;
	int ret = kstrtou32(buf, 0, &v);

	if (ret)
		return ret;
	if (v > w->cfg->max_speed)
		return -EINVAL;
	w->speed = v;
	writel(v, w->base + WIDGET_SPEED);
	return n;
}
static DEVICE_ATTR_RW(speed);

static ssize_t irq_count_show(struct device *dev, struct device_attribute *a, char *buf)
{
	return sysfs_emit(buf, "%llu\n", ((struct widget *)dev_get_drvdata(dev))->irq_count);
}
static DEVICE_ATTR_RO(irq_count);

static struct attribute *widget_attrs[] = {
	&dev_attr_speed.attr, &dev_attr_irq_count.attr, NULL,
};
ATTRIBUTE_GROUPS(widget);

static int widget_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	struct widget *w;
	u32 val;
	int ret;

	w = devm_kzalloc(dev, sizeof(*w), GFP_KERNEL);
	if (!w)
		return -ENOMEM;
	w->dev = dev;
	platform_set_drvdata(pdev, w);

	w->cfg = device_get_match_data(dev);                    /* T.4 */
	if (!w->cfg)
		return dev_err_probe(dev, -ENODEV, "no match data\n");
	dev_info(dev, "variant %s (max speed %u)\n", w->cfg->name, w->cfg->max_speed);

	w->base = devm_platform_ioremap_resource(pdev, 0);      /* T.3 */
	if (IS_ERR(w->base))
		return PTR_ERR(w->base);

	w->clk = devm_clk_get_enabled(dev, NULL);
	if (IS_ERR(w->clk))
		return dev_err_probe(dev, PTR_ERR(w->clk), "clock\n");

	w->vdd = devm_regulator_get_enable_optional(dev, "vdd");
	if (IS_ERR(w->vdd) && PTR_ERR(w->vdd) != -ENODEV)
		return dev_err_probe(dev, PTR_ERR(w->vdd), "vdd\n");

	if (w->cfg->has_reset) {
		w->rst = devm_reset_control_get_exclusive(dev, NULL);
		if (IS_ERR(w->rst))
			return dev_err_probe(dev, PTR_ERR(w->rst), "reset\n");
		reset_control_deassert(w->rst);
	}

	w->enable_gpio = devm_gpiod_get_optional(dev, "enable", GPIOD_OUT_LOW);
	if (IS_ERR(w->enable_gpio))
		return dev_err_probe(dev, PTR_ERR(w->enable_gpio), "enable gpio\n");

	/* Optional properties with defaults (T.4) */
	if (device_property_read_u32(dev, "acme,speed-hz", &w->speed))
		w->speed = w->cfg->max_speed / 2;
	if (w->speed > w->cfg->max_speed)
		return dev_err_probe(dev, -EINVAL, "speed %u > max %u\n",
				     w->speed, w->cfg->max_speed);
	w->inverted = device_property_read_bool(dev, "acme,inverted");

	ret = dma_set_mask_and_coherent(dev, DMA_BIT_MASK(w->cfg->dma_bits));
	if (ret)
		return dev_err_probe(dev, ret, "dma mask\n");

	/* Bring the hardware up BEFORE requesting the IRQ (T.7) */
	writel(CTRL_RESET, w->base + WIDGET_CTRL);
	udelay(10);
	writel(0, w->base + WIDGET_CTRL);
	ret = readl_poll_timeout(w->base + WIDGET_STATUS, val,
				 val & STATUS_READY, 100, 100000);
	if (ret)
		return dev_err_probe(dev, ret, "hardware not ready\n");

	writel(w->speed, w->base + WIDGET_SPEED);
	gpiod_set_value_cansleep(w->enable_gpio, 1);

	w->irq = platform_get_irq_optional(pdev, 0);
	if (w->irq > 0) {
		ret = devm_request_irq(dev, w->irq, widget_isr, 0, dev_name(dev), w);
		if (ret)
			return dev_err_probe(dev, ret, "irq\n");
	}

	writel(CTRL_ENABLE, w->base + WIDGET_CTRL);

	pm_runtime_set_active(dev);
	devm_pm_runtime_enable(dev);

	dev_info(dev, "probed at %pR, id=0x%08x\n",
		 platform_get_resource(pdev, IORESOURCE_MEM, 0),
		 readl(w->base + WIDGET_ID));
	return 0;
}

static void widget_remove(struct platform_device *pdev)
{
	struct widget *w = platform_get_drvdata(pdev);

	writel(0, w->base + WIDGET_CTRL);          /* STOP before devres unwinds */
	gpiod_set_value_cansleep(w->enable_gpio, 0);
}

static void widget_shutdown(struct platform_device *pdev)
{
	struct widget *w = platform_get_drvdata(pdev);

	writel(0, w->base + WIDGET_CTRL);          /* ★ T.6: quiesce before reboot */
}

static const struct of_device_id widget_of_match[] = {
	{ .compatible = "acme,widget-v2", .data = &widget_v2 },
	{ .compatible = "acme,widget",    .data = &widget_v1 },
	{ }
};
MODULE_DEVICE_TABLE(of, widget_of_match);

static const struct platform_device_id widget_ids[] = {
	{ "acme-widget", (kernel_ulong_t)&widget_v1 },
	{ }
};
MODULE_DEVICE_TABLE(platform, widget_ids);

static struct platform_driver widget_driver = {
	.driver = {
		.name           = "acme-widget",
		.of_match_table = widget_of_match,
		.dev_groups     = widget_groups,
	},
	.id_table = widget_ids,
	.probe    = widget_probe,
	.remove   = widget_remove,
	.shutdown = widget_shutdown,
};
module_platform_driver(widget_driver);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Platform driver reference implementation");
```

**The device tree overlay** (`acme-widget.dts`):

```dts
/dts-v1/;
/plugin/;

&{/} {
	acme_widget: widget@10000000 {
		compatible = "acme,widget-v2";
		reg = <0x0 0x10000000 0x0 0x1000>;
		interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>;
		clocks = <&dummy_clk>;
		acme,speed-hz = <200000>;
		acme,inverted;
		enable-gpios = <&gpio0 5 GPIO_ACTIVE_HIGH>;
		status = "okay";
	};
};
```
```bash
dtc -@ -I dts -O dtb -o acme-widget.dtbo acme-widget.dts
sudo mkdir -p /sys/kernel/config/device-tree/overlays/acme
cat acme-widget.dtbo | sudo tee /sys/kernel/config/device-tree/overlays/acme/dtbo > /dev/null

sudo insmod acme-widget.ko
dmesg | tail -10
ls /sys/bus/platform/devices/ | grep widget
cat /sys/bus/platform/devices/10000000.widget/speed
cat /proc/iomem | grep -i widget
```

### Lab 31.2 — Instantiate the *same driver* three ways (T.4)

Prove the firmware-agnostic claim by binding the driver via DT, via a software node, and via
a bare name.

```c
/* (b) software node + property entries */
static const struct property_entry widget_props[] = {
	PROPERTY_ENTRY_U32("acme,speed-hz", 50000),
	PROPERTY_ENTRY_BOOL("acme,inverted"),
	{ }
};
static struct resource widget_res[] = {
	DEFINE_RES_MEM(0x10000000, 0x1000),
	DEFINE_RES_IRQ(42),
};
static const struct platform_device_info widget_info = {
	.name = "acme-widget", .id = PLATFORM_DEVID_AUTO,
	.res = widget_res, .num_res = ARRAY_SIZE(widget_res),
	.properties = widget_props,
};

pdev = platform_device_register_full(&widget_info);
```
```bash
# The driver code is IDENTICAL for all three. Verify:
dmesg | grep 'variant'
ls -l /sys/bus/platform/devices/*/of_node 2>/dev/null   # present only for the DT one
ls /sys/bus/platform/devices/acme-widget.*/             # the swnode one
```
**That is the whole point of T.4.** Write it down.

### Lab 31.3 — Explore the resource tree (T.3)

```bash
cat /proc/iomem
cat /proc/ioports

# The nesting IS the tree:
awk '{ n=gsub(/^ +/,""); printf "%*s%s\n", n, "", $0 }' /proc/iomem | head -40

# Who claimed what?
grep -n 'request_mem_region\|devm_request_mem_region' drivers/ -r | wc -l

# Provoke a conflict: two drivers claiming the same range
# (modify Lab 31.1 to register two devices at the same address)
dmesg | grep -i 'can.t request region\|resource conflict'

# The kernel's own reservations:
dmesg | grep -iE 'reserved|e820|memblock' | head -20
cat /sys/firmware/memmap/*/[ts]* 2>/dev/null | head
$EDITOR kernel/resource.c        # __request_region, __insert_resource
```

### Lab 31.4 — DT → platform_device, traced

```bash
# On an arm64 QEMU virt machine:
qemu-system-aarch64 -M virt,dumpdtb=/tmp/virt.dtb -cpu cortex-a72 -nographic
dtc -I dtb -O dts /tmp/virt.dtb | head -80

# Boot it, then inside the guest:
ls /sys/bus/platform/devices/
for d in /sys/bus/platform/devices/*/; do
  c=$(tr -d '\0' < $d/of_node/compatible 2>/dev/null)
  echo "$(basename $d): ${c:-<no of_node>}"
done

# Which DT nodes did NOT become devices, and why?
ls /proc/device-tree/
for n in /proc/device-tree/*/; do
  s=$(tr -d '\0' < $n/status 2>/dev/null)
  echo "$(basename $n): status=${s:-okay}"
done

# Trace the creation:
sudo bpftrace -e 'kprobe:of_platform_device_create_pdata { @ = count(); }'
grep -n 'simple-bus\|of_default_bus_match_table' drivers/of/platform.c
```

### Lab 31.5 — The `-EPROBE_DEFER` chain, for real

Build a three-driver dependency: a clock provider, a regulator provider, and a consumer.
Load them in the *wrong* order and watch deferral resolve it.

```bash
sudo insmod consumer.ko        # defers: no clock, no regulator
cat /sys/kernel/debug/devices_deferred
sudo insmod clkprovider.ko     # deferred list is retried; still needs the regulator
cat /sys/kernel/debug/devices_deferred
sudo insmod regprovider.ko     # now it probes
dmesg | tail -10

# And with fw_devlink, the order is enforced UP FRONT:
ls -l /sys/devices/platform/consumer*/supplier:* 2>/dev/null
dmesg | grep -i 'supplier\|fw_devlink'
```

### Lab 31.6 — Find board-file survivors and DT conversions

```bash
# Generation 1 remnants:
find arch/arm/mach-* -name 'board-*.c' 2>/dev/null | head
git log --oneline --grep='convert to DT\|device tree conversion' -- arch/arm/ | head -20

# How much did DT remove?
git diff --stat v3.0..v6.0 --  arch/arm/mach-omap2/ | tail -3

# Drivers still accepting platform_data (legacy):
git grep -n 'dev.platform_data\|dev_get_platdata' -- drivers/ | wc -l
git grep -l 'dev_get_platdata' -- drivers/ | head -20
# For each: is there an fwnode path too? Could platform_data be removed?
```

### Lab 31.7 — `shutdown` matters (T.6)

```c
/* Omit .shutdown, start a DMA loop, then reboot into kdump */
static void widget_shutdown(struct platform_device *pdev) { }   /* ← BAD */
```
```bash
# With kdump configured (Ch. 06):
echo c | sudo tee /proc/sysrq-trigger
# Examine the vmcore for DMA corruption from the still-running device.
# Then add a real shutdown and compare.
grep -rn '\.shutdown' drivers/net/ethernet/intel/*/ | head
grep -rn '\.shutdown' drivers/nvme/host/pci.c
```

---

## 3. Mastery drills

1. **Read `drivers/base/platform.c`** completely. Explain `platform_match()`'s ordering,
   `platform_get_irq()`'s deferral logic, and what `platform_device_register_full()` does
   with `properties`.

2. **The discoverability line.** For each bus — PCI, USB, I²C, SPI, MMC, AMBA, MDIO, virtio,
   platform — state which side of T.1 it is on and *why*, citing the hardware mechanism
   (or lack of one).

3. **Board files vs DT.** Read a surviving board file in `arch/arm/mach-*/`. Rewrite one
   device's description as a DT node. Then find the commit that did the real conversion and
   compare.

4. **Resource tree.** Read `kernel/resource.c`'s `__request_region()` and `__insert_resource()`.
   Explain how conflict detection works and what happens with `IORESOURCE_MUXED`.

5. **`of_platform_populate()` recursion.** Read `drivers/of/platform.c`. List exactly which
   `compatible` values cause recursion and explain why an I²C controller's children must
   *not* be populated as platform devices.

6. **fwnode audit.** Pick a driver using `of_property_read_*`. Determine whether every use
   could be `device_property_read_*`. Convert it and verify it still binds on DT.
   (`git log --grep="use device_property"` — hundreds of precedents.)

7. **AMBA.** Read `drivers/amba/bus.c` and `drivers/tty/serial/amba-pl011.c`. Explain the
   partial-discoverability model of T.8 and why the DT is still required.

8. **MFD.** Read an MFD driver (`drivers/mfd/`) that creates platform sub-devices. Explain
   why that is a legitimate use of `platform_device_register_*` and how the sub-devices get
   their resources and properties.

9. **Design question.** You have a new SoC block that exists in three variants across five
   SoCs, with different register offsets, clock counts, and an optional DMA engine. Design
   the driver: the match table, the `soc_data` struct, how variants are distinguished, and
   how an integrator adds a sixth SoC without touching driver code.

10. **Write the binding.** For Lab 31.1's device, write a proper DT binding in YAML
    (`Documentation/devicetree/bindings/`) and validate it with `make dt_binding_check`.
    (Ch. 32 covers this in depth — do it now and refine later.)

---

## 4. Further reading

**Kernel documentation:**
- `Documentation/driver-api/driver-model/platform.rst` ★
- `Documentation/driver-api/driver-model/overview.rst`
- `Documentation/devicetree/usage-model.rst` ★★ — **how DT maps to the device model**;
  the single best document for this chapter
- `Documentation/firmware-guide/acpi/enumeration.rst` — the ACPI equivalent
- `Documentation/driver-api/firmware/` — `request_firmware` and friends

**Source:**
- `drivers/base/platform.c` ★★
- `drivers/of/platform.c`, `drivers/of/address.c`
- `kernel/resource.c`
- `drivers/base/property.c`, `drivers/base/swnode.c`
- Exemplary drivers: `drivers/tty/serial/8250/8250_of.c` (simple),
  `drivers/i2c/busses/i2c-designware-platdrv.c` (multi-firmware),
  `drivers/mmc/host/sdhci-of-*.c` (variant handling)

**History (worth reading once, for context):**
- The 2011 ARM/board-file discussions on LKML and the "ARM consolidation" tree
- Grant Likely & Josh Boyer, "Flattened Device Trees for Embedded Linux" (LinuxCon)
- LWN: "Device trees I: the flattened device tree", "Device trees II", "ARM, the device
  tree, and the kernel", "The platform device problem" (and the `faux` follow-up)

→ Next: [32-device-tree.md](32-device-tree.md)

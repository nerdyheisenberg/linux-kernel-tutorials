# Chapter 32 — Device Tree: Syntax, Bindings, Overlays, and `fwnode`

> **Goal:** read and write any device tree, write a schema-validated binding, understand
> address translation and phandle specifiers from first principles, and know exactly what
> belongs in a DT and what does not.

---

## Theory & First Principles

### T.0 — Start here: the million lines of board files

Ch. 31 left the question open: the kernel must be *told* about non-discoverable hardware.
Told how?

Until roughly 2011 the answer was C, compiled into the kernel, one file per board:

```bash
# What arch/arm/ looked like at its peak:
#   arch/arm/mach-omap2/board-n8x0.c          1,100 lines
#   arch/arm/mach-omap2/board-rx51.c          1,300 lines
#   arch/arm/mach-omap2/board-overo.c           600 lines
#   ... times about 1,500 boards
```

**Why this was untenable**, and each reason maps to a property device tree was designed to
have:

| Problem | Consequence |
|---|---|
| A board revision that moves one IRQ needs a **new kernel** | You cannot ship one binary for a product family |
| A distribution cannot support ARM at all | One kernel cannot contain every board file |
| 1500 near-identical files | Every core API change touches all of them |
| The data is **executable** | A board file can do anything, so nothing can be validated |
| The hardware description is **in the kernel's licence and release cycle** | A board vendor must upstream or fork |

**The insight is a very old one**, and stating it generally is what makes this chapter worth
studying rather than memorizing:

> **Move configuration out of code and into data.** Code that varies per board becomes data
> that varies per board, and the code becomes one generic implementation.

```
   BEFORE                             AFTER
   ┌─────────────────┐              ┌─────────────────┐   ┌─────────────┐
   │ 1500 board .c   │              │ ONE generic     │   │ 1500 .dtb  │
   │ files, in the   │              │ driver, in the  │ + │ blobs, in  │
   │ kernel          │              │ kernel          │   │ FIRMWARE   │
   └─────────────────┘              └─────────────────┘   └─────────────┘
```

**What device tree actually is:** a tree of nodes, each with properties, describing hardware
that exists and how it is wired together.

```dts
uart0: serial@2020000 {
	compatible = "vendor,my-uart", "ns16550a";  /* ← what driver to use */
	reg = <0x02020000 0x1000>;                  /* ← where it is */
	interrupts = <GIC_SPI 72 IRQ_TYPE_LEVEL_HIGH>;
	clocks = <&clkc CLK_UART0>;                 /* ← what feeds it */
	clock-names = "core";
	status = "okay";
};
```

Four things to notice, because each is a design decision rather than syntax:

1. **`compatible` is a *list*, most specific first.** A driver claims `"ns16550a"`; a newer,
   better driver claims `"vendor,my-uart"`. **Old kernels work with new device trees** — this
   is forward compatibility built into the data format, and it is the same versioning
   problem as Ch. 24 §T.3 solved with a size field.
2. **`clocks = <&clkc CLK_UART0>` is a *phandle*** — a pointer into the tree. The device tree
   is not a flat list of devices; it is a **graph of relationships**, which is what lets the
   kernel resolve dependencies (and produce `-EPROBE_DEFER`, Ch. 27 §T.0).
3. **`status = "okay"`** means the board actually populated this. The SoC `.dtsi` describes
   everything the chip *could* have; the board `.dts` says what is *present*.
4. **It is data, so it is validatable.** Bindings are YAML schemas and
   `make dt_binding_check` verifies them — something no board file could ever support.

**And the crucial distinction that §T.2 develops:** the device tree describes **hardware**,
not Linux. It is an OS-independent description (it came from Open Firmware and is used by
BSD, U-Boot, Xen, and Zephyr). Putting Linux driver configuration in it — buffer sizes,
policy, which driver you prefer — is the most common mistake in DT review, and it is rejected
every time.

```bash
# The device tree the RUNNING kernel actually got (after bootloader fixups!):
dtc -I fs -O dts /sys/firmware/devicetree/base 2>/dev/null | head -40
ls /proc/device-tree/

# Compare with the source you built:
dtc -I dtb -O dts arch/arm64/boot/dts/vendor/board.dtb | head -40
```

---

### T.1 Hardware description as data, and why that is the whole point

Ch. 31 T.2 traced the move from board files (description as **code**, compiled into the
kernel) to device tree (description as **data**, passed by the bootloader). The consequences
of that single change are worth stating explicitly, because they are the argument for the
entire design:

| Property | Description as code | Description as **data** |
|---|---|---|
| Adding a board | patch and rebuild the kernel | ship a new `.dtb` |
| One kernel, many boards | impossible | **the norm** |
| Who owns the description | kernel developers | board/firmware vendors |
| Validation | the C compiler (types only) | **a schema** (T.7) |
| Introspection at runtime | none | `/proc/device-tree`, `/sys/firmware/devicetree` |
| Modification without rebuild | no | overlays (T.8) |
| Other consumers | none | U-Boot, ATF, Xen, FreeBSD, Zephyr, QEMU |

That last row matters more than people expect: because the DT is *data in a defined format*,
the same description is consumed by the bootloader, the hypervisor, the secure firmware, and
several operating systems. It became an **industry interchange format**, which is why it
outlived its Linux-specific motivation.

**The origin** is IEEE 1275 *Open Firmware* (1994), developed at Sun for SPARC/OpenBoot and
adopted by Apple and IBM for PowerPC. Open Firmware devices carried a live, interactive tree
(you could browse it from a Forth prompt). Embedded systems had no such firmware, so Benjamin
Herrenschmidt and others defined the **Flattened Device Tree (FDT)** — the same data model
serialized into a compact blob (`.dtb`) that a dumb bootloader can just hand over in a
register. ARM adopted it in 2011 under the pressure described in Ch. 31 T.2; RISC-V mandated
it from the start.

```bash
# The blob, and the tree it becomes:
ls /sys/firmware/fdt                       # the raw blob, if present
dtc -I fs -O dts /proc/device-tree | head -60
ls /proc/device-tree/
cat /proc/device-tree/model; echo
cat /proc/device-tree/compatible | tr '\0' '\n'
```

### T.2 The data model: a tree of nodes with typed-by-convention properties

```dts
/dts-v1/;

/ {
	model = "Acme Development Board v2";
	compatible = "acme,devboard-v2", "acme,devboard";
	#address-cells = <2>;
	#size-cells = <2>;

	chosen {
		bootargs = "console=ttyS0,115200 root=/dev/mmcblk0p2";
		stdout-path = &uart0;
	};

	memory@40000000 {
		device_type = "memory";
		reg = <0x0 0x40000000 0x0 0x80000000>;   /* 2 GiB at 0x40000000 */
	};

	cpus {
		#address-cells = <1>;
		#size-cells = <0>;
		cpu@0 {
			device_type = "cpu";
			compatible = "arm,cortex-a72";
			reg = <0>;
			enable-method = "psci";
		};
	};

	soc {
		compatible = "simple-bus";
		#address-cells = <2>;
		#size-cells = <2>;
		ranges;                                   /* identity mapping */

		uart0: serial@1c020000 {
			compatible = "snps,dw-apb-uart";
			reg = <0x0 0x1c020000 0x0 0x1000>;
			interrupts = <GIC_SPI 32 IRQ_TYPE_LEVEL_HIGH>;
			clocks = <&ccu CLK_BUS_UART0>;
			clock-names = "apb";
			resets = <&ccu RST_BUS_UART0>;
			reg-shift = <2>;
			reg-io-width = <4>;
			status = "okay";
		};
	};
};
```

The data model is deliberately tiny:

- A **node** has a name (`serial@1c020000` = `<name>@<unit-address>`), zero or more
  properties, and zero or more child nodes.
- A **property** is a name and an opaque byte array. That is all. There are **no types** in
  the format.
- A **label** (`uart0:`) is a source-level convenience that becomes a **phandle** (a u32
  identifier) in the blob.

**The absence of types is the single most important fact about DT.** `reg = <0x0 0x1c020000
0x0 0x1000>` is just sixteen bytes. Whether that means "one 64-bit address and one 64-bit
size" or "four 32-bit values" is determined *entirely* by `#address-cells` and `#size-cells`
in the **parent** node. Meaning lives in the **binding** (T.6), not in the format.

Conventional encodings (by convention, enforced only by the schema):

| Form | Meaning |
|---|---|
| `prop;` | boolean true (present = true, absent = false) |
| `prop = <1 2 3>;` | array of u32 "cells" |
| `prop = <0x0 0x40000000>;` | a 64-bit value as two cells (big-endian order) |
| `prop = "a string";` | NUL-terminated string |
| `prop = "one", "two";` | string list (concatenated, NUL-separated) |
| `prop = [00 11 22];` | raw bytes |
| `prop = <&label>;` | a phandle (cross-reference) |
| `prop = <&label 1 2>;` | a phandle **plus a specifier** (T.5) |

### T.3 `compatible`: a namespaced, ordered fallback list

```dts
compatible = "acme,widget-v2", "acme,widget", "generic,widget";
```

Two rules with real consequences:

1. **`"vendor,model"`**, where `vendor` must be registered in
   `Documentation/devicetree/bindings/vendor-prefixes.yaml`. This is a genuine namespace, and
   `make dt_binding_check` enforces it.
2. **Most specific first.** The kernel tries each string in order against every driver's
   `of_match_table`. So a new SoC can list its specific string *and* a generic fallback, and
   an old kernel that only knows the generic one still binds and works with reduced features.

That ordering is a **forward-compatibility mechanism** and it is the DT's answer to Ch. 24
T.3's extensibility problem. Adding a more-specific compatible is always safe; removing a
generic one is an ABI break.

```bash
# Every compatible in use on this machine:
find /proc/device-tree -name compatible -exec sh -c 'tr "\0" "\n" < {}' \; | sort -u | head -30
# Registered vendor prefixes:
grep -c '^  "\^' Documentation/devicetree/bindings/vendor-prefixes.yaml
# Which driver claims a given compatible?
git grep -rn '"snps,dw-apb-uart"' drivers/ | head
```

### T.4 Addressing: `reg`, `#address-cells`, `#size-cells`, and `ranges`

This is where most people get lost, and it is genuinely elegant once seen clearly.

**The DT models a *hierarchy of address spaces*.** Each bus node declares how many u32 cells
its children use to express an address and a size:

```dts
soc {
	#address-cells = <2>;     /* children's addresses take 2 cells (64-bit) */
	#size-cells    = <2>;     /* children's sizes take 2 cells */
	ranges = <0x0 0x0  0x0 0x0  0x1 0x0>;   /* child-addr, parent-addr, size */

	device@1c020000 {
		reg = <0x0 0x1c020000   0x0 0x1000>;
		/*     └── address ──┘  └── size ─┘  (2 cells each) */
	};
};

i2c@1c2ac00 {
	#address-cells = <1>;     /* I²C children use ONE cell... */
	#size-cells    = <0>;     /* ...and have no size */

	rtc@68 {
		reg = <0x68>;     /* just the I²C slave address */
	};
};
```

**`ranges` is the translation rule** from a child address space to the parent's:

```
ranges = <child-address  parent-address  length>;
```
- `ranges;` (empty) = **identity mapping**, child addresses are parent addresses.
- *Absent* `ranges` = **no translation possible** — the child address space is not mapped
  into the parent. This is correct for I²C/SPI (a slave address is not a CPU address) and is
  why `of_address_to_resource()` fails on such nodes.

Address translation therefore walks up the tree applying each `ranges` in turn, exactly like
a PCI bridge's address decoder — which is not a coincidence, since Open Firmware was designed
around such buses.

```dts
/* A non-identity example: a bus window */
pcie@1000000 {
	#address-cells = <3>;      /* PCI addresses are 3 cells: phys.hi/mid/lo */
	#size-cells = <2>;
	ranges = <0x02000000 0x0 0x40000000   0x0 0x40000000   0x0 0x40000000>;
	/*        PCI mem     PCI addr         CPU addr         size            */
};
```

```bash
# Watch the translation happen:
cat /proc/device-tree/soc/\#address-cells | xxd
cat /proc/device-tree/soc/ranges | xxd
# The result, after translation:
cat /proc/iomem | head -20
grep -n 'of_translate_address\|__of_translate_address' drivers/of/address.c
```

### T.5 Phandles and specifiers: provider-defined argument encoding

A phandle turns the tree into a **DAG** — nodes reference other nodes:

```dts
uart0: serial@1c020000 {
	clocks = <&ccu CLK_BUS_UART0>, <&osc24M>;
	clock-names = "apb", "mod";
	interrupts = <GIC_SPI 32 IRQ_TYPE_LEVEL_HIGH>;
	interrupt-parent = <&gic>;
	dmas = <&dma 6>, <&dma 6>;
	dma-names = "rx", "tx";
	reset-gpios = <&pio 3 5 GPIO_ACTIVE_LOW>;
	pinctrl-names = "default";
	pinctrl-0 = <&uart0_pins>;
};
```

The genuinely clever part is **specifiers**. `<&ccu CLK_BUS_UART0>` is a phandle followed by
*N* cells, where **the provider declares N**:

```dts
ccu: clock-controller@1c20000 {
	compatible = "allwinner,sun50i-a64-ccu";
	#clock-cells = <1>;        /* ★ my consumers pass ONE cell to identify a clock */
	#reset-cells = <1>;
};

gic: interrupt-controller@1c81000 {
	interrupt-controller;
	#interrupt-cells = <3>;    /* ★ type, number, flags */
};

pio: pinctrl@1c20800 {
	gpio-controller;
	#gpio-cells = <3>;         /* ★ bank, pin, flags */
};
```

So `#<thing>-cells` is a **provider-defined calling convention**, and the consumer's property
is an argument list. This is a form of *polymorphism*: the DT format does not know what a
clock specifier means; the clock controller's driver does, via `->of_xlate()`:

```c
static struct clk_hw *my_of_clk_get(struct of_phandle_args *spec, void *data)
{
	unsigned int idx = spec->args[0];        /* ← the cell the consumer wrote */

	if (idx >= my_nr_clocks)
		return ERR_PTR(-EINVAL);
	return &my_clocks[idx].hw;
}
of_clk_add_hw_provider(node, my_of_clk_get, priv);
```

**And `*-names` gives them symbolic access**, which is what lets a driver say
`devm_clk_get(dev, "apb")` instead of caring about index 0. **Always use the `-names` form in
drivers** — index-based access breaks the moment someone adds a clock.

**This phandle graph is exactly what `fw_devlink` parses** (Ch. 27 T.5) to build the
dependency DAG before any driver probes. `drivers/of/property.c` contains a table
(`of_supplier_bindings[]`) mapping each property name to how to extract the supplier — read
it; it is the complete list of what DT relationships the kernel understands.

### T.6 The binding *is* the specification

The `.dts` file says `acme,speed-hz = <200000>`. Nothing in the format says what that means,
what units, what range, or whether it is required. **The binding does.** And the binding is
the actual contract between hardware vendors and the kernel.

For twenty years bindings were prose in `.txt` files — unvalidated, inconsistent, and
routinely violated. Since ~2018 they are **YAML schemas** (`dt-schema` / `dtschema`),
machine-checkable against every `.dts` in the tree:

```yaml
# Documentation/devicetree/bindings/misc/acme,widget.yaml
# SPDX-License-Identifier: GPL-2.0
%YAML 1.2
---
$id: http://devicetree.org/schemas/misc/acme,widget.yaml#
$schema: http://devicetree.org/meta-schemas/core.yaml#

title: Acme Widget Controller

maintainers:
  - Your Name <you@example.com>

properties:
  compatible:
    oneOf:
      - const: acme,widget
      - items:
          - const: acme,widget-v2
          - const: acme,widget

  reg:
    maxItems: 1

  interrupts:
    maxItems: 1

  clocks:
    items:
      - description: APB bus clock
      - description: module clock

  clock-names:
    items:
      - const: apb
      - const: mod

  resets:
    maxItems: 1

  vdd-supply:
    description: Main power supply

  enable-gpios:
    maxItems: 1

  acme,speed-hz:
    $ref: /schemas/types.yaml#/definitions/uint32
    minimum: 1000
    maximum: 400000
    default: 100000
    description: Operating frequency of the widget engine.

  acme,inverted:
    type: boolean
    description: Set if the output polarity is inverted on this board.

required:
  - compatible
  - reg
  - clocks
  - clock-names

additionalProperties: false        # ★ reject anything undocumented

examples:
  - |
    #include <dt-bindings/interrupt-controller/arm-gic.h>
    widget@10000000 {
        compatible = "acme,widget-v2", "acme,widget";
        reg = <0x10000000 0x1000>;
        interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>;
        clocks = <&ccu 10>, <&ccu 11>;
        clock-names = "apb", "mod";
        acme,speed-hz = <200000>;
    };
```

```bash
pip3 install --user dtschema yamllint
make dt_binding_check DT_SCHEMA_FILES=Documentation/devicetree/bindings/misc/acme,widget.yaml
make dtbs_check DT_SCHEMA_FILES=...            # validate every in-tree .dts against it
make CHECK_DTBS=y arch/arm64/boot/dts/vendor/board.dtb
```

**`additionalProperties: false` is the line that matters.** It converts "undocumented
properties are tolerated" into "undocumented properties are an error" — the same
strictness-first principle as Ch. 24 T.1 (*strictness relaxes, permissiveness never
tightens*). The migration of ~4000 `.txt` bindings to YAML is one of the largest
documentation efforts in kernel history and is still ongoing — **converting one is an
excellent, genuinely welcome first contribution.**

```bash
ls Documentation/devicetree/bindings/**/*.txt 2>/dev/null | wc -l   # remaining work
ls Documentation/devicetree/bindings/**/*.yaml 2>/dev/null | wc -l
git log --oneline --grep='dt-bindings.*convert to YAML' | head -10
```

### T.7 The DT is ABI — with a specific, limited meaning

A `.dtb` is often flashed into a board's SPI-NOR alongside the bootloader and **never
updated**, while the kernel is updated for years. Therefore:

> **A device tree that worked with kernel version *N* must keep working with version *N+k*.**

Consequences, and they constrain design hard:

- **You cannot rename a property or a compatible string.** Ever.
- **You cannot make an optional property required.**
- **You cannot change a specifier's cell count** (`#clock-cells`) — every existing consumer
  breaks.
- **You cannot change the meaning of an existing cell.**
- You *can* add new optional properties and new (more specific) compatible strings.

The counter-pressure is that DTs in the kernel tree are *also* updated with the kernel, so
there is a persistent argument about how strictly this applies. The practical position
(stated by DT maintainers): **treat it as ABI; if you must break it, you need a very strong
justification and a transition plan.** The `dtbs_check` infrastructure exists partly to catch
accidental breakage.

This stability requirement is *why* the binding review process is so demanding. A binding is
forever; a driver is not.

### T.8 What does **not** belong in a device tree

This is the single most common review rejection, and the rule is principled:

> **The DT describes *hardware*, not *software configuration*, and not *policy*.**

| Belongs in DT | Does **not** belong in DT |
|---|---|
| Register addresses, IRQ numbers | Which driver to use |
| Clock/regulator/GPIO connections | Buffer sizes, timeouts, thread counts |
| Bus topology and addressing | Debug flags, log levels |
| Physical board wiring (`gpios`, `phy-mode`, `bus-width`) | Filesystem paths, IP addresses |
| Immutable properties of the silicon | Anything a user might reasonably want to change |
| `chosen` (bootloader→kernel handoff) | Kernel policy decisions |

The test: **"if I redesigned the board, would this change?"** If yes, it is hardware. If it
would change because you changed your mind about software behaviour, it is configuration, and
it belongs in a module parameter, sysfs, or a config file.

Grey areas that generate real arguments: `max-frequency` (a board-level electrical limit —
legitimate), `num-cs` (hardware — legitimate), `fifo-depth` (hardware — legitimate),
`rx-buffer-size` (software — not legitimate), `interrupt-affinity` (policy — not legitimate).

### T.9 Overlays: runtime modification, and why they are hard

An **overlay** is a DT fragment applied at runtime to add or modify nodes — the mechanism
behind Raspberry Pi HATs, BeagleBone capes, and FPGA partial reconfiguration:

```dts
/dts-v1/;
/plugin/;

&i2c1 {                          /* target an existing node by label */
	status = "okay";

	rtc@68 {
		compatible = "dallas,ds1307";
		reg = <0x68>;
	};
};
```

```bash
dtc -@ -I dts -O dtb -o rtc.dtbo rtc-overlay.dts     # -@ keeps symbols for label resolution
sudo mkdir -p /sys/kernel/config/device-tree/overlays/rtc
cat rtc.dtbo | sudo tee /sys/kernel/config/device-tree/overlays/rtc/dtbo >/dev/null
ls /sys/bus/i2c/devices/
sudo rmdir /sys/kernel/config/device-tree/overlays/rtc    # remove
```

The hard problems, which is why overlay support remains partly out-of-tree and contentious:

1. **Removal is genuinely hard.** Applying an overlay creates devices; removing it must
   destroy them. But a device may be in use (mounted filesystem, open fd, running DMA). The
   general "remove arbitrary hardware described by a fragment" problem is equivalent to
   forced hot-unplug of arbitrary devices.
2. **Ordering and dependencies.** An overlay may reference nodes created by another overlay.
3. **Validation.** An overlay can describe hardware that is not present, or conflict with the
   base tree; the kernel cannot check.
4. **`-@` symbol tables** must exist in the base DTB for label targets to resolve, which
   requires cooperation from whoever built it.

The kernel exposes overlays via `configfs` (`CONFIG_OF_OVERLAY` + `OF_CONFIGFS`) and via
in-kernel APIs (`of_overlay_fdt_apply()`) used by the FPGA manager and some connector
drivers. **For most development, applying an overlay from U-Boot before boot is far simpler
and is what production systems do.**

---

## 1. Internals

### 1.1 The blob format

```
┌─────────────────────┐
│ struct fdt_header   │  magic 0xd00dfeed, totalsize, offsets, version
├─────────────────────┤
│ memory reservation  │  ranges the kernel must not use
│ block               │
├─────────────────────┤
│ structure block     │  FDT_BEGIN_NODE / FDT_PROP / FDT_END_NODE tokens
├─────────────────────┤
│ strings block       │  all property NAMES, deduplicated
└─────────────────────┘
```
Everything is **big-endian** regardless of the CPU — a consequence of its PowerPC origin, and
the reason you see `be32_to_cpu()` throughout `drivers/of/`. Property names live in a
deduplicated string block, which is why a DTB for a complex SoC is only ~50–100 KB.

At boot, `unflatten_device_tree()` converts the blob into a linked structure of
`struct device_node` / `struct property` that the kernel walks:

```c
struct device_node {
	const char        *name, *full_name;
	phandle            phandle;
	struct property   *properties;
	struct device_node *parent, *child, *sibling;
	struct kobject     kobj;          /* → /sys/firmware/devicetree/base/ */
	unsigned long      _flags;
	void              *data;
};
```

### 1.2 Source map

```
drivers/of/base.c        ★ of_find_*, of_property_read_*, phandle resolution
drivers/of/address.c     ★ of_translate_address, of_address_to_resource (T.4)
drivers/of/irq.c         of_irq_parse_one, irq_of_parse_and_map (T.5)
drivers/of/platform.c    ★ of_platform_populate (Ch. 31 T.5)
drivers/of/property.c    ★★ of_supplier_bindings[] — the fw_devlink table (T.5)
drivers/of/fdt.c         ★ unflatten_device_tree, early_init_dt_scan
drivers/of/overlay.c     T.9
drivers/of/unittest.c    ★ an executable specification of DT semantics
scripts/dtc/             the device tree compiler (imported from upstream dtc)
include/linux/of.h, of_device.h, of_address.h, of_irq.h
include/dt-bindings/     ★ the C/DTS shared constant headers
Documentation/devicetree/
   usage-model.rst       ★★ how DT maps to the device model
   bindings/             ★★ the schemas
   bindings/writing-bindings.rst   ★ the review checklist
   bindings/writing-schema.rst     ★ how to write YAML
   overlay-notes.rst
```

`include/dt-bindings/` is worth calling out: it contains headers included by **both** `.dts`
files and C code, so `CLK_BUS_UART0` means the same number in both. That shared-constant
mechanism is what makes specifiers readable.

---

## 2. Practice

### Lab 32.1 — Read your machine's device tree, completely

```bash
# Decompile the live tree
dtc -I fs -O dts /proc/device-tree > /tmp/live.dts 2>/dev/null
wc -l /tmp/live.dts && head -80 /tmp/live.dts

# Or from the raw blob
dtc -I dtb -O dts /sys/firmware/fdt > /tmp/fdt.dts 2>/dev/null

# QEMU can hand you one directly:
qemu-system-aarch64 -M virt,dumpdtb=/tmp/virt.dtb -cpu cortex-a72 -nographic
dtc -I dtb -O dts /tmp/virt.dtb | less

# Navigate the filesystem view
ls /proc/device-tree/
cat /proc/device-tree/model; echo
cat /proc/device-tree/compatible | tr '\0' '\n'
cat /proc/device-tree/chosen/bootargs; echo
xxd /proc/device-tree/soc/serial@*/reg 2>/dev/null | head -2

# Every node with a compatible, and whether a device was created for it:
find /proc/device-tree -name compatible | while read f; do
  n=$(dirname "$f")
  printf "%-50s %s\n" "${n#/proc/device-tree}" "$(tr '\0' ' ' < "$f")"
done | head -40
```

### Lab 32.2 — Decode address translation by hand (T.4)

```bash
N=/proc/device-tree/soc
echo "#address-cells = $(xxd -p $N/\#address-cells)"
echo "#size-cells    = $(xxd -p $N/\#size-cells)"
xxd $N/ranges

# Pick a device and decode its reg by hand:
D=$(find /proc/device-tree/soc -maxdepth 1 -name 'serial@*' | head -1)
xxd $D/reg
# With #address-cells=2 #size-cells=2, that is:
#   bytes 0-7   = address (big-endian 64-bit)
#   bytes 8-15  = size
# Verify against:
cat /proc/iomem | grep -i serial

# Now the non-translatable case:
I=$(find /proc/device-tree -maxdepth 3 -name 'i2c@*' | head -1)
ls $I/ranges 2>&1        # absent → no translation (T.4)
xxd $I/\#address-cells   # 1
xxd $I/\#size-cells      # 0
```
**Write down the decoding by hand before checking.** Doing this once makes DT addressing
permanent knowledge.

### Lab 32.3 — Write a binding and validate it (T.6)

```bash
pip3 install --user dtschema yamllint
mkdir -p Documentation/devicetree/bindings/misc
$EDITOR Documentation/devicetree/bindings/misc/acme,widget.yaml   # use the T.6 template

# Validate the schema itself
make dt_binding_check DT_SCHEMA_FILES=Documentation/devicetree/bindings/misc/acme,widget.yaml

# Now deliberately break it and watch the checker:
#   - use an unregistered vendor prefix     -> "compatible: ... does not match"
#   - omit 'maintainers'                    -> schema error
#   - add a property not in the schema with additionalProperties:false
#   - put acme,speed-hz = <999999> (> maximum)

# Validate a real board against it:
make ARCH=arm64 CHECK_DTBS=y arch/arm64/boot/dts/vendor/board.dtb
make ARCH=arm64 dtbs_check 2>&1 | head -40
```

### Lab 32.4 — Convert a `.txt` binding to YAML (a real contribution)

```bash
ls Documentation/devicetree/bindings/**/*.txt | shuf | head -5
# Pick a small one. Then:
#  1. Read it carefully; note every property, required/optional, and type.
#  2. Write the YAML with additionalProperties: false.
#  3. make dt_binding_check DT_SCHEMA_FILES=<your file>
#  4. make dtbs_check — this WILL find real .dts violations.
#  5. Fix the .dts files too, or note them.
#  6. git rm the .txt, git add the .yaml.
git log --oneline --grep='convert.*to.*YAML' -- Documentation/devicetree/ | head -5
git show <one of those>    # study the format of a good conversion patch
```
**Step 4 is the interesting one:** schemas routinely reveal that in-tree device trees have
been violating their own documentation for years.

### Lab 32.5 — Build and apply an overlay (T.9)

```bash
# Base kernel needs: CONFIG_OF_OVERLAY=y, CONFIG_OF_CONFIGFS=y
./scripts/config -e OF_OVERLAY -e OF_CONFIGFS && make -j$(nproc)

cat > /tmp/rtc-overlay.dts <<'EOF'
/dts-v1/;
/plugin/;

&{/soc/i2c@1c2ac00} {
	status = "okay";
	#address-cells = <1>;
	#size-cells = <0>;

	rtc@68 {
		compatible = "dallas,ds1307";
		reg = <0x68>;
	};
};
EOF
dtc -@ -I dts -O dtb -o /tmp/rtc.dtbo /tmp/rtc-overlay.dts

sudo mount -t configfs none /sys/kernel/config 2>/dev/null
sudo mkdir -p /sys/kernel/config/device-tree/overlays/rtc
cat /tmp/rtc.dtbo | sudo tee /sys/kernel/config/device-tree/overlays/rtc/dtbo >/dev/null
cat /sys/kernel/config/device-tree/overlays/rtc/status

ls /sys/bus/i2c/devices/
ls /proc/device-tree/soc/i2c@1c2ac00/
dmesg | tail -10

# Removal — and the hard part:
sudo rmdir /sys/kernel/config/device-tree/overlays/rtc
dmesg | tail -5
```
**Then make removal fail**: bind a driver that holds a reference (open an fd to the RTC's
`/dev/rtc0`) and try to remove the overlay. Document what happens — that is T.9's problem,
first-hand.

### Lab 32.6 — Trace the phandle graph and `fw_devlink` (T.5)

```bash
# Every phandle reference in the tree:
find /proc/device-tree -name 'clocks' -o -name 'interrupts-extended' -o -name 'dmas' \
  -o -name '*-gpios' -o -name 'phys' -o -name 'resets' | head -20

# The table the kernel uses to interpret them:
grep -n -A80 'of_supplier_bindings\[\]' drivers/of/property.c | head -90

# The resulting device links:
for d in /sys/devices/platform/*/; do
  s=$(ls -d $d/supplier:* 2>/dev/null | wc -l)
  c=$(ls -d $d/consumer:* 2>/dev/null | wc -l)
  [ "$s$c" != "00" ] && echo "$(basename $d): suppliers=$s consumers=$c"
done

# Render the dependency DAG:
{ echo 'digraph G { rankdir=LR;'
  for d in /sys/devices/platform/*/; do
    for s in $d/supplier:*; do
      [ -e "$s" ] || continue
      echo "  \"$(basename $s | sed 's/supplier://')\" -> \"$(basename $d)\";"
    done
  done
  echo '}'; } > /tmp/devlinks.dot
dot -Tsvg /tmp/devlinks.dot -o /tmp/devlinks.svg
```

### Lab 32.7 — Add a device to QEMU's virt machine

```bash
# 1. Dump the base tree
qemu-system-aarch64 -M virt,dumpdtb=/tmp/virt.dtb -cpu cortex-a72 -nographic
dtc -I dtb -O dts /tmp/virt.dtb > /tmp/virt.dts

# 2. Add your node (from Ch. 31 Lab 31.1)
cat >> /tmp/virt-mod.dts <<'EOF'
/ {
	widget@9020000 {
		compatible = "acme,widget-v2", "acme,widget";
		reg = <0x0 0x9020000 0x0 0x1000>;
		interrupts = <0 10 4>;
		clocks = <&apb_pclk>;
		clock-names = "apb";
		acme,speed-hz = <200000>;
	};
};
EOF
# (merge into virt.dts, then:)
dtc -I dts -O dtb -o /tmp/virt-mod.dtb /tmp/virt-mod.dts

# 3. Boot with your modified tree
qemu-system-aarch64 -M virt -cpu cortex-a72 -m 1G \
  -dtb /tmp/virt-mod.dtb \
  -kernel Image -initrd initramfs.cpio.gz \
  -append "console=ttyAMA0 rdinit=/init" -nographic

# 4. In the guest:
ls /proc/device-tree/widget@9020000/
insmod acme-widget.ko
dmesg | tail
ls /sys/bus/platform/devices/ | grep widget
```
**This closes the loop**: you wrote a driver (Ch. 31), a binding (Lab 32.3), and a device
tree, and bound them together. That is the complete embedded workflow.

### Lab 32.8 — Debug "my device didn't probe"

The most common bring-up failure, with a systematic procedure:

```bash
# 1. Does the node exist and is it enabled?
ls /proc/device-tree/soc/mydev@10000000/ || echo "NODE MISSING"
tr -d '\0' < /proc/device-tree/soc/mydev@10000000/status; echo   # must be okay/absent

# 2. Was a platform_device created? (Ch. 31 T.5)
ls /sys/bus/platform/devices/ | grep 10000000
#   If not: is the parent a simple-bus? Does it have a reg with a unit-address?

# 3. Is the driver loaded and does its compatible match EXACTLY?
lsmod | grep mydrv
tr '\0' '\n' < /proc/device-tree/soc/mydev@10000000/compatible
git grep -n 'compatible.*=.*"' drivers/.../mydrv.c

# 4. Is it deferred? (Ch. 27 T.4)
cat /sys/kernel/debug/devices_deferred

# 5. Did probe fail?
dmesg | grep -i 'mydrv\|probe'
echo 'file drivers/.../mydrv.c +p' | sudo tee /sys/kernel/debug/dynamic_debug/control

# 6. fw_devlink blocking it?
dmesg | grep -i 'supplier\|fw_devlink'
ls -l /sys/devices/platform/10000000.mydev/supplier:* 2>/dev/null
# temporarily: boot with fw_devlink=permissive

# 7. Force it, to isolate:
echo 10000000.mydev | sudo tee /sys/bus/platform/drivers/mydrv/bind
```
**Turn this into a checklist and keep it.** It will save you days in Part 7.

---

## 3. Mastery drills

1. **Decode a DTB by hand.** `xxd /sys/firmware/fdt | head -20`. Identify the magic,
   `totalsize`, and the offsets. Then find the first `FDT_BEGIN_NODE` token (0x00000001) and
   parse one node manually. Check with `dtc`.

2. **Address translation proof.** Find a node three levels deep with non-identity `ranges` at
   two levels. Compute its CPU physical address by hand and verify against `/proc/iomem`.
   Then read `of_translate_address()` and confirm your algorithm matches.

3. **Write a provider.** Implement a dummy clock controller with `#clock-cells = <1>` and an
   `->of_xlate` that maps cell values to clocks. Then consume it from Lab 31.1's driver with
   `clocks = <&dummy 2>`. Explain what `of_clk_add_hw_provider()` registers.

4. **`of_supplier_bindings[]`.** Read the table in `drivers/of/property.c`. For three
   entries, trace how `fw_devlink` extracts the supplier node from the property. What happens
   for a property not in the table?

5. **Binding review.** Read `Documentation/devicetree/bindings/writing-bindings.rst` and then
   review three recent binding patches on the `devicetree@vger.kernel.org` list. Write the
   review you would have sent. Compare with Rob Herring's and Krzysztof Kozlowski's.

6. **The hardware/software line (T.8).** Find three in-tree bindings containing properties
   you believe are software configuration, not hardware. Justify each. (They exist — early
   bindings were reviewed less strictly.)

7. **ABI stability.** Find a case where a DT binding *was* broken
   (`git log --grep='dt-bindings' --grep='incompatible\|break' --all-match`). What was the
   justification and the transition plan?

8. **Overlay removal.** Read `drivers/of/overlay.c`. Explain the `changeset` mechanism and
   why removal can fail. Then design (on paper) what would be needed to make removal always
   safe. Why has nobody done it?

9. **Compare with ACPI.** For the same hypothetical device, write the DT node *and* the ACPI
   ASL (Ch. 33). Compare expressiveness: what can each say that the other cannot?

10. **Read `drivers/of/unittest.c`** and the `drivers/of/unittest-data/` tree. Explain what
    it validates and run it (`CONFIG_OF_UNITTEST=y`). This is an executable specification of
    DT semantics — the fastest way to settle "does DT allow X?".

11. **Full board bring-up rehearsal.** Take QEMU's `virt` DTS, add: an I²C controller node,
    an EEPROM at 0x50 under it, a GPIO controller with `#gpio-cells = <2>`, and a LED node
    using a GPIO from it. Write bindings for anything new. Boot it and verify all four
    devices appear.

---

## 4. Further reading

**Specifications:**
- **Devicetree Specification v0.4+** — https://devicetree-specification.readthedocs.io ★★
  (the normative document; ~60 pages, read the chapters on the data model and addressing)
- IEEE 1275-1994 (Open Firmware) — the ancestor; skim for context
- https://devicetree.org — schemas, tooling, the `dt-schema` project

**Kernel documentation:**
- `Documentation/devicetree/usage-model.rst` ★★ — **read this first**
- `Documentation/devicetree/bindings/writing-bindings.rst` ★★ — the review checklist
- `Documentation/devicetree/bindings/writing-schema.rst` ★ — YAML how-to
- `Documentation/devicetree/bindings/example-schema.yaml` — an annotated template
- `Documentation/devicetree/overlay-notes.rst`
- `Documentation/devicetree/bindings/vendor-prefixes.yaml`
- `Documentation/devicetree/booting-without-of.rst` — the boot-protocol side

**Source:**
- `drivers/of/base.c`, `address.c`, `property.c`, `platform.c`, `fdt.c`
- `drivers/of/unittest.c` — the executable spec
- `scripts/dtc/` — the compiler; `libfdt` is worth knowing for bootloader work
- `include/dt-bindings/` — the shared-constant mechanism

**Tools:**
- `dtc`, `fdtget`, `fdtput`, `fdtdump` (from `device-tree-compiler`)
- `dtschema` / `dt-doc-validate` / `dt-validate`
- `dtx_diff` (in `scripts/`) — diff two device trees meaningfully

**LWN & talks:**
- "Device trees I/II/III" (Corbet) — the introductory series
- "Device tree schemas" / "Validating device trees" (Rob Herring)
- Rob Herring's ELC talks on DT validation and the YAML migration
- Thomas Petazzoni / Bootlin's "Device Tree for Dummies" (ELC) — **the best introductory
  talk**, and the slides are excellent reference material
- Frank Rowand's ELC talks on overlays and DT debugging

→ Next: [33-acpi.md](33-acpi.md)

# Chapter 33 — ACPI for Driver Writers

> **Goal:** understand the *other* firmware description model — why it contains a bytecode
> interpreter, how `_DSD` made one driver work on both DT and ACPI, and how to debug a
> platform whose behaviour lives in firmware you cannot read.

---

## Theory & First Principles

### T.0 — Start here: firmware that runs code

Ch. 32's device tree is *data*: a tree of nodes and properties, parsed by the kernel. ACPI
solves the same problem — describing non-discoverable hardware — and makes the opposite
choice.

```asl
/* ACPI Source Language. This is not data. This is a PROGRAM. */
Device (LID0) {
    Name (_HID, EisaId ("PNP0C0D"))
    Method (_LID, 0, NotSerialized) {      /* ← a METHOD. The kernel CALLS it. */
        Store (^^GPIO.READ (0x1F), Local0)
        If (LEqual (Local0, One)) { Return (One) }
        Return (Zero)
    }
}
```

**The kernel contains an interpreter for this language** (ACPICA, ~50,000 lines in
`drivers/acpi/acpica/`) and executes firmware-supplied bytecode to answer questions about the
hardware.

**Why would anyone design it this way?** Because the two ecosystems had genuinely different
problems:

| | Device tree (ARM/embedded) | ACPI (x86/servers) |
|---|---|---|
| Who writes it | The **board designer**, usually with kernel access | The **OEM's firmware team**, who ship before the OS exists |
| Ships with | The kernel/bootloader, updatable together | The **BIOS**, updated separately and rarely |
| Hardware variety | Vast, but each board is fixed and known | Vast, and must work with an OS written years later |
| Model | **Describe** the hardware; the OS drives it | **Abstract** the hardware; firmware drives it |
| Consequence | New hardware behaviour → new kernel driver | New hardware behaviour → new AML, **same kernel** |

**That last row is the actual argument for ACPI**, and it is a strong one. Windows Vista can
boot a laptop designed in 2024 because the firmware, not the OS, knows how to read the lid
switch. A device tree cannot do that — it can only *describe*, so anything novel needs kernel
code.

**And the argument against is equally strong.** You are executing untrusted, unversioned,
vendor-written bytecode in the kernel, to answer questions about hardware:

- **You cannot debug it.** The source is not shipped. `acpidump` + `iasl -d` gives you
  decompiled AML, which is how people actually diagnose these.
- **It is frequently buggy**, and bugs are in firmware you cannot patch. Hence
  `drivers/acpi/` being full of DMI-keyed quirks for specific laptop models.
- **It has been a security boundary failure.** Firmware-supplied code running in the kernel
  is exactly what `lockdown` restricts (Ch. 102 §T.8: ACPI table override is blocked).
- **"It works on Windows"** is the real specification. AML is written and tested against
  whatever Windows does, so Linux must bug-compatibly match it — which is why
  `acpi_osi=` exists.

**The practical position, which is what you need for an interview:** both are the same
*architectural* answer — firmware describes non-discoverable hardware — with the
data-versus-code axis flipped. ARM servers use ACPI (SBBR requires it) precisely for the
OS-independence argument; ARM embedded uses DT because the board and kernel ship together.
**The mechanism you use follows from who controls the release cycle**, which is an
organizational fact, not a technical one (Ch. 00 §T.6).

```bash
ls /sys/firmware/acpi/tables/               # DSDT, SSDT, MADT, MCFG, FADT...
sudo acpidump -b -t DSDT -o /tmp/dsdt.dat && iasl -d /tmp/dsdt.dat
less /tmp/dsdt.dsl                          # read the code your firmware runs

cat /sys/firmware/acpi/interrupts/*         # SCI/GPE counts -- a storm here is a
                                            # classic "why is my laptop hot" cause
dmesg | grep -i 'ACPI.*BIOS bug\|ACPI Error'
```

---

### T.1 Two philosophies of firmware description

Ch. 32 presented device tree: **declarative data**. ACPI answers the same question — "what
hardware is here?" — with a fundamentally different philosophy, and the difference is worth
stating sharply because it explains everything else.

| | **Device Tree** | **ACPI** |
|---|---|---|
| Model | declarative **data** | data **+ executable bytecode (AML)** |
| Who implements platform behaviour | **the OS** (drivers) | **the firmware** (AML methods) |
| Who owns the description | the OS community, largely | the platform/BIOS vendor |
| Coupling | OS must know each SoC | OS calls generic methods |
| Debuggability | fully inspectable | **firmware is a black box you must interpret** |
| Updatability | reflash the DTB | BIOS update |
| Typical domain | embedded, SoCs, RISC-V | PCs, servers, ARM servers (SBBR) |

The design intent of ACPI (Intel/Microsoft/Toshiba, 1996) was: **the OS should not need to
know platform-specific sequencing.** Powering on a laptop's Wi-Fi card might require
asserting three GPIOs in a specific order with specific delays, flipping a PMIC register over
I²C, and waiting for a status bit. In the DT model, a driver encodes that. In the ACPI model,
the firmware provides a `_PS0` method containing that sequence, and the OS just *calls* it.

**That is why ACPI contains a programming language.** AML (ACPI Machine Language) is a
bytecode; `\_SB.PCI0.WLAN._PS0` is a *method* the kernel executes in an in-kernel
interpreter. The OS gets platform independence; the firmware gets responsibility.

The trade-off is severe and you will feel it:

- **The kernel ships a bytecode interpreter** (ACPICA, ~50k lines in `drivers/acpi/acpica/`)
  and runs vendor-supplied code in kernel context. Every AML bug is a kernel bug you cannot
  fix at the source.
- **Firmware quality is out of your control.** The kernel carries thousands of lines of
  quirks for broken AML (`drivers/acpi/`... `dmi_system_id` tables everywhere).
- **Debugging means decompiling someone else's binary.** `acpidump` + `iasl -d` is a routine
  part of laptop-driver development.
- Historically it was "designed to break Linux" in Linus's words (2003) — because the AML was
  only ever tested against one OS. This has improved but the structural issue remains.

**Neither model is right.** They are different placements of the same responsibility, and you
should be able to argue both sides in a design review.

### T.2 The tables: a linked structure rooted in memory

```
RSDP (found by scanning low memory / EFI config table)
  └─▶ XSDT (or RSDT on 32-bit) — an array of pointers to every other table
        ├─ FADT  — fixed hardware description; points to DSDT and FACS
        │    └─▶ DSDT — ★ the main namespace, in AML
        ├─ SSDT  — additional namespace fragments, in AML (often several)
        ├─ MADT  — interrupt controllers (APIC/GIC), CPUs
        ├─ MCFG  — PCIe ECAM base addresses
        ├─ HPET, SRAT, SLIT  — timers, NUMA topology and distances
        ├─ GTDT  — ARM generic timer (arm64)
        ├─ IORT  — ★ ARM I/O topology: SMMUs, ITS, device→stream-ID mapping
        ├─ PPTT  — ARM processor topology (caches, clusters)
        ├─ SPCR  — serial console (this is how `earlycon` works on ACPI arm64)
        ├─ DBG2, BGRT, TPM2, WSMT, HMAT, CEDT (CXL), PPTT, ERST/BERT/HEST (RAS)
        └─ ...
```

```bash
ls /sys/firmware/acpi/tables/
ls /sys/firmware/acpi/tables/dynamic/          # SSDTs loaded at runtime
sudo cat /sys/firmware/acpi/tables/DSDT > /tmp/dsdt.dat
dmesg | grep -i 'ACPI:' | head -30
```

Two tables deserve special mention because they solve problems DT solves structurally:

- **MCFG** gives the PCIe ECAM base, so PCI enumeration works without any per-platform
  knowledge. (DT expresses the same thing with a `pcie@` node's `reg` and `ranges`.)
- **IORT** (ARM) maps each device to its SMMU and ITS — the ACPI equivalent of DT's
  `iommus = <&smmu 0x100>` phandles. Without it, DMA on ARM servers would not work.

### T.3 The namespace: a tree, like DT, with methods

```
\
├── _SB              System Bus — where devices live
│    ├── PCI0        the PCI host bridge
│    │    ├── _HID = "PNP0A08"        hardware ID
│    │    ├── _CRS                     current resources (a METHOD)
│    │    └── LPCB
│    │         └── EC0
│    │              ├── _HID = "PNP0C09"
│    │              ├── _CRS
│    │              ├── _GPE = 0x17
│    │              └── _Q80          an event handler method
│    └── I2C1
│         └── TPAD
│              ├── _HID = "ELAN0501"
│              ├── _CRS               I²C address + GPIO interrupt
│              ├── _DSD               ★ device properties (T.5)
│              └── _STA               status (a METHOD)
├── _PR / _SB.CPUs   processors
├── _GPE             general purpose event handlers
├── _SI, _TZ         system indicators, thermal zones
└── _OSI / _OS       "which OS am I running on?" — the source of much pain
```

The structural parallel to DT is exact: a tree of named objects with properties. The
difference is that **many "properties" are methods** that the kernel must *execute*:

| Object | Meaning |
|---|---|
| `_HID` | primary hardware ID → the matching key (like DT `compatible`) |
| `_CID` | compatible ID(s) → the fallback list (exactly DT's ordered `compatible`) |
| `_UID` | unique id, to distinguish instances |
| `_STA` | **method**: is this device present/enabled/functioning? |
| `_CRS` | **method**: current resource settings (addresses, IRQs, GPIOs, I²C/SPI slaves) |
| `_PRS`/`_SRS` | possible/set resources (legacy ISA-style reconfiguration) |
| `_DSD` | ★ device-specific data — the DT-property bridge (T.5) |
| `_DSM` | **method**: device-specific method, dispatched by UUID + function index |
| `_PS0`.._PS3` | **methods**: enter power state D0..D3 |
| `_PR0`/`_PR3` | power resource dependencies |
| `_DEP` | ★ explicit dependency on another device (ACPI's `-EPROBE_DEFER` hint) |
| `_ADR` | address on the parent bus (PCI device/function, I²C address) |

```bash
ls /sys/bus/acpi/devices/ | head -20
cat /sys/bus/acpi/devices/*/hid 2>/dev/null | sort | uniq -c | sort -rn | head
cat /sys/bus/acpi/devices/*/status 2>/dev/null | head
# The namespace, rendered:
sudo cat /sys/kernel/debug/acpi/... 2>/dev/null   # if CONFIG_ACPI_DEBUGGER
```

### T.4 `_CRS`: resources as a returned buffer, not a static property

Where DT writes `reg = <0x0 0x1c020000 0x0 0x1000>`, ACPI *computes* resources:

```asl
Device (UART) {
	Name (_HID, "PNP0501")
	Name (_UID, 1)
	Method (_STA, 0) { Return (0x0F) }        /* present, enabled, functioning */
	Method (_CRS, 0) {
		Return (ResourceTemplate () {
			Memory32Fixed (ReadWrite, 0x1C020000, 0x1000)
			Interrupt (ResourceConsumer, Level, ActiveHigh, Exclusive) { 32 }
		})
	}
}
```

Because `_CRS` is a **method**, resources can depend on runtime state — a dock being present,
a jumper, an EC query. That expressiveness is exactly what DT cannot do and exactly what
makes ACPI platforms hard to reason about statically.

The kernel converts `_CRS` into the same `struct resource` array a DT platform device gets
(Ch. 31 T.3), so `platform_get_resource()` and `devm_platform_ioremap_resource()` work
unchanged. **This is the first half of the unification.**

For non-memory-mapped buses, `_CRS` carries bus-specific connectors:

```asl
Method (_CRS, 0) {
	Return (ResourceTemplate () {
		I2cSerialBusV2 (0x0015, ControllerInitiated, 400000,
				AddressingMode7Bit, "\\_SB.PCI0.I2C1", 0x00,
				ResourceConsumer,,)
		GpioInt (Edge, ActiveLow, ExclusiveAndWake, PullUp, 0x0000,
			 "\\_SB.PCI0.GPI0", 0x00, ResourceConsumer,,) { 0x005C }
		GpioIo (Exclusive, PullDefault, 0x0000, 0x0000, IoRestrictionOutputOnly,
			"\\_SB.PCI0.GPI0", 0x00, ResourceConsumer,,) { 0x0041 }
	})
}
```

The I²C core walks `_CRS` of every child of an I²C controller and instantiates `i2c_client`s
— precisely the role `of_i2c_register_devices()` plays for DT. Same for SPI and UART
(`serdev`).

### T.5 `_DSD`: the bridge that unified DT and ACPI

Here is the problem that existed until ~2014: DT bindings had evolved rich, expressive,
*documented* property vocabularies (`acme,speed-hz`, `interrupts`, `clocks`, `-names`
conventions). ACPI had `_HID`, `_CRS`, and vendor-specific `_DSM` methods with no shared
vocabulary at all. A driver had to be written twice.

**`_DSD` (Device Specific Data, ACPI 5.1)** fixes this by defining a standard way to attach
**arbitrary named properties** to an ACPI device, using a well-known UUID:

```asl
Device (WIDG) {
	Name (_HID, "ACME0002")
	Name (_DSD, Package () {
		ToUUID ("daffd814-6eba-4d8c-8a91-bc9bbf4aa301"),   /* ★ "device properties" */
		Package () {
			Package () { "acme,speed-hz", 200000 },
			Package () { "acme,inverted", 1 },
			Package () { "label", "front-panel" },
			Package () { "enable-gpios", Package () { ^WIDG, 0, 0, 0 } },
		}
	})
}
```

The kernel parses this into the **same `fwnode` property model** as DT. So:

```c
device_property_read_u32(dev, "acme,speed-hz", &speed);     /* works on BOTH */
device_property_read_bool(dev, "acme,inverted");
devm_gpiod_get(dev, "enable", GPIOD_OUT_LOW);               /* works on BOTH */
```

**This is why Ch. 31 T.4's discipline pays off.** A driver written against
`device_property_*` and the subsystem `devm_*_get()` helpers works, unmodified, on a DT SoC
and on an ACPI server. `of_property_read_u32()` does not.

There is a second UUID for **hierarchical properties** (child "nodes" inside `_DSD`), which
gives ACPI the equivalent of DT child nodes — used for multi-channel devices, LEDs, and
camera sensor ports.

```bash
# Find devices with _DSD:
for d in /sys/bus/acpi/devices/*/; do
  [ -d "$d/physical_node" ] && ls "$d" | grep -q . && true
done
sudo grep -a -c '_DSD' /sys/firmware/acpi/tables/DSDT
# Decompile and look (T.9):
sudo acpidump -b -n DSDT -o /tmp/dsdt.dat && iasl -d /tmp/dsdt.dat && grep -A15 '_DSD' /tmp/dsdt.dsl | head -40
```

### T.6 Matching, and the ACPI probe path

```c
static const struct acpi_device_id my_acpi_ids[] = {
	{ "ACME0002", (kernel_ulong_t)&widget_v2 },
	{ "ACME0001", (kernel_ulong_t)&widget_v1 },
	{ }
};
MODULE_DEVICE_TABLE(acpi, my_acpi_ids);

static struct platform_driver my_driver = {
	.driver = {
		.name             = "acme-widget",
		.of_match_table   = my_of_match,      /* DT */
		.acpi_match_table = my_acpi_ids,      /* ACPI — same driver */
	},
	.probe = my_probe,
};
```

`acpi_match_device()` compares against `_HID` then each `_CID`, in order — the same
most-specific-first semantics as DT's `compatible` (Ch. 32 T.3).

The PNP-ID namespace has two forms: legacy `PNP0501`-style (a 3-letter vendor code assigned
by UEFI Forum) and the newer `ACPI0007`/`VEND1234` ACPI IDs. `_HID` strings are **ABI** in
the same sense DT compatibles are.

`acpi_scan()` walks the namespace at boot, creates a `struct acpi_device` for each node with
`_HID`/`_CID`, evaluates `_STA` to decide whether it exists, and then creates a *physical*
device — a `platform_device`, `i2c_client`, `spi_device`, or `pci_dev` depending on context.
`/sys/bus/acpi/devices/*/physical_node` is the symlink between the two.

**`_DEP` is ACPI's dependency hint**: it names devices that must be ready first. The kernel
uses it to defer probing, giving ACPI a coarse equivalent of DT's phandle-derived
`fw_devlink` graph (Ch. 27 T.5).

```bash
ls -l /sys/bus/acpi/devices/*/physical_node 2>/dev/null | head
cat /sys/bus/acpi/devices/*/path 2>/dev/null | head
cat /sys/kernel/debug/devices_deferred
```

### T.7 Power management: the firmware does it

This is where ACPI's philosophy shows most clearly:

```asl
Device (WLAN) {
	Name (_HID, "VEND1234")
	Method (_PS0, 0) {                    /* enter D0 (on) */
		Store (1, \_SB.GPIO.PWR)      /* the firmware knows the sequence */
		Sleep (10)
		Store (1, \_SB.GPIO.RST)
		Sleep (50)
	}
	Method (_PS3, 0) { ... }              /* enter D3 (off) */
	Name (_PR0, Package () { \_SB.PWRR }) /* power resources needed for D0 */
}
```

The driver calls `acpi_device_set_power(adev, ACPI_STATE_D0)` — or, more usually, just uses
runtime PM (Ch. 48) and lets the ACPI PM domain do it. **The driver never learns the
sequence.** Contrast the DT model, where the driver explicitly does
`regulator_enable(); gpiod_set_value(); usleep_range(); reset_control_deassert();`.

Global sleep states (`S0`/`S0ix`/`S3`/`S4`/`S5`) are likewise orchestrated by AML methods
(`_PTS`, `_WAK`, `_TTS`), which is why suspend bugs on laptops so often trace to firmware.

```bash
cat /sys/power/state /sys/power/mem_sleep
cat /sys/bus/acpi/devices/*/power_state 2>/dev/null | sort | uniq -c
cat /sys/firmware/acpi/interrupts/* | head
sudo dmesg | grep -iE 'ACPI.*(PS0|PS3|_DSM|power resource)' | head
```

### T.8 `_DSM` and `_OSI`: the two escape hatches (and two problem sources)

**`_DSM`** is a general-purpose vendor extension: a method dispatched by a UUID plus a
function index, returning arbitrary data.

```c
union acpi_object *obj;
guid_t guid;

guid_parse("12345678-1234-1234-1234-123456789abc", &guid);
obj = acpi_evaluate_dsm(ACPI_HANDLE(dev), &guid, 1 /*rev*/, 3 /*func*/, NULL);
if (obj) {
	/* obj->type: ACPI_TYPE_INTEGER / BUFFER / PACKAGE ... */
	ACPI_FREE(obj);
}
```

It is how NVMe gets power-state hints, how Thunderbolt does connection management, how GPUs
do hybrid-graphics muxing (`_DSM` with the "optimus" UUID), and how a hundred laptop quirks
are implemented. It is powerful and completely undiscoverable — you learn the UUIDs from
vendor documentation, from Windows drivers, or by decompiling the DSDT.

**`_OSI`** is worse, and worth knowing as a cautionary tale. It lets AML ask *"which OS are
you?"*:

```asl
If (_OSI ("Windows 2015")) { /* one code path */ }
Else                        { /* another, often untested */ }
```

Linux originally answered truthfully (`_OSI("Linux")`), and firmware vendors used that to
select deliberately degraded paths. The kernel now **claims to be Windows** by default,
because the Windows path is the only one that was tested. This is a genuine,
still-current example of Hyrum's Law (Ch. 24 T.1) operating across an organizational
boundary: the *ability to distinguish* created an incentive to discriminate.

```bash
dmesg | grep -i '_OSI\|ACPI: Added _OSI'
cat /sys/module/acpi/parameters/* 2>/dev/null
# boot params: acpi_osi=! acpi_osi="Windows 2020"  acpi_osi=Linux
```

### T.9 Debugging: you will decompile firmware

Because the platform's behaviour is in a binary you did not write, ACPI debugging has a
distinct workflow that has no DT equivalent:

```bash
sudo apt install -y acpica-tools           # acpidump, iasl, acpixtract
sudo acpidump > /tmp/acpi.dat
acpixtract -a /tmp/acpi.dat                # splits into dsdt.dat, ssdt1.dat, ...
iasl -d dsdt.dat                           # → dsdt.dsl, human-readable ASL
less dsdt.dsl
```

**Table override** lets you *patch the firmware* without a BIOS update — the key technique
for laptop bring-up:

```bash
# 1. Decompile, edit, recompile
iasl -d dsdt.dat && $EDITOR dsdt.dsl && iasl -tc dsdt.dsl

# 2a. Build it into the kernel
./scripts/config -e ACPI_CUSTOM_DSDT --set-str ACPI_CUSTOM_DSDT_FILE dsdt.hex

# 2b. Or load an SSDT overlay at runtime (much better — additive, no full replace)
cat > /tmp/fix.asl <<'EOF'
DefinitionBlock ("", "SSDT", 2, "MYORG", "FIXUP", 0x00000001) {
    External (\_SB.PCI0.I2C1.TPAD, DeviceObj)
    Scope (\_SB.PCI0.I2C1.TPAD) {
        Name (_DSD, Package () {
            ToUUID ("daffd814-6eba-4d8c-8a91-bc9bbf4aa301"),
            Package () { Package () { "acme,fixed-property", 1 } }
        })
    }
}
EOF
iasl -tc /tmp/fix.asl
# load via initrd (acpi_override) or:
sudo mkdir -p /sys/kernel/config/acpi/table/fix
cat /tmp/fix.aml | sudo tee /sys/kernel/config/acpi/table/fix/aml >/dev/null

# 3. Runtime tracing and the AML debugger
echo 1 | sudo tee /sys/module/acpi/parameters/aml_debug_output
# boot with: acpi.debug_layer=0xffffffff acpi.debug_level=0x2
cat /sys/module/acpi/parameters/{debug_layer,debug_level}
./scripts/config -e ACPI_DEBUGGER -e ACPI_DEBUGGER_USER
sudo acpidbg                               # an interactive AML debugger
```

```bash
# Useful runtime views
sudo cat /sys/firmware/acpi/interrupts/sci      # System Control Interrupt count
sudo cat /sys/firmware/acpi/interrupts/gpe*     # per-GPE counts — find a storm
ls /sys/firmware/acpi/hotplug/
sudo acpi_listen                                 # ACPI events as they happen
sudo journalctl -k | grep -i 'ACPI Error\|ACPI Warning\|AE_'
```

An **AML method that fails** produces `ACPI Error: ... AE_NOT_FOUND` in dmesg. Those messages
are usually firmware bugs, not kernel bugs, and the fix is a DMI-matched quirk or an SSDT
override — which is why `drivers/acpi/` and `drivers/platform/x86/` are full of
`dmi_system_id` tables.

---

## 1. Internals

### 1.1 Source map

```
drivers/acpi/acpica/        ★ ACPICA — the AML interpreter; imported from Intel, DO NOT
                              modify locally; fixes go upstream to acpica.org first
drivers/acpi/scan.c         ★★ namespace walk → acpi_device → platform/i2c/spi device
drivers/acpi/bus.c          acpi_bus_type, matching, _STA/_DEP handling
drivers/acpi/resource.c     ★ _CRS → struct resource (T.4)
drivers/acpi/property.c     ★★ _DSD → fwnode properties (T.5) — read this
drivers/acpi/device_pm.c    _PSx, power resources (T.7)
drivers/acpi/utils.c        acpi_evaluate_dsm and friends
drivers/acpi/pci_root.c, pci_irq.c
drivers/acpi/arm64/         IORT, GTDT, AGDI
drivers/acpi/tables.c       early table parsing
drivers/acpi/osl.c          the OS services layer ACPICA calls back into
include/linux/acpi.h, include/acpi/
drivers/platform/x86/       ★ where laptop quirks live; read one for flavour
Documentation/firmware-guide/acpi/   ★★ the whole directory
```

### 1.2 The firmware-agnostic driver, one more time

```c
/* Works on DT, ACPI, and software nodes without a single #ifdef */
static int my_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	const struct my_data *cfg = device_get_match_data(dev);   /* DT .data or ACPI .driver_data */
	u32 speed;

	if (device_property_read_u32(dev, "acme,speed-hz", &speed))
		speed = cfg->default_speed;

	priv->base  = devm_platform_ioremap_resource(pdev, 0);    /* DT reg  / ACPI _CRS */
	priv->irq   = platform_get_irq(pdev, 0);                  /* DT irq  / ACPI _CRS */
	priv->gpiod = devm_gpiod_get(dev, "enable", GPIOD_OUT_LOW); /* DT *-gpios / ACPI _CRS+_DSD */
	priv->clk   = devm_clk_get_optional_enabled(dev, NULL);   /* DT only, usually */
	...
}
```

Note the asymmetry in the last line: **ACPI has no clock or regulator model.** Those are
handled by AML inside `_PS0`/`_PS3`, so an ACPI-only driver simply does not call them
(`devm_clk_get_optional*` returns NULL, which is why the `_optional` variants exist). That
asymmetry is the main practical difference when writing dual-firmware drivers.

---

## 2. Practice

### Lab 33.1 — Dump, decompile, and read your own firmware

```bash
sudo apt install -y acpica-tools
sudo acpidump > /tmp/acpi.dat
mkdir -p /tmp/acpi && cd /tmp/acpi && acpixtract -a /tmp/acpi.dat
ls -la                                 # every table as a separate .dat
for t in *.dat; do iasl -d "$t" >/dev/null 2>&1; done
wc -l *.dsl | sort -rn | head

# Find the interesting bits:
grep -n 'Device (' dsdt.dsl | head -40
grep -n '_HID' dsdt.dsl | head -30
grep -n -A20 '_DSD' dsdt.dsl | head -60
grep -n -A25 'Method (_CRS' dsdt.dsl | head -50
grep -n '_OSI' dsdt.dsl | head
grep -n '_DSM' dsdt.dsl | head

# Pick ONE device and read its complete definition:
awk '/Device \(TPAD\)/,/^    }/' dsdt.dsl
```
**Deliverable:** for one device on your machine, write out its `_HID`, `_CRS` contents, `_STA`
logic, `_DSD` properties (if any), and power methods. Then find the Linux driver that binds
it and show which of those the driver actually consumes.

### Lab 33.2 — Trace a device from AML to `struct device`

```bash
# Pick a device with a physical node
for d in /sys/bus/acpi/devices/*/; do
  [ -e "$d/physical_node" ] && echo "$(basename $d): $(cat $d/hid) -> $(readlink -f $d/physical_node)"
done | head -20

D=/sys/bus/acpi/devices/PNP0C09:00        # e.g. the embedded controller
cat $D/hid $D/path $D/status $D/uid 2>/dev/null
ls $D/
readlink -f $D/physical_node
cat $D/physical_node/modalias 2>/dev/null

# The resources it got from _CRS:
cat /proc/iomem | grep -i -A1 -B1 "$(basename $(readlink -f $D/physical_node))"
ls /sys/bus/platform/devices/*/ 2>/dev/null | head

# Trace the scan:
dmesg | grep -i 'ACPI: \(Added\|Enumeration\|bus type\)' | head -20
sudo bpftrace -e 'kprobe:acpi_device_add { @ = count(); }'
```

### Lab 33.3 — Write a driver that binds on both DT and ACPI

Extend Ch. 31 Lab 31.1's `acme-widget` with an ACPI match table, then bind it on a virtual
ACPI device.

```c
static const struct acpi_device_id widget_acpi_match[] = {
	{ "ACME0002", (kernel_ulong_t)&widget_v2 },
	{ "ACME0001", (kernel_ulong_t)&widget_v1 },
	{ }
};
MODULE_DEVICE_TABLE(acpi, widget_acpi_match);

static struct platform_driver widget_driver = {
	.driver = {
		.name             = "acme-widget",
		.of_match_table   = widget_of_match,
		.acpi_match_table = widget_acpi_match,
		.dev_groups       = widget_groups,
	},
	.probe = widget_probe,      /* ★ UNCHANGED */
};
```

Create the ACPI device with an SSDT overlay:

```asl
DefinitionBlock ("", "SSDT", 2, "LAB", "WIDGET", 0x00000001)
{
    External (\_SB, DeviceObj)
    Scope (\_SB)
    {
        Device (WIDG)
        {
            Name (_HID, "ACME0002")
            Name (_UID, 1)
            Method (_STA, 0, NotSerialized) { Return (0x0F) }
            Method (_CRS, 0, NotSerialized)
            {
                Return (ResourceTemplate ()
                {
                    Memory32Fixed (ReadWrite, 0xFED10000, 0x1000)
                    Interrupt (ResourceConsumer, Level, ActiveHigh, Exclusive) { 42 }
                })
            }
            Name (_DSD, Package ()
            {
                ToUUID ("daffd814-6eba-4d8c-8a91-bc9bbf4aa301"),
                Package ()
                {
                    Package () { "acme,speed-hz", 200000 },
                    Package () { "acme,inverted", 1 },
                }
            })
        }
    }
}
```
```bash
iasl -tc widget.asl                       # → widget.aml
./scripts/config -e ACPI_CONFIGFS
sudo mount -t configfs none /sys/kernel/config 2>/dev/null
sudo mkdir -p /sys/kernel/config/acpi/table/widget
cat widget.aml | sudo tee /sys/kernel/config/acpi/table/widget/aml >/dev/null

ls /sys/bus/acpi/devices/ | grep ACME
sudo insmod acme-widget.ko
dmesg | tail -10
cat /sys/bus/platform/devices/ACME0002:00/speed
```
**The driver source did not change between DT and ACPI.** That is T.5's payoff, demonstrated.

### Lab 33.4 — QEMU with a custom ACPI table

```bash
# QEMU can inject arbitrary tables:
qemu-system-x86_64 -M q35 -m 2G -enable-kvm \
  -acpitable file=widget.aml \
  -kernel bzImage -initrd initramfs.cpio.gz \
  -append "console=ttyS0 rdinit=/init acpi.debug_level=0x2" -nographic

# Or add a whole SSDT:
qemu-system-x86_64 ... -acpitable sig=SSDT,file=widget.aml

# Inside the guest:
ls /sys/firmware/acpi/tables/
sudo acpidump -b -n SSDT && iasl -d ssdt.dat && cat ssdt.dsl
```
This gives you a reproducible ACPI development environment without a physical machine — the
ACPI equivalent of Ch. 04's QEMU loop.

### Lab 33.5 — Evaluate an `_DSM` from a driver

```c
#include <linux/acpi.h>

static int my_call_dsm(struct device *dev)
{
	acpi_handle handle = ACPI_HANDLE(dev);
	union acpi_object *obj;
	guid_t guid;
	int ret = 0;

	if (!handle)
		return -ENODEV;

	guid_parse("12345678-1234-1234-1234-123456789abc", &guid);

	/* Function 0 is always "which functions do you support?" — a bitmap */
	obj = acpi_evaluate_dsm_typed(handle, &guid, 1, 0, NULL, ACPI_TYPE_BUFFER);
	if (!obj)
		return -ENODEV;
	dev_info(dev, "_DSM supported functions bitmap: 0x%02x\n", obj->buffer.pointer[0]);
	ACPI_FREE(obj);

	/* Now call function 3 with an integer argument */
	{
		union acpi_object arg = { .integer = { ACPI_TYPE_INTEGER, 1 } };

		obj = acpi_evaluate_dsm(handle, &guid, 1, 3, &arg);
		if (obj) {
			if (obj->type == ACPI_TYPE_INTEGER)
				dev_info(dev, "result %llu\n", obj->integer.value);
			ACPI_FREE(obj);
		} else {
			ret = -EIO;
		}
	}
	return ret;
}
```
```bash
# Find real _DSM users to study:
git grep -n 'acpi_evaluate_dsm' -- drivers/ | head -20
git grep -n 'guid_parse' -- drivers/nvme/ drivers/thunderbolt/ drivers/gpu/ | head
grep -n -B5 -A30 '_DSM' /tmp/acpi/dsdt.dsl | head -60
```

### Lab 33.6 — Diagnose a firmware problem

```bash
# 1. AML errors (almost always firmware bugs)
sudo journalctl -k | grep -E 'ACPI (Error|Warning|BIOS Error)|AE_' | head -20

# 2. GPE storms (a common laptop battery-drain / wakeup bug)
watch -n1 'sudo cat /sys/firmware/acpi/interrupts/* | paste -sd" "'
sudo grep -H . /sys/firmware/acpi/interrupts/gpe* | grep -v ' 0 ' | head
# Mask a storming GPE:
echo disable | sudo tee /sys/firmware/acpi/interrupts/gpe17

# 3. Wakeup sources
cat /proc/acpi/wakeup
echo LID0 | sudo tee /proc/acpi/wakeup       # toggle

# 4. The DMI quirk mechanism — how the kernel works around bad firmware
sudo dmidecode -t system -t bios | head -20
cat /sys/class/dmi/id/{sys_vendor,product_name,bios_version,bios_date}
git grep -n 'dmi_system_id' -- drivers/acpi/ drivers/platform/x86/ | wc -l
git grep -n -A10 'dmi_system_id.*quirk' drivers/platform/x86/thinkpad_acpi.c | head -30

# 5. The nuclear options (for bisecting a firmware issue)
#    acpi=off  acpi=noirq  pci=noacpi  acpi_osi=!  acpi_enforce_resources=lax
#    acpi_backlight=vendor  noapic  nolapic
```

### Lab 33.7 — Compare DT and ACPI for the same device

Take one device and write both descriptions:

| | DT | ACPI |
|---|---|---|
| Identity | `compatible = "acme,widget-v2";` | `Name (_HID, "ACME0002")` |
| Registers | `reg = <0x10000000 0x1000>;` | `Memory32Fixed (ReadWrite, 0x10000000, 0x1000)` in `_CRS` |
| IRQ | `interrupts = <0 42 4>;` | `Interrupt (...) { 42 }` in `_CRS` |
| Property | `acme,speed-hz = <200000>;` | `_DSD` package entry |
| GPIO | `enable-gpios = <&gpio 5 0>;` | `GpioIo (...)` in `_CRS` + `_DSD` name mapping |
| Clock | `clocks = <&ccu 10>;` | **no equivalent** — hidden in `_PS0` |
| Power sequence | driver code | `_PS0`/`_PS3` methods |
| Dependency | `clocks = <&ccu ...>` → fw_devlink | `_DEP` |

**Write both, boot both, and confirm the same driver binds.** Then write 400 words on which
model you would choose for (a) a new SoC, (b) a server platform, (c) a product that must
support firmware updates without kernel updates.

---

## 3. Mastery drills

1. **Read `drivers/acpi/property.c`** completely. Explain exactly how `_DSD` packages become
   `fwnode` properties, including the hierarchical-properties UUID and how `*-gpios` name
   mapping works.

2. **Read `drivers/acpi/scan.c`**'s `acpi_bus_scan()`. Trace a namespace node to a
   `platform_device`. Where is `_STA` evaluated, and what happens if it returns 0?

3. **The interpreter question.** Write 500 words arguing for and against shipping an AML
   interpreter in the kernel. Cover: attack surface, firmware quality, platform
   independence, and what the alternative (DT) costs. Cite the `_OSI` history.

4. **`_CRS` expressiveness.** Find a `_CRS` in your DSDT that is *not* a simple
   `ResourceTemplate` return — i.e. one with conditionals. Explain what runtime state it
   depends on and why DT could not express it.

5. **IORT.** On an ARM server (or QEMU `virt` with `-machine acpi=on`), dump and decompile
   IORT. Explain how a PCIe device's stream ID reaches its SMMU, and compare with DT's
   `iommu-map` property.

6. **Quirk archaeology.** Pick a `dmi_system_id` table in `drivers/platform/x86/`. For three
   entries, find the bug report or commit that added it. What firmware behaviour was being
   worked around?

7. **`_OSI` and Hyrum's Law.** Explain the `_OSI` situation in terms of Ch. 24 T.1. What
   would a better design have been? (Consider: feature queries instead of identity queries.)

8. **Dual-firmware audit.** Find a driver with both `of_match_table` and `acpi_match_table`.
   Determine whether it has *any* firmware-specific code paths, and if so whether they could
   be eliminated.

9. **SSDT override in production.** Design a process for shipping an SSDT override with a
   product: where does it live (initrd? kernel? EFI?), how is it versioned, how do you verify
   it applied, and what happens on a BIOS update?

10. **ACPICA upstream.** Find the ACPICA release process (`drivers/acpi/acpica/README`).
    Explain why a bug in `drivers/acpi/acpica/` must be fixed upstream at acpica.org first,
    and what that implies for your fix's timeline.

---

## 4. Further reading

**Specifications:**
- **ACPI Specification 6.5+** — https://uefi.org/specifications ★★ (huge; read §5 Tables,
  §6 Device Configuration, §7 Power Management, and Appendix A)
- **ARM SBSA/SBBR** (Server Base System/Boot Requirements) — what ARM servers must provide
- UEFI Specification — the table-discovery mechanism

**Kernel documentation:**
- `Documentation/firmware-guide/acpi/` ★★ — **the whole directory**, especially
  `enumeration.rst`, `dsd/`, `gpio-properties.rst`, `i2c-muxes.rst`, `acpi-lid.rst`,
  `method-customizing.rst`, `method-tracing.rst`, `debug.rst`, `ssdt-overlays.rst`,
  `chromeos-acpi-device.rst`
- `Documentation/admin-guide/acpi/` — `initrd_table_override.rst`, `dsdt-override.rst`,
  `cppc_sysfs.rst`
- `Documentation/admin-guide/kernel-parameters.txt` — every `acpi*=` option
- `Documentation/arch/arm64/acpi_object_usage.rst` ★ — which ACPI objects ARM64 supports

**Source:**
- `drivers/acpi/property.c` ★★ (T.5)
- `drivers/acpi/scan.c`, `drivers/acpi/resource.c`
- `drivers/acpi/arm64/iort.c`
- `drivers/platform/x86/thinkpad_acpi.c` — a masterclass in firmware-quirk handling
- `drivers/acpi/acpica/README` — the upstream relationship

**Tools:**
- `acpica-tools`: `acpidump`, `acpixtract`, `iasl`, `acpiexec`, `acpibin`
- `acpi_listen`, `acpitool`, `fwts` (**Firmware Test Suite** — Canonical's ACPI validator;
  run it on any new platform)
- `dmidecode`, `efivar`, `chipsec`

**Articles & history:**
- Linus's 2003 "ACPI is a complete design disaster" post — context for T.1
- Matthew Garrett's blog and talks on ACPI, `_OSI`, and firmware quality ★ — the best
  practitioner writing on this subject
- LWN: "ACPI, Linux, and the _OSI mess", "Device properties and ACPI",
  "ACPI on ARM64", "SSDT overlays for hardware enablement"
- Bootlin / Linaro talks comparing DT and ACPI on ARM

→ Next: [34-mmio-regmap.md](34-mmio-regmap.md)

# Chapter 37 — PCI and PCIe: Enumeration, Configuration Space, BARs, Capabilities

> **Goal:** understand the canonical self-describing bus. Be able to explain how the kernel
> discovers every device in a machine from nothing, how a BAR reports its own size, and how
> PCIe kept twenty-five years of software compatible while replacing the physical layer
> entirely.

---

## Theory & First Principles

### T.0 — Start here: how does the kernel know what is plugged in?

Ch. 31 established the divide: SoC peripherals must be described, PCI devices describe
themselves. How?

**Every PCI device must implement a 256-byte configuration space at a known location**, and
the first four bytes are the whole trick:

```
 offset 0x00: [ Vendor ID ][ Device ID ]     <- 0xFFFF vendor = NOBODY HOME
        0x04: [ Command   ][ Status    ]
        0x08: [Rev][ Class Code        ]     <- "I am a network controller"
        0x10: [ BAR0 ]  Base Address Register  <- WHERE my registers should go
        0x14: [ BAR1 ]
        ...
        0x34: [ Cap Ptr ]  -> a linked list of CAPABILITIES (MSI, MSI-X, PCIe, ...)
```

Enumeration is then mechanical, and this is the entire discovery algorithm:

```
 for bus in 0..255, device in 0..31, function in 0..7:
     vendor = config_read16(bus, dev, fn, 0x00)
     if vendor == 0xFFFF: continue          # nothing there
     -> a device exists. Read its IDs, class, and BARs.
     -> if it is a BRIDGE (class 0x06), recurse into the bus behind it.
```

**Three consequences follow immediately**, and they are why PCI won:

1. **The kernel needs no prior knowledge of the machine.** One binary enumerates any PCI
   system. This is the thing device tree cannot do and it is why x86 never needed board
   files (Ch. 31 §T.0).
2. **Driver matching is a table lookup**, not a guess:
   ```c
   static const struct pci_device_id my_ids[] = {
	{ PCI_DEVICE(0x8086, 0x15bb) },
	{ PCI_DEVICE_CLASS(PCI_CLASS_STORAGE_EXPRESS, ~0) },  /* ANY NVMe device */
	{ }
   };
   MODULE_DEVICE_TABLE(pci, my_ids);   /* -> depmod -> autoload on hotplug */
   ```
   The `CLASS` form is why one `nvme` driver binds every NVMe device ever made — the
   *class code* is a standardized interface, so the device is described by what it *is*
   rather than by who made it.
3. **The BARs are *requests*, not addresses.** A device says "I need 64 KiB of memory space";
   firmware or the kernel decides *where*. That indirection is what makes hotplug and
   resource rebalancing possible.

**Now the sequence every PCI driver must perform**, and the order is not negotiable:

```c
pci_enable_device(pdev);          /* power up, enable decoding -- BEFORE any access */
pci_request_regions(pdev, name);  /* claim the BARs; fails if someone else has them */
base = pci_iomap(pdev, 0, 0);     /* map BAR 0 -- now MMIO (Ch. 34) works */
dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64));  /* BEFORE any DMA alloc */
pci_set_master(pdev);             /* allow the device to initiate DMA -- Ch. 36 */
pci_alloc_irq_vectors(...);       /* MSI/MSI-X -- Ch. 38 */
```

Each line is a hardware state change with a precondition. `pci_set_master` in particular is
the moment a device becomes able to write your memory — **do not enable it before the IOMMU
mappings and the rings are ready**, and disable it first on teardown (Ch. 25 P12).

```bash
lspci -nn                             # vendor:device IDs
lspci -vv -s 00:1f.6                  # BARs, capabilities, link state
lspci -t                              # the TOPOLOGY -- bridges and what is behind them
sudo lspci -xxx -s 00:1f.6 | head -4  # raw config space; byte 0-3 is the ID
ls /sys/bus/pci/devices/*/            # resource, config, enable, driver...
```

---

### T.1 PCI is the answer to "how can hardware describe itself?"

Ch. 31 T.1 drew the line between discoverable and non-discoverable buses. PCI (1992) is the
archetype of the discoverable side, and its design decisions are worth studying because
almost every later bus copied them.

The problem PCI solved: in the ISA era, installing a card meant setting DIP switches for its
I/O port, IRQ, and DMA channel, then telling the driver those values. Conflicts were common
and diagnosis was manual. PCI's answer has three parts:

1. **A separate configuration address space**, accessible before the device has any assigned
   resources. This is the bootstrap: you must be able to talk to a device to find out where
   it wants to live.
2. **A standardized 256-byte header** in that space that *every* device implements
   identically — vendor ID, device ID, class code, and resource requests.
3. **Self-describing resource requirements** (BARs, T.4) so the OS can allocate addresses
   rather than the user.

The result is that an OS can enumerate an arbitrary machine with **no prior knowledge**, which
is exactly what a device tree provides by other means for non-discoverable hardware.

```bash
lspci -nn | head -20
lspci -tv | head -30                 # the topology
sudo lspci -xxx -s 00:1f.2 | head    # the raw config space
```

### T.2 Configuration space: the bootstrap address space

Every PCI function has a config space addressed by **BDF**: Bus (8 bits), Device (5 bits),
Function (3 bits) — so 256 buses × 32 devices × 8 functions, and the familiar
`0000:03:00.0` notation is `domain:bus:device.function`.

```
Type 0 header (a normal device), offsets:
 0x00  Vendor ID        │  Device ID
 0x04  Command          │  Status
 0x08  Revision │ Class Code (prog-if, subclass, base class)
 0x0c  Cache Line │ Latency │ Header Type │ BIST
 0x10  BAR0
 0x14  BAR1
 0x18  BAR2
 0x1c  BAR3
 0x20  BAR4
 0x24  BAR5
 0x28  Cardbus CIS pointer
 0x2c  Subsystem Vendor ID │ Subsystem ID
 0x30  Expansion ROM base
 0x34  Capabilities pointer   ★ head of the capability linked list (T.5)
 0x3c  Interrupt Line │ Interrupt Pin │ Min_Gnt │ Max_Lat
 0x40..0xff  device-specific / capabilities
 0x100..0xfff  ★ PCIe EXTENDED capabilities (T.5)
```

**Header Type** distinguishes a device (type 0) from a **bridge** (type 1), whose header
replaces BARs 2–5 with bus-number and window registers — the key to enumeration (T.3).

Two access mechanisms, and the difference matters:

| Mechanism | How | Reach |
|---|---|---|
| **CF8/CFC** (legacy x86) | write BDF+offset to port 0xCF8, read/write 0xCFC | first **256 bytes** only |
| **ECAM / MMCONFIG** | a flat MMIO region: `base + (bus<<20 \| dev<<15 \| fn<<12 \| off)` | all **4096 bytes** |

ECAM is what makes PCIe extended capabilities reachable. Its base comes from the ACPI
**MCFG** table (Ch. 33 T.2) on x86/ARM-ACPI, or from a DT `pcie@` node's `reg` on embedded
platforms. `pci=nommconf` forces the legacy path — a useful bisection tool when extended
capabilities misbehave.

```bash
dmesg | grep -i 'MMCONFIG\|ECAM\|PCI: Using'
sudo cat /sys/firmware/acpi/tables/MCFG | xxd | head -5
ls /sys/bus/pci/devices/0000:00:00.0/config   # 256 or 4096 bytes?
stat -c %s /sys/bus/pci/devices/*/config | sort | uniq -c
```

The kernel's accessors:

```c
pci_read_config_byte(pdev, PCI_VENDOR_ID, &v8);
pci_read_config_word(pdev, PCI_DEVICE_ID, &v16);
pci_read_config_dword(pdev, PCI_CLASS_REVISION, &v32);
pci_write_config_word(pdev, PCI_COMMAND, cmd);
```
**Config accesses are slow** (~1 µs on the legacy path, still hundreds of ns via ECAM) and
**non-posted** — they are for setup, never for a datapath.

### T.3 Enumeration: a recursive depth-first scan

The algorithm is elegant and you should be able to reproduce it on a whiteboard:

```
scan_bus(bus):
    for device in 0..31:
        for function in 0..7:
            vendor = read16(bus, device, function, 0x00)
            if vendor == 0xFFFF: continue          # nothing there
            record the device
            if function == 0 and not (header_type & 0x80):
                break                              # single-function device
            if (header_type & 0x7f) == 1:          # it's a BRIDGE
                write(secondary_bus   = ++next_bus)
                write(subordinate_bus = 0xFF)      # temporarily maximal
                scan_bus(next_bus)                 # ★ recurse
                write(subordinate_bus = highest_bus_found)
```

Two details carry all the weight:

**(a) `0xFFFF` means "no device".** Reading config space at an unpopulated BDF returns all
ones because nothing drives the bus. This is the *entire* presence-detection mechanism — a
convention, not a protocol.

**(b) Bridge bus-number programming is why the scan must be depth-first.** A bridge forwards
transactions for a *contiguous range* of bus numbers `[secondary, subordinate]`. You cannot
know the subordinate number until you have scanned everything behind it. So you set it to
0xFF (accept everything), recurse, then narrow it. This is a classic
"optimistic-then-tighten" algorithm.

The same contiguity constraint applies to **address windows**, and that makes resource
assignment a genuine bin-packing problem (T.4).

```bash
# Watch the enumeration
dmesg | grep -E 'pci_bus|PCI: |pci 0000:' | head -40
dmesg | grep -i 'bus:.*secondary\|PCI: bridge'
lspci -t
# Every bus and its bridge windows:
sudo lspci -vv -s 00:1c.0 | grep -E 'Bus:|Memory behind|I/O behind|Prefetchable'
```

### T.4 BARs: self-describing resources

This is the cleverest piece of PCI and it deserves careful explanation.

A **Base Address Register** serves two purposes with one 32-bit register:

- When the OS *writes* to it, it sets the device's base address.
- When the OS *probes* it, the device reports its **size and type**.

The probing protocol exploits a physical fact: the device only implements the address bits it
needs. A device wanting 4 KiB implements bits 31..12 and hardwires 11..4 to zero.

```
 1. Save the original value.
 2. Write 0xFFFFFFFF to the BAR.
 3. Read it back. The device returns 1s only in the bits it IMPLEMENTS;
    the low bits it doesn't decode read back as 0.
 4. size = ~(value & ~0xF) + 1        ← the lowest set bit IS the size
 5. Restore the original value.
```

So a BAR reading back `0xFFFFF000` after step 3 means the low 12 bits are not decoded ⇒
**4 KiB**. One register, no extra protocol, no vendor table. That is self-description done
right.

The low bits encode the type:

```
 bit 0    : 0 = memory space, 1 = I/O space
 bits 2:1 : 00 = 32-bit, 10 = 64-bit (this BAR and the NEXT form one 64-bit address)
 bit 3    : 1 = prefetchable  ★
```

**Prefetchable** is a real semantic claim, not a hint: it asserts that reads have **no side
effects** and that the host may **merge writes and prefetch speculatively**. A framebuffer is
prefetchable; a register block with a read-to-clear FIFO is emphatically not. Getting this
wrong in hardware causes data loss; in software it determines whether you may use
`ioremap_wc()` (Ch. 34 T.2).

**The bridge-window packing problem.** A bridge forwards only *one* memory window, *one*
prefetchable window, and *one* I/O window, each contiguous and naturally aligned. So all
devices behind a bridge must fit in windows that are themselves aligned and sized as powers
of two. The kernel's `pci_assign_unassigned_resources()` solves this bin-packing problem —
and when it fails you see:

```
pci 0000:03:00.0: BAR 0: no space for [mem size 0x10000000]
pci 0000:03:00.0: BAR 0: failed to assign [mem size 0x10000000]
```
which is why "enable Above 4G Decoding" is standard advice for large-BAR GPUs: without it the
BIOS confines everything below 4 GiB and a 16 GiB BAR cannot be placed.

```bash
sudo lspci -vv -s 03:00.0 | grep -A2 'Region'
cat /sys/bus/pci/devices/0000:03:00.0/resource     # start end flags, one line per BAR
cat /proc/iomem | grep -A2 -i 'pci bus'
dmesg | grep -iE 'BAR .*: assigned|no space|failed to assign'
```

### T.5 Capabilities: extensibility by linked list

PCI's config header is fixed at 64 bytes, but hardware kept gaining features. The solution is
a **linked list** of capability structures in the 0x40–0xFF region:

```
 0x34 → Capabilities Pointer = 0x40
        0x40: [ID=0x01 (PM)]      [next=0x50]  ...
        0x50: [ID=0x05 (MSI)]     [next=0x70]  ...
        0x70: [ID=0x10 (PCIe)]    [next=0xB0]  ...
        0xB0: [ID=0x11 (MSI-X)]   [next=0x00]  ← end
```

PCIe adds **extended capabilities** at 0x100–0xFFF with a 16-bit ID space and the same
linked-list structure. This is an **extensibility mechanism** in the Ch. 24 T.3 sense:
new features are added without moving anything, and an OS that does not know a capability
simply skips it. Compare with a version field (which serializes evolution) — the linked list
allows independent, concurrent extension by different working groups.

```c
int pos = pci_find_capability(pdev, PCI_CAP_ID_EXP);       /* standard caps */
int pos = pci_find_ext_capability(pdev, PCI_EXT_CAP_ID_ERR); /* extended caps */
/* Usually you don't need the offset — the core caches what it needs: */
pcie_capability_read_word(pdev, PCI_EXP_DEVCTL, &ctl);
pcie_capability_set_word(pdev, PCI_EXP_DEVCTL, PCI_EXP_DEVCTL_RELAX_EN);
```

Capabilities you will meet: Power Management (0x01), MSI (0x05), PCIe (0x10), MSI-X (0x11),
Vendor-Specific (0x09); extended: AER (0x0001), VC (0x0002), Serial Number (0x0003),
SR-IOV (0x0010), ACS (0x000D), ATS (0x000F), PASID (0x001B), Resizable BAR (0x0015),
DOE (0x002E).

### T.6 PCIe: a completely different wire, deliberately identical software

PCIe (2003) replaced PCI's parallel, shared, arbitrated bus with **point-to-point serial
links carrying packets (TLPs)**. Physically nothing is the same: differential pairs, 8b/10b
or 128b/130b encoding, link training, lanes, credit-based flow control, a switch fabric
instead of a bus.

And yet: **the same enumeration algorithm, the same config space, the same BARs, the same
drivers.** Every PCIe link appears to software as a PCI-to-PCI bridge with one device behind
it. A 1992 PCI driver binds to a 2024 PCIe device unchanged.

This is one of the most successful compatibility engineering efforts in computing history,
and the lesson generalizes: **they kept the *programming model* and replaced the
*implementation*.** The software abstraction (self-describing config space + BARs) was
independent enough of the electrical reality to survive its complete replacement.

What PCIe *adds* to the software model, all as capabilities (T.5): MSI-X, AER, ASPM, SR-IOV,
ATS/PASID, resizable BARs, DOE. Ch. 38 covers these.

```bash
sudo lspci -vv -s 03:00.0 | grep -A8 'Express (v2) Endpoint'
sudo lspci -vv | grep -E 'LnkCap:|LnkSta:' | head
# Current vs maximum link speed and width — the #1 "why is my card slow" check:
cat /sys/bus/pci/devices/0000:03:00.0/{current_link_speed,current_link_width,max_link_speed,max_link_width}
```

### T.7 The Command register and bus mastering

```c
#define PCI_COMMAND_IO          0x1   /* respond to I/O space accesses  */
#define PCI_COMMAND_MEMORY      0x2   /* respond to memory space accesses */
#define PCI_COMMAND_MASTER      0x4   /* ★ may initiate transactions (DMA) */
#define PCI_COMMAND_INTX_DISABLE 0x400
```

Three facts that cause real bugs:

1. **A device does not respond to its BARs until `PCI_COMMAND_MEMORY` is set.** That is what
   `pci_enable_device()` does. MMIO before it reads as all-ones.
2. **A device cannot DMA until `PCI_COMMAND_MASTER` is set.** `pci_set_master()`. A driver
   that programs a DMA descriptor and rings a doorbell without it sees... nothing happen, with
   no error. This is a classic bring-up mistake.
3. **Clearing MASTER is how you quiesce a device**, which is what `pci_clear_master()` in
   `remove()`/`shutdown()` is for (Ch. 31 T.6 — kexec correctness).

### T.8 The `pci_driver` shape, modern form

```c
static const struct pci_device_id my_ids[] = {
	{ PCI_DEVICE(0x1234, 0x5678) },
	{ PCI_DEVICE_SUB(0x1234, 0x5678, 0xABCD, 0x0001) },      /* by subsystem */
	{ PCI_DEVICE_CLASS(PCI_CLASS_STORAGE_EXPRESS, 0xffffff) },/* by class */
	{ PCI_DEVICE_DATA(INTEL, FOO, &foo_config) },             /* with driver_data */
	{ }
};
MODULE_DEVICE_TABLE(pci, my_ids);          /* ★ autoload (Ch. 27 T.2) */

static int my_probe(struct pci_dev *pdev, const struct pci_device_id *id)
{
	struct my_priv *p;
	int ret;

	ret = pcim_enable_device(pdev);        /* ★ managed: auto-disable at remove */
	if (ret)
		return ret;

	p = devm_kzalloc(&pdev->dev, sizeof(*p), GFP_KERNEL);
	if (!p)
		return -ENOMEM;
	pci_set_drvdata(pdev, p);
	p->pdev = pdev;
	p->cfg = (const struct my_config *)id->driver_data;

	p->bar0 = pcim_iomap_region(pdev, 0, KBUILD_MODNAME);   /* request + map */
	if (IS_ERR(p->bar0))
		return dev_err_probe(&pdev->dev, PTR_ERR(p->bar0), "BAR0\n");

	ret = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64));
	if (ret)
		return dev_err_probe(&pdev->dev, ret, "DMA mask\n");

	pci_set_master(pdev);                  /* ★ T.7: enable DMA */

	ret = pci_alloc_irq_vectors(pdev, 1, p->cfg->nr_queues,
				    PCI_IRQ_MSIX | PCI_IRQ_MSI | PCI_IRQ_INTX |
				    PCI_IRQ_AFFINITY);
	if (ret < 0)
		return dev_err_probe(&pdev->dev, ret, "irq vectors\n");
	p->nvec = ret;

	ret = my_hw_init(p);                   /* ★ hardware quiesced BEFORE request_irq */
	if (ret)
		return ret;

	ret = devm_request_irq(&pdev->dev, pci_irq_vector(pdev, 0),
			       my_isr, 0, KBUILD_MODNAME, p);
	if (ret)
		return ret;

	return my_register_to_subsystem(p);    /* publish LAST */
}

static void my_remove(struct pci_dev *pdev)
{
	struct my_priv *p = pci_get_drvdata(pdev);

	my_unregister(p);                      /* stop new work */
	my_hw_disable(p);                      /* stop the engine */
	pci_clear_master(pdev);                /* ★ no more DMA */
	/* pcim_/devm_ unwind the rest in reverse (Ch. 28) */
}

static void my_shutdown(struct pci_dev *pdev)
{
	my_hw_disable(pci_get_drvdata(pdev));
	pci_clear_master(pdev);                /* ★ kexec safety (Ch. 31 T.6) */
}

static struct pci_driver my_driver = {
	.name     = KBUILD_MODNAME,
	.id_table = my_ids,
	.probe    = my_probe,
	.remove   = my_remove,
	.shutdown = my_shutdown,
	.driver   = { .pm = pm_ptr(&my_pm_ops) },
	.err_handler = &my_err_handler,        /* AER — Ch. 38 */
	.sriov_configure = pci_sriov_configure_simple,
};
module_pci_driver(my_driver);
```

**The `pcim_` family** (Ch. 28 T.6) deserves a note: `pcim_enable_device()` retroactively
makes *subsequent* `pci_*` calls managed — a genuinely surprising design that was reworked in
6.9+ to be explicit (`pcim_request_all_regions()`, `pcim_iomap_region()`). Read
`drivers/pci/devres.c` before mixing managed and unmanaged PCI calls.

---

## 1. Internals

### 1.1 Source map

```
drivers/pci/probe.c          ★★ enumeration: pci_scan_bus, pci_scan_bridge, BAR sizing
drivers/pci/setup-res.c      ★ resource assignment (the bin-packing of T.4)
drivers/pci/setup-bus.c      ★ bridge window sizing
drivers/pci/pci.c            ★ the core API: enable, set_master, save/restore state
drivers/pci/access.c         config-space accessors and locking
drivers/pci/pci-driver.c     ★ matching, probe, PM
drivers/pci/devres.c         pcim_*
drivers/pci/msi/             MSI/MSI-X (Ch. 38)
drivers/pci/pcie/            AER, ASPM, DPC, PTM, hotplug (Ch. 38)
drivers/pci/iov.c            SR-IOV (Ch. 38)
drivers/pci/quirks.c         ★ ~6000 lines of hardware workarounds — read some
drivers/pci/controller/      ★ host bridge drivers (the embedded side)
include/linux/pci.h          ★★ the API
include/uapi/linux/pci_regs.h ★ every register and bit, by name
Documentation/PCI/           ★ pci.rst, msi-howto.rst, pci-error-recovery.rst, sysfs-pci.rst
```

### 1.2 BAR sizing, in code

```c
/* drivers/pci/probe.c — __pci_read_base(), simplified */
pci_read_config_dword(dev, pos, &orig);
pci_write_config_dword(dev, pos, 0xffffffff);
pci_read_config_dword(dev, pos, &sz);
pci_write_config_dword(dev, pos, orig);          /* ★ always restore */

sz = pci_size(orig, sz, mask);                   /* = (~(sz & mask)) + 1 */
```
Reading this once makes T.4 concrete. Note the care taken to restore the original value —
and that the whole sequence must be atomic with respect to other config accesses, which is
why `pci_lock` exists.

### 1.3 The sysfs surface

```bash
D=/sys/bus/pci/devices/0000:03:00.0
ls $D
cat $D/{vendor,device,subsystem_vendor,subsystem_device,class,revision}
cat $D/{irq,local_cpulist,numa_node}
cat $D/resource                       # BAR list: start end flags
ls $D/resource0 $D/resource0_wc 2>/dev/null   # mmap-able BAR windows
hexdump -C $D/config | head -8
cat $D/enable $D/msi_bus $D/d3cold_allowed
echo 1 | sudo tee $D/remove           # hot-remove
echo 1 | sudo tee /sys/bus/pci/rescan # re-enumerate
echo 1 | sudo tee $D/reset            # function-level reset
```

---

## 2. Practice

### Lab 37.1 — Enumerate the bus yourself, from userspace

Reimplement T.3's algorithm. This is the single most instructive lab in the chapter.

```c
/* pciscan.c — gcc -O2 -o pciscan pciscan.c ; run as root */
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <stdint.h>
#include <string.h>
#include <sys/mman.h>
#include <unistd.h>

static void *ecam;                 /* mmap of the MCFG base */
#define CFG(bus, dev, fn, off) \
	((volatile uint8_t *)ecam + ((bus) << 20 | (dev) << 15 | (fn) << 12 | (off)))

static uint16_t r16(int b, int d, int f, int o) { return *(volatile uint16_t *)CFG(b,d,f,o); }
static uint32_t r32(int b, int d, int f, int o) { return *(volatile uint32_t *)CFG(b,d,f,o); }
static uint8_t  r8 (int b, int d, int f, int o) { return *(volatile uint8_t  *)CFG(b,d,f,o); }

static void scan_bus(int bus, int depth);

static void dump_fn(int b, int d, int f, int depth)
{
	uint16_t vend = r16(b,d,f,0x00), devid = r16(b,d,f,0x02);
	uint32_t cls  = r32(b,d,f,0x08) >> 8;
	uint8_t  hdr  = r8 (b,d,f,0x0e) & 0x7f;
	int i;

	printf("%*s%02x:%02x.%d  %04x:%04x  class %06x  hdr %d\n",
	       depth * 2, "", b, d, f, vend, devid, cls, hdr);

	if (hdr == 0) {                                   /* T.4: size the BARs */
		for (i = 0; i < 6; i++) {
			uint32_t off = 0x10 + i * 4, orig, sz;

			orig = r32(b,d,f,off);
			if (!orig)
				continue;
			/* NOTE: a real sizing writes 0xffffffff; we only DECODE here,
			 * because writing would disturb a live system. Use the kernel's
			 * /sys/.../resource for sizes instead. */
			printf("%*s   BAR%d = 0x%08x  %s %s %s\n", depth*2, "", i, orig,
			       (orig & 1) ? "I/O" : "MEM",
			       ((orig >> 1) & 3) == 2 ? "64-bit" : "32-bit",
			       (orig & 8) ? "prefetchable" : "");
			if (((orig >> 1) & 3) == 2)
				i++;                       /* 64-bit BAR eats the next slot */
		}
	} else if (hdr == 1) {                            /* a BRIDGE: recurse (T.3) */
		uint8_t sec = r8(b,d,f,0x19), sub = r8(b,d,f,0x1a);

		printf("%*s   bridge: secondary=%02x subordinate=%02x\n",
		       depth*2, "", sec, sub);
		if (sec)
			scan_bus(sec, depth + 1);
	}
}

static void scan_bus(int bus, int depth)
{
	int d, f;

	for (d = 0; d < 32; d++) {
		if (r16(bus, d, 0, 0x00) == 0xffff)       /* ★ T.3(a): no device */
			continue;
		dump_fn(bus, d, 0, depth);
		if (!(r8(bus, d, 0, 0x0e) & 0x80))        /* not multifunction */
			continue;
		for (f = 1; f < 8; f++)
			if (r16(bus, d, f, 0x00) != 0xffff)
				dump_fn(bus, d, f, depth);
	}
}

int main(void)
{
	int fd = open("/dev/mem", O_RDONLY | O_SYNC);
	/* Find the ECAM base from /sys/firmware/acpi/tables/MCFG, or hardcode 0xE0000000 */
	off_t base = 0xE0000000;

	ecam = mmap(NULL, 256UL << 20, PROT_READ, MAP_SHARED, fd, base);
	if (ecam == MAP_FAILED) { perror("mmap (try iomem=relaxed)"); return 1; }
	scan_bus(0, 0);
	return 0;
}
```
```bash
# Get the real ECAM base:
sudo cat /sys/firmware/acpi/tables/MCFG | xxd -s 44 -l 8
dmesg | grep -i MMCONFIG
# boot with iomem=relaxed to allow /dev/mem access, or use sysfs instead:
sudo hexdump -C /sys/bus/pci/devices/0000:00:00.0/config | head -4
sudo ./pciscan | head -40
lspci -t                                  # compare your output with lspci's
```
**Compare your tree with `lspci -t`.** When they match you understand PCI enumeration.

### Lab 37.2 — Size a BAR by hand, safely

Do it on a device you have unbound (Ch. 27 Lab 27.7), so nothing is using it:

```bash
D=0000:03:00.0
echo $D | sudo tee /sys/bus/pci/devices/$D/driver/unbind

# Read BAR0, write all-ones, read back, restore — via setpci
ORIG=$(sudo setpci -s $D BASE_ADDRESS_0)
echo "original BAR0 = $ORIG"
sudo setpci -s $D BASE_ADDRESS_0=ffffffff
SZ=$(sudo setpci -s $D BASE_ADDRESS_0)
sudo setpci -s $D BASE_ADDRESS_0=$ORIG
echo "sized value   = $SZ"
python3 -c "
sz=int('$SZ',16) & ~0xF
print('size =', hex((~sz & 0xffffffff) + 1))"

# Check against what the kernel computed:
cat /sys/bus/pci/devices/$D/resource | head -1
sudo lspci -vv -s $D | grep Region

echo $D | sudo tee /sys/bus/pci/drivers_probe
```

### Lab 37.3 — Walk the capability lists (T.5)

```c
/* In a driver, or reimplement over /sys/.../config */
static void dump_caps(struct pci_dev *pdev)
{
	u8 pos;
	u16 epos;
	u32 header;

	pci_read_config_byte(pdev, PCI_CAPABILITY_LIST, &pos);
	while (pos >= 0x40) {
		u8 id, next;

		pci_read_config_byte(pdev, pos, &id);
		pci_read_config_byte(pdev, pos + 1, &next);
		dev_info(&pdev->dev, "cap @0x%02x id=0x%02x (%s)\n", pos, id,
			 id == PCI_CAP_ID_PM  ? "PM"   : id == PCI_CAP_ID_MSI ? "MSI" :
			 id == PCI_CAP_ID_EXP ? "PCIe" : id == PCI_CAP_ID_MSIX ? "MSI-X" : "?");
		pos = next;
	}

	epos = PCI_CFG_SPACE_SIZE;                     /* 0x100 */
	while (epos) {
		pci_read_config_dword(pdev, epos, &header);
		if (header == 0 || header == 0xffffffff)
			break;
		dev_info(&pdev->dev, "ext cap @0x%03x id=0x%04x ver=%d\n",
			 epos, PCI_EXT_CAP_ID(header), PCI_EXT_CAP_VER(header));
		epos = PCI_EXT_CAP_NEXT(header);
	}
}
```
```bash
# Compare with:
sudo lspci -vv -s 03:00.0 | grep -E '^\s+Capabilities:'
# Walk it by hand from the raw config space:
sudo hexdump -C /sys/bus/pci/devices/0000:03:00.0/config | sed -n '4,20p'
grep -n 'PCI_CAP_ID_\|PCI_EXT_CAP_ID_' include/uapi/linux/pci_regs.h | head -40
```

### Lab 37.4 — A complete driver for QEMU's `edu` device

Extends Ch. 17 Lab 17.B with BARs, DMA, and capabilities.

```c
// SPDX-License-Identifier: GPL-2.0
/* QEMU 'edu' device: -device edu   (docs/specs/edu.rst in the QEMU tree) */
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

#include <linux/dma-mapping.h>
#include <linux/interrupt.h>
#include <linux/module.h>
#include <linux/pci.h>

#define EDU_ID            0x00
#define EDU_LIVENESS      0x04    /* write x, read ~x */
#define EDU_FACTORIAL     0x08
#define EDU_STATUS        0x20
#define  EDU_STATUS_BUSY  BIT(0)
#define  EDU_STATUS_IRQ   BIT(7)
#define EDU_IRQ_STATUS    0x24
#define EDU_IRQ_RAISE     0x60
#define EDU_IRQ_ACK       0x64
#define EDU_DMA_SRC       0x80
#define EDU_DMA_DST       0x88
#define EDU_DMA_CNT       0x90
#define EDU_DMA_CMD       0x98
#define  EDU_DMA_START    BIT(0)
#define  EDU_DMA_TO_DEV   0
#define  EDU_DMA_FROM_DEV BIT(1)
#define  EDU_DMA_IRQ      BIT(2)
#define EDU_DMA_BASE      0x40000   /* the device's internal buffer address */

struct edu {
	struct pci_dev   *pdev;
	void __iomem     *mmio;
	struct completion done;
	void             *dma_buf;
	dma_addr_t        dma_handle;
};

static irqreturn_t edu_isr(int irq, void *data)
{
	struct edu *e = data;
	u32 st = readl(e->mmio + EDU_IRQ_STATUS);

	if (!st)
		return IRQ_NONE;
	writel(st, e->mmio + EDU_IRQ_ACK);
	readl(e->mmio + EDU_IRQ_STATUS);          /* ★ flush (Ch. 34 T.4) */
	complete(&e->done);
	return IRQ_HANDLED;
}

static int edu_dma_test(struct edu *e)
{
	struct device *dev = &e->pdev->dev;

	strscpy(e->dma_buf, "hello from the kernel", 64);

	/* ★ ownership transfer to the device (Ch. 35 T.4) */
	dma_sync_single_for_device(dev, e->dma_handle, 64, DMA_TO_DEVICE);

	writeq(e->dma_handle,  e->mmio + EDU_DMA_SRC);
	writeq(EDU_DMA_BASE,   e->mmio + EDU_DMA_DST);
	writeq(64,             e->mmio + EDU_DMA_CNT);
	reinit_completion(&e->done);
	writeq(EDU_DMA_START | EDU_DMA_TO_DEV | EDU_DMA_IRQ, e->mmio + EDU_DMA_CMD);

	if (!wait_for_completion_timeout(&e->done, HZ)) {
		dev_err(dev, "DMA to device timed out\n");
		return -ETIMEDOUT;                /* ★ Ch. 25 P7: must also abort in real code */
	}

	memset(e->dma_buf, 0, 64);
	dma_sync_single_for_device(dev, e->dma_handle, 64, DMA_FROM_DEVICE);

	writeq(EDU_DMA_BASE,  e->mmio + EDU_DMA_SRC);
	writeq(e->dma_handle, e->mmio + EDU_DMA_DST);
	writeq(64,            e->mmio + EDU_DMA_CNT);
	reinit_completion(&e->done);
	writeq(EDU_DMA_START | EDU_DMA_FROM_DEV | EDU_DMA_IRQ, e->mmio + EDU_DMA_CMD);

	if (!wait_for_completion_timeout(&e->done, HZ))
		return -ETIMEDOUT;

	dma_sync_single_for_cpu(dev, e->dma_handle, 64, DMA_FROM_DEVICE);
	dev_info(dev, "DMA round trip: '%s'\n", (char *)e->dma_buf);
	return 0;
}

static int edu_probe(struct pci_dev *pdev, const struct pci_device_id *id)
{
	struct device *dev = &pdev->dev;
	struct edu *e;
	u32 v;
	int ret;

	e = devm_kzalloc(dev, sizeof(*e), GFP_KERNEL);
	if (!e)
		return -ENOMEM;
	e->pdev = pdev;
	init_completion(&e->done);
	pci_set_drvdata(pdev, e);

	ret = pcim_enable_device(pdev);
	if (ret)
		return ret;

	e->mmio = pcim_iomap_region(pdev, 0, KBUILD_MODNAME);
	if (IS_ERR(e->mmio))
		return dev_err_probe(dev, PTR_ERR(e->mmio), "BAR0\n");

	/* Identify and liveness-check */
	v = readl(e->mmio + EDU_ID);
	dev_info(dev, "edu device, version %u.%u\n", (v >> 16) & 0xff, v & 0xff);
	writel(0xdeadbeef, e->mmio + EDU_LIVENESS);
	v = readl(e->mmio + EDU_LIVENESS);
	if (v != ~0xdeadbeefu)
		return dev_err_probe(dev, -EIO, "liveness check failed (0x%08x)\n", v);

	ret = dma_set_mask_and_coherent(dev, DMA_BIT_MASK(64));
	if (ret)
		return ret;
	pci_set_master(pdev);                      /* ★ T.7: without this, DMA does nothing */

	e->dma_buf = dmam_alloc_coherent(dev, 4096, &e->dma_handle, GFP_KERNEL);
	if (!e->dma_buf)
		return -ENOMEM;

	ret = pci_alloc_irq_vectors(pdev, 1, 1, PCI_IRQ_MSI | PCI_IRQ_INTX);
	if (ret < 0)
		return dev_err_probe(dev, ret, "irq vectors\n");
	dev_info(dev, "using %s\n", pdev->msi_enabled ? "MSI" : "INTx");

	ret = devm_request_irq(dev, pci_irq_vector(pdev, 0), edu_isr,
			       pdev->msi_enabled ? 0 : IRQF_SHARED, KBUILD_MODNAME, e);
	if (ret)
		return ret;

	/* Exercise the device: factorial via interrupt */
	reinit_completion(&e->done);
	writel(10, e->mmio + EDU_FACTORIAL);
	writel(0x100, e->mmio + EDU_IRQ_RAISE);
	wait_for_completion_timeout(&e->done, HZ);
	dev_info(dev, "10! = %u\n", readl(e->mmio + EDU_FACTORIAL));

	return edu_dma_test(e);
}

static void edu_remove(struct pci_dev *pdev)
{
	struct edu *e = pci_get_drvdata(pdev);

	writel(0, e->mmio + EDU_DMA_CMD);
	readl(e->mmio + EDU_DMA_CMD);
	pci_clear_master(pdev);                    /* ★ stop DMA before devres frees the buffer */
}

static const struct pci_device_id edu_ids[] = {
	{ PCI_DEVICE(0x1234, 0x11e8) },
	{ }
};
MODULE_DEVICE_TABLE(pci, edu_ids);

static struct pci_driver edu_driver = {
	.name = KBUILD_MODNAME, .id_table = edu_ids,
	.probe = edu_probe, .remove = edu_remove, .shutdown = edu_remove,
};
module_pci_driver(edu_driver);
MODULE_LICENSE("GPL");
```
```bash
qemu-system-x86_64 -M q35 -enable-kvm -m 2G -device edu \
  -kernel bzImage -initrd initramfs.cpio.gz -append "console=ttyS0 rdinit=/init" -nographic
# in the guest:
lspci -nn | grep 1234:11e8
insmod edu.ko && dmesg | tail -10
cat /sys/bus/pci/devices/*/resource | head
grep edu /proc/interrupts
```
**Then break it deliberately:** remove `pci_set_master()` and watch the DMA silently never
complete. That single experiment teaches T.7 permanently.

### Lab 37.5 — Explore the full PCI sysfs and `setpci` surface

```bash
D=0000:03:00.0
# Identity
cat /sys/bus/pci/devices/$D/{vendor,device,subsystem_vendor,subsystem_device,class}
lspci -nnvvv -s $D | head -40

# Resources
cat /sys/bus/pci/devices/$D/resource
sudo hexdump -C /sys/bus/pci/devices/$D/resource0 | head -4   # read BAR0 directly!

# Topology and NUMA
cat /sys/bus/pci/devices/$D/{numa_node,local_cpulist}
readlink -f /sys/bus/pci/devices/$D

# Link status (T.6)
cat /sys/bus/pci/devices/$D/{current_link_speed,max_link_speed}
cat /sys/bus/pci/devices/$D/{current_link_width,max_link_width}

# Raw register access
sudo setpci -s $D COMMAND
sudo setpci -s $D COMMAND=0x0007        # IO|MEM|MASTER
sudo setpci -s $D CAP_EXP+0x08.w        # PCIe device capabilities

# Hot remove / rescan / reset
echo 1 | sudo tee /sys/bus/pci/devices/$D/remove
echo 1 | sudo tee /sys/bus/pci/rescan
echo 1 | sudo tee /sys/bus/pci/devices/$D/reset

# What did enumeration do at boot?
dmesg | grep -E "pci $D|pci_bus" | head -20
```

### Lab 37.6 — Reproduce a BAR-assignment failure (T.4)

```bash
# On a machine with a large-BAR GPU, or in QEMU with a big BAR:
qemu-system-x86_64 -M q35 -m 2G \
  -device ivshmem-plain,memdev=hostmem -object memory-backend-file,size=4G,id=hostmem,mem-path=/dev/shm/ivshmem,share=on \
  ...

dmesg | grep -iE 'BAR .*: (assigned|no space|failed|reserving)'
dmesg | grep -i 'above 4G\|64bit'
cat /proc/iomem | grep -i 'PCI Bus'

# Force reallocation and watch the bin-packing:
# boot with: pci=realloc=on  or  pci=nocrs  or  pci=assign-busses
dmesg | grep -i 'pci.*realloc\|releasing\|reassign'
$EDITOR drivers/pci/setup-bus.c   # pbus_size_mem(), the window sizing algorithm
```

### Lab 37.7 — Read the quirks file

```bash
wc -l drivers/pci/quirks.c
grep -c 'DECLARE_PCI_FIXUP' drivers/pci/quirks.c
grep -n 'DECLARE_PCI_FIXUP_EARLY\|DECLARE_PCI_FIXUP_HEADER\|DECLARE_PCI_FIXUP_FINAL' \
     drivers/pci/quirks.c | head -20
# Pick three quirks and find the bug report or commit that motivated each:
git log -L :quirk_disable_msi:drivers/pci/quirks.c | head -60
```
**This file is 6000 lines of hardware being wrong.** Reading a sample of it is the fastest
cure for the belief that specifications are followed.

---

## 3. Mastery drills

1. **Read `drivers/pci/probe.c`** end to end. Write out the complete enumeration algorithm
   including multifunction detection, bridge programming, and BAR sizing. Compare with your
   Lab 37.1 implementation.

2. **BAR sizing proof.** Explain mathematically why `~(value & ~0xF) + 1` yields the size,
   and why it requires that the device implement a *contiguous* run of high address bits.
   What happens with a non-power-of-two size request? (It cannot be expressed.)

3. **Bridge windows.** Take a machine with several bridges. For each, record the memory, prefetchable, and I/O windows and the devices behind it. Verify each window contains all
   its children and is naturally aligned. Explain why this forces power-of-two sizing.

4. **Capability list.** Walk both lists on five different devices. Build a table of which
   capabilities appear on which device classes. Which extended capabilities require ECAM?

5. **PCIe compatibility.** Write 500 words on how PCIe preserved software compatibility while
   replacing the physical layer. What was kept, what was abstracted, and what is the general
   lesson for interface design? (Relate to Ch. 24 T.1 and Ch. 32 T.7.)

6. **Prefetchable semantics.** Find a device with both prefetchable and non-prefetchable
   BARs. Explain what each region contains and why the distinction is correct. Then find a
   driver that uses `ioremap_wc()` on a prefetchable BAR.

7. **Config access cost.** Measure `pci_read_config_dword()` latency in a module. Compare
   ECAM and `pci=nommconf`. Explain why config space must never appear in a datapath.

8. **Quirk archaeology.** Pick a `DECLARE_PCI_FIXUP_FINAL` quirk. Determine what the hardware
   does wrong, when the fixup runs relative to enumeration, and what breaks without it.

9. **Host bridge drivers.** Read one `drivers/pci/controller/` driver (`pcie-designware.c`,
   `pci-aardvark.c`, or `pcie-rcar.c`). Explain how an SoC's PCIe root complex is described
   in DT and how ECAM (or a config-access workaround) is implemented.

10. **Design question.** You are writing a driver for a device with 4 BARs: registers (64 KiB,
    non-prefetchable), a doorbell page (4 KiB, must be write-combining), a 2 GiB frame buffer
    (prefetchable, 64-bit), and a small MSI-X table BAR. Write the probe: which mapping
    function for each, what DMA mask, how many vectors, and what your `remove`/`shutdown` do.

---

## 4. Further reading

**Specifications:**
- **PCI Express Base Specification** (PCI-SIG) — §1–2 (architecture, TLPs), §6 (capabilities,
  ACS, ATS), §7 (config space). The config-space and capability chapters are the ones you
  will reread.
- PCI Local Bus Specification 3.0 — the classic config-space and BAR definitions
- PCI Firmware Specification — `_CRS`/MCFG/ECAM and OS/firmware handoff

**Kernel documentation:**
- `Documentation/PCI/pci.rst` ★★ — the driver-writer's guide
- `Documentation/PCI/sysfs-pci.rst`, `pci-iov-howto.rst`, `msi-howto.rst`,
  `pci-error-recovery.rst`, `acpi-info.rst`
- `Documentation/driver-api/pci/` — `pci.rst`, `p2pdma.rst`
- `include/uapi/linux/pci_regs.h` ★ — **keep this open**; every register by name

**Source:**
- `drivers/pci/probe.c` ★★, `setup-res.c`, `setup-bus.c`
- `drivers/pci/pci.c`, `pci-driver.c`, `devres.c`
- `drivers/pci/quirks.c` — a sample
- `drivers/pci/controller/` — the embedded/host-bridge side
- Exemplary drivers: `drivers/nvme/host/pci.c` ★ (modern, clean),
  `drivers/net/ethernet/intel/e1000e/` (classic), `drivers/misc/pci_endpoint_test.c`

**Books & references:**
- Mindshare, *PCI Express System Architecture* — the standard reference, if dated
- Corbet/Rubini/Kroah-Hartman, *Linux Device Drivers* 3e, Ch. 12 (PCI) — the model, not the API
- QEMU `docs/specs/edu.rst` and `docs/specs/ivshmem-spec.txt` — the toy devices for labs

**LWN:**
- "The PCI subsystem" series
- "Resizable BARs and the kernel", "PCI peer-to-peer DMA"
- "pcim_* rework" (6.9) — the managed-API cleanup

→ Next: [38-pcie-advanced.md](38-pcie-advanced.md)

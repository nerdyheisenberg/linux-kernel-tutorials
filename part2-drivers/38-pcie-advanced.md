# Chapter 38 — PCIe Advanced: MSI/MSI-X, SR-IOV, AER, Hotplug, P2PDMA, ASPM

> **Goal:** the PCIe features that separate a toy driver from a production one — interrupt
> scaling, error recovery, virtualization, power management, and peer-to-peer data movement.

---

## Theory & First Principles

### T.0 — Start here: PCIe is not a bus

The name says "PCI Express" and the software model is deliberately identical — same config
space, same BARs, same enumeration (Ch. 37). **The hardware is completely different**, and
the differences leak into driver design in ways that matter.

```
  PCI (1992)                          PCIe (2003)
  +----------------------+        +--------------------------------+
  |  A PARALLEL BUS.     |        |  A PACKET-SWITCHED NETWORK.    |
  |  Everyone shares it. |        |  Point-to-point serial LINKS.  |
  |  One transfer at a   |        |  A SWITCH routes packets.      |
  |  time. Arbitration.  |        |  Everyone has full bandwidth.  |
  |  Shared interrupt    |        |  Credit-based FLOW CONTROL.    |
  |  lines (INTx).       |        |  Retries, CRC, error recovery. |
  +----------------------+        +--------------------------------+
```

**PCIe is a network**, with packets (TLPs), routing, flow control, error detection and
retransmission. The PCI software model is an *emulation* layered on top for compatibility,
and it is a remarkably successful one — but four things escape the emulation and you must
know them.

**1. Interrupts stopped being wires.** PCI had four physical interrupt lines shared by every
device, so every handler had to ask "was it me?" (Ch. 17 §T.0). PCIe has no interrupt wires
at all — an interrupt is a **memory write** to a magic address:

```
  MSI-X:  device writes  <data>  to  <address>   ->  the CPU takes an interrupt
          Up to 2048 vectors per device, each independently targetable.
```

That is not a minor change. It means **one interrupt vector per queue, per CPU**, which is
what makes multi-queue NICs (Ch. 46) and NVMe (Ch. 69) possible at all. With one shared
interrupt line you have a global serialization point, and you cannot exceed a few hundred
thousand IOPS no matter what else you do.

**2. Bandwidth is per-link, and the topology is now load-bearing.** A device behind a x4
switch uplink shares that uplink with everything else behind it. `lspci -t` is not cosmetic
— it tells you where your bandwidth actually goes.

**3. Errors are recoverable, so there is a recovery protocol.** A parallel-bus error was
fatal. A packet-switched link can retry, and when it cannot, **AER** reports it and the
kernel can reset the device and ask the driver to recover via `pci_error_handlers`. A driver
without those callbacks turns a recoverable link error into a dead device.

**4. Power management became a link-level concern.** ASPM (L0s, L1, L1.1, L1.2) powers down
an *idle link*, saving real milliwatts and adding real microseconds of exit latency. This is
the single most common cause of "my NVMe has terrible tail latency" and of "my laptop's NIC
misbehaves after idle."

**The general lesson, which recurs in Ch. 46 and Ch. 69:**

> When a shared resource becomes point-to-point and a serialized one becomes parallel, the
> *software* bottleneck moves. PCIe removed the bus-arbitration bottleneck, which exposed the
> single-interrupt bottleneck, which MSI-X removed, which exposed the single-queue
> bottleneck, which blk-mq and multiqueue networking removed.

```bash
lspci -vv -s <dev> | grep -A3 'LnkCap\|LnkSta'   # negotiated width and speed
lspci -vv -s <dev> | grep -A10 'MSI-X'           # how many vectors
cat /proc/interrupts | grep -c nvme              # one line per queue
lspci -vv | grep -i 'ASPM\|L1Sub'
dmesg | grep -i 'AER\|PCIe Bus Error'
```

---

### T.1 MSI-X: interrupts as a scalability mechanism

Ch. 17 T.5 established *why* MSI exists (unlimited vectors, ordering with DMA, no sharing).
This chapter is about what you *build* with that.

The insight that reshaped modern device design: **if each interrupt vector can be delivered
to a specific CPU, then you can build an entire I/O path with no cross-CPU sharing.**

```
   ┌── CPU0 ──┐   ┌── CPU1 ──┐   ┌── CPU2 ──┐   ┌── CPU3 ──┐
   │ queue 0  │   │ queue 1  │   │ queue 2  │   │ queue 3  │   ← per-CPU submission/completion
   │ vector 0 │   │ vector 1 │   │ vector 2 │   │ vector 3 │   ← per-CPU MSI-X vector
   └────┬─────┘   └────┬─────┘   └────┬─────┘   └────┬─────┘
        └──────────────┴──────┬───────┴──────────────┘
                          the device
```

This is Ch. 16's partitioning principle (T.4) realized in *hardware*. Every layer cooperates:
NVMe has per-CPU submission/completion queue pairs; `blk-mq` has per-CPU software queues
mapped to hardware queues; a NIC has RSS steering flows to per-CPU rings. The result is an
I/O path where **no cache line is shared between CPUs** from the application down to the
device.

`PCI_IRQ_AFFINITY` is the mechanism that ties it together:

```c
struct irq_affinity affd = {
	.pre_vectors  = 1,      /* e.g. an admin/control vector, not per-CPU */
	.post_vectors = 0,
};

nvec = pci_alloc_irq_vectors_affinity(pdev, min_vecs, max_vecs,
				      PCI_IRQ_MSIX | PCI_IRQ_MSI | PCI_IRQ_AFFINITY,
				      &affd);

/* The core spread the vectors across CPUs. Ask it how: */
const struct cpumask *m = pci_irq_get_affinity(pdev, vec);
```

Two consequences you must know:

1. **The resulting IRQs are `IRQD_AFFINITY_MANAGED`** — userspace *cannot* change
   `/proc/irq/N/smp_affinity` (it returns `-EIO`). This surprises administrators constantly.
   The reason is correctness: `blk-mq` built its queue→CPU map from this affinity, so
   changing it would send completions to a CPU with no matching queue.
2. **CPU hotplug is handled for you.** When a CPU goes offline, a managed IRQ is *shut down*
   rather than migrated, and the block layer drains that queue. That coordination is why the
   affinity must be managed.

```c
/* The block layer's side of the contract: */
blk_mq_pci_map_queues(&set->map[HCTX_TYPE_DEFAULT], pdev, offset);
```

```bash
cat /proc/interrupts | grep -E 'nvme|mlx|ice' | head
for i in $(grep nvme /proc/interrupts | awk '{gsub(":","",$1); print $1}'); do
  echo "irq $i: affinity=$(cat /proc/irq/$i/smp_affinity_list) effective=$(cat /proc/irq/$i/effective_affinity_list)"
done
echo 3 | sudo tee /proc/irq/$IRQ/smp_affinity_list      # → -EIO on managed IRQs
cat /sys/block/nvme0n1/mq/*/cpu_list
```

**MSI vs MSI-X, concretely:**

| | MSI | MSI-X |
|---|---|---|
| Vectors | 1–32, **power of two, contiguous** | 1–2048, **independent** |
| Where configured | config space capability | **a table in device memory (a BAR)** |
| Per-vector masking | no (one mask for all) | **yes** |
| Per-vector affinity | no — one address for all | **yes** |
| Usable for multi-queue | no | **yes** |

MSI-X's table-in-a-BAR is the enabling difference: 2048 independent address/data pairs cannot
fit in config space.

### T.2 AER and error recovery: a state machine across the stack

PCIe defines error reporting at the link level. **Correctable** errors (a recovered link
error) are counted; **uncorrectable** errors are either *non-fatal* (a bad TLP, contained) or
*fatal* (the link is gone).

The kernel turns this into a **driver callback protocol** — one of the few places a driver is
told "your device just broke, cope":

```c
static pci_ers_result_t my_error_detected(struct pci_dev *pdev, pci_channel_state_t state)
{
	struct my_priv *p = pci_get_drvdata(pdev);

	/* ★ The device may be unreachable. Do NOT touch MMIO if state == perm_failure. */
	switch (state) {
	case pci_channel_io_normal:
		return PCI_ERS_RESULT_CAN_RECOVER;
	case pci_channel_io_frozen:
		my_quiesce(p);                     /* stop queues, stop DMA */
		return PCI_ERS_RESULT_NEED_RESET;
	case pci_channel_io_perm_failure:
		my_mark_dead(p);
		return PCI_ERS_RESULT_DISCONNECT;
	}
	return PCI_ERS_RESULT_NEED_RESET;
}

static pci_ers_result_t my_slot_reset(struct pci_dev *pdev)
{
	/* The bus has been reset. The device is at power-on defaults. */
	if (pci_enable_device_mem(pdev))
		return PCI_ERS_RESULT_DISCONNECT;
	pci_set_master(pdev);
	pci_restore_state(pdev);
	my_hw_reinit(pci_get_drvdata(pdev));
	return PCI_ERS_RESULT_RECOVERED;
}

static void my_resume(struct pci_dev *pdev)
{
	my_restart_queues(pci_get_drvdata(pdev));   /* resume normal operation */
}

static const struct pci_error_handlers my_err_handler = {
	.error_detected = my_error_detected,
	.mmio_enabled   = my_mmio_enabled,      /* optional: MMIO restored, DMA not yet */
	.slot_reset     = my_slot_reset,
	.resume         = my_resume,
};
```

The protocol's design deserves attention: it is a **two-phase commit across all drivers in
the error domain**. Every affected driver is asked `error_detected()`; the *most severe*
answer wins; then the reset is performed once; then every driver gets `slot_reset()` and
`resume()`. A single driver that says "disconnect" tears everything down.

**The critical rule:** during `error_detected()` with `io_frozen`, all MMIO reads return
`~0` and writes are discarded. A driver that does not check for `~0` will happily interpret
garbage as a status value. Hence the idiom:

```c
val = readl(base + STATUS);
if (val == ~0u && pci_channel_offline(pdev))
	return -EIO;                 /* the device is gone, not "all bits set" */
```

**DPC (Downstream Port Containment)** is the modern hardware assist: on an uncorrectable
error the *port* automatically disables the link, containing the error before a poisoned TLP
propagates. It converts "unpredictable corruption" into "clean device removal", and it is what
makes hot-unplug of a busy NVMe device survivable.

```bash
sudo lspci -vv | grep -A8 'Advanced Error Reporting'
cat /sys/bus/pci/devices/*/aer_dev_correctable 2>/dev/null | head
cat /sys/bus/pci/devices/*/aer_dev_nonfatal 2>/dev/null | head
dmesg | grep -iE 'AER|DPC|PCIe Bus Error'
ls /sys/kernel/debug/pcie_aer_inject 2>/dev/null
```

### T.3 SR-IOV: hardware-implemented device multiplication

A single physical device presents itself as **one Physical Function (PF)** plus **N Virtual
Functions (VFs)**, each with its own BARs, its own requester ID (and hence its own IOMMU
group and domain), and its own MSI-X vectors.

```
      ┌──────────── one physical NIC ────────────┐
      │  PF (0000:03:00.0)   — the full device   │  ← the host driver configures
      │  ├── VF0 (0000:03:10.0)                  │  ← passed to VM 1
      │  ├── VF1 (0000:03:10.1)                  │  ← passed to VM 2
      │  └── VF2 (0000:03:10.2)                  │  ← used by DPDK in a container
      │  internal switch / queue partitioning     │
      └───────────────────────────────────────────┘
```

Why it matters: a VM with a passed-through VF does I/O at **native speed** with no hypervisor
involvement on the datapath — no virtio ring, no VM exit per packet. The cost is loss of
migration flexibility (the VM is now tied to specific hardware) and a much larger attack
surface.

The VF's BARs are *not* individually sized (T.4 of Ch. 37 does not apply): the SR-IOV
capability declares a single BAR size and a stride, and all VF BARs are carved from one
contiguous region. That is why enabling SR-IOV can fail with "not enough MMIO resources" on
machines with a constrained PCI window.

```c
static int my_sriov_configure(struct pci_dev *pdev, int num_vfs)
{
	if (num_vfs == 0) {
		pci_disable_sriov(pdev);
		return 0;
	}
	my_configure_vf_resources(pci_get_drvdata(pdev), num_vfs);
	return pci_enable_sriov(pdev, num_vfs) ?: num_vfs;
}
/* .sriov_configure = my_sriov_configure  (or pci_sriov_configure_simple) */
```
```bash
cat /sys/bus/pci/devices/0000:03:00.0/sriov_totalvfs
echo 4 | sudo tee /sys/bus/pci/devices/0000:03:00.0/sriov_numvfs
lspci | grep 'Virtual Function'
ls /sys/bus/pci/devices/0000:03:00.0/virtfn*
# Each VF is in its OWN IOMMU group (Ch. 36 T.2) — that's the point:
for v in /sys/bus/pci/devices/0000:03:00.0/virtfn*; do
  echo "$(basename $(readlink $v)) → group $(basename $(readlink $v/iommu_group))"
done
ip link show dev enp3s0                     # VF MAC/VLAN/trust, set via the PF
sudo ip link set enp3s0 vf 0 mac 52:54:00:11:22:33 trust on
```

**SIOV / SF (Scalable IOV, Subfunctions)** is the successor: instead of full PCIe functions,
a device exposes lighter-weight "subfunctions" identified by PASID, avoiding the hard limit on
function numbers and the BAR-space cost. Worth knowing by name; `devlink` is its control
plane.

### T.4 Hotplug and surprise removal

PCIe supports hotplug natively (`pciehp`) and, increasingly, **surprise** removal —
Thunderbolt, U.2/NVMe drive bays, CXL.

The hard part is not the insertion path; it is **surprise removal while I/O is in flight**.
Every outstanding request must complete (with an error), every mapping must be torn down, and
the driver must not touch the device again. This is Ch. 25 P12's quiescence problem with the
additional twist that *the device is already gone*.

```c
/* Every MMIO read must be validated once removal is possible */
if (pci_device_is_present(pdev)) { ... }
if (pci_dev_is_disconnected(pdev)) return -ENODEV;

/* The surprise-removal idiom */
val = readl(base + REG);
if (val == ~0u) {
	if (!pci_device_is_present(pdev))
		return -ENODEV;
}
```

```bash
dmesg | grep -i 'pciehp\|shpchp\|Card present\|Link Up\|Link Down'
cat /sys/bus/pci/slots/*/address /sys/bus/pci/slots/*/power 2>/dev/null
echo 0 | sudo tee /sys/bus/pci/slots/1/power      # simulated removal
# Thunderbolt:
cat /sys/bus/thunderbolt/devices/*/authorized 2>/dev/null
boltctl list 2>/dev/null
```

### T.5 ASPM and power management: latency you did not ask for

**ASPM (Active State Power Management)** lets a link enter low-power states when idle:

| State | Behaviour | Exit latency |
|---|---|---|
| L0 | fully active | — |
| **L0s** | one direction idle | ~1 µs |
| **L1** | both directions idle, PLL off | ~10–60 µs |
| **L1.1 / L1.2** (LTR/OBFF) | deeper, common-mode voltage off | **up to ms** |

The exit latency is *added to every transaction that finds the link asleep*. For an NVMe
device this can turn a 10 µs read into a 70 µs read. Hence:

- Servers and latency-sensitive systems often disable ASPM (`pcie_aspm=off`).
- Laptops depend on L1.2 for battery life; disabling it costs hours.
- The **LTR (Latency Tolerance Reporting)** capability is how a device tells the platform how
  much wake latency it can tolerate, so the platform can choose a state. Broken LTR values are
  a known cause of both "disk is slow" and "battery drains".

This is a genuine, quantifiable **latency-vs-power dial**, and knowing it exists is what
separates "the NVMe is randomly slow" from a diagnosis.

```bash
sudo lspci -vv | grep -E 'ASPM|LnkCtl:|L1SubCtl' | head -20
cat /sys/module/pcie_aspm/parameters/policy
echo performance | sudo tee /sys/module/pcie_aspm/parameters/policy   # or powersave/default
cat /sys/bus/pci/devices/*/link/l1_aspm 2>/dev/null
sudo powertop --auto-tune        # then re-measure your I/O latency
# Device-level PM:
cat /sys/bus/pci/devices/*/power/control | sort | uniq -c
cat /sys/bus/pci/devices/0000:03:00.0/power_state
```

### T.6 P2PDMA: device-to-device transfers

Normally data flows device → RAM → device. **Peer-to-peer DMA** lets one device write
directly into another's BAR, bypassing system memory:

```
   Traditional:   NVMe ──▶ RAM ──▶ GPU        (two crossings of the memory bus)
   P2PDMA:        NVMe ─────────▶ GPU BAR     (one hop, over the PCIe fabric)
```

Use cases: GPUDirect Storage, RDMA NICs reading from NVMe, accelerator pipelines.

The constraints are severe and explain why the API is narrow:

1. **The path must not cross a root complex** on many platforms — some CPUs simply do not
   route peer-to-peer traffic between root ports, or do so at terrible speed. The kernel
   maintains a whitelist (`pci_p2pdma_distance()`).
2. **ACS must be disabled** on the intervening bridges for traffic to stay local — which is
   exactly the isolation you wanted in Ch. 36 T.2. **P2PDMA and IOMMU isolation are in direct
   tension**, and the kernel makes you choose.
3. The target BAR must be exposed as memory the DMA API understands
   (`pci_p2pdma_add_resource()` registers it as `ZONE_DEVICE` memory).

```c
pci_p2pdma_add_resource(pdev, bar, size, offset);   /* provider */
p2p_mem = pci_alloc_p2pmem(provider, len);          /* consumer allocates from it */
pci_p2pdma_map_sg(dev, sgl, nents, dir);
```

```bash
ls /sys/bus/pci/devices/*/p2pmem/ 2>/dev/null
cat /sys/bus/pci/devices/*/p2pmem/size 2>/dev/null
$EDITOR Documentation/driver-api/pci/p2pdma.rst
grep -rn 'pci_p2pdma' drivers/nvme/ drivers/infiniband/ | head
```

### T.7 Other capabilities worth recognizing

| Capability | What it does | Why you care |
|---|---|---|
| **Resizable BAR** | negotiate BAR size at runtime | GPUs exposing all VRAM ("Smart Access Memory"); needs Above-4G |
| **ATS / PRI / PASID** | device TLB + device page faults | SVA (Ch. 36 T.4); accelerator programming models |
| **PTM (Precision Time)** | sub-µs clock sync across the fabric | audio, industrial, time-sensitive networking |
| **DOE (Data Object Exchange)** | a mailbox in config space | CMA/SPDM device attestation, CXL |
| **VC / TC** | traffic classes and virtual channels | QoS; rarely used |
| **CXL** | cache-coherent attach over the PCIe PHY | memory expansion/tiering (Ch. 23 T.10) |

```bash
sudo lspci -vv | grep -E 'Resizable BAR|Precision Time|Data Object Exchange|Address Translation Service|Process Address Space'
cat /sys/bus/pci/devices/*/resource0_resize 2>/dev/null
ls /sys/bus/cxl/devices/ 2>/dev/null
```

---

## 1. Internals

### 1.1 Source map

```
drivers/pci/msi/msi.c, irqdomain.c   ★ MSI/MSI-X allocation and the IRQ domain hierarchy
drivers/pci/pcie/aer.c               ★★ AER: error decoding and the recovery state machine
drivers/pci/pcie/err.c               ★ the driver-callback protocol (T.2)
drivers/pci/pcie/dpc.c               downstream port containment
drivers/pci/pcie/aspm.c              ★ ASPM policy and L1 substates
drivers/pci/pcie/ptm.c, edr.c, ptm.c
drivers/pci/hotplug/pciehp_*.c       ★ hotplug
drivers/pci/iov.c                    ★ SR-IOV
drivers/pci/p2pdma.c                 ★ peer-to-peer DMA
drivers/pci/ats.c                    ATS/PASID/PRI
kernel/irq/affinity.c                ★ irq_create_affinity_masks — the spreading algorithm
block/blk-mq-pci.c                   the blk-mq ↔ PCI affinity bridge (T.1)
Documentation/PCI/msi-howto.rst ★, pci-error-recovery.rst ★★, pci-iov-howto.rst
Documentation/driver-api/pci/p2pdma.rst
Documentation/PCI/pcieaer-howto.rst
```

### 1.2 The MSI IRQ domain hierarchy

MSI is implemented as a **hierarchical `irq_domain`** (Ch. 17 T.6):

```
   PCI-MSI domain          (per-device: allocates entries in the MSI-X table)
        └─▶ IR-PCI-MSI     (interrupt remapping, if the IOMMU provides it)
              └─▶ VECTOR   (x86: allocates a CPU vector via the matrix allocator)
                    └─▶ APIC
```
Each level's `irq_chip` forwards mask/unmask/affinity to its parent
(`irq_chip_mask_parent()`). Reading this is the best concrete example of why the hierarchy
exists — the same `pci_alloc_irq_vectors()` call works on x86 with interrupt remapping, on
ARM with an ITS, and on a system with neither.

```bash
sudo cat /sys/kernel/debug/irq/domains/default
sudo ls /sys/kernel/debug/irq/domains/
sudo cat /sys/kernel/debug/irq/irqs/$IRQ     # shows the full domain chain
```

---

## 2. Practice

### Lab 38.1 — Multi-queue MSI-X with managed affinity (T.1)

```c
// SPDX-License-Identifier: GPL-2.0
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/interrupt.h>
#include <linux/module.h>
#include <linux/pci.h>

struct q_ctx {
	struct mq_dev *dev;
	int            idx;
	u64            irqs;
} ____cacheline_aligned;                      /* ★ Ch. 16 T.2: no false sharing */

struct mq_dev {
	struct pci_dev *pdev;
	void __iomem   *mmio;
	int             nvec;
	struct q_ctx   *q;
};

static irqreturn_t mq_isr(int irq, void *data)
{
	struct q_ctx *q = data;

	q->irqs++;                            /* per-queue, per-CPU: no shared line */
	return IRQ_HANDLED;
}

static int mq_probe(struct pci_dev *pdev, const struct pci_device_id *id)
{
	struct irq_affinity affd = { .pre_vectors = 1 };   /* vector 0 = admin */
	struct mq_dev *d;
	int i, ret;

	d = devm_kzalloc(&pdev->dev, sizeof(*d), GFP_KERNEL);
	if (!d)
		return -ENOMEM;
	d->pdev = pdev;
	pci_set_drvdata(pdev, d);

	ret = pcim_enable_device(pdev);
	if (ret)
		return ret;
	d->mmio = pcim_iomap_region(pdev, 0, KBUILD_MODNAME);
	if (IS_ERR(d->mmio))
		return PTR_ERR(d->mmio);
	pci_set_master(pdev);

	/* ★ Ask for one vector per CPU, spread by the core */
	ret = pci_alloc_irq_vectors_affinity(pdev, 2, num_online_cpus() + 1,
					     PCI_IRQ_MSIX | PCI_IRQ_AFFINITY, &affd);
	if (ret < 0)
		return dev_err_probe(&pdev->dev, ret, "msix\n");
	d->nvec = ret;
	dev_info(&pdev->dev, "got %d MSI-X vectors\n", d->nvec);

	d->q = devm_kcalloc(&pdev->dev, d->nvec, sizeof(*d->q), GFP_KERNEL);
	if (!d->q)
		return -ENOMEM;

	for (i = 0; i < d->nvec; i++) {
		const struct cpumask *m;
		char name[32];

		d->q[i].dev = d;
		d->q[i].idx = i;
		snprintf(name, sizeof(name), "%s-q%d", KBUILD_MODNAME, i);
		ret = devm_request_irq(&pdev->dev, pci_irq_vector(pdev, i),
				       mq_isr, 0, name, &d->q[i]);
		if (ret)
			return ret;

		m = pci_irq_get_affinity(pdev, i);          /* ★ where did it land? */
		dev_info(&pdev->dev, "vector %d (irq %d) -> cpus %*pbl\n",
			 i, pci_irq_vector(pdev, i), cpumask_pr_args(m));
	}
	return 0;
}
```
```bash
sudo insmod mqdemo.ko && dmesg | tail -20
grep mqdemo /proc/interrupts
for i in $(grep mqdemo /proc/interrupts | awk '{gsub(":","",$1);print $1}'); do
  echo "irq $i managed=$(cat /proc/irq/$i/smp_affinity_list)"
  echo 0 | sudo tee /proc/irq/$i/smp_affinity_list   # ★ -EIO: it's managed
done
$EDITOR kernel/irq/affinity.c        # irq_create_affinity_masks(): the spreading algorithm
```

### Lab 38.2 — Inject an AER error and watch recovery (T.2)

```bash
./scripts/config -e PCIEAER -e PCIEAER_INJECT -e PCIE_DPC
make -j$(nproc) && boot

sudo modprobe aer-inject
sudo apt install -y pciutils
git clone https://github.com/intel/aer-inject && cd aer-inject && make

cat > /tmp/err.aer <<'EOF'
AER
DOMAIN 0000
BUS 3
DEV 0
FN 0
UNCOR_STATUS   COMP_TIME
HEADER_LOG     0 1 2 3
EOF
sudo ./aer-inject /tmp/err.aer

dmesg | grep -A20 -iE 'AER|PCIe Bus Error|Recovery'
# Expect the full T.2 sequence:
#   pcieport: AER: Uncorrected (Non-Fatal) error received
#   nvme: AER: ... error_detected
#   pcieport: AER: Root Port link has been reset
#   nvme: AER: slot_reset
#   nvme: AER: resume
#   pcieport: AER: device recovery successful

# Counters:
cat /sys/bus/pci/devices/0000:03:00.0/aer_dev_correctable
cat /sys/bus/pci/devices/0000:03:00.0/aer_dev_nonfatal
cat /sys/bus/pci/devices/0000:03:00.0/aer_dev_fatal
cat /sys/bus/pci/devices/0000:00:1c.0/aer_rootport_total_err_cor

# DPC:
sudo lspci -vv | grep -A5 'Downstream Port Containment'
dmesg | grep -i dpc
```
**Then implement `pci_error_handlers` in your Lab 37.4 `edu` driver** and inject an error at
it. Verify each callback fires in order, and that your `error_detected()` does not touch
MMIO when the channel is frozen.

### Lab 38.3 — Enable SR-IOV and pass a VF to a VM (T.3)

```bash
# 1. Prerequisites
dmesg | grep -iE 'DMAR|IOMMU|AMD-Vi'                # IOMMU must be on
PF=0000:03:00.0
cat /sys/bus/pci/devices/$PF/sriov_totalvfs

# 2. Create VFs
echo 4 | sudo tee /sys/bus/pci/devices/$PF/sriov_numvfs
lspci | grep -i virtual
ls -l /sys/bus/pci/devices/$PF/virtfn*

# 3. Each VF is independently isolated (Ch. 36 T.2)
for v in /sys/bus/pci/devices/$PF/virtfn*; do
  VF=$(basename $(readlink $v))
  echo "$VF → iommu group $(basename $(readlink /sys/bus/pci/devices/$VF/iommu_group))"
done

# 4. Configure a VF from the PF (networking)
ip link show dev enp3s0
sudo ip link set enp3s0 vf 0 mac 52:54:00:aa:bb:cc vlan 100 spoofchk on trust off
sudo ip link set enp3s0 vf 0 state enable

# 5. Bind VF0 to vfio-pci and pass it through
VF0=$(basename $(readlink /sys/bus/pci/devices/$PF/virtfn0))
echo $VF0 | sudo tee /sys/bus/pci/devices/$VF0/driver/unbind
echo vfio-pci | sudo tee /sys/bus/pci/devices/$VF0/driver_override
echo $VF0 | sudo tee /sys/bus/pci/drivers_probe

qemu-system-x86_64 -enable-kvm -m 4G -cpu host \
  -device vfio-pci,host=$VF0 \
  -drive file=guest.qcow2,if=virtio -nographic

# 6. Tear down
echo 0 | sudo tee /sys/bus/pci/devices/$PF/sriov_numvfs
```
**Measure the difference:** run `iperf3` in the guest over (a) virtio-net, (b) the passed-through
VF. Then explain the result in terms of VM exits (`perf kvm stat`).

### Lab 38.4 — Measure the ASPM latency tax (T.5)

```bash
# Current state
sudo lspci -vv -s 03:00.0 | grep -E 'LnkCap:|LnkCtl:|ASPM'
cat /sys/module/pcie_aspm/parameters/policy

bench() {
  sudo fio --name=lat --filename=/dev/nvme0n1 --rw=randread --bs=4k \
           --iodepth=1 --ioengine=io_uring --direct=1 --runtime=20 --time_based \
           2>&1 | grep -E 'lat \(usec\).*(min|avg|99.00)'
}

for p in performance powersave default powersupersave; do
  echo "=== policy=$p"
  echo $p | sudo tee /sys/module/pcie_aspm/parameters/policy >/dev/null
  sleep 2
  bench
done

# Per-device L1 substates
cat /sys/bus/pci/devices/0000:03:00.0/link/l1_aspm 2>/dev/null
echo 0 | sudo tee /sys/bus/pci/devices/0000:03:00.0/link/l1_aspm

# Also check NVMe's own APST (autonomous power state transitions):
sudo nvme id-ctrl /dev/nvme0 | grep -i -A8 'ps    '
cat /sys/class/nvme/nvme0/power/pm_qos_latency_tolerance_us 2>/dev/null

# Power side:
sudo turbostat --show PkgWatt --interval 5
```
**Produce a table of p99 latency and package power for each policy.** That table is the
argument you take to a platform team.

### Lab 38.5 — Hotplug and surprise removal (T.4)

```bash
# Slot-based (if your machine has hotplug-capable slots)
ls /sys/bus/pci/slots/
cat /sys/bus/pci/slots/*/address
echo 0 | sudo tee /sys/bus/pci/slots/1/power      # power off = simulated removal
dmesg | tail -20
echo 1 | sudo tee /sys/bus/pci/slots/1/power

# Software removal + rescan (works anywhere)
D=0000:03:00.0
echo 1 | sudo tee /sys/bus/pci/devices/$D/remove
lspci | grep 03:00 || echo "gone"
echo 1 | sudo tee /sys/bus/pci/rescan
lspci | grep 03:00

# ★ The hard case: remove WHILE I/O is in flight
sudo fio --name=t --filename=/dev/nvme0n1 --rw=randread --bs=4k --iodepth=64 \
         --ioengine=libaio --runtime=60 --time_based &
sleep 5
echo 1 | sudo tee /sys/bus/pci/devices/$D/remove
dmesg | tail -40        # every in-flight request must complete with an error

# In QEMU, real hotplug:
# (monitor) device_del mynvme
# (monitor) device_add nvme,drive=d0,id=mynvme,bus=pcie.0
```
Then audit your own driver: does every MMIO read check for `~0`? Does every wait have a
timeout? Add `pci_device_is_present()` checks and re-test.

### Lab 38.6 — Walk the MSI IRQ domain hierarchy

```bash
sudo ls /sys/kernel/debug/irq/domains/
sudo cat /sys/kernel/debug/irq/domains/default | head -30
IRQ=$(grep nvme /proc/interrupts | head -1 | awk '{gsub(":","",$1);print $1}')
sudo cat /sys/kernel/debug/irq/irqs/$IRQ
# Shows: handler, chip, domain chain, hwirq, affinity, effective affinity

# The MSI-X table itself lives in a BAR:
sudo lspci -vv -s 03:00.0 | grep -A3 'MSI-X:'
# "MSI-X: Enable+ Count=32 Masked-  Table: BAR=0 offset=00002000"

# On ARM with a GIC ITS:
sudo cat /sys/kernel/debug/irq/domains/* | grep -i its
```

### Lab 38.7 — P2PDMA feasibility check (T.6)

```bash
# Does your platform allow it?
$EDITOR drivers/pci/p2pdma.c           # look at calc_map_type_and_dist()
ls /sys/bus/pci/devices/*/p2pmem/ 2>/dev/null

# NVMe CMB (controller memory buffer) is a common provider:
sudo nvme id-ctrl /dev/nvme0 | grep -i cmb
cat /sys/class/nvme/nvme0/device/p2pmem/size 2>/dev/null

# ACS is the blocker (Ch. 36 T.2):
sudo lspci -vvv | grep -B10 'ACSCtl' | grep -E '^[0-9a-f]{2}:|ACSCtl'

# Read the constraints:
$EDITOR Documentation/driver-api/pci/p2pdma.rst
grep -rn 'pci_p2pdma_distance\|P2PDMA_MAP_' drivers/pci/p2pdma.c | head
```
Write a short analysis: for your machine, between which pairs of devices would P2PDMA be
permitted, and what isolation would you be giving up?

---

## 3. Mastery drills

1. **Read `Documentation/PCI/pci-error-recovery.rst`** completely, then
   `drivers/pci/pcie/err.c`. Draw the recovery state machine including the "most severe
   answer wins" aggregation across drivers.

2. **Implement error handlers.** Add a complete `pci_error_handlers` to the Ch. 37 `edu`
   driver. Inject errors and verify each callback. Confirm you never touch MMIO while frozen.

3. **The affinity algorithm.** Read `kernel/irq/affinity.c`'s
   `irq_create_affinity_masks()`. Explain how it spreads vectors across NUMA nodes and CPUs,
   what `pre_vectors`/`post_vectors` are for, and what happens with fewer vectors than CPUs.

4. **Managed IRQ + hotplug.** Read `kernel/irq/cpuhotplug.c` and `blk-mq`'s CPU-hotplug
   handling. Explain what happens to an in-flight request on a queue whose CPU is going
   offline. Why must the IRQ be *shut down* rather than migrated?

5. **MSI vs MSI-X.** Explain why MSI requires a power-of-two contiguous vector allocation and
   MSI-X does not. What hardware difference causes it? What does that imply for a device that
   needs 5 vectors?

6. **SR-IOV BAR space.** Compute the MMIO required for 64 VFs of a device whose VF BAR0 is
   16 MiB. Explain why enabling SR-IOV can fail on a machine with a 256 MiB PCI window, and
   what "Above 4G Decoding" changes.

7. **ASPM economics.** For a laptop NVMe: estimate the energy saved per hour by L1.2 and the
   latency cost per I/O. At what I/O rate does the trade become unfavourable? Then measure.

8. **DPC vs AER.** Explain what DPC does that AER cannot, and why it matters for NVMe
   surprise removal specifically. Read `drivers/pci/pcie/dpc.c`.

9. **P2PDMA vs isolation.** Write 400 words for a security review explaining exactly what
   isolation is lost when ACS is disabled to enable P2PDMA, and under what threat model that
   is acceptable.

10. **Design question.** A 200 Gb/s NIC on a 128-core, 2-socket server. Design the interrupt
    and queue strategy: how many vectors, how spread, what `pre_vectors`, how do RSS, XPS,
    and `blk-mq`-style CPU mapping interact? What do you do when a CPU is offlined? What is
    your ASPM policy and why?

11. **Read a production driver's PCIe handling.** `drivers/nvme/host/pci.c`: find the vector
    allocation, the affinity mapping to `blk-mq`, the error handlers, the surprise-removal
    checks, and the shutdown path. Write a one-page summary of how it uses everything in this
    chapter.

---

## 4. Further reading

**Specifications:**
- PCI Express Base Specification: §6.1 (MSI/MSI-X), §6.2 (error reporting/AER),
  §6.12 (ACS), §6.18 (DPC), §6.20 (ATS), §5 (power management/ASPM), §9 (SR-IOV)
- PCI-SIG SR-IOV specification
- CXL specification (for T.7's last row)

**Kernel documentation:**
- `Documentation/PCI/msi-howto.rst` ★★
- `Documentation/PCI/pci-error-recovery.rst` ★★
- `Documentation/PCI/pcieaer-howto.rst` ★
- `Documentation/PCI/pci-iov-howto.rst`
- `Documentation/driver-api/pci/p2pdma.rst`
- `Documentation/power/pci.rst` — PCI device power management
- `Documentation/ABI/testing/sysfs-bus-pci` — every attribute in T.3/T.4/T.5's labs

**Source:**
- `drivers/pci/msi/` ★, `drivers/pci/pcie/aer.c` ★★, `err.c` ★, `aspm.c`, `dpc.c`
- `drivers/pci/iov.c`, `drivers/pci/p2pdma.c`
- `kernel/irq/affinity.c` ★, `block/blk-mq-pci.c`
- `drivers/nvme/host/pci.c` ★★ — the best single example of all of it together
- `drivers/net/ethernet/mellanox/mlx5/core/` — SR-IOV, SF/SIOV, devlink

**LWN:**
- "Interrupt affinity and multiqueue devices"
- "PCI error recovery" / "Improving PCIe error handling"
- "Peer-to-peer DMA" (Logan Gunthorpe) ★
- "SR-IOV and the kernel" / "Subfunctions and scalable IOV"
- "ASPM, latency, and power"
- "CXL: the next step for memory"

→ Next: [39-usb.md](39-usb.md)

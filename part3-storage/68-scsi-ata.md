# Chapter 68 — SCSI and ATA: libata, SAS, the SCSI midlayer, and UFS

> **Goal:** Understand the protocol layer beneath the block layer — the command sets, transports, and error-handling machinery that actually move data to physical devices. Understand why SCSI's three-layer architecture (upper/mid/lower) exists and what each layer owns, the CDB as a command encoding, SCSI's error hierarchy from sense keys to the error handler's escalation ladder, why ATA is a different protocol and how libata presents it as SCSI anyway, NCQ and its failure modes, SAS as SCSI over a switched fabric, multipath as a SCSI-level concern, and UFS as the mobile descendant. By the end you can read `drivers/scsi/` and `drivers/ata/`, interpret a sense-key dump, and diagnose a device that is failing slowly rather than cleanly.

---

## Theory & First Principles

### T.0 — Start here: why your SATA disk is called `sda`

```bash
ls /dev/sd*            # sda, sdb -- 's' for SCSI
lsscsi                 # your SATA drive appears as a SCSI device
sudo sg_inq /dev/sda   # a SCSI INQUIRY command... to a SATA disk
```

Your drive does not speak SCSI. The kernel wraps it in a **SCSI translation layer** (`libata`
plus SAT) so that every command goes through `struct scsi_cmnd`. Why would anyone emulate a
1981 parallel-bus protocol over a serial link designed in 2003?

**Because SCSI stopped being a bus and became a *command set*, and that turned out to be the
valuable part.**

```
  +-----------------------------------------------------+
  |  upper layers:  sd (disk)  sr (cdrom)  st (tape)     |
  +-----------------------------------------------------+
  |  SCSI MID-LAYER: queueing, retries, command timeouts,|   <- THE WAIST
  |  structured sense data, error-recovery escalation    |      (a command set,
  +-----------------------------------------------------+       not a wire)
  |  low-level drivers:                                  |
  |    libata (SATA)   mpt3sas (SAS)   iSCSI   FC   USB  |
  |    virtio-scsi     SRP             UAS           ... |
  +-----------------------------------------------------+
```

**One command set over a dozen physical transports** — parallel cable, serial cable, fibre,
Ethernet (iSCSI), USB, a hypervisor. Separating *what to ask the device* from *how the bytes
get there* is why a storage array 3,000 km away looks exactly like a local disk. Same
hourglass as Ch. 53 §T.0, one layer down.

**The second thing SCSI brought, which the ATA lineage lacked for two decades:**

| | ATA (1986 lineage) | SCSI (1981 lineage) |
|---|---|---|
| Origin | cheap PC disks, one device per cable | servers, many devices per bus |
| Command queueing | none until NCQ (2003), 32 deep | TCQ from the start, 256+ |
| Multiple initiators | no | yes |
| Error reporting | a status byte | **sense data**: key, ASC, ASCQ — structured |
| Addressing | one device | LUNs, targets, expanders, thousands |

**Sense data is the underrated part.** SCSI does not say "it failed"; it says *Medium Error,
unrecovered read error, at LBA X* — a structured three-field code that lets the mid-layer
decide **programmatically** whether to retry, to fail the request, or to escalate. Which is
why the SCSI error handler can run a real ladder:

```
  command times out
      -> abort the command
      -> device reset
      -> target reset
      -> bus reset
      -> host reset
      -> take the device offline
```

Each rung is more disruptive and more likely to work. **Designing errors as structured data
rather than as a boolean is what makes automated recovery possible at all** — a lesson that
applies far outside storage (compare `ERR_PTR` conventions and netlink extended acks,
Ch. 24).

**And now the point of this chapter's placement, which Ch. 69 pays off.** All of this
sophistication was designed for a device that took **10 ms** to answer. The mid-layer's
tagging, per-command state, translation, and locking cost a few microseconds per command —
invisible against 10 ms, dominant against **80 µs**. NVMe threw the whole stack away. The
interesting question to carry forward: of the four things SCSI provided — transport
independence, deep queueing, structured errors, multi-initiator — which did NVMe keep, and
which did it have to rebuild? (NVMe-oF is, in effect, the rediscovery of the first one.)

```bash
lsscsi -t                             # devices and their transports
sudo sg_inq /dev/sda && sudo sg_logs --all /dev/sda
sudo smartctl -a /dev/sda             # the device's own error log
dmesg | grep -i 'sense\|ata[0-9]'     # sense keys and ATA errors in the wild
cat /sys/block/sda/device/queue_depth
```

---

### T.1 Two lineages

Storage protocols descend from two traditions that solved different problems and met in the middle:

| | **SCSI** (1986) | **ATA/IDE** (1986) |
|---|---|---|
| Origin | SASI, for minicomputers | integrate the controller onto the drive |
| Target market | servers, arrays | consumer PCs |
| Model | **message-passing**: a command is a packet | **register-based**: write to I/O ports |
| Devices per bus | 8–16, addressable | 2 (master/slave) |
| Command queueing | from the start | added later (TCQ, then NCQ) |
| Error reporting | **rich**: sense keys, ASC/ASCQ | a status byte and an error byte |
| Cost | high | low |

SCSI won architecturally; ATA won commercially. The resolution is that **everything is SCSI now, at the command-set level**, even when the transport is not:

```
SCSI command set
   |
   +-- parallel SCSI      (dead)
   +-- SAS                (enterprise)
   +-- Fibre Channel      (SAN)
   +-- iSCSI              (SCSI over TCP)
   +-- USB Mass Storage / UAS   (SCSI over USB)
   +-- SRP / iSER         (SCSI over RDMA)
   +-- ATA via libata     (translated)
   +-- UFS                (SCSI over a mobile link)
```

NVMe (Ch. 69) is the first widely-deployed break from this, and it broke from it deliberately — SCSI's command set carries thirty years of assumptions about rotating media and slow buses.

**Why SCSI persists:** the command set is rich, extensible, well-specified, and every operating system already speaks it. A new transport that presents SCSI commands gets a complete software stack for free. That is Ch. 24's narrow-waist argument operating at the protocol level.

### T.2 The three-layer architecture

```
   ┌──────────────────────────────────────────────┐
   │ UPPER: sd, sr, st, ses, sg                   │  device type drivers
   │  - present a block device / char device      │
   │  - build CDBs for their device class         │
   ├──────────────────────────────────────────────┤
   │ MID: scsi_lib, scsi_error, scsi_scan, ...    │  the midlayer
   │  - queueing, tags, blk-mq integration        │
   │  - error handling and recovery escalation    │
   │  - device scanning and lifecycle             │
   │  - command timeouts                          │
   ├──────────────────────────────────────────────┤
   │ LOWER: HBA drivers (ahci, mpt3sas, lpfc, ..) │  transport drivers
   │  - take a scsi_cmnd, put it on the wire      │
   │  - DMA setup, interrupt handling             │
   └──────────────────────────────────────────────┘
```

The split is a genuine achievement: **a new HBA driver needs to implement only `queuecommand()` and an interrupt handler**, and it gets scanning, error handling, timeouts, queueing, and every upper-layer device type for free. A new device *type* (a new kind of SCSI peripheral) needs only an upper driver.

What each layer owns:

| Layer | Owns |
|---|---|
| Upper | the device's semantics: what commands mean for *this* device class |
| Mid | **policy**: retries, timeouts, error escalation, queue depth |
| Lower | **mechanism**: how bytes reach the device |

The addressing model is `H:C:T:L`:

```
Host    : the HBA (scsi0, scsi1, ...)
Channel : a bus on that HBA
Target  : a device on that bus
LUN     : a logical unit within the target
```

Visible everywhere:

```sh
lsscsi
ls /sys/class/scsi_device/
cat /sys/block/sda/device/{vendor,model,rev}
```

A disk array presents one target with many LUNs; a SAS expander presents many targets. The model is from an era of shared buses and has survived because it maps onto everything.

### T.3 The CDB

A SCSI command is a **Command Descriptor Block**: a 6, 10, 12, or 16-byte packet.

```
READ(10):
byte 0:  0x28            opcode
byte 1:  flags (DPO, FUA, ...)
bytes 2-5: LBA (32-bit, big-endian)
byte 6:  group number
bytes 7-8: transfer length (16-bit, in blocks)
byte 9:  control

READ(16):
byte 0:  0x88
bytes 2-9:  LBA (64-bit)
bytes 10-13: transfer length (32-bit)
```

The lengths exist for historical capacity reasons: `READ(6)` has a 21-bit LBA (1 GB with 512-byte blocks), `READ(10)` has 32 bits (2 TB), `READ(16)` has 64 bits. `sd` chooses based on the device's capacity — which is why `READ CAPACITY(16)` matters: a device larger than 2 TB must report its size with the 16-byte form, and a host that only issues `READ CAPACITY(10)` sees `0xFFFFFFFF` and must know to retry.

Commands worth recognising:

| Opcode | Command | Purpose |
|---|---|---|
| `0x00` | TEST UNIT READY | is the device responding? |
| `0x03` | REQUEST SENSE | fetch error detail (§T.4) |
| `0x12` | INQUIRY | identity, VPD pages |
| `0x1A`/`0x5A` | MODE SENSE(6/10) | configuration pages — **including the cache page** |
| `0x25` | READ CAPACITY(10) | size |
| `0x28`/`0x88` | READ(10/16) | |
| `0x2A`/`0x8A` | WRITE(10/16) | |
| `0x35` | SYNCHRONIZE CACHE(10) | **the flush of Ch. 61 §T.3** |
| `0x42` | UNMAP | discard |
| `0x93` | WRITE SAME(16) | write-zeroes / discard |
| `0x9E` | SERVICE ACTION IN(16) | READ CAPACITY(16), GET LBA STATUS |
| `0xA0` | REPORT LUNS | enumeration |
| `0x4D` | LOG SENSE | statistics, self-test results |
| `0x5E`/`0x5F` | PERSISTENT RESERVE IN/OUT | cluster fencing |
| `0xA1` | ATA PASS-THROUGH(12) | §T.6 |
| `0x85` | ATA PASS-THROUGH(16) | §T.6 |

**VPD (Vital Product Data) pages** are the extensibility mechanism: `INQUIRY` with `EVPD=1` and a page code returns structured data. Page `0x83` (Device Identification) carries the WWN, which is how multipath knows two paths lead to the same device. Page `0xB0`/`0xB1`/`0xB2` carry block limits, rotation rate, and thin-provisioning support — which is where `/sys/block/sdX/queue/rotational` and the discard limits come from.

### T.4 Error handling: the part that matters

SCSI's error model is genuinely rich, and understanding it is the difference between "the disk is broken" and an actual diagnosis.

**The sense data hierarchy:**

```
status byte (CHECK CONDITION = 0x02)
     |
     v
sense key (4 bits) -- the CATEGORY
     |
     v
ASC / ASCQ (2 bytes) -- the SPECIFIC condition
```

Sense keys:

| Key | Name | Meaning |
|---|---|---|
| 0x0 | NO SENSE | no error, or a non-error condition |
| 0x1 | RECOVERED ERROR | **succeeded, but with retries/ECC** — a warning |
| 0x2 | NOT READY | spinning up, no media, becoming ready |
| 0x3 | MEDIUM ERROR | **the media is bad** — this is a URE |
| 0x4 | HARDWARE ERROR | the drive electronics failed |
| 0x5 | ILLEGAL REQUEST | bad CDB, unsupported command, bad LBA |
| 0x6 | UNIT ATTENTION | **state changed**: reset, media changed, capacity changed |
| 0x7 | DATA PROTECT | write-protected |
| 0xB | ABORTED COMMAND | the transport aborted it; usually retryable |
| 0xD | VOLUME OVERFLOW | |
| 0xE | MISCOMPARE | verify failed |

**ASC/ASCQ is where the diagnosis lives.** Some worth knowing:

| ASC/ASCQ | Meaning |
|---|---|
| `04/01` | becoming ready (spinning up) — retry |
| `04/02` | needs START UNIT |
| `11/00` | unrecovered read error — **a bad sector** |
| `1C/00` | defect list not found |
| `29/00` | power-on or reset occurred |
| `2A/00` | parameters changed |
| `28/00` | **media changed** |
| `3F/0E` | reported LUNs data has changed |
| `5D/xx` | **failure prediction threshold exceeded (SMART)** |
| `0B/01` | warning: temperature exceeded |

`UNIT ATTENTION` (key 0x6) deserves special attention because it is not an error: it means "something about my state changed since you last talked to me, and I am telling you once." A device reports it after a reset, after media change, or after a capacity change. The midlayer must consume it and retry, and a driver that treats it as a failure produces spurious I/O errors after every bus reset.

**The error handler (`scsi_error.c`) escalates**, and the ladder is the important part:

```
1. Command times out or returns an error
2. scsi_eh wakes up (a per-host kernel thread)
3. For each failed command:
   a. RETRY the command (up to sd's retry count)
   b. TEST UNIT READY / REQUEST SENSE to learn the state
   c. ABORT the specific command (task abort)
   d. DEVICE RESET (LUN reset)
   e. TARGET RESET
   f. BUS RESET
   g. HOST RESET
   h. OFFLINE THE DEVICE
```

Each step is more disruptive than the last. A LUN reset kills all commands to one device; a bus reset kills every device on the bus. **This is why one failing disk can stall an entire enclosure**: the error handler escalates, resets the bus, and every device's in-flight commands are aborted and retried.

The pathology worth understanding: a disk that is *slow* rather than *dead* is the worst case. It times out (default 30 s for `sd`), the EH escalates through the ladder taking tens of seconds at each step, and meanwhile the host's queue to that device — and possibly the whole bus — is frozen. Total stall times of minutes are routine.

Mitigations:

```sh
cat /sys/block/sda/device/timeout          # 30 seconds
cat /sys/block/sda/device/eh_timeout       # 10 seconds (abort timeout)
echo 5 > /sys/block/sda/device/timeout     # fail faster

# For multipath: fail fast and let the other path take over
echo 5 > /sys/block/sda/device/timeout
echo 1 > /sys/block/sda/device/eh_timeout
```

`REQ_FAILFAST_DEV`/`_TRANSPORT`/`_DRIVER` (Ch. 63 §T.3) exist for exactly this: multipath sets them so a failing path errors immediately rather than being retried for minutes.

### T.5 Queueing and NCQ

**Tagged command queueing** lets a host have multiple commands outstanding, which the device may reorder. This is where the block layer's tags (Ch. 63 §T.6) meet the protocol.

| | SCSI TCQ | ATA NCQ |
|---|---|---|
| Max queue depth | 256+ (protocol allows more) | **32** (5-bit tag) |
| Ordering control | Simple / Ordered / Head-of-queue | none — the device reorders freely |
| Error granularity | per-command | **the whole queue is aborted** |

That last row is the important difference. On ATA NCQ, an error aborts **every** outstanding command, and the host must re-issue them. Combined with a 32-command limit, this makes NCQ error handling much cruder than SCSI's.

Queue depth is negotiated and adaptive:

```sh
cat /sys/block/sda/device/queue_depth
echo 16 > /sys/block/sda/device/queue_depth
cat /sys/block/sda/device/queue_type     # simple / ordered / none
```

The midlayer **reduces** the depth automatically when a device returns `TASK SET FULL` or `BUSY`, and slowly increases it again. This is a congestion-control loop at the protocol level, with the same shape as TCP's.

Historically, NCQ had a bad reputation because of firmware bugs: `libata` maintains a substantial blacklist (`ata_device_blacklist`) of drives with broken NCQ, broken TRIM, broken FUA, or broken LPM. Reading that table is an education in how much vendor firmware lies:

```c
static const struct ata_blacklist_entry ata_device_blacklist[] = {
	/* Devices that do not properly handle queued TRIM commands */
	{ "Micron_M500IT_*",	"MU01",	ATA_HORKAGE_NO_NCQ_TRIM | ... },
	{ "Crucial_CT*MX100*",	"MU01",	ATA_HORKAGE_NO_NCQ_TRIM | ... },
	{ "Samsung SSD 8*",	NULL,	ATA_HORKAGE_NO_NCQ_TRIM | ... },
	/* devices that don't properly handle TRIM commands */
	{ "SuperSSpeed S238*",	NULL,	ATA_HORKAGE_NOTRIM },
	/* Devices that report their write cache but lie about FUA */
	{ "Maxtor 4D080H4",	"MAXTOR 4D080H4", ATA_HORKAGE_NONCQ },
	...
};
```

**Every entry in that table is a drive that would corrupt data or hang without the workaround.** Ch. 61 §T.6's "does the device tell the truth" question, answered empirically over two decades.

### T.6 libata: ATA presented as SCSI

Linux's ATA support was rewritten around 2003 as `libata`, and the key design decision was:

> **Translate ATA into SCSI, so ATA devices use the entire SCSI stack.**

So an ATA disk appears as `/dev/sda` with a SCSI device node, `sd` as its upper driver, and the SCSI midlayer's error handling and queueing. `libata` sits in the lower-driver position and translates:

| SCSI command | ATA translation |
|---|---|
| `INQUIRY` | synthesised from `IDENTIFY DEVICE` |
| `READ(10)/(16)` | `READ DMA EXT` / `READ FPDMA QUEUED` (NCQ) |
| `WRITE(10)/(16)` | `WRITE DMA EXT` / `WRITE FPDMA QUEUED` |
| `SYNCHRONIZE CACHE` | `FLUSH CACHE EXT` |
| `MODE SENSE` (caching page) | synthesised from `IDENTIFY` |
| `UNMAP` / `WRITE SAME` | `DATA SET MANAGEMENT` (TRIM) |
| `START STOP UNIT` | `STANDBY IMMEDIATE` / `IDLE IMMEDIATE` |
| `REPORT LUNS` | synthesised (always LUN 0) |
| `ATA PASS-THROUGH(12/16)` | **passed through unmodified** |

The cost is a translation layer and some impedance mismatch — ATA's error reporting is much poorer, so `libata` synthesises sense data from ATA's status/error registers. The benefit is enormous: one error handler, one queueing implementation, one set of upper drivers, one tooling ecosystem.

`ATA PASS-THROUGH` is the escape hatch, and it is how `smartctl` and `hdparm` work: they wrap an ATA command in a SCSI CDB, the midlayer routes it to `libata`, and `libata` unwraps and issues it. **This is a general and useful pattern: when you translate a protocol, provide a pass-through for the untranslatable.**

`libata` has its own error handler (`libata-eh.c`) which runs *under* the SCSI EH, handling ATA-specific recovery: link resets, speed renegotiation, NCQ error recovery (read the log page to find which command failed, since the status only says "something failed"), and hotplug.

**Link power management (LPM)** is a common source of problems:

```sh
cat /sys/class/scsi_host/host0/link_power_management_policy
# max_performance / medium_power / med_power_with_dipm / min_power / min_power_with_partial
```

Aggressive LPM saves meaningful laptop battery but triggers firmware bugs in some drives and controllers, producing link resets and I/O errors. `med_power_with_dipm` is the usual compromise.

### T.7 SAS: SCSI over a switched fabric

SAS replaced parallel SCSI with point-to-point serial links and switches (expanders):

```
HBA ──┬── expander ──┬── SAS drive
      │              ├── SAS drive
      │              └── expander ──┬── SATA drive (via STP)
      │                             └── SAS drive
      └── SAS drive (direct)
```

Key properties:

| Property | Note |
|---|---|
| **Point-to-point** | no shared bus; a failing device does not disturb others electrically |
| **Expanders** | switches; thousands of devices per HBA |
| **Dual-port** | SAS drives have two ports for redundant paths (§T.8) |
| **Full duplex** | read and write simultaneously |
| **SATA tunnelling (STP)** | SATA drives work in SAS enclosures |
| **Wide ports** | several PHYs bonded for bandwidth |

The SAS address (a 64-bit WWN) is the device's permanent identity, independent of where it is plugged in — which is what makes persistent naming and multipath work.

`SATA drives in SAS enclosures` is worth a note: it works via STP, but SATA has no dual-port capability, so such drives cannot be multipathed, and the expander must arbitrate. The performance and error behaviour is measurably worse than a native SAS drive. Mixing them in one enclosure is common and often regrettable.

The SES (SCSI Enclosure Services) device is how enclosure management works — slot LEDs, temperature, fan status:

```sh
lsscsi -g | grep enclosu
sg_ses /dev/sg5
ledctl locate=/dev/sda     # light the slot LED
```

### T.8 Multipath

If a device is reachable by more than one path (dual-port SAS, dual-fabric FC, multiple iSCSI sessions), the kernel sees it as **several block devices that are actually the same LUN**.

Identifying them as the same device uses the VPD page 0x83 identifier:

```sh
/lib/udev/scsi_id --page=0x83 --whitelisted --device=/dev/sda
sg_inq --page=0x83 /dev/sda
```

`multipath-tools` then builds a `dm-multipath` device (Ch. 66) over the paths:

```sh
multipath -ll
```

```
mpatha (3600508b400105e210000900000490000) dm-0 VENDOR,MODEL
size=100G features='1 queue_if_no_path' hwhandler='0' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 1:0:0:0 sda 8:0  active ready running
`-+- policy='service-time 0' prio=10 status=enabled
  `- 2:0:0:0 sdb 8:16 active ready running
```

Two distinctions matter:

**Active/active versus active/passive.** Some arrays (ALUA-capable) accept I/O on all paths with different priorities; others designate one controller as the owner and require an explicit failover command to move a LUN. The `hwhandler` field names the handler that knows how to do that (`alua`, `rdac`, `emc`).

**`queue_if_no_path`.** When every path fails, multipath can either error immediately or queue indefinitely hoping a path returns. The default queues — same trade as dm-thin's `queue_if_no_space` (Ch. 66 §T.5), and the same consequence: processes hang in `D` state. `no_path_retry` bounds it.

Multipath is why §T.4's timeout tuning matters: a path that takes 120 seconds to fail defeats the purpose of having a second path. The correct configuration is aggressive `fast_io_fail_tmo` and `dev_loss_tmo` on the transport, plus `REQ_FAILFAST` so the block layer does not retry on a dead path.

### T.9 UFS and eMMC: the mobile descendants

**UFS (Universal Flash Storage)** is what modern phones use, and it is SCSI:

```
UFS = SCSI command set (UFS Protocol Information Units)
      over MIPI UniPro (link layer)
      over M-PHY (physical layer)
```

So `drivers/ufs/` implements a SCSI host, and a UFS device appears as `/dev/sda`. Features it adds:

| Feature | Purpose |
|---|---|
| **Command queueing** (32, or 64 with MCQ) | parallelism |
| **LUNs with different properties** | boot LUN, RPMB LUN, user LUNs |
| **RPMB** (Replay Protected Memory Block) | authenticated storage for secure boot state |
| **Write Booster** | a pSLC cache region for burst writes |
| **HPB** (Host Performance Booster) | the host caches the device's L2P mapping table |
| **Turbo Write / power modes** | aggressive power management |

HPB is conceptually interesting: the device's flash translation layer has a logical-to-physical map too large to keep in the device's small DRAM, so lookups sometimes require reading it from NAND. HPB lets the *host* cache portions of that map and send the physical address with the read — trading host memory for device latency. It is the same "the host has more information and more memory" argument as ZNS (Ch. 60 §T.5), applied to a different bottleneck.

**eMMC** is the older, simpler alternative: an MMC/SD command set, not SCSI, handled by `drivers/mmc/`. It appears as `/dev/mmcblk0`, has a much smaller command queue (or none), and is being displaced by UFS in anything performance-sensitive.

The lesson: **SCSI's command set reached mobile phones** because a well-specified, extensible command set is worth reusing even when nothing else about the context matches its origins.

### T.10 Diagnosing a failing device

The characteristic difficulty is that drives fail *gradually*. The diagnostic hierarchy:

| Signal | Where | Meaning |
|---|---|---|
| `RECOVERED ERROR` in logs | `dmesg` | the drive is retrying internally — **an early warning** |
| Rising `Reallocated_Sector_Ct` | `smartctl -A` | sectors are being remapped |
| Non-zero `Current_Pending_Sector` | `smartctl -A` | **sectors that failed to read and are not yet remapped** |
| `Offline_Uncorrectable` | `smartctl -A` | found during a self-test |
| Rising `UDMA_CRC_Error_Count` | `smartctl -A` | **a cable or connector problem, not the drive** |
| SCSI `5D/xx` sense | `dmesg` | the drive's own failure prediction |
| Rising read error rate in LOG SENSE | `sg_logs` | SCSI's equivalent |
| Link resets | `dmesg`, `libata` | cabling, power, or LPM |
| Timeouts and EH escalation | `dmesg` | the drive is not responding |

Two things are worth internalising:

**`Current_Pending_Sector` is the important one.** It means a read failed and the drive cannot remap the sector until it is *written* (remapping on read would fabricate data). If that sector is in your RAID array and you need it during a rebuild, you lose a stripe (Ch. 67 §T.9). A `check` scrub finds and rewrites such sectors while redundancy still exists — which is the argument for scheduled scrubbing, stated mechanically.

**`UDMA_CRC_Error_Count` blames the cable, not the drive.** It counts transmission errors on the link. Replacing the drive does not help; reseating the cable usually does. Many drives are replaced unnecessarily because this distinction is not made.

SMART self-tests are the active diagnostic:

```sh
smartctl -t short /dev/sda      # ~2 minutes
smartctl -t long /dev/sda       # hours; reads every sector
smartctl -l selftest /dev/sda
```

A long test reads the whole surface and is the cheapest way to find latent bad sectors. Running one monthly, alongside the RAID scrub, is the standard practice.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `drivers/scsi/scsi.c`, `scsi_lib.c` ★★★ | the midlayer core, blk-mq integration |
| `drivers/scsi/scsi_error.c` ★★★ | **§T.4's escalation ladder** |
| `drivers/scsi/scsi_scan.c` | §T.2's device discovery |
| `drivers/scsi/scsi_sysfs.c` | the sysfs interface |
| `drivers/scsi/sd.c` ★★★ | the disk upper driver; CDB construction |
| `drivers/scsi/sd_zbc.c` | zoned SCSI (Ch. 60 §T.5) |
| `drivers/scsi/sr.c`, `st.c`, `ses.c`, `sg.c` | CD, tape, enclosure, generic |
| `drivers/scsi/scsi_transport_{sas,fc,iscsi,spi}.c` | transport classes |
| `drivers/scsi/device_handler/` | ALUA and vendor failover handlers |
| `drivers/ata/libata-core.c` ★★★ | §T.6's core |
| `drivers/ata/libata-scsi.c` ★★★ | **the SCSI↔ATA translation** |
| `drivers/ata/libata-eh.c` ★★★ | ATA error handling |
| `drivers/ata/libata-sata.c` | NCQ, link power management |
| `drivers/ata/ahci.c`, `libahci.c` | the standard SATA controller driver |
| `drivers/ufs/core/`, `drivers/ufs/host/` | §T.9 |
| `drivers/mmc/` | eMMC/SD |
| `include/scsi/scsi.h`, `scsi_cmnd.h`, `scsi_device.h` ★★★ | |
| `include/linux/libata.h` ★★★ | |
| `Documentation/scsi/` ★★★ | per-driver and per-topic |

### 1.2 `scsi_cmnd` and `scsi_device`

```c
struct scsi_cmnd {
	struct scsi_device *device;
	struct list_head eh_entry;
	struct delayed_work abort_work;
	struct rcu_head rcu;
	int eh_eflags;
	int budget_token;
	unsigned long jiffies_at_alloc;
	int retries;
	int allowed;                /* max retries */
	unsigned char prot_op;
	unsigned char prot_type;
	unsigned char prot_flags;
	unsigned short cmd_len;
	enum dma_data_direction sc_data_direction;

	unsigned char cmnd[32];     /* THE CDB (T.3) */

	struct scsi_data_buffer sdb;
	struct scsi_data_buffer *prot_sdb;
	unsigned underflow;
	unsigned transfersize;
	unsigned resid_len;
	unsigned sense_len;
	unsigned char *sense_buffer; /* SCSI_SENSE_BUFFERSIZE (96) */
	int flags;
	unsigned long state;
	unsigned int extra_len;
	unsigned char *host_scribble;
	int result;                  /* status + host byte + driver byte */
};

struct scsi_device {
	struct Scsi_Host *host;
	struct request_queue *request_queue;
	struct list_head siblings;
	struct list_head same_target_siblings;

	struct sbitmap budget_map;   /* the device's queue depth */
	atomic_t device_blocked;

	unsigned int id, channel;
	u64 lun;
	unsigned int manufacturer;
	unsigned sector_size;

	void *hostdata;
	unsigned char type;
	char scsi_level;
	char inq_periph_qual;
	struct mutex inquiry_mutex;
	unsigned char inquiry_len;
	unsigned char *inquiry;
	char *vendor, *model, *rev;

	unsigned int queue_depth;
	unsigned short last_queue_full_depth;
	unsigned short last_queue_full_count;
	unsigned long last_queue_full_time;
	unsigned long queue_ramp_up_period;   /* T.5's congestion control */
	unsigned long last_queue_ramp_up;

	unsigned int id_time;
	/* A LOT of quirk bits: */
	unsigned removable:1;
	unsigned changed:1;
	unsigned busy:1;
	unsigned lockable:1;
	unsigned locked:1;
	unsigned borken:1;              /* "Tell the SCSI layer to be careful" */
	unsigned disconnect:1;
	unsigned soft_reset:1;
	unsigned sdtr:1, wdtr:1, ppr:1;
	unsigned tagged_supported:1;
	unsigned simple_tags:1;
	unsigned was_reset:1;
	unsigned use_10_for_rw:1;
	unsigned use_10_for_ms:1;
	unsigned use_16_for_rw:1;
	unsigned use_16_for_sync:1;
	unsigned skip_ms_page_8:1;
	unsigned skip_ms_page_3f:1;
	unsigned skip_vpd_pages:1;
	unsigned try_vpd_pages:1;
	unsigned use_192_bytes_for_3f:1;
	unsigned no_start_on_add:1;
	unsigned allow_restart:1;
	unsigned manage_start_stop:1;
	unsigned no_uld_attach:1;
	unsigned select_no_atn:1;
	unsigned fix_capacity:1;
	unsigned guess_capacity:1;
	unsigned retry_hwerror:1;
	unsigned last_sector_bug:1;
	unsigned no_read_disc_info:1;
	unsigned no_read_capacity_16:1;
	unsigned try_rc_10_first:1;
	unsigned security_supported:1;
	unsigned is_visible:1;
	unsigned wce_default_on:1;
	unsigned no_dif:1;
	unsigned broken_fua:1;
	unsigned lun_in_cdb:1;
	unsigned unmap_limit_for_ws:1;
	unsigned rpm_autosuspend:1;
	unsigned ignore_media_change:1;
	unsigned silence_suspend:1;
	unsigned no_vpd_size:1;
	...
};
```

**That quirk block is the chapter's most honest artefact.** Roughly forty single-bit flags, each one a class of device that does something wrong. `borken`, `last_sector_bug`, `fix_capacity`, `guess_capacity`, `broken_fua`, `skip_ms_page_8` — every one commemorates devices shipped with firmware that violates the specification. This is what "support real hardware" means in practice.

### 1.3 `queuecommand`: the lower driver's contract

```c
struct scsi_host_template {
	const char *name;
	int (*queuecommand)(struct Scsi_Host *, struct scsi_cmnd *);  /* THE one */
	void (*commit_rqs)(struct Scsi_Host *, u16);
	int (*eh_abort_handler)(struct scsi_cmnd *);
	int (*eh_device_reset_handler)(struct scsi_cmnd *);
	int (*eh_target_reset_handler)(struct scsi_cmnd *);
	int (*eh_bus_reset_handler)(struct scsi_cmnd *);
	int (*eh_host_reset_handler)(struct scsi_cmnd *);
	int (*slave_alloc)(struct scsi_device *);
	int (*slave_configure)(struct scsi_device *);
	void (*slave_destroy)(struct scsi_device *);
	int (*scan_finished)(struct Scsi_Host *, unsigned long);
	void (*scan_start)(struct Scsi_Host *);
	int (*change_queue_depth)(struct scsi_device *, int);
	int (*map_queues)(struct Scsi_Host *);
	int (*mq_poll)(struct Scsi_Host *, unsigned int);
	...
	int can_queue;              /* total commands the HBA supports */
	int this_id;
	unsigned short sg_tablesize;
	unsigned int max_sectors;
	unsigned int cmd_per_lun;
	...
};
```

The `eh_*_handler` set is **§T.4's ladder, as a driver interface.** A driver implements as many as its hardware supports; the midlayer escalates through them in order.

A minimal `queuecommand`:

```c
static int my_queuecommand(struct Scsi_Host *shost, struct scsi_cmnd *cmd)
{
	struct my_hba *hba = shost_priv(shost);
	struct my_cmd *mc = scsi_cmd_priv(cmd);
	int tag = scsi_cmd_to_rq(cmd)->tag;

	/* Build the hardware command from the CDB */
	memcpy(mc->cdb, cmd->cmnd, cmd->cmd_len);
	mc->lun = cmd->device->lun;
	mc->direction = cmd->sc_data_direction;

	/* Map the scatter-gather list for DMA (Ch. 35) */
	if (scsi_dma_map(cmd) < 0)
		return SCSI_MLQUEUE_HOST_BUSY;
	my_build_sgl(mc, scsi_sglist(cmd), scsi_sg_count(cmd));

	/* Submit */
	my_post_command(hba, tag, mc);
	return 0;
}

/* From the interrupt handler: */
static void my_complete(struct my_hba *hba, int tag, u8 status, u8 *sense)
{
	struct scsi_cmnd *cmd = scsi_host_find_tag(hba->shost, tag);

	scsi_dma_unmap(cmd);
	if (status == CHECK_CONDITION)
		memcpy(cmd->sense_buffer, sense, SCSI_SENSE_BUFFERSIZE);
	cmd->result = status;
	scsi_done(cmd);          /* -> blk_mq_complete_request (Ch. 65 T.6) */
}
```

Return values mirror the block layer's (Ch. 65 §T.4): `0` accepted, `SCSI_MLQUEUE_HOST_BUSY` / `_DEVICE_BUSY` / `_TARGET_BUSY` for backpressure.

### 1.4 `sd`'s CDB construction

```c
static blk_status_t sd_setup_read_write_cmnd(struct scsi_cmnd *cmd)
{
	struct request *rq = scsi_cmd_to_rq(cmd);
	struct scsi_device *sdp = cmd->device;
	struct scsi_disk *sdkp = scsi_disk(rq->q->disk);
	sector_t lba = sectors_to_logical(sdp, blk_rq_pos(rq));
	sector_t threshold;
	unsigned int nr_blocks = sectors_to_logical(sdp, blk_rq_sectors(rq));
	bool dif, dix;
	unsigned int mask = logical_to_sectors(sdp, 1) - 1;
	bool write = rq_data_dir(rq) == WRITE;
	unsigned char protect, fua;
	blk_status_t ret;

	...
	/* Alignment check: T.3's contract */
	if ((blk_rq_pos(rq) & mask) || (blk_rq_sectors(rq) & mask)) {
		scmd_printk(KERN_ERR, cmd, "request not aligned to the logical block size\n");
		return BLK_STS_IOERR;
	}
	...
	fua = rq->cmd_flags & REQ_FUA ? 0x8 : 0;     /* Ch. 61 T.3 */
	dix = scsi_prot_sg_count(cmd);
	dif = scsi_host_dif_capable(cmd->device->host, sdkp->protection_type);
	...
	/* Choose the CDB length based on capacity and the request (T.3) */
	if (protect || blk_integrity_rq(rq)) {
		ret = sd_setup_rw32_cmnd(cmd, write, lba, nr_blocks, protect | fua);
	} else if (sdp->use_16_for_rw || (nr_blocks > 0xffff)) {
		ret = sd_setup_rw16_cmnd(cmd, write, lba, nr_blocks, protect | fua);
	} else if ((nr_blocks > 0xff) || (lba > 0x1fffff) ||
		   sdp->use_10_for_rw || protect) {
		ret = sd_setup_rw10_cmnd(cmd, write, lba, nr_blocks, protect | fua);
	} else {
		ret = sd_setup_rw6_cmnd(cmd, write, lba, nr_blocks, protect | fua);
	}
	...
	cmd->transfersize = sdp->sector_size;
	cmd->underflow = nr_blocks << 9;
	cmd->allowed = sdkp->max_retries;
	cmd->sdb.length = nr_blocks * sdp->sector_size;
	...
	return BLK_STS_OK;
}
```

And the flush:

```c
static blk_status_t sd_setup_flush_cmnd(struct scsi_cmnd *cmd)
{
	/* Ch. 61 T.3's REQ_OP_FLUSH becomes SYNCHRONIZE CACHE. */
	cmd->cmnd[0] = SYNCHRONIZE_CACHE;
	cmd->cmd_len = 10;
	cmd->transfersize = 0;
	cmd->allowed = sdkp->max_retries;
	rq->timeout = rq->q->rq_timeout * SD_FLUSH_TIMEOUT_MULTIPLIER;
	return BLK_STS_OK;
}
```

The cache mode is read from MODE SENSE page 8 and is what sets `write_cache`:

```c
static void sd_read_cache_type(struct scsi_disk *sdkp, unsigned char *buffer)
{
	...
	/* Page 8: Caching mode page. WCE = write cache enable. */
	if (modepage == 8) {
		sdkp->WCE = ((buffer[offset + 2] & 0x04) != 0);
		sdkp->RCD = ((buffer[offset + 2] & 0x01) != 0);
	}
	...
	sd_printk(KERN_NOTICE, sdkp, "Write cache: %s, read cache: %s, %s\n",
		  sdkp->WCE ? "enabled" : "disabled",
		  sdkp->RCD ? "disabled" : "enabled",
		  sdkp->DPOFUA ? "supports DPO and FUA" : "doesn't support DPO or FUA");
}
```

That log line appears in `dmesg` for every disk at boot, and it is **the answer to Ch. 61 §T.6's question for this device**.

### 1.5 The error handler

```c
static int scsi_eh_action(struct scsi_cmnd *scmd, int rtn)
{
	...
}

void scsi_eh_ready_devs(struct Scsi_Host *shost,
			struct list_head *work_q,
			struct list_head *done_q)
{
	/* T.4's ladder, in order */
	if (!scsi_eh_stu(shost, work_q, done_q))
		if (!scsi_eh_bus_device_reset(shost, work_q, done_q))
			if (!scsi_eh_target_reset(shost, work_q, done_q))
				if (!scsi_eh_bus_reset(shost, work_q, done_q))
					if (!scsi_eh_host_reset(shost, work_q, done_q))
						scsi_eh_offline_sdevs(work_q, done_q);
}

static int scsi_check_sense(struct scsi_cmnd *scmd)
{
	struct scsi_device *sdev = scmd->device;
	struct scsi_sense_hdr sshdr;

	if (! scsi_command_normalize_sense(scmd, &sshdr))
		return FAILED;
	...
	switch (sshdr.sense_key) {
	case NO_SENSE:
		return SUCCESS;
	case RECOVERED_ERROR:
		return /* SUCCESS */ 0;
	case ABORTED_COMMAND:
		if (sshdr.asc == 0x10)      /* DIF/DIX failure */
			return SUCCESS;
		return NEEDS_RETRY;
	case NOT_READY:
	case UNIT_ATTENTION:
		/* T.4: UNIT ATTENTION is informational; consume and retry. */
		if (scmd->device->expecting_cc_ua) {
			if (sshdr.asc != 0x28 || sshdr.ascq != 0x00) {
				scmd->device->expecting_cc_ua = 0;
				return NEEDS_RETRY;
			}
		}
		if (scmd->device->sdev_bflags & BLIST_RETRY_ASC_C1 &&
		    sshdr.asc == 0xc1 && sshdr.ascq == 0x01)
			return NEEDS_RETRY;
		if (sshdr.asc == 0x3f && sshdr.ascq == 0x03)
			scsi_report_lun_change(scmd->device);
		...
		if (sshdr.asc == 0x04) {
			switch (sshdr.ascq) {
			case 0x01: /* becoming ready */
			case 0x04: /* format in progress */
			...
				return NEEDS_RETRY;      /* be patient */
			default:
				return SUCCESS;
			}
		}
		return SUCCESS;
	case MEDIUM_ERROR:
		if (sshdr.asc == 0x11 ||  /* UNRECOVERED READ ERR */
		    sshdr.asc == 0x13 ||  /* AMNF DATA FIELD */
		    sshdr.asc == 0x14) {  /* RECORD NOT FOUND */
			set_hostbyte(scmd, DID_MEDIUM_ERROR);
			return SUCCESS;       /* report it; do not retry */
		}
		return NEEDS_RETRY;
	case HARDWARE_ERROR:
		if (scmd->device->retry_hwerror)
			return ADD_TO_MLQUEUE;
		return SUCCESS;
	case ILLEGAL_REQUEST:
		...
		return SUCCESS;
	default:
		return SUCCESS;
	}
}
```

Note the `MEDIUM_ERROR` handling: an unrecovered read error is **not retried** by the midlayer, because the drive has already retried internally and failed. It is reported up, and MD (Ch. 67) or the filesystem handles it. Retrying would waste the 30-second timeout for nothing.

### 1.6 Observability

| Where | What |
|---|---|
| `lsscsi -ltv` ★★★ | the H:C:T:L topology with transport detail |
| `/sys/class/scsi_device/*/device/` ★★★ | per-device: `vendor`, `model`, `state`, `timeout`, `queue_depth`, `iocounterbits`, `ioerr_cnt` |
| `/sys/class/scsi_host/*/` | per-HBA: `link_power_management_policy`, `can_queue`, `proc_name` |
| `/sys/class/sas_device/`, `sas_phy/`, `sas_expander/` | SAS topology |
| `smartctl -a`, `-A`, `-l selftest`, `-t long` ★★★ | |
| `sg_inq`, `sg_vpd`, `sg_logs`, `sg_modes`, `sg_ses`, `sg_readcap` ★★★ | |
| `sg_turs`, `sg_start`, `sg_reset`, `sg_luns` | |
| `hdparm -I`, `-W`, `-C` | ATA identity and cache state |
| `dmesg` ★★★ | sense data, EH escalation, libata resets |
| `trace-cmd record -e scsi:\*` ★★★ | `scsi_dispatch_cmd_start/done/error`, `scsi_eh_wakeup` |
| `/sys/module/scsi_mod/parameters/*` | `scan`, `default_dev_flags`, `eh_deadline` |
| `scsi_logging_level` ★★★ | fine-grained midlayer logging |

`scsi_logging_level` is the deep diagnostic:

```sh
# Enable CDB dumping for every command
sudo sg_logs --help  # or:
echo 0x1b6db6db | sudo tee /proc/sys/dev/scsi/logging_level
# Then dmesg shows every CDB. Turn it off promptly; it is very verbose.
echo 0 | sudo tee /proc/sys/dev/scsi/logging_level
```

---

## 2. Practice

### Lab 68.1 — Explore the SCSI topology

```sh
sudo apt install -y sg3-utils lsscsi smartmontools hdparm
sudo modprobe scsi_debug dev_size_mb=256 num_tgts=3 max_luns=2 vpd_use_hostno=0
lsscsi -ltv
```

```
[6:0:0:0]    disk    Linux    scsi_debug       0191  /dev/sda
  dir: /sys/bus/scsi/devices/6:0:0:0
  [6:0:0:0]  attached_to  ...
```

Read the H:C:T:L model (§T.2):

```sh
ls /sys/class/scsi_host/
ls /sys/class/scsi_device/
ls /sys/class/scsi_disk/

for d in /sys/class/scsi_device/*/device; do
  echo "=== $(basename $(dirname $d)) ==="
  for f in vendor model rev type state timeout queue_depth; do
    printf "  %-14s %s\n" $f "$(cat $d/$f 2>/dev/null)"
  done
done
```

INQUIRY and VPD pages — §T.3's identity mechanism:

```sh
DEV=$(lsscsi | grep scsi_debug | head -1 | awk '{print $NF}')
sudo sg_inq $DEV
sudo sg_inq --page=0x00 $DEV      # supported VPD pages
sudo sg_vpd --all $DEV | head -40
sudo sg_vpd --page=di $DEV        # 0x83: device identification (multipath uses this)
sudo sg_vpd --page=bl $DEV        # 0xB0: block limits
sudo sg_vpd --page=bdc $DEV       # 0xB1: rotation rate, form factor
sudo sg_vpd --page=lbpv $DEV      # 0xB2: thin provisioning
```

Where the block layer's limits come from:

```sh
echo "=== from VPD page 0xB0 ==="
sudo sg_vpd --page=bl $DEV
echo "=== reflected in sysfs ==="
DN=$(basename $DEV)
for f in max_sectors_kb max_hw_sectors_kb discard_max_bytes discard_granularity \
         optimal_io_size minimum_io_size rotational; do
  printf "  %-22s %s\n" $f "$(cat /sys/block/$DN/queue/$f 2>/dev/null)"
done
```

MODE SENSE and the cache — **Ch. 61 §T.6's question**:

```sh
sudo sg_modes --page=8 $DEV       # caching mode page
sudo sg_modes --all $DEV | head -30
dmesg | grep -i "$DN.*[Ww]rite cache"
cat /sys/block/$DN/queue/write_cache
cat /sys/class/scsi_disk/*/cache_type
```

Change it and watch the block layer follow:

```sh
echo "write through" | sudo tee /sys/class/scsi_disk/*/cache_type
cat /sys/block/$DN/queue/write_cache
echo "write back" | sudo tee /sys/class/scsi_disk/*/cache_type
cat /sys/block/$DN/queue/write_cache
```

---

### Lab 68.2 — Watch CDBs on the wire

```sh
sudo trace-cmd record -e scsi:scsi_dispatch_cmd_start -e scsi:scsi_dispatch_cmd_done -- \
  sudo dd if=$DEV of=/dev/null bs=4k count=10 iflag=direct 2>/dev/null
sudo trace-cmd report | head -20
```

```
scsi_dispatch_cmd_start: host_no=6 channel=0 id=0 lun=0 data_sgl=1 prot_sgl=0
    prot_op=SCSI_PROT_NORMAL driver_tag=1 scheduler_tag=1 cmnd=(READ_10 lba=0 txlen=8 protect=0 raw=28 00 00 00 00 00 00 00 08 00)
```

**The raw CDB is right there.** Decode it against §T.3.

Now the full logging:

```sh
echo 0x1b6db6db | sudo tee /proc/sys/dev/scsi/logging_level > /dev/null
dmesg -C
sudo dd if=$DEV of=/dev/null bs=4k count=3 iflag=direct 2>/dev/null
sudo sync
dmesg | head -30
echo 0 | sudo tee /proc/sys/dev/scsi/logging_level > /dev/null
```

Watch a flush become SYNCHRONIZE CACHE (Ch. 61 §T.3 → §1.4):

```sh
sudo mkfs.ext4 -qF $DEV && sudo mkdir -p /mnt/s && sudo mount $DEV /mnt/s

sudo trace-cmd record -e scsi:scsi_dispatch_cmd_start -- \
  sudo sh -c 'echo test > /mnt/s/f; sync' 2>/dev/null
sudo trace-cmd report | grep -oP 'cmnd=\(\K[A-Z_0-9]+' | sort | uniq -c
# SYNCHRONIZE_CACHE appears
```

Issue commands by hand:

```sh
sudo sg_turs -v $DEV              # TEST UNIT READY
sudo sg_readcap $DEV              # READ CAPACITY(10)
sudo sg_readcap -l $DEV           # READ CAPACITY(16)
sudo sg_luns $DEV                 # REPORT LUNS
sudo sg_sync $DEV                 # SYNCHRONIZE CACHE
sudo sg_read blk_count=1 bs=512 if=$DEV count=1 2>&1 | head

# Raw CDB
sudo sg_raw -r 512 $DEV 28 00 00 00 00 00 00 00 01 00 | head -c 64 | hexdump -C
#                        ^READ(10) ^LBA=0              ^len=1
```

---

### Lab 68.3 — Sense data and error handling

`scsi_debug` can inject errors, which makes §T.4 directly observable.

```sh
sudo umount /mnt/s 2>/dev/null
sudo modprobe -r scsi_debug
sudo modprobe scsi_debug dev_size_mb=256 opts=2 every_nth=10
# opts=2 -> medium errors;  every_nth=10 -> every 10th command
DEV=$(lsscsi | grep scsi_debug | head -1 | awk '{print $NF}')
DN=$(basename $DEV)

dmesg -C
sudo dd if=$DEV of=/dev/null bs=512 count=50 iflag=direct 2>&1 | tail -2
dmesg | head -30
```

```
sd 6:0:0:0: [sda] tag#0 FAILED Result: hostbyte=DID_OK driverbyte=DRIVER_OK cmd_age=0s
sd 6:0:0:0: [sda] tag#0 Sense Key : Medium Error [current]
sd 6:0:0:0: [sda] tag#0 Add. Sense: Unrecovered read error
sd 6:0:0:0: [sda] tag#0 CDB: Read(10) 28 00 00 00 00 09 00 00 01 00
blk_update_request: critical medium error, dev sda, sector 9
```

**Sense key, ASC/ASCQ, and the exact CDB.** That is the diagnosis §T.4 describes.

Other injectable conditions:

```sh
for opts in 1 2 4 8 16; do
  sudo modprobe -r scsi_debug 2>/dev/null
  sudo modprobe scsi_debug dev_size_mb=64 opts=$opts every_nth=5
  DEV=$(lsscsi | grep scsi_debug | head -1 | awk '{print $NF}')
  dmesg -C
  sudo dd if=$DEV of=/dev/null bs=512 count=20 iflag=direct 2>/dev/null
  echo "=== opts=$opts ==="
  dmesg | grep -iE 'sense|error' | head -4
done
# 1=noise, 2=medium error, 4=timeout, 8=recovered error, 16=transport error
```

Decode sense data yourself:

```sh
sudo sg_decode_sense --err=0x02 2>/dev/null
# Or from a hex dump:
sudo sg_decode_sense 70 00 03 00 00 00 00 0a 00 00 00 00 11 00
#                        ^^key=3 (MEDIUM ERROR)          ^^ASC=11 ASCQ=00
```

Watch the error handler escalate — §T.4's ladder:

```sh
sudo modprobe -r scsi_debug
sudo modprobe scsi_debug dev_size_mb=64 opts=4 every_nth=3   # timeouts
DEV=$(lsscsi | grep scsi_debug | head -1 | awk '{print $NF}')

sudo trace-cmd record -e scsi:\* -- \
  sudo timeout 60 dd if=$DEV of=/dev/null bs=512 count=10 iflag=direct 2>/dev/null
sudo trace-cmd report | grep -iE 'eh_|error' | head -20
dmesg | grep -iE 'abort|reset|eh' | head -20
```

Timeout tuning — §T.4's mitigation:

```sh
cat /sys/block/$DN/device/timeout
cat /sys/block/$DN/device/eh_timeout
cat /sys/class/scsi_host/host*/eh_deadline 2>/dev/null

echo 5 | sudo tee /sys/block/$DN/device/timeout
echo 2 | sudo tee /sys/block/$DN/device/eh_timeout
echo 10 | sudo tee /sys/class/scsi_host/host*/eh_deadline 2>/dev/null

dmesg -C
time sudo dd if=$DEV of=/dev/null bs=512 count=10 iflag=direct 2>&1 | tail -1
# Fails much faster now.
```

Manual resets:

```sh
sudo sg_reset -d $DEV       # device (LUN) reset
sudo sg_reset -t $DEV       # target reset
sudo sg_reset -b $DEV       # bus reset
sudo sg_reset -H $DEV       # host reset
dmesg | tail -5
```

Take a device offline and back:

```sh
cat /sys/block/$DN/device/state
echo offline | sudo tee /sys/block/$DN/device/state
sudo dd if=$DEV of=/dev/null bs=512 count=1 iflag=direct 2>&1 | tail -2
echo running | sudo tee /sys/block/$DN/device/state
sudo dd if=$DEV of=/dev/null bs=512 count=1 iflag=direct 2>&1 | tail -1
```

---

### Lab 68.4 — libata: SCSI over ATA

On real hardware with a SATA disk:

```sh
lsscsi -ltv | grep -i ata
ls /sys/class/ata_port/
ls /sys/class/ata_link/
ls /sys/class/ata_device/

for p in /sys/class/ata_port/*/; do
  echo "=== $(basename $p) ==="
  cat $p/port_no 2>/dev/null
  cat $p/idle_irq 2>/dev/null
done

dmesg | grep -iE 'ata[0-9]+:' | head -20
```

The identity, both ways:

```sh
SATA=/dev/sda     # adjust
sudo hdparm -I $SATA | head -40          # ATA IDENTIFY
sudo sg_inq $SATA                        # the SCSI view, SYNTHESISED by libata
sudo sg_vpd --page=ato $SATA             # ATA information VPD page
```

**Compare the two: the SCSI INQUIRY is fabricated from IDENTIFY DEVICE.** That is §T.6.

ATA PASS-THROUGH — the escape hatch:

```sh
sudo trace-cmd record -e scsi:scsi_dispatch_cmd_start -- \
  sudo smartctl -a $SATA > /dev/null 2>&1
sudo trace-cmd report | grep -oP 'cmnd=\(\K[A-Z_0-9]+' | sort | uniq -c
# ATA_16 / ATA_12: SMART commands wrapped in SCSI CDBs (T.6)

sudo sg_sat_identify $SATA | head -20
sudo sg_sat_read_gplog --log=0x00 $SATA 2>/dev/null | head
```

NCQ — §T.5:

```sh
sudo hdparm -I $SATA | grep -i 'queue depth'
cat /sys/block/$(basename $SATA)/device/queue_depth
dmesg | grep -i 'ncq' | head

# Vary the depth and measure
for qd in 1 4 16 31; do
  echo $qd | sudo tee /sys/block/$(basename $SATA)/device/queue_depth > /dev/null
  echo -n "queue_depth=$qd: "
  sudo fio --name=t --filename=$SATA --direct=1 --rw=randread --bs=4k \
           --iodepth=32 --ioengine=libaio --runtime=8 --time_based \
           --readonly 2>/dev/null | grep -oP 'IOPS=\K[^,]+'
done
```

Link power management — §T.6's hazard:

```sh
for h in /sys/class/scsi_host/host*/; do
  [ -f $h/link_power_management_policy ] && \
    echo "$(basename $h): $(cat $h/link_power_management_policy)"
done

echo med_power_with_dipm | sudo tee /sys/class/scsi_host/host0/link_power_management_policy
dmesg | tail -5
# Watch for link resets after changing this; some drives misbehave.
```

The blacklist — §T.5's empirical record:

```sh
grep -A2 'ata_device_blacklist' /usr/src/linux*/drivers/ata/libata-core.c 2>/dev/null | head -40
# Or read it online. Count the entries:
dmesg | grep -iE 'horkage|blacklist|quirk' | head
```

Write cache and FUA:

```sh
sudo hdparm -W $SATA            # is the write cache on?
sudo hdparm -I $SATA | grep -iE 'write cache|FLUSH|NCQ|TRIM'
cat /sys/block/$(basename $SATA)/queue/{write_cache,fua}
dmesg | grep -i "$(basename $SATA).*Write cache"
```

---

### Lab 68.5 — SAS and enclosures

On SAS hardware:

```sh
ls /sys/class/sas_host/ /sys/class/sas_phy/ /sys/class/sas_port/ \
   /sys/class/sas_device/ /sys/class/sas_expander/ 2>/dev/null

for p in /sys/class/sas_phy/*/; do
  echo "=== $(basename $p) ==="
  for f in sas_address device_type negotiated_linkrate \
           invalid_dword_count running_disparity_error_count \
           loss_of_dword_sync_count phy_reset_problem_count; do
    printf "  %-34s %s\n" $f "$(cat $p/$f 2>/dev/null)"
  done
done
```

**The error counters are the SAS-level equivalent of `UDMA_CRC_Error_Count`** (§T.10): non-zero `invalid_dword_count` or `running_disparity_error_count` means a cable or connector problem.

Topology:

```sh
for d in /sys/class/sas_device/*/; do
  printf "%-22s sas_addr=%s type=%s\n" "$(basename $d)" \
    "$(cat $d/sas_address 2>/dev/null)" \
    "$(cat $d/device_type 2>/dev/null)"
done

lsscsi -tv
sudo sg_map -i -x
```

Enclosure services:

```sh
lsscsi -g | grep -i enclosu
SES=$(lsscsi -g | grep -i enclosu | awk '{print $NF}' | head -1)
[ -n "$SES" ] && {
  sudo sg_ses --page=cf $SES        # configuration
  sudo sg_ses --page=es $SES        # enclosure status
  sudo sg_ses --join $SES | head -40
}

ls /sys/class/enclosure/*/ 2>/dev/null
for e in /sys/class/enclosure/*/*/; do
  [ -d "$e" ] && printf "%-30s status=%s\n" "$(basename $e)" \
    "$(cat $e/status 2>/dev/null)"
done

# Light a slot LED
sudo ledctl locate=/dev/sda 2>/dev/null
sleep 5
sudo ledctl locate_off=/dev/sda 2>/dev/null
```

Transport attributes:

```sh
ls /sys/class/fc_host/ /sys/class/iscsi_session/ 2>/dev/null
for f in /sys/class/fc_host/*/; do
  echo "$(basename $f): $(cat $f/port_name 2>/dev/null)"
  cat $f/dev_loss_tmo 2>/dev/null
  cat $f/fast_io_fail_tmo 2>/dev/null
done
# dev_loss_tmo and fast_io_fail_tmo are T.8's multipath tuning.
```

---

### Lab 68.6 — Multipath

Simulate it with `scsi_debug`:

```sh
sudo modprobe -r scsi_debug
sudo modprobe scsi_debug dev_size_mb=256 num_tgts=1 max_luns=1 \
     vpd_use_hostno=0 add_host=2
lsscsi
# Two devices with the SAME VPD 0x83 identifier: two paths to one LUN.

for d in $(lsscsi | grep scsi_debug | awk '{print $NF}'); do
  echo -n "$d: "
  sudo /lib/udev/scsi_id --page=0x83 --whitelisted --device=$d
done
```

```sh
sudo apt install -y multipath-tools
sudo tee /etc/multipath.conf > /dev/null <<'EOF'
defaults {
    user_friendly_names yes
    find_multipaths no
    path_grouping_policy multibus
    path_selector "service-time 0"
    failback immediate
    no_path_retry 5
}
blacklist_exceptions {
    device {
        vendor "Linux"
        product "scsi_debug"
    }
}
EOF

sudo systemctl restart multipathd
sleep 3
sudo multipath -ll
sudo dmsetup ls --tree
```

```
mpatha (33333330000007d0) dm-0 Linux,scsi_debug
size=256M features='1 queue_if_no_path' hwhandler='0' wp=rw
`-+- policy='service-time 0' prio=1 status=active
  |- 8:0:0:0 sdb 8:16 active ready running
  `- 9:0:0:0 sdc 8:32 active ready running
```

**It is a dm target** (Ch. 66):

```sh
sudo dmsetup table /dev/mapper/mpatha
sudo dmsetup status /dev/mapper/mpatha
```

Path failure:

```sh
sudo mkfs.ext4 -qF /dev/mapper/mpatha
sudo mkdir -p /mnt/mp && sudo mount /dev/mapper/mpatha /mnt/mp

sudo fio --name=t --directory=/mnt/mp --size=50m --rw=randrw --bs=4k \
         --iodepth=4 --ioengine=libaio --runtime=40 --time_based > /dev/null 2>&1 &
FIO=$!
sleep 3

# Fail one path
FIRST=$(sudo multipath -ll | grep -oP '\d+:\d+:\d+:\d+ \Ksd\w+' | head -1)
echo offline | sudo tee /sys/block/$FIRST/device/state
sleep 3
sudo multipath -ll
dmesg | tail -5

# I/O continues on the other path
sleep 5
sudo multipath -ll | grep -E 'active|failed'

# Restore
echo running | sudo tee /sys/block/$FIRST/device/state
sudo multipathd -k'reconfigure' 2>/dev/null
sleep 5
sudo multipath -ll
kill $FIO 2>/dev/null; wait 2>/dev/null
```

Path-selector policies:

```sh
for sel in "round-robin 0" "queue-length 0" "service-time 0"; do
  sudo sed -i "s|path_selector.*|path_selector \"$sel\"|" /etc/multipath.conf
  sudo systemctl restart multipathd; sleep 3
  echo -n "$sel: "
  sudo fio --name=t --filename=/dev/mapper/mpatha --direct=1 --rw=randread \
           --bs=4k --iodepth=32 --ioengine=libaio --runtime=8 --time_based \
           2>/dev/null | grep -oP 'IOPS=\K[^,]+'
done
```

Timeout configuration — §T.8's key point:

```sh
for d in $(sudo multipath -ll | grep -oP '\d+:\d+:\d+:\d+ \Ksd\w+'); do
  echo "$d timeout: $(cat /sys/block/$d/device/timeout)"
  echo 5 | sudo tee /sys/block/$d/device/timeout > /dev/null
  echo 1 | sudo tee /sys/block/$d/device/eh_timeout > /dev/null
done
# Now a dead path fails in seconds, not minutes.
```

Cleanup:

```sh
sudo umount /mnt/mp
sudo multipath -F
sudo systemctl stop multipathd
```

---

### Lab 68.7 — SMART and failure prediction

```sh
SATA=/dev/sda    # a real disk
sudo smartctl -i $SATA
sudo smartctl -H $SATA
sudo smartctl -A $SATA
```

The attributes that matter (§T.10):

```sh
sudo smartctl -A $SATA | awk '
/Reallocated_Sector_Ct|Current_Pending_Sector|Offline_Uncorrectable|UDMA_CRC_Error|Reported_Uncorrect|Spin_Retry|Command_Timeout/ {
  printf "%-28s raw=%s value=%s thresh=%s\n", $2, $10, $4, $6
}'
```

Interpretation:

```sh
cat <<'EOF'
Reallocated_Sector_Ct    > 0      : sectors have already failed and been remapped
Current_Pending_Sector   > 0      : ** sectors that CANNOT be read and are not remapped **
                                     -- these will fail a RAID rebuild (Ch. 67 T.9)
Offline_Uncorrectable    > 0      : found bad during a self-test
UDMA_CRC_Error_Count     rising   : ** CABLE problem, not the drive **
Command_Timeout          rising   : the drive is hanging
Reported_Uncorrect       > 0      : errors reported to the host
EOF
```

Self-tests:

```sh
sudo smartctl -c $SATA | grep -A3 'Short self-test'
sudo smartctl -t short $SATA
sleep 130
sudo smartctl -l selftest $SATA

# The long test reads every sector -- the cheapest latent-error scan
sudo smartctl -t long $SATA
sudo smartctl -c $SATA | grep -i 'self-test routine in progress'
```

Error logs:

```sh
sudo smartctl -l error $SATA
sudo smartctl -l xerror $SATA         # the extended log
sudo smartctl -l devstat $SATA        # device statistics
sudo smartctl -l scttemp $SATA        # temperature history
```

SCSI's equivalent:

```sh
SCSIDEV=/dev/sdb
sudo sg_logs --list $SCSIDEV
sudo sg_logs --page=0x02 $SCSIDEV     # write error counters
sudo sg_logs --page=0x03 $SCSIDEV     # read error counters
sudo sg_logs --page=0x05 $SCSIDEV     # verify error counters
sudo sg_logs --page=0x06 $SCSIDEV     # non-medium error count
sudo sg_logs --page=0x0d $SCSIDEV     # temperature
sudo sg_logs --page=0x2f $SCSIDEV     # informational exceptions (SMART)
sudo sg_logs -a $SCSIDEV | head -60
```

Monitoring:

```sh
sudo systemctl status smartd
cat /etc/smartd.conf | grep -v '^#' | grep -v '^$'
# A useful line:
#   DEVICESCAN -a -o on -S on -n standby,q -s (S/../.././02|L/../../6/03) \
#              -W 4,45,55 -m root -M exec /usr/share/smartmontools/smartd-runner
```

Find early warnings in the logs:

```sh
dmesg | grep -iE 'sense key : recovered|5d/|failure prediction' | head
journalctl -k --since "1 week ago" | grep -iE 'I/O error|medium error|sense key' | head -20
```

`RECOVERED ERROR` entries are the signal to act on — the drive is succeeding, but only after internal retries.

---

### Lab 68.8 — Write a minimal SCSI host driver

```c
// SPDX-License-Identifier: GPL-2.0
/* fakescsi.c -- a minimal SCSI host with one RAM-backed LUN.
 * Demonstrates the T.2 lower-driver contract. */
#include <linux/module.h>
#include <linux/init.h>
#include <linux/slab.h>
#include <linux/vmalloc.h>
#include <scsi/scsi.h>
#include <scsi/scsi_cmnd.h>
#include <scsi/scsi_device.h>
#include <scsi/scsi_host.h>
#include <scsi/scsi_eh.h>

#define FAKE_SIZE_MB	64
#define FAKE_SECTOR_SIZE 512
#define FAKE_SECTORS	((FAKE_SIZE_MB << 20) / FAKE_SECTOR_SIZE)

static u8 *fake_data;
static struct Scsi_Host *fake_shost;

static void fake_inquiry(struct scsi_cmnd *cmd)
{
	u8 inq[36] = {
		0x00,              /* peripheral: direct access block device */
		0x00,              /* not removable */
		0x06,              /* SPC-4 */
		0x02,              /* response format */
		31,                /* additional length */
		0, 0, 0,
		'L','I','N','U','X',' ',' ',' ',
		'F','A','K','E',' ','D','I','S','K',' ',' ',' ',' ',' ',' ',' ',
		'0','0','0','1',
	};

	scsi_sg_copy_from_buffer(cmd, inq, sizeof(inq));
	scsi_set_resid(cmd, scsi_bufflen(cmd) - min_t(int, scsi_bufflen(cmd),
						      sizeof(inq)));
	cmd->result = DID_OK << 16;
}

static void fake_read_capacity10(struct scsi_cmnd *cmd)
{
	u8 buf[8];
	u32 last_lba = FAKE_SECTORS - 1;

	put_unaligned_be32(last_lba, &buf[0]);
	put_unaligned_be32(FAKE_SECTOR_SIZE, &buf[4]);
	scsi_sg_copy_from_buffer(cmd, buf, sizeof(buf));
	cmd->result = DID_OK << 16;
}

static void fake_sense(struct scsi_cmnd *cmd, u8 key, u8 asc, u8 ascq)
{
	scsi_build_sense_buffer(0, cmd->sense_buffer, key, asc, ascq);
	cmd->result = SAM_STAT_CHECK_CONDITION;
}

static int fake_rw(struct scsi_cmnd *cmd, bool write)
{
	u64 lba;
	u32 nr;
	u8 *cdb = cmd->cmnd;
	size_t bytes;

	switch (cdb[0]) {
	case READ_10:
	case WRITE_10:
		lba = get_unaligned_be32(&cdb[2]);
		nr = get_unaligned_be16(&cdb[7]);
		break;
	case READ_16:
	case WRITE_16:
		lba = get_unaligned_be64(&cdb[2]);
		nr = get_unaligned_be32(&cdb[10]);
		break;
	case READ_6:
	case WRITE_6:
		lba = ((cdb[1] & 0x1f) << 16) | (cdb[2] << 8) | cdb[3];
		nr = cdb[4] ? cdb[4] : 256;
		break;
	default:
		fake_sense(cmd, ILLEGAL_REQUEST, 0x20, 0x00);
		return 0;
	}

	if (lba + nr > FAKE_SECTORS) {
		/* T.4: LBA out of range */
		fake_sense(cmd, ILLEGAL_REQUEST, 0x21, 0x00);
		return 0;
	}

	bytes = (size_t)nr * FAKE_SECTOR_SIZE;
	if (write)
		scsi_sg_copy_to_buffer(cmd, fake_data + lba * FAKE_SECTOR_SIZE,
				       bytes);
	else
		scsi_sg_copy_from_buffer(cmd, fake_data + lba * FAKE_SECTOR_SIZE,
					 bytes);

	cmd->result = DID_OK << 16;
	return 0;
}

static int fake_queuecommand(struct Scsi_Host *shost, struct scsi_cmnd *cmd)
{
	if (cmd->device->lun != 0) {
		cmd->result = DID_BAD_TARGET << 16;
		scsi_done(cmd);
		return 0;
	}

	switch (cmd->cmnd[0]) {
	case TEST_UNIT_READY:
		cmd->result = DID_OK << 16;
		break;
	case INQUIRY:
		if (cmd->cmnd[1] & 0x01) {
			/* EVPD: we support no VPD pages */
			fake_sense(cmd, ILLEGAL_REQUEST, 0x24, 0x00);
		} else {
			fake_inquiry(cmd);
		}
		break;
	case READ_CAPACITY:
		fake_read_capacity10(cmd);
		break;
	case READ_6: case READ_10: case READ_16:
		fake_rw(cmd, false);
		break;
	case WRITE_6: case WRITE_10: case WRITE_16:
		fake_rw(cmd, true);
		break;
	case SYNCHRONIZE_CACHE:
		cmd->result = DID_OK << 16;     /* RAM: nothing to flush */
		break;
	case MODE_SENSE: case MODE_SENSE_10:
	case REPORT_LUNS:
	case REQUEST_SENSE:
		cmd->result = DID_OK << 16;
		break;
	default:
		/* T.4: unsupported command */
		fake_sense(cmd, ILLEGAL_REQUEST, 0x20, 0x00);
		break;
	}

	scsi_done(cmd);
	return 0;
}

static int fake_eh_abort(struct scsi_cmnd *cmd)
{
	scmd_printk(KERN_INFO, cmd, "abort handler called\n");
	return SUCCESS;
}

static int fake_eh_device_reset(struct scsi_cmnd *cmd)
{
	scmd_printk(KERN_INFO, cmd, "device reset handler called\n");
	return SUCCESS;
}

static const struct scsi_host_template fake_template = {
	.module			= THIS_MODULE,
	.name			= "fakescsi",
	.proc_name		= "fakescsi",
	.queuecommand		= fake_queuecommand,
	.eh_abort_handler	= fake_eh_abort,
	.eh_device_reset_handler = fake_eh_device_reset,
	.can_queue		= 64,
	.this_id		= -1,
	.sg_tablesize		= SG_ALL,
	.cmd_per_lun		= 32,
	.max_sectors		= 1024,
	.dma_boundary		= PAGE_SIZE - 1,
};

static int __init fake_init(void)
{
	int ret;

	fake_data = vzalloc(FAKE_SIZE_MB << 20);
	if (!fake_data)
		return -ENOMEM;

	fake_shost = scsi_host_alloc(&fake_template, 0);
	if (!fake_shost) { ret = -ENOMEM; goto out_free; }

	fake_shost->max_id = 1;
	fake_shost->max_lun = 1;
	fake_shost->max_channel = 0;

	ret = scsi_add_host(fake_shost, NULL);
	if (ret) goto out_put;

	scsi_scan_host(fake_shost);
	pr_info("fakescsi: host %d added, %d MiB\n",
		fake_shost->host_no, FAKE_SIZE_MB);
	return 0;

out_put:
	scsi_host_put(fake_shost);
out_free:
	vfree(fake_data);
	return ret;
}

static void __exit fake_exit(void)
{
	scsi_remove_host(fake_shost);
	scsi_host_put(fake_shost);
	vfree(fake_data);
}

module_init(fake_init);
module_exit(fake_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Minimal SCSI host driver");
```

```sh
cat > Makefile <<'EOF'
obj-m += fakescsi.o
all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
EOF
make && sudo insmod fakescsi.ko

lsscsi | grep -i fake
DEV=$(lsscsi | grep -i 'FAKE DISK' | awk '{print $NF}')
sudo sg_inq $DEV
sudo sg_readcap $DEV
sudo sg_turs -v $DEV

sudo mkfs.ext4 -qF $DEV
sudo mkdir -p /mnt/fake && sudo mount $DEV /mnt/fake
sudo cp -r /usr/include /mnt/fake/ 2>/dev/null
df -h /mnt/fake
sudo umount /mnt/fake
```

**Roughly 200 lines, and you have a working SCSI device with the full stack above it** — that is §T.2's payoff.

Watch its CDBs:

```sh
sudo trace-cmd record -e scsi:scsi_dispatch_cmd_start -- \
  sudo dd if=$DEV of=/dev/null bs=4k count=10 iflag=direct 2>/dev/null
sudo trace-cmd report | grep -oP 'cmnd=\(\K[A-Z_0-9]+' | sort | uniq -c
```

Exercises:

1. Add VPD page 0x83 with a unique identifier, and check `scsi_id` sees it.
2. Add `MODE SENSE` page 8 reporting a write cache, and verify `/sys/block/*/queue/write_cache` changes.
3. Add `UNMAP` support and watch `blkdiscard` and `fstrim` reach it.
4. Inject a `MEDIUM ERROR` on a specific LBA and observe the error handler.
5. Add a second LUN and verify `REPORT LUNS` and scanning find it.

```sh
sudo rmmod fakescsi
```

---

## 3. Mastery drills

1. Explain why the SCSI command set survived while parallel SCSI did not. Identify the same pattern in two other areas of the kernel.

2. For each of the three SCSI layers, state what it owns and construct the change that would require touching only that layer.

3. Decode the CDB `88 00 00 00 00 00 00 12 34 56 00 00 00 08 00 00` completely. Then state why `sd` chose a 16-byte form.

4. Explain the sense key / ASC / ASCQ hierarchy. Then, for `03/11/00`, `06/29/00`, `02/04/01`, and `01/18/00`, state what happened and what the midlayer should do.

5. `UNIT ATTENTION` is not an error. Explain what it means, construct the spurious-error bug that occurs if a driver treats it as one, and name three conditions that generate it.

6. Trace the error handler's escalation ladder for a device that stops responding. Compute the total stall time with default timeouts, and with tuned ones.

7. SCSI TCQ and ATA NCQ differ in error granularity. Construct the recovery sequence for each after one command fails out of 32 outstanding.

8. `libata` translates ATA to SCSI. For each of INQUIRY, MODE SENSE page 8, and SYNCHRONIZE CACHE, state what ATA information it is synthesised from and what is lost.

9. Explain why `ATA PASS-THROUGH` exists. Then state the general principle and find two other instances of it in the kernel.

10. The `scsi_device` quirk block has ~40 flags. Pick five and, for each, describe the device misbehaviour it works around and what would break without it.

11. Multipath identifies paths by VPD page 0x83. Construct the data-corruption scenario that occurs if two genuinely different LUNs report the same identifier.

12. Distinguish `Current_Pending_Sector` from `Reallocated_Sector_Ct` from `UDMA_CRC_Error_Count`. For each, state the physical cause and the correct remedial action.

13. You are given a server where one disk in a 24-bay SAS enclosure is failing, and the whole enclosure stalls for 60 seconds at a time. Give the ordered diagnostic procedure and the three configuration changes that would mitigate it.

---

## 4. Further reading

**Kernel documentation**

- `Documentation/scsi/scsi_mid_low_api.rst` ★★★ — **the lower-driver contract, in full.** What §1.3's template means, method by method.
- `Documentation/scsi/scsi_eh.rst` ★★★ — **§T.4's escalation ladder, normatively.** Read this before debugging any storage stall.
- `Documentation/scsi/scsi-generic.rst` — the `sg` interface.
- `Documentation/scsi/libsas.rst`, `scsi_fc_transport.rst`, `scsi-parameters.rst`
- `Documentation/scsi/ufs.rst` ★★★ — §T.9.
- `Documentation/driver-api/libata.rst` ★★★ — §T.6's architecture.
- `Documentation/ABI/testing/sysfs-class-scsi_host`, `sysfs-bus-scsi-devices-lun`
- `man 8 smartctl` ★★★, `man 8 sg_inq`, `sg_logs`, `sg_ses`, `sg_vpd`, `sg_modes`
- `man 8 multipath`, `man 5 multipath.conf` ★★★

**Specifications**

These are the primary sources, and unlike many standards they are readable:

- **SAM** (SCSI Architecture Model) ★★★ — the object model of §T.2.
- **SPC** (SCSI Primary Commands) ★★★ — INQUIRY, MODE SENSE, LOG SENSE, VPD pages, and **the complete sense key / ASC / ASCQ tables**. The ASC/ASCQ table alone is worth having.
- **SBC** (SCSI Block Commands) ★★★ — READ, WRITE, SYNCHRONIZE CACHE, UNMAP, WRITE SAME.
- **SAS** (Serial Attached SCSI) — §T.7.
- **SES** (SCSI Enclosure Services)
- **ACS** (ATA Command Set) and **SATA** specifications — §T.6.
- **JEDEC UFS** (JESD220) — §T.9.

Drafts are freely available from the T10 committee (`t10.org`) and T13 (`t13.org`). The published versions cost money; the drafts are near-identical and sufficient.

`www.t10.org/lists/asc-num.txt` ★★★ — **the complete ASC/ASCQ table, free.** Bookmark it.

**Books**

- Schmidt, *The SCSI Bus and IDE Interface*, 2nd ed. — dated on transports, still excellent on the command model and sense data.
- Field, Ridge et al., *Common RAID Disk Data Format* and the SNIA technical documents — for how arrays present themselves.
- Corbet, Rubini, Kroah-Hartman, *Linux Device Drivers 3rd ed.* has no SCSI chapter, deliberately — the authors judged it too large. That is accurate.

**LWN and articles**

- "SCSI in Linux" series and the `scsi-mq` conversion coverage ★★★
- "The SCSI error handler" discussions
- "libata: a new ATA layer" (2004) and the IDE-to-libata transition ★★★
- "The end of the old IDE layer" (2018)
- "UFS: Universal Flash Storage support"
- "Multipath and the block layer"
- "Device mapper multipath" coverage
- James Bottomley's and Christoph Hellwig's talks on the SCSI stack's evolution ★★★
- Martin Petersen's presentations on data integrity (DIF/DIX) ★★★ — the T10 protection information not covered in depth here.

**Source reading order**

1. `Documentation/scsi/scsi_mid_low_api.rst` and `scsi_eh.rst` first.
2. `include/scsi/scsi.h` ★★★ — the opcodes, sense keys, and status codes; a reference you will return to.
3. `drivers/scsi/scsi_debug.c` ★★★ — **the most instructive file here.** A complete, configurable, in-memory SCSI target that implements dozens of commands. Every command handler is a readable example of what that command means. Read it instead of the specifications when you can.
4. `drivers/scsi/sd.c`: `sd_init_command`, `sd_setup_read_write_cmnd` ★★★, `sd_read_cache_type`, `sd_revalidate_disk`.
5. `drivers/scsi/scsi_error.c` ★★★ — `scsi_check_sense`, `scsi_eh_ready_devs`, `scsi_error_handler`. §T.4.
6. `drivers/scsi/scsi_lib.c`: `scsi_queue_rq`, `scsi_io_completion` — the blk-mq integration.
7. `drivers/ata/libata-scsi.c` ★★★ — the translation table of §T.6: `ata_scsi_rw_xlat`, `ata_scsiop_inq_std`, `ata_scsi_flush_xlat`.
8. `drivers/ata/libata-eh.c` — ATA's own error handling, under the SCSI EH.
9. `drivers/ata/libata-core.c`'s `ata_device_blacklist` ★★★ — read the whole table; it is an education.

**Tools**

- `sg3_utils` ★★★ — **`sg_inq`, `sg_vpd`, `sg_logs`, `sg_modes`, `sg_ses`, `sg_readcap`, `sg_turs`, `sg_raw`, `sg_decode_sense`, `sg_reset`.** Learn `sg_raw` in particular; it lets you issue any CDB.
- `smartmontools` ★★★ — `smartctl -a`, `-A`, `-l selftest`, `-l error`, `-t long`; and `smartd` for monitoring
- `lsscsi -ltvg` ★★★
- `hdparm -I`, `-W`, `-C`, `--fibmap`
- `scsi_debug` ★★★ — **the laboratory.** `opts=`, `every_nth=`, `num_tgts=`, `max_luns=`, `sector_size=`, `physblk_exp=`, `lbpu=`, `zbc=`
- `trace-cmd record -e scsi:\*` ★★★ and `/proc/sys/dev/scsi/logging_level`
- `multipath -ll`, `multipathd -k` ★★★
- `ledctl`, `sg_ses` for enclosure management
- `dmesg` — SCSI's logging is unusually informative; read it rather than guessing

---

→ Next: [69-nvme.md](69-nvme.md)

# Chapter 70 — MTD, raw NAND, eMMC/SD, UBI/UBIFS, and flash reality

> **Goal:** Understand storage where there is no FTL hiding the physics — raw NAND, where the kernel must do wear levelling, bad-block management, and ECC itself. Understand why MTD is not a block device and cannot be one, the erase-block/page/OOB geometry that shapes everything, ECC as a requirement rather than an option, bad blocks as a fact of manufacture, UBI as the layer that solves wear levelling once so filesystems need not, UBIFS's design for unreliable flash, and why eMMC and SD sit awkwardly between raw NAND and NVMe. By the end you can read `drivers/mtd/`, bring up a NAND device, and reason about flash endurance from the cell upward.

---

## Theory & First Principles

### T.0 — Start here: a storage device with no sectors

Everything in Part 3 so far assumed a block device: a flat array of 4 KiB sectors you may
overwrite at will. **Raw NAND flash offers none of that.**

```
  READ   unit:  a PAGE      (2 KiB - 16 KiB)
  WRITE  unit:  a PAGE      -- and ONLY ONCE after the block is erased
  ERASE  unit:  a BLOCK     (128 KiB - 4 MiB)   <- 100x to 1000x larger

  Additional facts the block layer has no way to express:
    * a block survives only ~1,000 (TLC) to ~100,000 (SLC) erase cycles
    * SOME BLOCKS ARE BAD FROM THE FACTORY, and more go bad over time
    * every read returns bit errors; ECC is MANDATORY, not optional
    * reading a page disturbs its neighbours (read disturb)
    * charge leaks: data fades over months (retention)
```

**Four of those six facts have no representation in `struct bio`.** There is no "erase," no
"this block is worn out," no "this region needs rewriting before it fades." So the block
abstraction (Ch. 63 §T.0) does not merely perform badly here — it **cannot express the
device**.

**Hence two completely different strategies, and knowing which one you are in is the whole
chapter:**

```
  STRATEGY A -- HIDE IT                  STRATEGY B -- EXPOSE IT
  (SSD, eMMC, UFS, SD card)              (raw NAND on an embedded board)

  +--------------------------+           +---------------------------+
  | filesystem (ext4/F2FS)   |           | UBIFS / JFFS2 / YAFFS     |
  +--------------------------+           +---------------------------+
  | block layer              |           | UBI  (wear levelling,     |
  +--------------------------+           |       bad-block mgmt,     |
  | FTL in DEVICE FIRMWARE   |  <- the   |       logical volumes)    |
  |  mapping table, GC,      |   secret  +---------------------------+
  |  wear levelling, ECC     |   layer   | MTD (raw: read/write/erase|
  +--------------------------+           |      + ECC + bad blocks)  |
  | NAND                     |           +---------------------------+
  +--------------------------+           | NAND                      |
                                         +---------------------------+
```

**Strategy A is what you use every day, and its cost is invisibility.** The FTL is a
proprietary, unauditable log-structured filesystem running on a microcontroller inside your
drive. It decides when to garbage collect (causing latency spikes you cannot predict or
attribute), it holds a mapping table it may lose on power failure (cheap devices have
corrupted themselves this way), and `TRIM`/`discard` is the *only* channel you have for
telling it anything — which is exactly why `fstrim` matters: without it the FTL believes
every block you ever wrote is still live, and garbage collects data that no longer exists.

**Strategy B is the embedded answer, and MTD's interface is the honest one:**

```c
int (*_read) (struct mtd_info *, loff_t, size_t, size_t *, u_char *);
int (*_write)(struct mtd_info *, loff_t, size_t, size_t *, const u_char *);
int (*_erase)(struct mtd_info *, struct erase_info *);   /* <- no block device has this */
int (*_block_isbad)  (struct mtd_info *, loff_t);        /* <- nor this */
int (*_block_markbad)(struct mtd_info *, loff_t);
```

**Erase, bad-block query, and bad-block marking are first-class operations.** Above it, UBI
adds wear levelling and a logical-to-physical map so the filesystem above *that* can pretend
blocks are eternal; UBIFS then does compression, journalling, and crash recovery knowing the
real geometry underneath.

**The recurring question this closes Part 3 on, and it is a genuinely architectural one:**

> **Should a layer hide the hardware's true nature, or expose it?**

| Hide (FTL, SSD) | Expose (MTD/UBI, ZNS, SMR host-managed) |
|---|---|
| Every existing filesystem just works | the host must be rewritten to cope |
| One narrow, stable interface | a wider, device-shaped interface |
| Duplicated GC: two log structures fighting (Ch. 60 §T.0) | one GC, with full knowledge of what is live |
| Unpredictable latency you cannot attribute | latency you control and can schedule |
| Device firmware you cannot audit or fix | logic in open, reviewable kernel code |

There is no universal answer — **and notice that the industry has now tried both directions
and is currently moving back toward exposure** (ZNS, host-managed SMR, open-channel SSDs)
precisely because hiding cost too much predictability at scale. That oscillation, rather than
any single answer, is the mature view to take away.

```bash
cat /proc/mtd                          # partitions on raw flash
sudo mtdinfo -a && sudo nanddump -f /tmp/d /dev/mtd0
sudo ubinfo -a                         # UBI volumes, erase counts, bad PEBs
cat /sys/class/mtd/mtd0/{erasesize,writesize,oobsize}
sudo fstrim -v /                       # tell an FTL what is actually dead
```

---

### T.1 What raw flash actually is

Every abstraction in Chapters 51–69 assumed a device that can read and write any sector. NAND flash cannot:

| Property | Consequence |
|---|---|
| **Read granularity: a page** (2–16 KiB) | cannot read one byte efficiently |
| **Write granularity: a page** | cannot write less |
| **Erase granularity: a block** (128 KiB – 4 MiB) | **cannot overwrite a page; must erase its whole block** |
| **Erase sets bits to 1; writes clear them to 0** | a write can only turn 1→0 |
| **Limited program/erase cycles** | 1,000 (TLC/QLC) to 100,000 (SLC) |
| **Bit errors are normal** | ECC is mandatory, not optional |
| **Some blocks are bad from the factory** | must be detected and avoided |
| **Blocks wear out during use** | must be retired and replaced |
| **Reading disturbs neighbours** | read disturb requires periodic refresh |
| **Charge leaks over time** | data retention is finite, and worse when worn |

An SSD hides all of this behind a **Flash Translation Layer** — a microcontroller running a log-structured filesystem with garbage collection and wear levelling, presenting a block device (Ch. 60 §T.2).

**MTD is what you have when there is no FTL.** Raw NAND soldered to a board, an SPI-NOR chip holding a bootloader, the flash in a router or an industrial controller. The kernel must do the FTL's job — or, more accurately, must provide the abstractions that let a filesystem do it.

This chapter is therefore the only place in Part 3 where the physics is visible. Everything else has been built on the pretence that storage is an addressable array.

### T.2 MTD: the interface that admits the truth

```c
struct mtd_info {
	u_char type;                  /* MTD_NANDFLASH, MTD_NORFLASH, ... */
	uint32_t flags;
	uint64_t size;
	uint32_t erasesize;           /* THE fundamental unit */
	uint32_t writesize;           /* page size */
	uint32_t writebufsize;
	uint32_t oobsize;             /* spare bytes per page (T.3) */
	uint32_t oobavail;            /* after ECC takes its share */
	uint32_t erasesize_shift, writesize_shift;
	uint32_t erasesize_mask, writesize_mask;
	unsigned int bitflip_threshold;   /* T.4 */
	const char *name;
	int index;
	struct mtd_ooblayout_ops *ooblayout;
	unsigned int ecc_step_size;
	unsigned int ecc_strength;    /* bits correctable per step */
	int numeraseregions;
	struct mtd_erase_region_info *eraseregions;
	...
	int (*_erase)(struct mtd_info *, struct erase_info *);
	int (*_read)(struct mtd_info *, loff_t, size_t, size_t *, u_char *);
	int (*_write)(struct mtd_info *, loff_t, size_t, size_t *, const u_char *);
	int (*_read_oob)(struct mtd_info *, loff_t, struct mtd_oob_ops *);
	int (*_write_oob)(struct mtd_info *, loff_t, struct mtd_oob_ops *);
	int (*_block_isbad)(struct mtd_info *, loff_t);
	int (*_block_markbad)(struct mtd_info *, loff_t);
	int (*_block_isreserved)(struct mtd_info *, loff_t);
	...
};
```

**`_erase` is the operation no block device has.** Its presence is the whole difference. A block device's interface says "write these bytes here"; MTD's says "erase this block, then program these pages, and by the way some blocks are bad and some bits will flip."

The API is deliberately not a block device because making it one would require exactly the FTL the hardware lacks. Several attempts exist (`mtdblock`, `ftl`, `nftl`) and they are all either read-mostly hacks or obsolete:

| Layer | Note |
|---|---|
| `mtdblock` | read-modify-erase-write per block; **no wear levelling, no power-fail safety** |
| `mtdblock_ro` | read-only; safe |
| `ftl`, `nftl`, `inftl` | ancient PCMCIA-era FTLs; obsolete |
| **UBI + `ubiblock`** | read-only block device over UBI; **the correct way** |

Using `mtdblock` read-write on real NAND will eventually destroy the data. It exists for read-only root filesystems and for NOR flash where the cost model is different.

### T.3 Geometry: pages, blocks, and the OOB area

```
Chip
 └─ Erase block (e.g. 128 KiB)
     └─ Page (e.g. 2048 bytes data + 64 bytes OOB)
         └─ ECC step (e.g. 512 bytes data + parity)
```

The **OOB (Out Of Band) area**, also called the spare area, is extra bytes per page — originally intended for the filesystem's metadata, now almost entirely consumed by:

| Use | Note |
|---|---|
| **ECC parity** | the dominant consumer |
| **Bad block marker** | a byte in the first (or last) page of a bad block |
| Filesystem metadata | what JFFS2 used it for |
| UBI's erase counter header | partially |

Typical: a 2048+64 page with 4-bit BCH ECC over 512-byte steps uses 4 × 13 = 52 bytes for parity, leaving ~12 usable. A modern 4096+224 page with 24-bit ECC uses nearly all of it.

**`oobavail` is what is left after ECC.** Filesystems that wanted OOB space (JFFS2) became unusable as ECC requirements grew — a good example of a hardware trend invalidating a software design.

The **bad block marker** convention is a historical mess: the manufacturer marks bad blocks by writing a non-`0xFF` byte somewhere in the OOB of the first or second page of the block. *Which* byte and *which* page varies by manufacturer and by device generation. Getting it wrong means either using bad blocks (corruption) or discarding good ones (lost capacity). The kernel has a table of layouts, and `nand_bbt` builds an in-memory **Bad Block Table** at boot by scanning.

Because the markers are in the OOB and the OOB is also used by ECC, and because an erase would destroy the markers, **you must never erase a factory-bad block.** `nanderase` and `flash_erase` skip them; `dd` to `/dev/mtdN` does not, which is one way people destroy a device's bad-block information permanently.

The BBT itself is usually stored **on the flash** (in a reserved block at the end) so it need not be rescanned, with a mirror for safety. `NAND_BBT_USE_FLASH` controls this.

### T.4 ECC: correction as a requirement

NAND returns wrong bits. Not occasionally — **routinely**, as a designed property. The manufacturer specifies a required ECC strength:

| NAND type | Typical requirement |
|---|---|
| SLC, older | 1 bit per 512 bytes |
| MLC | 4–8 bits per 512 bytes |
| TLC | 24–40 bits per 1024 bytes |
| QLC | 60+ bits per 1024 bytes |

If you use weaker ECC than specified, **the device works fine when new and corrupts data as it wears.** This is one of the most common embedded-storage failures, and it appears months after deployment.

ECC implementations, in order of preference:

| Mode | Where | Note |
|---|---|---|
| **On-die** | inside the NAND chip | the chip corrects internally; simplest |
| **Hardware** | in the NAND controller | fast; what most SoCs provide |
| **Software BCH** | `lib/bch.c` | works anywhere; CPU cost |
| **Software Hamming** | 1-bit only | legacy SLC only |
| **None** | — | valid only for chips with on-die ECC |

The algorithms:

- **Hamming** corrects 1 bit, detects 2. Sufficient for old SLC.
- **BCH** (Bose–Chaudhuri–Hocquenghem) corrects `t` bits with roughly `t × log₂(n)` parity bits. The standard choice.
- **LDPC** — used inside modern SSD controllers, rarely exposed to the host.

**Bitflip reporting is the important part.** `mtd_read()` returns:

| Return | Meaning |
|---|---|
| `0` | clean read |
| `> 0` | **corrected N bitflips** — the data is fine *now* |
| `-EUCLEAN` | corrected, but the count exceeded `bitflip_threshold` |
| `-EBADMSG` | **uncorrectable** — the data is lost |

`-EUCLEAN` is the early warning, and reacting to it is mandatory: it means "this block is approaching the correction limit; move the data before it becomes uncorrectable." UBI does this automatically (§T.6); a filesystem operating on raw MTD must do it itself.

```sh
cat /sys/class/mtd/mtd0/bitflip_threshold
cat /sys/class/mtd/mtd0/ecc_strength
cat /sys/class/mtd/mtd0/ecc_step_size
cat /sys/class/mtd/mtd0/{ecc_failures,corrected_bits,bad_blocks,bbt_blocks}
```

`bitflip_threshold` defaults to `ecc_strength × 3/4` — react at 75 % of capacity, leaving margin.

### T.5 Wear levelling and the failure modes

An erase block has a finite P/E cycle count. If a filesystem repeatedly rewrites one region, that block dies while others are untouched.

**Two kinds of wear levelling:**

| | **Dynamic** | **Static** |
|---|---|---|
| Moves | data being rewritten | **cold data that never changes** |
| Why | spread writes over free blocks | otherwise cold blocks never wear, so hot blocks absorb everything |
| Cost | none extra | extra writes to move stable data |

Static wear levelling is essential and counterintuitive: **you must deliberately rewrite data that nobody asked you to rewrite**, because otherwise the blocks holding your bootloader (written once, never changed) stay at 0 erase cycles while the blocks holding your logs hit the limit.

Three other physical effects that require action:

**(a) Read disturb.** Reading a page applies voltage that slightly disturbs charge in *neighbouring* pages of the same block. After enough reads (10⁵–10⁶) without an erase, neighbours accumulate errors. Mitigation: track read counts and rewrite blocks that have been read heavily. UBI does this by reacting to `-EUCLEAN`.

**(b) Data retention.** Charge leaks. A block written and left alone for years — especially a worn block, especially at high temperature — accumulates errors. The specification is typically "1 year at 40 °C at end of rated life," which is much worse than people assume. Mitigation: periodic scrubbing, i.e. read everything and rewrite anything reporting bitflips.

**(c) Program disturb** and **partial-page programming.** Writing one page disturbs others in the block; writing the same page twice without erasing (allowed on SLC, forbidden on MLC/TLC) accelerates it. Modern NAND forbids partial-page programs entirely, which is why `writesize` is a hard constraint rather than a preference.

The consequence: **a flash device left powered off for a long time can lose data, and a flash device that is only ever read can also lose data.** Both are surprising if you think of flash as "writes wear it out, otherwise it is stable."

### T.6 UBI: solve it once

Writing wear levelling, bad-block management, and bitflip response into every filesystem is duplicated, subtle, and easy to get wrong. **UBI (Unsorted Block Images)** does it once:

```
        UBIFS / ubiblock / user
                  |
        ┌─────────────────────┐
        │        UBI          │   logical erase blocks (LEBs)
        │  LEB -> PEB mapping │   wear levelling, bad block handling,
        │  erase counters     │   scrubbing, volume management
        └─────────────────────┘
                  |
        ┌─────────────────────┐
        │        MTD          │   physical erase blocks (PEBs)
        └─────────────────────┘
```

UBI's abstraction: **logical erase blocks that never go bad and never wear out.** A LEB maps to some PEB; UBI is free to move it.

The mechanism, and it is elegantly simple:

- Every PEB has a header (`ec_hdr`) containing an **erase counter**.
- Every mapped PEB has a second header (`vid_hdr`) naming the volume and LEB it holds.
- On attach, UBI scans every PEB, reads the headers, and reconstructs the mapping.
- When a LEB is written, UBI picks a PEB with a low erase count.
- Periodically, UBI moves data from a low-erase-count PEB to a high-erase-count one — **static wear levelling** (§T.5).
- On `-EUCLEAN`, UBI moves the data and **scrubs** the PEB.
- On a write or erase failure, UBI marks the PEB bad and retires it.

Two features worth understanding:

**Atomic LEB change** (`UBI_IOCEBCH`). UBI can replace a LEB's contents atomically: write to a spare PEB, then update the mapping. Either the old or the new contents survive a power failure, never a mixture. **This gives filesystems a primitive that raw flash cannot provide**, and UBIFS depends on it for its superblock updates.

**Volumes.** UBI divides its space into named volumes, resizable at runtime:

```sh
ubimkvol /dev/ubi0 -N rootfs -s 200MiB
ubimkvol /dev/ubi0 -N data -m          # -m: maximum available
ubirsvol /dev/ubi0 -N data -s 300MiB
```

Unlike MTD partitions (fixed at build time, defined in the device tree), UBI volumes are dynamic. A device with one UBI on one large MTD partition and several volumes inside it is far more flexible than several fixed MTD partitions, and it gives UBI more PEBs to wear-level across — which is the more important benefit.

**The cost:** UBI reserves blocks for bad-block handling (`ubi.mtd=0,2048` style overhead, typically 2 % plus a minimum), needs a full scan at attach time (slow on large devices; mitigated by the **fastmap** feature, which checkpoints the mapping), and adds a layer of indirection.

### T.7 UBIFS: a filesystem for unreliable flash

JFFS2 was the earlier answer: a log-structured filesystem directly on MTD. Its fatal flaw was that it kept **the entire index in RAM**, rebuilt by scanning the whole device at mount. Mount time and memory use were both O(device size). On a 1 GB device that is minutes and hundreds of megabytes.

UBIFS fixes this by putting the index **on the flash**, as a wandering B+tree:

| | JFFS2 | UBIFS |
|---|---|---|
| Index location | **RAM only** | **on flash** (a B+tree) |
| Mount time | O(size) — scan everything | O(1) |
| RAM usage | O(size) | O(working set) |
| Underlying layer | MTD directly | **UBI** |
| Compression | zlib | zlib, LZO, **zstd** |
| Write-back | no (synchronous) | yes |
| Max practical size | ~100 MB | GB |

UBIFS's structures:

```
Superblock       (LEB 0)           -- static geometry
Master node      (LEB 1, 2)        -- the root of everything, two copies
Log area                            -- the journal
LPT (LEB Properties Tree)           -- free/dirty space per LEB
Orphan area                         -- inodes deleted while open
Main area                           -- the index B+tree and the data
```

The **wandering tree** is the same idea as btrfs's CoW (Ch. 59 §T.2): updating a leaf requires updating its parent, up to the root, and the root pointer lives in the master node. The master node is written to two LEBs alternately so one is always valid.

UBIFS's **journal** is a write-back buffer: changes accumulate in the log area and are periodically **committed** into the index. This means recent writes require replaying the log on mount, which is fast (the log is small and bounded).

Three practical points:

**(a) Free space reporting is approximate and pessimistic.** UBIFS cannot know how well future data will compress, so `df` reports a conservative estimate. Reported free space can *increase* as you write compressible data. This confuses monitoring systems.

**(b) `fsync` semantics.** UBIFS honours `fsync`, but the default `commit` interval and write-back mean unsynced data can be lost. The `sync` mount option makes everything synchronous at a large performance cost.

**(c) Power-fail safety depends on UBI's atomic LEB change** and on the flash actually finishing a page program when power drops. Some NAND partially programs a page on power loss, leaving it readable-but-weak — a page that reads correctly now and fails in a month. UBIFS cannot detect this, and it is the main reason embedded devices want a supercapacitor or a clean shutdown.

**When to use what:**

| Situation | Choice |
|---|---|
| Raw NAND, read-write | **UBIFS on UBI** |
| Raw NAND, read-only root | **SquashFS on ubiblock** ★★★ |
| Raw NAND, very small (< 16 MB) | JFFS2, reluctantly |
| NOR flash, small, read-mostly | **JFFS2** or SquashFS on mtdblock |
| NOR flash, execute-in-place | **AXFS** or a raw partition |
| eMMC/SD (has an FTL) | **ext4** or **F2FS** — not MTD at all |
| Read-only, compressed, any | **SquashFS** or **EROFS** ★★★ |

EROFS deserves a mention: it is the modern read-only compressed filesystem, designed at Huawei for Android, with better random-read performance than SquashFS because it supports fixed-size output compression clusters rather than fixed-size input.

### T.8 eMMC and SD: the FTL in a cheap package

eMMC and SD cards are NAND **with an FTL**, so they present a block device and none of §T.1–T.7 applies at the software level. But the FTL is small, cheap, and much less capable than an SSD's:

| | eMMC/SD | SSD |
|---|---|---|
| Controller RAM | tens of KB | hundreds of MB |
| Mapping granularity | **erase-block** (megabytes) | 4 KiB pages |
| Queue depth | 1, or 32 with CQ | thousands |
| GC | crude, blocking | sophisticated, background |
| Power-loss protection | usually none | often, on enterprise parts |

The **mapping granularity** is the important difference. An SSD maps 4 KiB pages, so a random 4 KiB write updates one mapping entry. An SD card maps whole erase blocks, so a random 4 KiB write may require **reading a 4 MB block, modifying it, erasing, and rewriting** — a write amplification of 1000×.

This is why:

- Random small writes on SD cards are catastrophically slow (often < 100 IOPS).
- SD cards die quickly under database or logging workloads.
- Alignment to the erase block matters enormously — a misaligned filesystem doubles the read-modify-write work.
- `F2FS` (Ch. 60 §T.3) helps substantially, because its sequential write pattern matches what the FTL wants.

The Raspberry Pi's reputation for SD card failures is entirely this: a general-purpose Linux system writing logs and updating a journal on a device whose FTL was designed for sequential camera writes.

eMMC-specific features worth knowing:

| Feature | Note |
|---|---|
| **Boot partitions** | separate small areas the SoC ROM reads first |
| **RPMB** | Replay Protected Memory Block, for secure counters |
| **Enhanced/SLC areas** | part of the device configured as SLC for reliability |
| **Command Queueing (CQ)** | eMMC 5.1+; up to 32 commands |
| **HS400/HS400ES** | the fast transfer modes |
| **Sanitize / Secure Erase** | actually erase, not just TRIM |
| **Write Protect groups** | hardware-enforced read-only regions |
| **Life time estimation** | `EXT_CSD` bytes 268–269: A and B type wear, in 10 % steps |

```sh
sudo mmc extcsd read /dev/mmcblk0 | grep -iE 'life|eol|pre.eol'
```

`PRE_EOL_INFO` and `DEVICE_LIFE_TIME_EST_TYP_A/B` are the eMMC equivalent of SMART, and checking them is the only way to know an eMMC is wearing out before it fails.

### T.9 SPI-NOR and the small end

NOR flash differs from NAND fundamentally:

| | NOR | NAND |
|---|---|---|
| Read granularity | **byte** — randomly addressable | page |
| Execute in place (XIP) | **yes** | no |
| Read speed | fast, low latency | fast, higher latency |
| Write speed | **very slow** | fast |
| Erase time | **very slow** (seconds per block) | ~ms |
| Erase block | smaller (4–64 KiB) | larger |
| Density | low | high |
| Cost per bit | **high** | low |
| Bit errors | rare | routine |

NOR is used where you need to *execute* directly from flash (a bootloader before RAM is initialised) or where reliability matters more than capacity. SPI-NOR is the ubiquitous form: a small serial chip holding a bootloader, a device tree, and sometimes a small read-only root.

```sh
ls /sys/bus/spi/devices/
cat /proc/mtd
```

The kernel's `drivers/mtd/spi-nor/` supports hundreds of chips with a per-manufacturer quirk table, plus **SFDP** (Serial Flash Discoverable Parameters) — a standard in-chip descriptor that reports geometry and supported commands, so new chips often work without a driver entry. SFDP is the same "make the device self-describing" idea as PCI configuration space (Ch. 37) or ACPI's `_DSD` (Ch. 33).

### T.10 The practical reality

Bringing up flash on an embedded board, in order:

**1. Get the geometry right.** Wrong page size or erase size corrupts silently.

```sh
cat /proc/mtd
mtdinfo /dev/mtd0 -u
```

**2. Get the ECC right.** Match the manufacturer's specification, not what boots.

```sh
cat /sys/class/mtd/mtd0/ecc_strength
cat /sys/class/mtd/mtd0/ecc_step_size
dmesg | grep -i 'ecc\|nand'
```

**This is the single most common embedded storage bug**, and it does not manifest for months.

**3. Preserve the bad block table.** Never `dd` over a whole NAND device. Never erase factory-bad blocks.

**4. Match the bootloader.** U-Boot and Linux must agree on page size, ECC layout, ECC strength, OOB layout, and bad-block marker position. They frequently do not, and the symptom is "the bootloader can read what it wrote, Linux cannot."

**5. Use UBI.** Unless there is a specific reason not to.

**6. Plan for power loss.** A device without a supercapacitor will lose power mid-program. UBI and UBIFS are designed for it; test it (`dm-flakey` cannot help here — you need actual power cycling, or `nandsim` with error injection).

**7. Monitor.** `-EUCLEAN` counts, bad block counts, UBI's erase counter spread. All are available and almost nobody watches them.

```sh
cat /sys/class/mtd/mtd0/{corrected_bits,ecc_failures,bad_blocks}
ubinfo -a | grep -iE 'bad|reserved'
cat /sys/class/ubi/ubi0/*
```

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `drivers/mtd/mtdcore.c` ★★★ | §T.2's core: registration, the generic read/write path |
| `drivers/mtd/mtdpart.c` | partitions |
| `drivers/mtd/mtdchar.c` ★★★ | `/dev/mtdN`, the ioctl interface |
| `drivers/mtd/mtdblock.c` | §T.2's block emulation; read the comments about why it is dangerous |
| `drivers/mtd/mtdconcat.c` | concatenating devices |
| `drivers/mtd/nand/raw/nand_base.c` ★★★ | the raw NAND core: read/write/erase, ONFI |
| `drivers/mtd/nand/raw/nand_bbt.c` ★★★ | §T.3's bad block table |
| `drivers/mtd/nand/raw/nand_ecc.c`, `nand_bch.c` | §T.4 |
| `drivers/mtd/nand/raw/nandsim.c` ★★★ | **a NAND simulator** — the lab tool |
| `drivers/mtd/nand/spi/` | SPI-NAND |
| `drivers/mtd/spi-nor/core.c`, `sfdp.c` ★★★ | §T.9 |
| `drivers/mtd/ubi/` ★★★ | §T.6: `build.c`, `wl.c`, `eba.c`, `attach.c`, `fastmap.c` |
| `drivers/mtd/ubi/wl.c` ★★★ | **the wear-levelling algorithm** |
| `fs/ubifs/` ★★★ | §T.7 |
| `fs/jffs2/` | the predecessor |
| `fs/squashfs/`, `fs/erofs/` ★★★ | read-only alternatives |
| `drivers/mmc/core/`, `drivers/mmc/host/` | §T.8 |
| `lib/bch.c` | the BCH implementation |
| `Documentation/driver-api/mtdnand.rst` ★★★ | |
| `Documentation/filesystems/ubifs.rst` ★★★ | |

### 1.2 The NAND chip abstraction

```c
struct nand_chip {
	struct nand_device base;
	struct nand_legacy legacy;
	unsigned int options;
	unsigned int bbt_options;
	int page_shift;
	int phys_erase_shift;
	int bbt_erase_shift;
	int chip_shift;
	int pagemask;
	u8 *data_buf;
	int pagecache_page;
	bool pagecache_bitflips;
	int subpagesize;
	int badblockpos;              /* T.3: WHERE the marker lives */
	int badblockbits;
	struct nand_id id;
	struct nand_parameters parameters;
	unsigned int data_interface_ts;
	...
	struct nand_operation *cur_cs;
	const struct nand_controller_ops *controller_ops;
	uint8_t *bbt;                 /* the in-memory bad block table */
	struct nand_bbt_descr *bbt_td, *bbt_md;
	struct nand_bbt_descr *badblock_pattern;
	...
};

struct nand_ecc_props {
	enum nand_ecc_engine_type engine_type;
	enum nand_ecc_placement placement;
	enum nand_ecc_algo algo;
	unsigned int strength;        /* T.4: bits correctable */
	unsigned int step_size;
	unsigned int flags;
};
```

`badblockpos` is §T.3's manufacturer variation, made a field. `NAND_LARGE_BADBLOCK_POS` (0) and `NAND_SMALL_BADBLOCK_POS` (5) are the two common values, and `options` carries flags like `NAND_BBM_FIRSTPAGE`, `NAND_BBM_SECONDPAGE`, `NAND_BBM_LASTPAGE` for which page holds it.

### 1.3 Read, with ECC

```c
static int nand_do_read_ops(struct nand_chip *chip, loff_t from,
			    struct mtd_oob_ops *ops)
{
	struct mtd_info *mtd = nand_to_mtd(chip);
	int ret = 0;
	uint32_t max_oobsize = mtd_oobavail(mtd, ops);
	unsigned int max_bitflips = 0;
	...
	while (1) {
		...
		if (realpage != chip->pagecache_page || oob) {
			bufpoi = use_bounce_buf ? chip->data_buf : buf;
			...
			ret = chip->ecc.read_page(chip, bufpoi,
						  oob_required, page);
			if (ret < 0) {
				if (use_bounce_buf)
					memset(buf, 0xff, ...);
				break;
			}

			/* T.4: the return value is the BITFLIP COUNT */
			max_bitflips = max_t(unsigned int, max_bitflips, ret);
			...
		}
		...
	}
	...
	if (ecc_fail)
		return -EBADMSG;          /* uncorrectable */

	return max_bitflips;              /* >= 0: how many were corrected */
}

int mtd_read_oob(struct mtd_info *mtd, loff_t from, struct mtd_oob_ops *ops)
{
	...
	ret_code = mtd_read_oob_std(mtd, from, ops);
	...
	if (unlikely(ret_code < 0))
		return ret_code;
	if (mtd->ecc_strength == 0)
		return 0;
	/* T.4: convert a high bitflip count into -EUCLEAN */
	return ret_code >= mtd->bitflip_threshold ? -EUCLEAN : 0;
}
```

**The bitflip count travels up as a positive return value, and `-EUCLEAN` is synthesised from it.** That is §T.4's mechanism in two functions.

The software BCH path:

```c
static int nand_read_page_swecc(struct nand_chip *chip, uint8_t *buf,
				int oob_required, int page)
{
	struct mtd_info *mtd = nand_to_mtd(chip);
	int i, eccsize = chip->ecc.size, ret;
	int eccbytes = chip->ecc.bytes;
	int eccsteps = chip->ecc.steps;
	uint8_t *p = buf;
	uint8_t *ecc_calc = chip->ecc.calc_buf;
	uint8_t *ecc_code = chip->ecc.code_buf;
	unsigned int max_bitflips = 0;

	chip->ecc.read_page_raw(chip, buf, 1, page);

	for (i = 0; eccsteps; eccsteps--, i += eccbytes, p += eccsize)
		chip->ecc.calculate(chip, p, &ecc_calc[i]);

	ret = mtd_ooblayout_get_eccbytes(mtd, ecc_code, chip->oob_poi, 0,
					 chip->ecc.total);
	if (ret) return ret;

	eccsteps = chip->ecc.steps;
	p = buf;

	for (i = 0; eccsteps; eccsteps--, i += eccbytes, p += eccsize) {
		int stat;

		stat = chip->ecc.correct(chip, p, &ecc_code[i], &ecc_calc[i]);
		if (stat < 0) {
			mtd->ecc_stats.failed++;       /* uncorrectable */
		} else {
			mtd->ecc_stats.corrected += stat;
			max_bitflips = max_t(unsigned int, max_bitflips, stat);
		}
	}
	return max_bitflips;
}
```

`mtd->ecc_stats` is what `/sys/class/mtd/mtdN/{corrected_bits,ecc_failures}` reports — the running tally that §T.10 says to monitor.

### 1.4 UBI's wear levelling

```c
struct ubi_wl_entry {
	union {
		struct rb_node rb;
		struct list_head list;
	} u;
	int ec;          /* THE erase counter */
	int pnum;        /* physical eraseblock number */
};

struct ubi_device {
	...
	struct rb_root used;       /* PEBs holding data */
	struct rb_root erroneous;
	struct rb_root free;       /* PEBs ready to use, sorted by ec */
	int free_count;
	struct rb_root scrub;      /* PEBs that need scrubbing (T.4) */
	struct list_head pq[UBI_PROT_QUEUE_LEN];
	int pq_head;
	...
	int max_ec;
	int mean_ec;
	...
};
```

The core decision:

```c
/* Is the erase-counter spread large enough to justify moving data? */
#define UBI_WL_THRESHOLD CONFIG_MTD_UBI_WL_THRESHOLD   /* default 4096 */

static int wear_leveling_worker(struct ubi_device *ubi,
				struct ubi_work *wrk, int shutdown)
{
	int err, scrubbing = 0, torture = 0, protect = 0, erroneous = 0;
	struct ubi_wl_entry *e1, *e2;
	...
	if (!ubi->scrub.rb_node) {
		/* No scrubbing needed. Is the wear spread too large? */
		e1 = rb_entry(rb_first(&ubi->used), struct ubi_wl_entry, u.rb);
		e2 = get_peb_for_wl(ubi);
		if (!e2) goto out_cancel;

		if (!(e2->ec - e1->ec >= UBI_WL_THRESHOLD)) {
			/* The spread is small; nothing to do. */
			dbg_wl("no WL needed: min used EC %d, max free EC %d",
			       e1->ec, e2->ec);
			...
			goto out_cancel;
		}
		self_check_in_wl_tree(ubi, e1, &ubi->used);
		rb_erase(&e1->u.rb, &ubi->used);
		dbg_wl("move PEB %d EC %d to PEB %d EC %d",
		       e1->pnum, e1->ec, e2->pnum, e2->ec);
	} else {
		/* T.4: a PEB reported -EUCLEAN; move its data and scrub it. */
		scrubbing = 1;
		e1 = rb_entry(rb_first(&ubi->scrub), struct ubi_wl_entry, u.rb);
		e2 = get_peb_for_wl(ubi);
		...
	}
	...
	err = ubi_eba_copy_leb(ubi, e1->pnum, e2->pnum, vid_hdr);
	...
}
```

**Two triggers, one worker**: the erase-counter spread exceeding a threshold (static wear levelling, §T.5) and a PEB reporting bitflips (scrubbing, §T.4). Both result in "copy this LEB to a different PEB."

The **protection queue** (`pq`) is a subtlety worth noting: a PEB that has just been written is put on a short queue so it is not immediately selected for wear levelling again. Without it, a PEB could ping-pong.

Atomic LEB change:

```c
int ubi_eba_atomic_leb_change(struct ubi_device *ubi, struct ubi_volume *vol,
			      int lnum, const void *buf, int len)
{
	...
	/* Get a spare PEB */
	pnum = ubi_wl_get_peb(ubi);
	...
	/* Write the VID header and the data to the NEW PEB */
	err = ubi_io_write_vid_hdr(ubi, pnum, vidb);
	...
	err = ubi_io_write_data(ubi, buf, pnum, 0, len);
	...
	/* ONLY NOW update the mapping. This is the atomic point. */
	down_read(&ubi->fm_eba_sem);
	vol->eba_tbl->entries[lnum].pnum = pnum;
	up_read(&ubi->fm_eba_sem);

	/* The old PEB can now be scheduled for erase. */
	if (old_pnum >= 0) {
		err = ubi_wl_put_peb(ubi, vol_id, lnum, old_pnum, 0);
		...
	}
	...
}
```

Write the new copy, then switch the pointer — **the same structure as btrfs's commit (Ch. 59 §T.2) and as write-ahead logging (Ch. 61 §T.2)**, at the erase-block level.

### 1.5 UBIFS's structures

```c
struct ubifs_sb_node {
	struct ubifs_ch ch;
	__u8 padding[2];
	__u8 key_hash;
	__u8 key_fmt;
	__le32 flags;
	__le32 min_io_size;
	__le32 leb_size;
	__le32 leb_cnt;
	__le32 max_leb_cnt;
	__le64 max_bud_bytes;
	__le32 log_lebs;
	__le32 lpt_lebs;
	__le32 orph_lebs;
	__le32 jhead_cnt;
	__le32 fanout;
	__le32 lsave_cnt;
	__le32 fmt_version;
	__le16 default_compr;
	...
};

struct ubifs_mst_node {              /* the master node: the root of everything */
	struct ubifs_ch ch;
	__le64 highest_inum;
	__le64 cmt_no;
	__le32 flags;
	__le32 log_lnum;
	__le32 root_lnum, root_offs, root_len;
	__le32 gc_lnum;
	__le32 ihead_lnum, ihead_offs;
	__le64 index_size;
	__le64 total_free, total_dirty, total_used, total_dead, total_dark;
	__le32 lpt_lnum, lpt_offs;
	__le32 nhead_lnum, nhead_offs;
	__le32 ltab_lnum, ltab_offs;
	__le32 lsave_lnum, lsave_offs;
	__le32 lscan_lnum;
	__le32 empty_lebs, idx_lebs;
	__le32 leb_cnt;
	...
};

/* Every node begins with this */
struct ubifs_ch {
	__le32 magic;        /* UBIFS_NODE_MAGIC */
	__le32 crc;          /* over the rest of the node */
	__le64 sqnum;        /* sequence number: ordering */
	__le32 len;
	__u8 node_type;
	__u8 group_type;
	__u8 padding[2];
};
```

**Every node is CRC-protected and sequence-numbered.** The CRC catches corruption that ECC missed (or that happened elsewhere); the sequence number establishes ordering during log replay. This is the end-to-end argument (Ch. 00 §T.4) applied to a filesystem that knows its storage is unreliable — exactly the discipline Ch. 61 §T.9 recommends for applications.

The key format:

```c
/* UBIFS keys: (inode number, type, offset-or-hash) */
#define UBIFS_INO_KEY	0
#define UBIFS_DATA_KEY	1
#define UBIFS_DENT_KEY	2
#define UBIFS_XENT_KEY	3
```

Compare with btrfs's `(objectid, type, offset)` (Ch. 59 §T.3): the same insight — one uniform key type, with the type byte selecting the meaning of the third field, so one B+tree serves everything.

### 1.6 Observability

| Where | What |
|---|---|
| `/proc/mtd` ★★★ | partitions, sizes, erase sizes |
| `/sys/class/mtd/mtdN/` ★★★ | `type`, `size`, `erasesize`, `writesize`, `oobsize`, `ecc_strength`, `ecc_step_size`, `bitflip_threshold`, `corrected_bits`, `ecc_failures`, `bad_blocks`, `bbt_blocks` |
| `mtdinfo -a` ★★★ | everything, readably |
| `nanddump`, `nandwrite`, `nandtest` ★★★ | |
| `flash_erase`, `flash_eraseall` | |
| `mtd_debug read/write/erase/info` | low-level |
| `ubinfo -a` ★★★ | UBI devices, volumes, bad/reserved PEBs |
| `/sys/class/ubi/ubiN/` ★★★ | `max_ec`, `bad_peb_count`, `reserved_for_bad`, `volumes_count` |
| `/sys/class/ubi/ubiN_M/` | per-volume |
| `ubihealthd` | a daemon that periodically scrubs |
| `mmc extcsd read` ★★★ | §T.8's eMMC health |
| `mmc-utils` | `mmc status`, `mmc writeprotect`, `mmc bkops` |
| `dmesg` ★★★ | ECC errors, bad blocks, UBI attach messages |
| `nandsim` ★★★ | the simulator, with error injection |

`/sys/class/ubi/ubi0/max_ec` versus the mean is the wear-levelling health metric: a large spread means wear levelling is not keeping up or the device is nearly full.

---

## 2. Practice

### Lab 70.1 — Simulate NAND

No hardware required.

```sh
sudo apt install -y mtd-utils
sudo modprobe nandsim \
  first_id_byte=0x2c second_id_byte=0xda \
  third_id_byte=0x90 fourth_id_byte=0x95
# 0x2c 0xda = Micron 256MB, 2048+64 page, 128KB erase block

cat /proc/mtd
sudo mtdinfo -a
```

```
mtd0
Name:                           NAND simulator partition 0
Type:                           nand
Eraseblock size:                131072 bytes, 128.0 KiB
Amount of eraseblocks:          2048 (268435456 bytes, 256.0 MiB)
Minimum input/output unit size: 2048 bytes
Sub-page size:                  512 bytes
OOB size:                       64 bytes
Character device major/minor:   90:0
Bad blocks are allowed:         true
Device is writable:             true
```

**Read every line against §T.3.** `writesize` (2048) is the page; `erasesize` (131072) is the erase block; `oobsize` (64) is the spare area.

```sh
for f in /sys/class/mtd/mtd0/*; do
  [ -f "$f" ] && printf "%-22s %s\n" "$(basename $f)" "$(cat $f 2>/dev/null)"
done
```

The fundamental operations:

```sh
# You CANNOT just write; you must erase first (T.1)
sudo dd if=/dev/urandom of=/tmp/data.bin bs=2048 count=64 2>/dev/null

sudo flash_erase /dev/mtd0 0 1          # erase one block
sudo nanddump -l 2048 -f /tmp/erased.bin /dev/mtd0
hexdump -C /tmp/erased.bin | head -3
# All 0xff: erase sets bits to 1 (T.1)

sudo nandwrite -p /dev/mtd0 /tmp/data.bin
sudo nanddump -l 131072 -f /tmp/readback.bin /dev/mtd0
cmp /tmp/data.bin <(head -c $((2048*64)) /tmp/readback.bin) && echo "match"

# Write again WITHOUT erasing: bits can only go 1->0
sudo nandwrite -p /dev/mtd0 /tmp/data.bin 2>&1 | tail -2
```

Try to write a partial page:

```sh
sudo dd if=/dev/urandom of=/tmp/small.bin bs=100 count=1 2>/dev/null
sudo nandwrite /dev/mtd0 /tmp/small.bin 2>&1 | tail -2
# "Data size is not page aligned" -- T.1's write granularity
```

The OOB area:

```sh
sudo nanddump --oob -l 2048 -f /tmp/withoob.bin /dev/mtd0
ls -l /tmp/withoob.bin              # 2048 + 64
sudo nanddump --oob --bb=dumpbad -l 4096 /dev/mtd0 2>/dev/null | head -20
```

---

### Lab 70.2 — Bad blocks

```sh
sudo modprobe -r nandsim
sudo modprobe nandsim \
  first_id_byte=0x2c second_id_byte=0xda \
  third_id_byte=0x90 fourth_id_byte=0x95 \
  badblocks=100,200,300,1000

cat /sys/class/mtd/mtd0/bad_blocks
sudo nanddump --bb=dumpbad -l 131072 -s $((100*131072)) /dev/mtd0 2>&1 | head -3
```

Scan for them:

```sh
cat > scanbad.sh <<'EOF'
#!/bin/sh
DEV=$1
ESIZE=$(cat /sys/class/mtd/$(basename $DEV)/erasesize)
COUNT=$(( $(cat /sys/class/mtd/$(basename $DEV)/size) / ESIZE ))
BAD=0
for i in $(seq 0 $((COUNT-1))); do
  if ! nanddump -l 1 -s $((i * ESIZE)) -f /dev/null $DEV 2>/dev/null; then
    echo "bad block at $i (offset $((i * ESIZE)))"
    BAD=$((BAD+1))
  fi
done
echo "$BAD bad blocks of $COUNT"
EOF
chmod +x scanbad.sh
sudo ./scanbad.sh /dev/mtd0 2>/dev/null | tail -8
```

Mark one bad by hand:

```sh
sudo nandtest -m /dev/mtd0 2>/dev/null | head -5
# Or via the ioctl:
cat > markbad.c <<'EOF'
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/ioctl.h>
#include <unistd.h>
#include <mtd/mtd-user.h>

int main(int argc, char **argv) {
	int fd = open(argv[1], O_RDWR);
	loff_t off = atoll(argv[2]);
	int bad;

	if (ioctl(fd, MEMGETBADBLOCK, &off) < 0) perror("MEMGETBADBLOCK");
	else printf("block at %lld: %s\n", (long long)off,
		    ioctl(fd, MEMGETBADBLOCK, &off) ? "BAD" : "good");

	if (argc > 3 && atoi(argv[3])) {
		if (ioctl(fd, MEMSETBADBLOCK, &off) < 0) perror("MEMSETBADBLOCK");
		else printf("marked bad\n");
	}
	close(fd);
	return 0;
}
EOF
gcc -o markbad markbad.c
sudo ./markbad /dev/mtd0 $((100 * 131072))
sudo ./markbad /dev/mtd0 $((500 * 131072))
sudo ./markbad /dev/mtd0 $((500 * 131072)) 1     # mark it
sudo ./markbad /dev/mtd0 $((500 * 131072))
cat /sys/class/mtd/mtd0/bad_blocks
```

**The destructive mistake** (§T.10 point 3):

```sh
# NEVER do this on real hardware:
# sudo dd if=/dev/zero of=/dev/mtd0        # destroys the bad block markers
# sudo flash_erase -N /dev/mtd0 0 0        # -N = don't skip bad blocks
#
# The correct forms:
sudo flash_erase /dev/mtd0 0 0             # skips bad blocks
sudo nandwrite -p -m /dev/mtd0 /tmp/data.bin   # -m = skip bad blocks
```

---

### Lab 70.3 — ECC and bitflips

```sh
cat /sys/class/mtd/mtd0/ecc_strength
cat /sys/class/mtd/mtd0/ecc_step_size
cat /sys/class/mtd/mtd0/bitflip_threshold
cat /sys/class/mtd/mtd0/{corrected_bits,ecc_failures}
dmesg | grep -i ecc | tail -5
```

Inject bit errors:

```sh
sudo modprobe -r nandsim
sudo modprobe nandsim \
  first_id_byte=0x2c second_id_byte=0xda \
  third_id_byte=0x90 fourth_id_byte=0x95 \
  bitflips=1

sudo flash_erase /dev/mtd0 0 10
sudo dd if=/dev/urandom of=/tmp/test.bin bs=2048 count=100 2>/dev/null
sudo nandwrite -p /dev/mtd0 /tmp/test.bin

BEFORE=$(cat /sys/class/mtd/mtd0/corrected_bits)
sudo nanddump -l $((2048*100)) -f /tmp/back.bin /dev/mtd0 2>&1 | tail -3
AFTER=$(cat /sys/class/mtd/mtd0/corrected_bits)
echo "corrected bits: $((AFTER - BEFORE))"
cmp /tmp/test.bin /tmp/back.bin && echo "data CORRECT despite bitflips"
```

**That is ECC working**: bits flipped, ECC corrected them, the data is right, and the counter recorded it.

Push past the correction limit:

```sh
sudo modprobe -r nandsim
sudo modprobe nandsim \
  first_id_byte=0x2c second_id_byte=0xda \
  third_id_byte=0x90 fourth_id_byte=0x95 \
  bitflips=10           # more than the ECC can correct

sudo flash_erase /dev/mtd0 0 5
sudo nandwrite -p /dev/mtd0 /tmp/test.bin 2>&1 | tail -2
sudo nanddump -l $((2048*50)) -f /tmp/bad.bin /dev/mtd0 2>&1 | tail -5
cat /sys/class/mtd/mtd0/ecc_failures
cmp /tmp/test.bin /tmp/bad.bin 2>&1 | head -2
dmesg | tail -10
```

`-EBADMSG`: the data is gone. **That is what happens when ECC is too weak for the NAND** — §T.4's most common embedded bug.

Watch the `-EUCLEAN` threshold:

```sh
cat > readtest.c <<'EOF'
#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <mtd/mtd-user.h>

int main(int argc, char **argv) {
	int fd = open(argv[1], O_RDONLY);
	struct mtd_info_user info;
	char *buf;
	int i, clean = 0, euclean = 0, ebadmsg = 0, other = 0;

	ioctl(fd, MEMGETINFO, &info);
	buf = malloc(info.writesize);

	for (i = 0; i < 200; i++) {
		ssize_t n = pread(fd, buf, info.writesize,
				  (off_t)i * info.writesize);
		if (n == info.writesize) clean++;
		else if (errno == EUCLEAN) euclean++;
		else if (errno == EBADMSG) ebadmsg++;
		else other++;
	}
	printf("clean=%d EUCLEAN=%d EBADMSG=%d other=%d\n",
	       clean, euclean, ebadmsg, other);
	return 0;
}
EOF
gcc -o readtest readtest.c
sudo ./readtest /dev/mtd0
```

Different ECC configurations:

```sh
dmesg | grep -iE 'ecc|bch|hamming' | tail -10
grep -r CONFIG_MTD_NAND_ECC /boot/config-$(uname -r)
ls /sys/class/mtd/mtd0/ | grep -i ecc
```

---

### Lab 70.4 — UBI

```sh
sudo modprobe -r nandsim 2>/dev/null
sudo modprobe nandsim \
  first_id_byte=0x2c second_id_byte=0xda \
  third_id_byte=0x90 fourth_id_byte=0x95 \
  badblocks=50,150,400

sudo flash_erase /dev/mtd0 0 0
sudo modprobe ubi
sudo ubiformat /dev/mtd0 -y
sudo ubiattach -m 0
sudo ubinfo -a
```

```
ubi0
Volumes count:                           0
Logical eraseblock size:                 126976 bytes, 124.0 KiB
Total amount of logical eraseblocks:     2028 (257556480 bytes, 245.6 MiB)
Amount of available logical eraseblocks: 2008 (254967808 bytes, 243.2 MiB)
Maximum count of volumes                 128
Count of bad physical eraseblocks:       3
Count of reserved physical eraseblocks:  37
Current maximum erase counter value:     1
```

Note the overhead (§T.6): 2048 PEBs become 2028 LEBs, of which 2008 are available — UBI reserved blocks for bad-block replacement and its own metadata. The LEB size (126976) is the PEB size (131072) minus two headers.

Volumes:

```sh
sudo ubimkvol /dev/ubi0 -N rootfs -s 100MiB
sudo ubimkvol /dev/ubi0 -N data -m
sudo ubinfo -a
ls -l /dev/ubi0_*

sudo ubirsvol /dev/ubi0 -N rootfs -s 150MiB
sudo ubinfo /dev/ubi0 -N rootfs
```

Wear levelling, observed:

```sh
cat /sys/class/ubi/ubi0/max_ec
cat /sys/class/ubi/ubi0/{bad_peb_count,reserved_for_bad,volumes_count}

# Hammer one volume
for i in $(seq 1 30); do
  sudo dd if=/dev/urandom bs=1M count=50 2>/dev/null | \
    sudo ubiupdatevol /dev/ubi0_0 -s 52428800 -
done
cat /sys/class/ubi/ubi0/max_ec
```

**The max erase counter climbs, but UBI spreads the writes** — the wear is distributed rather than concentrated. Compare with what `mtdblock` would do.

Watch it work:

```sh
sudo bpftrace -e '
kprobe:wear_leveling_worker { @wl = count(); }
kprobe:ubi_eba_copy_leb     { @copy = count(); }
kprobe:ubi_wl_scrub_peb     { @scrub = count(); }
interval:s:10 { print(@wl); print(@copy); print(@scrub);
                clear(@wl); clear(@copy); clear(@scrub); }' &

for i in $(seq 1 20); do
  sudo dd if=/dev/urandom bs=1M count=30 2>/dev/null | \
    sudo ubiupdatevol /dev/ubi0_0 -s 31457280 -
done
```

Atomic LEB change — §T.6:

```sh
cat > ubiatomic.c <<'EOF'
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <sys/ioctl.h>
#include <unistd.h>
#include <mtd/ubi-user.h>

int main(int argc, char **argv) {
	int fd = open(argv[1], O_RDWR);
	int32_t req[2] = { 0, 4096 };       /* LEB 0, 4096 bytes */
	char buf[4096];

	memset(buf, 'A', sizeof(buf));
	if (ioctl(fd, UBI_IOCEBCH, req) < 0) { perror("UBI_IOCEBCH"); return 1; }
	if (write(fd, buf, sizeof(buf)) != sizeof(buf)) { perror("write"); return 1; }
	printf("atomic LEB change complete\n");
	return 0;
}
EOF
gcc -o ubiatomic ubiatomic.c
sudo ./ubiatomic /dev/ubi0_1
```

Fastmap and attach time:

```sh
sudo ubidetach -m 0
time sudo ubiattach -m 0                   # full scan
sudo ubidetach -m 0
grep CONFIG_MTD_UBI_FASTMAP /boot/config-$(uname -r)
time sudo ubiattach -m 0 -f                # with fastmap, if built
```

Scrubbing:

```sh
which ubihealthd && sudo ubihealthd /dev/ubi0 &
cat /sys/class/ubi/ubi0/* 2>/dev/null | head
```

---

### Lab 70.5 — UBIFS

```sh
sudo mkfs.ubifs -r /dev/null -m 2048 -e 126976 -c 800 /dev/ubi0_0 2>/dev/null || \
  sudo ubiupdatevol /dev/ubi0_0 -t

sudo mkdir -p /mnt/ubifs
sudo mount -t ubifs ubi0_0 /mnt/ubifs
df -h /mnt/ubifs
mount | grep ubifs
```

Use it:

```sh
sudo cp -r /usr/include /mnt/ubifs/ 2>/dev/null
df -h /mnt/ubifs
sudo du -sh /mnt/ubifs
```

Compression — §T.7:

```sh
sudo umount /mnt/ubifs
for c in none lzo zlib zstd; do
  sudo ubiupdatevol /dev/ubi0_0 -t
  sudo mkfs.ubifs -r /usr/include -m 2048 -e 126976 -c 800 \
       -x $c -o /tmp/ubifs-$c.img 2>/dev/null
  echo -n "$c: "; ls -lh /tmp/ubifs-$c.img | awk '{print $5}'
done
```

Mount and measure:

```sh
sudo ubiupdatevol /dev/ubi0_0 /tmp/ubifs-zstd.img
sudo mount -t ubifs ubi0_0 /mnt/ubifs
ls /mnt/ubifs | head
df -h /mnt/ubifs
sudo mount -o remount,compr=none /mnt/ubifs 2>/dev/null
mount | grep ubifs
```

Free-space reporting — §T.7(a):

```sh
df -h /mnt/ubifs
sudo mount -o remount,rw /mnt/ubifs
# Write compressible data and watch reported free space behave oddly
sudo sh -c 'yes "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA" | head -c 20M > /mnt/ubifs/compressible'
df -h /mnt/ubifs
sudo dd if=/dev/urandom of=/mnt/ubifs/random bs=1M count=20 2>/dev/null
df -h /mnt/ubifs
sudo rm /mnt/ubifs/compressible /mnt/ubifs/random
```

Power-fail behaviour:

```sh
sudo dd if=/dev/urandom of=/mnt/ubifs/testfile bs=1M count=10 2>/dev/null &
DD=$!
sleep 0.5
sudo umount -f /mnt/ubifs 2>/dev/null   # abrupt
kill $DD 2>/dev/null

sudo mount -t ubifs ubi0_0 /mnt/ubifs
dmesg | grep -i ubifs | tail -10
ls -l /mnt/ubifs/testfile 2>/dev/null
# UBIFS replays its journal; the filesystem is consistent.
```

Compare with the read-only path — §T.7's recommendation:

```sh
sudo umount /mnt/ubifs
sudo modprobe ubiblock
sudo ubiblock --create /dev/ubi0_1 2>/dev/null
ls -l /dev/ubiblock*

sudo mksquashfs /usr/include /tmp/root.sqfs -comp zstd -noappend 2>/dev/null | tail -3
ls -lh /tmp/root.sqfs
sudo ubiupdatevol /dev/ubi0_1 /tmp/root.sqfs
sudo mount -t squashfs -o ro /dev/ubiblock0_1 /mnt/ubifs
ls /mnt/ubifs | head
df -h /mnt/ubifs
sudo umount /mnt/ubifs
```

**SquashFS on ubiblock is smaller, faster, and cannot be corrupted by a power failure** — for a read-only root it is strictly better than UBIFS.

EROFS, the modern alternative:

```sh
sudo apt install -y erofs-utils 2>/dev/null
mkfs.erofs -zlz4hc /tmp/root.erofs /usr/include 2>/dev/null
ls -lh /tmp/root.sqfs /tmp/root.erofs
sudo ubiupdatevol /dev/ubi0_1 /tmp/root.erofs
sudo mount -t erofs -o ro /dev/ubiblock0_1 /mnt/ubifs 2>/dev/null && {
  ls /mnt/ubifs | head -3
  sudo umount /mnt/ubifs
}
```

---

### Lab 70.6 — Why `mtdblock` is dangerous

```sh
sudo modprobe mtdblock
ls -l /dev/mtdblock0

sudo umount /mnt/ubifs 2>/dev/null
sudo ubidetach -m 0 2>/dev/null
sudo flash_erase /dev/mtd0 0 0

sudo mkfs.ext4 -qF /dev/mtdblock0 2>&1 | tail -2
sudo mkdir -p /mnt/mtdb && sudo mount /dev/mtdblock0 /mnt/mtdb
sudo cp -r /usr/include /mnt/mtdb/ 2>/dev/null
```

Watch the erase amplification:

```sh
sudo bpftrace -e '
kprobe:mtd_erase { @erases = count(); }
kprobe:mtd_write { @writes = count(); }
kprobe:mtd_read  { @reads = count(); }
interval:s:5 { print(@erases); print(@writes); print(@reads);
               clear(@erases); clear(@writes); clear(@reads); }' &

sudo sh -c 'for i in $(seq 1 100); do echo x > /mnt/mtdb/f$i; sync; done'
```

**Every small write erases and rewrites a whole 128 KiB block.** 100 small files could be 100 erases of the same block. Now compare:

```sh
sudo umount /mnt/mtdb
sudo flash_erase /dev/mtd0 0 0
sudo ubiformat /dev/mtd0 -y && sudo ubiattach -m 0
sudo ubimkvol /dev/ubi0 -N test -m
sudo mkfs.ubifs -r /dev/null -m 2048 -e 126976 -c 800 /dev/ubi0_0 2>/dev/null
sudo mount -t ubifs ubi0_0 /mnt/mtdb

sudo bpftrace -e '
kprobe:mtd_erase { @erases = count(); }
kprobe:mtd_write { @writes = count(); }
interval:s:5 { print(@erases); print(@writes); clear(@erases); clear(@writes); }' &

sudo sh -c 'for i in $(seq 1 100); do echo x > /mnt/mtdb/f$i; sync; done'
```

UBIFS's log structure means far fewer erases, and UBI spreads them. **That difference is the device's lifetime.**

The `mtdblock` source says so itself:

```sh
head -30 /usr/src/linux*/drivers/mtd/mtdblock.c 2>/dev/null
# Or read it online: the comments are explicit about the limitations.
```

---

### Lab 70.7 — eMMC and SD

On real hardware:

```sh
ls /sys/class/mmc_host/
ls /dev/mmcblk*
lsblk | grep mmc

for h in /sys/class/mmc_host/mmc*/; do
  echo "=== $(basename $h) ==="
  for d in $h/mmc*/; do
    [ -d "$d" ] || continue
    for f in type name manfid oemid serial date fwrev hwrev \
             life_time pre_eol_info ocr cid csd; do
      printf "  %-14s %s\n" $f "$(cat $d/$f 2>/dev/null | head -c 40)"
    done
  done
done
```

Health — §T.8:

```sh
sudo apt install -y mmc-utils
sudo mmc extcsd read /dev/mmcblk0 | grep -iE 'life|eol|sec_count|cache|hpi|bkops'
```

```
eMMC Life Time Estimation A [EXT_CSD_DEVICE_LIFE_TIME_EST_TYP_A]: 0x01
eMMC Life Time Estimation B [EXT_CSD_DEVICE_LIFE_TIME_EST_TYP_B]: 0x02
eMMC Pre EOL information [EXT_CSD_PRE_EOL_INFO]: 0x01
```

| Value | Meaning |
|---|---|
| Life time `0x01` | 0–10 % of rated life used |
| Life time `0x0B` | 100 % — **past rated endurance** |
| Pre-EOL `0x01` | normal |
| Pre-EOL `0x02` | **warning: 80 % of reserved blocks consumed** |
| Pre-EOL `0x03` | **urgent** |

This is the only health signal eMMC provides, and checking it is the only way to predict failure.

Boot partitions and RPMB:

```sh
ls -l /dev/mmcblk0*
# mmcblk0, mmcblk0boot0, mmcblk0boot1, mmcblk0rpmb
sudo mmc bootpart enable 1 1 /dev/mmcblk0 2>/dev/null
sudo mmc extcsd read /dev/mmcblk0 | grep -iE 'boot|rpmb'
cat /sys/block/mmcblk0boot0/force_ro
```

Write amplification — §T.8's core problem:

```sh
DEV=/dev/mmcblk0    # a SCRATCH device, not your root
cat /sys/block/mmcblk0/queue/{logical_block_size,physical_block_size,discard_granularity,optimal_io_size}

for bs in 4k 64k 512k 4m; do
  echo -n "bs=$bs random write: "
  sudo fio --name=t --filename=$DEV --direct=1 --rw=randwrite --bs=$bs \
           --size=100m --runtime=10 --time_based 2>/dev/null | grep -oP 'IOPS=\K[^,]+'
done
echo -n "sequential write:    "
sudo fio --name=t --filename=$DEV --direct=1 --rw=write --bs=1M \
         --size=100m --runtime=10 --time_based 2>/dev/null | grep -oP 'BW=\K[^ ]+'
```

The ratio between 4k random and 1M sequential on an SD card is often 1000:1. **That is the erase-block-granularity FTL of §T.8.**

Filesystem choice:

```sh
for fs in ext4 f2fs; do
  sudo mkfs.$fs -qf $DEV 2>/dev/null || sudo mkfs.$fs -qF $DEV 2>/dev/null
  sudo mount $DEV /mnt/mmc
  echo -n "$fs: "
  sudo fio --name=t --directory=/mnt/mmc --size=50m --rw=randwrite --bs=4k \
           --fsync=16 --runtime=15 --time_based 2>/dev/null | grep -oP 'IOPS=\K[^,]+'
  sudo umount /mnt/mmc
done
```

Alignment:

```sh
sudo fdisk -l $DEV | head
# Partitions should start at a multiple of the erase block size (often 4MB).
sudo parted -s $DEV mklabel gpt mkpart p1 ext4 4MiB 100%
sudo partprobe $DEV
cat /sys/block/mmcblk0/mmcblk0p1/alignment_offset
```

Command queueing and speed modes:

```sh
dmesg | grep -i mmc | grep -iE 'hs200|hs400|ddr|cqe|speed' | head
cat /sys/kernel/debug/mmc0/ios 2>/dev/null
cat /sys/class/mmc_host/mmc0/mmc0:0001/* 2>/dev/null | head -20
```

---

### Lab 70.8 — Bring-up checklist

Work through §T.10's list on the simulator, then know what to do on real hardware.

```sh
# 1. GEOMETRY
sudo mtdinfo -a
cat /sys/class/mtd/mtd0/{writesize,erasesize,oobsize,size}
# Must match the datasheet EXACTLY.

# 2. ECC
cat /sys/class/mtd/mtd0/{ecc_strength,ecc_step_size}
dmesg | grep -iE 'ecc|bch'
# Must be >= the manufacturer's requirement.
```

A verification script:

```sh
cat > flashcheck.sh <<'EOF'
#!/bin/bash
MTD=${1:-mtd0}
S=/sys/class/mtd/$MTD

echo "=== Geometry ==="
printf "  page size:     %s\n" "$(cat $S/writesize)"
printf "  erase size:    %s\n" "$(cat $S/erasesize)"
printf "  OOB size:      %s\n" "$(cat $S/oobsize)"
printf "  total size:    %s\n" "$(cat $S/size)"
printf "  blocks:        %s\n" "$(( $(cat $S/size) / $(cat $S/erasesize) ))"

echo "=== ECC ==="
printf "  strength:      %s bits per %s bytes\n" \
       "$(cat $S/ecc_strength)" "$(cat $S/ecc_step_size)"
printf "  threshold:     %s\n" "$(cat $S/bitflip_threshold)"
printf "  corrected:     %s\n" "$(cat $S/corrected_bits)"
printf "  failures:      %s\n" "$(cat $S/ecc_failures)"

echo "=== Bad blocks ==="
printf "  bad:           %s\n" "$(cat $S/bad_blocks)"
printf "  bbt blocks:    %s\n" "$(cat $S/bbt_blocks)"
BAD=$(cat $S/bad_blocks)
TOT=$(( $(cat $S/size) / $(cat $S/erasesize) ))
printf "  bad fraction:  %.2f%%\n" "$(echo "scale=4; $BAD * 100 / $TOT" | bc)"
[ "$BAD" -gt "$((TOT / 50))" ] && echo "  ** WARNING: >2% bad blocks **"

if [ -d /sys/class/ubi/ubi0 ]; then
  echo "=== UBI ==="
  printf "  max erase ct:  %s\n" "$(cat /sys/class/ubi/ubi0/max_ec)"
  printf "  bad PEBs:      %s\n" "$(cat /sys/class/ubi/ubi0/bad_peb_count)"
  printf "  reserved:      %s\n" "$(cat /sys/class/ubi/ubi0/reserved_for_bad)"
  RES=$(cat /sys/class/ubi/ubi0/reserved_for_bad)
  BADP=$(cat /sys/class/ubi/ubi0/bad_peb_count)
  [ "$BADP" -gt "$((RES / 2))" ] && \
    echo "  ** WARNING: over half the bad-block reserve consumed **"
fi

echo "=== Recent errors ==="
dmesg | grep -iE 'ecc|bad block|EUCLEAN|EBADMSG|ubi|ubifs.*error' | tail -8
EOF
chmod +x flashcheck.sh
sudo ./flashcheck.sh mtd0
```

Endurance testing:

```sh
# Write and verify repeatedly, watching the counters
sudo nandtest -k -p 20 /dev/mtd0 2>&1 | tail -10
sudo ./flashcheck.sh mtd0
```

A stress test with the simulator's wear model:

```sh
sudo modprobe -r nandsim
sudo modprobe nandsim \
  first_id_byte=0x2c second_id_byte=0xda \
  third_id_byte=0x90 fourth_id_byte=0x95 \
  bitflips=1 badblocks=

sudo ubiformat /dev/mtd0 -y && sudo ubiattach -m 0
sudo ubimkvol /dev/ubi0 -N stress -m
sudo mkfs.ubifs -r /dev/null -m 2048 -e 126976 -c 1900 /dev/ubi0_0 2>/dev/null
sudo mount -t ubifs ubi0_0 /mnt/mtdb

for round in $(seq 1 20); do
  sudo dd if=/dev/urandom of=/mnt/mtdb/f bs=1M count=100 2>/dev/null
  sudo rm /mnt/mtdb/f
  sync
  echo -n "round $round: max_ec=$(cat /sys/class/ubi/ubi0/max_ec) "
  echo "corrected=$(cat /sys/class/mtd/mtd0/corrected_bits)"
done
sudo ./flashcheck.sh mtd0
```

Power-fail testing — the one that matters and cannot be simulated well:

```sh
# On real hardware, with a relay or a controllable PSU:
#   1. Start a write loop
#   2. Cut power at a random point
#   3. Restore power, boot, fsck/mount
#   4. Verify data consistency
#   5. Repeat 1000 times
#
# UBI and UBIFS are designed to survive this. Verify that they do on
# YOUR hardware, because partial page programming behaviour varies.
```

Cleanup:

```sh
sudo umount /mnt/mtdb 2>/dev/null
sudo ubidetach -m 0 2>/dev/null
sudo modprobe -r nandsim
```

---

## 3. Mastery drills

1. State the four properties of NAND that make it not a block device. For each, name the software layer that hides it in an SSD and the one that handles it on raw NAND.

2. Compute the OOB space available for filesystem metadata on a 2048+64 page with 4-bit BCH over 512-byte steps, and on a 4096+224 page with 24-bit BCH over 1024-byte steps. Explain what this did to JFFS2.

3. Explain why factory bad-block markers must never be erased, and trace what happens to a device after `dd if=/dev/zero of=/dev/mtd0`.

4. A device specifies 8-bit ECC per 512 bytes and you configure 4-bit. Describe the observed behaviour at day 1, month 3, and month 12, and explain why.

5. Explain static wear levelling and construct the failure that occurs with only dynamic wear levelling, using a concrete device layout.

6. Read disturb, data retention, and program disturb are three distinct effects. For each, state the physical cause, the trigger, and the software mitigation.

7. Derive `-EUCLEAN` from the bitflip count and `bitflip_threshold`. Explain why the default is 75 % of ECC strength and what happens if you set it to 100 %.

8. Explain UBI's LEB→PEB mapping and prove that atomic LEB change survives a power failure at any point.

9. UBI's wear-levelling worker has two triggers. State both, and construct a workload that exercises each.

10. Compare JFFS2 and UBIFS on mount time, RAM usage, and maximum practical device size. Explain the single design decision responsible for all three differences.

11. An SD card maps erase blocks rather than pages. Compute the write amplification for a 4 KiB random write with a 4 MiB mapping granularity, and explain the Raspberry Pi's reputation.

12. `mtdblock` on NAND will eventually destroy the data. Construct the exact sequence, and state the two things UBI does that prevent it.

13. You are bringing up a new board with raw NAND and the bootloader can read what it writes but Linux cannot. Give the ordered diagnostic procedure and the five most likely causes.

---

## 4. Further reading

**Kernel documentation**

- `Documentation/driver-api/mtdnand.rst` ★★★ — **the NAND driver interface, in full.** Geometry, ECC, bad blocks, the controller abstraction.
- `Documentation/filesystems/ubifs.rst` ★★★ and `ubifs-authentication.rst`
- `Documentation/filesystems/jffs2.rst`
- `Documentation/filesystems/squashfs.rst`, `erofs.rst` ★★★
- `Documentation/ABI/testing/sysfs-class-mtd` ★★★ — every attribute of §1.6.
- `Documentation/ABI/testing/sysfs-class-ubi` ★★★
- `Documentation/mmc/` ★★★ — `mmc-dev-attrs.rst`, `mmc-async-req.rst`
- `Documentation/devicetree/bindings/mtd/` — how to describe flash in a device tree.
- `include/linux/mtd/mtd.h` ★★★, `nand.h`, `rawnand.h`, `spi-nor.h`
- `include/uapi/mtd/mtd-abi.h`, `ubi-user.h` ★★★ — the ioctls

**Project documentation**

- **linux-mtd.infradead.org** ★★★ — **the authoritative resource.** The UBI FAQ, the UBIFS FAQ, the NAND FAQ, and the "UBI design" and "UBIFS design" documents are all excellent and are the primary source for §T.6 and §T.7.
- The **UBIFS white paper** (Hunter, "A Brief Introduction to the Design of UBIFS") ★★★ — short and complete.
- The **UBI design document** ★★★ — §T.6 from its authors.
- `mtd-utils` documentation and the `ubiformat`/`ubiattach`/`mkfs.ubifs` man pages.

**Papers and technical notes**

- Woodhouse, "JFFS: The Journalling Flash File System," Ottawa Linux Symposium 2001 ★★★ — the predecessor, and the clearest statement of the problem.
- Hunter, "A Brief Introduction to the Design of UBIFS," 2008 ★★★
- Bityutskiy, "UBI — Unsorted Block Images," 2005 ★★★
- Grupp et al., "Characterizing Flash Memory: Anomalies, Observations, and Applications," MICRO 2009 ★★★ — **measured flash behaviour**, including retention and disturb.
- Cai et al., "Error Characterization, Mitigation, and Recovery in Flash-Memory-Based Solid-State Drives," Proceedings of the IEEE 2017 ★★★ — **the comprehensive survey of §T.5's physics.** Long, and the definitive reference.
- Schroeder, Lagisetty, Merchant, "Flash Reliability in Production: The Expected and the Unexpected," FAST 2016 ★★★ — **Google's field data.** Overturns several common beliefs, including that MLC is meaningfully less reliable than SLC in practice.
- Meza et al., "A Large-Scale Study of Flash Memory Failures in the Field," SIGMETRICS 2015 — Facebook's data.
- Micron, Toshiba/Kioxia, and Samsung NAND technical notes on ECC requirements, bad-block management, and program/erase behaviour — the manufacturers' own documentation is detailed and freely available.
- ONFI (Open NAND Flash Interface) and JEDEC's eMMC (JESD84) specifications.

**LWN**

- "The UBIFS filesystem" ★★★
- "Flash filesystems" and the JFFS2/UBIFS comparisons
- "UBI fastmap"
- "EROFS: the enhanced read-only filesystem" ★★★
- "SquashFS and the read-only filesystem options"
- "Flash storage in the field" coverage of the FAST papers
- "MTD and the block layer" discussions — why `mtdblock` remains what it is
- "Supporting zoned devices" (Ch. 60 §T.5) — the same physics, different abstraction

**Source reading order**

1. The linux-mtd UBI and UBIFS design documents first. Do not start with the code.
2. `include/linux/mtd/mtd.h` ★★★ — §T.2's interface.
3. `drivers/mtd/mtdcore.c`: `mtd_read_oob`, `mtd_write_oob`, `mtd_erase`, `mtd_block_isbad` ★★★
4. `drivers/mtd/nand/raw/nand_base.c`: `nand_do_read_ops`, `nand_do_write_ops`, `nand_erase_nand` ★★★
5. `drivers/mtd/nand/raw/nand_bbt.c` — §T.3's scanning and table management.
6. `drivers/mtd/ubi/wl.c` ★★★ — **§T.6's wear levelling.** ~2000 lines and the heart of UBI.
7. `drivers/mtd/ubi/eba.c`: `ubi_eba_atomic_leb_change`, `ubi_eba_copy_leb` ★★★
8. `drivers/mtd/ubi/attach.c` — the scan that builds the mapping.
9. `fs/ubifs/super.c`, `fs/ubifs/tnc.c` (the tree), `fs/ubifs/journal.c` — §T.7.
10. `drivers/mtd/nand/raw/nandsim.c` — how the simulator works, and what it can inject.

**Tools**

- `mtd-utils` ★★★ — `mtdinfo`, `nanddump`, `nandwrite`, `nandtest`, `flash_erase`, `mtd_debug`
- `ubi-utils` (in mtd-utils) ★★★ — `ubiformat`, `ubiattach`, `ubimkvol`, `ubirsvol`, `ubiupdatevol`, `ubinfo`, `ubiblock`, `ubihealthd`
- `mkfs.ubifs`, `ubinize` ★★★ — building images for flashing
- `nandsim` ★★★ — **the laboratory**; `bitflips=`, `badblocks=`, `weakblocks=`, `weakpages=`, `gravepages=`
- `mmc-utils` ★★★ — `mmc extcsd read`, `mmc status`, `mmc bootpart`, `mmc writeprotect`
- `/sys/class/mtd/`, `/sys/class/ubi/`, `/sys/class/mmc_host/` ★★★
- `mksquashfs`, `mkfs.erofs` ★★★ — for read-only roots
- `flashbench` — empirically determine an SD card's erase block size and page size
- `fio` with small block sizes — the SD-card write-amplification test
- A controllable power supply or relay — **there is no substitute for real power-fail testing**

---

**Part 3 complete.** Chapters 51–70 have covered the storage stack from `write()` to the NAND cell: the page cache and writeback, the VFS and its three layers, writing a filesystem, five production filesystems and the design space they occupy, crash consistency and durability, network and stacking filesystems, the block layer and its schedulers, writing block drivers, device mapper and MD RAID, SCSI/ATA and NVMe, and finally raw flash where every abstraction is stripped away.

The recurring themes worth carrying forward: **the narrow waist** (a small interface that everything above and below agrees on); **indirection to decouple identity from location** (inodes, LEBs, L2P maps, dentries); **intent logged before action** (journals, bitmaps, write-ahead logging); **reserves that break circular dependencies** (mempools, AGFL, global reserve, the system chunk array); **the end-to-end argument** (checksums belong where the meaning is, which is why btrfs and ZFS detect what RAID cannot); and **that every layer which cannot allocate must preallocate**.

→ Next: [../part4-net/71-skb-netdev.md](../part4-net/71-skb-netdev.md)

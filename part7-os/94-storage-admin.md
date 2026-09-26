# Chapter 94 — Storage Administration in Production

> Part 3 taught you how the block layer, filesystems, device mapper, and MD work. This
> chapter is about operating them: choosing a layout, sizing a filesystem, tuning for a
> workload, detecting failure before it is fatal, and recovering when it is. It is the
> chapter that turns Ch. 51–70 into an answer to "how should we store this?"

---

## Theory & First Principles

### T.0 — Start here: "the database is slow" — now find out why

You get a ticket. Latency is up. **There are at least nine places the problem could be, and
the skill being tested is narrowing before measuring.**

```
   application           <- doing fsync per row? 4 KiB random writes?
        |
   page cache            <- dirty_ratio stall? cache thrash? (Ch. 52 §T.0)
        |
   filesystem            <- fragmentation? journal commits? mount options?
        |
   device mapper / LVM   <- a snapshot doing copy-on-write per write?
        |
   MD RAID               <- RAID5 small-write penalty? a rebuild running? (Ch. 67)
        |
   block layer           <- wrong scheduler? queue depth? cgroup throttling?
        |
   driver                <- D-to-C latency (Ch. 65 §T.0)
        |
   device                <- SSD garbage collection? thermal throttling? worn out?
        |
   (if remote)           <- the network, and everything in Ch. 62
```

**The one habit that separates people who solve this from people who guess:**

> **Measure at two adjacent layers and compare.** A number alone means nothing; a *difference*
> localizes the problem. If the application sees 10 ms and `biolatency` sees 100 µs, the
> device is innocent and the problem is above the block layer. If both see 10 ms, go down.

That is bisection applied to a stack instead of to a commit history — and it is why the tools
in this chapter are organized by *layer*, not by name.

**The second habit: know which numbers are lies.**

| Metric | What people think | What it is |
|---|---|---|
| `%util` in `iostat` | "the device is 100% busy, it is saturated" | **meaningless on NVMe** — it measures "at least one request outstanding," and a device with 64 queues can be 100% util at 2% capacity |
| `await` | device latency | queueing **plus** device time; it rises from *your own* queue depth |
| `free` "used" | memory in trouble | cache is not used memory (Ch. 23 §T.0) |
| IOPS | throughput | meaningless without the block size and the queue depth |
| An average latency | typical behaviour | **the p99 is the user experience**; averages hide the tail entirely |

**`%util` is the single most misread number in Linux performance work**, and understanding why
it broke is Ch. 64 §T.0's lesson again: the metric encoded an assumption (one request at a
time) that modern hardware invalidated. **A metric is also a contract with a hardware model.**

**The third habit: the tunables that matter, and the two that bite hardest.**

```bash
# 1. dirty ratios are PERCENTAGES OF RAM by default.
#    On a 512 GiB box, vm.dirty_ratio=20 means 100 GiB of dirty pages --
#    a 50-second stall when it drains (Ch. 52 §T.0). ALWAYS use _bytes on big machines:
sysctl vm.dirty_background_bytes vm.dirty_bytes

# 2. the scheduler. 'none' for NVMe, 'mq-deadline' or 'bfq' for spinning/SATA.
cat /sys/block/*/queue/scheduler

# 3. readahead. Huge for sequential, pure waste (and cache pollution) for random.
blockdev --getra /dev/sda
```

**And the framing to carry into the labs:** every layer in that stack exists to bridge a gap
(Ch. 51 §T.0), and **every layer can therefore be the thing that is in the way.** Performance
work on storage is almost never about making a layer faster; it is about finding the layer
that is doing work you did not want, and telling it not to.

```bash
iostat -xz 1                             # per-device, with queue depth and await
sudo /usr/share/bcc/tools/biolatency -mT 1
sudo /usr/share/bcc/tools/biosnoop | head -40
sudo /usr/share/bcc/tools/ext4slower 10  # filesystem-level slow operations
cat /proc/pressure/io                    # PSI: is anyone actually STALLED?
```

---

### T.1 — The stack, and where to intervene

```
 application
   │  O_DIRECT? fsync discipline? io_uring?          <- Ch. 76
 ─────────────────────────────────────────
 page cache / writeback                              <- Ch. 52
   │  dirty ratios, writeback tuning
 ─────────────────────────────────────────
 filesystem      ext4 / XFS / Btrfs / F2FS           <- Ch. 56-62
   │  journal mode, mount options, allocation
 ─────────────────────────────────────────
 device mapper   dm-crypt / dm-thin / dm-cache       <- Ch. 66
   │  LVM, encryption, snapshots
 ─────────────────────────────────────────
 MD RAID                                             <- Ch. 67
   │  level, chunk size, stripe cache
 ─────────────────────────────────────────
 block layer     blk-mq, scheduler, queue limits     <- Ch. 63-64
   │  nr_requests, read_ahead_kb, scheduler
 ─────────────────────────────────────────
 driver          NVMe / SCSI / ATA / MMC             <- Ch. 68-70
   │  queue depth, timeouts, power states
 ─────────────────────────────────────────
 device          write cache, PLP, over-provisioning
```

**The operational rule: intervene at the highest layer that can solve the problem.** An
application that fsyncs correctly needs no block-layer tuning; a block-layer workaround for
an application that fsyncs per-record is treating a symptom.

**The second rule: every layer must pass durability through.** A flush issued by the
filesystem must reach the device. If any layer lies — a RAID controller with a volatile
cache, a virtualization layer with `cache=unsafe`, a device that ignores FLUSH — the entire
stack above it is built on sand. This is the single most important thing to verify when
someone reports data loss.

### T.2 — Choosing a filesystem

| Workload | Choice | Why |
|---|---|---|
| **General server** | XFS | Parallel metadata (allocation groups), scales to many cores, excellent large-file performance |
| **Boot/root** | ext4 | Boring, universally supported, shrinkable, best `fsck` |
| **Database** | XFS | Predictable latency, no CoW fragmentation, delayed logging. `ext4` if you prefer boring |
| **Many small files** | XFS or ext4 with `dir_index` | XFS's B+tree directories win above ~1M entries |
| **Snapshots, checksums** | Btrfs or ZFS | CoW gives them structurally |
| **Flash/embedded (raw NAND)** | UBIFS | Wear levelling, bad-block management (Ch. 70) |
| **Flash/embedded (eMMC/SD)** | ext4 or F2FS | F2FS is log-structured; matches FTL behaviour |
| **Read-only image** | SquashFS or EROFS | Compressed, immutable; EROFS is the modern choice |
| **Container images** | overlayfs over ext4/XFS | The layering model |
| **Network** | NFSv4.2 / CIFS | (Ch. 62) |

**The three things people get wrong:**

1. **Btrfs for a database.** CoW plus random in-place overwrite equals severe fragmentation
   and write amplification. You must set `nodatacow` on the DB directory — which disables
   checksums, which removes most of the reason you chose Btrfs. Ch. 59 §T.6.
2. **ext4 on a 100-core box with metadata-heavy load.** One journal, one lock. XFS's
   allocation groups exist for exactly this.
3. **Not knowing that XFS cannot shrink.** It can grow online, never shrink. If the volume
   might need to shrink, that decision is made at `mkfs` time.

### T.3 — Sizing and `mkfs` decisions

These are **permanent**. Get them right once.

```bash
# --- ext4 ---
mkfs.ext4 \
	-b 4096 \                 # block size; 4K unless you know better
	-i 16384 \                # bytes per inode: ONE INODE PER 16K.
	                          #   Too few inodes = ENOSPC with free space.
	                          #   Many small files -> lower this.
	-I 256 \                  # inode size; 256 for xattrs/SELinux
	-m 1 \                    # reserved blocks % (default 5 is a lot on big volumes)
	-E stride=16,stripe_width=64 \   # MATCH THE RAID GEOMETRY
	-O ^has_journal \         # no journal: ONLY for scratch/ephemeral
	-L mylabel \
	/dev/sda1

# --- XFS ---
mkfs.xfs \
	-b size=4096 \
	-d su=64k,sw=4 \          # stripe unit and width -- RAID geometry
	-l size=128m,su=64k \     # log size; larger = better metadata throughput
	-i size=512 \             # inode size
	-n size=8192 \            # directory block size; larger for huge dirs
	-m crc=1,finobt=1,reflink=1 \   # metadata CRC, free inode btree, reflinks
	-L mylabel \
	/dev/sda1
```

**The stride/stripe (ext4) and su/sw (XFS) parameters matter enormously on RAID.** If the
filesystem does not know the stripe geometry, it will place metadata so that a single
metadata update triggers a full read-modify-write of a RAID-5 stripe. Getting this right can
be a 2–5× difference on parity RAID.

```
 stride       = chunk_size / block_size            (e.g. 64K / 4K = 16)
 stripe_width = stride × number_of_DATA_disks      (RAID5 with 5 disks: 16 × 4 = 64)
```

`mkfs.xfs` reads the geometry from `md`/LVM automatically; `mkfs.ext4` usually does too, but
**verify** with `dumpe2fs -h` / `xfs_info`.

**Inode exhaustion** is the classic ext4 surprise: `df` shows free space, `df -i` shows 100%
inodes, and writes fail with `ENOSPC`. It is unfixable without a `mkfs`. XFS allocates inodes
dynamically and does not have this problem.

### T.4 — Mount options that matter

```bash
# --- ext4 ---
/dev/sda1 /data ext4 defaults,noatime,data=ordered,commit=5,barrier=1 0 2

data=journal      # metadata AND data journalled. Strongest, ~2x writes
data=ordered      # default: data written BEFORE the metadata commit
data=writeback    # fastest; a crash can expose stale blocks in a file
noatime           # do not update access times. ALWAYS. relatime is the default
                  #   and is fine; noatime is better if nothing needs atime
commit=N          # journal commit interval, seconds (default 5)
barrier=0         # DANGEROUS: disables flush. Only with battery-backed cache
discard           # inline TRIM -- usually WORSE than periodic fstrim
nodelalloc        # disable delayed allocation; rarely right

# --- XFS ---
/dev/sda1 /data xfs defaults,noatime,logbsize=256k,inode64 0 0

logbsize=256k     # larger log buffer -> better metadata throughput
inode64           # default since 3.7; allows inodes above 1TB
allocsize=1m      # preallocation size for streaming writes
nobarrier         # REMOVED in modern kernels. If a guide says this, it is old
```

**The `discard` versus `fstrim` question**, which comes up constantly: inline `discard` issues
a TRIM on every delete, which on many devices is synchronous and slow, and can cause latency
spikes. **Periodic `fstrim` (weekly, via `fstrim.timer`) is almost always better.** Some
modern NVMe devices handle inline discard well; measure before choosing.

### T.5 — LVM and the layering decision

```
 Physical Volume (PV)   /dev/sda1, /dev/sdb1
       ↓
 Volume Group (VG)      vg0  (a pool of extents)
       ↓
 Logical Volume (LV)    lv_root, lv_data, lv_snap
       ↓
 Filesystem
```

**What LVM buys:** online resize, snapshots, moving data between physical devices without
downtime (`pvmove`), thin provisioning, and caching (`dm-cache`/`lvmcache`).

**What it costs:** a layer of indirection, slightly more complex recovery, and snapshots have
a real performance cost (copy-on-write on every first write to a snapshotted extent).

```bash
# The standard setup
pvcreate /dev/sda1 /dev/sdb1
vgcreate vg0 /dev/sda1 /dev/sdb1
lvcreate -L 100G -n lv_data vg0
mkfs.xfs /dev/vg0/lv_data

# Online grow (XFS: grow only)
lvextend -L +50G /dev/vg0/lv_data
xfs_growfs /data                      # or: resize2fs for ext4

# Snapshot for a consistent backup
lvcreate -s -L 10G -n lv_snap /dev/vg0/lv_data
mount -o ro,nouuid /dev/vg0/lv_snap /mnt/snap    # nouuid for XFS!
# ... back up /mnt/snap ...
umount /mnt/snap && lvremove -f /dev/vg0/lv_snap

# Thin provisioning -- CAUTION
lvcreate -L 900G --thinpool tp0 vg0
lvcreate -V 2T -T vg0/tp0 -n lv_thin      # 2T virtual on 900G real
```

**Thin provisioning's failure mode must be understood before use:** when the pool fills, I/O
to any thin volume fails or blocks, and the filesystems on top — which believe they have
space — corrupt or go read-only. Monitor `lvs -o+data_percent,metadata_percent` and alert
well before 100%. `thin_pool_autoextend_threshold` helps. **Metadata exhaustion is worse than
data exhaustion** and is easy to overlook.

### T.6 — Encryption

```bash
# LUKS2: the standard
cryptsetup luksFormat --type luks2 \
	--cipher aes-xts-plain64 \
	--key-size 512 \
	--hash sha256 \
	--pbkdf argon2id \
	--sector-size 4096 \          # match the device; better performance
	/dev/sda1

cryptsetup open /dev/sda1 cryptdata
mkfs.xfs /dev/mapper/cryptdata

# Performance
cryptsetup --perf-no_read_workqueue --perf-no_write_workqueue \
	--allow-discards open /dev/sda1 cryptdata
#   The no-workqueue options remove a scheduling hop; on NVMe they are
#   a significant win. --allow-discards leaks some information about
#   used space -- a deliberate trade.

cryptsetup benchmark            # what your CPU can do
grep -E 'aes|sha' /proc/cpuinfo # AES-NI present? If not, expect 10x slower
```

**The embedded question: where does the key live?** A key in the initramfs or on the
filesystem is not protection against a physical attacker. The answers, in order of strength:
a TPM-sealed key bound to PCR values (so it only unseals if the boot chain is intact), a
secure element, an OTP fuse plus a hardware crypto engine, or a passphrase (which requires a
human, so not for unattended devices). Ch. 90 §T.6 and Ch. 102.

`--allow-discards` deserves a note: without it, TRIM does not pass through and the SSD cannot
garbage-collect, which hurts performance and endurance. With it, an attacker can see which
blocks are in use. For most deployments the trade favours enabling it; for high-security
ones it does not. **Decide deliberately.**

### T.7 — RAID in practice

| Level | Use | Do not use for |
|---|---|---|
| **0** | Scratch, reproducible data | Anything you care about |
| **1** | Boot volumes, small critical data | Capacity efficiency |
| **5** | Archival, read-heavy | **Anything write-heavy; large drives** |
| **6** | Large archival | Write-heavy |
| **10** | Databases, VMs, general | Capacity-constrained budgets |

**The RAID-5 arguments against, which you should be able to give:**

1. **The write hole** (Ch. 67 §T.4): a partial-stripe write must read old data and parity,
   compute, and write both. Power loss between the two leaves an inconsistent stripe that
   cannot be detected. Fixes: battery-backed cache, `md`'s write journal, or CoW filesystems.
2. **Rebuild time and URE risk.** Rebuilding a 20 TB array reads every sector of every
   remaining drive. At a consumer URE rate of 1 in 10¹⁴ bits, a full read of ~12 TB has a
   meaningful chance of hitting one — during which you have no redundancy.
3. **Rebuild duration.** Days on large arrays, during which performance is degraded and a
   second failure is fatal.

**RAID-10 for anything write-heavy or large.** The capacity cost is real; so is the
alternative.

```bash
# Create
mdadm --create /dev/md0 --level=10 --raid-devices=4 --chunk=512K \
	/dev/sd[abcd]1
mdadm --detail --scan >> /etc/mdadm/mdadm.conf
update-initramfs -u            # or dracut -f -- the array must assemble at boot

# Monitor -- SET THIS UP, or you will not know a disk failed
mdadm --monitor --daemonise --mail=ops@example.com /dev/md0
cat /proc/mdstat

# Scrub monthly. This is not optional: it finds latent bad sectors
# BEFORE a rebuild needs them.
echo check > /sys/block/md0/md/sync_action
cat /sys/block/md0/md/mismatch_cnt      # nonzero = investigate

# Tune the rebuild rate
echo 200000 > /proc/sys/dev/raid/speed_limit_min
echo 8192   > /sys/block/md0/md/stripe_cache_size    # RAID5/6 only
```

**Monthly scrubbing is the single highest-value RAID practice.** Latent sector errors
accumulate silently; a scrub finds and rewrites them while you still have redundancy. Most
distributions ship a `mdcheck` timer — verify it is enabled.

### T.8 — Detecting failure before it is fatal

```bash
# --- SMART ---
smartctl -a /dev/sda
smartctl -t long /dev/sda           # a full surface scan; hours
smartctl -l selftest /dev/sda

# The attributes that actually predict failure (Backblaze's data):
#   5   Reallocated_Sector_Ct     <- ANY growth is a warning
#   187 Reported_Uncorrect        <- nonzero is bad
#   188 Command_Timeout
#   197 Current_Pending_Sector    <- sectors that could not be read
#   198 Offline_Uncorrectable
# Raw values matter more than the normalized ones.

# --- NVMe ---
nvme smart-log /dev/nvme0
#   percentage_used     <- endurance consumed, 0-100+
#   media_errors        <- uncorrectable
#   critical_warning    <- a bitmask; nonzero means act now
#   data_units_written  <- x1000 x512 bytes. Compute TBW vs the rating.
nvme error-log /dev/nvme0
nvme id-ctrl /dev/nvme0 | grep -E 'tnvmcap|unvmcap'

# --- Continuous monitoring ---
systemctl enable --now smartd
# /etc/smartd.conf:
#   DEVICESCAN -a -o on -S on -n standby,q -s (S/../.././02|L/../../6/03) -m ops@example.com

# --- Filesystem-level errors ---
dmesg | grep -iE 'I/O error|EXT4-fs error|XFS.*(error|corruption)|medium error'
cat /sys/fs/ext4/*/errors_count
xfs_spaceman -c "health" /mountpoint
```

**Interpreting SMART, honestly:** Backblaze's published data shows five attributes correlate
with imminent failure (5, 187, 188, 197, 198), and that **most drives fail with no SMART
warning at all.** SMART is a useful positive signal ("this drive is dying") and a useless
negative one ("this drive is fine"). Plan for failures you did not predict.

### T.9 — Tuning

```bash
# --- Scheduler ---
cat /sys/block/nvme0n1/queue/scheduler
echo none        > /sys/block/nvme0n1/queue/scheduler   # NVMe: nothing to reorder
echo mq-deadline > /sys/block/sda/queue/scheduler       # SATA SSD/HDD
echo bfq         > /sys/block/sda/queue/scheduler       # desktop interactivity

# --- Queue depth and readahead ---
cat /sys/block/nvme0n1/queue/nr_requests
echo 1024 > /sys/block/nvme0n1/queue/nr_requests
cat /sys/block/sda/queue/read_ahead_kb
echo 128 > /sys/block/sda/queue/read_ahead_kb   # default 128; raise for
                                                #   sequential, lower for random

# --- Rotational hint (get this right on virtual/SSD devices) ---
cat /sys/block/sda/queue/rotational

# --- Writeback ---
sysctl vm.dirty_background_ratio=5     # start background writeback at 5%
sysctl vm.dirty_ratio=10               # BLOCK writers at 10%
sysctl vm.dirty_expire_centisecs=3000  # write back after 30s
sysctl vm.dirty_writeback_centisecs=500
#   Use the *_bytes variants on large-memory machines: 20% of 512 GB is
#   100 GB of dirty data, and flushing it is a multi-minute stall.
sysctl vm.dirty_background_bytes=$((256*1024*1024))
sysctl vm.dirty_bytes=$((1024*1024*1024))

# --- Swappiness and cache pressure ---
sysctl vm.swappiness=10                # server default 60 is often too high
sysctl vm.vfs_cache_pressure=50        # keep dentry/inode cache longer
```

**The `dirty_ratio` percentage-versus-bytes issue is a real production trap.** On a 512 GB
machine, the default `dirty_ratio=20` permits 100 GB of dirty pages. When writeback finally
starts, the system stalls for minutes. Always set the `_bytes` variants on large-memory
systems.

### T.10 — Backup, and what "backup" means

```
 RAID is not backup.        (it protects against DISK failure, not deletion,
                             corruption, ransomware, or your own mistakes)
 Snapshot is not backup.    (same storage; same failure domain)
 Replication is not backup. (it replicates your mistakes, instantly)
```

**The 3-2-1 rule:** 3 copies, 2 different media, 1 offsite. And a fourth condition everyone
omits: **1 tested restore.** An untested backup is a hypothesis.

```bash
# Consistent snapshot -> backup
lvcreate -s -L 20G -n snap /dev/vg0/data
mount -o ro,nouuid /dev/vg0/snap /mnt/snap
restic backup /mnt/snap --repo sftp:backup:/repo
umount /mnt/snap && lvremove -f /dev/vg0/snap

# For a database, the application's own mechanism beats a filesystem snapshot:
pg_basebackup / mysqldump --single-transaction / etcdctl snapshot save

# Verify. Every time.
restic check --read-data-subset=5%
restic restore latest --target /tmp/restore-test --include /etc
```

**The recovery objectives that should drive the design:**

| Term | Question |
|---|---|
| **RPO** (Recovery Point Objective) | How much data may we lose? Determines backup *frequency* |
| **RTO** (Recovery Time Objective) | How long may recovery take? Determines the *method* |

An hourly backup has a 1-hour RPO. If restoring 10 TB takes 12 hours, your RTO is 12 hours no
matter how often you back up. **These two numbers determine the entire design**, and asking
for them is the first move in any storage design discussion.

---

## 1. Internals

### Source map and interfaces

| Path | Contents |
|---|---|
| `/sys/block/<dev>/queue/` | Scheduler, `nr_requests`, `read_ahead_kb`, `rotational`, `discard_*`, `max_sectors_kb` |
| `/sys/block/md*/md/` | RAID state, `sync_action`, `mismatch_cnt`, `stripe_cache_size` |
| `/sys/fs/ext4/<dev>/` | ext4 tunables and error counters |
| `/proc/mdstat` | RAID status |
| `/proc/diskstats` | Raw per-device I/O counters (what `iostat` reads) |
| `/sys/kernel/debug/block/<dev>/` | blk-mq internals: `hctx*/tags`, `hctx*/busy`, `sched/` |
| `/proc/pressure/io` | PSI for I/O |
| `Documentation/block/` | queue sysfs, blk-mq, schedulers |
| `Documentation/admin-guide/device-mapper/` | All the dm targets |
| `Documentation/filesystems/ext4/`, `xfs/` | Per-filesystem admin docs |

### Observability

```bash
# --- The first three commands, always ---
iostat -xz 1                # %util, aqu-sz, r_await, w_await, rareq-sz
                            #   HIGH await + LOW %util = queueing upstream
                            #   HIGH await + HIGH %util = the device is the limit
cat /proc/pressure/io       # stall time -- the metric that actually matters
df -h && df -i              # space AND inodes

# --- Per-process attribution ---
iotop -oPa
pidstat -d 1
sudo bpftrace -e 'tracepoint:block:block_rq_issue {
	@[comm, args->rwbs] = sum(args->bytes / 1024); }'

# --- Latency distribution, not the mean ---
sudo biolatency-bpfcc -D 10 1
sudo biosnoop-bpfcc            # every I/O, with the issuing process
sudo bitesize-bpfcc            # request size distribution

# --- Where is the time in the stack? ---
sudo funclatency-bpfcc -m 'blk_mq_*'
sudo bpftrace -e '
tracepoint:block:block_rq_issue    { @start[args->dev, args->sector] = nsecs; }
tracepoint:block:block_rq_complete /@start[args->dev, args->sector]/ {
	@us = hist((nsecs - @start[args->dev, args->sector]) / 1000);
	delete(@start[args->dev, args->sector]); }'

# --- Filesystem-specific ---
xfs_info /mnt && xfs_db -r -c frag /dev/sda1
dumpe2fs -h /dev/sda1 && e2freefrag /dev/sda1
btrfs filesystem usage /mnt && btrfs scrub status /mnt

# --- In-flight requests (for the hung-task case, debugging-scenarios §2) ---
cat /sys/kernel/debug/block/nvme0n1/hctx*/busy
cat /sys/kernel/debug/block/nvme0n1/hctx*/tags
```

---

## 2. Practice

### Lab 94.1 — Build and benchmark a full stack

```bash
#!/bin/bash
# stack_lab.sh — RAID -> LUKS -> LVM -> XFS, and measure each layer's cost.
# Uses loop devices; safe to run anywhere.
set -e
WORK=/tmp/storage-lab && mkdir -p $WORK && cd $WORK

echo "=== 1. Four 2G backing files -> loop devices ==="
for i in 0 1 2 3; do
	[ -f disk$i.img ] || truncate -s 2G disk$i.img
	LOOP[$i]=$(sudo losetup --find --show disk$i.img)
done
echo "loops: ${LOOP[@]}"

echo "=== 2. RAID 10 ==="
sudo mdadm --create /dev/md99 --level=10 --raid-devices=4 --chunk=512K \
	--run ${LOOP[@]}
cat /proc/mdstat

echo "=== 3. LUKS2 ==="
echo -n "testpassphrase" | sudo cryptsetup luksFormat --type luks2 \
	--cipher aes-xts-plain64 --key-size 512 --sector-size 4096 \
	--batch-mode /dev/md99 -
echo -n "testpassphrase" | sudo cryptsetup open /dev/md99 cryptlab -

echo "=== 4. LVM ==="
sudo pvcreate -f /dev/mapper/cryptlab
sudo vgcreate vglab /dev/mapper/cryptlab
sudo lvcreate -l 80%FREE -n lvdata vglab

echo "=== 5. XFS, with the RAID geometry ==="
# chunk 512K / block 4K = su 512k ; 4 disks RAID10 = 2 data stripes
sudo mkfs.xfs -f -d su=512k,sw=2 -l size=64m /dev/vglab/lvdata
sudo mkdir -p /mnt/lab
sudo mount -o noatime,logbsize=256k /dev/vglab/lvdata /mnt/lab
xfs_info /mnt/lab

echo "=== 6. Benchmark each layer ==="
bench() {
	local dev=$1 name=$2
	printf '%-24s ' "$name"
	sudo fio --name=t --filename=$dev --ioengine=libaio --direct=1 \
		--rw=randwrite --bs=4k --iodepth=32 --numjobs=4 --size=512M \
		--runtime=15 --time_based --group_reporting --output-format=json \
		2>/dev/null | jq -r '"iops=\(.jobs[0].write.iops|floor) lat_us=\(.jobs[0].write.clat_ns.mean/1000|floor)"'
}
bench ${LOOP[0]}            "raw loop"
bench /dev/md99             "md raid10"
bench /dev/mapper/cryptlab  "+ luks"
bench /dev/vglab/lvdata     "+ lvm"
printf '%-24s ' "+ xfs (file)"
sudo fio --name=t --directory=/mnt/lab --ioengine=libaio --direct=1 \
	--rw=randwrite --bs=4k --iodepth=32 --numjobs=4 --size=512M \
	--runtime=15 --time_based --group_reporting --output-format=json \
	2>/dev/null | jq -r '"iops=\(.jobs[0].write.iops|floor) lat_us=\(.jobs[0].write.clat_ns.mean/1000|floor)"'

echo
echo "=== Cleanup ==="
cat <<'EOF'
 sudo umount /mnt/lab
 sudo lvremove -f vglab && sudo vgremove vglab
 sudo pvremove -f /dev/mapper/cryptlab
 sudo cryptsetup close cryptlab
 sudo mdadm --stop /dev/md99
 for l in $(losetup -a | grep storage-lab | cut -d: -f1); do sudo losetup -d $l; done
EOF
```

**Record the per-layer cost.** Typically: LUKS costs 5–20% (much more without AES-NI), LVM
costs ~0, the filesystem costs 5–15% depending on the pattern. Having those numbers turns
"encryption is slow" into an engineering statement.

### Lab 94.2 — Verify durability end to end

The most important lab in the chapter.

```c
/* durability.c — write, fsync, and record what was acknowledged.
 * Build: gcc -O2 -o durability durability.c
 * Run:   ./durability /mnt/lab/testfile
 * Then cut power (or `echo b > /proc/sysrq-trigger`) and check.    */
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>

int main(int argc, char **argv)
{
	int fd, dirfd;
	char buf[4096];
	unsigned long n = 0;
	char *dir, *path = argv[1];

	if (argc < 2) { fprintf(stderr, "usage: %s <file>\n", argv[0]); return 1; }

	fd = open(path, O_WRONLY | O_CREAT | O_TRUNC, 0644);
	if (fd < 0) { perror("open"); return 1; }

	/* fsync the DIRECTORY so the file's existence is durable.
	 * Forgetting this is the classic bug (Ch. 53 / question-bank E2). */
	dir = strdup(path);
	*strrchr(dir, '/') = '\0';
	dirfd = open(dir, O_RDONLY | O_DIRECTORY);
	fsync(dirfd);

	for (;;) {
		memset(buf, 0, sizeof(buf));
		snprintf(buf, sizeof(buf), "record %lu\n", n);
		if (pwrite(fd, buf, sizeof(buf), n * sizeof(buf)) != sizeof(buf)) {
			perror("pwrite"); return 1;
		}
		if (fdatasync(fd)) { perror("fdatasync"); return 1; }

		/* ONLY NOW is record n durable. Announce it. */
		printf("%lu\n", n);
		fflush(stdout);
		n++;
	}
}
```

```bash
# 1. Run it, capturing the acknowledged records.
./durability /mnt/lab/testfile > /tmp/acked.txt &
sleep 10

# 2. Hard reset -- this bypasses all shutdown handling.
echo b | sudo tee /proc/sysrq-trigger
#    (In a VM: `virsh destroy`, or kill -9 the qemu process.)

# 3. After reboot: does the file contain every acknowledged record?
LAST=$(tail -1 /tmp/acked.txt)
EXPECTED=$(( (LAST + 1) * 4096 ))
ACTUAL=$(stat -c%s /mnt/lab/testfile)
echo "last acked: $LAST  expected size: $EXPECTED  actual: $ACTUAL"
[ "$ACTUAL" -ge "$EXPECTED" ] && echo "DURABLE" || echo "DATA LOSS"

# 4. Now break it, one layer at a time, and re-run:
#    a. mount -o barrier=0        (ext4: disables flush)
#    b. hdparm -W1 /dev/sda       (enable the volatile write cache)
#    c. qemu -drive cache=unsafe  (the hypervisor lies)
#    d. an md RAID5 without a write journal
#    Each should produce measurable loss. IDENTIFY WHICH ONE.
```

**This lab is how you answer "is our storage stack actually durable?" with evidence rather
than belief.** Run it on every new platform.

### Lab 94.3 — Break and recover

```bash
# --- Failure 1: a failed RAID member ---
sudo mdadm --fail /dev/md99 /dev/loop0
cat /proc/mdstat                    # degraded
sudo mdadm --remove /dev/md99 /dev/loop0
# "Replace" it:
sudo mdadm --add /dev/md99 /dev/loop0
watch cat /proc/mdstat              # watch the rebuild
#   MEASURE the rebuild time and the performance during it.

# --- Failure 2: filesystem corruption ---
sudo umount /mnt/lab
# Corrupt the superblock area:
sudo dd if=/dev/urandom of=/dev/vglab/lvdata bs=1 seek=1024 count=512
sudo mount /dev/vglab/lvdata /mnt/lab       # fails
sudo xfs_repair -n /dev/vglab/lvdata        # DRY RUN first, always
sudo xfs_repair /dev/vglab/lvdata
#   For ext4:
#     e2fsck -n /dev/...                     # dry run
#     dumpe2fs /dev/... | grep -i superblock # find backups
#     e2fsck -b 32768 /dev/...               # use a backup superblock

# --- Failure 3: a full thin pool ---
sudo lvcreate -L 1G --thinpool tp /dev/vglab
sudo lvcreate -V 5G -T vglab/tp -n thinlv
sudo mkfs.xfs -f /dev/vglab/thinlv
sudo mount /dev/vglab/thinlv /mnt/thin
sudo dd if=/dev/zero of=/mnt/thin/fill bs=1M count=2000
#   Watch: lvs -o+data_percent
#   When the pool fills: I/O errors, filesystem goes read-only.
dmesg | tail -20
#   RECOVERY: extend the pool, then xfs_repair.

# --- Failure 4: inode exhaustion (ext4) ---
sudo mkfs.ext4 -N 1000 -F /dev/vglab/small   # only 1000 inodes
sudo mount /dev/vglab/small /mnt/small
for i in $(seq 1 2000); do touch /mnt/small/f$i 2>/dev/null; done
df -h /mnt/small && df -i /mnt/small
#   Space free, inodes exhausted, ENOSPC. UNFIXABLE without mkfs.
```

**For each failure, write down: the symptom in `dmesg`, the diagnostic command, the recovery
procedure, and how to prevent it.** That document is your storage runbook.

### Lab 94.4 — Find the tuning that matters

```bash
#!/bin/bash
# tune_sweep.sh — measure, do not guess.
DEV=${1:-/dev/nvme0n1}
MNT=${2:-/mnt/lab}

run() {
	fio --name=t --directory=$MNT --ioengine=io_uring --direct=1 \
	    --rw=$1 --bs=$2 --iodepth=$3 --numjobs=$4 --size=1G \
	    --runtime=20 --time_based --group_reporting --output-format=json \
	    2>/dev/null | jq -r --arg l "$5" \
	    '"\($l): iops=\(.jobs[0].read.iops + .jobs[0].write.iops|floor) p99=\(.jobs[0].read.clat_ns.percentile."99.000000"/1000|floor)us"'
}

echo "=== Scheduler ==="
for s in none mq-deadline bfq kyber; do
	echo $s | sudo tee /sys/block/$(basename $DEV)/queue/scheduler >/dev/null 2>&1 || continue
	run randread 4k 32 4 "sched=$s"
done

echo; echo "=== Readahead (sequential read) ==="
for ra in 4 128 512 2048; do
	echo $ra | sudo tee /sys/block/$(basename $DEV)/queue/read_ahead_kb >/dev/null
	run read 128k 8 1 "ra=${ra}k"
done

echo; echo "=== Queue depth ==="
for qd in 1 4 16 64 256; do
	run randread 4k $qd 4 "qd=$qd"
done

echo; echo "=== Dirty ratio (buffered write) ==="
for db in 64 512 4096; do
	sudo sysctl -q vm.dirty_background_bytes=$((db*1024*1024))
	sudo sysctl -q vm.dirty_bytes=$((db*2*1024*1024))
	fio --name=t --directory=$MNT --rw=write --bs=1M --size=4G \
	    --end_fsync=1 --output-format=json 2>/dev/null \
	 | jq -r --arg l "dirty=${db}M" \
	   '"\($l): bw=\(.jobs[0].write.bw_bytes/1048576|floor)MB/s p99=\(.jobs[0].write.clat_ns.percentile."99.000000"/1000000|floor)ms"'
done
```

**The expected findings**, which you should verify rather than assume: `none` wins on NVMe;
readahead matters enormously for sequential and hurts random; queue depth has a knee; and
large dirty ratios improve throughput while destroying p99.

### Lab 94.5 — Design a layout for a stated workload

For each scenario, produce: the device layout, RAID level, LVM structure, filesystem and
`mkfs` options, mount options, tuning, monitoring, and backup design. Justify every choice.

1. **PostgreSQL, 2 TB, 50k TPS, 5-minute RPO, 1-hour RTO.**
2. **Video ingest, 100 TB, 2 GB/s sequential write, files never modified.**
3. **Build server, 50k small files/minute, data is reproducible.**
4. **Embedded gateway, 8 GB eMMC, 10-year life, power loss at any time, OTA updates.**
5. **Kubernetes node, 20 containers, mixed workloads, noisy-neighbour isolation required.**

**Scenario 4 is the hardest and the most instructive.** The answer involves: A/B partitions,
a read-only root with `dm-verity` (Ch. 102), a separate writable data partition with a
journalling or log-structured filesystem, `commit=1` or synchronous writes for critical
state, eMMC write-endurance budgeting (`data_units_written` versus the TBW rating over 10
years), and a power-fail-safe update mechanism. Work it fully.

---

## 3. Mastery drills

1. Build the full stack of Lab 94.1 on real hardware and produce the per-layer cost table.
   Explain every number.

2. Run the durability test on five configurations and identify exactly which layer loses
   data in each. Write the verification procedure for a new platform.

3. Measure eMMC/SD write endurance: write a known volume, read `data_units_written` or the
   eMMC life-time estimate, and project the device's lifetime for a given workload.

4. Recover from each of: a lost LUKS header (use `luksHeaderBackup` first), a corrupted XFS
   log, a doubly-degraded RAID-6, a full thin pool, and an ext4 with a destroyed primary
   superblock. Time each recovery.

5. Compare ext4, XFS, Btrfs, and F2FS on the same hardware with four workloads (sequential,
   random, metadata-heavy, mixed). Produce the matrix and a recommendation per workload.

6. Reproduce the RAID-5 write hole in a VM: partial-stripe write, kill the VM mid-write,
   and detect the inconsistent stripe. Then demonstrate that the write journal prevents it.

7. Build a storage monitoring and alerting system: SMART, filesystem errors, RAID state,
   capacity forecasting, PSI, and latency percentiles. Define the alert thresholds and
   justify each.

8. Tune a real workload end to end using the measure-hypothesize-verify loop. Document every
   change and its effect; discard the ones that did not help.

9. Design and test the OTA update mechanism for scenario 4 above: A/B partitions, atomic
   switch, rollback on failure, and power loss at every point in the sequence.

10. Implement a backup system meeting a stated RPO/RTO, and **prove it by restoring** to a
    clean machine and measuring the actual recovery time.

11. Investigate a mysterious latency spike using only `iostat`, PSI, `biolatency`, and
    `blktrace`. Have a colleague introduce the fault (cgroup throttling, a degraded array, a
    dying disk, dirty-ratio stalls).

12. Write your organization's storage standard: default layouts per workload class,
    mandatory monitoring, the durability verification procedure, and the backup/restore test
    schedule.

---

## 4. Further reading

**Kernel documentation**
- `Documentation/block/` — `queue-sysfs.rst`, `blk-mq.rst`, the scheduler docs
- `Documentation/admin-guide/device-mapper/` — every dm target
- `Documentation/filesystems/ext4/`, `xfs/`, `f2fs.rst`
- `Documentation/admin-guide/md.rst`
- `Documentation/accounting/psi.rst`

**Man pages worth reading fully**
- `mkfs.xfs(8)`, `xfs_admin(8)`, `xfs_repair(8)` — the XFS man pages are exceptionally good
- `mke2fs(8)`, `tune2fs(8)`, `e2fsck(8)`
- `lvm(8)`, `lvmthin(7)`, `lvmcache(7)`
- `cryptsetup(8)`, `mdadm(8)`, `smartctl(8)`, `nvme(1)`

**Data and papers**
- **Backblaze Drive Stats** — quarterly, public, the largest real-world failure dataset in
  existence. Read the SMART-attribute analysis
- Bairavasundaram et al., "An Analysis of Latent Sector Errors in Disk Drives," *SIGMETRICS*,
  2007 — why scrubbing matters
- Schroeder & Gibson, "Disk Failures in the Real World," *FAST*, 2007 — the paper that
  showed MTBF ratings are fiction
- Pillai et al., "All File Systems Are Not Created Equal," *OSDI*, 2014 — **the crash-
  consistency paper.** Essential for anyone writing storage code
- Rebello et al., "Can Applications Recover from fsync Failures?," *ATC*, 2020 — they mostly
  cannot

**Books**
- Brendan Gregg, *Systems Performance*, 2nd ed., ch. 9 (disks) — the best treatment of
  storage performance analysis anywhere
- Brendan Gregg, *BPF Performance Tools*, ch. 9
- Chris Simmonds, *Mastering Embedded Linux Programming*, ch. 9 (flash storage) — for the
  embedded side
- *Linux Storage Stack Diagram* (Werner Fischer) — print it and put it on the wall

**Tools**
- `fio` — learn it properly; it is the only benchmark that matters
- `blktrace`/`blkparse`/`btt` — full block-layer tracing
- `biolatency`, `biosnoop`, `bitesize`, `ext4slower`, `xfsslower` (bcc)
- `smartd`, `nvme-cli`, `sdparm`, `hdparm`
- `restic`, `borgbackup` — modern deduplicating backup

**Cross-references**
- Ch. 51–52 — the storage overview and page cache
- Ch. 57–61 — ext4, XFS, Btrfs, journalling
- Ch. 63–64 — the block layer and schedulers
- Ch. 66–67 — device mapper and MD
- Ch. 76 — `io_uring`, the modern application interface
- `reference/debugging-scenarios.md` §2 — the hung-task-on-I/O walkthrough

→ Next: [95-observability.md](95-observability.md)

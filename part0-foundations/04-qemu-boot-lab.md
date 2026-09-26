# Chapter 04 — QEMU, initramfs, and a Bootable Development Loop

> **Goal:** a single command boots your freshly built kernel to a shell in under 3 seconds,
> with gdb attached, for x86_64 **and** arm64. This loop is the foundation of everything
> that follows — never develop against your real machine.

---

## Theory & First Principles

> **How to read this section.** T.0 is the concrete motivation — why this chapter is arguably
> the highest-return one in Part 0. T.1–T.4 are the theory of virtualization you need to use
> QEMU intelligently rather than by copy-paste. T.5–T.8 are what QEMU buys you and what it
> cannot tell you.

---

### T.0 — Start here: boot a kernel you built, in ten seconds

Before any theory about virtualization, experience the thing this chapter is for. Assuming
you have built a kernel (Ch. 01):

```bash
cd ~/src/linux

# A one-file initramfs containing a statically-linked shell:
mkdir -p /tmp/rfs/{bin,proc,sys,dev}
cp "$(which busybox)" /tmp/rfs/bin/ && (cd /tmp/rfs && ./bin/busybox --install -s bin)
printf '#!/bin/sh\nmount -t proc proc /proc\nmount -t sysfs sysfs /sys\necho "=== hello from PID $$ ==="\nexec /bin/sh\n' > /tmp/rfs/init
chmod +x /tmp/rfs/init
(cd /tmp/rfs && find . | cpio -o -H newc --quiet) | gzip -9 > /tmp/initramfs.gz

# Boot it:
time qemu-system-x86_64 -enable-kvm -m 1G -nographic \
    -kernel arch/x86/boot/bzImage -initrd /tmp/initramfs.gz \
    -append "console=ttyS0 quiet"
# exit the guest with Ctrl-A then X
```

**That is the whole loop.** Roughly 2–5 seconds from `qemu` to a shell, on a kernel you
compiled, with no hardware, no bootloader, no disk image, and no risk of bricking anything.

**Now do the thing that makes it a development environment**, not just a demo:

```bash
# Add -s (gdbstub on :1234) and -S (freeze until gdb attaches):
qemu-system-x86_64 -enable-kvm -m 1G -nographic -s -S \
    -kernel arch/x86/boot/bzImage -initrd /tmp/initramfs.gz -append "console=ttyS0 nokaslr" &

gdb vmlinux -ex 'target remote :1234' -ex 'b start_kernel' -ex 'c'
```

You are now **single-stepping the kernel from its first C function.** On real hardware this
requires a JTAG probe and a day of setup. Here it took one flag.

**The four things to take from this before the theory:**

1. **The inner loop is the whole game.** §T.1 quantifies it: below ~60 seconds you stop
   *reasoning about* what code will do and start *asking* it, and that is a qualitative shift
   in how you work, not a productivity percentage.
2. **You can break anything.** Panic the kernel, corrupt memory, trigger the OOM killer, wedge
   the scheduler — then Ctrl-A X and try again. **Fearlessness is the actual product** of
   this chapter; every lab in the remaining 100 chapters assumes it.
3. **`-s` gives you what hardware cannot**: whole-machine, non-intrusive inspection with the
   guest genuinely stopped. §T.5 explains why that is impossible on real hardware and what
   else follows from it (reverse debugging, deterministic replay, snapshots).
4. **`-enable-kvm` versus not is a 10–100× difference**, and understanding *why* is §T.2–T.3
   — hardware-assisted virtualization versus dynamic binary translation. Drop the flag and
   re-run the `time` above to feel it.

**If you do one thing after this chapter:** script that boot into a single command and bind
it to a key. Ch. 101 §T.10 makes the same argument for real hardware, and it is the single
most reliable predictor of how fast a kernel engineer actually moves.

---

### T.1 The economics of the feedback loop

Before the virtualization theory, the *reason* this chapter exists. Kernel development is
gated by one number: **the time from "I edited a line" to "I observed the result."**

| Loop | Time | Iterations/hour | What you can attempt |
|---|---|---|---|
| Edit → build full kernel → flash board → boot → test | 20–40 min | ~2 | almost nothing; you *plan* instead of *experiment* |
| Edit → build module → `scp` → `insmod` on hardware | 2–5 min | ~15 | careful, hypothesis-driven work |
| **Edit → build module → QEMU boot → test** | **10–40 s** | **~150** | **experimentation; you can afford to be wrong** |
| Edit → KUnit test in userspace | 2–5 s | ~800 | TDD on pure logic |

This is not a minor productivity point. Below roughly 60 seconds, a qualitative change occurs:
you stop *reasoning about* what the code will do and start *asking* it. Debugging shifts from
deduction to measurement. Above ~10 minutes, you batch changes — and batching changes is how
you introduce bugs that interact.

The research framing is **flow** (Csikszentmihalyi) and the interactive-latency literature
(Doherty & Thadhani, IBM 1982, found that sub-second system response produced
*super-linear* gains in user productivity). The engineering framing is simply: *optimize the
inner loop first, always.* Every hour spent making your boot loop faster pays back within a
week.

**Corollary:** always have three loops available and pick the cheapest one that can answer
your question — userspace/KUnit for logic, QEMU for kernel behaviour, real hardware only for
hardware-specific truth.

### T.2 The taxonomy: emulation, virtualization, paravirtualization

These three words are used interchangeably by the careless. They are not the same.

| Technique | How guest instructions execute | Speed | Can run foreign ISA? |
|---|---|---|---|
| **Interpretation** | decoded and simulated one at a time | 100–1000× slower | yes |
| **Dynamic binary translation (DBT)** | blocks of guest code JIT-compiled to host code, cached | 2–10× slower | **yes** (QEMU TCG) |
| **Hardware-assisted virtualization** | guest runs *natively* on the CPU in a less-privileged mode; only sensitive operations trap | ~1.05–1.2× | no (same ISA only) |
| **Paravirtualization** | guest is *modified* to call the hypervisor explicitly instead of trapping | near-native for I/O | n/a |
| **Containers** | no virtualization at all; namespace + cgroup isolation on one kernel | 1× | no (same kernel) |

QEMU is really two products sharing a codebase: **TCG mode** (a DBT emulator — this is how
you run an arm64 kernel on your x86 laptop) and **KVM mode** (QEMU provides device emulation;
the Linux `kvm` module provides CPU/memory virtualization via hardware). You will use both:
TCG for cross-architecture work, KVM for speed.

### T.3 Popek & Goldberg: when is an ISA virtualizable?

Popek & Goldberg, *"Formal Requirements for Virtualizable Third Generation Architectures"*
(CACM 1974), gave the field its foundational theorem. Define:

- **Privileged instructions** — trap when executed in user mode.
- **Sensitive instructions** — those that either *change* the machine's resource
  configuration (control-sensitive) or whose *behaviour depends* on it (behaviour-sensitive).

> **Theorem:** an architecture is (classically) virtualizable iff **the set of sensitive
> instructions is a subset of the set of privileged instructions.**

The intuition is **trap-and-emulate**: run the guest OS at user privilege; every operation
that could observe or alter the real machine state traps to the hypervisor, which emulates it
against the *virtual* machine state. If some sensitive instruction does *not* trap, the guest
silently observes the host's real state and the illusion breaks.

**x86 famously failed this test.** Robin & Irvine (USENIX Security 2000) identified 17
sensitive-but-unprivileged instructions. The canonical example is `POPF`: in user mode it
*silently ignores* writes to the interrupt-enable flag instead of trapping. So a guest kernel
executing `cli`/`popf` to disable interrupts would appear to succeed while nothing happened —
undetectably.

The three historical responses, all of which you should be able to name:

1. **Binary translation** (VMware, 1999) — scan guest kernel code and rewrite the offending
   instructions before execution. Brilliant, complex, and patented into a business.
2. **Paravirtualization** (Xen, SOSP 2003) — *modify the guest kernel* to replace sensitive
   sequences with explicit hypercalls. Fast, but requires guest cooperation.
3. **Hardware extensions** (Intel VT-x 2005, AMD-V) — add a new, *more* privileged mode
   (VMX root) so the guest can run at ring 0 of a virtual machine, with a
   hardware-defined trap list (`VMCS`). This retroactively makes x86 satisfy Popek &
   Goldberg. ARM did it properly from the start with **EL2**; RISC-V with **HS-mode**.

The second big cost after CPU is **memory**: the guest maintains its own page tables
(GVA→GPA), and the host must map GPA→HPA. Software "shadow page tables" were expensive;
hardware **two-dimensional paging** (Intel EPT, AMD NPT, ARM stage-2) walks both in hardware.
The cost moved from traps to TLB-miss latency (a 2-D walk is up to 24 memory accesses on
x86-64 instead of 4) — which is exactly why huge pages matter so much in VMs.

### T.4 Device virtualization, and why virtio exists

Emulating a real device (an Intel e1000 NIC, an IDE controller) is *correct* but slow: every
MMIO register access is a trap to the hypervisor (~1–2 µs), and a real NIC driver touches
registers many times per packet. A 10 Gb/s workload would spend all its time trapping.

**virtio** (Rusty Russell, "virtio: towards a de-facto standard for virtual I/O devices",
OSR 2008) is paravirtualization applied to devices. Its design ideas are worth knowing
because they recur in real hardware (NVMe, modern NICs) and in Ch. 26/63:

- **A shared-memory ring buffer (`virtqueue`)** instead of registers. The guest writes
  descriptors into memory the host can read; no trap per operation.
- **Batching + explicit notification.** The guest fills many descriptors, then performs *one*
  MMIO write to "kick" the host. Notification suppression flags let either side say "don't
  interrupt me, I'm polling" — this is Ch. 17's interrupt/polling hybrid, again.
- **A transport-independent device model**, so the same `virtio-net` driver works over PCI,
  MMIO (for embedded/arm `virt`), or channel I/O (s390).
- **Feature negotiation** — a bitmap exchanged at init, allowing forward/backward
  compatibility. A good UAPI design pattern to study (Ch. 24).

`vhost` moves the datapath into the *host kernel* to eliminate the QEMU userspace hop, and
`vDPA` lets real hardware speak virtio directly. The trajectory — software interface →
hardware implements it — is a recurring pattern in systems.

**Practical rule for this curriculum:** always prefer `virtio-*` devices in QEMU (`virtio-blk`,
`virtio-net`, `virtio-9p`, `virtio-rng`, `virtio-serial`) unless you are specifically testing
a legacy device driver.

### T.5 What QEMU gives you that hardware cannot

Beyond speed, emulation provides capabilities that are *impossible* on real hardware, and
using them is what separates efficient kernel developers from slow ones:

| Capability | Why it's impossible on hardware | QEMU flag |
|---|---|---|
| **Stop the whole machine and inspect it** | JTAG only, if at all | `-s -S` (gdbstub) |
| **Break on a physical address** | needs hardware watchpoints, limited to 4 | gdb `watch` via gdbstub |
| **Deterministic replay** | real interrupts/timing are nondeterministic | `-icount shift=N,rr=record` |
| **Inject arbitrary hardware faults** | requires a fault-injection rig | device models, `qom-set` |
| **Snapshot/restore machine state** | impossible | `savevm`/`loadvm`, `-snapshot` |
| **Run an ISA you don't own** | needs the silicon | `qemu-system-aarch64` on x86 |
| **Inspect guest physical memory live** | needs a probe | QMP `pmemsave`, `-monitor` |
| **Unlimited "hardware" instances** | budget | `-smp 64 -m 64G` on a laptop |

**Record/replay deserves emphasis.** With `-icount shift=auto,rr=record,rrfile=replay.bin`,
QEMU records every nondeterministic input (interrupt timing, device data, RNG). Replaying
gives a bit-exact re-execution — so a race that reproduces once can be replayed *forever*,
including under gdb, including backwards. For a heisenbug this is the difference between
"unfixable" and "fixed this afternoon". Very few kernel developers know this exists.

### T.6 The boot contract: what must exist for a kernel to reach userspace

To boot *anything*, four things must be agreed between bootloader and kernel:

1. **An entry point and image format.** x86: the `bzImage` real-mode header + decompressor
   (`Documentation/arch/x86/boot.rst`). arm64: the `Image` header with magic `ARM\x64` and a
   text offset (`Documentation/arch/arm64/booting.rst`). RISC-V similar.
2. **A hardware description.** x86: firmware tables (E820 memory map, ACPI, MP tables),
   discovered by the kernel. ARM/RISC-V: a **device tree blob** passed in a register
   (Ch. 32), because there is no firmware enumeration standard.
3. **A command line.** Passed via the boot protocol; parsed by `__setup`/`early_param`/
   `module_param` handlers.
4. **A root filesystem.** Either an **initramfs** (a cpio archive unpacked into the
   `rootfs` tmpfs the kernel already has) or a real block device named by `root=`.

The initramfs deserves a theoretical note because people find it mysterious: the kernel
*always* has a rootfs (a tmpfs mounted at `/` during `vfs_caches_init()`). An initramfs is
simply a cpio archive that `populate_rootfs()` unpacks into it. There is no mount, no
filesystem driver, no block device involved — which is precisely why it works before any
storage driver has loaded. That solves a genuine bootstrapping problem: *the driver needed to
mount the root filesystem may itself live on the root filesystem.*

The older **initrd** was different — a *block device* image (a ramdisk) that had to be
mounted, requiring a filesystem driver. initramfs replaced it because it is simpler, needs no
fixed size, and shares the page cache. You will still see `initrd=` used as the parameter
name for historical reasons.

---

### T.7 — What is still argued about

**1. Does QEMU testing prove anything about real hardware?**
It proves logic, not timing. QEMU's virtio devices complete instantly and deterministically;
real devices have latency, reordering, errata, and cache-coherency behaviour QEMU does not
model. A driver that works in QEMU and fails on hardware is the *normal* case for
DMA-ordering and cache-maintenance bugs (Ch. 35, Ch. 83 §T.7). The correct stance: QEMU for
logic and iteration, hardware for truth, and know which questions belong to which.

**2. Is `-enable-kvm` ever the wrong choice?**
For speed, never. But TCG (software emulation) is *deterministic*, which KVM is not — so a
race that reproduces under TCG reproduces reliably, and record/replay only works there.
Deliberately running slowly to gain determinism is a real technique and an under-used one.

**3. Is a virtual board a substitute for a real one in bring-up?**
Partly. You can validate the kernel, the DTS grammar, and the driver logic. You cannot
validate DRAM timing, power sequencing, pinmux, or signal integrity — which is where
bring-up actually fails (Ch. 101 §T.4). QEMU removes the *software* unknowns so that what
remains is genuinely hardware, and that is a large win, but it is not the same claim.

**4. Should the whole kernel test suite run in a VM?**
It largely does — `kunit`, `kselftest`, and syzkaller all run virtualized, and the 0-day bot
boots QEMU. The residue that cannot be virtualized (real NICs, real storage, real SoCs) is
exactly the residue that has the worst test coverage in the kernel, and that correlation is
not a coincidence.

### T.8 — The compressed model

```
 THE INNER LOOP IS THE WHOLE GAME. Below ~60 s you stop reasoning and start
 asking. Optimize the loop before optimizing anything else.

 EMULATION != VIRTUALIZATION. Interpretation and DBT (TCG) can run a foreign
 ISA and are 2-1000x slower. Hardware-assisted virtualization (KVM) runs the
 guest NATIVELY in a less-privileged mode and traps only sensitive ops.

 POPEK & GOLDBERG: an ISA is efficiently virtualizable iff SENSITIVE
 instructions are a subset of PRIVILEGED ones. x86 failed this with 17
 instructions, which is why VT-x exists.

 VIRTIO EXISTS BECAUSE EMULATING REAL HARDWARE IS ABSURD: a shared-memory
 ring gives one exit per BATCH instead of one per register access.

 QEMU GIVES YOU WHAT HARDWARE CANNOT: whole-machine stop, non-intrusive
 inspection, snapshots, deterministic replay, and zero risk.
```

Five questions before reaching for hardware:

1. **Is this question about logic or about timing?** Logic → QEMU.
2. **Do I need determinism more than speed?** Yes → drop `-enable-kvm`.
3. **Can I reproduce it with a virtual device** (`edu`, `pci-testdev`, `virtio`)?
4. **Am I testing the kernel, or the hardware's behaviour?**
5. **Is my loop scripted to one command?** If not, fix that first.

---

## 1. Concept: why QEMU is non-negotiable

| Problem | QEMU answer |
|---|---|
| Kernel panic kills your workstation | VM reboots in 2 s |
| Need a fresh kernel per test | `-kernel vmlinux` — no install, no bootloader |
| Need to single-step the kernel | `-s -S` gdb stub, full register access |
| Need arm64/riscv hardware | `-M virt -cpu cortex-a76` |
| Need an NVMe / SCSI / virtio device | `-device nvme,...` — synthetic hardware on demand |
| Need to test a driver without the chip | write a QEMU device model, or use `qtest` |
| Need deterministic replay | `-icount shift=auto,rr=record` |

The entire kernel community develops this way. Real hardware is for the *last* 10%.

---

## 2. Building a minimal root filesystem

Three options, from fastest to most capable.

### 2.1 Option A — statically linked busybox initramfs (fastest, ~3 s boot)

```bash
# Build a static busybox once
mkdir -p ~/kdev && cd ~/kdev
wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2
tar xf busybox-1.36.1.tar.bz2 && cd busybox-1.36.1
make defconfig
sed -i 's/# CONFIG_STATIC is not set/CONFIG_STATIC=y/' .config
sed -i 's/CONFIG_TC=y/# CONFIG_TC is not set/' .config   # avoids a known build break
make -j$(nproc)
make CONFIG_PREFIX=../initramfs install
cd ..
```

Now build the initramfs tree:

```bash
cd ~/kdev/initramfs
mkdir -p proc sys dev tmp run etc mnt lib/modules

cat > init <<'EOF'
#!/bin/sh
mount -t proc     none /proc
mount -t sysfs    none /sys
mount -t devtmpfs none /dev
mount -t tmpfs    none /tmp
mount -t debugfs  none /sys/kernel/debug 2>/dev/null
mount -t tracefs  none /sys/kernel/tracing 2>/dev/null

echo 0 > /proc/sys/kernel/printk_ratelimit 2>/dev/null
echo "=== kernel $(uname -r) up ==="

# Auto-load any modules shipped in the image
for m in /lib/modules/*.ko; do [ -e "$m" ] && insmod "$m"; done

# Run a command passed via kernel cmdline: run=/path/to/script
for x in $(cat /proc/cmdline); do
    case "$x" in run=*) sh "${x#run=}";; esac
done

exec setsid cttyhack /bin/sh
EOF
chmod +x init

# Pack it
find . -print0 | cpio --null -ov --format=newc 2>/dev/null | gzip -9 > ../initramfs.cpio.gz
```

One-liner repack script `~/kdev/mkinitramfs.sh`:
```bash
#!/bin/bash
set -e
cd ~/kdev/initramfs
find . -print0 | cpio --null -o --format=newc 2>/dev/null | gzip -9 > ../initramfs.cpio.gz
echo "initramfs: $(du -h ~/kdev/initramfs.cpio.gz | cut -f1)"
```

### 2.2 Option B — Debian/Alpine rootfs image (realistic userspace)

```bash
# Alpine mini root — small and fast
mkdir -p ~/kdev/alpine && cd ~/kdev/alpine
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
qemu-img create -f raw rootfs.img 2G
mkfs.ext4 rootfs.img
mkdir -p mnt && sudo mount -o loop rootfs.img mnt
sudo tar xzf alpine-minirootfs-*.tar.gz -C mnt
sudo cp /etc/resolv.conf mnt/etc/
echo 'ttyS0::respawn:/sbin/getty -L ttyS0 115200 vt100' | sudo tee -a mnt/etc/inittab
sudo sed -i 's/^root:.*/root::0:0:root:\/root:\/bin\/sh/' mnt/etc/passwd  # no password
sudo umount mnt
```

Or Debian:
```bash
sudo debootstrap --arch=amd64 --include=openssh-server,build-essential,gdb,strace \
     bookworm ./debian-rootfs http://deb.debian.org/debian
```

### 2.3 Option C — `virtme-ng` (the pro move)

```bash
pipx install virtme-ng     # or: pip install --user virtme-ng
cd linux
vng --build --config ../dev.config     # builds a kernel
vng                                    # boots it, YOUR HOST FS as rootfs, read-only
vng -- uname -a                        # run one command and exit
vng --user root -- ./my-test.sh
```

`virtme-ng` boots your built kernel with your **real host filesystem** mounted via
virtio-9p/virtiofs. No initramfs, no image management. This is how many maintainers work.
Use it for quick tests; use handcrafted images when you need exact control.

---

## 3. Booting

### 3.1 x86_64

```bash
qemu-system-x86_64 \
  -kernel ~/build-x86/arch/x86/boot/bzImage \
  -initrd ~/kdev/initramfs.cpio.gz \
  -append "console=ttyS0 earlyprintk=serial,ttyS0,115200 nokaslr panic=-1 oops=panic loglevel=8" \
  -m 2G -smp 4 \
  -nographic \
  -enable-kvm -cpu host \
  -no-reboot
```

Key `-append` options for development:

| Option | Why |
|---|---|
| `console=ttyS0` | serial console → your terminal |
| `earlyprintk=serial,ttyS0,115200` | output *before* console init (x86) |
| `earlycon` | arch-generic early console (arm64/riscv) |
| `nokaslr` | **essential for gdb** — fixed symbol addresses |
| `panic=-1` + `-no-reboot` | freeze on panic so you can read it |
| `oops=panic` | turn any Oops into a panic (fail fast) |
| `loglevel=8` | print everything |
| `ftrace=function` / `trace_event=...` | boot-time tracing |
| `slub_debug=FZPU` | full slab debugging |
| `kasan.fault=panic` | stop on first KASAN hit |
| `nosmp` / `maxcpus=1` | simplify race debugging |
| `init=/bin/sh` | skip init |
| `root=/dev/vda rw` | for disk-image boots |
| `systemd.unit=rescue.target` | systemd rootfs |

### 3.2 arm64

```bash
qemu-system-aarch64 \
  -M virt -cpu cortex-a76 -m 2G -smp 4 \
  -kernel ~/build-arm64/arch/arm64/boot/Image \
  -initrd ~/kdev/initramfs-arm64.cpio.gz \
  -append "console=ttyAMA0 earlycon nokaslr panic=-1" \
  -nographic -no-reboot
```

Dump the generated device tree (QEMU synthesizes one):
```bash
qemu-system-aarch64 -M virt,dumpdtb=virt.dtb -cpu cortex-a76 -m 2G
dtc -I dtb -O dts virt.dtb -o virt.dts && less virt.dts
```
This is how you learn DT without owning hardware (Chapter 32).

### 3.3 riscv64

```bash
qemu-system-riscv64 -M virt -m 2G -smp 4 \
  -kernel ~/build-riscv/arch/riscv/boot/Image \
  -initrd ~/kdev/initramfs-riscv.cpio.gz \
  -append "console=ttyS0 earlycon" -nographic
```

### 3.4 Attaching devices you want to drive

```bash
# virtio block
-drive file=disk.img,if=none,id=d0,format=raw -device virtio-blk-pci,drive=d0

# NVMe (great for Chapter 69)
-drive file=nvme.img,if=none,id=n0,format=raw \
-device nvme,serial=deadbeef,drive=n0,num_queues=8

# NVMe with Zoned Namespace
-device nvme,serial=zns1,id=nvme0 \
-device nvme-ns,drive=z0,bus=nvme0,zoned=true,zoned.zone_size=64M

# SCSI
-device virtio-scsi-pci,id=scsi0 \
-drive file=scsi.img,if=none,id=s0,format=raw -device scsi-hd,drive=s0

# SATA / AHCI (libata path)
-device ich9-ahci,id=ahci -drive file=sata.img,if=none,id=a0 -device ide-hd,drive=a0,bus=ahci.0

# USB (xHCI + mass storage)
-device qemu-xhci,id=xhci -drive file=usb.img,if=none,id=u0 \
-device usb-storage,bus=xhci.0,drive=u0

# Networking with host access + port forward
-netdev user,id=n0,hostfwd=tcp::2222-:22 -device virtio-net-pci,netdev=n0

# Share a host directory (virtiofs / 9p)
-virtfs local,path=/home/you/shared,mount_tag=host0,security_model=none,id=h0
#  guest:  mount -t 9p -o trans=virtio,version=9p2000.L host0 /mnt

# PCI device for driver experiments
-device edu           # QEMU's educational PCI device — PERFECT for Chapter 37
-device pci-testdev
-device ivshmem-plain,memdev=hostmem

# Emulate an i2c/SPI-attached sensor on arm64 virt
-device ...           # or use QEMU's 'aspeed' / 'raspi' machines which model real buses
```

> **`-device edu` is a gift.** It is a tiny PCI device with BARs, MMIO registers, an IRQ,
> and a DMA engine, documented in QEMU's `docs/specs/edu.rst`. Writing a driver for it
> exercises Chapters 34, 35, 37 with zero hardware.

### 3.5 A driver script you'll use every day

`~/kdev/run.sh`:
```bash
#!/bin/bash
set -euo pipefail
ARCH=${ARCH:-x86_64}
BUILD=${BUILD:-$HOME/build-x86}
EXTRA_APPEND=${APPEND:-}
GDB=${GDB:-0}

COMMON="-m 2G -smp 4 -nographic -no-reboot"
APPEND="console=ttyS0 earlycon nokaslr panic=-1 oops=panic loglevel=8 $EXTRA_APPEND"
[[ $GDB == 1 ]] && COMMON="$COMMON -s -S"

case $ARCH in
  x86_64)
    exec qemu-system-x86_64 $COMMON -enable-kvm -cpu host \
      -kernel "$BUILD/arch/x86/boot/bzImage" \
      -initrd "$HOME/kdev/initramfs.cpio.gz" \
      -append "$APPEND" \
      -device edu \
      -netdev user,id=n0,hostfwd=tcp::2222-:22 -device virtio-net-pci,netdev=n0 \
      "$@" ;;
  arm64)
    exec qemu-system-aarch64 $COMMON -M virt -cpu cortex-a76 \
      -kernel "$BUILD/arch/arm64/boot/Image" \
      -initrd "$HOME/kdev/initramfs-arm64.cpio.gz" \
      -append "${APPEND/ttyS0/ttyAMA0}" "$@" ;;
esac
```

```bash
chmod +x ~/kdev/run.sh
~/kdev/run.sh                                  # boot
APPEND="run=/tests/t1.sh" ~/kdev/run.sh        # boot and run a script
GDB=1 ~/kdev/run.sh                            # boot frozen, waiting for gdb
```

### 3.6 Escaping QEMU `-nographic`

`Ctrl-a x` quits. `Ctrl-a c` toggles the QEMU monitor. `Ctrl-a h` help.
In the monitor: `info qtree`, `info pci`, `info mtree`, `info registers`, `system_reset`.

---

## 4. GDB on the running kernel

```bash
# Terminal 1
GDB=1 ~/kdev/run.sh

# Terminal 2
cd ~/build-x86
gdb vmlinux
(gdb) target remote :1234
(gdb) hbreak start_kernel
(gdb) continue
```

Enable the kernel's GDB helpers (requires `CONFIG_GDB_SCRIPTS=y`):
```bash
echo "add-auto-load-safe-path $HOME/build-x86" >> ~/.gdbinit
```
Then:
```
(gdb) lx-symbols              # load module symbols automatically
(gdb) lx-dmesg                # print the kernel log ring buffer
(gdb) lx-ps                   # list processes
(gdb) lx-lsmod
(gdb) lx-device-list-bus pci
(gdb) lx-mounts
(gdb) lx-clk-summary
(gdb) p $lx_current().comm
(gdb) p $lx_per_cpu(runqueues, 0)
(gdb) p *(struct task_struct *)0xffff...
```

For arm64: `gdb-multiarch vmlinux`, `set architecture aarch64`.

**Breakpoint tips:**
- Use `hbreak` (hardware breakpoint) early in boot before the kernel's page tables settle.
- `nokaslr` is mandatory or symbols won't match.
- To break in a module, `insmod` first, then `lx-symbols`, then `break my_func`.

---

## 5. Faster iteration: sharing modules into the VM

```bash
# Build modules into a staging dir
make O=~/build-x86 modules -j$(nproc)
make O=~/build-x86 modules_install INSTALL_MOD_PATH=~/kdev/initramfs
~/kdev/mkinitramfs.sh
```

Or skip the repack entirely with 9p:
```bash
# add to run.sh:
-virtfs local,path=$HOME/kdev/share,mount_tag=host0,security_model=none,id=h0
# in init:
mkdir -p /mnt/host && mount -t 9p -o trans=virtio,version=9p2000.L host0 /mnt/host
```
Now `cp foo.ko ~/kdev/share/` on the host and `insmod /mnt/host/foo.ko` in the guest.
**Zero rebuild of the image.** This is the loop you want.

---

## 6. Practice

### Lab 4.1 — Sub-3-second boot
Build busybox initramfs + your dev kernel. Measure:
```bash
time (echo | ~/kdev/run.sh -append "console=ttyS0 panic=-1 init=/bin/true")
dmesg | tail -1          # "Freeing unused kernel image memory"
```
Then in the guest: `systemd-analyze` isn't available, so use:
```bash
dmesg | grep -i 'boot\|initcall' | tail
```
Boot with `initcall_debug` and find your slowest initcall.

### Lab 4.2 — Cause and read a panic
```bash
# in guest
echo c > /proc/sysrq-trigger
```
Read the full panic. Then symbolize offline:
```bash
./scripts/decode_stacktrace.sh ~/build-x86/vmlinux < panic.txt
```

### Lab 4.3 — Use the `edu` PCI device
```bash
# in guest
lspci -vnn | grep -A5 1234:11e8
ls /sys/bus/pci/devices/*/resource*
```
You'll write a driver for this in Chapter 37.

### Lab 4.4 — Boot arm64 and dump its device tree
Do §3.2, then in the guest:
```bash
ls /proc/device-tree/
find /proc/device-tree -name compatible -exec sh -c 'echo "{}: $(tr -d "\0" < {})"' \;
```

### Lab 4.5 — Snapshot workflow
```bash
qemu-img create -f qcow2 -b base.img -F raw work.qcow2
# boot against work.qcow2; throw it away when you corrupt the FS
```

### Lab 4.6 — Measure your feedback loop (T.1) and then halve it

```bash
# Baseline: time a full cold loop
time ( make -j$(nproc) O=b && \
       make -j$(nproc) O=b M=$PWD/mymod && \
       ./run-qemu.sh --test )

# Now apply each optimization and re-measure:
# 1. ccache
export PATH=/usr/lib/ccache:$PATH; ccache -M 50G
# 2. build only the module, not the kernel
make -j$(nproc) O=b M=$PWD/mymod
# 3. skip the rootfs rebuild: 9p-share the module dir (see §5)
# 4. -kernel direct boot (no bootloader), -nographic, virtio only
# 5. KVM instead of TCG for same-arch
qemu-system-x86_64 -enable-kvm -cpu host ...
# 6. Kill the boot delay:
#    append: quiet loglevel=3 tsc=reliable no_timer_check noreplace-smp
```
Record your before/after in seconds. **Target: under 20 seconds, edit to test output.**
Anything above 60 s should be treated as a bug in your environment, not a fact of life.

### Lab 4.7 — Deterministic record/replay: catch a race and replay it (T.5)

```bash
# Record a boot + workload. icount makes execution deterministic.
qemu-system-x86_64 -M q35 -m 1G -smp 2 \
  -icount shift=7,rr=record,rrfile=/tmp/rr.bin \
  -kernel b/arch/x86/boot/bzImage -initrd initramfs.cpio.gz \
  -append "console=ttyS0 root=/dev/ram0 rdinit=/init" \
  -netdev user,id=n0 -device virtio-net-pci,netdev=n0 \
  -nographic

# Replay it — bit-identical, every time, including the race
qemu-system-x86_64 -M q35 -m 1G -smp 2 \
  -icount shift=7,rr=replay,rrfile=/tmp/rr.bin \
  -kernel b/arch/x86/boot/bzImage -initrd initramfs.cpio.gz \
  -append "console=ttyS0 root=/dev/ram0 rdinit=/init" \
  -netdev user,id=n0 -device virtio-net-pci,netdev=n0 \
  -nographic -s -S
# In another terminal:
gdb b/vmlinux -ex 'target remote :1234'
# and now REVERSE debugging works:
(gdb) reverse-continue
(gdb) reverse-step
```
Reverse execution over a recorded race is the single most powerful kernel debugging technique
that almost nobody uses. Do this once and it will be in your toolkit forever.

### Lab 4.8 — Compare TCG vs KVM, and measure the virtualization tax (T.3)

```bash
# Same kernel, same workload, two engines:
time qemu-system-x86_64 -M q35 -m 2G -smp 4 -kernel ... -append "... rdinit=/bench"  # TCG
time qemu-system-x86_64 -M q35 -m 2G -smp 4 -enable-kvm -cpu host -kernel ... same   # KVM

# Inside the guest, measure the trap cost directly:
#   count VM exits from the host side:
sudo perf kvm stat live          # or:
sudo perf stat -e 'kvm:kvm_exit' -a sleep 5
sudo cat /sys/kernel/debug/kvm/*/vcpu0/* 2>/dev/null | head

# Which exit reasons dominate? (EPT_VIOLATION, IO_INSTRUCTION, MSR_WRITE, HLT...)
sudo perf record -e kvm:kvm_exit -a -- sleep 5 && sudo perf script | \
  awk '{print $NF}' | sort | uniq -c | sort -rn | head
```
Expect TCG to be 5–20× slower. Then explain, using T.3, *why* an `EPT_VIOLATION` exit is
cheaper than an `IO_INSTRUCTION` exit, and why virtio eliminates most of the latter.

### Lab 4.9 — Emulated device vs virtio: measure the difference (T.4)

```bash
# Same guest, two NICs. Run iperf3 or a simple throughput test through each.
#  (a) emulated Intel e1000:
-netdev user,id=n0,hostfwd=tcp::5201-:5201 -device e1000,netdev=n0
#  (b) virtio-net:
-netdev user,id=n0,hostfwd=tcp::5201-:5201 -device virtio-net-pci,netdev=n0

# Same for block:
-drive file=disk.img,if=none,id=d0,format=raw -device ide-hd,drive=d0        # emulated
-drive file=disk.img,if=none,id=d0,format=raw -device virtio-blk-pci,drive=d0 # virtio
```
Then, inside the guest, observe the driver difference:
```bash
lspci -vnn ; ls /sys/bus/virtio/devices/
cat /sys/bus/virtio/devices/virtio0/features      # feature negotiation bitmap (T.4)
sudo cat /proc/interrupts | grep -i virtio
```
Read `drivers/virtio/virtio_ring.c` afterwards — it is the cleanest example of a
descriptor-ring device model in the tree, and it is the mental model you will need for NVMe
(Ch. 69) and modern NICs (Ch. 46).

### Lab 4.10 — Build a *useful* rootfs, three ways

```bash
# (a) BusyBox static initramfs — smallest, fastest (§2)
# (b) Debian via mmdebstrap — full userspace, apt, gdb, perf, trace-cmd
sudo apt install -y mmdebstrap
sudo mmdebstrap --variant=apt --include=systemd-sysv,udev,kmod,gdb,linux-perf,trace-cmd,strace,iproute2,openssh-server \
     stable /tmp/rootfs http://deb.debian.org/debian
sudo tar -C /tmp/rootfs -cf rootfs.tar .
virt-make-fs --format=qcow2 --size=4G rootfs.tar rootfs.qcow2   # or mkfs+cp manually

# (c) Buildroot — reproducible, tiny, cross-arch (Part 7 preview)
git clone --depth=1 https://git.buildroot.net/buildroot && cd buildroot
make qemu_aarch64_virt_defconfig && make -j$(nproc)
# → output/images/{Image,rootfs.ext4,start-qemu.sh}
```
Keep (a) for fast iteration and (b) for anything needing `perf`/`gdb`/package installs.
Being able to produce (c) is a Part 7 prerequisite.

### Lab 4.11 — Boot the same kernel source on four architectures

```bash
# x86_64
qemu-system-x86_64 -M q35 -m 1G -kernel b-x86/arch/x86/boot/bzImage \
  -initrd initramfs.cpio.gz -append "console=ttyS0 rdinit=/init" -nographic
# arm64
qemu-system-aarch64 -M virt -cpu cortex-a72 -m 1G -kernel b-arm64/arch/arm64/boot/Image \
  -initrd initramfs-arm64.cpio.gz -append "console=ttyAMA0 rdinit=/init" -nographic
# riscv64
qemu-system-riscv64 -M virt -m 1G -kernel b-riscv/arch/riscv/boot/Image \
  -initrd initramfs-riscv.cpio.gz -append "console=ttyS0 rdinit=/init" -nographic
# arm32
qemu-system-arm -M virt -m 512M -kernel b-arm/arch/arm/boot/zImage \
  -initrd initramfs-arm.cpio.gz -append "console=ttyAMA0 rdinit=/init" -nographic
```
Note what changed: the console device name, the image name, and the machine model. Nothing
else. That is T.3 of Ch. 02 (portability) made concrete.

### Lab 4.12 — Inspect the boot contract (T.6)

```bash
# x86: read the bzImage header
hexdump -C b/arch/x86/boot/bzImage | head -5
grep -n 'HdrS\|boot_flag\|setup_header' arch/x86/boot/header.S | head
$EDITOR Documentation/arch/x86/boot.rst

# arm64: read the Image header
hexdump -C b-arm64/arch/arm64/boot/Image | head -3      # magic "ARM\x64" at offset 0x38
$EDITOR Documentation/arch/arm64/booting.rst

# See the DTB QEMU passes to an arm64 guest:
qemu-system-aarch64 -M virt,dumpdtb=/tmp/virt.dtb -cpu cortex-a72 -nographic
dtc -I dtb -O dts /tmp/virt.dtb | head -60

# Inside any guest: what did the kernel actually receive?
cat /proc/cmdline
cat /proc/iomem | head -20            # the memory map it was given
dmesg | grep -iE 'e820|memory map|Machine model|earlycon'
```

---

## 7. Mastery drills

1. Boot with `initcall_debug` and `printk.time=1`. Produce a sorted list of the 10 slowest
   initcalls. Explain each.
2. Set a gdb breakpoint on `do_sys_openat2` and inspect the `struct filename`. Walk up
   the stack to the syscall entry.
3. Build a kernel with `CONFIG_SMP=n` and boot it. What breaks? What gets faster?
4. Use `-icount shift=auto,rr=record,rrfile=rec.bin` to record a boot, then replay it.
   Explain why deterministic replay matters for race debugging.
5. Write a QEMU launch profile for each of: NVMe with 4 queues, a zoned namespace,
   an AHCI SATA disk, and an xHCI USB stick. Verify each enumerates in the guest.
6. **Popek & Goldberg.** Read the 1974 paper (9 pages). Then read Robin & Irvine's list of
   x86's 17 problematic instructions. Pick `POPF` and explain precisely why it defeats
   trap-and-emulate, and how VT-x fixes it.
7. **Count the exits.** Using `perf kvm stat`, find the top three VM-exit reasons for
   (a) a `kernel build` workload and (b) an `iperf3` workload inside a guest. Explain the
   difference in terms of what each workload does.
8. **Write a virtio driver trace.** With `trace-cmd` in the guest, record
   `virtio*` and `kvm*` events during a `dd` to a `virtio-blk` disk. Draw the sequence:
   guest driver → virtqueue → kick (MMIO exit) → host → completion interrupt.
9. **Nested virtualization.** Enable it (`kvm_intel.nested=1` / `kvm_amd.nested=1`), boot a
   VM inside your VM, and measure the slowdown. Explain where the cost comes from.
10. **Build the fastest possible loop.** Produce a single `make && boot && run test && exit`
    script that completes in under 15 seconds on your machine and prints PASS/FAIL. Commit
    it. This script is the most valuable artifact in this chapter.
11. **Break the boot contract deliberately.** Corrupt the arm64 `Image` magic, or pass a
    truncated DTB, and observe the failure mode. Then use `earlycon` to get output before
    the console driver initializes. Knowing how to debug a kernel that produces *no* output
    is a core BSP skill (Ch. 101).

---

## 8. Further reading

**Kernel documentation:**
- `Documentation/dev-tools/gdb-kernel-debugging.rst` ★
- `Documentation/admin-guide/kernel-parameters.txt` ★★ (skim all of it once)
- `Documentation/arch/x86/boot.rst`, `Documentation/arch/arm64/booting.rst` — the boot
  contracts from T.6
- `Documentation/filesystems/ramfs-rootfs-initramfs.rst` — **read this**; it settles the
  initrd/initramfs confusion permanently
- `Documentation/driver-api/virtio/` and `drivers/virtio/virtio_ring.c`

**Papers (the virtualization canon):**
- Popek & Goldberg, "Formal Requirements for Virtualizable Third Generation Architectures"
  (CACM 1974) — **the theorem**
- Robin & Irvine, "Analysis of the Intel Pentium's Ability to Support a Secure Virtual
  Machine Monitor" (USENIX Security 2000) — why x86 failed it
- Bugnion, Devine, Rosenblum et al., "Bringing Virtualization to the x86 Architecture with
  the Original VMware Workstation" (TOCS 2012) — binary translation, told properly
- Barham et al., "Xen and the Art of Virtualization" (SOSP 2003) — paravirtualization
- Adams & Agesen, "A Comparison of Software and Hardware Techniques for x86 Virtualization"
  (ASPLOS 2006) — the surprising result that early VT-x was *slower* than BT
- Russell, "virtio: towards a de-facto standard for virtual I/O devices" (OSR 2008)
- Kivity et al., "kvm: the Linux Virtual Machine Monitor" (OLS 2007)
- Bellard, "QEMU, a Fast and Portable Dynamic Translator" (USENIX ATC 2005)

**Specs:**
- VIRTIO v1.2 specification (OASIS) — skim §2 (Basic Facilities) and §5 (Device Types)
- Intel SDM Vol. 3C, Ch. 23–33 (VMX) — reference only

**QEMU docs:**
- `docs/system/` — machine types, devices, `-object`/`qom`
- `docs/system/replay.rst` — record/replay (Lab 4.7)
- `docs/specs/edu.rst` — the toy PCI device you'll drive in Ch. 37
- virtme-ng: https://github.com/arighi/virtme-ng
- `Documentation/admin-guide/init.rst`

→ Next: [05-first-module.md](05-first-module.md)

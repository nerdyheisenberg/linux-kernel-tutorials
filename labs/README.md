# Labs — Environment, Conventions, and Index

> Every chapter has labs. This document sets up the environment they all assume, establishes
> the conventions, and indexes the labs by what they teach — so you can find "the lab that
> demonstrates DMA ownership" without reading 106 chapters.

---

## 1. The environment

Almost every lab runs in QEMU against a kernel you built. Set this up once.

```bash
#!/bin/bash
# setup.sh — the environment every lab in this curriculum assumes.
set -e

echo "=== 1. Build dependencies (Debian/Ubuntu) ==="
sudo apt update && sudo apt install -y \
	build-essential bc bison flex libssl-dev libelf-dev \
	libncurses-dev dwarves rsync cpio kmod \
	git git-email b4 \
	qemu-system-x86 qemu-system-arm qemu-user-static \
	gcc-aarch64-linux-gnu gcc-riscv64-linux-gnu \
	clang lld llvm \
	gdb crash kexec-tools makedumpfile \
	linux-tools-common linux-tools-generic \
	trace-cmd kernelshark bpfcc-tools bpftrace libbpf-dev \
	python3-drgn drgn \
	sparse smatch coccinelle codespell \
	fio sysstat iotop blktrace stress-ng \
	busybox-static debootstrap \
	pahole # (part of dwarves)

echo "=== 2. Kernel source ==="
mkdir -p ~/src && cd ~/src
[ -d linux ] || git clone --depth=1 -b v6.12 \
	https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux

echo "=== 3. A lab-friendly config ==="
make defconfig
./scripts/config \
	--enable DEBUG_INFO_DWARF5 --enable GDB_SCRIPTS \
	--enable DEBUG_KERNEL --enable DEBUG_FS \
	--enable MAGIC_SYSRQ --enable KALLSYMS_ALL \
	--enable DYNAMIC_DEBUG \
	--enable FTRACE --enable FUNCTION_TRACER \
	--enable FUNCTION_GRAPH_TRACER --enable DYNAMIC_FTRACE \
	--enable IRQSOFF_TRACER --enable PREEMPT_TRACER \
	--enable OSNOISE_TRACER --enable TIMERLAT_TRACER \
	--enable KPROBES --enable UPROBES \
	--enable BPF_SYSCALL --enable BPF_JIT --enable DEBUG_INFO_BTF \
	--enable PERF_EVENTS --enable SCHEDSTATS --enable PSI \
	--enable PROVE_LOCKING --enable LOCK_STAT \
	--enable DEBUG_ATOMIC_SLEEP --enable DEBUG_LIST \
	--enable DEBUG_OBJECTS --enable DEBUG_OBJECTS_TIMERS \
	--enable KASAN --enable KASAN_INLINE \
	--enable UBSAN --enable KFENCE \
	--enable VMAP_STACK \
	--enable KEXEC --enable CRASH_DUMP \
	--enable MODULES --enable MODULE_UNLOAD --enable MODULE_FORCE_UNLOAD \
	--enable SAMPLES --enable SAMPLES_KOBJECT
make olddefconfig
make -j"$(nproc)"

echo "=== 4. A minimal rootfs ==="
cd ~/src && mkdir -p rootfs/{bin,dev,proc,sys,tmp,etc,lib,mnt,root}
cp "$(which busybox)" rootfs/bin/
( cd rootfs && ./bin/busybox --install -s bin )
cat > rootfs/init <<'EOF'
#!/bin/sh
mount -t proc     proc     /proc
mount -t sysfs    sysfs    /sys
mount -t devtmpfs devtmpfs /dev
mount -t debugfs  debugfs  /sys/kernel/debug 2>/dev/null
mount -t tracefs  tracefs  /sys/kernel/tracing 2>/dev/null
mkdir -p /dev/pts && mount -t devpts devpts /dev/pts
echo "=== LAB ENVIRONMENT ($(uname -r)) ==="
exec /bin/sh
EOF
chmod +x rootfs/init
( cd rootfs && find . | cpio -o -H newc --quiet ) | gzip -9 > ~/src/initramfs.cpio.gz

echo "=== 5. The run script ==="
cat > ~/src/run.sh <<'EOF'
#!/bin/bash
# run.sh [extra kernel cmdline...]
KDIR=${KDIR:-~/src/linux}
exec qemu-system-x86_64 \
	-enable-kvm -cpu host -smp 4 -m 2G -nographic \
	-kernel "$KDIR/arch/x86/boot/bzImage" \
	-initrd ~/src/initramfs.cpio.gz \
	-append "console=ttyS0 nokaslr $*" \
	-s
	# -s = gdbstub on :1234.  Add -S to wait for gdb.
EOF
chmod +x ~/src/run.sh

echo "=== Done. Test it: ~/src/run.sh ==="
```

**Verify it works before starting any lab:**

```bash
~/src/run.sh
# At the shell:
uname -a
ls /sys/kernel/debug/tracing/
cat /proc/pressure/cpu
exit   # or Ctrl-A X
```

---

## 2. Conventions used throughout

**Every lab C file that builds a module** assumes this Makefile:

```makefile
obj-m := mymodule.o

KDIR ?= ~/src/linux
PWD  := $(shell pwd)

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean

# For the QEMU rootfs:
install:
	cp *.ko ~/src/rootfs/root/
	( cd ~/src/rootfs && find . | cpio -o -H newc --quiet ) \
		| gzip -9 > ~/src/initramfs.cpio.gz
```

**Every lab that needs a specific kernel config** says so at the top. Enable it with
`./scripts/config --enable FOO && make olddefconfig && make -j$(nproc)`.

**Labs marked DANGEROUS** run only in a VM. They deliberately crash, corrupt, or disable
security. Never on a machine you care about.

**Labs marked AUTHORIZED TESTING ONLY** involve exploitation or escape techniques. Run them
only on systems you own.

**The `→ Ch. NN §T.M` notation** points at the theory section a lab demonstrates.

---

## 3. Debugging with GDB

Used by many labs:

```bash
# Terminal 1
~/src/run.sh          # -s gives a gdbstub on :1234; add -S to wait

# Terminal 2
cd ~/src/linux
gdb vmlinux
(gdb) target remote :1234
(gdb) lx-symbols                 # load module symbols (needs GDB_SCRIPTS)
(gdb) lx-dmesg
(gdb) lx-ps
(gdb) lx-lsmod
(gdb) b do_sys_openat2
(gdb) c
```

`~/.gdbinit` for kernel work:

```
add-auto-load-safe-path ~/src/linux
set disassembly-flavor intel
set print pretty on
```

---

## 4. Labs by what they teach

### Things that are hard to believe until you see them

| Lab | Demonstrates |
|---|---|
| **Ch. 102 Lab 2** | A syscall costs 200–500 ns with mitigations, 20 ns via vDSO. The number that reshaped kernel API design |
| **Ch. 105 Lab 2** | False sharing makes throughput *decrease* as you add threads. The USL β term, live |
| **Ch. 105 Lab 3** | 1.5–2× from correct NUMA first-touch, with no algorithmic change |
| **Ch. 102 Lab 4** | Hardened usercopy turns a silent info leak into a precise kill |
| **Ch. 92 Lab 5** | musl's 128 KB thread stack segfaults code that works on glibc's 8 MB |
| **Ch. 80 Lab 3** | `pin-init` generates exactly the right partial teardown on every failure path |
| **Ch. 76 Lab 7** | seccomp does **not** filter `io_uring` operations |
| **Ch. 94 Lab 2** | Whether your storage stack is actually durable — measured, not assumed |
| **Ch. 104 Lab 3** | 2× CPU oversubscription can cost far more than 2×, because of lock holder preemption |
| **Ch. 103 Lab 5** | Priority inversion, and priority inheritance fixing it, observable in real time |

### Build something from scratch

| Lab | Build |
|---|---|
| **Ch. 104 Lab 1** | A complete hypervisor in 200 lines of C |
| **Ch. 93 Lab 1** | A container in 200 lines of C: namespaces, cgroups, pivot_root, seccomp, caps |
| **Ch. 76 Lab 1** | `io_uring` with no library — the rings and the barriers by hand |
| **Ch. 91 Lab 1** | An initramfs by hand, with a correct `switch_root` |
| **Ch. 84 Lab 2** | A safe Rust abstraction over `struct completion`, and then attack it |
| **Ch. 98 Lab 1** | A complete Yocto BSP: machine, kernel, module, wic layout, image |
| **Ch. 101 Lab 1** | The bring-up harness: one command from source to a running board |

### Diagnose a broken system

| Lab | Fault |
|---|---|
| **Ch. 101 Lab 4** | Ten broken boards, serial console only |
| **Ch. 95 Lab 6** | Five synthetic production faults, USE method |
| **Ch. 90 Lab 3** | Six ways to break a boot, and their exact signatures |
| **Ch. 94 Lab 3** | Failed RAID member, corrupted filesystem, full thin pool, inode exhaustion |
| **Ch. 83 Lab 3** | Compile-time versus runtime versus never-caught driver bugs |
| **Ch. 97 Lab 2** | Every Yocto QA check, triggered on purpose |
| **Ch. 84 Lab 3** | Six unsoundness patterns in Rust abstractions |

### Measure something that decides a design

| Lab | Measurement |
|---|---|
| **Ch. 76 Lab 3** | `pread` vs batched `io_uring` vs registered vs SQPOLL vs IOPOLL |
| **Ch. 105 Lab 5** | Fit α and β, and **predict** the core count at which scaling stops |
| **Ch. 100 Lab 3** | Incremental rebuild cost: Buildroot vs Yocto. The number that decides the project |
| **Ch. 94 Lab 1** | Per-layer cost of RAID → LUKS → LVM → XFS |
| **Ch. 104 Lab 2** | VM exit cost by reason; e1000 vs virtio vs vhost |
| **Ch. 103 Lab 1** | `cyclictest` under load, before and after isolation |
| **Ch. 85 Lab 1** | Rust `rnull` vs C `null_blk` — the performance claim, verified by you |

### Interview preparation

| Artefact | Where |
|---|---|
| The 90-second answers to the twelve most-asked questions | `interview-playbook.md` §8 |
| ~200 questions with model answers, graded by level | `question-bank.md` |
| Six design problems worked end to end | `system-design.md` |
| Ten debugging scenarios, worked | `debugging-scenarios.md` |
| The classic OS theory interviews ask verbatim | `os-fundamentals.md` |
| Latency numbers, decision procedures, frameworks | `cheatsheets.md` |

---

## 5. Hardware, if you have it

Most labs run in QEMU. These benefit from real hardware:

| Hardware | Enables |
|---|---|
| **Any ARM SBC** (Raspberry Pi, BeagleBone, Rock Pi) | Ch. 31–45 peripheral work, Ch. 90 boot, Ch. 101 bring-up |
| **A USB-serial adapter** | Everything in Part 7. Non-negotiable for real bring-up |
| **A logic analyser** (even a $10 one) | I²C, SPI, GPIO verification (Ch. 40–42) |
| **An NVMe SSD** | Ch. 63–69, Ch. 76, Ch. 94 performance work |
| **A multi-socket machine** | Ch. 105 NUMA labs. A cloud instance works |
| **A PCIe device you can bind to vfio-pci** | Ch. 36, Ch. 104 passthrough |
| **A machine with 64+ cores** | Ch. 105 scaling curves |

**If you have none of it:** cloud instances cover the multi-socket and NVMe cases for a few
dollars an hour, and QEMU's `edu` and `pci-testdev` devices cover most driver work.

---

## 6. Suggested lab progression

If you are working through the curriculum rather than dipping in:

| Weeks | Focus | Key labs |
|---|---|---|
| 1–4 | Environment, first module, debugging | Ch. 01, 04, 05, 06 |
| 5–12 | Kernel C, data structures, memory | Ch. 08–12 |
| 13–20 | Concurrency — **spend the time here** | Ch. 13–16, 25 |
| 21–26 | Interrupts, time, processes, scheduling | Ch. 17–21 |
| 27–34 | MM and syscalls | Ch. 22–24 |
| 35–48 | Drivers: build three real ones | Ch. 26–50 |
| 49–60 | Storage: write a filesystem and a block driver | Ch. 51–70 |
| 61–66 | Networking and `io_uring` | Ch. 71–76 |
| 67–74 | Rust: port a driver | Ch. 77–85 |
| 75–78 | Upstream: **get one patch merged** | Ch. 86–89 |
| 79–92 | The OS around the kernel; bring up a board | Ch. 90–101 |
| 93–100 | Specialized domains | Ch. 102–105 |
| Ongoing | Interview readiness | `reference/` |

**The two places to over-invest:** concurrency (weeks 13–20) and getting one patch actually
merged (weeks 75–78). Everything else is knowledge; those two are capability.

---

## 7. How to get value from a lab

1. **Predict before you run.** Write down what you expect. The gap between prediction and
   result is where the learning is; running a lab and watching output teaches almost nothing.

2. **Break it deliberately.** Most labs include a "now break it" section. That section is the
   lab; the working version is the setup.

3. **Change one thing at a time.** Under time pressure the temptation is to change five. It
   never works and it destroys attribution.

4. **Measure, do not assume.** Several labs exist specifically to falsify a widely-believed
   claim.

5. **Write down what you found.** In a month you will not remember. The write-up is also the
   artefact you bring to an interview.

6. **Do the mastery drills.** The labs teach the mechanism; the drills build the capability.
   A chapter's drills typically take 5–20× the lab time, and that is where competence is
   actually built.

---

→ Back: [../README.md](../README.md) | Next: [capstones.md](capstones.md)

# Chapter 90 — Boot Flow: Firmware, UEFI, U-Boot, ATF, Secure Boot

> **Part 7 — The OS around the kernel.** Twelve chapters covering everything that is not the
> kernel but without which the kernel is useless: how it gets started, what runs before it,
> what runs after it, how it is built, and how it is operated. This is the material an
> embedded Linux architect lives in, and it is almost entirely absent from kernel books.
>
> Ch. 04 taught you to boot a kernel in QEMU. This chapter is about what happens on real
> hardware, from the first instruction the CPU executes to `start_kernel()`.

---

## Theory & First Principles

### T.0 — Start here: from one instruction to a login prompt

Press the power button. The CPU begins executing at a fixed physical address, in a mode with
no memory management, no interrupts, no stack, one core, and no knowledge of anything. A few
seconds later you have a preemptive multitasking OS with virtual memory, a network, and a
shell.

**Every step of that journey exists to solve one problem:**

> **At each stage, the code that is running is too small or too restricted to do the next
> thing — so its only job is to set up just enough to load something bigger.**

```
  Boot ROM       masked in silicon, a few KB.  Cannot be changed. Knows almost nothing.
     |           Its ONLY job: find and load the next stage from somewhere fixed.
     v
  SPL / 1st stage  runs in SRAM (a few hundred KB) because DRAM IS NOT INITIALIZED YET.
     |             Its job: initialize the DRAM controller. Now there is real memory.
     v
  Bootloader     U-Boot / GRUB / systemd-boot, now in DRAM with megabytes to work with.
     |           Its job: find a kernel, load it, build the boot arguments, jump.
     v
  Kernel         decompresses itself, sets up page tables, enables the MMU, starts the
     |           other CPUs, probes devices.  Its job: mount a root filesystem.
     v
  initramfs      a tiny userspace in RAM whose ONLY job is to make the REAL root
     |           mountable -- load the RAID/LVM/crypt/NVMe drivers, then switch_root.
     v
  init (PID 1)   the first real process. Its job: start everything else.
     v
  login
```

**Read that chain again and notice the shape: it is a chain of bootstraps, each one loading a
more capable version of itself.** The constraint driving each link is *physical*, not
stylistic: the boot ROM is in mask ROM because there is nowhere else to be; the SPL is in
SRAM because DRAM does not work until something programs its controller; the initramfs exists
because the kernel cannot mount a root filesystem it lacks the driver for, and the driver
lives on that filesystem (Ch. 91 §T.0's chicken-and-egg).

**The general principle, which is the reason to study boot even if you never debug one:**

> **Bootstrapping is the art of ordering initialization when everything depends on something
> else.** The order is not a convention — it is forced by the dependency graph, and every
> "why is it done this way?" question about boot has a physical answer.

You will meet this again in Ch. 101 (board bring-up), where the order of operations is the
entire methodology, and in Ch. 27 §T.0 (probe order versus dependency order) where the same
problem is solved *dynamically* with `-EPROBE_DEFER`.

**Second: where does the kernel learn what hardware exists?** This is the x86/embedded split,
and it is worth having crisply:

| | x86 / server | ARM / embedded |
|---|---|---|
| Discovery | **enumerable buses** — PCI config space, ACPI tables (Ch. 33, 37) | **not enumerable** — an I2C sensor is just wires |
| So the hardware is described by | firmware, at runtime | a **device tree**, compiled and passed in a register |
| Consequence | one kernel image boots any PC | historically one kernel per board (Ch. 32 §T.0's board files) |

**The device tree is the fix for that last row**, and it is exactly Ch. 00 §T.1's principle:
move the *policy* (what hardware this board has) out of the kernel *mechanism* (the drivers)
and into data. One `arm64` kernel now boots thousands of boards.

**Third, and this is the part that makes boot debuggable:** the handoff between each stage is
a small, documented contract — a register holding a pointer to the device tree, a `zImage`
header, an EFI stub's entry point, a `bootparams` struct. **Every boot failure is a contract
violated at one of those handoffs**, and knowing the list turns "it doesn't boot" from
hopeless into a five-way bisection.

```bash
systemd-analyze && systemd-analyze blame | head -15
systemd-analyze critical-chain
dmesg | head -60                        # the kernel's own account of early boot
cat /proc/cmdline                       # what the bootloader passed you
ls /sys/firmware/devicetree/base/       # the device tree, as the kernel sees it
cat /sys/firmware/efi/fw_platform_size 2>/dev/null
```

---

### T.1 — The universal shape of boot

Every platform, from a $2 microcontroller to a 128-core server, follows the same pattern:

```
 power on
   │
   ▼
 [ ROM ]        immutable, in silicon. Minimal. Knows how to load the next stage.
   │            Typically: initialize just enough to read from one boot device.
   ▼
 [ SPL ]        "secondary program loader". Small enough to fit in on-chip SRAM,
   │            because DRAM is not yet initialized. Its job: initialize DRAM.
   ▼
 [ BOOTLOADER ] Now DRAM works, so this can be large. Finds, verifies, and loads
   │            the kernel; builds the handoff structures; hands over.
   ▼
 [ KERNEL ]     decompress, set up paging, start_kernel()
   │
   ▼
 [ INIT ]       userspace
```

**The recurring constraint that explains everything:** each stage runs in a more capable
environment than the last, and its primary job is to *create* the environment the next stage
needs. The ROM cannot be large because it is in silicon. The SPL cannot be large because
DRAM does not work yet. The bootloader can be large because the SPL fixed that.

The second recurring theme is **the chain of trust**: each stage verifies the next before
executing it, rooted in something immutable. A chain whose root is writable is not a chain.

### T.2 — x86: UEFI

```
 power on
   ▼
 CPU starts in 16-bit real mode at 0xFFFFFFF0 (the reset vector)
   ▼
 [ SEC ]   Security phase: CAR (cache-as-RAM), because DRAM is not up
   ▼
 [ PEI ]   Pre-EFI Init: memory init, early chipset
   ▼
 [ DXE ]   Driver Execution Env: the bulk of UEFI. Drivers, protocols, the
   │       services the boot loader will call
   ▼
 [ BDS ]   Boot Device Selection: read the boot order from NVRAM, find an
   │       EFI System Partition, load \EFI\BOOT\BOOTX64.EFI or the
   │       configured loader
   ▼
 [ bootloader ] GRUB, systemd-boot, or the kernel directly (EFI stub)
   ▼
 [ kernel ]
```

**Key UEFI concepts:**

| Concept | What it is |
|---|---|
| **ESP** | EFI System Partition — a FAT32 partition, type `EF00`, usually `/boot/efi` |
| **EFI variables** | NVRAM key-value store; `BootOrder`, `Boot0000`…, `SecureBoot`. Exposed at `/sys/firmware/efi/efivars/` |
| **Boot services** | Available only before `ExitBootServices()`: memory allocation, file I/O, protocol lookup |
| **Runtime services** | Available after: variable access, real-time clock, `ResetSystem`. Called by the kernel |
| **EFI stub** | The kernel itself is a valid PE/COFF EFI application. **The kernel can be its own bootloader** |
| **ACPI tables** | How firmware describes hardware to the OS on x86 (→ Ch. 33) |
| **memory map** | `GetMemoryMap()` — the authoritative description of usable RAM |

**The EFI stub is worth understanding** because it collapses the architecture: with
`CONFIG_EFI_STUB=y`, `vmlinuz` is a PE binary that UEFI can execute directly. It then calls
boot services itself to load the initramfs, get the memory map, set up the graphics mode, and
call `ExitBootServices()`. No GRUB required. This is what `systemd-boot` and Unified Kernel
Images rely on.

**`ExitBootServices()` is the point of no return.** Before it, the firmware owns memory and
devices; after it, the kernel does. The memory map obtained immediately before it is what the
kernel uses forever.

### T.3 — ARM64: the Trusted Firmware chain

ARM's model is built around **exception levels**, which is the security architecture:

```
              Secure World                    Normal World
  EL3   ┌──────────────────────────────────────────────┐
        │  BL31: Secure Monitor (TF-A runtime)         │  ← PSCI lives here
  EL2   ├──────────────────────┬───────────────────────┤
        │  (Secure EL2)        │  Hypervisor (KVM)     │
  EL1   ├──────────────────────┼───────────────────────┤
        │  Trusted OS (OP-TEE) │  Linux kernel         │
  EL0   ├──────────────────────┼───────────────────────┤
        │  Trusted apps        │  Applications         │
        └──────────────────────┴───────────────────────┘
```

The boot sequence:

```
 BL1   ROM. In silicon. Verifies and loads BL2.                        (EL3)
   ▼
 BL2   Trusted Boot Firmware. Initializes DRAM. Loads BL31/32/33.      (EL1S)
   ▼
 BL31  EL3 Runtime: the Secure Monitor. STAYS RESIDENT FOREVER.        (EL3)
   │   Implements PSCI (CPU on/off, suspend, system reset) and SMC
   │   dispatch. Linux calls into it for the rest of the system's life.
   ▼
 BL32  Trusted OS (OP-TEE), optional. Also stays resident.             (S-EL1)
   ▼
 BL33  Non-secure bootloader: U-Boot or UEFI (EDK2).                   (EL2/EL1)
   ▼
 Linux                                                                  (EL2 or EL1)
```

**Three things to internalize:**

1. **BL31 never exits.** It is a resident service, not a boot stage. Every `cpu_up()`, every
   suspend, every `reboot` on ARM64 is an `SMC` instruction trapping to EL3 and calling
   PSCI. If PSCI is broken, SMP does not work and you will chase it for a week.

2. **Linux normally boots at EL2**, not EL1, so that KVM can use EL2 for virtualization. If
   the firmware drops to EL1, KVM is unavailable — a common and confusing BSP bug. Check
   `dmesg | grep "CPU: All CPU(s) started at EL"`.

3. **The secure world is invisible to Linux.** You cannot debug it with kernel tools. When a
   platform misbehaves in a way that makes no sense, suspect firmware — and `hwlatdetect`
   (Ch. 103) is the closest you get to seeing it.

### T.4 — U-Boot

The dominant embedded bootloader, and the one you will actually use.

**The boot sequence inside U-Boot:**

```
 SPL (u-boot-spl.bin)         ~32-64 KB, runs from SRAM
   │  - minimal clock/pinmux
   │  - DRAM init (the whole point)
   │  - load full U-Boot from the same or another boot medium
   ▼
 U-Boot proper (u-boot.img)   ~500 KB-1 MB, runs from DRAM
   │  - full driver model (modelled on Linux's, Ch. 26)
   │  - device tree driven
   │  - environment (variables in NVRAM/eMMC/SPI)
   │  - network, USB, storage, filesystems
   │  - "distro boot" or a boot script
   ▼
 load kernel + DTB (+ initramfs) into DRAM
   ▼
 bootm / booti / bootefi  ->  hand off
```

**The handoff contract on ARM64 (`Documentation/arch/arm64/booting.rst`):**

| Requirement | |
|---|---|
| CPU state | EL2 (preferred) or EL1(NS), MMU off, D-cache off, I-cache on or off |
| `x0` | Physical address of the **device tree blob** |
| `x1`–`x3` | Zero (reserved) |
| Caches | Clean and invalidated for the kernel image and DTB |
| Interrupts | Masked |
| Secondary CPUs | Parked, or PSCI-enabled |

**Getting this wrong is the most common bring-up failure**, and the symptom is a completely
silent boot: the kernel starts, tries to use a stale cacheline or a wrong DTB pointer, and
dies before the console is up. `earlycon` is the tool (§T.7).

**U-Boot's essential commands:**

```
 # Inspect
 printenv                    bdinfo              version
 fdt addr $fdt_addr          fdt print /          fdt list /soc

 # Load
 load mmc 0:1 $kernel_addr_r /boot/Image
 load mmc 0:1 $fdt_addr_r    /boot/dtb/my-board.dtb
 load mmc 0:1 $ramdisk_addr_r /boot/initramfs.cpio.gz
 tftp $kernel_addr_r Image           # over the network -- use this for dev
 nfs  $kernel_addr_r 10.0.0.1:/srv/Image

 # Boot
 booti $kernel_addr_r $ramdisk_addr_r:$ramdisk_size $fdt_addr_r   # arm64
 bootz $kernel_addr_r - $fdt_addr_r                               # arm32
 bootm $kernel_addr_r - $fdt_addr_r                               # legacy uImage
 bootefi $kernel_addr_r $fdt_addr_r                               # EFI stub

 # Modify the DT from the prompt -- invaluable during bring-up
 fdt set /soc/i2c@1000 status "disabled"
 fdt set /chosen bootargs "console=ttyS0,115200 earlycon"

 # Debug
 md 0x40000000 0x40           # memory dump
 mw 0x40000000 0xdeadbeef     # memory write
 i2c probe                    # scan the I2C bus
 mmc info                     # storage
```

**The `fdt set` commands are the single most useful bring-up tool in U-Boot.** You can
disable a misbehaving device, change a property, or fix the bootargs without rebuilding
anything.

### T.5 — The device tree handoff

Both U-Boot and the kernel consume the same DTB (Ch. 32), but U-Boot **modifies** it before
handing over:

| Fixup | What |
|---|---|
| `/chosen/bootargs` | The kernel command line |
| `/chosen/linux,initrd-start` / `-end` | Where the initramfs is |
| `/memory` | The actual detected DRAM size and layout |
| `/chosen/kaslr-seed` | Entropy for KASLR |
| MAC addresses | Read from an EEPROM or fuses, patched into the netdev nodes |
| Serial numbers, board revisions | From fuses |
| Reserved memory | Carve-outs for the secure world, DSPs, framebuffers |

This is why "the DTB in my source tree" and "the DTB the kernel sees" can differ, and why the
first thing to check when a device does not probe is the *runtime* DT:

```bash
# On the running system, the DT the kernel actually got:
ls /sys/firmware/devicetree/base/
dtc -I fs -O dts /sys/firmware/devicetree/base | less
```

### T.6 — Secure boot and the chain of trust

```
 [ ROM / fuses ]  ── verifies ──►  [ SPL/BL2 ]
        immutable root                  │
        (public key hash in eFuse)      ├─ verifies ──► [ BL31 ]
                                        └─ verifies ──► [ U-Boot / UEFI ]
                                                              │
                                                              ├─ verifies ──► [ kernel + DTB ]
                                                              │                    │
                                                              └─ (+ initramfs)     ├─ dm-verity ──► rootfs
                                                                                   └─ IMA ──► runtime files
```

**The requirements for this to be worth anything:**

1. **The root must be immutable.** A public key hash burned into eFuses, or a key in mask
   ROM. If an attacker can replace the root, everything below is theatre.
2. **Every link must verify the next *before* executing it.** A gap anywhere breaks the chain.
3. **Anti-rollback.** A monotonic counter in fuses, so an attacker cannot install an old,
   signed, vulnerable image.
4. **The verified thing must be everything.** Kernel *and* DTB *and* command line — a signed
   kernel with an attacker-controlled `init=/bin/sh` is not secure.
5. **Debug interfaces must be disabled** in production fuses: JTAG, the U-Boot console,
   serial download modes.

**x86 Secure Boot** specifics:

| Key | Role |
|---|---|
| **PK** (Platform Key) | The root; owned by the OEM |
| **KEK** (Key Exchange Key) | Authorizes updates to db/dbx |
| **db** | Allowed signatures/hashes |
| **dbx** | Revoked signatures — the blocklist |
| **MOK** | Machine Owner Key; shim's mechanism for user-enrolled keys |

The **shim** exists because Microsoft's key is in every PC's `db`, and distributions get shim
signed by Microsoft; shim then verifies GRUB and the kernel against the distribution's own
key or a MOK. It is a political solution to a technical problem, and knowing that is useful.

**And the crucial connection to Ch. 102:** secure boot is pointless without **lockdown**. If
root can write `/dev/mem`, load an unsigned module, or `kexec` an unsigned kernel, the
verified chain is bypassed in one command. `CONFIG_SECURITY_LOCKDOWN_LSM` with
`lockdown=integrity` is what closes it, and it is automatically enabled when Secure Boot is
detected.

### T.7 — Debugging a boot that produces nothing

The scenario that defines embedded Linux work. A structured approach:

**Stage 1 — Is anything running at all?**
- Measure current draw. A board in reset draws differently from one executing.
- Toggle a GPIO in the SPL. An LED or a scope probe is the most reliable early signal.
- Check the boot-mode straps/fuses. Is it even trying to boot from the device you flashed?

**Stage 2 — Does the bootloader run?**
- Serial console: correct UART, correct pinmux, correct baud. Verify with a scope if in doubt.
- U-Boot's `preconsole buffer` (`CONFIG_PRE_CONSOLE_BUFFER`) captures output from before the
  console is up.
- `CONFIG_DEBUG_UART` in SPL — direct register writes to the UART, no driver model needed.

**Stage 3 — Does the kernel start?**
- **`earlycon`** is the single most important kernel parameter in bring-up. It uses a
  polled, minimal UART driver available before the full serial driver probes:
  ```
  earlycon=uart8250,mmio32,0x10000000,115200n8
  earlycon=pl011,mmio32,0x9000000
  earlycon                          # let the DT /chosen/stdout-path decide
  ```
- `keep_bootcon` so early output is not discarded when the real console takes over.
- `initcall_debug` to see exactly which initcall hangs.
- `ignore_loglevel` and `debug`.

**Stage 4 — Does it get to userspace?**
- `init=/bin/sh` to bypass init entirely.
- `rdinit=/bin/sh` to stop in the initramfs.
- `root=` wrong? `rootwait`, `rootdelay=10` for slow storage.
- Check the `Kernel panic - not syncing: VFS: Unable to mount root fs` message; it lists the
  block devices the kernel *can* see, which tells you whether it is a driver problem or a
  path problem.

**Stage 5 — It boots but something is wrong.**
- Compare the runtime DT with the source DT (§T.5).
- `/sys/kernel/debug/devices_deferred` — what failed to probe and why.
- `dmesg | grep -i 'fail\|error\|timeout'`.

### T.8 — Boot time

For products, boot time is a requirement, and the engineering is measurable.

**Where the time goes, typically:**

| Stage | Typical | Reducible to |
|---|---|---|
| ROM + SPL | 100–300 ms | mostly fixed; SPL DRAM init dominates |
| U-Boot | 500–2000 ms | **50–200 ms** — this is where the easy wins are |
| Kernel | 300–2000 ms | 200–500 ms |
| initramfs + userspace | 1–20 s | very variable |

**U-Boot reductions, in order of payoff:**
- `bootdelay=0` — removes a literal 2-second wait. Free.
- Disable USB, network, and PCI enumeration if you are not booting from them. Often 500+ ms.
- `CONFIG_SPL_*` — go directly from SPL to the kernel (Falcon mode), skipping U-Boot proper.
  Saves everything above but loses all flexibility; use only when the product is stable.
- Use a smaller, faster-decompressing kernel compression (`lz4` over `gzip`/`xz`) — or no
  compression at all if the storage is fast.

**Kernel reductions:**
- `initcall_debug` + `scripts/bootgraph.pl` to find the expensive initcalls.
- Build in only what you need; modules load later and cost less at boot.
- `quiet` (console output is genuinely slow at 115200 baud — hundreds of ms).
- Defer probe of slow devices; `deferred_probe_timeout`.
- `CONFIG_RD_*` — only the decompressor you use.

**Userspace:**
- `systemd-analyze blame` and `critical-chain` (→ Ch. 91).
- The single biggest win is usually **not starting things you do not need**.

**Measurement discipline:** use a GPIO toggle at each stage and a scope, or `grabserial` with
timestamps. `printk` timestamps are relative to kernel start and miss everything before it.

---

## 1. Internals

### Source map

| Path | Contents |
|---|---|
| `arch/x86/boot/` | The x86 boot protocol, real-mode setup code |
| `arch/x86/boot/header.S` | The boot protocol header — `setup_header` |
| `drivers/firmware/efi/libstub/` | **The EFI stub** — the kernel as an EFI application |
| `arch/arm64/kernel/head.S` | ARM64 entry point; the first kernel instruction |
| `Documentation/arch/arm64/booting.rst` | **The handoff contract.** Read it before any bring-up |
| `Documentation/arch/arm/booting.rst` | Same, for 32-bit |
| `Documentation/arch/x86/boot.rst` | The x86 boot protocol |
| `drivers/of/fdt.c` | Early DT parsing — `early_init_dt_scan` |
| `drivers/firmware/psci/` | PSCI: the SMC calls into BL31 |
| `init/main.c` | `start_kernel()` — where everything converges |
| `kernel/printk/` | `earlycon` and the console handoff |

### `start_kernel()` — what actually happens

```c
/* init/main.c, in outline */
asmlinkage __visible void __init start_kernel(void)
{
	set_task_stack_end_magic(&init_task);
	smp_setup_processor_id();
	boot_cpu_init();
	page_address_init();
	setup_arch(&command_line);      /* arch-specific: memory map, DT/ACPI parse */
	setup_boot_config();
	setup_command_line(command_line);
	setup_nr_cpu_ids();
	setup_per_cpu_areas();          /* per-CPU data (Ch. 16) */
	boot_cpu_hotplug_init();
	build_all_zonelists(NULL);      /* NUMA zonelists (Ch. 105) */
	page_alloc_init();
	parse_early_param();            /* earlycon happens around here */
	...
	trap_init();
	mm_core_init();                 /* buddy allocator live (Ch. 11) */
	sched_init();                   /* scheduler live (Ch. 21) */
	rcu_init();
	early_irq_init();
	init_IRQ();
	tick_init();
	init_timers();
	hrtimers_init();
	softirq_init();
	time_init();
	local_irq_enable();             /* <-- interrupts on for the first time */
	kmem_cache_init_late();
	console_init();                 /* <-- the REAL console; earlycon handed off */
	...
	arch_call_rest_init()
	  -> rest_init()
	       -> kernel_thread(kernel_init, ...)   /* PID 1 eventually */
	       -> kernel_thread(kthreadd, ...)      /* PID 2 */
	       -> cpu_startup_entry(CPUHP_ONLINE)   /* becomes the idle task */
}
```

Then `kernel_init` runs the initcalls in level order, mounts the root filesystem, and
`execve`s `/sbin/init`:

```c
static int __ref kernel_init(void *unused)
{
	kernel_init_freeable();       /* do_initcalls(), SMP bringup, rootfs */
	free_initmem();               /* __init sections freed (Ch. 08) */
	mark_readonly();              /* __ro_after_init enforced (Ch. 102) */
	...
	if (execute_command) {
		ret = run_init_process(execute_command);   /* init= */
		...
	}
	if (!try_to_run_init_process("/sbin/init") ||
	    !try_to_run_init_process("/etc/init")  ||
	    !try_to_run_init_process("/bin/init")  ||
	    !try_to_run_init_process("/bin/sh"))
		return 0;
	panic("No working init found. ...");
}
```

**`free_initmem()` and `mark_readonly()` are the transition from "booting" to "running"**, and
the `Freeing unused kernel image memory` line in dmesg is your marker for it.

### Observability

```bash
# --- What firmware am I on? ---
ls /sys/firmware/                    # efi/ acpi/ devicetree/ dmi/
cat /sys/firmware/efi/fw_platform_size 2>/dev/null   # 64 or 32
dmidecode -t bios                    # x86
cat /sys/firmware/devicetree/base/model 2>/dev/null  # ARM

# --- EFI variables ---
efibootmgr -v                        # boot order and entries
ls /sys/firmware/efi/efivars/ | head
mokutil --sb-state                   # is Secure Boot on?
cat /sys/kernel/security/lockdown

# --- The device tree the kernel ACTUALLY got ---
dtc -I fs -O dts /sys/firmware/devicetree/base > /tmp/runtime.dts
diff <(dtc -I dtb -O dts my-board.dtb) /tmp/runtime.dts

# --- The command line ---
cat /proc/cmdline

# --- Memory map as the firmware described it ---
dmesg | grep -A40 'BIOS-e820\|memblock\|Memory:'
cat /sys/firmware/memmap/*/type 2>/dev/null

# --- Boot timing ---
dmesg -T | head -50                  # kernel timestamps
systemd-analyze                      # firmware/loader/kernel/userspace split
systemd-analyze blame | head -20
systemd-analyze critical-chain
cat /proc/uptime

# --- PSCI (ARM64) ---
dmesg | grep -i psci
dmesg | grep "CPU: All CPU(s) started at EL"

# --- What failed to probe ---
cat /sys/kernel/debug/devices_deferred
```

---

## 2. Practice

### Lab 90.1 — Boot an ARM64 system end to end in QEMU

```bash
#!/bin/bash
# arm64_full_boot.sh — TF-A + U-Boot + kernel, the whole chain.
set -e
WORK=~/boot-lab && mkdir -p $WORK && cd $WORK

echo "=== 1. Trusted Firmware-A ==="
[ -d trusted-firmware-a ] || \
	git clone --depth 1 https://github.com/ARM-software/arm-trusted-firmware.git trusted-firmware-a
cd trusted-firmware-a
make CROSS_COMPILE=aarch64-linux-gnu- PLAT=qemu DEBUG=1 \
     BL33=$WORK/u-boot/u-boot.bin all fip 2>/dev/null || true
cd $WORK

echo "=== 2. U-Boot ==="
[ -d u-boot ] || git clone --depth 1 https://source.denx.de/u-boot/u-boot.git
cd u-boot
make CROSS_COMPILE=aarch64-linux-gnu- qemu_arm64_defconfig
# Turn on the things you want for bring-up:
./scripts/config --enable CMD_FDT --enable CMD_MEMORY --enable CMD_BOOTEFI
make CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
cd $WORK

echo "=== 3. Kernel ==="
cd ~/src/linux
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
./scripts/config --enable EFI_STUB --enable SERIAL_AMBA_PL011_CONSOLE
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc) Image dtbs
cp arch/arm64/boot/Image $WORK/
cd $WORK

echo "=== 4. A minimal rootfs ==="
# (Use busybox or the initramfs from Ch. 04.)

echo "=== 5. Boot it, with the TF-A chain ==="
qemu-system-aarch64 \
	-machine virt,secure=on,virtualization=on \
	-cpu cortex-a57 -m 2G -nographic \
	-bios trusted-firmware-a/build/qemu/debug/bl1.bin \
	-drive if=pflash,format=raw,file=trusted-firmware-a/build/qemu/debug/fip.bin \
	-drive file=disk.img,format=raw,if=virtio
```

Watch the console. You should see, in order: TF-A's `NOTICE: BL1:` banner, `NOTICE: BL2:`,
`NOTICE: BL31:`, then U-Boot's banner, then the kernel. **Being able to identify which stage
a boot died in, from the last line printed, is the core bring-up skill.**

### Lab 90.2 — Interact with U-Boot properly

```
# At the U-Boot prompt (hit a key during the countdown):

=> printenv
=> bdinfo                    # DRAM banks, the DT address, the boot CPU
=> version

# Inspect the device tree U-Boot is holding:
=> fdt addr $fdtcontroladdr
=> fdt print /
=> fdt print /soc/serial@9000000

# Modify it -- this is the bring-up superpower:
=> fdt set /soc/i2c@... status "disabled"
=> fdt set /chosen bootargs "console=ttyAMA0 earlycon=pl011,0x9000000 initcall_debug"

# Load over the network (set this up once; it saves hours):
=> setenv ipaddr 10.0.2.15
=> setenv serverip 10.0.2.2
=> tftp $kernel_addr_r Image
=> tftp $fdt_addr_r my-board.dtb

# Boot:
=> booti $kernel_addr_r - $fdt_addr_r

# Make it persistent:
=> setenv bootcmd 'tftp $kernel_addr_r Image; tftp $fdt_addr_r board.dtb; booti $kernel_addr_r - $fdt_addr_r'
=> setenv bootdelay 1
=> saveenv
```

**Set up TFTP/NFS booting on day one of any bring-up.** Flashing an SD card for each kernel
build is a tax you pay hundreds of times; netboot removes it entirely.

### Lab 90.3 — Deliberately break the boot, five ways

The most valuable lab in the chapter. For each, predict the symptom, then verify.

```bash
# 1. Wrong DTB pointer.
#    U-Boot: booti $kernel_addr_r - 0x0
#    Symptom: completely silent. The kernel cannot find /chosen or
#             /memory. Nothing, not even earlycon (unless it is
#             specified explicitly on the command line).
#    Lesson:  ALWAYS put an explicit earlycon= with hardcoded
#             addresses on the command line during bring-up.

# 2. Wrong load address (overlapping the DTB or the initramfs).
#    Symptom: boots partway, then random corruption, or a hang in
#             decompression ("Uncompressing Linux... " and nothing).
#    Debug:   bdinfo, and check the addresses do not overlap given
#             the image sizes.

# 3. Wrong console in the DT (/chosen/stdout-path).
#    Symptom: kernel boots fine, you see nothing, and it reaches
#             userspace (you can tell by whether it responds to
#             a network ping if networking is up).
#    Debug:   earlycon with an explicit address bypasses the DT.

# 4. Missing root filesystem.
#    Symptom: "VFS: Unable to mount root fs on unknown-block(0,0)"
#             followed by a list of available partitions -- READ THAT
#             LIST, it tells you whether the storage driver worked.
#    Debug:   rootwait, rootdelay=10, or rdinit=/bin/sh to stop in
#             the initramfs and look around.

# 5. Broken PSCI (ARM64).
#    Symptom: boots, but only one CPU comes up. "psci: failed to boot
#             CPU1" or silence about secondary CPUs.
#    Debug:   dmesg | grep -i psci; check /cpus/cpu@N/enable-method
#             in the DT; check that BL31 is actually running.

# 6. Bonus: wrong exception level.
#    Symptom: KVM unavailable. dmesg: "CPU: All CPU(s) started at EL1"
#             and "kvm [1]: HYP mode not available".
#    Debug:   firmware must hand off at EL2.
```

### Lab 90.4 — Set up verified boot

```bash
# --- U-Boot FIT image with signature verification ---

# 1. Generate keys (keep the private key OFF the device, obviously).
mkdir -p keys
openssl genpkey -algorithm RSA -out keys/dev.key \
	-pkeyopt rsa_keygen_bits:2048
openssl req -batch -new -x509 -key keys/dev.key -out keys/dev.crt \
	-subj "/CN=dev signing key"

# 2. An image tree source describing what to sign.
cat > kernel.its <<'EOF'
/dts-v1/;
/ {
	description = "Signed kernel + DTB";
	#address-cells = <1>;

	images {
		kernel {
			description = "Linux kernel";
			data = /incbin/("./Image.gz");
			type = "kernel";
			arch = "arm64";
			os = "linux";
			compression = "gzip";
			load = <0x40080000>;
			entry = <0x40080000>;
			hash-1 { algo = "sha256"; };
		};
		fdt-1 {
			description = "Device tree";
			data = /incbin/("./board.dtb");
			type = "flat_dt";
			arch = "arm64";
			compression = "none";
			hash-1 { algo = "sha256"; };
		};
	};

	configurations {
		default = "conf-1";
		conf-1 {
			description = "Signed boot configuration";
			kernel = "kernel";
			fdt = "fdt-1";
			/* Signing the CONFIGURATION, not just the images,
			 * is what prevents mix-and-match attacks: an
			 * attacker cannot pair a signed old kernel with a
			 * signed new DTB.                              */
			signature-1 {
				algo = "sha256,rsa2048";
				key-name-hint = "dev";
				sign-images = "kernel", "fdt";
			};
		};
	};
};
EOF

# 3. Build and sign. -K embeds the public key into U-Boot's DTB.
mkimage -f kernel.its -k keys -K u-boot.dtb -r kernel.itb

# 4. Rebuild U-Boot with the public key in its control DTB and with
#    CONFIG_FIT_SIGNATURE=y, CONFIG_RSA=y.
#    Now U-Boot will REFUSE to boot an unsigned or modified image.

# 5. TEST THE NEGATIVE CASE. This is the step people skip.
dd if=/dev/urandom of=kernel.itb bs=1 seek=5000 count=16 conv=notrunc
# Boot it. U-Boot must refuse:
#   "Verifying Hash Integrity ... sha256,rsa2048:dev- FAILED"
#   "Bad Data Hash"
# If it boots anyway, your verified boot does not work.
```

**Step 5 is the lab.** A verified-boot setup that has never been tested against a tampered
image is not a verified-boot setup.

### Lab 90.5 — Measure and reduce boot time

```bash
#!/bin/bash
# boot_time.sh — find where the time goes.

echo "=== Overall breakdown ==="
systemd-analyze
#   firmware + loader + kernel + initrd + userspace

echo; echo "=== Kernel: which initcalls are slow? ==="
# boot with: initcall_debug printk.time=1 ignore_loglevel
dmesg | grep initcall | awk '{
	for (i=1;i<=NF;i++) if ($i ~ /after/) { print $(i+1), $2, $3 }
}' | sed 's/usecs//' | sort -rn | head -20

# Or the graphical version:
dmesg > /tmp/dmesg.txt
perl scripts/bootgraph.pl /tmp/dmesg.txt > /tmp/bootgraph.svg

echo; echo "=== Userspace ==="
systemd-analyze blame | head -20
systemd-analyze critical-chain

echo; echo "=== Now cut ==="
cat <<'EOF'
 U-Boot:
   setenv bootdelay 0
   CONFIG_USB=n CONFIG_NET=n CONFIG_PCI=n   (if not booting from them)
   Consider Falcon mode (SPL -> kernel directly)

 Kernel:
   quiet                            # console I/O at 115200 is SLOW
   CONFIG_KERNEL_LZ4=y              # faster decompress than gzip/xz
   Modularize anything not needed at boot
   Trim the initramfs

 Userspace:
   systemctl disable <everything you do not need>
   Static /dev instead of udev, if the device set is fixed
   Avoid shell scripts in the critical path
EOF
```

Target a specific number and iterate. **Measure with a GPIO and a scope for the pre-kernel
stages** — `systemd-analyze`'s firmware/loader numbers come from EFI and are unavailable on
most embedded platforms.

### Lab 90.6 — Bring up a board you have never seen

The simulation of the real job. Given a board with a new SoC and a vendor BSP:

```bash
# --- Phase 1: Signs of life ---
# 1. Find the serial console. Schematic, or probe the likely pins.
# 2. Boot-mode straps: what device is it trying to boot from?
# 3. Vendor's SoC boot ROM protocol (USB DFU, UART download) to load
#    an SPL without flashing anything. Nearly every SoC has one.

# --- Phase 2: Bootloader ---
# 4. Get the vendor's U-Boot building.
# 5. DRAM calibration -- almost always vendor-supplied data. Get it right
#    or you will chase phantom corruption for weeks.
# 6. CONFIG_DEBUG_UART in the SPL for output before the console.

# --- Phase 3: Kernel ---
# 7. Start from the vendor DTS for the nearest reference board.
# 8. Boot with: earlycon=<explicit> initcall_debug ignore_loglevel
#    rdinit=/bin/sh and a busybox initramfs. Get to a SHELL before
#    you care about anything else.
# 9. Then bring up peripherals ONE AT A TIME, in dependency order:
#    clocks -> regulators -> pinctrl -> i2c/spi -> storage -> net -> rest
#    (this is the Part 2 order, and it is the order for a reason)

# --- Phase 4: Make it real ---
# 10. Rootfs on real storage.
# 11. Power management, suspend/resume.
# 12. Secure boot.
# 13. Automate: netboot + a test script, so a kernel change is one command.
```

**The single most important rule: get a shell as early as possible.** With a shell you can
inspect, poke registers with `devmem2`, load modules, and iterate in seconds. Without one,
every experiment is a reflash-and-reboot cycle. Everything else is secondary to that.

---

## 3. Mastery drills

1. Boot an ARM64 system through the full TF-A → U-Boot → kernel chain in QEMU, and identify
   the exact console line at which each stage hands over.

2. Write a U-Boot boot script that implements A/B fallback: try slot A, and on failure (a
   watchdog reset counter) boot slot B. Test both paths.

3. Set up FIT-image signature verification and prove it rejects a tampered image, a
   mismatched configuration, and an unsigned image.

4. Reduce a system's boot time by 50%. Document where every millisecond went before and
   after, measured with a GPIO and a scope for the pre-kernel stages.

5. Implement anti-rollback using a monotonic fuse counter (or emulate it). Demonstrate that a
   correctly-signed older image is refused.

6. Port U-Boot to a new board in QEMU (define a new machine, or use an existing SoC with a
   different DT). Get DRAM, console, and storage working.

7. Trace the ARM64 boot handoff in the source: from U-Boot's `booti` implementation, through
   the kernel's `arch/arm64/kernel/head.S`, to `start_kernel()`. Document every register and
   memory contract.

8. Build a Unified Kernel Image (kernel + initramfs + cmdline in one signed PE binary) and
   boot it with systemd-boot under Secure Boot. Explain why UKIs exist.

9. Debug five deliberately-broken boots created by a colleague, using only the serial console.
   Time yourself.

10. Read `Documentation/arch/arm64/booting.rst` and write a checklist a bootloader author
    could follow. Then audit a real bootloader against it.

11. Set up a complete netboot development loop: TFTP for the kernel and DTB, NFS root, and a
    single command that builds, deploys, and boots. Measure the edit-to-boot cycle time.

12. Design the secure-boot architecture for a product: key hierarchy, key storage and
    rotation, the signing infrastructure, anti-rollback, recovery mode, and what happens when
    the signing key is compromised.

---

## 4. Further reading

**Kernel documentation**
- `Documentation/arch/arm64/booting.rst` — **the contract.** Non-negotiable reading
- `Documentation/arch/arm/booting.rst`
- `Documentation/arch/x86/boot.rst` — the x86 boot protocol
- `Documentation/admin-guide/kernel-parameters.txt` — especially `earlycon`, `initcall_debug`,
  `root=`, `rdinit=`
- `Documentation/admin-guide/efi-stub.rst`

**Firmware**
- U-Boot documentation (`doc/` in the tree) — particularly `doc/develop/driver-model/`,
  `doc/usage/fit/`, and the board-porting guide
- Trusted Firmware-A documentation — the firmware design guide and the PSCI specification
- ARM PSCI specification — read the `CPU_ON`/`CPU_OFF`/`SYSTEM_RESET` sections
- UEFI specification — dense; read the chapters on boot services, variables, and the
  memory map
- ARM Architecture Reference Manual, the exception-model chapter

**Books**
- Chris Simmonds, ***Mastering Embedded Linux Programming***, 3rd ed. — **the best single
  book for Part 7 as a whole.** Chapters 3–4 cover this material
- Vasquez & Simmonds, *Mastering Embedded Linux Development*
- Frank Vasquez's and Chris Simmonds's conference talks on boot time

**Practical**
- The `bootlin` training materials (free, excellent) — particularly the embedded Linux and
  boot-time courses
- `elinux.org` — the Boot Time wiki pages
- Alexandre Belloni and Thomas Petazzoni's boot-time optimization talks
- The `grabserial` tool, for timestamped serial capture

**Cross-references**
- Ch. 04 — QEMU booting, which this generalizes to real hardware
- Ch. 32–33 — device tree and ACPI, the two hardware-description mechanisms
- Ch. 91 — what happens after `execve("/sbin/init")`
- Ch. 98 — Yocto's kernel and bootloader recipes
- Ch. 101 — the full board bring-up, end to end
- Ch. 102 — lockdown, without which secure boot is decorative

→ Next: [91-init-systemd.md](91-init-systemd.md)

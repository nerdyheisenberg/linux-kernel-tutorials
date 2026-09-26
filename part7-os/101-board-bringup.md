# Chapter 101 — Board Bring-Up End to End: A Full BSP from Bare Metal

> The capstone of Part 7, and the chapter that uses everything. You are handed a board that
> has never run Linux. Six to twelve weeks later it must boot a production image with a
> verified chain of trust, an update mechanism, and a maintenance plan. This chapter is the
> sequence, the tooling, the failure modes, and the judgement.

---

## Theory & First Principles

### T.0 — Start here: a board, a schematic, and nothing works

A cardboard box arrives containing the first revision of a custom board. No software has ever
run on it. You have a schematic, a 4,000-page SoC reference manual, and a JTAG probe.
**Where do you start?**

**The wrong answer — and the one that costs teams months — is to build a full image and try to
boot it.** If nothing happens, you have learned nothing: the failure could be power, clocks,
DRAM timing, the boot mode straps, the SD card format, the bootloader, the device tree, or
the kernel. **Nine unknowns, one bit of information.**

**The correct approach is to establish, in order, a chain of things you have *proven*:**

```
  PHASE 0   POWER          Measure every rail with a meter. In the right ORDER.
            & CLOCKS       Verify the crystal is oscillating at the right frequency.
                           -- Nothing else can possibly work. Do not skip this.
  PHASE 1   BOOT ROM       Set the boot mode straps. Does the ROM emit anything on
                           UART? Does it enumerate as a USB device in recovery mode?
                           -- First proof that the CPU is executing instructions.
  PHASE 2   SPL + DRAM     Run the vendor's DRAM calibration. Then run a MEMORY TEST.
                           -- Marginal DRAM is the #1 cause of "random" kernel
                              crashes six months later. Test it now, not then.
  PHASE 3   U-BOOT         A prompt. Now you have an interactive tool: peek/poke
                           registers, test storage, test Ethernet, load over TFTP.
  PHASE 4   KERNEL         earlycon FIRST. A kernel that boots but prints nothing
                           is indistinguishable from one that does not boot.
  PHASE 5   DEVICE TREE    Bring up peripherals ONE AT A TIME, each verified.
  PHASE 6   USERSPACE      Now, finally, an image.
```

**The order is not a convention — it is forced by the dependency graph** (Ch. 90 §T.0). Each
phase can only be debugged if the previous one is *known good*, and the whole method is:

> **Never debug two unknowns at once.** Establish a known-good foundation, then add exactly
> one variable. When something fails, you know what changed.

**That is bisection applied to hardware**, and it is the same discipline as Ch. 94 §T.0's
"measure at two adjacent layers."

**Three specific pieces of hard-won advice**, each of which routinely saves weeks:

1. **Get `earlycon` working before anything else in the kernel.** `earlycon` uses a
   hardcoded UART address and starts before the device tree, before `console=`, before driver
   probing. Without it, every early-boot failure looks identical: silence. **Observability
   must be established before the thing you want to observe.**
2. **Test DRAM properly, under temperature.** Vendor calibration gets you booting; it does
   not prove margin. A board that passes at 25 °C and fails at 70 °C will present as random
   kernel oopses, KASAN reports in unrelated code, and filesystem corruption — and you will
   spend a month looking for a software bug that does not exist.
3. **Suspect the hardware, but prove it.** First-revision boards have real defects: an
   unpopulated pull-up, a swapped differential pair, a strap resistor on the wrong side. The
   discipline is to form a falsifiable hypothesis and test it with a scope or a meter —
   **not** to assume hardware whenever software is hard.

**And the political reality, which is as much of the job as the technical part:** you are
working with a schematic that may be wrong, a reference manual that is certainly incomplete,
a vendor BSP of unknown quality (Ch. 98 §T.0), and hardware engineers who need your findings
in time to influence the next board revision. **Bring-up is a communication task wearing a
debugging costume**, and the deliverable is not "it boots" — it is a list of board errata with
evidence, delivered early enough to matter.

```bash
# earlycon, before anything else:
setenv bootargs "earlycon=uart8250,mmio32,0x02530000 console=ttyS0,115200 initcall_debug"

# in U-Boot, an interactive hardware debugger:
md 0x02530000 4          # read registers
mw 0x02530000 0x1        # write them
mtest 0x80000000 0x90000000   # a DRAM test

# in the kernel:
dmesg | grep -i 'probe\|defer\|fail'
cat /sys/kernel/debug/clk/clk_summary
cat /sys/kernel/debug/pinctrl/*/pinmux-pins
cat /sys/kernel/debug/regmap/*/registers
```

---

### T.1 — The phases, and why the order is fixed

```
 0. PREPARATION       schematics, datasheets, vendor BSP, lab setup       (1 week)
 1. SIGNS OF LIFE     power, clocks, boot ROM, serial console             (days)
 2. BOOTLOADER        SPL, DRAM calibration, U-Boot, storage, network     (1-2 weeks)
 3. KERNEL BOOT       earlycon to a shell on an initramfs                 (days-1 week)
 4. PERIPHERALS       one at a time, in dependency order                  (2-6 weeks)
 5. USERSPACE         rootfs on real storage, init, applications          (1 week)
 6. PRODUCTION        secure boot, update, hardening, compliance          (2-4 weeks)
 7. MAINTENANCE       upstreaming, uprev cadence, support plan            (forever)
```

**The order is not negotiable, and the reason is that each phase creates the instrument for
the next.** You cannot debug DRAM without a console. You cannot debug a driver without a
shell. You cannot productionize what does not boot. Teams that try to parallelize by having
someone "start on the application" while the board does not boot waste that person's time.

**The single most important goal: get a shell, as early as possible.** With a shell you have
`devmem2`, `dmesg`, module loading, and a hundred experiments per hour. Without one, every
experiment is a reflash-and-reboot cycle measured in minutes. **Phase 3 exists to buy you
Phase 4's velocity**, and it is worth over-investing in.

### T.2 — Phase 0: preparation

What you need before touching the board:

| Item | Why |
|---|---|
| **Schematics** | Which pins, which peripherals, which power rails, which straps |
| **SoC datasheet and TRM** | Register maps, clock trees, boot modes |
| **DRAM datasheet + the vendor's calibration tool** | You will not derive timings yourself |
| **The vendor BSP** | Even if you will not ship it, it is the reference |
| **A reference board that works** | Comparison is the most powerful debugging technique available |
| **Boot-mode strap documentation** | Which pins select which boot device |
| **Power sequencing requirements** | Getting this wrong can damage the board |

Lab setup, and every item earns its cost within a week:

| Tool | Use |
|---|---|
| **USB-serial adapter** (3.3 V, verify the level!) | The console. Non-negotiable |
| **Bench power supply with current display** | "Is it executing?" answered by current draw |
| **Oscilloscope** or logic analyser | Clocks, resets, GPIO toggles, serial verification |
| **JTAG debugger** (J-Link, DSTREAM, or OpenOCD + FT2232) | The only way to debug before the console |
| **Network with TFTP/NFS/DHCP** | Netboot; this alone saves days |
| **A relay or USB-controlled power switch** | Automated power cycling for CI |
| **SD card / eMMC programmer** | Recovery when you brick the boot medium |

**Set up netboot on day one.** Flashing an SD card for each kernel build is a tax you pay
hundreds of times. TFTP for the kernel and DTB plus NFS root turns a 5-minute cycle into a
20-second one.

### T.3 — Phase 1: signs of life

The question is binary: **is the CPU executing anything?**

```
 1. Power rails       measure each with a meter. In sequence, to spec.
 2. Reset             is it released? Scope it.
 3. Clocks            is the main oscillator running? Scope it.
 4. Current draw      a CPU in reset draws differently from one executing.
                      A change when you release reset means it is running.
 5. Boot-mode straps  read them with a meter. Is it trying to boot from
                      the device you think?
 6. Boot ROM          does it respond to the vendor's serial/USB download
                      protocol? THIS IS THE KEY TEST -- nearly every SoC
                      has one, and a response proves the ROM is running.
```

**The SoC's serial download mode is the most valuable bring-up facility that exists.**
i.MX has `imx-usb-loader`/`uuu`, TI has the UART/USB boot ROM, Rockchip has `rkdeveloptool`,
Allwinner has FEL, Qualcomm has EDL. It lets you load an SPL into SRAM over USB **without
flashing anything**, which means a broken image never bricks the board and the edit-test
cycle is seconds.

If there is no response: check straps, check the power sequence, check the oscillator,
check that the reset is actually released, and check for a shorted rail. Then get the JTAG
out.

### T.4 — Phase 2: the bootloader

**The console first.** Before anything else, get output:

```c
/* U-Boot SPL: CONFIG_DEBUG_UART writes directly to the UART registers,
 * with no driver model, no clock framework, no pinmux -- so it works
 * before anything else does.                                          */
CONFIG_DEBUG_UART=y
CONFIG_DEBUG_UART_BASE=0x02020000
CONFIG_DEBUG_UART_CLOCK=24000000
CONFIG_DEBUG_UART_SHIFT=0
CONFIG_DEBUG_UART_ANNOUNCE=y
```

Verify the serial line with a scope if you get nothing: wrong baud produces garbage, wrong
pin produces silence, wrong voltage level can produce either.

**Then DRAM. This is the hardest part of bring-up and the one that produces the worst bugs.**

- The timings come from the vendor's calibration tool, driven by the DRAM part number and
  the PCB routing. **Do not guess.**
- Marginal DRAM does not fail cleanly. It produces occasional bit flips that manifest weeks
  later as random crashes, filesystem corruption, and "the kernel is buggy."
- **Test it exhaustively before proceeding.** `memtester` in U-Boot, the vendor's stress
  tool, across the full temperature range and at both voltage extremes.

```
 => mtest 0x40000000 0x40100000 0xdeadbeef 1000
 => md 0x40000000 0x40
 # Better: the vendor's DDR stress tool, at -40C, 25C, and +85C.
 # A board that passes at 25C and fails at 85C has a timing margin
 # problem that WILL be a field return.
```

Then, in order: storage (eMMC/SD/SPI-NOR), network (for TFTP), and the environment. Finally,
set up the boot sequence.

**The handoff contract (Ch. 90 §T.4) must be exactly right**: the DTB address in `x0`, caches
cleaned, EL2 on ARM64, interrupts masked. Getting it wrong produces a completely silent
failure.

### T.5 — Phase 3: the kernel to a shell

The minimum-viable kernel boot, with every diagnostic on:

```bash
# Kernel command line -- use ALL of these during bring-up:
earlycon=uart8250,mmio32,0x02020000,115200n8   # EXPLICIT address, not the DT
console=ttyS0,115200
initcall_debug                                  # which initcall hangs
ignore_loglevel
keep_bootcon                                    # do not drop early output
rdinit=/bin/sh                                  # stop in the initramfs
```

```bash
# A minimal kernel config: the less there is, the less can break.
make ARCH=arm64 defconfig
./scripts/config --enable SERIAL_8250_CONSOLE --enable SERIAL_EARLYCON
./scripts/config --enable BLK_DEV_INITRD --enable RD_GZIP
./scripts/config --enable DEVTMPFS --enable DEVTMPFS_MOUNT
./scripts/config --enable MAGIC_SYSRQ --enable DEBUG_INFO_DWARF5
./scripts/config --disable MODULE_SIG --disable SECURITY_LOCKDOWN_LSM
#   ^ hardening comes in Phase 6, not now.
```

**The `earlycon` with an explicit address is the single most important thing here**, because
it works before the DT is parsed and before any driver probes. A kernel that dies before
`earlycon` is a handoff problem (Ch. 90 Lab 3); one that dies after gives you output to read.

Start from the vendor DTS for the nearest reference board and delete everything you do not
need. A DTS with fewer nodes has fewer things that can fail to probe.

**Success criterion for Phase 3: a shell prompt on a busybox initramfs.** Nothing else
matters yet. Not storage, not networking, not the application.

### T.6 — Phase 4: peripherals, in dependency order

**The order is a dependency graph, and it matches Part 2's chapter order for a reason:**

```
 1. Clocks       (Ch. 43)  -- everything depends on a clock
 2. Regulators   (Ch. 43)  -- everything depends on power
 3. Pinctrl/GPIO (Ch. 42)  -- every pin must be muxed correctly
 4. I2C / SPI    (Ch. 40-41) -- the buses that configure everything else
 5. PMIC         (over I2C) -- rails for the rest
 6. Storage      (Ch. 68-70) -- eMMC/SD/NAND
 7. Network      (Ch. 46)   -- MAC + PHY
 8. USB          (Ch. 39)
 9. Display/camera (Ch. 47)
 10. Audio, sensors, the rest
```

**Bring up one at a time and verify each before moving on.** The temptation to enable
everything and see what works produces a board where five things are broken and each is
masking the others.

```bash
# The verification loop for each peripheral:
dmesg | grep -i <peripheral>
ls /sys/bus/<bus>/devices/
cat /sys/kernel/debug/devices_deferred        # what failed to probe, and why
cat /sys/kernel/debug/clk/clk_summary          # the whole clock tree
cat /sys/kernel/debug/pinctrl/*/pinmux-pins    # every pin's mux state
cat /sys/kernel/debug/regulator/regulator_summary
i2cdetect -y 1                                 # what is on the bus
devmem2 0x02020000 w                           # read a register directly
```

**`/sys/kernel/debug/devices_deferred` is the first thing to check** when a driver does not
appear. It lists every device whose probe returned `-EPROBE_DEFER` and, in recent kernels,
what it was waiting for.

**The three most common Phase 4 failures:**

1. **Pinmux wrong.** The peripheral's driver probes fine and the hardware does nothing,
   because the pins are muxed to something else. Check `pinmux-pins`, and scope the pin.
2. **Clock missing or wrong rate.** Check `clk_summary`; a peripheral clocked at 0 Hz is
   silently dead.
3. **Regulator not enabled.** Check `regulator_summary`. `regulator-always-on` during
   bring-up eliminates a variable.

### T.7 — Phase 5: userspace

Now the build system (Ch. 96–100) enters:

```
 1. Choose: Yocto or Buildroot (Ch. 100 §T.7)
 2. Create the BSP layer (Ch. 98)
      - machine conf
      - kernel recipe with your defconfig and fragments
      - bootloader recipe
      - wic layout
 3. Build a minimal image, boot from real storage
 4. Add init, networking, and the services
 5. Add the application
 6. Automate: a single command from source to running board
```

**The automation in step 6 is the deliverable that matters.** By the end of Phase 5 there
should be one command that builds everything, writes it to the board (or netboots it), and
runs a smoke test. Every subsequent phase depends on that loop being fast.

### T.8 — Phase 6: production

Everything from Ch. 99, plus the secure-boot chain from Ch. 90:

```
 [ ] Secure boot chain, rooted in fuses          (Ch. 90 §T.6)
 [ ] Anti-rollback counter
 [ ] JTAG and serial download disabled in production fuses
 [ ] Kernel hardening config                     (Ch. 102)
 [ ] lockdown=integrity (or confidentiality)
 [ ] Module signing enforced
 [ ] dm-verity on the root filesystem
 [ ] Production image: no debug-tweaks, real credentials  (Ch. 99 §T.2)
 [ ] A/B update with verified bundles and rollback  (Ch. 99 §T.8)
 [ ] Licence compliance package                  (Ch. 99 §T.3)
 [ ] SBOM                                        (Ch. 99 §T.4)
 [ ] Reproducible build                          (Ch. 99 §T.7)
 [ ] Watchdog enabled and fed
 [ ] Power-fail safety verified                  (Ch. 94 Lab 2)
 [ ] Thermal and power characterization
 [ ] Ten-year archive                            (Ch. 99 §T.10)
```

**The fuse-burning decisions are irreversible**, so they need a documented procedure, a
staging process, and sign-off. A mistake here scraps boards.

### T.9 — Phase 7: the maintenance plan

The phase everyone skips, and the one that determines the product's cost over ten years.

| Question | Decision needed |
|---|---|
| **Which kernel?** | The newest LTS the vendor supports. Check its EOL against the product's (Ch. 88 §T.9) |
| **Uprev cadence?** | Weekly stable, annually for LTS. Automate it |
| **What is the delta?** | Measure it (Ch. 88 Lab 1). Categorize every patch. Own each one |
| **Upstreaming plan?** | Which patches, by when, by whom (Ch. 86) |
| **CVE process?** | Continuous stable adoption; document exceptions (Ch. 88 §T.6) |
| **What happens at LTS EOL?** | Plan the migration *before* you need it |
| **Who maintains this?** | A named person, with a successor (Ch. 87 §T.7) |

**The failure mode to name explicitly:** a product picks an old LTS because the vendor BSP
targets it, accumulates a growing delta, and discovers at year six that moving to a supported
kernel is a multi-quarter project it cannot fund. This is the most common serious failure in
embedded Linux, it is entirely preventable, and prevention costs a few hours a month from day
one.

### T.10 — The judgement

Things that separate a bring-up that goes well from one that does not:

1. **Compare with a working board.** If a reference design works and yours does not, the
   difference is the bug. Bisect the difference — schematic, DTS, config — rather than
   theorizing.

2. **Change one thing at a time.** Under schedule pressure the temptation is to change five
   things and see. It never works and it destroys your ability to attribute the fix.

3. **Get the shell early.** §T.1. Over-invest in it.

4. **Automate the loop immediately.** Netboot, a build script, a smoke test. An engineer
   doing a 5-minute manual cycle 60 times a day is spending 5 hours flashing.

5. **Do not skip DRAM validation.** Marginal DRAM produces bugs that look like software bugs
   for months.

6. **Write it down as you go.** Every register poke, every workaround, every "we tried X and
   it did not work." In month four you will not remember, and your successor certainly will
   not.

7. **Know when to ask the vendor.** Undocumented errata are real, and the vendor's FAE has
   seen your problem before. An hour of your time versus a week of theirs is a good trade.

8. **Know when the answer is "this should not run Linux."** A 100 µs hard deadline on a
   single-core device is an RTOS problem (Ch. 103 §T.10, Ch. 100 §T.10). Saying so is
   judgement, not defeat.

---

## 1. Internals

### The complete diagnostic surface

```bash
# ===== BOOTLOADER (U-Boot) =====
bdinfo                    # DRAM banks, DT address, boot CPU
printenv                  # the environment
version
mtest 0x40000000 0x40100000 0xdeadbeef 100     # DRAM test
md 0x02020000 0x20        # read registers directly
mw 0x02020000 0x1         # write
i2c dev 0 && i2c probe    # what is on the bus
mmc info && mmc list
fdt addr $fdtcontroladdr && fdt print /
gpio status -a
clk dump

# ===== KERNEL: early =====
# cmdline: earlycon=<explicit> initcall_debug ignore_loglevel keep_bootcon
dmesg | head -50
cat /proc/cmdline
cat /proc/iomem /proc/interrupts /proc/ioports

# ===== DEVICE TREE: what the kernel ACTUALLY got =====
dtc -I fs -O dts /sys/firmware/devicetree/base > /tmp/runtime.dts
diff <(dtc -I dtb -O dts board.dtb 2>/dev/null) /tmp/runtime.dts
ls /proc/device-tree/

# ===== PROBE FAILURES =====
cat /sys/kernel/debug/devices_deferred     # <- FIRST THING TO CHECK
ls /sys/bus/platform/devices/
ls /sys/bus/platform/drivers/
for d in /sys/bus/*/devices/*; do
    [ -e "$d/driver" ] || echo "UNBOUND: $d"
done

# ===== CLOCKS =====
cat /sys/kernel/debug/clk/clk_summary
cat /sys/kernel/debug/clk/<clk>/clk_rate

# ===== PINCTRL =====
cat /sys/kernel/debug/pinctrl/*/pinmux-pins
cat /sys/kernel/debug/pinctrl/*/pins
cat /sys/kernel/debug/pinctrl/*/pinconf-pins

# ===== REGULATORS =====
cat /sys/kernel/debug/regulator/regulator_summary

# ===== GPIO =====
gpioinfo
gpiodetect
gpioget gpiochip0 5
gpioset gpiochip0 5=1

# ===== BUSES =====
i2cdetect -l && i2cdetect -y 1
i2cdump -y 1 0x50
spidev_test -D /dev/spidev0.0 -v
lsusb -t
lspci -vv

# ===== POWER =====
cat /sys/kernel/debug/pm_genpd/pm_genpd_summary
cat /sys/power/state
echo devices > /sys/power/pm_test && echo mem > /sys/power/state

# ===== RAW REGISTERS =====
devmem2 0x02020000 w
busybox devmem 0x02020000 32

# ===== KERNEL DEBUGGING =====
echo 'file drivers/foo/* +p' > /sys/kernel/debug/dynamic_debug/control
echo 8 > /proc/sys/kernel/printk
echo function_graph > /sys/kernel/debug/tracing/current_tracer
```

### The bring-up device tree, minimal

```dts
// Start here. Add nodes ONE AT A TIME.
/dts-v1/;
#include "mysoc.dtsi"

/ {
	model = "My Board Rev A";
	compatible = "mycompany,myboard", "vendor,mysoc";

	chosen {
		stdout-path = &uart0;
		bootargs = "console=ttyS0,115200 earlycon";
	};

	memory@40000000 {
		device_type = "memory";
		reg = <0x0 0x40000000 0x0 0x40000000>;  /* 1 GB */
	};

	/* During bring-up: always-on, so power is never the variable. */
	reg_3v3: regulator-3v3 {
		compatible = "regulator-fixed";
		regulator-name = "3v3";
		regulator-min-microvolt = <3300000>;
		regulator-max-microvolt = <3300000>;
		regulator-always-on;
		regulator-boot-on;
	};
};

&uart0 {
	status = "okay";
	pinctrl-names = "default";
	pinctrl-0 = <&uart0_pins>;
};

/* Everything else starts disabled. Enable one at a time. */
&i2c0  { status = "disabled"; };
&spi0  { status = "disabled"; };
&mmc0  { status = "disabled"; };
&eth0  { status = "disabled"; };
&usb0  { status = "disabled"; };
```

---

## 2. Practice

### Lab 101.1 — The bring-up harness

```bash
#!/bin/bash
# bringup.sh — one command from source to running board.
# THE most valuable artefact of the whole bring-up.
set -e

BOARD=${BOARD:-myboard}
KDIR=${KDIR:-$HOME/src/linux}
UDIR=${UDIR:-$HOME/src/u-boot}
TFTP=${TFTP:-/srv/tftp}
NFSROOT=${NFSROOT:-/srv/nfs/$BOARD}
SERIAL=${SERIAL:-/dev/ttyUSB0}
BAUD=${BAUD:-115200}
POWER_CTL=${POWER_CTL:-}          # a script that power-cycles the board

log() { printf '\033[1;34m==> %s\033[0m\n' "$*"; }

cmd_kernel() {
	log "Building kernel"
	cd "$KDIR"
	make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j"$(nproc)" \
		Image dtbs modules
	cp arch/arm64/boot/Image "$TFTP/"
	cp arch/arm64/boot/dts/vendor/${BOARD}.dtb "$TFTP/"
	log "Installing modules into the NFS root"
	sudo make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
		INSTALL_MOD_PATH="$NFSROOT" modules_install
	sync
}

cmd_uboot() {
	log "Building U-Boot"
	cd "$UDIR"
	make CROSS_COMPILE=aarch64-linux-gnu- ${BOARD}_defconfig
	make CROSS_COMPILE=aarch64-linux-gnu- -j"$(nproc)"
	cp u-boot.bin spl/u-boot-spl.bin "$TFTP/" 2>/dev/null || true
}

cmd_initramfs() {
	log "Building a busybox initramfs"
	D=$(mktemp -d)
	mkdir -p "$D"/{bin,sbin,dev,proc,sys,mnt}
	cp "$(which busybox)" "$D/bin/"
	( cd "$D" && ./bin/busybox --install -s bin )
	cat > "$D/init" <<'EOF'
#!/bin/sh
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev
echo "=== BRING-UP SHELL ==="
echo "cmdline: $(cat /proc/cmdline)"
exec /bin/sh
EOF
	chmod +x "$D/init"
	( cd "$D" && find . | cpio -o -H newc --quiet ) | gzip -9 \
		> "$TFTP/initramfs.cpio.gz"
	rm -rf "$D"
}

cmd_boot() {
	log "Power-cycling and booting"
	[ -n "$POWER_CTL" ] && $POWER_CTL off && sleep 2 && $POWER_CTL on
	# Interrupt U-Boot and drive it from the host:
	( sleep 1
	  printf '\n'
	  printf 'setenv autoload no; dhcp\n'
	  printf "tftp \$kernel_addr_r Image\n"
	  printf "tftp \$fdt_addr_r ${BOARD}.dtb\n"
	  printf "tftp \$ramdisk_addr_r initramfs.cpio.gz\n"
	  printf "setenv bootargs 'console=ttyS0,%s earlycon initcall_debug ignore_loglevel'\n" "$BAUD"
	  printf 'booti $kernel_addr_r $ramdisk_addr_r:$filesize $fdt_addr_r\n'
	) | picocom -b "$BAUD" "$SERIAL" --omap crlf
}

cmd_smoke() {
	log "Smoke test over serial"
	# Expect a prompt, then run checks and compare output.
	# (Use `expect` or pexpect for a real implementation.)
	cat <<'EOF'
 Checks to run at the shell:
   cat /proc/cmdline
   dmesg | grep -ci error
   cat /sys/kernel/debug/devices_deferred      # must be EMPTY
   ls /sys/bus/platform/devices/ | wc -l
   cat /sys/kernel/debug/clk/clk_summary | head -20
EOF
}

case "${1:-all}" in
	kernel)    cmd_kernel ;;
	uboot)     cmd_uboot ;;
	initramfs) cmd_initramfs ;;
	boot)      cmd_boot ;;
	smoke)     cmd_smoke ;;
	all)       cmd_kernel; cmd_initramfs; cmd_boot ;;
	*) echo "usage: $0 {kernel|uboot|initramfs|boot|smoke|all}"; exit 1 ;;
esac
```

**Build this on day one.** Every hour spent on it returns tenfold.

### Lab 101.2 — Simulate a bring-up in QEMU

You can practise the whole sequence without hardware.

```bash
#!/bin/bash
# virtual_bringup.sh — a "new board" in QEMU, brought up from scratch.
set -e
W=~/bringup-sim && mkdir -p $W && cd $W

# --- Phase 2: bootloader ---
[ -d u-boot ] || git clone --depth 1 https://source.denx.de/u-boot/u-boot.git
cd u-boot
make CROSS_COMPILE=aarch64-linux-gnu- qemu_arm64_defconfig
./scripts/config --enable CMD_FDT --enable CMD_MEMORY --enable CMD_BOOTEFI \
                 --enable DEBUG_UART
make CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
cd $W

# --- Phase 3: kernel to a shell ---
cd ~/src/linux
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc) Image
cp arch/arm64/boot/Image $W/
cd $W

# Minimal initramfs
D=$(mktemp -d)
mkdir -p $D/{bin,dev,proc,sys}
cp $(which busybox) $D/bin/ 2>/dev/null || \
	cp /usr/bin/busybox $D/bin/
(cd $D && ./bin/busybox --install -s bin)
cat > $D/init <<'EOF'
#!/bin/sh
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev
echo "=== PHASE 3 COMPLETE: SHELL ==="
exec /bin/sh
EOF
chmod +x $D/init
(cd $D && find . | cpio -o -H newc --quiet) | gzip -9 > initramfs.cpio.gz
rm -rf $D

# --- Boot, with every diagnostic on ---
qemu-system-aarch64 \
	-machine virt -cpu cortex-a57 -smp 4 -m 2048 -nographic \
	-kernel Image -initrd initramfs.cpio.gz \
	-append "console=ttyAMA0 earlycon=pl011,0x9000000 initcall_debug ignore_loglevel rdinit=/bin/sh"

# --- Then practise the failures of Ch. 90 Lab 3 ---
#   wrong DTB address, wrong console, missing root, overlapping loads.
```

### Lab 101.3 — Peripheral bring-up, one at a time

```bash
#!/bin/bash
# peripheral.sh — the verification loop for each peripheral.
PERIPH=${1:?peripheral name, e.g. i2c0}

echo "=== 1. Is it in the device tree, and enabled? ==="
dtc -I fs -O dts /sys/firmware/devicetree/base 2>/dev/null \
	| grep -A20 "$PERIPH" | head -25

echo; echo "=== 2. Did a device get created? ==="
ls -d /sys/bus/*/devices/*${PERIPH}* 2>/dev/null || echo "  NO DEVICE"

echo; echo "=== 3. Is a driver bound? ==="
for d in /sys/bus/*/devices/*${PERIPH}*; do
	[ -e "$d/driver" ] && echo "  BOUND: $(basename $(readlink $d/driver))" \
	                   || echo "  UNBOUND: $d"
done

echo; echo "=== 4. Deferred probe? ==="
grep -i "$PERIPH" /sys/kernel/debug/devices_deferred 2>/dev/null \
	|| echo "  not deferred"

echo; echo "=== 5. Clocks ==="
grep -i "$PERIPH" /sys/kernel/debug/clk/clk_summary 2>/dev/null \
	|| echo "  (grep the clk_summary manually; names rarely match)"

echo; echo "=== 6. Pinmux ==="
grep -i "$PERIPH" /sys/kernel/debug/pinctrl/*/pinmux-pins 2>/dev/null \
	|| echo "  NO PINS MUXED -- this is failure mode #1"

echo; echo "=== 7. Regulators ==="
cat /sys/kernel/debug/regulator/regulator_summary 2>/dev/null | head -20

echo; echo "=== 8. Interrupts ==="
grep -i "$PERIPH" /proc/interrupts || echo "  no interrupts registered"

echo; echo "=== 9. Kernel messages ==="
dmesg | grep -i "$PERIPH" | tail -20

echo; echo "=== 10. Turn on dynamic debug for its driver ==="
echo "  echo 'file drivers/<subsys>/<driver>.c +p' > /sys/kernel/debug/dynamic_debug/control"
echo "  dmesg -w"
```

Work through: clocks → regulators → pinctrl → i2c → PMIC → storage → network → USB → the
rest. **Verify each before moving on.**

### Lab 101.4 — Diagnose ten broken boards

The highest-value lab in the chapter. Have a colleague introduce one fault at a time; you
diagnose it using only the serial console and the tools in §T.9.

| # | Fault | Expected symptom |
|---|---|---|
| 1 | Wrong DTB address in `booti` | Completely silent, even with `earlycon` in the DT |
| 2 | Kernel load address overlaps the initramfs | Hangs at "Uncompressing Linux" or random corruption |
| 3 | Wrong `stdout-path` in `/chosen` | Boots to userspace, no output; explicit `earlycon` still works |
| 4 | Peripheral pin muxed to the wrong function | Driver probes, hardware does nothing |
| 5 | Clock parent wrong → peripheral clocked at 0 Hz | Driver probes, timeouts on every transaction |
| 6 | Regulator not enabled | Driver probes, device does not respond at all |
| 7 | Missing `rootwait`, slow eMMC | "VFS: Unable to mount root fs", but only sometimes |
| 8 | PSCI broken in firmware | Boots on one CPU; "psci: failed to boot CPU1" |
| 9 | Firmware hands off at EL1 not EL2 | Boots fine; KVM unavailable, "HYP mode not available" |
| 10 | Marginal DRAM timing | Boots, works, then random crashes and FS corruption hours later |

**Fault 10 is the one that matters most and is the hardest.** The symptom looks like a
software bug for weeks. The diagnostic is: run the vendor's DRAM stress test across the
temperature range, and compare a known-good board.

For each: record the symptom, the diagnostic that identified it, and the fix. **That document
is your bring-up runbook** and it is what you hand to the next person.

### Lab 101.5 — The full BSP deliverable

Produce, for a real or simulated board, the complete set:

```
 meta-myboard/                          (Ch. 98)
 ├── conf/machine/myboard.conf
 ├── recipes-kernel/linux/
 │   ├── linux-myboard_6.6.bb
 │   └── linux-myboard/
 │       ├── defconfig
 │       ├── myboard-hardware.cfg
 │       ├── myboard-security.cfg      (Ch. 102)
 │       └── *.patch                    (each with Upstream-Status)
 ├── recipes-bsp/u-boot/
 ├── wic/myboard.wks                    (A/B layout, Ch. 99)
 ├── recipes-images/images/
 │   ├── myboard-dev-image.bb
 │   └── myboard-production-image.bb    (Ch. 99 §T.2)
 └── README.md

 docs/
 ├── BRINGUP.md          what was done, in order, with dates
 ├── HARDWARE.md         pin assignments, straps, power sequence
 ├── REGISTERS.md        every register poke, and why
 ├── WORKAROUNDS.md      every erratum and hack, with a link to the datasheet
 ├── RUNBOOK.md          the ten failure modes and their diagnostics
 ├── MAINTENANCE.md      kernel version, uprev cadence, delta, owners
 └── DELTA.md            every out-of-tree patch, categorized  (Ch. 88)

 scripts/
 ├── bringup.sh          the harness from Lab 101.1
 ├── flash.sh
 ├── smoke-test.sh
 └── delta-report.sh     (Ch. 88 Lab 1)
```

**`WORKAROUNDS.md` and `DELTA.md` are the two that save the project later.** Every hack you
do not write down becomes an unexplained line of code that nobody dares touch.

### Lab 101.6 — The maintenance plan

Write the document. It is a real deliverable and a good interview artefact.

```markdown
# myboard Kernel Maintenance Plan

## Current state
- Kernel: 6.6.32 (LTS), vendor fork at <sha>
- Vendor branch EOL: 2026-12 (vendor's commitment, in writing)
- Upstream LTS EOL:  2026-12 (kernel.org)
- Product support commitment: 2035-06     <-- THE GAP IS 8.5 YEARS
- Out-of-tree delta: 247 commits, 18,400 lines

## The gap
Our product must be supported for 8.5 years beyond our kernel's EOL.
This is the central risk. Mitigation below.

## Delta breakdown            (Ch. 88 Lab 1)
| Category            | Count | Plan |
|---------------------|-------|------|
| Already upstream    |    31 | DROP at the next uprev |
| Upstreamable        |    84 | submit, 10/quarter, owner: A. Engineer |
| Blocked upstream    |    22 | tracked in issues #100-#121 |
| Product-specific    |    97 | document why; minimize |
| Vendor, unowned     |    13 | escalate to the vendor |

## Cadence
- Weekly:    rebase onto the latest v6.6.y stable. Automated; CI gates it.
- Monthly:   delta report; review new out-of-tree patches.
- Quarterly: submit 10 upstreamable patches; review the blocked list.
- Annually:  evaluate migrating to a newer LTS; test the 10-year archive.

## LTS migration trigger
Begin the move to the next LTS when EITHER:
  (a) our current LTS is within 18 months of EOL, OR
  (b) the delta exceeds 300 commits.
Budget: one engineer-quarter, based on the last migration.

## CVE process                (Ch. 88 §T.6)
We take the whole stable tree weekly. We do NOT triage per-CVE.
Exceptions are documented in CVE_STATUS with reasons.
The SBOM is regenerated per release and archived.

## Ownership
- Kernel and BSP: A. Engineer (deputy: B. Engineer)
- Build system:   C. Engineer
- Escalation:     the platform lead

## Archive                    (Ch. 99 §T.10)
Per release: sources, layer revisions, build container, artefacts,
SBOM, licence package, debug symbols. Offline rebuild tested annually.
Last verified: <date>.
```

---

## 3. Mastery drills

1. Bring up a real board you have never used before, from the vendor's EVK to a custom
   image, documenting every phase. Budget six weeks and record where the time actually went.

2. Build and use the bring-up harness of Lab 101.1. Measure the edit-to-boot cycle time
   before and after.

3. Diagnose all ten faults in Lab 101.4, introduced by a colleague, using only the serial
   console. Time each.

4. Validate DRAM properly: the vendor stress tool, across the temperature range, at both
   voltage extremes, for 24 hours. Document the margin.

5. Bring up ten peripherals in dependency order, verifying each with the Lab 101.3 loop
   before proceeding.

6. Produce the complete BSP deliverable of Lab 101.5 and have someone else use it to build
   and flash the board without your help.

7. Implement the full secure-boot chain (Ch. 90 §T.6) and prove it rejects a tampered kernel,
   a tampered DTB, and a tampered rootfs.

8. Implement A/B updates and test all six failure paths (Ch. 99 Lab 4), including power loss
   mid-update.

9. Measure and optimize boot time from power-on to the application being ready. Use a GPIO
   and a scope for the pre-kernel phases. Target under two seconds.

10. Write and execute the maintenance plan of Lab 101.6, including one full stable uprev and
    one round of upstream submissions.

11. Take a vendor BSP with a 2000-commit delta and reduce it by 25% in a quarter. Report the
    method and the result.

12. Write the bring-up runbook for your organization: the phase checklist, the diagnostic
    procedures, the ten failure modes, the deliverable list, and the maintenance template.
    Test it on the next board.

---

## 4. Further reading

**Books**
- **Chris Simmonds, *Mastering Embedded Linux Programming*, 3rd ed.** — the closest thing to
  a bring-up textbook. Chapters 3–6 and 10–11 are directly this material
- Vasquez & Simmonds, *Mastering Embedded Linux Development*
- Alberto Liberal de los Ríos, *Linux Driver Development for Embedded Processors* — the
  peripheral bring-up companion

**Kernel documentation**
- `Documentation/arch/arm64/booting.rst` and `arm/booting.rst` — **the handoff contract**
- `Documentation/devicetree/bindings/` — every binding you will write against
- `Documentation/driver-api/` — the frameworks of Part 2
- `Documentation/admin-guide/kernel-parameters.txt` — `earlycon`, `initcall_debug`

**Firmware**
- U-Boot's `doc/board/` — porting guides, and read an existing board's
- Trusted Firmware-A porting guide
- The vendor's TRM, errata sheet, and DRAM calibration documentation. **Read the errata
  sheet.** It is where the week-long mysteries are already documented

**Training**
- **Bootlin's embedded Linux and kernel training materials** — free, comprehensive, and the
  best available for this material specifically
- `elinux.org` — the Board Bring-Up and Boot Time pages
- Embedded Open Source Summit / ELCE talks on bring-up and boot time, annually

**Tools**
- `picocom`, `minicom`, `grabserial` (timestamped serial capture)
- `devmem2`, `i2c-tools`, `spidev_test`, `libgpiod` (`gpioinfo`, `gpioset`)
- `memtester`, the vendor's DDR tools
- OpenOCD, J-Link, and the SoC's serial download tool
- `dtc`, `fdtdump`, `dt-validate`

**Cross-references — this chapter uses all of them**
- Ch. 04 — QEMU, for practising without hardware
- Ch. 32–33 — device tree and ACPI
- Ch. 39–49 — every peripheral subsystem, in the bring-up order
- Ch. 90 — the boot flow and secure boot
- Ch. 91 — init and initramfs
- Ch. 94 — storage, and durability validation
- Ch. 95 — the diagnostic tools
- Ch. 96–100 — the build system
- Ch. 88 — the delta and the maintenance plan
- Ch. 102 — the hardening the production image needs
- Ch. 103 — when the answer is an RTOS instead

---

## Part 7 completion checkpoint

Twelve chapters covering everything around the kernel. Confirm you can:

- [ ] Describe the boot chain from the reset vector to `start_kernel()` on both x86 and ARM64
- [ ] Debug a silent boot using `earlycon`, `initcall_debug`, and `rdinit=/bin/sh`
- [ ] Build an initramfs by hand and explain `switch_root` versus `chroot`
- [ ] Write a hardened systemd unit and verify the sandbox actually restricts
- [ ] Explain ELF's dual view, the PLT/GOT, symbol versioning, and the libc decision
- [ ] Build a container from namespaces and cgroups, and state what is *not* isolated
- [ ] Choose a filesystem and `mkfs` parameters for a stated workload, and verify durability
- [ ] Diagnose a production problem with `perf`, ftrace, bpftrace, or a crash dump
- [ ] Write a Yocto recipe, a `.bbappend`, a class, and a machine configuration
- [ ] Produce a production image with compliance artefacts, an SBOM, and A/B updates
- [ ] Choose between Yocto, Buildroot, and the alternatives, with reasons
- [ ] Run a board bring-up from signs of life to a maintained, shipping BSP

→ Next: [../part8-specialized/102-kernel-security.md](../part8-specialized/102-kernel-security.md)

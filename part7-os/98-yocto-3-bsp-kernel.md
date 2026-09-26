# Chapter 98 — Yocto III: BSPs, Kernel Recipes, Images, Distro Config

> This is where the kernel work of Parts 0–5 meets the build system. A BSP is the artefact
> that makes a board buildable: a machine configuration, a kernel recipe with your patches and
> config, a bootloader recipe, a device tree, and the image layout that ties them together.
> Doing this well is most of what an embedded Linux platform engineer is paid for.

---

## Theory & First Principles

### T.0 — Start here: the vendor gave you a 4.19 kernel with 3,000 patches

This is the actual situation on nearly every embedded project:

```
  The SoC vendor ships:  linux-4.19.y + 3,000 out-of-tree patches
                         a U-Boot fork
                         a binary GPU blob that only builds against THEIR kernel

  You need:              security updates until 2038
                         a kernel that upstream will support
                         to not maintain 3,000 patches yourself
```

**This is Ch. 88 §T.0's divergence problem, at its worst.** Every one of those 3,000 patches
is a merge conflict waiting to happen, forever, and a vendor that stops supporting the part
leaves you holding all of it. **The single most valuable thing an embedded architect can do is
reduce that number** — by getting support upstream, which is what Part 6 was for.

**Given that reality, a BSP layer's job is to keep your board-specific content separable.**
It contains exactly four kinds of thing:

```
  meta-myboard/
   +- conf/machine/myboard.conf         ① WHAT the machine is:
   |                                       arch, KERNEL_IMAGETYPE, DTB name,
   |                                       SERIAL_CONSOLES, MACHINE_FEATURES
   +- recipes-kernel/linux/
   |    linux-myvendor_5.15.bb          ② the kernel recipe
   |    linux-myvendor/
   |      myboard.cfg                   ③ a KCONFIG FRAGMENT, not a full defconfig
   |      myboard.dts                   ④ the DEVICE TREE
   |      0001-*.patch
   +- recipes-bsp/u-boot/...            ① the bootloader
   +- recipes-graphics/...                 vendor blobs, firmware
```

**③ is the one worth understanding properly.** Do not ship a `defconfig`:

| A full `defconfig` | A `.cfg` fragment |
|---|---|
| 5,000 lines, 95% of it the default | 12 lines: exactly your deltas |
| Impossible to review or diff meaningfully | a reviewer sees precisely what you changed |
| Silently loses new upstream options on a kernel bump | new defaults are picked up automatically |
| Your intent is invisible | your intent **is** the file |

**Express the delta, not the result.** This is the same principle as a patch versus a whole
file, and as `.bbappend` versus forking a recipe (Ch. 97 §T.0). `kern-tools` merges fragments
and will warn you when an option you asked for was not actually set — which catches the
classic silent failure where a dependency was missing.

**④ The device tree is the other half, and Ch. 90 §T.0 explained why it exists:** on a
non-enumerable bus, nothing can discover that an I2C temperature sensor is at address 0x48.
So you describe it:

```dts
&i2c1 {
    status = "okay";
    clock-frequency = <100000>;

    temp@48 {
        compatible = "ti,tmp102";     /* <- the driver binds on THIS string */
        reg = <0x48>;                 /* <- the I2C address */
        interrupt-parent = <&gpio2>;
        interrupts = <11 IRQ_TYPE_LEVEL_LOW>;
    };
};
```

**Two facts about device trees that are easy to miss and cause real pain:**

1. **The device tree is an ABI.** It is a contract between the firmware and the kernel, and
   old device trees are expected to keep working with new kernels. Bindings are reviewed and
   documented in `Documentation/devicetree/bindings/` as schemas, validated by `dtbs_check`.
   **Ch. 24 §T.0's "interfaces are forever" applies here in full.**
2. **It describes hardware, not configuration.** "This SPI controller exists at this address"
   is hardware. "Log at level 7" is not, and does not belong there. The line gets blurred
   constantly, and blurring it is how a device tree becomes unmaintainable.

**And the practical loop that makes kernel work in Yocto bearable**, because the naive loop
(edit, `bitbake`, wait, flash) is unusable:

```bash
devtool modify virtual/kernel      # gives you a real git tree you can edit
# ... edit, build, test ...
devtool finish virtual/kernel meta-myboard    # turns your commits into patches
```

```bash
bitbake -c menuconfig virtual/kernel
bitbake -c diffconfig virtual/kernel      # generates a .cfg fragment from your changes
bitbake -c kernel_configcheck virtual/kernel
dtc -I fs /sys/firmware/devicetree/base   # decompile the LIVE device tree
make dtbs_check DT_SCHEMA_FILES=...       # validate against the bindings
```

---

### T.1 — What a BSP layer contains

```
 meta-myboard/
 ├── conf/
 │   ├── layer.conf
 │   └── machine/
 │       ├── myboard.conf              <- THE machine definition
 │       └── include/
 │           └── mysoc.inc             <- shared across a SoC family
 ├── recipes-kernel/
 │   └── linux/
 │       ├── linux-myboard_6.6.bb      <- or a .bbappend to linux-yocto
 │       └── linux-myboard/
 │           ├── defconfig             <- or .cfg fragments (preferred)
 │           ├── myboard.cfg
 │           ├── myboard.scc           <- kernel-yocto fragment descriptor
 │           └── 0001-driver.patch
 ├── recipes-bsp/
 │   ├── u-boot/
 │   │   ├── u-boot-myboard_2024.01.bb
 │   │   └── u-boot-myboard/
 │   │       └── myboard_defconfig
 │   ├── formfactor/
 │   └── firmware/
 │       └── myboard-firmware_1.0.bb   <- binary blobs, licensed
 ├── recipes-core/
 │   └── base-files/
 │       └── base-files_%.bbappend     <- fstab, hostname
 ├── wic/
 │   └── myboard.wks                   <- the disk image layout
 └── README                            <- how to build and flash it
```

**The discipline that makes a BSP reusable:** it contains *hardware* facts only. Distro
policy (systemd vs sysvinit, glibc vs musl) belongs in a distro layer; product content
belongs in a product layer. A BSP that installs your application is not a BSP.

### T.2 — The machine configuration

```bash
# conf/machine/myboard.conf
#@TYPE: Machine
#@NAME: My Board
#@DESCRIPTION: Machine configuration for the My Board platform

require conf/machine/include/arm/armv8a/tune-cortexa53.inc

# --- What kernel and bootloader ---
PREFERRED_PROVIDER_virtual/kernel ?= "linux-myboard"
PREFERRED_PROVIDER_virtual/bootloader ?= "u-boot-myboard"
PREFERRED_PROVIDER_u-boot ?= "u-boot-myboard"

# --- Kernel output ---
KERNEL_IMAGETYPE = "Image"
KERNEL_IMAGETYPES = "Image Image.gz"
KERNEL_DEVICETREE = "vendor/myboard.dtb vendor/myboard-rev2.dtb"
KERNEL_EXTRA_ARGS = "LOADADDR=0x40080000"

# --- U-Boot ---
UBOOT_MACHINE = "myboard_defconfig"
UBOOT_ENTRYPOINT = "0x40080000"
UBOOT_LOADADDRESS = "0x40080000"
SPL_BINARY = "spl/u-boot-spl.bin"

# --- What the hardware has ---
MACHINE_FEATURES = "usbhost usbgadget ext2 vfat rtc wifi bluetooth serial"

# --- Extra packages every image for this board needs ---
MACHINE_ESSENTIAL_EXTRA_RDEPENDS += "kernel-modules"
MACHINE_EXTRA_RRECOMMENDS += "myboard-firmware linux-firmware-brcm"

# --- Serial console ---
SERIAL_CONSOLES = "115200;ttyS0"

# --- Image output ---
IMAGE_FSTYPES ?= "wic wic.bmap tar.xz"
WKS_FILE ?= "myboard.wks"
IMAGE_BOOT_FILES ?= "Image ${@make_dtb_boot_files(d)} boot.scr"

# --- Override chain: lets recipes target the SoC family ---
MACHINEOVERRIDES =. "mysoc:"

# --- Toolchain tuning ---
DEFAULTTUNE ?= "cortexa53-crypto"
```

**`MACHINEOVERRIDES =. "mysoc:"` is the mechanism that makes SoC-family layers work.** With
it, `SRC_URI:append:mysoc = " file://soc-fix.patch"` applies to every board in the family,
and `:myboard` applies to just this one. The `=.` prepends without a space so the resulting
`OVERRIDES` chain is `...:mysoc:myboard:...` — board-specific wins over family-specific,
which is the right precedence.

**`virtual/kernel` and `virtual/bootloader`** are the indirection that lets a machine choose
its provider. A recipe `DEPENDS`ing on `virtual/kernel` gets whatever the machine selected.

### T.3 — The two kernel recipe styles

**Style A: `linux-yocto` + `.bbappend`.** Use the Yocto kernel with its `kernel-yocto`
tooling, and add your configuration and patches.

```bash
# recipes-kernel/linux/linux-yocto_%.bbappend
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"

COMPATIBLE_MACHINE:myboard = "myboard"
KERNEL_FEATURES:append:myboard = " features/myfeature/myfeature.scc"

SRC_URI:append:myboard = " \
    file://myboard.cfg \
    file://myboard-extra.scc \
    file://0001-add-myboard-driver.patch \
"
```

**Style B: your own kernel recipe.** Use when the vendor kernel is a fork that cannot
reasonably track `linux-yocto`.

```bash
# recipes-kernel/linux/linux-myboard_6.6.bb
SUMMARY = "Linux kernel for My Board"
LICENSE = "GPL-2.0-only"
LIC_FILES_CHKSUM = "file://COPYING;md5=6bc538ed5bd9a7fc9398086aedcd7e46"

inherit kernel

SRC_URI = "git://git.myvendor.com/linux.git;protocol=https;branch=myboard-6.6 \
           file://defconfig \
           file://myboard.cfg \
           file://0001-fix-dma-on-rev2.patch \
          "
SRCREV = "a1b2c3d4e5f60718293a4b5c6d7e8f9012345678"
LINUX_VERSION = "6.6.32"
LINUX_VERSION_EXTENSION = "-myboard"
PV = "${LINUX_VERSION}+git${SRCPV}"

S = "${WORKDIR}/git"

COMPATIBLE_MACHINE = "myboard"

KERNEL_DEVICETREE = "vendor/myboard.dtb"

# Let cfg fragments merge into the defconfig.
KERNEL_CONFIG_COMMAND = "oe_runmake_call -C ${S} O=${B} olddefconfig"
```

**Which to choose:**

| Use `linux-yocto` when | Use your own recipe when |
|---|---|
| The hardware is supported upstream or nearly so | The vendor has a large fork |
| You want the Yocto kernel-tooling features (`.scc`, `kernel-cache`) | The vendor tree has its own branch structure |
| You intend to track mainline | You are pinned to a vendor release |

The honest note: **most real BSPs use style B**, because the SoC vendor ships a fork
(Ch. 88 §T.1). The goal should be to shrink that fork over time, and the `Upstream-Status`
discipline from Ch. 97 §T.10 is how you track progress.

### T.4 — Kernel configuration: fragments, not defconfigs

**The anti-pattern:** check in a full `defconfig` (5000 lines). Nobody can tell what you
changed or why, upstream `defconfig` improvements are lost, and every merge is a conflict.

**The right way:** a base `defconfig` (usually the vendor's or upstream's `<board>_defconfig`)
plus small `.cfg` fragments, one per concern:

```bash
# myboard-networking.cfg
CONFIG_MYBOARD_ETH=y
CONFIG_PHY_MYVENDOR=y
CONFIG_BRIDGE=m

# myboard-security.cfg   (Ch. 102)
CONFIG_SECURITY=y
CONFIG_SECURITY_LOCKDOWN_LSM=y
CONFIG_SECURITY_LOCKDOWN_LSM_EARLY=y
CONFIG_MODULE_SIG=y
CONFIG_MODULE_SIG_FORCE=y
CONFIG_STRICT_KERNEL_RWX=y
CONFIG_STRICT_MODULE_RWX=y
CONFIG_INIT_ON_ALLOC_DEFAULT_ON=y
CONFIG_HARDENED_USERCOPY=y
CONFIG_FORTIFY_SOURCE=y
CONFIG_RANDOMIZE_BASE=y
# CONFIG_DEVMEM is not set
# CONFIG_DEBUG_FS is not set

# myboard-debug.cfg      (a separate fragment; NOT in production images)
CONFIG_DEBUG_INFO_DWARF5=y
CONFIG_DEBUG_FS=y
CONFIG_KGDB=y
CONFIG_MAGIC_SYSRQ=y
CONFIG_DYNAMIC_DEBUG=y
```

```bash
# In the recipe:
SRC_URI += " \
    file://defconfig \
    file://myboard-networking.cfg \
    file://myboard-security.cfg \
    ${@bb.utils.contains('IMAGE_FEATURES','debug-tweaks','file://myboard-debug.cfg','',d)} \
"
```

**The critical verification step:** a `.cfg` fragment can be *silently ignored* if its symbol
has unmet dependencies. Yocto's `do_kernel_configcheck` reports this, and you must treat its
warnings as errors:

```bash
bitbake -c kernel_configcheck -f virtual/kernel
# Look for:
#   [INFO]: the following symbols were not set: CONFIG_FOO
#   [INFO]: specified values did not make it into the kernel's final
#           configuration: CONFIG_BAR
#
# Cause: CONFIG_BAR depends on CONFIG_BAZ which is not set.
# This is the single most common kernel-config bug in Yocto, and it
# fails SILENTLY unless you check.
```

### T.5 — `kernel-yocto`: `.scc` files and features

`linux-yocto` uses a metadata system layered on top of config fragments.

```bash
# myboard.scc
define KMACHINE myboard
define KTYPE standard
define KARCH arm64

include ktypes/standard/standard.scc
include bsp/myvendor/mysoc.scc

branch myboard

patch 0001-add-myboard-support.patch
patch 0002-fix-dma-erratum.patch

kconf hardware myboard-hardware.cfg
kconf non-hardware myboard-policy.cfg
```

| Directive | Meaning |
|---|---|
| `define KMACHINE/KTYPE/KARCH` | Identify this BSP |
| `include` | Pull in another `.scc` |
| `branch` | Create a git branch for this BSP's patches |
| `patch` | Apply a patch |
| `kconf hardware` | A fragment describing **hardware** — drivers, SoC options |
| `kconf non-hardware` | A fragment describing **policy** — features, debug, security |

**The hardware/non-hardware split is a genuinely good idea** that most BSPs ignore: hardware
fragments are facts about the board and belong in the BSP; policy fragments are choices and
belong in a distro or image configuration. Keeping them separate means a different distro can
reuse your BSP with different policy.

`KERNEL_FEATURES` pulls in prebuilt feature fragments:

```bash
KERNEL_FEATURES:append = " \
    features/netfilter/netfilter.scc \
    features/security/security.scc \
    cfg/virtio.scc \
    cfg/fs/ext4.scc \
"
```

### T.6 — Device tree

```bash
# In the machine conf:
KERNEL_DEVICETREE = "vendor/myboard.dtb vendor/myboard-rev2.dtb"

# Overlays:
KERNEL_DEVICETREE += "overlays/myboard-camera.dtbo"
```

Adding a DT file to the kernel source tree from your layer:

```bash
# recipes-kernel/linux/linux-myboard/myboard.dts  (in your layer)
do_configure:prepend() {
    install -m 0644 ${WORKDIR}/myboard.dts ${S}/arch/arm64/boot/dts/vendor/
    # And register it in the Makefile:
    if ! grep -q myboard.dtb ${S}/arch/arm64/boot/dts/vendor/Makefile; then
        echo 'dtb-$(CONFIG_ARCH_MYSOC) += myboard.dtb' \
            >> ${S}/arch/arm64/boot/dts/vendor/Makefile
    fi
}
```

**A better approach for anything non-trivial: a patch.** Editing the source tree from a task
is fragile and invisible in `git log`. A proper patch with an `Upstream-Status` header is
reviewable and upstreamable (Ch. 88).

Validate the bindings, always:

```bash
bitbake -c compile virtual/kernel
# In devshell:
make ARCH=arm64 dt_binding_check DT_SCHEMA_FILES=vendor,myboard.yaml
make ARCH=arm64 dtbs_check
```

### T.7 — Out-of-tree kernel modules

```bash
# recipes-kernel/mymodule/mymodule_1.0.bb
SUMMARY = "Out-of-tree kernel module"
LICENSE = "GPL-2.0-only"
LIC_FILES_CHKSUM = "file://COPYING;md5=..."

inherit module

SRC_URI = "file://Makefile file://mymodule.c file://COPYING"
S = "${WORKDIR}/sources"
UNPACKDIR = "${S}"

# Autoload at boot:
KERNEL_MODULE_AUTOLOAD += "mymodule"
KERNEL_MODULE_PROBECONF += "mymodule"
module_conf_mymodule = "options mymodule debug=1"

# If the module needs a specific kernel version:
# COMPATIBLE_MACHINE = "myboard"
```

with the standard out-of-tree Makefile:

```makefile
obj-m := mymodule.o

SRC := $(shell pwd)

all:
	$(MAKE) -C $(KERNEL_SRC) M=$(SRC) modules

modules_install:
	$(MAKE) -C $(KERNEL_SRC) M=$(SRC) modules_install

clean:
	rm -f *.o *~ core .depend .*.cmd *.ko *.mod.c *.mod modules.order Module.symvers
```

`module.bbclass` sets `KERNEL_SRC`, adds the `DEPENDS` on `virtual/kernel`, handles
`modules_install` into `${D}`, and packages the result as `kernel-module-mymodule`.

**`KERNEL_MODULE_AUTOLOAD` writes `/etc/modules-load.d/`**, and `module_conf_*` writes
`/etc/modprobe.d/`. Both are what you want instead of an init script.

### T.8 — The bootloader and image layout

```bash
# recipes-bsp/u-boot/u-boot-myboard_2024.01.bb
require recipes-bsp/u-boot/u-boot-common.inc
require recipes-bsp/u-boot/u-boot.inc

DEPENDS += "bc-native dtc-native python3-setuptools-native gnutls-native"

SRC_URI = "git://source.denx.de/u-boot/u-boot.git;protocol=https;branch=master \
           file://0001-myboard-support.patch \
           file://myboard_defconfig \
          "
SRCREV = "..."
PV = "2024.01+git${SRCPV}"
S = "${WORKDIR}/git"

COMPATIBLE_MACHINE = "myboard"
UBOOT_MACHINE = "myboard_defconfig"
```

**The `wic` kickstart file** defines the partition layout:

```bash
# wic/myboard.wks
# short-description: My Board SD card layout

# The SPL must be at a byte offset the boot ROM knows about.
part --source rawcopy --sourceparams="file=u-boot-spl.bin" --no-table --align 8

# U-Boot proper at another fixed offset.
part --source rawcopy --sourceparams="file=u-boot.img" --no-table --align 512

# The boot partition: kernel, DTB, boot script. FAT so U-Boot can read it.
part /boot --source bootimg-partition --ondisk mmcblk0 --fstype=vfat \
      --label boot --active --align 8192 --size 64

# A/B root partitions for atomic updates (Ch. 99).
part / --source rootfs --ondisk mmcblk0 --fstype=ext4 --label rootA \
      --align 8192 --size 1024
part --ondisk mmcblk0 --fstype=ext4 --label rootB --align 8192 --size 1024

# Persistent data, separate so an update never touches it.
part /data --ondisk mmcblk0 --fstype=ext4 --label data --align 8192 --size 512

bootloader --ptable msdos
```

```bash
bitbake core-image-minimal            # produces .wic
wic ls tmp/deploy/images/myboard/core-image-minimal-myboard.wic
bmaptool copy core-image-minimal-myboard.wic.gz /dev/sdX   # fast, sparse-aware
```

**`bmaptool` over `dd`:** the `.bmap` file records which blocks are actually used, so writing
a 2 GB image with 300 MB of data takes 300 MB of I/O and verifies the checksum as it goes.
There is no reason to use `dd` for this.

### T.9 — The distro configuration

```bash
# conf/distro/mydistro.conf
DISTRO = "mydistro"
DISTRO_NAME = "My Product Distribution"
DISTRO_VERSION = "3.1.0"
DISTRO_CODENAME = "stable"
SDK_VENDOR = "-mydistrosdk"
MAINTAINER = "Platform Team <platform@example.com>"

# --- Base policy ---
require conf/distro/poky.conf

TCLIBC = "glibc"                       # or "musl" (Ch. 92 §T.6)
INIT_MANAGER = "systemd"               # or "sysvinit", "mdev-busybox"

# --- What gets built. THE big lever. ---
DISTRO_FEATURES = "\
    ${DISTRO_FEATURES_DEFAULT} \
    systemd usrmerge seccomp \
    ipv4 ipv6 \
"
DISTRO_FEATURES:remove = "x11 wayland opengl vulkan 3g nfc ptest"
#   Removing x11/wayland from a headless product removes hundreds of
#   packages and hours of build time.

# --- Package format ---
PACKAGE_CLASSES ?= "package_ipk"       # ipk: smallest; rpm: most featureful

# --- Security defaults (Ch. 102) ---
require conf/distro/include/security_flags.inc
DISTRO_FEATURES:append = " pam"

# --- Reproducibility and provenance (Ch. 99) ---
INHERIT += "create-spdx"
SPDX_PRETTY = "1"
INHERIT += "cve-check"
CVE_CHECK_REPORT_PATCHED = "0"

# --- Licence policy ---
INCOMPATIBLE_LICENSE = "GPL-3.0* LGPL-3.0* AGPL-3.0*"
#   Common in embedded: GPLv3's anti-tivoization clause conflicts with
#   locked bootloaders. Decide this EARLY -- it changes which packages
#   you can use (Ch. 99 §T.3).

# --- Version pinning ---
PREFERRED_VERSION_linux-myboard = "6.6%"
PREFERRED_VERSION_openssl = "3.0%"

# --- Sanity ---
SANITY_TESTED_DISTROS ?= ""
```

**`INCOMPATIBLE_LICENSE = "GPL-3.0*"` is the decision with the widest blast radius.** It
excludes bash, coreutils (GNU), gdb, and much else, forcing busybox/toybox equivalents. It is
sometimes legally necessary and it should be decided at project start, not discovered at
integration.

### T.10 — The image recipe

```bash
# recipes-images/images/myproduct-image.bb
SUMMARY = "My product image"
LICENSE = "MIT"

inherit core-image extrausers

IMAGE_FEATURES += "ssh-server-openssh"
# NOTE: no debug-tweaks. See Ch. 99.

IMAGE_INSTALL = "\
    packagegroup-core-boot \
    packagegroup-myproduct-base \
    myapp \
    ${CORE_IMAGE_EXTRA_INSTALL} \
"

IMAGE_LINGUAS = " "                    # no locales: saves ~30 MB
IMAGE_ROOTFS_SIZE = "8192"
IMAGE_ROOTFS_EXTRA_SPACE = "2048"
IMAGE_OVERHEAD_FACTOR = "1.0"          # do not over-allocate

IMAGE_FSTYPES = "wic.gz wic.bmap ext4 tar.xz"

# A real root password, set from a build variable, not a literal.
EXTRA_USERS_PARAMS = "usermod -p '${@d.getVar('ROOT_PASSWORD_HASH')}' root;"

# Post-processing.
ROOTFS_POSTPROCESS_COMMAND += "myproduct_rootfs_tweaks; "

myproduct_rootfs_tweaks() {
    # Remove anything that should not ship.
    rm -rf ${IMAGE_ROOTFS}/usr/share/doc
    rm -f  ${IMAGE_ROOTFS}/etc/ssh/ssh_host_*_key*   # regenerate on first boot

    # Make the build reproducible: no timestamps in the image.
    echo "${DISTRO_VERSION}" > ${IMAGE_ROOTFS}/etc/product-version

    # Verify no setuid binaries we did not authorize.
    find ${IMAGE_ROOTFS} -perm /6000 -type f | while read f; do
        bbwarn "setuid/setgid in image: ${f#${IMAGE_ROOTFS}}"
    done
}
```

**Package groups** are how you keep image recipes readable:

```bash
# recipes-core/packagegroups/packagegroup-myproduct-base.bb
SUMMARY = "Base packages for my product"
inherit packagegroup

PACKAGES = "${PN} ${PN}-network ${PN}-debug"

RDEPENDS:${PN} = "\
    ${PN}-network \
    util-linux \
    e2fsprogs \
    myconfig \
"
RDEPENDS:${PN}-network = "\
    systemd-networkd \
    openssh-sshd \
    iproute2 \
"
RDEPENDS:${PN}-debug = "\
    strace gdbserver tcpdump ethtool \
"
```

---

## 1. Internals

### Key classes and files

| Path | Contents |
|---|---|
| `meta/classes-recipe/kernel.bbclass` | The kernel build: tasks, packaging, `KERNEL_IMAGETYPE` |
| `meta/classes-recipe/kernel-yocto.bbclass` | `.scc` processing, `kernel-cache`, config merging |
| `meta/classes-recipe/kernel-devicetree.bbclass` | DTB building and deployment |
| `meta/classes-recipe/kernel-fitimage.bbclass` | FIT images with signatures (Ch. 90 §T.6) |
| `meta/classes-recipe/module.bbclass` | Out-of-tree modules |
| `meta/classes-recipe/image.bbclass` | Rootfs construction |
| `meta/classes-recipe/image_types.bbclass` | Every `IMAGE_FSTYPES` |
| `meta/classes-recipe/image_types_wic.bbclass` | wic |
| `meta/recipes-kernel/linux/linux-yocto*.bb` | The reference kernel recipes |
| `meta/recipes-bsp/u-boot/u-boot*.inc` | The U-Boot recipe structure |
| `scripts/lib/wic/` | The wic implementation and plugins |
| `meta/conf/machine/include/` | **The tune files** — read `arm/arch-armv8a.inc` |

### Kernel recipe tasks

```
 do_fetch / do_unpack / do_patch
 do_kernel_metadata          (kernel-yocto: process .scc files)
 do_kernel_checkout          (kernel-yocto: build the git branches)
 do_validate_branches
 do_kernel_configme          merge defconfig + fragments -> .config
 do_kernel_configcheck       <-- REPORTS IGNORED FRAGMENTS. Check it.
 do_configure                olddefconfig
 do_compile                  build the kernel image
 do_compile_kernelmodules    build modules
 do_shared_workdir           export headers for module recipes
 do_install                  into ${D}
 do_package / do_package_write_*
 do_deploy                   copy to tmp/deploy/images/<machine>/
```

Useful invocations:

```bash
bitbake -c menuconfig virtual/kernel       # interactive config
bitbake -c diffconfig virtual/kernel       # menuconfig changes -> a .cfg FRAGMENT
bitbake -c savedefconfig virtual/kernel    # minimal defconfig
bitbake -c kernel_configcheck -f virtual/kernel
bitbake -c devshell virtual/kernel         # a shell in the kernel build
bitbake -c cleansstate virtual/kernel && bitbake virtual/kernel
```

**`bitbake -c diffconfig` is the one to know:** run `menuconfig`, make changes, then
`diffconfig` writes a `.cfg` fragment containing exactly your changes. That fragment goes
into your layer. This is the correct workflow and it replaces hand-editing defconfigs.

### The tune files

```bash
# meta/conf/machine/include/arm/armv8a/tune-cortexa53.inc
DEFAULTTUNE ?= "cortexa53"
TUNEVALID[cortexa53] = "Enable Cortex-A53 specific processor optimizations"
TUNE_CCARGS .= "${@bb.utils.contains('TUNE_FEATURES', 'cortexa53', ' -mcpu=cortex-a53', '', d)}"
```

The tune system determines `-march`/`-mcpu`, the ABI, and `PACKAGE_ARCH`. Two consequences:

- **`PACKAGE_ARCH` determines sstate sharing.** Two machines with the same tune share
  compiled packages; different tunes do not. On a multi-board product, aligning tunes where
  possible is a large build-time saving.
- **`MACHINE_ARCH` packages** (kernel, machine-specific config) never share. Keep the
  machine-specific set small.

---

## 2. Practice

### Lab 98.1 — A complete BSP from scratch

```bash
#!/bin/bash
# make_bsp.sh — build a BSP layer for a QEMU "board", end to end.
set -e
cd ~/yocto

bitbake-layers create-layer meta-myboard
cd meta-myboard

# --- 1. Machine configuration ---
mkdir -p conf/machine
cat > conf/machine/myboard.conf <<'EOF'
#@TYPE: Machine
#@NAME: myboard
#@DESCRIPTION: A QEMU-based virtual board for learning

require conf/machine/include/arm/armv8a/tune-cortexa57.inc
require conf/machine/include/qemu.inc

PREFERRED_PROVIDER_virtual/kernel ?= "linux-yocto"
PREFERRED_VERSION_linux-yocto ?= "6.6%"

KERNEL_IMAGETYPE = "Image"
KERNEL_DEVICETREE = ""

MACHINE_FEATURES = "serial usbhost ext2 vfat rtc"
SERIAL_CONSOLES = "115200;ttyAMA0"

IMAGE_FSTYPES += "ext4 wic wic.bmap"
WKS_FILE ?= "myboard.wks"
IMAGE_BOOT_FILES ?= "Image"

MACHINEOVERRIDES =. "myvirtsoc:"

# QEMU runtime config
QB_SYSTEM_NAME = "qemu-system-aarch64"
QB_MACHINE = "-machine virt"
QB_CPU = "-cpu cortex-a57"
QB_SMP = "-smp 4"
QB_MEM = "-m 2048"
QB_KERNEL_CMDLINE_APPEND = "console=ttyAMA0,115200 earlycon"
QB_DEFAULT_FSTYPE = "ext4"
QB_ROOTFS_OPT = "-drive id=disk0,file=@ROOTFS@,if=none,format=raw -device virtio-blk-device,drive=disk0"
QB_SERIAL_OPT = "-device virtio-serial-device -chardev null,id=virtcon -device virtconsole,chardev=virtcon"
QB_TCPSERIAL_OPT = "-device virtio-serial-device -chardev socket,id=virtcon,port=@PORT@,host=127.0.0.1 -device virtconsole,chardev=virtcon"
EOF

# --- 2. Kernel configuration fragments ---
mkdir -p recipes-kernel/linux/linux-yocto

cat > recipes-kernel/linux/linux-yocto/myboard-hardware.cfg <<'EOF'
CONFIG_ARCH_VIRT=y
CONFIG_VIRTIO=y
CONFIG_VIRTIO_PCI=y
CONFIG_VIRTIO_BLK=y
CONFIG_VIRTIO_NET=y
CONFIG_VIRTIO_CONSOLE=y
CONFIG_SERIAL_AMBA_PL011=y
CONFIG_SERIAL_AMBA_PL011_CONSOLE=y
EOF

cat > recipes-kernel/linux/linux-yocto/myboard-security.cfg <<'EOF'
CONFIG_STRICT_KERNEL_RWX=y
CONFIG_STRICT_MODULE_RWX=y
CONFIG_RANDOMIZE_BASE=y
CONFIG_HARDENED_USERCOPY=y
CONFIG_FORTIFY_SOURCE=y
CONFIG_INIT_ON_ALLOC_DEFAULT_ON=y
CONFIG_SECCOMP=y
CONFIG_SECURITY=y
CONFIG_SECURITY_LOCKDOWN_LSM=y
# CONFIG_DEVMEM is not set
# CONFIG_DEVKMEM is not set
EOF

cat > recipes-kernel/linux/linux-yocto_%.bbappend <<'EOF'
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"

COMPATIBLE_MACHINE:myboard = "myboard"

SRC_URI:append:myboard = " \
    file://myboard-hardware.cfg \
    file://myboard-security.cfg \
"

KERNEL_FEATURES:append:myboard = " \
    features/debug/printk.scc \
"
EOF

# --- 3. A driver as an out-of-tree module ---
mkdir -p recipes-kernel/myboard-driver/files
cat > recipes-kernel/myboard-driver/files/myboard_drv.c <<'EOF'
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>
#include <linux/kernel.h>

static int debug;
module_param(debug, int, 0644);
MODULE_PARM_DESC(debug, "enable debug output");

static int __init myboard_init(void)
{
	pr_info("myboard_drv: loaded (debug=%d)\n", debug);
	return 0;
}

static void __exit myboard_exit(void)
{
	pr_info("myboard_drv: unloaded\n");
}

module_init(myboard_init);
module_exit(myboard_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("My Board platform driver");
EOF

cat > recipes-kernel/myboard-driver/files/Makefile <<'EOF'
obj-m := myboard_drv.o
SRC := $(shell pwd)
all:
	$(MAKE) -C $(KERNEL_SRC) M=$(SRC) modules
modules_install:
	$(MAKE) -C $(KERNEL_SRC) M=$(SRC) modules_install
clean:
	rm -f *.o *.ko *.mod.c *.mod modules.order Module.symvers
EOF

cat > recipes-kernel/myboard-driver/myboard-driver_1.0.bb <<'EOF'
SUMMARY = "My Board platform driver"
LICENSE = "GPL-2.0-only"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/GPL-2.0-only;md5=801f80980d171dd6425610833a22dbe6"

inherit module

SRC_URI = "file://myboard_drv.c file://Makefile"
S = "${WORKDIR}/sources"
UNPACKDIR = "${S}"

KERNEL_MODULE_AUTOLOAD += "myboard_drv"
module_conf_myboard_drv = "options myboard_drv debug=1"

COMPATIBLE_MACHINE = "myboard"
EOF

# --- 4. wic layout ---
mkdir -p wic
cat > wic/myboard.wks <<'EOF'
# short-description: myboard SD layout with A/B roots
part /boot --source bootimg-partition --ondisk vda --fstype=vfat \
      --label boot --active --align 4096 --size 32
part / --source rootfs --ondisk vda --fstype=ext4 --label rootA \
      --align 4096 --size 512
part --ondisk vda --fstype=ext4 --label rootB --align 4096 --size 512
part /data --ondisk vda --fstype=ext4 --label data --align 4096 --size 128
bootloader --ptable msdos
EOF

# --- 5. Image recipe ---
mkdir -p recipes-images/images
cat > recipes-images/images/myboard-image.bb <<'EOF'
SUMMARY = "My Board image"
LICENSE = "MIT"

inherit core-image

IMAGE_FEATURES += "ssh-server-dropbear"
IMAGE_INSTALL:append = " \
    kernel-modules \
    myboard-driver \
    util-linux \
    e2fsprogs \
"
IMAGE_LINGUAS = " "
IMAGE_ROOTFS_EXTRA_SPACE = "65536"
EOF

# --- 6. Build ---
cd ~/yocto/build
bitbake-layers add-layer ../meta-myboard
echo 'MACHINE = "myboard"' >> conf/local.conf
bitbake myboard-image

# --- 7. Run ---
runqemu myboard myboard-image nographic
#   In the target:
#     lsmod | grep myboard
#     dmesg | grep myboard
#     cat /proc/cmdline
```

### Lab 98.2 — Kernel configuration, correctly

```bash
cd ~/yocto/build

# 1. Configure interactively.
bitbake -c menuconfig virtual/kernel
#    Enable, say, CONFIG_NFS_FS and CONFIG_ROOT_NFS.

# 2. Extract ONLY your changes as a fragment. THIS is the workflow.
bitbake -c diffconfig virtual/kernel
#    -> "Config fragment has been dumped into:
#        .../fragment.cfg"
cat tmp/work/*/linux-yocto/*/fragment.cfg
cp tmp/work/*/linux-yocto/*/fragment.cfg \
   ~/yocto/meta-myboard/recipes-kernel/linux/linux-yocto/myboard-nfs.cfg

# 3. Add it to the bbappend, rebuild.
sed -i 's|file://myboard-security.cfg|&  \\\n    file://myboard-nfs.cfg|' \
    ~/yocto/meta-myboard/recipes-kernel/linux/linux-yocto_%.bbappend
bitbake -c cleansstate virtual/kernel
bitbake virtual/kernel

# 4. VERIFY IT WAS APPLIED. This is the step that catches silent failures.
bitbake -c kernel_configcheck -f virtual/kernel 2>&1 | tee /tmp/cfgcheck.txt
grep -A5 'not set\|did not make it' /tmp/cfgcheck.txt

grep -E '^CONFIG_NFS_FS|^CONFIG_ROOT_NFS' \
    tmp/work/*/linux-yocto/*/linux-*-build/.config

# 5. Deliberately add a fragment that CANNOT apply, and see what happens.
cat > ~/yocto/meta-myboard/recipes-kernel/linux/linux-yocto/bad.cfg <<'EOF'
CONFIG_SOME_X86_ONLY_THING=y
EOF
# Add it, rebuild, and observe:
#   do_kernel_configcheck reports it was not set.
#   The build SUCCEEDS. This is why you must check.
```

### Lab 98.3 — Custom distro

```bash
cd ~/yocto
mkdir -p meta-mydistro/conf/distro
cat > meta-mydistro/conf/layer.conf <<'EOF'
BBPATH .= ":${LAYERDIR}"
BBFILES += "${LAYERDIR}/recipes-*/*/*.bb ${LAYERDIR}/recipes-*/*/*.bbappend"
BBFILE_COLLECTIONS += "mydistro"
BBFILE_PATTERN_mydistro = "^${LAYERDIR}/"
BBFILE_PRIORITY_mydistro = "10"
LAYERDEPENDS_mydistro = "core"
LAYERSERIES_COMPAT_mydistro = "scarthgap"
EOF

cat > meta-mydistro/conf/distro/mydistro.conf <<'EOF'
DISTRO = "mydistro"
DISTRO_NAME = "My Product Distro"
DISTRO_VERSION = "1.0"
MAINTAINER = "you@example.com"

require conf/distro/poky.conf

TCLIBC = "glibc"
INIT_MANAGER = "systemd"

DISTRO_FEATURES:remove = "x11 wayland opengl vulkan 3g nfc bluetooth ptest"
DISTRO_FEATURES:append = " seccomp usrmerge"

PACKAGE_CLASSES = "package_ipk"

# Security (Ch. 102)
require conf/distro/include/security_flags.inc

# Compliance (Ch. 99)
INHERIT += "create-spdx cve-check"
INCOMPATIBLE_LICENSE = "GPL-3.0* LGPL-3.0* AGPL-3.0*"

# Reproducibility
BUILD_REPRODUCIBLE_BINARIES = "1"
SOURCE_DATE_EPOCH = "1700000000"
EOF

cd ~/yocto/build
bitbake-layers add-layer ../meta-mydistro
sed -i 's/^DISTRO ?*=.*/DISTRO = "mydistro"/' conf/local.conf

# Measure the effect of the DISTRO_FEATURES removal:
bitbake -g myboard-image && wc -l pn-buildlist   # before
# (compare with the poky build's count)
time bitbake myboard-image

# And the licence exclusion:
bitbake bash 2>&1 | tail -5
#   -> "ERROR: Nothing PROVIDES 'bash' ... was skipped: because it has
#      incompatible license(s): GPL-3.0-or-later"
#   This is INCOMPATIBLE_LICENSE working. Now you need busybox's ash.
```

### Lab 98.4 — Patch the kernel properly

```bash
cd ~/yocto/build

# --- The devtool way (correct) ---
devtool modify virtual/kernel
cd workspace/sources/linux-yocto

# Make a real change.
cat >> drivers/misc/Kconfig <<'EOF'

config MYBOARD_MISC
	tristate "My Board misc driver"
	help
	  A driver for My Board's miscellaneous hardware.
EOF

git add -A
git commit -s -m "misc: add MYBOARD_MISC Kconfig entry

Our board has a miscellaneous control block that needs its own driver.
Add the Kconfig entry; the driver itself follows.

Upstream-Status: Pending
"

cd ~/yocto/build
devtool build virtual/kernel
devtool finish virtual/kernel ~/yocto/meta-myboard

ls ~/yocto/meta-myboard/recipes-kernel/linux/linux-yocto/
cat ~/yocto/meta-myboard/recipes-kernel/linux/linux-yocto_%.bbappend

# --- Verify the Upstream-Status discipline (Ch. 88 / Ch. 97 §T.10) ---
cd ~/yocto/meta-myboard
grep -L 'Upstream-Status' $(find . -name '*.patch')
#   Any patch listed here is missing the header. Fix it.

for p in $(find . -name '*.patch'); do
    printf '%-60s %s\n' "$p" \
        "$(grep -m1 'Upstream-Status' $p || echo 'MISSING')"
done
```

### Lab 98.5 — Build for two boards, share the cache

```bash
cd ~/yocto/build

# 1. Add a second machine that differs only in configuration.
cp ~/yocto/meta-myboard/conf/machine/myboard.conf \
   ~/yocto/meta-myboard/conf/machine/myboard-lite.conf
sed -i 's/cortexa57/cortexa53/' ~/yocto/meta-myboard/conf/machine/myboard-lite.conf

# 2. Build both.
MACHINE=myboard      bitbake myboard-image
MACHINE=myboard-lite bitbake myboard-image

# 3. What was shared, and what was rebuilt?
#    Look at PACKAGE_ARCH: recipes with PACKAGE_ARCH=${TUNE_PKGARCH}
#    are shared IF the tunes match; MACHINE_ARCH recipes never are.
bitbake -e -e virtual/kernel | grep '^PACKAGE_ARCH='
bitbake -e zlib | grep '^PACKAGE_ARCH='

ls tmp/deploy/ipk/
#   -> all/  cortexa57/  cortexa53/  myboard/  myboard_lite/
#      "all" and the tune dirs are shared; machine dirs are not.

# 4. Now make the tunes MATCH and re-measure.
sed -i 's/cortexa53/cortexa57/' ~/yocto/meta-myboard/conf/machine/myboard-lite.conf
time (MACHINE=myboard-lite bitbake myboard-image)
#   Much faster: everything tune-specific is now shared.
#   THIS is why aligning tunes across a product family matters.
```

### Lab 98.6 — Produce and flash a real image

```bash
cd ~/yocto/build
bitbake myboard-image

cd tmp/deploy/images/myboard/
ls -lh

# Inspect the wic image without mounting it:
wic ls myboard-image-myboard.wic
wic ls myboard-image-myboard.wic:1     # the boot partition
wic cp myboard-image-myboard.wic:1/Image /tmp/     # extract a file

# Write it, properly:
bmaptool copy myboard-image-myboard.wic.gz /dev/sdX
#   bmaptool skips unallocated blocks and verifies the checksum.
#   Never use dd for a Yocto image if a .bmap exists.

# Verify the partition layout matches the .wks:
sudo fdisk -l /dev/sdX
sudo blkid /dev/sdX*

# For QEMU:
runqemu myboard myboard-image wic nographic
```

---

## 3. Mastery drills

1. Build a complete BSP for a real board you own (Raspberry Pi, BeagleBone, a vendor EVK) and
   boot a custom image on it. Do not use the vendor's prebuilt layer.

2. Convert a checked-in 5000-line `defconfig` into a base defconfig plus themed fragments.
   Verify every fragment applies with `kernel_configcheck`.

3. Write an out-of-tree module recipe with autoload, module parameters, and a `modprobe.d`
   configuration. Verify all three on the target.

4. Create a distro layer that removes `x11`, `wayland`, and `opengl`, and measure the effect
   on build time, package count, and image size.

5. Set up `INCOMPATIBLE_LICENSE = "GPL-3.0*"` and resolve every resulting build failure.
   Document the substitutions and what functionality was lost.

6. Design and implement a `.wks` with A/B root partitions, a shared data partition, and a
   bootloader environment partition. Test switching between A and B.

7. Build one image for three machines that share a SoC. Use `MACHINEOVERRIDES` to share
   everything possible and measure the sstate reuse.

8. Enable FIT image signing (`kernel-fitimage.bbclass`), sign a kernel, and verify U-Boot
   rejects a tampered image (Ch. 90 Lab 4).

9. Audit a vendor BSP: count copied-vs-appended recipes, patches without `Upstream-Status`,
   and machine-specific packages that need not be. Write the remediation plan.

10. Implement a kernel recipe that tracks a vendor git branch, pins `SRCREV`, and has a
    documented procedure for uprevving. Perform one uprev and record the effort.

11. Build the same image with `linux-yocto` and with a custom kernel recipe from the same
    source. Compare the workflow, and state which you would use for which situation.

12. Write the BSP maintenance plan for a 10-year product: kernel version strategy, uprev
    cadence, patch-queue management, and the criteria for moving to a newer LTS (Ch. 88).

---

## 4. Further reading

**Documentation**
- **Yocto Project BSP Developer's Guide** — the authoritative BSP document
- **Yocto Project Linux Kernel Development Manual** — `.scc` files, `kernel-yocto`, patching
- Yocto Project Reference Manual — the `KERNEL_*`, `MACHINE_*`, and `IMAGE_*` variables
- `meta/classes-recipe/kernel.bbclass` and `kernel-yocto.bbclass` — read both
- `scripts/lib/wic/` — the wic plugins, if you need a custom partition source

**Related kernel material**
- Ch. 03 — kbuild and Kconfig, which this automates
- Ch. 32 — device tree
- Ch. 90 — the boot flow the BSP must satisfy
- Ch. 101 — board bring-up, which this chapter's artefacts are the output of

**Reference BSPs to read**
- `meta-yocto-bsp` — the Yocto reference BSPs; small and clean
- `meta-raspberrypi` — well-maintained, readable, and a good model for a community BSP
- `meta-intel`, `meta-ti`, `meta-freescale` — large vendor BSPs; read them critically, noting
  how much they copy rather than append
- `meta-arm` — TF-A, OP-TEE, and the ARM reference platforms

**Books and training**
- Chris Simmonds, *Mastering Embedded Linux Programming*, ch. 3–6
- Bootlin's Yocto and embedded Linux training materials — free
- Rudolf Streif, *Embedded Linux Systems with the Yocto Project*, the BSP chapters

**Cross-references**
- Ch. 96–97 — the concepts and the language
- Ch. 99 — SDK, licence compliance, CVE, SBOM, OTA, reproducibility
- Ch. 88 — the out-of-tree delta this chapter's patches constitute
- Ch. 102 — the security configuration the `.cfg` fragments encode

→ Next: [99-yocto-4-production.md](99-yocto-4-production.md)

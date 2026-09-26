# Chapter 100 — Buildroot, OpenWrt, and the Alternatives

> Yocto is not the only answer and is frequently the wrong one. This chapter covers the
> alternatives, what each is genuinely good at, and — most importantly — how to choose. The
> choice is an architectural decision with a ten-year horizon, and getting it wrong is
> expensive in a way that is not obvious for the first two years.

---

## Theory & First Principles

### T.0 — Start here: Yocto is often the wrong answer

Ch. 96–99 argued for Yocto. **Now argue against it**, because an architect who only knows one
tool will use it everywhere, and that is how projects die.

```
   Buildroot, from zero to a booting image:

     make raspberrypi4_defconfig
     make                        # ~30 minutes
     # output/images/sdcard.img  -- done.

   Yocto, from zero to a booting image:

     ~4 hours, ~100 GB of disk, and roughly a week of learning
     before the error messages mean anything.
```

**For a single-board product with a small team and a two-year life, Buildroot is very often
the correct engineering decision.** Saying so is not heresy; it is Ch. 89 §T.0's discipline of
matching the mechanism to the actual requirement.

**The real axis of comparison is not "which is better" but *what kind of complexity you are
buying*:**

| | **Buildroot** | **Yocto / OE** |
|---|---|---|
| Mental model | **a big Makefile** — you can read it | a task graph with signature-based caching |
| Output | a single image | **packages** (rpm/deb/ipk) plus images |
| Incremental rebuild | **weak** — changing config often means a full rebuild | strong — sstate reuses everything unaffected |
| Multiple products from one source | painful | **its core competence** — layers and machines |
| Runtime package management | essentially no | yes |
| Learning curve | days | **months** |
| License manifest / SBOM | basic | **thorough** |
| Community BSP availability | good | **excellent** — most SoC vendors ship `meta-` layers |

**The decisive question is almost always this one:**

> **How many products, how many variants, and for how long?**

One board, one variant, two years, three engineers → **Buildroot**. Five boards sharing 80% of
a platform, shipped for a decade, with a compliance requirement and a vendor BSP → **Yocto**,
and the cost of learning it is amortized over all of that.

**And there are more than two options**, each of which is a different answer to "what is an
image?":

| | Its distinguishing idea |
|---|---|
| **Debian / Ubuntu Core + `debootstrap`** | do not build anything — use a distribution and its security team |
| **Nix / NixOS** | **content-addressed, purely functional** builds — the strongest reproducibility story of any of them, and the strangest to learn |
| **Alpine / `apk`** | musl + busybox; tiny, and the default substrate for containers |
| **OSTree / `rpm-ostree`** | **atomic, git-like image commits with rollback** — the A/B idea of Ch. 99 §T.0 generalized to a whole OS |
| **Containers on a minimal host** | move the application's dependency problem out of the firmware entirely |

**That last row is the genuinely modern shift, and it is worth taking seriously.** If the
device is powerful enough, you can ship a small, rarely-changing base OS (updated A/B) and
deliver the application as a container updated on its own schedule. **You have decoupled two
things that were fused** — the OS lifecycle and the application lifecycle — which is the same
move as separating mechanism from policy, applied to release engineering. The cost is a
runtime, more RAM and flash, and a second update path to secure.

**The meta-lesson, and it is the reason this chapter exists at all:**

> **Every build system is a position on one trade: how much up-front complexity will you
> accept in exchange for how much long-term control?** Buildroot takes almost none and gives
> you little leverage. Nix takes an enormous amount and gives you the strongest guarantees.
> Yocto sits between, and is the industry default for long-lived embedded products because
> that is where most such products actually sit.

**Being able to argue this both ways, with the requirement in hand, is the architect-level
skill — not knowing Yocto.**

```bash
make list-defconfigs | head -20            # Buildroot
make menuconfig && make graph-depends
nix-shell -p hello --run hello             # Nix, if installed
rpm-ostree status                          # atomic image commits, on a Fedora host
```

---

### T.1 — The landscape

| System | Model | Sweet spot |
|---|---|---|
| **Yocto / OpenEmbedded** | Recipe-based, package-producing, layered | Product lines, long lifetimes, compliance |
| **Buildroot** | Kconfig-driven, builds a rootfs image directly | Single small product, fast to learn |
| **OpenWrt** | Buildroot-derived, networking-specialized | Routers, networking devices |
| **PTXdist** | Kconfig-driven, similar to Buildroot | European industrial; strong on config management |
| **Debian/Ubuntu rootfs** (`debootstrap`, `mmdebstrap`) | Assemble from binary packages | Standard hardware, size not critical |
| **Alpine Linux** | musl + apk, minimal | Containers, small systems |
| **Container-based** (balena, torizon) | An OS that runs containers | Fleet-managed application devices |
| **Android / AOSP** | A complete platform | Consumer devices needing the Android app ecosystem |
| **NixOS / Nix** | Purely functional package management | Reproducibility purists; growing embedded interest |
| **Bespoke scripts** | A shell script and a cross-toolchain | Almost never right; extremely common |

The last row deserves naming because it is what many teams actually have: a `build.sh` that
calls `make` in fifteen directories. It works until the person who wrote it leaves.

### T.2 — Buildroot: the design

**The model:** `make menuconfig` selects packages and options; `make` produces a rootfs
image. There is no package manager, no packages, and no on-target installation. The output is
an image.

```
 buildroot/
 ├── configs/             defconfigs, one per board
 ├── package/             ~3000 packages, each a Config.in + .mk
 ├── board/               per-board files: kernel config, rootfs overlay, post-scripts
 ├── toolchain/           internal toolchain build, or external toolchain support
 ├── linux/               the kernel package
 ├── fs/                  filesystem image generators
 ├── output/              everything generated
 │   ├── build/           per-package build directories
 │   ├── host/            the host toolchain and tools
 │   ├── target/          the rootfs being assembled
 │   ├── staging/         symlink into host/<tuple>/sysroot
 │   └── images/          THE OUTPUT
 └── .config             your configuration
```

A Buildroot package is two small files:

```makefile
# package/mypackage/Config.in
config BR2_PACKAGE_MYPACKAGE
	bool "mypackage"
	depends on BR2_USE_MMU
	select BR2_PACKAGE_ZLIB
	help
	  A description of mypackage.

	  https://example.com/mypackage
```

```makefile
# package/mypackage/mypackage.mk
MYPACKAGE_VERSION = 1.2.3
MYPACKAGE_SITE = https://example.com/releases
MYPACKAGE_SOURCE = mypackage-$(MYPACKAGE_VERSION).tar.xz
MYPACKAGE_LICENSE = GPL-2.0+
MYPACKAGE_LICENSE_FILES = COPYING
MYPACKAGE_DEPENDENCIES = zlib
MYPACKAGE_INSTALL_STAGING = YES

MYPACKAGE_CONF_OPTS = --enable-foo --disable-bar

define MYPACKAGE_INSTALL_INIT_SYSTEMD
	$(INSTALL) -D -m 0644 $(MYPACKAGE_PKGDIR)/mypackage.service \
		$(TARGET_DIR)/usr/lib/systemd/system/mypackage.service
endef

$(eval $(autotools-package))
```

**Compare with the Yocto equivalent (Ch. 97 Lab 1a).** Buildroot's is shorter and more
obvious; Yocto's carries licence checksums, packaging splits, and `PACKAGECONFIG`. That
difference is the whole trade in miniature.

### T.3 — Buildroot's strengths and weaknesses, honestly

**Strengths:**

| | |
|---|---|
| **Learning curve** | A day, versus weeks for Yocto |
| **Debuggability** | It is Make. You can read it, and `make -p` shows you everything |
| **Build time** | A clean build is 30–60 minutes, versus 2–8 hours |
| **Simplicity** | No sstate, no signatures, no overrides, no layers |
| **Size** | Minimal images are genuinely tiny — 2 MB is achievable |
| **Kconfig** | Familiar to anyone who has configured a kernel |

**Weaknesses, and these are the ones that bite at scale:**

| | |
|---|---|
| **Incremental rebuilds** | **The big one.** Changing a package's configuration usually requires `make clean` and a full rebuild. There is no sstate |
| **No packages** | You cannot install software on the target, and you cannot do a partial update |
| **Multi-product** | Sharing between board configurations is awkward — `BR2_EXTERNAL` helps but is not layers |
| **Licence tracking** | `make legal-info` exists and is basic compared to Yocto's |
| **SBOM** | Partial; CycloneDX support is recent |
| **Multi-libc/multi-arch in one build** | Not really |

**The incremental-rebuild issue deserves emphasis** because it is the one people underestimate.
Buildroot's documentation says explicitly that after changing configuration you should do a
full rebuild, because it cannot reliably track what a change affects. On a 45-minute build
that is annoying; on a team of ten it is hours of aggregate waiting every day. Yocto's sstate
exists precisely for this and is the main reason large projects use it.

### T.4 — OpenWrt

Buildroot-derived, but it diverged early and is now its own thing, specialized for network
devices.

**What it adds:**

| Feature | Detail |
|---|---|
| **`opkg`** | A real package manager, so you *can* install on-target |
| **UCI** | Unified Configuration Interface — a single, consistent configuration model |
| **LuCI** | A web UI, which routers need |
| **`procd`** | Its own init and process manager |
| **`netifd`** | Network interface daemon, config-driven |
| **`ubus`** | A lightweight IPC bus |
| **OverlayFS root** | A read-only squashfs base plus a JFFS2 overlay, so "factory reset" is `rm -rf` of the overlay |
| **Huge device database** | Thousands of supported routers |

**The overlay-root design is genuinely clever and worth stealing:** the base system is an
immutable squashfs, and all changes go to a writable overlay. Benefits: the base is verifiable
(Ch. 102's dm-verity idea, achieved differently), factory reset is trivial, flash wear is
confined to the overlay, and an upgrade replaces the squashfs without touching configuration.

**UCI is the other idea worth stealing.** One configuration format, one command-line tool, one
API, for every subsystem:

```bash
uci show network
uci set network.lan.ipaddr='192.168.2.1'
uci commit network
/etc/init.d/network restart
```

versus the usual Linux situation of a different config format per daemon. For a product with
a management UI, having exactly one configuration model is a large simplification.

**Use OpenWrt when** the product is a network device, or when you want its device support and
its configuration model. **Do not use it** for a general-purpose embedded product — it is
opinionated in ways that fight you.

### T.5 — Debian-based rootfs

```bash
# The classic
sudo debootstrap --arch=arm64 --foreign bookworm rootfs \
	http://deb.debian.org/debian
sudo cp /usr/bin/qemu-aarch64-static rootfs/usr/bin/
sudo chroot rootfs /debootstrap/debootstrap --second-stage

# The modern, better tool -- no root required, more reproducible
mmdebstrap --architectures=arm64 --variant=minbase \
	--include=systemd-sysv,openssh-server \
	bookworm rootfs.tar http://deb.debian.org/debian

# Declarative, with customization:
#   - debos      (Debian's image builder, YAML-driven)
#   - elbe       (E.L.B.E., XML-driven, strong on reproducibility)
#   - isar       (Debian packages built with a BitBake-like flow)
```

**When this is right:**
- Standard hardware (x86, a well-supported ARM SBC).
- Storage is measured in gigabytes, not megabytes.
- You want a huge package ecosystem and apt.
- You want Debian's security updates without maintaining the update infrastructure.
- Your team knows Debian and does not know Yocto.

**When it is wrong:**
- You need a custom kernel and bootloader for custom hardware (possible, but you are now
  maintaining a Debian derivative).
- Size matters — a minimal Debian rootfs is ~120 MB versus 2–40 MB.
- You need reproducibility or a precise SBOM.
- You need to remove GPLv3 (Ch. 99 §T.3) — not feasible.

**`isar` is the interesting hybrid**: BitBake-driven, but it builds *Debian packages* and
assembles a Debian rootfs. You get Debian's security updates and package ecosystem with
Yocto's build reproducibility and layer model. Worth knowing about; used in automotive.

### T.6 — Container-based and application-centric

A genuinely different model: the OS is a thin, immutable, auto-updating base, and the
application ships as a container.

| Platform | Model |
|---|---|
| **balenaOS** | Yocto-built minimal OS + balenaEngine (a Docker fork) + a fleet-management cloud |
| **Torizon** (Toradex) | Yocto-built OSTree base + Docker + Uptane-based updates |
| **Fedora IoT / RHEL for Edge** | OSTree-based immutable OS + Podman |
| **Ubuntu Core** | Snap-based; everything including the kernel is a snap |

**The argument:** application developers do not need embedded Linux expertise. They build a
container; the platform handles the OS, updates, and fleet management. Rollback is per-app.
Development uses the same tooling as the cloud.

**The costs**, which are real:
- **Overhead**: a container runtime plus image storage. Not viable under ~1 GB of storage and
  512 MB of RAM.
- **Hardware access** is awkward — device passthrough, privileged containers, and you have
  reintroduced the isolation questions of Ch. 93.
- **Real-time** is hard through a container runtime (Ch. 103).
- **Vendor lock-in** to the fleet-management platform, which is the commercial model.
- **Boot time** — a container runtime plus image start is slower than a native process.

**Use it when** the device is effectively a small server, the team is cloud-native, and fleet
management is a large part of the problem. **Do not** when resources are constrained, latency
matters, or hardware access is central.

### T.7 — The decision framework

Ask these, in order. The first one that gives a clear answer usually decides it.

**1. What are the resource constraints?**
- < 32 MB flash / < 64 MB RAM → Buildroot, or a bespoke minimal system.
- 32–512 MB → Buildroot or Yocto.
- > 512 MB with a general-purpose CPU → anything, including Debian.
- > 1 GB storage, cloud-adjacent team → consider container-based.

**2. How many products, over how long?**
- One product, 2–3 years → Buildroot.
- A product *line*, 5–10 years → **Yocto.** The layer model is the reason it exists.
- One product, 10+ years → Yocto, for the archival and reproducibility story (Ch. 99 §T.10).

**3. What are the compliance requirements?**
- Formal SBOM, licence audit, CRA compliance → Yocto.
- Best-effort → Buildroot's `legal-info` is adequate.

**4. Do you need on-target package management or partial updates?**
- Yes → Yocto (rpm/deb/ipk), OpenWrt (opkg), or Debian (apt).
- No, full-image A/B updates → Buildroot is fine.

**5. What does the team know, and who will maintain it in year five?**
- This is the criterion people weight too low. A Yocto build nobody on the team understands
  is worse than a Buildroot build everybody does. **But** a Buildroot build that has
  outgrown Buildroot is worse than either.

**6. What does the silicon vendor support?**
- In practice this often decides it. If the vendor ships a `meta-vendor` layer with a
  maintained BSP, using anything else means porting the BSP yourself. Weigh that cost
  honestly — it is usually months.

**The summary, and a defensible interview answer:**

> Buildroot for a single constrained product with a short life and a small team. Yocto for a
> product line, a long lifetime, or anything with compliance obligations. Debian when the
> hardware is standard and size does not matter. OpenWrt if it is a router. And the silicon
> vendor's choice usually wins regardless, so factor the porting cost before overriding it.

### T.8 — Migration

Migrations happen — usually Buildroot → Yocto when a product line grows, occasionally the
reverse when a Yocto build has become an unmaintainable pile.

**Buildroot → Yocto**, which is the common direction:

| Buildroot artefact | Yocto equivalent |
|---|---|
| `configs/myboard_defconfig` | `conf/machine/myboard.conf` + a distro conf |
| `board/mycompany/myboard/` | `meta-myboard/` |
| `package/mypackage/` | `recipes-*/mypackage/mypackage_x.y.bb` |
| Rootfs overlay | a recipe installing the files, or `IMAGE_ROOTFS_EXTRA` |
| `post-build.sh` | `ROOTFS_POSTPROCESS_COMMAND` |
| `post-image.sh` | `IMAGE_CMD_*` or a `.wks` |
| Kernel fragment | a `.cfg` in the kernel `.bbappend` |
| `BR2_EXTERNAL` tree | a layer |

**The honest estimate: 2–6 weeks for a non-trivial product**, dominated by learning the model
rather than by the mechanical conversion. Do it incrementally: get a minimal image booting
first, then add packages, then the BSP, then production concerns.

**The migration that is usually wrong** is Yocto → Buildroot. If a Yocto build is painful, the
problem is almost always a layer that copies rather than appends (Ch. 96 §T.3) and an
undisciplined patch queue (Ch. 88 §T.10). Moving to Buildroot does not fix either and loses
sstate.

### T.9 — The thing every system has in common

Whichever you choose, you are solving the same problems, and the architectural decisions
transfer:

| Problem | Where it appears |
|---|---|
| Cross-compilation and sysroots | Ch. 92 §T.10 |
| libc choice | Ch. 92 §T.6 |
| Kernel configuration and patches | Ch. 98, Ch. 88 |
| Bootloader and boot flow | Ch. 90 |
| Init system | Ch. 91 |
| Filesystem layout and durability | Ch. 94 |
| Licence compliance | Ch. 99 §T.3 |
| Update mechanism | Ch. 99 §T.8 |
| Security hardening | Ch. 102 |
| The out-of-tree delta | Ch. 88 §T.10 |

**The build system is the least important of these.** It is a tool for expressing decisions
you would have to make anyway. Teams that agonize over Yocto-versus-Buildroot and then ship a
`debug-tweaks` image with no update mechanism have optimized the wrong variable.

### T.10 — The alternatives worth knowing about

**PTXdist** — Pengutronix's Kconfig-driven system. Similar to Buildroot in model, stronger on
configuration management and on a strict separation between platform and project
configuration. Widely used in German industrial automation. Worth knowing because you will
meet it.

**NixOS / Nix** — purely functional package management: every build is a pure function of its
inputs, producing a content-addressed output. This gives reproducibility and atomic
rollback *by construction* rather than by discipline. Embedded use is growing
(`nixos-hardware`, cross-compilation support) but the learning curve is steeper than Yocto's
and the embedded ecosystem is thin. Watch it; do not bet a product on it yet.

**Android / AOSP** — if you need the Android application ecosystem, this is the answer and
nothing else is. It is a complete platform with its own build system (Soong/Blueprint), its
own libc (Bionic), its own init, its own HAL model, and its own update mechanism. Enormous
and opinionated. The GKI work (Ch. 88 §T.8) is the most instructive part for non-Android
engineers.

**Zephyr / FreeRTOS** — not Linux at all. If the device has 256 KB of RAM, Linux is the wrong
answer and an RTOS is the right one. Knowing when to say "this should not run Linux" is a
real architectural skill, and the AMP pattern (Linux on the application core, an RTOS on a
real-time core, communicating via `rpmsg` — Ch. 49, Ch. 103 §T.10) is extremely common in
modern SoCs.

---

## 1. Internals

### Buildroot mechanics

```bash
# --- Setup ---
git clone https://git.buildroot.net/buildroot
cd buildroot
git checkout 2024.02.x           # the LTS branches are .x

# --- Configure ---
make list-defconfigs
make raspberrypi4_64_defconfig
make menuconfig
make linux-menuconfig            # kernel config
make busybox-menuconfig
make uclibc-menuconfig

# --- Build ---
make -j$(nproc)                  # NOTE: top-level make is NOT parallel-safe
                                 #   in older versions; per-package builds are

# --- Per-package ---
make mypackage                   # build one package
make mypackage-rebuild           # force a rebuild of it
make mypackage-reconfigure       # re-run configure
make mypackage-dirclean          # wipe its build dir
make mypackage-source            # just fetch
make mypackage-show-depends
make mypackage-show-recursive-depends

# --- Inspect ---
make show-info                   # JSON of every enabled package
make graph-depends               # a dependency graph
make graph-build                 # build time per package
make graph-size                  # ROOTFS SIZE per package  <- very useful
make printvars VARS='BR2_%'
make legal-info                  # licence report + source archive

# --- The output ---
ls output/images/                # kernel, dtb, rootfs, sdcard.img
ls output/target/                # the rootfs tree (NOT directly usable --
                                 #   permissions are applied at image time)
ls output/host/                  # the toolchain and host tools

# --- Save your configuration ---
make savedefconfig               # minimal defconfig -> configs/
BR2_DEFCONFIG=$(pwd)/configs/myboard_defconfig make savedefconfig
```

**`make graph-size` is the underused one.** It produces a pie chart of rootfs size by package,
which immediately shows where your 40 MB went.

### `BR2_EXTERNAL`: Buildroot's layering

```
 my-external/
 ├── external.desc           name and description
 ├── external.mk             include the package .mk files
 ├── Config.in               include the package Config.in files
 ├── configs/
 │   └── myboard_defconfig
 ├── board/
 │   └── myboard/
 │       ├── linux.config
 │       ├── linux-fragment.cfg
 │       ├── rootfs_overlay/
 │       │   └── etc/init.d/S99myapp
 │       ├── post-build.sh
 │       ├── post-image.sh
 │       └── genimage.cfg
 └── package/
     └── myapp/
         ├── Config.in
         └── myapp.mk
```

```bash
# external.desc
name: MYCOMPANY
desc: My Company's Buildroot customizations

# external.mk
include $(sort $(wildcard $(BR2_EXTERNAL_MYCOMPANY_PATH)/package/*/*.mk))

# Config.in
source "$BR2_EXTERNAL_MYCOMPANY_PATH/package/myapp/Config.in"

# Use it:
make BR2_EXTERNAL=/path/to/my-external myboard_defconfig
make BR2_EXTERNAL=/path/to/my-external
```

Multiple `BR2_EXTERNAL` trees can be combined with a colon separator — this is as close as
Buildroot gets to Yocto's layers, and the difference is that there is no priority mechanism
and no `.bbappend` equivalent. To modify an upstream package you patch it in
`package/<name>/*.patch` from your external tree, which works, or you override the `.mk`,
which does not compose.

### OpenWrt mechanics

```bash
git clone https://git.openwrt.org/openwrt/openwrt.git
cd openwrt
./scripts/feeds update -a
./scripts/feeds install -a

make menuconfig                  # target, profile (device), packages
make -j$(nproc)

ls bin/targets/<target>/<subtarget>/
#   *-squashfs-sysupgrade.bin    for upgrading an existing device
#   *-squashfs-factory.bin       for flashing from the vendor firmware

# Per-package
make package/mypackage/compile V=s
make package/mypackage/clean

# The Image Builder: assemble an image from prebuilt packages, in seconds
make image PROFILE=<device> PACKAGES="luci wireguard-tools -ppp"
```

An OpenWrt package Makefile:

```makefile
include $(TOPDIR)/rules.mk

PKG_NAME:=myapp
PKG_VERSION:=1.0
PKG_RELEASE:=1
PKG_LICENSE:=GPL-2.0-only
PKG_LICENSE_FILES:=COPYING

include $(INCLUDE_DIR)/package.mk

define Package/myapp
  SECTION:=utils
  CATEGORY:=Utilities
  TITLE:=My application
  DEPENDS:=+libubox +libubus
endef

define Package/myapp/install
	$(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/myapp $(1)/usr/bin/
	$(INSTALL_DIR) $(1)/etc/init.d
	$(INSTALL_BIN) ./files/myapp.init $(1)/etc/init.d/myapp
	$(INSTALL_DIR) $(1)/etc/config
	$(INSTALL_CONF) ./files/myapp.config $(1)/etc/config/myapp
endef

$(eval $(call BuildPackage,myapp))
```

---

## 2. Practice

### Lab 100.1 — Build the same product three ways

The lab that actually answers the question.

```bash
#!/bin/bash
# three_ways.sh — same requirements, three build systems, measured.
#
# Requirements: boot on qemu-aarch64, systemd or equivalent init,
# ssh server, a small custom application, and a working network.

mkdir -p ~/compare && cd ~/compare

# ============ 1. BUILDROOT ============
git clone --depth 1 -b 2024.02.x https://git.buildroot.net/buildroot
cd buildroot
make qemu_aarch64_virt_defconfig
cat >> .config <<'EOF'
BR2_PACKAGE_DROPBEAR=y
BR2_PACKAGE_UTIL_LINUX=y
BR2_TARGET_ROOTFS_EXT2=y
BR2_TARGET_ROOTFS_EXT2_4=y
EOF
make olddefconfig
time make -j$(nproc) 2>&1 | tail -5
echo "BUILDROOT rootfs: $(du -sh output/images/rootfs.ext2 | cut -f1)"
echo "BUILDROOT images: $(du -sh output/images | cut -f1)"
echo "BUILDROOT total:  $(du -sh output | cut -f1)"
cd ..

# ============ 2. YOCTO ============
git clone --depth 1 -b scarthgap git://git.yoctoproject.org/poky
cd poky && source oe-init-build-env ~/compare/yocto-build
cat >> conf/local.conf <<'EOF'
MACHINE = "qemuarm64"
DL_DIR = "${HOME}/compare/downloads"
SSTATE_DIR = "${HOME}/compare/sstate"
EXTRA_IMAGE_FEATURES = "ssh-server-dropbear"
INHERIT += "rm_work"
EOF
time bitbake core-image-minimal 2>&1 | tail -5
echo "YOCTO rootfs: $(du -sh tmp/deploy/images/qemuarm64/*.rootfs.ext4 | cut -f1)"
echo "YOCTO total:  $(du -sh tmp | cut -f1)"
cd ~/compare

# ============ 3. DEBIAN ============
mkdir debian-build && cd debian-build
time sudo mmdebstrap --architectures=arm64 --variant=minbase \
	--include=systemd-sysv,openssh-server,iproute2,ifupdown \
	bookworm rootfs.tar http://deb.debian.org/debian
echo "DEBIAN rootfs: $(du -sh rootfs.tar | cut -f1)"
cd ~/compare

# ============ THE COMPARISON ============
cat <<'EOF'

 Measure and record:
   1. Clean build time
   2. INCREMENTAL build time after changing one package's config  <- key
   3. Rootfs size
   4. Total disk consumed by the build tree
   5. Lines of configuration you wrote
   6. Time to first boot from a clean checkout
   7. Can you do a partial update on the target?
   8. Can you produce an SBOM?
   9. How long did it take you to learn enough to do it?
EOF
```

**Item 2 is the one that decides real projects.** Change one kernel config option in each and
re-measure.

### Lab 100.2 — A Buildroot external tree

```bash
#!/bin/bash
# br_external.sh — the correct way to customize Buildroot.
set -e
EXT=~/compare/my-external
mkdir -p $EXT/{configs,board/myboard/rootfs_overlay/etc,package/myapp}

cat > $EXT/external.desc <<'EOF'
name: MYCOMPANY
desc: My Company customizations
EOF

cat > $EXT/external.mk <<'EOF'
include $(sort $(wildcard $(BR2_EXTERNAL_MYCOMPANY_PATH)/package/*/*.mk))
EOF

cat > $EXT/Config.in <<'EOF'
source "$BR2_EXTERNAL_MYCOMPANY_PATH/package/myapp/Config.in"
EOF

# --- The application package ---
cat > $EXT/package/myapp/Config.in <<'EOF'
config BR2_PACKAGE_MYAPP
	bool "myapp"
	help
	  My custom application.
EOF

mkdir -p $EXT/package/myapp/src
cat > $EXT/package/myapp/src/myapp.c <<'EOF'
#include <stdio.h>
#include <unistd.h>
int main(void) {
	char host[64];
	gethostname(host, sizeof(host));
	printf("myapp running on %s\n", host);
	return 0;
}
EOF

cat > $EXT/package/myapp/src/Makefile <<'EOF'
CC ?= gcc
CFLAGS ?= -O2
myapp: myapp.c
	$(CC) $(CFLAGS) $(LDFLAGS) -o $@ $<
install:
	install -D -m 0755 myapp $(DESTDIR)/usr/bin/myapp
EOF

cat > $EXT/package/myapp/myapp.mk <<'EOF'
MYAPP_VERSION = 1.0
MYAPP_SITE = $(BR2_EXTERNAL_MYCOMPANY_PATH)/package/myapp/src
MYAPP_SITE_METHOD = local
MYAPP_LICENSE = MIT

define MYAPP_BUILD_CMDS
	$(TARGET_MAKE_ENV) $(MAKE) CC="$(TARGET_CC)" CFLAGS="$(TARGET_CFLAGS)" \
		LDFLAGS="$(TARGET_LDFLAGS)" -C $(@D)
endef

define MYAPP_INSTALL_TARGET_CMDS
	$(INSTALL) -D -m 0755 $(@D)/myapp $(TARGET_DIR)/usr/bin/myapp
endef

define MYAPP_INSTALL_INIT_SYSV
	$(INSTALL) -D -m 0755 $(MYAPP_PKGDIR)/S99myapp \
		$(TARGET_DIR)/etc/init.d/S99myapp
endef

$(eval $(generic-package))
EOF

cat > $EXT/package/myapp/S99myapp <<'EOF'
#!/bin/sh
case "$1" in
  start) echo "Starting myapp"; /usr/bin/myapp ;;
  stop)  ;;
  *) echo "Usage: $0 {start|stop}"; exit 1 ;;
esac
EOF

# --- Rootfs overlay: files copied verbatim ---
echo "myboard" > $EXT/board/myboard/rootfs_overlay/etc/hostname

# --- Post-build: run after the rootfs is assembled ---
cat > $EXT/board/myboard/post-build.sh <<'EOF'
#!/bin/sh
set -e
TARGET_DIR=$1
# Production hardening (Ch. 99):
sed -i 's|^root::|root:*:|' $TARGET_DIR/etc/shadow 2>/dev/null || true
rm -rf $TARGET_DIR/usr/share/man $TARGET_DIR/usr/share/doc
echo "build: $(date -u -d @${SOURCE_DATE_EPOCH:-$(date +%s)} -Is)" \
	> $TARGET_DIR/etc/build-info
EOF
chmod +x $EXT/board/myboard/post-build.sh

# --- Build with it ---
cd ~/compare/buildroot
make BR2_EXTERNAL=$EXT qemu_aarch64_virt_defconfig
cat >> .config <<EOF
BR2_PACKAGE_MYAPP=y
BR2_ROOTFS_OVERLAY="$EXT/board/myboard/rootfs_overlay"
BR2_ROOTFS_POST_BUILD_SCRIPT="$EXT/board/myboard/post-build.sh"
EOF
make olddefconfig
make BR2_EXTERNAL=$EXT -j$(nproc)

# Save the configuration properly:
make BR2_EXTERNAL=$EXT BR2_DEFCONFIG=$EXT/configs/myboard_defconfig savedefconfig
cat $EXT/configs/myboard_defconfig
```

### Lab 100.3 — Feel the incremental-rebuild difference

The experiment that settles the Buildroot-vs-Yocto question for your project.

```bash
#!/bin/bash
# incremental.sh — the measurement that matters.

echo "=== BUILDROOT: change one kernel option ==="
cd ~/compare/buildroot
time make linux-rebuild all 2>&1 | tail -3
#   Rebuilds the kernel and re-assembles the rootfs.

echo "=== BUILDROOT: change a LIBRARY's configuration ==="
# e.g. enable a zlib option
make zlib-reconfigure
time make -j$(nproc) 2>&1 | tail -3
#   Buildroot's documentation: after a config change you SHOULD do a
#   full rebuild, because dependency tracking across packages is
#   incomplete. Try it and see what actually happens.

echo "=== BUILDROOT: full rebuild ==="
time make clean all -j$(nproc) 2>&1 | tail -3

echo "=== YOCTO: the same three changes ==="
cd ~/compare/poky && source oe-init-build-env ~/compare/yocto-build
# Kernel option:
time bitbake core-image-minimal 2>&1 | tail -3
# Library config:
echo 'PACKAGECONFIG:append:pn-zlib = " something"' >> conf/local.conf
bitbake-whatchanged core-image-minimal | wc -l
time bitbake core-image-minimal 2>&1 | tail -3

cat <<'EOF'

 Record, for each system:
   - kernel-only change:     ___ min
   - leaf-package change:    ___ min
   - core-library change:    ___ min
   - full rebuild:           ___ min

 Then multiply the leaf-package number by (changes per day) x
 (team size) x (working days per year). That number is the real
 cost of the choice.
EOF
```

### Lab 100.4 — OpenWrt

```bash
cd ~/compare
git clone --depth 1 -b openwrt-23.05 https://git.openwrt.org/openwrt/openwrt.git
cd openwrt
./scripts/feeds update -a && ./scripts/feeds install -a

# Target: x86_64 so it runs in QEMU
cat > .config <<'EOF'
CONFIG_TARGET_x86=y
CONFIG_TARGET_x86_64=y
CONFIG_TARGET_x86_64_DEVICE_generic=y
CONFIG_PACKAGE_luci=y
EOF
make defconfig
make -j$(nproc) 2>&1 | tail -5

ls -lh bin/targets/x86/64/

# Run it:
gunzip -k bin/targets/x86/64/openwrt-*-generic-ext4-combined.img.gz
qemu-system-x86_64 -m 512 -nographic \
	-drive file=bin/targets/x86/64/openwrt-*-combined.img,format=raw \
	-netdev user,id=n0,hostfwd=tcp::8080-:80 -device e1000,netdev=n0

# --- Explore the ideas worth stealing ---
# 1. UCI: ONE configuration model for everything
uci show
uci show network
uci set network.lan.ipaddr='192.168.2.1'
uci commit network
/etc/init.d/network restart

# 2. Overlay root: the base is immutable
mount | grep overlay
cat /proc/mounts | grep -E 'squashfs|overlay'
ls /overlay/upper/            # EVERYTHING that has been changed
firstboot                     # factory reset = wipe the overlay

# 3. procd and ubus
ubus list
ubus call system board
/etc/init.d/network status

# --- The Image Builder: seconds, not hours ---
cd bin/targets/x86/64/
tar xf openwrt-imagebuilder-*.tar.xz && cd openwrt-imagebuilder-*
time make image PROFILE=generic PACKAGES="luci wireguard-tools -ppp -ppp-mod-pppoe"
#   Assembles an image from PREBUILT packages. This is what you use for
#   per-customer variants, and it is a model Yocto lacks a direct
#   equivalent for.
```

### Lab 100.5 — Migrate Buildroot to Yocto

```bash
#!/bin/bash
# migrate.sh — a systematic Buildroot -> Yocto conversion.

echo "=== 1. Inventory the Buildroot configuration ==="
cd ~/compare/buildroot
make BR2_EXTERNAL=$EXT show-info > /tmp/br-packages.json
python3 - <<'PY'
import json
d = json.load(open('/tmp/br-packages.json'))
target = [k for k, v in d.items() if v.get('type') == 'target']
print(f"{len(target)} target packages")
for p in sorted(target)[:40]:
    print(" ", p)
PY

echo "=== 2. Map each to a Yocto recipe ==="
cd ~/compare/yocto-build
for pkg in $(python3 -c "
import json
d=json.load(open('/tmp/br-packages.json'))
print(' '.join(k for k,v in d.items() if v.get('type')=='target'))
"); do
    if bitbake -s 2>/dev/null | grep -q "^$pkg "; then
        echo "FOUND:   $pkg"
    else
        echo "MISSING: $pkg   <- needs a recipe, or a different name"
    fi
done

cat <<'EOF'

=== 3. The conversion map ===

 configs/myboard_defconfig   -> conf/machine/myboard.conf
                                + conf/distro/mydistro.conf
                                + the image recipe
 board/*/linux.config        -> kernel .cfg fragments (Ch. 98 §T.4)
 board/*/rootfs_overlay/     -> a recipe installing the files
 board/*/post-build.sh       -> ROOTFS_POSTPROCESS_COMMAND
 board/*/post-image.sh       -> a .wks file, or IMAGE_CMD_*
 board/*/genimage.cfg        -> a .wks file
 package/<p>/Config.in       -> PACKAGECONFIG in the recipe
 package/<p>/<p>.mk          -> the recipe
 package/<p>/*.patch         -> SRC_URI += "file://*.patch"
 BR2_EXTERNAL tree           -> a layer

=== 4. Order of work ===
 a. Get core-image-minimal booting on the target hardware.
 b. Port the kernel configuration as fragments; verify with
    kernel_configcheck.
 c. Port the BSP: machine conf, bootloader, device tree.
 d. Add packages, in dependency order. Most already exist.
 e. Port custom packages, using devtool add as a starting point.
 f. Port the rootfs overlay and post-build scripts.
 g. Production concerns (Ch. 99).

=== 5. Budget: 2-6 weeks for a non-trivial product. ===
EOF
```

### Lab 100.6 — Make the decision, for real

For each scenario, write a one-page recommendation using §T.7, with the criteria weighted and
the answer defended.

1. **A smart thermostat.** 64 MB flash, 128 MB RAM, ARM Cortex-A7, 5-year life, one SKU, a
   3-person team, OTA updates required.

2. **An industrial gateway product line.** 6 variants, 1 GB flash, 10-year support
   commitment, IEC 62443 certification, EU market (CRA applies), a 15-person team.

3. **A prototype robot.** x86, 32 GB SSD, ROS2, a research team that knows Ubuntu, 18-month
   horizon, likely to be thrown away.

4. **A consumer router.** 16 MB flash, 128 MB RAM, must have a web UI, thousands of SKU
   variants across retail partners.

5. **A medical device.** Certification required, a 15-year lifetime, an immutable rootfs, a
   full audit trail, and a complete SBOM.

6. **A camera with cloud-connected ML inference.** 4 GB storage, a team of cloud engineers,
   frequent application updates, infrequent OS updates.

**The interesting ones are 1 and 6.** Scenario 1 looks like Buildroot (small, one SKU, small
team) but the 5-year life and OTA requirement push toward Yocto's update and compliance
story. Scenario 6 looks like a container platform, and the question is whether the overhead
fits. Defend your answer either way; the reasoning is what is graded.

---

## 3. Mastery drills

1. Build the same product in Buildroot, Yocto, and a Debian rootfs. Produce the full
   comparison table from Lab 100.1, including the incremental-rebuild numbers.

2. Create a complete `BR2_EXTERNAL` tree for a real board with a custom package, a kernel
   fragment, a rootfs overlay, and post-build hardening.

3. Migrate a real Buildroot project to Yocto. Record the actual effort and compare it with
   the 2–6 week estimate.

4. Build an OpenWrt image for a real router and add a custom package with a UCI configuration
   file and a `procd` init script.

5. Measure the incremental-rebuild cost difference over a simulated week of development (20
   changes) in Buildroot and Yocto. Extrapolate to a team and a year.

6. Implement the same production hardening (Ch. 99 §T.2) in Buildroot, Yocto, and a Debian
   image. Compare the effort and the verifiability.

7. Investigate `isar`: build a Debian-based image with BitBake. Assess whether it genuinely
   combines the advantages or inherits both sets of problems.

8. Build a NixOS image for an embedded target. Assess the reproducibility story against
   Yocto's, honestly, including where Nix is genuinely better.

9. Produce an SBOM from each of Buildroot, Yocto, and Debian. Compare completeness and effort.

10. Write the decision documents for all six scenarios in Lab 100.6 and have a colleague
    argue the opposite case for each.

11. Take the OpenWrt overlay-root and UCI ideas and design equivalents for a Yocto-based
    product. Assess whether they are worth implementing.

12. Write your organization's build-system standard: which system for which product class,
    what the migration triggers are, and what the shared infrastructure is.

---

## 4. Further reading

**Buildroot**
- **The Buildroot user manual** — genuinely good, and short enough to read completely. Do
  that; it takes an afternoon and saves weeks
- Thomas Petazzoni's Buildroot talks (ELCE, every year) — the maintainer's perspective on
  design decisions
- `docs/manual/` in the tree
- The `buildroot@buildroot.org` list

**OpenWrt**
- OpenWrt developer guide, especially the package and UCI documentation
- The `procd` and `ubus` documentation — the ideas are transferable
- OpenWrt's Table of Hardware, as an example of a device database done well

**Debian-based**
- `mmdebstrap(1)` — read the man page; it explains why `debootstrap` is superseded
- `debos`, `elbe`, and `isar` documentation
- The Debian Reproducible Builds project

**Comparison and decision**
- Chris Simmonds, *Mastering Embedded Linux Programming*, ch. 6 — the fairest published
  comparison of Yocto and Buildroot
- Bootlin's "Buildroot vs Yocto" talks and blog posts — from people who train both
- The ELCE/Embedded Open Source Summit talks comparing build systems, annually

**The alternatives**
- PTXdist documentation (Pengutronix)
- NixOS manual and the `nixos-hardware` repository
- AOSP's build system documentation (Soong, Blueprint) and the GKI documentation
- balenaOS and Torizon architecture documentation
- Zephyr's documentation, for when Linux is the wrong answer

**Cross-references**
- Ch. 92 §T.10 — cross-compilation, the problem all of these solve
- Ch. 96–99 — Yocto in detail
- Ch. 88 — delta management, which every system faces
- Ch. 101 — board bring-up, where the choice becomes concrete
- Ch. 89 — the decision framework this chapter applies

→ Next: [101-board-bringup.md](101-board-bringup.md)

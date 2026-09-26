# Chapter 96 — Yocto I: Architecture and Concepts

> Four chapters on Yocto, because it is what the embedded Linux industry actually uses and
> because it is genuinely hard to learn from its own documentation — which is excellent as a
> reference and poor as an explanation. This chapter builds the mental model: what the pieces
> are, what problem each solves, and why the system is shaped the way it is. Ch. 97 covers
> the language, Ch. 98 the BSP and kernel, Ch. 99 production concerns.

---

## Theory & First Principles

### T.0 — Start here: ship the same firmware in 2031

Your product ships in 2025. It is a medical device, or a car, or an industrial controller, and
it must be **supported until 2038**. In 2031 a security bug is reported. You must produce a
new firmware image that differs from the shipped one **in exactly one package** — and you must
prove it.

**Now consider doing that with the obvious approach:**

```bash
apt-get install build-essential
./configure && make && make install
```

In 2031: the distro is EOL, the package versions have moved, `configure` picks up whatever
happens to be on the build machine, and your image differs from the shipped one in ten
thousand ways you cannot enumerate. **You cannot ship that to a regulator, and you cannot
debug a field failure against it.**

**The requirements an embedded build system actually has to meet are brutal, and they are
unlike a desktop distribution's:**

| Requirement | Why |
|---|---|
| **Reproducible** | same inputs → bit-identical output, years later |
| **Cross-compiled** | you build on x86-64; the target is ARM with 64 MB of RAM |
| **Minimal** | every package is attack surface, flash cost, and boot time |
| **Auditable** | an SBOM and a license manifest — often a legal requirement (GPL, FDA, ISO 26262) |
| **Hermetic** | the build must **not** depend on what is installed on the build host |
| **Layered** | your product code, the vendor's BSP, and the core system must be separable |

**"Hermetic" is the one that dictates the architecture.** If the build can see the host's
`/usr/include`, it is not reproducible. So Yocto builds its own compiler, its own `make`, its
own `python` — the `-native` recipes — and then uses *those* to cross-compile everything for
the target. **It bootstraps a toolchain in order to trust it**, which is Ch. 90 §T.0's chain
of bootstraps appearing again in an entirely different domain.

**The second core idea is the task graph.** Yocto is not a script runner; it is a dependency
engine:

```
  every recipe (foo_1.2.bb) expands to a chain of TASKS:

    do_fetch -> do_unpack -> do_patch -> do_configure -> do_compile
             -> do_install -> do_package -> do_package_write_rpm

  each task has a SIGNATURE: a hash of its inputs --
    the recipe text, the variables it uses, the source checksums,
    and the signatures of everything it depends on.

  signature unchanged -> the task is SKIPPED and its output restored
                         from the shared state cache (sstate).
```

**That signature scheme is the whole of Yocto's performance and correctness story in one
mechanism.** Change one line of one recipe and only the tasks whose signatures changed re-run
— typically a handful out of 30,000. **Content-addressed, dependency-aware caching:** the same
idea as `ccache`, Nix, Bazel, and Docker layers, and the reason a rebuild takes two minutes
rather than six hours.

**And the third idea — layers — is the organizational one, and it is really Ch. 00 §T.1 again:**

```
  meta-mycompany-product   <- YOUR policy: what goes in the image
  meta-mycompany-bsp       <- your board: device tree, kernel config
  meta-vendor              <- the SoC vendor's BSP (you do not own this)
  meta-openembedded        <- community recipes
  meta (oe-core)           <- the core system (you do not modify this)
```

**You never edit a lower layer. You override from a higher one** (`.bbappend`, `_append`,
`PREFERRED_VERSION`). Which means when the vendor ships a new BSP, you replace one layer
instead of re-merging a fork. **Layering is how you keep your changes separable from other
people's code over a ten-year support horizon** — and it is the same lesson as Ch. 88 §T.0's
warning about divergence from upstream.

**The honest cost:** Yocto is genuinely hard. The learning curve is steep, the first build is
hours, the error messages are opaque, and the layer-override semantics take months to
internalize. **It buys you reproducibility and auditability, and it charges you complexity.**
Ch. 100 exists precisely to ask when that trade is wrong.

```bash
source oe-init-build-env && bitbake -e core-image-minimal | grep '^DISTRO'
bitbake -g core-image-minimal && cat pn-buildlist | wc -l
bitbake-layers show-layers && bitbake-layers show-recipes | head
bitbake -S none core-image-minimal        # dump task signatures
```

---

### T.1 — The problem Yocto solves

You need a Linux system for a custom board. The requirements, all at once:

| Requirement | Why it is hard |
|---|---|
| **Cross-compiled** | Nothing runs on the target during the build (Ch. 92 §T.10) |
| **Reproducible** | The same inputs must give bit-identical outputs, for audit and for debugging |
| **Minimal** | 40 MB, not 4 GB. Every package is attack surface and flash cost |
| **Custom kernel + bootloader** | Your board, your drivers, your defconfig |
| **Licence-compliant** | You must know and ship every licence, and provide sources for GPL |
| **Auditable** | An SBOM, and the ability to answer "does this contain CVE-X?" |
| **Maintainable for a decade** | Security updates, and a build that still works in 2035 |
| **Multiple products** | Shared layers, per-product configuration |

Doing this by hand is a build system. Doing it by hand *well* is a multi-year project. Yocto
is that project, done, and widely used.

**The alternatives** and when each is right:

| | Yocto/OE | Buildroot | Debian/Ubuntu rootfs | Container images |
|---|---|---|---|---|
| Learning curve | **steep** | gentle | none | none |
| Flexibility | total | high | low | medium |
| Build time (clean) | 2–8 hours | 30–60 min | minutes | minutes |
| Incremental | good (sstate) | **rebuilds a lot** | n/a | layer cache |
| Package management on target | yes (rpm/deb/ipk) | no | yes | n/a |
| Licence tracking | **excellent** | basic | distro's | poor |
| SBOM | built in (SPDX) | partial | distro's | tooling |
| Multi-product | **designed for it** | awkward | awkward | fine |

**The rule of thumb:** Buildroot for a small, single, simple product; Yocto for a product
line, a long lifetime, or anything with compliance requirements; a Debian-derived rootfs when
hardware is standard and size does not matter.

### T.2 — The components, and what each actually is

The naming is a real obstacle. Get this straight once:

| Name | What it is |
|---|---|
| **BitBake** | The build *engine*. A Python task executor with a dependency graph. Knows nothing about Linux |
| **OpenEmbedded-Core (OE-Core)** | The *metadata*: ~1000 recipes for the base system, plus the classes that make them work |
| **Poky** | A *reference distribution*: OE-Core + BitBake + the `meta-poky` distro config + `meta-yocto-bsp`. What you download to get started |
| **The Yocto Project** | The *umbrella*: releases, testing, documentation, the autobuilder, and governance |
| **A layer** (`meta-*`) | A directory of recipes and configuration that can be added to a build |
| **A recipe** (`.bb`) | How to build one piece of software |
| **A class** (`.bbclass`) | Reusable recipe logic, inherited |
| **An image** (`.bb` inheriting `image.bbclass`) | A list of packages plus rootfs construction |
| **A machine** (`.conf`) | A board: architecture, kernel, bootloader, tuning |
| **A distro** (`.conf`) | Policy: libc, init system, features, package format |

**The distinction that unlocks it:** *machine* is hardware, *distro* is policy, *image* is
contents. Three orthogonal axes. You can build the same image for three machines with two
distros, and that is the point.

```
        MACHINE=raspberrypi4-64    DISTRO=poky        IMAGE=core-image-minimal
              ↓                         ↓                        ↓
        which hardware            which policy             which packages
        (kernel, u-boot,          (libc, init,             (the package list
         tune flags, DT)           features, format)        and rootfs setup)
```

### T.3 — Layers: the composition model

```
 meta-myproduct          <- your product (highest priority)
 meta-mycompany-bsp      <- your board support
 meta-openembedded/*     <- meta-oe, meta-python, meta-networking, ...
 meta-<vendor>           <- meta-ti, meta-intel, meta-raspberrypi, ...
 meta-poky               <- the reference distro
 meta                    <- OE-Core (lowest priority)
```

A layer is just a directory with a `conf/layer.conf`. The rules:

1. **Layers are additive.** Adding a layer adds recipes.
2. **`BBFILE_PRIORITY` breaks ties** when two layers provide the same recipe.
3. **`.bbappend` files extend a recipe without forking it.** This is the crucial mechanism:
   you never copy a recipe to modify it; you append.
4. **`LAYERDEPENDS` declares dependencies** between layers.

```
 meta-mylayer/
 ├── conf/layer.conf
 ├── recipes-core/
 │   └── busybox/
 │       ├── busybox_%.bbappend          <- extends OE-Core's busybox
 │       └── busybox/myconfig.cfg
 ├── recipes-myapp/
 │   └── myapp/
 │       └── myapp_1.0.bb
 └── recipes-kernel/
     └── linux/
         ├── linux-yocto_%.bbappend
         └── linux-yocto/my.cfg
```

**Why `.bbappend` matters so much:** it is the difference between a maintainable product and
a fork. When OE-Core updates BusyBox from 1.36 to 1.37, your `busybox_%.bbappend` (the `%`
matches any version) still applies. If you had copied the recipe, you now have a merge
conflict and a stale version. This is Ch. 88's delta-management principle, expressed as a
build-system feature.

### T.4 — BitBake's execution model

```
 1. PARSE      read every .conf, .bbclass, and .bb in the layers
               -> a giant key/value datastore per recipe
               (this is why the first parse takes minutes; it is cached after)

 2. DEPENDENCY build the task graph from DEPENDS/RDEPENDS
               -> a DAG of TASKS, not of recipes

 3. EXECUTE    run tasks in dependency order, in parallel (BB_NUMBER_THREADS)
```

**The unit of work is a task, not a recipe.** Every recipe has a standard task sequence:

```
 do_fetch          download sources into DL_DIR
 do_unpack         extract into WORKDIR
 do_patch          apply patches (quilt)
 do_prepare_recipe_sysroot    populate the recipe-specific sysroot
 do_configure      autotools/cmake/meson configure
 do_compile        make
 do_install        make install into ${D} (a staging directory)
 do_package        split ${D} into packages: main, -dev, -dbg, -doc, -staticdev
 do_package_write_*  produce .rpm/.deb/.ipk
 do_populate_sysroot   export headers/libs for OTHER recipes to build against
 do_build          (the default goal)
```

You can run any single task:

```bash
bitbake -c compile myapp
bitbake -c devshell myapp      # a shell in the build environment. INVALUABLE
bitbake -c listtasks myapp
bitbake -c cleansstate myapp   # force a full rebuild of this recipe
```

### T.5 — sstate: the thing that makes Yocto usable

A clean build is hours. An incremental build is minutes. The mechanism is **shared state**.

```
 For each task:
   1. Compute a SIGNATURE: a hash of
        - the task's own code
        - every variable the task reads
        - the signatures of all its dependencies
   2. Is there an sstate archive with that signature?
        YES -> unpack it. Task done. (a "setscene" task)
        NO  -> run the task, then archive the result
```

**Consequences that explain most Yocto behaviour:**

- **Changing a variable rebuilds everything that reads it.** Change `CFLAGS` and you rebuild
  the world, because every compile task's signature changed. This is *correct* — the output
  genuinely differs — but it is why a one-character change to `local.conf` can cost four
  hours.
- **A shared sstate cache across a team is enormously valuable.** Set `SSTATE_MIRRORS` to an
  HTTP server and a developer's first build takes 20 minutes instead of 6 hours.
- **`bitbake-diffsigs` tells you *why* something rebuilt.** This is the single most useful
  debugging tool in Yocto and almost nobody knows it.

```bash
bitbake -S none myapp                    # dump signatures
bitbake-diffsigs tmp/stamps/.../myapp.do_compile.sigdata.*
#   -> "Variable CFLAGS value changed from X to Y"
bitbake-whatchanged core-image-minimal   # what will rebuild and why
```

### T.6 — Sysroots and the cross-compilation model

```
 tmp/work/<arch>/<recipe>/<version>/
 ├── <source>/                      unpacked and patched sources
 ├── build/                         out-of-tree build dir
 ├── image/                         ${D} -- what `make install` produced
 ├── packages-split/                after do_package
 ├── recipe-sysroot/                TARGET headers+libs this recipe builds against
 ├── recipe-sysroot-native/         HOST tools this recipe needs
 └── temp/                          LOG FILES. This is where you look when it breaks
     ├── log.do_compile
     ├── run.do_compile             the actual script that was executed
     └── log.task_order
```

**Per-recipe sysroots** (since 2.3) were a major correctness improvement: each recipe sees
*only* what it declared in `DEPENDS`. Before that, a shared sysroot meant a recipe could
accidentally build against something another recipe happened to have installed — so builds
were order-dependent and non-reproducible. If a recipe fails with "cannot find foo.h," the
answer is almost always a missing `DEPENDS`.

**The three classes of recipe**, and the distinction is fundamental:

| | Runs on | Built for | Inherit |
|---|---|---|---|
| **target** | the device | `${TARGET_ARCH}` | (default) |
| **native** | the build host | `${BUILD_ARCH}` | `native` or `BBCLASSEXTEND = "native"` |
| **nativesdk** | an SDK user's host | `${SDK_ARCH}` | `nativesdk` |

A recipe that must run during the build — a code generator, `pkg-config`, a compiler — needs
a `-native` variant, and dependencies on it go in `DEPENDS` as `foo-native`.

### T.7 — Packaging: one recipe, many packages

`do_package` splits `${D}` into packages by `FILES_*`:

| Package | Contains | `FILES` default |
|---|---|---|
| `${PN}` | binaries, libraries | `/usr/bin`, `/usr/lib/lib*.so.*`, `/etc` |
| `${PN}-dev` | headers, `.so` symlinks, `.pc` files | `/usr/include`, `/usr/lib/*.so` |
| `${PN}-dbg` | debug symbols | `/usr/lib/.debug` |
| `${PN}-doc` | documentation, man pages | `/usr/share/doc`, `/usr/share/man` |
| `${PN}-staticdev` | static libraries | `/usr/lib/*.a` |
| `${PN}-locale` | translations | `/usr/share/locale` |

**The `-dev`/`-dbg` split is why Yocto images are small.** The target gets `${PN}`; the
headers and symbols stay on the build host, available for an SDK or for offline debugging.

**`DEPENDS` versus `RDEPENDS`**, the distinction people get wrong constantly:

```
 DEPENDS  = "zlib openssl"       # BUILD-time: I need their headers and libs
                                 #   -> they must be in my recipe-sysroot
 RDEPENDS:${PN} = "bash python3" # RUN-time: my package needs theirs installed
                                 #   -> they go in the image
```

A library needs a `DEPENDS` on zlib to compile and gets an *automatic* `RDEPENDS` on
`libz1` from the shared-library scan. A shell script needs only an `RDEPENDS` on `bash`. Most
"it built but fails on the target" bugs are a missing `RDEPENDS`.

### T.8 — Variable semantics

BitBake's variable operators are its own language and the source of most confusion.

| Operator | Meaning | When evaluated |
|---|---|---|
| `=` | Lazy assignment | Expanded at *use* time |
| `:=` | Immediate | Expanded *now* |
| `?=` | Default; set only if unset | at parse |
| `??=` | Weaker default; the last one wins | at parse |
| `+=` | Append **with a space** | at parse |
| `=+` | Prepend with a space | at parse |
| `.=` | Append, **no space** | at parse |
| `=.` | Prepend, no space | at parse |
| `:append` | Append **after all parsing** | **after** |
| `:prepend` | Prepend after all parsing | **after** |
| `:remove` | Remove a word after all parsing | **after** |

**The rule that saves you: in a `.bbappend`, always use `:append`/`:prepend`/`:remove`, never
`+=`.** `+=` is applied at parse time in file order, so whether it wins depends on parse
order, which is not stable. `:append` is applied after everything has been parsed, so it
always wins.

```bash
# In a .bbappend:
EXTRA_OECONF:append = " --enable-myfeature"     # CORRECT. Note the leading space.
EXTRA_OECONF += "--enable-myfeature"            # WRONG. Order-dependent.
```

**Overrides** are the conditional mechanism, and they are elegant:

```bash
OVERRIDES = "arm:armv7a:mymachine:poky:class-target:forcevariable"
#   (a colon-separated list, LOW to HIGH priority, built from
#    MACHINE, DISTRO, TARGET_ARCH, etc.)

SRC_URI = "file://generic.patch"
SRC_URI:mymachine = "file://special.patch"      # only for MACHINE=mymachine
SRC_URI:append:arm = " file://arm-fix.patch"    # appended only on ARM
RDEPENDS:${PN}:remove = "bash"                  # drop a runtime dep
```

A variable with an override suffix matching an entry in `OVERRIDES` replaces the base
variable, highest-priority override winning. This is how one recipe serves many machines
without any `if` statements.

### T.9 — Images, features, and package groups

```bash
# An image recipe
SUMMARY = "My product image"
LICENSE = "MIT"
inherit core-image

IMAGE_FEATURES += "ssh-server-openssh"
IMAGE_INSTALL += "myapp packagegroup-myproduct"
IMAGE_ROOTFS_SIZE = "8192"
IMAGE_FSTYPES = "wic.gz ext4 tar.bz2"
```

The three feature variables, which are constantly confused:

| Variable | Set in | Controls |
|---|---|---|
| `DISTRO_FEATURES` | distro `.conf` | What is *built*: systemd vs sysvinit, wayland, x11, bluetooth, pam, ... |
| `MACHINE_FEATURES` | machine `.conf` | What the *hardware* has: wifi, bluetooth, usbhost, screen, ... |
| `IMAGE_FEATURES` | image recipe | What goes in *this image*: debug-tweaks, ssh-server, tools-debug, dev-pkgs |

`DISTRO_FEATURES` is the big lever. Removing `x11` and `wayland` from a headless product
removes hundreds of packages and hours of build time. Changing it rebuilds nearly everything
(§T.5), so decide early.

**`IMAGE_FEATURES = "debug-tweaks"` sets an empty root password and enables a serial login.**
It is in the default images and it must not ship. Ch. 99 covers the production image.

### T.10 — The mental model, assembled

```
 conf/bblayers.conf       which layers
        │
        ├── conf/local.conf         MACHINE, DISTRO, and your overrides
        │
        ▼
    BitBake parses everything -> a datastore per recipe
        │
        ▼
    Task graph from DEPENDS/RDEPENDS
        │
        ├── sstate hit?  -> unpack, skip the work
        └── sstate miss? -> fetch, unpack, patch, configure, compile,
                            install, package, populate_sysroot
        │
        ▼
    do_rootfs: install packages into a rootfs
        │
        ▼
    do_image_*: produce ext4 / wic / tar / squashfs
```

**The three files you edit most:** `conf/local.conf` (your build), `conf/bblayers.conf` (which
layers), and your own layer's recipes. Everything else you should be *appending* to, not
editing.

---

## 1. Internals

### Directory layout

```
 poky/
 ├── bitbake/                    the engine (a separate project)
 ├── meta/                       OE-CORE -- ~1000 recipes
 │   ├── classes/                *.bbclass: autotools, cmake, systemd, kernel, image
 │   ├── conf/
 │   │   ├── bitbake.conf        THE base configuration. Read it once
 │   │   ├── documentation.conf  every variable, documented
 │   │   ├── machine/            qemu* reference machines
 │   │   └── distro/             defaultsetup
 │   ├── recipes-core/           glibc, busybox, systemd, base-files, images
 │   ├── recipes-devtools/       gcc, binutils, python, cmake, qemu
 │   ├── recipes-kernel/         linux-yocto, linux-firmware, perf, lttng
 │   ├── recipes-bsp/            u-boot, grub, efibootmgr
 │   ├── recipes-connectivity/   openssh, openssl, wpa-supplicant
 │   ├── recipes-extended/       ...
 │   └── lib/oe/                 Python helper libraries
 ├── meta-poky/                  the poky DISTRO
 ├── meta-yocto-bsp/             reference BSPs
 ├── scripts/                    bitbake-layers, devtool, wic, runqemu, oe-pkgdata-util
 └── oe-init-build-env           the setup script

 build/                          created by oe-init-build-env
 ├── conf/
 │   ├── local.conf              YOUR configuration
 │   ├── bblayers.conf           which layers
 │   └── site.conf               optional, shared site settings
 ├── downloads/                  DL_DIR -- source tarballs. SHARE THIS
 ├── sstate-cache/               SSTATE_DIR -- shared state. SHARE THIS TOO
 └── tmp/
     ├── deploy/
     │   ├── images/<machine>/   THE OUTPUT: kernel, dtb, rootfs, wic
     │   ├── rpm|deb|ipk/        packages
     │   ├── licenses/           per-image licence manifests
     │   └── sdk/                SDK installers
     ├── work/                   per-recipe build directories
     ├── work-shared/            kernel and gcc sources (shared)
     ├── sysroots-components/    staged sysroot pieces
     ├── stamps/                 task completion stamps and signature data
     ├── cache/                  parse cache
     └── log/                    cooker logs
```

**`meta/conf/bitbake.conf` and `meta/conf/documentation.conf` are worth an hour each.** The
first defines every base variable; the second documents them. Between them they answer most
"what does X mean" questions faster than the manual.

### Essential commands

```bash
# --- Setup ---
source oe-init-build-env [builddir]

# --- Build ---
bitbake core-image-minimal
bitbake -k core-image-minimal        # keep going after failures
bitbake -c <task> <recipe>
bitbake -c devshell <recipe>         # shell with the cross environment set up
bitbake -c devpyshell <recipe>       # Python shell with the datastore

# --- Inspect ---
bitbake -e <recipe> | less           # the FULLY EXPANDED datastore
bitbake -e <recipe> | grep '^SRC_URI='
bitbake -e <recipe> | grep -B5 '^# \$FOO'   # the VARIABLE HISTORY: who set it, where
bitbake -g <recipe>                  # dependency graphs (.dot)
bitbake -s                           # every recipe and version
bitbake-layers show-layers
bitbake-layers show-recipes
bitbake-layers show-appends
bitbake-layers show-overlayed        # recipes shadowed by a higher-priority layer
bitbake-layers create-layer meta-mylayer
bitbake-layers add-layer ../meta-mylayer

# --- Why did it rebuild? ---
bitbake -S none <recipe>
bitbake-diffsigs <sigdata-old> <sigdata-new>
bitbake-whatchanged <image>

# --- Packages ---
oe-pkgdata-util list-pkg-files -p <recipe>
oe-pkgdata-util find-path /usr/bin/foo     # which package provides this?
oe-pkgdata-util lookup-recipe <package>

# --- Run it ---
runqemu qemux86-64 core-image-minimal nographic
runqemu qemuarm64 core-image-minimal nographic slirp

# --- Cleanup ---
bitbake -c clean <recipe>            # remove WORKDIR
bitbake -c cleansstate <recipe>      # + remove the sstate entries
bitbake -c cleanall <recipe>         # + remove downloads
```

**`bitbake -e <recipe> | grep -B5 '^# \$VARNAME'` is the killer command.** It prints the
complete history of a variable: every file and line that touched it, in order. When you
cannot work out why a variable has the value it does, this answers it in seconds.

---

## 2. Practice

### Lab 96.1 — First build

```bash
#!/bin/bash
# yocto_first.sh — from nothing to a booting image.
set -e

echo "=== Host dependencies (Ubuntu/Debian) ==="
sudo apt install -y gawk wget git diffstat unzip texinfo gcc build-essential \
	chrpath socat cpio python3 python3-pip python3-pexpect xz-utils \
	debianutils iputils-ping python3-git python3-jinja2 python3-subunit \
	zstd liblz4-tool file locales libacl1 lz4

echo "=== Clone ==="
mkdir -p ~/yocto && cd ~/yocto
[ -d poky ] || git clone git://git.yoctoproject.org/poky
cd poky
git checkout scarthgap          # or the current LTS

echo "=== Initialize ==="
source oe-init-build-env ~/yocto/build

echo "=== Configure ==="
cat >> conf/local.conf <<'EOF'

# ---- Our settings ----
MACHINE ?= "qemux86-64"
# Share these ACROSS builds -- the single biggest time saver.
DL_DIR     ?= "${HOME}/yocto/downloads"
SSTATE_DIR ?= "${HOME}/yocto/sstate-cache"

# Parallelism. Tune to your machine.
BB_NUMBER_THREADS ?= "8"
PARALLEL_MAKE     ?= "-j 8"

# Do not keep every intermediate directory (saves a LOT of disk).
INHERIT += "rm_work"

# Tell us what licences we are shipping.
INHERIT += "archiver"
COPY_LIC_MANIFEST = "1"
COPY_LIC_DIRS = "1"

# Keep the build deterministic and reportable.
BB_DISKMON_DIRS = "STOPTASKS,${TMPDIR},1G,100K ABORT,${TMPDIR},100M,1K"
EOF

echo "=== Build (this takes 1-4 hours the first time) ==="
time bitbake core-image-minimal

echo "=== Output ==="
ls -lh tmp/deploy/images/qemux86-64/

echo "=== Boot it ==="
echo "  runqemu qemux86-64 core-image-minimal nographic"
echo "  (login: root, no password -- because debug-tweaks is on. Ch. 99.)"
```

**While it builds, read `meta/conf/bitbake.conf`.** Genuinely — it is the best use of that
time and it demystifies half of what follows.

### Lab 96.2 — Explore the datastore

The lab that teaches the model.

```bash
cd ~/yocto/build

# 1. Everything about a recipe, fully expanded:
bitbake -e busybox > /tmp/busybox.env
wc -l /tmp/busybox.env                # ~10,000 lines. This is the datastore.
grep '^SRC_URI=' /tmp/busybox.env
grep '^S=' /tmp/busybox.env           # source dir
grep '^B=' /tmp/busybox.env           # build dir
grep '^D=' /tmp/busybox.env           # install destdir
grep '^WORKDIR=' /tmp/busybox.env

# 2. THE VARIABLE HISTORY -- the command that answers "why is this set?"
bitbake -e busybox | grep -B20 '^# \$CFLAGS'
#   -> every file:line that touched CFLAGS, in order.

# 3. Where does a variable come from?
bitbake -e core-image-minimal | grep -B10 '^# \$IMAGE_INSTALL'

# 4. What tasks exist, and in what order?
bitbake -c listtasks busybox

# 5. Get a shell INSIDE the build environment:
bitbake -c devshell busybox
#   You are now in ${S} with CC, CFLAGS, and the sysroot all set.
#   Try:  echo $CC ; $CC --version ; make menuconfig
#   This is how you debug a build failure interactively.
exit

# 6. What did a failing task actually run?
cat tmp/work/*/busybox/*/temp/run.do_compile
cat tmp/work/*/busybox/*/temp/log.do_compile
#   run.* is the generated SCRIPT. log.* is its output. When a build
#   fails, read run.* to see exactly what was executed.

# 7. Dependency graph
bitbake -g core-image-minimal
head -40 task-depends.dot
grep '"busybox' pn-buildlist

# 8. What is in the image, and why?
oe-pkgdata-util list-pkgs -p busybox
oe-pkgdata-util find-path /bin/sh
cat tmp/deploy/images/qemux86-64/core-image-minimal-*.manifest
```

### Lab 96.3 — Create a layer and a recipe

```bash
cd ~/yocto

# 1. Create the layer.
bitbake-layers create-layer meta-mylayer
bitbake-layers add-layer ../meta-mylayer
bitbake-layers show-layers

cat meta-mylayer/conf/layer.conf
#   BBFILE_COLLECTIONS, BBFILE_PATTERN, BBFILE_PRIORITY, LAYERDEPENDS,
#   LAYERSERIES_COMPAT  <- this last one must list your Yocto release

# 2. A trivial application to package.
mkdir -p meta-mylayer/recipes-myapp/hello/files
cat > meta-mylayer/recipes-myapp/hello/files/hello.c <<'EOF'
#include <stdio.h>
#include <unistd.h>
int main(void) {
	char host[64];
	gethostname(host, sizeof(host));
	printf("hello from %s\n", host);
	return 0;
}
EOF

cat > meta-mylayer/recipes-myapp/hello/files/hello.service <<'EOF'
[Unit]
Description=Hello service
[Service]
Type=oneshot
ExecStart=/usr/bin/hello
RemainAfterExit=yes
[Install]
WantedBy=multi-user.target
EOF

# 3. The recipe.
cat > meta-mylayer/recipes-myapp/hello/hello_1.0.bb <<'EOF'
SUMMARY = "A hello world application"
DESCRIPTION = "Demonstrates a minimal Yocto recipe"
HOMEPAGE = "https://example.com"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

SRC_URI = "file://hello.c \
           file://hello.service \
          "

S = "${WORKDIR}/sources"
UNPACKDIR = "${S}"

inherit systemd

SYSTEMD_SERVICE:${PN} = "hello.service"
SYSTEMD_AUTO_ENABLE = "enable"

do_compile() {
	${CC} ${CFLAGS} ${LDFLAGS} -o hello ${S}/hello.c
}

do_install() {
	install -d ${D}${bindir}
	install -m 0755 hello ${D}${bindir}/

	if ${@bb.utils.contains('DISTRO_FEATURES','systemd','true','false',d)}; then
		install -d ${D}${systemd_system_unitdir}
		install -m 0644 ${S}/hello.service ${D}${systemd_system_unitdir}/
	fi
}

FILES:${PN} += "${systemd_system_unitdir}/hello.service"
EOF

# 4. Build it.
cd ~/yocto/build
bitbake hello
oe-pkgdata-util list-pkg-files -p hello

# 5. Put it in an image.
echo 'IMAGE_INSTALL:append = " hello"' >> conf/local.conf
bitbake core-image-minimal
runqemu qemux86-64 core-image-minimal nographic
#   In the target:  hello ; systemctl status hello
```

**Note `LIC_FILES_CHKSUM`.** It is mandatory, and it is a checksum of the licence text. If
upstream changes the licence, the checksum mismatches and the build fails loudly — which is
exactly what you want for compliance (Ch. 99).

### Lab 96.4 — `.bbappend`: extend without forking

```bash
# Customize BusyBox without touching OE-Core.
mkdir -p ~/yocto/meta-mylayer/recipes-core/busybox/busybox

cat > ~/yocto/meta-mylayer/recipes-core/busybox/busybox/myconfig.cfg <<'EOF'
CONFIG_ASH_RANDOM_SUPPORT=y
CONFIG_FEATURE_EDITING_SAVEHISTORY=y
CONFIG_TRACEROUTE=y
CONFIG_TRACEROUTE6=y
# CONFIG_TELNETD is not set
EOF

cat > ~/yocto/meta-mylayer/recipes-core/busybox/busybox_%.bbappend <<'EOF'
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"

SRC_URI += "file://myconfig.cfg"
EOF

cd ~/yocto/build
bitbake busybox
bitbake-layers show-appends | grep -A2 busybox

# Verify the fragment was applied:
grep -E 'TRACEROUTE|TELNETD' tmp/work/*/busybox/*/build/.config
```

The three idioms in that `.bbappend`, all of which you will use constantly:

1. **`FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"`** — adds this layer's directory to the
   file search path. Without it, `file://myconfig.cfg` is not found. Note `:=` (immediate) so
   `${THISDIR}` expands to *this* file's directory.
2. **`busybox_%.bbappend`** — `%` matches any version, so it survives upstream version bumps.
3. **`SRC_URI += `** in an append is acceptable here because there is only one append; in
   general, prefer `:append`.

### Lab 96.5 — Trace a rebuild

```bash
cd ~/yocto/build

# 1. Build, then change something trivial and see what rebuilds.
bitbake core-image-minimal
bitbake -S none core-image-minimal        # snapshot the signatures
cp -r tmp/stamps /tmp/stamps-before

# 2. Change a variable.
echo 'EXTRA_IMAGE_FEATURES += "tools-debug"' >> conf/local.conf

# 3. What will rebuild, and why?
bitbake-whatchanged core-image-minimal

# 4. For a specific recipe, diff the signatures:
bitbake -S none busybox
bitbake-diffsigs tmp/stamps/*/busybox/*.do_compile.sigdata.* | head -30

# 5. Now do the EXPENSIVE change and predict the cost first.
echo 'DISTRO_FEATURES:remove = "x11 wayland"' >> conf/local.conf
bitbake-whatchanged core-image-minimal | wc -l
#   -> a very large number. This is §T.5 made visible.

# 6. Set up an sstate mirror (do this for any team).
cat >> conf/local.conf <<'EOF'
SSTATE_MIRRORS ?= "file://.* http://sstate.example.com/PATH;downloadfilename=PATH"
BB_SIGNATURE_HANDLER = "OEEquivHash"
BB_HASHSERVE = "auto"
BB_HASHSERVE_UPSTREAM = "hashserv.yoctoproject.org:8686"
EOF
#   BB_HASHSERVE is "hash equivalence": if two different signatures
#   produce IDENTICAL output, they are treated as equivalent, so a
#   cosmetic change does not cascade a rebuild. This is a large win.
```

### Lab 96.6 — Compare with Buildroot

```bash
# Build an equivalent system with Buildroot and measure the difference.
cd ~ && git clone --depth 1 https://git.buildroot.net/buildroot
cd buildroot
make qemu_x86_64_defconfig
time make -j$(nproc)
ls -lh output/images/

# Compare:
#   - clean build time
#   - INCREMENTAL build time after a one-line change   <- the big one
#   - rootfs size
#   - number of configuration files you had to touch
#   - what you gave up (package management, SBOM, multiconfig, SDK)

# The incremental comparison is the point: change one package's config
# and rebuild in each. Yocto's sstate rebuilds only what changed;
# Buildroot often rebuilds a great deal more.
```

**Write the comparison table and a recommendation for a specific product.** That artifact is
directly a staff-level deliverable and is asked about in embedded interviews.

---

## 3. Mastery drills

1. Build `core-image-minimal` for three machines (`qemux86-64`, `qemuarm64`, a real board) and
   boot each. Document what changed between them and where each change came from.

2. Read `meta/conf/bitbake.conf` completely and write a one-page summary of the variable
   categories it establishes.

3. Create a layer with five recipes exercising: a plain Makefile, autotools, CMake, a Python
   package, and a systemd service. Get each to build and install correctly.

4. For a recipe that fails to build, use `devshell` to reproduce the failure interactively and
   fix it without leaving the shell. Then encode the fix as a patch in the recipe.

5. Use `bitbake -e | grep -B20 '# $VAR'` to trace five variables to their origin. Explain the
   override mechanism from what you find.

6. Set up a shared sstate mirror and a hash-equivalence server for a team. Measure a new
   developer's first-build time with and without them.

7. Reduce `core-image-minimal` by 30% by removing `DISTRO_FEATURES`. Document what each
   removal cost in packages and build time.

8. Use `bitbake -g` to produce the dependency graph for an image, and find the recipe with the
   most dependents. Explain why a change to it is expensive.

9. Write a recipe for a real third-party project of your choice, from upstream source, with
   correct `LICENSE`, `LIC_FILES_CHKSUM`, `DEPENDS`, `RDEPENDS`, and packaging splits.

10. Deliberately create each of these failures and learn the error message: a missing
    `DEPENDS`, a missing `RDEPENDS`, a wrong `LIC_FILES_CHKSUM`, files installed but not
    packaged (`installed-vs-shipped`), and a `.bbappend` whose file is not found.

11. Compare Yocto, Buildroot, and a Debian rootfs for the same product. Produce the decision
    matrix with real numbers.

12. Write the onboarding guide for your team: how to set up, build, add a recipe, add a patch,
    and debug a failure. Test it on someone who has never used Yocto.

---

## 4. Further reading

**Official documentation — the reference, not the tutorial**
- **Yocto Project Reference Manual** — the variable glossary is the part you will use daily
- **Yocto Project Development Tasks Manual** — recipe-writing recipes
- **BitBake User Manual** — the language, the operators, the execution model
- **Yocto Project Overview and Concepts Manual** — read this first if the official docs are
  your entry point
- `meta/conf/documentation.conf` — every variable, documented, in the tree

**Books**
- Rudolf Streif, ***Embedded Linux Systems with the Yocto Project*** — the best book; dated
  in details, sound on concepts
- Otavio Salvador & Daiane Angolini, *Embedded Linux Development Using Yocto Project* —
  more current, more practical
- Chris Simmonds, *Mastering Embedded Linux Programming*, 3rd ed., ch. 6 — the best concise
  introduction, and it compares Yocto and Buildroot fairly

**Training**
- **Bootlin's Yocto training materials** — free, excellent, and the slides are better than
  most books
- The Yocto Project's own tutorial videos and `LFD460` (Linux Foundation)

**Community**
- `lists.yoctoproject.org` (`yocto@`, `poky@`) and `lists.openembedded.org`
  (`openembedded-core@`, `openembedded-devel@`)
- `#yocto` on Libera Chat — unusually helpful
- The OpenEmbedded layer index: `layers.openembedded.org` — **find a layer before writing
  one**

**Cross-references**
- Ch. 92 §T.10 — cross-compilation, sysroots, and the build/host/target distinction
- Ch. 97 — the BitBake language in detail
- Ch. 98 — kernel and BSP recipes
- Ch. 99 — SDK, licensing, CVE, SBOM, OTA, reproducibility
- Ch. 100 — Buildroot and the alternatives
- Ch. 88 — patch management, which Yocto's `SRC_URI` patches implement

→ Next: [97-yocto-2-recipes.md](97-yocto-2-recipes.md)

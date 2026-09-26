# Chapter 97 — Yocto II: BitBake Language, Recipes, Layers, Classes

> Chapter 96 built the mental model. This is the reference chapter: the language in detail,
> the recipe patterns you will write hundreds of times, how classes work, and how to debug a
> build when it goes wrong — which it will, constantly, and in ways whose error messages are
> not immediately helpful.

---

## Theory & First Principles

### T.0 — Start here: a complete recipe, annotated

This builds, packages, and installs a piece of software for a cross-compiled target. It is
fourteen lines.

```bash
SUMMARY  = "A tiny daemon"                                   # ① metadata
LICENSE  = "MIT"                                             # ② LEGALLY REQUIRED
LIC_FILES_CHKSUM = "file://LICENSE;md5=0835ade698e0bcf8506e" # ③ and CHECKED

SRC_URI  = "git://github.com/example/tinyd.git;branch=main;protocol=https \
            file://0001-fix-cross-compile.patch"             # ④ source + patches
SRCREV   = "a1b2c3d4e5f6..."                                 # ⑤ EXACT commit

DEPENDS  = "zlib openssl"          # ⑥ build-time deps (headers, libs)
RDEPENDS:${PN} = "bash"            # ⑦ RUN-time deps (a different graph!)

inherit autotools systemd          # ⑧ CLASSES: inherited task implementations

SYSTEMD_SERVICE:${PN} = "tinyd.service"
```

**Nine annotations, and five of them teach something non-obvious:**

②③ **The license is not documentation — it is a checked input.** `LIC_FILES_CHKSUM` is the
md5 of the actual license text in the source. If upstream silently changes their license, the
build **fails**. That is a legal-compliance mechanism implemented as a build dependency, and
it is the reason Yocto can emit a defensible license manifest for a shipped product.

⑤ **`SRCREV` is an exact commit, never a branch.** "Track main" is not reproducible; the same
recipe must produce the same bits in 2031 (Ch. 96 §T.0).

⑥⑦ **`DEPENDS` and `RDEPENDS` are two different graphs, and conflating them is the most common
beginner error.** `DEPENDS` is what must be *built* first (headers and link libraries, on the
build host, for the target architecture). `RDEPENDS` is what must be *installed* on the target
at runtime. They overlap but are not the same: you build against `openssl` headers, and you
run against the `libssl` package; you may `RDEPENDS` on `bash` while never building it.
**Build-time and run-time dependency graphs are genuinely distinct, and every package manager
that pretends otherwise has a bug.**

⑧ **`inherit` is the actual power of the system.** `autotools` supplies working
`do_configure`, `do_compile`, and `do_install` tasks that already know the cross-compilation
sysroot, the target triplet, and the install prefix. **You wrote zero build logic.** That is
Ch. 89 §T.0's narrow-waist principle applied to a build system: the class is the reusable
mechanism, the recipe is the policy.

**Now the one thing that trips absolutely everyone — override syntax.** Yocto variables are
not ordinary variables; they carry *deferred operations*:

```bash
FOO  = "a"          # lazy: expanded when USED
FOO := "a"          # immediate: expanded NOW
FOO ?= "a"          # only if not already set
FOO ??= "a"         # weaker default; loses to ?=

FOO:append  = " b"  # appended AFTER all parsing. NO space added -- put one in.
FOO:prepend = "b "
FOO += "b"          # appended DURING parsing, WITH a space. Different timing!
FOO:remove  = "a"

FOO:arm     = "..."        # applies only when building for arm
FOO:class-native = "..."   # only for the -native (build-host) variant
```

**`+=` versus `:append` is the single most common source of "why is my override ignored?"**
The difference is *when* they apply: `+=` happens while the file is parsed, so a later `=` in
a `.bbappend` wipes it out; `:append` is applied after all parsing is complete, so it always
wins. **The syntax encodes evaluation order**, and once you hold that, the rest of BitBake's
language stops being arbitrary.

**And `.bbappend` is the mechanism that makes layering work** (Ch. 96 §T.0): you never edit
`meta/recipes-core/foo/foo_1.2.bb`; you add `meta-mine/recipes-core/foo/foo_%.bbappend` with
your additions. **Extend, never fork** — the same discipline as Ch. 88 §T.0's mainline-first
rule.

```bash
bitbake -e tinyd | grep -E '^(S|B|D|WORKDIR|DEPENDS|RDEPENDS)='
bitbake -c devshell tinyd          # a shell in the exact cross-build environment
bitbake -c listtasks tinyd
bitbake-layers show-appends tinyd  # who is overriding this recipe, and from where
```

---

### T.1 — The language, precisely

BitBake metadata is not shell and is not Python, though it embeds both. Four syntactic
elements:

```bash
# 1. Variable assignment
FOO = "bar"

# 2. Shell functions -- run by /bin/sh, with the datastore exported as env vars
do_compile() {
    ${CC} ${CFLAGS} -o app app.c
}

# 3. Python functions -- run in-process, with `d` as the datastore
python do_something() {
    version = d.getVar('PV')
    bb.note("building version %s" % version)
    d.setVar('MYVAR', 'value')
}

# 4. Inline Python -- evaluated during variable expansion
FOO = "${@bb.utils.contains('DISTRO_FEATURES', 'systemd', 'yes', 'no', d)}"
BAR = "${@d.getVar('PV').split('.')[0]}"
```

**The `${@...}` inline-Python form is everywhere** and is the main source of unreadable
recipes. The most common helpers:

```bash
${@bb.utils.contains('VAR', 'item', 'if_yes', 'if_no', d)}
${@bb.utils.contains_any('VAR', 'a b', 'yes', 'no', d)}
${@bb.utils.filter('DISTRO_FEATURES', 'systemd sysvinit', d)}
${@oe.utils.conditional('VAR', 'value', 'if_equal', 'else', d)}
${@d.getVar('PV').replace('.', '_')}
```

### T.2 — Variable operators, with the trap explained

| Operator | Applied | Notes |
|---|---|---|
| `=` | lazy expansion at use | the default |
| `:=` | expanded **immediately** | use for `${THISDIR}` and anything position-dependent |
| `?=` | default if unset | |
| `??=` | weak default; **last one parsed wins** | |
| `+=` / `=+` | append/prepend with a space, **at parse time** | order-dependent |
| `.=` / `=.` | append/prepend, no space, at parse time | |
| `:append` / `:prepend` | **after all parsing** | order-independent. Use this |
| `:remove` | remove a whitespace-delimited word, after parsing | |

**The trap, spelled out:**

```bash
# base.bb
FOO = "a"
FOO += "b"          # -> "a b"

# override.bbappend
FOO = "z"           # parsed AFTER, so -> "z". The += is lost.

# Whereas:
FOO:append = " y"   # applied after EVERYTHING -> "z y"
```

`+=` is applied in parse order; `:append` is applied in a final pass. In a `.bbappend` — where
you do not control parse order — **always use `:append`**. And note the leading space:
`:append` concatenates literally, so `FOO:append = "bar"` gives `"zbar"`, not `"z bar"`.

`:remove` is the odd one: it operates on whitespace-separated words and is applied last, so it
beats every append.

```bash
RDEPENDS:${PN} = "a b c"
RDEPENDS:${PN}:append = " d"
RDEPENDS:${PN}:remove = "b"
# -> "a c d"
```

### T.3 — Overrides

```bash
OVERRIDES = "${TARGET_OS}:${TRANSLATED_TARGET_ARCH}:pn-${PN}:${MACHINEOVERRIDES}:${DISTROOVERRIDES}:${CLASSOVERRIDE}:forcevariable"
```

A colon-separated list, **low priority to high**. A variable suffixed with an override that
appears in `OVERRIDES` replaces the base variable; the highest-priority match wins.

```bash
SRC_URI = "file://base.patch"
SRC_URI:arm = "file://arm.patch"                 # replaces, for ARM
SRC_URI:append:mymachine = " file://extra.patch" # appends, for one machine
SRC_URI:remove:qemuall = "file://base.patch"     # removes, for any qemu machine

# Commonly used override namespaces:
#   pn-<recipename>     this recipe only
#   class-native        the -native variant
#   class-target        the target variant
#   class-nativesdk
#   <MACHINE>           the machine name
#   <MACHINEOVERRIDES>  a chain: e.g. "beaglebone:ti-soc:armv7a:arm"
#   <DISTRO>            the distro name
#   libc-glibc / libc-musl
#   forcevariable       highest priority; use to win unconditionally
```

**`MACHINEOVERRIDES` is the mechanism that makes SoC-family BSPs work.** A machine `.conf`
sets `MACHINEOVERRIDES = "ti-soc:${MACHINE}"`, and a recipe can then say
`SRC_URI:append:ti-soc = ...` to affect every TI board at once.

### T.4 — Anatomy of a recipe

```bash
# --- Metadata (required) ---
SUMMARY  = "One line, under 72 chars"
DESCRIPTION = "Longer prose."
HOMEPAGE = "https://example.com"
SECTION  = "libs"
LICENSE  = "GPL-2.0-only & MIT"          # SPDX identifiers; & = AND, | = OR
LIC_FILES_CHKSUM = "file://COPYING;md5=751419260aa954499f7abaabaa882bbe \
                    file://src/lib.c;beginline=3;endline=20;md5=..."

# --- Sources ---
SRC_URI = "https://example.com/${BPN}-${PV}.tar.gz \
           git://github.com/x/y.git;protocol=https;branch=main \
           file://0001-fix-build.patch \
           file://config.cfg;subdir=cfg \
          "
SRC_URI[sha256sum] = "abc123..."
SRCREV = "a1b2c3d4e5f6..."               # for git: an exact commit. NOT a branch
PV = "1.0+git${SRCPV}"                   # version including the revision

# --- Layout ---
S = "${WORKDIR}/${BPN}-${PV}"            # where the sources ended up
B = "${WORKDIR}/build"                   # out-of-tree build dir (autotools sets this)

# --- Dependencies ---
DEPENDS  = "zlib openssl virtual/kernel"      # BUILD time
RDEPENDS:${PN} = "bash"                       # RUN time
RRECOMMENDS:${PN} = "extra-tool"              # installed unless excluded
RSUGGESTS:${PN} = "optional-thing"
RCONFLICTS:${PN} = "old-package"
RPROVIDES:${PN} = "virtual-thing"
RREPLACES:${PN} = "old-package"

# --- Build ---
inherit autotools pkgconfig systemd
EXTRA_OECONF = "--enable-foo --disable-bar"
EXTRA_OEMAKE = "V=1"
EXTRA_OECMAKE = "-DBUILD_TESTS=OFF"
PACKAGECONFIG ??= "ssl ${@bb.utils.filter('DISTRO_FEATURES', 'systemd', d)}"
PACKAGECONFIG[ssl]     = "--with-ssl,--without-ssl,openssl,"
PACKAGECONFIG[systemd] = "--with-systemd,--without-systemd,systemd,"
#                          ^enable       ^disable          ^DEPENDS ^RDEPENDS

# --- Packaging ---
PACKAGES =+ "${PN}-extras"
FILES:${PN}        += "${datadir}/myapp"
FILES:${PN}-extras  = "${bindir}/extra-tool"
FILES:${PN}-dev    += "${libdir}/myapp/*.so"
CONFFILES:${PN}     = "${sysconfdir}/myapp.conf"   # preserved on upgrade

# --- Systemd ---
SYSTEMD_SERVICE:${PN} = "myapp.service"
SYSTEMD_AUTO_ENABLE = "enable"

# --- Variants ---
BBCLASSEXTEND = "native nativesdk"
```

**The details that cause the most grief:**

- **`LIC_FILES_CHKSUM` is mandatory and is a feature, not a nuisance.** It fails the build
  when upstream changes the licence text, which is exactly when you need to know.
- **`SRCREV` must be a full SHA, never a branch name.** A branch is not reproducible.
- **`S` must point at the unpacked source.** If the tarball extracts to a differently-named
  directory, set it. "No such file or directory" at `do_configure` is almost always this.
- **`PACKAGECONFIG` is the right way to make features optional.** The five comma-separated
  fields are: enable-flag, disable-flag, build-deps, runtime-deps, and conflicting features.

### T.5 — The classes you will use

| Class | Provides |
|---|---|
| `autotools` | `./configure && make && make install`, with the right cross flags |
| `cmake` | CMake with a generated toolchain file |
| `meson` | Meson with a cross file |
| `setuptools3` / `python_setuptools_build_meta` | Python packages |
| `cargo` / `cargo_bin` | Rust |
| `go` / `go-mod` | Go |
| `kernel` | Everything for building a kernel (Ch. 98) |
| `module` | Out-of-tree kernel modules |
| `image` | Rootfs construction |
| `core-image` | `image` + `IMAGE_FEATURES` handling |
| `systemd` | Unit installation and enablement |
| `update-rc.d` | SysV init scripts |
| `useradd` | Creating users at rootfs time |
| `pkgconfig` | `pkg-config` setup |
| `native` / `nativesdk` / `cross` | Build-variant machinery |
| `bin_package` | Install a prebuilt binary package |
| `externalsrc` | Build from a local directory instead of `SRC_URI` |
| `devtool` support classes | |
| `cve-check` | CVE scanning (Ch. 99) |
| `create-spdx` | SBOM generation (Ch. 99) |
| `rm_work` | Delete `WORKDIR` after building (saves enormous disk) |
| `archiver` | Archive sources for GPL compliance (Ch. 99) |
| `insane` | **QA checks** — the source of most "why did my build fail" |

**Read `meta/classes-recipe/autotools.bbclass` once.** It is the best available explanation of
how a class works: it defines `do_configure`, `do_compile`, and `do_install`, sets up the
cross environment, and handles the `B != S` case.

### T.6 — The QA checks (`insane.bbclass`)

Yocto refuses to build things that are wrong. The messages are terse; here is what they mean.

| Error | Cause | Fix |
|---|---|---|
| `installed-vs-shipped` | Files in `${D}` not matched by any `FILES` | add to `FILES:${PN}` or delete in `do_install` |
| `file-rdeps` | A package needs something not in `RDEPENDS` | add the `RDEPENDS` |
| `build-deps` | A `.so` links a library not in `DEPENDS` | add the `DEPENDS` |
| `dev-so` | A `.so` symlink in the main package | it belongs in `-dev` |
| `staticdev` | A `.a` outside `-staticdev` | fix `FILES` |
| `ldflags` | The binary did not get `${LDFLAGS}` | pass them through in `do_compile` |
| `buildpaths` | **A host build path leaked into the binary** | a reproducibility failure; see Ch. 99 |
| `host-user-contaminated` | Files owned by the build user | `chown` in `do_install` |
| `textrel` | Text relocations (not built `-fPIC`) | build with `-fPIC` |
| `already-stripped` | The recipe stripped its own binaries | remove the strip from the build |
| `useless-rpaths` | An `RPATH` pointing at a standard dir | usually harmless; often a libtool artefact |
| `version-going-backwards` | The new package version sorts lower | fix `PV`/`PR` |

```bash
# See what is checked:
bitbake -e | grep '^WARN_QA=\|^ERROR_QA='

# Downgrade a check while you work (do NOT ship like this):
ERROR_QA:remove = "buildpaths"
WARN_QA:append = " buildpaths"

# Per-recipe:
INSANE_SKIP:${PN} += "dev-so"
```

**`buildpaths` deserves attention:** it fires when `/home/you/yocto/build/...` appears inside
a shipped binary. That is a reproducibility failure (the output depends on where you built
it) and often an information leak. The fixes are `-ffile-prefix-map` and
`--remap-path-prefix`, which OE sets by default — a recipe that overrides `CFLAGS` and drops
them will trip this.

### T.7 — `devtool`: the development loop

The tool that makes Yocto tolerable for active development.

```bash
# --- Work on an existing recipe ---
devtool modify busybox
#   -> extracts the source to workspace/sources/busybox as a GIT REPO
#      with each patch as a commit, and creates a workspace layer.

cd workspace/sources/busybox
# ... edit, commit ...
devtool build busybox
devtool deploy-target busybox root@192.168.1.10   # scp it to the device
# ... iterate ...
devtool finish busybox ../meta-mylayer
#   -> turns your commits back into .patch files in the recipe's directory
#      and updates the .bbappend. THIS is the killer feature.

# --- Create a recipe from upstream ---
devtool add myapp https://github.com/x/myapp.git
#   -> inspects the source, guesses the build system, licence, and
#      dependencies, and writes a starter recipe.
devtool build myapp
devtool finish myapp ../meta-mylayer

# --- Upgrade a recipe ---
devtool upgrade busybox -V 1.37.0
#   -> fetches the new version, rebases the patches, reports conflicts.
devtool finish busybox ../meta-mylayer

# --- Build against a local checkout, no patches ---
devtool modify -n myapp /path/to/my/existing/source

# --- Status and cleanup ---
devtool status
devtool reset myapp
```

**`devtool modify` + edit + `devtool finish` is the correct way to patch upstream software.**
Hand-writing `.patch` files and `SRC_URI` entries is error-prone; letting git produce them is
not. And the resulting patches have proper commit messages, which matters for Ch. 88's
upstreaming discipline.

### T.8 — Debugging a build failure

The procedure, in order.

```bash
# 1. READ THE ERROR. BitBake prints the failing task and the log path.
#    ERROR: myapp-1.0-r0 do_compile: ... see
#    /path/tmp/work/.../temp/log.do_compile

# 2. Read the log, and the SCRIPT that produced it.
less tmp/work/*/myapp/*/temp/log.do_compile
less tmp/work/*/myapp/*/temp/run.do_compile    # <- the generated script

# 3. Reproduce interactively.
bitbake -c devshell myapp
#    You are in ${S} with the full cross environment.
echo $CC $CFLAGS $LDFLAGS
./configure --help
make V=1 2>&1 | tail -40

# 4. Python-level inspection.
bitbake -c devpyshell myapp
>>> d.getVar('EXTRA_OECONF')
>>> d.getVar('DEPENDS')

# 5. Is the variable what you think?
bitbake -e myapp | grep '^CFLAGS='
bitbake -e myapp | grep -B20 '^# \$CFLAGS'     # WHO SET IT

# 6. Force a clean rebuild of just this recipe.
bitbake -c cleansstate myapp && bitbake myapp

# 7. Is it a sysroot problem?
ls tmp/work/*/myapp/*/recipe-sysroot/usr/include/
#    Missing header -> missing DEPENDS.

# 8. Verbose everything.
bitbake -v -DDD myapp 2>&1 | tee /tmp/build.log

# 9. Task ordering / dependency questions.
bitbake -g myapp && grep myapp task-depends.dot
```

**Step 2 is the one people skip.** `run.do_compile` is the exact shell script BitBake
generated and ran, with every variable expanded. Reading it answers "why did it pass that
flag" instantly.

### T.9 — Multiconfig

Building several configurations in one BitBake invocation — a firmware image plus a host
tool, or two machines sharing sstate.

```bash
# build/conf/multiconfig/board.conf
MACHINE = "myboard"
TMPDIR = "${TOPDIR}/tmp-board"

# build/conf/multiconfig/host.conf
MACHINE = "qemux86-64"
TMPDIR = "${TOPDIR}/tmp-host"

# build/conf/local.conf
BBMULTICONFIG = "board host"

# Build both:
bitbake mc:board:core-image-minimal mc:host:my-host-tool

# Cross-multiconfig dependency (e.g. the image needs the host tool):
do_image[mcdepends] = "mc::host:my-host-tool:do_build"
```

Used for: multi-SoC products (an application processor plus an MCU firmware blob), building
a signing tool for the host alongside the image, and building an initramfs with a different
configuration from the main rootfs.

### T.10 — Recipe style and maintenance

Things that separate a maintainable layer from a liability:

1. **Never copy a recipe to modify it.** Use `.bbappend`. Ch. 96 §T.3.
2. **Use `%` in `.bbappend` filenames** so they survive version bumps — unless the change is
   genuinely version-specific, in which case pin it deliberately.
3. **Patches get real commit messages and an `Upstream-Status:` header.** This is an OE
   convention and it is the mechanism that keeps a patch queue from rotting:
   ```
   Upstream-Status: Submitted [https://lore.kernel.org/...]
   Upstream-Status: Backport [commit abc123]
   Upstream-Status: Pending
   Upstream-Status: Inappropriate [configuration]
   Upstream-Status: Denied [maintainer rejected it, see link]
   ```
   `Pending` with no link means "nobody has tried to upstream this," and a layer full of
   `Pending` is Ch. 88's problem accumulating.
4. **Pin `SRCREV`, never a branch.**
5. **Set `LIC_FILES_CHKSUM` honestly**, including per-file checksums when a project has mixed
   licensing.
6. **`PACKAGECONFIG` for optional features**, not `if` statements in `do_configure`.
7. **Keep machine-specific things in the BSP layer**, distro policy in the distro, and
   product content in the product layer. Mixing them is what makes a layer un-reusable.
8. **Run `yocto-check-layer`** before claiming a layer is compatible.

---

## 1. Internals

### Key files to read

| Path | Why |
|---|---|
| `meta/conf/bitbake.conf` | Every base variable, defined |
| `meta/conf/documentation.conf` | Every variable, documented |
| `meta/classes-recipe/autotools.bbclass` | The model class; read it once |
| `meta/classes-recipe/cmake.bbclass` | Compare with autotools |
| `meta/classes-global/insane.bbclass` | Every QA check and its message |
| `meta/classes-global/package.bbclass` | How `do_package` splits things |
| `meta/classes-recipe/image.bbclass` | Rootfs construction |
| `meta/classes-recipe/kernel.bbclass` | Ch. 98 |
| `meta/lib/oe/` | The Python helper library (`oe.utils`, `oe.path`, `oe.package`) |
| `bitbake/lib/bb/` | The engine: `data.py`, `parse/`, `runqueue.py`, `fetch2/` |

### The `SRC_URI` fetchers

```bash
# http/https/ftp -- requires a checksum
SRC_URI = "https://example.com/foo-${PV}.tar.gz"
SRC_URI[sha256sum] = "..."

# git
SRC_URI = "git://github.com/x/y.git;protocol=https;branch=main;tag=v${PV}"
SRCREV = "full-sha1"
# Or track a branch's tip (NOT reproducible; use only during development):
SRCREV = "${AUTOREV}"

# git with submodules
SRC_URI = "gitsm://github.com/x/y.git;protocol=https;branch=main"

# Local files -- searched along FILESPATH
SRC_URI = "file://config.cfg"
SRC_URI = "file://0001-patch.patch;striplevel=2"
SRC_URI = "file://dir;subdir=target-subdir"

# Other fetchers
SRC_URI = "svn://...;module=trunk;protocol=https"
SRC_URI = "npm://registry.npmjs.org;package=foo;version=${PV}"
SRC_URI = "crate://crates.io/serde/1.0.0"
SRC_URI = "go://github.com/x/y;version=v1.0.0"

# Useful parameters
;name=foo          # for multiple sources with separate checksums
;destsuffix=path   # where to unpack
;unpack=0          # do not extract
;apply=no          # a file ending in .patch that is not a patch
;downloadfilename= # rename the downloaded file
```

**`FILESPATH` determines where `file://` is searched**, and it is built from `FILESEXTRAPATHS`
plus the standard directories: `${BPN}-${PV}/`, `${BPN}/`, `files/`, with machine and
override subdirectories. This is why
`SRC_URI:append:myboard = " file://board.cfg"` finds `files/myboard/board.cfg` automatically.

### Writing a class

```bash
# meta-mylayer/classes/myfeature.bbclass
#
# Adds a signature file to every package that inherits this.

MYFEATURE_KEY ?= "${THISDIR}/files/signing.key"

do_sign_package() {
    if [ -f "${MYFEATURE_KEY}" ]; then
        openssl dgst -sha256 -sign "${MYFEATURE_KEY}" \
            -out ${D}${bindir}/${PN}.sig ${D}${bindir}/${PN}
    else
        bbwarn "No signing key at ${MYFEATURE_KEY}; not signing"
    fi
}

# Insert the task into the sequence.
addtask sign_package after do_install before do_package

# Declare what the task depends on, for correct sstate signatures.
do_sign_package[depends] += "openssl-native:do_populate_sysroot"
do_sign_package[vardeps] += "MYFEATURE_KEY"

FILES:${PN} += "${bindir}/${PN}.sig"
```

The task flags that matter:

| Flag | Meaning |
|---|---|
| `[depends]` | Build-time task dependencies |
| `[rdepends]` | Runtime dependencies for this task |
| `[vardeps]` | **Extra variables whose change should invalidate the signature** |
| `[vardepsexclude]` | Variables to *ignore* for the signature (e.g. `DATETIME`) |
| `[nostamp]` | Always rerun; never cached |
| `[noexec]` | A placeholder task |
| `[dirs]` | Directories to create before running |
| `[cleandirs]` | Directories to wipe before running |
| `[fakeroot]` | Run under pseudo (for file ownership) |

**`[vardeps]` and `[vardepsexclude]` are how you control rebuilds.** If a task reads a
variable indirectly (through a shell function, say), BitBake cannot see the dependency and
will not rebuild when it changes — add `[vardeps]`. Conversely, if a task embeds a timestamp,
add `[vardepsexclude]` or every build rebuilds everything.

---

## 2. Practice

### Lab 97.1 — Recipe patterns, one of each

```bash
mkdir -p ~/yocto/meta-mylayer/recipes-examples
cd ~/yocto/meta-mylayer/recipes-examples
```

**(a) Autotools, from a release tarball**

```bash
# autotools-example/libexample_1.2.3.bb
SUMMARY = "An autotools library"
LICENSE = "LGPL-2.1-or-later"
LIC_FILES_CHKSUM = "file://COPYING;md5=4fbd65380cdd255951079008b364516c"

SRC_URI = "https://example.com/libexample-${PV}.tar.xz \
           file://0001-fix-cross-build.patch \
          "
SRC_URI[sha256sum] = "0000000000000000000000000000000000000000000000000000000000000000"

inherit autotools pkgconfig

DEPENDS = "zlib"

PACKAGECONFIG ??= "ssl"
PACKAGECONFIG[ssl]   = "--with-ssl,--without-ssl,openssl,"
PACKAGECONFIG[tests] = "--enable-tests,--disable-tests,cmocka,"

EXTRA_OECONF = "--disable-static"

BBCLASSEXTEND = "native nativesdk"
```

**(b) CMake, from git**

```bash
# cmake-example/mytool_git.bb
SUMMARY = "A CMake tool"
LICENSE = "Apache-2.0"
LIC_FILES_CHKSUM = "file://LICENSE;md5=89aea4e17d99a7cacdbeed46a0096b10"

SRC_URI = "git://github.com/example/mytool.git;protocol=https;branch=main"
SRCREV = "a1b2c3d4e5f60718293a4b5c6d7e8f9012345678"
PV = "1.0+git${SRCPV}"

S = "${WORKDIR}/git"

inherit cmake pkgconfig

DEPENDS = "boost jsoncpp"

EXTRA_OECMAKE = " \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_TESTING=OFF \
    -DENABLE_DOCS=OFF \
"
```

**(c) A plain Makefile — the one that trips people up**

```bash
# makefile-example/simpletool_1.0.bb
SUMMARY = "A tool with a hand-written Makefile"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

SRC_URI = "file://Makefile file://main.c"
S = "${WORKDIR}/sources"
UNPACKDIR = "${S}"

do_compile() {
    # oe_runmake passes CC, CFLAGS, LDFLAGS correctly. Using bare `make`
    # is the single most common recipe bug: it builds for the HOST.
    oe_runmake
}

do_install() {
    install -d ${D}${bindir}
    install -m 0755 ${B}/simpletool ${D}${bindir}/
}
```

with a Makefile that actually honours the environment:

```makefile
# The Makefile MUST use ?= or the environment, or cross-compilation fails.
CC      ?= gcc
CFLAGS  ?= -O2
LDFLAGS ?=

simpletool: main.c
	$(CC) $(CFLAGS) $(LDFLAGS) -o $@ $<
```

**(d) Python**

```bash
# python-example/python3-myapp_1.0.bb
SUMMARY = "A Python application"
LICENSE = "BSD-3-Clause"
LIC_FILES_CHKSUM = "file://LICENSE;md5=..."

SRC_URI = "git://github.com/example/myapp.git;protocol=https;branch=main"
SRCREV = "..."
S = "${WORKDIR}/git"

inherit python_setuptools_build_meta

RDEPENDS:${PN} = " \
    python3-requests \
    python3-json \
    python3-logging \
"
```

**(e) A systemd service with a user**

```bash
# service-example/myservice_1.0.bb
SUMMARY = "A daemon with its own user"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

SRC_URI = "file://myservice.c file://myservice.service file://myservice.conf"
S = "${WORKDIR}/sources"
UNPACKDIR = "${S}"

inherit systemd useradd

USERADD_PACKAGES = "${PN}"
USERADD_PARAM:${PN} = "--system --no-create-home --shell /bin/false \
                       --user-group myservice"

SYSTEMD_SERVICE:${PN} = "myservice.service"
SYSTEMD_AUTO_ENABLE = "enable"

do_compile() {
    ${CC} ${CFLAGS} ${LDFLAGS} -o myservice ${S}/myservice.c
}

do_install() {
    install -d ${D}${bindir} ${D}${sysconfdir} ${D}${systemd_system_unitdir}
    install -m 0755 myservice               ${D}${bindir}/
    install -m 0644 ${S}/myservice.conf     ${D}${sysconfdir}/
    install -m 0644 ${S}/myservice.service  ${D}${systemd_system_unitdir}/
}

CONFFILES:${PN} = "${sysconfdir}/myservice.conf"
FILES:${PN} += "${systemd_system_unitdir}/myservice.service"
```

Build all five and inspect the packaging of each:

```bash
cd ~/yocto/build
for r in libexample mytool simpletool python3-myapp myservice; do
    bitbake $r && oe-pkgdata-util list-pkg-files -p $r
done
```

### Lab 97.2 — Break every QA check on purpose

The fastest way to learn the error messages.

```bash
# Start from the simpletool recipe and introduce one fault at a time.

# 1. installed-vs-shipped
do_install:append() {
    install -d ${D}${datadir}/orphan
    touch ${D}${datadir}/orphan/file
}
#   ERROR: ... installed and not shipped in any package
#   FIX:   FILES:${PN} += "${datadir}/orphan"

# 2. ldflags -- the binary did not get ${LDFLAGS}
do_compile() { ${CC} ${CFLAGS} -o simpletool ${S}/main.c; }   # LDFLAGS dropped
#   ERROR: ... does not have GNU_HASH (didn't pass LDFLAGS?)

# 3. build-deps -- link something not in DEPENDS
do_compile() { ${CC} ${CFLAGS} ${LDFLAGS} -o simpletool ${S}/main.c -lz; }
#   ERROR: ... requires libz.so.1, but no providers found in RDEPENDS
#   FIX:   DEPENDS += "zlib"

# 4. buildpaths -- a reproducibility failure
do_compile() {
    ${CC} ${CFLAGS} ${LDFLAGS} -DBUILD_DIR=\"${B}\" -o simpletool ${S}/main.c
}
#   ERROR: ... contains reference to TMPDIR
#   FIX:   do not embed paths; or use -ffile-prefix-map

# 5. dev-so
do_install:append() { ln -s libfoo.so.1 ${D}${libdir}/libfoo.so; }
#   ERROR: ... non -dev/-dbg/nativesdk- package contains symlink .so

# 6. already-stripped
do_compile:append() { ${STRIP} simpletool; }
#   ERROR: ... is already stripped

# For each: read the message, find the check in insane.bbclass, fix it.
grep -n 'installed-vs-shipped\|buildpaths\|dev-so' \
    ~/yocto/poky/meta/classes-global/insane.bbclass
```

### Lab 97.3 — `devtool`, the real workflow

```bash
cd ~/yocto/build

# --- Patch an upstream package properly ---
devtool modify busybox
cd workspace/sources/busybox
git log --oneline | head              # existing patches, as commits

# Make a change.
$EDITOR shell/ash.c
git add -A
git commit -s -m "ash: increase the history size

Our product's operators want more shell history than the default 999
entries. Bump it to 5000.

Upstream-Status: Inappropriate [product-specific configuration]
"

cd ~/yocto/build
devtool build busybox
devtool deploy-target busybox root@192.168.1.10   # if you have a target

# Turn the commit into a patch in your layer:
devtool finish busybox ~/yocto/meta-mylayer
ls ~/yocto/meta-mylayer/recipes-core/busybox/busybox/
cat ~/yocto/meta-mylayer/recipes-core/busybox/busybox_%.bbappend
#   -> the patch, with your commit message and Upstream-Status, plus
#      a correctly-written bbappend. You wrote none of the boilerplate.

# --- Create a recipe from scratch ---
devtool add hello-world https://github.com/example/hello.git
cat workspace/recipes/hello-world/hello-world_git.bb
#   devtool guessed the build system, licence, and dependencies.
devtool build hello-world
devtool finish hello-world ~/yocto/meta-mylayer

# --- Version upgrade ---
devtool upgrade busybox -V 1.37.0
#   Rebases every patch onto the new version and reports conflicts.
devtool finish busybox ~/yocto/meta-mylayer

devtool status
devtool reset --all
```

### Lab 97.4 — Write a class

```bash
mkdir -p ~/yocto/meta-mylayer/classes
cat > ~/yocto/meta-mylayer/classes/product-hardening.bbclass <<'EOF'
# Applies our product's hardening policy to any recipe that inherits it.
#
# Usage:  inherit product-hardening

# Compiler hardening (Ch. 102 §T.6).
CFLAGS:append  = " -D_FORTIFY_SOURCE=3 -fstack-protector-strong \
                   -fstack-clash-protection -fcf-protection=full"
LDFLAGS:append = " -Wl,-z,relro,-z,now -Wl,-z,noexecstack"

# Refuse to ship anything that is setuid without an explicit exception.
PRODUCT_SETUID_ALLOWED ?= ""

python do_check_setuid() {
    import os, stat
    d_dir = d.getVar('D')
    allowed = (d.getVar('PRODUCT_SETUID_ALLOWED') or '').split()
    bad = []
    for root, dirs, files in os.walk(d_dir):
        for f in files:
            p = os.path.join(root, f)
            if os.path.islink(p):
                continue
            mode = os.lstat(p).st_mode
            if mode & (stat.S_ISUID | stat.S_ISGID):
                rel = p[len(d_dir):]
                if rel not in allowed:
                    bad.append(rel)
    if bad:
        bb.fatal("setuid/setgid files not in PRODUCT_SETUID_ALLOWED: %s"
                 % ", ".join(bad))
}
addtask check_setuid after do_install before do_package
do_check_setuid[vardeps] += "PRODUCT_SETUID_ALLOWED"

# Verify the hardening actually took effect.
python do_verify_hardening() {
    import subprocess, os
    # (a real implementation would use pyelftools; this is the shape)
    bb.note("hardening flags: %s" % d.getVar('LDFLAGS'))
}
addtask verify_hardening after do_compile before do_install
EOF

# Use it:
cat >> ~/yocto/meta-mylayer/recipes-examples/makefile-example/simpletool_1.0.bb <<'EOF'

inherit product-hardening
EOF

cd ~/yocto/build
bitbake -c cleansstate simpletool && bitbake simpletool
# Verify:
readelf -d tmp/work/*/simpletool/*/image/usr/bin/simpletool | grep -E 'BIND_NOW|FLAGS'
```

### Lab 97.5 — Understand rebuild cascades

```bash
cd ~/yocto/build

# 1. Baseline.
bitbake core-image-minimal
bitbake -S none core-image-minimal

# 2. A change that should rebuild ONE recipe.
echo 'EXTRA_OECONF:pn-busybox:append = " --foo"' >> conf/local.conf
bitbake-whatchanged core-image-minimal | tee /tmp/change1.txt
wc -l /tmp/change1.txt

# 3. A change that rebuilds EVERYTHING.
echo 'CFLAGS:append = " -DFOO"' >> conf/local.conf
bitbake-whatchanged core-image-minimal | wc -l
#   -> hundreds. Every compile task's signature changed.

# 4. Find out WHY a specific recipe rebuilt.
bitbake -S none zlib
bitbake-diffsigs $(ls -t tmp/stamps/*/zlib/*do_compile.sigdata.* | head -2 | tac)

# 5. Demonstrate [vardepsexclude].
#    Add a task that embeds the build date:
cat >> ~/yocto/meta-mylayer/recipes-examples/makefile-example/simpletool_1.0.bb <<'EOF'

do_stamp() {
    echo "built ${DATETIME}" > ${B}/build-stamp
}
addtask stamp after do_compile before do_install
# WITHOUT this, every build rebuilds because DATETIME always changes:
do_stamp[vardepsexclude] = "DATETIME"
EOF

# 6. Hash equivalence: prove it prevents a cascade.
cat >> conf/local.conf <<'EOF'
BB_SIGNATURE_HANDLER = "OEEquivHash"
BB_HASHSERVE = "auto"
EOF
#   Now a change that produces IDENTICAL output does not cascade to
#   dependents. Test it: add a comment to a recipe and rebuild.
```

### Lab 97.6 — Validate a layer

```bash
cd ~/yocto/poky

# The official layer-compatibility checker.
scripts/yocto-check-layer ~/yocto/meta-mylayer

# What it checks:
#   - conf/layer.conf exists and is well-formed
#   - LAYERSERIES_COMPAT matches this release
#   - the layer does not break a build when added
#   - no signature changes to recipes in OTHER layers
#     ^ THIS IS THE IMPORTANT ONE: a layer must not silently alter
#       other layers' output, or it is not composable.
#   - README, LICENSE, and maintainer info present

# Also run these before shipping a layer:
bitbake-layers show-overlayed        # what are we shadowing?
bitbake-layers show-appends          # what are we appending to?
bitbake-layers show-cross-depends    # cross-layer dependencies
```

---

## 3. Mastery drills

1. Write ten recipes covering every build system you are likely to meet: autotools, CMake,
   Meson, plain Makefile, Python, Go, Rust, a prebuilt binary, a kernel module, and a data-only
   package.

2. Deliberately trigger all twelve QA checks in §T.6, and fix each. Write the one-line
   explanation of what each protects against.

3. Use `devtool` for a full cycle on a real package: modify, build, deploy to hardware,
   iterate, finish. Then upgrade it two versions and rebase the patches.

4. Write a class that enforces a real policy in your organization (hardening, licence
   allow-listing, a required metadata field) and apply it to a whole layer with
   `INHERIT`.

5. Trace a single variable through five layers of override and append, and predict its final
   value before checking with `bitbake -e`. Do this until you are right every time.

6. Build the same recipe for target, native, and nativesdk. Explain what differs in each and
   why `BBCLASSEXTEND` is not always sufficient.

7. Set up multiconfig to build a main image and a separate initramfs with different
   `DISTRO_FEATURES`, with a cross-multiconfig dependency.

8. Write a `PACKAGECONFIG`-driven recipe with six optional features and verify each
   combination builds and packages correctly.

9. Investigate a rebuild cascade in a real project: find the variable causing it and fix it
   with `[vardepsexclude]` or by restructuring. Measure the build-time saving.

10. Read `insane.bbclass` and `package.bbclass` completely. Write a summary of what
    `do_package` actually does, step by step.

11. Write a fetcher-level exercise: package software from a git repo with submodules, from
    npm, and from crates.io. Handle the offline-build case for each.

12. Audit a real vendor BSP layer: run `yocto-check-layer`, count the recipes that copy rather
    than append, count patches with `Upstream-Status: Pending`, and write the remediation
    plan (Ch. 88).

---

## 4. Further reading

**Documentation**
- **BitBake User Manual** — the language reference; read the syntax and execution chapters in
  full
- **Yocto Project Reference Manual** — the variable glossary; you will use it constantly
- **Yocto Project Development Tasks Manual** — the recipe cookbook
- `meta/conf/documentation.conf` — every variable, in the tree
- `meta/classes-recipe/*.bbclass` — read `autotools`, `cmake`, `systemd`, `useradd`

**Style and conventions**
- OpenEmbedded Styleguide (on the OE wiki) — recipe formatting and naming
- `Upstream-Status` conventions — documented on the OE wiki; enforce them
- `scripts/yocto-check-layer` and the Yocto Project Compatible programme requirements

**Tools**
- `devtool` — the Development Tasks Manual's devtool chapter
- `recipetool` — `recipetool create <url>` for a starter recipe
- `bitbake-layers`, `bitbake-diffsigs`, `bitbake-whatchanged`, `oe-pkgdata-util`
- `bblayers`/`bitbake` tab completion — install it

**Community**
- The OpenEmbedded layer index (`layers.openembedded.org`) — search before writing
- `openembedded-devel@lists.openembedded.org` — where recipes are reviewed; read it to learn
  the conventions
- `#yocto` on Libera Chat

**Cross-references**
- Ch. 96 — the concepts this chapter operationalizes
- Ch. 98 — kernel and BSP recipes, which use everything here
- Ch. 99 — SDK, licensing, CVE, SBOM, reproducibility
- Ch. 88 — patch management and `Upstream-Status` as a delta-management practice

→ Next: [98-yocto-3-bsp-kernel.md](98-yocto-3-bsp-kernel.md)

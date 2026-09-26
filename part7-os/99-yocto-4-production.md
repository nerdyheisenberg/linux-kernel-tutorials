# Chapter 99 — Yocto IV: Production — SDK, Licensing, CVE, SBOM, OTA, Reproducibility

> The chapters so far produce an image that boots. This one is about everything between that
> and a product you can ship, support for a decade, and defend in an audit. It is the least
> glamorous material in the curriculum and the most likely to be the thing that actually
> blocks a release.

---

## Theory & First Principles

### T.0 — Start here: the update that bricks 10,000 devices

Your device is in a field, on a pole, 400 km away. You push a firmware update. **Power fails
halfway through the write.** What happens next decides whether you have a product or a recall.

```
   NAIVE:  erase flash -> write new image -> reboot
           power loss anywhere in the middle -> UNBOOTABLE BRICK
           x 10,000 devices, each requiring a truck.
```

**The fix is the pattern you have now seen five times in this book:**

```
   A/B (dual-bank):

     +--------+--------+----------+
     | slot A | slot B |   data   |
     +--------+--------+----------+
         ^ running          ^ write the update HERE
                            |
     1. write slot B completely, and VERIFY it (signature + hash)
     2. atomically flip ONE bootloader variable: boot_slot = B
     3. reboot into B
     4. B must "confirm" itself within N minutes, or the bootloader
        counts a failed boot and AUTOMATICALLY ROLLS BACK to A.
```

**Step 2 is the entire design: everything expensive and failure-prone happens first, and the
commit is one atomic write.** That is validate-then-apply (Ch. 89 §T.0, principle 6) — the
same structure as a filesystem journal's commit block (Ch. 61 §T.0), dm's suspend/reload/
resume (Ch. 66 §T.0), and btrfs's root-pointer swap (Ch. 59 §T.0). **Step 4 adds the piece
those do not need: an automatic rollback on a failure you cannot detect at commit time.**

**The cost is honest and should be stated: you pay double the flash.** Alternatives
(delta updates, a recovery partition, RAUC's various modes) trade robustness for space, and
choosing among them is a real engineering decision rather than a default.

**Now the other four things that separate a prototype from a shipped product**, none of which
a demo image has:

**1. Secure boot is a chain, and a chain is only as strong as its root.**

```
   ROM (keys fused into the SoC -- IRREVOCABLE)
     verifies -> bootloader
       verifies -> kernel + device tree      <- the DTB must be signed too;
         verifies -> rootfs (dm-verity)         an attacker who can change it
                                                can change the kernel cmdline
```

**Forgetting to sign the device tree is a classic and complete bypass.** And note the
permanence: fusing a key is a one-way operation on a physical device. **A secure-boot key
management mistake is not recoverable in the field**, which makes it the highest-stakes
decision in the whole product.

**2. `dm-verity` for the root filesystem** — a Merkle tree over the block device (Ch. 66),
verified *per block, on read*, with the root hash signed. Read-only, tamper-evident, and
verified lazily so boot is not slowed by hashing gigabytes.

**3. Licence compliance is a build output, not a document.** Yocto can emit the license
manifest, the source archives for every GPL component, and an SPDX SBOM — **because the build
system knows exactly what went in** (Ch. 96 §T.0). Reconstructing this after the fact is
miserable and often impossible; generating it is a configuration line.

**4. Reproducibility must be tested, not assumed.** Build twice and diff:

```bash
INHERIT += "reproducible_build"
SOURCE_DATE_EPOCH, BUILD_REPRODUCIBLE_BINARIES=1
diffoscope image-1.rootfs.tar image-2.rootfs.tar
```

Timestamps, build paths, `__DATE__`, hash-order-dependent output, and parallel-build
nondeterminism will all break it, silently, until someone checks.

**The framing for this chapter:** everything here is about **the failure you cannot attend
to.** You will not be there when the power fails, when the update is interrupted, or when the
device is stolen and its flash is read. **Production engineering is designing for the
absence of an operator** — which is why every mechanism in this chapter is automatic, atomic,
and self-verifying.

```bash
bitbake core-image-minimal -c populate_sdk
cat tmp/deploy/licenses/core-image-minimal-*/license.manifest | head
rauc status && rauc install update.raucb
veritysetup verify /dev/mmcblk0p2 /dev/mmcblk0p3 <root-hash>
```

---

### T.1 — The production checklist

Before an image ships:

| Concern | Requirement |
|---|---|
| **Security** | No `debug-tweaks`, real credentials, no debug tooling, hardened kernel and userspace |
| **Licence compliance** | Manifest, source archive, licence texts, written offer |
| **SBOM** | A machine-readable inventory (SPDX or CycloneDX) |
| **CVE posture** | A known, documented state and an update path |
| **Reproducibility** | The same inputs produce bit-identical outputs |
| **Update mechanism** | Atomic, verified, rollback-capable |
| **Provenance** | Which commit of which layer produced this artefact |
| **Recovery** | What happens when an update fails or the storage is corrupt |
| **Support lifetime** | The kernel and distro outlive the product (Ch. 88 §T.9) |

**The one that surprises people:** `IMAGE_FEATURES = "debug-tweaks"` is in every
`core-image-*` by default. It sets an **empty root password**, enables serial autologin, and
installs debug tooling. Shipping it is the single most common embedded Linux security
failure, and it is a one-line fix that nobody makes because the development image works.

### T.2 — The production image

```bash
# recipes-images/images/myproduct-image.bb
SUMMARY = "Production image"
LICENSE = "MIT"

inherit core-image extrausers

# --- NOT debug-tweaks. Be explicit about what you DO want. ---
IMAGE_FEATURES = "ssh-server-openssh"

# --- Remove everything development-only ---
IMAGE_FEATURES:remove = "debug-tweaks tools-debug tools-profile dev-pkgs \
                         staticdev-pkgs dbg-pkgs empty-root-password \
                         allow-empty-password allow-root-login \
                         post-install-logging"

# --- Real credentials, injected from the build environment ---
#     NEVER a literal in a recipe that lives in git.
EXTRA_USERS_PARAMS = "\
    usermod -p '${@d.getVar('ROOT_PASSWORD_HASH') or '!'}' root; \
    useradd -m -s /bin/sh operator; \
    usermod -p '${@d.getVar('OPERATOR_PASSWORD_HASH') or '!'}' operator; \
"
#   '!' means "locked account, no password login" -- the right default.

# --- Size and layout ---
IMAGE_LINGUAS = " "
IMAGE_ROOTFS_EXTRA_SPACE = "0"
IMAGE_OVERHEAD_FACTOR = "1.0"
IMAGE_FSTYPES = "wic.gz wic.bmap squashfs-lz4"

# --- Read-only root, if the product can manage it ---
IMAGE_FEATURES += "read-only-rootfs"
#   Forces you to discover every path that writes to /, which is a
#   valuable exercise regardless.

ROOTFS_POSTPROCESS_COMMAND += "production_checks; production_cleanup; "

production_cleanup() {
    rm -rf ${IMAGE_ROOTFS}/usr/share/doc
    rm -rf ${IMAGE_ROOTFS}/usr/share/man
    rm -f  ${IMAGE_ROOTFS}/etc/ssh/ssh_host_*_key*     # regenerate on first boot
    rm -rf ${IMAGE_ROOTFS}/var/log/*
    # Remove the package manager if not doing on-target updates:
    # rm -rf ${IMAGE_ROOTFS}/var/lib/opkg
}

python production_checks() {
    import os, stat
    rootfs = d.getVar('IMAGE_ROOTFS')
    fail = []

    # 1. No empty passwords.
    shadow = os.path.join(rootfs, 'etc/shadow')
    if os.path.exists(shadow):
        for line in open(shadow):
            f = line.split(':')
            if len(f) > 1 and f[1] == '':
                fail.append("empty password for user %s" % f[0])

    # 2. No unexpected setuid binaries.
    allowed = set((d.getVar('PRODUCTION_SETUID_ALLOWED') or '').split())
    for root, dirs, files in os.walk(rootfs):
        for fn in files:
            p = os.path.join(root, fn)
            if os.path.islink(p):
                continue
            if os.lstat(p).st_mode & (stat.S_ISUID | stat.S_ISGID):
                rel = p[len(rootfs):]
                if rel not in allowed:
                    fail.append("unexpected setuid: %s" % rel)

    # 3. No world-writable files.
    for root, dirs, files in os.walk(rootfs):
        for fn in files:
            p = os.path.join(root, fn)
            if not os.path.islink(p) and os.lstat(p).st_mode & stat.S_IWOTH:
                fail.append("world-writable: %s" % p[len(rootfs):])

    if fail:
        bb.fatal("production checks failed:\n  " + "\n  ".join(fail))
}
```

**Make the checks fail the build.** A warning in a 40,000-line build log is not a control.

### T.3 — Licence compliance

The legal obligations, and the mechanisms that satisfy them.

| Licence class | Obligation |
|---|---|
| Permissive (MIT, BSD, Apache) | Reproduce the licence text and copyright notice |
| **LGPL** | The above, **plus** the ability to relink with a modified library |
| **GPLv2** | The above, plus **complete corresponding source** for the whole derived work |
| **GPLv3 / LGPLv3** | The above, plus **Installation Information** — the ability to install modified versions ("anti-tivoization") |
| **AGPL** | GPLv3, plus source for network-accessible use |

**GPLv3's Installation Information requirement is why `INCOMPATIBLE_LICENSE = "GPL-3.0*"`
is common in embedded**: if your product has a locked bootloader (Ch. 90 §T.6), you cannot
provide the ability to install modified software, so you cannot ship GPLv3 code. This is a
genuine conflict between secure boot and GPLv3, it has no clean resolution, and **it must be
decided at project start** because it determines which packages you may use.

```bash
# --- What Yocto produces ---
INHERIT += "archiver"
ARCHIVER_MODE[src] = "original"     # or "patched", "configured"
ARCHIVER_MODE[diff] = "1"
ARCHIVER_MODE[recipe] = "1"
COPY_LIC_MANIFEST = "1"
COPY_LIC_DIRS = "1"
LICENSE_CREATE_PACKAGE = "1"

INCOMPATIBLE_LICENSE = "GPL-3.0* LGPL-3.0* AGPL-3.0*"

# Per-recipe exception, when you have legal sign-off:
# INCOMPATIBLE_LICENSE:pn-somepackage = ""
```

```bash
# --- The artefacts ---
tmp/deploy/licenses/<image>/license.manifest      # every package + licence
tmp/deploy/licenses/<image>/package.manifest
tmp/deploy/licenses/<recipe>/                     # the licence texts
tmp/deploy/sources/                               # the source archives

# On the target, if LICENSE_CREATE_PACKAGE=1:
/usr/share/common-licenses/
```

**The compliance package to produce for each release:**
1. `license.manifest` — every package and its licence.
2. The full text of every licence used.
3. Complete corresponding source for every GPL/LGPL component, **including your patches**
   and the exact build configuration.
4. A written offer valid for three years (GPLv2) with a working contact.
5. For LGPL: object files or a mechanism allowing relinking.
6. Attribution notices in the product documentation.

**Automate the generation and store it with the release artefacts.** Reconstructing "what
was in version 2.3.1" two years later, from a build system that has since changed, is how
compliance incidents happen.

### T.4 — SBOM

```bash
INHERIT += "create-spdx"
SPDX_PRETTY = "1"
SPDX_INCLUDE_SOURCES = "1"

# Output:
#   tmp/deploy/spdx/<machine>/<image>.spdx.json
#   tmp/deploy/images/<machine>/<image>.spdx.tar.zst
```

An SPDX document records, per package: name, version, supplier, download location, checksum,
declared and detected licences, and the dependency relationships. That makes it possible to
answer, mechanically:

- Does this image contain log4j / xz / any specific component?
- What version?
- Which licences are we shipping?
- What changed between release 2.3.0 and 2.3.1?

**The regulatory driver:** US Executive Order 14028 requires SBOMs for federal software
procurement, and the **EU Cyber Resilience Act** (entering force 2024–2027) requires them for
products with digital elements sold in the EU, along with vulnerability handling and a defined
support period. For any product sold in Europe after 2027 this stops being optional.

The CRA's other requirements are worth knowing because they change embedded Linux practice:
a documented support period, a coordinated vulnerability disclosure process, security
updates for the support period, and documentation of the security properties. **All of them
push toward the continuous-stable-adoption model of Ch. 88.**

### T.5 — CVE management

```bash
INHERIT += "cve-check"
CVE_CHECK_REPORT_PATCHED = "0"
CVE_CHECK_SHOW_WARNINGS = "1"
CVE_CHECK_CREATE_MANIFEST = "1"

# Output:
#   tmp/deploy/cve/<image>_cve.json
#   tmp/deploy/images/<machine>/<image>.cve
```

The mechanism: `cve-check` maps each recipe to CPE identifiers, queries the NVD database, and
reports CVEs affecting the version you build — minus ones your patches fix, if the patch
names the CVE.

```bash
# Tell cve-check a patch fixes something:
SRC_URI += "file://0001-fix-CVE-2024-12345.patch"
#   The filename containing "CVE-2024-12345" is enough. Or, in the patch:
#   CVE: CVE-2024-12345

# Mark one as not applicable, WITH A REASON:
CVE_STATUS[CVE-2024-99999] = "not-applicable-config: \
    the vulnerable code requires --enable-foo which we do not build"
CVE_STATUS[CVE-2024-88888] = "fixed-version: \
    backported in our 1.2.3-r5 patch queue"
```

**The honest limitations, which you must state to anyone relying on the output:**

1. **CPE matching is imperfect.** A recipe whose `CVE_PRODUCT` is not set correctly reports
   nothing, which looks like "no CVEs" and is not.
2. **False positives dominate.** Many reported CVEs are unreachable in your configuration.
3. **It does not know about vendored code.** A package that bundles a copy of zlib reports
   zlib's CVEs only if the recipe declares it.
4. **The kernel is a special case.** Since the kernel became a CNA (Ch. 88 §T.6), thousands
   of CVEs are assigned annually and the guidance is explicitly *not* to triage them
   individually but to take the stable tree continuously.

**The correct posture**, and this is the memo you will have to write:

> Per-CVE triage is not a viable strategy for a Linux-based product. The supported strategy
> is: track a supported LTS, take stable updates continuously, maintain an SBOM, and use
> CVE scanning to *detect drift* — not as a to-do list. Document exceptions with reasons.

### T.6 — The SDK

```bash
# The standard SDK: a self-extracting cross-toolchain.
bitbake myproduct-image -c populate_sdk
# -> tmp/deploy/sdk/poky-glibc-x86_64-myproduct-image-aarch64-myboard-toolchain-4.0.sh

# The extensible SDK: adds devtool and the ability to build recipes.
bitbake myproduct-image -c populate_sdk_ext
```

```bash
# Using it:
./poky-glibc-...-toolchain-4.0.sh -d ~/mysdk
source ~/mysdk/environment-setup-aarch64-poky-linux
echo $CC                 # aarch64-poky-linux-gcc --sysroot=...
$CC -o hello hello.c
file hello               # ARM aarch64
```

Adding what application developers need:

```bash
# In the image recipe:
TOOLCHAIN_HOST_TASK:append = " nativesdk-cmake nativesdk-ninja nativesdk-gdb"
TOOLCHAIN_TARGET_TASK:append = " libmyapp-dev libmyapp-staticdev"
SDKIMAGE_FEATURES = "dev-pkgs dbg-pkgs staticdev-pkgs"
```

**Why the SDK matters organizationally:** it decouples application developers from the
platform team. Without one, every application developer needs a full Yocto build (hours, tens
of gigabytes, and platform expertise). With one, they get a tarball, `source`, and `make`.
That is the difference between a platform team that scales and one that is a bottleneck.

The **extensible SDK** additionally lets an application developer run `devtool add` and
produce a recipe — bridging the gap between "my app builds" and "my app is in the image."

### T.7 — Reproducible builds

**The definition:** given the same source, the same configuration, and the same tooling, the
build produces **bit-identical** output.

**Why it matters**, and the arguments in order of strength:

1. **Verification.** A third party can rebuild and confirm the binary matches the source —
   the only defence against a compromised build server (the SolarWinds attack class).
2. **Debugging.** A bug in a shipped binary can be reproduced exactly.
3. **Caching.** Identical output means sstate and hash equivalence work, which is a large
   build-time saving.
4. **Compliance.** "This binary came from this source" becomes provable rather than asserted.

**The sources of non-determinism**, all of which OE now handles:

| Source | Fix |
|---|---|
| Timestamps | `SOURCE_DATE_EPOCH`, `-Wno-builtin-macro-redefined -D__DATE__=...` |
| Build paths | `-ffile-prefix-map`, `--remap-path-prefix` (the `buildpaths` QA check) |
| Build host details (hostname, user) | scrubbed by the build environment |
| Filesystem ordering | sorted `find`, `--sort=name` for tar |
| Parallelism-dependent output | make the build order-independent |
| Random values (uuids, keys) | seed deterministically or generate at first boot |
| Locale and timezone | `LC_ALL=C`, `TZ=UTC` |
| `mtime` in archives | `--mtime=@${SOURCE_DATE_EPOCH}` |

```bash
BUILD_REPRODUCIBLE_BINARIES = "1"
SOURCE_DATE_EPOCH = "1700000000"

# Verify:
bitbake myproduct-image
cp -r tmp/deploy/images/myboard /tmp/build1
rm -rf tmp sstate-cache
bitbake myproduct-image
diffoscope /tmp/build1/rootfs.tar.xz tmp/deploy/images/myboard/rootfs.tar.xz
```

`diffoscope` is the tool: it recursively unpacks archives, disassembles binaries, and shows
exactly what differs. **Run it; the first result is always surprising.**

OE-Core runs a reproducibility test in its own CI (`oe-selftest -r reproducible`) and the
core is very close to fully reproducible — but *your* layer probably is not, and the
`buildpaths` QA check is the first line of defence.

### T.8 — Update mechanisms

| Approach | Mechanism | Atomic? | Rollback? | Storage |
|---|---|---|---|---|
| **Package updates** (`opkg`/`dnf`) | Install individual packages | **No** | No | small |
| **A/B (dual-bank)** | Two full rootfs slots; switch and reboot | **Yes** | **Yes** | 2× rootfs |
| **Delta A/B** | A/B plus binary diffs | Yes | Yes | 2× + delta |
| **OSTree / libostree** | Content-addressed filesystem trees, hardlinked | Yes | Yes | ~1.2× |
| **Container images** | Update the app, not the OS | Yes (per app) | Yes | varies |

**The A/B argument, which is the default answer for embedded:** power can fail at any moment.
A package-based update that is interrupted leaves a system in an undefined state that may not
boot. An A/B update writes the inactive slot, verifies it, and then flips a single bootloader
variable — one atomic operation. If the new slot fails to boot, the bootloader's
try-count mechanism falls back.

```bash
# --- SWUpdate (meta-swupdate) ---
IMAGE_INSTALL:append = " swupdate"
# sw-description:
software =
{
    version = "1.2.0";
    hardware-compatibility = [ "1.0", "1.1" ];
    stable: {
        myboard: {
            images: (
                { filename = "rootfs.ext4.gz";
                  device = "/dev/mmcblk0p3";     /* the INACTIVE slot */
                  compressed = "zlib";
                  sha256 = "...";
                },
                { filename = "Image";
                  device = "/dev/mmcblk0p1";
                  filesystem = "vfat"; path = "/Image";
                }
            );
            scripts: (
                { filename = "post-update.sh"; type = "postinstall"; }
            );
        };
    };
}
```

```bash
# --- RAUC (meta-rauc) ---
# system.conf
[system]
compatible=myproduct
bootloader=uboot
bundle-formats=verity           # hash-tree verified bundles

[slot.rootfs.0]
device=/dev/mmcblk0p2
type=ext4
bootname=A

[slot.rootfs.1]
device=/dev/mmcblk0p3
type=ext4
bootname=B

[keyring]
path=/etc/rauc/keyring.pem      # signature verification
```

```bash
# --- The U-Boot side: try-count and fallback ---
setenv bootcount 0
setenv bootlimit 3
setenv altbootcmd 'run bootcmd_B'       # what to do after bootlimit failures
saveenv
#   U-Boot increments bootcount each boot; userspace resets it to 0 on a
#   successful start. Three failures and U-Boot runs altbootcmd.
#   THIS is the rollback mechanism, and it must be tested.
```

**The update requirements checklist:**
- [ ] **Atomic** — no partially-updated state is reachable.
- [ ] **Verified** — the bundle is signed, and the signature is checked before installing.
- [ ] **Rollback** — automatic, on boot failure, with a bounded try count.
- [ ] **Anti-rollback** — a monotonic counter so an attacker cannot install an old, signed,
      vulnerable version (Ch. 90 §T.6).
- [ ] **Compatible** — hardware-compatibility checks so an image for rev2 does not brick a
      rev1.
- [ ] **Resumable / power-fail-safe** at every point.
- [ ] **Data preserved** — a separate data partition the update never touches.
- [ ] **Tested** — including power loss during the update, and the rollback path.

### T.9 — Build infrastructure

```yaml
# .gitlab-ci.yml (the shape; adapt to your CI)
variables:
  DL_DIR: "/cache/downloads"
  SSTATE_DIR: "/cache/sstate"

stages: [build, test, compliance, release]

build:
  stage: build
  script:
    - source poky/oe-init-build-env build
    - echo "MACHINE = \"myboard\"" >> conf/local.conf
    - echo "DISTRO = \"mydistro\"" >> conf/local.conf
    - bitbake myproduct-image
  artifacts:
    paths:
      - build/tmp/deploy/images/myboard/
      - build/tmp/deploy/licenses/
      - build/tmp/deploy/spdx/
      - build/tmp/deploy/cve/

test:
  stage: test
  script:
    - bitbake myproduct-image -c testimage     # boots it in QEMU and runs tests
    - ./scripts/run-hardware-tests.sh

compliance:
  stage: compliance
  script:
    - ./scripts/check-licenses.sh              # fail on unexpected licences
    - ./scripts/check-cves.sh --max-severity high
    - ./scripts/verify-sbom.sh

reproducibility:
  stage: test
  script:
    - ./scripts/build-twice-and-diffoscope.sh
```

**The non-negotiables for a Yocto CI:**
1. **Shared `DL_DIR` and `SSTATE_DIR`.** Without them every build is hours.
2. **A manifest of exact layer revisions** per build, stored with the artefacts.
3. **`bitbake -c testimage`** — Yocto's built-in runtime testing, which boots the image in
   QEMU and runs tests.
4. **Artefact retention** including licences, SBOM, and CVE reports, not just the image.
5. **Build on the same OS you will build on in five years**, which in practice means a
   container image that is itself versioned.

`kas` or `repo` manages the layer set reproducibly:

```yaml
# kas.yml
header: { version: 14 }
machine: myboard
distro: mydistro
target: myproduct-image
repos:
  meta-myboard:
  poky:
    url: https://git.yoctoproject.org/poky
    commit: a1b2c3d4...            # EXACT commits, never branches
    layers: { meta: , meta-poky: }
  meta-openembedded:
    url: https://github.com/openembedded/meta-openembedded
    commit: e5f6a7b8...
    layers: { meta-oe: , meta-python: , meta-networking: }
```

### T.10 — The ten-year problem

Products outlive their build systems. The failure mode is specific and common: in year six
you need a security fix, and the build no longer works — the host OS is gone, a download URL
is dead, a toolchain will not compile on a modern host.

**What to archive, per release:**

1. **A container image of the complete build environment**, versioned and stored in a
   registry you control. This is the single most effective measure.
2. **A full `DL_DIR` mirror** — every source tarball. Upstream URLs die.
   (`BB_GENERATE_MIRROR_TARBALLS = "1"` makes git repos into archivable tarballs.)
3. **Exact layer revisions** (the `kas.yml` or a `repo` manifest with commits).
4. **The sstate cache**, if storage permits — it makes a rebuild minutes instead of hours.
5. **The output artefacts**: image, SBOM, licence manifest, CVE report, symbols.
6. **The debug symbols** (`-dbg` packages), so a crash from the field can be analysed.

```bash
# Generate archivable mirror tarballs for everything, including git:
BB_GENERATE_MIRROR_TARBALLS = "1"
BB_NO_NETWORK = "1"             # verify the offline build actually works

# Then test it: on a machine with no network, from the archive, rebuild.
# If it does not work, your archive is incomplete. FIND OUT NOW, not in
# year six.
```

**Test the archive annually.** An untested archive is a hypothesis, exactly as an untested
backup is (Ch. 94 §T.10).

---

## 1. Internals

### Classes and outputs

| Class | Produces |
|---|---|
| `archiver` | `tmp/deploy/sources/` — source archives for compliance |
| `create-spdx` | `tmp/deploy/spdx/` — SBOM |
| `cve-check` | `tmp/deploy/cve/` — CVE reports |
| `populate_sdk` / `populate_sdk_ext` | `tmp/deploy/sdk/` |
| `testimage` | Runtime test results |
| `license_image` | `tmp/deploy/licenses/<image>/` |
| `reproducible_build` | `SOURCE_DATE_EPOCH` handling |
| `rootfs-postcommands` | The `read-only-rootfs`, `debug-tweaks` implementations |
| `image_types` | Every `IMAGE_FSTYPES` |

```bash
# Where everything lands:
tmp/deploy/images/<machine>/      image, kernel, dtb, wic, bmap
tmp/deploy/licenses/              per-image and per-recipe licence data
tmp/deploy/sources/               source archives
tmp/deploy/spdx/<machine>/        SBOM
tmp/deploy/cve/                   CVE reports
tmp/deploy/sdk/                   SDK installers
tmp/deploy/rpm|deb|ipk/           packages
tmp/log/                          cooker logs
```

### `testimage`

```bash
IMAGE_CLASSES += "testimage"
TEST_SUITES = "ping ssh df connman syslog systemd date parselogs \
               myproduct_tests"
TEST_TARGET = "qemu"            # or "simpleremote" for real hardware
TEST_SERVER_IP = "192.168.1.1"
TEST_TARGET_IP = "192.168.1.10"

bitbake myproduct-image -c testimage
```

A test case:

```python
# lib/oeqa/runtime/cases/myproduct.py
from oeqa.runtime.case import OERuntimeTestCase
from oeqa.core.decorator.depends import OETestDepends

class MyProductTest(OERuntimeTestCase):

    @OETestDepends(['ssh.SSHTest.test_ssh'])
    def test_myapp_running(self):
        status, output = self.target.run('systemctl is-active myapp')
        self.assertEqual(status, 0, msg='myapp not active: %s' % output)

    def test_no_empty_passwords(self):
        status, output = self.target.run(
            "awk -F: '$2 == \"\" {print $1}' /etc/shadow")
        self.assertEqual(output.strip(), '',
                         msg='accounts with empty passwords: %s' % output)

    def test_no_debug_tools(self):
        for tool in ['gdb', 'strace', 'tcpdump']:
            status, _ = self.target.run('which %s' % tool)
            self.assertNotEqual(status, 0,
                                msg='%s present in production image' % tool)

    def test_readonly_rootfs(self):
        status, _ = self.target.run('touch /probe-readonly')
        self.assertNotEqual(status, 0, msg='rootfs is writable')
```

**Runtime tests that assert security properties are the control that actually works.** A
checklist in a wiki is not; a test that fails the build is.

---

## 2. Practice

### Lab 99.1 — Harden a production image

```bash
cd ~/yocto/meta-myboard/recipes-images/images

cat > myproduct-image.bb <<'PRODEOF'
SUMMARY = "Production image with hardening and checks"
LICENSE = "MIT"

inherit core-image extrausers

IMAGE_FEATURES = "ssh-server-openssh read-only-rootfs"
IMAGE_FEATURES:remove = "debug-tweaks tools-debug tools-profile dev-pkgs \
                         dbg-pkgs staticdev-pkgs empty-root-password \
                         allow-empty-password allow-root-login"

IMAGE_INSTALL:append = " \
    kernel-modules \
    myboard-driver \
    util-linux \
"

IMAGE_LINGUAS = " "
IMAGE_OVERHEAD_FACTOR = "1.0"
IMAGE_FSTYPES = "wic.gz wic.bmap squashfs-lz4 tar.xz"

PRODUCTION_SETUID_ALLOWED ?= "/usr/bin/newgrp /bin/su /usr/bin/passwd"

EXTRA_USERS_PARAMS = "usermod -p '!' root;"

ROOTFS_POSTPROCESS_COMMAND += "prod_cleanup; prod_checks; "

prod_cleanup() {
    rm -rf ${IMAGE_ROOTFS}/usr/share/doc ${IMAGE_ROOTFS}/usr/share/man
    rm -f  ${IMAGE_ROOTFS}/etc/ssh/ssh_host_*_key*
    rm -rf ${IMAGE_ROOTFS}/var/log/* 2>/dev/null || true
}

python prod_checks() {
    import os, stat
    rootfs = d.getVar('IMAGE_ROOTFS')
    allowed = set((d.getVar('PRODUCTION_SETUID_ALLOWED') or '').split())
    problems = []

    shadow = os.path.join(rootfs, 'etc/shadow')
    if os.path.exists(shadow):
        for line in open(shadow):
            fields = line.strip().split(':')
            if len(fields) > 1 and fields[1] == '':
                problems.append("EMPTY PASSWORD: %s" % fields[0])

    for root, dirs, files in os.walk(rootfs):
        for fn in files:
            p = os.path.join(root, fn)
            if os.path.islink(p):
                continue
            st = os.lstat(p)
            rel = p[len(rootfs):]
            if st.st_mode & (stat.S_ISUID | stat.S_ISGID):
                if rel not in allowed:
                    problems.append("SETUID: %s" % rel)
            if st.st_mode & stat.S_IWOTH:
                problems.append("WORLD-WRITABLE: %s" % rel)

    for banned in ['usr/bin/gdb', 'usr/bin/strace', 'usr/sbin/tcpdump']:
        if os.path.exists(os.path.join(rootfs, banned)):
            problems.append("DEBUG TOOL: /%s" % banned)

    if problems:
        bb.fatal("production image checks FAILED:\n  " + "\n  ".join(problems))
    bb.note("production image checks passed")
}
PRODEOF

cd ~/yocto/build
bitbake myproduct-image

# Compare with the development image:
ls -lh tmp/deploy/images/myboard/*.rootfs.tar.xz
#   The production image should be substantially smaller.

# Verify on the target:
runqemu myboard myproduct-image nographic
#   root login should FAIL (account locked).
#   touch /test should FAIL (read-only rootfs).
#   which gdb strace should find nothing.
```

### Lab 99.2 — Compliance package

```bash
#!/bin/bash
# compliance.sh — produce the full compliance artefact set.
set -e
cd ~/yocto/build
IMAGE=${1:-myproduct-image}
MACHINE=${2:-myboard}
OUT=~/releases/$(date +%Y%m%d)-${IMAGE}
mkdir -p $OUT/{licenses,sources,sbom,cve}

cat >> conf/local.conf <<'EOF'
INHERIT += "archiver create-spdx cve-check"
ARCHIVER_MODE[src] = "original"
ARCHIVER_MODE[diff] = "1"
ARCHIVER_MODE[recipe] = "1"
COPY_LIC_MANIFEST = "1"
COPY_LIC_DIRS = "1"
LICENSE_CREATE_PACKAGE = "1"
CVE_CHECK_CREATE_MANIFEST = "1"
EOF

bitbake $IMAGE

echo "=== 1. Licence manifest ==="
cp tmp/deploy/licenses/${IMAGE}-${MACHINE}*/license.manifest $OUT/licenses/
awk '/^LICENSE:/ {print $2}' $OUT/licenses/license.manifest \
	| tr '&|' '\n\n' | tr -d '()' | sort -u > $OUT/licenses/licenses-used.txt
cat $OUT/licenses/licenses-used.txt

echo; echo "=== 2. Which packages are GPL? (these need source) ==="
grep -B2 'GPL' $OUT/licenses/license.manifest | grep '^PACKAGE NAME' \
	| awk '{print $3}' | sort -u | tee $OUT/licenses/gpl-packages.txt

echo; echo "=== 3. Source archives ==="
cp -r tmp/deploy/sources/* $OUT/sources/ 2>/dev/null || \
	echo "  (archiver output not found -- did you rebuild after enabling it?)"
du -sh $OUT/sources

echo; echo "=== 4. Licence texts ==="
cp -r tmp/deploy/licenses/* $OUT/licenses/

echo; echo "=== 5. SBOM ==="
cp tmp/deploy/spdx/${MACHINE}/*.spdx.json $OUT/sbom/ 2>/dev/null || true
python3 - "$OUT/sbom" <<'PY'
import json, sys, glob, os
for f in glob.glob(os.path.join(sys.argv[1], '*.spdx.json')):
    d = json.load(open(f))
    pkgs = d.get('packages', [])
    print(f"{os.path.basename(f)}: {len(pkgs)} packages")
    lic = {}
    for p in pkgs:
        l = p.get('licenseDeclared', 'NOASSERTION')
        lic[l] = lic.get(l, 0) + 1
    for l, n in sorted(lic.items(), key=lambda kv: -kv[1])[:15]:
        print(f"   {n:4d}  {l}")
PY

echo; echo "=== 6. CVE report ==="
cp tmp/deploy/cve/* $OUT/cve/ 2>/dev/null || true
python3 - "$OUT/cve" <<'PY'
import json, sys, glob, os
for f in glob.glob(os.path.join(sys.argv[1], '*.json')):
    d = json.load(open(f))
    unpatched = []
    for pkg in d.get('package', []):
        for issue in pkg.get('issue', []):
            if issue.get('status') == 'Unpatched':
                unpatched.append((pkg['name'], issue['id'],
                                  issue.get('scorev3', '?')))
    print(f"{os.path.basename(f)}: {len(unpatched)} unpatched")
    for name, cve, score in sorted(unpatched, key=lambda x: -float(x[2] or 0))[:20]:
        print(f"   {score:>4}  {cve}  {name}")
PY

echo; echo "=== 7. Build provenance ==="
{
  echo "date: $(date -Is)"
  echo "image: $IMAGE"
  echo "machine: $MACHINE"
  echo "layers:"
  bitbake-layers show-layers
  echo "layer revisions:"
  for d in $(bitbake-layers show-layers | awk '/^meta/ {print $2}'); do
      [ -d "$d/.git" ] && echo "  $d: $(git -C $d rev-parse HEAD)"
  done
} > $OUT/provenance.txt

echo; echo "Compliance package: $OUT"
du -sh $OUT
```

### Lab 99.3 — Verify reproducibility

```bash
#!/bin/bash
# reprotest.sh — build twice and diff.
set -e
cd ~/yocto/build

cat >> conf/local.conf <<'EOF'
BUILD_REPRODUCIBLE_BINARIES = "1"
SOURCE_DATE_EPOCH = "1700000000"
EOF

echo "=== Build 1 ==="
bitbake myproduct-image
mkdir -p /tmp/repro1
cp tmp/deploy/images/myboard/*.rootfs.tar.xz /tmp/repro1/
cp tmp/deploy/images/myboard/*.manifest /tmp/repro1/

echo "=== Wipe and build again, in a DIFFERENT path ==="
cd ~/yocto
rm -rf build/tmp
# Build from a different directory to catch buildpaths leaks:
source poky/oe-init-build-env build2
cp ../build/conf/local.conf conf/
cp ../build/conf/bblayers.conf conf/
sed -i 's|/build/|/build2/|g' conf/bblayers.conf 2>/dev/null || true
bitbake myproduct-image
mkdir -p /tmp/repro2
cp tmp/deploy/images/myboard/*.rootfs.tar.xz /tmp/repro2/

echo "=== Compare ==="
sha256sum /tmp/repro1/*.tar.xz /tmp/repro2/*.tar.xz

if ! diff -q /tmp/repro1/*.tar.xz /tmp/repro2/*.tar.xz >/dev/null 2>&1; then
	echo "NOT REPRODUCIBLE -- investigating with diffoscope"
	diffoscope --html /tmp/repro.html \
		/tmp/repro1/*.tar.xz /tmp/repro2/*.tar.xz || true
	echo "  open /tmp/repro.html"
else
	echo "REPRODUCIBLE"
fi

echo "=== Also run OE's own test ==="
cd ~/yocto/poky
oe-selftest -r reproducible
```

**The first run is always not-reproducible in your own layer.** Finding out why — usually a
timestamp, a build path, or an unsorted file list — is the lab.

### Lab 99.4 — A/B updates with RAUC

```bash
# 1. Add the layer.
cd ~/yocto
git clone https://github.com/rauc/meta-rauc.git
cd build && bitbake-layers add-layer ../meta-rauc

# 2. System configuration.
mkdir -p ~/yocto/meta-myboard/recipes-core/rauc/files
cat > ~/yocto/meta-myboard/recipes-core/rauc/files/system.conf <<'EOF'
[system]
compatible=myproduct
bootloader=uboot
bundle-formats=verity

[keyring]
path=/etc/rauc/keyring.pem

[slot.rootfs.0]
device=/dev/mmcblk0p2
type=ext4
bootname=A

[slot.rootfs.1]
device=/dev/mmcblk0p3
type=ext4
bootname=B
EOF

cat > ~/yocto/meta-myboard/recipes-core/rauc/rauc_%.bbappend <<'EOF'
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"
SRC_URI:append = " file://system.conf"
RAUC_KEYRING_FILE = "${THISDIR}/files/ca.cert.pem"
EOF

# 3. Signing keys (production keys live in an HSM, not in git).
mkdir -p ~/yocto/meta-myboard/recipes-core/rauc/files
cd ~/yocto/meta-myboard/recipes-core/rauc/files
openssl req -x509 -newkey rsa:4096 -nodes -keyout ca.key.pem \
	-out ca.cert.pem -days 7300 -subj "/CN=myproduct-ca"

# 4. The bundle recipe.
mkdir -p ~/yocto/meta-myboard/recipes-core/bundles
cat > ~/yocto/meta-myboard/recipes-core/bundles/myproduct-bundle.bb <<'EOF'
inherit bundle

RAUC_BUNDLE_COMPATIBLE = "myproduct"
RAUC_BUNDLE_VERSION = "${DISTRO_VERSION}"
RAUC_BUNDLE_DESCRIPTION = "My Product update bundle"
RAUC_BUNDLE_FORMAT = "verity"

RAUC_BUNDLE_SLOTS = "rootfs"
RAUC_SLOT_rootfs = "myproduct-image"
RAUC_SLOT_rootfs[fstype] = "ext4"

RAUC_KEY_FILE = "${THISDIR}/../rauc/files/ca.key.pem"
RAUC_CERT_FILE = "${THISDIR}/../rauc/files/ca.cert.pem"
EOF

cd ~/yocto/build
bitbake myproduct-bundle
ls -lh tmp/deploy/images/myboard/*.raucb

# 5. U-Boot side: bootcount and fallback.
cat <<'EOF'
 # In the U-Boot environment:
 setenv bootlimit 3
 setenv BOOT_A_LEFT 3
 setenv BOOT_B_LEFT 3
 setenv BOOT_ORDER "A B"
 setenv altbootcmd 'run bootcmd'
 saveenv
EOF

# 6. TEST THE FAILURE PATHS. This is the actual lab.
#    a. Normal update:      rauc install bundle.raucb ; reboot ; verify slot B
#    b. Corrupt bundle:      dd random bytes into it; rauc must REFUSE
#    c. Unsigned bundle:     must be refused
#    d. Power loss mid-write: the old slot must still boot
#    e. New slot does not boot: after `bootlimit` tries, fall back to A
#    f. Wrong compatible:    must refuse
rauc status
rauc info bundle.raucb
```

### Lab 99.5 — Build the SDK and use it

```bash
cd ~/yocto/build

cat >> conf/local.conf <<'EOF'
TOOLCHAIN_HOST_TASK:append = " nativesdk-cmake nativesdk-ninja nativesdk-gdb"
SDKIMAGE_FEATURES = "dev-pkgs dbg-pkgs"
EOF

bitbake myproduct-image -c populate_sdk
ls -lh tmp/deploy/sdk/

# Install and use it, as an application developer would:
./tmp/deploy/sdk/poky-*-toolchain-*.sh -y -d ~/mysdk
source ~/mysdk/environment-setup-*

echo $CC $CXX $CFLAGS $SDKTARGETSYSROOT
cat > /tmp/hello.c <<'EOF'
#include <stdio.h>
int main(void) { printf("hello from the SDK\n"); return 0; }
EOF
$CC -o /tmp/hello /tmp/hello.c
file /tmp/hello

# CMake:
cat > /tmp/CMakeLists.txt <<'EOF'
cmake_minimum_required(VERSION 3.10)
project(hello C)
add_executable(hello hello.c)
EOF
cd /tmp && cmake . && make && file hello

# The EXTENSIBLE SDK: an app developer can produce a recipe.
cd ~/yocto/build
bitbake myproduct-image -c populate_sdk_ext
./tmp/deploy/sdk/poky-*-ext-*.sh -y -d ~/myesdk
source ~/myesdk/environment-setup-*
devtool add myapp /path/to/my/source
devtool build myapp
devtool deploy-target myapp root@target
```

### Lab 99.6 — Archive for ten years

```bash
#!/bin/bash
# archive_release.sh — everything needed to rebuild in 2035.
set -e
VERSION=${1:?version}
ARCHIVE=~/archives/myproduct-$VERSION
mkdir -p $ARCHIVE/{sources,artefacts,env,docs}
cd ~/yocto/build

echo "=== 1. Mirror tarballs for EVERY source, including git ==="
cat >> conf/local.conf <<'EOF'
BB_GENERATE_MIRROR_TARBALLS = "1"
EOF
bitbake myproduct-image --runall=fetch
cp -r downloads/* $ARCHIVE/sources/
rm -f $ARCHIVE/sources/*.done
du -sh $ARCHIVE/sources

echo "=== 2. Exact layer revisions ==="
{
  for d in $(bitbake-layers show-layers | awk '/^[a-z]/ {print $2}'); do
      if [ -d "$d/.git" ]; then
          echo "$(basename $d) $(git -C $d remote get-url origin 2>/dev/null) $(git -C $d rev-parse HEAD)"
      fi
  done
} | tee $ARCHIVE/env/layer-revisions.txt

echo "=== 3. Configuration ==="
cp conf/local.conf conf/bblayers.conf $ARCHIVE/env/
bitbake -e myproduct-image > $ARCHIVE/env/full-datastore.txt

echo "=== 4. Build environment as a container ==="
cat > $ARCHIVE/env/Dockerfile <<'EOF'
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y --no-install-recommends \
    gawk wget git diffstat unzip texinfo gcc build-essential chrpath \
    socat cpio python3 python3-pip python3-pexpect xz-utils debianutils \
    iputils-ping python3-git python3-jinja2 python3-subunit zstd \
    liblz4-tool file locales libacl1 \
 && rm -rf /var/lib/apt/lists/*
RUN locale-gen en_US.UTF-8
RUN useradd -m builder
USER builder
WORKDIR /home/builder
EOF
docker build -t myproduct-build:$VERSION $ARCHIVE/env/
docker save myproduct-build:$VERSION | zstd -19 > $ARCHIVE/env/build-image.tar.zst

echo "=== 5. Artefacts ==="
cp tmp/deploy/images/myboard/*.wic.gz \
   tmp/deploy/images/myboard/*.manifest \
   tmp/deploy/images/myboard/*.bmap $ARCHIVE/artefacts/ 2>/dev/null || true
cp -r tmp/deploy/licenses $ARCHIVE/artefacts/
cp -r tmp/deploy/spdx $ARCHIVE/artefacts/ 2>/dev/null || true
cp -r tmp/deploy/cve $ARCHIVE/artefacts/ 2>/dev/null || true

echo "=== 6. Debug symbols, for field crash analysis ==="
cp tmp/deploy/ipk/*/​*-dbg_* $ARCHIVE/artefacts/ 2>/dev/null || true
cp tmp/work/*/linux-*/​*/linux-*-build/vmlinux $ARCHIVE/artefacts/ 2>/dev/null || true

echo "=== 7. The rebuild instructions ==="
cat > $ARCHIVE/docs/REBUILD.md <<EOF
# Rebuilding myproduct $VERSION

    docker load < env/build-image.tar.zst
    docker run -it -v \$PWD:/work myproduct-build:$VERSION

Inside the container:

    # Clone each layer at the exact revision in env/layer-revisions.txt
    source poky/oe-init-build-env build
    cp /work/env/local.conf /work/env/bblayers.conf conf/
    # Point at the archived sources and forbid network access:
    echo 'SOURCE_MIRROR_URL = "file:///work/sources"' >> conf/local.conf
    echo 'INHERIT += "own-mirrors"' >> conf/local.conf
    echo 'BB_NO_NETWORK = "1"' >> conf/local.conf
    bitbake myproduct-image
EOF

echo "=== 8. VERIFY THE ARCHIVE WORKS -- offline ==="
echo "Run the REBUILD.md procedure now, with the network disconnected."
echo "If it fails, the archive is incomplete. Fix it TODAY."

sha256sum -r $(find $ARCHIVE -type f) > $ARCHIVE/SHA256SUMS
du -sh $ARCHIVE
```

---

## 3. Mastery drills

1. Take a development image to production: harden it, add build-failing checks, verify on
   hardware, and document every difference.

2. Produce a complete compliance package for a real image and have someone unfamiliar with
   the project verify it satisfies GPLv2 §3.

3. Make your layer fully reproducible. Use `diffoscope` to find every source of
   non-determinism and fix each.

4. Implement A/B updates end to end and test all six failure paths from Lab 99.4.

5. Set up a complete CI: build, `testimage`, licence check, CVE check, reproducibility check,
   and artefact archival. Make each gate able to fail the pipeline.

6. Write runtime tests asserting ten security properties, and verify each fails when the
   property is violated.

7. Produce an SBOM and use it to answer, mechanically: does this image contain component X,
   what changed between two releases, and what licences are we shipping?

8. Build and distribute an SDK. Have an application developer with no Yocto knowledge build
   and deploy an application with it. Fix whatever they get stuck on.

9. Execute the ten-year archive procedure, then **actually rebuild from it on an air-gapped
   machine.** Report what was missing.

10. Implement anti-rollback with a monotonic counter and prove that a correctly-signed older
    bundle is refused.

11. Write the CVE-strategy memo (Ch. 88 Lab 4) specifically for a Yocto-based product,
    including how `cve-check` fits and what its limitations mean for the process.

12. Design the complete release process for a product: what is built, what is tested, what is
    archived, what is signed, who approves, and what the rollback plan is. Then execute it
    once end to end.

---

## 4. Further reading

**Documentation**
- Yocto Project Reference Manual — the `INCOMPATIBLE_LICENSE`, `SPDX_*`, `CVE_*`, and
  `ARCHIVER_*` variables
- Yocto Project Development Tasks Manual — the SDK, licence compliance, and testing chapters
- Yocto Project Test Environment Manual — `testimage` and `oe-selftest`
- `meta/classes-recipe/testimage.bbclass` and `meta/lib/oeqa/`

**Compliance**
- **Software Freedom Conservancy / FSF, "Copyleft Compliance Projects"** — the practical
  guidance, from the people who enforce
- The Linux Foundation OpenChain specification (ISO/IEC 5230) — the process standard
- SPDX specification — the SBOM format
- CycloneDX — the alternative SBOM format; know both exist
- Bradley Kuhn's writing on GPL compliance — the clearest available

**Regulation**
- US Executive Order 14028 and NTIA's "Minimum Elements for an SBOM"
- **EU Cyber Resilience Act** — read the obligations for "products with digital elements";
  this changes embedded Linux practice materially from 2027
- IEC 62443 (industrial), ISO 21434 (automotive), FDA premarket cybersecurity guidance
  (medical) — whichever applies to you

**Updates**
- RAUC documentation — the best-documented A/B updater; read the design rationale
- SWUpdate documentation
- Mender, OSTree/libostree, and `systemd-sysupdate` — the alternatives
- Chris Simmonds's talks on embedded update strategies

**Reproducible builds**
- `reproducible-builds.org` — the project, the tools, and the taxonomy of non-determinism
- `diffoscope` documentation
- The Yocto Project's reproducibility status and `oe-selftest -r reproducible`

**Cross-references**
- Ch. 88 — stable, CVE strategy, and delta management
- Ch. 90 §T.6 — secure boot, which the update mechanism must respect
- Ch. 94 §T.10 — backup, RPO/RTO, and testing your recovery
- Ch. 102 — the security properties the production image must have
- Ch. 101 — the full bring-up, of which this is the last phase

→ Next: [100-buildroot-alternatives.md](100-buildroot-alternatives.md)

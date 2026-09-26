# Chapter 03 — Kbuild & Kconfig Deep Dive

> **Goal:** you can add a new config symbol with correct dependencies, wire a multi-file
> driver into the build, write an out-of-tree module Makefile, and explain what
> `obj-$(CONFIG_X) += foo.o` becomes.

---

## Theory & First Principles

> **How to read this section.** T.0 is for someone who has only ever run `make`. T.1–T.4
> develop the theory — configuration as a product line, as a constraint system, and as a
> build-invalidation problem. T.5–T.9 are the practice and the arguments.

---

### T.0 — Start here: why is there a `.config` at all?

Most C projects have `./configure` or a `CMakeLists.txt` with a dozen options. The kernel has
**more than 20,000** of them. Before any theory, feel the scale:

```bash
cd ~/src/linux
make defconfig
wc -l .config                                    # ~2,500 lines
grep -c '^CONFIG_' .config                       # symbols actually set
grep -rho 'CONFIG_[A-Z0-9_]*' --include='*.c' --include='*.h' . | sort -u | wc -l
                                                 # symbols REFERENCED in code: ~20,000+
```

**Why so many?** Because one source tree must produce all of these, from the same commit:

| Target | Image size | What must be excluded |
|---|---|---|
| A router with 4 MiB of flash | ~1.5 MiB | almost everything |
| A phone | ~10 MiB | other SoCs, most filesystems, most of `drivers/` |
| A cloud VM | ~10 MiB | all physical drivers, replaced by virtio |
| A distribution kernel | ~12 MiB + **~5000 modules** | nothing — it must boot *any* machine |
| A supercomputer node | ~10 MiB | interactivity features; add NUMA, RDMA, RT |

One codebase, radically different products. That is the definition of a **software product
line** (§T.1), and 20,000 switches is what it costs.

**Now see the consequence that matters.** Those switches are not independent:

```bash
# Try to turn something off and watch Kconfig fight you:
./scripts/config --disable BLOCK
make olddefconfig
grep -E '^CONFIG_(EXT4_FS|BLK_DEV|SCSI)' .config    # gone: they DEPEND on BLOCK
```

With even a few thousand *binary* switches the configuration space exceeds the number of
atoms in the universe, so **nobody can test more than a vanishing fraction of it.** Two
consequences follow immediately, and they are the reason this chapter exists at all:

1. **Every `#ifdef` you add creates configurations nobody will ever compile**, let alone run.
   That is why §T.5's stub idiom — keeping *all* code paths compiled — is a hard rule rather
   than a style preference.
2. **The dependency system must be a real constraint solver**, not a list of flags. That is
   §T.2, and it is why `select` is dangerous: it is the one construct that *bypasses* the
   solver.

**One more thing to try before the theory**, because it makes §T.4 concrete:

```bash
make -j$(nproc) >/dev/null 2>&1                  # full build, once
./scripts/config --enable DEBUG_INFO_DWARF5      # change ONE symbol
make -j$(nproc) 2>&1 | grep -c '^  CC'           # how many files rebuilt?
```

If changing one symbol rebuilt the whole tree, incremental development would be impossible.
It does not, and the mechanism that prevents it (`fixdep`, §T.4) is one of the cleverest
250 lines in the source tree.

---

### T.1 The Linux kernel is a software product line

Linux is not one program. It is a **software product line**: a family of ~2^n related
programs generated from a common code base by selecting features. With ~20,000 Kconfig
symbols, the configuration space is larger than the number of atoms in the observable
universe, by an enormous margin.

This is the academic field of **variability management**, and Kconfig is a **feature model**
in exactly the sense of Kang et al.'s FODA method (1990). The research literature (Berger,
She, Lotufo, Wąsowski, Czarnecki — "A Study of Variability Models and Languages in the
Systems Software Domain", TSE 2013) analyses Kconfig formally and finds it is expressive
enough to encode arbitrary propositional constraints.

Three consequences that matter to you daily:

1. **You cannot test all configurations.** Not even close. Therefore
   *configuration-dependent bugs are a permanent, structural feature of kernel development*:
   code that compiles under `defconfig` and fails under `allmodconfig`, or under
   `allnoconfig`, or on an architecture where `CONFIG_SMP=n`. This is why `randconfig`
   build testing (kbuild test robot / Intel 0-day) exists and why it finds so much.
2. **Every `#ifdef` you add doubles a subspace.** The cost is not the three lines of code;
   it is the combinatorial burden on everyone who ever tests the kernel. Hence the strong
   preference for **static inline no-op stubs** over `#ifdef` at call sites (T.5 below).
3. **Constraints must be machine-checkable**, because humans cannot reason about 20,000
   interacting booleans. That's what the Kconfig language *is*.

```bash
cd linux
git ls-files '*Kconfig*' | wc -l                 # ~1500 Kconfig files
grep -rh '^config ' $(git ls-files '*Kconfig*') | wc -l   # ~20000 symbols
```

### T.2 Kconfig as a constraint system

Strip the syntax and Kconfig is a **constraint satisfaction problem** over a three-valued
lattice:

```
   n  <  m  <  y
```

Each symbol has a *type* (`bool`, `tristate`, `int`, `hex`, `string`) and constraints. The
core relations:

| Construct | Meaning (as logic) | Who is responsible |
|---|---|---|
| `depends on E` | this symbol's value ≤ E; it is **invisible** unless E | *the dependent* |
| `select S` | **force** S ≥ this symbol's value, **ignoring S's own dependencies** | *the selector* |
| `imply S` | set S if possible, but let the user override | *the selector* |
| `default v if E` | initial value | |
| `visible if E` | menu visibility without affecting the value | |
| `range a b` | numeric bound | |

**`select` is the dangerous one, and understanding why is a real interview question.**

`depends on` is a *constraint*: it restricts the solution space and the solver respects all
other constraints simultaneously. `select` is an *assignment*: it forces a value and
**deliberately ignores the selected symbol's own `depends on`**. Therefore:

```kconfig
config A
	bool "A"
	select B          # forces B=y

config B
	bool
	depends on C      # ...but C may be n!  →  B=y with C=n: an unsatisfiable state
```

The result is a `.config` that Kconfig itself considers inconsistent, and a build that fails
in a confusing place far from the cause. The kernel's own documentation says:
*"`select` should be used with care. `select` will force a symbol to a value without visiting
the dependencies."*

**The rules, which you will be held to in review:**

- Use `select` **only** for symbols that are pure "library" flags with no dependencies of
  their own — typically `bool` symbols under `lib/` that just enable compilation of a helper.
- Use `depends on` for anything with its own requirements, and let the user resolve it.
- If you must `select`, also add the selected symbol's dependencies to your own
  `depends on` — i.e. manually propagate the constraint. This is exactly the work the
  solver would have done, which tells you the design is fighting the tool.
- `imply` was added (4.10) precisely to give a "soft select" that *does* respect
  dependencies. Prefer it.

### T.3 Why tristate exists: the module boundary is a real constraint

`m` is not a convenience; it encodes a hard linking constraint:

> **Built-in code (`y`) cannot call into a module (`m`).**
> The kernel image is linked before modules exist.

So a `bool` symbol that depends on a `tristate` symbol creates the invalid combination
`FOO=y && BAR=m`. Kconfig's `depends on` handles this automatically because of the lattice
ordering (`y ≤ y`, but `y ≰ m`). This is why:

```kconfig
config MY_FEATURE
	bool "My feature"
	depends on SOME_MODULE          # if SOME_MODULE=m, MY_FEATURE is not offered
```
and why you frequently see the idiom:

```kconfig
	depends on USB || USB=n         # "if USB exists at all, we must not be built-in when it's m"
```
That `|| X=n` construction is the standard encoding of *"either X is available at my level,
or X is entirely absent"*. Recognizing it instantly marks you as someone who has read real
Kconfig files.

### T.4 The `.config` → build pipeline, and the invalidation problem

```
  Kconfig files  ──┐
                   ├─▶  conf/mconf  ──▶  .config
  user choices   ──┘                       │
                                           ├─▶ include/generated/autoconf.h   (#define CONFIG_*)
                                           ├─▶ include/config/*               (stamp files)
                                           └─▶ (Make variables CONFIG_* for Kbuild)
```

The interesting engineering is the **fine-grained invalidation problem** from Ch. 01 T.9.
Naively, changing any symbol invalidates `autoconf.h`, which every file includes, so *every*
file rebuilds. A 20,000-symbol config would make incremental builds useless.

Kbuild's solution, `scripts/basic/fixdep.c`:

1. GCC's `-MD` emits `foo.o.d` listing headers actually read.
2. `fixdep` post-processes it, scanning the *preprocessed source* for every `CONFIG_*`
   identifier that was actually consulted.
3. It rewrites the dependency to reference `include/config/<symbol>` — an **empty stamp
   file** whose mtime is touched only when that symbol's value changes.

Result: a file rebuilds iff a config symbol it actually uses changed. This is a
content-addressed invalidation scheme built on top of a timestamp-based one, and it is
genuinely clever. `fixdep.c` is ~250 lines — **read it**; it's one of the best
small-program-worth-studying examples in the tree.

```bash
cat .config | head
cat include/generated/autoconf.h | head
ls include/config/ | head                       # stamp files
cat fs/ext4/.inode.o.cmd | tr ' ' '\n' | grep include/config | head
```

### T.5 `#ifdef` considered harmful (and the stub idiom)

From T.1: every `#ifdef` at a *call site* creates an untested configuration. The kernel's
preferred technique is to keep conditional compilation **in headers**, so that call sites are
unconditional:

```c
/* include/linux/foo.h */
#ifdef CONFIG_FOO
int foo_register(struct foo *f);
void foo_unregister(struct foo *f);
#else
static inline int foo_register(struct foo *f) { return 0; }
static inline void foo_unregister(struct foo *f) { }
#endif
```

Now callers write `foo_register(f);` unconditionally, and with `CONFIG_FOO=n` the optimizer
deletes it entirely. Benefits, and they compound:

- **Both configurations get type-checked and compiled**, so `=n` builds don't bit-rot.
- Call sites stay readable.
- The `=n` path costs nothing at runtime.
- `IS_ENABLED(CONFIG_FOO)` gives you the same property inside a function body:
  ```c
  if (IS_ENABLED(CONFIG_FOO))
          foo_thing();      /* compiled always, eliminated when =n */
  ```
  Note `IS_ENABLED()` is true for `y` **or** `m`; `IS_BUILTIN()` and `IS_MODULE()` split them.

This is a general principle: **prefer mechanisms that keep all code paths compiled.** The
same reasoning motivates `static_branch` (runtime-patched branches), `dev_dbg` (always
compiled, `dynamic_debug`-controlled), and `WARN_ON` over `#ifdef DEBUG`.

### T.6 Kbuild's "recursive make" trade-off

Kbuild is recursive: the top-level Makefile descends into directories, each with its own
`Makefile` contributing to `obj-y`/`obj-m`. Peter Miller's *"Recursive Make Considered
Harmful"* (1997) enumerates the costs, and Kbuild pays some of them:

| Miller's complaint | Kbuild's situation |
|---|---|
| Incomplete dependency graph across directories | mitigated: dependency files are global; `fixdep` handles config |
| Cannot parallelize across the whole tree | mitigated: `make -j` does parallelize across subdirs via `$(MAKE)` job server |
| Repeated work / redundant rules | accepted |
| Order-dependence between directories | **real**: link order determines initcall order within a level, which is why `obj-y` ordering in `drivers/Makefile` is load-bearing |

That last row is important and under-appreciated: **the order of `obj-y` entries determines
initcall ordering within an initcall level**, because the linker concatenates
`.initcall<N>.init` sections in link order. So editing `drivers/Makefile` can change boot
behaviour. Code that *relies* on this is fragile and reviewers will ask you to use explicit
dependencies (deferred probe, Ch. 27) instead — but you must know the mechanism exists.

### T.7 Configuration as a correctness surface

Two failure classes that only exist because of configurability, both with real CVEs and both
worth watching for in review:

1. **Dead / unsatisfiable features.** A symbol whose dependencies can never all be true.
   Research tooling (`undertaker`, `vampyr`) has found hundreds of dead `#ifdef` blocks in
   Linux — code that is present in the tree and compiled *never*.
2. **Configuration-dependent semantics.** Code that is correct under `CONFIG_SMP=y` and
   racy under `=n` (or vice versa: `spin_lock()` compiles to nothing on UP, so a bug hidden
   by the lock becomes visible). `CONFIG_PREEMPT` variants, `CONFIG_HIGHMEM`,
   `CONFIG_MMU=n` (nommu) are classic sources.

Defensive practice: build `allmodconfig`, `allnoconfig`, `allyesconfig`, and several
`randconfig`s before posting a series. The 0-day bot will do it anyway, and it is better to
find it yourself.

```bash
for t in allnoconfig allmodconfig allyesconfig; do
  make O=b-$t $t && make O=b-$t -j$(nproc) 2>&1 | tail -5
done
make O=b-rand randconfig && make O=b-rand -j$(nproc)
# Reproducible randconfig for bisecting a config bug:
KCONFIG_SEED=0xdeadbeef make O=b-rand randconfig
```

---

### T.8 — What is still argued about

**1. Is 20,000 config symbols a feature or a disease?**
The defence: it is what lets one tree serve a 4 MiB router and a 400-core server, and that
breadth is Linux's entire competitive position. The prosecution: the configuration space is
untestable, dead `#ifdef` blocks number in the hundreds, and config-dependent bugs are a real
CVE class (§T.7). Nobody has proposed a credible alternative, which is itself informative.

**2. Should `select` be removed?**
It creates unsatisfiable states by design (§T.2), and `imply` was added specifically to offer
a safe alternative. But thousands of existing uses depend on it, and for genuine
no-dependency library symbols it is the right tool. The pragmatic position — restrict it by
convention and enforce in review — is where the community has landed, which means **you are
the enforcement mechanism.**

**3. Recursive make: still the right call?**
Miller's 1997 critique is largely correct in theory (§T.6) and largely mitigated in practice.
The real residual cost is the load-bearing `obj-y` ordering, which couples build files to
runtime behaviour — genuinely bad, and the reason deferred probe (Ch. 27 §T.5) exists.
Proposals to move to Ninja or Bazel resurface periodically and die on the size of the
migration, not on the merits.

**4. Should defconfigs be maintained at all?**
A `defconfig` rots: upstream adds symbols, and a checked-in defconfig silently keeps the old
default. The alternative — a base plus small fragments (Ch. 98 §T.4) — is strictly better for
products and is what Yocto and Android do. Checked-in full defconfigs in a BSP layer are a
reliable sign that nobody is maintaining the configuration.

### T.9 — The compressed model

```
 Linux is a SOFTWARE PRODUCT LINE. 20,000 switches, one tree, many products.
 The configuration space is astronomically larger than anything testable.

 Kconfig is a CONSTRAINT SYSTEM over n < m < y.
   depends on  = a constraint (the solver respects everything else)
   select      = an ASSIGNMENT that BYPASSES the solver  <- the dangerous one
   imply       = a soft select that respects dependencies

 TRISTATE ENCODES A LINKING FACT: built-in code cannot call a module.
 That is why `depends on X || X=n` exists.

 #ifdef AT A CALL SITE CREATES UNTESTED CONFIGURATIONS. Push conditionals
 into HEADERS with static-inline stubs, or use IS_ENABLED(), so every path
 is always COMPILED and the optimizer deletes the dead one.

 fixdep GIVES PER-SYMBOL BUILD INVALIDATION: a file rebuilds iff a symbol it
 actually consulted changed. Without it, incremental builds are impossible.
```

Five questions for any Kconfig change:

1. **`depends on` or `select`?** If the symbol has its own dependencies, it is `depends on`
   (or `imply`), not `select`.
2. **Does `=n` still compile?** Write the stub, not the `#ifdef` at the call site.
3. **Does `=m` make sense**, and if so does everything that calls it also work as `=m`?
4. **Have I built `allnoconfig`, `allmodconfig`, `allyesconfig`, and a `randconfig`?**
5. **Is this symbol user-visible, and if so is the help text good enough for someone who does
   not already know the answer?**

---

## 1. Concept

Two separate systems that meet at `.config`:

```
Kconfig files  ──►  conf/mconf  ──►  .config  ──►  syncconfig  ──┬─► include/generated/autoconf.h
(dependency                                                      ├─► include/config/*      (timestamps)
 language)                                                       └─► $(KCONFIG_CONFIG) read by Makefiles
                                                                        │
Kbuild Makefiles ───────────────────────────────────────────────────────┘
(obj-y / obj-m)  ──► scripts/Makefile.build ──► .o ──► built-in.a / *.ko ──► vmlinux
```

- **Kconfig** = a declarative constraint language. Answers *"can this feature be enabled?"*
- **Kbuild** = a thin DSL on top of GNU make. Answers *"what gets compiled?"*

---

## 2. Kconfig in depth

### 2.1 Anatomy

```kconfig
menuconfig MYSUBSYS
	bool "My subsystem support"
	depends on OF && !UML
	select REGMAP_MMIO
	imply CRC32
	default n
	help
	  Long help text, indented with TABs then two spaces.
	  Wrapped at 75 columns. Say Y if unsure... (actually say N if unsure)

if MYSUBSYS

config MYSUBSYS_FOO
	tristate "Foo backend"
	depends on I2C
	default MYSUBSYS
	help
	  Build the Foo backend. Builds as module named mysubsys-foo.

config MYSUBSYS_BUFSZ
	int "Buffer size in KiB"
	range 4 1024
	default 64

config MYSUBSYS_NAME
	string "Device name"
	default "mysubsys0"

choice
	prompt "Default scheduling policy"
	default MYSUBSYS_SCHED_FIFO

config MYSUBSYS_SCHED_FIFO
	bool "FIFO"

config MYSUBSYS_SCHED_RR
	bool "Round robin"

endchoice

endif # MYSUBSYS
```

### 2.2 Symbol types

| Type | Values | Generates |
|---|---|---|
| `bool` | `y` / `n` | `#define CONFIG_X 1` or nothing |
| `tristate` | `y` / `m` / `n` | `CONFIG_X=y` → `1`; `=m` → `CONFIG_X_MODULE 1` |
| `int` | integer | `#define CONFIG_X 64` |
| `hex` | `0x…` | `#define CONFIG_X 0x1000` |
| `string` | text | `#define CONFIG_X "name"` |

### 2.3 The dependency operators — **this is where people go wrong**

| Keyword | Meaning | Danger |
|---|---|---|
| `depends on A` | this symbol is invisible/unselectable unless A | ✅ safe, preferred |
| `select A` | **forcibly** enables A, *ignoring A's own `depends on`* | ⚠️ causes unmet-dependency warnings & broken builds |
| `imply A` | enables A by default but user can turn it off | ✅ soft version of select |
| `default A` | initial value | |
| `visible if A` | (menus) hide but keep value | |
| `range a b` | numeric bounds | |

**Rules of thumb (upstream-enforced):**
- Use `select` **only** for non-visible "library" symbols (e.g. `select CRC32`,
  `select REGMAP_I2C`) that have no prompt and minimal dependencies.
- Never `select` a symbol that has `depends on` clauses you don't also satisfy.
- Prefer `depends on` for anything with a user-visible prompt.

Classic failure:
```kconfig
config MY_DRIVER
	tristate "My driver"
	select SOME_FRAMEWORK      # SOME_FRAMEWORK depends on OF
# → if !OF, you get:
#   WARNING: unmet direct dependencies detected for SOME_FRAMEWORK
```
Fix: `depends on OF` too, or use `depends on SOME_FRAMEWORK`.

### 2.4 Tristate arithmetic

Kconfig has a 3-valued logic where `n < m < y`:

```kconfig
config A
	tristate
config B
	tristate
	depends on A          # B <= A   (B=y requires A=y; B=m allows A=m or y)
```

The canonical idiom for "my built-in driver must not call into a module":
```kconfig
config MY_DRIVER
	tristate "My driver"
	depends on SOME_OPTIONAL_THING || SOME_OPTIONAL_THING=n
```
This reads: *if `SOME_OPTIONAL_THING` is `m`, I must also be `m` (not `y`)*.
You will see this a lot; now you know what it means.

### 2.5 Where Kconfig files live and how they're included

```
Kconfig                       (top level)
  → arch/$(SRCARCH)/Kconfig
      → source "drivers/Kconfig"
          → source "drivers/net/Kconfig"
              → source "drivers/net/ethernet/Kconfig"
```

`source` takes a path relative to the **tree root**, and supports globs:
```kconfig
source "drivers/net/ethernet/*/Kconfig"
```

### 2.6 Generated artifacts

```bash
make O=b defconfig
ls b/include/generated/
#   autoconf.h        ← #define CONFIG_*  (included via -include)
#   rustc_cfg         ← --cfg CONFIG_* for Rust
#   uapi/linux/version.h
ls b/include/config/        # empty stamp files; make dependency tracking per-symbol
```

`include/config/` stamp files are how Kbuild rebuilds only the objects affected by a
single config change (`scripts/basic/fixdep` rewrites `.cmd` files to depend on them).

### 2.7 Config manipulation scripts

```bash
scripts/config --file b/.config --enable  CONFIG_KASAN
scripts/config --file b/.config --module  CONFIG_EXT4_FS
scripts/config --file b/.config --disable CONFIG_DEBUG_INFO_REDUCED
scripts/config --file b/.config --set-val CONFIG_LOG_BUF_SHIFT 21
scripts/config --file b/.config --set-str CONFIG_LOCALVERSION "-mykernel"
make O=b olddefconfig        # ALWAYS re-run this after scripts/config

scripts/kconfig/merge_config.sh -O b b/.config frag1.config frag2.config
scripts/diffconfig b/.config.old b/.config
make O=b savedefconfig && cat b/defconfig   # minimal reproducible config
```

`scripts/diffconfig` is the tool you'll use when bisecting a config regression.

---

## 3. Kbuild in depth

### 3.1 The core assignment

A `Makefile` in any directory:

```makefile
# Build into vmlinux (or into the parent's built-in.a)
obj-y                    += always.o

# Build as a module (foo.ko) when CONFIG_FOO=m, into vmlinux when =y, skip when =n
obj-$(CONFIG_FOO)        += foo.o

# Multi-file module: foo.ko made of a.o b.o c.o
obj-$(CONFIG_FOO)        += foo.o
foo-y                    := a.o b.o
foo-$(CONFIG_FOO_EXTRA)  += c.o
# (foo-objs and foo-m are older spellings; foo-y is current)

# Descend into a subdirectory
obj-$(CONFIG_BAR)        += bar/

# Library archive: only linked if a symbol is referenced
lib-y                    += helpers.o

# Host programs (run on build machine)
hostprogs                := mkfoo
always-y                 += $(hostprogs)

# Userspace programs for the target (e.g. selftests)
userprogs                := mytool

# Per-file compiler flags
CFLAGS_foo.o             += -DDEBUG -Wno-unused
CFLAGS_REMOVE_foo.o      += -pg          # exclude from ftrace
ccflags-y                += -I$(src)/include
asflags-y                += -Wa,--noexecstack
ldflags-y                +=
subdir-ccflags-y         += -Werror      # applies to this dir AND subdirs

# Disable instrumentation for this dir (used in early boot / entry code)
KASAN_SANITIZE           := n
KCSAN_SANITIZE           := n
UBSAN_SANITIZE           := n
KCOV_INSTRUMENT          := n
GCOV_PROFILE             := n
OBJECT_FILES_NON_STANDARD := y           # skip objtool
```

### 3.2 What `obj-y += foo.o` actually does

```
scripts/Makefile.build reads the Makefile
  → compiles foo.c → foo.o  (with .foo.o.cmd recording the exact command)
  → appends foo.o to this directory's built-in.a
  → parent directory's built-in.a includes the child's
  → scripts/link-vmlinux.sh links all built-in.a into vmlinux
```

For `obj-m`:
```
foo.o  →  modpost (scripts/mod/modpost) → foo.mod.c (metadata, versions, aliases)
       →  compile foo.mod.c → foo.mod.o
       →  ld -r foo.o foo.mod.o → foo.ko
       →  optional: sign (scripts/sign-file), compress (zstd/xz)
```

`modpost` is where:
- undefined symbols are checked against `Module.symvers`
- `MODULE_DEVICE_TABLE()` becomes `modalias` strings (→ `modules.alias` → udev autoload)
- section mismatches are detected (`__init` function called from non-`__init`)
- `MODULE_LICENSE()` is validated (missing → taint + build warning)

### 3.3 Variables you must know

| Variable | Meaning |
|---|---|
| `$(src)` | source directory of current Makefile |
| `$(obj)` | output directory of current Makefile |
| `$(srctree)` | tree root (source) |
| `$(objtree)` | tree root (output) |
| `$(M)` | external module directory |
| `$(KERNELRELEASE)` | e.g. `6.12.0-mykernel+` |

> **Since 6.x**, `$(src)` in an external module Makefile refers to the module's own dir.
> Older trees used `$(srctree)/$(src)`. Use `$(src)` for includes:
> `ccflags-y += -I$(src)/include`

### 3.4 Out-of-tree (external) modules

`Makefile`:
```makefile
obj-m += hello.o
hello-y := hello_main.o hello_sysfs.o

KDIR ?= /lib/modules/$(shell uname -r)/build
PWD  := $(CURDIR)

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

modules_install:
	$(MAKE) -C $(KDIR) M=$(PWD) modules_install

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean

# Useful extras
check:
	$(MAKE) -C $(KDIR) M=$(PWD) C=1 modules
compile_commands.json:
	$(MAKE) -C $(KDIR) M=$(PWD) compile_commands.json

.PHONY: all modules_install clean check
```

Cross-compiled:
```bash
make -C ~/build-arm64 M=$PWD ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- modules
```

**Kbuild vs Makefile:** if a directory has both `Kbuild` and `Makefile`, Kbuild wins for
the kernel build. Out-of-tree modules commonly ship a `Kbuild` for the obj-m rules and a
`Makefile` for the human-facing targets.

### 3.5 Adding a new driver to the tree — full checklist

Suppose you add `drivers/misc/acme/`:

1. `drivers/misc/acme/Kconfig`
```kconfig
config ACME_WIDGET
	tristate "ACME widget driver"
	depends on I2C && OF
	select REGMAP_I2C
	help
	  Driver for the ACME widget. Module will be called acme-widget.
```
2. `drivers/misc/acme/Makefile`
```makefile
obj-$(CONFIG_ACME_WIDGET) += acme-widget.o
acme-widget-y := core.o regs.o
acme-widget-$(CONFIG_DEBUG_FS) += debugfs.o
```
3. `drivers/misc/Kconfig`: add `source "drivers/misc/acme/Kconfig"`
4. `drivers/misc/Makefile`: add `obj-$(CONFIG_ACME_WIDGET) += acme/`
5. `MAINTAINERS`:
```
ACME WIDGET DRIVER
M:	Your Name <you@example.com>
L:	linux-kernel@vger.kernel.org
S:	Maintained
F:	Documentation/devicetree/bindings/misc/acme,widget.yaml
F:	drivers/misc/acme/
```
6. DT binding YAML in `Documentation/devicetree/bindings/` (Chapter 32).
7. `MODULE_DEVICE_TABLE(of, acme_of_match)` for autoloading.

### 3.6 Section attributes and why modpost yells at you

```c
static int __init  my_init(void);   /* .init.text  — freed after boot */
static void __exit my_exit(void);   /* .exit.text  — dropped if built-in */
static int __initdata counter;      /* .init.data */
static const struct x __initconst tbl[]; 
static int __ref borderline(void);  /* "I know what I'm doing", silences modpost */
__used  __maybe_unused  __always_inline  __noinline  noinline_for_stack
```

Section mismatch example:
```
WARNING: modpost: vmlinux.o: section mismatch in reference:
  acme_probe (section: .text) -> acme_setup (section: .init.text)
```
Meaning: `acme_probe` can run after boot (deferred probe, hotplug), but `acme_setup` was
freed. Real bug. Fix: remove `__init` from `acme_setup`.

### 3.7 The generated `.cmd` files

```bash
cat b/fs/ext4/.inode.o.cmd | tr ' ' '\n' | head -40
```
This shows the *exact* compiler invocation plus the dependency list. When you're confused
about which header or flag applies, read this file. `fixdep` post-processes it to add
`include/config/*` dependencies.

---

## 4. Practice

### Lab 3.1 — Add a config symbol and prove it works
Add `CONFIG_ACME_WIDGET` per §3.5, with a stub `core.c` that just prints. Build as `=y`
and `=m`. Confirm with `grep ACME b/include/generated/autoconf.h` and `modinfo acme-widget.ko`.

### Lab 3.2 — Break `select` on purpose
Make `ACME_WIDGET` `select REGMAP_I2C` but remove `depends on I2C`. Run
`make olddefconfig` with `CONFIG_I2C=n`. Read the unmet-dependency warning. Fix it two
different ways and compare.

### Lab 3.3 — Trigger a section mismatch
Mark a function `__init` and call it from `.probe`. Build. Read the modpost message.
Then use `scripts/mod/modpost -h` and look at `--section-mismatch`.

### Lab 3.4 — Config bisection
```bash
cp b/.config good.config
# enable 30 random options, break something
scripts/diffconfig good.config b/.config
```
Practice narrowing a config-dependent failure by halving the diff.

### Lab 3.5 — Read the build
```bash
make O=b V=1 fs/ext4/inode.o 2>&1 | head -1 | tr ' ' '\n'
make O=b V=2 fs/ext4/inode.o     # V=2 explains WHY a target was rebuilt
```

### Lab 3.6 — Prove the `fixdep` invalidation scheme (T.4)

```bash
cd linux && make O=b defconfig && make O=b -j$(nproc) fs/ext4/

# 1. What config symbols does one object depend on?
tr ' ' '\n' < b/fs/ext4/.inode.o.cmd | grep 'include/config' | sed 's|.*include/config/||' | sort -u

# 2. Touch a symbol it does NOT use — nothing should rebuild
touch b/include/config/usb.h 2>/dev/null || touch b/include/config/USB
make O=b fs/ext4/inode.o            # "make: Nothing to be done" or no recompile

# 3. Touch a symbol it DOES use
touch b/include/config/ext4_fs_posix_acl 2>/dev/null
make O=b fs/ext4/inode.o            # recompiles

# 4. Now read the implementation
$EDITOR scripts/basic/fixdep.c      # ~250 lines; read the header comment first
```

### Lab 3.7 — Measure the configuration space

```bash
cd linux
# How many symbols, by type?
grep -rh -A3 '^config ' $(git ls-files '*Kconfig*') \
 | grep -E '^\s+(bool|tristate|int|hex|string)' | awk '{print $1}' | sort | uniq -c

# How many use select (the dangerous one)?
grep -rh '^\s*select ' $(git ls-files '*Kconfig*') | wc -l
grep -rh '^\s*imply ' $(git ls-files '*Kconfig*') | wc -l

# Find the |X=n idiom in the wild and explain three instances:
grep -rn 'depends on .*|| .*=n' $(git ls-files '*Kconfig*') | head -10

# Which symbols have the most dependents?
grep -rho 'depends on [A-Z0-9_]*' $(git ls-files '*Kconfig*') \
 | awk '{print $3}' | sort | uniq -c | sort -rn | head -20
```

### Lab 3.8 — Find a configuration-dependent bug (T.7)

```bash
# Build the four extremes and diff the warning sets.
for t in allnoconfig tinyconfig allmodconfig; do
  make O=b-$t $t >/dev/null && \
  make O=b-$t -j$(nproc) 2>&1 | grep -E 'warning:|error:' > /tmp/$t.warn
  echo "$t: $(wc -l < /tmp/$t.warn) diagnostics"
done
diff /tmp/allnoconfig.warn /tmp/allmodconfig.warn | head -20

# Randconfig loop — this is literally what the 0-day bot does:
for i in $(seq 1 5); do
  make O=b-r$i randconfig >/dev/null
  make O=b-r$i -j$(nproc) >/tmp/r$i.log 2>&1 || echo "FAILED: b-r$i (config saved)"
done
```
If you find a real build failure, **that is a publishable patch**. Configuration-dependent
build breaks are among the easiest first contributions and are always welcome.

### Lab 3.9 — Convert an `#ifdef` to the stub idiom (T.5)

```bash
# Find an #ifdef at a call site (not in a header):
git grep -n -B2 -A6 '#ifdef CONFIG_' -- drivers/ | grep -A6 '\.c[-:]' | head -40
```
Pick one, move the conditional into the header as `static inline` stubs, and verify:
1. `CONFIG_X=y` build is byte-identical (`objdump -d` diff).
2. `CONFIG_X=n` build now *compiles* the call site (check with `-Wunused`).
3. `size` of the `=n` object is unchanged (the optimizer removed it).

This is a very common, very welcome cleanup patch. `git log --grep="IS_ENABLED" --oneline`
shows hundreds of precedents.

### Lab 3.10 — Trace an initcall ordering dependency (T.6)

```bash
# Boot with initcall_debug (Ch. 00 Lab 0.10), then:
dmesg | grep -E 'calling|initcall' | head -40

# Now find where link order is decided:
sed -n '1,60p' drivers/Makefile
# Move an entry and observe the initcall order change (in a VM!).
grep -n 'INIT_CALLS\|__initcall._init' include/asm-generic/vmlinux.lds.h | head
```

---

## 5. Mastery drills

1. Explain, precisely, what `obj-$(CONFIG_FOO) += bar/` does when `CONFIG_FOO=m`.
2. Find three places in the tree that use `depends on X || X=n` and explain each.
3. Read `scripts/Makefile.build` and identify where `CFLAGS_$(target).o` is applied.
4. Read `scripts/mod/modpost.c`, function `check_sec_ref()`. Explain the whitelist logic.
5. Write a Kconfig fragment that makes a driver buildable only when
   (ARM64 **or** RISC-V) **and** (OF) **and** not under `COMPILE_TEST`, and explain why
   `COMPILE_TEST` exists and why maintainers love it.
6. Add `compile_commands.json` support to an out-of-tree module and get clangd working.
7. **`select` forensics.** `git log --oneline --grep='unmet direct dependencies' | head -20`.
   Read five. For each, state whether the fix was to add `depends on`, convert to `imply`,
   or restructure. Derive your own rule for when `select` is acceptable.
8. **Feature-model reasoning.** Given `A select B`, `B depends on C`, `C depends on !A` —
   prove no satisfying assignment exists with `A=y`. Then find whether Kconfig detects it,
   and what it prints. Explain why `select` makes this class of bug possible at all.
9. **Read `scripts/kconfig/`.** Specifically `symbol.c:sym_calc_value()` and
   `expr.c`. Explain how tristate values propagate through `&&`/`||` on the `n<m<y` lattice.
10. **Write a Kconfig for a real feature.** Design the config for a hypothetical driver that
    needs: I2C or SPI (at least one), optional regmap, optional DT, requires `CONFIG_IIO`,
    and should be `COMPILE_TEST`-able on any architecture. Write it, then verify with
    `make menuconfig` that every intended combination is reachable and no unintended one is.
11. **Build-time measurement.** Instrument a full build with `make -j1 V=1` piped through a
    timestamping script. Find the 20 slowest single compilations in the tree and explain
    what makes them slow (hint: header bloat; try `-H` to dump the include tree).

---

## 6. Further reading

**Kernel documentation:**
- `Documentation/kbuild/kconfig-language.rst` ★ — normative; read `select` section twice
- `Documentation/kbuild/kconfig-macro-language.rst`
- `Documentation/kbuild/kconfig.rst` — the tooling (`menuconfig`, `KCONFIG_*` env vars)
- `Documentation/kbuild/makefiles.rst` ★★ (the authoritative reference)
- `Documentation/kbuild/modules.rst` (out-of-tree)
- `Documentation/kbuild/kbuild.rst` (env vars)
- `Documentation/kbuild/reproducible-builds.rst`
- `scripts/Makefile.build`, `scripts/Makefile.lib`, `scripts/link-vmlinux.sh`,
  `scripts/basic/fixdep.c`, `scripts/kconfig/symbol.c`

**Research on kernel variability (T.1, T.7) — genuinely useful, not just academic:**
- Kang et al., "Feature-Oriented Domain Analysis (FODA)" (SEI, 1990) — feature models
- She, Lotufo, Berger, Wąsowski, Czarnecki, "Reverse Engineering Feature Models"
  (ICSE 2011) — on the Linux Kconfig model specifically
- Berger et al., "A Study of Variability Models and Languages in the Systems Software
  Domain" (IEEE TSE 2013) — Kconfig vs CDL, formal semantics
- Tartler, Lohmann, Sincero, Schröder-Preikschat, "Feature Consistency in Compile-Time
  Configurable System Software" (EuroSys 2011) — the `undertaker` tool; found real dead
  code in Linux
- Nadi, Berger, Kästner, Czarnecki, "Mining Configuration Constraints" (ICSE 2014)
- Melo et al., "A Quantitative Analysis of Variability Warnings in Linux" (2016) — measures
  exactly the T.7 problem

**Build-system theory:**
- Miller, "Recursive Make Considered Harmful" (1997)
- Mokhov, Mitchell & Peyton Jones, "Build Systems à la Carte" (ICFP 2018)

**LWN:**
- "Kconfig complexity" / "A new Kconfig?" discussions
- "The 0-day kernel test robot" — how randconfig testing actually works
- "Toward better configuration management"

→ Next: [04-qemu-boot-lab.md](04-qemu-boot-lab.md)

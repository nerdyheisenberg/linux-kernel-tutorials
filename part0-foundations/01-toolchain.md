# Chapter 01 — Toolchain, Cross-Compilation, Reproducible Builds

> **Goal:** you can build any kernel, for any architecture, from any host, with full debug
> info, in under 5 minutes of wall time, and you understand every tool in the chain.

---

## Theory & First Principles

> **How to read this section.** T.0 assumes you have only ever typed `gcc hello.c`. T.1–T.5
> build the model a working kernel engineer needs. T.6–T.11 are what you are expected to
> reason about in review and in design.

---

### T.0 — Start here: what `gcc hello.c` actually did

You have typed this a thousand times. You have probably never looked inside it.

```bash
$ echo 'int main(void){return 0;}' > /tmp/h.c
$ gcc -v /tmp/h.c -o /tmp/h 2>&1 | grep -E 'cc1|collect2|as ' | head
```

Four separate programs ran, in sequence:

```
 h.c
  │
  ① cpp        (preprocessor)  ── #include, #define, #if.  Pure TEXT substitution.
  ▼           h.i    (~800 lines for a 1-line program, from stdio.h alone)
  ② cc1        (compiler)      ── C  ->  target assembly.  ALL optimization here.
  ▼           h.s
  ③ as         (assembler)     ── assembly -> machine code + a SYMBOL TABLE.
  ▼           h.o             relocatable: "call printf" is a HOLE to be filled
  ④ ld         (linker)        ── resolve holes, lay out sections, pick addresses.
  ▼           h               executable
```

**Run each stage yourself.** This takes two minutes and it removes the magic permanently:

```bash
gcc -E /tmp/h.c | wc -l          # preprocessed: how much text is really there
gcc -S /tmp/h.c -o -             # the assembly your C became
gcc -c /tmp/h.c -o /tmp/h.o      # object file, unlinked
nm /tmp/h.o                      # its symbols:  T = defined, U = undefined (a hole)
readelf -S /tmp/h.o              # its sections: .text .data .bss .rodata
```

**The five facts that the whole chapter rests on:**

1. **The preprocessor is dumb text substitution**, and it runs before anything understands C.
   That is why kernel macros are full of `do { } while (0)` and paranoid parenthesization
   (Ch. 08 §T.2), and why `#ifdef` creates configurations nobody compiles (§T.4, Ch. 03 §T.5).
2. **The compiler optimizes one translation unit at a time**, and it assumes your code has no
   undefined behaviour. Those two facts together are where kernel miscompilations come from
   (§T.4).
3. **The object file is a *relocatable* thing with holes**, and a symbol table naming them.
   `insmod` is a linker that fills those holes at runtime (Ch. 05 §T.1) — modules are not a
   special kernel feature so much as ordinary linking, done late.
4. **The linker chooses addresses and section layout.** The kernel exploits this heavily:
   initcall ordering (Ch. 03 §T.6), `__init` sections freed after boot, per-CPU sections,
   and the `vmlinux.lds.S` script that controls it all.
5. **libc was linked in** and you did not ask for it. **The kernel has no libc** (§T.2). That
   single absence explains `printk`, `kmalloc`, the kernel's own string functions, and why
   `-ffreestanding` exists.

**The one-line summary to carry forward:** a toolchain turns text into an object with holes,
and then fills the holes. Everything the kernel does unusually — freestanding mode, custom
linker scripts, late linking, section tricks, plugins — is a variation on that sentence.

---

### T.1 The compilation model: why a toolchain has this shape

C's compilation model is **separate compilation with late binding of names**, and every tool
in the chain exists to serve one step of it:

```
  foo.c ──cpp──▶ foo.i ──cc1──▶ foo.s ──as──▶ foo.o ──ld──▶ vmlinux ──objcopy──▶ bzImage
       preprocess     compile        assemble      link             extract/compress
       (textual)      (semantic)     (encoding)    (relocation)
```

The key theoretical idea is **relocation**. A `.o` file is compiled without knowing where in
memory it will live or where the functions it calls are. So the assembler emits placeholders
and a **relocation table** saying "at offset X, patch in the address of symbol S, using
encoding R". The linker resolves symbols across all objects and applies the relocations.

```bash
readelf -r fs/read_write.o | head -20        # relocation entries
readelf -s fs/read_write.o | head -20        # symbol table: UND = to be resolved
nm -u fs/read_write.o                        # undefined symbols = this file's dependencies
```

This matters for the kernel in three specific ways:

1. **Modules are relocatable objects resolved at runtime.** `insmod` *is* a linker
   (Ch. 05). The relocation types your architecture supports determine how far a module can
   be from the kernel text — which is why arm64 needs a module PLT and why x86-64 keeps
   modules within ±2 GiB of the kernel (the `R_X86_64_PC32` reach).
2. **KASLR** works by choosing the load address at boot and re-applying relocations, which is
   why the kernel is built with `--emit-relocs` and carries a relocation table into runtime.
3. **The linker script** (`arch/*/kernel/vmlinux.lds.S`) is not boilerplate — it defines
   section ordering, initcall levels, per-CPU areas, and the `__init` regions that get freed.
   Read it; it is the kernel's memory map in source form.

### T.2 Freestanding vs hosted: the kernel is not a C program in the usual sense

The C standard (§4, §5.1.2) defines two conformance modes:

| | Hosted | Freestanding |
|---|---|---|
| Entry point | `main()` | implementation-defined |
| Standard library | all of it | only `<float.h> <iso646.h> <limits.h> <stdarg.h> <stdbool.h> <stddef.h> <stdint.h>` |
| Startup/teardown | C runtime runs constructors, `atexit` | none |

The kernel is **freestanding** (`-ffreestanding` is implied by `-nostdinc` + its own headers).
Consequences that are not stylistic but *semantic*:

- No `libc`. `printk` is not `printf` with a different name; it has different format
  specifiers (`%pS`, `%pK`, `%pI4`), different buffering, and different context rules.
- No `atexit`, no global constructors (except the explicit initcall mechanism, which is the
  kernel *re-inventing* constructors with explicit ordering because implicit ordering is
  unusable at this scale).
- `memcpy`/`memset` still exist because the compiler may *emit calls to them* even in
  freestanding mode — so the kernel must provide them (`lib/string.c`, `arch/*/lib/`).
  This is a subtle and frequently-surprising fact: you cannot escape libc's symbols entirely.

### T.3 The ABI is a contract, and the kernel amends it

An **ABI** (Application Binary Interface) specifies: how arguments are passed, which registers
are caller- vs callee-saved, how the stack is laid out, struct layout and padding, name
mangling (none in C), and how the OS enters your code. On x86-64 that's the System V AMD64
psABI; on ARM64 the AAPCS64.

The kernel deliberately **deviates** from the userspace ABI in ways you must know:

| Deviation | Flag | Why |
|---|---|---|
| **No red zone** | `-mno-red-zone` | the SysV ABI lets leaf functions use 128 bytes below `rsp` without adjusting it. An interrupt pushes onto the same stack and **corrupts it**. |
| **No SSE/AVX/FP** | `-mno-sse -mno-mmx -mgeneral-regs-only` | FPU state isn't saved on kernel entry; using it silently corrupts userspace FP state. Explicit `kernel_fpu_begin()/end()` required. |
| **Different syscall convention** | — | Linux syscalls use `rdi, rsi, rdx, r10, r8, r9` — note **`r10` not `rcx`**, because `syscall` clobbers `rcx` with the return address. |
| **Frame pointers** | `-fno-omit-frame-pointer` (optional) | reliable stack unwinding for ftrace/perf, at ~1–5% cost. ORC unwinder (`CONFIG_UNWINDER_ORC`) gets reliability *without* the cost by building a side table. |
| **Own stack protector** | `-mstack-protector-guard=...` | the canary lives in a per-CPU/per-task location, not TLS. |

**The red-zone point is worth internalizing**: it is a case where a perfectly valid, standard
ABI feature is *catastrophically unsafe* in an environment where the stack can be used
asynchronously. That class of reasoning — "this optimization assumes nothing else touches my
stack; is that true here?" — recurs constantly in kernel work.

### T.4 Undefined behaviour is a security problem, not a portability problem

In userspace, UB usually means "works until it doesn't". In the kernel, UB means the compiler
may **delete your safety checks**, and that becomes a CVE.

The canonical example, a real 2009 Linux vulnerability (CVE-2009-1897):

```c
struct sock *sk = tun->sk;        /* dereference */
...
if (!tun)                          /* NULL check — AFTER the deref */
	return -EBADF;
```
The compiler reasoned: "`tun` was dereferenced, so by the standard it cannot be NULL, so
`!tun` is always false" — and **deleted the check**. On a system where page 0 could be mapped
by an attacker (`mmap_min_addr=0`), this became a root exploit.

The kernel's response is to *narrow the C abstract machine* by disabling the optimizations
that turn UB into miscompilation:

| Flag | Disables |
|---|---|
| `-fno-strict-aliasing` | type-based alias analysis. The kernel casts between types constantly (`container_of`, network headers over `skb->data`). |
| `-fno-strict-overflow` / `-fwrapv` | assuming signed overflow can't happen. Makes signed overflow wrap, so overflow checks work. |
| `-fno-delete-null-pointer-checks` | the exact bug above. |
| `-fno-allow-store-data-races` | inventing stores (Ch. 13, T.5). |
| `-fno-common` | tentative definitions merging silently across TUs. |

**This is a profound design decision worth understanding:** the kernel does not write "correct
standard C and hope". It *contractually redefines* the language via compiler flags, producing
a dialect with stronger guarantees than ISO C. That is why kernel code that looks like UB
often isn't — within the kernel's dialect. And it is why you cannot reason about kernel C
using a C-standard-lawyer mindset alone; you must know which flags are in force.

Complementing this, `CONFIG_UBSAN` instruments the *remaining* UB at runtime:
array bounds, shifts, signed overflow, misaligned access, `__builtin_unreachable`.

### T.5 What the optimizer actually does, and why `-O2` is mandatory

The kernel **cannot be built at `-O0`**. This is not a preference:

- `static inline` functions in headers would become real calls, and many of them use
  `__builtin_constant_p()` or arch-specific inline asm that only works when inlined.
- `BUILD_BUG_ON()` relies on dead-code elimination: it expands to a negative-sized array (or
  `__compiletime_error`) inside a branch the optimizer must prove unreachable.
- Several `asm goto` and alternatives-patching constructs require constant folding.

`CONFIG_CC_OPTIMIZE_FOR_SIZE` (`-Os`) vs performance (`-O2`) is a real trade: `-Os` can be
*faster* on I-cache-bound workloads (smaller footprint → fewer misses) and is standard for
embedded. Measure; don't assume.

**LTO** (`CONFIG_LTO_CLANG`) defers code generation to link time so the optimizer sees the
whole program: cross-TU inlining, better devirtualization of function pointers, dead-code
elimination across files. Costs: much more link RAM/time, harder debugging, and it interacts
badly with code that depends on *not* being optimized across boundaries (which is why LTO
required auditing inline asm and `noinstr` sections). It is also a **prerequisite for
Clang CFI**, because CFI needs whole-program knowledge of function-pointer types.

### T.6 Hardening as a compiler feature

Modern kernel security is substantially implemented *in the toolchain*:

| Feature | Mechanism | Cost |
|---|---|---|
| **Stack protector** (`CC_STACKPROTECTOR_STRONG`) | canary between locals and return address, checked on return | ~1–3% |
| **Shadow call stack** (arm64) | return addresses kept in a separate, hidden stack in `x18` | ~1% |
| **kCFI** (`CONFIG_CFI_CLANG`) | hash of the function *type* checked before every indirect call | ~1–2%, needs LTO |
| **KASLR** | randomize kernel base at boot; needs `--emit-relocs` | ~0 |
| **FGKASLR** / function-granular | randomize function order | larger |
| **`-ftrivial-auto-var-init=zero`** | initialize all locals | small; kills a whole bug class |
| **`__counted_by`** (Clang 18+) | annotate flex-array length for runtime bounds checks | needs UBSAN |
| **`fortify-source`** | compile-time + runtime `memcpy`/`strcpy` bounds checks | small |

The theoretical framing: these are all attacks on **exploit primitives**, not on bugs.
A stack canary doesn't fix the overflow; it makes the overflow non-exploitable via return
address overwrite. CFI doesn't fix the UAF; it makes the freed-object function pointer
unusable for arbitrary control flow. This is **defense in depth as an economic strategy** —
raise the cost of weaponizing a bug above the attacker's budget. The Kernel Self-Protection
Project (KSPP) is the organizing effort; knowing its roadmap is expected of a senior engineer.

### T.7 Cross-compilation and the bootstrap problem

A toolchain is described by a **target triplet** (really a quadruple):

```
  aarch64 - unknown - linux - gnu
  │         │         │       └── ABI / libc  (gnu, musl, gnueabi, gnueabihf, elf, android)
  │         │         └────────── OS          (linux, none, freebsd)
  │         └──────────────────── vendor      (usually ignored)
  └────────────────────────────── architecture
```

Three distinct machines exist in any build, and confusing them is the #1 cross-compilation
error:

| Name | Meaning |
|---|---|
| **build** | where the compiler is *compiled* |
| **host** | where the compiler *runs* |
| **target** | what the compiler *produces code for* |

A normal compiler: build = host = target. A cross compiler: host ≠ target. A *Canadian cross*
(used by Yocto's SDK, Ch. 99): build ≠ host ≠ target.

**The kernel is the easy case** — it is freestanding, so a cross toolchain for the kernel
needs only `gcc` + `binutils` + kernel headers, *no libc at all*. Building a full userspace
cross toolchain requires the chicken-and-egg bootstrap: you need libc to build gcc, and gcc to
build libc. The standard resolution is a three-stage build (bootstrap gcc without libc →
build libc → rebuild full gcc). Crosstool-NG, Buildroot, and Yocto all automate exactly this,
and Ch. 92/99 return to it.

`ARCH=` and `CROSS_COMPILE=` are the kernel's entire cross-compilation interface. Clang
simplifies it further: one binary targets everything via `--target=`, hence
`make LLVM=1 ARCH=arm64`.

### T.8 Reproducible builds: determinism as a security property

A build is **reproducible** if the same source + same toolchain ⇒ bit-identical output.
Why this matters is not aesthetics — it is the **trusting-trust** problem (Ken Thompson,
"Reflections on Trusting Trust", Turing Award lecture 1984): if you cannot independently
verify that a binary corresponds to its source, a compromised build machine is undetectable.

Reproducibility lets *anyone* rebuild and compare, turning a single trusted builder into an
N-of-M verification problem. It also makes `ccache`/`sccache` correct, makes bisection
meaningful, and makes "did this config change actually change anything?" answerable.

Sources of nondeterminism the kernel had to eliminate:

| Source | Fix |
|---|---|
| Build timestamp/host/user in the banner | `KBUILD_BUILD_TIMESTAMP`, `_USER`, `_HOST`, `SOURCE_DATE_EPOCH` |
| `__FILE__` containing absolute paths | `-fdebug-prefix-map`, `-ffile-prefix-map` |
| Randomized struct layout | `CONFIG_GCC_PLUGIN_RANDSTRUCT` with a fixed seed |
| Module signing keys generated per build | pre-generate and pin the key |
| Filesystem readdir order | sorted wildcards in Kbuild |
| `bzImage` compression metadata | deterministic compressors |

```bash
# Verify it yourself:
export KBUILD_BUILD_TIMESTAMP='Thu Jan  1 00:00:00 UTC 1970'
export KBUILD_BUILD_USER=user KBUILD_BUILD_HOST=host
make -j$(nproc) && sha256sum vmlinux
make mrproper && <rebuild identically> && sha256sum vmlinux    # must match
```

### T.9 The build as a dependency graph

Kbuild is a **dependency-tracking incremental build system**, and its correctness requirement
is precise: *after any source change, everything that transitively depends on it must be
rebuilt, and nothing else.* Under-approximation gives you stale binaries (the worst kind of
bug — you debug code that isn't running). Over-approximation costs time.

The tricky dependencies in a kernel build:

1. **Header dependencies** — handled by `gcc -MD` generating `.foo.o.d` files.
2. **Config dependencies** — the hard one. If `CONFIG_FOO` changes, every file that *tests*
   `CONFIG_FOO` must rebuild, but files that don't reference it must not. Kbuild solves this
   with `scripts/basic/fixdep`, which post-processes the depfile, finds every
   `CONFIG_*` symbol the compilation actually consulted, and rewrites the dependency to point
   at `include/config/foo` — an empty file touched only when that symbol's value changes.
   **This is a genuinely clever solution to a fine-grained-invalidation problem** and is
   worth reading (`scripts/basic/fixdep.c`, ~200 lines).
3. **Generated files** — syscall tables, `asm-offsets.h`, device-tree binaries, `.mod.c`.

Recommended reading on the theory: *"Build Systems à la Carte"* (Mokhov, Mitchell &
Peyton Jones, ICFP 2018) classifies build systems by their scheduler and rebuilder;
Kbuild is a "topological scheduler + verifying-traces rebuilder". Also Peter Miller's
*"Recursive Make Considered Harmful"* (1997) — Kbuild is recursive, and the article explains
the costs it pays for that (incomplete dependency graphs across directories).

### T.10 — What is still argued about

**1. GCC or Clang?**
Both build the kernel; both are supported; the tree is tested with both. Clang brings CFI,
KCSAN's full capability, better sanitizer coverage, and the fact that Rust uses LLVM too —
so `LLVM=1` builds avoid a class of cross-frontend mismatch (Ch. 77 §T.1). GCC brings the
plugin infrastructure (`structleak`, `randstruct`, `stackleak`), broader architecture
support, and two decades of kernel-specific bug-compatibility. Android and ChromeOS ship
Clang-built kernels; most distributions ship GCC-built. **The honest answer is that this is
now a genuine choice rather than a default**, and the deciding factor is usually which
hardening features you need.

**2. Is `-O2` versus `-Os` a real trade any more?**
`-Os` produces smaller images and is tempting for embedded. But it disables inlining
decisions the kernel's macro-heavy code depends on, and `CONFIG_CC_OPTIMIZE_FOR_SIZE` has
historically exposed bugs that `-O2` hides. Meanwhile branch density, not code size, often
dominates on modern I-caches. Measure before choosing; the folklore on both sides is old.

**3. Frame pointers: 1% throughput or usable profiles?**
`-fomit-frame-pointer` was the default for a decade. Fedora and Ubuntu re-enabled frame
pointers in 2023–24 after concluding that unprofitable-to-profile production systems cost
more than ~1%. This is a clean example of a *whole-fleet* cost being weighed against a
*per-instruction* cost, and it is worth being able to argue either side (Ch. 95 §T.3).

**4. Should the kernel depend on compiler-specific behaviour at all?**
The kernel relies on statement expressions, `typeof`, `__builtin_*`, inline asm,
transparent unions, and a dozen other extensions. This makes it effectively unportable to a
standards-only compiler — and there is a recurring argument that this is a self-inflicted
constraint. The counter-argument is that the extensions buy real type-safety
(`container_of`'s `__same_type` check, Ch. 08 §T.3) that standard C cannot express.

### T.11 — The compressed model

```
 A toolchain turns TEXT into an OBJECT WITH HOLES, then FILLS THE HOLES.
   cpp (dumb text) -> cc1 (all optimization, one TU at a time) -> as -> ld

 The kernel is FREESTANDING: no libc, no standard startup, its own linker
 script. Everything unusual about the kernel build follows from that.

 The compiler assumes YOUR CODE HAS NO UNDEFINED BEHAVIOUR, and optimizes on
 that assumption. In the kernel, a miscompilation from UB is a SECURITY BUG,
 not a portability bug. Hence -fno-strict-aliasing, -fno-strict-overflow,
 -fno-delete-null-pointer-checks.

 The ABI is a CONTRACT the kernel AMENDS: no red zone, no FPU/SIMD, a custom
 stack protector, mcount/fentry hooks, and its own calling-convention quirks.

 A BUILD IS A DEPENDENCY GRAPH. Under-approximate it and you debug code that
 isn't running -- the worst bug class there is.
```

Five questions for any build problem you will ever hit:

1. **Which stage failed** — preprocess, compile, assemble, or link? The error's vocabulary
   tells you (`undefined reference` is always the linker).
2. **Is the toolchain the one I think?** `make V=1` and read the actual command line.
3. **Is this a stale artefact?** If the symptom makes no sense, `touch` the source or
   `make clean` and see whether it changes.
4. **Is this UB the optimizer exploited?** Try `-O0`, then `-fno-strict-aliasing`. If the
   behaviour changes, you have your answer (§T.4).
5. **Does it build under `allmodconfig`, `allnoconfig`, `W=1`, and a cross-compiler?** Most
   toolchain bugs are configuration bugs wearing a disguise (Ch. 03 §T.7).

---

## 1. Concept

### 1.1 What building a kernel actually requires

A kernel build is unusual compared to normal C projects:

- **Freestanding** — `-ffreestanding -nostdinc`. No libc, no `stdio.h`. Headers come only
  from `include/`, `arch/*/include/`, and the compiler's own `include` (for `stdarg.h` etc.).
- **No standard startup** — custom linker script (`arch/*/kernel/vmlinux.lds.S`), custom
  entry symbol.
- **Position-dependent (mostly)** — with optional `CONFIG_RELOCATABLE` + KASLR.
- **Self-hosted generators** — the build first compiles host tools (`scripts/`), then targets.
- **Two-phase** — `vmlinux` (ELF) → arch-specific compressed image (`bzImage`, `Image.gz`,
  `zImage`, `uImage`).

### 1.2 The toolchain triangle

```
   BUILD machine        HOST machine         TARGET machine
   (where you compile)  (where tools run)    (where the kernel runs)
        x86_64      →       x86_64        →      aarch64
                         (scripts/ tools)      (vmlinux, *.ko)
```

Kbuild variables:

| Variable | Meaning |
|---|---|
| `ARCH` | Target architecture directory under `arch/` (`x86`, `arm64`, `riscv`, `powerpc`) |
| `CROSS_COMPILE` | Prefix for the target toolchain, e.g. `aarch64-linux-gnu-` |
| `HOSTCC` / `HOSTCXX` | Compiler for `scripts/` host tools |
| `LLVM=1` | Use the entire LLVM toolchain (clang, ld.lld, llvm-objcopy…) |
| `O=` | Out-of-tree build directory (**always use this**) |
| `KBUILD_BUILD_TIMESTAMP` | For reproducible builds |

---

## 2. Internals

### 2.1 GCC vs Clang

Both are first-class upstream since ~5.15 (ClangBuiltLinux). Differences that matter:

| | GCC | Clang/LLVM |
|---|---|---|
| Default for most distros | ✅ | Android, ChromeOS |
| LTO | `CONFIG_LTO_GCC` (limited) | `CONFIG_LTO_CLANG_FULL/THIN` ✅ |
| CFI | — | `CONFIG_CFI_CLANG` (kernel Control-Flow Integrity) |
| KASAN modes | generic, sw_tags | generic, sw_tags, **hw_tags (MTE)** |
| `-fsanitize=kernel-*` coverage | good | better (KCSAN, KMSAN are clang-only) |
| Rust interop | via bindgen only | shares LLVM IR → needed for LTO+Rust |
| Build speed | slower | faster |

**Use Clang when**: you need KMSAN/KCSAN, CFI, Rust+LTO, or you target arm64 Android.
**Use GCC when**: you're matching a distro kernel, or on architectures with weak clang support.

### 2.2 Getting a cross toolchain

Three good options, in order of preference:

**(a) kernel.org prebuilt (cleanest):**
```bash
mkdir -p ~/x-tools && cd ~/x-tools
wget https://mirrors.edge.kernel.org/pub/tools/crosstool/files/bin/x86_64/14.2.0/\
x86_64-gcc-14.2.0-nolibc-aarch64-linux.tar.xz
tar xf x86_64-gcc-14.2.0-nolibc-aarch64-linux.tar.xz
export PATH=$HOME/x-tools/gcc-14.2.0-nolibc/aarch64-linux/bin:$PATH
export CROSS_COMPILE=aarch64-linux-
```
These are `nolibc` toolchains — perfect for kernels (you don't need a libc to build a kernel).

**(b) Distro packages:**
```bash
# Debian/Ubuntu
sudo apt install gcc-aarch64-linux-gnu gcc-riscv64-linux-gnu \
                 gcc-powerpc64le-linux-gnu binutils-aarch64-linux-gnu
# Fedora
sudo dnf install gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu
```

**(c) LLVM (one toolchain, all targets):**
```bash
sudo apt install clang lld llvm
make LLVM=1 ARCH=arm64 defconfig
```
Clang is a native cross-compiler — no per-target install. This is the modern answer.

### 2.3 Build dependencies

```bash
# Debian / Ubuntu
sudo apt install -y build-essential bc bison flex libssl-dev libelf-dev \
  libncurses-dev dwarves rsync cpio kmod git ccache zstd \
  qemu-system-x86 qemu-system-arm qemu-utils \
  gdb-multiarch device-tree-compiler u-boot-tools \
  python3-pip pahole sparse coccinelle clang lld llvm

# Fedora
sudo dnf install -y @development-tools bc bison flex openssl-devel elfutils-libelf-devel \
  ncurses-devel dwarves rsync cpio kmod ccache zstd qemu gdb dtc sparse clang lld llvm
```

`dwarves` (for `pahole`) is **mandatory** if `CONFIG_DEBUG_INFO_BTF=y` — which you want for eBPF.

### 2.4 Rust toolchain (needed from Part 5 on — set it up now)

The kernel pins an exact Rust version per release. Check it:

```bash
cat scripts/min-tool-version.sh | grep -A3 rustc
# or
make rustavailable        # tells you exactly what's missing
```

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup override set $(scripts/min-tool-version.sh rustc)   # per-tree pin
rustup component add rust-src rustfmt clippy
cargo install --locked bindgen-cli --version $(scripts/min-tool-version.sh bindgen)
```

Verify:
```bash
make LLVM=1 rustavailable
# "Rust is available!"
```

> **Gotcha:** Rust requires `LLVM=1` (or at minimum `CC=clang`) in practice for
> `CONFIG_RUST=y` on most configs, because bindgen uses libclang and codegen must agree.

---

## 3. Practice

### Lab 1.1 — Native build with sane defaults

```bash
git clone --depth=1 https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git -b linux-6.12.y
cd linux

# Out-of-tree build dir keeps the source pristine — ALWAYS do this
make O=../build-x86 defconfig
make O=../build-x86 -j$(nproc)
ls -lh ../build-x86/arch/x86/boot/bzImage
```

Time it. Then enable ccache and rebuild from scratch:

```bash
export PATH="/usr/lib/ccache:$PATH"     # Debian
make O=../build-x86 CC="ccache gcc" -j$(nproc)
ccache -s
```

Expect 5–10× speedup on rebuilds. **Set `CCACHE_MAXSIZE=50G`.**

### Lab 1.2 — Cross-build for arm64 and riscv

```bash
make O=../build-arm64 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
make O=../build-arm64 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc) Image dtbs

make O=../build-riscv ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- defconfig
make O=../build-riscv ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- -j$(nproc)
```

With LLVM (no cross toolchain install needed):
```bash
make O=../build-arm64-llvm ARCH=arm64 LLVM=1 defconfig
make O=../build-arm64-llvm ARCH=arm64 LLVM=1 -j$(nproc)
```

### Lab 1.3 — A proper development config

`defconfig` is for shipping. For *learning*, you want every debug feature on.
Create `dev.config`:

```
# --- Debug info & symbols ---
CONFIG_DEBUG_INFO_DWARF5=y
CONFIG_DEBUG_INFO_BTF=y
CONFIG_GDB_SCRIPTS=y
CONFIG_KALLSYMS_ALL=y
CONFIG_DEBUG_KERNEL=y
CONFIG_FRAME_POINTER=y

# --- Lock & concurrency debugging (SLOW but priceless) ---
CONFIG_PROVE_LOCKING=y
CONFIG_DEBUG_ATOMIC_SLEEP=y
CONFIG_DEBUG_SPINLOCK=y
CONFIG_DEBUG_MUTEXES=y
CONFIG_DEBUG_RWSEMS=y
CONFIG_LOCK_STAT=y
CONFIG_DEBUG_LOCKDEP=y
CONFIG_RCU_EQS_DEBUG=y

# --- Memory debugging ---
CONFIG_KASAN=y
CONFIG_KASAN_INLINE=y
CONFIG_DEBUG_PAGEALLOC=y
CONFIG_SLUB_DEBUG=y
CONFIG_DEBUG_OBJECTS=y
CONFIG_DEBUG_OBJECTS_FREE=y
CONFIG_DEBUG_VM=y
CONFIG_DEBUG_LIST=y
CONFIG_DEBUG_SG=y
CONFIG_DEBUG_KMEMLEAK=y

# --- Tracing ---
CONFIG_FTRACE=y
CONFIG_FUNCTION_TRACER=y
CONFIG_FUNCTION_GRAPH_TRACER=y
CONFIG_DYNAMIC_FTRACE=y
CONFIG_KPROBES=y
CONFIG_UPROBES=y
CONFIG_BPF_SYSCALL=y
CONFIG_BPF_JIT=y
CONFIG_STACK_TRACER=y
CONFIG_SCHED_TRACER=y
CONFIG_PREEMPTIRQ_TRACEPOINTS=y

# --- Debugging aids ---
CONFIG_MAGIC_SYSRQ=y
CONFIG_KGDB=y
CONFIG_KGDB_SERIAL_CONSOLE=y
CONFIG_DEBUG_FS=y
CONFIG_DEBUG_INFO_REDUCED=n
CONFIG_FAULT_INJECTION=y
CONFIG_FAILSLAB=y
CONFIG_FAIL_PAGE_ALLOC=y
CONFIG_FAIL_MAKE_REQUEST=y
CONFIG_KUNIT=y

# --- Compile-time strictness ---
CONFIG_WERROR=y
CONFIG_UBSAN=y
```

Apply it:
```bash
make O=../build-x86 defconfig
./scripts/kconfig/merge_config.sh -O ../build-x86 ../build-x86/.config dev.config
make O=../build-x86 olddefconfig
make O=../build-x86 -j$(nproc)
```

> This kernel is **3–10× slower**. That is the point. All your development happens here.

### Lab 1.4 — Useful make targets you must know

```bash
make help                     # read this once, fully
make O=b menuconfig           # ncurses config UI
make O=b nconfig              # nicer ncurses UI
make O=b xconfig              # Qt UI
make O=b olddefconfig         # accept defaults for new symbols (scripting-friendly)
make O=b oldconfig            # interactive for new symbols
make O=b savedefconfig        # minimal config → ./defconfig
make O=b localmodconfig       # only modules currently loaded — fast builds
make O=b kernelversion
make O=b kernelrelease
make O=b M=drivers/foo        # build one external/out-of-tree dir
make O=b drivers/net/ethernet/intel/  # build one subdir
make O=b fs/ext4/inode.o      # build ONE object — fastest syntax check
make O=b fs/ext4/inode.i      # preprocessed output — invaluable for macro debugging
make O=b fs/ext4/inode.s      # assembly output
make O=b vmlinux              # ELF kernel with symbols
make O=b modules_install INSTALL_MOD_PATH=/tmp/rootfs
make O=b headers_install INSTALL_HDR_PATH=/tmp/uapi   # UAPI headers
make O=b htmldocs             # Documentation/ → HTML
make O=b C=1                  # run sparse on changed files
make O=b C=2                  # run sparse on everything
make O=b W=1                  # extra warnings (W=1,2,3 increasing noise)
make O=b coccicheck MODE=report
make O=b clean                # remove objects
make O=b mrproper             # remove objects + config
make O=b distclean            # + editor backup files
make O=b tags / cscope / compile_commands.json
```

**`compile_commands.json`** is the single highest-leverage target for tooling:
```bash
make O=../build-x86 -j$(nproc) compile_commands.json
ln -sf ../build-x86/compile_commands.json .     # clangd now works on the whole tree
```

### Lab 1.5 — Reproducible builds

```bash
export KBUILD_BUILD_TIMESTAMP='Thu Jan  1 00:00:00 UTC 1970'
export KBUILD_BUILD_USER=nobody
export KBUILD_BUILD_HOST=nowhere
export SOURCE_DATE_EPOCH=0
make O=b -j$(nproc)
sha256sum b/vmlinux
```
Build twice in different directories; hashes must match. Yocto/Debian rely on this
(Chapter 99 covers reproducibility in product builds).

### Lab 1.6 — See the ABI deviations for yourself (T.3)

```bash
cd linux
# 1. Prove the red zone is disabled
grep -rn 'mno-red-zone' arch/x86/Makefile
# Compile the same function with and without and diff the asm:
cat > /tmp/rz.c <<'EOF'
int leaf(int a) { int buf[4]; buf[0] = a; return buf[0] + 1; }
EOF
gcc -O2 -S -o - /tmp/rz.c | head -20                 # may use -8(%rsp) etc. WITHOUT sub
gcc -O2 -mno-red-zone -S -o - /tmp/rz.c | head -20   # must adjust %rsp first

# 2. Prove FP is rejected
cat > /tmp/fp.c <<'EOF'
double f(double a, double b) { return a * b; }
EOF
gcc -O2 -mno-sse -mno-mmx -mno-sse2 -c /tmp/fp.c -o /dev/null   # → error / SW emulation

# 3. See the syscall convention (r10, not rcx)
sed -n '1,80p' arch/x86/entry/entry_64.S | grep -n -A5 -B5 'r10'
grep -rn 'r10' Documentation/arch/x86/entry_64.rst 2>/dev/null || \
  grep -rn 'syscall' Documentation/arch/x86/ | head
```

### Lab 1.7 — Reproduce the CVE-2009-1897 class of miscompilation (T.4)

```bash
cat > /tmp/ub.c <<'EOF'
struct s { int x; };
int g;
int f(struct s *p)
{
	int v = p->x;          /* dereference */
	if (!p)                /* check AFTER the deref */
		return -1;
	return v;
}
EOF
gcc -O2 -S -o - /tmp/ub.c | grep -c test        # the NULL test is GONE
gcc -O2 -fno-delete-null-pointer-checks -S -o - /tmp/ub.c | grep -c test   # it's back
```
Now find the flag in the kernel: `grep -rn 'delete-null-pointer-checks' Makefile`.
**Write down what you just proved.** This single experiment justifies half of T.4.

### Lab 1.8 — Strict aliasing, and why the kernel disables it

```bash
cat > /tmp/alias.c <<'EOF'
#include <stdio.h>
int f(int *i, float *fp) { *i = 1; *fp = 2.0f; return *i; }   /* may return 1, not 2 */
int main(void) { int x; printf("%d\n", f(&x, (float *)&x)); }
EOF
gcc -O2 /tmp/alias.c -o /tmp/a && /tmp/a                      # often prints 1
gcc -O2 -fno-strict-aliasing /tmp/alias.c -o /tmp/a && /tmp/a # prints 2
```
Then find real kernel code that depends on this: network header casts over `skb->data`
(`net/ipv4/ip_input.c`, `ip_hdr()`), and every `container_of()`.

### Lab 1.9 — Dissect a relocation, then watch `insmod` apply it (T.1)

```bash
# Build a trivial module (Ch. 05 has the Makefile)
readelf -r hello.ko | head -30
readelf -S hello.ko | grep -E 'rela|text|modinfo|gnu_linkonce'
nm -u hello.ko                                   # undefined: printk, __fentry__, ...

# Where did the kernel place it, and what got patched?
sudo insmod hello.ko
sudo cat /proc/modules | grep hello
sudo cat /sys/module/hello/sections/.text
sudo grep hello /proc/kallsyms | head

# Compare: the symbol was UND in the .ko, resolved at load time.
```

### Lab 1.10 — Build the same kernel with GCC and Clang and compare

```bash
make O=b-gcc  defconfig && make O=b-gcc  -j$(nproc)
make LLVM=1 O=b-llvm defconfig && make LLVM=1 O=b-llvm -j$(nproc)

# Size comparison
size b-gcc/vmlinux b-llvm/vmlinux
ls -l b-gcc/arch/x86/boot/bzImage b-llvm/arch/x86/boot/bzImage

# Warning comparison — Clang finds different bugs
make O=b-gcc  -j$(nproc) 2>&1 | grep -c warning:
make LLVM=1 O=b-llvm -j$(nproc) 2>&1 | grep -c warning:

# Now enable the Clang-only features:
./scripts/config --file b-llvm/.config -e LTO_CLANG_THIN -e CFI_CLANG -e KCFI
make LLVM=1 O=b-llvm olddefconfig && make LLVM=1 O=b-llvm -j$(nproc)
```
Both toolchains are first-class upstream. Being fluent in both is expected; many subsystems
(Android, ChromeOS) ship Clang-only features.

### Lab 1.11 — Inspect the hardening you just built in

```bash
# Which hardening options are on?
./scripts/config --file .config -s STACKPROTECTOR_STRONG
grep -E 'CONFIG_(STACKPROTECTOR|CFI_CLANG|SHADOW_CALL_STACK|RANDSTRUCT|FORTIFY|UBSAN|INIT_STACK)' .config

# Use the kernel's own checker:
sudo apt install -y python3-pip && pip3 install --user kernel-hardening-checker
kernel-hardening-checker -c .config | head -60

# See a stack canary in generated code:
objdump -d vmlinux | grep -A3 -B10 '__stack_chk_fail' | head -40

# See CFI type hashes (Clang builds only):
objdump -d vmlinux | grep -B2 '__cfi_' | head -20
```

### Lab 1.12 — Read the linker script (T.1)

```bash
$EDITOR arch/x86/kernel/vmlinux.lds.S
# Find and explain each of these:
grep -n 'INIT_TEXT\|INIT_DATA\|PERCPU_SECTION\|__initcall\|_sinittext\|__start_orc' \
	arch/x86/kernel/vmlinux.lds.S include/asm-generic/vmlinux.lds.h | head -40

# Now see the result:
readelf -S vmlinux | head -40
nm -n vmlinux | grep -E '__init_begin|__init_end|_stext|_etext|__start_rodata'
# And watch __init get freed at boot:
dmesg | grep 'Freeing unused kernel'
```
Compute: how many KiB does `__init` reclaim? That is memory you get back because of one
section attribute.

### Lab 1.13 — Cross-compile for three architectures in one sitting

```bash
sudo apt install -y gcc-aarch64-linux-gnu gcc-riscv64-linux-gnu gcc-arm-linux-gnueabihf \
                    qemu-system-arm qemu-system-misc

for A in "arm64 aarch64-linux-gnu- defconfig Image" \
         "riscv riscv64-linux-gnu- defconfig Image" \
         "arm   arm-linux-gnueabihf- multi_v7_defconfig zImage"; do
  set -- $A
  make ARCH=$1 CROSS_COMPILE=$2 O=b-$1 $3
  make ARCH=$1 CROSS_COMPILE=$2 O=b-$1 -j$(nproc) $4
done

# Or with clang, one toolchain for all:
make LLVM=1 ARCH=arm64 O=b-arm64-llvm defconfig && make LLVM=1 ARCH=arm64 O=b-arm64-llvm -j$(nproc)

file b-arm64/arch/arm64/boot/Image b-riscv/arch/riscv/boot/Image
```
Boot each in QEMU in Ch. 04. Being able to do this from muscle memory is a hard requirement
for embedded/BSP work (Part 7).

---

## 4. Build system performance

| Technique | Speedup | Notes |
|---|---|---|
| `O=` out-of-tree | — | hygiene, enables parallel configs |
| `ccache` | 5–10× on rebuild | `CCACHE_MAXSIZE=50G`, `CCACHE_SLOPPINESS=time_macros` |
| `make -j$(nproc)` | N× | use `-j$(( $(nproc) * 3 / 2 ))` with ccache |
| `localmodconfig` | 5–20× | builds only what your machine loads |
| `CONFIG_DEBUG_INFO_REDUCED=y` | ~2× link | but breaks gdb; don't use for dev |
| `CONFIG_MODULE_COMPRESS_ZSTD` | smaller | slower install |
| `LLVM=1 ld.lld` | 2–4× link | much faster linking than BFD ld |
| `distcc` / `icecc` | N× | for big teams |
| `make -s` | — | silence; use `make V=1` to see full commands |

Single-file iteration loop (the one you'll live in):
```bash
make O=b fs/ext4/inode.o && echo OK      # ~1 second
```

---

## 5. Mastery drills

1. Build the same tree with GCC and with `LLVM=1`. Diff `nm -S --size-sort vmlinux`
   output. Which functions differ most in size? Why?
2. Use `make fs/ext4/inode.i` to expand `EXT4_SB(sb)`. Follow the macro chain manually
   first, then confirm.
3. Build with `CONFIG_LTO_CLANG_THIN=y`. Measure build time and `vmlinux` size delta.
4. Find where `CROSS_COMPILE` is consumed in the top-level `Makefile`. Trace how `CC`
   is finally set.
5. Set up a `~/.bashrc` function `kbuild <arch>` that configures env + `O=` dir for
   x86_64/arm64/riscv in one command. You will use it daily.
6. **Flag archaeology.** For each of `-fno-strict-aliasing`, `-fno-strict-overflow`,
   `-fno-delete-null-pointer-checks`, `-mno-red-zone`, find the commit that added it
   (`git log -S'<flag>' --oneline -- Makefile arch/x86/Makefile`) and read the commit
   message. Write one sentence per flag on what broke without it.
7. **Read `scripts/basic/fixdep.c`** (~200 lines). Explain in writing how config-symbol
   granularity is achieved and why a naive `.d` file is insufficient. Then verify: touch
   `include/config/…`, run `make`, and confirm only the expected files rebuild.
8. **Reproducibility audit.** Build twice on two different machines with different
   usernames/paths. Diff the `vmlinux` bytes (`cmp -l`). If they differ, find the source of
   nondeterminism and eliminate it. Report the flags you needed.
9. **Measure the hardening tax.** Build three kernels: (a) baseline, (b) +stack protector
   +fortify +auto-var-init, (c) +LTO +CFI. Boot each in QEMU and run a syscall-heavy
   benchmark (`lat_syscall`, `sysbench`, kernel compile inside the VM). Report the %
   overhead of each. This is exactly the data a security architect must produce.
10. **Bootstrap a toolchain.** Use `crosstool-NG` to build an `aarch64-none-linux-gnu`
    toolchain from source. Document each of the three stages and what it produces. Then
    build the kernel with it. (Part 7 assumes you can do this.)
11. **ORC vs frame pointers.** Build with `CONFIG_UNWINDER_FRAME_POINTER=y` and with
    `CONFIG_UNWINDER_ORC=y`. Compare `vmlinux` size, the `.orc_unwind*` section sizes
    (`readelf -S`), and run a `perf record -g` benchmark. Explain the trade in one paragraph.
12. **Trusting trust.** Read Thompson's 1984 lecture. Explain how reproducible builds plus
    diverse double-compilation (David A. Wheeler's countermeasure) address it, and what
    residual risk remains.

---

## 6. Further reading

**Kernel documentation:**
- `Documentation/kbuild/` — **all of it**: `makefiles.rst`, `kconfig-language.rst`,
  `kbuild.rst`, `reproducible-builds.rst`, `llvm.rst`
- `Documentation/process/changes.rst` — minimum tool versions
- `Documentation/process/maintainer-kbuild.rst`
- `Documentation/dev-tools/` — sparse, coccinelle, kasan, kunit, gdb-kernel-debugging, ubsan
- `arch/*/Makefile` and `arch/*/kernel/vmlinux.lds.S` — read both for your architecture

**Specifications (skim, then keep as reference):**
- System V AMD64 psABI — argument passing, the red zone, stack alignment
- ARM AAPCS64 — the arm64 equivalent
- ELF specification (TIS/Generic ABI) + the per-architecture supplements for relocation types
- DWARF 5 spec — only §1–2 unless you're writing a debugger

**Papers / classic articles:**
- Thompson, "Reflections on Trusting Trust" (Turing lecture, 1984) — three pages, mandatory
- Wheeler, "Countering Trusting Trust through Diverse Double-Compiling" (ACSAC 2005)
- Mokhov, Mitchell & Peyton Jones, "Build Systems à la Carte" (ICFP 2018)
- Miller, "Recursive Make Considered Harmful" (1997)
- Levine, *Linkers and Loaders* — **the** book on T.1; free online
- Drepper, "How To Write Shared Libraries" — dynamic linking theory (relevant to Ch. 92)
- Wang et al., "Undefined Behavior: What Happened to My Code?" (APSYS 2012) — the STACK
  paper; empirical study of UB-induced bugs *in Linux*
- Regehr's blog series "A Guide to Undefined Behavior in C and C++"

**Projects/sites:**
- ClangBuiltLinux: https://clangbuiltlinux.github.io/
- Kernel Self-Protection Project: https://kernsec.org/wiki/index.php/Kernel_Self_Protection_Project
- Reproducible Builds: https://reproducible-builds.org/
- `kernel-hardening-checker` (formerly `kconfig-hardened-check`)

→ Next: [02-source-tree-map.md](02-source-tree-map.md)

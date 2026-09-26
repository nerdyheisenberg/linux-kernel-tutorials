# Chapter 92 — ELF, libc, Dynamic Linking, and the Userspace ABI

> The kernel's side of `execve` was Ch. 20 and Ch. 24. This chapter is the other side: what
> an executable actually is, what happens between `execve` returning and `main()` being
> called, how symbols get resolved, and why libc choice is an architectural decision in
> embedded Linux. It is also where a large fraction of "it works on my machine" problems
> live.

---

## Theory & First Principles

### T.0 — Start here: your program is not the first code that runs

```bash
cat > hi.c <<'EOF'
int main(void) { return 0; }
EOF
gcc hi.c -o hi && ltrace -S ./hi 2>&1 | head -30
```

Before `main` is entered, a **different program** has already run inside your process: the
dynamic linker, `/lib64/ld-linux-x86-64.so.2`. It mapped your libraries, resolved symbols,
applied relocations, ran constructors, and only then jumped to your entry point.

```bash
file hi                      # "dynamically linked, interpreter /lib64/ld-linux..."
readelf -l hi | grep -A1 INTERP
```

**The kernel's `execve` is far simpler than people assume. It does five things:**

```
  1. read the first bytes -> recognize \x7fELF (or #! -> a different handler)
  2. tear down the old address space  <- the POINT OF NO RETURN.
                                         After this, failure = SIGKILL, not an errno.
  3. mmap the PT_LOAD segments at their prescribed addresses/permissions
  4. if there is a PT_INTERP, map THAT too, and set the entry point to IT
  5. build the initial stack: argv, envp, and the AUXILIARY VECTOR
```

**Step 2 deserves attention as a design observation.** `execve` cannot fail cleanly past a
certain point, because the old program is already gone — which is exactly why all the
validation (permissions, format, interpreter existence) happens *before* it. **Validate, then
apply** (Ch. 89 §T.0, principle 6), in the most literal form there is.

**Step 5 is the underrated one.** The auxiliary vector is how the kernel tells the new program
about itself:

```bash
LD_SHOW_AUXV=1 ./hi
# AT_PHDR, AT_ENTRY   -- where your program headers are
# AT_PAGESZ           -- 4096
# AT_HWCAP, AT_HWCAP2 -- which CPU instructions are available (AVX-512? SVE?)
# AT_SYSINFO_EHDR     -- the vDSO: kernel code mapped into YOUR address space
# AT_RANDOM           -- 16 random bytes, used to seed the stack canary
```

**`AT_SYSINFO_EHDR` is worth a pause.** The vDSO is a small shared library the kernel maps
into every process, containing implementations of `gettimeofday`, `clock_gettime`, and
`getcpu` that read a shared page **without a syscall at all**. A `clock_gettime` costs ~25 ns
instead of ~1 µs. **When an operation is read-only and frequent, expose the data instead of
providing a call** — the same idea as `io_uring`'s shared rings (Ch. 76 §T.0), at a much
smaller scale.

**Now the four faces of ELF**, because the format's economy is the chapter's point:

| Use | Read by | Key structure |
|---|---|---|
| An executable | the kernel | **program headers** (`PT_LOAD`): what to map where |
| A shared library | the dynamic linker | `PT_DYNAMIC`, symbol tables, relocations |
| An object file | the static linker | **section headers**: what to combine |
| A core dump / `vmlinux` | a debugger | notes, and DWARF sections |

**One format, two completely different views of the same bytes.** Section headers are for
*linking*; program headers are for *loading*; they describe the same file at different
granularities, and neither is needed by the other consumer. **A file format that serves both a
build-time and a run-time consumer by offering two independent indices over one dataset** —
note the resemblance to XFS's two free-space B+trees (Ch. 58 §T.0).

**And the lazy-binding trick, which explains a security mitigation you have seen:** by default
a function's address is resolved on its *first call*, via the PLT and GOT — so a program using
5 of a library's 2,000 functions resolves 5. The cost is a writable, indirect jump table,
which is an attractive target for an attacker. `-z now` plus `RELRO` resolves everything at
startup and makes the GOT read-only. **A startup-time cost traded against an exploitation
surface** — and now you know why distributions enable it by default.

```bash
readelf -h hi && readelf -l hi && readelf -d hi
LD_DEBUG=all ./hi 2>&1 | head -40      # watch the dynamic linker work
ldd hi && objdump -R hi | head
cat /proc/self/maps | grep -E 'vdso|vvar'
```

---

### T.1 — ELF: one format, four uses

```
 ┌──────────────────────┐
 │ ELF header           │  magic, class (32/64), endianness, type, machine, entry
 ├──────────────────────┤
 │ Program headers      │  ── the EXECUTION view: what the kernel maps
 │ (segments)           │      PT_LOAD, PT_DYNAMIC, PT_INTERP, PT_NOTE, PT_GNU_STACK
 ├──────────────────────┤
 │                      │
 │ Sections             │  ── the LINKING view: what the linker manipulates
 │  .text .rodata       │      .symtab .strtab .rela.* .dynamic .got .plt
 │  .data .bss          │
 │  .dynsym .dynstr     │
 │  .init_array         │
 ├──────────────────────┤
 │ Section headers      │
 └──────────────────────┘
```

**The dual view is the central idea.** Sections are for the *linker*; segments are for the
*loader*. `ld` merges sections from many objects and groups them into a few segments by
permission. The kernel only ever reads program headers — you can strip every section header
and the binary still runs.

Four ELF types:

| `e_type` | What | Loaded by |
|---|---|---|
| `ET_REL` | Relocatable object (`.o`) | the linker |
| `ET_EXEC` | Fixed-address executable | the kernel, at its stated addresses |
| `ET_DYN` | Shared object (`.so`) **or PIE executable** | the kernel/loader, anywhere |
| `ET_CORE` | Core dump | debuggers |

That `ET_DYN` covers both `.so` and PIE is the source of endless confusion. A PIE executable
*is* a shared object with an entry point; `file` says "shared object" and people assume
something is wrong. Distributions default to PIE so that ASLR can randomize the executable
itself.

### T.2 — What `execve` actually does

```
 execve("/bin/ls", argv, envp)
   │
   ├─ 1. Open and read the first 128 bytes. Match against registered
   │     binfmt handlers: ELF, script (#!), misc (binfmt_misc), flat.
   │
   ├─ 2. binfmt_elf: parse the ELF header and program headers.
   │
   ├─ 3. Point of no return: flush the old mm, create a new one.
   │     From here, failure means SIGSEGV, not an error return.
   │
   ├─ 4. mmap each PT_LOAD segment at its vaddr (+ a random base if ET_DYN).
   │     .text  -> r-xp   .rodata -> r--p   .data -> rw-p
   │     .bss   -> anonymous, zero-filled
   │
   ├─ 5. If PT_INTERP exists, ALSO map the interpreter (/lib64/ld-linux-x86-64.so.2)
   │     at a random base.
   │
   ├─ 6. Set up the stack: argv, envp, and the AUXILIARY VECTOR.
   │
   ├─ 7. Map the vDSO and vvar pages.
   │
   └─ 8. Jump to: the interpreter's entry point if there is one,
         otherwise the executable's e_entry.
```

**The auxiliary vector (auxv) is the kernel→loader ABI** and is worth knowing by name:

| Tag | Value |
|---|---|
| `AT_PHDR`, `AT_PHENT`, `AT_PHNUM` | Where the program headers were mapped |
| `AT_BASE` | The interpreter's load base |
| `AT_ENTRY` | The executable's entry point |
| `AT_PAGESZ` | Page size |
| `AT_HWCAP`, `AT_HWCAP2` | **CPU feature bits** — how glibc picks an optimized `memcpy` |
| `AT_SYSINFO_EHDR` | **The vDSO base** — how libc finds `clock_gettime` |
| `AT_RANDOM` | 16 random bytes; the stack-canary and pointer-guard seed |
| `AT_SECURE` | Set for setuid/setgid; the loader ignores `LD_PRELOAD` when set |
| `AT_EXECFN` | The pathname as given |

```bash
LD_SHOW_AUXV=1 /bin/true          # print the whole thing
cat /proc/self/auxv | xxd | head
```

**`AT_SECURE` is a security-critical detail:** it is why `LD_PRELOAD` and `LD_LIBRARY_PATH`
are ignored for setuid binaries. Without it, every setuid binary would be trivially
exploitable.

### T.3 — The dynamic linker's job

`ld.so` runs before `main()` and does four things.

**1. Load dependencies.** Read `DT_NEEDED` entries, search for each, `mmap` it, recurse. The
search order:

```
 1. DT_RPATH        (deprecated; ignored if DT_RUNPATH is present)
 2. LD_LIBRARY_PATH (ignored if AT_SECURE)
 3. DT_RUNPATH      (applies only to THIS object's direct dependencies)
 4. /etc/ld.so.cache  (built by ldconfig -- the fast path)
 5. /lib, /usr/lib, /lib64, /usr/lib64
```

**2. Relocate.** Patch addresses that could not be known at link time.

| Relocation | For |
|---|---|
| `R_X86_64_RELATIVE` | "Add the load base" — the bulk of a PIE's relocations |
| `R_X86_64_GLOB_DAT` | Fill a GOT entry with a symbol's address |
| `R_X86_64_JUMP_SLOT` | A PLT entry; **lazily** resolved by default |
| `R_X86_64_COPY` | Copy a variable's contents into the executable's `.bss` (an ugly legacy mechanism) |
| `R_X86_64_TPOFF64` | Thread-local storage offset |

**3. Resolve symbols**, by the *global scope* rule: the first definition found wins,
searching in load order — executable, then `LD_PRELOAD`, then dependencies breadth-first.
**This is why `LD_PRELOAD` works**: it inserts an object early in the search order, so its
definitions shadow the real ones.

**4. Run initializers.** `DT_INIT` (legacy `_init`), then every function pointer in
`.init_array`, in dependency order. C++ static constructors and
`__attribute__((constructor))` functions run here — **before `main()`**, and therefore before
any of your own setup. This is a recurring source of order-dependent bugs.

### T.4 — PLT and GOT: how a call to a shared function works

```
 Your code:                  call printf@plt
                                  │
 .plt section:              printf@plt:
                                jmp *printf@got(%rip)   ─┐
                                push $index              │  first call:
                                jmp  .plt[0]  ───────────┼──► _dl_runtime_resolve
                                                         │       resolves the symbol,
 .got.plt section:          printf@got: ◄────────────────┘       WRITES the real address
                                [initially -> the next            into the GOT
                                 instruction in the PLT]
                                [after resolution -> printf]
                                                          subsequent calls: one
                                                          indirect jump, no overhead
```

**Lazy binding** means a symbol is resolved on first call, not at startup. It makes startup
faster for large binaries where most symbols are never called.

**Why you often want to turn it off:**

```bash
LD_BIND_NOW=1 ./program           # resolve everything at startup
gcc -Wl,-z,now -Wl,-z,relro ...   # bake it in: FULL RELRO
```

`-z now` plus `-z relro` makes the GOT **read-only after relocation**. Without it, the GOT is
writable for the process's entire life, and "overwrite a GOT entry" is a classic exploitation
primitive. **Full RELRO is the default in most distributions now, and it should be in
anything you ship.**

```bash
checksec --file=/bin/ls           # RELRO, canary, NX, PIE, fortify
readelf -d /bin/ls | grep -E 'BIND_NOW|FLAGS'
```

### T.5 — Symbol versioning

The mechanism that lets glibc change a function's behaviour without breaking existing
binaries — a userspace analogue of the kernel's ABI problem (Ch. 24).

```bash
$ objdump -T /lib/x86_64-linux-gnu/libc.so.6 | grep -w memcpy
0000000000098c40 g    DF .text  ... GLIBC_2.14   memcpy
0000000000098c50 g    DF .text  ... GLIBC_2.2.5  memcpy
```

Two `memcpy` symbols. A binary linked against glibc 2.13 records a dependency on
`memcpy@GLIBC_2.2.5` (which allowed overlapping copies in practice); a newer binary gets
`memcpy@GLIBC_2.14` (which does not). Both work, forever, from one library.

The practical consequences, which are the ones you meet:

1. **A binary built on a newer glibc will not run on an older one.** `version 'GLIBC_2.34'
   not found`. The versioning is a *minimum* requirement, not a negotiation.
2. **The reverse works fine.** Old binaries run on new glibc. This is the whole point.
3. **Therefore: build on the oldest system you must support**, or use a toolchain targeting
   an old glibc, or link statically, or use musl.

```bash
# What glibc version does this binary require?
objdump -T ./myapp | grep GLIBC_ | sed 's/.*GLIBC_\([0-9.]*\).*/\1/' | sort -uV | tail -1

# Which symbols are the problem?
objdump -T ./myapp | grep GLIBC_2.3[4-9]
```

### T.6 — The libc decision

Genuinely architectural in embedded Linux, and a common interview topic.

| | **glibc** | **musl** | **uClibc-ng** | **Bionic** |
|---|---|---|---|---|
| Size (shared) | ~2 MB | ~600 KB | ~400 KB | ~1 MB |
| Static hello-world | ~800 KB | ~30 KB | ~50 KB | n/a |
| Standards | POSIX + many GNU extensions | POSIX, strict | POSIX subset | POSIX subset |
| Static linking | works but discouraged (NSS, dlopen) | **excellent** | good | n/a |
| Locale/i18n | full | minimal (UTF-8 only) | minimal | partial |
| NSS plugins | yes (dlopen'd) | **no** | no | no |
| Performance | best-tuned (ifunc, SIMD) | good, simpler | adequate | good |
| Thread stack default | 8 MB | **128 KB** | small | 1 MB |
| Used by | most distributions | Alpine, containers | deep embedded | Android |

**The failure modes to know:**

- **musl + static + NSS.** glibc's `getpwnam`/`gethostbyname` `dlopen` NSS modules, so a
  statically-linked glibc binary silently loses LDAP/NIS/mDNS name resolution. musl has no
  NSS at all — it reads `/etc/passwd` and `/etc/resolv.conf` directly, which is simpler and
  usually what you want on an embedded device but breaks enterprise integrations.
- **musl's 128 KB default thread stack.** Code that works on glibc (8 MB default) overflows
  on musl. This is the single most common musl porting problem and it presents as a mysterious
  crash. Fix: `pthread_attr_setstacksize`.
- **DNS.** musl's resolver historically lacked TCP fallback and had different search
  semantics. Mostly fixed, still a source of surprises.
- **glibc's `__libc_start_main` version bumps** breaking forward compatibility (§T.5).
- **Bionic** is not a general-purpose libc; do not plan to use it off Android.

**The decision rule:** glibc unless you have a reason; musl for containers and
size-constrained embedded where you control the whole userspace; uClibc-ng only for very
constrained systems or legacy; and **never mix** — a system with both is a debugging
nightmare.

### T.7 — Static versus dynamic linking

| | Static | Dynamic |
|---|---|---|
| Binary size | large | small |
| Total system size (many binaries) | **very large** | small — the library is shared once |
| Memory (many processes) | each has its own copy of `.text` | **shared physical pages** |
| Startup time | **fast** — no relocation, no symbol resolution | slower |
| Security updates | **rebuild and redeploy everything** | update one `.so` |
| `dlopen`, NSS, plugins | broken or unavailable | works |
| ASLR of library code | none | yes |
| Deployment | one file, no dependencies | must ship the right libraries |
| Determinism | total | depends on the target system |

**The security-update argument is decisive for most products.** A heap-overflow CVE in zlib
means updating one `.so` on a dynamic system, versus finding and rebuilding every statically
linked binary that vendored it — and knowing which those are requires an SBOM (→ Ch. 99).

**Where static wins**: a single-binary appliance, a rescue/recovery image, an initramfs tool,
anything that must work when the filesystem is damaged, and Go/Rust binaries where static is
the norm anyway.

`-static-pie` gives you static linking *and* ASLR, which used to be mutually exclusive. Use
it if you link statically.

### T.8 — The vDSO

The kernel maps a small shared object into every process:

```bash
$ cat /proc/self/maps | grep vdso
7ffd8b3f1000-7ffd8b3f3000 r-xp 00000000 00:00 0    [vdso]
```

It provides syscall-free implementations of a few very hot calls:

| Function | Mechanism |
|---|---|
| `clock_gettime` | Reads the TSC and a shared page of clocksource data; no ring transition |
| `gettimeofday` | Same |
| `time` | Same |
| `getcpu` | Reads a per-CPU segment register |
| `sched_getcpu` | Same |
| `__vdso_sgx_enter_enclave` | SGX |

**The numbers from Ch. 102 Lab 2:** a syscall is ~200–500 ns with mitigations; a vDSO call is
~20–25 ns. A 10–20× difference on the most frequently called function in most programs.

The mechanism: the kernel keeps a `vvar` page containing the current clocksource, its
multiplier and shift, and a seqlock (Ch. 14 §T.6). The vDSO reads it under the seqlock and
computes the time in userspace. **The seqlock is what makes it safe** — the kernel can update
the timekeeping data while a reader is mid-read, and the reader retries.

libc finds the vDSO through `AT_SYSINFO_EHDR` and parses it as a mini-ELF at startup, which
is why `clock_gettime` "just works" without you knowing any of this.

### T.9 — Thread-local storage

Four models, chosen by the compiler based on visibility:

| Model | Cost | When |
|---|---|---|
| **Initial-exec** | one load from the GOT, then `%fs:offset` | `-ftls-model=initial-exec`; the executable or a `DT_NEEDED` library |
| **Local-exec** | `%fs:constant` — fastest | the main executable only |
| **General-dynamic** | a call to `__tls_get_addr` | `dlopen`'d libraries |
| **Local-dynamic** | one `__tls_get_addr` for several variables | |

**The practical issue:** a library that uses initial-exec TLS **cannot be `dlopen`'d** after
startup, because initial-exec requires a slot in the static TLS block, which is sized at
program start. You get `cannot allocate memory in static TLS block`. glibc reserves some
surplus for this, but it is finite.

This bites real systems: a plugin that pulls in OpenMP or a library built with
`-ftls-model=initial-exec` fails to load, and the error message points nowhere useful.

### T.10 — Cross-compilation and sysroots

```
 HOST (x86_64 build machine)          TARGET (arm64 device)
 ┌───────────────────────┐            ┌──────────────────────┐
 │ aarch64-linux-gnu-gcc │            │  the running system  │
 │ aarch64-linux-gnu-ld  │            │                      │
 │ ...                   │            │                      │
 └───────────┬───────────┘            └──────────────────────┘
             │ --sysroot=
             ▼
 ┌───────────────────────┐
 │ SYSROOT               │  a copy of the TARGET's /usr and /lib:
 │  usr/include/*.h      │  headers to compile against
 │  usr/lib/*.so         │  libraries to link against
 │  lib/ld-linux-*.so    │
 └───────────────────────┘
```

**Three terms that are constantly confused:**

| Term | Meaning |
|---|---|
| **build** | Where the compiler is being built |
| **host** | Where the compiler will *run* |
| **target** | What the compiler will *produce code for* |

A normal compiler: build = host = target. A cross-compiler: host ≠ target. A *Canadian
cross*: all three differ (building an ARM-hosted compiler that targets MIPS, on x86).

**The failure modes:**
- Linking against the **host's** libraries instead of the sysroot's. Symptom: "file in wrong
  format" or, worse, it links and then segfaults on the target.
- `pkg-config` returning host paths. Fix: `PKG_CONFIG_SYSROOT_DIR` and
  `PKG_CONFIG_LIBDIR`.
- A build system that runs a compiled test program to detect features — impossible when
  cross-compiling. Autoconf's `--enable-cross-compile` guesses, often wrongly.
- **`rpath` pointing at build-time paths**, leaking into the shipped binary.

This is exactly the problem Yocto and Buildroot exist to solve, and Ch. 96–100 are about how.

---

## 1. Internals

### Source map

| Path | Contents |
|---|---|
| `fs/binfmt_elf.c` | **The ELF loader.** `load_elf_binary()` is the whole of §T.2 |
| `fs/binfmt_script.c` | `#!` handling — shorter than you expect, and full of edge cases |
| `fs/binfmt_misc.c` | Registering arbitrary interpreters (how `qemu-user` and Wine work) |
| `fs/exec.c` | `do_execve`, the point of no return, credential changes |
| `include/uapi/linux/elf.h` | The ELF structures |
| `include/uapi/linux/auxvec.h` | `AT_*` definitions |
| `arch/*/kernel/vdso/` | The vDSO sources |
| `lib/vdso/gettimeofday.c` | The generic vDSO timekeeping implementation |
| glibc: `elf/rtld.c`, `elf/dl-*.c` | The dynamic linker |
| glibc: `csu/libc-start.c` | `__libc_start_main` — what calls `main()` |

### From `_start` to `main`

```
 kernel jumps to ld.so's entry
   │
 _dl_start
   ├─ bootstrap: relocate ld.so itself (it cannot call anything yet)
   ├─ _dl_sysdep_start: parse auxv
   ├─ dl_main:
   │    ├─ map the executable's dependencies (DT_NEEDED, recursively)
   │    ├─ relocate everything (or lazily, per §T.4)
   │    ├─ set up TLS
   │    └─ call each object's .init_array, in dependency order
   ▼
 executable's _start  (crt1.o)
   │
 __libc_start_main(main, argc, argv, init, fini, rtld_fini, stack_end)
   ├─ set up the stack guard from AT_RANDOM
   ├─ set up the pointer guard
   ├─ __libc_init_first: initialize stdio, locale, environ
   ├─ call __libc_csu_init -> .init_array (the executable's constructors)
   ├─ exit(main(argc, argv, envp))
   └─ (never returns)
```

**Things that run before `main`**, and are therefore invisible in a naive reading:
- Every shared library's `.init_array`
- C++ static constructors
- `__attribute__((constructor))` functions
- glibc's IFUNC resolvers (choosing SIMD `memcpy` from `AT_HWCAP`)
- The stack canary setup

Gotcha worth knowing: **an `__attribute__((constructor))` in a shared library runs before
`main`, but the order relative to other libraries' constructors is only guaranteed by
dependency order**, not by link order. Code that depends on cross-library constructor
ordering is fragile.

### Reading an ELF

```bash
readelf -h  binary        # header: type, machine, entry
readelf -l  binary        # program headers (segments) -- the LOADER's view
readelf -S  binary        # section headers -- the LINKER's view
readelf -d  binary        # .dynamic: DT_NEEDED, DT_RUNPATH, flags
readelf -s  binary        # symbol table
readelf -r  binary        # relocations
readelf -n  binary        # notes: build-id, ABI tag

objdump -d  binary        # disassemble
objdump -T  binary        # dynamic symbols with versions
objdump -R  binary        # dynamic relocations
objdump -p  binary        # program headers + dynamic info

nm -D       lib.so        # dynamic symbols
nm -C       binary        # demangled C++
ldd         binary        # dependencies (CAUTION: it EXECUTES the binary!)
objdump -p binary | grep NEEDED   # safer: does not execute

file        binary
strings -a  binary | head
checksec --file=binary    # security hardening flags
pahole      binary        # struct layouts (needs DWARF)
```

**`ldd` executes the binary** by setting `LD_TRACE_LOADED_OBJECTS=1` and running it. Never run
it on an untrusted binary. `objdump -p | grep NEEDED` or `readelf -d` is the safe equivalent.

### Debugging the loader

```bash
LD_DEBUG=help ./prog          # list the options
LD_DEBUG=libs ./prog          # library search and load order
LD_DEBUG=bindings ./prog      # every symbol resolution
LD_DEBUG=reloc ./prog         # relocation processing
LD_DEBUG=statistics ./prog    # how much time startup took, and where
LD_DEBUG=all LD_DEBUG_OUTPUT=/tmp/ld ./prog

LD_PRELOAD=./mylib.so ./prog  # interpose
LD_BIND_NOW=1 ./prog          # eager binding
LD_SHOW_AUXV=1 ./prog         # the auxiliary vector
```

**`LD_DEBUG=libs` is the answer to "why is it loading the wrong library?"**, and
`LD_DEBUG=statistics` is the answer to "why does startup take 300 ms?" — usually the answer is
thousands of relocations in a C++ binary.

---

## 2. Practice

### Lab 92.1 — Dissect a binary

```bash
#!/bin/bash
# elf_tour.sh — take a binary apart.
BIN=${1:-/bin/ls}

echo "=== Type and entry ==="
readelf -h "$BIN" | grep -E 'Type|Entry|Machine|Class'
#   ET_DYN on a modern distro = PIE, not a library. See T.1.

echo; echo "=== Segments: what the KERNEL maps ==="
readelf -lW "$BIN" | grep -E 'LOAD|INTERP|DYNAMIC|GNU_STACK|GNU_RELRO'
#   GNU_STACK with RW (no E) = non-executable stack. Good.
#   GNU_RELRO present = the GOT is made read-only. Good.

echo; echo "=== Interpreter ==="
readelf -lW "$BIN" | grep -A1 INTERP | tail -1

echo; echo "=== Dependencies (WITHOUT executing it) ==="
readelf -d "$BIN" | grep -E 'NEEDED|RUNPATH|RPATH|FLAGS'

echo; echo "=== Hardening ==="
checksec --file="$BIN" 2>/dev/null || {
	echo -n "RELRO: "; readelf -lW "$BIN" | grep -q GNU_RELRO && \
		{ readelf -d "$BIN" | grep -q BIND_NOW && echo full || echo partial; } || echo none
	echo -n "NX:    "; readelf -lW "$BIN" | grep GNU_STACK | grep -q E && echo NO || echo yes
	echo -n "PIE:   "; readelf -h "$BIN" | grep -q 'Type:.*DYN' && echo yes || echo no
	echo -n "Canary:"; nm -D "$BIN" 2>/dev/null | grep -q stack_chk && echo " yes" || echo " no"
}

echo; echo "=== glibc version required ==="
objdump -T "$BIN" 2>/dev/null | grep -o 'GLIBC_[0-9.]*' | sort -uV | tail -3

echo; echo "=== Build ID (for matching debug symbols) ==="
readelf -n "$BIN" | grep -A1 'Build ID'

echo; echo "=== Now watch it load ==="
echo "  LD_DEBUG=libs $BIN 2>&1 | head -40"
echo "  LD_DEBUG=statistics $BIN 2>&1 | tail -20"
```

### Lab 92.2 — Observe `execve` from the kernel side

```bash
# 1. Trace it.
strace -f -e trace=execve,mmap,openat,arch_prctl /bin/true 2>&1 | head -40
#    Read the sequence: execve, then the loader opening libc, mmapping
#    four segments with different permissions, arch_prctl for TLS.

# 2. Watch the mappings appear.
cat /proc/self/maps
#    Identify: the executable's three or four segments, ld.so, libc,
#    [heap], [stack], [vdso], [vvar].

# 3. From the kernel side, with bpftrace:
sudo bpftrace -e '
tracepoint:sched:sched_process_exec {
	printf("%s -> %s\n", comm, str(args->filename));
}
kprobe:load_elf_binary { @elf = count(); }
kprobe:load_script     { @script = count(); }
interval:s:10 { exit(); }'

# 4. Read the loader's own work:
LD_DEBUG=libs /bin/ls 2>&1 | grep -E 'find library|trying file|calling init'
LD_DEBUG=statistics /bin/ls 2>&1 | tail -20
#    -> "total startup time in dynamic loader" and the relocation counts.
#       For a C++ binary this can be tens of milliseconds.
```

### Lab 92.3 — Interpose a library

```c
/* interpose.c — intercept malloc/free to count and trace allocations.
 * Build: gcc -shared -fPIC -o interpose.so interpose.c -ldl
 * Run:   LD_PRELOAD=./interpose.so ls                              */
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <dlfcn.h>
#include <string.h>
#include <unistd.h>

static void *(*real_malloc)(size_t);
static void  (*real_free)(void *);
static long n_malloc, n_free, bytes;
static __thread int in_hook;      /* prevent recursion via dlsym/printf */

/* Runs BEFORE main, and before the program's own constructors if we
 * are earlier in the link order. This is where you resolve the real
 * symbols.                                                          */
__attribute__((constructor))
static void init(void)
{
	real_malloc = dlsym(RTLD_NEXT, "malloc");
	real_free   = dlsym(RTLD_NEXT, "free");
	/* RTLD_NEXT: "the next definition AFTER me in the search order",
	 * i.e. the real libc one. Without this we would recurse forever. */
}

__attribute__((destructor))
static void fini(void)
{
	char buf[128];
	int n = snprintf(buf, sizeof(buf),
		"[interpose] malloc=%ld free=%ld bytes=%ld leaked_calls=%ld\n",
		n_malloc, n_free, bytes, n_malloc - n_free);
	write(2, buf, n);   /* write(), not fprintf: stdio may be torn down */
}

void *malloc(size_t size)
{
	void *p;
	if (!real_malloc) init();
	p = real_malloc(size);
	if (!in_hook) {
		in_hook = 1;
		n_malloc++;
		bytes += size;
		in_hook = 0;
	}
	return p;
}

void free(void *ptr)
{
	if (!real_free) init();
	if (ptr && !in_hook) { in_hook = 1; n_free++; in_hook = 0; }
	real_free(ptr);
}
```

```bash
gcc -shared -fPIC -O2 -o interpose.so interpose.c -ldl
LD_PRELOAD=./interpose.so ls /usr
LD_PRELOAD=./interpose.so python3 -c 'pass'

# Prove AT_SECURE blocks it for setuid binaries:
LD_PRELOAD=./interpose.so /usr/bin/sudo -V 2>&1 | head -2
#   -> no interpose output. The loader ignored LD_PRELOAD because
#      AT_SECURE was set. THAT is the security property of §T.2.
```

This is exactly how `valgrind`, `ltrace`, `jemalloc`, `tcmalloc`, `faketime`, and every
sanitizer shim work.

### Lab 92.4 — Symbol versioning, hands on

```bash
# --- Build a versioned library ---
mkdir -p verdemo && cd verdemo

cat > lib.c <<'EOF'
#include <stdio.h>

/* The old implementation. */
int myfunc_v1(int x) { printf("v1: %d\n", x); return x; }
/* The new one, with different behaviour. */
int myfunc_v2(int x) { printf("v2: %d\n", x); return x * 2; }

__asm__(".symver myfunc_v1,myfunc@MYLIB_1.0");
__asm__(".symver myfunc_v2,myfunc@@MYLIB_2.0");   /* @@ = the DEFAULT */
EOF

cat > lib.map <<'EOF'
MYLIB_1.0 { global: myfunc; local: *; };
MYLIB_2.0 { global: myfunc; } MYLIB_1.0;
EOF

gcc -shared -fPIC -o libmy.so.1 lib.c -Wl,--version-script=lib.map \
	-Wl,-soname,libmy.so.1
ln -sf libmy.so.1 libmy.so

objdump -T libmy.so.1 | grep myfunc
#   -> two myfunc symbols, one per version.

# --- A client binds to the DEFAULT (2.0) ---
cat > main.c <<'EOF'
#include <stdio.h>
int myfunc(int);
int main(void) { printf("got %d\n", myfunc(21)); return 0; }
EOF
gcc -o main main.c -L. -lmy -Wl,-rpath,'$ORIGIN'
./main                      # -> v2: 21 / got 42
objdump -T main | grep myfunc
#   -> myfunc@MYLIB_2.0

# --- An OLD binary keeps working ---
# (Simulate: force binding to 1.0)
cat > main1.c <<'EOF'
#include <stdio.h>
__asm__(".symver myfunc,myfunc@MYLIB_1.0");
int myfunc(int);
int main(void) { printf("got %d\n", myfunc(21)); return 0; }
EOF
gcc -o main1 main1.c -L. -lmy -Wl,-rpath,'$ORIGIN'
./main1                     # -> v1: 21 / got 21
#   BOTH binaries work against ONE library. That is the whole point.
```

Now the realistic failure:

```bash
# What glibc does a binary require, and will it run on the target?
objdump -T /usr/bin/some-modern-binary | grep -o 'GLIBC_[0-9.]*' | sort -uV | tail -1
# Compare with the target's:
ssh target 'ldd --version | head -1'
# If the binary requires more than the target has, it will not run,
# and the error is "version `GLIBC_2.34' not found".
```

### Lab 92.5 — Compare libcs

```bash
#!/bin/bash
# libc_compare.sh — measure the difference.

cat > hello.c <<'EOF'
#include <stdio.h>
int main(void) { printf("hello\n"); return 0; }
EOF

echo "=== glibc dynamic ==="
gcc -O2 -o hello-glibc hello.c
size hello-glibc | tail -1
ls -l hello-glibc

echo "=== glibc static ==="
gcc -O2 -static -o hello-glibc-static hello.c
ls -l hello-glibc-static

echo "=== musl dynamic (needs musl-gcc) ==="
musl-gcc -O2 -o hello-musl hello.c 2>/dev/null && ls -l hello-musl

echo "=== musl static ==="
musl-gcc -O2 -static -o hello-musl-static hello.c 2>/dev/null && \
	ls -l hello-musl-static

echo; echo "=== Startup time (10000 execs) ==="
for b in hello-glibc hello-glibc-static hello-musl hello-musl-static; do
	[ -x "./$b" ] || continue
	printf '%-22s ' "$b"
	( time (for i in $(seq 1 2000); do ./$b >/dev/null; done) ) 2>&1 | grep real
done

echo; echo "=== The thread-stack trap ==="
cat > stack.c <<'EOF'
#include <pthread.h>
#include <stdio.h>
#include <string.h>
static void *worker(void *a) {
	char big[512 * 1024];        /* 512 KB on the stack */
	memset(big, 0, sizeof(big));
	printf("survived 512 KB stack use\n");
	return NULL;
}
int main(void) {
	pthread_t t;
	pthread_attr_t attr;
	size_t sz;
	pthread_attr_init(&attr);
	pthread_attr_getstacksize(&attr, &sz);
	printf("default thread stack: %zu KB\n", sz / 1024);
	pthread_create(&t, NULL, worker, NULL);
	pthread_join(t, NULL);
	return 0;
}
EOF
gcc -O0 -o stack-glibc stack.c -lpthread && ./stack-glibc
musl-gcc -O0 -o stack-musl stack.c 2>/dev/null && ./stack-musl
#   glibc: 8192 KB, survives.
#   musl:   128 KB, SEGFAULTS. This is T.6's most common porting bug.
```

### Lab 92.6 — Measure the vDSO

```c
/* vdso_bench.c — quantify the vDSO's value.
 * Build: gcc -O2 -o vdso_bench vdso_bench.c                        */
#define _GNU_SOURCE
#include <stdio.h>
#include <time.h>
#include <unistd.h>
#include <sys/syscall.h>

#define N 3000000

static unsigned long long ns(void)
{
	struct timespec t;
	clock_gettime(CLOCK_MONOTONIC, &t);
	return (unsigned long long)t.tv_sec * 1000000000ULL + t.tv_nsec;
}

int main(void)
{
	struct timespec ts;
	unsigned long long t0, t1;

	/* Through the vDSO (libc routes it there automatically). */
	t0 = ns();
	for (int i = 0; i < N; i++) clock_gettime(CLOCK_MONOTONIC, &ts);
	t1 = ns();
	printf("clock_gettime (vDSO)   : %6.1f ns\n", (double)(t1 - t0) / N);

	/* Forcing the real syscall, bypassing the vDSO. */
	t0 = ns();
	for (int i = 0; i < N; i++)
		syscall(SYS_clock_gettime, CLOCK_MONOTONIC, &ts);
	t1 = ns();
	printf("clock_gettime (syscall): %6.1f ns\n", (double)(t1 - t0) / N);

	return 0;
}
```

```bash
gcc -O2 -o vdso_bench vdso_bench.c && ./vdso_bench
# Expect roughly 20-25 ns vs 200-500 ns. A 10-20x difference.

# Extract and disassemble the vDSO itself:
cat /proc/self/maps | grep vdso
sudo dd if=/proc/self/mem of=/tmp/vdso.so bs=1 \
	skip=$((0x$(grep vdso /proc/self/maps | cut -d- -f1))) count=8192 2>/dev/null
objdump -T /tmp/vdso.so

# Watch the syscall count difference:
strace -c -e trace=clock_gettime ./vdso_bench 2>&1 | tail -5
#   -> ~3 million syscalls from the second loop, ~0 from the first.
```

### Lab 92.7 — Cross-compile and diagnose

```bash
# 1. Cross-compile, correctly.
SYSROOT=/usr/aarch64-linux-gnu
aarch64-linux-gnu-gcc --sysroot=$SYSROOT -O2 -o hello-arm64 hello.c
file hello-arm64
readelf -h hello-arm64 | grep Machine

# 2. Run it with qemu-user (which uses binfmt_misc, T.10 -> fs/binfmt_misc.c)
qemu-aarch64 -L $SYSROOT ./hello-arm64
# Or transparently, if binfmt_misc is registered:
./hello-arm64

# 3. NOW BREAK IT deliberately and learn the symptoms:

#    (a) Link against a host library.
aarch64-linux-gnu-gcc -O2 -o bad hello.c -L/usr/lib/x86_64-linux-gnu -lz
#    -> "skipping incompatible ... when searching for -lz"
#       then "cannot find -lz". The FIRST message is the real one;
#       people fix the second and make it worse.

#    (b) pkg-config returning host paths.
PKG_CONFIG_PATH=$SYSROOT/usr/lib/pkgconfig \
PKG_CONFIG_SYSROOT_DIR=$SYSROOT \
PKG_CONFIG_LIBDIR=$SYSROOT/usr/lib/pkgconfig \
	pkg-config --cflags --libs zlib
#    Without SYSROOT_DIR, the -I and -L paths are absolute HOST paths.

#    (c) rpath leaking a build path.
aarch64-linux-gnu-gcc -o leaky hello.c -Wl,-rpath,/home/me/build/lib
readelf -d leaky | grep RUNPATH
#    -> ships a path that does not exist on the target. Use $ORIGIN.

# 4. Check what a cross-built binary actually requires:
aarch64-linux-gnu-objdump -p hello-arm64 | grep NEEDED
aarch64-linux-gnu-readelf -d hello-arm64 | grep -E 'RUNPATH|RPATH'
```

---

## 3. Mastery drills

1. Write an ELF parser that dumps the header, program headers, sections, dynamic entries, and
   symbols. Compare your output with `readelf` until they match.

2. Write a minimal ELF loader in userspace: `mmap` the segments, set up a stack with argv,
   envp and auxv, and jump to the entry point. Make it run a static binary.

3. Implement a `dlopen`-based plugin system, then deliberately hit the static-TLS-block
   problem and diagnose it from the error message alone.

4. Build the same application against glibc, musl, and uClibc-ng. Measure size, startup time,
   RSS, and throughput on a real workload. Document every source incompatibility you hit.

5. Take a binary that requires a new glibc and make it run on an old system, three different
   ways: rebuild on an old toolchain, static linking, and shipping the loader plus libraries
   with `$ORIGIN` rpath. Compare the results.

6. Write an `LD_PRELOAD` shim that intercepts `open`/`openat` and logs every file a program
   touches. Then use it to audit an application's real dependencies.

7. Measure the cost of dynamic linking: compare startup time for a binary with 5, 50, and 200
   shared library dependencies, with and without `LD_BIND_NOW`. Explain the curve with
   `LD_DEBUG=statistics`.

8. Read `fs/binfmt_elf.c`'s `load_elf_binary()` completely and write out the exact sequence,
   including where the point of no return is and what happens if something fails after it.

9. Explore `binfmt_misc`: register a handler for a custom format and make the kernel execute
   it. Explain how `qemu-user` transparent execution and Java `.class` execution work.

10. Extract the vDSO from a running process, disassemble `__vdso_clock_gettime`, and trace
    exactly how it reads the timekeeping data under the seqlock. Explain what happens if the
    kernel updates the data mid-read.

11. Build a fully static, `-static-pie`, musl-based single-binary appliance. Verify ASLR is
    active, measure the size, and enumerate what you gave up.

12. Design the toolchain and libc strategy for a product line with a 10-year lifetime,
    multiple SoCs, and third-party binary blobs. State the constraints the blobs impose.

---

## 4. Further reading

**Specifications**
- The ELF specification (System V ABI, generic part) and the per-architecture supplements
  (x86-64 psABI, AArch64 psABI) — the authoritative documents
- `man elf(5)`, `ld.so(8)`, `dlopen(3)`, `execve(2)`

**Essential reading**
- Ulrich Drepper, ***How To Write Shared Libraries*** — free PDF. **The definitive document
  on dynamic linking**, PLT/GOT, symbol versioning, and visibility. Dense and worth it
- Ulrich Drepper, "What Every Programmer Should Know About Memory" — Ch. 105's reference, but
  the TLS and memory-layout sections apply here
- John R. Levine, *Linkers and Loaders* — the textbook; free drafts online

**Kernel**
- `fs/binfmt_elf.c` — read `load_elf_binary()` in full
- `fs/exec.c` — the credential and point-of-no-return handling
- `Documentation/admin-guide/binfmt-misc.rst`
- `Documentation/ABI/` — the userspace ABI commitments

**libc**
- musl's source — **genuinely readable**, unlike glibc. Read `src/thread/pthread_create.c`
  and `src/ldso/dynlink.c` for a clear implementation of everything in this chapter
- glibc's `elf/rtld.c` and the `LD_DEBUG` implementation
- The musl-vs-glibc comparison on musl.libc.org — written by musl's author but honest
- Rich Felker's writing on musl's design decisions

**Books**
- Bryant & O'Hallaron, *Computer Systems: A Programmer's Perspective*, ch. 7 (linking) —
  the best pedagogical treatment
- Michael Kerrisk, *The Linux Programming Interface*, ch. 41–42 (shared libraries)
- Chris Simmonds, *Mastering Embedded Linux Programming*, ch. 2 (toolchains), ch. 5

**Tools**
- `readelf`, `objdump`, `nm`, `ldd`, `strings`, `checksec`, `pahole`
- `LD_DEBUG` — read `LD_DEBUG=help` output and try each mode once
- `lddtree` (from pax-utils) — a dependency tree, safer than `ldd`
- `patchelf` — modify rpath/interpreter of an existing binary
- `bloaty` — what is actually in your binary, by size

**Cross-references**
- Ch. 20 — `task_struct` and process creation, the kernel side
- Ch. 24 — the syscall ABI, `vdso`, and UAPI stability
- Ch. 91 — what PID 1 execs, and why daemonization is wrong
- Ch. 96–100 — Yocto and Buildroot, which exist to make §T.10 tractable
- Ch. 102 — RELRO, PIE, canaries, and `AT_SECURE` as security mechanisms

→ Next: [93-containers.md](93-containers.md)

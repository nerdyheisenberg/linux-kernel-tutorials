# Chapter 05 — Your First Module, and What `insmod` Really Does

> **Goal:** write a non-trivial module with parameters, sysfs, debugfs and proper cleanup,
> and be able to explain every step the kernel takes from `insmod` to `module_init()`.

---

## Theory & First Principles

> **How to read this section.** T.0 builds a module from nothing and explains every line.
> T.1–T.4 are the theory: late binding, symbol namespaces, structural typing across a binary
> boundary, and lifecycle. T.5–T.8 cover `printk`, error handling, and the arguments.

---

### T.0 — Start here: the smallest real module, line by line

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>      // ① module_init, MODULE_LICENSE, THIS_MODULE
#include <linux/kernel.h>      // ② pr_info and friends

static int __init hello_init(void)   // ③ __init: freed after this runs
{
	pr_info("hello: loaded\n");  // ④ not printf -- there is no libc
	return 0;                    // ⑤ 0 = success; NEGATIVE errno = failure
}

static void __exit hello_exit(void)  // ⑥ __exit: omitted entirely if built-in
{
	pr_info("hello: unloaded\n");
}

module_init(hello_init);             // ⑦ register the constructor
module_exit(hello_exit);             //   and the destructor

MODULE_LICENSE("GPL");               // ⑧ NOT decoration -- see below
MODULE_AUTHOR("You");
MODULE_DESCRIPTION("Minimal module");
```

```makefile
obj-m := hello.o
all:  ; $(MAKE) -C ~/src/linux M=$(PWD) modules
clean:; $(MAKE) -C ~/src/linux M=$(PWD) clean
```

```bash
make && sudo insmod hello.ko && dmesg | tail -2 && sudo rmmod hello && dmesg | tail -1
```

**Now the eight annotations, because each one is a door into a real subsystem:**

① **`THIS_MODULE`** is a pointer to your `struct module`, which the build system generates
into `hello.mod.c`. Look at it — `cat hello.mod.c` — it is generated code you never wrote,
containing your metadata, your symbol CRCs (§T.3), and your dependency list. Modules are not
magic; they are ordinary ELF objects plus this generated descriptor.

② **There is no libc.** `pr_info` is not `printf`; `kmalloc` is not `malloc`; there is no
`stdio.h`, no `errno` global, no floating point. Ch. 01 §T.2 explains the `-ffreestanding`
machinery; the consequence is that **every convenience you know from userspace has a
different kernel spelling and slightly different semantics.**

③ **`__init` puts the function in a section (`.init.text`) that is freed after boot.** Look
for `Freeing unused kernel image memory` in `dmesg`. This is the linker-section trick from
Ch. 01 §T.0 fact 4, used for real: a few hundred kilobytes recovered on every boot.

④ `pr_info` writes into a lock-free ring buffer with a priority level (§T.5). It is safe
from almost any context — including interrupt handlers — which is exactly why it is the
debugging tool of last resort that always works.

⑤ **Return 0 or a negative errno.** There are no exceptions in the kernel. This convention
propagates into every function you will write, into `ERR_PTR`/`IS_ERR` for functions
returning pointers (§T.6), and into Rust's `Result<T, Error>` (Ch. 79 §T.2).

⑥ **`__exit` is *discarded* when the code is built in**, because a built-in module can never
be unloaded. This is why `__exit` functions may not be called from `__init` ones, and why
`checkpatch` complains if you try.

⑦ `module_init` does not "call at load time" in any simple sense. For a loadable module it
registers the entry point in the module descriptor; **for a built-in it places a function
pointer in an initcall section**, and the ordering of those sections is the boot ordering
(Ch. 03 §T.6). One macro, two completely different mechanisms.

⑧ **`MODULE_LICENSE("GPL")` is load-bearing.** Without a GPL-compatible string the kernel
sets the `P` taint flag *and* refuses you access to every `EXPORT_SYMBOL_GPL` symbol — which
is most of the interesting ones. Try it: change it to `"Proprietary"`, add a call to a
GPL-only symbol, and watch the load fail with `Unknown symbol`. That is §T.2's subject.

**The one question to hold in your head through this chapter:** `insmod` took an `.o`-like
file and made it part of a running kernel. *How?* The answer — that `insmod` is a linker,
running at runtime, against a symbol table the kernel exports — is §T.1, and it makes
modules stop being mysterious forever.

---

### T.1 A module is a late-bound plugin, and `insmod` is a linker

Ch. 01 T.1 explained relocation at *build* time. A kernel module applies the same theory at
*run* time. A `.ko` is an **ELF relocatable object** (`ET_REL`), not an executable and not a
shared object:

```bash
readelf -h hello.ko | grep Type      # → REL (Relocatable file)
readelf -h /bin/ls   | grep Type     # → DYN (Position-Independent Executable)
```

That distinction is deliberate and carries real consequences. A userspace `.so` is `ET_DYN`:
position-independent, resolved lazily through a PLT/GOT, and mapped by `ld.so` in userspace.
A kernel module is `ET_REL`: the kernel's in-tree linker (`kernel/module/main.c`) must
**allocate memory, copy sections, resolve every symbol eagerly, and apply every relocation
by hand** — there is no dynamic loader, no PLT indirection (except on architectures like
arm64 where the module might land outside branch range), and no lazy binding.

Why eager, non-lazy resolution? Three reasons, all of which are good design lessons:

1. **Failure must be detectable before the code can run.** A lazily-resolved missing symbol
   would fault at an arbitrary point with the kernel in an arbitrary state. Eager resolution
   turns it into a clean `insmod: ERROR: could not insert module: Unknown symbol in module`.
2. **No fault handling available.** Lazy binding requires trapping a call to an unresolved
   stub — a mechanism that is fine in userspace but ugly in a kernel that may be in any
   context when the call happens.
3. **Determinism.** Load-time cost is predictable; no surprise latency on first call.

This is the general principle: **move failure as early as possible in the lifecycle,
where the error path is simple.** You will apply the same reasoning to driver `probe()`
(Ch. 27) and to `Kconfig` dependencies (Ch. 03).

### T.2 The symbol namespace problem, and three layers of answer

The kernel exports ~35,000 symbols. Three distinct problems arise, with three mechanisms:

**(a) Which symbols are even visible?** `EXPORT_SYMBOL()` places an entry in a dedicated
section (`__ksymtab`) that the module loader searches. Everything else is invisible to
modules, even if it's globally visible within `vmlinux`. This is *information hiding*
(Ch. 02 T.1) enforced by the linker.

```bash
grep -c . /proc/kallsyms                      # all symbols
awk '$2 ~ /[TtWw]/' /proc/kallsyms | wc -l     # text symbols
sudo cat /proc/kallsyms | grep -c '\[.*\]'     # symbols from modules
readelf -p __ksymtab_strings vmlinux | head    # the exported names
```

**(b) Is this symbol legally usable by proprietary code?** `EXPORT_SYMBOL_GPL()` marks a
symbol as usable only by modules declaring a GPL-compatible `MODULE_LICENSE()`. This is a
*technical enforcement of a legal position*: the kernel community asserts that a module
using a core kernel interface is a derived work. Whether that holds in court is untested;
what matters practically is that `insmod` refuses to load, and that the kernel is marked
**tainted** (`/proc/sys/kernel/tainted`) when a proprietary module loads — so maintainers can
decline to debug the resulting crashes. Understanding that taint is *a triage mechanism*,
not a punishment, is important context.

```bash
cat /proc/sys/kernel/tainted
# decode it:
for i in $(seq 0 18); do echo "bit $i: $(( ($(cat /proc/sys/kernel/tainted) >> i) & 1 ))"; done
$EDITOR Documentation/admin-guide/tainted-kernels.rst
```

**(c) Should *this* subsystem's internals be exported to everyone?** Even `_GPL` exports are
global. **Symbol namespaces** (5.4+, `EXPORT_SYMBOL_NS()`) add a second dimension: a module
must declare `MODULE_IMPORT_NS("SUBSYS")` to use them. This turns "please don't use this
outside USB core" from a comment into a build-time check.

```c
EXPORT_SYMBOL_NS_GPL(usb_foo, "USB_STORAGE");   /* provider */
MODULE_IMPORT_NS("USB_STORAGE");                /* consumer */
```

The progression — global → license-gated → namespace-gated — is the kernel repeatedly
discovering the same lesson: **make the intended coupling mechanically checkable** (Ch. 02
T.5, Ch. 03 T.5).

### T.3 Modversions: structural typing across a binary boundary

The kernel has no stable in-kernel ABI (Ch. 00 T.5), so a module built against 6.6 may be
silently incompatible with 6.7 — same symbol name, different struct layout or argument list.
Calling it would corrupt memory with no diagnostic.

`CONFIG_MODVERSIONS` solves this with a **structural checksum**. `genksyms` (or, for
Clang/LTO builds, `gendwarfksyms`) computes a CRC over the *expanded type graph* of each
exported symbol — the function signature, plus every struct it transitively references, fully
expanded. Change a field in `struct file` and every symbol whose signature reaches `struct
file` gets a new CRC.

```bash
head -5 /path/to/Module.symvers            # CRC  symbol  module  export-type  namespace
modprobe --dump-modversions hello.ko | head
readelf -x __versions hello.ko | head
```

This is **nominal-name + structural-type checking across a binary boundary** — exactly what
a type system does, implemented with a hash because the "type checker" runs at `insmod` time
with no type information available. Recognize the pattern: when you need type safety across
a boundary that can't carry types, hash the type and compare.

Its limits are worth stating precisely, because people over-trust it: it catches *signature*
changes, not *semantic* changes. If `foo()` keeps its prototype but changes its locking
requirements, the CRC is identical and you get a deadlock. This is why distributions' "kABI
stability" promises are always accompanied by a whitelist and a lot of human review (Ch. 88).

### T.4 The module lifecycle is a constructor/destructor protocol with refcounting

```
insmod
  │  load_module():  copy sections → resolve symbols → apply relocations → mark RO/NX
  │  parse module_param()s
  ├─▶ module_init(fn)  ──▶ fn() returns 0  → state = MODULE_STATE_LIVE
  │                        fn() returns <0 → unwind, free, insmod fails with that errno
  │
  │  ... module is live. refcount tracks users ...
  │
rmmod
  │  if (module_refcount() != 0) → -EBUSY
  ├─▶ module_exit(fn) ──▶ fn() must undo EVERYTHING init did, in reverse order
  └─▶ free_module(): unmap text/data
```

Two invariants that cause almost every module bug:

**Invariant 1 — `exit` must be the exact inverse of `init`, in reverse order.** This is
manual destructor ordering. The reason reverse order matters is the same reason it matters
for C++ destructors or Rust `Drop`: later-created objects may reference earlier ones.
Tearing down an earlier object first leaves a dangling reference. (Ch. 28's `devres`
automates exactly this for drivers, and it unwinds in reverse for the same reason.)

**Invariant 2 — nothing may still be able to call into the module's text when `free_module()`
runs.** The module's code is *unmapped*. Any surviving reference is a jump into freed memory.
The sources of such references are a checklist you should memorize:

| Left behind | Must call in exit |
|---|---|
| `call_rcu()` callback | `rcu_barrier()` |
| `work_struct` / `delayed_work` | `cancel_work_sync()` / `cancel_delayed_work_sync()` |
| `timer_list` | `timer_shutdown_sync()` |
| `hrtimer` | `hrtimer_cancel()` |
| kthread | `kthread_stop()` |
| registered IRQ handler | `free_irq()` |
| any `*_register()` | the matching `*_unregister()` |
| sysfs/debugfs nodes | `debugfs_remove_recursive()` etc. |
| an open file with your `f_ops` | `.owner = THIS_MODULE` handles it via refcount |

`.owner = THIS_MODULE` in an ops table is the kernel's answer to "userspace holds a reference
to my code": the VFS takes a module reference on `open()` and drops it on `release()`, so
`rmmod` returns `-EBUSY` instead of crashing. **Forgetting `.owner` is a classic
use-after-free.**

### T.5 `printk`: a lock-free ring buffer with a priority channel

`printk` looks trivial and is not. Its requirements are brutal and mutually hostile:

- Callable from **any** context, including NMI, early boot before memory management, and
  from inside the scheduler and the console driver itself.
- Must not deadlock against itself (a console driver that calls `printk`).
- Must not lose the last messages before a panic.
- Must not add unbounded latency (a slow serial console at 115200 baud emits ~11 KB/s;
  a verbose `printk` storm can add *seconds* of latency and is a known RT killer).

The solution (`kernel/printk/printk_ringbuffer.c`) is a **multi-writer lock-free ring
buffer** with separate descriptor and data rings, where writers reserve space with a CAS and
commit with a state transition. Printing to the console is decoupled from writing to the
buffer, and since 6.x there is ongoing work on threaded/atomic consoles precisely to bound
the latency.

Two design ideas transfer to your own work:
1. **Separate the fast, always-possible operation (record) from the slow, context-dependent
   one (emit).** Ch. 17's top/bottom half is the same idea.
2. **When you must work in NMI, you must be lock-free.** There is no other option.

**Log levels** are a priority channel, not decoration:

```c
KERN_EMERG   "0"  /* system unusable */
KERN_ALERT   "1"  /* action must be taken immediately */
KERN_CRIT    "2"
KERN_ERR     "3"  /* pr_err()   — a real error a user should act on */
KERN_WARNING "4"  /* pr_warn()  — something unexpected but handled */
KERN_NOTICE  "5"
KERN_INFO    "6"  /* pr_info()  — normal operational info; be sparing */
KERN_DEBUG   "7"  /* pr_debug() — compiled out unless DEBUG/dynamic_debug */
```
`/proc/sys/kernel/printk` holds `current default minimum boot-time`. The console shows
messages *below* the current level numerically.

The discipline reviewers enforce: **a driver that prints on every successful operation is
broken.** Success is silent. Use `dev_dbg()` for detail, `dev_err()` for failures the admin
can act on, and `_ratelimited` variants for anything reachable from a hostile input path
(an unratelimited `pr_err` in a packet-receive path is a denial-of-service vector).

### T.6 Error handling without exceptions: three conventions

C has no exceptions, so the kernel standardizes three patterns. Using them correctly is the
single most visible marker of whether you've written kernel code before.

**(a) Negative errno for `int`-returning functions.**
```c
int foo(void) { if (bad) return -EINVAL; return 0; }
```
Errno *values* are a UAPI (`include/uapi/asm-generic/errno-base.h`). Choosing the right one
matters: `-EINVAL` (bad argument), `-ENODEV` (no such device), `-EPROBE_DEFER` (try later —
Ch. 27), `-ENOMEM`, `-EBUSY`, `-EOPNOTSUPP` vs `-ENOTSUPP` (the former is UAPI-visible;
the latter is kernel-internal and **must not** reach userspace), `-ERESTARTSYS` (restartable
after a signal).

**(b) `ERR_PTR` — encoding an error in a pointer.** This is a lovely piece of engineering
worth understanding rather than memorizing:

```c
/* include/linux/err.h */
#define MAX_ERRNO 4095
static inline void *ERR_PTR(long error)   { return (void *)error; }
static inline long  PTR_ERR(const void *p){ return (long)p; }
static inline bool  IS_ERR(const void *p) { return (unsigned long)p >= (unsigned long)-MAX_ERRNO; }
```
It works because the **last 4096 bytes of the address space are never a valid kernel
pointer** — the architecture reserves them, and the kernel guarantees no object lives there.
So the top 4096 addresses form a disjoint tag space for error codes. This is
**pointer tagging**: stealing unused bits (here, the top of the range) to encode a second
kind of value in one word, avoiding an out-parameter or a wrapper struct.

```c
struct foo *foo_create(void)
{
	struct foo *f = kzalloc(sizeof(*f), GFP_KERNEL);
	if (!f)
		return ERR_PTR(-ENOMEM);
	return f;
}
/* caller: */
f = foo_create();
if (IS_ERR(f))
	return PTR_ERR(f);
/* or, for APIs that may return NULL meaning "none": */
if (IS_ERR_OR_NULL(f)) ...
```
**Rule:** a function returns *either* `NULL`-on-failure *or* `ERR_PTR`-on-failure, never
both, and the choice must be documented. Mixing them is a common bug.

**(c) The `goto` unwind ladder — manual RAII.** Since C has no destructors, the kernel
standardizes on a single-exit unwinding pattern:

```c
static int probe(void)
{
	int ret;

	a = alloc_a();
	if (!a)
		return -ENOMEM;

	b = alloc_b();
	if (!b) {
		ret = -ENOMEM;
		goto err_free_a;
	}

	ret = register_c();
	if (ret)
		goto err_free_b;

	return 0;

err_free_b:
	free_b(b);
err_free_a:
	free_a(a);
	return ret;
}
```
Labels are named for **what they undo**, and the ladder unwinds in reverse construction
order — the same invariant as T.4. This is manual, error-prone, and the source of an entire
class of leak/UAF bugs, which is precisely why the kernel added:

- `devm_*` (Ch. 28) — tie resource lifetime to a device.
- `guard()` / `scoped_guard()` / `__free()` (`include/linux/cleanup.h`, 6.4+) — real RAII in
  C via `__attribute__((cleanup))`.
- Rust (Part 5) — where `Drop` makes this the compiler's problem.

Seeing the ladder as *"hand-written destructors"* rather than *"weird goto style"* is the
conceptual step that makes the rest of the kernel legible.

---

### T.7 — What is still argued about

**1. Should modules exist at all?**
They are a large attack surface (a signed-module bypass is a kernel compromise), they create
the symbol-namespace problem (§T.2), and `CONFIG_MODULES=n` is a real hardening measure for
fixed-function products (Ch. 102). The counter-argument is decisive for distributions: one
binary cannot contain every driver and also fit in memory. **The resolution is that both are
right for their context**, and knowing which context you are in is the skill.

**2. Is `EXPORT_SYMBOL_GPL` a technical mechanism or a legal one?**
It is technically trivial to bypass and is not a licence-enforcement mechanism in any
meaningful sense. What it *is* is a clear statement of intent that makes a proprietary
module's derivative-work status harder to dispute. Whether that is the right use of a
technical facility is genuinely contested; what is not contested is that you must respect the
distinction in code you write.

**3. Is modversions (`CONFIG_MODVERSIONS`) worth it?**
It catches struct-layout mismatches that would otherwise be silent memory corruption (§T.3)
— a real and severe failure mode. It also produces the widely-hated situation where a module
refuses to load against a kernel that would have worked fine. Distributions rely on it;
upstream is ambivalent; the underlying tension is that Linux refuses a stable internal ABI
(Ch. 00 §T.5) and modversions is the resulting workaround.

**4. `printk` in 2026?**
It is slow (console I/O at 115200 baud is hundreds of milliseconds), it perturbs timing, and
it has caused more production outages than some of the bugs it was added to find. The modern
answer is tracepoints — zero cost when disabled, structured, filterable (Ch. 95 §T.1). The
counter-argument is that `printk` works in NMI context, during early boot, and after the
system is otherwise dead, which nothing else does. **Both are true: `printk` for the
unrecoverable, tracepoints for everything else.**

### T.8 — The compressed model

```
 A MODULE IS ORDINARY ELF PLUS A GENERATED DESCRIPTOR. `insmod` is a LINKER
 running at runtime: it resolves undefined symbols against the kernel's
 exported symbol table, applies relocations, and calls the init function.

 THREE LAYERS OF SYMBOL CONTROL:
   EXPORT_SYMBOL      -- available to any module
   EXPORT_SYMBOL_GPL  -- available only to GPL-declared modules
   EXPORT_SYMBOL_NS   -- available only to modules importing the namespace

 MODVERSIONS IS STRUCTURAL TYPING ACROSS A BINARY BOUNDARY: a CRC over the
 expanded type of every exported symbol. Mismatch = refuse to load, instead
 of silent memory corruption.

 THE LIFECYCLE IS A CONSTRUCTOR/DESTRUCTOR PROTOCOL WITH REFCOUNTING.
 init returns 0 or -errno. exit must undo EXACTLY what init did, in reverse.
 try_module_get/module_put keep the module alive while it is in use.

 NO EXCEPTIONS. Return -errno; unwind by hand with goto labels; ERR_PTR for
 pointer-returning functions. This convention reaches all the way into Rust.
```

Five questions for any module you write or review:

1. **Does `exit` undo exactly what `init` did, in reverse order?** Diff them side by side.
2. **Is every error path in `init` unwinding correctly?** This is where the bugs are.
3. **Can anything still reference me after `exit` starts?** Open fds, queued work, armed
   timers, registered callbacks (Ch. 25 P12).
4. **Is the licence string right, and am I using GPL-only symbols?**
5. **Does it build and work as both `=y` and `=m`?**

---

## 1. Practice first: a real module

`hello.c`:

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/moduleparam.h>
#include <linux/slab.h>
#include <linux/debugfs.h>
#include <linux/device.h>
#include <linux/mutex.h>

#define DRV_NAME "hello"

static char *whom = "world";
module_param(whom, charp, 0444);
MODULE_PARM_DESC(whom, "Who to greet");

static int count = 1;
module_param(count, int, 0644);          /* writable via /sys/module/hello/parameters */
MODULE_PARM_DESC(count, "How many times to greet");

struct hello_ctx {
	struct mutex	lock;
	unsigned long	greetings;
	char		*buf;
	struct dentry	*dbg_dir;
};

static struct hello_ctx *ctx;

static ssize_t greetings_show(struct kobject *kobj, struct kobj_attribute *attr,
			      char *page)
{
	unsigned long v;

	mutex_lock(&ctx->lock);
	v = ctx->greetings;
	mutex_unlock(&ctx->lock);

	return sysfs_emit(page, "%lu\n", v);
}

static ssize_t greetings_store(struct kobject *kobj, struct kobj_attribute *attr,
			       const char *page, size_t len)
{
	unsigned long v;
	int ret;

	ret = kstrtoul(page, 0, &v);
	if (ret)
		return ret;

	mutex_lock(&ctx->lock);
	ctx->greetings = v;
	mutex_unlock(&ctx->lock);

	return len;
}
static struct kobj_attribute greetings_attr = __ATTR_RW(greetings);

static struct attribute *hello_attrs[] = {
	&greetings_attr.attr,
	NULL,
};
ATTRIBUTE_GROUPS(hello);

static struct kobject *hello_kobj;

static int __init hello_init(void)
{
	int i, ret;

	ctx = kzalloc(sizeof(*ctx), GFP_KERNEL);
	if (!ctx)
		return -ENOMEM;

	mutex_init(&ctx->lock);

	ctx->buf = kmalloc(PAGE_SIZE, GFP_KERNEL);
	if (!ctx->buf) {
		ret = -ENOMEM;
		goto err_free_ctx;
	}

	hello_kobj = kobject_create_and_add(DRV_NAME, kernel_kobj);
	if (!hello_kobj) {
		ret = -ENOMEM;
		goto err_free_buf;
	}

	ret = sysfs_create_groups(hello_kobj, hello_groups);
	if (ret)
		goto err_put_kobj;

	ctx->dbg_dir = debugfs_create_dir(DRV_NAME, NULL);
	debugfs_create_ulong("greetings", 0444, ctx->dbg_dir, &ctx->greetings);

	for (i = 0; i < count; i++) {
		pr_info("hello, %s! (%d/%d)\n", whom, i + 1, count);
		ctx->greetings++;
	}

	pr_info("loaded at %p, THIS_MODULE=%p name=%s\n",
		hello_init, THIS_MODULE, THIS_MODULE->name);
	return 0;

err_put_kobj:
	kobject_put(hello_kobj);
err_free_buf:
	kfree(ctx->buf);
err_free_ctx:
	kfree(ctx);
	return ret;
}

static void __exit hello_exit(void)
{
	debugfs_remove_recursive(ctx->dbg_dir);
	sysfs_remove_groups(hello_kobj, hello_groups);
	kobject_put(hello_kobj);
	kfree(ctx->buf);
	mutex_destroy(&ctx->lock);
	kfree(ctx);
	pr_info("goodbye\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("You <you@example.com>");
MODULE_DESCRIPTION("A first module with params, sysfs and debugfs");
MODULE_VERSION("1.0");
```

`Makefile`:
```makefile
obj-m += hello.o
KDIR ?= /lib/modules/$(shell uname -r)/build
all:
	$(MAKE) -C $(KDIR) M=$(CURDIR) modules
clean:
	$(MAKE) -C $(KDIR) M=$(CURDIR) clean
```

Run:
```bash
make
sudo insmod hello.ko whom=kernel count=3
dmesg | tail
cat /sys/kernel/hello/greetings
cat /sys/module/hello/parameters/whom
echo 100 | sudo tee /sys/kernel/hello/greetings
sudo cat /sys/kernel/debug/hello/greetings
modinfo hello.ko
sudo rmmod hello
```

### Things this example teaches
- `pr_fmt` you should add: put `#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt` **before**
  any include so every `pr_*` is prefixed automatically.
- **Goto-based unwind** — the canonical kernel error-handling idiom. Labels named after
  what they undo, reverse order of acquisition.
- `sysfs_emit()` not `sprintf()` — it enforces the PAGE_SIZE contract.
- `__ATTR_RW` / `ATTRIBUTE_GROUPS` — the modern sysfs boilerplate.
- debugfs calls **do not need error checking** (they return error pointers that later
  calls tolerate) — deliberate API design.
- `__init` / `__exit` section placement.

---

## 2. Internals: what `insmod` actually does

### 2.1 Userspace side

```
insmod foo.ko
  → open("foo.ko")
  → finit_module(fd, "param=value", flags)      /* preferred; kernel reads the file */
     (older: read whole file, init_module(buf, len, args))
```

`modprobe` differs: it reads `/lib/modules/$(uname -r)/modules.dep`, loads dependencies
first, applies `/etc/modprobe.d/*.conf` options and blacklists, then calls `finit_module`.

```bash
strace -f -e trace=finit_module,init_module modprobe brd 2>&1 | tail
```

### 2.2 Kernel side — `kernel/module/main.c`

```
SYSCALL_DEFINE3(finit_module, ...)
  └─ load_module(&info, uargs, flags)
       1. elf_validity_cache_copy()   – sanity-check ELF header, sections
       2. layout_and_allocate()
            – rewrite_section_headers()
            – layout_sections()        → assign offsets in the module's
                                         text/rodata/data/bss "core" and "init" regions
            – module_memory_alloc()    → module_alloc() : vmalloc in the module region
                                         (x86: within ±2GB of kernel text so that
                                          direct call/jmp relocations fit)
       3. find_module_sections()       – locate __versions, __param, .altinstructions,
                                         __ksymtab, __bug_table, .init.text, __tracepoints,
                                         .orc_unwind, __jump_table, .BTF …
       4. check_module_license_and_versions()
            – MODULE_LICENSE missing/proprietary → set TAINT_PROPRIETARY_MODULE
            – GPL-only symbol use by non-GPL module → reject
       5. module_sig_check()           – verify appended PKCS#7 signature
                                         (CONFIG_MODULE_SIG_FORCE → mandatory)
       6. simplify_symbols()           – resolve undefined symbols:
                                         resolve_symbol_wait() → find_symbol()
                                         searches the kernel's __ksymtab and every
                                         loaded module's exports
            – CONFIG_MODVERSIONS: compare CRC in __versions against Module.symvers
              → "disagrees about version of symbol X" if mismatched
       7. apply_relocations()          – arch-specific: apply_relocate_add()
                                         in arch/*/kernel/module.c
       8. post_relocation()
            – module_finalize(): alternatives, paravirt patching, ftrace records,
              jump labels, objtool ORC tables, set memory RO/NX
       9. add_unformed_module / complete_formation()
            – module goes MODULE_STATE_UNFORMED → COMING
            – module_enable_ro()/nx(), mark text RO+X, rodata RO, data NX
      10. do_init_module()
            – parse_args()             ★ module parameters applied HERE
            – do_one_initcall(mod->init)   ★ YOUR module_init() runs
            – state → MODULE_STATE_LIVE
            – blocking_notifier_call_chain(MODULE_STATE_LIVE)
            – module_put_and_kthread_exit / free the .init sections (async)
```

> **Key insight:** a module is an ordinary ELF *relocatable* object (`ET_REL`, like a `.o`),
> not a shared library. The kernel is a static linker at runtime.

Inspect it:
```bash
readelf -h hello.ko | grep Type              # REL (Relocatable file)
readelf -S hello.ko | head -40               # sections
readelf -r hello.ko | head -30               # relocations to resolve
readelf -p .modinfo hello.ko                 # modinfo strings
objdump -d hello.ko | head -40
```

### 2.3 `rmmod` path

```
delete_module(name, flags)
  → find_module()
  → try_stop_module()
       – module refcount must be 0  (module_refcount(mod))
       – state → MODULE_STATE_GOING
       – uses stop_machine() unless CONFIG_MODULE_FORCE_UNLOAD tricks apply
  → mod->exit()                   ★ your module_exit()
  → free_module()
       – remove from lists, synchronize_rcu(), ftrace/kprobe cleanup
       – module_memory_free()
```

The refcount is what `try_module_get()` / `module_put()` manipulate.
Every open file whose `f_op->owner = THIS_MODULE` holds a reference — that's why you
can't `rmmod` a driver with an open `/dev` node.

```bash
lsmod                    # Used-by column = refcount + dependents
cat /proc/modules        # raw: name size refcnt deps state addr
cat /sys/module/hello/refcnt
cat /sys/module/hello/sections/.text   # load address (root only)
```

### 2.4 Symbol export

```c
EXPORT_SYMBOL(foo);          /* any module */
EXPORT_SYMBOL_GPL(foo);      /* only MODULE_LICENSE("GPL"*) modules */
EXPORT_SYMBOL_NS(foo, "NS"); /* namespaced; importer needs MODULE_IMPORT_NS("NS") */
EXPORT_SYMBOL_GPL_FOR_MODULES(foo, "a,b");  /* 6.13+: restrict to named modules */
```

These place entries in `__ksymtab`, `__ksymtab_gpl`, `__kcrctab*`, `__ksymtab_strings`.

```bash
grep ' T \| t ' /proc/kallsyms | wc -l
cat Module.symvers | head        # CRC  symbol  module  export-type  namespace
```

**Namespaces** exist so that e.g. `USB_STORAGE` internal symbols aren't casually used
by unrelated drivers. If you write a subsystem with "internal but must be exported"
symbols, use a namespace.

### 2.5 MODVERSIONS (`CONFIG_MODVERSIONS=y`)

`genksyms` (or the newer `gendwarfksyms`) computes a CRC over the *expanded type* of each
exported symbol's prototype. If a struct in the signature changes, the CRC changes, and
the module is refused. This is what gives distro kernels a semi-stable module ABI within
a release series.

```bash
modprobe --dump-modversions hello.ko
```

### 2.6 Module signing & lockdown

```bash
# In .config
CONFIG_MODULE_SIG=y
CONFIG_MODULE_SIG_ALL=y
CONFIG_MODULE_SIG_SHA256=y
CONFIG_MODULE_SIG_FORCE=y     # refuse unsigned modules
```
Signature is appended after the ELF, with a `~Module signature appended~` magic trailer.
Under Secure Boot + `lockdown=integrity`, unsigned modules are rejected and
`/dev/mem`, kprobes on some paths, and unsigned BPF are restricted.

```bash
tail -c 100 hello.ko | xxd | tail -3
/usr/src/linux-headers-$(uname -r)/scripts/sign-file sha256 key.pem cert.der hello.ko
```

### 2.7 Module aliases and auto-loading

```c
MODULE_DEVICE_TABLE(of, acme_of_match);
MODULE_DEVICE_TABLE(pci, acme_pci_ids);
MODULE_ALIAS("platform:acme-widget");
MODULE_ALIAS_CRYPTO("sha256");
MODULE_SOFTDEP("pre: foo post: bar");
```
`modpost` turns these into `alias=` entries in `.modinfo`; `depmod` collects them into
`/lib/modules/*/modules.alias`. When the bus core reports a `MODALIAS` uevent, udev runs
`modprobe $MODALIAS`. That is the entire autoload mechanism.

```bash
modinfo -F alias hello.ko
cat /sys/bus/pci/devices/0000:00:02.0/modalias
grep -m3 pci: /lib/modules/$(uname -r)/modules.alias
```

---

## 3. `printk` and logging — do it right

```c
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt   /* MUST be before includes */
#include <linux/kernel.h>

pr_emerg() pr_alert() pr_crit() pr_err() pr_warn() pr_notice() pr_info() pr_debug()

/* With a struct device — ALWAYS prefer these in drivers */
dev_err(dev, "failed: %d\n", ret);
dev_warn(dev, ...); dev_info(dev, ...); dev_dbg(dev, ...);
dev_err_probe(dev, ret, "failed to get clock\n");  /* ★ handles -EPROBE_DEFER quietly */

/* Rate limiting */
pr_err_ratelimited(...);  dev_err_ratelimited(dev, ...);
printk_once(...);         pr_warn_once(...);

/* Hex dump */
print_hex_dump(KERN_DEBUG, "regs: ", DUMP_PREFIX_OFFSET, 16, 1, buf, len, true);
```

**Format specifiers unique to the kernel** (`Documentation/core-api/printk-formats.rst`):

| Spec | Prints |
|---|---|
| `%pK` | pointer, hashed/hidden per `kptr_restrict` |
| `%p` | **hashed** by default (since 4.15) — not the real address |
| `%px` | real address (use only when justified) |
| `%pS` / `%ps` | symbol name `func+0x1f/0x40` |
| `%pB` | symbol for a backtrace address |
| `%pF` | function descriptor (ia64/ppc64) |
| `%pI4` / `%pI6` / `%pi4` | IPv4/IPv6 addresses |
| `%pM` / `%pMR` | MAC address |
| `%pd` / `%pD` | dentry / file name |
| `%pOF` / `%pfw` | device_node / fwnode full name |
| `%pr` / `%pR` | `struct resource` |
| `%pa` | `phys_addr_t` / `dma_addr_t` |
| `%pUb`/`%pUl` | UUID |
| `%pe` | error pointer → `-EINVAL` as text |
| `%pV` | `struct va_format` (for wrapper functions) |

**`dev_dbg` / `pr_debug`** are compiled out unless `CONFIG_DYNAMIC_DEBUG=y` (then runtime
controllable) or `DEBUG` is defined:
```bash
echo 'module hello +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file hello.c line 1-200 +pflmt' > /sys/kernel/debug/dynamic_debug/control
cat /sys/kernel/debug/dynamic_debug/control | grep hello
# boot param: dyndbg="module hello +p"
```

---

## 4. Error handling: the kernel idiom

```c
static int probe(struct device *dev)
{
	int ret;

	a = alloc_a();
	if (!a)
		return -ENOMEM;

	ret = init_b(a);
	if (ret)
		goto err_free_a;

	ret = register_c(a);
	if (ret)
		goto err_exit_b;

	return 0;

err_exit_b:
	exit_b(a);
err_free_a:
	free_a(a);
	return ret;
}
```

Rules:
1. Labels are named for **what they do**, not where they are (`err_free_a`, not `err1`).
2. Unwind in **exact reverse** order.
3. Return the error code, never `-1`.
4. `ERR_PTR()` / `IS_ERR()` / `PTR_ERR()` / `ERR_CAST()` for functions returning pointers:
```c
	struct foo *f = get_foo();
	if (IS_ERR(f))
		return PTR_ERR(f);
	/* IS_ERR_OR_NULL() when NULL is also possible */
```
5. Prefer `devm_*` (Chapter 28) so most of this vanishes.
6. Modern alternative: cleanup guards (`include/linux/cleanup.h`, 6.5+):
```c
	struct foo *f __free(kfree) = kmalloc(sz, GFP_KERNEL);
	guard(mutex)(&lock);           /* auto-unlock at scope exit */
	scoped_guard(spinlock_irqsave, &s->lock) { ... }
```

---

## 5. Extended practice

### Lab 5.A — Watch `insmod` link your module (T.1)

```bash
# Before loading: unresolved symbols and relocations
readelf -h hello.ko | grep Type                 # REL
nm -u hello.ko                                  # UND: _printk, __fentry__, ...
readelf -r hello.ko | head -30                  # relocation entries
readelf -S hello.ko | awk '{print $2,$3,$7}' | head -25

# Load it, then find where the sections landed
sudo insmod hello.ko
for s in /sys/module/hello/sections/.*; do echo "$(basename $s) $(sudo cat $s)"; done

# The symbols are now in the global namespace, tagged [hello]
sudo grep ' \[hello\]$' /proc/kallsyms

# Compare an address in the .ko file with the loaded address — the delta is the relocation
nm hello.ko | grep hello_init
sudo grep hello_init /proc/kallsyms
```

Now read the two functions that did this work:
```bash
$EDITOR kernel/module/main.c    # simplify_symbols(), apply_relocations(), post_relocation()
$EDITOR arch/x86/kernel/module.c # apply_relocate_add() — the actual R_X86_64_* handling
```

### Lab 5.B — Break modversions deliberately (T.3)

```bash
# 1. Build with MODVERSIONS
./scripts/config -e MODVERSIONS && make olddefconfig && make -j$(nproc)
make M=$PWD/hello modules
modprobe --dump-modversions hello/hello.ko

# 2. Change a struct the symbol depends on, rebuild the KERNEL only
#    e.g. add a field to a struct used in an exported function's signature
#    then compare Module.symvers CRCs:
cp Module.symvers /tmp/symvers.before
# ...edit include/linux/something.h ...
make -j$(nproc) && diff <(sort /tmp/symvers.before) <(sort Module.symvers) | head

# 3. Try to load the OLD module against the NEW kernel:
sudo insmod hello/hello.ko
dmesg | tail -3      # "disagrees about version of symbol ..."
```

### Lab 5.C — Symbol namespaces (T.2)

```c
/* provider.c */
int prov_thing(void) { return 42; }
EXPORT_SYMBOL_NS_GPL(prov_thing, "MY_SUBSYS");

/* consumer.c — omit the import first, then add it */
extern int prov_thing(void);
/* MODULE_IMPORT_NS("MY_SUBSYS"); */
```
```bash
make M=$PWD/ns modules
# Without the import, modpost fails:
#   ERROR: modpost: module consumer uses symbol prov_thing from namespace MY_SUBSYS,
#   but does not import it.
# Add MODULE_IMPORT_NS and rebuild.
grep -rn 'EXPORT_SYMBOL_NS' drivers/ | head -10      # real users
modinfo some_module.ko | grep import_ns
```

### Lab 5.D — Every way to leave a dangling reference (T.4)

Write one module that deliberately commits each sin, then fix each and verify:

```c
/* Sin 1: call_rcu without rcu_barrier */
static void my_cb(struct rcu_head *h) { kfree(container_of(h, struct obj, rcu)); }
/* ... call_rcu(&o->rcu, my_cb); ... and NO rcu_barrier() in exit */

/* Sin 2: work still queued */
schedule_delayed_work(&dw, HZ * 5);     /* and no cancel_delayed_work_sync() */

/* Sin 3: timer still armed */
mod_timer(&t, jiffies + HZ * 5);        /* and no timer_shutdown_sync() */

/* Sin 4: kthread still running */
kthread_run(loop_fn, NULL, "sinner");   /* and no kthread_stop() */

/* Sin 5: file_operations without .owner, with an open fd held by userspace */
```
```bash
# Build with KASAN + DEBUG_OBJECTS and loop:
./scripts/config -e KASAN -e DEBUG_OBJECTS -e DEBUG_OBJECTS_FREE \
                 -e DEBUG_OBJECTS_TIMERS -e DEBUG_OBJECTS_WORK -e DEBUG_OBJECTS_RCU_HEAD
while true; do sudo insmod sinner.ko && sudo rmmod sinner; done
dmesg -w
```
Each sin produces a *different, recognizable* splat. Learn to recognize all five; they cover
the majority of real module-unload crashes.

### Lab 5.E — `ERR_PTR` mechanics (T.6b)

```c
static int __init errdemo_init(void)
{
	void *p;

	p = ERR_PTR(-EINVAL);
	pr_info("ERR_PTR(-EINVAL) = %px, IS_ERR=%d, PTR_ERR=%ld, %%pe=%pe\n",
		p, IS_ERR(p), PTR_ERR(p), p);

	pr_info("MAX_ERRNO=%d, boundary=%px\n", MAX_ERRNO, (void *)-MAX_ERRNO);
	pr_info("a real pointer: %px IS_ERR=%d\n", &jiffies, IS_ERR(&jiffies));
	return 0;
}
```
Then answer in writing: why is `MAX_ERRNO` 4095 and not, say, 255? What would break if a
kernel object were allocated at address `0xfffffffffffff000`? (Hint: `mmap_min_addr`,
and the fact that the last page is never mapped.)

### Lab 5.F — Module parameters, all of them

```c
static int  count = 1;
static char *name = "default";
static bool verbose;
static int  arr[4];
static int  arr_len;

module_param(count, int, 0644);
MODULE_PARM_DESC(count, "how many widgets");
module_param(name, charp, 0444);
module_param(verbose, bool, 0644);
module_param_array(arr, int, &arr_len, 0444);

/* A parameter with a custom setter, live-updatable from sysfs: */
static int set_count(const char *val, const struct kernel_param *kp)
{
	int ret = param_set_int(val, kp);

	if (!ret)
		pr_info("count changed to %d\n", count);
	return ret;
}
static const struct kernel_param_ops count_ops = {
	.set = set_count, .get = param_get_int,
};
module_param_cb(count2, &count_ops, &count, 0644);
```
```bash
sudo insmod p.ko count=5 name=foo verbose=1 arr=1,2,3
cat /sys/module/p/parameters/*
echo 9 | sudo tee /sys/module/p/parameters/count
# Built-in (=y) modules take params on the kernel cmdline as  p.count=5
```

### Lab 5.G — Module signing and lockdown

```bash
# See whether your distro kernel enforces signatures:
cat /sys/kernel/security/lockdown 2>/dev/null
grep -E 'MODULE_SIG|LOCK_DOWN' /boot/config-$(uname -r)
mokutil --sb-state 2>/dev/null       # Secure Boot state

# Sign your own module against your build's key:
sudo ./scripts/sign-file sha256 certs/signing_key.pem certs/signing_key.x509 hello.ko
modinfo hello.ko | grep -E 'sig_id|signer|sig_key'
$EDITOR Documentation/admin-guide/module-signing.rst
```
This is essential background for Part 7 (secure boot, production embedded images).

### Lab 5.H — Convert the goto ladder to `guard()`/`__free()` (T.6c)

```c
#include <linux/cleanup.h>

/* Before */
static int old_way(void)
{
	char *buf = kmalloc(128, GFP_KERNEL);
	int ret;

	if (!buf)
		return -ENOMEM;
	mutex_lock(&lock);
	ret = do_thing(buf);
	if (ret)
		goto out;
	ret = do_other(buf);
out:
	mutex_unlock(&lock);
	kfree(buf);
	return ret;
}

/* After */
static int new_way(void)
{
	char *buf __free(kfree) = kmalloc(128, GFP_KERNEL);
	int ret;

	if (!buf)
		return -ENOMEM;

	guard(mutex)(&lock);
	ret = do_thing(buf);
	if (ret)
		return ret;
	return do_other(buf);
}
```
Verify the generated code is equivalent: `objdump -d` both and compare. Then read
`include/linux/cleanup.h`'s header comment for the pitfalls (notably: `__free` variables
must be initialized at declaration, and conditional ownership transfer needs
`no_free_ptr()`/`return_ptr()`).

---

## 6. Mastery drills

1. Add `pr_fmt` to `hello.c`, rebuild, and diff dmesg output.
2. Make `hello.ko` export a symbol and write a second module that uses it. Observe
   `lsmod` "Used by" and try to `rmmod` in the wrong order.
3. Add `MODULE_LICENSE("Proprietary")` and try to use an `EXPORT_SYMBOL_GPL` symbol.
   Read the exact error. Check `/proc/sys/kernel/tainted` and decode it via
   `Documentation/admin-guide/tainted-kernels.rst`.
4. Deliberately leak `ctx` in `hello_exit`. Boot with `CONFIG_DEBUG_KMEMLEAK=y`,
   `echo scan > /sys/kernel/debug/kmemleak`, read the report.
5. Read `kernel/module/main.c:load_module()` end to end. Write down the 10 stages.
6. Use `readelf -r hello.ko` to find an undefined symbol; then find its `__ksymtab` entry
   in `/proc/kallsyms`. Explain how `simplify_symbols()` connects them.
7. Rewrite `hello_init` using `__free()` and `guard()` from `cleanup.h`. Compare readability.
8. **Relocation types.** For your architecture, list the relocation types the module loader
   supports (`arch/*/kernel/module.c`). Explain what happens on arm64 when a module is
   loaded more than ±128 MiB from the kernel text (hint: `module_emit_plt_entry`).
9. **Eager vs lazy.** Write 300 words arguing why the kernel resolves symbols eagerly,
   using the three reasons in T.1 plus at least one you derive yourself.
10. **`EXPORT_SYMBOL` audit.** Pick a subsystem. Count `EXPORT_SYMBOL` vs
    `EXPORT_SYMBOL_GPL` vs `EXPORT_SYMBOL_NS*`. Find one plain `EXPORT_SYMBOL` that you
    think should be `_GPL` and find the discussion (`git log -S`) about it.
11. **printk discipline review.** Take any driver in `drivers/`. Count its `pr_info`/
    `dev_info` calls on success paths. Propose which should be `dev_dbg`. This is a real,
    frequently-accepted patch type (`git log --grep="reduce log noise"`).
12. **errno selection.** For each scenario pick the correct errno and justify:
    (a) userspace passed a size larger than the device supports;
    (b) the regulator this driver needs isn't registered yet;
    (c) the device is present but its firmware is missing;
    (d) an ioctl command number you don't implement;
    (e) a `read()` interrupted by a signal with data already copied.
13. **Read `printk_ringbuffer.c`'s design comment** (~150 lines at the top). Explain how a
    writer reserves space and why the descriptor and data rings are separate. Then explain
    why this must be lock-free (T.5).

---

## 7. Further reading

**Kernel documentation:**
- `Documentation/kbuild/modules.rst`
- `Documentation/core-api/printk-formats.rst` ★ — memorize `%pS %pe %pI4 %pOF %pK`
- `Documentation/core-api/printk-basics.rst`
- `Documentation/admin-guide/dynamic-debug-howto.rst`
- `Documentation/admin-guide/module-signing.rst`
- `Documentation/admin-guide/tainted-kernels.rst`
- `Documentation/core-api/symbol-namespaces.rst`
- `Documentation/process/license-rules.rst` — SPDX, and what `MODULE_LICENSE` means

**Source to read:**
- `kernel/module/main.c` — `load_module()`, `simplify_symbols()`, `apply_relocations()`
- `kernel/module/version.c`, `scripts/genksyms/`, `scripts/gendwarfksyms/`
- `kernel/params.c`
- `include/linux/err.h` — 40 lines, read all of it
- `include/linux/cleanup.h` — read the header comment; it's a tutorial
- `kernel/printk/printk_ringbuffer.c` — the design comment at the top

**Background:**
- Levine, *Linkers and Loaders* — Ch. 4 (relocation) and Ch. 8/10 (dynamic linking) make
  T.1 rigorous. Free online.
- Drepper, "How To Write Shared Libraries" — the userspace contrast to T.1
- LWN: "Symbol namespaces", "Module loading and the kernel", "The perils of EXPORT_SYMBOL",
  "A new API for dynamic debug", "Atomic consoles and printk"

→ Next: [06-debug-setup.md](06-debug-setup.md)

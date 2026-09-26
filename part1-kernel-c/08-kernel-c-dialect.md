# Chapter 08 — The Kernel C Dialect

> **Goal:** you write C that a kernel maintainer would accept, and you can explain every
> non-ISO construct you encounter in the tree.

---

## Theory & First Principles

> **How to read this section.** T.0 shows you kernel C that will not compile in userspace and
> explains why. T.1–T.4 are the theory of the dialect. T.5–T.10 are the idioms and the
> arguments about them.

---

### T.0 — Start here: why this C looks wrong

Here is real kernel code. Every line of it would confuse an experienced userspace C
programmer, and each confusion is a door into a real design decision.

```c
/* ① */  #define container_of(ptr, type, member) ({                    \
             void *__mptr = (void *)(ptr);                              \
             static_assert(__same_type(*(ptr), ((type *)0)->member) ||  \
                           __same_type(*(ptr), void),                   \
                           "pointer type mismatch in container_of()");  \
             ((type *)(__mptr - offsetof(type, member))); })

/* ② */  static inline long IS_ERR(const void *ptr)
         { return unlikely((unsigned long)ptr >= (unsigned long)-MAX_ERRNO); }

/* ③ */  struct foo {
             int len;
             char data[];           /* no size! */
         };

/* ④ */  void __iomem *base;
         u32 v = readl(base + 0x40);      /* not  *(u32*)(base + 0x40) */

/* ⑤ */  #define min(x, y) ({ typeof(x) _x = (x); typeof(y) _y = (y);   \
                              (void)(&_x == &_y); _x < _y ? _x : _y; })

/* ⑥ */  if (copy_from_user(&k, u, sizeof(k)))
             return -EFAULT;        /* not  k = *u; */
```

**Each of these is solving a problem C does not have a feature for:**

① **`container_of` is a downcast.** Kernel objects are related by *embedding*, not
inheritance — a `struct my_device` contains a `struct device`. Given a pointer to the inner
one, `container_of` recovers the outer. The `__same_type` assertion is the type-safety C
refuses to give you, bolted on with a GCC extension. §T.3.

② **`IS_ERR` packs an error into a pointer.** There are no exceptions and no out-parameters
in the common case, so functions returning pointers encode `-errno` in the top page of the
address space — addresses no valid kernel object can occupy. §T.5.

③ **A flexible array member** lets one allocation hold a header and a variable-length body,
saving a pointer chase and a second allocation. Modern kernels annotate it `__counted_by(len)`
so the compiler and UBSAN can bounds-check it. §T.6.

④ **`__iomem` is a `sparse` annotation** marking a pointer as device memory, which must not
be dereferenced — `readl` emits the right instruction with the right barriers and byte order.
A plain dereference compiles fine and is a bug. §T.2, Ch. 34.

⑤ **`min` is a statement expression** using `typeof` to evaluate each argument exactly once
(no double evaluation of `min(i++, j)`) and `(void)(&_x == &_y)` to force a compiler warning
if the types differ. Three hard macro problems, solved with extensions. §T.4.

⑥ **User pointers are not dereferenceable.** They may be invalid, may be unmapped, may race
with another thread's `munmap`, and on a 32-bit compat task they mean something different.
`copy_from_user` handles the fault safely and returns bytes-not-copied. §T.2, Ch. 24.

**The unifying observation, and it is the thesis of this chapter:**

> Kernel C is C **narrowed** (no UB the optimizer may exploit, no libc, no FPU, a small
> stack) and **extended** (statement expressions, `typeof`, `__attribute__`, sparse
> annotations, inline asm). Every extension exists because plain C could not express a
> safety property the kernel needs.

That framing matters for Part 5: **Rust's appeal is that it expresses these properties in the
language rather than in macros and a separate checker** (Ch. 78 §T.1). `container_of` becomes
a trait; `__user` becomes a type; `IS_ERR` becomes `Result`. Read this chapter knowing that
every workaround here has a Rust counterpart there — it makes both chapters land harder.

**Try it now:**

```bash
cd ~/src/linux
grep -n 'container_of' include/linux/container_of.h
grep -rn '__counted_by' include/linux/ | head -5
make C=1 M=drivers/foo/          # run sparse; watch it find __user/__iomem misuse
```

---

### T.1 The C abstract machine, and how the kernel amends it

ISO C does not define "what the CPU does". It defines an **abstract machine** and an
*as-if* rule: the implementation may do anything, provided observable behaviour matches the
abstract machine for programs with **defined** behaviour. Four categories of non-definition
exist, and conflating them is a common source of confusion:

| Category | Meaning | Example |
|---|---|---|
| **Implementation-defined** | must pick one and **document** it | `sizeof(int)`, signedness of `char`, `>>` on negatives |
| **Unspecified** | must pick one, need not document or be consistent | order of evaluation of function arguments |
| **Undefined (UB)** | *no requirements whatsoever* | signed overflow, OOB access, data race, null deref |
| **Locally undefined** | the construct is fine, a particular use isn't | `x << 32` for 32-bit `x` |

UB is not "it will crash" and not "it will do something machine-specific". It licenses the
compiler to assume it **cannot happen** and to optimize on that assumption — which is how
you get deleted NULL checks (Ch. 01 T.4) and vanished bounds checks.

**The kernel narrows the abstract machine deliberately.** Via compiler flags it converts a
set of UB constructs into *defined* ones, producing a dialect strictly stronger than ISO C:

```
Kernel C  =  ISO C (freestanding, gnu11/gnu17)
           + GCC/Clang extensions (statement expressions, typeof, __attribute__, asm goto,
             case ranges, zero-length/flexible arrays, labels-as-values, builtins)
           − strict aliasing            (-fno-strict-aliasing)
           − signed-overflow assumptions(-fno-strict-overflow / -fwrapv)
           − null-check deletion        (-fno-delete-null-pointer-checks)
           − invented stores            (-fno-allow-store-data-races)
           + its own memory model       (LKMM — Ch. 13, which C11 does NOT provide)
           + its own type annotations   (sparse: __user/__iomem/__percpu/__rcu/__bitwise)
           + its own "library"          (no libc; lib/, include/linux/)
```

Two things follow that you must internalize:

1. **You cannot reason about kernel C purely as a standards lawyer.** Code that is UB in ISO
   C may be perfectly defined in kernel C. `container_of` is the canonical example: it
   computes a pointer to an enclosing object from a pointer to a member, which the standard
   does not bless in general, but which is guaranteed by the kernel's flag set plus
   `offsetof`.
2. **Conversely, the remaining UB is more dangerous here**, because there is no exception
   handler, no `SIGSEGV` to catch, no process boundary — and an attacker on the other side.
   `CONFIG_UBSAN` exists to catch what the flags do not.

### T.2 Type theory in a language without types: the sparse annotations

C's type system cannot express the distinctions kernel code most needs. The kernel therefore
built a **second, optional type system** on top, checked by `sparse` (`make C=1`). This is
not lint — it is a real static analysis with real theory behind it.

**(a) Address spaces = an effect system.** `__user`, `__iomem`, `__percpu`, `__rcu` are
declared as distinct *address spaces*:

```c
# define __user     __attribute__((noderef, address_space(__user)))
# define __iomem    __attribute__((noderef, address_space(__iomem)))
# define __percpu   __attribute__((noderef, address_space(__percpu)))
# define __rcu      __attribute__((noderef, address_space(__rcu)))
```

`noderef` means **you may not dereference this pointer directly**; you must pass it through
a designated accessor (`copy_from_user`, `readl`, `this_cpu_ptr`, `rcu_dereference`).
Those accessors are the only legal "coercions" out of the address space.

In type-theory terms, this is a **graded/effect type system**: the annotation tracks *how a
value may be used*, and the accessors are the elimination forms. It catches an entire class
of catastrophic bugs (dereferencing a user pointer directly = an arbitrary-read/write
primitive for an attacker) **at compile time, in C**. That is a remarkable engineering
achievement and worth studying as a model: *when your language lacks the type you need,
build a checker that adds it.*

**(b) `__bitwise` = nominal typing / units.** By default C's typedefs are *transparent*
(structural): `__le32` and `u32` are interchangeable. `__bitwise` makes them **nominal** —
distinct types that do not implicitly convert:

```c
typedef __u32 __bitwise __le32;
typedef __u32 __bitwise __be32;
typedef unsigned int __bitwise gfp_t;
typedef unsigned int __bitwise slab_flags_t;
typedef unsigned int __bitwise fmode_t;
```

So assigning a `__be32` to a `u32` without `be32_to_cpu()` is a type error. This is exactly
**dimensional analysis / units-of-measure typing** (as in F# units or Boost.Units), applied
to endianness and flag namespaces. It catches endianness bugs that would otherwise only
appear on a big-endian machine you don't own (Ch. 02 T.3).

Run it. Always:
```bash
make C=1 M=$PWD/mymod          # check changed files
make C=2 M=$PWD/mymod          # check all files
make W=1 ...                   # extra compiler warnings
make CHECK='smatch -p=kernel' C=1 ...   # smatch: flow-sensitive analysis
```

### T.3 `container_of`: structural subtyping in C

Object orientation in the kernel is built on one macro:

```c
#define container_of(ptr, type, member) ({                       \
	void *__mptr = (void *)(ptr);                            \
	static_assert(__same_type(*(ptr), ((type *)0)->member) || \
		      __same_type(*(ptr), void),                 \
		      "pointer type mismatch in container_of()");\
	((type *)(__mptr - offsetof(type, member))); })
```

The theory: if `B` is embedded in `D`, then a `B *` pointing into a `D` can be converted back
to `D *` by subtracting a compile-time-known offset. That is **downcasting in a structural
subtyping relation**, with the "subtyping" established by physical embedding rather than by
declaration.

Compare the alternatives:

| Approach | Memory | Type safety | Lifetime |
|---|---|---|---|
| **Embedding + `container_of`** (kernel) | zero extra | checked by `__same_type` assert | shared with container |
| `void *private` pointer | one pointer | none | separate allocation, separate lifetime |
| C++ inheritance | vtable pointer | full | language-managed |

Embedding wins on all three axes that matter in a kernel: no extra allocation (so no extra
failure path), no extra indirection (so no extra cache miss), and one lifetime instead of two.
This is why `struct device`, `struct kobject`, `struct list_head`, `struct rb_node`,
`struct work_struct`, `struct hrtimer`, `struct file_operations` users, and essentially
every callback in the kernel use it.

A structure can embed **many** bases, giving multiple inheritance for free:

```c
struct my_dev {
	struct device      dev;      /* is-a device */
	struct cdev        cdev;     /* is-a char device */
	struct list_head   node;     /* is-in a list */
	struct work_struct work;     /* is-a work item */
	struct kref        ref;      /* is-refcounted */
};
```
Each subsystem holds a pointer to *its own* member and recovers `struct my_dev *` with
`container_of`. **Reading a struct definition as a list of "is-a" relations is the single
most useful reading skill for kernel code** (Ch. 07 T.5).

### T.4 Ops tables: vtables, and the cost of an indirect call

```c
struct file_operations {
	struct module *owner;
	ssize_t (*read)(struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *);
	long (*unlocked_ioctl)(struct file *, unsigned int, unsigned long);
	...
};
```
This is a **vtable**, made explicit. Differences from C++'s implicit vtable, all deliberate:

- The table is `const` and shared (one per *type*, not per object) — so it lives in `.rodata`
  and can be write-protected.
- The object holds a pointer to it explicitly (`file->f_op`), so you can see the dispatch.
- **Optional methods are `NULL`**, and callers check. C++ would need a pure-virtual or a
  default implementation. The kernel's convention (`if (f_op->read) ...`) makes partial
  interfaces natural — which matters when a subsystem has 40 optional callbacks.

**The cost:** an indirect call is ~1–2 ns when predicted, but since Spectre v2 it may go
through a **retpoline** (~10–30 cycles) or IBRS/eIBRS. And with `CONFIG_CFI_CLANG` every
indirect call carries a type-hash check (Ch. 01 T.6). So indirect dispatch is no longer
free, and the kernel has responded with:

- **`static_call()`** — a runtime-patched *direct* call. When there is effectively one
  implementation (e.g. the active scheduler class, the chosen crypto implementation), the
  call site is rewritten to a direct `call` and the retpoline disappears.
  ```c
  DEFINE_STATIC_CALL(my_op, default_impl);
  static_call(my_op)(args);              /* patched to a direct call */
  static_call_update(my_op, other_impl); /* re-patch at runtime */
  ```
- **`static_branch`/`static_key`** — runtime-patched *branches* (a `nop` or a `jmp`), for
  features that are off almost always (tracepoints, debug checks).

Recognizing when a hot indirect call should become a `static_call` is a genuine performance
skill (Ch. 29).

### T.5 The preprocessor as an unhygienic macro system

C macros are **textual substitution without hygiene**: no scoping, no types, no guarantee of
single evaluation. The kernel uses them heavily anyway, because they're the only way to get
generic code, and it has standardized the defensive idioms:

**Statement expressions `({ ... })`** — a GCC extension giving a block a value, which is what
makes safe macros possible:

```c
#define min(x, y) ({                        \
	typeof(x) _min1 = (x);              \
	typeof(y) _min2 = (y);              \
	(void) (&_min1 == &_min2);          /* type-compatibility check */ \
	_min1 < _min2 ? _min1 : _min2; })
```
Three techniques in four lines: (1) evaluate each argument **exactly once** into a temporary,
(2) `typeof` to get the argument's type without naming it, (3) the `&_min1 == &_min2`
comparison produces a warning if the types differ — an assertion encoded as a side-effect-free
expression. (Modern `min()`/`max()` in `include/linux/minmax.h` are considerably more
elaborate, handling constant folding and signedness; read them.)

**The naming convention** (leading underscores on temporaries) is the manual substitute for
hygiene: it reduces, but does not eliminate, capture. `min(a, _min1)` still breaks. This is a
real limitation, and it's one of the arguments for Rust's hygienic macros (Part 5).

**`do { ... } while (0)`** — wraps multi-statement macros so they behave syntactically like a
single statement, surviving `if (x) MACRO(); else ...`.

Other essential compiler-intrinsic idioms:

```c
__builtin_constant_p(x)        /* is x a compile-time constant? enables dual implementations */
__builtin_types_compatible_p(a, b)
__builtin_expect(x, 1)         /* → likely()/unlikely() */
__builtin_unreachable()        /* → unreachable() */
BUILD_BUG_ON(cond)             /* compile-time assertion via a negative-size array */
static_assert(cond, msg)       /* C11; preferred in new code */
BUILD_BUG_ON_ZERO(cond)        /* usable inside expressions (see ARRAY_SIZE) */
ARRAY_SIZE(arr)                /* sizeof(a)/sizeof((a)[0]) + a check that a is a real array */
```

`ARRAY_SIZE` is worth dissecting because it shows the technique:
```c
#define ARRAY_SIZE(arr) (sizeof(arr) / sizeof((arr)[0]) + __must_be_array(arr))
#define __must_be_array(a) BUILD_BUG_ON_ZERO(__same_type((a), &(a)[0]))
```
`__must_be_array` evaluates to `0` but **fails to compile** if `arr` is a pointer rather than
an array — catching the classic `sizeof(ptr)/sizeof(ptr[0])` bug at compile time.

### T.6 Integers: the traps that actually bite

C's integer rules are the single most common source of silent kernel bugs after concurrency.

**(a) Integer promotion.** Anything narrower than `int` is promoted to `int` in expressions:
```c
u8 a = 200, b = 100;
if (a + b > 255)        /* TRUE: a+b is computed as int = 300, not u8 = 44 */
```
**(b) Usual arithmetic conversions — signed becomes unsigned.** This is the deadly one:
```c
int len = -1;
if (len < sizeof(buf))       /* sizeof is size_t (unsigned) → len converts to a HUGE value
                                → condition is FALSE → the bounds check is bypassed */
```
This exact pattern has produced multiple kernel CVEs. Rule: **never compare a signed value
with `sizeof` or any `size_t`.** Check `len < 0` separately first, or use signed types
throughout.

**(c) Overflow in size computations.** `kmalloc(n * size, ...)` overflows to a small
allocation, and the subsequent write is a heap overflow. The kernel provides checked helpers,
and **reviewers will require them**:
```c
kmalloc_array(n, size, GFP_KERNEL);
kcalloc(n, size, GFP_KERNEL);
kvmalloc_array(n, size, GFP_KERNEL);
struct_size(p, member, count)          /* sizeof(*p) + count*sizeof(member[0]), saturating */
size_mul(a, b), size_add(a, b), size_sub(a, b)   /* saturate at SIZE_MAX */
check_add_overflow(a, b, &res)          /* → true on overflow */
check_mul_overflow(a, b, &res)
array_size(a, b), array3_size(a, b, c)
```
**(d) Shifts.** `x << n` is UB if `n >= width(x)` or if `x` is signed and the result
overflows. `BIT(n)`, `BIT_ULL(n)`, `GENMASK(h, l)`, `GENMASK_ULL(h, l)` exist to make intent
explicit and bounds-checkable.

**(e) Division.** No floating point, and 64-bit division on 32-bit architectures requires
helpers or you get a link error against `__udivdi3`:
```c
div_u64(dividend, divisor); div64_u64(); div_s64(); do_div(n, base);   /* modifies n! */
DIV_ROUND_UP(n, d); DIV_ROUND_CLOSEST(n, d); roundup(x, y); round_up(x, y_pow2);
```

### T.7 Alignment, packing, and layout

- **Natural alignment** is required (or at least much faster) on most non-x86 architectures.
  A misaligned access may fault (SPARC), be fixed up slowly in software (older ARM), or just
  be slow.
- `__packed` removes padding — and thereby tells the compiler *every* member may be
  misaligned, so on strict-alignment targets it generates byte-at-a-time accesses for the
  whole struct. Use it only for genuine on-disk/on-wire layouts, and prefer
  `get_unaligned_le32()` style accessors over packing whole structs.
- `__aligned(n)`, `____cacheline_aligned`, `____cacheline_aligned_in_smp` control the other
  direction — see Ch. 16 T.2 for why.
- **Flexible array members** are the modern form of the old `[0]`/`[1]` hacks:
  ```c
  struct msg {
	  u32 len;
	  u8  data[];            /* flexible array member: C99, and what the kernel requires */
  };
  ```
  With `__counted_by(len)` (Clang 18+/GCC 15+), the compiler and `UBSAN`/`FORTIFY` can
  bounds-check accesses to `data` at runtime. There is an ongoing tree-wide conversion from
  `[0]` to `[]` and from `[]` to `__counted_by` — an excellent source of first patches.
- `struct_group()` lets you name a contiguous subset of members so `memcpy` over a range
  doesn't trip `FORTIFY_SOURCE`'s field-bounds checking.

Verify layout, don't assume it:
```bash
pahole -C my_struct my_module.ko          # offsets, holes, cachelines
pahole --hex -C sk_buff vmlinux
pahole -C task_struct --reorganize vmlinux   # what a better layout would look like
```

### T.8 `const`, `volatile`, and what they do *not* mean

- **`const` in C is a promise, not an enforcement**, and it does not propagate through
  pointers the way people expect. It is still valuable as documentation and it enables
  placement in `.rodata` (which the kernel then marks read-only via
  `CONFIG_STRICT_KERNEL_RWX`). All ops tables should be `const`.
- **`volatile` is almost always wrong in kernel code.** Read
  `Documentation/process/volatile-considered-harmful.rst`. It prevents compiler optimization
  but provides **no CPU ordering and no atomicity**, so it neither makes a variable
  thread-safe nor substitutes for a barrier. The legitimate uses are: jiffies
  (`extern unsigned long volatile jiffies`), inline-asm constraints, and MMIO accessors —
  all of which are already wrapped for you. If you are reaching for `volatile`, you want
  `READ_ONCE`/`WRITE_ONCE` (Ch. 13 T.5) or a proper lock.

### T.9 Section attributes: the linker as an API

The kernel uses ELF sections as a *programming mechanism*, not just a build detail:

```c
__init          /* .init.text — freed after boot */
__exit          /* discarded entirely when built-in */
__initdata, __initconst, __exitdata
__read_mostly   /* .data..read_mostly — keep away from hot written data (Ch.16 T.2) */
__ro_after_init /* writable during init, then mapped read-only — great for function ptr tables */
__percpu        /* .data..percpu */
__weak          /* overridable definition */
__section(".foo") /* build a table the linker collects */
```

The **linker-collected table** pattern is used everywhere and is worth recognizing:
initcalls, tracepoints, `__ksymtab`, exception tables (`__ex_table`), jump labels, BPF
`BTF_ID` sets, and `KUnit` test suites all work by emitting entries into a named section and
having the linker script provide `__start_foo`/`__stop_foo` symbols to iterate. It is
**compile-time registration with zero runtime cost** — a genuinely elegant technique, and one
you can use in your own subsystems.

`modpost` enforces the rules (a non-`__init` function may not call an `__init` one, because
the latter's text is gone) and emits **section mismatch** warnings when you violate them
(Ch. 03).

---

## 1. Concept: this is not the C you learned

The kernel is compiled with, roughly:

```
-std=gnu11 -ffreestanding -nostdinc -fno-strict-aliasing -fno-common
-fno-PIE -fno-stack-protector(or with) -mno-red-zone -mcmodel=kernel
-fno-delete-null-pointer-checks -fno-allow-store-data-races
-Wall -Wundef -Werror=implicit-function-declaration -Wno-trigraphs
-falign-functions=16 -fconserve-stack
```

Consequences:

| Fact | Consequence |
|---|---|
| **No libc** | No `printf`, `malloc`, `strdup` semantics you know. Kernel has its own. |
| **No floating point** | FPU/SIMD state isn't saved on kernel entry. Using FP corrupts userspace. |
| **Tiny stacks** | 8 KiB (x86-32/arm32) or **16 KiB** (x86-64, arm64). Recursion and big locals are forbidden. |
| **`-fno-strict-aliasing`** | Type-punning through pointers is legal here (unlike ISO C). |
| **GNU extensions everywhere** | statement expressions, typeof, `__attribute__`, case ranges. |
| **No exceptions, no RTTI, no C++** | (Rust is the answer to "we want more than C") |
| **Preprocessor-heavy** | Macro metaprogramming is idiomatic, not a smell. |

---

## 2. Internals

### 2.1 Types

```c
#include <linux/types.h>

/* Fixed-width — ALWAYS use these, never `long` for ABI */
u8 u16 u32 u64        s8 s16 s32 s64        /* kernel-internal */
__u8 __u16 __u32 __u64 __s8 ...             /* UAPI-safe (no linux/types.h needed) */
__le16 __le32 __le64  __be16 __be32 __be64  /* endian-annotated; sparse-checked */
__sum16 __wsum                              /* checksums */

size_t ssize_t ptrdiff_t
loff_t        /* 64-bit file offset */
off_t         /* may be 32-bit; avoid in kernel */
phys_addr_t   /* physical address (may be > sizeof(void*) with PAE) */
dma_addr_t    /* bus/IOVA address — NOT a CPU address */
resource_size_t
pgoff_t       /* page-cache index */
sector_t      /* 512-byte sector index (u64) */
gfp_t         /* allocation flags — __bitwise, sparse-checked */
pid_t uid_t gid_t   /* and kuid_t/kgid_t for namespaced ids */
atomic_t atomic64_t atomic_long_t refcount_t
bool          /* <linux/types.h> pulls stdbool */
```

**Rules:**
- Never use `int` where a size can exceed 2 GiB.
- Never use `unsigned long` in UAPI structs (differs between 32/64-bit).
- Use `__le32`/`__be32` for on-disk and on-wire fields. Access only via
  `le32_to_cpu()`/`cpu_to_le32()`. `make C=1` will catch violations.
- `dma_addr_t` != `phys_addr_t` != `virt` address. Confusing them is a top-5 driver bug.

Endian helpers:
```c
le32_to_cpu(x)  cpu_to_le32(x)  be16_to_cpu(x)  cpu_to_be64(x)
le32_to_cpup(p) get_unaligned_le32(p)  put_unaligned_be16(v, p)
le32_add_cpu(&field, 1)
```

### 2.2 The essential macros

```c
/* include/linux/kernel.h, container_of.h, array_size.h, minmax.h */

container_of(ptr, type, member)      /* ★ member pointer → container pointer */
container_of_const(ptr, type, member)
offsetof(type, member)
sizeof_field(type, member)
struct_size(p, member, n)            /* overflow-safe sizeof for flex arrays */
flex_array_size(p, member, n)

ARRAY_SIZE(arr)                      /* + compile error if arr is a pointer */
min(a,b) max(a,b) clamp(v,lo,hi)     /* type-checked */
min_t(type,a,b) max_t(type,a,b) clamp_t(...)
swap(a,b)
abs(x)

ALIGN(x, a)  ALIGN_DOWN(x, a)  IS_ALIGNED(x, a)  PTR_ALIGN(p, a)
round_up(x, y)  round_down(x, y)     /* y need not be a power of 2 */
DIV_ROUND_UP(n, d)  DIV_ROUND_CLOSEST(n, d)
roundup_pow_of_two(n)  rounddown_pow_of_two(n)  is_power_of_2(n)
ilog2(n)  order_base_2(n)  fls(x) ffs(x) __ffs(x) hweight32(x)

BIT(n)  BIT_ULL(n)  GENMASK(hi, lo)  GENMASK_ULL(hi, lo)
FIELD_PREP(mask, val)  FIELD_GET(mask, reg)   /* ★ <linux/bitfield.h> — use these! */

BUILD_BUG_ON(cond)                   /* compile-time assert */
BUILD_BUG_ON_MSG(cond, "msg")
static_assert(cond, "msg")
BUILD_BUG_ON_ZERO(e)

likely(cond)  unlikely(cond)         /* branch hints — use sparingly, only hot paths */
barrier()                            /* compiler barrier */
READ_ONCE(x)  WRITE_ONCE(x, v)       /* ★ prevent compiler tearing/refetch */

__is_constexpr(x)
typeof(x)  __auto_type
```

`FIELD_PREP`/`FIELD_GET` deserve emphasis — they replace hand-rolled shift/mask bugs:
```c
#define REG_CTRL_MODE   GENMASK(5, 3)
#define REG_CTRL_ENABLE BIT(0)

val = FIELD_PREP(REG_CTRL_MODE, 3) | REG_CTRL_ENABLE;
mode = FIELD_GET(REG_CTRL_MODE, readl(base + REG_CTRL));
```

### 2.3 GCC extensions you will meet

```c
/* Statement expressions */
#define max(a, b) ({ typeof(a) _a = (a); typeof(b) _b = (b); _a > _b ? _a : _b; })

/* typeof */
typeof(*ptr) tmp;

/* Designated initializers — pervasive for ops tables */
static const struct file_operations fops = {
	.owner   = THIS_MODULE,
	.read    = my_read,
	.llseek  = no_llseek,
};

/* Ranged cases */
switch (c) { case 'a' ... 'z': ... }

/* Labels as values / computed goto (BPF interpreter, some fast paths) */
static void *jt[] = { &&op_add, &&op_sub };
goto *jt[op];

/* Zero-length / flexible array members */
struct msg { u32 len; u8 data[]; };          /* C99 flexible array — correct */
struct msg { u32 len; u8 data[0]; };         /* old GCC style — being removed */
struct msg { u32 len; u8 data[] __counted_by(len); };  /* 6.6+ : bounds-checkable */

/* __attribute__ */
__packed __aligned(64) __printf(2,3) __malloc __alloc_size(1)
__pure __const __noreturn __cold __hot __weak __alias("x")
__section(".init.text") __used __visible __always_inline __noclone
__no_sanitize_address __nocfi
__cleanup(func)                              /* basis of cleanup.h */

/* Case fallthrough must be explicit (kernel uses -Wimplicit-fallthrough) */
switch (x) {
case 1:
	do_a();
	fallthrough;
case 2:
	do_b();
	break;
}
```

### 2.4 `READ_ONCE`/`WRITE_ONCE` — non-optional knowledge

The compiler may:
- **tear** a store (write a 64-bit value in two 32-bit halves),
- **invent** loads (re-read a variable it already loaded),
- **fuse** loads (cache a value in a register across a loop),
- **reorder** accesses.

For any variable accessed concurrently *without* a lock, you must use:
```c
	x = READ_ONCE(shared);
	WRITE_ONCE(shared, y);
```
These expand to `volatile` accesses plus a compiler barrier, and are only valid for
machine-word-sized, naturally-aligned data.

Classic bug:
```c
	while (!flag)             /* compiler hoists the load out of the loop → hangs */
		cpu_relax();
	while (!READ_ONCE(flag))  /* correct */
		cpu_relax();
```

This is *compiler* ordering only. CPU ordering needs barriers (Chapter 13).

### 2.5 Strings, numbers, and the missing libc

```c
/* Available (lib/string.c) */
strlen strnlen strcpy strncpy strscpy strlcpy(deprecated) strcat strncat
strcmp strncmp strcasecmp strchr strrchr strstr strsep strim skip_spaces
memcpy memmove memset memcmp memchr memzero_explicit
kstrdup kstrndup kmemdup kstrdup_const kasprintf kvasprintf

/* ★ Use strscpy(), not strcpy/strncpy/strlcpy */
ssize_t strscpy(char *dst, const char *src, size_t count);
/* returns -E2BIG on truncation, always NUL-terminates */
strscpy_pad(dst, src, len);     /* also zero-fills the remainder */

/* Number parsing — NEVER use atoi/simple_strtoul */
int kstrtoint(const char *s, unsigned int base, int *res);
int kstrtoul(const char *s, unsigned int base, unsigned long *res);
int kstrtou32 / kstrtos64 / kstrtobool / kstrtoull ...
int kstrtoint_from_user(const char __user *s, size_t count, unsigned base, int *res);

/* Formatting */
int snprintf(char *buf, size_t size, const char *fmt, ...);
int scnprintf(...);        /* ★ returns chars ACTUALLY written — use in loops */
char *kasprintf(gfp_t gfp, const char *fmt, ...);   /* allocates */
int sysfs_emit(char *buf, const char *fmt, ...);    /* ★ for sysfs show() */
int sysfs_emit_at(char *buf, int at, const char *fmt, ...);
```

**Why `scnprintf` over `snprintf`:** `snprintf` returns what *would* have been written,
so `p += snprintf(p, end - p, ...)` can advance past the buffer. `scnprintf` cannot.

### 2.6 Userspace memory access

```c
copy_from_user(kdst, usrc, n);    /* returns BYTES NOT COPIED, not an errno! */
copy_to_user(udst, ksrc, n);
get_user(val, uptr);              /* returns 0 or -EFAULT */
put_user(val, uptr);
clear_user(uptr, n);
strncpy_from_user(kdst, usrc, n);
strnlen_user(usrc, n);
access_ok(uptr, size);            /* range check only — NOT a permission check */

/* Batched (avoid repeated SMAP/PAN toggling) */
if (!user_access_begin(uptr, size)) return -EFAULT;
unsafe_get_user(v, uptr, efault);
user_access_end();
```

Critical points:
- Return value of `copy_*_user` is a **count**, not an error:
  `if (copy_from_user(k, u, n)) return -EFAULT;`
- These can **sleep** (page faults). Never in atomic context.
- Never dereference a `__user` pointer directly. `make C=1` enforces this.
- `access_ok()` alone is insufficient — TOCTOU. Always use the copy helpers.
- Never read a user value twice ("double fetch") — copy once into kernel memory,
  then validate that copy. This is a whole class of CVEs.

### 2.7 Stack discipline

```c
/* 16 KiB total on x86-64 (THREAD_SIZE), shared with IRQ nesting on some arches */
static int f(void)
{
	char buf[4096];   /* ✗ 25% of your stack */
	struct big s;     /* ✗ check sizeof */
}
```
Rules:
- Keep frames under ~128 bytes; the build warns at 1024 (`-Wframe-larger-than`).
- No recursion (bounded recursion only with explicit depth limits).
- No VLAs — they're banned tree-wide (`-Wvla`).
- Big buffers → `kmalloc`/`kvmalloc`.
- `noinline_for_stack` to push a big frame into a leaf function.

Check:
```bash
make O=b W=1 drivers/foo/bar.o 2>&1 | grep frame
objdump -d bar.o | grep -A2 'sub .*,%rsp'
scripts/checkstack.pl < vmlinux.od   # or: make O=b checkstack
```

### 2.8 No floating point / SIMD

```c
/* WRONG in kernel code */
double x = 1.5;
```
Because kernel entry does not save FPU state. If you *must* (crypto, RAID parity, some
codecs):
```c
#include <asm/fpu/api.h>
kernel_fpu_begin();
/* SIMD here */
kernel_fpu_end();
```
arm64: `kernel_neon_begin()` / `kernel_neon_end()`. These disable preemption — keep the
region short and chunked.

### 2.9 Integer overflow and safe arithmetic

```c
#include <linux/overflow.h>

if (check_add_overflow(a, b, &sum)) return -EOVERFLOW;
check_sub_overflow / check_mul_overflow / check_shl_overflow
size_mul(a, b)  size_add(a, b)  size_sub(a, b)    /* saturate to SIZE_MAX */
array_size(n, size)  array3_size(a, b, c)  struct_size(p, member, n)

/* Allocation helpers that do this for you */
kmalloc_array(n, size, gfp);
kcalloc(n, size, gfp);
kvmalloc_array(n, size, gfp);
devm_kmalloc_array(dev, n, size, gfp);
kmalloc(struct_size(obj, items, n), gfp);
```
`kmalloc(n * size, gfp)` is a **security bug**. Reviewers will reject it.

### 2.10 Modern cleanup: `__free`, `guard`, `scoped_guard`

`include/linux/cleanup.h` (5.19+, widely used from 6.4+):

```c
#include <linux/cleanup.h>

static int f(void)
{
	struct foo *p __free(kfree) = kmalloc(sizeof(*p), GFP_KERNEL);
	if (!p)
		return -ENOMEM;

	guard(mutex)(&my_mutex);        /* unlocked at end of scope */

	if (bad)
		return -EINVAL;          /* p freed, mutex unlocked automatically */

	scoped_guard(spinlock_irqsave, &dev->lock) {
		dev->state = READY;
	}

	return 0;
}

/* Transfer ownership out of a __free variable */
	return_ptr(p);      /* == no_free_ptr(p) then return */
```

Defining your own:
```c
DEFINE_FREE(my_put, struct my *, if (_T) my_put(_T))
DEFINE_GUARD(my_lock, struct my *, my_lock(_T), my_unlock(_T))
DEFINE_CLASS(...)
```

This is genuinely changing how new kernel C is written. Learn it now.

---

## 3. Coding style — non-negotiable for upstream

`Documentation/process/coding-style.rst`. The essentials:

```c
/* Tabs, 8 wide. Braces K&R (opening brace on same line, except functions). */
if (x) {
	foo();
} else {
	bar();
}

int function(int a, int b)
{
	/* function opening brace on its OWN line */
}

/* No braces for single statements */
if (x)
	foo();

/* But if ANY branch needs braces, all do */
if (x) {
	foo();
	bar();
} else {
	baz();
}

/* 80 columns is a strong guideline (100 tolerated since 5.7 for readability) */

/* Naming: lower_snake_case. Short local names, descriptive global names. */
/* No Hungarian notation. No CamelCase. No typedef for structs. */

/* Comments: /* */ for multi-line, prefer explaining WHY */
/*
 * Multi-line kernel comment
 * style looks like this.
 */

/* One declaration per line; declarations at top of block (relaxed in newer code) */

/* Labels at column 0 */
out_free:
	kfree(p);
```

Hard rules that trip people up:
- **No typedefs for structs.** `struct foo *f`, never `foo_t *f`.
  (Exceptions: opaque types like `atomic_t`, `gfp_t`.)
- **No `inline` in .c files** unless measured. Let GCC decide.
- **Functions should fit on one or two screens** and do one thing.
- `goto` for error unwind is **good style** here.
- SPDX line must be line 1: `// SPDX-License-Identifier: GPL-2.0` (`/* */` for headers).

Check before every patch:
```bash
scripts/checkpatch.pl --strict --codespell -g HEAD-1
scripts/checkpatch.pl --strict -f drivers/foo/bar.c
clang-format -i --style=file drivers/foo/bar.c   # .clang-format is in the tree
```

### kernel-doc comments
```c
/**
 * my_function() - Short one-line description.
 * @arg1: Description of first argument.
 * @arg2: Description of second argument.
 *
 * Longer description. Explain context requirements:
 *
 * Context: Process context. Takes and releases &my_mutex. May sleep.
 * Return: 0 on success, negative errno on failure.
 */
int my_function(int arg1, char *arg2)
```
Validate: `scripts/kernel-doc -v -none drivers/foo/bar.c` or `make htmldocs`.

---

## 4. Practice

### Lab 8.1 — Bitfield hygiene
Take a fictional register:
```
bits 31..24  VERSION (ro)
bits 15..8   TIMEOUT
bit  4       IRQ_EN
bits 3..0    MODE
```
Write the `GENMASK`/`BIT` definitions and a `set_mode()` using `FIELD_PREP`/`FIELD_GET`.
Then write the buggy shift-and-mask version and diff readability.

### Lab 8.2 — `container_of` from scratch
Implement your own `container_of` with `offsetof` and `typeof`. Explain the
`__same_type` static assertion in the real one (`include/linux/container_of.h`).

### Lab 8.3 — Break the compiler's assumptions
Write a loop polling a flag without `READ_ONCE`. Compile with `-O2`, dump assembly,
show the hoisted load. Add `READ_ONCE`, re-dump.

### Lab 8.4 — Convert to `cleanup.h`
Take the `hello.c` from Chapter 05 and rewrite `hello_init` using `__free()` and
`guard()`. Count the lines removed.

### Lab 8.5 — checkpatch gauntlet
Write a 100-line driver skeleton. Run `checkpatch.pl --strict`. Fix every single
warning. Read the checkpatch source for any rule you don't understand.

### Lab 8.6 — Prove the sparse address-space system works (T.2)

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>
#include <linux/uaccess.h>
#include <linux/io.h>
#include <linux/percpu.h>

static DEFINE_PER_CPU(int, counter);

static long bad_ioctl(unsigned long arg)
{
	int __user *uptr = (int __user *)arg;

	return *uptr;                    /* sparse: dereference of noderef expression */
}

static long good_ioctl(unsigned long arg)
{
	int __user *uptr = (int __user *)arg;
	int v;

	if (get_user(v, uptr))
		return -EFAULT;
	return v;
}

static u32 bad_mmio(void __iomem *base)
{
	return *(u32 *)base;             /* sparse: dereference of noderef */
}

static u32 good_mmio(void __iomem *base)
{
	return readl(base);
}

static int bad_percpu(void)
{
	return *(&counter);              /* sparse: percpu address space mismatch */
}

static int good_percpu(void)
{
	return this_cpu_read(counter);
}
```
```bash
sudo apt install -y sparse smatch
make C=2 M=$PWD/sparsedemo modules
# Expect: "warning: dereference of noderef expression" ×3
```
**Do this once and the annotations stop looking like noise.** Then find a real driver and
count how many `__iomem`/`__user` annotations it has.

### Lab 8.7 — Endianness typing with `__bitwise` (T.2b)

```c
struct fs_super {
	__le32 magic;
	__le64 blocks;
	__be16 net_port;
};

static int check(struct fs_super *s)
{
	if (s->magic != 0xDEADBEEF)              /* sparse: restricted __le32 degrades */
		return -EINVAL;
	if (le32_to_cpu(s->magic) != 0xDEADBEEF) /* correct */
		return -EINVAL;
	return 0;
}
```
Build with `C=2`, read the warnings, then grep real code:
```bash
git grep -c '__le32\|__be32' fs/ext4/ | head
git grep -n 'cpu_to_le\|le32_to_cpu' fs/ext4/super.c | head
```

### Lab 8.8 — The integer traps, demonstrated (T.6)

```c
static int __init inttrap_init(void)
{
	u8  a = 200, b = 100;
	int len = -1;
	char buf[64];
	unsigned int n = 0x10000;
	size_t sz;

	pr_info("promotion: (u8)200 + (u8)100 = %d (not %u)\n", a + b, (u8)(a + b));

	/* The classic bounds-check bypass */
	if (len < sizeof(buf))
		pr_info("BUG: -1 < sizeof(buf) evaluated TRUE (signed->unsigned)\n");
	else
		pr_info("ok\n");

	/* Multiplication overflow */
	if (check_mul_overflow(n, n, &sz))
		pr_info("n*n overflows size_t on this arch\n");
	else
		pr_info("n*n = %zu\n", sz);

	/* What kmalloc(n*n) would have done: */
	pr_info("silent wrap: n*n as u32 = %u\n", (unsigned int)(n * n));

	/* Shift UB */
	pr_info("BIT(3)=%lu GENMASK(7,4)=0x%lx\n", BIT(3), GENMASK(7, 4));
	return 0;
}
```
Build with `CONFIG_UBSAN=y` and add `pr_info("%d\n", 1 << 40);` to see UBSAN fire.
Then `git grep -n 'kmalloc(.*\* ' drivers/ | head` and assess each for overflow risk.

### Lab 8.9 — Dissect `container_of` and build your own object system (T.3)

```c
struct base { int id; void (*speak)(struct base *); };

struct dog  { struct base b; int barks; };
struct cat  { struct base b; int lives; };

static void dog_speak(struct base *b)
{
	struct dog *d = container_of(b, struct dog, b);

	pr_info("dog %d: woof x%d\n", b->id, ++d->barks);
}
static void cat_speak(struct base *b)
{
	struct cat *c = container_of(b, struct cat, b);

	pr_info("cat %d: meow (%d lives)\n", b->id, c->lives);
}

static int __init oo_init(void)
{
	struct dog d = { .b = { .id = 1, .speak = dog_speak } };
	struct cat c = { .b = { .id = 2, .speak = cat_speak }, .lives = 9 };
	struct base *zoo[] = { &d.b, &c.b };
	int i;

	for (i = 0; i < ARRAY_SIZE(zoo); i++)
		zoo[i]->speak(zoo[i]);       /* virtual dispatch, in C */
	return 0;
}
```
Then break it deliberately: pass the wrong member name to `container_of` and read the
`__same_type` static assert. Finally, preprocess it:
```bash
make M=$PWD/oo oo.i && sed -n '/oo_init/,/^}/p' oo/oo.i
```

### Lab 8.10 — `static_call` vs indirect call, measured (T.4)

```c
#include <linux/static_call.h>

static int impl_a(int x) { return x + 1; }
static int impl_b(int x) { return x * 2; }

DEFINE_STATIC_CALL(myop, impl_a);

static int (*indirect)(int) = impl_a;

static int __init sc_init(void)
{
	u64 t0, t1, t2;
	int i, acc = 0;

	t0 = ktime_get_ns();
	for (i = 0; i < 10000000; i++) acc += indirect(i);
	t1 = ktime_get_ns();
	for (i = 0; i < 10000000; i++) acc += static_call(myop)(i);
	t2 = ktime_get_ns();

	pr_info("indirect: %llu ns/call, static_call: %llu ns/call (acc=%d)\n",
		(t1 - t0) / 10000000, (t2 - t1) / 10000000, acc);

	static_call_update(myop, impl_b);   /* re-patch at runtime */
	pr_info("after update: %d\n", static_call(myop)(10));
	return 0;
}
```
Run with and without `spectre_v2=retpoline` on the kernel command line. The delta is the
retpoline tax, and it is why `static_call` exists.
```bash
cat /sys/devices/system/cpu/vulnerabilities/spectre_v2
git grep -l DEFINE_STATIC_CALL kernel/ | head       # real users
```

### Lab 8.11 — Section attributes and the linker-table pattern (T.9)

```c
/* Build a linker-collected table of your own. */
struct my_entry { const char *name; int (*fn)(void); };

#define MY_REGISTER(nm, f)                                        \
	static const struct my_entry __my_##nm                    \
	__used __section(".my_table") = { .name = #nm, .fn = f }

extern const struct my_entry __start_my_table[], __stop_my_table[];

static int one(void) { return 1; }
static int two(void) { return 2; }
MY_REGISTER(one, one);
MY_REGISTER(two, two);

static int __init tbl_init(void)
{
	const struct my_entry *e;

	for (e = __start_my_table; e < __stop_my_table; e++)
		pr_info("%s -> %d\n", e->name, e->fn());
	return 0;
}
```
(In a module you need the section in your link; in-tree you'd add it to the linker script.)
Then find the real instances:
```bash
grep -n '__start_\|__stop_' include/asm-generic/vmlinux.lds.h | head -30
grep -rn '__section("__ksymtab' include/linux/export.h
grep -rn '__section("_ftrace_events")' include/linux/trace_events.h
```

### Lab 8.12 — `__ro_after_init` and `__init` memory recovery

```c
static int config_value __ro_after_init;
static int __initdata boot_only_table[256];

static int __init ro_init(void)
{
	config_value = 42;          /* legal: still writable during init */
	return 0;
}
/* Any later write to config_value faults with STRICT_KERNEL_RWX. Try it from a
 * debugfs write handler and read the resulting Oops. */
```
```bash
dmesg | grep 'Freeing unused kernel'       # how much __init memory came back
grep -rn '__ro_after_init' include/linux/init.h
git grep -c '__ro_after_init' -- '*.c' | head
```

---

## 5. Mastery drills

1. Explain why the kernel uses `-fno-strict-aliasing` and give a concrete example of
   kernel code that would be UB without it.
2. Find five uses of `__bitwise` in the tree. What does sparse actually check?
3. Explain the difference between `__attribute__((pure))` and `__attribute__((const))`
   and find one example of each in the tree.
4. Why does `copy_from_user()` return a byte count instead of an errno? (Historical +
   partial-copy semantics. Find a caller that uses the partial count meaningfully.)
5. `DEFINE_FREE(kfree, void *, if (_T) kfree(_T))` — explain what `_T` is and how the
   macro expands. Preprocess a file that uses `__free(kfree)`.
6. Find a function in `mm/` with a documented `Context:` kernel-doc line. Verify the
   claim by inspecting its callers.
7. **Classify the UB.** For each of the four categories in T.1, find a real kernel construct
   that falls into it and explain what the kernel does about it.
8. **Read `include/linux/minmax.h`** in full. Explain every layer of `min()`'s expansion,
   including why `__careful_cmp` exists and what problem `__cmp_once` solves.
9. **Read `include/linux/overflow.h`** in full. Implement `struct_size()` yourself from
   first principles, then compare with the real one and explain the differences.
10. **Effect-system argument.** Write 300 words explaining `__user` as a type-system feature,
    including what class of vulnerability it prevents and why C alone cannot express it.
    Then explain how Rust expresses the same thing (preview of Part 5: `UserSlice`).
11. **Find a `volatile` misuse.** `git grep -n '\bvolatile\b' -- drivers/ | head -40`.
    For each, decide whether it is one of the three legitimate cases or should be
    `READ_ONCE`/a lock. Several will be wrong; fixing one is a real patch.
12. **Flexible arrays.** Find a `[0]` or `[1]` trailing array in the tree and convert it to
    `[]` with `__counted_by()`. Verify `UBSAN_BOUNDS` catches an overflow afterwards.
    (`git log --grep="__counted_by" --oneline | head` for precedents.)
13. **Preprocessor hygiene.** Construct a variable-capture bug with a kernel macro
    (e.g. call `min(x, _min1)`), observe the failure, and explain what hygiene would do.

---

## 6. Further reading

**Kernel documentation & headers:**
- `Documentation/process/coding-style.rst` ★★
- `Documentation/process/deprecated.rst` ★ (what not to use, and why)
- `Documentation/process/volatile-considered-harmful.rst`
- `Documentation/core-api/kernel-api.rst`
- `Documentation/doc-guide/kernel-doc.rst`
- `Documentation/dev-tools/sparse.rst`, `coccinelle.rst`, `checkpatch.rst`
- **Read these headers cover to cover** — they *are* the dialect:
  `include/linux/kernel.h`, `compiler.h`, `compiler_attributes.h`, `compiler_types.h`,
  `container_of.h`, `cleanup.h`, `overflow.h`, `minmax.h`, `bits.h`, `bitfield.h`,
  `build_bug.h`, `err.h`, `types.h`, `init.h`, `export.h`, `static_call.h`, `jump_label.h`

**Standards & compiler docs:**
- ISO C17 (N2176 draft) §6.5 (expressions), §6.3 (conversions), Annex J (UB list)
- GCC manual: "C Extensions" chapter — statement expressions, `typeof`, attributes,
  `asm goto`, `__builtin_*`
- Clang Language Extensions documentation

**Papers & articles:**
- Wang, Chen, Jia, Zeldovich, Kaashoek, "Undefined Behavior: What Happened to My Code?"
  (APSYS 2012) — the STACK paper; measures UB-induced bugs *in Linux* specifically
- Wang et al., "Towards Optimization-Safe Systems: Analyzing the Impact of Undefined
  Behavior" (SOSP 2013)
- Regehr, "A Guide to Undefined Behavior in C and C++" (3-part blog series)
- Lattner, "What Every C Programmer Should Know About Undefined Behavior" (LLVM blog, 3 parts)
- Memarian et al., "Into the Depths of C: Elaborating the De Facto Standards" (PLDI 2016) —
  what C *actually* means in practice vs. the standard

**Other:**
- Kernel Self-Protection Project: https://kernsec.org/wiki/index.php/Kernel_Self_Protection_Project
- LWN: "The kernel's preferred coding style", "Sparse and the kernel", "static_call()",
  "Flexible arrays and __counted_by", "Cleaning up with guard()"

→ Next: [09-data-structures-1.md](09-data-structures-1.md)

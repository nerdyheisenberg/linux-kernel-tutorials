# Chapter 50 — Driver testing: KUnit, fault injection, virtual hardware, and CI

> **Goal:** Understand why driver testing is genuinely harder than testing other kernel code, why "I tested it on my board" is not a testing strategy, how to make hardware-dependent code testable by design rather than by heroics, what each tool in the kernel's testing arsenal actually proves and what it cannot, and how to build a feedback loop where a regression is caught in seconds instead of in a customer's field deployment. By the end you can write KUnit tests for driver logic, inject every error your error paths claim to handle, emulate a device in QEMU, and assemble all of it into CI.

> **This chapter closes Part 2.** It is also the one that determines whether the previous 24 chapters produce working drivers or plausible-looking ones.

---

## Theory & First Principles

### T.0 — Start here: you cannot unit-test a driver

Everything you know about testing assumes you can call a function and check its return value.
A driver breaks that assumption in five distinct ways — and each one is why a *different*
tool exists.

```c
static irqreturn_t my_isr(int irq, void *data)
{
	struct my_dev *d = data;
	u32 status = readl(d->base + REG_STATUS);   /* ① needs HARDWARE */
	if (!(status & IRQ_PENDING))
		return IRQ_NONE;
	writel(status, d->base + REG_ACK);
	queue_work(d->wq, &d->work);                /* ② ASYNCHRONOUS */
	return IRQ_HANDLED;
}
```

| Obstacle | Why the usual approach fails | The answer |
|---|---|---|
| ① **Needs real hardware** | You cannot `assert()` against a device you do not have | a QEMU virtual device, or mock the bus |
| ② **Asynchronous** | The result appears in a completion callback, later, on another CPU | tracepoints + event-based assertions |
| ⁢ **Concurrent** | The bug needs an interrupt at one exact instruction | KCSAN, lockdep, fault injection |
| ⁣ **Failure paths are unreachable** | You cannot make `kmalloc` fail on demand... | ...except you can: `failslab` |
| ⁤ **Correctness is partly external** | "the packet went out" is not observable from inside | loopback, a second machine, a protocol analyser |

**So driver testing is not one activity.** It is five, at different levels, and a mature
project does all of them:

```
   LEVEL              TOOL                 CATCHES               COST
   -----              ----                 -------               ----
 1 pure logic         KUnit                arithmetic, parsing   seconds
 2 kernel behaviour   QEMU + virtual dev   probe/remove, ioctl   ~30 s
 3 concurrency        KCSAN, lockdep       races, lock order     a build
 4 error paths        fault injection      the 90% never run     scripted
 5 the real thing     hardware + stress    timing, errata        hours
```

**Level 4 is the one almost nobody does, and it is the highest-yield.** Consider what fraction
of your driver is error handling — in Ch. 28's example it was 40% of the probe function — and
then ask how much of it has ever executed. The answer is usually *none*, which means 40% of
your code is untested by construction:

```bash
# Make every 10th allocation fail, and watch your unwinding actually run:
echo 10 > /sys/kernel/debug/failslab/probability
echo 1  > /sys/kernel/debug/failslab/times
echo 1  > /sys/kernel/debug/failslab/verbose
modprobe mydriver        # does probe unwind correctly? Does it leak?
```

**And level 3 needs the reframing from Ch. 06 §T.0**, which is the single most important idea
in this chapter:

> These tools do not find bugs that *happened*. They find bugs that are *possible*. lockdep
> reports a lock inversion it never saw deadlock; KCSAN reports a race that did not corrupt
> anything this time. **Testing samples the interleaving space; these tools reason about it.**

Which is why the answer to "how do I test for races?" is not "run it a lot" — that samples a
vanishing fraction of the space (Ch. 13 §T.0) — but "turn on the checkers and exercise the
code once."

**The practical target to aim for**, and it is achievable: a single command that builds,
boots in QEMU, loads the driver, runs the functional tests, runs a bind/unbind loop, runs
with fault injection, and reports — in under two minutes, on every commit. Ch. 101 §T.10
makes the same argument for hardware. **The loop is the deliverable.**

```bash
make kunit.py run --arch=x86_64 drivers/mydrv
for i in $(seq 1 1000); do echo $DEV > .../unbind; echo $DEV > .../bind; done
cat /sys/kernel/debug/kmemleak
dmesg | grep -iE 'KASAN|BUG|WARN|lockdep'
```

---

### T.1 Why driver testing is a different problem

Testing ordinary software means: supply inputs, observe outputs, compare to expectations. Driver testing violates every part of that.

| Assumption of normal testing | How drivers break it |
|---|---|
| The system under test is deterministic | hardware timing, interrupts, and DMA introduce nondeterminism you do not control |
| Inputs are values you choose | inputs are *device behaviour*, including behaviour the datasheet does not describe |
| You can observe the output | much of the output is register writes to hardware you cannot read back |
| Failures are reproducible | many are timing-dependent and appear once per thousand boots |
| You can run many tests quickly | each test may require a physical reset cycle |
| The environment is available | you have one board; the customer has forty variants |
| Failure is cheap | a bad test can brick the device or corrupt a disk |

Add the kernel-specific difficulties from Ch. 06: no debugger by default, failures manifest far from their cause, and a memory-safety bug can silently corrupt an unrelated subsystem.

So the goal cannot be "test the driver against the hardware." It must be:

> **Decompose the driver so that most of it can be tested without hardware, and make the remaining hardware-dependent surface as small, explicit, and observable as possible.**

That is a *design* statement, not a testing statement, which is the chapter's central claim: **testability is an architectural property you build in, not a phase you add.**

### T.2 What is actually in a driver, and what each part needs

Take any driver from Part 2 and partition it:

| Part | Fraction | Depends on hardware? | How to test |
|---|---|---|---|
| Parsing (DT/ACPI/descriptors/firmware) | 5–15 % | **no** | KUnit, unit tests with synthetic input |
| Data transformation (format conversion, checksums, packing, register value computation) | 10–30 % | **no** | KUnit, property-based tests |
| State machines (link state, power state, protocol state) | 10–20 % | no, if separated | KUnit with a mock backend |
| Error paths | 20–40 % | no — they are triggered by failures you can inject | **fault injection** |
| Register I/O sequences | 10–20 % | yes | regmap mock, QEMU device, real hardware |
| Interrupt/DMA/timing | 5–15 % | yes | QEMU device, real hardware, stress |
| Integration with subsystems | 10–20 % | partially | virtual providers, conformance suites |

The striking number is the error paths. Palix et al. (ASPLOS 2011) found error-handling code has the highest defect density in Linux drivers, and the reason is structural: **error paths are the code that runs least often in testing and most often in the field**. A driver whose happy path is exercised a million times a day may have a `-ENOMEM` path that has literally never executed.

This is what makes fault injection (§T.5) not a nice-to-have but the single highest-value testing technique for drivers.

### T.3 The testing pyramid, translated to drivers

| Level | Speed | What it proves | Tools |
|---|---|---|---|
| **Static** | seconds | properties true of all executions | sparse, smatch, coccinelle, clang analyzer, `checkpatch` |
| **Unit (KUnit)** | seconds | one function does what it claims | KUnit, `kunit_tool` with UML/QEMU |
| **Fault injection** | minutes | every error path is reachable and correct | `failslab`, `fail_page_alloc`, `should_fail_bio`, custom |
| **Virtual hardware** | minutes | the driver drives *a* device correctly | QEMU devices, `vivid`/`vkms`/`vimc`/`dummy_hcd`/`gpio-sim` |
| **Subsystem conformance** | minutes | the driver obeys its subsystem's ABI | `v4l2-compliance`, `igt`, `blktests`, `kselftest` |
| **Integration (kselftest, LTP)** | minutes–hours | the whole stack works | kselftest, LTP, xfstests |
| **Sanitizers + stress** | hours | no memory/concurrency bugs under load | KASAN, KCSAN, lockdep, syzkaller |
| **Real hardware** | hours–days | the datasheet was right | LAVA, KernelCI, your lab |

The economics: each level up is ~10× slower and ~10× more expensive to maintain, and catches a *different* class of bug. The rule that follows:

> **Push every test as far down the pyramid as the bug class allows. A bug that could have been caught by KUnit but was caught on hardware cost you 1000× more to find.**

The corollary drivers routinely violate: if you find yourself needing hardware to test your DT parsing, your DT parsing is entangled with your hardware access, and *that* is the bug.

### T.4 KUnit: unit testing inside the kernel

KUnit (5.5+) runs tests **in kernel context**, either in User-Mode Linux (fast, no VM) or in a normal kernel. It is not a userspace framework testing kernel code through syscalls — the test function runs with full access to kernel APIs.

```c
#include <kunit/test.h>

static void parse_config_test(struct kunit *test)
{
	struct my_config cfg;
	int ret;

	ret = my_parse_config(&cfg, "width=1920,height=1080");
	KUNIT_ASSERT_EQ(test, ret, 0);          /* ASSERT: abort on failure */
	KUNIT_EXPECT_EQ(test, cfg.width, 1920); /* EXPECT: record and continue */
	KUNIT_EXPECT_EQ(test, cfg.height, 1080);
}

static void parse_config_invalid_test(struct kunit *test)
{
	struct my_config cfg;

	KUNIT_EXPECT_EQ(test, my_parse_config(&cfg, "width=abc"), -EINVAL);
	KUNIT_EXPECT_EQ(test, my_parse_config(&cfg, ""),          -EINVAL);
	KUNIT_EXPECT_EQ(test, my_parse_config(&cfg, NULL),        -EINVAL);
}

static struct kunit_case my_test_cases[] = {
	KUNIT_CASE(parse_config_test),
	KUNIT_CASE(parse_config_invalid_test),
	{}
};

static struct kunit_suite my_test_suite = {
	.name = "my-driver",
	.init = my_test_init,         /* per-test setup */
	.exit = my_test_exit,
	.test_cases = my_test_cases,
};
kunit_test_suite(my_test_suite);
```

The `ASSERT` vs `EXPECT` distinction matters: `ASSERT` aborts the test (use when continuing would crash or be meaningless — e.g. after a pointer allocation), `EXPECT` records a failure and continues (use for independent checks, so one run reports all failures).

Two KUnit features that make it usable for drivers specifically:

**(a) `kunit_kzalloc()` and managed resources.** Memory allocated with `kunit_kzalloc(test, ...)` is freed automatically when the test ends, even if it fails. `kunit_add_action()` registers arbitrary cleanup. This is `devres` (Ch. 28) for tests, and it eliminates the leak-on-failure problem that makes hand-written cleanup in tests unreliable.

**(b) Device and fake-device helpers.** `kunit_device_register()` (6.6+) creates a real `struct device` on a fake bus, so code that needs a `struct device *` — which is nearly all driver code, for `dev_err()`, `devm_*`, and `dev_get_drvdata()` — can be tested. Before this existed, testing driver code meant either `root_device_register()` hacks or not testing it.

What KUnit **cannot** do: test anything requiring real hardware, real interrupts, or real DMA. It also cannot easily test code that is not exported or not in a separate function — which brings us to the design point.

> **The refactoring that makes a driver testable is almost always the same one that makes it readable: extract pure functions.**

A driver that computes a divider inline inside a `set_rate` callback has untestable logic. The same driver with

```c
static u32 mydev_calc_divider(unsigned long parent, unsigned long target);
```

has one testable function and a trivially correct caller. This is not test-driven design as ideology; it is the observation that *hardware access and computation are different concerns and mixing them is what makes the code untestable*.

### T.5 Fault injection: making error paths execute

The problem: your driver has

```c
	buf = kmalloc(size, GFP_KERNEL);
	if (!buf) {
		clk_disable_unprepare(priv->clk);   /* is this right? */
		return -ENOMEM;
	}
```

and that branch has never run. Fault injection makes it run.

Linux's framework (`lib/fault-inject.c`) provides parameterised, filterable failure injection:

| Injector | Fails |
|---|---|
| `failslab` | `kmalloc`/`kmem_cache_alloc` |
| `fail_page_alloc` | `alloc_pages` |
| `fail_futex` | futex operations |
| `fail_make_request` | block I/O |
| `fail_usercopy` | `copy_to/from_user` |
| `fail_mmc_request` | MMC commands |
| `fail_function` | **any function you name, returning a value you choose** |
| `fail_sunrpc`, `fail_io_timeout`, ... | subsystem-specific |

Each has a debugfs directory with:

```
probability      # percent chance per eligible call
interval         # fail every Nth call instead
times            # how many failures before stopping (-1 = unlimited)
space            # skip the first N bytes/calls
verbose          # 0=silent, 1=message, 2=message+stacktrace
task-filter      # only fail for tasks with make-it-fail set
require-start/end, ignore-start/end   # address range filters
stacktrace-depth
```

The **task filter** is the feature that makes this usable: without it, injecting 10 % `kmalloc` failures kills the whole system before your test runs. With it:

```sh
echo 1 > /proc/self/make-it-fail     # this shell and its children only
exec ./my_test
```

`fail_function` (`CONFIG_FAIL_FUNCTION`, requires `ALLOW_ERROR_INJECTION` annotation on the target) is the most precise tool:

```c
/* In the function you want to be injectable: */
ALLOW_ERROR_INJECTION(my_hw_init, ERRNO);
```

```sh
echo my_hw_init > /sys/kernel/debug/fail_function/inject
echo -12 > /sys/kernel/debug/fail_function/my_hw_init/retval   # -ENOMEM
echo 100 > /sys/kernel/debug/fail_function/probability
```

Now `my_hw_init()` returns `-ENOMEM` without executing, and you can verify the caller's cleanup exactly.

The theoretical framing: fault injection is **coverage of the error-handling control-flow graph**. Normal testing covers the happy path; fault injection covers the edges leaving each fallible call. The number of such edges in a driver's probe is typically 10–20, and covering all of them is a finite, achievable goal — Lab 50.3 does exactly that.

What fault injection proves and does not:

- ✅ The error path executes without crashing.
- ✅ Combined with KASAN/kmemleak, resources are released correctly.
- ✅ The error is propagated (the caller sees a failure).
- ❌ That the error is the *right* error (`-ENOMEM` vs `-EINVAL`) — that needs assertions.
- ❌ That the hardware is left in a sane state — that needs a mock or real device.

### T.6 Virtual hardware: the highest-leverage investment

The kernel contains an increasingly complete set of virtual devices whose entire purpose is to let driver and subsystem code be developed and tested without hardware:

| Virtual device | Emulates |
|---|---|
| `vivid` | a full-featured V4L2 capture/output device (Ch. 47) |
| `vimc` | a Media Controller camera pipeline (Ch. 47) |
| `vkms` | a complete KMS display (Ch. 47) |
| `dummy_hcd` + `gadget` | a USB host and device, connected (Ch. 39) |
| `gpio-sim`, `gpio-mockup` | GPIO controllers with userspace-driven line state (Ch. 42) |
| `i2c-stub`, `i2c-gpio` | I²C buses and devices (Ch. 40) |
| `spi-loopback-test` | SPI transfers (Ch. 41) |
| `scsi_debug`, `null_blk`, `nbd`, `brd` | block devices with configurable behaviour (Part 3) |
| `dummy`, `veth`, `netdevsim` | network devices (Ch. 46) |
| `uinput`, `uhid` | input devices (Ch. 45) |
| `vhci-hcd`, `usbip` | remote USB |
| `iio_dummy`, `hwmon` test drivers | sensors (Ch. 44) |
| `fake` / `faux` bus devices | bus infrastructure (Ch. 30) |
| QEMU `edu`, `pci-testdev`, `ivshmem` | PCI devices with documented, simple behaviour (Ch. 37) |

Two distinct uses, often confused:

1. **Testing the *subsystem*** — `vkms` exists so `igt` can test the DRM core on any machine.
2. **Testing *your driver*** — you write a QEMU device model matching your hardware's register interface, and develop the driver against it.

The second is the one engineers underuse, and the economics strongly favour it. Writing a QEMU device model for a peripheral takes days. It buys you: a reproducible target, the ability to inject arbitrary device behaviour (including illegal behaviour the real chip cannot produce), full visibility into what the driver wrote, no reset cycles, parallel test execution, and a target for CI that runs on any machine. For any driver you will maintain for years, this pays back quickly.

Additionally, `record/replay` in QEMU (Ch. 04 §T.6) makes even the nondeterministic parts reproducible, which directly attacks §T.1's worst property.

### T.7 The sanitizers and what each one proves

From Ch. 06, applied specifically to drivers:

| Tool | Finds | Cost | Driver-specific note |
|---|---|---|---|
| **KASAN** | use-after-free, out-of-bounds | 2–3× slow, 2× memory | **cannot see DMA writes** — a device writing past a buffer is invisible |
| **KFENCE** | same, sampled | ~0 | usable in production; catches what KASAN would, eventually |
| **KMSAN** | use of uninitialised memory | 4× | catches "driver read a register into a struct field it forgot to set" |
| **UBSAN** | shifts, overflows, misaligned access | small | catches register-field arithmetic bugs |
| **KCSAN** | data races | high | finds the ISR-vs-process-context races that define driver concurrency |
| **lockdep** | lock ordering, IRQ-safety violations | moderate | **essential**; catches the sleeping-in-atomic and ABBA bugs of Ch. 14 |
| **DEBUG_ATOMIC_SLEEP** | `might_sleep()` violations | small | catches `msleep()` in an IRQ handler |
| **PROVE_RCU** | RCU misuse | moderate | Ch. 15 |
| **kmemleak** | unreferenced allocations | high | catches `devm_`-less allocations on error paths |
| **DEBUG_KOBJECT_RELEASE** | premature object free | small | catches missing `release()` (Ch. 26) |

The KASAN caveat deserves emphasis because it is a real blind spot:

> **KASAN instruments CPU accesses. A device DMAing past the end of a buffer writes memory with no CPU instruction involved, so KASAN sees nothing.** The corruption surfaces later, somewhere else.

Mitigations: use `dma_alloc_coherent` sizes that match exactly, use an IOMMU with strict invalidation (Ch. 36 §T.4) so out-of-range DMA faults, and in QEMU, make your device model assert on out-of-range writes.

The canonical driver-testing `.config` fragment:

```
CONFIG_KASAN=y
CONFIG_KASAN_INLINE=y
CONFIG_UBSAN=y
CONFIG_KMSAN=y                     # (separate build; conflicts with KASAN)
CONFIG_PROVE_LOCKING=y
CONFIG_DEBUG_ATOMIC_SLEEP=y
CONFIG_DEBUG_LOCK_ALLOC=y
CONFIG_PROVE_RCU=y
CONFIG_DEBUG_OBJECTS=y
CONFIG_DEBUG_OBJECTS_TIMERS=y
CONFIG_DEBUG_OBJECTS_WORK=y
CONFIG_DEBUG_KOBJECT_RELEASE=y
CONFIG_DEBUG_LIST=y
CONFIG_DEBUG_SG=y
CONFIG_DMA_API_DEBUG=y             # <-- driver-specific, see below
CONFIG_FAULT_INJECTION=y
CONFIG_FAILSLAB=y
CONFIG_FAIL_PAGE_ALLOC=y
CONFIG_FAIL_FUNCTION=y
CONFIG_FAULT_INJECTION_DEBUG_FS=y
CONFIG_FAULT_INJECTION_STACKTRACE_FILTER=y
CONFIG_KUNIT=y
CONFIG_KUNIT_ALL_TESTS=y
```

`CONFIG_DMA_API_DEBUG` is the one specific to this chapter's subject: it tracks every DMA mapping and reports unmapped frees, double unmaps, mappings freed while still mapped, wrong direction on sync, and use of a mapping after unmap. Every bug from Ch. 35 §T.5's catalogue becomes a loud warning. **If you write a DMA-capable driver and have not run it with `DMA_API_DEBUG`, you have not tested it.**

### T.8 Syzkaller and why fuzzing finds what you cannot

Coverage-guided fuzzing (Ch. 06 §T.7) generates syscall sequences, observes coverage via KCOV, and mutates toward new coverage. Applied to drivers it is extraordinarily effective, because:

- It explores **interleavings and orderings** a human would not try (ioctl B before ioctl A, `close()` during a blocking read, `mmap` then unbind).
- It combines with sanitizers, so a reached bug is *reported* rather than silently corrupting.
- It never gets bored.

For a driver to be fuzzable it must be reachable from syscalls, which means `/dev` nodes, sysfs, netlink, or ioctl. That covers most of what a driver exposes, and the fuzzer will find:

- ioctl argument validation gaps (missing bounds checks, integer overflow in size computations),
- state-machine violations (operations in the wrong order),
- lifetime bugs (use after `close()`, races between unbind and I/O),
- memory-safety bugs in parsing anything userspace supplies.

Writing a syzkaller **description** for your ioctl interface (`sys/linux/dev_mydriver.txt`) multiplies its effectiveness by orders of magnitude, because the fuzzer then generates structurally valid inputs and spends its time on semantics rather than on rediscovering that the first argument is a magic number.

What fuzzing does **not** find: wrong behaviour that is not a crash. A driver that returns the wrong data, programs the wrong register, or silently drops packets passes fuzzing perfectly. Fuzzing tests *safety*, not *correctness*.

### T.9 CI, and the feedback-loop argument

Everything above is worthless if it runs once, manually, before a release. The value of a test is roughly

$$
V \approx \frac{(\text{bugs caught}) \times (\text{cost if it escaped})}{\text{latency to result} \times \text{maintenance cost}}
$$

and *latency* is in the denominator for a reason developed in Ch. 04 §T.1: a test that reports in 30 seconds changes how you work; one that reports in 6 hours does not.

A practical driver CI ladder:

| Stage | Runs on | Latency | Content |
|---|---|---|---|
| Pre-commit | your machine | < 1 min | `checkpatch`, `make C=1` (sparse), `smatch`, KUnit |
| Per-push | CI runner | < 10 min | full allmodconfig build, KUnit, QEMU boot + virtual-device tests |
| Nightly | CI | hours | KASAN/KCSAN builds, fault injection sweep, kselftest, conformance suites |
| Weekly | hardware lab | hours | real boards via LAVA, power measurement, long-run stress |
| Continuous | fleet | ∞ | syzkaller, KernelCI |

The kernel's own infrastructure to know about and use:

- **KernelCI** (`kernelci.org`) — builds and boots mainline on hundreds of boards, results public.
- **LAVA** (Linaro Automated Validation Architecture) — the board-farm automation most hardware labs run.
- **0-day / kernel test robot** — builds every posted patch in hundreds of configs and emails you within hours. If you post a patch and it breaks `m68k allmodconfig`, you will hear about it. This is free, comprehensive build coverage.
- **syzbot** — runs syzkaller continuously on mainline and linux-next, files and bisects bugs automatically.
- **kunit_tool** — `./tools/testing/kunit/kunit.py run` builds a UML kernel and runs KUnit in ~30 seconds.

A final point on culture, because it is the one that actually determines outcomes:

> **A test suite's value is destroyed by flaky tests.** A test that fails 2 % of the time for unrelated reasons trains everyone to ignore failures, which makes the entire suite worthless. Fix or delete flaky tests immediately; there is no acceptable middle state.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `lib/kunit/` | the KUnit framework: `test.c`, `assert.c`, `executor.c`, `device.c` |
| `include/kunit/test.h`, `kunit/device.h`, `kunit/resource.h` | the API |
| `tools/testing/kunit/kunit.py` | the runner (UML or QEMU) |
| `lib/fault-inject.c`, `include/linux/fault-inject.h` | the injection framework |
| `kernel/fail_function.c` | `fail_function` |
| `mm/failslab.c`, `mm/page_alloc.c` (`fail_page_alloc`) | allocator injectors |
| `tools/testing/selftests/` | kselftest: userspace tests, one directory per subsystem |
| `tools/testing/selftests/kselftest_harness.h` | the C test harness (fixtures, assertions) |
| `Documentation/dev-tools/` | **all of it**; see §4 |
| `scripts/coccinelle/` | semantic patches, including many bug-finding ones |
| `lib/test_*.c`, `lib/*_kunit.c` | in-tree test modules; good examples |
| `drivers/*/…-test.c`, `*_test.c` | driver-specific KUnit suites (e.g. `drivers/gpu/drm/tests/`) |

### 1.2 KUnit's managed resources

```c
	/* Freed automatically at test end, even on failure. */
	void *buf = kunit_kzalloc(test, size, GFP_KERNEL);

	KUNIT_ASSERT_NOT_ERR_OR_NULL(test, buf);

	/* Arbitrary cleanup. */
	kunit_add_action(test, my_cleanup_fn, my_ptr);
	/* Or, failing the test if registration fails: */
	KUNIT_ASSERT_EQ(test, kunit_add_action_or_reset(test, fn, p), 0);

	/* A real struct device on a fake bus (6.6+). */
	struct device *dev = kunit_device_register(test, "my-test-dev");
	KUNIT_ASSERT_NOT_ERR_OR_NULL(test, dev);
```

`kunit_device_register()` is the key enabler for driver testing: it gives you a device whose `devm_` allocations are tracked and released, so you can call driver init functions that expect a `struct device *`.

### 1.3 The assertion vocabulary

```c
KUNIT_EXPECT_EQ / NE / LT / LE / GT / GE (test, a, b)
KUNIT_EXPECT_PTR_EQ / PTR_NE
KUNIT_EXPECT_TRUE / FALSE
KUNIT_EXPECT_NULL / NOT_NULL
KUNIT_EXPECT_STREQ / STRNEQ
KUNIT_EXPECT_MEMEQ / MEMNEQ           (test, a, b, size)
KUNIT_EXPECT_NOT_ERR_OR_NULL
KUNIT_EXPECT_*_MSG(..., "fmt", args)  /* every macro has a _MSG variant */
KUNIT_ASSERT_*                         /* same set, aborts on failure */
KUNIT_FAIL(test, "fmt", ...)
kunit_skip(test, "reason")             /* conditional skip, e.g. no hardware */
```

`KUNIT_EXPECT_MEMEQ` is particularly useful for drivers: compare a computed register image against a golden buffer and get a hex diff on failure.

### 1.4 Parameterised tests

For table-driven driver logic (format tables, divider tables, quirk tables), parameterised tests avoid writing N nearly identical test functions:

```c
struct div_case {
	const char *desc;
	unsigned long parent, target;
	u32 expected_div;
};

static const struct div_case div_cases[] = {
	{ "exact",        24000000,  12000000,  2 },
	{ "round down",   24000000,   7000000,  3 },
	{ "clamp to max", 24000000,       100, 16 },
	{ "clamp to min", 24000000, 100000000,  1 },
};

KUNIT_ARRAY_PARAM_DESC(div, div_cases, desc);

static void divider_test(struct kunit *test)
{
	const struct div_case *c = test->param_value;

	KUNIT_EXPECT_EQ(test, mydev_calc_divider(c->parent, c->target),
			c->expected_div);
}

static struct kunit_case cases[] = {
	KUNIT_CASE_PARAM(divider_test, div_gen_params),
	{}
};
```

Each row runs as a separately named, separately reported test.

---

## 2. Practice

### Lab 50.1 — KUnit in 60 seconds, then on your own code

```sh
cd /path/to/linux
./tools/testing/kunit/kunit.py run --timeout=120 --jobs=$(nproc)
```

That builds a UML kernel, runs every enabled KUnit suite, and prints TAP output. Time it — this is the feedback loop of §T.9.

Run a subset and see the config it generates:

```sh
./tools/testing/kunit/kunit.py run 'drm_*' --kunitconfig=drivers/gpu/drm/tests/.kunitconfig
cat .kunit/.kunitconfig
./tools/testing/kunit/kunit.py run --arch=x86_64 --kconfig_add CONFIG_KASAN=y
```

Now write tests for real driver logic. Take the divider from Ch. 43's lab and make it testable:

`mydev_calc.c`:

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/kernel.h>
#include <linux/math64.h>
#include "mydev.h"

/* Pure function: no hardware, no allocation, no locks. Testable. */
u32 mydev_calc_divider(unsigned long parent, unsigned long target)
{
	u32 d;

	if (!target || !parent)
		return 1;
	d = DIV_ROUND_CLOSEST_ULL((u64)parent, target);
	return clamp_t(u32, d, 1, 16);
}

/* Pure parser: input is a string, output is a struct. Testable. */
int mydev_parse_mode(const char *s, struct mydev_mode *out)
{
	unsigned int w, h, r;

	if (!s || !out)
		return -EINVAL;
	if (sscanf(s, "%ux%u@%u", &w, &h, &r) != 3)
		return -EINVAL;
	if (!w || !h || !r || w > 8192 || h > 8192 || r > 240)
		return -ERANGE;

	out->width = w;
	out->height = h;
	out->refresh = r;
	return 0;
}

/* Pure register-image builder: compare against a golden buffer. */
void mydev_build_ctrl(u8 *buf, const struct mydev_mode *m, bool enable)
{
	buf[0] = enable ? 0x01 : 0x00;
	buf[1] = m->width  >> 8;
	buf[2] = m->width  & 0xff;
	buf[3] = m->height >> 8;
	buf[4] = m->height & 0xff;
	buf[5] = m->refresh;
}
```

`mydev_test.c`:

```c
// SPDX-License-Identifier: GPL-2.0
#include <kunit/test.h>
#include <kunit/device.h>
#include "mydev.h"

/* ---- table-driven divider tests ---- */

struct div_case {
	const char *desc;
	unsigned long parent, target;
	u32 expected;
};

static const struct div_case div_cases[] = {
	{ "exact divide",       24000000, 12000000,  2 },
	{ "round to nearest",   24000000,  7000000,  3 },
	{ "clamp to maximum",   24000000,      100, 16 },
	{ "clamp to minimum",   24000000, 99000000,  1 },
	{ "zero target",        24000000,        0,  1 },
	{ "zero parent",               0, 12000000,  1 },
};

KUNIT_ARRAY_PARAM_DESC(div, div_cases, desc);

static void divider_test(struct kunit *test)
{
	const struct div_case *c = test->param_value;

	KUNIT_EXPECT_EQ_MSG(test, mydev_calc_divider(c->parent, c->target),
			    c->expected,
			    "parent=%lu target=%lu", c->parent, c->target);
}

/* ---- parser tests, including every rejection path ---- */

static void parse_valid_test(struct kunit *test)
{
	struct mydev_mode m;

	KUNIT_ASSERT_EQ(test, mydev_parse_mode("1920x1080@60", &m), 0);
	KUNIT_EXPECT_EQ(test, m.width, 1920);
	KUNIT_EXPECT_EQ(test, m.height, 1080);
	KUNIT_EXPECT_EQ(test, m.refresh, 60);
}

static void parse_invalid_test(struct kunit *test)
{
	struct mydev_mode m;

	KUNIT_EXPECT_EQ(test, mydev_parse_mode(NULL, &m),            -EINVAL);
	KUNIT_EXPECT_EQ(test, mydev_parse_mode("1920x1080", &m),     -EINVAL);
	KUNIT_EXPECT_EQ(test, mydev_parse_mode("garbage", &m),       -EINVAL);
	KUNIT_EXPECT_EQ(test, mydev_parse_mode("", &m),              -EINVAL);
	KUNIT_EXPECT_EQ(test, mydev_parse_mode("0x1080@60", &m),     -ERANGE);
	KUNIT_EXPECT_EQ(test, mydev_parse_mode("99999x1080@60", &m), -ERANGE);
	KUNIT_EXPECT_EQ(test, mydev_parse_mode("1920x1080@999", &m), -ERANGE);
}

/* ---- golden-image comparison ---- */

static void build_ctrl_test(struct kunit *test)
{
	struct mydev_mode m = { .width = 1920, .height = 1080, .refresh = 60 };
	static const u8 golden[6] = { 0x01, 0x07, 0x80, 0x04, 0x38, 0x3c };
	u8 buf[6] = {};

	mydev_build_ctrl(buf, &m, true);
	KUNIT_EXPECT_MEMEQ(test, buf, golden, sizeof(golden));
}

/* ---- a test that needs a struct device ---- */

static void devres_test(struct kunit *test)
{
	struct device *dev = kunit_device_register(test, "mydev-test");
	void *p;

	KUNIT_ASSERT_NOT_ERR_OR_NULL(test, dev);

	p = devm_kzalloc(dev, 128, GFP_KERNEL);
	KUNIT_ASSERT_NOT_NULL(test, p);
	/* kunit_device_register's cleanup unbinds the device, releasing
	 * devres -- so this allocation cannot leak. */
}

static struct kunit_case mydev_cases[] = {
	KUNIT_CASE_PARAM(divider_test, div_gen_params),
	KUNIT_CASE(parse_valid_test),
	KUNIT_CASE(parse_invalid_test),
	KUNIT_CASE(build_ctrl_test),
	KUNIT_CASE(devres_test),
	{}
};

static struct kunit_suite mydev_suite = {
	.name = "mydev",
	.test_cases = mydev_cases,
};
kunit_test_suite(mydev_suite);
MODULE_LICENSE("GPL");
```

`Kconfig`:

```
config MYDEV_KUNIT_TEST
	tristate "KUnit tests for mydev" if !KUNIT_ALL_TESTS
	depends on MYDEV && KUNIT
	default KUNIT_ALL_TESTS
```

`Makefile`:

```make
obj-$(CONFIG_MYDEV) += mydev.o
obj-$(CONFIG_MYDEV_KUNIT_TEST) += mydev_test.o
```

Run:

```sh
./tools/testing/kunit/kunit.py run 'mydev*' \
	--kconfig_add CONFIG_MYDEV=y --kconfig_add CONFIG_MYDEV_KUNIT_TEST=y
```

Now deliberately break `mydev_calc_divider` (change `clamp_t(u32, d, 1, 16)` to `1, 15`) and observe the exact failing parameter row is named. That precision is the payoff.

---

### Lab 50.2 — Make an untestable driver testable

Take this realistic, untestable code:

```c
/* BEFORE: logic entangled with hardware access */
static int mydev_set_rate(struct mydev *md, unsigned long rate)
{
	u32 val, div;

	div = md->parent_rate / rate;
	if (div == 0)
		div = 1;
	if (div > 16)
		div = 16;

	val = readl(md->regs + REG_CTRL);
	val &= ~CTRL_DIV_MASK;
	val |= FIELD_PREP(CTRL_DIV_MASK, div - 1);
	writel(val, md->regs + REG_CTRL);

	return 0;
}
```

Refactor it into three pieces:

```c
/* AFTER: pure computation (testable) */
u32 mydev_calc_divider(unsigned long parent, unsigned long target);

/* AFTER: pure register-value transformation (testable) */
u32 mydev_apply_div(u32 cur_ctrl, u32 div)
{
	return (cur_ctrl & ~CTRL_DIV_MASK) | FIELD_PREP(CTRL_DIV_MASK, div - 1);
}

/* AFTER: trivially correct glue (not worth testing) */
static int mydev_set_rate(struct mydev *md, unsigned long rate)
{
	u32 div = mydev_calc_divider(md->parent_rate, rate);

	writel(mydev_apply_div(readl(md->regs + REG_CTRL), div),
	       md->regs + REG_CTRL);
	return 0;
}
```

Now test the first two with KUnit and note that the third contains no decisions.

Then, for drivers using **regmap** (Ch. 34), you get something better: a mock register map.

```c
static const struct regmap_config test_cfg = {
	.reg_bits = 8, .val_bits = 8, .max_register = 0xff,
	.cache_type = REGCACHE_FLAT,
};

static void regmap_sequence_test(struct kunit *test)
{
	struct device *dev = kunit_device_register(test, "mydev-rm");
	struct regmap *map;
	unsigned int v;

	KUNIT_ASSERT_NOT_ERR_OR_NULL(test, dev);

	/* No bus: all reads/writes go to the cache. A perfect mock. */
	map = devm_regmap_init(dev, NULL, NULL, &test_cfg);
	KUNIT_ASSERT_FALSE(test, IS_ERR(map));

	KUNIT_EXPECT_EQ(test, mydev_hw_init(map), 0);

	/* Assert on the exact register image the init sequence produced. */
	KUNIT_EXPECT_EQ(test, regmap_read(map, REG_CTRL, &v), 0);
	KUNIT_EXPECT_EQ(test, v, 0x8a);
	KUNIT_EXPECT_EQ(test, regmap_read(map, REG_MODE, &v), 0);
	KUNIT_EXPECT_EQ(test, v, 0x03);
}
```

This is a genuinely powerful technique and under-used: **regmap with no bus is a free mock for any register-based driver**, and it lets you unit-test entire initialisation sequences. `drivers/base/regmap/regmap-kunit.c` in the tree does exactly this for regmap itself and is worth reading as an example.

For a bus-backed mock that records the actual traffic:

```c
struct mock_bus_ctx {
	struct kunit *test;
	u8 regs[256];
	unsigned int writes, reads;
};

static int mock_reg_write(void *ctx, unsigned int reg, unsigned int val)
{
	struct mock_bus_ctx *c = ctx;

	KUNIT_ASSERT_LT(c->test, reg, ARRAY_SIZE(c->regs));
	c->regs[reg] = val;
	c->writes++;
	return 0;
}

static int mock_reg_read(void *ctx, unsigned int reg, unsigned int *val)
{
	struct mock_bus_ctx *c = ctx;

	KUNIT_ASSERT_LT(c->test, reg, ARRAY_SIZE(c->regs));
	*val = c->regs[reg];
	c->reads++;
	return 0;
}

	map = devm_regmap_init(dev, NULL, &ctx, &cfg);   /* cfg.reg_write/read set */
```

Now you can assert not just on the final state but on the *number and order* of accesses — which is what actually matters for hardware that cares about sequence.

---

### Lab 50.3 — Exhaustively test every error path in probe

The goal: prove that every fallible call in `probe()` has a correct cleanup path.

**Step 1: count the failure points.**

```sh
# How many fallible calls does probe have?
sed -n '/static int mydev_probe/,/^}/p' mydev.c | grep -cE 'if \(.*(IS_ERR|< 0|!= 0|ret)\)'
```

**Step 2: fail them one at a time with `fail_function`.**

Annotate the internal functions:

```c
static int mydev_hw_init(struct mydev *md) { ... }
ALLOW_ERROR_INJECTION(mydev_hw_init, ERRNO);

static int mydev_register_subsys(struct mydev *md) { ... }
ALLOW_ERROR_INJECTION(mydev_register_subsys, ERRNO);
```

```sh
#!/bin/bash
# inject-probe-failures.sh
FF=/sys/kernel/debug/fail_function
DEV=mydev.0
BUS=/sys/bus/platform/drivers/mydev

for fn in mydev_hw_init mydev_register_subsys mydev_setup_irq; do
  for err in -12 -22 -110 -517; do        # ENOMEM EINVAL ETIMEDOUT EPROBE_DEFER
    echo "=== $fn returning $err"
    echo $DEV | sudo tee $BUS/unbind >/dev/null 2>&1

    echo $fn        | sudo tee $FF/inject   >/dev/null
    echo $err       | sudo tee $FF/$fn/retval >/dev/null
    echo 100        | sudo tee $FF/probability >/dev/null
    echo 1          | sudo tee $FF/times     >/dev/null

    echo $DEV | sudo tee $BUS/bind >/dev/null 2>&1
    dmesg | tail -5

    # Clean up the injection
    echo '' | sudo tee $FF/inject >/dev/null
  done
done

# Now: did anything leak?
echo scan | sudo tee /sys/kernel/debug/kmemleak
sleep 5
sudo cat /sys/kernel/debug/kmemleak
```

**Step 3: fail allocations, scoped to your test.**

```sh
FI=/sys/kernel/debug/failslab
echo 100  | sudo tee $FI/probability
echo 1    | sudo tee $FI/times
echo 0    | sudo tee $FI/space
echo 1    | sudo tee $FI/verbose
echo 1    | sudo tee $FI/task-filter      # ONLY tasks with make-it-fail

# Fail the Nth allocation specifically, sweeping N:
for n in $(seq 0 30); do
  echo $n | sudo tee $FI/space >/dev/null    # skip the first n
  echo 1  | sudo tee $FI/times  >/dev/null
  sudo sh -c 'echo 1 > /proc/self/make-it-fail; echo mydev.0 > /sys/bus/platform/drivers/mydev/bind'
  dmesg | tail -3
  sudo sh -c 'echo mydev.0 > /sys/bus/platform/drivers/mydev/unbind' 2>/dev/null
done
```

**Step 4: verify with the sanitizers.** Run the whole sweep on a kernel with `KASAN + kmemleak + DEBUG_OBJECTS + DMA_API_DEBUG + PROVE_LOCKING`. The combination is what makes this conclusive: fault injection makes the path *execute*, and the sanitizers determine whether it was *correct*.

**Step 5: automate it as a kselftest** so it runs forever:

```c
// tools/testing/selftests/drivers/mydev/probe_faults.c
#include "../../kselftest_harness.h"

FIXTURE(mydev) { int ff_fd; };

FIXTURE_SETUP(mydev) { /* open debugfs knobs */ }
FIXTURE_TEARDOWN(mydev) { /* clear injections, rebind */ }

TEST_F(mydev, probe_fails_cleanly_on_enomem)
{
	inject(self, "mydev_hw_init", -ENOMEM);
	ASSERT_LT(bind_device(), 0);
	EXPECT_EQ(count_leaked_objects(), 0);
	clear_injection(self);
	ASSERT_EQ(bind_device(), 0);   /* recovers: can bind after failure */
}

TEST_HARNESS_MAIN
```

That last assertion is the one people forget: **after a failed probe, the driver must be able to probe successfully.** A cleanup that leaves a stale registration or a held clock breaks the retry, and `-EPROBE_DEFER` (Ch. 27) means probe retries are *normal*.

---

### Lab 50.4 — Test against virtual hardware

**(a) Use the kernel's virtual devices.** Build a complete test environment with no hardware:

```sh
sudo modprobe gpio-sim
sudo modprobe i2c-stub chip_addr=0x48
sudo modprobe vivid n_devs=2
sudo modprobe vkms
sudo modprobe dummy_hcd
sudo modprobe netdevsim
sudo modprobe scsi_debug dev_size_mb=64

ls /dev/video* /dev/dri/* /dev/gpiochip*
```

Configure `gpio-sim` via configfs (the modern interface):

```sh
CF=/sys/kernel/config/gpio-sim/labchip
sudo mkdir -p $CF/bank0
echo 16 | sudo tee $CF/bank0/num_lines
sudo mkdir -p $CF/bank0/line4/hog
echo 1 | sudo tee $CF/live
gpiodetect
gpioinfo
# Drive a line from "hardware" side:
ls /sys/devices/platform/gpio-sim.0/gpiochip*/sim_gpio*/
```

**(b) Run the subsystem conformance suites** against them:

```sh
v4l2-compliance -d /dev/video0 -s -a        # V4L2
sudo igt_runner -t kms_ --device drm:/sys/.../vkms/drm/card0   # DRM
sudo ./tools/testing/selftests/gpio/gpio-sim.sh
sudo blktests -q block/                      # block (Part 3)
```

Then run them against a **real** device of the same class and compare failures. Everything `vivid` passes and your webcam fails is either a driver bug or an ABI ambiguity worth reporting.

**(c) Write a QEMU device model for your hardware.** This is the highest-value item in the chapter. Minimal skeleton:

```c
/* hw/misc/lab-testdev.c */
#include "qemu/osdep.h"
#include "hw/pci/pci_device.h"
#include "hw/pci/msi.h"
#include "qemu/module.h"
#include "qom/object.h"

#define TYPE_LAB_TESTDEV "lab-testdev"
OBJECT_DECLARE_SIMPLE_TYPE(LabTestDev, LAB_TESTDEV)

struct LabTestDev {
	PCIDevice	pdev;
	MemoryRegion	mmio;
	uint32_t	ctrl, status, data;
	QEMUTimer	*timer;
	bool		fail_next;      /* <-- inject device misbehaviour */
};

static uint64_t lab_mmio_read(void *opaque, hwaddr addr, unsigned size)
{
	LabTestDev *d = opaque;

	switch (addr) {
	case 0x00: return 0x1ab00001;       /* ID */
	case 0x04: return d->ctrl;
	case 0x08: return d->fail_next ? 0xffffffff : d->status;
	case 0x0c: return d->data;
	}
	/* Catch driver bugs: reading an undefined register. */
	qemu_log_mask(LOG_GUEST_ERROR,
		      "lab-testdev: read from undefined reg 0x%" HWADDR_PRIx "\n",
		      addr);
	return 0;
}

static void lab_mmio_write(void *opaque, hwaddr addr, uint64_t val,
			   unsigned size)
{
	LabTestDev *d = opaque;

	switch (addr) {
	case 0x04:
		d->ctrl = val;
		if (val & 0x1) {   /* START: complete after 1ms, raise MSI */
			timer_mod(d->timer,
				  qemu_clock_get_ms(QEMU_CLOCK_VIRTUAL) + 1);
		}
		return;
	case 0x0c: d->data = val; return;
	case 0x10: d->fail_next = !!val; return;   /* test hook */
	}
	qemu_log_mask(LOG_GUEST_ERROR,
		      "lab-testdev: write to undefined reg 0x%" HWADDR_PRIx "\n",
		      addr);
}

static const MemoryRegionOps lab_mmio_ops = {
	.read = lab_mmio_read,
	.write = lab_mmio_write,
	.endianness = DEVICE_LITTLE_ENDIAN,
	.valid.min_access_size = 4,
	.valid.max_access_size = 4,
};

static void lab_timer_cb(void *opaque)
{
	LabTestDev *d = opaque;

	d->status |= 0x1;                       /* DONE */
	msi_notify(&d->pdev, 0);
}

static void lab_realize(PCIDevice *pdev, Error **errp)
{
	LabTestDev *d = LAB_TESTDEV(pdev);

	if (msi_init(pdev, 0, 1, true, false, errp) < 0)
		return;
	d->timer = timer_new_ms(QEMU_CLOCK_VIRTUAL, lab_timer_cb, d);
	memory_region_init_io(&d->mmio, OBJECT(d), &lab_mmio_ops, d,
			      "lab-testdev-mmio", 0x1000);
	pci_register_bar(pdev, 0, PCI_BASE_ADDRESS_SPACE_MEMORY, &d->mmio);
}

static void lab_class_init(ObjectClass *klass, void *data)
{
	PCIDeviceClass *k = PCI_DEVICE_CLASS(klass);

	k->realize   = lab_realize;
	k->vendor_id = 0x1af4;
	k->device_id = 0xdead;
	k->class_id  = PCI_CLASS_OTHERS;
}

static const TypeInfo lab_info = {
	.name          = TYPE_LAB_TESTDEV,
	.parent        = TYPE_PCI_DEVICE,
	.instance_size = sizeof(LabTestDev),
	.class_init    = lab_class_init,
	.interfaces = (InterfaceInfo[]) {
		{ INTERFACE_CONVENTIONAL_PCI_DEVICE }, { },
	},
};

static void lab_register(void) { type_register_static(&lab_info); }
type_init(lab_register)
```

```sh
qemu-system-x86_64 -kernel bzImage -device lab-testdev -d guest_errors ...
```

Two features of this model that you cannot get from real hardware and should exploit:

1. **`qemu_log_mask(LOG_GUEST_ERROR, ...)` on any illegal access.** Run with `-d guest_errors` and your driver's out-of-range register accesses become loud errors. Real silicon silently ignores them.
2. **The `fail_next` hook.** You can make the device return `0xffffffff` (the "device gone" pattern of Ch. 38 §T.3), NAK a transfer, complete out of order, or not complete at all — behaviours that are hard or impossible to provoke on real hardware but that your driver must survive.

Start from `hw/misc/edu.c` in the QEMU tree, which is a documented teaching device with DMA and interrupts, and `docs/specs/edu.rst` describing its register interface.

---

### Lab 50.5 — Run every sanitizer and interpret the results

Build a dedicated test kernel:

```sh
cp .config .config.prod
scripts/config --enable KASAN --enable KASAN_INLINE
scripts/config --enable UBSAN --enable UBSAN_BOUNDS --enable UBSAN_SHIFT
scripts/config --enable PROVE_LOCKING --enable DEBUG_ATOMIC_SLEEP
scripts/config --enable DEBUG_OBJECTS --enable DEBUG_OBJECTS_TIMERS \
               --enable DEBUG_OBJECTS_WORK --enable DEBUG_OBJECTS_FREE
scripts/config --enable DEBUG_KOBJECT_RELEASE
scripts/config --enable DEBUG_LIST --enable DEBUG_SG
scripts/config --enable DMA_API_DEBUG
scripts/config --enable DEBUG_KMEMLEAK
scripts/config --enable FAULT_INJECTION --enable FAILSLAB \
               --enable FAIL_FUNCTION --enable FAULT_INJECTION_DEBUG_FS
make olddefconfig && make -j$(nproc)
```

Write a module that triggers each one, confirm you can read each report, and build a personal reference:

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>
#include <linux/slab.h>
#include <linux/dma-mapping.h>
#include <linux/delay.h>
#include <linux/platform_device.h>

static int which;
module_param(which, int, 0);

static void trigger_kasan_uaf(void)
{
	char *p = kmalloc(64, GFP_KERNEL);

	kfree(p);
	pr_info("uaf read: %d\n", p[0]);      /* KASAN: use-after-free */
}

static void trigger_kasan_oob(void)
{
	char *p = kmalloc(64, GFP_KERNEL);

	p[64] = 1;                            /* KASAN: slab-out-of-bounds */
	kfree(p);
}

static void trigger_ubsan(void)
{
	int n = 32;

	pr_info("shift: %d\n", 1 << n);       /* UBSAN: shift out of bounds */
}

static void trigger_sleep_in_atomic(void)
{
	preempt_disable();
	msleep(1);                            /* DEBUG_ATOMIC_SLEEP */
	preempt_enable();
}

static void trigger_kmemleak(void)
{
	void *p = kmalloc(1024, GFP_KERNEL);

	(void)p;                              /* never freed, never referenced */
	pr_info("leaked 1024 bytes; echo scan > /sys/kernel/debug/kmemleak\n");
}

static void trigger_dma_debug(struct device *dev)
{
	void *buf = kmalloc(256, GFP_KERNEL);
	dma_addr_t dma;

	dma = dma_map_single(dev, buf, 256, DMA_TO_DEVICE);
	kfree(buf);                           /* DMA_API_DEBUG: freed while mapped */
	dma_unmap_single(dev, dma, 256, DMA_TO_DEVICE);
}

/* ... plus lockdep ABBA, DEBUG_OBJECTS double-init, DEBUG_LIST corruption ... */
```

For each, record: the exact first line of the report, how to identify the offending line, and how long the report took to appear. Keep this table; when a real report appears at 2 a.m. you will recognise it instantly. This extends Ch. 06's Lab 6.12 with the driver-specific injectors.

**Then run KCSAN** separately (it conflicts with KASAN in some configs) against a driver with an ISR and a process context sharing state. KCSAN is the only tool that finds the race in

```c
	/* Process context */
	priv->count++;
	/* IRQ context */
	priv->count++;         /* KCSAN: data race */
```

which is the defining bug class of driver concurrency.

---

### Lab 50.6 — Fuzz your driver's userspace interface

```sh
git clone https://github.com/google/syzkaller && cd syzkaller && make
```

Build a kernel with:

```
CONFIG_KCOV=y
CONFIG_KCOV_INSTRUMENT_ALL=y
CONFIG_DEBUG_INFO_DWARF4=y
CONFIG_KASAN=y
CONFIG_KASAN_INLINE=y
CONFIG_CONFIGFS_FS=y
CONFIG_SECURITYFS=y
CONFIG_DEBUG_FS=y
CONFIG_FAULT_INJECTION=y
CONFIG_FAULT_INJECTION_DEBUG_FS=y
CONFIG_FAULT_INJECTION_USERCOPY=y
```

Describe your interface so the fuzzer generates valid structures (`sys/linux/dev_mydev.txt`):

```
include <uapi/linux/mydev.h>

resource fd_mydev[fd]

openat$mydev(fd const[AT_FDCWD], file ptr[in, string["/dev/mydev0"]],
	     flags flags[open_flags], mode const[0]) fd_mydev

ioctl$MYDEV_SET_MODE(fd fd_mydev, cmd const[MYDEV_SET_MODE],
		     arg ptr[in, mydev_mode])
ioctl$MYDEV_GET_STATE(fd fd_mydev, cmd const[MYDEV_GET_STATE],
		      arg ptr[out, mydev_state])
ioctl$MYDEV_SUBMIT(fd fd_mydev, cmd const[MYDEV_SUBMIT],
		   arg ptr[in, mydev_request])

mydev_mode {
	width	int32[0:8192]
	height	int32[0:8192]
	flags	flags[mydev_mode_flags, int32]
}

mydev_request {
	len	len[buf, int32]
	buf	ptr[in, array[int8]]
	flags	int32
}
```

```sh
make generate && make
./bin/syz-manager -config my.cfg
```

Then do the exercise that teaches the most: **before running the fuzzer, write down every input validation your ioctl handler performs.** Run the fuzzer for an hour. Compare what it found against your list. The gap is your calibration of how good your own review is — and for most engineers, the first time is humbling.

Useful minimal alternative if syzkaller is too heavy: a 30-line ioctl fuzzer with random `cmd` and random pointer arguments will still find missing `_IOC_TYPE` checks and unvalidated size fields. Run it under KASAN.

---

### Lab 50.7 — Build the CI

Assemble everything into a pipeline you actually run. A concrete GitHub Actions / GitLab CI equivalent:

```yaml
# .github/workflows/kernel-driver-ci.yml
name: driver-ci
on: [push, pull_request]

jobs:
  static:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: checkpatch
        run: ./scripts/checkpatch.pl --strict -g HEAD~1..HEAD
      - name: sparse
        run: make C=2 drivers/mydev/ 2>&1 | tee sparse.log; ! grep -q warning sparse.log
      - name: smatch
        run: make CHECK="smatch -p=kernel" C=1 drivers/mydev/
      - name: coccinelle
        run: make coccicheck MODE=report M=drivers/mydev/

  kunit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./tools/testing/kunit/kunit.py run 'mydev*' --jobs=$(nproc)
      - run: ./tools/testing/kunit/kunit.py run 'mydev*' --kconfig_add CONFIG_KASAN=y

  build-matrix:
    strategy:
      matrix:
        arch: [x86_64, arm64, i386]
        config: [defconfig, allmodconfig]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: make ARCH=${{ matrix.arch }} ${{ matrix.config }}
      - run: make ARCH=${{ matrix.arch }} -j$(nproc) drivers/mydev/

  qemu-integration:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: scripts/config --enable MYDEV --enable KASAN --enable PROVE_LOCKING
      - run: make -j$(nproc) bzImage
      - name: boot and run tests
        run: |
          qemu-system-x86_64 -kernel arch/x86/boot/bzImage \
            -device lab-testdev -append "console=ttyS0 root=/dev/vda" \
            -nographic -no-reboot -d guest_errors \
            -initrd tests/initramfs.cpio.gz | tee boot.log
          grep -q "ALL TESTS PASSED" boot.log
          ! grep -qE "KASAN|BUG:|WARNING:|guest_errors" boot.log
```

The last two lines are the crucial pattern: **fail the build on any kernel warning, not just on test failures.** A `WARNING:` in `dmesg` during a passing test is a bug that has not manifested yet.

Additional pieces worth adding:

```sh
# Config coverage: does your driver build in every configuration?
make randconfig && scripts/config --enable MYDEV && make olddefconfig && make -j$(nproc)
# Repeat 50 times; this finds missing Kconfig depends and missing #include

# Does it build as a module AND built-in?
scripts/config --module MYDEV && make -j$(nproc)
scripts/config --enable MYDEV && make -j$(nproc)

# Does it build with CONFIG_PM=n, CONFIG_OF=n, CONFIG_ACPI=n?
# (this is what pm_ptr() and of_match_ptr() are for -- verify they work)

# DT binding validation (Ch. 32)
make dt_binding_check DT_SCHEMA_FILES=vendor,mydev.yaml
make dtbs_check DT_SCHEMA_FILES=vendor,mydev.yaml
```

Then measure your own loop:

| Stage | Your latency | Target |
|---|---|---|
| Syntax/style error caught | ? | < 10 s |
| Logic error caught | ? | < 60 s |
| Error-path bug caught | ? | < 10 min |
| Memory-safety bug caught | ? | < 30 min |
| Hardware-specific bug caught | ? | < 1 day |

If any row is an order of magnitude off, that is where to invest — not in writing more tests at a level you already cover.

---

## 3. Mastery drills

1. Take a driver you have written (or `drivers/input/keyboard/gpio_keys.c`) and partition every line into the seven categories of §T.2. Compute the percentage that could be tested without hardware. Then refactor to raise it by 20 points and state exactly what you changed.

2. Fault injection makes error paths execute; sanitizers determine whether they were correct. Construct an error-path bug that fault injection *alone* would not detect, and name the specific sanitizer that would.

3. KASAN cannot see DMA writes. Design a scheme that would detect device-initiated buffer overruns, using facilities that exist (IOMMU, guard pages, QEMU models). Estimate the overhead of each.

4. Prove that covering every edge leaving every fallible call in `probe()` is sufficient to exercise all reachable states of the cleanup logic, given that cleanup is a reverse-ordered goto ladder (Ch. 05). Then construct a `probe()` for which it is *not* sufficient.

5. `kunit_device_register()` creates a device on a fake bus. Enumerate what driver code this makes testable that was not before, and what still is not.

6. Design a KUnit test for a driver's *interrupt handler*. What must be true about the handler's structure for this to be possible, and what part fundamentally cannot be unit-tested?

7. A regmap with no bus is a free mock. Identify three classes of register-sequencing bug this catches and two it cannot, and propose what would catch the latter.

8. Coverage-guided fuzzing tests safety, not correctness. Design a *differential* testing scheme that would catch correctness bugs in a driver, using a reference implementation. What is your reference?

9. Flaky tests destroy a suite's value. Derive the probability that a 500-test suite passes cleanly if each test independently flakes 0.5 % of the time, and use it to argue a policy.

10. Your driver must work with `CONFIG_PM=n`, `CONFIG_OF=n`, and as both module and built-in. Design the minimal set of builds that covers all meaningful combinations, and justify why the full cross-product is unnecessary.

11. A bug is found in the field that none of your tests caught. Write the post-mortem template: what test level *should* have caught it, why it did not, and what change to the suite prevents recurrence. Apply it to a real bug from the kernel's git history.

12. Compare the cost of writing a QEMU device model for your hardware against the cost of a hardware test lab, over a five-year product lifetime. Include engineer time, capacity, reproducibility, and CI integration. State the conditions under which each wins.

13. You inherit a 12,000-line driver with zero tests, and one week. Order your work to maximise the probability of catching the next regression, and justify the ordering with the economics of §T.3.

---

## 4. Further reading

**Kernel documentation** (`Documentation/dev-tools/` — read this whole directory)

- `kunit/index.rst`, `kunit/start.rst`, `kunit/usage.rst`, `kunit/api/*` ★★★ — the complete KUnit story. `usage.rst` covers mocking strategies and the testability argument of §T.4.
- `kunit/running_tips.rst` ★★ — `kunit.py` options, coverage, debugging failures.
- `fault-injection/fault-injection.rst` ★★★ — every knob in §T.5, with examples. Also `notifier-error-inject.rst` and `nvme-fault-injection.rst` for subsystem-specific injectors.
- `kasan.rst`, `kmsan.rst`, `kcsan.rst`, `ubsan.rst`, `kfence.rst`, `kmemleak.rst` ★★★ — one per tool; each is short and tells you exactly what it finds.
- `kselftest.rst` ★★★ — writing and running userspace tests; the harness macros.
- `kcov.rst` ★★, `gcov.rst` ★★ — coverage measurement; use to find untested driver code.
- `sparse.rst`, `coccinelle.rst`, `checkpatch.rst`, `clang-tools.rst`, `gdb-kernel-debugging.rst` ★★★
- `testing-overview.rst` ★★★ — the maintainers' own version of §T.3's pyramid.
- `Documentation/process/*` — `submitting-patches.rst` and `maintainer-*.rst` describe the testing expectations for upstreaming (Part 6).

**Source worth reading as examples**

- `drivers/base/regmap/regmap-kunit.c` ★★★ — the busless-regmap mocking technique of Lab 50.2, used properly and at scale.
- `drivers/gpu/drm/tests/` ★★★ — a large, modern KUnit suite for a complex subsystem; `drm_format_helper_test.c` shows golden-image comparison.
- `lib/kunit/kunit-example-test.c` ★★★ — every KUnit feature demonstrated in one readable file.
- `lib/test_*.c` (e.g. `test_bitmap.c`, `test_list_sort.c`, `test_xarray.c`) — pre-KUnit test modules; still instructive.
- `tools/testing/selftests/` — browse `gpio/`, `drivers/`, `net/`, `dma/`; `kselftest_harness.h` is worth reading in full.
- `drivers/media/test-drivers/vivid/` ★★★ — the most complete virtual device in the tree; its error-injection controls are a model for testable hardware emulation.
- `hw/misc/edu.c` and `docs/specs/edu.rst` in QEMU ★★★ — the teaching device for Lab 50.4.

**Papers**

- N. Palix, G. Thomas, S. Saha, C. Calvès, J. Lawall, G. Muller, "Faults in Linux: Ten Years Later," ASPLOS 2011 ★★★ — the empirical basis for §T.2's claim about error paths. Also introduces Coccinelle's bug-finding use.
- A. Chou, J. Yang, B. Chelf, S. Hallem, D. Engler, "An Empirical Study of Operating System Errors," SOSP 2001 — the original version; driver code has 3–7× the bug rate of the rest of the kernel.
- D. Engler et al., "Bugs as Deviant Behavior," SOSP 2001 — the static-analysis technique behind much of what smatch does.
- M. Zalewski, "American Fuzzy Lop" technical whitepaper, and D. Vyukov's syzkaller design documents — coverage-guided fuzzing.
- K. Serebryany et al., "AddressSanitizer: A Fast Address Sanity Checker," USENIX ATC 2012 — the shadow-memory design KASAN implements.
- J. Corbet's "Kernel regression tracking" writeups and the `regzbot` documentation — the process side.
- A. Zeller, *Why Programs Fail: A Guide to Systematic Debugging*, 2nd ed. — the defect→infection→failure model from Ch. 06, and delta debugging, both directly applicable.

**Books**

- Gerard Meszaros, *xUnit Test Patterns* — the vocabulary (mock/stub/fake/spy, fixtures, test smells) that makes design discussions precise.
- Michael Feathers, *Working Effectively with Legacy Code* ★★★ — directly addresses drill 13: how to add tests to code that was not designed for them. The "seams" concept is exactly §T.4's extract-pure-functions argument.
- Titus Winters et al., *Software Engineering at Google*, chapters 11–14 — testing at scale, flakiness, and the economics of §T.9.

**LWN**

- "KUnit: unit testing for the Linux kernel" (2019) and the follow-up series
- "Fault injection" coverage and `fail_function` introduction
- "Testing kernel code with KUnit and kselftest: when to use which"
- "syzbot: automated kernel fuzzing" and "Syzbot: reproducing and fixing" ★★★
- "The kernel test robot" / 0-day infrastructure descriptions
- Annual "Testing and fuzzing" microconference summaries from Linux Plumbers ★★★ — the state of the art, every year

**Infrastructure**

- **KernelCI** (`kernelci.org`) — public build/boot results across hundreds of boards
- **syzbot** (`syzkaller.appspot.com`) — live fuzzing results, with reproducers and bisections
- **0-day test robot** — posts to LKML; subscribe to its findings on your patches
- **LAVA** (`lavasoftware.org`) — board farm automation
- **LTP** (`github.com/linux-test-project/ltp`) — the Linux Test Project; system-level coverage
- **blktests**, **xfstests/fstests** — storage conformance (Part 3)
- **igt-gpu-tools**, **v4l2-compliance**, **v4l-utils** — subsystem conformance (Ch. 47)

**Tools quick reference**

```sh
./scripts/checkpatch.pl --strict -f drivers/mydev/mydev.c
make C=2 drivers/mydev/                        # sparse
make CHECK="smatch -p=kernel" C=1 drivers/mydev/
make coccicheck MODE=report M=drivers/mydev/
make CC=clang analyzer                          # clang static analyzer
./tools/testing/kunit/kunit.py run --raw_output
make kselftest TARGETS=drivers/gpio
make dt_binding_check DT_SCHEMA_FILES=vendor,mydev.yaml
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/fail_function/...
qemu-system-x86_64 -d guest_errors,unimp -device lab-testdev ...
```

---

## Part 2 completion checkpoint

You have now covered device drivers end to end: the driver model and binding (26–28), the device-node interfaces (29–30), firmware description (31–33), hardware access (34–36), the major buses (37–42), the SoC resource subsystems (43–44), the human- and network-facing classes (45–47), power (48), coprocessors (49), and testing (50).

Before moving to Part 3, verify you can do each of these **without reference**:

- [ ] Explain why probe ordering is undecidable and how `-EPROBE_DEFER` resolves it (Ch. 27).
- [ ] Write a `probe()` that acquires regulators, clocks, resets, and a power domain in the correct order, with correct error handling (Ch. 43 §T.10).
- [ ] State the three address spaces of DMA and why `virt_to_phys()` is never a DMA address (Ch. 35).
- [ ] Describe the ownership protocol for a descriptor ring and place the two barriers correctly (Ch. 46 §T.3).
- [ ] Explain why NAPI cannot livelock (Ch. 46 §T.2).
- [ ] Explain "check may fail, commit may not" and name three subsystems that implement it (Ch. 43, 47).
- [ ] Write the complete teardown order for a driver with a running poll loop, an attached eBPF program, and in-flight DMA (Ch. 25 P12 + Ch. 46).
- [ ] Explain what `pm_runtime_resume_and_get()` fixes and why (Ch. 48 §T.3).
- [ ] Name the one counter that explains each class of packet drop (Ch. 46, Lab 6).
- [ ] Take any driver and say which 60 % of it could be tested without hardware (Ch. 50 §T.2).

**Part 3 (Chapters 51–70) is storage and filesystems**: the block layer, blk-mq, I/O schedulers, NVMe, device mapper, the VFS, page cache, journalling, ext4, XFS, Btrfs, writing a filesystem, and FUSE. It leans heavily on Ch. 11 (memory), Ch. 22–23 (MM), Ch. 25 (sync patterns), and Ch. 35 (DMA) — and on the testing discipline you just built, because storage is the one subsystem where a bug destroys data rather than just a session.

---

→ Next: [../part3-storage/51-storage-overview.md](../part3-storage/51-storage-overview.md)

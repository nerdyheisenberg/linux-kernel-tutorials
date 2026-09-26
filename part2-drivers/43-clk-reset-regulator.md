# Chapter 43 — Clocks, Resets, Regulators, and Power Domains

> **Goal:** master the four "shared provider" subsystems. They look like four unrelated APIs
> and they are actually one pattern applied four times — and getting that pattern wrong is
> how you brown-out a board or hang a bus.

---

## Theory & First Principles

### T.0 — Start here: who is allowed to turn this off?

A UART needs a clock and a power rail. Your driver is done with it. Turn them off:

```c
clk_disable(uart_clk);
regulator_disable(vdd_1v8);
```

**And the I²C controller stops working**, because it shared that clock. And the display goes
black, because it shared that rail. **You had no way to know.**

```
       PLL (1200 MHz)
          │
     ┌────┼──────────┐
   div/4  div/8      div/2
     │      │          │
   UART0  ┌─┴───┐    DISPLAY
          │     │
        I2C0  SPI0          <- three consumers, one parent divider
```

**This is a shared-resource problem with a reference count**, which is exactly the shape of
Ch. 12. And it appears **four times** in every SoC, with the same structure each time:

| Subsystem | Resource | Provider | Consumer asks for |
|---|---|---|---|
| **clk** | a clock signal | PLLs, dividers, gates | enable + a *rate* |
| **regulator** | a power rail | PMIC outputs, LDOs | enable + a *voltage* |
| **reset** | a reset line | reset controllers | assert / deassert |
| **power domain** | a power island | the SoC's PM controller | (implicit, via runtime PM) |

> **One pattern, four subsystems: a provider/consumer framework with refcounting, a tree of
> dependencies, and a rate/voltage negotiation on top.** Learn it once and you know all four.

**The refcount half is straightforward:**

```c
clk_prepare_enable(clk);      /* refcount 0->1: actually ungate. 1->2: just count. */
...
clk_disable_unprepare(clk);   /* refcount 1->0: gate it. Others' counts protect them. */
```

and it recursively enables parents, so enabling a leaf clock enables its divider and its PLL.

**The rate half is where it gets interesting**, because a rate change affects *other people*:

```c
clk_set_rate(uart_clk, 24000000);
/* This may require changing a PARENT divider, which changes the rate seen by
   I2C0 and SPI0 -- who are actively using it. */
```

So the clock framework uses a **three-phase notification protocol**, and it is the same
validate-then-apply shape you will meet in KMS atomic modesetting (Ch. 47) and V4L2's
`TRY_FMT`:

```
  PRE_RATE_CHANGE   -> every consumer may VETO. Nothing has changed yet.
  (apply)           -> only if nobody objected
  POST_RATE_CHANGE  -> consumers reconfigure themselves
  ABORT_RATE_CHANGE -> if a later step failed, unwind
```

> **Validate everything, commit nothing until all checks pass.** (Ch. 89 §T.1 #3.) It turns
> "partial failure" from a reachable state into an impossible one — which for a shared clock
> tree is the difference between a failed `clk_set_rate()` and a hung machine.

**Two practical rules that follow:**

1. **`prepare` may sleep; `enable` may not.** That is why they are separate calls. A PLL takes
   milliseconds to lock (sleep), but an I²C-attached clock's gate must be togglable from
   atomic context. Same split as Ch. 42's `_cansleep`.
2. **`regulator_get()` on a rail with no driver returns a *dummy* regulator that silently
   succeeds.** This is deliberate — so a driver works on boards where the rail is always on —
   and it means "my regulator calls succeed but nothing happens" is a normal, confusing
   bring-up state. Check `regulator_summary`.

```bash
cat /sys/kernel/debug/clk/clk_summary          # the whole tree, rates, refcounts
cat /sys/kernel/debug/regulator/regulator_summary
cat /sys/kernel/debug/pm_genpd/pm_genpd_summary
```

---

### T.1 One pattern, four subsystems

A clock, a regulator, a reset line, and a power domain are all instances of the same problem:

> **A single hardware resource is shared by many consumers, each of which independently needs
> it "on". The resource must be enabled when *any* consumer needs it and disabled only when
> *no* consumer does — and no consumer may know about the others.**

That is **reference counting over a shared provider** (Ch. 12), with a provider/consumer split
(Ch. 26 T.3's two-taxonomy idea applied to resources rather than devices):

```
   Consumers                     Provider
   ┌──────────┐ clk_get("apb")
   │ UART     │────────┐
   ├──────────┤        ├──▶ ┌─────────────────┐
   │ SPI      │────────┤    │ APB bus clock   │  enable_count = 3
   ├──────────┤        │    │ (a gate in the  │  → gate is OPEN
   │ I²C      │────────┘    │  clock tree)    │
   └──────────┘             └─────────────────┘
```

Every one of the four APIs has the same shape, and recognizing it means you learn one thing,
not four:

| Subsystem | Acquire | Enable | Disable | Release |
|---|---|---|---|---|
| **clk** | `clk_get(dev, "apb")` | `clk_prepare_enable()` | `clk_disable_unprepare()` | `clk_put()` |
| **regulator** | `regulator_get(dev, "vdd")` | `regulator_enable()` | `regulator_disable()` | `regulator_put()` |
| **reset** | `reset_control_get(dev, NULL)` | `reset_control_deassert()` | `reset_control_assert()` | `reset_control_put()` |
| **genpd** | (implicit, via DT) | `pm_runtime_get_sync()` | `pm_runtime_put()` | — |

And the crucial consumer-side discipline follows directly:

> **Every `enable` must be matched by exactly one `disable`, and a consumer must never assume
> the resource is off just because it disabled it** — someone else may still hold it on.

The classic bug is a driver that calls `regulator_disable()` in an error path after an
`enable` that failed, driving the refcount negative and disabling a rail other devices need.
The kernel WARNs loudly: `unbalanced disables for vdd`.

### T.2 The clock tree is a DAG with rate propagation

An SoC's clock topology is a directed acyclic graph rooted at oscillators:

```
   osc24M (fixed, 24 MHz)
     ├─▶ PLL_CPU  (÷/× configurable)
     │     └─▶ mux ─▶ cpu_clk ─▶ divider ─▶ axi_clk
     ├─▶ PLL_PERIPH0 (600 MHz)
     │     ├─▶ ahb_div ─▶ ahb_clk ─┬─▶ gate ─▶ bus_uart0
     │     │                       ├─▶ gate ─▶ bus_spi0
     │     │                       └─▶ gate ─▶ bus_i2c0
     │     └─▶ mmc_mux ─▶ mmc_div ─▶ gate ─▶ mmc0_clk
     └─▶ PLL_VIDEO ─▶ ...
```

Five primitive types compose the whole thing, and `drivers/clk/clk-*.c` provides all five:

| Type | `clk_hw` provider | Rate behaviour |
|---|---|---|
| **fixed-rate** | `clk-fixed-rate.c` | constant |
| **gate** | `clk-gate.c` | rate = parent's |
| **divider** | `clk-divider.c` | rate = parent / N |
| **mux** | `clk-mux.c` | rate = selected parent's |
| **PLL / composite** | SoC-specific | rate = f(parent, config) |

**Rate propagation is the hard part.** When a driver calls `clk_set_rate(mmc_clk, 50000000)`,
what should happen?

- Adjust only the local divider? (`mmc_div`) — cheap, but may not reach the target.
- Change the parent PLL? — reaches any rate, **but every other consumer of that PLL changes
  rate underneath them**.

The kernel makes this an explicit, per-clock policy flag set by the *provider*:

```c
CLK_SET_RATE_PARENT     /* ★ propagate a rate request up to the parent */
CLK_SET_RATE_GATE       /* rate may only change while the clock is DISABLED */
CLK_SET_PARENT_GATE     /* parent may only be switched while disabled */
CLK_SET_RATE_NO_REPARENT/* don't switch parents to achieve a rate */
CLK_GET_RATE_NOCACHE    /* always read from hardware (rate changes behind our back) */
CLK_IGNORE_UNUSED       /* don't gate at boot even if nobody claimed it */
CLK_IS_CRITICAL         /* ★ never disable — the system dies (DRAM, CPU) */
CLK_OPS_PARENT_ENABLE   /* parent must be on to touch our registers */
```

`CLK_IS_CRITICAL` deserves note: at late boot the clock core **disables every clock nobody
claimed** (`clk_disable_unused()`), which is a superb way to find missing `clk_get()` calls —
and a superb way to kill a board if the DRAM controller's clock is unmarked. That single flag
is the difference between "efficient" and "hangs at `Freeing unused kernel memory`".

```bash
sudo cat /sys/kernel/debug/clk/clk_summary       # ★★ THE clock-debugging file
#    clock              enable  prepare  protect  rate     accuracy  phase  duty  hardware
#    osc24M                  1        1        0  24000000     0        0    50/50   Y
#     pll-periph0            1        1        0 600000000     0        0    50/50   Y
#      ahb                   1        1        0 200000000 ...
#       bus-uart0            1        1        0 200000000 ...
sudo cat /sys/kernel/debug/clk/clk_dump          # JSON
sudo cat /sys/kernel/debug/clk/bus-uart0/{clk_rate,clk_enable_count,clk_prepare_count,clk_flags}
sudo cat /sys/kernel/debug/clk/bus-uart0/clk_parent
```

**`clk_summary` is the single most useful file in embedded bring-up.** It shows the entire
tree, every rate, and every refcount. Learn to read it fluently.

### T.3 `prepare` vs `enable`: a genuinely important API split

```c
clk_prepare(clk);      /* ★ MAY SLEEP */
clk_enable(clk);       /* ★ MUST NOT SLEEP — safe from atomic context */
clk_disable(clk);
clk_unprepare(clk);

clk_prepare_enable(clk);       /* the common combination */
clk_disable_unprepare(clk);
```

Why two stages? Because clocks live in two very different places:

- A **memory-mapped SoC gate** is one register write: fast, non-sleeping.
- A **clock on an I²C PMIC** requires an I²C transaction: hundreds of microseconds, sleeps
  (Ch. 40 T.5).
- A **PLL** must be programmed, then *waited on* until it locks — tens of microseconds to
  milliseconds of sleeping.

If there were one `clk_enable()` that might sleep, no driver could enable a clock from an
interrupt handler. If there were one that must not sleep, PLLs and I²C clocks would be
impossible.

So the API splits the work: **`prepare` does everything slow (PLL lock, I²C, power-up);
`enable` does only the final ungate, which is always a fast register write.** A driver can
then `clk_prepare()` at probe time and `clk_enable()`/`clk_disable()` from a hot path or an
atomic context.

This is a reusable design idea: **when an operation has a slow setup phase and a fast trigger
phase, split the API so callers can pay the slow cost once.** The same shape appears in
`dma_map` vs `dma_sync` (Ch. 35), `pm_runtime_get` vs the hot path, and `request_irq` vs
`enable_irq`.

The managed forms handle the common case in one line:

```c
clk = devm_clk_get_enabled(dev, "apb");             /* ★ get + prepare_enable + auto-undo */
clk = devm_clk_get_optional_enabled(dev, "mod");    /* NULL if absent */
ret = devm_clk_bulk_get_all_enabled(dev, &clks);    /* ★ every clock in the DT node */
```

### T.4 Regulators: constraints as a safety system

A regulator supplies voltage. The dangerous part is that **the wrong voltage destroys
silicon**, so the subsystem is built around a *constraint* system rather than raw control.

```dts
&pmic {
	regulators {
		vdd_core: dcdc1 {
			regulator-name = "vdd-core";
			regulator-min-microvolt = <900000>;     /* ★ the BOARD's safe range */
			regulator-max-microvolt = <1200000>;
			regulator-always-on;                    /* never turn off */
			regulator-boot-on;                      /* it is already on at boot */
		};
		vdd_sensor: ldo3 {
			regulator-name = "vdd-sensor";
			regulator-min-microvolt = <3300000>;
			regulator-max-microvolt = <3300000>;    /* ★ fixed: no voltage changes */
		};
	};
};

&sensor {
	vdd-supply = <&vdd_sensor>;      /* ★ "<name>-supply" is the binding convention */
	vddio-supply = <&vdd_io>;
};
```

The constraints are **board knowledge**, expressed once in the DT, and the core **clamps and
rejects** requests outside them. A driver asking for 5 V on a rail constrained to 3.3 V gets
`-EINVAL`, not a dead board. That is Ch. 32 T.8's "the DT describes hardware" principle doing
real safety work.

```c
priv->vdd = devm_regulator_get(dev, "vdd");     /* looks up "vdd-supply" */
if (IS_ERR(priv->vdd))
	return dev_err_probe(dev, PTR_ERR(priv->vdd), "vdd\n");

ret = regulator_set_voltage(priv->vdd, 1800000, 1800000);  /* min, max in µV */
ret = regulator_enable(priv->vdd);
...
regulator_disable(priv->vdd);

/* Managed forms — strongly preferred */
priv->vdd = devm_regulator_get_enable(dev, "vdd");            /* enable + auto-disable */
priv->vdd = devm_regulator_get_enable_optional(dev, "vddio");
ret = devm_regulator_bulk_get_enable(dev, n, names);
```

**The dummy regulator** is an important piece of pragmatism. If a driver asks for `vdd-supply`
and the DT does not provide one, `regulator_get()` returns a **dummy** that always succeeds:

```
mydev: supply vdd not found, using dummy regulator
```

This exists because on many boards a rail is hardwired always-on and nobody bothered to
describe it. The driver's code works either way. It is also a trap: that message means
"nobody is actually controlling this rail" — on a board where the rail *is* switchable, it is
a bug. **Treat "using dummy regulator" as a warning to investigate, not noise.**
(`regulator_get_optional()` returns `-ENODEV` instead, for drivers that must know.)

```bash
sudo cat /sys/kernel/debug/regulator/regulator_summary    # ★ the regulator equivalent of clk_summary
ls /sys/class/regulator/
for r in /sys/class/regulator/regulator.*/; do
  printf "%-20s %-10s %8s uV  users=%s\n" "$(cat $r/name)" "$(cat $r/state)" \
    "$(cat $r/microvolts 2>/dev/null)" "$(cat $r/num_users)"
done
```

### T.5 Resets: shared vs exclusive is a correctness distinction

```c
rst = devm_reset_control_get_exclusive(dev, NULL);    /* ★ only I may reset this */
rst = devm_reset_control_get_shared(dev, NULL);       /* several devices share the line */
rst = devm_reset_control_get_optional_exclusive(dev, NULL);

reset_control_assert(rst);       /* hold in reset */
reset_control_deassert(rst);     /* release */
reset_control_reset(rst);        /* a pulse: assert, wait, deassert */

/* Managed, the common case: */
rst = devm_reset_control_get_exclusive_deasserted(dev, NULL);
```

The exclusive/shared distinction is not bookkeeping — it changes semantics:

- **Exclusive**: `assert()` takes effect immediately. Only one consumer may hold the control.
  Correct when the reset line goes to exactly one block.
- **Shared**: `assert()` is **reference-counted** — the line is asserted only when *every*
  sharer has asserted. Correct when one reset line resets a group of blocks. A shared reset
  control **cannot** be used with `reset_control_reset()` pulses, because a pulse would reset
  the other consumers mid-operation.

Getting this backwards produces a device that resets its neighbours. The core enforces it:
requesting exclusive access to a line someone else shares returns `-EBUSY`.

### T.6 Power domains: making it hierarchical

A **power domain** (`genpd`) is a switchable power island containing one or more devices. It
is the same refcount pattern, but with two additions that make it structurally different:

1. **It hooks into runtime PM.** A device's `pm_runtime_get()` powers its domain on;
   `pm_runtime_put()` may power it off. The driver does not call domain APIs at all.
2. **Domains nest.** A subdomain can only be on if its parent is on, and the core maintains
   that invariant across the whole tree.

```dts
pd_video: power-domain@1 {
	#power-domain-cells = <0>;
	power-domains = <&pd_media>;     /* ★ a SUBdomain of pd_media */
};

&vpu {
	power-domains = <&pd_video>;     /* the consumer just names its domain */
};
```

```c
/* The driver's entire interaction with power domains: */
pm_runtime_enable(dev);
...
ret = pm_runtime_resume_and_get(dev);    /* ★ powers the domain on, if needed */
do_work();
pm_runtime_put_autosuspend(dev);          /* may power it off */
```

That is the whole point: **the driver expresses *need*, not *mechanism*.** Ch. 48 covers
runtime PM properly; what matters here is that genpd is where "this device needs power" turns
into "assert these clocks, these regulators, and this power switch, in this order."

A genpd provider implements exactly two callbacks:
```c
static int mydomain_power_on(struct generic_pm_domain *gpd)  { ... }
static int mydomain_power_off(struct generic_pm_domain *gpd) { ... }
pm_genpd_init(&pd->genpd, NULL, true /* is_off */);
of_genpd_add_provider_simple(np, &pd->genpd);
```

```bash
sudo cat /sys/kernel/debug/pm_genpd/pm_genpd_summary    # ★ every domain, state, and device
#    domain      status    children    performance
#    pd_media    on                    0
#      /device/  active
sudo ls /sys/kernel/debug/pm_genpd/
cat /sys/devices/.../power/runtime_status
```

### T.7 OPP and DVFS: tying clock and voltage together

Clock frequency and supply voltage are **not independent**. Running a core at 1.4 GHz
requires more voltage than 600 MHz, and raising the frequency *before* the voltage browns out
the silicon. The **Operating Performance Point (OPP)** framework encodes the valid
(frequency, voltage) pairs and enforces the ordering:

```dts
cpu_opp_table: opp-table {
	compatible = "operating-points-v2";
	opp-shared;

	opp-600000000 {
		opp-hz = /bits/ 64 <600000000>;
		opp-microvolt = <1000000 1000000 1100000>;   /* target, min, max */
		opp-supported-hw = <0x1>;
	};
	opp-1200000000 {
		opp-hz = /bits/ 64 <1200000000>;
		opp-microvolt = <1200000 1200000 1300000>;
		clock-latency-ns = <244144>;
	};
};

&cpu0 {
	operating-points-v2 = <&cpu_opp_table>;
	clocks = <&ccu CLK_CPU>;
	cpu-supply = <&vdd_cpu>;
};
```

```c
ret = devm_pm_opp_of_add_table(dev);
ret = dev_pm_opp_set_rate(dev, target_hz);     /* ★ handles voltage ordering for you */
```

`dev_pm_opp_set_rate()` implements the rule: **when increasing frequency, raise voltage
first; when decreasing, lower frequency first.** Getting that order wrong is a
silicon-destroying bug, and it is exactly the kind of cross-subsystem invariant that belongs
in shared infrastructure rather than in every driver.

```bash
sudo cat /sys/kernel/debug/opp/*/opp_table 2>/dev/null
cat /sys/devices/system/cpu/cpu0/cpufreq/{scaling_available_frequencies,scaling_cur_freq}
cat /sys/class/devfreq/*/available_frequencies 2>/dev/null
```

### T.8 The acquisition order, and why it matters

There is a **correct order** for bringing a device up, and it is dictated by physics:

```c
static int my_probe(struct platform_device *pdev)
{
	/* 1. POWER first — nothing else works without it */
	priv->vdd = devm_regulator_get_enable(dev, "vdd");

	/* 2. CLOCKS — registers are often unreadable without the bus clock */
	priv->clk = devm_clk_get_enabled(dev, "apb");

	/* 3. Release RESET — only once clocked and powered */
	priv->rst = devm_reset_control_get_exclusive_deasserted(dev, NULL);

	/* 4. Now MMIO is valid */
	priv->base = devm_platform_ioremap_resource(pdev, 0);
	id = readl(priv->base + REG_ID);          /* ★ reads garbage if 1-3 are wrong */

	/* 5. Hardware init, then IRQ, then publish (Ch. 31 T.7) */
}
```

And teardown is the **exact reverse** — which `devres` (Ch. 28 T.4) gives you for free,
provided you acquire in this order and use `devm_` throughout.

**The classic bring-up failure:** `readl()` returns `0xFFFFFFFF` or `0x00000000` and the
driver concludes the device is absent. Nine times out of ten the bus clock is gated or the
block is in reset. Check `clk_summary` before suspecting the hardware.

---

## 1. Internals

### 1.1 Source map

```
drivers/clk/clk.c                ★★ the core: tree, rates, prepare/enable refcounts
drivers/clk/clk-divider.c, clk-gate.c, clk-mux.c, clk-fixed-rate.c, clk-composite.c ★
drivers/clk/clk-devres.c         devm_clk_get_enabled and friends
drivers/clk/sunxi-ng/, qcom/, imx/, samsung/   real SoC clock drivers
include/linux/clk.h              ★★ the consumer API — read the comments
include/linux/clk-provider.h     the provider API
drivers/regulator/core.c         ★★ constraints, refcounts, the dummy regulator
drivers/regulator/devres.c, fixed.c, of_regulator.c
drivers/reset/core.c             ★ shared vs exclusive (T.5)
drivers/base/power/domain.c      ★★ genpd
drivers/opp/core.c, of.c         ★ OPP/DVFS (T.7)
drivers/cpufreq/cpufreq-dt.c     the generic DT cpufreq driver
drivers/devfreq/                 DVFS for non-CPU devices
Documentation/driver-api/clk.rst ★★
Documentation/power/regulator/   ★ overview.rst, consumer.rst, machine.rst, regulator.rst
Documentation/driver-api/reset.rst ★
Documentation/devicetree/bindings/clock/, regulator/, reset/, power/, opp/
```

### 1.2 Writing a clock provider

```c
struct my_clk {
	struct clk_hw  hw;
	void __iomem  *reg;
	u8             bit;
};
#define to_my_clk(_hw) container_of(_hw, struct my_clk, hw)

static int my_clk_enable(struct clk_hw *hw)
{
	struct my_clk *c = to_my_clk(hw);

	writel(readl(c->reg) | BIT(c->bit), c->reg);
	return 0;
}
static void my_clk_disable(struct clk_hw *hw) { ... }
static int my_clk_is_enabled(struct clk_hw *hw) { ... }

static unsigned long my_clk_recalc_rate(struct clk_hw *hw, unsigned long parent_rate)
{
	return parent_rate / (FIELD_GET(DIV_MASK, readl(c->reg)) + 1);
}
static long my_clk_round_rate(struct clk_hw *hw, unsigned long rate,
			      unsigned long *parent_rate)
{
	/* ★ return the CLOSEST ACHIEVABLE rate; may adjust *parent_rate if
	 * CLK_SET_RATE_PARENT is set */
}
static int my_clk_set_rate(struct clk_hw *hw, unsigned long rate,
			   unsigned long parent_rate) { ... }

static const struct clk_ops my_clk_ops = {
	.prepare      = my_clk_prepare,      /* ★ may sleep: PLL lock, I²C */
	.unprepare    = my_clk_unprepare,
	.enable       = my_clk_enable,       /* ★ must NOT sleep */
	.disable      = my_clk_disable,
	.is_enabled   = my_clk_is_enabled,
	.recalc_rate  = my_clk_recalc_rate,
	.round_rate   = my_clk_round_rate,
	.determine_rate = my_clk_determine_rate,   /* modern replacement for round_rate */
	.set_rate     = my_clk_set_rate,
	.set_parent   = my_clk_set_parent,
	.get_parent   = my_clk_get_parent,
};

static int my_clk_probe(struct platform_device *pdev)
{
	struct clk_init_data init = {
		.name         = "my-gate",
		.ops          = &my_clk_ops,
		.parent_names = (const char *[]){ "ahb" },
		.num_parents  = 1,
		.flags        = CLK_SET_RATE_PARENT,
	};
	struct my_clk *c = devm_kzalloc(dev, sizeof(*c), GFP_KERNEL);

	c->hw.init = &init;
	ret = devm_clk_hw_register(dev, &c->hw);
	if (ret)
		return ret;
	return devm_of_clk_add_hw_provider(dev, of_clk_hw_simple_get, &c->hw);
}
```

`of_clk_hw_onecell_get` is the multi-clock version and implements the `#clock-cells = <1>`
specifier translation from Ch. 32 T.5.

---

## 2. Practice

### Lab 43.1 — Read your board's clock tree

```bash
sudo cat /sys/kernel/debug/clk/clk_summary
# Render it as a tree with rates:
sudo awk 'NR>2 {
  match($0, /^ */); ind = RLENGTH;
  printf "%*s%-24s %10s Hz  en=%s prep=%s\n", ind, "", $1, $5, $2, $3
}' /sys/kernel/debug/clk/clk_summary | head -60

# One clock in detail
C=/sys/kernel/debug/clk/bus-uart0
sudo cat $C/clk_rate $C/clk_enable_count $C/clk_prepare_count $C/clk_flags $C/clk_parent

# ★ Find clocks nobody claimed (candidates for CLK_IS_CRITICAL bugs or power savings)
dmesg | grep -i 'clk.*unused\|disabling unused'
sudo awk 'NR>2 && $2 == 0 && $3 == 0 {print $1}' /sys/kernel/debug/clk/clk_summary | head -20

# Trace rate changes
sudo trace-cmd record -e clk -- sleep 10
trace-cmd report | grep -E 'clk_set_rate|clk_enable|clk_disable' | head -30
sudo bpftrace -e '
tracepoint:clk:clk_set_rate    { printf("set_rate %s -> %llu\n", str(args->name), args->rate); }
tracepoint:clk:clk_enable      { @en[str(args->name)] = count(); }
tracepoint:clk:clk_disable     { @dis[str(args->name)] = count(); }'
```

### Lab 43.2 — A complete consumer with all four resources

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * powerdemo — demonstrates the correct acquisition order (T.8) and the
 * refcounted-provider discipline (T.1).
 */
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

#include <linux/clk.h>
#include <linux/io.h>
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/pm_opp.h>
#include <linux/pm_runtime.h>
#include <linux/regulator/consumer.h>
#include <linux/reset.h>

#define REG_ID     0x00
#define REG_CTRL   0x04
#define  CTRL_EN   BIT(0)

struct powerdemo {
	struct device        *dev;
	void __iomem         *base;
	struct clk           *bus_clk;
	struct clk           *mod_clk;
	struct clk_bulk_data *extra_clks;
	int                   num_extra;
	struct regulator     *vdd;
	struct reset_control *rst;
	unsigned long         cur_rate;
};

static ssize_t rate_show(struct device *dev, struct device_attribute *a, char *buf)
{
	struct powerdemo *p = dev_get_drvdata(dev);

	return sysfs_emit(buf, "%lu\n", clk_get_rate(p->mod_clk));
}

static ssize_t rate_store(struct device *dev, struct device_attribute *a,
			  const char *buf, size_t n)
{
	struct powerdemo *p = dev_get_drvdata(dev);
	unsigned long rate, rounded;
	int ret = kstrtoul(buf, 0, &rate);

	if (ret)
		return ret;

	/* ★ T.2: ask what is achievable BEFORE committing */
	rounded = clk_round_rate(p->mod_clk, rate);
	dev_info(dev, "requested %lu, achievable %lu\n", rate, rounded);

	ret = clk_set_rate(p->mod_clk, rate);
	if (ret)
		return ret;

	p->cur_rate = clk_get_rate(p->mod_clk);
	return n;
}
static DEVICE_ATTR_RW(rate);

static ssize_t voltage_show(struct device *dev, struct device_attribute *a, char *buf)
{
	struct powerdemo *p = dev_get_drvdata(dev);
	int uV = regulator_get_voltage(p->vdd);

	return uV < 0 ? uV : sysfs_emit(buf, "%d\n", uV);
}
static DEVICE_ATTR_RO(voltage);

static struct attribute *powerdemo_attrs[] = {
	&dev_attr_rate.attr, &dev_attr_voltage.attr, NULL,
};
ATTRIBUTE_GROUPS(powerdemo);

static int powerdemo_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	struct powerdemo *p;
	u32 id;
	int ret;

	p = devm_kzalloc(dev, sizeof(*p), GFP_KERNEL);
	if (!p)
		return -ENOMEM;
	p->dev = dev;
	platform_set_drvdata(pdev, p);

	/* ---- T.8 STEP 1: POWER ---- */
	p->vdd = devm_regulator_get(dev, "vdd");
	if (IS_ERR(p->vdd))
		return dev_err_probe(dev, PTR_ERR(p->vdd), "vdd supply\n");

	ret = regulator_set_voltage(p->vdd, 1800000, 1800000);
	if (ret && ret != -EPERM)      /* -EPERM: fixed regulator, already correct */
		return dev_err_probe(dev, ret, "cannot set 1.8V\n");

	ret = regulator_enable(p->vdd);
	if (ret)
		return dev_err_probe(dev, ret, "cannot enable vdd\n");
	/* ★ T.1: register the matching disable IMMEDIATELY, so every error path
	 * below unwinds correctly (Ch. 28 T.5) */
	ret = devm_add_action_or_reset(dev,
			(void (*)(void *))regulator_disable, p->vdd);
	if (ret)
		return ret;

	dev_info(dev, "vdd = %d uV\n", regulator_get_voltage(p->vdd));

	/* ---- T.8 STEP 2: CLOCKS ---- */
	p->bus_clk = devm_clk_get_enabled(dev, "bus");
	if (IS_ERR(p->bus_clk))
		return dev_err_probe(dev, PTR_ERR(p->bus_clk), "bus clock\n");

	p->mod_clk = devm_clk_get_enabled(dev, "mod");
	if (IS_ERR(p->mod_clk))
		return dev_err_probe(dev, PTR_ERR(p->mod_clk), "mod clock\n");

	/* Any remaining clocks the DT lists, in bulk */
	ret = devm_clk_bulk_get_all_enabled(dev, &p->extra_clks);
	if (ret < 0)
		return dev_err_probe(dev, ret, "bulk clocks\n");
	p->num_extra = ret;

	dev_info(dev, "bus=%lu Hz mod=%lu Hz (+%d more)\n",
		 clk_get_rate(p->bus_clk), clk_get_rate(p->mod_clk), p->num_extra);

	/* ---- T.8 STEP 3: RESET ---- */
	p->rst = devm_reset_control_get_optional_exclusive_deasserted(dev, NULL);
	if (IS_ERR(p->rst))
		return dev_err_probe(dev, PTR_ERR(p->rst), "reset\n");

	/* ---- T.8 STEP 4: only NOW is MMIO valid ---- */
	p->base = devm_platform_ioremap_resource(pdev, 0);
	if (IS_ERR(p->base))
		return PTR_ERR(p->base);

	id = readl(p->base + REG_ID);
	if (id == 0xffffffff || id == 0) {
		/* ★ The classic symptom: check clk_summary before blaming the hardware */
		return dev_err_probe(dev, -ENODEV,
				     "ID reads 0x%08x — clock gated or held in reset?\n", id);
	}
	dev_info(dev, "device ID 0x%08x\n", id);

	/* ---- OPP/DVFS (T.7) ---- */
	ret = devm_pm_opp_of_add_table(dev);
	if (ret && ret != -ENODEV)
		return dev_err_probe(dev, ret, "opp table\n");
	if (!ret) {
		unsigned long freq = ULONG_MAX;
		struct dev_pm_opp *opp = dev_pm_opp_find_freq_floor(dev, &freq);

		if (!IS_ERR(opp)) {
			dev_info(dev, "max OPP: %lu Hz @ %lu uV\n",
				 freq, dev_pm_opp_get_voltage(opp));
			dev_pm_opp_put(opp);
		}
		/* ★ set_rate handles the frequency/voltage ORDER for us */
		dev_pm_opp_set_rate(dev, freq);
	}

	/* ---- runtime PM / power domain (T.6) ---- */
	pm_runtime_set_active(dev);
	pm_runtime_set_autosuspend_delay(dev, 500);
	pm_runtime_use_autosuspend(dev);
	ret = devm_pm_runtime_enable(dev);
	if (ret)
		return ret;

	writel(CTRL_EN, p->base + REG_CTRL);
	return 0;
}

static void powerdemo_remove(struct platform_device *pdev)
{
	struct powerdemo *p = platform_get_drvdata(pdev);

	writel(0, p->base + REG_CTRL);
	readl(p->base + REG_CTRL);       /* flush (Ch. 34 T.4) */
	/* ★ devres unwinds reset → clocks → regulator, exactly reversing T.8 */
}

static int powerdemo_runtime_suspend(struct device *dev)
{
	struct powerdemo *p = dev_get_drvdata(dev);

	clk_disable_unprepare(p->mod_clk);      /* ★ T.3: the fast path */
	return 0;
}
static int powerdemo_runtime_resume(struct device *dev)
{
	struct powerdemo *p = dev_get_drvdata(dev);

	return clk_prepare_enable(p->mod_clk);
}
static const struct dev_pm_ops powerdemo_pm = {
	RUNTIME_PM_OPS(powerdemo_runtime_suspend, powerdemo_runtime_resume, NULL)
	SYSTEM_SLEEP_PM_OPS(pm_runtime_force_suspend, pm_runtime_force_resume)
};

static const struct of_device_id powerdemo_of_ids[] = {
	{ .compatible = "acme,powerdemo" }, { }
};
MODULE_DEVICE_TABLE(of, powerdemo_of_ids);

static struct platform_driver powerdemo_driver = {
	.driver = {
		.name = "powerdemo",
		.of_match_table = powerdemo_of_ids,
		.pm = pm_ptr(&powerdemo_pm),
		.dev_groups = powerdemo_groups,
	},
	.probe  = powerdemo_probe,
	.remove = powerdemo_remove,
};
module_platform_driver(powerdemo_driver);
MODULE_LICENSE("GPL");
```

The DT:
```dts
powerdemo@10000000 {
	compatible = "acme,powerdemo";
	reg = <0x10000000 0x1000>;
	clocks = <&ccu CLK_BUS_DEMO>, <&ccu CLK_DEMO>;
	clock-names = "bus", "mod";
	resets = <&ccu RST_BUS_DEMO>;
	vdd-supply = <&reg_vdd_1v8>;
	power-domains = <&pd_periph>;
	operating-points-v2 = <&demo_opp_table>;
};
```

```bash
sudo insmod powerdemo.ko
dmesg | tail -10
cat /sys/bus/platform/devices/*powerdemo/rate
echo 100000000 | sudo tee /sys/bus/platform/devices/*powerdemo/rate
sudo cat /sys/kernel/debug/clk/clk_summary | grep -i demo
sudo cat /sys/kernel/debug/regulator/regulator_summary | grep -i -A2 vdd
```

### Lab 43.3 — Break the acquisition order (T.8)

Deliberately reorder the probe and observe each failure mode:

```c
/* (a) MMIO before the clock */
p->base = devm_platform_ioremap_resource(pdev, 0);
id = readl(p->base + REG_ID);          /* → 0xffffffff or a bus hang */
p->clk = devm_clk_get_enabled(dev, "bus");

/* (b) Clock before power */
p->clk = devm_clk_get_enabled(dev, "bus");   /* may succeed but the block is unpowered */
p->vdd = devm_regulator_get_enable(dev, "vdd");

/* (c) MMIO while held in reset */
p->rst = devm_reset_control_get_exclusive(dev, NULL);
reset_control_assert(p->rst);
id = readl(p->base + REG_ID);          /* → 0 or garbage */
```
```bash
dmesg | grep -i 'powerdemo'
sudo cat /sys/kernel/debug/clk/clk_summary | grep -i demo   # ★ check this FIRST
# On some SoCs, MMIO to a gated block hangs the bus entirely:
dmesg | grep -i 'imprecise external abort\|SError\|bus error\|serror'
```
**Write down each symptom.** Recognizing "reads 0xffffffff" → check the clock, and
"reads 0x00000000" → check the reset, will save you days in Part 7.

### Lab 43.4 — Refcount discipline and unbalanced disables (T.1)

```c
/* ★ The classic bug */
ret = regulator_enable(p->vdd);
if (ret)
	goto err;
ret = clk_prepare_enable(p->clk);
if (ret)
	goto err_reg;
...
err_reg:
	regulator_disable(p->vdd);
err:
	regulator_disable(p->vdd);      /* ★ DOUBLE DISABLE on the first path */
	return ret;
```
```bash
sudo insmod buggy.ko
dmesg | grep -i 'unbalanced\|WARNING.*regulator'
#   WARNING: ... _regulator_disable+0x...
#   unbalanced disables for vdd-1v8

# Watch the refcounts move:
watch -n1 'for r in /sys/class/regulator/regulator.*/; do
  printf "%-16s %-8s users=%s\n" "$(cat $r/name)" "$(cat $r/state)" "$(cat $r/num_users)"; done'

sudo bpftrace -e '
kprobe:regulator_enable  { @en[str(((struct regulator *)arg0)->supply_name)] = count(); }
kprobe:regulator_disable { @dis[str(((struct regulator *)arg0)->supply_name)] = count(); }'
```
Then fix it with `devm_add_action_or_reset()` (Ch. 28 T.5) and show every error path is
correct by construction.

### Lab 43.5 — Rate propagation and `CLK_SET_RATE_PARENT` (T.2)

```bash
# Find a clock with CLK_SET_RATE_PARENT set
sudo grep -l . /sys/kernel/debug/clk/*/clk_flags 2>/dev/null | while read f; do
  grep -q 'CLK_SET_RATE_PARENT' "$f" && echo "$(dirname $f)"
done | head

# Set a rate and watch what else changed
sudo cp /sys/kernel/debug/clk/clk_summary /tmp/before
echo 100000000 | sudo tee /sys/bus/platform/devices/*powerdemo/rate
sudo cp /sys/kernel/debug/clk/clk_summary /tmp/after
diff /tmp/before /tmp/after          # ★ how many clocks moved?

sudo trace-cmd record -e clk:clk_set_rate -e clk:clk_set_parent -- \
  sh -c 'echo 100000000 > /sys/bus/platform/devices/*powerdemo/rate'
trace-cmd report

# The notifier mechanism that lets consumers react:
grep -n -A20 'clk_notifier_register' drivers/clk/clk.c | head -30
grep -rn 'clk_notifier_register' drivers/ | head
```
**The diff is the point:** with `CLK_SET_RATE_PARENT`, one `clk_set_rate()` can change a PLL
and every sibling consumer's rate. Explain why `CLK_SET_RATE_GATE` exists.

### Lab 43.6 — Regulator constraints as a safety system (T.4)

```bash
# What are the constraints?
sudo cat /sys/kernel/debug/regulator/regulator_summary
for r in /sys/class/regulator/regulator.*/; do
  echo "$(cat $r/name): $(cat $r/min_microvolts 2>/dev/null)-$(cat $r/max_microvolts 2>/dev/null) uV, type=$(cat $r/type)"
done
```
```c
/* Try to violate them */
ret = regulator_set_voltage(p->vdd, 5000000, 5000000);   /* 5V on a 1.8V rail */
dev_info(dev, "set 5V returned %d (expect -EINVAL)\n", ret);

/* And check the valid-range query API */
ret = regulator_is_supported_voltage(p->vdd, 1700000, 1900000);
dev_info(dev, "1.7-1.9V supported: %d\n", ret);
n = regulator_count_voltages(p->vdd);
for (i = 0; i < n; i++)
	dev_info(dev, "  sel %d = %d uV\n", i, regulator_list_voltage(p->vdd, i));
```
```bash
# The dummy-regulator trap:
dmesg | grep -i 'using dummy regulator'
# ★ For each hit, determine whether the rail is genuinely fixed or the DT is incomplete.
grep -rn 'vdd-supply\|vcc-supply' arch/*/boot/dts/ | head -10
```

### Lab 43.7 — Power domains and runtime PM (T.6)

```bash
sudo cat /sys/kernel/debug/pm_genpd/pm_genpd_summary
sudo ls /sys/kernel/debug/pm_genpd/*/

# Watch a domain power down when its last consumer idles
D=/sys/bus/platform/devices/*powerdemo
cat $D/power/{runtime_status,runtime_active_time,runtime_suspended_time,autosuspend_delay_ms}
echo 0 | sudo tee $D/power/autosuspend_delay_ms
watch -n1 "cat $D/power/runtime_status; sudo grep -A3 pd_periph /sys/kernel/debug/pm_genpd/pm_genpd_summary"

# Force it on/off from userspace:
echo on   | sudo tee $D/power/control     # ★ disables runtime PM: stays powered
echo auto | sudo tee $D/power/control

sudo bpftrace -e '
kprobe:genpd_power_on  { printf("domain ON\n"); }
kprobe:genpd_power_off { printf("domain OFF\n"); }'
sudo trace-cmd record -e power -e runtime_pm -- sleep 10
trace-cmd report | head -30
```

### Lab 43.8 — OPP and DVFS ordering (T.7)

```bash
# The CPU's OPP table
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies
sudo cat /sys/kernel/debug/opp/*/opp_table 2>/dev/null

# Force a frequency change and watch the voltage follow
echo userspace | sudo tee /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
for f in $(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies); do
  echo $f | sudo tee /sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed >/dev/null
  sleep 1
  printf "%10s Hz -> vdd_cpu = %s uV\n" "$f" \
    "$(cat /sys/class/regulator/*/microvolts 2>/dev/null | head -1)"
done

# ★ Prove the ordering rule
sudo trace-cmd record -e power:cpu_frequency -e regulator -- \
  sh -c 'echo 1200000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed; sleep 1;
         echo 600000  > /sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed'
trace-cmd report | grep -E 'cpu_frequency|regulator_set_voltage'
# Going UP:   regulator_set_voltage BEFORE cpu_frequency
# Going DOWN: cpu_frequency BEFORE regulator_set_voltage
$EDITOR drivers/opp/core.c      # _set_opp() — the ordering logic
```

### Lab 43.9 — Write a clock provider

Implement a gate + divider pair over `gpio-sim`-backed fake registers, register it with
`#clock-cells = <1>`, and consume it from Lab 43.2's driver.

```c
static const struct clk_ops fake_gate_ops = {
	.enable     = fake_gate_enable,
	.disable    = fake_gate_disable,
	.is_enabled = fake_gate_is_enabled,
};

static struct clk_hw *fake_of_clk_get(struct of_phandle_args *spec, void *data)
{
	struct fake_ccu *ccu = data;
	unsigned int idx = spec->args[0];      /* ★ Ch. 32 T.5's specifier */

	if (idx >= ccu->nr_clks)
		return ERR_PTR(-EINVAL);
	return &ccu->clks[idx].hw;
}
of_clk_add_hw_provider(np, fake_of_clk_get, ccu);
```
```bash
sudo insmod fakeccu.ko
sudo cat /sys/kernel/debug/clk/clk_summary | grep -i fake
# Then consume it:  clocks = <&fakeccu 0>, <&fakeccu 1>;
```

---

## 3. Mastery drills

1. **Read `Documentation/driver-api/clk.rst`** completely — it is the maintainer's own
   explanation of T.2 and T.3. Then read `drivers/clk/clk.c`'s `clk_core_set_rate_nolock()`
   and trace how a rate request propagates.

2. **The one-pattern claim (T.1).** For each of the four subsystems, write out the acquire /
   enable / disable / release quadruple and identify where the refcount lives in the source.
   Then find a fifth subsystem in the kernel with the same shape (candidates: `phy`,
   `pinctrl`, `interconnect`, `icc`).

3. **prepare/enable.** Explain to a colleague why the split exists, with a concrete example of
   a clock that *must* sleep to enable and a caller that *cannot* sleep. Then find a driver
   that calls `clk_enable()` from an interrupt handler and verify its clock is memory-mapped.

4. **`CLK_IS_CRITICAL`.** Find every use in your SoC's clock driver. For three of them,
   explain what would break without the flag. Then boot with
   `clk_ignore_unused` and compare power draw.

5. **Rate propagation policy.** For a clock feeding both an MMC controller (needs exact rates)
   and a UART (tolerates approximation) from one divider, design the flags. What goes wrong
   with each wrong choice?

6. **Regulator constraints.** Read `drivers/regulator/of_regulator.c`. List every
   `regulator-*` DT property and what it constrains. Then find a board DT that over-constrains
   a rail and would prevent a valid DVFS point.

7. **The dummy regulator.** Find three drivers where `regulator_get()` returns a dummy on
   your board. For each, determine whether the rail is genuinely fixed. File the DT fix for
   one.

8. **Shared vs exclusive resets.** Find a DT where one reset line goes to several blocks.
   Verify every consumer uses `_shared`. Then construct the failure that occurs if one uses
   `_exclusive` (`-EBUSY`) and if one uses `reset_control_reset()` on a shared line.

9. **genpd hierarchy.** Draw your SoC's power-domain tree from
   `pm_genpd_summary`. For one subdomain, trace what `pm_runtime_get()` on a leaf device does
   all the way to the power switch.

10. **DVFS ordering.** Read `drivers/opp/core.c`'s `_set_opp()`. Write out the exact sequence
    for raising and lowering frequency, including `regulator_set_voltage_triplet()` and the
    `clk_set_rate()` placement. Explain why a driver must never implement this itself.

11. **Design question.** A camera ISP block needs: two clocks (bus at a fixed rate, pixel
    clock at 6 rates), a dedicated 1.2 V rail plus a shared 1.8 V I/O rail, a reset shared
    with the CSI receiver, its own power domain nested under a media domain, and DVFS tied to
    resolution. Write the DT node, the probe's acquisition sequence, the runtime-PM
    callbacks, and state every flag you would set on the pixel clock.

---

## 4. Further reading

**Kernel documentation:**
- `Documentation/driver-api/clk.rst` ★★ — the normative clock document
- `Documentation/power/regulator/overview.rst` ★★, `consumer.rst` ★, `machine.rst`,
  `regulator.rst`, `design.rst`
- `Documentation/driver-api/reset.rst` ★ — the shared/exclusive rationale
- `Documentation/power/pm_qos_interface.rst`, `Documentation/power/runtime_pm.rst` ★★ (Ch. 48)
- `Documentation/power/opp.rst` ★
- `Documentation/devicetree/bindings/clock/clock-bindings.yaml` ★,
  `regulator/regulator.yaml` ★, `reset/reset.txt`, `power/power-domain.yaml`,
  `opp/opp-v2-base.yaml`

**Source:**
- `drivers/clk/clk.c` ★★ — rate propagation, refcounts, the notifier chain
- `drivers/clk/clk-divider.c`, `clk-gate.c`, `clk-mux.c` ★ — the five primitives
- `drivers/clk/sunxi-ng/` ★ — a well-structured modern SoC clock driver worth reading whole
- `drivers/regulator/core.c` ★★ — constraint enforcement and the dummy regulator
- `drivers/reset/core.c` ★, `drivers/base/power/domain.c` ★★, `drivers/opp/core.c` ★
- Exemplary consumers: `drivers/mmc/host/sunxi-mmc.c`, `drivers/spi/spi-imx.c`,
  `drivers/media/platform/` (ISP drivers use all four)

**Tools & debugging:**
- `/sys/kernel/debug/clk/clk_summary` ★★, `regulator_summary` ★★, `pm_genpd_summary` ★
- `trace-cmd record -e clk -e regulator -e power -e runtime_pm`
- `powertop`, `turbostat`, `cpupower frequency-info`
- `devmem2` for raw clock/reset register inspection during bring-up

**LWN & talks:**
- "The common clock framework" (Mike Turquette's introduction)
- "Regulators and the kernel" / "The voltage regulator framework"
- "Generic power domains" / "PM domains and runtime PM"
- Mike Turquette / Stephen Boyd ELC talks on the CCF — **the maintainers explaining T.2/T.3**
- Mark Brown's talks on the regulator framework
- Bootlin's "Power management in the Linux kernel" training materials (free slides)

→ Next: [44-iio-hwmon.md](44-iio-hwmon.md)

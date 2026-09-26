# Chapter 42 — GPIO and Pin Control

> **Goal:** understand the two subsystems every SoC driver depends on and nobody explains
> properly — why a "pin" and a "GPIO" are different things, why the descriptor API replaced
> integer GPIO numbers, and why polarity belongs in the device tree and not in your driver.

---

## Theory & First Principles

### T.0 — Start here: your driver probes, and the hardware does nothing

This is the single most common bring-up symptom (Ch. 101 §T.6), and it has nothing to do
with your driver:

```
  dmesg:  mydrv 2020000.serial: probe succeeded
  reality: no bytes ever appear on the wire
```

**The pins are connected to something else.** An SoC has perhaps 200 physical pins and 800
internal functions, so each pin is a **multiplexer**:

```
            physical pin PA7
                   │
        ┌─────────┼──────────────┐
     [mux function select register]
        │    │    │    │    │
      GPIO  UART SPI  I2C  PWM         <- FIVE functions, ONE pin.
              ↑                            Exactly one is connected.
         you wanted this
```

If nobody programmed the mux, the pin is whatever reset left it as — usually GPIO input. Your
UART is transmitting into a disconnected block.

**This is why `pinctrl` exists**, and why it is separate from the driver: pin muxing is a
*board* property, not a *device* property. The same SoC on two boards routes the same UART to
different pins.

```dts
uart0: serial@2020000 {
	compatible = "vendor,uart";
	pinctrl-names = "default";
	pinctrl-0 = <&uart0_pins>;       /* ← apply this mux config before probe */
};

uart0_pins: uart0-pins {
	function = "uart0";
	groups = "uart0_data";
	bias-pull-up;                    /* also: drive strength, slew rate, open-drain */
};
```

The driver core applies the `default` state **automatically before `probe()`**, which is why
most drivers contain no pinctrl code at all — and why, when it is wrong, the driver looks
innocent.

**Now the second idea, which is the more interesting one.** A GPIO is a pin you drive or read
by hand. Almost every use of one is really a *device*:

```c
/* The 2010 way -- a global integer namespace, and a board-specific number */
gpio_request(137, "reset");        /* what is 137? Nobody knows. */
gpio_direction_output(137, 0);
msleep(10);
gpio_set_value(137, 1);            /* and is the reset ACTIVE HIGH or LOW? */
```

Every problem here is a naming problem: a flat global integer space that differs per board,
no indication of polarity, and no relationship to the device that owns the pin. The modern
API fixes all three:

```c
/* The descriptor way: named, board-independent, polarity-aware */
struct gpio_desc *rst = devm_gpiod_get(dev, "reset", GPIOD_OUT_HIGH);
gpiod_set_value_cansleep(rst, 1);   /* 1 = ASSERTED, whatever the wiring */
```

```dts
	reset-gpios = <&gpio1 9 GPIO_ACTIVE_LOW>;
	/*              ^chip ^line ^POLARITY IS IN THE DEVICE TREE  */
```

**`gpiod_set_value(desc, 1)` means "assert", not "drive high."** The polarity lives in the
hardware description where it belongs, so the driver is identical for active-high and
active-low wiring. That is Ch. 32's move — configuration out of code, into data — applied to
a single bit.

**And the trap:** `gpiod_set_value()` versus `gpiod_set_value_cansleep()`. A GPIO on an I²C
expander takes ~200 µs to change (Ch. 40 §T.0) and **sleeps**. The same driver code may drive
an SoC GPIO (instant) or an expander GPIO (sleeps), depending on the board. Using the
non-sleeping variant on an expander is a `BUG: sleeping function called from invalid
context` — which is why the API forces you to say which context you are in.

```bash
gpiodetect && gpioinfo                      # every line, its name, direction, consumer
cat /sys/kernel/debug/gpio                  # the older view
cat /sys/kernel/debug/pinctrl/*/pinmux-pins # WHAT EACH PIN IS MUXED TO  <- the one
cat /sys/kernel/debug/pinctrl/*/pinconf-pins
```

---

### T.1 A pin is not a GPIO: the multiplexing problem

An SoC has perhaps 200 physical package pins and perhaps 2000 internal functions that could
use them. The silicon therefore contains a **multiplexer** per pin, plus configuration for
electrical properties.

```
                      ┌─────────────────────────────────────┐
   physical pin ──────┤  pin controller                     │
     "PA5"            │   mux select:  0 = GPIO             │
                      │                1 = UART0_TX         │
                      │                2 = SPI0_MOSI        │
                      │                3 = PWM1             │
                      │   config:  pull-up / pull-down      │
                      │            drive strength (2–20 mA) │
                      │            open-drain / push-pull   │
                      │            slew rate, schmitt       │
                      └─────────────────────────────────────┘
```

So there are **two orthogonal questions**:

| Question | Subsystem |
|---|---|
| *Which function is this pin connected to, and with what electrical config?* | **pinctrl** |
| *If it is connected to GPIO, what is its direction and value?* | **gpiolib** |

Conflating them is the single most common confusion. GPIO is **one of the functions** a pin
can be muxed to. A pin muxed to UART TX is not a GPIO at all and has no direction or value.

This mirrors Ch. 26 T.3's two-taxonomy insight: the same physical resource viewed through two
independent classifications. And the separation has a concrete payoff — a pin controller
driver knows nothing about GPIO semantics, and a GPIO consumer knows nothing about muxing.

```bash
sudo cat /sys/kernel/debug/pinctrl/*/pins          # every pin and its current state
sudo cat /sys/kernel/debug/pinctrl/*/pinmux-pins   # ★ which pin belongs to which owner
sudo cat /sys/kernel/debug/pinctrl/*/pinconf-pins  # electrical config
sudo cat /sys/kernel/debug/pinctrl/*/pingroups
sudo cat /sys/kernel/debug/pinctrl/*/pinmux-functions
sudo cat /sys/kernel/debug/gpio                    # ★ every GPIO, direction, value, owner
```

**`pinmux-pins` and the `gpio` debugfs file are the two most valuable files in embedded
bring-up.** They answer "is this pin doing what I think?" directly.

### T.2 pinctrl: states, not calls

The pinctrl API is deliberately *declarative*. A driver does not say "mux pin 42 to function
3"; it says "put me in my `default` state", and the device tree says what that means.

```dts
&pinctrl {
	uart0_pins: uart0-pins {
		pins = "PA4", "PA5";
		function = "uart0";
		bias-pull-up;
		drive-strength = <10>;
	};

	uart0_sleep_pins: uart0-sleep-pins {
		pins = "PA4", "PA5";
		function = "gpio_in";       /* park as inputs to save power */
		bias-pull-down;
	};
};

&uart0 {
	pinctrl-names = "default", "sleep";   /* ★ well-known state names */
	pinctrl-0 = <&uart0_pins>;
	pinctrl-1 = <&uart0_sleep_pins>;
};
```

**The core applies `default` automatically before `probe()` is called** (Ch. 27 T.6's
`really_probe()` sequence) and `sleep` on suspend. A driver that only needs those two states
writes **no pinctrl code at all** — which is why you rarely see `pinctrl_select_state()` in
drivers.

Explicit use is for runtime switching:

```c
priv->pinctrl = devm_pinctrl_get(dev);
priv->state_normal   = pinctrl_lookup_state(priv->pinctrl, "default");
priv->state_recovery = pinctrl_lookup_state(priv->pinctrl, "gpio");
...
pinctrl_select_state(priv->pinctrl, priv->state_recovery);   /* e.g. I²C bus recovery */
```
That is exactly Ch. 40 T.6's `prepare_recovery` hook: re-mux the I²C pins to GPIO, bit-bang
the recovery, mux back.

Standard state names: `default`, `init`, `sleep`, `idle`. Standard config properties are in
`Documentation/devicetree/bindings/pinctrl/pincfg-node.yaml`:
`bias-pull-up`, `bias-pull-down`, `bias-disable`, `drive-open-drain`, `drive-push-pull`,
`drive-strength`, `input-enable`, `input-schmitt-enable`, `slew-rate`, `power-source`,
`output-low`, `output-high`.

**Conflict detection is a real feature.** `pinctrl` refuses to let two devices claim the same
pin:
```
pinctrl core: pin PA5 already requested by 1c28000.serial; cannot claim for 1c68000.spi
```
That message has saved countless hours of "why doesn't my SPI work". Without pinctrl, the two
drivers would silently fight over the mux register.

### T.3 GPIO: why the integer API had to die

The original API was:

```c
/* ★ THE OLD, DEPRECATED, REMOVED-FROM-NEW-CODE API */
gpio_request(42, "my-reset");
gpio_direction_output(42, 1);
gpio_set_value(42, 0);
gpio_free(42);
```

The problems with a global integer namespace are structural, and worth enumerating because
the same reasoning applies whenever you are tempted to use one:

1. **No namespacing.** GPIO 42 on which controller? The base was assigned at registration
   time in probe order — so adding an expander shifted everyone's numbers.
2. **The number is not stable.** It depended on probe order, which depended on link order and
   module load order (Ch. 27 T.4). Board files hardcoded numbers that broke when anything
   changed.
3. **No polarity.** A reset line that is active-low forced *every driver* to know the board's
   wiring: `gpio_set_value(42, 0); /* active low! */`. Board knowledge leaked into drivers.
4. **No lifetime.** Nothing tied the GPIO to a device, so `devm_` was impossible.
5. **No way to express "the reset line"** — only "GPIO 42", which is meaningless to a driver
   that must work on ten boards.

The **descriptor API** (Linus Walleij, 3.13+) fixes all five:

```c
struct gpio_desc *reset;

reset = devm_gpiod_get(dev, "reset", GPIOD_OUT_HIGH);   /* ★ by NAME, with a direction */
if (IS_ERR(reset))
	return dev_err_probe(dev, PTR_ERR(reset), "reset gpio\n");

gpiod_set_value_cansleep(reset, 1);      /* ★ 1 = ASSERTED, whatever the polarity */
msleep(10);
gpiod_set_value_cansleep(reset, 0);      /* 0 = DEASSERTED */
```

**The polarity point is the important one.** `gpiod_set_value(desc, 1)` means *logically
asserted*. If the DT says `GPIO_ACTIVE_LOW`, the core drives the physical line low. **The
driver never knows and never cares.** That is the board knowledge staying in the board
description, where it belongs (Ch. 32 T.8).

```dts
mydevice {
	reset-gpios  = <&gpio0 5 GPIO_ACTIVE_LOW>;      /* ★ the board says it's active-low */
	enable-gpios = <&gpio0 6 GPIO_ACTIVE_HIGH>;
	irq-gpios    = <&gpio1 3 GPIO_ACTIVE_LOW>;
	cs-gpios     = <&gpio1 4 GPIO_ACTIVE_LOW>;
};
```
The property name convention is `<name>-gpios` (or `<name>-gpio` for a single one), and
`devm_gpiod_get(dev, "reset", ...)` looks up `reset-gpios`. The same lookup works on ACPI via
`_CRS` + `_DSD` (Ch. 33 T.5) and on software nodes.

### T.4 The descriptor API surface

```c
/* Acquisition */
d = devm_gpiod_get(dev, "reset", GPIOD_OUT_HIGH);
d = devm_gpiod_get_optional(dev, "enable", GPIOD_OUT_LOW);   /* NULL if absent — ★ common */
d = devm_gpiod_get_index(dev, "data", 3, GPIOD_IN);
arr = devm_gpiod_get_array(dev, "data", GPIOD_OUT_LOW);      /* a whole bus of pins */

/* Flags: GPIOD_IN, GPIOD_OUT_LOW, GPIOD_OUT_HIGH,
 *        GPIOD_OUT_LOW_OPEN_DRAIN, GPIOD_ASIS,
 *        | GPIOD_FLAGS_BIT_NONEXCLUSIVE (shared, e.g. a common enable) */

/* Value access — TWO variants, and the distinction matters */
gpiod_get_value(d);              /* may NOT sleep — memory-mapped GPIO only */
gpiod_set_value(d, 1);
gpiod_get_value_cansleep(d);     /* ★ MAY sleep — required for I²C/SPI expanders */
gpiod_set_value_cansleep(d, 1);

/* Raw access — bypasses polarity. Use ONLY when you genuinely mean the electrical level */
gpiod_get_raw_value(d);
gpiod_set_raw_value(d, 1);

/* Direction, config */
gpiod_direction_input(d);
gpiod_direction_output(d, 1);
gpiod_set_config(d, PIN_CONF_PACKED(PIN_CONFIG_BIAS_PULL_UP, 1));
gpiod_set_consumer_name(d, "my-reset");

/* Arrays — ★ set many pins in ONE controller access if possible */
gpiod_set_array_value_cansleep(arr->ndescs, arr->desc, arr->info, values);
gpiod_get_array_value_cansleep(arr->ndescs, arr->desc, arr->info, values);

/* As an interrupt (T.6) */
irq = gpiod_to_irq(d);
```

**`_cansleep` is not optional politeness.** A GPIO on a PCA9555 I²C expander requires an I²C
transaction to read or write (Ch. 40 T.5). Calling the non-sleeping variant on it triggers
`WARN_ON` and, on a `CONFIG_DEBUG_ATOMIC_SLEEP` kernel, a "sleeping function called from
invalid context" splat. **Default to `_cansleep` unless you are in atomic context and have
verified the GPIO is memory-mapped.**

The array API deserves emphasis: setting 8 GPIOs on an expander individually costs 8 I²C
transactions (~2 ms); `gpiod_set_array_value()` can do it in one register write if they are
contiguous on the same chip. `drivers/gpio/gpiolib.c` does that optimization automatically.

### T.5 Writing a GPIO controller

```c
struct my_gpio {
	struct gpio_chip  gc;
	void __iomem     *base;
	raw_spinlock_t    lock;         /* ★ raw: may be used from hardirq */
};

static int my_get(struct gpio_chip *gc, unsigned int off)
{
	struct my_gpio *g = gpiochip_get_data(gc);

	return !!(readl(g->base + DATA) & BIT(off));
}

static int my_set(struct gpio_chip *gc, unsigned int off, int val)
{
	struct my_gpio *g = gpiochip_get_data(gc);
	u32 v;

	guard(raw_spinlock_irqsave)(&g->lock);     /* ★ read-modify-write must be atomic */
	v = readl(g->base + DATA);
	if (val)
		v |= BIT(off);
	else
		v &= ~BIT(off);
	writel(v, g->base + DATA);
	return 0;
}

static int my_direction_input(struct gpio_chip *gc, unsigned int off) { ... }
static int my_direction_output(struct gpio_chip *gc, unsigned int off, int val) { ... }
static int my_get_direction(struct gpio_chip *gc, unsigned int off) { ... }

static int my_probe(struct platform_device *pdev)
{
	struct my_gpio *g = devm_kzalloc(&pdev->dev, sizeof(*g), GFP_KERNEL);

	g->base = devm_platform_ioremap_resource(pdev, 0);
	raw_spin_lock_init(&g->lock);

	g->gc.label             = dev_name(&pdev->dev);
	g->gc.parent            = &pdev->dev;
	g->gc.owner             = THIS_MODULE;
	g->gc.base              = -1;              /* ★ dynamic numbering; never hardcode */
	g->gc.ngpio             = 32;
	g->gc.get               = my_get;
	g->gc.set_rv            = my_set;          /* returning int (6.x) */
	g->gc.direction_input   = my_direction_input;
	g->gc.direction_output  = my_direction_output;
	g->gc.get_direction     = my_get_direction;
	g->gc.can_sleep         = false;           /* ★ memory-mapped: does not sleep */
	g->gc.request           = gpiochip_generic_request;   /* → pinctrl (T.7) */
	g->gc.free              = gpiochip_generic_free;
	g->gc.set_config        = gpiochip_generic_config;

	return devm_gpiochip_add_data(&pdev->dev, &g->gc, g);
}
```

For the extremely common "a few registers, one bit per line" controller, **`gpio-mmio`**
generates all of this:

```c
ret = bgpio_init(&g->gc, dev, 4 /* bytes per reg */,
		 base + DAT, base + SET, base + CLR,
		 base + DIROUT, base + DIRIN, 0);
```

### T.6 GPIO as an interrupt source

Most SoC GPIO controllers can generate interrupts. That makes the GPIO chip an
**interrupt controller**, and it slots into the `irq_domain` hierarchy of Ch. 17 T.6:

```c
static const struct irq_chip my_irq_chip = {
	.name              = "my-gpio",
	.irq_ack           = my_irq_ack,
	.irq_mask          = my_irq_mask,
	.irq_unmask        = my_irq_unmask,
	.irq_set_type      = my_irq_set_type,
	.irq_set_wake      = my_irq_set_wake,
	.flags             = IRQCHIP_IMMUTABLE,      /* ★ required in modern code */
	GPIOCHIP_IRQ_RESOURCE_HELPERS,
};

girq = &g->gc.irq;
gpio_irq_chip_set_chip(girq, &my_irq_chip);
girq->parent_handler = my_irq_handler;       /* the cascaded handler */
girq->num_parents    = 1;
girq->parents        = devm_kcalloc(dev, 1, sizeof(*girq->parents), GFP_KERNEL);
girq->parents[0]     = platform_get_irq(pdev, 0);
girq->default_type   = IRQ_TYPE_NONE;
girq->handler        = handle_bad_irq;
```

A consumer then does:

```c
d = devm_gpiod_get(dev, "irq", GPIOD_IN);
irq = gpiod_to_irq(d);
ret = devm_request_threaded_irq(dev, irq, NULL, my_thread_fn,
				IRQF_ONESHOT | IRQF_TRIGGER_FALLING, name, priv);
```
or, more commonly, the DT names the GPIO controller as the interrupt parent and the consumer
uses `platform_get_irq()`/`client->irq` without knowing a GPIO is involved:

```dts
touchscreen@5d {
	interrupt-parent = <&gpio1>;
	interrupts = <3 IRQ_TYPE_LEVEL_LOW>;
};
```

**For expanders on I²C/SPI, the interrupt path necessarily involves a threaded handler**: the
expander raises one line, the kernel must do an I²C read to find which pin changed, and that
read sleeps. `regmap_irq` (Ch. 34 T.7) implements this generically.

### T.7 The gpiolib↔pinctrl handshake

When a driver requests a GPIO, the pin must actually be *muxed* to GPIO mode. That is the
`gpiochip_generic_request()` → `pinctrl_gpio_request()` path, and it requires the GPIO
controller and the pin controller to agree on numbering:

```c
/* In the pin controller driver: */
static const struct pinctrl_gpio_range my_range = {
	.name = "my-gpio", .id = 0,
	.base = /* gpio number base */, .pin_base = /* pin number base */, .npins = 32,
};
pinctrl_add_gpio_range(pctldev, &my_range);
```
or, from DT, with the `gpio-ranges` property:
```dts
gpio0: gpio@1c20800 {
	gpio-controller;
	#gpio-cells = <2>;
	gpio-ranges = <&pinctrl 0 0 32>;    /* gpio 0..31 map to pins 0..31 */
};
```

This is why requesting a GPIO can fail with `-EBUSY` when the pin is muxed to UART — the two
subsystems are consulting each other. **Understanding this handshake explains most
"gpiod_get returns -EBUSY" bring-up failures.**

### T.8 The userspace ABI: chardev, not sysfs

The old `/sys/class/gpio` interface exported integer GPIO numbers to userspace and inherited
every problem from T.3, plus two more: no way to tie a line's lifetime to a process (a crashed
program left a line exported and configured), and no way to read multiple lines atomically.

**It was deprecated and removed.** The replacement is a character device:

```
/dev/gpiochip0      → ioctl(GPIO_V2_GET_LINEINFO_IOCTL)     query lines
                    → ioctl(GPIO_V2_GET_LINE_IOCTL)         request lines → a NEW fd
   that line fd     → ioctl(GPIO_V2_LINE_GET_VALUES_IOCTL)  read
                    → ioctl(GPIO_V2_LINE_SET_VALUES_IOCTL)  write
                    → read()                                 edge events, with timestamps
```

This is Ch. 29 T.1's argument applied: making it an **fd** gives lifetime (close = release),
`poll()` for edge events, permissions, and atomic multi-line operations for free.

```bash
sudo apt install -y gpiod libgpiod-dev
gpiodetect                                   # every gpiochip
gpioinfo                                     # ★ every line, name, direction, consumer
gpioinfo gpiochip0 | head -20

gpioget gpiochip0 5
gpioset gpiochip0 5=1
gpioset --mode=time --sec=2 gpiochip0 5=1    # hold for 2s, then release
gpiomon --num-events=5 --falling-edge gpiochip0 3   # ★ timestamped edge events
gpiofind "SPI0_CS"                           # find a line by name
```

Line names come from the DT `gpio-line-names` property — **set them**; they turn
`gpiochip0 line 17` into `"ETH_RESET"` for everyone debugging the board later.

```dts
gpio0: gpio@1c20800 {
	gpio-line-names = "", "", "ETH_RESET", "LED_STATUS", "", "USER_BUTTON";
};
```

---

## 1. Internals

### 1.1 Source map

```
drivers/gpio/gpiolib.c          ★★ the core: descriptors, arrays, DT/ACPI lookup
drivers/gpio/gpiolib-cdev.c     ★ the chardev ABI (T.8)
drivers/gpio/gpiolib-of.c       ★ *-gpios parsing, gpio-ranges
drivers/gpio/gpiolib-acpi.c     _CRS GpioIo/GpioInt + _DSD name mapping (Ch. 33)
drivers/gpio/gpiolib-sysfs.c    the legacy interface (deprecated)
drivers/gpio/gpio-mmio.c        ★ bgpio_init — the generic register-based chip
drivers/gpio/gpio-sim.c         ★ a simulated GPIO chip for testing — use it
drivers/gpio/gpio-aggregator.c  expose a subset of lines as a new chip (for VMs/containers)
drivers/pinctrl/core.c          ★★ pinctrl core: states, ranges, conflict detection
drivers/pinctrl/pinmux.c, pinconf.c, pinconf-generic.c
drivers/pinctrl/pinctrl-single.c ★ a generic driver for "one register per pin" SoCs
include/linux/gpio/consumer.h   ★★ the API you use
include/linux/gpio/driver.h     the API you implement
include/linux/pinctrl/
include/uapi/linux/gpio.h       ★ the v2 chardev ABI
Documentation/driver-api/gpio/  ★★ intro.rst, consumer.rst, driver.rst, board.rst,
                                   using-gpio.rst, legacy.rst
Documentation/driver-api/pin-control.rst ★★
Documentation/devicetree/bindings/gpio/gpio.txt, pinctrl/
```

### 1.2 `gpio-sim`: GPIO without hardware

```bash
./scripts/config -m GPIO_SIM && make modules
sudo modprobe gpio-sim
sudo mount -t configfs none /sys/kernel/config 2>/dev/null

D=/sys/kernel/config/gpio-sim/mychip
sudo mkdir -p $D/bank0
echo 16      | sudo tee $D/bank0/num_lines
echo mybank  | sudo tee $D/bank0/label
sudo mkdir -p $D/bank0/line3/hog
echo "ETH_RESET" | sudo tee $D/bank0/line3/name
echo 1       | sudo tee $D/live                     # ★ instantiate

gpiodetect
gpioinfo | grep -A3 mybank

# Drive lines from the "hardware" side via sysfs:
CHIP=$(cat $D/bank0/chip_name)
ls /sys/devices/platform/gpio-sim.0/$CHIP/sim_gpio*/
echo 1 | sudo tee /sys/devices/platform/gpio-sim.0/$CHIP/sim_gpio3/pull
gpioget $(gpiofind ETH_RESET)
```
**This gives you a full GPIO development environment on a laptop**, and combined with
`i2c-gpio` (Ch. 40) and `spi-gpio` (Ch. 41) you can build an entire virtual board.

---

## 2. Practice

### Lab 42.1 — Map your board's pins and GPIOs

```bash
# Pin controller view
sudo ls /sys/kernel/debug/pinctrl/
P=$(sudo ls -d /sys/kernel/debug/pinctrl/*/ | head -1)
sudo cat $P/pins | head -30
sudo cat $P/pinmux-pins | grep -v 'UNCLAIMED' | head -30   # ★ who owns what
sudo cat $P/pinconf-pins | head -20
sudo cat $P/pingroups | head -20
sudo cat $P/pinmux-functions | head -20

# GPIO view
sudo cat /sys/kernel/debug/gpio
gpiodetect
gpioinfo

# ★ Correlate: for one pin, show its mux state AND its GPIO state
sudo cat $P/pinmux-pins | grep 'PA5'
gpioinfo | grep -i 'line.*5'
```
**Deliverable:** for five pins on your board, state: the physical pin, the current mux
function, the electrical config, whether it is a GPIO, and which driver owns it.

### Lab 42.2 — A complete GPIO consumer

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * gpiodemo — demonstrates the descriptor API, polarity abstraction,
 * arrays, and GPIO-as-interrupt.
 */
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

#include <linux/gpio/consumer.h>
#include <linux/interrupt.h>
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/property.h>

struct gpiodemo {
	struct device           *dev;
	struct gpio_desc        *reset;
	struct gpio_desc        *enable;
	struct gpio_desc        *irq_gpio;
	struct gpio_descs       *data_bus;
	int                      irq;
	u64                      edges;
	u32                      reset_us;
};

static irqreturn_t gpiodemo_irq(int irq, void *data)
{
	struct gpiodemo *g = data;

	g->edges++;
	/* ★ _cansleep is safe here: this is the THREADED handler */
	dev_info_ratelimited(g->dev, "edge #%llu, line now %d\n",
			     g->edges, gpiod_get_value_cansleep(g->irq_gpio));
	return IRQ_HANDLED;
}

static ssize_t edges_show(struct device *dev, struct device_attribute *a, char *buf)
{
	return sysfs_emit(buf, "%llu\n", ((struct gpiodemo *)dev_get_drvdata(dev))->edges);
}
static DEVICE_ATTR_RO(edges);

static ssize_t databus_store(struct device *dev, struct device_attribute *a,
			     const char *buf, size_t n)
{
	struct gpiodemo *g = dev_get_drvdata(dev);
	DECLARE_BITMAP(values, BITS_PER_TYPE(u32));
	u32 v;
	int ret = kstrtou32(buf, 0, &v);

	if (ret)
		return ret;
	bitmap_from_arr32(values, &v, g->data_bus->ndescs);

	/* ★ T.4: ONE controller access for the whole bus, not N */
	ret = gpiod_set_array_value_cansleep(g->data_bus->ndescs, g->data_bus->desc,
					     g->data_bus->info, values);
	return ret ?: n;
}
static DEVICE_ATTR_WO(databus);

static struct attribute *gpiodemo_attrs[] = {
	&dev_attr_edges.attr, &dev_attr_databus.attr, NULL,
};
ATTRIBUTE_GROUPS(gpiodemo);

static int gpiodemo_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	struct gpiodemo *g;
	int ret;

	g = devm_kzalloc(dev, sizeof(*g), GFP_KERNEL);
	if (!g)
		return -ENOMEM;
	g->dev = dev;
	platform_set_drvdata(pdev, g);

	if (device_property_read_u32(dev, "acme,reset-duration-us", &g->reset_us))
		g->reset_us = 10000;

	/* ★ T.3: by NAME, with an initial direction and value.
	 * GPIOD_OUT_HIGH means LOGICALLY asserted — the core applies polarity. */
	g->reset = devm_gpiod_get(dev, "reset", GPIOD_OUT_HIGH);
	if (IS_ERR(g->reset))
		return dev_err_probe(dev, PTR_ERR(g->reset), "reset gpio\n");

	g->enable = devm_gpiod_get_optional(dev, "enable", GPIOD_OUT_LOW);
	if (IS_ERR(g->enable))
		return dev_err_probe(dev, PTR_ERR(g->enable), "enable gpio\n");

	g->data_bus = devm_gpiod_get_array_optional(dev, "data", GPIOD_OUT_LOW);
	if (IS_ERR(g->data_bus))
		return dev_err_probe(dev, PTR_ERR(g->data_bus), "data gpios\n");
	if (g->data_bus)
		dev_info(dev, "data bus: %u lines, can_sleep=%d\n",
			 g->data_bus->ndescs,
			 gpiod_cansleep(g->data_bus->desc[0]));

	/* Reset pulse — no polarity knowledge in this code (T.3) */
	gpiod_set_value_cansleep(g->reset, 1);     /* assert */
	fsleep(g->reset_us);
	gpiod_set_value_cansleep(g->reset, 0);     /* release */
	if (g->enable)
		gpiod_set_value_cansleep(g->enable, 1);

	/* GPIO as an interrupt source (T.6) */
	g->irq_gpio = devm_gpiod_get_optional(dev, "irq", GPIOD_IN);
	if (IS_ERR(g->irq_gpio))
		return dev_err_probe(dev, PTR_ERR(g->irq_gpio), "irq gpio\n");
	if (g->irq_gpio) {
		g->irq = gpiod_to_irq(g->irq_gpio);
		if (g->irq < 0)
			return dev_err_probe(dev, g->irq, "gpio is not irq-capable\n");

		/* ★ THREADED: the handler may do I²C/SPI, and an expander GPIO sleeps */
		ret = devm_request_threaded_irq(dev, g->irq, NULL, gpiodemo_irq,
						IRQF_ONESHOT | IRQF_TRIGGER_FALLING,
						dev_name(dev), g);
		if (ret)
			return dev_err_probe(dev, ret, "request irq\n");
	}

	/* Give the lines meaningful names for gpioinfo (T.8) */
	gpiod_set_consumer_name(g->reset, "gpiodemo-reset");

	dev_info(dev, "probed: reset=%d enable=%d irq=%d\n",
		 desc_to_gpio(g->reset),
		 g->enable ? desc_to_gpio(g->enable) : -1, g->irq);
	return 0;
}

static const struct of_device_id gpiodemo_of_ids[] = {
	{ .compatible = "acme,gpiodemo" }, { }
};
MODULE_DEVICE_TABLE(of, gpiodemo_of_ids);

static struct platform_driver gpiodemo_driver = {
	.driver = {
		.name = "gpiodemo",
		.of_match_table = gpiodemo_of_ids,
		.dev_groups = gpiodemo_groups,
	},
	.probe = gpiodemo_probe,
};
module_platform_driver(gpiodemo_driver);
MODULE_LICENSE("GPL");
```

The matching overlay:
```dts
/dts-v1/;
/plugin/;
#include <dt-bindings/gpio/gpio.h>
#include <dt-bindings/interrupt-controller/irq.h>

&{/} {
	gpiodemo {
		compatible = "acme,gpiodemo";
		reset-gpios  = <&gpio0 3 GPIO_ACTIVE_LOW>;      /* ★ active LOW */
		enable-gpios = <&gpio0 4 GPIO_ACTIVE_HIGH>;
		irq-gpios    = <&gpio0 5 GPIO_ACTIVE_LOW>;
		data-gpios   = <&gpio0 8  GPIO_ACTIVE_HIGH>,
			       <&gpio0 9  GPIO_ACTIVE_HIGH>,
			       <&gpio0 10 GPIO_ACTIVE_HIGH>,
			       <&gpio0 11 GPIO_ACTIVE_HIGH>;
		acme,reset-duration-us = <5000>;
	};
};
```

```bash
# With gpio-sim as the controller (§1.2):
sudo insmod gpiodemo.ko
dmesg | tail -5
gpioinfo | grep gpiodemo
cat /sys/bus/platform/devices/gpiodemo/edges
echo 0xA | sudo tee /sys/bus/platform/devices/gpiodemo/databus

# ★ Prove the polarity abstraction: flip GPIO_ACTIVE_LOW to GPIO_ACTIVE_HIGH
# in the overlay, rebuild, and observe the PHYSICAL line invert while the
# DRIVER CODE is unchanged.
sudo cat /sys/devices/platform/gpio-sim.0/*/sim_gpio3/value
```

### Lab 42.3 — Prove the `_cansleep` rule (T.4)

```c
/* Deliberately call the non-sleeping variant on an expander GPIO */
static irqreturn_t bad_hardirq(int irq, void *data)
{
	struct gpiodemo *g = data;

	gpiod_set_value(g->reset, 1);     /* ★ if reset is on an I²C expander: BUG */
	return IRQ_HANDLED;
}
```
```bash
./scripts/config -e DEBUG_ATOMIC_SLEEP -e PROVE_LOCKING
# Wire the GPIO to a pca9555 (or i2c-stub-backed expander) and trigger it:
dmesg | grep -A20 'BUG: sleeping function called from invalid context'
grep -n 'WARN_ON(desc->gdev->chip->can_sleep)' drivers/gpio/gpiolib.c
grep -n -A5 'gpiod_get_value(' drivers/gpio/gpiolib.c | head -20
```

### Lab 42.4 — Pin conflict detection (T.2)

```bash
# Try to claim a pin that is already muxed to a peripheral
sudo cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep -v UNCLAIMED | head -5
# Pick a pin owned by, say, the UART, and request it as a GPIO:
gpioinfo | grep -i uart
gpioget gpiochip0 4        # → "Device or resource busy"

dmesg | grep -i 'pin.*already requested\|cannot claim'

# In a driver:
d = devm_gpiod_get(dev, "conflicting", GPIOD_IN);   /* → -EBUSY */

# The handshake that produced it (T.7):
grep -n -A20 'pinctrl_gpio_request' drivers/pinctrl/core.c
grep -n 'gpio-ranges' drivers/gpio/gpiolib-of.c
sudo cat /sys/kernel/debug/pinctrl/*/gpio-ranges 2>/dev/null
```

### Lab 42.5 — Runtime state switching (T.2)

Implement the I²C bus-recovery re-mux from Ch. 40 T.6:

```c
struct pinctrl       *p       = devm_pinctrl_get(dev);
struct pinctrl_state *s_i2c   = pinctrl_lookup_state(p, "default");
struct pinctrl_state *s_gpio  = pinctrl_lookup_state(p, "gpio");

/* On a stuck bus: */
pinctrl_select_state(p, s_gpio);       /* pins become GPIOs */
bitbang_recovery(scl_desc, sda_desc);  /* 9 clock pulses + STOP */
pinctrl_select_state(p, s_i2c);        /* back to the controller */
```
```bash
sudo bpftrace -e 'kprobe:pinctrl_select_state { @ = count(); }'
sudo cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep -i i2c
# Watch the mux register change across the switch:
sudo devmem2 0x1c20800 w        # or your SoC's mux register
```

### Lab 42.6 — The chardev ABI from C (T.8)

```c
/* gpiocdev.c — gcc -O2 -o gpiocdev gpiocdev.c */
#include <fcntl.h>
#include <linux/gpio.h>
#include <poll.h>
#include <stdio.h>
#include <string.h>
#include <sys/ioctl.h>
#include <unistd.h>

int main(int argc, char **argv)
{
	int chip = open(argv[1], O_RDONLY);
	struct gpiochip_info ci;
	struct gpio_v2_line_request req = {0};
	struct gpio_v2_line_values vals = {0};
	unsigned int i;

	ioctl(chip, GPIO_GET_CHIPINFO_IOCTL, &ci);
	printf("chip '%s' label '%s' lines %u\n", ci.name, ci.label, ci.lines);

	/* Enumerate lines */
	for (i = 0; i < ci.lines && i < 8; i++) {
		struct gpio_v2_line_info li = { .offset = i };

		ioctl(chip, GPIO_V2_GET_LINEINFO_IOCTL, &li);
		printf("  line %2u: '%s' consumer '%s' flags 0x%llx\n",
		       i, li.name, li.consumer, (unsigned long long)li.flags);
	}

	/* Request two lines as inputs with both-edge events */
	req.offsets[0] = 3;
	req.offsets[1] = 5;
	req.num_lines  = 2;
	req.config.flags = GPIO_V2_LINE_FLAG_INPUT |
			   GPIO_V2_LINE_FLAG_EDGE_RISING |
			   GPIO_V2_LINE_FLAG_EDGE_FALLING;
	strcpy(req.consumer, "gpiocdev-lab");

	if (ioctl(chip, GPIO_V2_GET_LINE_IOCTL, &req) < 0) {
		perror("GET_LINE");
		return 1;
	}
	printf("got line fd %d — ★ closing it releases the lines automatically\n", req.fd);

	/* Atomic multi-line read */
	vals.mask = 0x3;
	ioctl(req.fd, GPIO_V2_LINE_GET_VALUES_IOCTL, &vals);
	printf("values: line3=%d line5=%d\n",
	       !!(vals.bits & 1), !!(vals.bits & 2));

	/* Blocking, timestamped edge events */
	{
		struct pollfd pfd = { .fd = req.fd, .events = POLLIN };

		printf("waiting for edges...\n");
		while (poll(&pfd, 1, 5000) > 0) {
			struct gpio_v2_line_event ev;

			read(req.fd, &ev, sizeof(ev));
			printf("edge: line %u %s at %llu ns (seq %u)\n",
			       ev.offset,
			       ev.id == GPIO_V2_LINE_EVENT_RISING_EDGE ? "RISING" : "FALLING",
			       (unsigned long long)ev.timestamp_ns, ev.line_seqno);
		}
	}
	close(req.fd);
	close(chip);
	return 0;
}
```
```bash
sudo ./gpiocdev /dev/gpiochip0
# Meanwhile, from the "hardware" side with gpio-sim:
echo 1 | sudo tee /sys/devices/platform/gpio-sim.0/*/sim_gpio3/pull
echo 0 | sudo tee /sys/devices/platform/gpio-sim.0/*/sim_gpio3/pull

# The same thing with libgpiod:
gpiomon --format="%e %o %s.%n" gpiochip0 3 5
```

### Lab 42.7 — Build a virtual board

Combine everything: a simulated GPIO chip driving a bit-banged I²C bus and a bit-banged SPI
bus, with devices on both.

```bash
sudo modprobe gpio-sim
D=/sys/kernel/config/gpio-sim/virtboard
sudo mkdir -p $D/bank0
echo 32 | sudo tee $D/bank0/num_lines
for i in 0 1 2 3 4 5; do
  sudo mkdir -p $D/bank0/line$i
done
echo "I2C_SCL" | sudo tee $D/bank0/line0/name
echo "I2C_SDA" | sudo tee $D/bank0/line1/name
echo "SPI_SCK" | sudo tee $D/bank0/line2/name
echo "SPI_MOSI"| sudo tee $D/bank0/line3/name
echo "SPI_MISO"| sudo tee $D/bank0/line4/name
echo "SPI_CS"  | sudo tee $D/bank0/line5/name
echo 1 | sudo tee $D/live

gpioinfo
# Then apply the i2c-gpio and spi-gpio overlays from Ch. 40/41.
i2cdetect -l
ls /sys/bus/spi/devices/
```
**You now have an entire embedded board's bus topology on a laptop**, suitable for developing
and testing every driver in Chapters 40–44 without hardware.

---

## 3. Mastery drills

1. **Read `Documentation/driver-api/gpio/`** completely — it is short and it is the normative
   text. Then read `Documentation/driver-api/pin-control.rst`.

2. **The five problems.** For each of T.3's five failures of the integer API, find a real
   historical bug or workaround in the git history that it caused
   (`git log --grep='gpio.*base' --oneline | head -40`).

3. **Polarity.** Take a driver that uses `gpiod_set_value(d, 1)`. Change the DT from
   `GPIO_ACTIVE_HIGH` to `GPIO_ACTIVE_LOW` and verify with a logic analyzer (or gpio-sim)
   that the physical level inverts while the code is unchanged. Then find a driver that uses
   `gpiod_set_raw_value()` and justify it.

4. **`_cansleep` audit.** `git grep -n 'gpiod_set_value(' drivers/ | head -30`. For each,
   determine whether the GPIO could be on a sleeping expander. Find one that is a latent bug.

5. **Write a GPIO controller.** Implement one over `gpio-mmio` (`bgpio_init`) for a
   hypothetical 3-register chip (DATA, DIR, and a separate SET/CLR pair). Add interrupt
   support with an immutable `irq_chip`. Test with gpio-sim's pull interface as a model.

6. **The gpiolib↔pinctrl handshake.** Read `pinctrl_gpio_request()` and `gpio-ranges`
   parsing. Construct a board where the ranges are wrong and show the resulting failure mode.

7. **Array optimization.** Instrument `gpiod_set_array_value_cansleep()` on an I²C expander
   and count the resulting I²C transactions for 8 contiguous lines versus 8 individual
   `gpiod_set_value()` calls. Report the ratio and the time saved.

8. **The chardev ABI.** Read `drivers/gpio/gpiolib-cdev.c`. Explain how line requests are
   refcounted, how edge events are timestamped (which clock? — see
   `GPIO_V2_LINE_FLAG_EVENT_CLOCK_REALTIME`), and how the debounce filter works.

9. **Deprecation archaeology.** Find the commits removing `/sys/class/gpio`
   (`git log --grep='gpio.*sysfs' --oneline`). Read the deprecation rationale. What did
   userspace have to change, and how was the transition managed given Ch. 24 T.1's rules?

10. **pinctrl-single.** Read `drivers/pinctrl/pinctrl-single.c`. Explain how one driver
    serves dozens of SoCs with "one register per pin", and what the DT expresses.

11. **Design question.** A board needs 48 GPIOs: 20 from the SoC (mixed with peripheral
    functions), 16 from an I²C expander (with an interrupt), and 12 from an FPGA over SPI.
    Six of them must be readable from a hardirq handler; four are interrupt sources; eight
    form a parallel data bus written atomically. Design the DT, state which lines can be on
    which controller, and specify the driver's GPIO acquisition and access patterns.

---

## 4. Further reading

**Kernel documentation:**
- `Documentation/driver-api/gpio/intro.rst` ★★, `consumer.rst` ★★, `driver.rst` ★,
  `board.rst`, `using-gpio.rst`, `legacy.rst` (what *not* to do), `bt8xxgpio.rst`
- `Documentation/driver-api/pin-control.rst` ★★ — long, thorough, written by the maintainer
- `Documentation/userspace-api/gpio/` ★ — the chardev ABI, v1 and v2
- `Documentation/devicetree/bindings/gpio/gpio.txt` ★ — the `*-gpios` conventions
- `Documentation/devicetree/bindings/pinctrl/pinctrl-bindings.txt` ★ and
  `pincfg-node.yaml`, `pinmux-node.yaml` — the standard properties
- `Documentation/admin-guide/gpio/` — `gpio-sim.rst`, `gpio-aggregator.rst`

**Source:**
- `drivers/gpio/gpiolib.c` ★★, `gpiolib-cdev.c` ★, `gpiolib-of.c`
- `drivers/gpio/gpio-mmio.c` ★ — read `bgpio_init()`; it generates most simple drivers
- `drivers/gpio/gpio-sim.c` ★ — and use it
- `drivers/pinctrl/core.c`, `pinmux.c`, `pinconf-generic.c`
- `drivers/pinctrl/pinctrl-single.c` ★ — the generic SoC driver
- Exemplary: `drivers/gpio/gpio-pca953x.c` (I²C expander with regmap_irq),
  `drivers/pinctrl/sunxi/`, `drivers/pinctrl/qcom/`

**Tools:**
- `libgpiod` / `gpiod` tools: `gpiodetect`, `gpioinfo`, `gpioget`, `gpioset`, `gpiomon`,
  `gpiofind` ★ — learn all six
- `gpio-sim` + configfs for hardware-free development
- `devmem2` / `busybox devmem` for raw register pokes during bring-up
- A logic analyzer with `sigrok`/PulseView

**LWN & talks:**
- "The new GPIO character device API" / "Deprecating the GPIO sysfs interface"
- "Pin control subsystem" (Linus Walleij's introduction)
- Linus Walleij's ELC talks on GPIO and pinctrl — **the maintainer explaining his own
  design**; the clearest available account of T.1–T.3
- Bartosz Golaszewski's talks on libgpiod and the chardev ABI

→ Next: [43-clk-reset-regulator.md](43-clk-reset-regulator.md)

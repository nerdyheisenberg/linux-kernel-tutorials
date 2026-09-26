# Chapter 44 — Sensors: IIO, hwmon, and the Problem of Reporting Numbers

> **Goal:** understand why the kernel has two sensor subsystems, why IIO reports
> `raw × scale + offset` instead of real units, and how triggers turn a polling API into a
> megasample-per-second acquisition pipeline.

---

## Theory & First Principles

### T.0 — Start here: a temperature sensor. What is the interface?

You have a driver that can read a number from a chip. Expose it to userspace. Every decision
here is permanent (Ch. 24), and every one of them has been got wrong:

```
  "25"                 ← degrees? Celsius? Since when?
  "25.3"               ← floating point in a sysfs file. The kernel has no FPU.
  "temp=25.3C"         ← now every reader needs a parser, forever.
  "0x19"               ← raw register value; userspace must know the chip.
  25300                ← millidegrees Celsius. Integer. Unambiguous.
```

**The last one is the answer, and the reason is worth stating as a rule:**

> **Fix the unit in the ABI, use a fixed-point integer, and one value per file.** The kernel
> has no floating point (Ch. 00 §T.8), text parsing is a permanent liability, and "one value
> per file" is what makes `cat`, shell globs, and every monitoring tool work without a
> library.

So hwmon says: `temp1_input` is **millidegrees Celsius**, `in0_input` is **millivolts**,
`fan1_input` is **RPM**. Every hwmon driver on every chip, forever. A monitoring tool written
in 2005 reads a sensor shipped in 2026.

**Now the harder question, and it is why there are *two* subsystems here.** Consider an
accelerometer sampling at 10 kHz:

| | A thermal sensor | An accelerometer |
|---|---|---|
| Rate | ~1 Hz | 10–100 **kHz** |
| Reader | a human, or a daemon polling | a control loop that must not miss samples |
| Interface | `cat temp1_input` | ? |
| Timestamps | irrelevant | **essential** |
| Buffering | none | **mandatory** |

At 10 kHz, `read()` on a sysfs file is 10,000 syscalls per second producing text you must
parse — and you will still drop samples, with no way to know you did. **The interface that is
right for a thermometer is catastrophically wrong for a sensor.**

Hence the split, and knowing which to use is the actual decision this chapter asks you to
make:

| | **hwmon** | **IIO** |
|---|---|---|
| For | health monitoring: temperature, fan, voltage | data acquisition: accel, gyro, ADC, light, magnetometer |
| Rate | low, polled | **high, buffered** |
| Interface | sysfs, fixed units | sysfs for config + **a chardev with a ring buffer** |
| Sample metadata | none | **timestamps**, channel layout, scan masks |
| Trigger model | none | **triggers**: timer, interrupt, or another device |
| Scaling | driver converts to fixed units | `raw × scale + offset`, computed in **userspace** |

**That last row is a genuine design disagreement, and it is worth having a view on.** hwmon
converts in the kernel and guarantees units. IIO exports `_raw`, `_scale`, and `_offset` and
makes userspace do the arithmetic — because at 100 kHz, doing a multiply per sample in the
kernel is wasted work when userspace must touch the data anyway, and because integer
conversion loses precision the application may want. **hwmon optimizes for the ABI being
self-describing; IIO optimizes for throughput and fidelity.** Neither is wrong; they serve
different rates.

**And the trigger concept is the one genuinely novel idea here:** in IIO, *when to sample* is
decoupled from *what to sample*. A hrtimer trigger, a GPIO interrupt from a data-ready pin,
or another device's completion can all drive a capture, and the same driver works with all
three. That separation is why one accelerometer driver serves both "poll it occasionally" and
"capture 10 kHz synchronized with an external strobe."

```bash
sensors                                   # hwmon, via libsensors
grep . /sys/class/hwmon/hwmon*/temp*_input
ls /sys/bus/iio/devices/iio:device0/
cat /sys/bus/iio/devices/iio:device0/in_accel_x_raw \
    /sys/bus/iio/devices/iio:device0/in_accel_scale
iio_info && iio_readdev -b 256 iio:device0    # from libiio
```

---

### T.1 The problem: what is the ABI for a number?

A sensor driver's job sounds trivial — read a register, give the value to userspace. The
hard question is *what contract* that value carries. Consider four plausible ABIs:

| ABI | Example | Problem |
|---|---|---|
| Raw register value | `read() → 0x3A7` | Userspace must know the device, the reference voltage, the gain… |
| Device-specific text | `read() → "temp=42.5C"` | Every tool needs a parser per device |
| An ioctl with a struct | `TEMP_GET_IOC` | Ch. 24 T.9: unfilterable, per-device, no shell access |
| **Fixed-unit sysfs files** | `temp1_input → 42500` | ★ one tool reads every device ever made |

The kernel chose the last, and that choice is the entire content of both subsystems:

> **Standardize the *units and names* at the ABI, so that a generic tool can display any
> sensor from any vendor without device-specific knowledge.**

This is the sysfs one-value-per-file rule from Ch. 24 T.8 applied to a whole device class,
and it is what makes `sensors`, `iio_readdev`, and every monitoring agent possible.
`lm-sensors` predates almost every chip it supports.

The names and units are **normative ABI**, documented in
`Documentation/ABI/testing/sysfs-bus-iio` and `Documentation/hwmon/sysfs-interface.rst`:

| File | Unit | Note |
|---|---|---|
| `in_temp_input` / `temp1_input` | **milli°C** | integer, never float |
| `in_voltage0_raw` / `in0_input` | millivolts (hwmon) / raw (IIO) | |
| `in_accel_x_raw` | raw | × `in_accel_scale` → m/s² |
| `in_pressure_input` | kilopascal | |
| `fan1_input` | RPM | |
| `power1_average` | microwatt | |

**Milli-units are not an aesthetic choice — they exist because the kernel cannot use floating
point** (Ch. 0's Love constraint: the FPU state is not saved on kernel entry). Fixed-point
integers in a documented unit are the only representation that works. Every odd-looking
decision in these subsystems traces back to that one constraint.

### T.2 Two subsystems, because there are two problems

| | **hwmon** | **IIO** |
|---|---|---|
| Question answered | "What is the CPU temperature *right now*?" | "Give me 3200 accelerometer samples/s" |
| Consumers | humans, monitoring agents | signal processing, motion algorithms, DAQ |
| Rate | ~1 Hz | 1 Hz – 1 MHz |
| Interface | sysfs only | sysfs **+ a chardev with a ring buffer** |
| Value form | processed, fixed unit | usually `raw` + `scale` + `offset` |
| Timestamps | none | ★ per-sample, nanosecond |
| Typical devices | fan controllers, VRMs, PSUs, CPU thermal | ADCs, IMUs, light/pressure/proximity, DACs |
| Directory | `drivers/hwmon/` | `drivers/iio/` |

They overlap — a temperature sensor could live in either — and the kernel provides **bridges**
in both directions (`drivers/iio/adc/iio_hwmon`-style `iio-hwmon`, and
`drivers/hwmon/`'s IIO consumers), so a device registered in one can be exposed through the
other. The decision rule:

> **If the interesting question is "what is the value now?", use hwmon. If it is "give me a
> stream of values with timestamps", use IIO.**

### T.3 Why IIO reports `raw × scale + offset`

IIO's most-questioned design is that `in_voltage0_raw` gives you `1723`, not `1.723 V`. The
conversion is published separately:

```
processed = (raw + offset) × scale
```
```bash
cat /sys/bus/iio/devices/iio:device0/in_voltage0_raw     # 1723
cat /sys/bus/iio/devices/iio:device0/in_voltage_scale    # 0.610351562   (mV per LSB)
cat /sys/bus/iio/devices/iio:device0/in_voltage_offset   # 0
# → 1723 × 0.610351562 = 1051.6 mV
```

Three reasons, in order of importance:

1. **The kernel has no floating point.** Scale is frequently irrational-ish
   (`Vref / 2^n = 3.3/4096`). Doing the multiply in kernel either loses precision or requires
   fixed-point contortions in *every driver*. Publishing the factor moves one multiply to
   userspace, where doubles exist.
2. **Buffered mode must be zero-conversion.** At 100 kSPS the driver copies raw ADC words
   into a ring buffer with no arithmetic at all (T.5). If the sysfs value were processed and
   the buffered value raw, the two interfaces would disagree — so both are raw.
3. **Scale is dynamic.** Changing the PGA gain changes `scale`, not the driver. Userspace
   re-reads one file.

`IIO_CHAN_INFO_PROCESSED` exists for when the conversion genuinely needs device-specific
knowledge (a thermocouple's polynomial), giving `in_temp_input` in milli°C directly. **Use
`RAW`+`SCALE` when the conversion is a linear factor; use `PROCESSED` only when it is not.**

`libiio` does the conversion for you and is the correct userspace entry point.

### T.4 The channel: IIO's unit of description

Everything in IIO is described declaratively by an array of `struct iio_chan_spec`, and the
core generates all the sysfs files from it:

```c
static const struct iio_chan_spec my_channels[] = {
	{
		.type = IIO_VOLTAGE,
		.indexed = 1,
		.channel = 0,                                   /* → in_voltage0_* */
		.info_mask_separate   = BIT(IIO_CHAN_INFO_RAW), /* ★ per-channel file */
		.info_mask_shared_by_type =                     /* ★ one file for all voltages */
			BIT(IIO_CHAN_INFO_SCALE) |
			BIT(IIO_CHAN_INFO_SAMP_FREQ),
		.scan_index = 0,                                /* ★ position in the buffer */
		.scan_type = {
			.sign = 'u', .realbits = 12, .storagebits = 16,
			.shift = 0, .endianness = IIO_LE,
		},
	},
	IIO_CHAN_SOFT_TIMESTAMP(4),    /* ★ the timestamp is itself a channel */
};
```

Two ideas here are worth internalizing:

**(a) `separate` vs `shared_by_type` / `shared_by_all` is deduplication with meaning.** If
every voltage channel has the same scale, there is one `in_voltage_scale` file, not eight.
The mask *declares* which properties are per-channel and which are global, and the core turns
that declaration into the right filenames. Compare with Ch. 32 T.4: description as data.

**(b) `scan_type` is a wire format.** `realbits` (meaningful bits), `storagebits` (bits on the
wire), `shift`, `sign`, and `endianness` fully specify how to decode a sample from the buffer
without knowing the device. Userspace reads these from
`scan_elements/in_voltage0_type` as a compact string:

```bash
cat /sys/bus/iio/devices/iio:device0/scan_elements/in_voltage0_type
# le:u12/16>>0     ★ little-endian, unsigned, 12 real bits in 16 storage bits, shift 0
```
That string is a **self-describing binary format** — the same problem netlink solves with
attributes (Ch. 24 T.8) and DT solves with bindings (Ch. 32 T.3), solved here by publishing
the layout.

### T.5 Triggers: decoupling *when* from *what*

The deepest design idea in IIO is that **the thing being sampled and the thing that decides
*when* to sample are separate objects.**

```
   ┌───────────┐         ┌──────────────┐        ┌──────────────┐
   │  Trigger  │────────▶│ iio_poll_    │───────▶│ Device's     │
   │           │ (a      │ func / top-  │        │ trigger      │
   │ • data-rdy│  poll   │ half + IRQ   │        │ handler:     │
   │   IRQ     │  event) │ thread)      │        │ read+push    │
   │ • hrtimer │         └──────────────┘        └──────┬───────┘
   │ • sysfs   │                                        │
   │ • another │                                   ┌────▼──────┐
   │   device! │                                   │ kfifo ring│──▶ /dev/iio:deviceN
   └───────────┘                                   └───────────┘
```

Any trigger can drive any device, and one trigger can drive several devices **synchronously**
— which is exactly what you need to correlate an accelerometer and a gyroscope, or to sample
four ADCs on the same edge. If each driver had its own private timer, that composition would
be impossible.

The trigger kinds:

| Trigger | Source | Use |
|---|---|---|
| `iio-trig-hrtimer` | kernel hrtimer | ★ software sampling at an exact rate, no hardware needed |
| `iio-trig-sysfs` | a userspace write | testing, one-shot capture |
| `iio-trig-interrupt` | a GPIO/IRQ line | ★ external sync, data-ready pins |
| device-provided | the chip's own DRDY | best: sample exactly when data exists |

```bash
# Create a 100 Hz software trigger out of thin air
sudo modprobe iio-trig-hrtimer
sudo mkdir -p /config/iio/triggers/hrtimer/mytrig     # configfs!
echo 100 | sudo tee /sys/bus/iio/devices/trigger0/sampling_frequency
# Attach it to a device
echo mytrig | sudo tee /sys/bus/iio/devices/iio:device0/trigger/current_trigger
```

The trigger is created through **configfs** (Ch. 30) — a rare case where creating a kernel
object from userspace is exactly the right model, because the *number* of triggers is a
userspace policy decision.

### T.6 The buffered path and where the time goes

Buffered mode is a producer/consumer pipeline (Ch. 25 P6) whose correctness hinges on one
question: **when is the timestamp taken?**

```c
static irqreturn_t my_trigger_handler(int irq, void *p)
{
	struct iio_poll_func *pf = p;
	struct iio_dev *indio_dev = pf->indio_dev;
	struct my_state *st = iio_priv(indio_dev);

	/* ★ this runs in a THREAD (iio_triggered_buffer_setup uses a threaded IRQ),
	 * so it MAY sleep — I²C/SPI reads are legal here (Ch. 40 T.5, Ch. 41) */
	ret = regmap_bulk_read(st->regmap, REG_DATA, st->scan.chans,
			       ARRAY_SIZE(st->scan.chans));
	if (ret)
		goto done;

	iio_push_to_buffers_with_timestamp(indio_dev, &st->scan,
					   pf->timestamp);   /* ★ captured in the TOP half */
done:
	iio_trigger_notify_done(indio_dev->trig);
	return IRQ_HANDLED;
}
```

`pf->timestamp` is captured in the **hard-IRQ top half** (`iio_pollfunc_store_time`), *before*
the slow bus read. That matters: a 100 µs I²C transaction would otherwise smear jitter into
every sample and ruin any frequency-domain analysis. Ch. 17's latency decomposition applied
to data quality.

The buffer itself is a **kfifo** (Ch. 10 T.4) with the standard readiness predicate (Ch. 29
T.5) so `poll()`, blocking `read()`, and `O_NONBLOCK` all agree:

```bash
D=/sys/bus/iio/devices/iio:device0
echo 1 | sudo tee $D/scan_elements/in_voltage0_en      # ★ select channels
echo 1 | sudo tee $D/scan_elements/in_timestamp_en
echo 128 | sudo tee $D/buffer/length
echo 1 | sudo tee $D/buffer/enable                     # ★ device becomes read-only-buffered
sudo cat /dev/iio:device0 | xxd | head
echo 0 | sudo tee $D/buffer/enable
```

**Enabling the buffer is a mode change**: sysfs `_raw` reads return `-EBUSY` while buffered
capture runs, because the hardware is now being driven by the trigger. `iio_device_claim_direct_mode()`
is the primitive that enforces it — a nice example of an explicit, checkable state machine
(Ch. 25 P8) replacing a comment saying "don't do both at once".

### T.7 hwmon: the value of being boring

hwmon deliberately has no chardev, no buffers, no triggers. Its modern API is a single
`_info` registration where the driver describes its channels and provides three callbacks:

```c
static umode_t my_hwmon_is_visible(const void *data, enum hwmon_sensor_types type,
				   u32 attr, int channel)
{
	switch (type) {
	case hwmon_temp:
		switch (attr) {
		case hwmon_temp_input: case hwmon_temp_label:  return 0444;
		case hwmon_temp_max:                            return 0644;
		}
		break;
	}
	return 0;    /* ★ returning 0 means "this file does not exist" */
}

static int my_hwmon_read(struct device *dev, enum hwmon_sensor_types type,
			 u32 attr, int channel, long *val)
{
	*val = raw_to_millidegrees(...);    /* ★ hwmon values ARE in fixed units */
	return 0;
}

static const struct hwmon_channel_info * const my_info[] = {
	HWMON_CHANNEL_INFO(chip, HWMON_C_REGISTER_TZ),
	HWMON_CHANNEL_INFO(temp,
			   HWMON_T_INPUT | HWMON_T_MAX | HWMON_T_ALARM | HWMON_T_LABEL,
			   HWMON_T_INPUT | HWMON_T_MAX),
	HWMON_CHANNEL_INFO(fan, HWMON_F_INPUT | HWMON_F_FAULT),
	NULL
};
static const struct hwmon_ops my_hwmon_ops = {
	.is_visible = my_hwmon_is_visible,
	.read       = my_hwmon_read,
	.write      = my_hwmon_write,
	.read_string= my_hwmon_read_string,
};
static const struct hwmon_chip_info my_chip_info = {
	.ops = &my_hwmon_ops, .info = my_info,
};

hwmon = devm_hwmon_device_register_with_info(dev, "mychip", st, &my_chip_info, NULL);
```

This replaced hundreds of hand-rolled `SENSOR_DEVICE_ATTR` arrays. The win is the same as
`dev_groups` in Ch. 26: **the core generates the attributes, so the naming ABI cannot be
violated by a typo in a driver.** `is_visible` returning a mode is a small, elegant way to
express "this chip has `temp1_max` but not `temp2_max`" without conditional attribute arrays.

`HWMON_C_REGISTER_TZ` wires the sensor into the thermal framework automatically, so a DT
thermal zone can throttle on it — an in-kernel consumer, which brings us to:

### T.8 In-kernel consumers: sensors as a provider/consumer resource

Both subsystems let *other kernel code* consume a channel, using the same provider/consumer
DT pattern as Ch. 43:

```dts
&adc {
	#io-channel-cells = <1>;
};

battery {
	io-channels = <&adc 3>;
	io-channel-names = "battery-voltage";
};
```
```c
chan = devm_iio_channel_get(dev, "battery-voltage");
ret  = iio_read_channel_processed(chan, &microvolts);
ret  = iio_read_channel_raw(chan, &raw);
ret  = iio_convert_raw_to_processed(chan, raw, &processed, 1000);
```

This is why `iio-hwmon` (expose IIO channels as hwmon), `iio_bat`, thermal zones backed by
ADCs, and joystick-over-ADC all exist without any device-specific glue: **the channel is a
first-class shareable resource**, not a private driver detail. Same shape as T.1 of Ch. 43,
a fifth instance of the provider/consumer pattern.

---

## 1. Internals

### 1.1 Source map

```
drivers/iio/industrialio-core.c       ★★ channels → sysfs, the whole ABI generator
drivers/iio/industrialio-buffer.c     ★★ scan masks, demux, the kfifo path
drivers/iio/industrialio-trigger.c    ★ T.5
drivers/iio/buffer/industrialio-triggered-buffer.c ★ the 3-line setup helper
drivers/iio/buffer/kfifo_buf.c        the default buffer implementation
drivers/iio/trigger/iio-trig-hrtimer.c, iio-trig-sysfs.c, iio-trig-interrupt.c ★
drivers/iio/inkern.c                  ★ T.8 in-kernel consumers
include/linux/iio/iio.h               ★★ iio_chan_spec — read every field
include/linux/iio/trigger*.h, buffer.h, sysfs.h
drivers/hwmon/hwmon.c                 ★★ the _with_info core, is_visible
include/linux/hwmon.h                 ★ HWMON_CHANNEL_INFO and the attr enums
Documentation/ABI/testing/sysfs-bus-iio ★★ THE normative unit/name list — 2000+ lines
Documentation/hwmon/hwmon-kernel-api.rst ★★, sysfs-interface.rst ★★, submitting-patches.rst
Documentation/iio/                    iio_configfs.rst, ep93xx_adc.rst, etc.
```

**Exemplary drivers to read:**
```
drivers/iio/adc/ti-ads1015.c       ★ I²C ADC, regmap, buffered
drivers/iio/imu/inv_mpu6050/       ★★ FIFO, hardware trigger, multiple sensors
drivers/iio/accel/bma180.c         ★ classic triggered-buffer structure
drivers/iio/light/opt3001.c        ★ small and complete
drivers/iio/dummy/                 ★★ iio_simple_dummy — the teaching driver, builds anywhere
drivers/hwmon/lm75.c               ★ the canonical simple hwmon driver
drivers/hwmon/nct6775-core.c       a big real-world one
```

---

## 2. Practice

### Lab 44.1 — Explore IIO with no hardware

```bash
sudo modprobe iio_dummy           # ★ CONFIG_IIO_SIMPLE_DUMMY=m
sudo modprobe industrialio-sw-device
# Instantiate via configfs
sudo mount -t configfs none /config 2>/dev/null
sudo mkdir /config/iio/devices/dummy/mydev

D=$(ls -d /sys/bus/iio/devices/iio:device* | head -1)
ls $D
cat $D/name
cat $D/in_voltage0_raw $D/in_voltage_scale 2>/dev/null
ls $D/scan_elements/
cat $D/scan_elements/in_voltage0_type       # ★ decode this string using T.4

# The real hardware on your machine, if any:
for d in /sys/bus/iio/devices/iio:device*; do
  echo "=== $(cat $d/name) ==="; ls $d | head -20
done
sudo apt install libiio-utils 2>/dev/null || true
iio_info                                    # ★★ the best overview tool
```

### Lab 44.2 — A complete triggered-buffer IIO driver

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * fakeadc — a 4-channel 12-bit ADC with sysfs and triggered-buffer support.
 * Demonstrates T.3 (raw+scale), T.4 (channel spec), T.6 (timestamp in the top half).
 */
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

#include <linux/bitops.h>
#include <linux/iio/buffer.h>
#include <linux/iio/iio.h>
#include <linux/iio/sysfs.h>
#include <linux/iio/trigger_consumer.h>
#include <linux/iio/triggered_buffer.h>
#include <linux/module.h>
#include <linux/mutex.h>
#include <linux/platform_device.h>
#include <linux/random.h>

#define FAKEADC_NUM_CH   4
#define FAKEADC_VREF_MV  3300
#define FAKEADC_BITS     12

struct fakeadc {
	struct mutex lock;               /* serialises hardware access */
	unsigned int sampfreq;
	u32          phase;
	/* ★ The scan buffer must be naturally aligned for the s64 timestamp and
	 * must not share a cache line with anything DMA'd (Ch. 35 T.7). */
	struct {
		u16 chan[FAKEADC_NUM_CH];
		aligned_s64 timestamp;
	} scan;
};

static int fakeadc_read_hw(struct fakeadc *st, int ch)
{
	/* A synthetic waveform so the data is recognisable in a plot. */
	u32 v = (int_sqrt((st->phase + ch * 512) & 0xfff) * 64) & 0xfff;

	st->phase += 37;
	return v;
}

#define FAKEADC_CHAN(idx) {						\
	.type = IIO_VOLTAGE,						\
	.indexed = 1,							\
	.channel = (idx),						\
	.info_mask_separate = BIT(IIO_CHAN_INFO_RAW),			\
	.info_mask_shared_by_type = BIT(IIO_CHAN_INFO_SCALE),		\
	.info_mask_shared_by_all  = BIT(IIO_CHAN_INFO_SAMP_FREQ),	\
	.scan_index = (idx),						\
	.scan_type = {							\
		.sign = 'u',						\
		.realbits = FAKEADC_BITS,				\
		.storagebits = 16,					\
		.shift = 0,						\
		.endianness = IIO_CPU,					\
	},								\
}

static const struct iio_chan_spec fakeadc_channels[] = {
	FAKEADC_CHAN(0), FAKEADC_CHAN(1), FAKEADC_CHAN(2), FAKEADC_CHAN(3),
	IIO_CHAN_SOFT_TIMESTAMP(FAKEADC_NUM_CH),
};

static int fakeadc_read_raw(struct iio_dev *indio_dev,
			    struct iio_chan_spec const *chan,
			    int *val, int *val2, long mask)
{
	struct fakeadc *st = iio_priv(indio_dev);
	int ret;

	switch (mask) {
	case IIO_CHAN_INFO_RAW:
		/* ★ T.6: refuse direct reads while the buffer owns the hardware */
		ret = iio_device_claim_direct_mode(indio_dev);
		if (ret)
			return ret;
		mutex_lock(&st->lock);
		*val = fakeadc_read_hw(st, chan->channel);
		mutex_unlock(&st->lock);
		iio_device_release_direct_mode(indio_dev);
		return IIO_VAL_INT;

	case IIO_CHAN_INFO_SCALE:
		/* ★ T.3: publish the factor, don't apply it */
		*val  = FAKEADC_VREF_MV;
		*val2 = FAKEADC_BITS;
		return IIO_VAL_FRACTIONAL_LOG2;   /* scale = val / 2^val2 mV per LSB */

	case IIO_CHAN_INFO_SAMP_FREQ:
		*val = st->sampfreq;
		return IIO_VAL_INT;
	}
	return -EINVAL;
}

static int fakeadc_write_raw(struct iio_dev *indio_dev,
			     struct iio_chan_spec const *chan,
			     int val, int val2, long mask)
{
	struct fakeadc *st = iio_priv(indio_dev);

	if (mask != IIO_CHAN_INFO_SAMP_FREQ)
		return -EINVAL;
	if (val < 1 || val > 10000)
		return -EINVAL;

	guard(mutex)(&st->lock);
	st->sampfreq = val;
	return 0;
}

static IIO_CONST_ATTR_SAMP_FREQ_AVAIL("1 10 100 1000 10000");
static struct attribute *fakeadc_attributes[] = {
	&iio_const_attr_sampling_frequency_available.dev_attr.attr,
	NULL,
};
static const struct attribute_group fakeadc_attr_group = {
	.attrs = fakeadc_attributes,
};

static const struct iio_info fakeadc_info = {
	.read_raw  = fakeadc_read_raw,
	.write_raw = fakeadc_write_raw,
	.attrs     = &fakeadc_attr_group,
};

/*
 * The trigger handler. Runs in a kernel THREAD (threaded IRQ), so it may sleep.
 * pf->timestamp was captured in the hard-IRQ top half — see T.6.
 */
static irqreturn_t fakeadc_trigger_handler(int irq, void *p)
{
	struct iio_poll_func *pf = p;
	struct iio_dev *indio_dev = pf->indio_dev;
	struct fakeadc *st = iio_priv(indio_dev);
	int i, j = 0;

	mutex_lock(&st->lock);
	/* ★ Only read the channels actually enabled in the scan mask */
	iio_for_each_active_channel(indio_dev, i)
		st->scan.chan[j++] = fakeadc_read_hw(st, i);
	mutex_unlock(&st->lock);

	iio_push_to_buffers_with_timestamp(indio_dev, &st->scan, pf->timestamp);
	iio_trigger_notify_done(indio_dev->trig);
	return IRQ_HANDLED;
}

static int fakeadc_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	struct iio_dev *indio_dev;
	struct fakeadc *st;
	int ret;

	/* ★ iio_priv() memory is allocated WITH the iio_dev — one allocation */
	indio_dev = devm_iio_device_alloc(dev, sizeof(*st));
	if (!indio_dev)
		return -ENOMEM;

	st = iio_priv(indio_dev);
	mutex_init(&st->lock);
	st->sampfreq = 100;

	indio_dev->name = "fakeadc";
	indio_dev->info = &fakeadc_info;
	indio_dev->modes = INDIO_DIRECT_MODE;      /* buffer setup adds BUFFER_TRIGGERED */
	indio_dev->channels = fakeadc_channels;
	indio_dev->num_channels = ARRAY_SIZE(fakeadc_channels);

	/* ★ Three lines to get a chardev, kfifo, scan mask, demux and poll support */
	ret = devm_iio_triggered_buffer_setup(dev, indio_dev,
					      iio_pollfunc_store_time,   /* top half: timestamp */
					      fakeadc_trigger_handler,   /* thread: read + push */
					      NULL);
	if (ret)
		return dev_err_probe(dev, ret, "triggered buffer setup\n");

	/* Publish LAST (Ch. 31 T.7) — sysfs and the chardev appear here */
	return devm_iio_device_register(dev, indio_dev);
}

static struct platform_driver fakeadc_driver = {
	.driver = { .name = "fakeadc" },
	.probe  = fakeadc_probe,
};

static struct platform_device *fakeadc_pdev;

static int __init fakeadc_init(void)
{
	int ret = platform_driver_register(&fakeadc_driver);

	if (ret)
		return ret;
	fakeadc_pdev = platform_device_register_simple("fakeadc", -1, NULL, 0);
	if (IS_ERR(fakeadc_pdev)) {
		platform_driver_unregister(&fakeadc_driver);
		return PTR_ERR(fakeadc_pdev);
	}
	return 0;
}
static void __exit fakeadc_exit(void)
{
	platform_device_unregister(fakeadc_pdev);
	platform_driver_unregister(&fakeadc_driver);
}
module_init(fakeadc_init);
module_exit(fakeadc_exit);
MODULE_DESCRIPTION("Synthetic 4-channel IIO ADC with triggered buffer");
MODULE_LICENSE("GPL");
```

```bash
sudo insmod fakeadc.ko
D=$(grep -l fakeadc /sys/bus/iio/devices/iio:device*/name | xargs dirname)
echo $D
cat $D/in_voltage0_raw
cat $D/in_voltage_scale                 # 3300/4096 = 0.805664062
cat $D/sampling_frequency{,_available}
# ★ processed value, computed in userspace:
python3 -c "print($(cat $D/in_voltage0_raw) * $(cat $D/in_voltage_scale), 'mV')"
```

### Lab 44.3 — Capture a buffer with an hrtimer trigger

```bash
sudo modprobe industrialio-sw-trigger iio-trig-hrtimer
sudo mount -t configfs none /config 2>/dev/null
sudo mkdir /config/iio/triggers/hrtimer/trig100        # ★ create a trigger from nothing

T=$(grep -l trig100 /sys/bus/iio/devices/trigger*/name | xargs dirname)
echo 100 | sudo tee $T/sampling_frequency

D=$(grep -l fakeadc /sys/bus/iio/devices/iio:device*/name | xargs dirname)
echo trig100 | sudo tee $D/trigger/current_trigger

# Select channels (the scan mask)
echo 1 | sudo tee $D/scan_elements/in_voltage0_en
echo 1 | sudo tee $D/scan_elements/in_voltage1_en
echo 1 | sudo tee $D/scan_elements/in_timestamp_en
cat $D/scan_elements/*_type
cat $D/scan_elements/*_index

echo 512 | sudo tee $D/buffer/length
echo 1   | sudo tee $D/buffer/enable

# ★ Direct reads are now refused:
cat $D/in_voltage0_raw       # → Device or resource busy   (T.6)

sudo timeout 2 cat /dev/iio:device0 > /tmp/cap.bin
echo 0 | sudo tee $D/buffer/enable
ls -l /tmp/cap.bin
xxd /tmp/cap.bin | head

# Work out the sample size from scan_elements and verify:
#   2 × u16 = 4 bytes, then padding to 8 for the s64 timestamp = 16 bytes/sample
python3 - <<'EOF'
import struct
d = open('/tmp/cap.bin','rb').read()
n = len(d)//16
print("samples:", n, "expected ~200 for 2s @ 100Hz")
prev = None
for i in range(min(n, 8)):
    c0, c1, _, ts = struct.unpack_from('<HHIq', d, i*16)
    dt = (ts - prev)/1e6 if prev else 0
    prev = ts
    print(f"ch0={c0:4d} ch1={c1:4d} ts={ts} dt={dt:.3f} ms")
EOF
```

```bash
# ★ Or let libiio do all the decoding:
iio_readdev -t trig100 -b 256 -s 1000 fakeadc in_voltage0 > /tmp/cap2.bin
iio_attr -a
```

### Lab 44.4 — Measure timestamp jitter (T.6)

```bash
# Capture at 1 kHz and histogram the inter-sample intervals
echo 1000 | sudo tee $T/sampling_frequency
echo 1 | sudo tee $D/buffer/enable
sudo timeout 5 cat /dev/iio:device0 > /tmp/jit.bin
echo 0 | sudo tee $D/buffer/enable

python3 - <<'EOF'
import struct, statistics
d = open('/tmp/jit.bin','rb').read()
ts = [struct.unpack_from('<q', d, i*16+8)[0] for i in range(len(d)//16)]
dt = [(b-a)/1000 for a,b in zip(ts, ts[1:])]          # microseconds
print(f"n={len(dt)} mean={statistics.mean(dt):.1f}us "
      f"stdev={statistics.stdev(dt):.1f}us min={min(dt):.1f} max={max(dt):.1f}")
buckets = {}
for x in dt: buckets[round(x/50)*50] = buckets.get(round(x/50)*50,0)+1
for k in sorted(buckets): print(f"{k:6d}us {'#'*min(60,buckets[k])}")
EOF
```
Now repeat under load (`stress-ng --cpu $(nproc)`) and with `PREEMPT_RT` if available.
Then **move the timestamp into the threaded handler** (`iio_get_time_ns(indio_dev)` instead of
`pf->timestamp`) and re-measure. The difference is T.6's entire argument, quantified.

### Lab 44.5 — A complete hwmon driver (T.7)

```c
// SPDX-License-Identifier: GPL-2.0
/* faketemp — a hwmon chip with two temperature channels and one fan. */
#include <linux/hwmon.h>
#include <linux/module.h>
#include <linux/platform_device.h>

struct faketemp {
	long temp[2], temp_max[2];
	long fan;
};

static const char * const faketemp_labels[] = { "core", "ambient" };

static umode_t faketemp_is_visible(const void *data, enum hwmon_sensor_types type,
				   u32 attr, int channel)
{
	switch (type) {
	case hwmon_temp:
		switch (attr) {
		case hwmon_temp_input:
		case hwmon_temp_label:
		case hwmon_temp_alarm:
			return 0444;
		case hwmon_temp_max:
			return 0644;                 /* ★ writable threshold */
		}
		break;
	case hwmon_fan:
		if (attr == hwmon_fan_input)
			return 0444;
		break;
	default:
		break;
	}
	return 0;    /* ★ file simply does not appear */
}

static int faketemp_read(struct device *dev, enum hwmon_sensor_types type,
			 u32 attr, int channel, long *val)
{
	struct faketemp *st = dev_get_drvdata(dev);

	switch (type) {
	case hwmon_temp:
		switch (attr) {
		case hwmon_temp_input:
			/* ★ hwmon values ARE processed: milli-degrees C */
			st->temp[channel] += (get_random_u32() % 400) - 200;
			*val = clamp(st->temp[channel], 20000L, 95000L);
			return 0;
		case hwmon_temp_max:
			*val = st->temp_max[channel];
			return 0;
		case hwmon_temp_alarm:
			*val = st->temp[channel] > st->temp_max[channel];
			return 0;
		}
		break;
	case hwmon_fan:
		*val = st->fan;
		return 0;
	default:
		break;
	}
	return -EOPNOTSUPP;
}

static int faketemp_write(struct device *dev, enum hwmon_sensor_types type,
			  u32 attr, int channel, long val)
{
	struct faketemp *st = dev_get_drvdata(dev);

	if (type == hwmon_temp && attr == hwmon_temp_max) {
		st->temp_max[channel] = clamp(val, 0L, 125000L);
		return 0;
	}
	return -EOPNOTSUPP;
}

static int faketemp_read_string(struct device *dev, enum hwmon_sensor_types type,
				u32 attr, int channel, const char **str)
{
	if (type == hwmon_temp && attr == hwmon_temp_label) {
		*str = faketemp_labels[channel];
		return 0;
	}
	return -EOPNOTSUPP;
}

static const struct hwmon_ops faketemp_ops = {
	.is_visible  = faketemp_is_visible,
	.read        = faketemp_read,
	.write       = faketemp_write,
	.read_string = faketemp_read_string,
};

static const struct hwmon_channel_info * const faketemp_info[] = {
	HWMON_CHANNEL_INFO(temp,
		HWMON_T_INPUT | HWMON_T_MAX | HWMON_T_ALARM | HWMON_T_LABEL,
		HWMON_T_INPUT | HWMON_T_MAX | HWMON_T_ALARM | HWMON_T_LABEL),
	HWMON_CHANNEL_INFO(fan, HWMON_F_INPUT),
	NULL
};

static const struct hwmon_chip_info faketemp_chip = {
	.ops = &faketemp_ops,
	.info = faketemp_info,
};

static int faketemp_probe(struct platform_device *pdev)
{
	struct faketemp *st = devm_kzalloc(&pdev->dev, sizeof(*st), GFP_KERNEL);
	struct device *hwmon;

	if (!st)
		return -ENOMEM;
	st->temp[0] = 45000; st->temp[1] = 30000;
	st->temp_max[0] = st->temp_max[1] = 85000;
	st->fan = 2400;

	hwmon = devm_hwmon_device_register_with_info(&pdev->dev, "faketemp",
						     st, &faketemp_chip, NULL);
	return PTR_ERR_OR_ZERO(hwmon);
}
```
```bash
sudo insmod faketemp.ko
H=$(grep -l faketemp /sys/class/hwmon/hwmon*/name | xargs dirname)
ls $H
cat $H/temp1_label $H/temp1_input $H/temp1_max $H/temp1_alarm
echo 50000 | sudo tee $H/temp1_max
cat $H/temp1_alarm                     # ★ now 1

sudo apt install lm-sensors 2>/dev/null
sensors                                # ★ a generic tool, zero driver knowledge (T.1)
sensors -j | head -20
```

### Lab 44.6 — Bridge IIO to hwmon and the thermal framework (T.8)

```bash
sudo modprobe iio_hwmon
# With a DT node:
#   iio-hwmon { compatible = "iio-hwmon"; io-channels = <&fakeadc 0>, <&fakeadc 1>; };
sensors | grep -A5 iio

# Thermal zones consuming a sensor:
ls /sys/class/thermal/
for z in /sys/class/thermal/thermal_zone*; do
  echo "$(cat $z/type): $(cat $z/temp) mC, policy=$(cat $z/policy)"
  ls $z | grep trip_point | head
done
cat /sys/class/thermal/thermal_zone0/trip_point_0_{type,temp}
grep -rn 'HWMON_C_REGISTER_TZ' drivers/hwmon/ | head
```

### Lab 44.7 — Read `sysfs-bus-iio` and audit a driver

```bash
wc -l Documentation/ABI/testing/sysfs-bus-iio
grep -n 'What:.*in_' Documentation/ABI/testing/sysfs-bus-iio | head -40

# Pick a real driver and check every attribute it creates against the ABI doc:
$EDITOR drivers/iio/light/opt3001.c
# For each IIO_CHAN_INFO_* used, find the generated filename and its ABI entry.

# ★ The core's mapping table:
grep -n -A60 'iio_chan_info_postfix' drivers/iio/industrialio-core.c
```

---

## 3. Mastery drills

1. **The no-FPU consequence (T.1/T.3).** Write down every design decision in IIO and hwmon
   that exists because the kernel cannot use floating point. There are at least five.

2. **Choose the subsystem.** For each of: a CPU package thermal sensor, a 200 kSPS
   oscilloscope front end, a laptop lid accelerometer, a PSU current monitor, a barometer for
   indoor navigation, a battery fuel gauge — pick hwmon or IIO and justify in one sentence.
   Then find the real kernel driver and check whether you agreed with its author.

3. **Decode a type string.** Given `be:s14/16>>2`, write the C code that extracts the signed
   value from a buffer sample. Then find a real driver with a non-zero `shift` and explain
   why the hardware needs it.

4. **`separate` vs `shared_by_type` vs `shared_by_all`.** Design the masks for a 4-channel ADC
   with a per-channel PGA, a shared reference voltage, and a global sample rate. Now for one
   with a *shared* PGA. Show the resulting filenames in each case.

5. **Trigger composition (T.5).** Configure one hrtimer trigger driving two different IIO
   devices simultaneously. Capture from both and show the timestamps are correlated. Explain
   why per-driver private timers could not achieve this.

6. **Timestamp placement.** Read `iio_pollfunc_store_time`. Find a driver that timestamps in
   the threaded handler instead and decide whether it is a bug. Then find one with a hardware
   FIFO where neither placement is correct, and explain what it does instead.

7. **Mode exclusion.** Read `iio_device_claim_direct_mode()` / the newer
   `iio_device_claim_direct()` scoped form. Construct a race that the older
   "just check `iio_buffer_enabled()`" approach would lose.

8. **hwmon `is_visible`.** Explain why returning a `umode_t` is better than a
   `bool` here, and better than building the attribute array conditionally. Then find a driver
   whose `is_visible` depends on a chip variant detected at probe.

9. **Write a hwmon driver for real hardware.** Use `i2c-stub` (Ch. 40 Lab) to emulate an LM75
   and write the driver from the datasheet without reading `drivers/hwmon/lm75.c`. Then diff
   your design against the real one.

10. **In-kernel consumers (T.8).** Trace `devm_iio_channel_get()` through `drivers/iio/inkern.c`.
    What happens if the provider driver is unbound while a consumer holds a channel? Compare
    with Ch. 28 T.4's devres lifetime problem.

11. **Buffer watermark.** Set `$D/buffer/watermark` and measure how wakeup frequency and
    latency trade off. Relate to Ch. 18's deferred-work batching and Ch. 17's
    interrupt-mitigation theory.

12. **Design question.** A 6-axis IMU has a 1024-sample hardware FIFO, a data-ready interrupt,
    per-axis scales that change with the configured range, an on-chip temperature sensor, and
    a "motion detected" event. Design the full IIO driver: channel specs, scan types, trigger
    strategy, how the hardware FIFO interacts with `iio_push_to_buffers`, how timestamps are
    assigned to FIFO-batched samples, and how events reach userspace.

---

## 4. Further reading

**Kernel documentation:**
- `Documentation/ABI/testing/sysfs-bus-iio` ★★★ — the normative ABI. Read it end to end once.
- `Documentation/hwmon/sysfs-interface.rst` ★★★ — the hwmon equivalent
- `Documentation/hwmon/hwmon-kernel-api.rst` ★★ — the `_with_info` API
- `Documentation/hwmon/submitting-patches.rst` ★ — the maintainer's checklist; read before
  writing any sensor driver
- `Documentation/iio/iio_configfs.rst` ★, `Documentation/driver-api/iio/` (where present)
- `Documentation/devicetree/bindings/iio/` — hundreds of real examples

**Source:**
- `drivers/iio/dummy/iio_simple_dummy*.c` ★★★ — the teaching driver; builds on any machine
- `drivers/iio/industrialio-core.c` ★★, `industrialio-buffer.c` ★★, `industrialio-trigger.c` ★
- `include/linux/iio/iio.h` ★★ — `iio_chan_spec` field by field
- `drivers/iio/imu/inv_mpu6050/` ★★ — hardware FIFO + trigger, a real-world masterclass
- `drivers/hwmon/hwmon.c` ★★, `drivers/hwmon/lm75.c` ★
- `drivers/iio/adc/iio_hwmon.c`, `drivers/iio/inkern.c` ★ — T.8

**Userspace:**
- **libiio** (Analog Devices) ★★ — `iio_info`, `iio_readdev`, `iio_attr`, and the C/Python API
- `lm-sensors` / `sensors` / `sensors-detect` ★
- `tools/iio/` in the kernel tree: `iio_generic_buffer.c` ★★ (the reference consumer),
  `iio_event_monitor.c`, `lsiio.c`

**Talks & articles:**
- Jonathan Cameron's ELC/LPC talks on IIO — **the maintainer explaining T.4 and T.5**
- "The Industrial I/O subsystem" (LWN)
- Analog Devices' IIO wiki — the best practical tutorial outside the kernel tree
- Guenter Roeck's hwmon talks and the hwmon patch-review checklist

→ Next: [45-input-hid.md](45-input-hid.md)

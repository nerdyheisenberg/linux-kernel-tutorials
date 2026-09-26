# Chapter 41 — SPI, QSPI, and SPI-NOR

> **Goal:** understand the bus with *no protocol at all* — just a shift register — and why
> that makes it both the fastest simple bus and the one where everything is a board-specific
> convention.

---

## Theory & First Principles

### T.0 — Start here: there is no protocol

I²C (Ch. 40) has a specification: start conditions, addresses, ACK/NAK, stop conditions. SPI
has essentially none. It is **two shift registers clocked together**, and everything else is
convention.

```
   MASTER                              SLAVE
   ┌───────────────┐                  ┌───────────────┐
   │ shift register │── MOSI ────────►│ shift register │
   │  10110010      │◄─ MISO ─────────│  01101001      │
   └───────────────┘── SCLK ────────►└───────────────┘
                     ── CS   ────────►  (chip select, one per device)

   Each clock edge: one bit out, one bit in. SIMULTANEOUSLY.
   After 8 clocks the registers have SWAPPED CONTENTS.
```

**That is the entire hardware.** Which produces a strange set of properties:

| Property | Consequence |
|---|---|
| **Full duplex, always** | You cannot read without writing. To read, you clock out dummy bytes |
| **No addressing** | A separate **chip-select wire per device**. N devices = N+3 pins |
| **No acknowledgement** | The master cannot tell whether anything is connected. A read from an absent device returns 0xFF or 0x00, not an error |
| **No defined speed** | 1 MHz to 100 MHz, whatever both ends tolerate. **10–100× faster than I²C** |
| **No defined framing** | "Register read = send 0x80\|addr, then clock 8 dummy bits" is *this chip's* convention, not SPI's |
| **Four clock modes** | CPOL/CPHA: which edge samples, and the idle clock level. Get it wrong and you read garbage with no error |

**The absence of a protocol is the design.** Because SPI specifies nothing above the wire,
every chip invents its own framing — and that is why SPI drivers are the least reusable
drivers in the kernel. There is no equivalent of `i2c_smbus_read_byte_data()` that works
across vendors, because there *is* no standard register-access convention.

**So the trade against I²C is explicit**, and choosing between them is a real design decision:

| | I²C | SPI |
|---|---|---|
| Pins | **2, shared** | 3 + one CS **per device** |
| Speed | 100–400 kHz (3.4 MHz HS) | **1–100 MHz** |
| Error detection | ACK/NAK | **none** |
| Addressing | 7-bit, in-band | a wire per device |
| Use it for | many slow sensors | **fast** things: flash, displays, ADCs, radios |

**Three things that bite in practice:**

1. **CPOL/CPHA mismatch produces plausible garbage**, not an error. When a new SPI device
   returns nonsense, try all four modes before suspecting anything else.
2. **Chip select timing matters.** Some devices require CS to stay asserted across multiple
   transfers (`cs_change`), some require a delay after assertion. This is per-chip and is in
   the datasheet, not the spec.
3. **DMA has an alignment and size threshold.** Below it, PIO is faster; above it, DMA wins.
   Getting a large display update to use DMA is usually the difference between 5 fps and
   60 fps.

```bash
ls /sys/bus/spi/devices/
cat /sys/class/spi_master/spi0/*/modalias 2>/dev/null
spidev_test -D /dev/spidev0.0 -s 1000000 -v    # loopback: tie MOSI to MISO
sudo cat /sys/kernel/debug/spi*/  2>/dev/null
```

---

### T.1 SPI is not a protocol; it is two shift registers

I²C (Ch. 40) has a defined frame: START, address, ACK, data, STOP. **SPI has none of that.**
It is a synchronous serial link between two shift registers:

```
   Controller                                  Peripheral
   ┌──────────────┐    SCLK ───────────────▶  ┌──────────────┐
   │ shift reg    │    MOSI ───────────────▶  │ shift reg    │
   │  [10110010]  │ ◀───────────────── MISO   │  [01001101]  │
   └──────────────┘    CS#  ───────────────▶  └──────────────┘

   Each clock edge: one bit out of each register, one bit in.
   After 8 clocks the two registers have EXCHANGED their contents.
```

That is the entire bus. Consequences, and every one of them matters:

| SPI has no… | Consequence |
|---|---|
| **Addressing** | a separate **chip-select line per device** — N devices need N GPIOs |
| **Acknowledgement** | the controller cannot tell whether anything is connected |
| **Flow control** | the peripheral must keep up, or you must insert delays |
| **Error detection** | corruption is silent |
| **Defined frame format** | **everything is a per-device convention** |
| **Standard speed** | anything from kHz to >100 MHz |
| **Arbitration** | single master only |

And it gains: **full duplex** (both directions simultaneously), **simplicity** (a peripheral
can be a 74HC595 shift register), and **speed** (no open-drain, so push-pull at 50–100 MHz
is routine — 250× faster than standard I²C).

> **The defining fact:** *every SPI transaction is an exchange of exactly N bits in both
> directions.* To "read" you must clock out something (usually dummy bytes). A driver that
> thinks in terms of "send a command, then read a reply" is thinking in a layer the bus does
> not have — that layering is imposed entirely by the device's datasheet.

```bash
ls /sys/bus/spi/devices/            # spi0.0 = controller 0, chip select 0
cat /sys/bus/spi/devices/spi0.0/{modalias,of_node/compatible} 2>/dev/null
cat /sys/class/spi_master/spi0/of_node/compatible 2>/dev/null | tr '\0' '\n'
```

### T.2 The four modes: CPOL and CPHA

Because there is no frame format, the *only* thing both ends must agree on is **when to
sample**. Two bits define it:

- **CPOL (clock polarity)**: is SCLK idle low (0) or idle high (1)?
- **CPHA (clock phase)**: sample on the *first* edge (0) or the *second* edge (1)?

```
   Mode 0 (CPOL=0, CPHA=0)  idle low,  sample on rising edge   ← ~80% of devices
   Mode 1 (CPOL=0, CPHA=1)  idle low,  sample on falling edge
   Mode 2 (CPOL=1, CPHA=0)  idle high, sample on falling edge
   Mode 3 (CPOL=1, CPHA=1)  idle high, sample on rising edge   ← common for flash
```

The subtlety that catches people: **CPHA=0 means the first data bit must be valid *before*
the first clock edge** — i.e. it is driven on the falling edge of CS. That places a setup-time
requirement on CS assertion that some controllers handle poorly, which is why
`SPI_CS_HIGH`, `spi-cs-setup-delay-ns`, and cs-gpios exist.

```c
spi->mode = SPI_MODE_0;              /* or SPI_MODE_1/2/3 */
spi->mode |= SPI_CS_HIGH;            /* active-high chip select (unusual) */
spi->mode |= SPI_LSB_FIRST;          /* bit order (rare) */
spi->mode |= SPI_3WIRE;              /* half-duplex, shared data line */
spi->mode |= SPI_NO_CS;              /* single device, CS tied low */
spi->mode |= SPI_READY;              /* peripheral has a READY signal */
spi->bits_per_word = 8;              /* 8, 16, 32 — or odd sizes like 12 for some ADCs */
spi->max_speed_hz = 10 * 1000 * 1000;
ret = spi_setup(spi);                /* ★ validate against controller capabilities */
```

`spi_setup()` is not decoration: it asks the controller whether it can do this combination
and returns `-EINVAL` if not. **Always check its return.**

### T.3 The message/transfer model

The API's central abstraction is a **message**: an ordered list of transfers that execute
**atomically with respect to other messages on the same bus**, with fine control over chip
select between them.

```c
struct spi_transfer {
	const void *tx_buf;          /* NULL = send zeros/dummy */
	void       *rx_buf;          /* NULL = discard received data */
	unsigned    len;             /* ★ ONE length — it is an EXCHANGE (T.1) */
	dma_addr_t  tx_dma, rx_dma;
	unsigned    cs_change:1;     /* ★ deassert CS after this transfer */
	u8          bits_per_word;   /* override per transfer */
	u32         speed_hz;        /* override per transfer */
	u16         delay_usecs;     /* legacy */
	struct spi_delay delay;      /* ★ modern: after this transfer */
	struct spi_delay cs_change_delay;
	u8          tx_nbits, rx_nbits;  /* 1/2/4/8 — dual/quad/octal SPI */
	struct list_head transfer_list;
};
```

The canonical register read — note that it is **two transfers in one message**, not two
messages:

```c
static int my_read_reg(struct spi_device *spi, u8 reg, u8 *val)
{
	u8 tx[2] = { reg | 0x80, 0x00 };     /* bit 7 = read, per THIS device's datasheet */
	u8 rx[2] = { 0 };
	struct spi_transfer xfer = {
		.tx_buf = tx, .rx_buf = rx, .len = 2,
	};
	struct spi_message msg;
	int ret;

	spi_message_init(&msg);
	spi_message_add_tail(&xfer, &msg);
	ret = spi_sync(spi, &msg);           /* ★ sleeps until complete */
	if (ret)
		return ret;
	*val = rx[1];                        /* byte 0 was clocked out while we sent the cmd */
	return 0;
}
```

**Why one message and not two:** the bus lock is held for the whole message, and CS stays
asserted across transfers unless `cs_change` says otherwise. Splitting it into two
`spi_sync()` calls lets another device's message interleave, deasserting CS mid-command and
corrupting the device's state machine. **This is the SPI equivalent of Ch. 40 T.3's repeated
start.**

The convenience wrappers cover most cases:

```c
spi_write(spi, buf, len);
spi_read(spi, buf, len);
spi_write_then_read(spi, txbuf, n_tx, rxbuf, n_rx);   /* ★ the most common form */
spi_w8r8(spi, cmd);          /* write one byte, read one byte */
spi_w8r16be(spi, cmd);       /* write one byte, read a big-endian u16 */
spi_sync_transfer(spi, xfers, ARRAY_SIZE(xfers));
spi_async(spi, &msg);        /* ★ non-blocking; completion callback */
```

`spi_write_then_read()` uses a bounce buffer and is limited (`SPI_BUFSIZ`, 32 bytes by
default) but is correct and simple. For bulk transfers build the message yourself.

**Bus locking** for multi-transaction atomicity:
```c
spi_bus_lock(spi->controller);       /* exclusive access across MESSAGES */
spi_sync_locked(spi, &msg1);
spi_sync_locked(spi, &msg2);
spi_bus_unlock(spi->controller);
```

### T.4 Chip select: where boards go wrong

CS is nominally simple and is the source of most SPI bring-up problems.

**Controller-native CS** is driven by the SPI peripheral itself. It is fast and correctly
timed, but controllers usually have only 1–4 of them, and many deassert CS between
transfers *unconditionally* — which breaks devices that require CS held across a
command/data sequence.

**`cs-gpios`** lets the DT specify arbitrary GPIOs as chip selects, and the SPI core drives
them. This is now the preferred approach: unlimited chip selects, and the core guarantees CS
semantics independent of controller quirks.

```dts
spi0: spi@1c68000 {
	compatible = "allwinner,sun6i-a31-spi";
	reg = <0x01c68000 0x1000>;
	#address-cells = <1>;
	#size-cells = <0>;
	cs-gpios = <0>,                              /* CS0: native */
		   <&pio 2 3 GPIO_ACTIVE_LOW>;       /* CS1: a GPIO */

	flash@0 {
		compatible = "jedec,spi-nor";
		reg = <0>;                           /* ★ chip-select INDEX, not an address */
		spi-max-frequency = <50000000>;
		spi-tx-bus-width = <4>;              /* quad (T.6) */
		spi-rx-bus-width = <4>;
	};

	adc@1 {
		compatible = "ti,ads7950";
		reg = <1>;
		spi-max-frequency = <1000000>;
		spi-cpha;                            /* CPHA=1 */
		spi-cpol;                            /* CPOL=1  → mode 3 */
	};
};
```

Note `reg = <0>` here is a **chip-select index**, not a bus address — a different meaning from
I²C's `reg` (Ch. 40 T.2). The DT binding's meaning is always local to the bus (Ch. 32 T.6).

Timing knobs that exist because real devices are fussy:
```dts
	spi-cs-setup-delay-ns  = <100>;    /* CS low → first clock */
	spi-cs-hold-delay-ns   = <100>;    /* last clock → CS high */
	spi-cs-inactive-delay-ns = <500>;  /* minimum CS-high time between messages */
```

### T.5 The queue: `spi_sync`, `spi_async`, and the message pump

```
   driver: spi_async(spi, &msg)
        │  adds msg to ctlr->queue
        ▼
   ctlr->kworker  (a kthread_worker, RT-capable)
        │  __spi_pump_messages()
        │    ├─ ctlr->prepare_transfer_hardware()   (clocks on, DMA mapped)
        │    ├─ for each transfer:
        │    │     ctlr->transfer_one()  or  ctlr->transfer_one_message()
        │    │     (PIO or DMA; may complete via interrupt)
        │    └─ ctlr->unprepare_transfer_hardware()
        ▼
   msg->complete(msg->context)         ← your callback
```

`spi_sync()` is `spi_async()` + `wait_for_completion()`. But there is an optimization worth
knowing: if the calling context allows, the core may run the message **inline on the caller's
thread** (`ctlr->can_dma` / the "sync path") rather than waking the pump thread, which saves
two context switches. On a hot SPI path (a display, an ADC at 10 kHz) that is significant.

```c
/* Setting the pump thread's priority — matters for real-time SPI */
ctlr->rt = true;                    /* run the kworker as SCHED_FIFO */
/* or from DT: */
	spi-rt;
```

**DMA vs PIO** is a per-transfer decision made by the controller driver:
```c
static bool my_can_dma(struct spi_controller *ctlr, struct spi_device *spi,
		       struct spi_transfer *xfer)
{
	return xfer->len > MY_DMA_THRESHOLD;     /* small transfers: PIO is cheaper */
}
```
Below the threshold, DMA setup (mapping, descriptor, interrupt) costs more than just shifting
the bytes. Typical thresholds are 16–64 bytes. This is the same fixed-cost-vs-marginal-cost
reasoning as Ch. 09 T.3's array-vs-tree crossover.

Buffers for DMA must satisfy Ch. 35 T.8's rules — which is why the SPI core offers
`spi->controller->dma_alignment`, and why `spi_write_then_read()`'s bounce buffer is
`kmalloc`ed (and therefore `ARCH_DMA_MINALIGN`-aligned).

### T.6 Dual, Quad, Octal: more wires, same idea

Flash memories want more bandwidth than one data line allows. The extension is mechanical:
use MOSI and MISO (and two more pins) **as a parallel bus in one direction**.

```
   Single:  1 bit/clock   (MOSI out, MISO in — full duplex)
   Dual:    2 bits/clock  (IO0, IO1 — half duplex)
   Quad:    4 bits/clock  (IO0..IO3)
   Octal:   8 bits/clock  (IO0..IO7)
```

**Full duplex is lost** — the pins are bidirectional and the direction turns around during the
transaction. A quad read looks like:

```
   CMD (1 bit) → ADDR (4 bits) → DUMMY cycles → DATA (4 bits)
   ^ opcode      ^ address       ^ turnaround   ^ payload
```

Expressing that with `struct spi_transfer` is awkward, which is why **`spi-mem`** exists: an
abstraction for *memory-like* SPI devices with a command/address/dummy/data structure:

```c
struct spi_mem_op op = SPI_MEM_OP(
	SPI_MEM_OP_CMD(0x6B, 1),          /* opcode 0x6B on 1 line */
	SPI_MEM_OP_ADDR(3, addr, 1),      /* 3-byte address on 1 line */
	SPI_MEM_OP_DUMMY(8, 1),           /* 8 dummy cycles */
	SPI_MEM_OP_DATA_IN(len, buf, 4)); /* data in on 4 lines ← QUAD */

if (!spi_mem_supports_op(mem, &op))
	return -EOPNOTSUPP;               /* ★ ask the controller first */
ret = spi_mem_exec_op(mem, &op);
```

This lets a **dedicated QSPI controller** (which has hardware for exactly this shape, often
with an XIP memory-mapped window) and a **generic SPI controller** (which must emulate it)
present one interface. `spi_mem_supports_op()` is the negotiation.

### T.7 SPI-NOR: the layering above

A SPI flash chip is the most common SPI device, and it has its own stack:

```
   filesystem (JFFS2 / UBIFS / squashfs)  or  mtdblock
        │
   MTD core (drivers/mtd/)                ← Ch. 70
        │
   SPI-NOR framework (drivers/mtd/spi-nor/)   ← erase/write/read, SFDP parsing
        │
   spi-mem (T.6)
        │
   SPI controller  or  dedicated QSPI controller
```

**SFDP (Serial Flash Discoverable Parameters)** deserves note: it is JEDEC's answer to the
"no discovery" problem (Ch. 40 T.2, Ch. 31 T.1). A modern flash chip contains a standardized
parameter table describing its size, erase sizes, supported opcodes, dummy-cycle counts, and
4-byte-addressing support. So `drivers/mtd/spi-nor/sfdp.c` can drive a flash chip it has
never heard of.

That is a rare case of a non-discoverable bus gaining discovery *at the device level*, and it
eliminated thousands of lines of per-chip tables. The per-chip tables still exist
(`drivers/mtd/spi-nor/*.c`) for chips with broken or absent SFDP — i.e. for the usual
reason.

```bash
cat /proc/mtd
ls /sys/class/mtd/
cat /sys/class/mtd/mtd0/{name,size,erasesize,type}
sudo mtdinfo /dev/mtd0
sudo dd if=/dev/mtd0 bs=4096 count=1 | hexdump -C | head
dmesg | grep -i 'spi-nor\|SFDP\|jedec'
```

---

## 1. Internals

### 1.1 Source map

```
drivers/spi/spi.c              ★★ the core: message queue, pump, cs handling, DMA mapping
drivers/spi/spidev.c           ★ /dev/spidevB.C — userspace access
drivers/spi/spi-mem.c          ★ T.6's memory-operation abstraction
drivers/spi/spi-bitbang.c      ★ bit-banged SPI — the protocol in software
drivers/spi/spi-gpio.c         a complete bit-banged controller; ~350 lines
drivers/spi/spi-loopback-test.c ★ an in-tree test harness — run it
drivers/spi/spi-dw-*.c, spi-pl022.c, spi-imx.c, spi-bcm2835.c   real controllers
drivers/mtd/spi-nor/core.c ★, sfdp.c ★, winbond.c, macronix.c   T.7
include/linux/spi/spi.h        ★★ read the whole header
include/linux/spi/spi-mem.h
include/uapi/linux/spi/spidev.h
Documentation/spi/             ★ spi-summary.rst, spidev.rst, pxa2xx.rst, butterfly.rst
Documentation/devicetree/bindings/spi/   ★ spi-controller.yaml, spi-peripheral-props.yaml
```

### 1.2 Writing a controller driver

```c
static int my_transfer_one(struct spi_controller *ctlr, struct spi_device *spi,
			   struct spi_transfer *xfer)
{
	struct my_spi *s = spi_controller_get_devdata(ctlr);

	my_set_clock(s, xfer->speed_hz);
	my_set_bits(s, xfer->bits_per_word);
	my_start_transfer(s, xfer);
	return 1;             /* ★ 1 = in progress; call spi_finalize_current_transfer() later */
}

static int my_probe(struct platform_device *pdev)
{
	struct spi_controller *ctlr;
	struct my_spi *s;

	ctlr = devm_spi_alloc_host(&pdev->dev, sizeof(*s));    /* ★ was spi_alloc_master */
	if (!ctlr)
		return -ENOMEM;
	s = spi_controller_get_devdata(ctlr);

	ctlr->mode_bits      = SPI_CPOL | SPI_CPHA | SPI_CS_HIGH | SPI_LSB_FIRST;
	ctlr->bits_per_word_mask = SPI_BPW_MASK(8) | SPI_BPW_MASK(16);
	ctlr->min_speed_hz   = 1000;
	ctlr->max_speed_hz   = 50000000;
	ctlr->num_chipselect = 4;
	ctlr->use_gpio_descriptors = true;           /* ★ let the core handle cs-gpios */
	ctlr->transfer_one   = my_transfer_one;
	ctlr->set_cs         = my_set_cs;
	ctlr->can_dma        = my_can_dma;
	ctlr->prepare_transfer_hardware   = my_prepare_hw;
	ctlr->unprepare_transfer_hardware = my_unprepare_hw;
	ctlr->auto_runtime_pm = true;
	ctlr->dev.of_node    = pdev->dev.of_node;

	return devm_spi_register_controller(&pdev->dev, ctlr);
}
```

Terminology note: the kernel renamed `master`/`slave` to `host`/`target` (and
`spi_alloc_master` → `spi_alloc_host`). Both exist during the transition; new code uses the
new names.

---

## 2. Practice

### Lab 41.1 — SPI without hardware

```bash
# (a) The loopback test module — exercises the whole core
sudo modprobe spi-loopback-test
dmesg | tail -40

# (b) spi-gpio: a real bit-banged controller over gpio-sim
sudo modprobe gpio-sim
# (configfs setup for gpio-sim — see Ch. 42)
sudo modprobe spi-gpio

# (c) A DT overlay wiring it up (Ch. 32 Lab 32.5)
cat > /tmp/spi-gpio.dts <<'EOF'
/dts-v1/;
/plugin/;
&{/} {
	spi-gpio-0 {
		compatible = "spi-gpio";
		#address-cells = <1>;
		#size-cells = <0>;
		sck-gpios  = <&gpio0 0 GPIO_ACTIVE_HIGH>;
		mosi-gpios = <&gpio0 1 GPIO_ACTIVE_HIGH>;
		miso-gpios = <&gpio0 2 GPIO_ACTIVE_HIGH>;
		cs-gpios   = <&gpio0 3 GPIO_ACTIVE_LOW>;
		num-chipselects = <1>;

		spidev@0 {
			compatible = "rohm,dh2228fv";   /* a generic spidev-compatible */
			reg = <0>;
			spi-max-frequency = <1000000>;
		};
	};
};
EOF
dtc -@ -I dts -O dtb -o /tmp/spi-gpio.dtbo /tmp/spi-gpio.dts

ls /sys/bus/spi/devices/
ls /dev/spidev*
```

### Lab 41.2 — A complete SPI sensor/ADC driver

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * adcdemo — a SPI ADC driver.
 *
 * Demonstrates: the message/transfer model (T.3), mode negotiation (T.2),
 * spi_async with a completion callback, IIO registration, and DMA-safe buffers.
 */
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

#include <linux/bitfield.h>
#include <linux/iio/iio.h>
#include <linux/iio/buffer.h>
#include <linux/iio/trigger_consumer.h>
#include <linux/iio/triggered_buffer.h>
#include <linux/module.h>
#include <linux/regulator/consumer.h>
#include <linux/spi/spi.h>

#define ADCD_CMD_READ(ch)   (0x80 | ((ch) << 3))
#define ADCD_NUM_CHANNELS   8

struct adcd_variant {
	const char *name;
	int         bits;
	u32         max_speed_hz;
};
static const struct adcd_variant adcd_12bit = { "adcdemo-12", 12, 10000000 };
static const struct adcd_variant adcd_16bit = { "adcdemo-16", 16, 20000000 };

struct adcd {
	struct spi_device         *spi;
	const struct adcd_variant *var;
	struct regulator          *vref;
	int                        vref_mV;
	struct mutex               lock;

	/* ★ Ch. 35 T.8: DMA-safe buffers must not share a cache line with other data */
	struct {
		__be16 sample[ADCD_NUM_CHANNELS];
		aligned_s64 timestamp;
	} scan __aligned(IIO_DMA_MINALIGN);

	u8  tx_buf[2] __aligned(IIO_DMA_MINALIGN);
	u8  rx_buf[2];
};

static int adcd_read_channel(struct adcd *a, int ch, int *val)
{
	struct spi_transfer xfer = {
		.tx_buf = a->tx_buf,
		.rx_buf = a->rx_buf,
		.len    = 2,
		.speed_hz = a->var->max_speed_hz,
		.delay  = { .value = 1, .unit = SPI_DELAY_UNIT_USECS },  /* conversion time */
	};
	struct spi_message msg;
	int ret;

	guard(mutex)(&a->lock);

	a->tx_buf[0] = ADCD_CMD_READ(ch);
	a->tx_buf[1] = 0x00;                       /* ★ T.1: must clock out something to read */

	spi_message_init_with_transfers(&msg, &xfer, 1);
	ret = spi_sync(a->spi, &msg);
	if (ret)
		return ret;

	*val = ((a->rx_buf[0] << 8) | a->rx_buf[1]) >> (16 - a->var->bits);
	return 0;
}

/* ---- IIO: the right subsystem for an ADC (Ch. 29 T.9) ---- */

static int adcd_read_raw(struct iio_dev *indio, struct iio_chan_spec const *chan,
			 int *val, int *val2, long mask)
{
	struct adcd *a = iio_priv(indio);
	int ret;

	switch (mask) {
	case IIO_CHAN_INFO_RAW:
		if (!iio_device_claim_direct(indio))
			return -EBUSY;
		ret = adcd_read_channel(a, chan->channel, val);
		iio_device_release_direct(indio);
		return ret ?: IIO_VAL_INT;

	case IIO_CHAN_INFO_SCALE:
		*val  = a->vref_mV;
		*val2 = a->var->bits;
		return IIO_VAL_FRACTIONAL_LOG2;         /* mV per LSB */

	default:
		return -EINVAL;
	}
}

static const struct iio_info adcd_info = { .read_raw = adcd_read_raw };

#define ADCD_CHAN(idx) {                                     \
	.type = IIO_VOLTAGE,                                 \
	.indexed = 1, .channel = (idx),                      \
	.info_mask_separate = BIT(IIO_CHAN_INFO_RAW),        \
	.info_mask_shared_by_type = BIT(IIO_CHAN_INFO_SCALE),\
	.scan_index = (idx),                                 \
	.scan_type = { .sign = 'u', .realbits = 12,          \
		       .storagebits = 16, .endianness = IIO_BE }, \
}
static const struct iio_chan_spec adcd_channels[] = {
	ADCD_CHAN(0), ADCD_CHAN(1), ADCD_CHAN(2), ADCD_CHAN(3),
	ADCD_CHAN(4), ADCD_CHAN(5), ADCD_CHAN(6), ADCD_CHAN(7),
	IIO_CHAN_SOFT_TIMESTAMP(8),
};

/* Triggered buffer: read all enabled channels on each trigger */
static irqreturn_t adcd_trigger_handler(int irq, void *p)
{
	struct iio_poll_func *pf = p;
	struct iio_dev *indio = pf->indio_dev;
	struct adcd *a = iio_priv(indio);
	int i, val, j = 0;

	iio_for_each_active_channel(indio, i) {
		if (adcd_read_channel(a, i, &val) == 0)
			a->scan.sample[j++] = cpu_to_be16(val);
	}
	iio_push_to_buffers_with_timestamp(indio, &a->scan, pf->timestamp);
	iio_trigger_notify_done(indio->trig);
	return IRQ_HANDLED;
}

static int adcd_probe(struct spi_device *spi)
{
	struct device *dev = &spi->dev;
	struct iio_dev *indio;
	struct adcd *a;
	int ret;

	indio = devm_iio_device_alloc(dev, sizeof(*a));
	if (!indio)
		return -ENOMEM;
	a = iio_priv(indio);
	a->spi = spi;
	mutex_init(&a->lock);
	spi_set_drvdata(spi, indio);

	a->var = spi_get_device_match_data(spi);
	if (!a->var)
		return -ENODEV;

	/* ★ T.2: negotiate the mode with the controller and CHECK the result */
	spi->mode          = SPI_MODE_0;
	spi->bits_per_word = 8;
	if (spi->max_speed_hz > a->var->max_speed_hz)
		spi->max_speed_hz = a->var->max_speed_hz;
	ret = spi_setup(spi);
	if (ret)
		return dev_err_probe(dev, ret, "controller cannot do mode 0 @ %u Hz\n",
				     spi->max_speed_hz);

	a->vref = devm_regulator_get(dev, "vref");
	if (IS_ERR(a->vref))
		return dev_err_probe(dev, PTR_ERR(a->vref), "vref\n");
	ret = regulator_enable(a->vref);
	if (ret)
		return ret;
	ret = devm_add_action_or_reset(dev, (void (*)(void *))regulator_disable, a->vref);
	if (ret)
		return ret;

	ret = regulator_get_voltage(a->vref);
	if (ret < 0)
		return ret;
	a->vref_mV = ret / 1000;

	indio->name          = a->var->name;
	indio->modes         = INDIO_DIRECT_MODE;
	indio->info          = &adcd_info;
	indio->channels      = adcd_channels;
	indio->num_channels  = ARRAY_SIZE(adcd_channels);

	ret = devm_iio_triggered_buffer_setup(dev, indio, NULL,
					      adcd_trigger_handler, NULL);
	if (ret)
		return ret;

	dev_info(dev, "%s, %d-bit, vref %d mV, %u Hz, mode %d\n",
		 a->var->name, a->var->bits, a->vref_mV, spi->max_speed_hz,
		 spi->mode & (SPI_CPOL | SPI_CPHA));

	return devm_iio_device_register(dev, indio);
}

static const struct spi_device_id adcd_ids[] = {
	{ "adcdemo-12", (kernel_ulong_t)&adcd_12bit },
	{ "adcdemo-16", (kernel_ulong_t)&adcd_16bit },
	{ }
};
MODULE_DEVICE_TABLE(spi, adcd_ids);

static const struct of_device_id adcd_of_ids[] = {
	{ .compatible = "acme,adcdemo-12", .data = &adcd_12bit },
	{ .compatible = "acme,adcdemo-16", .data = &adcd_16bit },
	{ }
};
MODULE_DEVICE_TABLE(of, adcd_of_ids);

static struct spi_driver adcd_driver = {
	.driver = { .name = "adcdemo", .of_match_table = adcd_of_ids },
	.probe    = adcd_probe,
	.id_table = adcd_ids,
};
module_spi_driver(adcd_driver);
MODULE_LICENSE("GPL");
```

```bash
sudo insmod adcdemo.ko
ls /sys/bus/iio/devices/iio:device0/
cat /sys/bus/iio/devices/iio:device0/in_voltage0_raw
cat /sys/bus/iio/devices/iio:device0/in_voltage_scale
# The whole IIO userspace toolchain works with no extra code:
iio_info
iio_readdev -b 64 iio:device0
```

### Lab 41.3 — Userspace SPI with `spidev`

```c
/* spiuser.c — gcc -O2 -o spiuser spiuser.c */
#include <fcntl.h>
#include <linux/spi/spidev.h>
#include <stdio.h>
#include <string.h>
#include <sys/ioctl.h>
#include <unistd.h>

int main(int argc, char **argv)
{
	int fd = open(argv[1], O_RDWR);
	uint8_t mode = SPI_MODE_0, bits = 8;
	uint32_t speed = 1000000;
	uint8_t tx[4] = { 0x9F, 0, 0, 0 };     /* JEDEC READ ID */
	uint8_t rx[4] = { 0 };
	struct spi_ioc_transfer tr = {
		.tx_buf = (unsigned long)tx,
		.rx_buf = (unsigned long)rx,
		.len = 4,
		.speed_hz = speed,
		.bits_per_word = bits,
		.delay_usecs = 0,
		.cs_change = 0,
	};

	ioctl(fd, SPI_IOC_WR_MODE, &mode);
	ioctl(fd, SPI_IOC_WR_BITS_PER_WORD, &bits);
	ioctl(fd, SPI_IOC_WR_MAX_SPEED_HZ, &speed);

	/* Read back what the controller actually accepted (T.2) */
	ioctl(fd, SPI_IOC_RD_MODE, &mode);
	ioctl(fd, SPI_IOC_RD_MAX_SPEED_HZ, &speed);
	printf("mode=%d bits=%d speed=%u\n", mode, bits, speed);

	if (ioctl(fd, SPI_IOC_MESSAGE(1), &tr) < 0)
		perror("SPI_IOC_MESSAGE");
	else
		printf("JEDEC ID: %02x %02x %02x\n", rx[1], rx[2], rx[3]);

	close(fd);
	return 0;
}
```
```bash
ls /dev/spidev*
sudo ./spiuser /dev/spidev0.0
# spi-tools has ready-made equivalents:
sudo spi-config -d /dev/spidev0.0 -q
echo -ne '\x9f\x00\x00\x00' | sudo spi-pipe -d /dev/spidev0.0 | hexdump -C
$EDITOR Documentation/spi/spidev.rst
```

### Lab 41.4 — Watch a transfer on a logic analyzer

```bash
# Kernel-side trace
sudo trace-cmd record -e spi -- sleep 5
trace-cmd report | head -40

sudo bpftrace -e '
tracepoint:spi:spi_message_start  { @msgs = count(); }
tracepoint:spi:spi_transfer_start { @xfers = count(); @bytes = sum(args->len); }
tracepoint:spi:spi_message_done   { @done = count(); }
interval:s:5 { print(@msgs); print(@xfers); print(@bytes); clear(@msgs); clear(@xfers); clear(@bytes); }'

# Latency per message
sudo bpftrace -e '
kprobe:spi_sync   { @s[tid] = nsecs; }
kretprobe:spi_sync /@s[tid]/ { @us = hist((nsecs - @s[tid])/1000); delete(@s[tid]); }'

# Controller statistics (per-device and per-controller)
ls /sys/class/spi_master/spi0/statistics/
cat /sys/class/spi_master/spi0/statistics/{messages,transfers,bytes,errors,timedout}
cat /sys/bus/spi/devices/spi0.0/statistics/* 2>/dev/null
```
**Then capture with PulseView** and decode. Verify: the mode (which edge samples?), the
actual clock frequency, CS timing, and that your two-transfer message keeps CS asserted
throughout. **The CS-held-across-transfers check is the one that catches real bugs.**

### Lab 41.5 — Prove the atomicity of a message (T.3)

Write two drivers on the same bus, both doing register reads in a tight loop. Implement the
read (a) as one two-transfer message, (b) as two separate `spi_sync()` calls with
`cs_change`. Under (b), instrument for interleaving:

```c
/* In the completion path, detect a corrupted response */
if (rx[1] != expected_echo)
	atomic_inc(&corruptions);
```
```bash
# Then watch CS on a logic analyzer: under (b) you will see CS deassert
# between the command and the data phase, and another device's traffic interleave.
sudo bpftrace -e 'tracepoint:spi:spi_set_cs { @[args->bus_num, args->chip_select, args->enable] = count(); }'
```

### Lab 41.6 — SPI-NOR and SFDP (T.7)

```bash
# On a board with SPI flash, or in QEMU with -drive if=mtd
cat /proc/mtd
sudo mtdinfo -a
dmesg | grep -iE 'spi-nor|SFDP|jedec|mx25|w25q'

# The SFDP table itself:
sudo cat /sys/kernel/debug/spi-nor/spi0.0/sfdp 2>/dev/null | hexdump -C | head -20
sudo cat /sys/kernel/debug/spi-nor/spi0.0/params 2>/dev/null
sudo cat /sys/kernel/debug/spi-nor/spi0.0/capabilities 2>/dev/null

# Read the flash
sudo dd if=/dev/mtd0 bs=4096 count=1 2>/dev/null | hexdump -C | head
sudo flash_erase /dev/mtd0 0 1        # ★ DESTRUCTIVE — only on a scratch device
sudo flashcp -v /tmp/image.bin /dev/mtd0

# The parsing code:
$EDITOR drivers/mtd/spi-nor/sfdp.c     # spi_nor_parse_sfdp, BFPT, 4BAIT, SCCR
grep -n 'SFDP_\|BFPT_DWORD' drivers/mtd/spi-nor/sfdp.h | head -30
```

### Lab 41.7 — Dual/Quad and `spi-mem` (T.6)

```c
static int qspi_fast_read(struct spi_mem *mem, u32 addr, void *buf, size_t len)
{
	struct spi_mem_op op = SPI_MEM_OP(
		SPI_MEM_OP_CMD(0x6B, 1),           /* Quad Output Fast Read */
		SPI_MEM_OP_ADDR(3, addr, 1),
		SPI_MEM_OP_DUMMY(8, 1),
		SPI_MEM_OP_DATA_IN(len, buf, 4));  /* ★ 4 lines */

	if (!spi_mem_supports_op(mem, &op)) {
		/* Fall back to single-line */
		op.data.buswidth = 1;
		op.cmd.opcode = 0x0B;
		if (!spi_mem_supports_op(mem, &op))
			return -EOPNOTSUPP;
	}
	spi_mem_adjust_op_size(mem, &op);          /* controller FIFO limits */
	return spi_mem_exec_op(mem, &op);
}
```
```bash
# Measure the bandwidth difference:
sudo dd if=/dev/mtd0 of=/dev/null bs=1M count=8
# with spi-rx-bus-width = <1> vs <4> in the DT
grep -rn 'spi-rx-bus-width\|spi-tx-bus-width' arch/*/boot/dts/ | head
$EDITOR drivers/spi/spi-mem.c
```

---

## 3. Mastery drills

1. **Read `drivers/spi/spi-bitbang.c` and `spi-gpio.c`.** They implement SPI in software over
   GPIOs. Identify where CPOL/CPHA are honoured and draw the resulting waveform for each of
   the four modes.

2. **The exchange model.** Explain why `spi_read()` must clock out data, and what the
   peripheral sees during a "read". Then find a device whose datasheet specifies what must be
   sent during the read phase, and one that says "don't care".

3. **Mode determination.** Given only a datasheet timing diagram, determine CPOL and CPHA.
   Practise on three real datasheets (an ADC, a display controller, a flash chip).

4. **Message atomicity.** Construct the exact failure that occurs when a command and its data
   phase are split into two messages on a shared bus. Then find a driver that uses
   `spi_bus_lock()` and explain why a single message was insufficient there.

5. **CS semantics.** Read `spi_set_cs()` in `drivers/spi/spi.c`. Explain `cs_change`,
   `SPI_CS_HIGH`, `use_gpio_descriptors`, and the three CS delay properties. Construct a
   device that requires each.

6. **DMA threshold.** Instrument a controller driver's `can_dma()` and measure transfer
   latency versus length for PIO and DMA. Find the crossover. Explain the fixed costs on each
   side (Ch. 35 T.8, Ch. 09 T.3).

7. **The pump thread.** Read `__spi_pump_messages()`. Explain the sync-path optimization
   (running inline on the caller's thread) and when it applies. Measure the difference with
   `ctlr->rt` on and off under RT load.

8. **spi-mem.** Read `drivers/spi/spi-mem.c` and one dedicated QSPI controller
   (`spi-cadence-quadspi.c` or `spi-mxic.c`). Explain what `spi_mem_supports_op()` must check
   and why a generic SPI controller can emulate only some operations.

9. **SFDP.** Read `drivers/mtd/spi-nor/sfdp.c`'s BFPT parsing. List what the table tells you.
   Then explain why per-chip tables still exist and find a chip whose SFDP the kernel
   overrides (`git grep -n 'fixups' drivers/mtd/spi-nor/*.c`).

10. **Compare the buses.** Produce a table comparing I²C (Ch. 40) and SPI on: wires, speed,
    addressing, error detection, discovery, multi-master, power, and typical use. Then state
    the rule you would give a hardware engineer for choosing between them.

11. **Design question.** A board has: a 128 Mbit QSPI flash (boot), a 1 MHz 16-bit ADC
    sampled at 10 kHz (hard real-time), a 60 MHz display controller, and a slow EEPROM —
    all on one SPI controller with two native chip selects. Design the topology, DT, and
    driver strategy. How do you guarantee the ADC's timing? What do `spi-rt`, `spi_async`,
    and per-transfer `speed_hz` buy you? Would you add a second controller?

---

## 4. Further reading

**Specifications (there is no official SPI spec — that is the point of T.1):**
- Motorola's original SPI block description (in any 68HC11 datasheet) — the *de facto* spec
- JEDEC JESD216 (**SFDP**) — T.7's discovery mechanism
- JEDEC eXtended SPI / xSPI (JESD251) — octal and DDR
- Individual device datasheets **are** the protocol specification. Read three.

**Kernel documentation:**
- `Documentation/spi/spi-summary.rst` ★★ — the core concepts, well written
- `Documentation/spi/spidev.rst` ★ — the userspace ABI
- `Documentation/devicetree/bindings/spi/spi-controller.yaml` ★ and
  `spi-peripheral-props.yaml` ★ — every standard property
- `Documentation/driver-api/mtd/spi-nor.rst`

**Source:**
- `drivers/spi/spi.c` ★★ — the message pump, CS handling, DMA mapping
- `drivers/spi/spi-bitbang.c`, `spi-gpio.c` ★ — the protocol in software
- `drivers/spi/spi-mem.c` ★ — T.6
- `drivers/spi/spi-loopback-test.c` — an executable specification of the transfer model
- `drivers/mtd/spi-nor/core.c`, `sfdp.c` ★
- Exemplary clients: `drivers/iio/adc/ti-ads7950.c`, `drivers/gpu/drm/tiny/` (SPI displays),
  `drivers/net/ethernet/micrel/ks8851_spi.c`

**Tools:**
- `spidev_test` (in `tools/spi/`) ★ — build it and use it
- `spi-tools` (`spi-config`, `spi-pipe`)
- `sigrok`/PulseView with a logic analyzer — **essential for SPI bring-up**
- `flashrom`, `mtd-utils` (`mtdinfo`, `flash_erase`, `flashcp`, `nanddump`)

**LWN & talks:**
- "The SPI subsystem" / "SPI and the device tree"
- "spi-mem: a new abstraction for SPI memories" (Boris Brezillon)
- Mark Brown's ELC talks on the SPI and regmap subsystems

→ Next: [42-gpio-pinctrl.md](42-gpio-pinctrl.md)

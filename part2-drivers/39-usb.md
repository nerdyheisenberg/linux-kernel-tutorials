# Chapter 39 — USB: Descriptors, URBs, Host and Gadget

> **Goal:** understand the bus that got hot-plug right. Write a USB device driver, understand
> why every transfer is an asynchronous *request* rather than a function call, and know how
> the same Linux machine can be a host or a device.

---

## Theory & First Principles

### T.0 — Start here: why a USB driver looks nothing like a PCI driver

A PCI driver maps registers and reads them. Try that with USB:

```c
void __iomem *base = pci_iomap(...);   /* PCI: the device's registers ARE memory */
u32 status = readl(base + 0x40);       /* a load instruction. ~100 ns. */
```

**There is no equivalent.** A USB device is not on the memory bus and has no MMIO. It is at
the end of a **serial cable**, and the only way to interact with it is to send a *message*
and wait for a reply:

```c
usb_control_msg(udev, usb_rcvctrlpipe(udev, 0),
                USB_REQ_GET_STATUS, USB_DIR_IN | USB_RECIP_DEVICE,
                0, 0, &status, sizeof(status), 1000 /* ms timeout */);
/*                                                    ^^^^^^^^^^^
 *  Milliseconds. This SLEEPS. It can FAIL. It can TIME OUT.        */
```

**That single difference propagates into everything:**

| | PCI | USB |
|---|---|---|
| Register access | a load/store, ~100 ns, cannot fail | a **message**, ~1 ms, can fail or time out |
| Can you do it in an interrupt handler? | yes | **no** — it sleeps |
| Who initiates transfers? | the device (bus master DMA) | **the host, always.** Devices cannot speak unbidden |
| Interrupt delivery | the device raises a real IRQ | the host **polls** on a schedule ("interrupt transfer") |
| Errors | rare; a bus error is fatal | **normal.** Cables get unplugged mid-transfer |

**"The host initiates everything" is the load-bearing fact.** USB has no device-initiated
communication at all — what is called an *interrupt endpoint* is the host asking "anything
for me?" every N milliseconds. That design choice made devices dramatically cheaper (no bus
arbitration logic, no DMA engine) which is exactly why USB won the peripheral market and
Firewire did not.

**So the programming model is asynchronous by necessity**, and the URB is the consequence:

```c
struct urb *urb = usb_alloc_urb(0, GFP_KERNEL);
usb_fill_bulk_urb(urb, udev, usb_rcvbulkpipe(udev, ep),
                  buf, len, my_completion, ctx);
usb_submit_urb(urb, GFP_KERNEL);    /* returns IMMEDIATELY. */
/* ... my_completion() is called later, in interrupt context ... */
```

A **URB (USB Request Block)** is a self-describing I/O request with a completion callback —
the same shape as a `bio` in the block layer (Ch. 63) and an `sk_buff` on the transmit path
(Ch. 71). **When a transport is slow and failure-prone, the interface becomes a queue of
request objects with callbacks**, and that convergence across three unrelated subsystems is
worth noticing.

**And the four transfer types are a QoS taxonomy**, which is unusual for a peripheral bus and
explains the rest of the design:

| Type | Guarantees | Used by |
|---|---|---|
| **Control** | reliable, low bandwidth | enumeration, configuration — every device has endpoint 0 |
| **Bulk** | reliable, **no timing guarantee**, uses leftover bandwidth | storage, network, printers |
| **Interrupt** | bounded **latency**, small, polled | keyboards, mice, HID |
| **Isochronous** | guaranteed **bandwidth**, **no retransmission** | audio, video — a late frame is worse than a lost one |

Isochronous is the interesting one: it deliberately abandons reliability, because for audio a
retransmitted sample arriving 10 ms late is useless. **Reliability is not always the goal**,
and a transport that lets you say so is more useful than one that does not — the same
argument as UDP versus TCP.

```bash
lsusb -t                          # the topology: hubs, and what is behind them
lsusb -v -d 046d: 2>/dev/null | head -40   # descriptors: endpoints, classes
cat /sys/kernel/debug/usb/devices | head -30
sudo modprobe usbmon && sudo cat /sys/kernel/debug/usb/usbmon/0u | head  # sniff it
```

---

### T.1 USB solved a different problem than PCI

PCI (Ch. 37) answered *"how does hardware inside the box describe itself?"* USB (1996)
answered a harder question: *"how does hardware the user plugs in at random describe itself,
over a cable, with power, at low cost, with no configuration?"*

The consequences shape everything:

| | PCI/PCIe | USB |
|---|---|---|
| Topology | tree of bridges, memory-mapped | **tree of hubs, no memory mapping at all** |
| Device addressing | BDF in config space | a 7-bit address **assigned by the host** at enumeration |
| Data movement | device **DMAs** into host memory | **the host polls**; the device never initiates |
| Interrupts | device raises MSI/INTx | **there are none** — "interrupt" transfers are polled |
| Discovery | scan config space | **hub port status change → get descriptors over the wire** |
| Hotplug | bolted on later | **designed in from day one** |
| Power | from the slot | **negotiated over the same cable** |
| Cost target | expensive silicon acceptable | **must be pennies** |

**The single most important architectural fact:** *USB is host-scheduled.* A USB device
cannot initiate a transaction. It cannot DMA. It cannot raise an interrupt. Everything happens
because the host controller asked. A mouse "interrupt" is the host polling the mouse every
8 ms and usually getting a NAK.

This is a deliberate cost trade: it makes devices cheap and dumb (no bus mastering, no address
decoding, no arbitration), and it makes the *host controller* complicated. It also means
**bandwidth and latency are scheduled resources**, which is why USB has bandwidth reservation
and why plugging in a webcam can make your audio interface stop working.

### T.2 The descriptor hierarchy: self-description over a wire

Since there is no config space, a USB device describes itself by **returning descriptor
bytes when asked**:

```
Device Descriptor            (one per device: VID, PID, class, #configurations)
 └── Configuration Descriptor (power budget, #interfaces; ★ only ONE active at a time)
      ├── Interface Descriptor  (a FUNCTION: class/subclass/protocol)
      │    ├── Alternate Setting 0  (bandwidth = 0, the "idle" setting)
      │    ├── Alternate Setting 1  (e.g. 128 kB/s isochronous)
      │    └── Endpoint Descriptors (address, direction, type, max packet, interval)
      └── Interface Descriptor  (another function — e.g. audio control + audio streaming)
```

Two structural facts that drive the Linux driver model:

**(a) The *interface*, not the device, is the unit of driver binding.** A USB headset is one
device with three interfaces: audio control, audio streaming, and HID (the volume buttons).
Three different drivers bind to the same physical device. Hence `struct usb_interface` is what
`usb_driver->probe()` receives, and `MODULE_DEVICE_TABLE(usb, ...)` matches interfaces.

**(b) Alternate settings are a bandwidth-reservation mechanism.** An isochronous interface
starts in alt-setting 0 which reserves *no* bandwidth. The driver calls
`usb_set_interface()` to switch to a setting with real endpoints only when streaming starts.
This is how the host controller can admission-control isochronous bandwidth — and why a
webcam that is open but not streaming consumes nothing.

```bash
lsusb
lsusb -t                                  # the topology, with drivers
lsusb -v -d 046d:c52b | head -60          # full descriptors
cat /sys/kernel/debug/usb/devices | head -40   # the classic text dump
ls /sys/bus/usb/devices/                  # 1-1, 1-1:1.0  (device vs interface)
cat /sys/bus/usb/devices/1-1/{idVendor,idProduct,bNumInterfaces,bMaxPower,speed}
```

Note the sysfs naming: `1-1` is a *device* (bus 1, port 1); `1-1:1.0` is *configuration 1,
interface 0* — the thing a driver binds to.

### T.3 Endpoints and the four transfer types

An **endpoint** is a unidirectional data pipe with a fixed type. The four types exist because
USB must serve mice, disks, speakers, and printers on one wire, and those have irreconcilable
requirements:

| Type | Guarantee | Error handling | Bandwidth | Used by |
|---|---|---|---|---|
| **Control** | best-effort, but **reserved 10%** | retried | small | enumeration, every device (EP0) |
| **Bulk** | **no latency guarantee**, uses leftover bandwidth | **retried, guaranteed delivery** | maximal | storage, network, printers |
| **Interrupt** | **bounded latency** (polled every N frames) | retried | small, reserved | mice, keyboards, HID |
| **Isochronous** | **guaranteed bandwidth and timing** | ★ **no retry — data is dropped** | reserved | audio, video |

The two extremes are the interesting ones:

- **Bulk trades latency for reliability**: it gets whatever bandwidth is left, but it never
  loses data. A USB drive may stall for a millisecond; it will not corrupt.
- **Isochronous trades reliability for timing**: a dropped audio packet is *better* than a
  late one, so there is no retry at all. Your driver must handle missing data.

This is the classic real-time/best-effort split, implemented at the bus level. Recognizing it
explains why USB audio glitches when you plug in a busy webcam: both reserved isochronous
bandwidth from the same budget.

**Endpoint 0** is special: it is bidirectional, always present, control-type, and is how
enumeration happens before the device has an address.

### T.4 The URB: every transfer is an asynchronous request

The **URB (USB Request Block)** is the central abstraction, and its shape follows directly
from T.1. Because the host schedules everything, a driver cannot say "read 64 bytes now"; it
says "here is a request; call me when the controller gets around to it."

```c
struct urb {
	struct usb_device *dev;
	unsigned int       pipe;            /* endpoint + direction + type, encoded */
	void              *transfer_buffer;
	dma_addr_t         transfer_dma;
	u32                transfer_buffer_length;
	u32                actual_length;   /* ★ filled in on completion — SHORT reads are normal */
	int                status;          /* ★ 0, -EPIPE (stall), -ESHUTDOWN, -ENOENT, ... */
	usb_complete_t     complete;        /* ★ called in INTERRUPT context */
	void              *context;
	int                interval;        /* interrupt/isoc polling interval */
	int                number_of_packets;
	struct usb_iso_packet_descriptor iso_frame_desc[];
	...
};
```

The lifecycle is a textbook Ch. 25 P7 (completion) plus P12 (quiescence):

```c
urb = usb_alloc_urb(0, GFP_KERNEL);
usb_fill_bulk_urb(urb, udev, usb_rcvbulkpipe(udev, ep_addr),
		  buf, len, my_complete, priv);
ret = usb_submit_urb(urb, GFP_KERNEL);     /* returns IMMEDIATELY */
...
/* later, in interrupt context: */
static void my_complete(struct urb *urb)
{
	if (urb->status) {
		if (urb->status == -ENOENT || urb->status == -ECONNRESET ||
		    urb->status == -ESHUTDOWN)
			return;                 /* ★ we are being torn down: just return */
		dev_err_ratelimited(dev, "urb failed: %d\n", urb->status);
	}
	process(urb->transfer_buffer, urb->actual_length);
	usb_submit_urb(urb, GFP_ATOMIC);        /* ★ resubmit for continuous streaming */
}
...
usb_kill_urb(urb);        /* ★ P12: cancel AND wait for the completion to finish */
usb_free_urb(urb);
```

Four rules that are the difference between a working and a crashing USB driver:

1. **The completion handler runs in interrupt context.** No sleeping, no mutexes, no
   `GFP_KERNEL`. Resubmit with `GFP_ATOMIC`.
2. **`usb_kill_urb()` blocks** until any in-flight completion has finished and guarantees no
   further completion will run. `usb_unlink_urb()` is the async variant and does *not*
   provide that guarantee. **Use `kill` in teardown paths.** (`usb_poison_urb()` additionally
   makes future submissions fail — the right tool when you must stop permanently.)
3. **Short transfers are normal.** `actual_length < transfer_buffer_length` is not an error
   unless you set `URB_SHORT_NOT_OK`. Bulk reads routinely return less than requested.
4. **`-EPIPE` means the endpoint stalled** (a protocol error). Recover with
   `usb_clear_halt()`; do not just resubmit.

**Anchors** (`struct usb_anchor`) are the idiomatic way to manage a set of URBs:

```c
usb_anchor_urb(urb, &priv->submitted);
usb_submit_urb(urb, GFP_KERNEL);
...
usb_kill_anchored_urbs(&priv->submitted);       /* kill them ALL at disconnect */
usb_wait_anchor_empty_timeout(&priv->submitted, 1000);
```
Use them. Hand-rolled URB lists are a recurring source of disconnect-time UAFs.

### T.5 Enumeration: how a device gets an address

When a hub reports a port status change:

```
1. Hub reports "port connect change" via its own INTERRUPT endpoint
2. Host: reset the port  → the device is now at address 0, default state
3. GET_DESCRIPTOR(Device, 8 bytes)  → learn bMaxPacketSize0
4. Reset again
5. SET_ADDRESS(n)                   → the device now owns address n
6. GET_DESCRIPTOR(Device, full)     → VID/PID/class
7. GET_DESCRIPTOR(Configuration)    → the whole config tree, in one blob
8. SET_CONFIGURATION(1)             → the device is now "configured"
9. For each interface: match a driver, call probe()
```

Two things to notice:

- **Address 0 is the shared bootstrap address.** Only one device may be in the default state
  at a time, which is why enumeration is serialized per bus and why plugging in ten devices
  at once is slow.
- **Step 7 returns the entire configuration tree as one contiguous byte blob** — device,
  interface, endpoint, and class-specific descriptors concatenated. Parsing it is a
  walk over a `bLength`-linked list, which is why `usb_get_extra_descriptor()` and manual
  parsing appear in class drivers.

**Power negotiation** is part of configuration: `bMaxPower` in the configuration descriptor
requests current in 2 mA units, and the hub/host may refuse. USB-C/PD extends this
enormously (up to 240 W) via a separate protocol on the CC pins, handled by the
`typec`/`tcpm` subsystems.

```bash
sudo udevadm monitor --kernel --subsystem-match=usb &
# plug something in
dmesg | grep -i 'usb 1-1'
#   usb 1-1: new high-speed USB device number 5 using xhci_hcd
#   usb 1-1: New USB device found, idVendor=046d, idProduct=c52b
#   usb 1-1: Product: USB Receiver
```

### T.6 The host-controller abstraction: HCD

```
   your driver (usb_driver)
        │ usb_submit_urb()
   USB core (drivers/usb/core/)
        │ hcd->driver->urb_enqueue()
   HCD driver: xhci-hcd / ehci-hcd / ohci-hcd / dwc3 / dwc2 / musb
        │
   hardware: schedules transactions on the wire
```

`struct usb_hcd` is the interface. The core handles descriptors, enumeration, driver binding,
and URB bookkeeping; the HCD handles actual scheduling and DMA. A driver never knows which
controller it is on.

Generations, and what changed:

| HCD | USB | Notable |
|---|---|---|
| **OHCI/UHCI** | 1.1 | 12 Mb/s; UHCI puts more work in software |
| **EHCI** | 2.0 | 480 Mb/s; needs companion controllers for low/full speed |
| **xHCI** | 3.x | 5–20 Gb/s; **one controller for all speeds**; ring-based, per-endpoint rings, MSI-X — architecturally much closer to NVMe than to EHCI |

xHCI's design is worth noting: it replaced EHCI's frame-list scheduling with **per-endpoint
transfer rings in host memory plus doorbell registers** — exactly the pattern from Ch. 35
(descriptor ring + doorbell). USB 3's host side became a normal modern DMA device, even
though the *bus* is still host-scheduled.

### T.7 USB gadget: the same machine, on the other end of the cable

A Linux SoC with a USB **device controller (UDC)** can *be* a USB device: a mass-storage
disk, a serial port, an Ethernet adapter, a MIDI device, or several at once. This is how
Android's ADB, Raspberry Pi Zero's "USB gadget" mode, and BMC virtual media work.

```
   Function drivers      f_mass_storage, f_ecm, f_acm, f_hid, f_fs (FunctionFS), f_uvc
        │
   Composite framework   builds the descriptor tree from the chosen functions
        │
   UDC driver            dwc3, dwc2, musb, cdns3, ...
        │
   hardware
```

Configured from userspace via **configfs** (Ch. 30 T.5) — a perfect fit, because the *set* of
functions is a userspace policy decision:

```bash
sudo modprobe libcomposite
cd /sys/kernel/config/usb_gadget
sudo mkdir g1 && cd g1
echo 0x1d6b | sudo tee idVendor                 # Linux Foundation
echo 0x0104 | sudo tee idProduct                # Multifunction Composite Gadget
sudo mkdir -p strings/0x409
echo "0123456789"   | sudo tee strings/0x409/serialnumber
echo "My Company"   | sudo tee strings/0x409/manufacturer
echo "My Gadget"    | sudo tee strings/0x409/product

sudo mkdir -p configs/c.1/strings/0x409
echo "Config 1" | sudo tee configs/c.1/strings/0x409/configuration
echo 250        | sudo tee configs/c.1/MaxPower

sudo mkdir -p functions/acm.usb0                # a serial port
sudo mkdir -p functions/ecm.usb0                # an ethernet adapter
sudo ln -s functions/acm.usb0 configs/c.1/
sudo ln -s functions/ecm.usb0 configs/c.1/

ls /sys/class/udc                               # the available device controllers
echo "$(ls /sys/class/udc | head -1)" | sudo tee UDC    # ★ bind = go live
```

**`FunctionFS` (`f_fs`)** is the escape hatch: it lets a *userspace* program implement an
arbitrary USB function by writing descriptors and reading/writing endpoint files. ADB uses
it. It is the USB equivalent of FUSE.

---

## 1. Internals

### 1.1 Source map

```
drivers/usb/core/            ★★ the core
   hub.c                     ★ enumeration, port handling (T.5)
   message.c                 synchronous helpers (usb_control_msg, usb_bulk_msg)
   urb.c                     ★ submit/kill/anchor (T.4)
   driver.c                  ★ matching and probe (T.2)
   config.c                  descriptor parsing
   devio.c                   usbfs — /dev/bus/usb (libusb's kernel side)
drivers/usb/host/            xhci-*.c ★, ehci-*.c, ohci-*.c
drivers/usb/dwc3/, dwc2/, musb/   dual-role SoC controllers
drivers/usb/gadget/          ★ composite.c, configfs.c, function/f_*.c, udc/
drivers/usb/storage/, drivers/usb/serial/, drivers/hid/usbhid/, drivers/net/usb/
drivers/usb/typec/           USB-C, PD, alternate modes
drivers/usb/misc/usb-ljca.c, usb-skeleton.c  ★ start here
include/linux/usb.h          ★★ read the whole header
include/uapi/linux/usb/      the wire-format structures
Documentation/driver-api/usb/  ★★ usb.rst, URB.rst, anchors.rst, gadget.rst, error-codes.rst
Documentation/usb/           ★ gadget_configfs.rst, functionfs.rst, usbmon.rst
```

### 1.2 `usb_driver` and matching

```c
static const struct usb_device_id my_ids[] = {
	{ USB_DEVICE(0x1234, 0x5678) },
	{ USB_DEVICE_AND_INTERFACE_INFO(0x1234, 0x5678, 0xff, 0x01, 0x01) },
	{ USB_INTERFACE_INFO(USB_CLASS_HID, 0, 0) },        /* ★ any HID device */
	{ USB_DEVICE_INTERFACE_CLASS(0x1234, 0x5678, 0xff) },
	{ }
};
MODULE_DEVICE_TABLE(usb, my_ids);

static struct usb_driver my_driver = {
	.name        = "mydrv",
	.id_table    = my_ids,
	.probe       = my_probe,
	.disconnect  = my_disconnect,
	.suspend     = my_suspend,
	.resume      = my_resume,
	.pre_reset   = my_pre_reset,      /* ★ before a device reset */
	.post_reset  = my_post_reset,
	.supports_autosuspend = 1,
};
module_usb_driver(my_driver);
```

Matching by **class/subclass/protocol** rather than VID/PID is what makes USB work without
vendor drivers: `usb-storage`, `usbhid`, `cdc_acm`, `cdc_ether`, `uvcvideo`, and
`snd-usb-audio` bind to *any* device implementing the standard class. That standardization —
"implement this class and every OS supports you" — is a large part of why USB won.

---

## 2. Practice

### Lab 39.1 — Read every descriptor on your machine

```bash
lsusb -t                                   # topology + bound drivers
for d in /sys/bus/usb/devices/[0-9]*-[0-9]*; do
  [ -f $d/idVendor ] || continue
  printf "%-10s %s:%s  %s  %s  %sspeed  %smA\n" "$(basename $d)" \
    "$(cat $d/idVendor)" "$(cat $d/idProduct)" \
    "$(cat $d/manufacturer 2>/dev/null)" "$(cat $d/product 2>/dev/null)" \
    "$(cat $d/speed)" "$(cat $d/bMaxPower 2>/dev/null | tr -d 'mA')"
done

# The interface layer (T.2a)
for i in /sys/bus/usb/devices/*:*; do
  printf "%-14s class=%s sub=%s proto=%s eps=%s driver=%s\n" "$(basename $i)" \
    "$(cat $i/bInterfaceClass)" "$(cat $i/bInterfaceSubClass)" \
    "$(cat $i/bInterfaceProtocol)" "$(cat $i/bNumEndpoints)" \
    "$(basename $(readlink $i/driver 2>/dev/null) 2>/dev/null)"
done

# Raw descriptor bytes — parse one by hand
sudo hexdump -C /sys/bus/usb/devices/1-1/descriptors | head -20
lsusb -v -d $(lsusb | head -1 | awk '{print $6}') 2>/dev/null | head -60
```
**Deliverable:** for one multi-interface device (a headset, a webcam, a wireless dongle),
draw the full T.2 hierarchy and name the driver bound to each interface.

### Lab 39.2 — Capture USB traffic with `usbmon`

This is the `tcpdump` of USB and it is indispensable.

```bash
sudo modprobe usbmon
ls /sys/kernel/debug/usb/usbmon/

# Text capture on bus 1
sudo cat /sys/kernel/debug/usb/usbmon/1u | head -40

# Wireshark can decode it properly:
sudo setfacl -m u:$USER:r /dev/usbmon1 2>/dev/null
wireshark -i usbmon1 &

# Or tcpdump-style:
sudo tcpdump -i usbmon1 -w /tmp/usb.pcap
```
Capture the **enumeration** of a device you plug in, and identify every step of T.5 in the
trace: the GET_DESCRIPTOR(8), the SET_ADDRESS, the full descriptor fetch, SET_CONFIGURATION.

```
# usbmon text format:
# URB-ID  timestamp  S/C  type:dir:bus:dev:ep  status  length  data
ffff888...  3423.123456 S Ci:1:005:0 s 80 06 0100 0000 0012 18 <
ffff888...  3423.123789 C Ci:1:005:0 0 18 = 12010002 00000040 6d04 2bc5 ...
#                       ^ C=Complete  i=in  control  status=0  18 bytes
```

### Lab 39.3 — A complete USB driver

Based on `drivers/usb/usb-skeleton.c`, but modernized and annotated.

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * usbdemo — bulk in/out with a char device front end.
 *
 * Lifetime (Ch. 25 §3):
 *   usb_interface holds our priv via usb_set_intfdata(); we take a kref.
 *   An open fd also holds a kref — so priv survives disconnect (Ch. 28 T.3).
 *   disconnect(): mark dead, kill anchored URBs, wake waiters, drop the driver's ref.
 */
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

#include <linux/kref.h>
#include <linux/module.h>
#include <linux/mutex.h>
#include <linux/slab.h>
#include <linux/uaccess.h>
#include <linux/usb.h>

#define WRITES_IN_FLIGHT 8
#define BULK_TIMEOUT_MS  5000

struct usbdemo {
	struct usb_device    *udev;
	struct usb_interface *intf;
	struct kref           kref;
	struct mutex          io_mutex;

	__u8                  bulk_in_ep, bulk_out_ep;
	size_t                bulk_in_size;
	unsigned char        *bulk_in_buf;
	struct urb           *bulk_in_urb;

	struct usb_anchor     submitted;        /* ★ T.4: all in-flight URBs */
	struct semaphore      limit_sem;        /* bound the write queue depth */
	wait_queue_head_t     bulk_in_wait;

	bool                  ongoing_read;
	size_t                bulk_in_filled, bulk_in_copied;
	int                   errors;
	spinlock_t            err_lock;
	bool                  disconnected;
};

static struct usb_driver usbdemo_driver;

static void usbdemo_delete(struct kref *kref)
{
	struct usbdemo *d = container_of(kref, struct usbdemo, kref);

	usb_free_urb(d->bulk_in_urb);
	usb_put_intf(d->intf);
	usb_put_dev(d->udev);
	kfree(d->bulk_in_buf);
	kfree(d);
}

/* ---- file operations ---- */

static int usbdemo_open(struct inode *inode, struct file *file)
{
	struct usb_interface *intf;
	struct usbdemo *d;
	int subminor = iminor(inode), ret;

	intf = usb_find_interface(&usbdemo_driver, subminor);
	if (!intf)
		return -ENODEV;
	d = usb_get_intfdata(intf);
	if (!d)
		return -ENODEV;

	ret = usb_autopm_get_interface(intf);   /* ★ runtime PM: wake the device */
	if (ret)
		return ret;

	kref_get(&d->kref);                     /* ★ the fd holds a reference */
	file->private_data = d;
	return 0;
}

static int usbdemo_release(struct inode *inode, struct file *file)
{
	struct usbdemo *d = file->private_data;

	mutex_lock(&d->io_mutex);
	if (d->intf)
		usb_autopm_put_interface(d->intf);
	mutex_unlock(&d->io_mutex);

	kref_put(&d->kref, usbdemo_delete);
	return 0;
}

static void usbdemo_read_complete(struct urb *urb)
{
	struct usbdemo *d = urb->context;
	unsigned long flags;

	spin_lock_irqsave(&d->err_lock, flags);
	if (urb->status) {
		if (!(urb->status == -ENOENT || urb->status == -ECONNRESET ||
		      urb->status == -ESHUTDOWN))           /* ★ T.4 rule 2: teardown */
			dev_err_ratelimited(&d->intf->dev, "read status %d\n", urb->status);
		d->errors = urb->status;
	} else {
		d->bulk_in_filled = urb->actual_length;     /* ★ T.4 rule 3: short is OK */
	}
	d->ongoing_read = false;
	spin_unlock_irqrestore(&d->err_lock, flags);

	wake_up_interruptible(&d->bulk_in_wait);        /* ★ Ch. 25 P3 */
}

static int usbdemo_do_read_io(struct usbdemo *d, size_t count)
{
	int ret;

	usb_fill_bulk_urb(d->bulk_in_urb, d->udev,
			  usb_rcvbulkpipe(d->udev, d->bulk_in_ep),
			  d->bulk_in_buf, min(d->bulk_in_size, count),
			  usbdemo_read_complete, d);

	spin_lock_irq(&d->err_lock);
	d->ongoing_read = true;
	spin_unlock_irq(&d->err_lock);

	d->bulk_in_filled = 0;
	d->bulk_in_copied = 0;

	usb_anchor_urb(d->bulk_in_urb, &d->submitted);   /* ★ T.4 anchors */
	ret = usb_submit_urb(d->bulk_in_urb, GFP_KERNEL);
	if (ret) {
		dev_err(&d->intf->dev, "submit read failed: %d\n", ret);
		usb_unanchor_urb(d->bulk_in_urb);
		spin_lock_irq(&d->err_lock);
		d->ongoing_read = false;
		spin_unlock_irq(&d->err_lock);
	}
	return ret;
}

static ssize_t usbdemo_read(struct file *file, char __user *buf,
			    size_t count, loff_t *ppos)
{
	struct usbdemo *d = file->private_data;
	size_t available;
	int ret;

	if (!count)
		return 0;

	ret = mutex_lock_interruptible(&d->io_mutex);
	if (ret)
		return ret;
	if (d->disconnected) {                            /* ★ Ch. 29 T.7 */
		ret = -ENODEV;
		goto out;
	}

retry:
	spin_lock_irq(&d->err_lock);
	if (d->ongoing_read) {
		spin_unlock_irq(&d->err_lock);
		if (file->f_flags & O_NONBLOCK) {
			ret = -EAGAIN;
			goto out;
		}
		ret = wait_event_interruptible(d->bulk_in_wait,
					       !d->ongoing_read || d->disconnected);
		if (ret < 0)
			goto out;
		if (d->disconnected) {
			ret = -ENODEV;
			goto out;
		}
		spin_lock_irq(&d->err_lock);
	}
	if (d->errors) {
		ret = (d->errors == -EPIPE) ? -EPIPE : -EIO;
		d->errors = 0;
		spin_unlock_irq(&d->err_lock);
		goto out;
	}
	spin_unlock_irq(&d->err_lock);

	available = d->bulk_in_filled - d->bulk_in_copied;
	if (!available) {
		ret = usbdemo_do_read_io(d, count);
		if (ret < 0)
			goto out;
		goto retry;
	}

	count = min(count, available);
	if (copy_to_user(buf, d->bulk_in_buf + d->bulk_in_copied, count)) {
		ret = -EFAULT;
		goto out;
	}
	d->bulk_in_copied += count;
	ret = count;
out:
	mutex_unlock(&d->io_mutex);
	return ret;
}

static void usbdemo_write_complete(struct urb *urb)
{
	struct usbdemo *d = urb->context;
	unsigned long flags;

	if (urb->status &&
	    !(urb->status == -ENOENT || urb->status == -ECONNRESET ||
	      urb->status == -ESHUTDOWN)) {
		spin_lock_irqsave(&d->err_lock, flags);
		d->errors = urb->status;
		spin_unlock_irqrestore(&d->err_lock, flags);
	}
	/* ★ the buffer was allocated with usb_alloc_coherent — free it the same way */
	usb_free_coherent(urb->dev, urb->transfer_buffer_length,
			  urb->transfer_buffer, urb->transfer_dma);
	up(&d->limit_sem);
}

static ssize_t usbdemo_write(struct file *file, const char __user *ubuf,
			     size_t count, loff_t *ppos)
{
	struct usbdemo *d = file->private_data;
	struct urb *urb = NULL;
	char *buf = NULL;
	int ret;

	if (!count)
		return 0;

	if (file->f_flags & O_NONBLOCK) {
		if (down_trylock(&d->limit_sem))
			return -EAGAIN;
	} else {
		if (down_interruptible(&d->limit_sem))
			return -ERESTARTSYS;
	}

	urb = usb_alloc_urb(0, GFP_KERNEL);
	if (!urb) { ret = -ENOMEM; goto err_sem; }

	/* ★ usb_alloc_coherent gives DMA-able memory (Ch. 35) */
	buf = usb_alloc_coherent(d->udev, count, GFP_KERNEL, &urb->transfer_dma);
	if (!buf) { ret = -ENOMEM; goto err_urb; }

	if (copy_from_user(buf, ubuf, count)) { ret = -EFAULT; goto err_buf; }

	mutex_lock(&d->io_mutex);
	if (d->disconnected) { mutex_unlock(&d->io_mutex); ret = -ENODEV; goto err_buf; }

	usb_fill_bulk_urb(urb, d->udev, usb_sndbulkpipe(d->udev, d->bulk_out_ep),
			  buf, count, usbdemo_write_complete, d);
	urb->transfer_flags |= URB_NO_TRANSFER_DMA_MAP;
	usb_anchor_urb(urb, &d->submitted);

	ret = usb_submit_urb(urb, GFP_KERNEL);
	mutex_unlock(&d->io_mutex);
	if (ret) {
		dev_err(&d->intf->dev, "submit write failed: %d\n", ret);
		usb_unanchor_urb(urb);
		goto err_buf;
	}

	usb_free_urb(urb);          /* the URB core holds its own reference now */
	return count;

err_buf:
	usb_free_coherent(d->udev, count, buf, urb->transfer_dma);
err_urb:
	usb_free_urb(urb);
err_sem:
	up(&d->limit_sem);
	return ret;
}

static const struct file_operations usbdemo_fops = {
	.owner   = THIS_MODULE,
	.open    = usbdemo_open,
	.release = usbdemo_release,
	.read    = usbdemo_read,
	.write   = usbdemo_write,
	.llseek  = noop_llseek,
};

static struct usb_class_driver usbdemo_class = {
	.name       = "usbdemo%d",
	.fops       = &usbdemo_fops,
	.minor_base = 192,
};

/* ---- probe / disconnect ---- */

static int usbdemo_probe(struct usb_interface *intf, const struct usb_device_id *id)
{
	struct usbdemo *d;
	struct usb_endpoint_descriptor *bulk_in, *bulk_out;
	int ret;

	d = kzalloc(sizeof(*d), GFP_KERNEL);     /* ★ NOT devm_ (Ch. 28 T.3) */
	if (!d)
		return -ENOMEM;

	kref_init(&d->kref);
	mutex_init(&d->io_mutex);
	spin_lock_init(&d->err_lock);
	init_usb_anchor(&d->submitted);
	init_waitqueue_head(&d->bulk_in_wait);
	sema_init(&d->limit_sem, WRITES_IN_FLIGHT);

	d->udev = usb_get_dev(interface_to_usbdev(intf));
	d->intf = usb_get_intf(intf);

	/* ★ the modern helper: find the endpoints we need, or fail */
	ret = usb_find_common_endpoints(intf->cur_altsetting,
					&bulk_in, &bulk_out, NULL, NULL);
	if (ret) {
		dev_err(&intf->dev, "no bulk-in/bulk-out endpoints\n");
		goto err;
	}
	d->bulk_in_size = usb_endpoint_maxp(bulk_in);
	d->bulk_in_ep   = bulk_in->bEndpointAddress;
	d->bulk_out_ep  = bulk_out->bEndpointAddress;

	d->bulk_in_buf = kmalloc(d->bulk_in_size, GFP_KERNEL);
	d->bulk_in_urb = usb_alloc_urb(0, GFP_KERNEL);
	if (!d->bulk_in_buf || !d->bulk_in_urb) { ret = -ENOMEM; goto err; }

	usb_set_intfdata(intf, d);

	ret = usb_register_dev(intf, &usbdemo_class);
	if (ret) {
		dev_err(&intf->dev, "cannot get a minor\n");
		usb_set_intfdata(intf, NULL);
		goto err;
	}

	dev_info(&intf->dev, "attached to /dev/usbdemo%d (in ep 0x%02x, out ep 0x%02x)\n",
		 intf->minor - usbdemo_class.minor_base, d->bulk_in_ep, d->bulk_out_ep);
	return 0;

err:
	kref_put(&d->kref, usbdemo_delete);
	return ret;
}

static void usbdemo_disconnect(struct usb_interface *intf)
{
	struct usbdemo *d = usb_get_intfdata(intf);

	usb_deregister_dev(intf, &usbdemo_class);   /* ★ STOP: no new opens */

	mutex_lock(&d->io_mutex);
	d->disconnected = true;
	mutex_unlock(&d->io_mutex);

	usb_kill_anchored_urbs(&d->submitted);      /* ★ DRAIN: kill + wait (T.4 rule 2) */
	wake_up_interruptible(&d->bulk_in_wait);    /* kick blocked readers */

	kref_put(&d->kref, usbdemo_delete);         /* ★ FREE: drop the driver's ref;
						     *   open fds keep it alive */
}

static int usbdemo_suspend(struct usb_interface *intf, pm_message_t msg)
{
	struct usbdemo *d = usb_get_intfdata(intf);

	if (d)
		usb_kill_anchored_urbs(&d->submitted);
	return 0;
}
static int usbdemo_resume(struct usb_interface *intf) { return 0; }

static const struct usb_device_id usbdemo_ids[] = {
	{ USB_DEVICE(0x1234, 0x5678) },
	{ }
};
MODULE_DEVICE_TABLE(usb, usbdemo_ids);

static struct usb_driver usbdemo_driver = {
	.name       = "usbdemo",
	.id_table   = usbdemo_ids,
	.probe      = usbdemo_probe,
	.disconnect = usbdemo_disconnect,
	.suspend    = usbdemo_suspend,
	.resume     = usbdemo_resume,
	.supports_autosuspend = 1,
};
module_usb_driver(usbdemo_driver);
MODULE_LICENSE("GPL");
```

```bash
# Test against a QEMU-emulated device, or against a real gadget (Lab 39.5):
qemu-system-x86_64 -M q35 -device qemu-xhci -device usb-storage,drive=d0 ...
lsusb; dmesg | tail
```

### Lab 39.4 — Drive a device from userspace with libusb

```c
/* libusbdemo.c — gcc -O2 -o libusbdemo libusbdemo.c $(pkg-config --cflags --libs libusb-1.0) */
#include <libusb-1.0/libusb.h>
#include <stdio.h>

int main(void)
{
	libusb_device **list;
	libusb_context *ctx = NULL;
	ssize_t n, i;

	libusb_init(&ctx);
	n = libusb_get_device_list(ctx, &list);

	for (i = 0; i < n; i++) {
		struct libusb_device_descriptor dd;
		struct libusb_config_descriptor *cfg;
		int j, k;

		libusb_get_device_descriptor(list[i], &dd);
		printf("%04x:%04x  class=%02x  bus=%d addr=%d\n",
		       dd.idVendor, dd.idProduct, dd.bDeviceClass,
		       libusb_get_bus_number(list[i]), libusb_get_device_address(list[i]));

		if (libusb_get_active_config_descriptor(list[i], &cfg))
			continue;
		for (j = 0; j < cfg->bNumInterfaces; j++) {
			const struct libusb_interface_descriptor *id =
				&cfg->interface[j].altsetting[0];

			printf("   if%d class=%02x/%02x/%02x  %d endpoints\n", j,
			       id->bInterfaceClass, id->bInterfaceSubClass,
			       id->bInterfaceProtocol, id->bNumEndpoints);
			for (k = 0; k < id->bNumEndpoints; k++) {
				const struct libusb_endpoint_descriptor *ep = &id->endpoint[k];
				static const char *types[] = {"ctrl","isoc","bulk","intr"};

				printf("      ep 0x%02x %s %s maxp=%d interval=%d\n",
				       ep->bEndpointAddress,
				       (ep->bEndpointAddress & 0x80) ? "IN " : "OUT",
				       types[ep->bmAttributes & 3],
				       ep->wMaxPacketSize, ep->bInterval);
			}
		}
		libusb_free_config_descriptor(cfg);
	}
	libusb_free_device_list(list, 1);
	libusb_exit(ctx);
	return 0;
}
```
```bash
sudo apt install -y libusb-1.0-0-dev
gcc -O2 -o /tmp/lu /tmp/libusbdemo.c $(pkg-config --cflags --libs libusb-1.0) && /tmp/lu
```
**Then detach the kernel driver and take over a device**
(`libusb_detach_kernel_driver()`), do a control transfer, and reattach. This is exactly what
`usbfs` (`drivers/usb/core/devio.c`) exposes — read it to see the kernel side.

### Lab 39.5 — Turn a machine into a USB device (T.7)

On a Raspberry Pi Zero/4, a BeagleBone, or any board with a UDC — or in QEMU with `dummy_hcd`:

```bash
# QEMU/VM option: a virtual UDC + HCD pair, so one machine is both ends
sudo modprobe dummy_hcd
ls /sys/class/udc/               # dummy_udc.0

sudo modprobe libcomposite
cd /sys/kernel/config/usb_gadget
sudo mkdir -p g1 && cd g1
echo 0x1d6b | sudo tee idVendor
echo 0x0104 | sudo tee idProduct
echo 0x0200 | sudo tee bcdUSB
sudo mkdir -p strings/0x409
echo "1234567890"  | sudo tee strings/0x409/serialnumber
echo "LabCorp"     | sudo tee strings/0x409/manufacturer
echo "Lab Gadget"  | sudo tee strings/0x409/product

sudo mkdir -p configs/c.1/strings/0x409
echo "CDC ACM + ECM" | sudo tee configs/c.1/strings/0x409/configuration
echo 250 | sudo tee configs/c.1/MaxPower

sudo mkdir -p functions/acm.usb0
sudo mkdir -p functions/mass_storage.usb0
truncate -s 64M /tmp/disk.img && mkfs.vfat /tmp/disk.img
echo /tmp/disk.img | sudo tee functions/mass_storage.usb0/lun.0/file

sudo ln -s functions/acm.usb0 configs/c.1/
sudo ln -s functions/mass_storage.usb0 configs/c.1/

echo dummy_udc.0 | sudo tee UDC          # ★ go live

# Now the SAME machine sees it as a host:
lsusb | grep 1d6b:0104
ls /dev/ttyACM* /dev/sd*
dmesg | tail -20

# Tear down (order matters):
echo "" | sudo tee UDC
sudo rm configs/c.1/acm.usb0 configs/c.1/mass_storage.usb0
sudo rmdir functions/* configs/c.1/strings/0x409 configs/c.1 strings/0x409
cd .. && sudo rmdir g1
```
**This is the single best USB lab available** — you built a device, and a host enumerated it,
on one machine, with no hardware.

### Lab 39.6 — Bandwidth, reservation, and isochronous starvation (T.3)

```bash
# What is reserved on each bus?
cat /sys/kernel/debug/usb/devices | grep -E '^T:|^B:'
#  B:  Alloc=...  Int=...  Iso=...   ← reserved bandwidth

# Plug in a webcam and start streaming:
v4l2-ctl --list-devices
ffmpeg -f v4l2 -i /dev/video0 -t 10 -f null - 2>&1 | tail -5
# ...while a USB audio device is also active. Watch for:
dmesg | grep -iE 'not enough bandwidth|no bandwidth|cannot submit urb'

# Alternate settings (the reservation mechanism):
cat /sys/bus/usb/devices/*/bAlternateSetting 2>/dev/null
lsusb -v -d <webcam> | grep -A6 'bAlternateSetting'

# xHCI's view:
sudo cat /sys/kernel/debug/usb/xhci/*/command-ring 2>/dev/null | head
sudo ls /sys/kernel/debug/usb/xhci/
```

### Lab 39.7 — Surprise disconnect while I/O is in flight

```bash
# Start a long transfer, then yank the device
dd if=/dev/sdX of=/dev/null bs=1M &
sleep 2
# physically unplug, or:
echo 1 | sudo tee /sys/bus/usb/devices/1-1/remove
dmesg | tail -30

# Under KASAN, this is where URB lifetime bugs surface:
./scripts/config -e KASAN
# loop: plug/unplug (or bind/unbind) while transferring
for i in $(seq 1 100); do
  echo 1-1 | sudo tee /sys/bus/usb/drivers/usb-storage/unbind 2>/dev/null
  echo 1-1 | sudo tee /sys/bus/usb/drivers/usb-storage/bind 2>/dev/null
done
dmesg | grep -i kasan
```
Then audit your Lab 39.3 driver: is every URB anchored? Does `disconnect()` kill them all?
Does every blocked waiter check `disconnected`?

---

## 3. Mastery drills

1. **Read `drivers/usb/usb-skeleton.c`** completely — it is the canonical annotated example.
   Then read `Documentation/driver-api/usb/URB.rst` and `anchors.rst`.

2. **Host-scheduled implications.** Write 400 words explaining what changes about driver
   design because a USB device cannot initiate a transfer. Compare a USB NIC's RX path with a
   PCIe NIC's.

3. **Parse a descriptor blob by hand.** Take `hexdump /sys/bus/usb/devices/1-1/descriptors`
   and decode the whole configuration tree manually using the USB spec's descriptor
   definitions. Verify against `lsusb -v`.

4. **The four transfer types.** For each of: a gaming mouse (1000 Hz), a USB 3 SSD, a DAC
   playing 96 kHz audio, a firmware update — choose the transfer type and justify from T.3's
   guarantees. What happens if you choose wrong?

5. **URB error codes.** Read `Documentation/driver-api/usb/error-codes.rst`. For each of
   `-EPIPE`, `-EPROTO`, `-EOVERFLOW`, `-ETIMEDOUT`, `-ENOENT`, `-ECONNRESET`, `-ESHUTDOWN`,
   `-ENODEV`: what caused it and what should the driver do?

6. **`kill` vs `unlink` vs `poison`.** Read `drivers/usb/core/urb.c`. Explain the exact
   guarantee each provides and construct a scenario where using `unlink` in `disconnect()`
   causes a use-after-free.

7. **Enumeration trace.** Capture a full enumeration with `usbmon` in Wireshark. Annotate
   every control transfer against T.5's numbered steps. How long did enumeration take?

8. **xHCI architecture.** Read the xHCI specification's chapter on transfer rings, then
   `drivers/usb/host/xhci-ring.c`. Compare the ring + doorbell model with Ch. 35's DMA ring.
   What is a TRB? How does the driver know a transfer completed?

9. **Gadget composite.** Read `drivers/usb/gadget/composite.c`. Explain how the descriptor
   tree is assembled from independent function drivers, and how `f_fs` lets userspace
   implement a function.

10. **Class drivers.** Explain why `usb_find_interface`-style class matching (HID, MSC, CDC,
    UVC, UAC) made USB successful, and contrast with PCI's VID/PID-dominated matching. What
    is the equivalent in PCI? (Class codes — but far less used. Why?)

11. **Design question.** You must write a driver for a device with: a bulk-in/bulk-out pair
    for data, an interrupt-in endpoint for status, and an isochronous-in for audio, plus a
    firmware-upload mode that changes the VID/PID after loading. Design the driver structure:
    how many `usb_driver`s, how many interfaces, URB management, teardown, and how you handle
    the re-enumeration after firmware load.

---

## 4. Further reading

**Specifications:**
- **USB 2.0 Specification** — Ch. 4 (architecture), 5 (data flow model), 9 (device
  framework: descriptors, requests, states) ★★. Chapter 9 is the one you will reread.
- USB 3.2 / USB4 specifications; xHCI specification (Intel)
- USB Device Class specifications (HID, MSC, CDC, UVC, UAC) — usb.org
- USB Power Delivery specification (for `typec`/`tcpm`)

**Kernel documentation:**
- `Documentation/driver-api/usb/usb.rst` ★★
- `Documentation/driver-api/usb/URB.rst` ★★, `anchors.rst` ★, `error-codes.rst` ★
- `Documentation/driver-api/usb/writing_usb_driver.rst`
- `Documentation/driver-api/usb/gadget.rst`, `Documentation/usb/gadget_configfs.rst` ★
- `Documentation/usb/usbmon.rst` ★, `functionfs.rst`, `raw-gadget.rst`
- `Documentation/driver-api/usb/dma.rst` — `usb_alloc_coherent` and URB DMA flags

**Source:**
- `drivers/usb/usb-skeleton.c` ★★ — read it line by line
- `drivers/usb/core/hub.c` (enumeration), `urb.c`, `message.c`, `driver.c`
- `drivers/usb/host/xhci*.c` — modern HCD design
- `drivers/usb/gadget/` — `composite.c`, `function/f_fs.c`
- `drivers/hid/usbhid/hid-core.c`, `drivers/net/usb/cdc_ether.c` — good class drivers

**Tools:**
- `lsusb`, `usbmon` + Wireshark ★, `libusb`, `usbip` (USB over IP!), `usbguard`
- `raw-gadget` — implement a USB device entirely from userspace; used for fuzzing
- `syzkaller`'s USB fuzzing (`vusb`) — it found hundreds of bugs; read how it works

**LWN:**
- "The USB gadget subsystem" / "USB gadgets and configfs"
- "Fuzzing the USB stack" (Andrey Konovalov) ★ — how raw-gadget + syzkaller found ~300 bugs
- "USB Type-C and Power Delivery in Linux"
- "usbip: USB over the network"

→ Next: [40-i2c.md](40-i2c.md)

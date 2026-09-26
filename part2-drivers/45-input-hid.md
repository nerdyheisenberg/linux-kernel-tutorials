# Chapter 45 — The input subsystem: evdev, HID, and touchscreens

> **Goal:** Understand why human input needed its own subsystem rather than being a hundred character devices, why the event protocol is a *stream of typed deltas terminated by a synchronisation barrier* rather than a stream of states, how HID turned every input device into a self-describing one and what that cost, how multitouch was retrofitted onto a protocol designed for a mouse, and why `evdev` is one of the few kernel ABIs that successfully survived three decades of hardware change. By the end you can write an input driver, a HID driver, read raw event streams, decode a report descriptor by hand, and debug the "why does my touchscreen report coordinates from the wrong corner" class of problem.

---

## Theory & First Principles

### T.0 — Start here: why `A` is not in the keyboard driver

Press `A`. What does the kernel send to userspace?

```bash
sudo evtest /dev/input/event3     # press one key
# Event: type 4 (EV_MSC), code 4 (MSC_SCAN), value 1e
# Event: type 1 (EV_KEY), code 30 (KEY_A), value 1     <- PRESSED
# Event: type 0 (EV_SYN), code 0 (SYN_REPORT), value 0 <- end of this "frame"
# Event: type 1 (EV_KEY), code 30 (KEY_A), value 0     <- RELEASED
```

Not `'A'`. Not `'a'`. **`KEY_A`, a number, plus press/release.** The driver has no idea
whether you meant a lowercase `a`, an uppercase `A`, a Cyrillic ф, or "move left" in a game.
It reports **which physical key changed state**, and nothing more.

**That is the entire design, and it is a policy/mechanism separation** (Ch. 00 §T.1):

```
  hardware scancode  ->  keymap  ->  KEY_A  ->  X11/Wayland keymap  ->  'A'
     (0x1e, USB HID)     kernel     kernel        USERSPACE            app
                                       ↑
                     the kernel STOPS HERE. Deliberately.
```

The kernel cannot know the layout (Dvorak? AZERTY?), the modifier state per-window, the input
method (how would it type Japanese?), or the application's intent. **All of that is policy,
so all of it is userspace.** The kernel's job is to normalize *mechanism*: many different
devices, one meaning.

**And "many devices, one meaning" is the load-bearing part.** A USB keyboard, a PS/2
keyboard, a Bluetooth keyboard, an on-screen keyboard, and a barcode scanner all produce
`KEY_A`. An application reads one interface and works with all of them — the narrow waist
(Ch. 00 §T.3b) applied to human input.

**Three details that are not obvious and that matter:**

**1. `EV_SYN` is not padding.** A mouse moving diagonally reports `REL_X` and `REL_Y` as two
separate events — but they happened *simultaneously*. `SYN_REPORT` marks the frame boundary:
everything before it is one atomic observation. Without it, a diagonal movement would be
indistinguishable from two sequential movements, and multitouch would be impossible.

**2. State, not edges, for absolute devices.** A touchscreen reports `ABS_X`/`ABS_Y`
positions; a mouse reports `REL_X`/`REL_Y` deltas. The device's *type* determines which, and
getting it wrong produces a pointer that drifts or one that cannot be positioned.

**3. HID is a self-describing protocol, which is genuinely unusual.** A USB HID device ships
a **report descriptor** — a bytecode program describing its own report format:

```
 Usage Page (Generic Desktop), Usage (Mouse), Collection (Application)
   Usage (Pointer), Collection (Physical)
     Usage Page (Button), Usage Minimum (1), Usage Maximum (3)
     Report Count (3), Report Size (1), Input (Data, Variable, Absolute)
     ...
```

The kernel *parses* this and builds the input device from it. **That is why one `usbhid`
driver handles every mouse, keyboard, gamepad, tablet, and VR controller ever made** — the
device describes its own semantics, so no per-device code is needed. It is Ch. 37's
self-description taken one level further: not just "what am I" but "what does my data mean."

The cost is that report descriptors are frequently wrong, which is why `drivers/hid/` is full
of per-vendor quirk drivers fixing up broken descriptors — the same shape as ACPI's DMI
quirks (Ch. 33 §T.0).

```bash
ls /dev/input/                         # event*, mouse*, js*
cat /proc/bus/input/devices | head -30 # what each device CAN report (EV bitmaps)
sudo evtest                            # interactive
sudo usbhid-dump | head                # raw report descriptors
```

---

### T.1 The problem: many devices, one meaning

Before the input subsystem (Linux 2.4, Vojtech Pavlík), every input device was its own character device with its own protocol: `/dev/psaux` spoke PS/2 mouse packets, `/dev/js0` spoke joystick structs, the keyboard went straight into the tty layer, and USB HID devices emulated whichever of those they resembled. Userspace — X11, `gpm`, every game — carried a driver per protocol.

This is the same economic argument as Ch. 29 §T.1 ("everything is a file"), one level up. The observation that makes a subsystem possible is:

> **Almost all human input devices produce the same *kind* of information: "some named axis or button changed to some value at some time."** They differ in which axes and buttons exist, not in the shape of the information.

Once you accept that, the design follows:

1. **A single event format** carrying (type, code, value) plus a timestamp. Type says *what kind* of thing changed (key, relative axis, absolute axis, switch, LED, sound); code says *which* one (`KEY_A`, `REL_X`, `ABS_MT_POSITION_X`); value says *how much*.
2. **A single device node format** (`/dev/input/eventN`) so that userspace has one reader for everything.
3. **A capability advertisement** so userspace can ask "what does this device have?" before deciding how to treat it. This is the bit that makes one API work for a keyboard and a 10-finger touchscreen.
4. **A hardware-facing half** where drivers translate device-specific wire formats into events.

The subsystem is therefore an *adapter layer with a narrow waist*: N hardware protocols on top, 1 event protocol in the middle, M userspace consumers below. The narrow waist is exactly the IP-hourglass argument, and it is why the input ABI has outlived X11, three generations of touch hardware, and the entire concept of the PS/2 port.

### T.2 Why events are deltas terminated by a barrier

The event struct is:

```c
struct input_event {
	struct timeval time;	/* or __kernel_ulong_t pair on 64-bit ABI */
	__u16 type;
	__u16 code;
	__s32 value;
};
```

Sixteen (or 24) bytes. A device reporting "finger moved from (100,200) to (105,203) while button 1 is still down" emits:

```
EV_ABS  ABS_X  105
EV_ABS  ABS_Y  203
EV_SYN  SYN_REPORT 0
```

Note two decisions.

**(a) Only changes are reported.** The button state is not repeated. This is a *delta* protocol, not a *state* protocol. The alternative — send the full device state every time — was rejected because:

- Device state is unbounded in size (a 10-finger touchscreen with 5 axes per finger, plus 200 keys) while a change is almost always one or two values.
- Most changes are one axis. A state protocol would make the common case cost the worst case.
- A delta protocol composes: filters and virtual devices can pass through, drop, or inject individual events without understanding the whole device.

The cost of a delta protocol is that **the reader must be stateful**, and if the reader loses a single event its model diverges from reality forever. The subsystem addresses that with `SYN_DROPPED` (see §T.4) rather than by abandoning deltas.

**(b) `EV_SYN`/`SYN_REPORT` is a transaction barrier, not a "flush".** This is the most misunderstood part of the protocol and the most important.

A pointer moving diagonally produces a change in X and a change in Y that are *physically simultaneous*. If userspace acted on each event as it arrived, it would see the pointer move right, then down — a staircase — and would report two motion events where the hardware reported one. `SYN_REPORT` says:

> Everything since the last `SYN_REPORT` happened *at the same instant*. Apply it atomically.

So the protocol is not "stream of events" but "stream of **event packets**", where a packet is an atomic state transition. This matters enormously for multitouch, where a single packet describes the simultaneous positions of five fingers, and for gesture recognition, where interpreting X and Y separately is meaningless.

Formally: the event stream is a sequence of transactions, and `SYN_REPORT` is the commit. A driver that forgets `input_sync()` produces a device that appears to work (events arrive) but whose consumers see torn state — the input equivalent of a missing memory barrier, and with a similar debugging profile.

There are four sync codes and each is a distinct protocol-level statement:

| Code | Meaning |
|---|---|
| `SYN_REPORT` | end of packet; state is now consistent |
| `SYN_MT_REPORT` | end of one contact's data (protocol A only — see §T.6) |
| `SYN_DROPPED` | **the kernel discarded events for this client; your state is stale — resync** |
| `SYN_CONFIG` | device configuration changed (rare) |

### T.3 Types and codes: a shared namespace as the actual ABI

`include/uapi/linux/input-event-codes.h` is the real interface of this subsystem. It defines, for all time, that `KEY_A` is 30 and `BTN_LEFT` is 0x110 and `ABS_MT_POSITION_X` is 0x35. Two consequences:

- **The numbers can never change** (Ch. 24 §T.1: Hyrum's Law, permissiveness never tightens). New codes can be added in the gaps; existing ones are frozen.
- **The namespace is a taxonomy imposed on hardware.** A device's buttons must be mapped to the *semantic* codes, not to positional indices. A gaming mouse's fourth button is `BTN_SIDE`, not "button 4". Getting this mapping right is most of the design work in an input driver, and it is a judgement call, not a mechanical translation.

The major types:

| Type | Semantics | Examples |
|---|---|---|
| `EV_KEY` | binary, with 0/1/2 (release/press/**autorepeat**) | `KEY_*`, `BTN_*` |
| `EV_REL` | relative displacement, no absolute meaning | `REL_X`, `REL_WHEEL`, `REL_WHEEL_HI_RES` |
| `EV_ABS` | absolute position within an advertised range | `ABS_X`, `ABS_PRESSURE`, `ABS_MT_*` |
| `EV_SW` | stable binary state, not a keypress | `SW_LID`, `SW_TABLET_MODE`, `SW_HEADPHONE_INSERT` |
| `EV_MSC` | miscellaneous, notably `MSC_SCAN` (raw scancode) and `MSC_TIMESTAMP` | |
| `EV_LED`, `EV_SND`, `EV_FF` | **output** — userspace writes these to the device | caps lock LED, beeper, force feedback |
| `EV_SYN` | packet framing | |

Two distinctions worth internalising because drivers get them wrong:

- **`EV_REL` vs `EV_ABS` is not "mouse vs touchscreen", it is "does the value mean anything on its own?"** A mouse has no position; it has motion. A touchscreen has position. A trackpoint has motion. A graphics tablet has position *and* reports when the pen leaves proximity. Choosing `ABS` for a relative device forces userspace to invent a coordinate space that does not exist.
- **`EV_KEY` vs `EV_SW` is "momentary vs latching."** A lid switch is not a key that is held down for three hours. `EV_SW` state is queryable at any time via `EVIOCGSW` and is re-reported on open, because a client that starts with the lid already closed must learn that.

The fact that `EV_LED`/`EV_SND`/`EV_FF` flow *into* the device means an input device is bidirectional — a fact that surprises people and that shapes the driver API (`input_dev->event` is the write-side callback).

### T.4 Buffering, and why `SYN_DROPPED` exists

Each open file description on `/dev/input/eventN` gets its own **ring buffer** (`struct evdev_client`), default 64 events, grown for high-rate devices. Per-client buffering, not per-device, because clients read at different speeds and a slow client must not stall a fast one (the same argument as per-open state in Ch. 29 §T.3).

When a client's buffer overflows, the kernel cannot block the driver (which may be in an interrupt handler) and cannot silently drop (the client's delta-derived state would be permanently wrong). So it does the only correct third thing:

> Discard the buffer contents, then insert `EV_SYN/SYN_DROPPED`, then resume.

`SYN_DROPPED` is a protocol-level admission of failure that hands the client a defined recovery procedure:

1. Discard everything you have read up to and including the next `SYN_REPORT`.
2. Re-query the full device state with `EVIOCGKEY`, `EVIOCGABS`, `EVIOCGSW`, `EVIOCGLED`, and the MT slot state via `EVIOCGMTSLOTS`.
3. Resume normal processing.

This is the design pattern "when you cannot deliver, deliver the *fact* that you could not, plus a way to recover" — the same shape as `NLMSG_OVERRUN` in netlink and `PERF_RECORD_LOST` in perf. Almost no application implemented it correctly until libinput did; this is a good argument for §T.9's "do not talk to evdev directly."

Because the buffer is per-client and events are small, the subsystem also supports **event compaction is NOT done** — deliberately. Merging two `REL_X` events would be correct arithmetically but would destroy timing information that gesture recognisers depend on.

### T.5 The capability model: how one API describes every device

Before reading events, a client asks what the device *is*:

```c
ioctl(fd, EVIOCGBIT(0, EV_MAX), evbits);          /* which types exist */
ioctl(fd, EVIOCGBIT(EV_KEY, KEY_MAX), keybits);   /* which keys exist */
ioctl(fd, EVIOCGABS(ABS_X), &absinfo);            /* range of this axis */
ioctl(fd, EVIOCGNAME(len), name);
ioctl(fd, EVIOCGID, &id);                          /* bustype/vendor/product/version */
ioctl(fd, EVIOCGPROP(len), propbits);              /* INPUT_PROP_* */
```

Capabilities are **bitmaps**, one bit per code. This is why `input_set_capability()` / `set_bit(KEY_A, dev->keybit)` appear in every driver: the driver is populating the bitmaps that userspace will read back.

`struct input_absinfo` is where absolute axes become usable:

```c
struct input_absinfo {
	__s32 value;       /* current */
	__s32 minimum, maximum;
	__s32 fuzz;        /* noise threshold: ignore changes smaller than this */
	__s32 flat;        /* deadzone around centre (joysticks) */
	__s32 resolution;  /* units per mm (or per radian for rotational) */
};
```

`resolution` is the field that makes the difference between a touchscreen that works and one that does not, because it converts device units into **physical units**. Without it userspace cannot know whether a 200-unit movement is a twitch or a swipe, and gesture thresholds become device-specific magic numbers. Setting `resolution` correctly (via `input_abs_set_res()`) is a driver obligation that is frequently skipped.

`fuzz` implements hardware noise filtering *in the kernel*, so every client does not reimplement it. The input core drops changes smaller than `fuzz` — a rare case of the kernel being allowed to apply a policy, justified because the noise floor is a property of the hardware, which only the driver knows.

`INPUT_PROP_*` were added when the type/code taxonomy proved insufficient to distinguish device *classes* with identical capability sets:

| Property | Distinguishes |
|---|---|
| `INPUT_PROP_POINTER` | device needs an on-screen cursor (touchpad) |
| `INPUT_PROP_DIRECT` | device is on the screen (touchscreen) |
| `INPUT_PROP_BUTTONPAD` | the whole pad is the button (clickpad) |
| `INPUT_PROP_SEMI_MT` | reports a bounding box, not real per-finger positions |
| `INPUT_PROP_ACCELEROMETER` | `ABS_X/Y/Z` are acceleration, not position |

The existence of this list is a lesson in ABI evolution: when a taxonomy cannot express a needed distinction, adding an orthogonal property bitmap is cheaper and safer than redefining existing codes. Compare Ch. 24 §T.3's extensibility mechanisms.

### T.6 Multitouch: retrofitting N contacts onto a 1-pointer protocol

The original protocol had one `ABS_X`. Touchscreens report several simultaneous contacts, each with position, size, pressure, and identity. Two solutions were tried, and **both are still in the kernel**, which is itself the lesson.

**Protocol A (stateless, 2008).** Emit each contact's data followed by `SYN_MT_REPORT`, then `SYN_REPORT` at the end of the frame:

```
ABS_MT_POSITION_X 100 ; ABS_MT_POSITION_Y 200 ; SYN_MT_REPORT
ABS_MT_POSITION_X 500 ; ABS_MT_POSITION_Y 600 ; SYN_MT_REPORT
SYN_REPORT
```

Elegant and minimal, but it has a fatal flaw: **contacts have no identity across frames.** If two fingers are at (100,200) and (500,600) in one frame and at (300,400) and (300,410) in the next, did they converge or cross? Userspace has to solve an assignment problem with incomplete information, every frame, for every application. That is both expensive and *not uniquely solvable*.

**Protocol B (slotted, stateful, 2010).** Each contact occupies a **slot**, and slots persist across frames. `ABS_MT_SLOT` selects which slot subsequent `ABS_MT_*` events apply to; `ABS_MT_TRACKING_ID` gives the contact an identity (and `-1` means "this slot is now empty"):

```
ABS_MT_SLOT 0 ; ABS_MT_TRACKING_ID 45 ; ABS_MT_POSITION_X 100 ; ABS_MT_POSITION_Y 200
ABS_MT_SLOT 1 ; ABS_MT_TRACKING_ID 46 ; ABS_MT_POSITION_X 500 ; ABS_MT_POSITION_Y 600
SYN_REPORT
...
ABS_MT_SLOT 1 ; ABS_MT_TRACKING_ID -1
SYN_REPORT                                  /* finger 2 lifted */
```

Protocol B moves the tracking problem into the kernel (or the hardware, which usually already solved it) and makes the wire format a *delta on a persistent array* — consistent with §T.2's delta philosophy, now applied per-slot. It is strictly better, and it is what all new drivers use.

The kernel provides `input/input-mt.c` to make Protocol B easy:

```c
input_mt_init_slots(dev, max_contacts, INPUT_MT_DIRECT | INPUT_MT_DROP_UNUSED);
/* per frame: */
for each contact:
	input_mt_slot(dev, i);
	input_mt_report_slot_state(dev, MT_TOOL_FINGER, active);
	if (active) { input_report_abs(dev, ABS_MT_POSITION_X, x); ... }
input_mt_sync_frame(dev);	/* closes unused slots automatically */
input_sync(dev);
```

and — critically — `input_mt_report_pointer_emulation()`, which synthesises single-touch `ABS_X`/`ABS_Y`/`BTN_TOUCH` from the MT stream so that **legacy clients that predate multitouch still work**. That function is a compatibility shim in the purest sense: the new protocol is primary, the old one is derived, and no driver maintains two code paths. This is the right answer to the ABI-evolution problem (Ch. 24) and worth copying in your own designs.

Why does Protocol A still exist? Because drivers were written against it and the ABI rule says you do not break userspace. `input-mt.c` can even generate A from B for those clients. The whole episode is a case study: **a protocol that omits identity will have identity bolted on later at higher cost.**

### T.7 HID: self-description, and what it bought and cost

USB faced the same problem PCI solved with config space (Ch. 37 §T.1) and I²C failed to solve (Ch. 40 §T.3): how does the host know what this thing is? For input devices the USB-IF's answer was **HID — Human Interface Device**, and it is far more ambitious than a device-ID table:

> A HID device carries a **report descriptor**: a small bytecode program that describes the bit-level layout and semantic meaning of every field in the data packets it will send.

The descriptor is a sequence of items in a tag/type/size encoding, forming a stack machine with global state (usage page, logical/physical min-max, report size, report count) and local state (usages), committed by `Input`/`Output`/`Feature` main items. Decoding a mouse descriptor by hand (Lab 45.5) is a rite of passage.

The **usage tables** are the shared vocabulary — the HID equivalent of `input-event-codes.h`. Usage page 0x01 is Generic Desktop (X, Y, Wheel), 0x07 is Keyboard/Keypad, 0x09 is Button, 0x0D is Digitizers. A generic driver can therefore handle a device it has never seen: parse the descriptor, map usages to input codes, done. **`hid-generic` drives the majority of input hardware in the world with no device-specific code.** That is the payoff, and it is enormous.

The costs are equally real and worth being honest about:

| Cost | Detail |
|---|---|
| **Descriptors are frequently wrong** | Vendors ship descriptors that do not match the reports, use vendor-defined usage pages for standard functions, or declare 8 buttons and send 12. `drivers/hid/hid-*.c` is largely a museum of per-vendor fixups. |
| **Complexity** | The parser (`hid-core.c`) is ~2000 lines handling a Turing-incomplete but genuinely intricate language, and it parses **untrusted input from a device**, which is an attack surface (multiple CVEs). |
| **Expressiveness ≠ semantics** | The descriptor says a field is "Usage: X, 16 bits, logical 0..32767." It does not say whether that is a screen coordinate, a joystick axis, or a scroll amount. Convention fills the gap, and convention is not enforceable. |
| **Report descriptors are not versioned** | Hyrum's Law again: fix a descriptor bug in firmware and some OS breaks. |

The fixup mechanism deserves a note because it is a good pattern: `hid_driver->report_fixup()` lets a quirk driver **rewrite the descriptor bytes before parsing**. Rather than special-casing behaviour throughout the stack, you repair the device's self-description at the boundary and let the generic machinery proceed. That is the same move as `_DSD` in ACPI (Ch. 33 §T.4): normalise at the edge, keep the core generic.

HID is also **transport-independent** by design — `hid-core` sits above `usbhid`, `hid-i2c` (I²C-HID, Ch. 40), Bluetooth HIDP, and `hid-uclogic`-style vendor transports. One parser, many buses. And it is bidirectional (Output reports drive LEDs and force feedback; Feature reports are get/set configuration), which maps onto the input subsystem's output events from §T.3.

### T.8 Where input drivers actually live, and the three-layer pattern

There is no single "input bus." Input drivers attach to whatever bus the hardware is on and register an `input_dev`:

```
   USB device ──► usbhid ──┐
   I²C device ──► i2c-hid ─┼──► hid-core ──► hid-input ──► input_dev ──► evdev ──► /dev/input/eventN
   BT device  ──► hidp ────┘                     ▲
                                                 │
   PS/2 ──► i8042 (serio) ──► atkbd/psmouse ─────┘ (direct input_dev)
   I²C touch ──► goodix/edt-ft5x06 ───────────────┘ (direct input_dev)
   GPIO ──► gpio-keys ────────────────────────────┘ (direct input_dev)
```

Two layering styles, and choosing between them is a real design decision:

- **Direct `input_dev`**: the driver knows the hardware's protocol and emits events itself. Right for non-HID hardware (most I²C touch controllers, `gpio-keys`, sensors).
- **Via HID**: the driver is a `hid_driver` and lets `hid-input` do the usage→code mapping, optionally intercepting with `->raw_event()` or `->input_mapping()`. Right whenever the device speaks HID, *even badly* — fixing a descriptor beats reimplementing a parser.

`serio` is a third thing worth knowing about: a small bus for legacy serial input ports (i8042 keyboard controller, PS/2, some touchpads) with its own probe/protocol-detection protocol. It exists because those ports have no identification at all, so drivers must *guess* by probing — the danger discussed in Ch. 40 §T.4, institutionalised.

### T.9 Why you should not read `/dev/input/event*` in an application

Everything above describes a *kernel* ABI, not an application API. Between the two sits **libinput**, and the reason is a clean statement of where policy belongs:

The kernel reports what the hardware did. It does not, and should not, decide:

- whether two near-simultaneous contacts are a two-finger scroll or a right-click;
- pointer acceleration curves;
- palm rejection;
- tap-to-click timing;
- whether a clickpad's bottom-left region is "button 1" or "button 3".

These are **user-visible policy**, they change with fashion and user preference, and they require state machines with timers and heuristics that have no place in kernel context. Putting them in the kernel would also fix them forever (ABI). So the rule is:

> **The kernel's job is to report hardware events faithfully and describe the device accurately. Everything interpretive belongs in userspace.**

This is Ch. 00 §T.2's policy/mechanism split, and the input subsystem is one of the cleanest applications of it in Linux. The practical consequence for you as a driver author: if you find yourself adding a timer to distinguish a tap from a press, or a threshold to decide "this is a swipe", you are writing userspace code in the kernel. Stop, and export the raw data plus accurate `resolution`/`fuzz` instead.

The corollary for you as a debugger: `libinput debug-events` and `libinput record` show you the *interpreted* view, `evtest`/`evemu-record` show you the *raw* view, and comparing them tells you immediately which side the bug is on.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `drivers/input/input.c` | the core: `input_dev` registration, event routing, `fuzz`/repeat handling |
| `drivers/input/evdev.c` | the `/dev/input/eventN` character device, per-client ring buffers, all the ioctls |
| `drivers/input/input-mt.c` | slot management, `input_mt_sync_frame()`, pointer emulation |
| `drivers/input/mousedev.c`, `joydev.c` | legacy compatibility nodes |
| `drivers/input/keyboard/`, `mouse/`, `touchscreen/`, `misc/` | device drivers by class |
| `drivers/input/serio/` | the serio bus, `i8042`, protocol detection |
| `drivers/hid/hid-core.c` | report descriptor parser, report dispatch, `hid_driver` matching |
| `drivers/hid/hid-input.c` | HID usage → input event-code mapping |
| `drivers/hid/hid-multitouch.c` | the generic HID MT driver — read this one |
| `drivers/hid/usbhid/`, `i2c-hid/` | transports |
| `drivers/hid/hid-*.c` (≈200 files) | per-vendor quirks and fixups |
| `include/uapi/linux/input.h` | `struct input_event`, all the ioctls |
| `include/uapi/linux/input-event-codes.h` | **the actual ABI** |
| `include/linux/input.h`, `input/mt.h`, `hid.h` | in-kernel APIs |
| `Documentation/input/` | excellent; see §4 |

### 1.2 `struct input_dev`, the parts that matter

```c
struct input_dev {
	const char *name, *phys, *uniq;
	struct input_id id;

	unsigned long propbit[BITS_TO_LONGS(INPUT_PROP_CNT)];
	unsigned long evbit [BITS_TO_LONGS(EV_CNT)];
	unsigned long keybit[BITS_TO_LONGS(KEY_CNT)];
	unsigned long relbit[BITS_TO_LONGS(REL_CNT)];
	unsigned long absbit[BITS_TO_LONGS(ABS_CNT)];
	unsigned long swbit [BITS_TO_LONGS(SW_CNT)];
	unsigned long ledbit[BITS_TO_LONGS(LED_CNT)];

	struct input_absinfo *absinfo;
	unsigned int keycodemax, keycodesize;
	void *keycode;

	int (*open)(struct input_dev *);          /* first user opened us */
	void (*close)(struct input_dev *);        /* last user closed */
	int (*event)(struct input_dev *, unsigned int type,
		     unsigned int code, int value);  /* output events in */

	struct input_mt *mt;
	spinlock_t event_lock;
	struct device dev;                         /* Ch. 26: it is a device */
};
```

Three observations:

- The bitmaps *are* the capability ABI; `input_set_capability()` is a helper that sets `evbit` and the per-type bit together, and is preferred because forgetting the `evbit` is a classic bug (events are silently dropped by the core).
- `open`/`close` are **first-open / last-close**, not per-open. This is the natural place to power the hardware up and down — and, combined with runtime PM (Ch. 43 §T.8, Ch. 48), the reason an idle keyboard costs nothing.
- `event_lock` is a spinlock taken by the core around event delivery, which is why `input_report_*()` and `input_sync()` are **safe from interrupt context** — and why they must never sleep.

### 1.3 The event path

```
driver ISR / poll thread
  └─ input_report_abs(dev, ABS_X, x)      → input_event(dev, EV_ABS, ABS_X, x)
       └─ input_handle_event()            [spin_lock_irqsave(&dev->event_lock)]
            ├─ is the bit set in absbit?  no → drop silently
            ├─ fuzz/flat filtering, value unchanged? → drop
            ├─ store into dev->absinfo[code].value
            └─ input_pass_values() → for each attached handle:
                 evdev_events() → for each client:
                      __pass_event() into per-client ring, wake_up poll waiters
```

The "drop silently if the capability bit is not set" behaviour is the number-one cause of "my driver emits events but `evtest` shows nothing." There is no warning by design (the core cannot distinguish a bug from a driver that conditionally reports). `EV_SYN` and `EV_REP` are handled specially.

### 1.4 The HID path

```
usbhid URB completion (Ch. 39)
  └─ hid_input_report(hid, HID_INPUT_REPORT, data, len, 1)
       └─ hid_report_raw_event()
            ├─ driver->raw_event()        /* quirk hook: rewrite/consume */
            ├─ hid_process_report(): walk fields, extract bitfields
            │    └─ for each usage: driver->event() or hidinput_hid_event()
            │         └─ input_event(...)
            └─ input_sync() at end of report
```

and at probe:

```
hid_add_device()
  └─ hid_parse() → driver->report_fixup() → hid_open_report()
       └─ parse descriptor into hid_device->report_enum[]
  └─ hid_hw_start() → hidinput_connect()
       └─ hidinput_configure_usage(): usage → (type, code), set capability bits
```

`->input_mapping()` and `->input_mapped()` are the per-driver hooks into that last step: return `-1` from `input_mapping()` to suppress a usage entirely, or `hid_map_usage_clear()` to redirect it.

---

## 2. Practice

### Lab 45.1 — Observe the protocol before writing any code

```sh
sudo apt install evtest evemu-tools libinput-tools     # or your distro's names
ls -l /dev/input/by-path/ /dev/input/by-id/
sudo evtest                                             # pick a device
```

Do all of these and write down what you see:

1. Move a mouse one pixel diagonally. Confirm you see `REL_X`, `REL_Y`, `SYN_REPORT` — **one** packet, not two (§T.2).
2. Press and hold a key. Observe value 1, then a stream of value 2 (autorepeat, generated by the **kernel** `EV_REP` timer), then value 0.
3. On a laptop, close the lid partially: `SW_LID` on a *different* event node. Note it is `EV_SW`, not `EV_KEY` (§T.3).
4. Dump capabilities:

```sh
sudo evtest /dev/input/event5 | head -40     # prints all supported types/codes
sudo libinput list-devices                    # the interpreted view
```

5. Compare views on a touchpad:

```sh
sudo evtest /dev/input/eventN          # raw: ABS_MT_SLOT, TRACKING_ID, ...
sudo libinput debug-events              # interpreted: POINTER_SCROLL_FINGER
```

Two-finger scroll on the touchpad and watch `evtest` show slots and `libinput` show a scroll event. That contrast **is** §T.9.

6. Record a reproducible trace for later replay:

```sh
sudo evemu-record /dev/input/eventN > touch.evemu
sudo evemu-device touch.evemu &     # creates a virtual clone
sudo evemu-play /dev/input/eventM < touch.evemu
```

This is the single most useful debugging technique in the subsystem: capture once on the broken hardware, replay anywhere.

---

### Lab 45.2 — A complete virtual input device driver

Creates a device that emits a synthetic circular pointer motion plus a button, driven by a timer. It demonstrates capabilities, relative axes, `input_sync()`, and the open/close hooks.

`vinput.c`:

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>
#include <linux/input.h>
#include <linux/timer.h>
#include <linux/kernel.h>

static struct input_dev *idev;
static struct timer_list tick;
static unsigned int phase;
static bool running;

/* 16 points of a circle, radius 10, as (dx, dy) deltas. */
static const s8 dx[16] = { 4, 3, 2, 0,-2,-3,-4,-4,-4,-3,-2, 0, 2, 3, 4, 4 };
static const s8 dy[16] = { 0, 2, 3, 4, 4, 3, 2, 0,-2,-3,-4,-4,-4,-3,-2, 0 };

static void vinput_tick(struct timer_list *t)
{
	unsigned int p = phase++ & 15;

	input_report_rel(idev, REL_X, dx[p]);
	input_report_rel(idev, REL_Y, dy[p]);

	/* Click once per revolution, as a proper press/release pair. */
	if (p == 0)
		input_report_key(idev, BTN_LEFT, 1);
	else if (p == 1)
		input_report_key(idev, BTN_LEFT, 0);

	input_sync(idev);	/* commit this packet - see T.2 */

	if (running)
		mod_timer(&tick, jiffies + msecs_to_jiffies(50));
}

/* Called on FIRST open only. Power the hardware here. */
static int vinput_open(struct input_dev *dev)
{
	pr_info("vinput: open -> starting\n");
	running = true;
	mod_timer(&tick, jiffies + msecs_to_jiffies(50));
	return 0;
}

/* Called on LAST close only. */
static void vinput_close(struct input_dev *dev)
{
	pr_info("vinput: close -> stopping\n");
	running = false;
	del_timer_sync(&tick);
}

/* Output events (LEDs) arrive here. */
static int vinput_event(struct input_dev *dev, unsigned int type,
			unsigned int code, int value)
{
	if (type == EV_LED)
		pr_info("vinput: host set LED %u to %d\n", code, value);
	return 0;
}

static int __init vinput_init(void)
{
	int ret;

	idev = input_allocate_device();
	if (!idev)
		return -ENOMEM;

	idev->name = "Lab Virtual Pointer";
	idev->phys = "lab/input0";
	idev->id.bustype = BUS_VIRTUAL;
	idev->id.vendor  = 0x0001;
	idev->id.product = 0x0001;
	idev->id.version = 0x0100;

	idev->open  = vinput_open;
	idev->close = vinput_close;
	idev->event = vinput_event;

	/* Capabilities: everything we will ever emit must be declared,
	 * or the core drops it silently (see 1.3). */
	input_set_capability(idev, EV_REL, REL_X);
	input_set_capability(idev, EV_REL, REL_Y);
	input_set_capability(idev, EV_KEY, BTN_LEFT);
	input_set_capability(idev, EV_LED, LED_CAPSL);

	timer_setup(&tick, vinput_tick, 0);

	ret = input_register_device(idev);
	if (ret) {
		input_free_device(idev);	/* only before register succeeds */
		return ret;
	}
	pr_info("vinput: registered\n");
	return 0;
}

static void __exit vinput_exit(void)
{
	input_unregister_device(idev);	/* frees idev; do NOT input_free_device */
	del_timer_sync(&tick);
}

module_init(vinput_init);
module_exit(vinput_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Teaching input driver: synthetic pointer");
```

Test:

```sh
sudo insmod vinput.ko
sudo evtest    # select "Lab Virtual Pointer"; the cursor will also move
sudo dmesg | tail
```

Exercises on this driver — do each and observe:

1. **Delete `input_sync()`.** Events still appear in `evtest` (because evtest prints raw events) but the cursor stops moving. This is the torn-state failure of §T.2.
2. **Delete the `input_set_capability(EV_KEY, BTN_LEFT)` line** but keep reporting it. Observe the events vanish with no warning (§1.3).
3. **Never open the device.** `dmesg` shows no "starting" — the timer never runs. Confirm the first-open/last-close semantics by running two `evtest`s and closing one.
4. Send an LED event from userspace and watch `vinput_event` fire:

```c
	struct input_event ev = { .type = EV_LED, .code = LED_CAPSL, .value = 1 };
	write(fd, &ev, sizeof(ev));
```

---

### Lab 45.3 — A multitouch touchscreen driver (protocol B)

Extend Lab 45.2 into a device with two independently moving contacts.

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>
#include <linux/input.h>
#include <linux/input/mt.h>
#include <linux/timer.h>

#define MAX_CONTACTS	5
#define XMAX		1920
#define YMAX		1080

static struct input_dev *ts;
static struct timer_list tick;
static unsigned int t;
static bool running;

static void ts_tick(struct timer_list *unused)
{
	int i, x, y;
	/* Two fingers, out of phase, both present for 40 ticks then lifted. */
	bool active[2] = { true, (t % 80) < 40 };

	for (i = 0; i < 2; i++) {
		input_mt_slot(ts, i);
		input_mt_report_slot_state(ts, MT_TOOL_FINGER, active[i]);
		if (!active[i])
			continue;
		x = XMAX / 2 + (i ? 300 : -300) + (int)(t % 100) * 2;
		y = YMAX / 2 + (int)((t * 3) % 200) - 100;
		input_report_abs(ts, ABS_MT_POSITION_X, x);
		input_report_abs(ts, ABS_MT_POSITION_Y, y);
		input_report_abs(ts, ABS_MT_PRESSURE, 40 + i * 10);
		input_report_abs(ts, ABS_MT_TOUCH_MAJOR, 8);
	}

	/* Closes any slot we did not touch this frame, and drives
	 * single-touch emulation for legacy clients. */
	input_mt_sync_frame(ts);
	input_mt_report_pointer_emulation(ts, true);
	input_sync(ts);

	t++;
	if (running)
		mod_timer(&tick, jiffies + msecs_to_jiffies(20));
}

static int ts_open(struct input_dev *d)
{
	running = true;
	mod_timer(&tick, jiffies + msecs_to_jiffies(20));
	return 0;
}

static void ts_close(struct input_dev *d)
{
	running = false;
	del_timer_sync(&tick);
}

static int __init ts_init(void)
{
	int ret;

	ts = input_allocate_device();
	if (!ts)
		return -ENOMEM;

	ts->name = "Lab Virtual Touchscreen";
	ts->phys = "lab/input1";
	ts->id.bustype = BUS_VIRTUAL;
	ts->open = ts_open;
	ts->close = ts_close;

	input_set_abs_params(ts, ABS_MT_POSITION_X, 0, XMAX, 0, 0);
	input_set_abs_params(ts, ABS_MT_POSITION_Y, 0, YMAX, 0, 0);
	input_set_abs_params(ts, ABS_MT_PRESSURE,   0, 255,  0, 0);
	input_set_abs_params(ts, ABS_MT_TOUCH_MAJOR, 0, 32,  0, 0);

	/* Physical units: essential for userspace gesture thresholds (T.5).
	 * 1920 px across a 154 mm wide panel ~= 12 units/mm. */
	input_abs_set_res(ts, ABS_MT_POSITION_X, 12);
	input_abs_set_res(ts, ABS_MT_POSITION_Y, 12);

	ret = input_mt_init_slots(ts, MAX_CONTACTS,
				  INPUT_MT_DIRECT | INPUT_MT_DROP_UNUSED);
	if (ret)
		goto err;

	timer_setup(&tick, ts_tick, 0);

	ret = input_register_device(ts);
	if (ret)
		goto err;
	return 0;
err:
	input_free_device(ts);
	return ret;
}

static void __exit ts_exit(void)
{
	input_unregister_device(ts);
	del_timer_sync(&tick);
}

module_init(ts_init);
module_exit(ts_exit);
MODULE_LICENSE("GPL");
```

Verify:

```sh
sudo insmod vts.ko
sudo evtest              # watch ABS_MT_SLOT / TRACKING_ID transitions
sudo libinput list-devices | grep -A8 Touchscreen
```

Then, the instructive experiments:

1. Remove `INPUT_MT_DIRECT` and re-load. `libinput` now classifies it as a touchpad, not a touchscreen, and applies pointer acceleration. You changed nothing about the events — only the *property bit* (§T.5).
2. Remove `input_mt_report_pointer_emulation()`. `evtest` still shows MT events, but a pre-2010 client sees a device with no `ABS_X` at all.
3. Remove `input_abs_set_res()`. Then run `libinput debug-events` and try to get consistent scroll thresholds. This is why `resolution` matters.
4. Remove `input_mt_sync_frame()` and instead never set `TRACKING_ID` to -1. Watch the second finger become permanently stuck down — the single most common real touchscreen bug.

---

### Lab 45.4 — `uinput`: create input devices from userspace

`uinput` is the mirror image of evdev: userspace *writes* a device definition and then writes events. It is how `evemu-device`, remote-desktop servers, and test harnesses work — and it is the fastest way to develop the userspace side of an input feature without hardware.

```c
// SPDX-License-Identifier: GPL-2.0
#include <fcntl.h>
#include <linux/uinput.h>
#include <string.h>
#include <unistd.h>
#include <stdio.h>

static void emit(int fd, int type, int code, int val)
{
	struct input_event ie = { .type = type, .code = code, .value = val };

	write(fd, &ie, sizeof(ie));
}

int main(void)
{
	struct uinput_setup us = {
		.id = { .bustype = BUS_USB, .vendor = 0x1234, .product = 0x5678 },
		.name = "Lab uinput Keyboard",
	};
	int fd = open("/dev/uinput", O_WRONLY | O_NONBLOCK);

	if (fd < 0) { perror("open /dev/uinput"); return 1; }

	ioctl(fd, UI_SET_EVBIT, EV_KEY);
	ioctl(fd, UI_SET_KEYBIT, KEY_H);
	ioctl(fd, UI_SET_KEYBIT, KEY_I);

	ioctl(fd, UI_DEV_SETUP, &us);
	ioctl(fd, UI_DEV_CREATE);

	sleep(1);   /* let udev/libinput notice the new device */

	emit(fd, EV_KEY, KEY_H, 1); emit(fd, EV_SYN, SYN_REPORT, 0);
	emit(fd, EV_KEY, KEY_H, 0); emit(fd, EV_SYN, SYN_REPORT, 0);
	emit(fd, EV_KEY, KEY_I, 1); emit(fd, EV_SYN, SYN_REPORT, 0);
	emit(fd, EV_KEY, KEY_I, 0); emit(fd, EV_SYN, SYN_REPORT, 0);

	sleep(1);
	ioctl(fd, UI_DEV_DESTROY);
	close(fd);
	return 0;
}
```

```sh
gcc -o uin uin.c && sudo ./uin      # types "hi" into the focused window
```

Exercises: (a) build a multitouch device with `UI_SET_ABSBIT` + `UI_ABS_SETUP` and replay a recorded gesture; (b) note the security implication — write access to `/dev/uinput` is equivalent to keystroke injection, which is why it is root-only by default. State precisely what a sandbox must do to safely expose it.

---

### Lab 45.5 — Decode a HID report descriptor by hand

```sh
sudo apt install usbutils
lsusb -vd 046d:            # find your device; look for HID Report Descriptor
# The full descriptor bytes:
sudo cat /sys/kernel/debug/hid/*/rdesc
```

`rdesc` prints both the raw bytes and the kernel's parse. Take a simple mouse and decode the first bytes yourself. The item encoding is `bTag[7:4] | bType[3:2] | bSize[1:0]`:

```
05 01        Usage Page (Generic Desktop)
09 02        Usage (Mouse)
A1 01        Collection (Application)
09 01          Usage (Pointer)
A1 00          Collection (Physical)
05 09            Usage Page (Button)
19 01            Usage Minimum (1)
29 03            Usage Maximum (3)
15 00            Logical Minimum (0)
25 01            Logical Maximum (1)
95 03            Report Count (3)
75 01            Report Size (1)
81 02            Input (Data,Var,Abs)     <-- 3 buttons, 1 bit each
95 01            Report Count (1)
75 05            Report Size (5)
81 03            Input (Cnst,Var,Abs)     <-- 5 bits padding
05 01            Usage Page (Generic Desktop)
09 30            Usage (X)
09 31            Usage (Y)
15 81            Logical Minimum (-127)
25 7F            Logical Maximum (127)
75 08            Report Size (8)
95 02            Report Count (2)
81 06            Input (Data,Var,Rel)     <-- X,Y as signed bytes, RELATIVE
C0             End Collection
C0           End Collection
```

Now answer:

1. How many bytes is one input report? (3.)
2. Which bit of byte 0 is the right button?
3. `81 06` has the Relative bit set, `81 02` does not. Trace in `hid-input.c` how that single bit decides `EV_REL` vs `EV_ABS`, and therefore whether the device acts as a mouse or a tablet.
4. Flip Logical Maximum for X to 32767 and Report Size to 16, keep it Absolute, and predict what device class the kernel will expose.

Then watch reports live:

```sh
sudo modprobe usbmon
sudo cat /sys/kernel/debug/hid/*/events     # decoded
sudo usbhid-dump -d 046d: -es               # raw reports
```

---

### Lab 45.6 — Write a HID quirk driver with a descriptor fixup

This is the most common real HID task: a device whose descriptor lies. The pattern:

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>
#include <linux/hid.h>

#define LAB_VENDOR	0x1234
#define LAB_PRODUCT	0x5678

/* The device declares Usage(Y) where it means Usage(Wheel). Repair the
 * descriptor at the boundary; the generic parser then does the rest. */
static const __u8 *lab_report_fixup(struct hid_device *hdev, __u8 *rdesc,
				    unsigned int *rsize)
{
	if (*rsize >= 50 && rdesc[42] == 0x31) {
		hid_info(hdev, "fixing up Usage(Y) -> Usage(Wheel)\n");
		rdesc[42] = 0x38;	/* Generic Desktop: Wheel */
	}
	return rdesc;
}

/* Alternative hook: intervene at mapping time instead of descriptor time. */
static int lab_input_mapping(struct hid_device *hdev, struct hid_input *hi,
			     struct hid_field *field, struct hid_usage *usage,
			     unsigned long **bit, int *max)
{
	if ((usage->hid & HID_USAGE_PAGE) == HID_UP_BUTTON &&
	    (usage->hid & HID_USAGE) == 4) {
		hid_map_usage(hi, usage, bit, max, EV_KEY, BTN_SIDE);
		return 1;	/* handled */
	}
	return 0;		/* let the generic code map it */
}

/* Last resort: rewrite raw report bytes before parsing. */
static int lab_raw_event(struct hid_device *hdev, struct hid_report *report,
			 u8 *data, int size)
{
	if (size == 8 && data[0] == 0x02)
		data[7] &= 0x0f;	/* device sets garbage in the high nibble */
	return 0;
}

static int lab_probe(struct hid_device *hdev, const struct hid_device_id *id)
{
	int ret;

	ret = hid_parse(hdev);		/* calls report_fixup */
	if (ret)
		return ret;

	return hid_hw_start(hdev, HID_CONNECT_DEFAULT);
}

static const struct hid_device_id lab_devices[] = {
	{ HID_USB_DEVICE(LAB_VENDOR, LAB_PRODUCT) },
	{ }
};
MODULE_DEVICE_TABLE(hid, lab_devices);

static struct hid_driver lab_driver = {
	.name		= "hid-lab",
	.id_table	= lab_devices,
	.probe		= lab_probe,
	.report_fixup	= lab_report_fixup,
	.input_mapping	= lab_input_mapping,
	.raw_event	= lab_raw_event,
};
module_hid_driver(lab_driver);
MODULE_LICENSE("GPL");
```

To test without the physical device, use **`uhid`** (`/dev/uhid`), the HID analogue of `uinput`: userspace supplies a report descriptor and feeds reports.

```sh
sudo apt install hid-tools        # python-hidtools
sudo python3 -m hidtools.cli.hid-replay -h
```

`hid-recorder` captures a real device's descriptor + report stream; `hid-replay` recreates it via `uhid` on any machine. Combined with the kernel's `tools/testing/selftests/hid/` (a full BPF+uhid test suite), this gives you a complete HID development loop with no hardware.

Choose deliberately between the three hooks above, and be able to justify it:

| Hook | Use when |
|---|---|
| `report_fixup` | the descriptor is wrong — **preferred**, keeps everything downstream generic |
| `input_mapping` | the descriptor is right but the semantic mapping should differ |
| `raw_event` | the report *data* is malformed, or reports must be split/merged |

---

### Lab 45.7 — Debug the classic touchscreen problems

**(a) Coordinates from the wrong corner / axes swapped.** The fix is *not* in the driver:

```dts
	touchscreen@38 {
		compatible = "edt,edt-ft5406";
		reg = <0x38>;
		interrupt-parent = <&gpio1>;
		interrupts = <5 IRQ_TYPE_EDGE_FALLING>;
		touchscreen-size-x = <1024>;
		touchscreen-size-y = <600>;
		touchscreen-inverted-x;
		touchscreen-swapped-x-y;
	};
```

`touchscreen_properties` / `touchscreen_parse_properties()` (`drivers/input/touchscreen.c`) implement these generically, and `touchscreen_report_pos()` applies them. Rule: **orientation is board data, not driver data** — the same argument as GPIO polarity in Ch. 42 §T.5 and clock rates in Ch. 43 §T.4. A driver with an `#ifdef MY_BOARD` axis flip is a bug.

**(b) Stuck contact.** Use `evemu-record`, look for a `TRACKING_ID` that goes positive and never returns to `-1`. Cause is almost always an error path that returns early without `input_mt_sync_frame()`.

**(c) Events stop after an I²C error.** Touch controllers assert an IRQ until read. If the read fails and you return from the threaded IRQ without clearing, the level-triggered IRQ never re-fires (Ch. 17 §T.3). Correct handling retries or resets the controller.

**(d) Jitter.** Set `fuzz` in `input_set_abs_params()` rather than filtering in the driver — one line, and it composes with everything (§T.5).

**(e) Verify end to end:**

```sh
sudo evemu-record  /dev/input/eventN > trace.evemu   # raw
sudo libinput record /dev/input/eventN > trace.yml   # raw + interpreted + context
sudo libinput debug-events --verbose
```

If `evemu-record` shows correct coordinates and `libinput` misbehaves, the bug is in userspace or in a missing property bit — not in your driver. Being able to make that call in thirty seconds is the point of this lab.

---

## 3. Mastery drills

1. Prove that a delta protocol plus `SYN_DROPPED` plus full-state query ioctls is sufficient for a client to always recover exact device state, and identify the assumption the proof requires about ioctl atomicity with respect to the event stream.

2. MT protocol A cannot track contact identity. Formalise the frame-to-frame assignment problem it forces on userspace (it is a minimum-weight bipartite matching) and give a concrete two-finger trajectory where the minimum-weight solution is physically wrong.

3. `input_mt_report_pointer_emulation()` derives a legacy single-touch stream from MT data. Specify its behaviour when three fingers are down and the "primary" one lifts. What choice preserves the illusion best, and what does it cost?

4. The input core silently drops events for codes whose capability bit is unset. Propose a `CONFIG_INPUT_DEBUG`-style change that warns instead, and argue why it cannot be unconditional.

5. HID report descriptors are parsed from untrusted device input. Enumerate the parser's attack surface and describe, for each, the bound that prevents it (read `hid_open_report()` and `hid_parser_main()` and cite line-level checks).

6. `report_fixup` mutates a descriptor before parsing. Is this composable — can two drivers both fix up one device? What does the architecture do instead, and what would it take to make chained fixups safe?

7. Design the kernel/userspace split for palm rejection. Argue where each of these belongs: contact area reporting, a size threshold, "ignore contacts near the edge while typing", and per-user sensitivity. Justify each with the criterion from §T.9.

8. `resolution` converts device units to millimetres. Some devices cannot know their own physical size (a USB touchscreen paired with an arbitrary panel). How should the stack handle that, and what does the current ABI actually do?

9. `EV_REP` autorepeat is generated in the kernel by a timer. Argue for and against moving it entirely to userspace, given the ABI constraint and the existence of consoles with no compositor.

10. Per-client ring buffers cost memory proportional to (clients × buffer size). A modern desktop has a compositor, an input-method daemon, and a screen reader on every device. Compute the cost for a 10-device machine, then design a shared-buffer alternative and explain why the per-client isolation property makes it unattractive.

11. `uinput` write access is keystroke-injection capability. Design a mechanism that allows a sandboxed application to create *only* a device that cannot inject into other applications' input focus. What kernel facility would have to change?

12. The `serio` bus probes PS/2 ports by sending commands and pattern-matching replies — exactly the dangerous probing warned against in Ch. 40 §T.4. Why is it acceptable here and not there? State the property of the port that makes the difference.

13. Trace one `REL_X` event from an interrupt handler to a compositor's `read()`, naming every lock taken, every buffer copied into, and every wakeup. Estimate the end-to-end latency and identify the largest term.

---

## 4. Further reading

**Kernel documentation** (`Documentation/input/`)

- `input.rst` ★★★ — the subsystem overview; short and authoritative.
- `event-codes.rst` ★★★ — the precise semantics of every type, including the `EV_KEY` vs `EV_SW` and `EV_REL` vs `EV_ABS` distinctions of §T.3. Read before choosing codes.
- `multi-touch-protocol.rst` ★★★ — protocols A and B, slots, tracking IDs, `SYN_MT_REPORT`. The definitive text on §T.6.
- `input-programming.rst` ★★ — writing a driver, essentially Lab 45.2 in prose.
- `uinput.rst` ★★, `gameport-programming.rst`, `ff.rst` (force feedback) ★★.
- `Documentation/hid/hid-transport.rst`, `hidintro.rst`, `hid-bpf.rst` ★★★ — `hid-bpf` (6.3+) is the modern way to write quirks *without a kernel module*; read it after Lab 45.6 and reconsider that lab's design.
- `Documentation/devicetree/bindings/input/touchscreen/touchscreen.yaml` ★★★ — the common properties from Lab 45.7.

**Specifications**

- *USB Device Class Definition for Human Interface Devices (HID)*, version 1.11, USB-IF — the report descriptor grammar. Sections 5–6 and Appendix B are what you actually need.
- *HID Usage Tables*, USB-IF (updated regularly) — the vocabulary. Usage pages 0x01, 0x07, 0x09, 0x0C (Consumer), 0x0D (Digitizers).
- *Windows Precision Touchpad* specification (Microsoft) — de facto standard that most modern touchpads implement; explains many otherwise inexplicable descriptor conventions.

**Source worth reading**

- `drivers/input/input.c` — `input_handle_event()` and `input_pass_values()`; 200 lines that contain the whole model.
- `drivers/input/evdev.c` — `evdev_pass_values()` for `SYN_DROPPED` handling; `evdev_do_ioctl()` for the full capability ABI.
- `drivers/input/input-mt.c` — all of it; it is short and it is §T.6.
- `drivers/hid/hid-core.c` — `hid_open_report()`, `hid_parser_main()`, `hid_report_raw_event()`.
- `drivers/hid/hid-input.c` — `hidinput_configure_usage()`: the usage→code mapping table, worth skimming end to end once.
- `drivers/hid/hid-multitouch.c` — a large, high-quality driver handling dozens of vendor variations through one generic path.
- `drivers/input/touchscreen/edt-ft5x06.c` — a clean, complete I²C touchscreen driver; combines Ch. 40, Ch. 42, Ch. 43 and this chapter.
- `drivers/input/keyboard/gpio_keys.c` — the simplest useful input driver; read it in ten minutes.

**Userspace**

- **libinput** documentation (wayland.freedesktop.org/libinput/doc/latest/) ★★★ — especially "Normalization of relative motion", "Touchpad gestures", "Palm detection", and the "Device quirks" pages. This is where the policy of §T.9 is documented, and it explains what your driver must provide for it to work.
- `libevdev` — the correct way to read evdev, including `SYN_DROPPED` resync done properly.
- **hid-tools** (`hid-recorder`, `hid-replay`) and `tools/testing/selftests/hid/` ★★★ — hardware-free HID development.

**LWN and history**

- Vojtech Pavlík's original input subsystem announcements (2000–2001) and `Documentation/input/` history in git — the design rationale for the narrow waist.
- Henrik Rydberg's multitouch series (2008–2010) and the protocol A→B transition discussion — §T.6 argued in public.
- "Input handling in Wayland" and libinput's introduction (Peter Hutterer's blog, `who-t.blogspot.com`) ★★★ — the clearest writing on the kernel/userspace policy boundary that exists.
- "HID-BPF" (LWN, 2022–2023) — replacing quirk modules with eBPF, and why the maintainers wanted it.

**Tools**

- `evtest`, `evemu-record`/`evemu-play`/`evemu-device`, `libinput record`/`replay`/`debug-events`/`measure`
- `usbhid-dump`, `lsusb -v`, `/sys/kernel/debug/hid/*/rdesc` and `/events`
- `udevadm info /dev/input/eventN`, `udevadm test` — how devices get their `ID_INPUT_*` classification
- `/proc/bus/input/devices` — the whole picture in one file

---

→ Next: [46-netdev-drivers.md](46-netdev-drivers.md)

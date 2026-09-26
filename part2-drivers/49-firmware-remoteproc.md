# Chapter 49 — Firmware loading, remoteproc, rpmsg, and mailboxes

> **Goal:** Understand why a modern SoC is not one computer but a *federation* of heterogeneous processors, why the kernel needed a way to boot and talk to peers it does not schedule, how firmware loading is a trust and lifetime problem disguised as a file read, why remoteproc reuses ELF and virtio rather than inventing formats, how a mailbox turns a shared-memory channel into a signalled one, and what happens to all of it across suspend and crash. By the end you can boot a remote core, establish an rpmsg channel, write both ends of a mailbox-mediated protocol, and debug the "the firmware loaded but the remote is silent" class of failure.

---

## Theory & First Principles

### T.0 — Start here: your "CPU" is a dozen computers

Run this on any modern SoC or laptop:

```bash
ls /lib/firmware/ | wc -l          # hundreds to thousands of blobs
dmesg | grep -i 'firmware\|loading.*\.bin' | head -20
ls /sys/class/remoteproc/ 2>/dev/null
```

A phone SoC labelled "8-core" typically contains, in addition to those eight:

```
  application cores (Linux)     8 x Cortex-A
  modem DSP                     running its own RTOS, ~10M lines of firmware
  audio DSP                     its own RTOS
  image signal processor        its own firmware
  GPU                           its own microcode + scheduler firmware
  security processor (TEE)      its own OS, MORE privileged than Linux
  sensor hub / always-on MCU    runs while Linux is asleep
  power management controller   runs before Linux boots (Ch. 90 BL31)
  Wi-Fi/BT combo                its own firmware, often larger than the driver
```

> **Linux is not "the OS on this machine." It is one process in a distributed system, and
> often not the most privileged one.**

That reframing is the point of the chapter, and it changes what a "driver" means. For these
devices the driver does not program hardware registers to make something happen. It:

1. **Loads an executable** into another processor's memory,
2. **Starts** that processor,
3. **Establishes a communication channel** with it,
4. **Exchanges messages** over that channel,
5. and **recovers** when the other processor crashes.

That is not device driving. **That is distributed systems**, and every distributed-systems
problem shows up:

| Problem | Where it appears |
|---|---|
| **Versioning** | Firmware and driver ship separately and must interoperate across versions |
| **Partial failure** | The DSP crashed. Linux did not. Now what? |
| **Recovery** | Reset it, reload it, replay in-flight requests — or give up |
| **Shared memory consistency** | Two processors, one buffer, no cache coherence between them (Ch. 13, Ch. 35) |
| **Trust** | The modem has DMA access. Is it in your TCB? Usually yes, and that should bother you |
| **Debugging** | You cannot attach a debugger. You get a `devcoredump` and a log ring |

**Two mechanisms carry most of the weight:**

**`request_firmware()`** — fetch a blob from userspace, because it cannot live in the kernel:

```c
ret = request_firmware(&fw, "vendor/dsp-v2.bin", dev);
/* Synchronous and SLEEPS. It waits for userspace. Which means:
 *   - it CANNOT be called from probe if probe runs before the rootfs is up
 *   - hence request_firmware_nowait() and the whole deferred-loading dance */
```

And the reason it is a separate file at all is *licensing*: the blob is usually not
GPL-compatible, so it lives in `linux-firmware`, ships separately, and is loaded at runtime.
A legal constraint produced an architectural one — which is worth noticing as a general
pattern (Ch. 99 §T.3).

**`remoteproc` + `rpmsg`** — lifecycle and messaging for the other processor:

```
  remoteproc:  parse the ELF, load segments into the remote's memory,
               set up carve-outs, release reset, monitor for crashes
  rpmsg:       a virtio-based message bus over shared memory,
               with named channels -- effectively sockets to another CPU
```

**And the recovery story is the part people skip and then regret.** When the DSP crashes:

```
  watchdog fires -> remoteproc detects -> devcoredump captured (for later analysis)
                 -> the rproc is reset and reloaded
                 -> rpmsg channels are torn down and re-announced
                 -> the driver must REPLAY or FAIL in-flight requests
```

If your driver does not handle step four, a recoverable firmware crash becomes a permanently
wedged device — which is the single most common field failure in this area, and is why
Ch. 101's bring-up checklist includes exercising it deliberately.

```bash
cat /sys/class/remoteproc/remoteproc0/state     # offline / running / crashed
echo stop  > /sys/class/remoteproc/remoteproc0/state
echo start > /sys/class/remoteproc/remoteproc0/state
ls /sys/class/devcoredump/                      # crash dumps waiting to be read
ls /sys/bus/rpmsg/devices/
dmesg | grep -i 'remoteproc\|rproc\|crash'
```

---

### T.1 The SoC is a distributed system

Every chapter so far treated the SoC as "a CPU running Linux, plus peripherals the CPU drives." That model is thirty years out of date. A current application processor contains:

| Processor | Typical role | Runs |
|---|---|---|
| Application cores (A-class) | Linux | Linux |
| Real-time cores (Cortex-M/R, Hexagon) | motor control, sensor fusion, audio | RTOS or bare metal |
| DSPs | audio, camera ISP control, modem baseband | vendor RTOS |
| GPU | graphics/compute | firmware + command streams |
| Modem | cellular | an entire separate OS |
| Security processor | key storage, secure boot | TEE / proprietary |
| Power management controller | voltage/clock sequencing | firmware, often before Linux boots |
| Sensor hub | always-on sensing | RTOS |
| NIC/storage controllers | offload | firmware |

Linux runs on *one* of these and is a peer, not a master, to several of the others. The modem may have booted first. The power controller definitely did. The security processor may be able to read Linux's memory while Linux cannot read its own.

This changes the problem from "driver controls device" to a genuine **distributed systems problem** with the usual concerns:

- **Lifecycle**: who boots whom, in what order, and what happens when one dies?
- **Naming and discovery**: how does Linux learn what services the remote offers?
- **Communication**: shared memory is not enough; you need signalling and flow control.
- **Failure**: a peer can hang or crash independently. Recovery must not require rebooting Linux.
- **Trust**: the peer executes code Linux loaded, in an address space Linux may not be able to inspect.

The subsystems in this chapter are Linux's answers, and their design is best understood as *distributed systems design constrained by having no network*.

### T.2 Firmware loading: a file read with four hard problems

`request_firmware(&fw, "name.bin", dev)` looks trivial. It is not, and each of its complications is instructive.

**(a) The kernel cannot read files, and should not want to.** The kernel has no notion of the user's filesystem layout, and at the moment a driver probes, the root filesystem may not be mounted. The original solution was a *userspace helper* (`hotplug`, then udev): the kernel emitted a uevent, userspace found the file and wrote it back through sysfs. This worked but was slow, racy, and could deadlock (§T.3). The modern default is **direct filesystem loading** (`firmware_class`'s `fw_get_filesystem_firmware()`), which reads from a fixed search path (`/lib/firmware`, plus `/lib/firmware/updates`, plus a configurable `path` parameter) using kernel file I/O — accepting that the kernel now knows one filesystem convention, in exchange for removing an entire class of race.

Four sources, tried in order:

| Source | When |
|---|---|
| **Built-in** (`CONFIG_EXTRA_FIRMWARE`) | firmware compiled into vmlinux; needed for devices required to boot |
| **Firmware cache** | a copy retained across suspend (§T.5) |
| **Filesystem** | `/lib/firmware/...`; the normal path |
| **Userspace fallback** | `CONFIG_FW_LOADER_USER_HELPER`; sysfs handshake, now mostly for special cases |

**(b) The lifetime problem.** `request_firmware()` allocates a buffer, fills it, and hands you `fw->data`/`fw->size`. You must `release_firmware()` — and, critically, **most devices need the firmware again later** (after a resume, after a reset). Keeping a copy costs memory; re-reading costs a filesystem access at the worst possible moment. Hence the cache (§T.5).

**(c) The context problem.** `request_firmware()` *sleeps* and does filesystem I/O. It cannot be called from atomic context, from a `_noirq` PM callback, or from any path that the filesystem might depend on. This is the same re-entrancy lattice as `GFP_NOFS` (Ch. 11 §T.4): the driver must know whether it is *below* the filesystem in the dependency graph. A storage controller driver calling `request_firmware()` during resume is asking the filesystem to read from a device that is not yet working.

The API therefore has a family:

| Function | Semantics |
|---|---|
| `request_firmware()` | blocking, warns if missing |
| `firmware_request_nowarn()` | blocking, silent if missing (for optional firmware) |
| `request_firmware_direct()` | blocking, **no** userspace fallback; for probe-time optional firmware |
| `request_firmware_nowait()` | **asynchronous**, callback-based; the right choice when probe must not block |
| `request_firmware_into_buf()` | load into a caller-provided (often DMA-capable, contiguous) buffer |
| `request_partial_firmware_into_buf()` | load a slice; for multi-segment images |

`request_firmware_nowait()` exists because a driver that blocks in probe blocks the whole device-probing sequence, and because firmware may legitimately arrive after boot (e.g. from a later-mounted partition). Its callback runs in a workqueue, and the driver must handle "the device was unbound before the firmware arrived" — a real lifetime hazard that `module_put`/`device_get` reference handling in the API addresses.

**(d) The trust problem.** Firmware is code that will execute on a processor with DMA access to all of memory. On a lockdown/Secure Boot system, loading unsigned code into such a processor is equivalent to loading an unsigned kernel module. Hence:

- `CONFIG_FW_LOADER_USER_HELPER` is disabled under lockdown (userspace could supply arbitrary bytes).
- Many SoCs require firmware to be **signed and verified by hardware or by a secure monitor**, not by Linux — Linux merely places the bytes and asks the secure world to authenticate and release the core.
- `request_firmware` itself does not verify anything. Verification, where it exists, is the platform's job.

This is worth stating plainly because it surprises people: **`request_firmware()` provides no integrity guarantee.** The guarantee, if any, comes from the filesystem's provenance and from hardware verification downstream.

### T.3 The historical deadlock, and what it teaches

The userspace-helper firmware loader could deadlock during system resume, and the shape of the bug is worth carrying:

1. System resumes; a driver's `->resume` calls `request_firmware()`.
2. The kernel emits a uevent; udev must run to answer it.
3. udev is a userspace process. Userspace is **frozen** until resume completes (Ch. 48 §T.6).
4. Deadlock.

Two fixes were applied, and both are good patterns:

- **Do the risky work when it is safe, and cache the result.** The firmware cache (§T.5) loads during `->prepare` (before freezing) and keeps the data across the sleep, so `->resume` never touches the filesystem.
- **Remove the dependency entirely.** Direct filesystem loading eliminated the userspace round trip for the common case.

> **General principle: during any global state transition, a component must not depend on another component that is also transitioning. If it must, hoist the dependency to before the transition begins.**

### T.4 remoteproc: booting a processor is a driver operation

Once you accept §T.1, "boot the DSP" becomes an ordinary device operation, and `remoteproc` is the framework that standardises it.

The lifecycle:

```
   OFFLINE ──start──► loading firmware ──► RUNNING ──stop──► OFFLINE
                                             │
                                          crash
                                             ▼
                                          CRASHED ──recover──► RUNNING
```

and the driver's job is four callbacks:

```c
struct rproc_ops {
	int  (*prepare)(struct rproc *rproc);        /* power/clocks/memory */
	int  (*start)(struct rproc *rproc);          /* release from reset */
	int  (*stop)(struct rproc *rproc);           /* halt and reset */
	void (*kick)(struct rproc *rproc, int vqid); /* signal the remote */
	void *(*da_to_va)(struct rproc *, u64 da, size_t len, bool *is_iomem);
	int  (*parse_fw)(struct rproc *, const struct firmware *);
	int  (*load)(struct rproc *, const struct firmware *);
	int  (*attach)(struct rproc *rproc);         /* remote already running */
	unsigned long (*panic)(struct rproc *rproc);
};
```

Three design decisions in remoteproc are worth extracting.

**(a) The firmware format is ELF.** Not a vendor blob format — ELF, with program headers giving load addresses and sizes. This is pure reuse: the kernel already knows ELF (Ch. 20 §T.4), toolchains already emit it, and `readelf` already debugs it. A remoteproc driver's `load` is essentially "for each PT_LOAD segment, `da_to_va()` the destination and `memcpy`."

**(b) `da_to_va()` is the address-translation abstraction.** The remote processor sees *device addresses* that may bear no relation to physical addresses — it may have its own MMU, its own tightly-coupled memory at address 0, or a fixed window. The framework knows nothing about this; the driver supplies the mapping. This is the same "three address spaces" discipline as Ch. 35 §T.1, extended to a fourth: the remote's view.

**(c) The resource table is the interface contract.** A special ELF section `.resource_table` in the firmware contains a list of typed entries declaring what the firmware *needs*:

| Resource type | Meaning |
|---|---|
| `RSC_CARVEOUT` | "allocate me N bytes of contiguous memory at (or mapped to) device address D" |
| `RSC_DEVMEM` | "map this physical region into my address space" |
| `RSC_TRACE` | "here is a ring buffer where I will write log output" |
| `RSC_VDEV` | "I implement this virtio device; here are its vrings" |

The resource table inverts the usual direction: **the firmware describes its own requirements, and the kernel satisfies them.** That makes one kernel driver work with many firmware images, and it makes the firmware self-describing in the same way a device tree node or a HID report descriptor is (Ch. 32, Ch. 45 §T.7). The same caveats apply: the table is *parsed from an untrusted image*, so bounds checking matters, and a wrong table produces confusing failures.

`RSC_TRACE` deserves special mention because it is the single most valuable debugging feature here: it gives you `/sys/kernel/debug/remoteproc/remoteproc0/trace0`, a live view of the remote's printf output. Without it, debugging a silent remote is nearly hopeless.

**Attach mode** (`->attach`) handles the increasingly common case where the remote was booted by the bootloader or by firmware before Linux started — Linux must *adopt* a running processor rather than boot it, recovering the resource table from a known location.

### T.5 The firmware cache: PM-aware resource retention

Introduced to solve §T.3, the firmware cache is a good study in PM-aware resource management:

- On `request_firmware()`, the core records that this device used firmware `name`.
- On `->prepare` (before the freezer runs, before any device suspends), the PM notifier reloads every cached firmware into memory.
- After resume completes, the cache is dropped (after a delay) to reclaim the memory.

So the memory cost is paid only around a suspend, and the filesystem access happens at the one moment when it is guaranteed safe. `request_firmware()` during resume then hits the cache and returns instantly.

Drivers can opt out (`FW_OPT_NOCACHE` via `firmware_request_nowarn` variants) when the firmware is huge and the driver can reload it lazily.

The generalisable pattern:

> **If a resource is expensive or unsafe to acquire at time T, acquire it at a safe time T′ < T and hold it, and make the holding period as short as correctness allows.**

### T.6 Mailboxes: turning shared memory into a channel

Two processors sharing memory can exchange data but cannot *notify* each other without polling. A **mailbox** (also "IPC interrupt", "doorbell", "hardware semaphore") is the signalling primitive: a register that, when written by one core, raises an interrupt on another.

Mailbox hardware varies enormously, and the framework's job is to hide that:

| Hardware style | Payload | Example |
|---|---|---|
| Pure doorbell | none — "look at shared memory" | most SoC IPC |
| FIFO mailbox | a few 32-bit words per message | TI OMAP, ST |
| Register mailbox | a fixed-size register block | Xilinx, some Qualcomm |
| Shared memory + doorbell | arbitrary, in SRAM | Qualcomm SMEM/SMD, most modern |

The Linux mailbox framework (`drivers/mailbox/`) models it as **channels**:

```c
/* Consumer side */
struct mbox_client cl = {
	.dev		= dev,
	.tx_block	= true,          /* block until TX completes */
	.tx_tout	= 500,           /* ms */
	.knows_txdone	= false,
	.rx_callback	= my_rx_callback,
	.tx_done	= my_tx_done,
};
chan = mbox_request_channel(&cl, 0);     /* index into DT "mboxes" */
mbox_send_message(chan, &msg);
```

```c
/* Provider side */
struct mbox_chan_ops {
	int  (*send_data)(struct mbox_chan *chan, void *data);
	int  (*startup)(struct mbox_chan *chan);
	void (*shutdown)(struct mbox_chan *chan);
	bool (*last_tx_done)(struct mbox_chan *chan);
	bool (*peek_data)(struct mbox_chan *chan);
};
```

Three points that determine whether your use is correct:

1. **`rx_callback` runs in interrupt context** (or a tasklet, depending on the controller). Do not sleep in it. Hand off to a workqueue (Ch. 18).
2. **TX completion is not message delivery.** `tx_done` means the mailbox hardware accepted the message, not that the remote processed it. Application-level acknowledgement is the client's problem — a distributed-systems truth that bites people who assume the framework provides it.
3. **The channel is a resource; requesting it twice fails.** Like exclusive resets (Ch. 43 §T.7), the framework makes conflicting ownership a probe error rather than silent corruption.

DT binding, following the `#*-cells` provider-defined-calling-convention pattern of Ch. 32 §T.5:

```dts
	mailbox: mailbox@2000 {
		compatible = "vendor,mbox";
		reg = <0x2000 0x100>;
		#mbox-cells = <1>;
		interrupts = <...>;
	};

	dsp@10000000 {
		compatible = "vendor,dsp-rproc";
		mboxes = <&mailbox 0>, <&mailbox 1>;   /* tx, rx */
		mbox-names = "tx", "rx";
		memory-region = <&dsp_reserved>;
		...
	};
```

### T.7 rpmsg: virtio between processors

Given remoteproc (boot) and mailbox (signal), you still need a *protocol*: named channels, multiplexing, flow control, discovery. Linux's answer is **rpmsg**, and its key design decision is:

> **Reuse virtio.** The transport between Linux and a remote core is a pair of virtqueues in shared memory, signalled by a mailbox. The remote is, from Linux's perspective, a virtio device.

This is a remarkable piece of reuse (Ch. 04 §T.5 introduced virtio as a *virtualisation* interface). The properties that made virtio right for hypervisor↔guest — shared-memory rings, descriptor chains, available/used index pairs, a signalling mechanism abstracted away — are exactly the properties needed for CPU↔DSP. The alternative, a bespoke ring protocol per vendor, is what existed before and it was a mess.

The layering:

```
   your driver (rpmsg_driver)          e.g. audio control, sensor hub protocol
        │  rpmsg_send() / ->callback()
   ┌────▼───────────────────────────┐
   │ rpmsg core: named endpoints,   │   channel names, address allocation,
   │ address multiplexing, NS       │   name-service announcements
   └────┬───────────────────────────┘
   ┌────▼───────────────────────────┐
   │ virtio_rpmsg_bus               │   messages -> virtqueue buffers
   └────┬───────────────────────────┘
   ┌────▼───────────────────────────┐
   │ remoteproc virtio              │   vrings from RSC_VDEV in the resource table
   └────┬───────────────────────────┘
   ┌────▼───────────────────────────┐
   │ mailbox (kick) + shared memory │
   └────────────────────────────────┘
```

The rpmsg message header is deliberately tiny:

```c
struct rpmsg_hdr {
	u32 src;       /* source address (endpoint) */
	u32 dst;       /* destination address */
	u32 reserved;
	u16 len;
	u16 flags;
	u8  data[];
};
```

Sixteen bytes of header, addresses rather than names on the wire, 512-byte default buffers. Names appear only in the **name service**: a well-known endpoint (address 53) over which either side announces "channel `rpmsg-client-sample` is at address 0x400". That announcement triggers the kernel's device model — an `rpmsg_device` is created, and the matching `rpmsg_driver` probes (Ch. 27's bus/probe/binding, applied across a processor boundary).

That is the elegant part: **a remote processor announcing a service causes a Linux driver to probe, exactly as plugging in USB hardware does.** The bus abstraction of Ch. 27 turns out to be general enough to cover "a peer advertised a service."

A driver:

```c
static int sample_probe(struct rpmsg_device *rpdev)
{
	dev_info(&rpdev->dev, "new channel: 0x%x -> 0x%x\n",
		 rpdev->src, rpdev->dst);
	return rpmsg_send(rpdev->ept, "hello", 5);
}

static int sample_cb(struct rpmsg_device *rpdev, void *data, int len,
		     void *priv, u32 src)
{
	print_hex_dump(KERN_DEBUG, "rx: ", DUMP_PREFIX_NONE, 16, 1,
		       data, len, true);
	return 0;
}

static struct rpmsg_device_id sample_id_table[] = {
	{ .name = "rpmsg-client-sample" }, { },
};
MODULE_DEVICE_TABLE(rpmsg, sample_id_table);

static struct rpmsg_driver sample_drv = {
	.drv.name	= KBUILD_MODNAME,
	.id_table	= sample_id_table,
	.probe		= sample_probe,
	.callback	= sample_cb,
};
module_rpmsg_driver(sample_drv);
```

**`rpmsg_char`** (`/dev/rpmsg_ctrlN` → `/dev/rpmsgN`) exposes endpoints to userspace as character devices, which is how most real applications talk to a DSP: the kernel provides transport, userspace provides protocol. That is the same policy/mechanism split as Mesa and libcamera in Ch. 47 §T.9.

### T.8 Failure, recovery, and coredumps

A remote processor will hang. The framework's response is the interesting part, because it must contain the failure:

1. **Detection.** Either a watchdog interrupt from the hardware, or the driver's own timeout, calls `rproc_report_crash(rproc, RPROC_WATCHDOG)`.
2. **Coredump.** Before resetting, `rproc_coredump()` captures the remote's memory into an ELF core file, available at `/sys/class/remoteproc/remoteproc0/coredump` — readable by `gdb` with the firmware's symbols. This is genuinely good tooling: a crashed DSP produces a debuggable core dump.
3. **Recovery.** `rproc_trigger_recovery()` stops, reloads firmware, and restarts. Every rpmsg channel is torn down (its drivers' `remove()` called) and rebuilt on announcement.
4. **Containment.** Linux keeps running. The whole point is that a peer's failure is not the system's failure.

The obligation this places on rpmsg *drivers* is the one people miss:

> **An rpmsg driver's `remove()` can be called at any time because the remote crashed.** Any in-flight request must be completed with an error, any waiting thread woken, and any state reset. This is Ch. 25's P12 stop-drain-free, triggered asynchronously by a peer.

`/sys/class/remoteproc/remoteproc0/recovery` controls whether recovery is automatic; setting it to `disabled` freezes the crashed state for debugging, which is the first thing to do when investigating.

### T.9 Where this fits: the other channels to firmware

remoteproc/rpmsg is one family. A complete picture of "Linux talking to other processors" includes:

| Mechanism | Used for |
|---|---|
| **remoteproc + rpmsg** | general-purpose: DSPs, M-cores, modems |
| **SCMI / SCPI** (`firmware/arm_scmi/`) | standardised protocol for clocks, voltages, sensors, resets provided by a power controller — a *firmware-provided implementation* of Ch. 43's subsystems |
| **PSCI** | CPU power control (core on/off, suspend, system reset) via SMC calls |
| **SMCCC / SMC calls** | direct synchronous calls into secure firmware/TEE |
| **TEE subsystem** (`drivers/tee/`) | OP-TEE and friends: a full secure-world RPC |
| **ACPI AML** (Ch. 33) | firmware *bytecode* the kernel interprets |
| **FFA** (Firmware Framework for Arm) | partitions and messaging in newer Arm systems |
| **Device-specific firmware protocols** | NIC/NVMe admin queues, GPU firmware interfaces |

SCMI is worth a closer look because it inverts Ch. 43: instead of Linux driving clock and regulator hardware, Linux *asks a power controller* over a mailbox-backed protocol, and registers `clk`/`regulator`/`reset` providers that forward requests. The consumer drivers are unchanged — the abstraction held. That is a strong validation of the subsystem boundaries from Ch. 43 §T.2.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `drivers/base/firmware_loader/main.c` | `request_firmware()` and friends, the search path |
| `drivers/base/firmware_loader/fallback.c` | the sysfs userspace helper |
| `drivers/base/firmware_loader/sysfs_upload.c` | firmware *upload* (device flashing), 5.19+ |
| `drivers/remoteproc/remoteproc_core.c` | lifecycle, ELF loading, resource table parsing |
| `drivers/remoteproc/remoteproc_elf_loader.c` | `rproc_elf_load_segments()` |
| `drivers/remoteproc/remoteproc_virtio.c` | vdev/vring setup from `RSC_VDEV` |
| `drivers/remoteproc/remoteproc_debugfs.c` | `trace0`, `carveout_memories`, `resource_table` |
| `drivers/remoteproc/remoteproc_coredump.c` | crash dumps |
| `drivers/rpmsg/rpmsg_core.c`, `virtio_rpmsg_bus.c`, `rpmsg_char.c`, `rpmsg_ns.c` | rpmsg |
| `drivers/mailbox/mailbox.c` | the framework; `*-mailbox.c` are controllers |
| `drivers/firmware/arm_scmi/`, `psci.c`, `smccc/` | firmware protocols |
| `samples/rpmsg/rpmsg_client_sample.c` | a working rpmsg driver |
| `include/linux/firmware.h`, `remoteproc.h`, `rpmsg.h`, `mailbox_client.h`, `mailbox_controller.h` | APIs |
| `Documentation/driver-api/firmware/`, `Documentation/staging/remoteproc.rst`, `rpmsg.rst` | docs |

### 1.2 The resource table

```c
struct resource_table {
	u32 ver;
	u32 num;
	u32 reserved[2];
	u32 offset[];        /* num offsets to entries */
};

struct fw_rsc_hdr {
	u32 type;
	u8  data[];
};

struct fw_rsc_carveout {
	u32 da, pa, len, flags, reserved;
	u8  name[32];
};

struct fw_rsc_vdev {
	u32 id;              /* VIRTIO_ID_RPMSG */
	u32 notifyid;
	u32 dfeatures, gfeatures;
	u32 config_len;
	u8  status, num_of_vrings, reserved[2];
	struct fw_rsc_vdev_vring vring[];
};

struct fw_rsc_trace {
	u32 da, len, reserved;
	u8  name[32];
};
```

`rproc_handle_resources()` walks it twice: once in `rproc_handle_resource_table()` to allocate carveouts (which must happen before loading, because segments load *into* them), and once after to create vdevs.

### 1.3 The boot sequence

```
rproc_boot(rproc)
 └─ rproc_fw_boot()
      ├─ request_firmware(&fw, rproc->firmware, dev)
      ├─ rproc->ops->prepare()               /* clocks, power domain, memory */
      ├─ rproc_parse_fw() → rproc_elf_load_rsc_table()
      │     → find .resource_table section, copy it (cached_table)
      ├─ rproc_handle_resources(RSC_CARVEOUT, RSC_DEVMEM, RSC_TRACE)
      │     → dma_alloc_coherent / ioremap / create debugfs trace files
      ├─ rproc_load_segments()
      │     → for each PT_LOAD: da_to_va(p_paddr) ; memcpy ; memset bss
      ├─ rproc_handle_resources(RSC_VDEV)
      │     → register a virtio device per vdev; vrings live in carveouts
      ├─ rproc->ops->start()                  /* release from reset */
      └─ state = RPROC_RUNNING
            ... remote boots, announces channels over the NS endpoint ...
            → rpmsg_device created → rpmsg_driver->probe()
```

### 1.4 The user-visible surface

```sh
/sys/class/remoteproc/remoteproc0/
	state            # offline|running|crashed|suspended ; write start/stop
	name
	firmware         # which file to load; writable when offline
	recovery         # enabled|disabled
	coredump         # disabled|enabled|inline
	power/

/sys/kernel/debug/remoteproc/remoteproc0/
	trace0           # the remote's log ring buffer  <-- the most useful file
	carveout_memories
	resource_table
	name
	crash            # write to inject a crash, for testing recovery

/sys/class/rpmsg/
/dev/rpmsg_ctrl0     # ioctl to create endpoints
/dev/rpmsg0          # an endpoint, read/write/poll

/sys/class/firmware/ # the fallback loader's handshake, when enabled
/sys/module/firmware_class/parameters/path   # extra search path
```

---

## 2. Practice

### Lab 49.1 — Firmware loading: the four sources

```sh
# What firmware does your system actually use?
sudo dmesg | grep -i firmware
ls /lib/firmware | head
sudo find /sys/devices -name 'firmware*' 2>/dev/null | head

# Which drivers request firmware, and did they get it?
sudo dmesg | grep -Ei 'direct firmware load|failed to load firmware|loaded firmware'
```

Now a driver that exercises every variant:

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/firmware.h>

static struct platform_device *pdev;

static void fw_cb(const struct firmware *fw, void *context)
{
	struct device *dev = context;

	if (!fw) {
		dev_warn(dev, "async: firmware not found\n");
		return;
	}
	dev_info(dev, "async: got %zu bytes, first 4: %*ph\n",
		 fw->size, 4, fw->data);
	release_firmware(fw);
}

static int fwdemo_probe(struct platform_device *p)
{
	struct device *dev = &p->dev;
	const struct firmware *fw;
	int ret;

	/* 1. Blocking, warns if missing. */
	ret = request_firmware(&fw, "fwdemo/present.bin", dev);
	dev_info(dev, "request_firmware -> %d\n", ret);
	if (!ret) {
		dev_info(dev, "  %zu bytes: %*ph\n", fw->size,
			 (int)min(fw->size, (size_t)16), fw->data);
		release_firmware(fw);
	}

	/* 2. Silent when optional and absent. */
	ret = firmware_request_nowarn(&fw, "fwdemo/optional.bin", dev);
	dev_info(dev, "firmware_request_nowarn(missing) -> %d\n", ret);
	if (!ret)
		release_firmware(fw);

	/* 3. No userspace fallback (safe under lockdown). */
	ret = request_firmware_direct(&fw, "fwdemo/present.bin", dev);
	dev_info(dev, "request_firmware_direct -> %d\n", ret);
	if (!ret)
		release_firmware(fw);

	/* 4. Asynchronous: probe does not block. */
	ret = request_firmware_nowait(THIS_MODULE, true, "fwdemo/present.bin",
				      dev, GFP_KERNEL, dev, fw_cb);
	dev_info(dev, "request_firmware_nowait -> %d (callback pending)\n", ret);

	return 0;
}

static struct platform_driver fwdemo_drv = {
	.driver.name = "fwdemo",
	.probe = fwdemo_probe,
};

static int __init m_init(void)
{
	int ret = platform_driver_register(&fwdemo_drv);

	if (ret)
		return ret;
	pdev = platform_device_register_simple("fwdemo", -1, NULL, 0);
	if (IS_ERR(pdev)) {
		platform_driver_unregister(&fwdemo_drv);
		return PTR_ERR(pdev);
	}
	return 0;
}

static void __exit m_exit(void)
{
	platform_device_unregister(pdev);
	platform_driver_unregister(&fwdemo_drv);
}

module_init(m_init);
module_exit(m_exit);
MODULE_LICENSE("GPL");
MODULE_FIRMWARE("fwdemo/present.bin");     /* records the dependency */
```

```sh
sudo mkdir -p /lib/firmware/fwdemo
echo -n "DEADBEEF-this-is-firmware" | sudo tee /lib/firmware/fwdemo/present.bin
sudo insmod fwdemo.ko
sudo dmesg | tail -20

# Alternate search path
sudo mkdir -p /opt/fw/fwdemo && echo -n ALT | sudo tee /opt/fw/fwdemo/present.bin
echo -n /opt/fw | sudo tee /sys/module/firmware_class/parameters/path
sudo rmmod fwdemo && sudo insmod fwdemo.ko && sudo dmesg | tail -5

# What does MODULE_FIRMWARE record?
modinfo fwdemo.ko | grep firmware
```

Exercises:

1. Time `request_firmware()` for a 50 MB file with `ktime_get()`. Now call it from probe of a driver that many devices depend on and measure the boot-time impact. Justify `request_firmware_nowait()`.
2. Enable the userspace fallback (`CONFIG_FW_LOADER_USER_HELPER=y` plus `CONFIG_FW_LOADER_USER_HELPER_FALLBACK`), remove the file, and watch `/sys/class/firmware/` appear. Write the three-line shell script that satisfies it (`echo 1 > loading; cat fw > data; echo 0 > loading`).
3. Demonstrate the cache: `echo mem > /sys/power/state` with `rtcwake`, and trace `request_firmware` during resume with `bpftrace -e 'kprobe:_request_firmware { printf("%s\n", str(arg1)); }'`. Confirm the filesystem is not touched.

---

### Lab 49.2 — Explore remoteproc on real or virtual hardware

If you have an SoC with a remote core (BeagleBone AI, STM32MP1, i.MX8, TI AM62, Zynq UltraScale+), use it. Otherwise, the framework itself can be explored on any machine via its virtual driver support and by reading state.

```sh
ls /sys/class/remoteproc/
for r in /sys/class/remoteproc/remoteproc*; do
  echo "=== $r"
  cat $r/name $r/state $r/firmware 2>/dev/null
done

sudo ls /sys/kernel/debug/remoteproc/remoteproc0/
sudo cat /sys/kernel/debug/remoteproc/remoteproc0/resource_table
sudo cat /sys/kernel/debug/remoteproc/remoteproc0/carveout_memories
sudo cat /sys/kernel/debug/remoteproc/remoteproc0/trace0    # remote's log!
```

Full lifecycle:

```sh
R=/sys/class/remoteproc/remoteproc0
echo stop  | sudo tee $R/state
echo my_firmware.elf | sudo tee $R/firmware      # must be in /lib/firmware
echo start | sudo tee $R/state
sudo dmesg | tail -30
sudo cat /sys/kernel/debug/remoteproc/remoteproc0/trace0
```

Inspect a real firmware image:

```sh
readelf -h /lib/firmware/my_firmware.elf         # it is just ELF
readelf -l /lib/firmware/my_firmware.elf         # PT_LOAD segments = what gets copied
readelf -S /lib/firmware/my_firmware.elf | grep resource_table
objcopy -O binary --only-section=.resource_table \
        /lib/firmware/my_firmware.elf rsc.bin && xxd rsc.bin | head
```

Decode the resource table by hand against the structs in §1.2 and compare with the debugfs `resource_table` output. This makes the self-description idea of §T.4(c) concrete.

Test crash recovery:

```sh
echo disabled | sudo tee $R/recovery      # freeze the crash for inspection
echo 1 | sudo tee /sys/kernel/debug/remoteproc/remoteproc0/crash
cat $R/state                              # crashed
sudo cp $R/coredump /tmp/rproc.core       # if coredump enabled
gdb /path/to/firmware.elf /tmp/rproc.core
echo enabled | sudo tee $R/recovery
```

---

### Lab 49.3 — Write a remoteproc driver

For a virtual target (no hardware), write a driver whose "remote" is a region of memory you inspect from the host. This teaches the framework without an SoC.

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/remoteproc.h>
#include <linux/dma-mapping.h>
#include <linux/io.h>

#define FAKE_MEM_SIZE	SZ_1M
#define FAKE_DA_BASE	0x80000000ULL

struct fake_rproc {
	struct rproc	*rproc;
	struct device	*dev;
	void		*mem;      /* CPU view */
	dma_addr_t	dma;
	bool		running;
};

/* Translate a remote device address to something the CPU can write. */
static void *fake_da_to_va(struct rproc *rproc, u64 da, size_t len,
			   bool *is_iomem)
{
	struct fake_rproc *f = rproc->priv;
	u64 off;

	if (da < FAKE_DA_BASE)
		return NULL;
	off = da - FAKE_DA_BASE;
	if (off + len > FAKE_MEM_SIZE)
		return NULL;

	if (is_iomem)
		*is_iomem = false;
	return f->mem + off;
}

static int fake_prepare(struct rproc *rproc)
{
	dev_info(rproc->dev.parent, "prepare: clocks/power would go here\n");
	return 0;
}

static int fake_start(struct rproc *rproc)
{
	struct fake_rproc *f = rproc->priv;

	f->running = true;
	dev_info(f->dev, "START: releasing remote from reset\n");
	/* Real driver: writel(BOOT_ADDR, ...); reset_control_deassert(...) */
	print_hex_dump(KERN_INFO, "loaded: ", DUMP_PREFIX_OFFSET, 16, 1,
		       f->mem, 64, true);
	return 0;
}

static int fake_stop(struct rproc *rproc)
{
	struct fake_rproc *f = rproc->priv;

	f->running = false;
	dev_info(f->dev, "STOP: asserting reset\n");
	return 0;
}

static void fake_kick(struct rproc *rproc, int vqid)
{
	dev_info(rproc->dev.parent, "KICK vq %d (would ring the mailbox)\n",
		 vqid);
}

static const struct rproc_ops fake_ops = {
	.prepare	= fake_prepare,
	.start		= fake_start,
	.stop		= fake_stop,
	.kick		= fake_kick,
	.da_to_va	= fake_da_to_va,
	.load		= rproc_elf_load_segments,
	.parse_fw	= rproc_elf_load_rsc_table,
	.find_loaded_rsc_table = rproc_elf_find_loaded_rsc_table,
	.sanity_check	= rproc_elf_sanity_check,
	.get_boot_addr	= rproc_elf_get_boot_addr,
};

static int fake_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	struct fake_rproc *f;
	struct rproc *rproc;
	int ret;

	ret = dma_set_coherent_mask(dev, DMA_BIT_MASK(32));
	if (ret)
		return ret;

	rproc = devm_rproc_alloc(dev, "fake-rproc", &fake_ops,
				 "fake_rproc_fw.elf", sizeof(*f));
	if (!rproc)
		return -ENOMEM;

	f = rproc->priv;
	f->rproc = rproc;
	f->dev = dev;

	f->mem = dmam_alloc_coherent(dev, FAKE_MEM_SIZE, &f->dma, GFP_KERNEL);
	if (!f->mem)
		return -ENOMEM;

	rproc->auto_boot = false;     /* wait for an explicit "start" */
	rproc->recovery_disabled = false;

	return devm_rproc_add(dev, rproc);
}

static struct platform_driver fake_drv = {
	.driver.name = "fake-rproc",
	.probe = fake_probe,
};

static struct platform_device *pdev;

static int __init m_init(void)
{
	int ret = platform_driver_register(&fake_drv);

	if (ret)
		return ret;
	pdev = platform_device_register_simple("fake-rproc", -1, NULL, 0);
	if (IS_ERR(pdev)) {
		platform_driver_unregister(&fake_drv);
		return PTR_ERR(pdev);
	}
	return 0;
}

static void __exit m_exit(void)
{
	platform_device_unregister(pdev);
	platform_driver_unregister(&fake_drv);
}

module_init(m_init);
module_exit(m_exit);
MODULE_LICENSE("GPL");
```

Now build a firmware image for it. A minimal ELF with a `.resource_table`:

`fw.c`:

```c
/* Compiled for the "remote", but for this lab we only need the ELF layout. */
#include <stdint.h>

struct fw_rsc_hdr { uint32_t type; };
struct fw_rsc_carveout {
	uint32_t da, pa, len, flags, reserved;
	char name[32];
};
struct fw_rsc_trace { uint32_t da, len, reserved; char name[32]; };

#define RSC_CARVEOUT 0
#define RSC_TRACE    2

struct my_rsc_table {
	uint32_t ver, num, reserved[2];
	uint32_t offset[2];
	struct fw_rsc_hdr hdr0; struct fw_rsc_carveout carveout;
	struct fw_rsc_hdr hdr1; struct fw_rsc_trace trace;
} __attribute__((packed));

__attribute__((section(".resource_table"), used))
struct my_rsc_table resource_table = {
	.ver = 1,
	.num = 2,
	.offset = {
		offsetof(struct my_rsc_table, hdr0),
		offsetof(struct my_rsc_table, hdr1),
	},
	.hdr0 = { RSC_CARVEOUT },
	.carveout = {
		.da = 0x80000000, .pa = 0xFFFFFFFF, .len = 0x10000,
		.flags = 0, .name = "text",
	},
	.hdr1 = { RSC_TRACE },
	.trace = { .da = 0x80010000, .len = 0x1000, .name = "trace0" },
};

const char payload[] __attribute__((section(".text.payload"), used)) =
	"HELLO FROM THE REMOTE PROCESSOR";
```

Link it with a script placing `.text.payload` at `0x80000000` and `.resource_table` in its own section, then:

```sh
sudo cp fw.elf /lib/firmware/fake_rproc_fw.elf
sudo insmod fake_rproc.ko
echo start | sudo tee /sys/class/remoteproc/remoteproc0/state
sudo dmesg | tail
sudo cat /sys/kernel/debug/remoteproc/remoteproc0/resource_table
```

`dmesg` will show the hex dump from `fake_start()` containing your payload — proof that the ELF was parsed, the carveout allocated, and the segment copied to the right device address. That is remoteproc in its entirety.

---

### Lab 49.4 — rpmsg from both sides

Kernel side, using the in-tree sample:

```sh
# Requires a running remoteproc with an rpmsg-capable firmware
sudo modprobe rpmsg_client_sample
sudo dmesg | grep rpmsg
```

Read `samples/rpmsg/rpmsg_client_sample.c` (110 lines) and note:
- `probe()` is called when the *remote* announces a channel — Ch. 27's binding across a processor boundary.
- The callback runs per message and echoes back until a counter expires.
- `rpmsg_send()` blocks if no buffer is available; `rpmsg_trysend()` does not.

Userspace side via `rpmsg_char`:

```c
// SPDX-License-Identifier: GPL-2.0
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/rpmsg.h>

int main(void)
{
	struct rpmsg_endpoint_info info = {
		.name = "rpmsg-client-sample",
		.src = 0xFFFFFFFF,     /* any */
		.dst = 0x400,          /* remote's advertised address */
	};
	int ctrl = open("/dev/rpmsg_ctrl0", O_RDWR);
	char buf[512];
	int ep, n;

	if (ctrl < 0) { perror("rpmsg_ctrl0"); return 1; }
	if (ioctl(ctrl, RPMSG_CREATE_EPT_IOCTL, &info)) {
		perror("CREATE_EPT"); return 1;
	}
	/* A new /dev/rpmsgN appeared. */
	ep = open("/dev/rpmsg0", O_RDWR);
	if (ep < 0) { perror("rpmsg0"); return 1; }

	write(ep, "ping", 4);
	n = read(ep, buf, sizeof(buf));
	printf("got %d bytes: %.*s\n", n, n, buf);

	ioctl(ep, RPMSG_DESTROY_EPT_IOCTL);
	close(ep); close(ctrl);
	return 0;
}
```

```sh
ls /dev/rpmsg*
sudo ./rpmsg_user
```

Exercises:

1. Trace the full path of one message: `bpftrace -e 'kprobe:rpmsg_send, kprobe:virtio_rpmsg_send, kprobe:virtqueue_kick { printf("%s\n", probe); }'`. Map it onto §T.7's layering diagram.
2. Crash the remote (`echo 1 > .../crash`) while your userspace program is blocked in `read()`. What does `read()` return? Now check whether your program handles it. That is §T.8's obligation, felt.
3. Send a message larger than 512 bytes minus the header. Observe the truncation or error, and find where in `virtio_rpmsg_bus.c` the limit is enforced.

---

### Lab 49.5 — Mailbox: write both a controller and a client

A loopback mailbox controller, so you can exercise the client API with no hardware.

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/mailbox_controller.h>
#include <linux/mailbox_client.h>
#include <linux/of.h>
#include <linux/workqueue.h>

#define NCHAN 2

struct loopmbox {
	struct mbox_controller	ctrl;
	struct mbox_chan	chans[NCHAN];
	struct work_struct	work;
	u32			pending[NCHAN];
	bool			has_pending[NCHAN];
};

/* Deliver asynchronously, as real hardware would (via an IRQ). */
static void loopmbox_work(struct work_struct *w)
{
	struct loopmbox *m = container_of(w, struct loopmbox, work);
	int i;

	for (i = 0; i < NCHAN; i++) {
		if (!m->has_pending[i])
			continue;
		m->has_pending[i] = false;
		/* RX into the SAME channel: loopback. */
		mbox_chan_received_data(&m->chans[i], &m->pending[i]);
		mbox_chan_txdone(&m->chans[i], 0);
	}
}

static int loopmbox_send(struct mbox_chan *chan, void *data)
{
	struct loopmbox *m = chan->con_priv;
	int idx = chan - m->chans;

	m->pending[idx] = *(u32 *)data;
	m->has_pending[idx] = true;
	schedule_work(&m->work);
	return 0;
}

static int loopmbox_startup(struct mbox_chan *chan)  { return 0; }
static void loopmbox_shutdown(struct mbox_chan *chan) { }

static const struct mbox_chan_ops loopmbox_ops = {
	.send_data = loopmbox_send,
	.startup   = loopmbox_startup,
	.shutdown  = loopmbox_shutdown,
};

static int loopmbox_probe(struct platform_device *pdev)
{
	struct loopmbox *m;
	int i;

	m = devm_kzalloc(&pdev->dev, sizeof(*m), GFP_KERNEL);
	if (!m)
		return -ENOMEM;

	INIT_WORK(&m->work, loopmbox_work);
	for (i = 0; i < NCHAN; i++)
		m->chans[i].con_priv = m;

	m->ctrl.dev = &pdev->dev;
	m->ctrl.ops = &loopmbox_ops;
	m->ctrl.chans = m->chans;
	m->ctrl.num_chans = NCHAN;
	m->ctrl.txdone_irq = true;

	platform_set_drvdata(pdev, m);
	return devm_mbox_controller_register(&pdev->dev, &m->ctrl);
}

static const struct of_device_id loopmbox_ids[] = {
	{ .compatible = "lab,loopmbox" }, { }
};
MODULE_DEVICE_TABLE(of, loopmbox_ids);

static struct platform_driver loopmbox_drv = {
	.driver = { .name = "loopmbox", .of_match_table = loopmbox_ids },
	.probe = loopmbox_probe,
};
module_platform_driver(loopmbox_drv);
MODULE_LICENSE("GPL");
```

Client:

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/mailbox_client.h>

struct mboxdemo {
	struct mbox_client	cl;
	struct mbox_chan	*chan;
	struct completion	rx_done;
	u32			last_rx;
};

/* NOTE: this runs in atomic/IRQ context on real hardware. */
static void demo_rx(struct mbox_client *cl, void *msg)
{
	struct mboxdemo *d = container_of(cl, struct mboxdemo, cl);

	d->last_rx = *(u32 *)msg;
	dev_info(cl->dev, "RX 0x%08x\n", d->last_rx);
	complete(&d->rx_done);
}

static void demo_tx_done(struct mbox_client *cl, void *msg, int r)
{
	/* TX accepted by hardware -- NOT processed by the remote (T.6). */
	dev_info(cl->dev, "TX done, r=%d\n", r);
}

static int demo_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	struct mboxdemo *d;
	u32 msg = 0xCAFEBABE;
	int ret;

	d = devm_kzalloc(dev, sizeof(*d), GFP_KERNEL);
	if (!d)
		return -ENOMEM;

	init_completion(&d->rx_done);
	d->cl.dev = dev;
	d->cl.tx_block = true;
	d->cl.tx_tout = 500;
	d->cl.knows_txdone = false;
	d->cl.rx_callback = demo_rx;
	d->cl.tx_done = demo_tx_done;

	d->chan = mbox_request_channel(&d->cl, 0);
	if (IS_ERR(d->chan))
		return dev_err_probe(dev, PTR_ERR(d->chan), "no mbox\n");

	platform_set_drvdata(pdev, d);

	ret = mbox_send_message(d->chan, &msg);
	dev_info(dev, "send -> %d\n", ret);

	if (!wait_for_completion_timeout(&d->rx_done, msecs_to_jiffies(1000)))
		dev_warn(dev, "no reply within 1s\n");

	return 0;
}

static void demo_remove(struct platform_device *pdev)
{
	struct mboxdemo *d = platform_get_drvdata(pdev);

	mbox_free_channel(d->chan);
}

static const struct of_device_id demo_ids[] = {
	{ .compatible = "lab,mbox-demo" }, { }
};
MODULE_DEVICE_TABLE(of, demo_ids);

static struct platform_driver demo_drv = {
	.driver = { .name = "mbox-demo", .of_match_table = demo_ids },
	.probe = demo_probe,
	.remove_new = demo_remove,
};
module_platform_driver(demo_drv);
MODULE_LICENSE("GPL");
```

DT:

```dts
	loopmbox: mailbox {
		compatible = "lab,loopmbox";
		#mbox-cells = <1>;
	};

	mboxdemo {
		compatible = "lab,mbox-demo";
		mboxes = <&loopmbox 0>;
		mbox-names = "chan0";
	};
```

Exercises:

1. Add a second client requesting the *same* channel index. Observe the `-EBUSY` and relate it to Ch. 43 §T.7's exclusive resets.
2. Set `tx_block = false` and send in a tight loop. Observe `-EBUSY` from `mbox_send_message()` when the previous TX has not completed — the framework's one-outstanding-message-per-channel rule.
3. Put a `msleep(10)` in `demo_rx()`. On this loopback controller it "works" (the callback runs in a workqueue); explain why it would be a bug on real hardware, and fix it properly with a workqueue hand-off (Ch. 18).

---

### Lab 49.6 — Debug a silent remote

The scenario: firmware loaded, `state` says `running`, nothing happens. Work the checklist.

```sh
R=/sys/class/remoteproc/remoteproc0
D=/sys/kernel/debug/remoteproc/remoteproc0

# 1. Did the ELF actually load?
sudo dmesg | grep -E 'remoteproc|rproc'
sudo cat $D/resource_table          # parsed correctly?
sudo cat $D/carveout_memories       # did the carveouts get the right addresses?

# 2. Is the remote producing any output?
sudo cat $D/trace0
sudo watch -n1 "tail -5 $D/trace0"

# 3. Did the virtio/rpmsg layer come up?
ls /sys/bus/rpmsg/devices/
sudo dmesg | grep -i 'virtio\|rpmsg'

# 4. Is the mailbox wired?
sudo cat /proc/interrupts | grep -i mbox
sudo grep . /sys/kernel/debug/mailbox/* 2>/dev/null

# 5. Is memory where the firmware thinks it is?
cat /proc/iomem | grep -i -A2 reserved
sudo cat /sys/kernel/debug/memblock/reserved | head
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A10 reserved-memory
```

The failure table, which is the real deliverable of this lab:

| Symptom | Likely cause | Confirm with |
|---|---|---|
| `start` returns `-ENOENT` | firmware file missing / wrong name | `dmesg`, `$R/firmware` |
| `start` returns `-EINVAL` | ELF sanity check failed, or resource table malformed | `readelf -l`, `$D/resource_table` |
| Loads but `trace0` empty | `da_to_va()` mapping wrong → segments written to the wrong place | hex-dump the carveout and compare with `objdump` |
| `trace0` has output, no rpmsg device | mailbox not delivering the kick, or `RSC_VDEV` misconfigured | `/proc/interrupts`, add a `pr_info` in `->kick` |
| rpmsg device appears, no data | vring addresses disagree between the two sides | dump the vring memory from both sides |
| Works once, dead after reload | `->stop` did not fully reset the remote | check reset/clock teardown |
| Works cold, fails after suspend | remote's state lost; no `->resume` restore | Ch. 48 §T.9 |
| Random corruption | carveout not in a `reserved-memory` node → Linux also allocated it | `/proc/iomem`, DT `reserved-memory` |

That last one is worth emphasising: **memory shared with a remote processor must be reserved in the device tree**, or Linux will happily allocate it to something else. The symptom is intermittent corruption that depends on boot-time allocation order — one of the nastiest bug classes in this area.

```dts
	reserved-memory {
		#address-cells = <2>;
		#size-cells = <2>;
		ranges;

		dsp_reserved: dsp@88000000 {
			reg = <0x0 0x88000000 0x0 0x1000000>;
			no-map;                /* Linux must not map it at all */
		};
	};
```

`no-map` vs `reusable` vs neither is a real distinction: `no-map` removes it from the kernel's linear map entirely (required if the remote has different cache attributes), `reusable` lets CMA use it until claimed.

---

### Lab 49.7 — SCMI: firmware-provided clocks and regulators

If your platform has SCMI (many Arm servers and SoCs, and QEMU's `virt` with the right firmware):

```sh
ls /sys/bus/scmi_protocol/devices/
sudo cat /sys/kernel/debug/scmi/*/instance_name 2>/dev/null
sudo cat /sys/kernel/debug/clk/clk_summary | grep -i scmi
ls /sys/class/hwmon/*/name | xargs grep -l scmi 2>/dev/null
```

Read `drivers/firmware/arm_scmi/clock.c` and `drivers/clk/clk-scmi.c` together. Note that `clk-scmi.c` is a completely ordinary clock provider (Ch. 43 §1.2) whose `set_rate` sends a mailbox message instead of writing a register.

The point to internalise: **every consumer driver in the system works unchanged.** A UART driver calling `clk_set_rate()` does not know whether that wrote an SoC register or sent an SCMI command over a mailbox to a separate processor. That is the abstraction from Ch. 43 §T.2 holding under a change of implementation medium — the strongest possible evidence that the boundary was drawn correctly.

Exercise: trace one `clk_set_rate()` through `clk-scmi` → `scmi_clock` → `scmi_do_xfer` → mailbox → shared memory, and count the layers. Estimate the latency and compare to a register write (Ch. 43 §T.3's "never in a hot path" rule becomes much more emphatic here).

---

## 3. Mastery drills

1. `request_firmware()` provides no integrity guarantee. Design a scheme where Linux can load firmware into a DMA-capable coprocessor on a locked-down system without being able to forge it. Identify which component must do the verification and why it cannot be Linux.

2. The firmware cache loads during `->prepare` and drops after resume. Derive the worst-case memory overhead for a system with 40 firmware-using devices, and propose a policy that bounds it without reintroducing the deadlock of §T.3.

3. The resource table is parsed from an untrusted ELF. Enumerate every field an attacker controls and, for each, the bound `rproc_handle_resources()` must enforce. Check your list against the actual code.

4. remoteproc reuses ELF. What does it gain, and name one thing a bespoke format could express that ELF cannot (hint: think about what must happen *before* any segment loads).

5. `->kick(rproc, vqid)` has no return value and no completion. Argue why this is correct for a virtqueue notification, and construct a protocol on top that does need end-to-end acknowledgement.

6. rpmsg reuses virtio, which was designed for hypervisor↔guest. Identify one assumption virtio makes that is *false* between two physical processors, and explain how rpmsg or remoteproc compensates.

7. A remote processor crashes while a userspace program is blocked in `read()` on `/dev/rpmsg0`. Specify the complete, correct behaviour: what `read()` returns, what happens to the fd, what userspace must do, and what the kernel must guarantee about buffer lifetime.

8. Compare remoteproc's crash recovery to a GPU driver's hang recovery (Ch. 47 §T.8(d)) and to a NIC's `ndo_tx_timeout` (Ch. 46). Identify the property all three share and design a generic "peer recovery" abstraction — then argue why the kernel does not have one.

9. `mbox_send_message()` with `tx_block = true` sleeps. Show that calling it from an rpmsg callback can deadlock, and identify the framework rule that prevents it.

10. Shared memory between Linux and a remote must be `reserved-memory`. Explain precisely what goes wrong with `no-map` omitted, including why the failure is intermittent and boot-order dependent.

11. SCMI implements clocks, regulators, and resets over a mailbox. Compute the latency of an SCMI `clk_set_rate()` versus a direct register write, and derive the rule this implies for where such calls may appear.

12. Design the suspend/resume story for a remoteproc device: what happens to the remote, what happens to in-flight rpmsg messages, what `->suspend` must do, and what `->restore` must assume. Relate each answer to Ch. 48.

13. A remote processor can DMA to all of system memory. Enumerate the isolation mechanisms available (IOMMU, carveouts, firmware verification, secure monitor) and rank them by the strength of guarantee. Then state what Linux can actually enforce on a typical SoC, and what it must trust.

---

## 4. Further reading

**Kernel documentation**

- `Documentation/driver-api/firmware/` ★★★ — `request_firmware.rst`, `firmware_cache.rst`, `direct-fs-lookup.rst`, `fallback-mechanisms.rst`, `lookup-order.rst`. Short, complete, and they answer §T.2 exactly.
- `Documentation/driver-api/firmware/other_interfaces.rst` — the DMI, EDD, and platform firmware interfaces.
- `Documentation/staging/remoteproc.rst` ★★★ — the remoteproc API and the resource table, by its author (Ohad Ben-Cohen).
- `Documentation/staging/rpmsg.rst` ★★★ — rpmsg's design and API.
- `Documentation/driver-api/mailbox.rst` ★★ — short; read the client/controller split.
- `Documentation/devicetree/bindings/remoteproc/`, `.../mailbox/`, `.../reserved-memory/reserved-memory.yaml` ★★★ — the last one explains `no-map` vs `reusable`, which Lab 49.6 shows is critical.
- `Documentation/firmware-guide/acpi/` (Ch. 33) and `Documentation/arm/firmware.rst`, `Documentation/arm64/booting.rst` — the other firmware interfaces of §T.9.
- `Documentation/driver-api/firmware/fw_upload.rst` — firmware *flashing* to a device, the inverse direction.

**Source worth reading**

- `drivers/base/firmware_loader/main.c` ★★★ — `_request_firmware()`; the source ordering of §T.2(a) is one readable function.
- `drivers/remoteproc/remoteproc_core.c` ★★★ — `rproc_fw_boot()`, `rproc_handle_resources()`, `rproc_trigger_recovery()`. This is the chapter.
- `drivers/remoteproc/remoteproc_elf_loader.c` — ~200 lines; ELF loading demystified.
- `drivers/remoteproc/remoteproc_virtio.c` — how `RSC_VDEV` becomes a virtio device.
- `drivers/rpmsg/virtio_rpmsg_bus.c` ★★★ — the entire rpmsg transport, including buffer management and the name service.
- `samples/rpmsg/rpmsg_client_sample.c` — 110 lines, a complete driver.
- `drivers/mailbox/mailbox.c` — the framework; `drivers/mailbox/omap-mailbox.c` and `qcom-apcs-ipc-mailbox.c` for two very different controllers.
- Real remoteproc drivers: `drivers/remoteproc/stm32_rproc.c` (clean, mailbox-based), `imx_rproc.c`, `ti_k3_r5_remoteproc.c`, `qcom_q6v5_mss.c` (the hard end: secure-world interaction).
- `drivers/firmware/arm_scmi/driver.c` + `clock.c`, and `drivers/clk/clk-scmi.c` — §T.9's abstraction demonstration.

**Specifications**

- **Arm SCMI** (System Control and Management Interface), DEN 0056 ★★★ — a well-written protocol spec and a good model for designing one.
- **Arm PSCI** (DEN 0022) and **SMCCC** (DEN 0028) — the CPU power and secure-call ABIs.
- **Arm FF-A** (DEN 0077) — the newer partition/messaging model.
- **OpenAMP** project specifications — the cross-OS standardisation of remoteproc/rpmsg concepts; the "Remoteproc and RPMsg" documents describe the same protocols from the remote's side, which is exactly what you need when writing firmware.
- **virtio** specification (OASIS) v1.2 — since rpmsg is virtio, this is the wire format.
- **ELF** specification (System V ABI) — for §T.4(a).

**Projects and ecosystem**

- **OpenAMP** (`github.com/OpenAMP/open-amp`) ★★★ — the firmware-side library implementing rpmsg/virtio for the remote. If you are writing both sides, this is what you run on the remote.
- **libmetal** — OpenAMP's hardware abstraction (shared memory, interrupts) for the remote side.
- **Zephyr RTOS** — has first-class OpenAMP support; the easiest way to build a real remote firmware image.
- `linux-firmware.git` — the canonical firmware repository; browse `WHENCE` for the licensing and provenance model, which is itself instructive.

**LWN**

- "Loading firmware from userspace" and the firmware-loader rework series (2012–2015)
- "The firmware cache and suspend" — §T.3's fix
- "Remote processor messaging" (2011) — the original announcement and rationale
- "Firmware signing and lockdown" coverage — §T.2(d)
- "SCMI and the firmware-controlled SoC" discussions
- Coverage of the `reserved-memory` binding evolution — Lab 49.6's trap

**Tools**

- `readelf`, `objdump`, `objcopy` — for firmware images (they are ELF)
- `gdb` with the firmware's `.elf` and the remoteproc `coredump`
- `/sys/kernel/debug/remoteproc/*/trace0` ★★★ — the remote's console
- `dtc -I fs /sys/firmware/devicetree/base` — verify reserved-memory
- `bpftrace` on `rproc_*`, `rpmsg_*`, `mbox_*` symbols
- `/proc/iomem`, `/sys/kernel/debug/memblock/reserved` — memory layout verification
- `cat /sys/kernel/debug/remoteproc/*/resource_table | xxd` — decode by hand against §1.2

---

→ Next: [50-driver-testing.md](50-driver-testing.md)

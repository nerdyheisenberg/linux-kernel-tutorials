# Chapter 47 — Media and display: V4L2, DRM/KMS, and GPU driver architecture

> **Goal:** Understand why graphics and media drivers are the largest and most complex in the kernel, why they abandoned the "one device node, one API" model for an *object-and-property* model, why atomic modesetting is a database transaction, why a GPU driver is really a memory manager plus a scheduler plus a security monitor with a rendering engine attached, and how `dma-buf` and `dma_fence` let three subsystems share buffers without copying. By the end you can write a working V4L2 capture driver and a complete KMS display driver, read `modetest` and `v4l2-ctl` output fluently, and reason about the buffer lifecycle across a camera → GPU → display pipeline.

> **Scale note.** `drivers/gpu/` is ~4.5 million lines — larger than the rest of the kernel's drivers combined for some metrics — and `drivers/media/` is ~800 thousand. You will not learn either exhaustively. What you *can* learn is the small set of structural ideas that organise them, which is what this chapter is.

---

## Theory & First Principles

### T.0 — Start here: 2 GB/s that must never touch the CPU

A 4K display at 60 Hz:

```
  3840 x 2160 pixels x 4 bytes x 60 Hz  =  1.99 GB/s, continuously, forever
```

And a camera feeding it, a GPU compositing, an encoder consuming. **Four devices, several
GB/s, on a phone with a 3 W budget.** A `memcpy` at 10 GB/s would consume most of the memory
bandwidth and much of the battery to move data nobody looked at.

**So the defining constraint here is the opposite of every previous chapter:**

> The data must **never** be copied, and ideally never touched by the CPU at all. The
> kernel's job is not to move bytes — it is to arrange for *devices* to move them to each
> other, and to say *when*.

```
    camera --DMA--> [ buffer ] --> GPU --> [ buffer ] --> display
                         ^                      ^
                         |                      |
                    ONE allocation, shared by fd (dma-buf, Ch. 36)
                    The CPU never reads a pixel.
```

That changes what the API *is*. A character device (Ch. 29) is about `read()`/`write()` —
moving bytes through the CPU. These subsystems are about **buffer ownership and
synchronization**, and the ioctls say so:

| Operation | What it actually means |
|---|---|
| `VIDIOC_QBUF` | "Device, this buffer is yours now" |
| `VIDIOC_DQBUF` | "Give it back; I want to look at it" |
| `PRIME_HANDLE_TO_FD` | "Export this buffer as an fd so another device can use it" |
| a **fence** | "Signal me when the GPU is done, so the display can scan it out" |

**Now the second defining problem, which is unique to display.** A mode-set involves many
interdependent pieces — planes, CRTCs, encoders, connectors, PLLs, bandwidth limits — and
**some combinations are physically impossible**:

```c
/* The old API: set each piece, one ioctl at a time. */
set_plane(0, fb_a);      /* ok */
set_plane(1, fb_b);      /* ok */
set_crtc_mode(4K_60);    /* FAILS: not enough bandwidth for both planes */
/* ...and the hardware is now in a state you never asked for, because the
   first two calls already took effect. Flicker, or a black screen. */
```

**The atomic API fixes this with validate-then-apply**, which you have now met three times
(Ch. 43 §T.0, and Ch. 89 §T.1 #3):

```c
state = drm_atomic_state_alloc(dev);
/* ... describe the ENTIRE desired configuration ... */
ret = drm_atomic_check_only(state);   /* TEST. Changes nothing. */
if (ret)
	return ret;                   /* the hardware never moved */
drm_atomic_commit(state);             /* all of it, or none of it */
```

V4L2 has the same idea in `VIDIOC_TRY_FMT`: "would this format work?" without setting it.

> **Partial failure is the enemy.** When a configuration change touches many coupled pieces,
> make "half-applied" an unreachable state by checking everything before committing anything.

**And the third idea: these are the most *userspace-heavy* subsystems in the kernel.** A
modern GPU driver does not implement OpenGL — it exposes command submission and memory
management, and Mesa in userspace compiles shaders and builds command buffers. The kernel
provides *mechanism* (submit this, isolate that, schedule fairly); userspace owns *policy*
(what to draw). That is Ch. 00 §T.1 taken further than anywhere else in the tree, and it is
why a GPU driver is ~100k lines of kernel code plus a million lines of userspace.

```bash
ls /dev/dri/                       # card0 = privileged, renderD128 = render-only
modetest -c 2>/dev/null | head -30 # connectors, modes, CRTCs (libdrm-tests)
v4l2-ctl --list-devices
v4l2-ctl -d /dev/video0 --list-formats-ext | head -20
cat /sys/kernel/debug/dri/0/state 2>/dev/null | head -20
```

---

### T.1 Why these subsystems are different: the data does not flow through the CPU

Every driver class so far moved data *to or from* the CPU: a character device copies to userspace, a NIC hands packets to the stack, an I²C sensor returns a value. Display and media are different in a way that determines everything:

> **The CPU's job is to arrange for data to move between devices, and then get out of the way.**

A camera writes frames into memory by DMA. A GPU reads those frames and writes new ones. A display controller scans out the result to a panel — 60 times a second, forever, with no interrupt per pixel. At 4K60 that is 1.5 GB/s of scanout alone. If a single byte of that traversed the CPU, the design would be wrong.

Three consequences follow, and they explain most of what looks strange about these subsystems:

1. **The primary abstraction is a buffer, not a stream.** The API is about allocating buffers, describing their format and layout, and passing *ownership* of them between components. Data never appears in a `read()` in the fast path.
2. **Time is a first-class constraint.** A frame delivered after vblank is a visible glitch. This is soft-real-time work with a hard periodic deadline supplied by hardware.
3. **The device is shared by mutually distrustful clients**, and it has a DMA engine and a programmable command processor. A GPU driver is therefore a *security boundary* with the same weight as a hypervisor's, which is why so much of the code is validation.

### T.2 V4L2: from one node with a hundred ioctls to a graph of entities

**The original model (V4L, then V4L2 in 1999).** One device node `/dev/video0`, one enormous ioctl set. This worked when a capture device was a single chip that produced frames.

It broke when hardware became a *pipeline*. A modern camera subsystem is:

```
  sensor ──► CSI-2 receiver ──► ISP ─┬─► scaler ──► /dev/video0 (full res)
                                      └─► scaler ──► /dev/video1 (preview)
```

Each block has its own format, its own cropping, and its own controls, and the *links between them* are configurable. A single node with a single `VIDIOC_S_FMT` cannot express "set the sensor to 4096×3072, crop to 3840×2160 in the ISP, and scale to 1920×1080 on this output." Worse, the pipeline's *topology* is board-specific.

**The Media Controller (MC) framework** (2010) is the answer, and it is the same structural move as the input subsystem's property bitmaps or ACPI's `_DSD` — when a flat API cannot express structure, expose the structure as data:

| Concept | Meaning |
|---|---|
| **Entity** | a processing block (sensor, ISP, scaler, DMA engine) |
| **Pad** | an input or output port on an entity |
| **Link** | a connection between two pads; may be enabled/disabled, may be immutable |
| **Interface** | a device node (`/dev/video0`, `/dev/v4l-subdev3`) associated with an entity |

`/dev/media0` exposes the graph; `MEDIA_IOC_G_TOPOLOGY` returns it; `media-ctl` prints and configures it. Each sub-device gets `/dev/v4l-subdevN` for per-block format negotiation.

Three principles this design embodies, all reusable:

- **Topology is data, discovered at runtime**, not compiled into a driver (Ch. 32 §T.1 again).
- **Format negotiation is a constraint-propagation problem across the graph.** Setting a format on one pad constrains its neighbours. `VIDIOC_SUBDEV_S_FMT` with `V4L2_SUBDEV_FORMAT_TRY` is a *query* that does not commit — the same two-phase pattern as clock `determine_rate`/`set_rate` (Ch. 43 §T.3) and, as we will see, as KMS atomic check/commit. **When a configuration change can fail partway, the kernel splits it into validate-then-apply.** That pattern appears three times in this book; it is worth naming.
- **Pipeline validation happens at stream-on.** `media_pipeline_start()` walks the enabled links and verifies that every link's two sides agree on format. Errors surface once, at a defined point, rather than as corrupted frames.

The cost is real: configuring a MC-based camera requires userspace to know the topology, which is why `libcamera` exists (§T.9).

### T.3 videobuf2: the buffer lifecycle as a state machine

`videobuf2` (vb2) is the framework that handles buffer allocation, memory mapping, queueing, and the DMA ownership dance. Every V4L2 driver should use it; writing buffer management by hand is how you get the bugs.

The core is a two-queue state machine per buffer:

```
     DEQUEUED ──VIDIOC_QBUF──► QUEUED ──driver takes it──► ACTIVE
        ▲                                                     │
        └────────VIDIOC_DQBUF────── DONE ◄──vb2_buffer_done────┘
```

and userspace's loop is:

```c
	/* setup */
	VIDIOC_REQBUFS   /* ask for N buffers of a memory type */
	VIDIOC_QUERYBUF + mmap()   /* or export as dma-buf fd */
	for (i = 0; i < N; i++) VIDIOC_QBUF(i);
	VIDIOC_STREAMON;
	/* steady state */
	while (running) {
		poll(fd);
		VIDIOC_DQBUF(&buf);    /* get a filled frame */
		process(buf);
		VIDIOC_QBUF(&buf);     /* give it back */
	}
	VIDIOC_STREAMOFF;          /* all buffers return to DEQUEUED */
```

Three design points worth extracting:

**(a) Buffers are recycled, never allocated per frame.** At 60 fps with 8 MB frames, per-frame allocation would be 480 MB/s of allocator traffic and unbounded latency jitter. The fixed-pool-plus-ownership-transfer model is the same one as the NIC descriptor ring (Ch. 46 §T.3) and for identical reasons.

**(b) Three memory models, and the choice is userspace's:**

| `V4L2_MEMORY_*` | Who allocates | Use |
|---|---|---|
| `MMAP` | kernel | simple; userspace maps the driver's buffers |
| `USERPTR` | userspace | legacy; requires pinning arbitrary user pages — problematic with IOMMUs and with `fork()` |
| `DMABUF` | a *third* subsystem | **the modern answer** — import buffers a GPU or display controller allocated (§T.7) |

`USERPTR`'s problems are instructive: pinning user memory for DMA means the pages cannot be migrated or reclaimed, `fork()` creates aliasing hazards, and the memory may not satisfy the device's contiguity or alignment needs. `DMABUF` solves all of these by making the *allocator* the component that knows the constraints.

**(c) `STREAMOFF` is a hard synchronisation point.** It must return every buffer to userspace and must not return until the hardware has stopped touching them. That is Ch. 25's P12 stop-drain-free, and `vb2_wait_for_all_buffers()` plus the driver's `stop_streaming()` implement it. A driver that returns from `stop_streaming()` with DMA still in flight produces corruption that appears in the *next* application to run.

### T.4 The DRM split: render vs display, and why they are one driver

"DRM" (Direct Rendering Manager) covers two genuinely different jobs that happen to live in the same silicon:

| | **Display (KMS)** | **Render (GPU)** |
|---|---|---|
| Job | scan a buffer out to a panel at a fixed rate | execute arbitrary programs on arbitrary buffers |
| API shape | small, standardised, fully in-kernel | large, vendor-specific, mostly in userspace |
| Failure mode | glitch, wrong colours | hang, security breach |
| Who uses it | one compositor | every 3D/compute client |
| Node | `/dev/dri/card0` (privileged) | `/dev/dri/renderD128` (unprivileged) |

The **node split** is the important design decision. Before it, any 3D client needed access to the modesetting node, which meant it could reprogram your display — and only one process could be "DRM master." Render nodes (2013) separate the two capabilities: `renderD128` can submit work and allocate buffers but cannot change a mode or scan out. That is what makes headless GPU compute, containers with GPU access, and untrusted 3D clients possible. It is a textbook application of least privilege, applied by *splitting the device node*, which is a technique worth remembering.

`DRM_MASTER` remains for KMS: exactly one process at a time may set modes, arbitrated by `DRM_IOCTL_SET_MASTER` / `DROP_MASTER`, which is how VT switching works and how a display manager hands the display to a session.

### T.5 KMS as an object model: the five object types

KMS models a display pipeline as typed objects with properties, connected by pointers. Learning these five and their relationships is 80% of understanding display:

```
    Framebuffer (a buffer + format + layout)
         │
         ▼
      Plane  ──────► CRTC ──────► Encoder ──────► Connector ──────► [monitor]
   (a source        (the         (signal        (a physical
    rectangle,       scanout      format         port: HDMI,
    scaled/          engine +     converter)     DP, eDP, DSI)
    positioned)      timing +
                     mode)
```

| Object | What it is | Key properties |
|---|---|---|
| **Connector** | a physical output port | status (connected/disconnected), EDID, supported modes, DPMS, HDR metadata, `link-status` |
| **Encoder** | converts CRTC output to a wire format | largely vestigial in modern drivers; often 1:1 with connector |
| **CRTC** | the scanout engine: reads planes, composites, generates timing | active, mode, gamma LUT, CTM, vblank counter |
| **Plane** | one source rectangle composited by the CRTC | `fb_id`, `src_x/y/w/h` (16.16 fixed point), `crtc_x/y/w/h`, `zpos`, `alpha`, `rotation`, `pixel blend mode` |
| **Framebuffer** | the memory: a set of GEM handles + format + per-plane pitch/offset + modifier | immutable once created |

Two facts that trip everyone:

- **Planes have three types**: `PRIMARY` (the main one, one per CRTC), `CURSOR` (small, often with hardware-specific constraints), `OVERLAY` (extra, often the whole point). A compositor that can push a video onto an overlay plane avoids a GPU composition pass entirely — that is real battery life on a phone.
- **Source coordinates are 16.16 fixed point.** `src_w` for a 1920-wide buffer is `1920 << 16`. This exists because scaling ratios are not integers. Forgetting the shift is the single most common first-time KMS bug.

**Format modifiers** deserve their own paragraph because they are the mechanism that makes buffer sharing work at all across vendors. A `DRM_FORMAT_XRGB8888` buffer says what the *pixels* are but nothing about how they are *arranged in memory*. GPUs use tiled and compressed layouts (`I915_FORMAT_MOD_Y_TILED_CCS`, `AMD_FMT_MOD_*`, `DRM_FORMAT_MOD_ARM_AFBC`) that are dramatically faster for the GPU and unreadable by anything that does not know the layout. A **modifier** is a 64-bit vendor-namespaced token naming the exact layout. Producers and consumers *negotiate* a modifier both understand (`IN_FORMATS` blob property on planes). Without modifiers, every cross-device buffer share would have to use linear layout and eat the bandwidth cost. With them, a GPU can render compressed and a display controller can scan out compressed.

### T.6 Atomic modesetting: a transaction over the whole display state

Before atomic (merged 4.2, Daniel Vetter et al.), modesetting was a sequence of independent ioctls: set the mode, then set the plane, then set the cursor. This had three unfixable problems:

1. **No way to ask "is this possible?"** You applied a change and found out. If step 3 failed, steps 1 and 2 had already happened and were sometimes not undoable.
2. **No way to change several things in one vblank.** Setting a plane and a cursor in separate ioctls could straddle a vblank, producing a visible tear. For a compositor updating four planes, this was hopeless.
3. **The state was in the hardware**, so drivers read it back, and read-modify-write of hardware state across concurrent ioctls is a race.

The atomic API replaces all of it with one ioctl over a **state object**:

```c
	state = drmModeAtomicAlloc();
	drmModeAtomicAddProperty(state, plane_id, prop_fb_id,   fb);
	drmModeAtomicAddProperty(state, plane_id, prop_crtc_x,  0);
	drmModeAtomicAddProperty(state, crtc_id,  prop_active,  1);
	drmModeAtomicAddProperty(state, conn_id,  prop_crtc_id, crtc);
	/* TEST_ONLY: validate, change nothing */
	ret = drmModeAtomicCommit(fd, state, DRM_MODE_ATOMIC_TEST_ONLY, NULL);
	if (!ret)
		drmModeAtomicCommit(fd, state, DRM_MODE_ATOMIC_NONBLOCK |
					       DRM_MODE_PAGE_FLIP_EVENT, user);
```

The properties are:

- **Atomic**: everything in one commit takes effect in the same vblank, or none of it does.
- **Validatable**: `TEST_ONLY` runs the full `atomic_check()` path — bandwidth limits, scaler availability, clock feasibility, plane placement rules — and returns an error without touching hardware. A compositor can therefore *probe* whether a 4-plane configuration will work and fall back to GPU composition if not, without ever showing a glitch.
- **Complete**: the state object describes the *whole* display state, not a delta, so there is no hidden hardware state to race on.

The in-kernel structure mirrors this exactly:

```c
struct drm_atomic_state {
	struct drm_plane_state     **planes;
	struct drm_crtc_state      **crtcs;
	struct drm_connector_state **connectors;
	bool allow_modeset, legacy_cursor_update, async_update;
};
```

and the driver implements a two-phase protocol:

| Phase | Callbacks | Rules |
|---|---|---|
| **Check** | `atomic_check()` on plane/crtc/connector + `mode_config.atomic_check` | **may fail; must not touch hardware; must not allocate in a way it cannot undo** |
| **Commit** | `atomic_begin`, `atomic_update`/`atomic_disable`, `atomic_flush`, `atomic_enable` | **must not fail** — all failure modes were eliminated in check |

> **"Check may fail, commit may not" is the central discipline of atomic KMS**, and it is exactly two-phase commit. Any resource a commit needs — a scaler, a memory bandwidth reservation, a PLL — must be *allocated during check* and stored in the state object. A driver that allocates in commit has a bug that manifests as a black screen under load.

This is the third appearance of validate-then-apply in this book (clocks, V4L2 subdev TRY, KMS atomic). The generalisation:

> When applying a configuration can fail *and* partial application is observable, split the operation into a pure validation phase that owns all the failure modes and a commit phase that is total.

Atomic also unified legacy ioctls: `drmModeSetCrtc` and `drmModePageFlip` are now implemented *in terms of* atomic commits by `drm_atomic_helper_*`, so drivers implement one path. That is how a 20-year-old ABI was replaced without breaking anything — the compatibility shim goes in the *kernel*, derived from the new primitive, exactly as `input_mt_report_pointer_emulation()` did in Ch. 45 §T.6.

### T.7 `dma-buf` and `dma_fence`: sharing buffers and sharing time

We met `dma-buf` in Ch. 36. Here is where it earns its keep. A camera→GPU→display pipeline involves three drivers, possibly three IOMMU domains, and a buffer that must never be copied.

**`dma-buf` shares the memory.** The exporter allocates (it knows the constraints), gets an fd, and passes it. Each importer *attaches* and gets its own `sg_table` mapped into its own address space. The key insight from Ch. 36 §T.5 bears repeating: the mapping is **per-attachment**, because the same physical pages have different device addresses in different IOMMU domains.

**`dma_fence` shares the *time*.** A buffer handed from GPU to display is not ready the instant the fd is passed — the GPU has queued work that will finish later. A `dma_fence` is a one-shot, monotonic "this work is done" object with two hard rules:

1. **A fence must signal in bounded time.** Not "eventually" — bounded. This is why GPU drivers must have hang detection and reset: a hung GPU that never signals would deadlock every consumer of that fence, including the display, forever.
2. **No allocation, no locking that can sleep indefinitely, in the signalling path.** Fence callbacks run in atomic context. The `dma_fence` lockdep annotations (`dma_fence_begin_signalling()`) exist to enforce this, and they caught a remarkable number of latent deadlocks when added.

Fences come in two flavours in the userspace ABI:

- **Implicit sync**: the fence lives in the `dma_resv` attached to the buffer. A consumer that imports the buffer automatically waits. Simple; what legacy and most V4L2/KMS paths use.
- **Explicit sync**: fences are passed as `sync_file` fds alongside the buffer (`IN_FENCE_FD` / `OUT_FENCE_PTR` properties on atomic commits). The consumer decides what to wait on. Necessary for Vulkan, for pipelining, and for avoiding false dependencies.

The combination gives you zero-copy, cross-device, correctly-ordered buffer flow with explicit lifetime — the thing this whole subsystem exists to provide.

### T.8 What a GPU driver actually is

Strip away the vendor-specific parts and a modern GPU driver is four things, none of which is "drawing":

**(a) A memory manager.** GPU memory is not system memory: it may be discrete VRAM, it may be system memory behind an IOMMU, it may need to be pinned, migrated, or evicted. **GEM** (Graphics Execution Manager) provides the handle/object model — buffer objects with refcounts, handles per-file, and `mmap` support. **TTM** (Translation Table Maps) adds placement and eviction for drivers with discrete VRAM: it is essentially a second virtual memory system with its own LRU and its own page-fault equivalent. For integrated GPUs, `drm_gem_shmem_helper` or `drm_gem_dma_helper` are far simpler and cover most SoC drivers.

**(b) A command submission path with validation.** Userspace builds a command buffer and submits it. The kernel must ensure that buffer cannot: access memory it does not own, hang the GPU indefinitely, or escape its GPU address space. Historically this was done by *parsing and validating* command streams — expensive and fragile. Modern hardware has per-context GPU page tables, so the kernel instead gives each context its own GPU virtual address space and the hardware enforces isolation. **This is the same transition CPUs made from segmentation to paging**, and it is why modern GPU drivers can have a thin submission path.

**(c) A scheduler.** Multiple contexts submit work; the driver must order it, respect dependencies (fences), enforce priorities, and preempt or reset hung work. `drivers/gpu/drm/scheduler/` (`drm_sched`) is the shared implementation: per-entity run queues, dependency resolution via fences, a timeout handler that calls the driver's reset. A GPU scheduler has the same shape as the CPU scheduler of Ch. 21 but with vastly longer context-switch costs and coarser preemption, which is why fairness guarantees are weaker.

**(d) A reset/recovery path.** GPUs hang. The driver must detect it (timeout), reset the engine, mark affected contexts' fences as errored (**signal them — rule 1 of §T.7**), and keep the rest of the system alive. `drm_sched`'s `timedout_job()` is where this lives. Getting this right is what separates a driver that annoys users from one that hangs machines.

What is *not* in the kernel: shader compilation, state tracking, the OpenGL/Vulkan API. Those live in Mesa. The kernel/userspace split is:

> **The kernel owns isolation, memory, and scheduling. Userspace owns everything about what to draw.**

This is `uAPI` minimalism as a security strategy: the smaller the ioctl surface, the smaller the attack surface, and the faster the API can evolve (Mesa ships with the OS; the kernel ABI is forever). It is also why "the GPU driver" is three components — kernel DRM driver, Mesa userspace driver, and firmware — and why version mismatches are a real support problem.

### T.9 Why you should not talk to DRM or V4L2 directly either

The same argument as Ch. 45 §T.9, one level up:

- **libdrm** wraps the ioctls, but you still need a compositor to arbitrate `DRM_MASTER`, handle hotplug, and manage the fence/flip loop.
- **libcamera** exists because configuring an MC pipeline requires per-platform knowledge (which entity is the ISP, what the sensor's 3A loop needs) that no application should carry. It is the userspace half of a modern camera driver, exactly as Mesa is the userspace half of a GPU driver.
- **Mesa** is the userspace half of every GPU driver.

The pattern across all three: **the kernel exposes a minimal, mechanism-only, stable interface; a userspace library holds the policy and the platform knowledge, and ships more often.** If you find yourself putting 3A algorithms, shader compilers, or compositing policy in the kernel, the architecture is telling you no.

---

## 1. Internals

### 1.1 Source map

| Path | Contents |
|---|---|
| `drivers/gpu/drm/drm_atomic.c`, `drm_atomic_helper.c`, `drm_atomic_uapi.c` | the atomic core and the helpers most drivers use |
| `drivers/gpu/drm/drm_crtc.c`, `drm_plane.c`, `drm_connector.c`, `drm_encoder.c`, `drm_framebuffer.c` | the five objects |
| `drivers/gpu/drm/drm_modes.c`, `drm_edid.c`, `drm_probe_helper.c` | mode lists, EDID parsing, hotplug |
| `drivers/gpu/drm/drm_gem.c`, `drm_gem_shmem_helper.c`, `drm_gem_dma_helper.c` | buffer objects |
| `drivers/gpu/drm/ttm/` | VRAM placement and eviction |
| `drivers/gpu/drm/scheduler/sched_main.c` | `drm_sched` |
| `drivers/gpu/drm/vkms/` | **virtual KMS — a complete software driver; start here** |
| `drivers/gpu/drm/tiny/` | ~15 small real drivers, each one file |
| `drivers/gpu/drm/bridge/`, `panel/` | the DRM bridge/panel chains for SoC displays |
| `drivers/dma-buf/dma-buf.c`, `dma-fence.c`, `dma-resv.c`, `sync_file.c` | sharing |
| `drivers/media/v4l2-core/v4l2-dev.c`, `v4l2-ioctl.c`, `v4l2-ctrls-*.c`, `v4l2-subdev.c` | V4L2 core |
| `drivers/media/common/videobuf2/` | vb2 |
| `drivers/media/mc/mc-*.c` | media controller |
| `drivers/media/test-drivers/vivid/`, `vimc/` | **virtual V4L2 drivers — start here** |
| `include/uapi/drm/drm_mode.h`, `drm.h` | the KMS ABI |
| `include/uapi/linux/videodev2.h`, `media.h` | the V4L2/MC ABI |

### 1.2 The KMS driver skeleton

```c
static const struct drm_driver my_drm_driver = {
	.driver_features = DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC,
	.fops            = &my_fops,          /* DEFINE_DRM_GEM_DMA_FOPS */
	.name = "mydrm", .desc = "...", .date = "20260101",
	.major = 1, .minor = 0,
	DRM_GEM_DMA_DRIVER_OPS,               /* dumb_create, prime import/export */
};

static const struct drm_mode_config_funcs my_mode_config_funcs = {
	.fb_create     = drm_gem_fb_create,
	.atomic_check  = drm_atomic_helper_check,
	.atomic_commit = drm_atomic_helper_commit,
};

/* Per-object helper vtables: */
static const struct drm_plane_helper_funcs my_plane_helper_funcs = {
	.atomic_check  = my_plane_atomic_check,   /* MAY fail */
	.atomic_update = my_plane_atomic_update,  /* MUST NOT fail */
	.atomic_disable= my_plane_atomic_disable,
};

static const struct drm_crtc_helper_funcs my_crtc_helper_funcs = {
	.mode_valid    = my_crtc_mode_valid,
	.atomic_check  = my_crtc_atomic_check,
	.atomic_enable = my_crtc_atomic_enable,
	.atomic_disable= my_crtc_atomic_disable,
	.atomic_flush  = my_crtc_atomic_flush,    /* arm the double buffer */
};
```

The `_funcs` vs `_helper_funcs` split is worth understanding: `drm_*_funcs` are the ABI-level operations the core calls (destroy, property set, state duplicate); `drm_*_helper_funcs` are the *atomic helper's* callbacks. A driver that uses `drm_atomic_helper_commit` implements the helper funcs and gets the whole commit sequence — including vblank waits, fence waits, and state swapping — for free. Almost every driver should.

### 1.3 The commit sequence, in order

```
drm_atomic_helper_commit(state)
  ├─ drm_atomic_helper_prepare_planes()       /* pin buffers, get fences */
  ├─ drm_atomic_helper_swap_state()           /* new state becomes current */
  └─ commit_tail():
       ├─ drm_atomic_helper_wait_for_fences() /* in-fences signalled */
       ├─ drm_atomic_helper_commit_modeset_disables()
       │     → crtc->atomic_disable, encoder/bridge disable
       ├─ drm_atomic_helper_commit_planes()
       │     → crtc->atomic_begin
       │     → plane->atomic_update / atomic_disable   (per plane)
       │     → crtc->atomic_flush        <-- arm the hardware double-buffer
       ├─ drm_atomic_helper_commit_modeset_enables()
       │     → crtc->atomic_enable, encoder/bridge enable
       ├─ drm_atomic_helper_wait_for_vblanks()  /* the flip took effect */
       ├─ drm_atomic_helper_commit_hw_done()    /* signal out-fences */
       └─ drm_atomic_helper_cleanup_planes()    /* unpin old buffers */
```

`atomic_flush` is the one to understand: most display hardware has shadow registers that latch at vblank. All the `atomic_update` calls write shadow registers; `atomic_flush` sets the "latch at next vblank" bit. **That is what makes the update atomic in hardware**, and it is why the software must batch.

### 1.4 The V4L2 driver skeleton

```c
struct my_dev {
	struct v4l2_device	v4l2_dev;
	struct video_device	vdev;
	struct vb2_queue	queue;
	struct mutex		lock;      /* serialises ioctls */
	struct v4l2_ctrl_handler ctrl_handler;
	struct list_head	buf_list;  /* buffers handed to hardware */
	spinlock_t		irqlock;   /* protects buf_list vs ISR */
	struct v4l2_format	fmt;
};

static const struct vb2_ops my_vb2_ops = {
	.queue_setup     = my_queue_setup,      /* how many buffers, what size */
	.buf_prepare     = my_buf_prepare,      /* validate a buffer */
	.buf_queue       = my_buf_queue,        /* hand to hardware */
	.start_streaming = my_start_streaming,
	.stop_streaming  = my_stop_streaming,   /* MUST return all buffers */
	.wait_prepare    = vb2_ops_wait_prepare,
	.wait_finish     = vb2_ops_wait_finish,
};

static const struct v4l2_ioctl_ops my_ioctl_ops = {
	.vidioc_querycap        = my_querycap,
	.vidioc_enum_fmt_vid_cap= my_enum_fmt,
	.vidioc_g_fmt_vid_cap   = my_g_fmt,
	.vidioc_s_fmt_vid_cap   = my_s_fmt,
	.vidioc_try_fmt_vid_cap = my_try_fmt,    /* validate without applying */
	.vidioc_reqbufs         = vb2_ioctl_reqbufs,
	.vidioc_querybuf        = vb2_ioctl_querybuf,
	.vidioc_qbuf            = vb2_ioctl_qbuf,
	.vidioc_dqbuf           = vb2_ioctl_dqbuf,
	.vidioc_streamon        = vb2_ioctl_streamon,
	.vidioc_streamoff       = vb2_ioctl_streamoff,
	.vidioc_enum_input      = my_enum_input,
	.vidioc_subscribe_event = v4l2_ctrl_subscribe_event,
};
```

Note how many ioctls are `vb2_ioctl_*` — vb2 implements the entire buffer ABI. The driver supplies format logic and four hardware callbacks.

`VIDIOC_TRY_FMT` vs `VIDIOC_S_FMT` is the validate/apply split again: `TRY` takes a format, adjusts it to something the hardware can do, and returns the adjusted version **without changing state**. `S_FMT` does the same and commits. A driver must implement `TRY` by clamping rather than failing — "return the nearest achievable" is the contract, identical to `clk_round_rate()`.

---

## 2. Practice

### Lab 47.1 — Explore KMS on a live system

```sh
sudo apt install libdrm-tests drm-info edid-decode   # names vary
ls -l /dev/dri/
sudo drm_info                     # everything: objects, properties, formats, modifiers
sudo modetest -M i915             # or amdgpu, vkms, virtio_gpu
sudo modetest -M i915 -p          # planes
sudo modetest -M i915 -c          # connectors, modes, EDID-derived
```

Even without hardware, load the virtual driver:

```sh
sudo modprobe vkms
sudo modetest -M vkms
sudo modetest -M vkms -s <connector_id>@<crtc_id>:1920x1080    # draws test pattern
```

Tasks:

1. From `drm_info`, list every property on your primary plane and classify each as *position*, *source*, *blending*, or *format*.
2. Find the `IN_FORMATS` blob and list the modifiers your plane supports (§T.5). Identify at least one non-linear one.
3. Find the `zpos` property and determine whether it is mutable. What does immutable `zpos` imply about the hardware?
4. Read `/sys/class/drm/card0-*/status` and `edid`; decode the EDID:

```sh
sudo cat /sys/class/drm/card0-HDMI-A-1/edid | edid-decode
```

5. Watch hotplug events:

```sh
udevadm monitor --property --subsystem-match=drm
# unplug/replug a monitor
```

---

### Lab 47.2 — A complete atomic KMS userspace client

This is the shortest program that puts a picture on screen through the modern API. It is ~200 lines and worth typing.

```c
// SPDX-License-Identifier: MIT
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/mman.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

static uint32_t find_prop(int fd, uint32_t id, uint32_t type, const char *name)
{
	drmModeObjectProperties *props =
		drmModeObjectGetProperties(fd, id, type);
	uint32_t ret = 0;

	for (uint32_t i = 0; i < props->count_props; i++) {
		drmModePropertyRes *p = drmModeGetProperty(fd, props->props[i]);

		if (!strcmp(p->name, name))
			ret = p->prop_id;
		drmModeFreeProperty(p);
	}
	drmModeFreeObjectProperties(props);
	return ret;
}

int main(int argc, char **argv)
{
	const char *path = argc > 1 ? argv[1] : "/dev/dri/card0";
	int fd = open(path, O_RDWR | O_CLOEXEC);
	drmModeRes *res;
	drmModeConnector *conn = NULL;
	drmModeModeInfo mode;
	uint32_t crtc_id = 0, conn_id = 0, plane_id = 0;

	if (fd < 0) { perror("open"); return 1; }

	/* Required before atomic properties are visible. */
	drmSetClientCap(fd, DRM_CLIENT_CAP_UNIVERSAL_PLANES, 1);
	drmSetClientCap(fd, DRM_CLIENT_CAP_ATOMIC, 1);

	res = drmModeGetResources(fd);
	for (int i = 0; i < res->count_connectors; i++) {
		conn = drmModeGetConnector(fd, res->connectors[i]);
		if (conn->connection == DRM_MODE_CONNECTED && conn->count_modes)
			break;
		drmModeFreeConnector(conn);
		conn = NULL;
	}
	if (!conn) { fprintf(stderr, "no connected connector\n"); return 1; }
	conn_id = conn->connector_id;
	mode = conn->modes[0];
	printf("using %dx%d@%d\n", mode.hdisplay, mode.vdisplay, mode.vrefresh);

	/* Any CRTC will do for a lab; real code checks possible_crtcs. */
	crtc_id = res->crtcs[0];

	{
		drmModePlaneRes *pr = drmModeGetPlaneResources(fd);

		for (uint32_t i = 0; i < pr->count_planes && !plane_id; i++) {
			drmModePlane *pl = drmModeGetPlane(fd, pr->planes[i]);
			uint32_t tp = find_prop(fd, pl->plane_id,
						DRM_MODE_OBJECT_PLANE, "type");
			drmModeObjectProperties *props =
				drmModeObjectGetProperties(fd, pl->plane_id,
							   DRM_MODE_OBJECT_PLANE);

			for (uint32_t j = 0; j < props->count_props; j++)
				if (props->props[j] == tp &&
				    props->prop_values[j] == DRM_PLANE_TYPE_PRIMARY)
					plane_id = pl->plane_id;
			drmModeFreeObjectProperties(props);
			drmModeFreePlane(pl);
		}
		drmModeFreePlaneResources(pr);
	}

	/* --- allocate a dumb buffer and fill it --- */
	struct drm_mode_create_dumb creq = {
		.width = mode.hdisplay, .height = mode.vdisplay, .bpp = 32,
	};
	drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB, &creq);

	uint32_t handles[4] = { creq.handle }, pitches[4] = { creq.pitch };
	uint32_t offsets[4] = { 0 }, fb_id;

	drmModeAddFB2(fd, mode.hdisplay, mode.vdisplay, DRM_FORMAT_XRGB8888,
		      handles, pitches, offsets, &fb_id, 0);

	struct drm_mode_map_dumb mreq = { .handle = creq.handle };
	drmIoctl(fd, DRM_IOCTL_MODE_MAP_DUMB, &mreq);
	uint32_t *pix = mmap(0, creq.size, PROT_READ | PROT_WRITE,
			     MAP_SHARED, fd, mreq.offset);

	for (uint32_t y = 0; y < creq.height; y++)
		for (uint32_t x = 0; x < creq.width; x++)
			pix[y * (creq.pitch / 4) + x] =
				(x * 255 / creq.width) << 16 |
				(y * 255 / creq.height) << 8 | 0x40;

	/* --- build the mode blob --- */
	uint32_t blob_id;

	drmModeCreatePropertyBlob(fd, &mode, sizeof(mode), &blob_id);

	/* --- one atomic commit sets up the entire pipeline --- */
	drmModeAtomicReq *req = drmModeAtomicAlloc();

#define ADD(obj, type, name, val) \
	drmModeAtomicAddProperty(req, obj, find_prop(fd, obj, type, name), val)

	ADD(conn_id,  DRM_MODE_OBJECT_CONNECTOR, "CRTC_ID", crtc_id);
	ADD(crtc_id,  DRM_MODE_OBJECT_CRTC,      "MODE_ID", blob_id);
	ADD(crtc_id,  DRM_MODE_OBJECT_CRTC,      "ACTIVE",  1);
	ADD(plane_id, DRM_MODE_OBJECT_PLANE,     "FB_ID",   fb_id);
	ADD(plane_id, DRM_MODE_OBJECT_PLANE,     "CRTC_ID", crtc_id);
	ADD(plane_id, DRM_MODE_OBJECT_PLANE,     "SRC_X",   0);
	ADD(plane_id, DRM_MODE_OBJECT_PLANE,     "SRC_Y",   0);
	/* 16.16 fixed point! (T.5) */
	ADD(plane_id, DRM_MODE_OBJECT_PLANE,     "SRC_W",   mode.hdisplay << 16);
	ADD(plane_id, DRM_MODE_OBJECT_PLANE,     "SRC_H",   mode.vdisplay << 16);
	ADD(plane_id, DRM_MODE_OBJECT_PLANE,     "CRTC_X",  0);
	ADD(plane_id, DRM_MODE_OBJECT_PLANE,     "CRTC_Y",  0);
	ADD(plane_id, DRM_MODE_OBJECT_PLANE,     "CRTC_W",  mode.hdisplay);
	ADD(plane_id, DRM_MODE_OBJECT_PLANE,     "CRTC_H",  mode.vdisplay);

	/* Phase 1: ask whether this is possible. Changes nothing. */
	int ret = drmModeAtomicCommit(fd, req,
				      DRM_MODE_ATOMIC_TEST_ONLY |
				      DRM_MODE_ATOMIC_ALLOW_MODESET, NULL);
	printf("TEST_ONLY -> %d (%s)\n", ret, ret ? strerror(-ret) : "ok");
	if (ret)
		return 1;

	/* Phase 2: apply. Cannot fail for reasons check would have caught. */
	ret = drmModeAtomicCommit(fd, req, DRM_MODE_ATOMIC_ALLOW_MODESET, NULL);
	printf("COMMIT -> %d\n", ret);

	sleep(5);

	drmModeAtomicFree(req);
	drmModeRmFB(fd, fb_id);
	close(fd);
	return 0;
}
```

```sh
gcc -o kmsdemo kmsdemo.c $(pkg-config --cflags --libs libdrm)
sudo systemctl isolate multi-user.target     # release DRM master, or use a spare VT
sudo ./kmsdemo /dev/dri/card0
```

Experiments that teach the theory:

1. Set `SRC_W` to `mode.hdisplay` **without** the `<< 16`. Observe `TEST_ONLY` fail with `-EINVAL`, and note that it failed *before* touching hardware. That is §T.6's entire value in one observation.
2. Request a `CRTC_W` larger than the mode. Observe which drivers accept it (those with scalers) and which reject it in check.
3. Add `DRM_MODE_PAGE_FLIP_EVENT`, create two buffers, and build a real double-buffered flip loop reading events from the fd. Measure the flip-to-flip interval and confirm it equals the refresh period.
4. Run two copies. The second fails to become master — observe `-EACCES` and explain it from §T.4.

---

### Lab 47.3 — Write a KMS driver: read and modify `vkms`

`vkms` (Virtual KMS) is a complete, small, well-commented KMS driver that composites in software and needs no hardware. It is the best possible starting point.

```sh
sudo modprobe vkms enable_cursor=1 enable_writeback=1
sudo modetest -M vkms
sudo drm_info -j | jq '.[] | .device'
```

Reading assignment:

1. `drivers/gpu/drm/vkms/vkms_drv.c` — the `drm_driver`, `mode_config` limits, and registration order. Map it onto §1.2.
2. `vkms_crtc.c` — `vkms_crtc_atomic_check()`, `_enable`, `_flush`. Find where vblank is emulated with an hrtimer (Ch. 19) and where `drm_crtc_handle_vblank()` is called.
3. `vkms_plane.c` — `vkms_plane_atomic_check()` uses `drm_atomic_helper_check_plane_state()`; read that helper and list every rule it enforces.
4. `vkms_composer.c` — the actual per-pixel blend. This is what real hardware does in silicon.
5. `vkms_writeback.c` — writeback connectors, which let you capture the composed output into a buffer. This is how `igt` tests KMS correctness without a camera.

Modifications to make (each is small and each teaches something):

- Add a second overlay plane and make `modetest -P` use it.
- Add a driver-private CRTC property (e.g. a "tint" value) using `drm_object_attach_property` and a custom `atomic_set_property`/`atomic_get_property` on the CRTC state. Observe it appear in `drm_info`.
- Make `vkms_plane_atomic_check()` reject scaling ratios above 2×, then verify with `modetest` that `TEST_ONLY` reports the failure.
- Introduce the classic bug deliberately: allocate memory in `atomic_update` instead of `atomic_check`, make it fail, and observe that there is no correct thing to do. Write down why in one sentence.

Then run the real test suite:

```sh
sudo apt install intel-gpu-tools      # provides igt
sudo igt_runner -t kms_ --device drm:/sys/devices/platform/vkms/drm/card*
sudo /usr/libexec/igt-gpu-tools/kms_atomic
sudo /usr/libexec/igt-gpu-tools/kms_plane
```

`igt-gpu-tools` is the KMS/DRM conformance suite. A new driver is expected to pass the generic `kms_*` tests; running them against `vkms` shows you what "correct" means operationally.

---

### Lab 47.4 — V4L2 from userspace, against a virtual camera

```sh
sudo modprobe vivid n_devs=1
v4l2-ctl --list-devices
v4l2-ctl -d /dev/video0 --all
v4l2-ctl -d /dev/video0 --list-formats-ext
v4l2-ctl -d /dev/video0 --list-ctrls-menus
```

`vivid` is a fully featured virtual video driver: capture, output, radio, SDR, metadata, with configurable formats, test patterns, and injectable errors. It is the V4L2 equivalent of `vkms`.

Capture frames:

```sh
v4l2-ctl -d /dev/video0 --set-fmt-video=width=640,height=480,pixelformat=YUYV
v4l2-ctl -d /dev/video0 --stream-mmap --stream-count=100 --stream-to=out.yuv
ffplay -f rawvideo -pixel_format yuyv422 -video_size 640x480 out.yuv
```

Now the minimal capture program, which makes the state machine of §T.3 concrete:

```c
// SPDX-License-Identifier: GPL-2.0
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <linux/videodev2.h>

#define NBUF 4

int main(int argc, char **argv)
{
	const char *dev = argc > 1 ? argv[1] : "/dev/video0";
	int fd = open(dev, O_RDWR);
	struct v4l2_capability cap;
	struct v4l2_format fmt = { .type = V4L2_BUF_TYPE_VIDEO_CAPTURE };
	struct v4l2_requestbuffers req = {
		.count = NBUF, .type = V4L2_BUF_TYPE_VIDEO_CAPTURE,
		.memory = V4L2_MEMORY_MMAP,
	};
	void *bufs[NBUF];
	size_t lens[NBUF];
	int type = V4L2_BUF_TYPE_VIDEO_CAPTURE;

	ioctl(fd, VIDIOC_QUERYCAP, &cap);
	printf("driver=%s card=%s caps=%#x\n", cap.driver, cap.card,
	       cap.device_caps);

	/* TRY first: the driver adjusts and tells us what it will do. */
	fmt.fmt.pix.width = 1280;
	fmt.fmt.pix.height = 720;
	fmt.fmt.pix.pixelformat = V4L2_PIX_FMT_YUYV;
	fmt.fmt.pix.field = V4L2_FIELD_NONE;
	ioctl(fd, VIDIOC_TRY_FMT, &fmt);
	printf("TRY  -> %ux%u fourcc=%.4s bpl=%u\n",
	       fmt.fmt.pix.width, fmt.fmt.pix.height,
	       (char *)&fmt.fmt.pix.pixelformat, fmt.fmt.pix.bytesperline);
	ioctl(fd, VIDIOC_S_FMT, &fmt);

	ioctl(fd, VIDIOC_REQBUFS, &req);
	printf("got %u buffers\n", req.count);

	for (unsigned i = 0; i < req.count; i++) {
		struct v4l2_buffer b = {
			.type = type, .memory = V4L2_MEMORY_MMAP, .index = i,
		};
		ioctl(fd, VIDIOC_QUERYBUF, &b);
		lens[i] = b.length;
		bufs[i] = mmap(NULL, b.length, PROT_READ | PROT_WRITE,
			       MAP_SHARED, fd, b.m.offset);
		ioctl(fd, VIDIOC_QBUF, &b);      /* DEQUEUED -> QUEUED */
	}

	ioctl(fd, VIDIOC_STREAMON, &type);

	for (int n = 0; n < 60; n++) {
		struct v4l2_buffer b = { .type = type,
					 .memory = V4L2_MEMORY_MMAP };

		ioctl(fd, VIDIOC_DQBUF, &b);     /* DONE -> DEQUEUED */
		printf("frame %2d idx=%u bytes=%u seq=%u ts=%ld.%06ld\n",
		       n, b.index, b.bytesused, b.sequence,
		       b.timestamp.tv_sec, b.timestamp.tv_usec);
		/* ... use bufs[b.index] ... */
		ioctl(fd, VIDIOC_QBUF, &b);      /* back to QUEUED */
	}

	ioctl(fd, VIDIOC_STREAMOFF, &type);
	close(fd);
	return 0;
}
```

Observations to make:

1. Ask for a size `vivid` cannot do (e.g. 1234×567). `TRY_FMT` returns an *adjusted* size and succeeds — it does not fail (§1.4).
2. Delete the `VIDIOC_QBUF` inside the loop. Frames stop after `NBUF`. This is the recycling model of §T.3(a), felt directly.
3. Print `b.sequence` and look for gaps — dropped frames. Then make your processing slow (`usleep(50000)`) and watch the gaps appear.
4. `v4l2-compliance -d /dev/video0 -s` — the V4L2 conformance suite, the counterpart of `igt`. Run it against `vivid`, then against your laptop's webcam, and compare the failure counts. Real hardware drivers frequently fail parts of it; that is informative.

---

### Lab 47.5 — Media Controller pipelines with `vimc`

`vimc` (Virtual Media Controller) models a full camera pipeline as separate entities, so you can practise topology configuration without hardware.

```sh
sudo modprobe vimc
media-ctl -d /dev/media0 -p                 # print the whole graph
media-ctl -d /dev/media0 --print-dot > g.dot && dot -Tpng g.dot -o g.png
```

The graph has a sensor → debayer → scaler → capture chain. Configure it end to end:

```sh
# Set the sensor's output format on its source pad
media-ctl -d /dev/media0 -V '"Sensor A":0[fmt:SBGGR8_1X8/640x480]'
# Propagate through debayer
media-ctl -d /dev/media0 -V '"Debayer A":0[fmt:SBGGR8_1X8/640x480]'
media-ctl -d /dev/media0 -V '"Debayer A":1[fmt:RGB888_1X24/640x480]'
# Scaler
media-ctl -d /dev/media0 -V '"Scaler":0[fmt:RGB888_1X24/640x480]'
media-ctl -d /dev/media0 -V '"Scaler":1[fmt:RGB888_1X24/1920x1440]'

v4l2-ctl -d /dev/video2 --set-fmt-video=width=1920,height=1440,pixelformat=RGB3
v4l2-ctl -d /dev/video2 --stream-mmap --stream-count=10
```

Now break it deliberately: set the scaler's sink format to a size that does not match the debayer's source. Then `STREAMON`:

```sh
media-ctl -d /dev/media0 -V '"Scaler":0[fmt:RGB888_1X24/320x240]'
v4l2-ctl -d /dev/video2 --stream-mmap --stream-count=1
# -> EPIPE / EINVAL from pipeline validation
```

That error comes from `media_pipeline_start()` walking the links and finding disagreement — §T.2's "validation at stream-on". Note that it is a *single* clear error at a *defined* point rather than corrupted frames, which is the entire argument for the design.

Then read `drivers/media/test-drivers/vimc/vimc-scaler.c` and find the `set_fmt` implementation with its TRY/ACTIVE `which` handling.

---

### Lab 47.6 — Cross-subsystem: dma-buf from V4L2 to DRM

The payoff lab: capture a frame with no copy and scan it out.

```sh
sudo modprobe vivid
sudo modprobe vkms
```

Sketch (the full program is ~300 lines; build it incrementally):

```c
	/* 1. Allocate on the DRM side — the display has the strictest
	 *    constraints, so it should be the exporter. */
	struct drm_mode_create_dumb creq = { .width = W, .height = H, .bpp = 32 };
	drmIoctl(drm_fd, DRM_IOCTL_MODE_CREATE_DUMB, &creq);

	int dmabuf_fd;
	drmPrimeHandleToFD(drm_fd, creq.handle, DRM_CLOEXEC | DRM_RDWR,
			   &dmabuf_fd);

	/* 2. Import it into V4L2 as a capture target. */
	struct v4l2_requestbuffers req = {
		.count = 1, .type = V4L2_BUF_TYPE_VIDEO_CAPTURE,
		.memory = V4L2_MEMORY_DMABUF,
	};
	ioctl(v4l_fd, VIDIOC_REQBUFS, &req);

	struct v4l2_buffer b = {
		.type = V4L2_BUF_TYPE_VIDEO_CAPTURE,
		.memory = V4L2_MEMORY_DMABUF,
		.index = 0,
		.m.fd = dmabuf_fd,       /* <-- the shared buffer */
	};
	ioctl(v4l_fd, VIDIOC_QBUF, &b);
	ioctl(v4l_fd, VIDIOC_STREAMON, &type);

	/* 3. Wait for capture, then scan out the SAME memory. */
	ioctl(v4l_fd, VIDIOC_DQBUF, &b);

	uint32_t handles[4] = { creq.handle }, pitches[4] = { creq.pitch };
	uint32_t offsets[4] = { 0 }, fb;
	drmModeAddFB2(drm_fd, W, H, DRM_FORMAT_XRGB8888,
		      handles, pitches, offsets, &fb, 0);
	/* ... atomic commit with FB_ID = fb, as in Lab 47.2 ... */
```

Verify no copy happened:

```sh
sudo cat /sys/kernel/debug/dma_buf/bufinfo     # exporter, size, attachments
```

You should see one buffer with **two** attachments. That single line is the whole point of Ch. 36 and §T.7.

Extensions:

- Add explicit fencing: get an `OUT_FENCE_PTR` from the atomic commit and pass it as `V4L2_BUF_FLAG_IN_FENCE` so capture waits for scanout to finish with the previous buffer.
- Try a format the display supports and the camera does not. Observe where the failure surfaces and argue where it *should*.
- Repeat with a tiled modifier and watch it fail, then explain why using §T.5.

---

### Lab 47.7 — Debugging display and media

```sh
# DRM: the most useful debug switches
echo 0x1e | sudo tee /sys/module/drm/parameters/debug   # CORE|DRIVER|KMS|PRIME
sudo dmesg -w

# Per-driver state dumps
sudo cat /sys/kernel/debug/dri/0/state          # the full atomic state, human-readable
sudo cat /sys/kernel/debug/dri/0/framebuffer
sudo cat /sys/kernel/debug/dri/0/clients
sudo cat /sys/kernel/debug/dri/0/gem_names      # or i915_gem_objects etc.

# Tracing
sudo trace-cmd record -e drm -e drm_msm_atomic -e dma_fence
sudo perf record -e drm:*  -a sleep 5

# V4L2
echo 3 | sudo tee /sys/module/videobuf2_common/parameters/debug
echo 3 | sudo tee /sys/class/video4linux/video0/dev_debug
sudo dmesg -w      # every ioctl with arguments, decoded
v4l2-compliance -d /dev/video0 -s -a

# dma-buf / fences
sudo cat /sys/kernel/debug/dma_buf/bufinfo
sudo cat /sys/kernel/debug/sync/info            # if CONFIG_SW_SYNC
```

`/sys/kernel/debug/dri/0/state` is remarkable and underused: it prints the *entire* current atomic state — every plane, its buffer, its coordinates, every CRTC's mode — in one readable dump. When a compositor shows a black screen, this file tells you in five seconds whether the problem is "no framebuffer attached", "CRTC inactive", "plane positioned off-screen", or "mode not set".

`dev_debug` on a V4L2 node is the equivalent: it logs every ioctl with decoded arguments, which turns "the app says EINVAL" into "it passed field=INTERLACED to a progressive-only driver."

Build a decision table from these tools:

| Symptom | First command | What it distinguishes |
|---|---|---|
| Black screen | `cat /sys/kernel/debug/dri/0/state` | inactive CRTC vs missing FB vs off-screen plane |
| Tearing | check for atomic + `PAGE_FLIP_EVENT` use | compositor not using flips |
| Flip timeout / stall | `cat /sys/kernel/debug/dma_buf/bufinfo`, `dmesg` for fence timeouts | a producer's fence never signalled (§T.7 rule 1) |
| `-EINVAL` on commit | `drm.debug=0x1e` | which `atomic_check` rejected it and why |
| No frames | `dev_debug=3` on the video node | ioctl sequence error vs hardware |
| Frame drops | `b.sequence` gaps + `vb2` debug | consumer too slow vs driver bug |

---

## 3. Mastery drills

1. Atomic KMS requires that `atomic_check` own every failure mode. Enumerate five resources a display controller might run out of during a commit, and for each describe how it must be reserved in check and stored in the state object.

2. `drm_atomic_helper_swap_state()` makes the new state current *before* the hardware is programmed. Explain why this ordering is correct, and what invariant it relies on regarding who may read `obj->state`.

3. A `dma_fence` must signal in bounded time. Construct a three-device pipeline (camera → GPU → display) and show how a single driver violating this rule deadlocks the other two. Then describe what `dma_fence`'s lockdep annotations detect and what they cannot.

4. Implicit sync puts the fence in the buffer's `dma_resv`; explicit sync passes it separately. Give a concrete pipelining scenario where implicit sync creates a false dependency that costs a frame, and show how explicit sync avoids it.

5. Format modifiers are 64-bit vendor-namespaced tokens with no semantics the kernel understands. Argue why the kernel deliberately does not interpret them, and what would break if it tried to convert between them.

6. V4L2's `TRY_FMT` must adjust rather than fail. Prove that "adjust to nearest achievable" plus "S_FMT applies exactly what TRY returned" gives userspace a terminating negotiation algorithm, and identify the driver bug that would make it non-terminating.

7. `vb2`'s `stop_streaming()` must return all buffers and guarantee DMA has ceased. Write the teardown for a driver whose hardware has no "abort DMA" command, only "stop after current frame." What is the worst-case latency, and how should the driver bound it?

8. Render nodes separate rendering from modesetting. Enumerate exactly which ioctls must be blocked on `renderD128` for the separation to hold, and identify one that is subtle (hint: think about what a buffer export could leak).

9. Modern GPUs give each context its own page tables instead of validating command streams. State the security property this provides, then name three things the kernel must *still* validate, and why.

10. `drm_sched` must handle a hung job. Write the full recovery sequence including: detecting, resetting, what happens to the hung context's fences, what happens to *other* contexts' in-flight work, and how userspace learns about it.

11. Compare the buffer-recycling designs of NIC RX rings (Ch. 46 §T.3), vb2, and GEM/TTM. Identify the property all three share and the one dimension on which they genuinely differ.

12. A compositor wants to put a video on an overlay plane to avoid GPU composition. List everything that could make `atomic_check` reject it, and design the fallback strategy that guarantees the compositor never misses a frame while probing.

13. The kernel/userspace split puts shader compilation in Mesa and isolation in DRM. Apply the same reasoning to the camera stack: draw the line for libcamera, and defend the placement of auto-exposure, lens shading correction, and sensor register programming.

---

## 4. Further reading

**Kernel documentation**

- `Documentation/gpu/drm-kms.rst` ★★★ — the object model of §T.5, authoritative and readable.
- `Documentation/gpu/drm-kms-helpers.rst` ★★★ — the atomic helper sequence of §1.3, step by step.
- `Documentation/gpu/drm-mm.rst` ★★★ — GEM, TTM, `drm_gem_*_helper`, the memory model of §T.8(a).
- `Documentation/gpu/drm-uapi.rst` ★★★ — render nodes, `DRM_MASTER`, the uAPI stability rules. Read the "Open-Source Userspace Requirements" section: **a new DRM uAPI will not be merged without an open-source userspace consumer.** That policy is itself a major architectural decision worth understanding.
- `Documentation/gpu/todo.rst` ★★ — the maintainers' own list of what is wrong; an excellent map of the subsystem's real structure.
- `Documentation/gpu/vkms.rst`, `drm-internals.rst`, `introduction.rst`
- `Documentation/driver-api/media/` ★★★ — the whole V4L2/MC architecture; `v4l2-core.rst`, `mc-core.rst`, `v4l2-subdev.rst`.
- `Documentation/userspace-api/media/v4l/` ★★★ — the **V4L2 API specification**. This is a complete, precise, book-length document. It is the single best piece of userspace-API documentation in the kernel tree; use it as a model for your own.
- `Documentation/driver-api/dma-buf.rst` ★★★ — `dma-buf`, `dma_fence`, `dma_resv`, and the "indefinite fences are forbidden" argument in the maintainers' own words.

**Source worth reading**

- `drivers/gpu/drm/vkms/` ★★★ — ~2500 lines, a complete KMS driver. Read all of it.
- `drivers/gpu/drm/tiny/*.c` ★★★ — `simpledrm.c`, `st7586.c`, `repaper.c`: real drivers in 300–700 lines each.
- `drivers/gpu/drm/drm_atomic_helper.c` ★★★ — `drm_atomic_helper_commit()` and `commit_tail()`; the sequence of §1.3 is here and commented.
- `drivers/gpu/drm/drm_simple_kms_helper.c` — for hardware with one plane, one CRTC, one connector; shows how much the helpers can absorb.
- `drivers/media/test-drivers/vivid/` ★★★ and `vimc/` ★★★ — complete virtual V4L2 and MC drivers.
- `drivers/media/platform/` — pick one SoC camera driver (e.g. `rockchip/rkisp1`) and trace one frame through it.
- `drivers/dma-buf/dma-fence.c` — short; read `dma_fence_signal()` and the lockdep annotations.

**Books and long-form**

- Daniel Vetter's blog (`blog.ffwll.ch`) ★★★ — "Atomic Modesetting Design Overview" (parts 1 and 2, also on LWN) is the definitive explanation of §T.6, by its author. Also "Botching up ioctls" — required reading for anyone designing a uAPI, and applicable far beyond graphics.
- Laurent Pinchart's talks on the Media Controller and libcamera — the clearest statements of §T.2 and §T.9.
- *Linux Media Infrastructure API* (the kernel's own `userspace-api/media` doc, built as a book).

**Papers and specs**

- VESA **E-EDID** and **DisplayID** specifications — what `edid-decode` is decoding.
- VESA **DisplayPort** and HDMI specifications (for link training, which is why `link-status` exists as a connector property).
- MIPI **DSI** and **CSI-2** specifications — the SoC display and camera interfaces.
- J. Owens et al., "GPU Computing," *Proceedings of the IEEE*, 2008 — background on the hardware model.
- "Dandelion"/"PTask"-era OS-GPU integration papers (SOSP'11, SOSP'13) for the scheduling and isolation arguments of §T.8.

**LWN**

- "Atomic mode setting design overview" (2015, two parts) ★★★
- "The rocky road to DRM render nodes" (2013) — §T.4's history
- "DMA buffer sharing" series and "Indefinite DMA fences" (2020) — the bounded-time rule argued
- "libcamera: a library for complex cameras" (2019)
- "Format modifiers" coverage — why linear-only sharing was untenable
- The annual X.Org/Wayland/graphics roundups

**Tools**

- `modetest`, `drm_info`, `kmscube`, `proptest`, `libdrm` tests
- `igt-gpu-tools` ★★★ — the DRM/KMS conformance and stress suite; the standard for "is this driver correct"
- `v4l2-ctl`, `v4l2-compliance` ★★★, `media-ctl`, `qv4l2`/`qvidcap`, `yavta`
- `edid-decode`, `read-edid`
- `weston`, `weston-info`, `wayland-info`, `kmscon` — minimal compositors for testing
- `cam` (libcamera's test tool), `libcamera-hello`
- `/sys/kernel/debug/dri/*/state` ★★★ and `/sys/kernel/debug/dma_buf/bufinfo`

---

→ Next: [48-driver-power.md](48-driver-power.md)

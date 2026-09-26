# Chapter 28 — Managed Resources: `devres`, `devm_*`, and Teardown Ordering

> **Goal:** understand `devm_` as a real memory-management discipline — not a convenience —
> know precisely when it is *wrong*, and be able to reason about teardown ordering without
> guessing.

---

## Theory & First Principles

### T.0 — Start here: count the bugs in this probe function

This is what a driver probe looks like without managed resources. It is also, almost
verbatim, what thousands of drivers looked like before 2007.

```c
static int my_probe(struct platform_device *pdev)
{
	struct my_dev *d;
	int ret;

	d = kzalloc(sizeof(*d), GFP_KERNEL);
	if (!d)
		return -ENOMEM;

	d->clk = clk_get(&pdev->dev, NULL);
	if (IS_ERR(d->clk)) { ret = PTR_ERR(d->clk); goto err_free; }

	ret = clk_prepare_enable(d->clk);
	if (ret) goto err_clk_put;

	d->base = ioremap(res->start, resource_size(res));
	if (!d->base) { ret = -ENOMEM; goto err_clk_disable; }

	d->irq = platform_get_irq(pdev, 0);
	if (d->irq < 0) { ret = d->irq; goto err_unmap; }

	ret = request_irq(d->irq, my_isr, 0, "mydev", d);
	if (ret) goto err_unmap;

	ret = misc_register(&d->miscdev);
	if (ret) goto err_free_irq;

	platform_set_drvdata(pdev, d);
	return 0;

err_free_irq:   free_irq(d->irq, d);
err_unmap:      iounmap(d->base);
err_clk_disable: clk_disable_unprepare(d->clk);
err_clk_put:    clk_put(d->clk);
err_free:       kfree(d);
	return ret;
}
```

**Count the lines that exist only to undo things: 5 labels, 5 statements — plus the whole
`remove()` function, which must mirror this exactly, in reverse.** That is roughly 40% of the
function, and it does nothing on the success path.

**Now find the bugs**, because there are always bugs here:

| Bug | Frequency in real code |
|---|---|
| `goto err_unmap` from the `platform_get_irq` failure — correct. From `request_irq` — should be `err_unmap` too, but people write `err_free_irq`, freeing an IRQ they never requested | extremely common |
| Add a sixth resource later, forget to add a label — the new resource leaks on every later failure | the classic |
| `remove()` drifts out of sync with `probe()` after three years of edits | near-universal |
| Labels in the wrong order, so resources are released out of order | subtle and nasty |

**The structural observation:** the unwinding is *completely determined* by what was acquired
and in what order. It contains no information. It is pure bookkeeping that a human is asked
to maintain by hand, forever, in two places that must stay in sync.

**`devm_` deletes all of it:**

```c
static int my_probe(struct platform_device *pdev)
{
	struct my_dev *d = devm_kzalloc(&pdev->dev, sizeof(*d), GFP_KERNEL);
	if (!d)
		return -ENOMEM;

	d->clk = devm_clk_get_enabled(&pdev->dev, NULL);
	if (IS_ERR(d->clk))
		return dev_err_probe(&pdev->dev, PTR_ERR(d->clk), "clock\n");

	d->base = devm_platform_ioremap_resource(pdev, 0);
	if (IS_ERR(d->base))
		return PTR_ERR(d->base);

	d->irq = platform_get_irq(pdev, 0);
	if (d->irq < 0)
		return d->irq;

	ret = devm_request_irq(&pdev->dev, d->irq, my_isr, 0, "mydev", d);
	if (ret)
		return ret;

	platform_set_drvdata(pdev, d);
	return devm_misc_register(&pdev->dev, &d->miscdev);
}
/* There is no remove(). There are no labels. There is nothing to get wrong. */
```

**The mechanism is simple and worth stating precisely:** each `devm_` call appends a release
action to a list on the `struct device`. When probe returns an error, or when the device is
unbound, the core walks that list **in reverse** and runs each action. It is RAII, implemented
with a linked list, in C.

> **When a correctness obligation is routinely forgotten, move it into the primitive.**

That is the principle (Ch. 89 §T.1 #4), and `devm_` is its clearest instance. You will meet
it again in folios (Ch. 52), in `mmiowb()` folding into `spin_unlock` (Ch. 34), in `guard()`
(Ch. 05), and — as an entire language feature rather than a convention — in Rust's `Drop`
(Ch. 82 §T.2).

**And the trap that §T.5 exists for:** `devm_` ties the lifetime to the *device*. If an object
can outlive the device — because userspace holds an open file descriptor to it — `devm_` frees
it too early, and you have converted a leak into a use-after-free. That is the char-device
lifetime problem (Ch. 29 §T.6), and it is the one place where reaching for `devm_` is wrong.

---

### T.1 The problem: error paths are where bugs live

A realistic `probe()` acquires eight to fifteen resources. Written by hand with the Ch. 05 T.6
goto ladder:

```c
static int my_probe(struct platform_device *pdev)
{
	struct my_priv *p;
	int ret;

	p = kzalloc(sizeof(*p), GFP_KERNEL);
	if (!p)
		return -ENOMEM;

	p->base = ioremap(res->start, resource_size(res));
	if (!p->base) { ret = -ENOMEM; goto err_free; }

	p->clk = clk_get(dev, NULL);
	if (IS_ERR(p->clk)) { ret = PTR_ERR(p->clk); goto err_unmap; }

	ret = clk_prepare_enable(p->clk);
	if (ret) goto err_clk_put;

	p->vdd = regulator_get(dev, "vdd");
	if (IS_ERR(p->vdd)) { ret = PTR_ERR(p->vdd); goto err_clk_disable; }

	ret = regulator_enable(p->vdd);
	if (ret) goto err_reg_put;

	ret = request_irq(irq, my_isr, 0, "mydev", p);
	if (ret) goto err_reg_disable;

	ret = misc_register(&p->miscdev);
	if (ret) goto err_free_irq;

	return 0;

err_free_irq:     free_irq(irq, p);
err_reg_disable:  regulator_disable(p->vdd);
err_reg_put:      regulator_put(p->vdd);
err_clk_disable:  clk_disable_unprepare(p->clk);
err_clk_put:      clk_put(p->clk);
err_unmap:        iounmap(p->base);
err_free:         kfree(p);
	return ret;
}
```

**Count the failure modes.** With *n* resources there are *n* error paths, each with a
distinct unwind sequence, and each must be exactly the reverse of what was done. Then
`remove()` must duplicate the whole teardown *again*. Empirically:

- Labels get misnamed or reordered during refactoring, so the ladder unwinds wrongly.
- Adding a resource in the middle requires touching every label below it.
- `remove()` drifts out of sync with `probe()` as the driver evolves.
- The `-EPROBE_DEFER` path (Ch. 27 T.4) runs the *entire* ladder, repeatedly — any leak there
  is multiplied by the retry count.

Empirical studies of Linux bugs (Palix et al., *"Faults in Linux: Ten Years Later"*,
ASPLOS 2011; Saha et al. on error-handling code) consistently find that **error-handling paths
have a bug density several times higher than main paths**, and that resource-release errors
are among the most common categories. The reason is structural: error paths are rarely
executed, so they are rarely tested (Ch. 06 T.6's fault-injection argument).

### T.2 The solution: region-based memory management

`devres` applies a technique with a name in the PL literature: **region-based resource
management** (Tofte & Talpin, 1994; also called *arena* or *zone* allocation).

> Associate every acquired resource with a **region** (here: the device's bind lifetime).
> Release the entire region at once, automatically, in reverse acquisition order.

The kernel keeps a per-device list:

```c
struct device {
	...
	struct list_head devres_head;     /* the region */
	spinlock_t       devres_lock;
};

struct devres_node {
	struct list_head  entry;
	dr_release_t      release;        /* how to free THIS resource */
	const char       *name;
	size_t            size;
};

struct devres {
	struct devres_node  node;
	u8                  data[];       /* the resource itself, or a pointer to it */
};
```

`devm_kmalloc()` allocates `sizeof(struct devres) + size` and puts the node on the list.
`devm_ioremap()` allocates a node holding the mapping and a release callback. At unbind,
`devres_release_all()` walks the list **backwards**, calling each `release()`.

The probe above becomes:

```c
static int my_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	struct my_priv *p;
	int ret;

	p = devm_kzalloc(dev, sizeof(*p), GFP_KERNEL);
	if (!p)
		return -ENOMEM;

	p->base = devm_platform_ioremap_resource(pdev, 0);
	if (IS_ERR(p->base))
		return PTR_ERR(p->base);

	p->clk = devm_clk_get_enabled(dev, NULL);
	if (IS_ERR(p->clk))
		return dev_err_probe(dev, PTR_ERR(p->clk), "clk\n");

	p->vdd = devm_regulator_get_enable(dev, "vdd");
	if (IS_ERR(p->vdd))
		return dev_err_probe(dev, PTR_ERR(p->vdd), "vdd\n");

	ret = devm_request_irq(dev, irq, my_isr, 0, "mydev", p);
	if (ret)
		return dev_err_probe(dev, ret, "irq\n");

	return devm_misc_register(dev, &p->miscdev);
}
/* remove() may not be needed at all. */
```

**Zero error labels. Zero teardown code. Zero possibility of unwinding in the wrong order.**
The `-EPROBE_DEFER` path is automatically correct, which matters enormously given Ch. 27 T.4.

This is the same idea as RAII in C++, `defer` in Go, `Drop` in Rust, and `with` in Python —
*binding cleanup to a scope so it cannot be forgotten.* The kernel just implements the scope
explicitly because C has no destructors. (Ch. 05 T.6's `guard()`/`__free()` is the
*function-scope* version of the same idea; `devres` is the *device-lifetime* version.)

### T.3 The region is "bound to this driver", not "this device exists"

This is the single most important and most misunderstood fact about `devm_`.

```
device_add()  ─────────────────────────────────────────────▶  device_del()
                    │                                   │
                    ├── really_probe()                  │
                    │     devres_open_group()  ◀── region STARTS
                    │     probe()
                    │       devm_* allocations
                    │   ...device operating...
                    │     remove()
                    │     devres_release_all() ◀── region ENDS
                    └── device_release_driver()
```

The region is the **driver binding lifetime**, which is strictly shorter than the device's
existence and *completely unrelated* to how long userspace holds a reference.

**Therefore `devm_` is correct if and only if:** the resource is used *only* by the bound
driver, and *nothing outside the driver can reference it after unbind*.

The classic fatal violation (Ch. 12 Lab 12.E):

```c
/* ★ BROKEN */
static int my_probe(struct platform_device *pdev)
{
	struct my_priv *p = devm_kzalloc(&pdev->dev, sizeof(*p), GFP_KERNEL);

	p->miscdev.fops = &my_fops;
	misc_register(&p->miscdev);          /* userspace can now open() it */
	return 0;
}
/* User: open("/dev/mydev"); then admin: echo unbind > .../unbind
 * → devres frees p → the open fd's private_data dangles → UAF on the next read() */
```

**The rule to memorize:**

> *If a userspace-visible object (a char device, a netdev, a block device, an input device,
> a DRM device, an IIO device) outlives unbind, its backing memory must be refcounted —
> not `devm_`.*

Subsystems solve this with their own two-stage allocation + refcount:
`alloc_netdev()`/`free_netdev()`, `cdev_add()`+`kobject` refcounting, `devm_drm_dev_alloc()`
(which uses `drm_dev_get/put` internally), `devm_iio_device_alloc()` similarly. When a
subsystem offers a `devm_*_alloc()` that returns a refcounted object, it is doing the
composition for you — but you must still not stash `devm_` memory *inside* it.

### T.4 Ordering: reverse acquisition, and its interaction with non-`devm_` code

`devres_release_all()` releases in **reverse order of acquisition**, which is exactly what
the goto ladder did by hand, for the same reason: later resources may depend on earlier ones.

The trap appears when you **mix** managed and unmanaged resources:

```c
/* ★ BROKEN ordering */
static int my_probe(...)
{
	p->base = devm_ioremap_resource(dev, res);     /* devres: released LAST */
	p->wq   = alloc_workqueue("mydev", 0, 0);      /* manual: released in remove() */
	devm_request_irq(dev, irq, isr, ...);          /* devres: released FIRST */
	return 0;
}

static void my_remove(struct platform_device *pdev)
{
	destroy_workqueue(p->wq);      /* runs BEFORE devres releases the IRQ! */
	/* ...but the IRQ is still armed, and its handler queues work on p->wq */
}
```

The order at unbind is: **`remove()` first, then `devres_release_all()`**. So anything you
free manually in `remove()` is freed *before* every `devm_` resource. If a `devm_`-managed
IRQ handler touches a manually-freed workqueue — UAF.

**Two correct strategies:**

1. **Use `devm_` for everything.** `devm_alloc_workqueue()` does not exist, but
   `devm_add_action_or_reset()` makes any cleanup managed (T.5). This is the preferred
   modern style.
2. **Quiesce explicitly at the top of `remove()`.** Stop the hardware and free the IRQ
   *manually* before tearing anything else down — i.e. apply Ch. 25 P12 by hand:
   ```c
   static void my_remove(struct platform_device *pdev)
   {
	struct my_priv *p = platform_get_drvdata(pdev);

	my_hw_disable_irq(p);                 /* 1. STOP */
	devm_free_irq(&pdev->dev, p->irq, p); /*    (explicitly, now) */
	cancel_work_sync(&p->work);           /* 2. DRAIN */
	destroy_workqueue(p->wq);             /* 3. FREE */
   }
   ```

`devm_free_irq()`, `devm_kfree()`, `devm_iounmap()` etc. exist precisely to let you release a
managed resource *early*, out of order, when you need to control the sequence.

### T.5 `devm_add_action_or_reset()`: making anything managed

The general escape hatch, and the most useful `devres` API to know:

```c
static void my_cleanup(void *data)
{
	struct my_priv *p = data;

	destroy_workqueue(p->wq);
}

static int my_probe(struct platform_device *pdev)
{
	struct my_priv *p = devm_kzalloc(...);

	p->wq = alloc_workqueue("mydev-%s", WQ_MEM_RECLAIM, 0, dev_name(dev));
	if (!p->wq)
		return -ENOMEM;

	/* ★ now the workqueue participates in devres ordering */
	return devm_add_action_or_reset(&pdev->dev, my_cleanup, p);
}
```

**Why `_or_reset` rather than plain `devm_add_action()`:** if registering the action itself
fails (out of memory), `devm_add_action_or_reset()` **calls the action immediately** and
returns the error. Plain `devm_add_action()` would leak. There is essentially never a reason
to use the non-`_or_reset` form; treat its appearance in review as a bug.

Related helpers worth knowing:

```c
devm_add_action_or_reset(dev, fn, data);     /* run fn(data) at unbind */
devm_kfree(dev, ptr);                        /* release early */
devm_kstrdup(dev, s, GFP_KERNEL);
devm_kmemdup(dev, p, n, GFP_KERNEL);
devm_kcalloc(dev, n, size, GFP_KERNEL);      /* ★ overflow-safe (Ch. 08 T.6) */
devm_krealloc(dev, p, size, GFP_KERNEL);
devm_bitmap_zalloc(dev, nbits, GFP_KERNEL);
devm_get_free_pages(dev, gfp, order);
```

And the group API, for when you need a *nested* region:

```c
void *grp = devres_open_group(dev, NULL, GFP_KERNEL);
...allocate several things...
if (failed)
	devres_release_group(dev, grp);      /* release just this group */
else
	devres_close_group(dev, grp);
```
This is what `really_probe()` itself uses, and what subsystems use when one logical operation
acquires several resources that must be released together.

### T.6 The full `devm_` inventory

Essentially every acquisition API has a managed twin. Knowing the list is most of what makes
a modern probe short.

| Domain | Managed API |
|---|---|
| Memory | `devm_kzalloc`, `devm_kcalloc`, `devm_kstrdup`, `devm_kmemdup`, `devm_krealloc` |
| MMIO | `devm_ioremap`, `devm_ioremap_resource`, `devm_platform_ioremap_resource`, `devm_platform_ioremap_resource_byname`, `devm_ioport_map` |
| IRQ | `devm_request_irq`, `devm_request_threaded_irq`, `devm_irq_alloc_descs` |
| Clocks | `devm_clk_get`, `devm_clk_get_enabled` ★, `devm_clk_get_optional_enabled`, `devm_clk_bulk_get_all_enabled` |
| Regulators | `devm_regulator_get`, `devm_regulator_get_enable` ★, `devm_regulator_bulk_get_enable` |
| Resets | `devm_reset_control_get`, `devm_reset_control_get_exclusive_deasserted` |
| GPIO | `devm_gpiod_get`, `devm_gpiod_get_array`, `devm_gpiochip_add_data` |
| Pinctrl | `devm_pinctrl_get`, `devm_pinctrl_register` |
| PHY | `devm_phy_get`, `devm_of_phy_get` |
| DMA | `dmam_alloc_coherent`, `dmam_pool_create`, `devm_request_dma` |
| PCI | `pcim_enable_device` ★, `pcim_iomap_regions`, `pcim_iomap_region`, `pcim_request_all_regions` |
| regmap | `devm_regmap_init_i2c/spi/mmio/...` |
| Subsystems | `devm_mfd_add_devices`, `devm_iio_device_register`, `devm_hwmon_device_register_with_info`, `devm_led_classdev_register`, `devm_input_allocate_device`, `devm_watchdog_register_device`, `devm_thermal_of_zone_register`, `devm_rtc_allocate_device`, `devm_nvmem_register`, `devm_drm_dev_alloc`, `devm_spi_register_controller`, `devm_i2c_add_adapter`, `devm_snd_soc_register_component` |
| Misc | `devm_device_add_group`, `devm_add_action_or_reset`, `devm_delayed_work_autocancel`, `devm_mutex_init` |

The `_enabled` / `_get_enable` variants (5.x/6.x additions) are a second-order improvement:
they combine *acquire* and *enable* into one managed call, removing the
`clk_get` → `clk_prepare_enable` → error-path-`clk_disable` pattern entirely. Prefer them.

```bash
git grep -c 'devm_' -- drivers/ | head
git grep -n 'devm_clk_get_enabled' -- drivers/ | wc -l    # the conversion in progress
git log --oneline --grep='devm_clk_get_enabled' | head
```

### T.7 When `devm_` is wrong — the complete list

Reviewers check for these. Learn them as a checklist:

| Situation | Why `devm_` fails | Use instead |
|---|---|---|
| Object exposed to userspace that survives unbind (chardev, netdev, block, input, DRM, IIO) | fd outlives binding (T.3) | refcount (`kref`, subsystem `*_get/put`) |
| Memory shared with another driver/subsystem after unbind | consumer holds a dangling pointer | refcount, or an explicit handoff protocol |
| Resource whose release order must interleave with manual cleanup | `remove()` runs before `devres` (T.4) | `devm_add_action_or_reset`, or release explicitly |
| Allocation on a hot path (per-request, per-packet) | `devres` adds a node + list op + spinlock per alloc | plain `kmalloc`, a slab cache, or a mempool |
| Allocation in atomic/interrupt context | `devres` takes a spinlock and allocates a node | preallocate at probe |
| Very large or variable-lifetime buffers | ties memory to binding, not to use | explicit alloc/free |
| Something needed *before* a device exists | no device to hang it on | module-level init |
| DMA buffers with hardware still running | hardware may DMA after the free | quiesce first, then release |

The hot-path point deserves emphasis because it is measurable: `devm_kmalloc()` is
**strictly more expensive** than `kmalloc()` — it allocates a larger block (header + data),
takes `dev->devres_lock`, and does a list insertion. For probe-time allocation that is
irrelevant; for per-I/O allocation it is a real regression.

### T.8 The deeper lesson: lifetime must be *designed*, not defaulted

`devm_` is not "the easy way to allocate". It is a **declaration that this resource's lifetime
is exactly the driver binding**. Using it means you have decided that.

The Ch. 12 T.10 decision procedure applies unchanged, and `devm_` is simply the
implementation of its second branch:

```
Does the object's lifetime end inside one function?     → guard()/__free()  (Ch. 05)
Does it die exactly when the DRIVER UNBINDS?            → devm_*            (this chapter)
Does it die when the DEVICE is destroyed?               → dev->release() / device refcount
Can more than one subsystem hold a pointer?             → kref / refcount_t (Ch. 12)
Do lockless readers look it up?                         → + RCU             (Ch. 15)
```

Writing which branch you chose, and why, in the driver's header comment (Ch. 25 §3) is what
separates a reviewable driver from an unreviewable one.

---

## 1. Internals

### 1.1 How it actually works

```c
/* drivers/base/devres.c */
void *devres_alloc_node(dr_release_t release, size_t size, gfp_t gfp, int nid)
{
	struct devres *dr = kmalloc_node_track_caller(
				tot_size(size), gfp, nid);

	if (unlikely(!dr))
		return NULL;
	memset(dr, 0, offsetof(struct devres, data));
	INIT_LIST_HEAD(&dr->node.entry);
	dr->node.release = release;
	return &dr->data;                 /* ★ the caller sees only the payload */
}

void devres_add(struct device *dev, void *res)
{
	struct devres *dr = container_of(res, struct devres, data);
	unsigned long flags;

	spin_lock_irqsave(&dev->devres_lock, flags);
	list_add_tail(&dr->node.entry, &dev->devres_head);
	spin_unlock_irqrestore(&dev->devres_lock, flags);
}

int devres_release_all(struct device *dev)
{
	/* walks dev->devres_head from the TAIL: reverse acquisition order */
}
```

The `container_of` trick is the same as Ch. 08 T.3: the caller receives a pointer to the
payload and never sees the bookkeeping header — so `devm_kzalloc()` is a drop-in replacement
for `kzalloc()` at every call site.

`devm_ioremap()` and friends store the *resource handle* in `data` and set a `release`
callback that knows how to undo it. `devres_find()`/`devres_destroy()` locate a node by
`(release_fn, match_fn, match_data)` — which is how `devm_kfree()` and `devm_free_irq()` find
and release a specific entry early.

### 1.2 Source map

```
drivers/base/devres.c       ★★ the whole implementation, ~1200 lines, very readable
include/linux/device.h      devm_kzalloc, devm_add_action_or_reset, devres_* prototypes
lib/devres.c                devm_ioremap and friends
drivers/pci/devres.c        pcim_* (the PCI managed API)
kernel/irq/devres.c         devm_request_irq
drivers/clk/clk-devres.c    devm_clk_get_enabled
drivers/regulator/devres.c
drivers/gpio/gpiolib-devres.c
Documentation/driver-api/driver-model/devres.rst ★★ — **the complete API list**
```

`Documentation/driver-api/driver-model/devres.rst` is an exhaustive, maintained index of
every `devm_` function grouped by subsystem. **Keep it open while writing a probe.**

---

## 2. Practice

### Lab 28.1 — Convert a goto ladder and diff the result

```bash
# Find a driver still using the manual style
git grep -l 'goto err_' -- drivers/ | xargs grep -l 'request_irq\|ioremap\|clk_get' | head -10
```
Pick one. Convert it fully to `devm_`. Then measure what you removed:

```bash
git diff --stat
# Verify equivalence:
make drivers/foo/bar.o && objdump -d drivers/foo/bar.o > /tmp/after.s
# and check the error paths still work:
./scripts/config -e FAULT_INJECTION -e FAILSLAB -e FAULT_INJECTION_DEBUG_FS
```
**Then inject failures to actually exercise the paths** (Ch. 06 Lab 6.8) — this is the step
everyone skips, and it is where the real bugs are:

```bash
cd /sys/kernel/debug/failslab
echo 20 | sudo tee probability; echo 1 | sudo tee task-filter
echo 1 | sudo tee /proc/self/make-it-fail
sudo modprobe mydrv        # probe fails at random points
# With devm_, EVERY failure point must leak nothing:
echo scan | sudo tee /sys/kernel/debug/kmemleak; sleep 6
echo scan | sudo tee /sys/kernel/debug/kmemleak
sudo cat /sys/kernel/debug/kmemleak
```

### Lab 28.2 — Observe the devres list and its ordering

```c
// SPDX-License-Identifier: GPL-2.0
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/slab.h>

static void action_a(void *d) { pr_info("release A (%s)\n", (char *)d); }
static void action_b(void *d) { pr_info("release B (%s)\n", (char *)d); }
static void action_c(void *d) { pr_info("release C (%s)\n", (char *)d); }

static int demo_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	void *m1, *m2;

	pr_info("--- acquiring in order: m1, A, m2, B, C\n");
	m1 = devm_kzalloc(dev, 64, GFP_KERNEL);
	devm_add_action_or_reset(dev, action_a, "first");
	m2 = devm_kzalloc(dev, 128, GFP_KERNEL);
	devm_add_action_or_reset(dev, action_b, "second");
	devm_add_action_or_reset(dev, action_c, "third");
	pr_info("m1=%p m2=%p\n", m1, m2);
	return 0;
}

static void demo_remove(struct platform_device *pdev)
{
	pr_info("--- remove() runs FIRST, before any devres release\n");
}

static struct platform_driver demo_driver = {
	.driver = { .name = "devresdemo" },
	.probe  = demo_probe,
	.remove = demo_remove,
};

static struct platform_device *pdev;
static int __init d_init(void)
{
	int ret = platform_driver_register(&demo_driver);

	if (ret)
		return ret;
	pdev = platform_device_register_simple("devresdemo", -1, NULL, 0);
	return PTR_ERR_OR_ZERO(pdev);
}
static void __exit d_exit(void)
{
	platform_device_unregister(pdev);
	platform_driver_unregister(&demo_driver);
}
module_init(d_init); module_exit(d_exit);
MODULE_LICENSE("GPL");
```
```bash
sudo insmod devresdemo.ko && sudo rmmod devresdemo && dmesg | tail -12
```
Expected output proves both facts from T.4:
```
--- acquiring in order: m1, A, m2, B, C
--- remove() runs FIRST, before any devres release     ← remove() precedes devres
release C (third)                                       ← reverse order
release B (second)
release A (first)
```

### Lab 28.3 — Reproduce the userspace-outlives-unbind UAF (T.3)

```c
/* ★ BROKEN version */
struct bad_priv {
	struct miscdevice miscdev;
	char              buf[64];
};

static ssize_t bad_read(struct file *f, char __user *ub, size_t n, loff_t *o)
{
	struct bad_priv *p = container_of(f->private_data, struct bad_priv, miscdev);

	return simple_read_from_buffer(ub, n, o, p->buf, sizeof(p->buf));
}
static const struct file_operations bad_fops = {
	.owner = THIS_MODULE, .read = bad_read, .llseek = default_llseek,
};

static int bad_probe(struct platform_device *pdev)
{
	struct bad_priv *p = devm_kzalloc(&pdev->dev, sizeof(*p), GFP_KERNEL);  /* ★ WRONG */

	strscpy(p->buf, "hello from a doomed allocation", sizeof(p->buf));
	p->miscdev.minor = MISC_DYNAMIC_MINOR;
	p->miscdev.name  = "baddev";
	p->miscdev.fops  = &bad_fops;
	platform_set_drvdata(pdev, p);
	return misc_register(&p->miscdev);
}
static void bad_remove(struct platform_device *pdev)
{
	misc_deregister(&platform_get_drvdata(pdev)->miscdev);
	/* devres then frees p — but an open fd still points at it */
}
```
```bash
./scripts/config -e KASAN
sudo insmod baddev.ko
exec 3< /dev/baddev              # hold it open
sudo rmmod baddev                # devres frees p
head -c 16 /dev/fd/3             # → KASAN slab-use-after-free
exec 3<&-
dmesg | grep -A25 KASAN
```
**Now fix it** with a `kref` (Ch. 12 Lab 12.E) and re-run: the `read()` must either succeed
(if you keep the object alive) or return `-ENODEV` (if you mark it dead), but never crash.

### Lab 28.4 — Reproduce the mixed-ordering UAF (T.4)

```c
static int mix_probe(struct platform_device *pdev)
{
	struct mix_priv *p = devm_kzalloc(&pdev->dev, sizeof(*p), GFP_KERNEL);

	p->wq = alloc_workqueue("mixdemo", 0, 0);       /* MANUAL */
	INIT_WORK(&p->work, mix_work);
	platform_set_drvdata(pdev, p);

	/* devres-managed IRQ whose handler queues onto the manual workqueue */
	return devm_request_irq(&pdev->dev, p->irq, mix_isr, IRQF_SHARED, "mix", p);
}

static void mix_remove(struct platform_device *pdev)
{
	struct mix_priv *p = platform_get_drvdata(pdev);

	destroy_workqueue(p->wq);      /* ★ IRQ is still registered and armed! */
}
```
Trigger interrupts during unbind and watch KASAN fire. Then fix it two ways — (a) all-devres
with `devm_add_action_or_reset`, (b) explicit `devm_free_irq()` at the top of `remove()` —
and verify both.

### Lab 28.5 — Measure the hot-path cost (T.7)

```c
#define N 200000
static int bench_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	u64 t0;
	int i;
	void **p = vmalloc(N * sizeof(void *));

	t0 = ktime_get_ns();
	for (i = 0; i < N; i++) p[i] = kmalloc(64, GFP_KERNEL);
	pr_info("kmalloc     : %llu ns/op\n", (ktime_get_ns() - t0) / N);
	for (i = 0; i < N; i++) kfree(p[i]);

	t0 = ktime_get_ns();
	for (i = 0; i < N; i++) p[i] = devm_kmalloc(dev, 64, GFP_KERNEL);
	pr_info("devm_kmalloc: %llu ns/op\n", (ktime_get_ns() - t0) / N);
	t0 = ktime_get_ns();
	for (i = 0; i < N; i++) devm_kfree(dev, p[i]);
	pr_info("devm_kfree  : %llu ns/op  ← note the LIST SEARCH\n",
		(ktime_get_ns() - t0) / N);

	vfree(p);
	return 0;
}
```
**`devm_kfree()` is O(n) in the devres list length** — it searches. With 200k entries this is
catastrophic. That single measurement is the whole T.7 hot-path argument, and it also explains
why you should not use `devm_` for anything you intend to free early in bulk.

### Lab 28.6 — Audit a subsystem for `devm_` misuse

```bash
# Drivers that devm_kzalloc a struct containing a userspace-visible object:
git grep -l 'devm_kzalloc' -- drivers/ | \
  xargs grep -l 'misc_register\|cdev_add\|register_netdev\|input_register_device' | head -20
```
For each hit, determine whether the subsystem's registration takes its own reference
(`input_allocate_device` does; `misc_register` does **not**). Several real bugs live here.

```bash
# The non-_or_reset form (always suspicious):
git grep -n 'devm_add_action(' -- drivers/ | head

# Drivers doing manual cleanup in remove() alongside devm_ (T.4 risk):
git grep -l 'devm_request_irq' -- drivers/ | xargs grep -l 'destroy_workqueue\|kthread_stop' | head

# The conversion opportunities:
git grep -n 'clk_prepare_enable' -- drivers/ | xargs -I{} echo {} | head -20
# → many of these can become devm_clk_get_enabled()
```

### Lab 28.7 — Read `devres.c`

```bash
$EDITOR drivers/base/devres.c
# Answer in writing:
#  1. How does devm_kzalloc() return a pointer the caller can use directly?
#  2. How does devm_kfree() find the right node? What is its complexity?
#  3. What does devres_open_group()/devres_release_group() do, and who calls it?
#  4. Where exactly in really_probe() is the group opened and released? (drivers/base/dd.c)
#  5. What happens to devres if probe() returns -EPROBE_DEFER? Trace it.
grep -n 'devres_open_group\|devres_release_group\|devres_release_all' drivers/base/dd.c
```

---

## 3. Mastery drills

1. **Read `Documentation/driver-api/driver-model/devres.rst`** completely and bookmark it.
   Count how many managed APIs exist. Pick five you did not know and find a user of each.

2. **The region-management connection.** Read about region/arena allocation (Tofte & Talpin,
   or any compiler-textbook treatment). Explain `devres` in those terms: what is the region,
   what is the allocation, what guarantees does the region discipline provide, and what does
   it *not* provide (hint: it does not prevent dangling *references*, only leaks).

3. **Write the decision procedure** from T.8 as a flowchart, and apply it to every allocation
   in a driver you read. Find one you believe is wrong and justify.

4. **Error-path bug study.** Read Palix et al., *"Faults in Linux: Ten Years Later"*
   (ASPLOS 2011). Which fault categories does `devm_` eliminate entirely? Which does it not
   touch? Then find a post-2011 CVE in an error path and classify it.

5. **`pcim_` vs `pci_`.** Read `drivers/pci/devres.c`. Explain how `pcim_enable_device()`
   changes the behaviour of *subsequent* `pci_*` calls (a genuinely surprising design), and
   why the API was reworked in 6.9+. What is `pcim_request_all_regions()` for?

6. **Ordering proof.** Construct a driver with five resources where the *only* correct
   teardown order is the reverse acquisition order, and where any other order causes a
   crash. Then verify `devres` achieves it.

7. **The `remove()`-runs-first gotcha.** Find three in-tree drivers that free something
   manually in `remove()` while holding `devm_` resources. For each, determine whether there
   is a window where a managed resource's callback could touch the freed object.

8. **Design a `devm_` API.** Your subsystem has a `foo_register()`/`foo_unregister()` pair.
   Write `devm_foo_register()` correctly, including the `_or_reset` semantics and what
   happens if the caller also calls `foo_unregister()` manually.

9. **Rust preview.** Explain how Rust's `Devres<T>` / `Device` bound lifetimes express T.3's
   invariant in the type system. What does the compiler catch that a C reviewer must catch by
   hand? (Part 5, Ch. 83.)

10. **Write the lifetime section** of a driver's model comment (Ch. 25 §3), covering: which
    allocations are `devm_`, which are refcounted, why each choice, and what the teardown
    order is. Do it for a driver you did not write.

---

## 4. Further reading

**Kernel documentation:**
- `Documentation/driver-api/driver-model/devres.rst` ★★ — **the complete API index**
- `Documentation/driver-api/driver-model/binding.rst` — where the region opens and closes
- `Documentation/process/coding-style.rst` §7 (centralized exiting) — the ladder `devm_`
  replaces
- `Documentation/dev-tools/kmemleak.rst`, `fault-injection/` — how to *verify* your paths

**Source:**
- `drivers/base/devres.c` ★★ — read it completely; it is short and excellent
- `drivers/base/dd.c` — `really_probe()`'s group open/release
- `include/linux/cleanup.h` — the function-scope sibling (`guard()`, `__free()`)
- `drivers/pci/devres.c` — the most subtle managed API in the tree

**Papers & background:**
- Tofte & Talpin, "Implementation of the Typed Call-by-Value λ-calculus using a Stack of
  Regions" (POPL 1994) — region-based memory management
- Palix, Thomas, Saha, Calvès, Lawall, Muller, "Faults in Linux: Ten Years Later"
  (ASPLOS 2011) ★ — the empirical case for T.1
- Saha, Lawall, Muller, "An Approach to Improving the Structure of Error-Handling Code in
  the Linux Kernel" (LCTES 2011) — Coccinelle-based analysis of exactly this problem
- Stroustrup on RAII; the Rust `Drop` documentation — the same idea in typed languages

**LWN:**
- "Managed device resources" (the original devres introduction, 2007)
- "devm_kmalloc() and the lifetime of device data"
- "A new API for pcim_* resources" (6.9 rework)
- "Cleaning up with guard()" — the scope-level complement

→ Next: [29-char-devices.md](29-char-devices.md)

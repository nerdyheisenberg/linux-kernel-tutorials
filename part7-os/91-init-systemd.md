# Chapter 91 — initramfs, Init Systems, and systemd Internals

> `kernel_init` has just `execve`'d PID 1. Everything from here is userspace, and it is where
> a surprising share of production problems live. This chapter covers the initramfs (what it
> is for and why it exists), the init problem in general, and systemd specifically — because
> whatever you think of it, it is what you will operate, and its internals are genuinely
> interesting systems design.

---

## Theory & First Principles

### T.0 — Start here: the driver you need is on the disk you cannot mount

```
   The kernel wants to mount /dev/nvme0n1p2 as root.
   To do that it needs the NVMe driver.
   The NVMe driver is a module at /lib/modules/.../nvme.ko
   ...which is ON /dev/nvme0n1p2.
```

**A perfect circular dependency.** You have exactly two ways out:

1. **Compile everything into the kernel.** Works — and produces a 200 MB kernel containing
   every storage, RAID, and filesystem driver in existence, because a distribution kernel must
   boot on *any* machine. Also: no `cryptsetup`, no `lvm`, no interactive passphrase prompt,
   because those are userspace programs and there is no userspace yet.
2. **Carry a tiny root filesystem in memory**, mount *that* first, and let it do whatever is
   needed to make the real root available.

**Option 2 is the initramfs**, and once you see it as the answer to a circular dependency, all
of its odd properties make sense:

```
  bootloader loads:  vmlinuz  +  initramfs.img  (a cpio archive, usually compressed)
        |
        v
  kernel unpacks the cpio into a tmpfs, and it becomes /
        |
        v
  /init runs (PID 1!) and does whatever is needed:
        load nvme.ko, mdadm --assemble, lvm vgchange -ay,
        cryptsetup luksOpen (ASK FOR A PASSPHRASE -- a real userspace!),
        then mount the real root
        |
        v
  switch_root /newroot /sbin/init   -- replaces / and EXECs the real init,
                                       keeping PID 1. The initramfs is freed.
```

**The key idea worth extracting is that the initramfs makes *arbitrary policy* available at a
point where previously only *compiled-in mechanism* was.** Unlocking an encrypted disk
requires a prompt, a keyfile lookup, maybe a TPM interaction, maybe a network call. None of
that belongs in the kernel. **So: create a minimal userspace early, and let userspace decide.**
Ch. 00 §T.1 again — and it is why `initrd`'s successor is so much more flexible.

**Now the second half of the chapter: PID 1 is not an ordinary process.**

| Property | Consequence |
|---|---|
| It is the **reaper of orphans** | every orphaned process is reparented to it; if it does not `wait()`, zombies accumulate forever |
| It **cannot be killed** by default signals | the kernel ignores fatal signals to PID 1 that lack a handler |
| If it exits, the kernel **panics** | `Attempted to kill init!` |
| It is the **root of every namespace hierarchy** | a container's PID 1 has the same obligations (Ch. 93) |

**That last row is why "my container does not stop on `docker stop`" is such a common bug:**
people run an application as PID 1 that does not handle `SIGTERM` and does not reap children.
**PID 1 is a job with specific obligations, and most programs do not fulfil them.**

**And the design argument worth being able to make both sides of.** SysV init was a set of
shell scripts run in a fixed, numbered order. systemd replaced it with a **declarative
dependency graph**:

| SysV init | systemd |
|---|---|
| imperative shell, fixed order | declarative units, computed order |
| serial — slow boot | parallel where dependencies allow |
| "is it running?" = a PID file, often stale | cgroup membership — authoritative (Ch. 92) |
| logging, sockets, timers, mounts = 5 tools | one model: socket activation, timers, mounts as units |
| small, simple, auditable | large; a single project owning a great deal of the system |

**The technically decisive argument is the cgroup one.** A PID file lies: the process may have
died, forked, or been replaced. **cgroup membership cannot lie** — it is enforced by the
kernel, survives double-forking, and lets you kill an entire service including every child it
spawned. **Service management became reliable when it started using a kernel-enforced
grouping instead of a userspace convention** — which is, once more, "move the obligation into
the primitive."

```bash
lsinitramfs /boot/initrd.img-$(uname -r) | head -30
systemctl list-dependencies default.target | head -30
systemd-analyze plot > boot.svg
systemctl status sshd          # note the cgroup and the process tree it shows
cat /proc/1/cgroup
```

---

### T.1 — Why an initramfs exists at all

A chicken-and-egg problem: to mount the root filesystem you need the driver for the storage
controller, the filesystem driver, possibly a decryption key, possibly to assemble a RAID or
LVM volume. Building all of that into the kernel means a kernel that must know about every
possible storage stack.

The initramfs breaks the cycle:

```
 kernel unpacks a cpio archive into a tmpfs, rooted at /
   │
   ├─ loads the modules it needs (storage controller, fs, crypto)
   ├─ assembles md/LVM, unlocks LUKS, waits for the device
   ├─ mounts the real root at /root (or /sysroot)
   │
   └─ switch_root: pivot to the real root, exec the real init
        └─ the tmpfs is freed
```

**The key insight: it is a `tmpfs`, not a block device.** The older `initrd` was a ramdisk —
a block device with a filesystem image on it, requiring a filesystem driver and double
memory (the image plus the page cache). `initramfs` is a cpio archive unpacked directly into
the page cache. Simpler, smaller, and the memory is reclaimed on `switch_root`.

**Two flavours:**

| Kind | Contents | Size | Use |
|---|---|---|---|
| **Generic / "host-only=no"** | Every plausible driver | 30–80 MB | Distributions; must boot any hardware |
| **Host-only** | Only this machine's drivers | 5–15 MB | Tuned servers, embedded |
| **Embedded / built-in** | A `CONFIG_INITRAMFS_SOURCE` archive linked into the kernel | 1–5 MB | Products with fixed hardware; no separate file to load or sign |

And the fourth case: **no initramfs at all.** If the storage driver and filesystem are built
in and the root device is a fixed path, you do not need one. Many embedded products do this
and save 200–500 ms of boot time.

### T.2 — What PID 1 must do

Four jobs, and every init system is a different answer to them:

| Job | Detail |
|---|---|
| **Start the system** | Mount filesystems, start services, in the right order |
| **Supervise** | Restart what dies, know what is running |
| **Reap orphans** | PID 1 inherits every orphaned process and **must `wait()`** or the process table fills with zombies |
| **Shut down** | Stop services in reverse order, unmount, sync, power off |

And two properties it must have:
- **It must never exit.** PID 1 exiting is a kernel panic (`Attempted to kill init!`).
- **It gets no default signal handlers.** The kernel does not apply default actions to PID 1
  for signals it has not handled — so a bug that would kill any other process leaves PID 1
  running but wedged.

### T.3 — The init systems, and the design argument

| System | Model | Where you meet it |
|---|---|---|
| **SysV init** | Sequential shell scripts, runlevels, `/etc/rc?.d/S??name` | Legacy, some embedded |
| **BusyBox init** | Minimal, `/etc/inittab`, ~10 KB | Embedded; the default in Buildroot |
| **OpenRC** | Dependency-based shell scripts | Gentoo, Alpine |
| **runit / s6** | Tiny supervision trees, one process per service | Void, minimal containers; excellent design |
| **Upstart** | Event-driven | Historical (Ubuntu 2006–2014) |
| **systemd** | Declarative units, socket/bus activation, cgroups | Every major distribution |

**The argument for systemd, stated fairly** — because the debate is usually conducted badly:

1. **Parallelism through socket activation.** The classic init serializes because service B
   needs service A's socket. systemd creates *all* the sockets first, then starts everything
   in parallel; a client connecting to a not-yet-started service simply blocks on the socket
   buffer. This is the single biggest boot-time improvement and it is a genuinely clever
   idea, borrowed from macOS's launchd.

2. **cgroups for process tracking.** SysV tracked processes with PID files, which are racy,
   and double-forking daemons could escape supervision entirely. systemd puts each service in
   its own cgroup, so **every** descendant is tracked and killing a service actually kills it
   (Ch. 93).

3. **Declarative dependencies** instead of numbered scripts. The ordering is data, not code,
   so it can be analysed (`systemd-analyze critical-chain`).

4. **Uniform service management**: logging, resource limits, sandboxing, restart policy, and
   watchdogs are unit-file options rather than per-service shell code.

**The argument against, also stated fairly:**

1. **Scope.** systemd has absorbed logging, DNS resolution, NTP, network configuration, device
   management, containers, boot loading, and home directories. Each is defensible; the
   aggregate is a large, tightly-coupled dependency.
2. **Complexity.** ~1.5M lines of code in PID 1's project, versus BusyBox init's ~1000.
3. **Binary logs.** `journald`'s format requires its tools; a corrupted journal is much harder
   to recover than a truncated text log.
4. **Portability.** Linux-only by design, and deeply tied to cgroups and kernel interfaces.
5. **Embedded footprint.** Too large for many constrained products.

**The architect's position:** systemd on anything with a general-purpose userspace;
BusyBox init or s6 on constrained embedded; and *know both*, because you will meet both.

### T.4 — systemd's model: units

Everything is a **unit**, and the type suffix determines the semantics.

| Type | Manages | Example |
|---|---|---|
| `.service` | A process | `sshd.service` |
| `.socket` | A listening socket; activates a service on connection | `sshd.socket` |
| `.target` | A synchronization point (the runlevel replacement) | `multi-user.target` |
| `.mount` | A mount point; generated from `/etc/fstab` | `home.mount` |
| `.automount` | On-demand mounting | `proc-sys-fs-binfmt_misc.automount` |
| `.device` | A device, from udev | `dev-sda1.device` |
| `.timer` | A cron replacement | `logrotate.timer` |
| `.path` | Activates on filesystem changes | `cups.path` |
| `.slice` | A cgroup hierarchy node for resource control | `system.slice` |
| `.scope` | Externally-created processes, grouped | `session-1.scope` |

**The dependency vocabulary, and the distinction people get wrong:**

| Directive | Means |
|---|---|
| `Requires=` | If that fails, this fails. **No ordering implied** |
| `Wants=` | Try to start it; continue regardless. **No ordering implied** |
| `BindsTo=` | Stronger than Requires: if that stops, this stops |
| `Requisite=` | That must already be running; do not start it |
| `Conflicts=` | Cannot run together |
| `Before=` / `After=` | **Ordering only. No dependency implied** |
| `PartOf=` | Stopping/restarting that propagates to this |

> **`Requires=` and `After=` are orthogonal.** `Requires=foo.service` without
> `After=foo.service` starts both *simultaneously* and hopes. This is the most common unit
> file bug, and it produces intermittent startup failures that look like race conditions —
> because they are.

### T.5 — Socket activation, in detail

The mechanism that makes parallel startup work.

```
 1. systemd creates and listens on ALL sockets, before any service starts.
        sshd.socket   -> listen(:22)
        docker.socket -> listen(/run/docker.sock)

 2. Everything starts in parallel. A client connects to :22.

 3. The kernel queues the connection in the socket's backlog.
    systemd notices readability and starts sshd.service, passing the
    already-listening fd as fd 3 (SD_LISTEN_FDS_START).

 4. sshd calls sd_listen_fds(), finds its socket already bound and
    listening, and accepts. The client never noticed the delay.
```

**Why it works:** the ordering dependency was never really "B must start after A"; it was "B
must be able to *connect* to A." Creating the socket satisfies that without starting
anything. This reframing — *the dependency is on the interface, not the implementation* — is
a genuinely good piece of systems design and worth being able to articulate.

The secondary benefits: a service can be restarted without dropping connections (the socket
survives), and on-demand activation means rarely-used services cost nothing until used.

The protocol a service implements:

```c
/* sd_listen_fds() equivalent, by hand: */
const char *e = getenv("LISTEN_FDS");
const char *pid = getenv("LISTEN_PID");
if (e && pid && atoi(pid) == getpid()) {
	int n = atoi(e);
	/* fds 3 .. 3+n-1 are already bound and listening. */
	for (int fd = SD_LISTEN_FDS_START; fd < SD_LISTEN_FDS_START + n; fd++)
		/* use it */;
}
```

### T.6 — cgroups, and why supervision finally works

Every systemd unit gets a cgroup (Ch. 93):

```
 /sys/fs/cgroup/
 ├── init.scope                  systemd itself
 ├── system.slice/               system services
 │   ├── sshd.service/
 │   │   └── cgroup.procs        <- EVERY process of this service
 │   └── nginx.service/
 ├── user.slice/
 │   └── user-1000.slice/
 │       └── session-3.scope/
 └── machine.slice/              containers and VMs
```

What this buys, concretely:

| Capability | Without cgroups |
|---|---|
| Kill a service completely | PID file + `kill`, and hope no child escaped |
| Know what is running | Parse PID files, which lie |
| Limit CPU/memory/IO per service | Not possible per-service |
| Account resource usage per service | Not possible |
| Contain a fork bomb | Not possible |

`systemctl status sshd` showing the process tree is reading `cgroup.procs`. **A double-forking
daemon that would have escaped SysV supervision entirely cannot escape a cgroup**, because
cgroup membership is inherited and only a privileged process can change it.

### T.7 — Service hardening: the unit file as a sandbox

An underused and genuinely valuable systemd feature: the security controls from Ch. 102,
available as one-line unit directives.

```ini
[Service]
# --- Filesystem ---
ProtectSystem=strict           # / read-only
ProtectHome=yes                # /home, /root, /run/user inaccessible
PrivateTmp=yes                 # private /tmp namespace
ReadWritePaths=/var/lib/myapp
InaccessiblePaths=/boot

# --- Kernel interfaces ---
ProtectKernelTunables=yes      # /proc/sys, /sys read-only
ProtectKernelModules=yes       # no module loading
ProtectKernelLogs=yes          # no dmesg
ProtectControlGroups=yes
ProtectProc=invisible          # cannot see other processes in /proc
ProtectClock=yes

# --- Privileges ---
User=myapp
NoNewPrivileges=yes            # the seccomp prerequisite (Ch. 102 §T.5)
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
RestrictSUIDSGID=yes

# --- Syscalls and namespaces ---
SystemCallFilter=@system-service
SystemCallFilter=~@privileged @resources
SystemCallArchitectures=native     # blocks the 32-bit-ABI bypass!
RestrictNamespaces=yes
LockPersonality=yes
MemoryDenyWriteExecute=yes         # W^X

# --- Network ---
PrivateNetwork=yes                 # or:
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
IPAddressDeny=any
IPAddressAllow=10.0.0.0/8

# --- Resources (cgroup v2) ---
MemoryMax=512M
MemoryHigh=384M                    # throttle before killing (Ch. 23)
CPUQuota=50%
TasksMax=64
IOWeight=50
```

**`systemd-analyze security <unit>` scores this**, 0–10, and tells you which directives would
improve it. Running it across a fleet's units is a cheap, high-yield security exercise.

Note `SystemCallArchitectures=native` specifically: it blocks the "invoke the 32-bit ABI to
get different syscall numbers" bypass from Ch. 102 Lab 3. A `SystemCallFilter` without it has
a hole on x86-64.

### T.8 — journald

Structured, binary, indexed logging. The design decisions:

| Decision | Rationale | Cost |
|---|---|---|
| **Binary format** | Indexed fields, fast queries, per-entry metadata | Needs `journalctl`; corruption is harder to recover |
| **Structured fields** | `_PID`, `_UID`, `_SYSTEMD_UNIT`, `_COMM`, `_BOOT_ID`, plus arbitrary app fields | More storage |
| **Trusted fields** (`_`-prefixed) | Added by journald from the sender's credentials — **cannot be forged by the logging process** | — |
| **Forward Secure Sealing** | Periodic HMAC so tampering is detectable | Needs a key |
| **Rate limiting** | A runaway logger cannot fill the disk | Logs get dropped; watch for "Suppressed N messages" |

```bash
journalctl -u nginx -f                    # follow one unit
journalctl -b -1 -p err                   # previous boot, errors and worse
journalctl --since "10 min ago" -o json-pretty
journalctl _PID=1234 _UID=1000            # structured field queries
journalctl -k                             # kernel messages (dmesg equivalent)
journalctl --disk-usage
journalctl --vacuum-size=500M
journalctl --list-boots
```

**The trusted-field property is the important one.** `_SYSTEMD_UNIT` and `_UID` are added by
journald based on the sender's socket credentials (`SO_PEERCRED`), so a compromised service
cannot log messages claiming to be from another unit. Text syslog has no such guarantee.

### T.9 — Boot time, the userspace half

```bash
systemd-analyze
# Startup finished in 1.2s (firmware) + 0.8s (loader) + 2.1s (kernel)
#                        + 4.3s (userspace) = 8.4s

systemd-analyze blame                # slowest units
systemd-analyze critical-chain       # THE ORDERING CHAIN -- more useful
systemd-analyze plot > boot.svg      # a timeline
systemd-analyze dot | dot -Tsvg > deps.svg
```

**`blame` versus `critical-chain` is the distinction to understand.** `blame` shows the
slowest units, but a slow unit that nothing waits for does not delay the boot. `critical-chain`
shows the actual serialized path — which is the only thing worth optimizing.

Common wins, in order:
1. **Disable what you do not need.** `systemctl disable`. Biggest win, zero risk.
2. **Socket-activate what is rarely used**, so it costs nothing until first use.
3. **Break false dependencies** — a unit with `After=network-online.target` that only needs a
   local socket is serializing for nothing.
4. **`DefaultTimeoutStartSec`** — a unit that times out at 90 s and then fails has cost you 90
   seconds.
5. `systemd-analyze critical-chain` again. Iterate.

### T.10 — Minimal init for embedded

When systemd is too large, the options:

**BusyBox init** (~10 KB), driven by `/etc/inittab`:
```
::sysinit:/etc/init.d/rcS
::respawn:/sbin/getty -L ttyS0 115200 vt100
::shutdown:/bin/umount -a -r
::ctrlaltdel:/sbin/reboot
```

**s6 / runit** — the design worth knowing even if you use systemd. A supervision *tree*:
every service is a directory containing a `run` script; the supervisor execs it, and restarts
it if it exits. No forking, no PID files, no daemonization. Logging is a second process in a
pipe.

```
 /etc/s6/sv/myservice/
   run         -> exec ./myservice          (NOT daemonized -- that is the point)
   finish      -> cleanup on exit
   log/run     -> exec s6-log t /var/log/myservice
```

The principle: **a service should not daemonize.** Daemonization (double fork, setsid, close
fds, write a PID file) exists solely so that a supervisor-less init could start something in
the background. With a real supervisor it is not only unnecessary but actively harmful — it
breaks supervision, loses the exit status, and creates the PID-file race. systemd
(`Type=simple`), s6, runit, and every container runtime all want a foreground process.

**This is a genuinely useful thing to know when containerizing legacy software**: the first
step is always to stop it daemonizing.

---

## 1. Internals

### Source map

| Path | Contents |
|---|---|
| `init/initramfs.c` | Kernel-side cpio unpacking |
| `init/do_mounts*.c` | Root filesystem mounting, `root=` parsing |
| `init/main.c` | `kernel_init`, the `execve` of PID 1 |
| `usr/` | Built-in initramfs generation (`CONFIG_INITRAMFS_SOURCE`) |
| `Documentation/filesystems/ramfs-rootfs-initramfs.rst` | **The definitive explanation** |
| `Documentation/driver-api/early-userspace/` | |
| systemd: `src/core/` | PID 1: units, job engine, transactions |
| systemd: `src/journal/` | journald |
| systemd: `src/shared/` | The unit-file parser |
| systemd: `NEWS`, `docs/` | Surprisingly good documentation |

### Building an initramfs by hand

```bash
#!/bin/bash
# mkinitramfs.sh — a minimal, understandable initramfs.
set -e
DIR=$(mktemp -d)
trap "rm -rf $DIR" EXIT

mkdir -p $DIR/{bin,sbin,dev,proc,sys,run,mnt/root,lib/modules}

# Static busybox: one binary, every tool.
cp $(which busybox) $DIR/bin/
for applet in sh mount umount switch_root mkdir cat ls sleep modprobe \
              insmod blkid findfs; do
	ln -sf busybox $DIR/bin/$applet
done

# The init script.
cat > $DIR/init <<'EOF'
#!/bin/sh
# PID 1 in the initramfs.
mount -t proc     proc     /proc
mount -t sysfs    sysfs    /sys
mount -t devtmpfs devtmpfs /dev

echo "initramfs: starting"

# Load the modules needed to see the root device.
for m in virtio_pci virtio_blk ext4; do
	modprobe $m 2>/dev/null
done

# Parse root= from the kernel command line.
for arg in $(cat /proc/cmdline); do
	case $arg in
		root=*)      ROOT="${arg#root=}" ;;
		rootfstype=*) FSTYPE="${arg#rootfstype=}" ;;
		rootwait)    ROOTWAIT=1 ;;
		rdinit=*)    RDINIT="${arg#rdinit=}" ;;
		init=*)      INIT="${arg#init=}" ;;
	esac
done
: ${INIT:=/sbin/init}
: ${FSTYPE:=ext4}

# rdinit= drops to a shell HERE, in the initramfs. Invaluable for debugging.
[ -n "$RDINIT" ] && exec $RDINIT

# Wait for the device to appear.
if [ -n "$ROOTWAIT" ]; then
	for i in $(seq 1 100); do
		[ -b "$ROOT" ] && break
		sleep 0.1
	done
fi

if ! mount -o ro -t "$FSTYPE" "$ROOT" /mnt/root; then
	echo "initramfs: FAILED to mount $ROOT ($FSTYPE)"
	echo "available block devices:"
	ls -l /dev/[hsv]d* /dev/mmcblk* /dev/nvme* 2>/dev/null
	exec /bin/sh                # drop to a shell instead of panicking
fi

# Hand over. switch_root frees the initramfs tmpfs.
exec switch_root /mnt/root "$INIT"
EOF
chmod +x $DIR/init

# Pack it. Note: the archive is rooted at ".", and /init must be at the top.
( cd $DIR && find . | cpio -o -H newc --quiet ) | gzip -9 > initramfs.cpio.gz
echo "built: $(du -h initramfs.cpio.gz)"

# Inspect an existing one:
#   lsinitramfs /boot/initramfs-$(uname -r).img     (Debian/Ubuntu)
#   lsinitrd    /boot/initramfs-$(uname -r).img     (Fedora/RHEL)
#   or: zcat img | cpio -idmv
```

**The `exec /bin/sh` on mount failure is the most important line.** The default behaviour of
most distribution initramfses is to drop to a rescue shell for exactly this reason: a shell
lets you look at `/dev`, `/proc/partitions`, and `dmesg` and diagnose in thirty seconds what
would otherwise take a reflash cycle.

### Unit file anatomy

```ini
[Unit]
Description=My Application Server
Documentation=man:myapp(8) https://example.com/docs
# Ordering AND dependency are separate. You usually want both.
After=network-online.target postgresql.service
Wants=network-online.target
Requires=postgresql.service
# Do not spin forever on a service that is fundamentally broken:
StartLimitIntervalSec=300
StartLimitBurst=5

[Service]
Type=notify                    # the service calls sd_notify(READY=1)
                               # simple:  ready immediately (default)
                               # forking: daemonizes (avoid)
                               # oneshot: runs and exits
                               # notify:  explicit readiness -- BEST
NotifyAccess=main
ExecStartPre=/usr/bin/myapp --check-config
ExecStart=/usr/bin/myapp --config /etc/myapp.conf
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5s
TimeoutStartSec=30s
TimeoutStopSec=30s
WatchdogSec=60s                # service must sd_notify(WATCHDOG=1)
                               # or systemd restarts it

User=myapp
Group=myapp
WorkingDirectory=/var/lib/myapp
StateDirectory=myapp           # creates and chowns /var/lib/myapp
RuntimeDirectory=myapp         # /run/myapp, removed on stop
LogsDirectory=myapp

# (hardening directives from §T.7)
NoNewPrivileges=yes
ProtectSystem=strict
ReadWritePaths=/var/lib/myapp
SystemCallFilter=@system-service
SystemCallArchitectures=native
MemoryMax=1G
CPUQuota=200%

[Install]
WantedBy=multi-user.target     # what `systemctl enable` hooks it into
```

**`Type=notify` is the right answer** for anything you control. `Type=simple` reports ready
the instant `exec` returns, so dependent units start before the service can actually serve.
`Type=forking` requires daemonization, which §T.10 argues against. `notify` gives systemd the
truth.

### Observability

```bash
# --- What is running, and why ---
systemctl status                      # the whole tree
systemctl list-units --failed
systemctl list-dependencies myapp.service
systemctl list-dependencies --reverse myapp.service   # who needs me?
systemctl show myapp.service          # EVERY property, resolved
systemctl cat myapp.service           # the unit file(s) including drop-ins

# --- Why did it not start? ---
systemctl status myapp -l --no-pager
journalctl -u myapp -b --no-pager
systemd-analyze verify /etc/systemd/system/myapp.service

# --- Resources ---
systemd-cgtop                         # top, but per-cgroup
systemd-cgls                          # the cgroup tree
cat /sys/fs/cgroup/system.slice/myapp.service/memory.current
cat /sys/fs/cgroup/system.slice/myapp.service/cpu.stat

# --- Security posture ---
systemd-analyze security               # score every unit
systemd-analyze security myapp.service # detail for one

# --- Boot ---
systemd-analyze
systemd-analyze critical-chain
systemd-analyze plot > /tmp/boot.svg

# --- Debugging PID 1 itself ---
systemd-analyze log-level debug
systemctl daemon-reload                # after editing a unit -- ALWAYS
systemctl daemon-reexec                # reexecute PID 1 (rare)
```

---

## 2. Practice

### Lab 91.1 — Build and boot a hand-made initramfs

```bash
# 1. Build it with the script above.
./mkinitramfs.sh

# 2. Boot it in QEMU with a real root.
qemu-system-x86_64 -enable-kvm -m 1G -nographic \
	-kernel arch/x86/boot/bzImage \
	-initrd initramfs.cpio.gz \
	-drive file=rootfs.ext4,format=raw,if=virtio \
	-append "console=ttyS0 root=/dev/vda rootfstype=ext4 rootwait"

# 3. Now stop IN the initramfs and look around:
#    -append "... rdinit=/bin/sh"
#    Then, at the shell:
cat /proc/cmdline
ls /dev
cat /proc/partitions
dmesg | tail -30
mount -t ext4 /dev/vda /mnt/root && echo "root is mountable"

# 4. Break it deliberately:
#    - root=/dev/vdb (nonexistent)  -> should drop to the rescue shell
#    - rootfstype=xfs (wrong)       -> mount fails, shell
#    - remove the virtio_blk modprobe -> no device at all
#    For each, note exactly what the failure looks like.
```

**Step 4 is the lab.** Recognizing these three failure signatures instantly is worth an hour
of production downtime.

### Lab 91.2 — Write a properly hardened unit

```bash
# 1. A trivial service to harden.
sudo tee /usr/local/bin/myapp <<'EOF'
#!/bin/bash
echo "myapp starting as uid=$(id -u)"
while :; do
	echo "tick $(date -Is)" >> /var/lib/myapp/log
	sleep 5
done
EOF
sudo chmod +x /usr/local/bin/myapp

# 2. The naive unit.
sudo tee /etc/systemd/system/myapp.service <<'EOF'
[Unit]
Description=My App
[Service]
ExecStart=/usr/local/bin/myapp
[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start myapp
systemd-analyze security myapp.service
#   -> expect a score around 9.6 UNSAFE. Read every line of the output.

# 3. Harden it, one directive at a time, re-scoring after each.
sudo tee /etc/systemd/system/myapp.service <<'EOF'
[Unit]
Description=My App
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/myapp
Restart=on-failure
RestartSec=5s

DynamicUser=yes
StateDirectory=myapp

NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
PrivateDevices=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectKernelLogs=yes
ProtectControlGroups=yes
ProtectProc=invisible
ProtectClock=yes
ProtectHostname=yes
RestrictNamespaces=yes
RestrictRealtime=yes
RestrictSUIDSGID=yes
LockPersonality=yes
MemoryDenyWriteExecute=yes
RestrictAddressFamilies=AF_UNIX
CapabilityBoundingSet=
SystemCallFilter=@system-service
SystemCallFilter=~@privileged @resources
SystemCallArchitectures=native

MemoryMax=64M
CPUQuota=10%
TasksMax=16

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload && sudo systemctl restart myapp
systemd-analyze security myapp.service
#   -> should now be in the 1-2 range.

# 4. VERIFY the sandbox actually works.
sudo systemd-run --unit=probe --service-type=oneshot \
	-p ProtectSystem=strict -p NoNewPrivileges=yes \
	/bin/sh -c 'touch /etc/probe 2>&1; cat /proc/1/maps 2>&1'
journalctl -u probe --no-pager
```

**Step 4 matters.** A high score with a sandbox that does not actually restrict anything is
worse than no sandbox, because it creates false confidence.

### Lab 91.3 — Socket activation from scratch

```c
/* sockact.c — a socket-activated echo server.
 * Build: gcc -O2 -o sockact sockact.c
 * Demonstrates the sd_listen_fds protocol WITHOUT libsystemd.        */
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <sys/socket.h>
#include <sys/types.h>

#define SD_LISTEN_FDS_START 3

static int listen_fds(void)
{
	const char *e_fds = getenv("LISTEN_FDS");
	const char *e_pid = getenv("LISTEN_PID");
	int n;

	if (!e_fds || !e_pid)
		return 0;
	/* LISTEN_PID guards against inheriting these across an exec to a
	 * different process -- check it, always.                        */
	if (atoi(e_pid) != getpid())
		return 0;
	n = atoi(e_fds);
	return n > 0 ? n : 0;
}

static void notify_ready(void)
{
	/* sd_notify(READY=1) without libsystemd: a datagram to $NOTIFY_SOCKET */
	const char *path = getenv("NOTIFY_SOCKET");
	struct sockaddr_un { unsigned short f; char p[108]; } sa = {0};
	int fd;
	if (!path) return;
	fd = socket(AF_UNIX, SOCK_DGRAM | SOCK_CLOEXEC, 0);
	if (fd < 0) return;
	sa.f = AF_UNIX;
	strncpy(sa.p, path, sizeof(sa.p) - 1);
	if (sa.p[0] == '@') sa.p[0] = '\0';        /* abstract socket */
	sendto(fd, "READY=1\n", 8, 0, (void *)&sa, sizeof(sa));
	close(fd);
}

int main(void)
{
	int n = listen_fds(), lfd;
	char buf[4096];

	if (n < 1) {
		fprintf(stderr, "not socket-activated; expected LISTEN_FDS\n");
		return 1;
	}
	lfd = SD_LISTEN_FDS_START;      /* already bound AND listening */
	fprintf(stderr, "got %d socket(s) from systemd; serving\n", n);

	notify_ready();

	for (;;) {
		int cfd = accept4(lfd, NULL, NULL, SOCK_CLOEXEC);
		ssize_t r;
		if (cfd < 0) {
			if (errno == EINTR) continue;
			perror("accept4");
			return 1;
		}
		while ((r = read(cfd, buf, sizeof(buf))) > 0)
			if (write(cfd, buf, r) != r) break;
		close(cfd);
	}
}
```

```ini
# /etc/systemd/system/sockact.socket
[Unit]
Description=Socket-activated echo
[Socket]
ListenStream=12345
Accept=no                      # one instance handles all connections
[Install]
WantedBy=sockets.target
```
```ini
# /etc/systemd/system/sockact.service
[Unit]
Description=Socket-activated echo server
Requires=sockact.socket
After=sockact.socket
[Service]
Type=notify
ExecStart=/usr/local/bin/sockact
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now sockact.socket
systemctl status sockact.service        # INACTIVE -- not started yet

nc localhost 12345                      # type something; it echoes
systemctl status sockact.service        # NOW it is running

# Prove connections are not lost during the start delay:
sudo systemctl stop sockact.service
time (echo hello | nc -q1 localhost 12345)
#   -> returns "hello", having transparently started the service.
```

### Lab 91.4 — Fix a real dependency bug

```bash
# 1. Create the classic bug: Requires without After.
sudo tee /etc/systemd/system/backend.service <<'EOF'
[Unit]
Description=Backend (slow to start)
[Service]
Type=simple
ExecStartPre=/bin/sleep 5
ExecStart=/bin/sh -c 'while :; do sleep 60; done'
EOF

sudo tee /etc/systemd/system/frontend.service <<'EOF'
[Unit]
Description=Frontend
Requires=backend.service
# BUG: no After=backend.service
[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo "frontend: backend is $(systemctl is-active backend.service)"'
RemainAfterExit=yes
EOF

sudo systemctl daemon-reload
sudo systemctl start frontend
journalctl -u frontend -n5 --no-pager
#   -> "backend is activating"  -- the race, made visible.

# 2. Fix it.
sudo sed -i '/Requires=backend.service/a After=backend.service' \
	/etc/systemd/system/frontend.service
sudo systemctl daemon-reload
sudo systemctl restart backend frontend
journalctl -u frontend -n5 --no-pager
#   -> "backend is active"

# 3. But this is still wrong: "active" != "ready to serve". Change
#    backend to Type=notify and have it signal readiness properly.
#    THAT is the real fix, and it is why Type=notify exists.

# 4. Audit your whole system for the same bug:
for u in /etc/systemd/system/*.service; do
	if grep -q '^Requires=' "$u" && ! grep -q '^After=' "$u"; then
		echo "SUSPECT: $u"
	fi
done
```

### Lab 91.5 — Boot-time optimization

```bash
# 1. Baseline.
systemd-analyze
systemd-analyze critical-chain | tee /tmp/before.txt
systemd-analyze plot > /tmp/before.svg

# 2. What is actually on the critical path? (Not the same as "slow".)
systemd-analyze critical-chain

# 3. What is enabled that you do not need?
systemctl list-unit-files --state=enabled

# 4. Common targets:
sudo systemctl disable NetworkManager-wait-online.service   # often 5-30s!
sudo systemctl disable systemd-networkd-wait-online.service
#    These WAIT FOR THE NETWORK. If nothing needs it at boot, that is
#    pure latency. This is the single most common boot-time waste.

# 5. Find units with unnecessary network-online dependencies:
grep -l 'network-online.target' /etc/systemd/system/*.service \
	/lib/systemd/system/*.service 2>/dev/null

# 6. Convert rarely-used services to socket activation.

# 7. Reduce the default timeout so failures cost less:
sudo mkdir -p /etc/systemd/system.conf.d
echo -e '[Manager]\nDefaultTimeoutStartSec=15s' | \
	sudo tee /etc/systemd/system.conf.d/timeout.conf

# 8. Re-measure.
sudo reboot
systemd-analyze
diff /tmp/before.txt <(systemd-analyze critical-chain)
```

### Lab 91.6 — Build a minimal embedded init

```bash
# --- BusyBox init ---
mkdir -p rootfs/{bin,sbin,etc/init.d,proc,sys,dev,tmp,var}
cp $(which busybox) rootfs/bin/
chroot rootfs /bin/busybox --install -s

cat > rootfs/etc/inittab <<'EOF'
::sysinit:/etc/init.d/rcS
::respawn:/sbin/getty -L ttyS0 115200 vt100
::restart:/sbin/init
::shutdown:/bin/umount -a -r
::ctrlaltdel:/sbin/reboot
EOF

cat > rootfs/etc/init.d/rcS <<'EOF'
#!/bin/sh
mount -t proc     proc     /proc
mount -t sysfs    sysfs    /sys
mount -t devtmpfs devtmpfs /dev
mount -t tmpfs    tmpfs    /tmp
mkdir -p /dev/pts && mount -t devpts devpts /dev/pts
hostname embedded
ip link set lo up
echo "system up in $(cut -d' ' -f1 /proc/uptime)s"
EOF
chmod +x rootfs/etc/init.d/rcS

# --- Measure ---
# Boot it and compare with systemd on the same hardware:
#   - time to a login prompt
#   - RSS of PID 1
#   - total rootfs size
#   - what you gave up (supervision, socket activation, resource limits,
#     structured logging, sandboxing)
```

**Write the comparison table.** The point is not that one wins; it is that you can state the
trade in concrete numbers, which is what an architect is asked for.

---

## 3. Mastery drills

1. Build a custom initramfs that unlocks a LUKS volume, assembles an LVM logical volume, and
   switches root. Handle every failure with a useful message and a rescue shell.

2. Take five services on a real system and harden them to a `systemd-analyze security` score
   under 3, verifying each restriction actually applies. Document anything that broke.

3. Convert a `Type=forking` daemon to `Type=notify` with proper `sd_notify` readiness and a
   watchdog. Verify the watchdog restarts it when it hangs.

4. Reduce a system's userspace boot time by 50%, using `critical-chain` rather than `blame`.
   Document every change and its measured effect.

5. Implement socket activation for a service you own, including the `LISTEN_PID` check and
   graceful restart without dropping connections.

6. Write an init system. Minimal, but correct: reap orphans, supervise, handle SIGTERM,
   shut down cleanly. Run it as PID 1 in a container. You will learn more from this than
   from reading about any of them.

7. Compare systemd, s6, runit, and BusyBox init on the same embedded target: boot time, RSS,
   binary size, and the feature table. Write the recommendation.

8. Read `Documentation/filesystems/ramfs-rootfs-initramfs.rst` completely and explain the
   difference between `rootfs`, `ramfs`, `tmpfs`, `initrd`, and `initramfs` — all five —
   without looking.

9. Instrument a boot with `systemd-analyze plot` and a kernel `initcall_debug` bootgraph, and
   produce a single timeline from power-on to login (Ch. 90 + this chapter).

10. Debug five deliberately-broken systemd configurations created by a colleague: a
    dependency cycle, a missing `After=`, a unit that times out, a failed sandbox restriction,
    and a socket/service mismatch.

11. Build a fully journald-free embedded logging setup (syslog to a ring buffer in tmpfs,
    rotated) and explain the trade against structured logging.

12. Design the service architecture for an embedded product: what runs at boot, what is
    socket-activated, what the restart policies are, what the resource limits are, and what
    the failure behaviour is when a critical service cannot start.

---

## 4. Further reading

**Kernel**
- `Documentation/filesystems/ramfs-rootfs-initramfs.rst` — **read this one properly**; it
  clears up a genuinely confusing area
- `Documentation/admin-guide/initrd.rst`
- `init/main.c` and `init/do_mounts.c`

**systemd**
- `man systemd.unit`, `systemd.service`, `systemd.exec`, `systemd.resource-control`,
  `systemd.socket` — `systemd.exec(5)` is where all the hardening directives are documented
  and is worth reading in full
- Lennart Poettering's "systemd for Administrators" blog series (20 parts) — dated in places,
  still the best explanation of the design rationale
- `systemd.io` — the developer documentation, including the `sd_notify` protocol and the
  container interface
- "Rethinking PID 1" (Poettering, 2010) — the original design argument

**Alternatives**
- `skarnet.org/software/s6/` — the s6 documentation; the best writing anywhere on the
  supervision-tree model and on why daemonization is wrong
- runit documentation
- BusyBox init source (`init/init.c`) — ~1000 lines; read it all in an afternoon

**Books**
- Chris Simmonds, *Mastering Embedded Linux Programming*, ch. 10–11
- *systemd for Administrators* / the Arch and Debian systemd wiki pages — practical and
  accurate
- Michael Kerrisk, *The Linux Programming Interface*, ch. 37 (daemons) — for why
  daemonization is the way it is, and why you should stop doing it

**Cross-references**
- Ch. 90 — what happens before PID 1
- Ch. 93 — namespaces and cgroups, which systemd uses for everything
- Ch. 95 — observability, including journald's place in it
- Ch. 102 — the security controls that §T.7's directives configure

→ Next: [92-elf-libc.md](92-elf-libc.md)

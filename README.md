# memory-control-config

A BitBake recipe that installs systemd drop-in files to apply cgroup v2 memory
limits to services **without modifying those services' own recipes**.

The recipe is layer-agnostic: it ships with an empty service list and does
nothing until a distro, machine, or product layer configures
`MEMORY_CONTROL_SERVICES` (typically from a `.bbappend`). It never changes
global kernel OOM policy.

---

## Requirements

The target system must provide:

- `systemd` (the recipe is skipped on non-systemd distros via
  `REQUIRED_DISTRO_FEATURES`)
- cgroup v2 mounted at `/sys/fs/cgroup`
- the `memory` controller listed in `cgroup.controllers`

Check the target with:

```bash
mount | grep cgroup2
cat /sys/fs/cgroup/cgroup.controllers
```

If `memory` is absent from `cgroup.controllers`, `MemoryMax` / `MemoryHigh`
will not be enforced until the kernel is built with memory cgroup support
(`CONFIG_CGROUPS=y`, `CONFIG_MEMCG=y`). How that is enabled is platform
specific — see [Notes](#notes).

---

## Configuration

Leave `MEMORY_CONTROL_SERVICES` empty in the recipe and set limits from your
own layer with a `.bbappend`:

```bitbake
# recipes-support/memory-control-config/memory-control-config_1.0.bbappend
MEMORY_CONTROL_SERVICES:append = " webui:256M:kill inventory-sync:128M"
```

Entry format:

```text
<unit>:<limit>[:<oom_policy>]
```

| Field        | Values                              | Notes                          |
|--------------|-------------------------------------|--------------------------------|
| `unit`       | systemd service filename            | `.service` suffix is optional  |
| `limit`      | raw bytes, or `256K` / `512M` / `1G`| sets `MemoryMax`               |
| `oom_policy` | `default` (default) or `kill`       | see below                      |

A malformed entry (bad size such as `1.5G`, unknown suffix, or an invalid
policy) fails the build with a clear error rather than installing a dangerous
value.

---

## Generated drop-ins

For each configured entry the recipe generates a drop-in under the systemd
system unit directory (`${systemd_system_unitdir}`):

```text
/lib/systemd/system/<unit>.service.d/memory-control.conf
```

(on usr-merged distros this is `/usr/lib/systemd/system/...`). This is a
location systemd always scans on a read-only root, so the limit applies at
boot with no `daemon-reload`. Because it sits below `/etc` in systemd's
precedence, an integrator can still override it by dropping a file at
`/etc/systemd/system/<unit>.service.d/memory-control.conf`.

Each generated file contains:

```ini
[Service]
MemoryMax=<bytes>
MemoryHigh=<70% of max>
MemorySwapMax=0
```

`MemoryHigh` is set to 70% of `MemoryMax` so the kernel can reclaim
file-backed pages before hitting the hard ceiling.

For entries using `kill`, the drop-in also adds:

```ini
OOMPolicy=continue
OOMScoreAdjust=500
```

- **`default`** — install only the cgroup memory limits. The unit keeps
  whatever OOM behaviour the platform and its own unit settings already define.
- **`kill`** — same limits, plus `OOMPolicy=continue` and `OOMScoreAdjust=500`
  so systemd keeps managing the unit after the kernel OOM-kills a process
  inside its cgroup, and that process is preferred by the OOM killer.

---

## Installation

Add the recipe to an image or packagegroup:

```bitbake
IMAGE_INSTALL:append = " memory-control-config"
```

The recipe always produces a package (`ALLOW_EMPTY`), so images can depend on
it even when no services are configured yet.

---

## Verification

After booting a target image:

```bash
# Find the installed drop-in (unit dir on a shipped image, or /etc if overridden)
find /usr/lib/systemd/system /lib/systemd/system /etc/systemd/system \
    -name memory-control.conf 2>/dev/null

# Confirm systemd actually loaded it (this is the file's real test):
systemctl show webui -p DropInPaths

# Show the limits systemd applied for the unit
systemctl show webui -p MemoryMax -p MemoryHigh -p OOMPolicy
```

If `DropInPaths` is empty, systemd is not reading the file — it is in a
directory systemd does not scan (a common cause on overlay/persist roots) or a
`daemon-reload` is needed after writing it post-boot. A file shipped in the
image at the unit dir is read at boot with no reload.

A host + on-target test suite lives in
[`tests/test_memory_control_config.py`](tests/test_memory_control_config.py):

```bash
# Host-side unit tests (no systemd/cgroups required):
python3 -m pytest tests/test_memory_control_config.py -v -k DropIn

# Full suite, run as root on the target:
python3 -m pytest tests/test_memory_control_config.py -v
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `MemoryMax` shows `infinity`, `DropInPaths` empty | file not in a directory systemd scans (e.g. stranded on a persist/overlay path) | ensure it is installed under the unit dir and shipped in the image |
| `MemoryMax` shows `infinity`, drop-in present | wrong unit name (e.g. socket-activated/templated unit) | check `systemctl show <unit> -p Id` and match the drop-in dir name |
| `memory` absent from `cgroup.controllers` | kernel missing memory cgroup support | enable `CONFIG_MEMCG` and rebuild |
| cgroups v2 not mounted | init not using the unified hierarchy | enable cgroup v2 on the target |
| service keeps getting OOM-killed | `MemoryMax` too low | watch `MemoryCurrent` and raise the limit |
| change to a limit had no effect after rebuild | stale sstate | recipe declares `do_install[vardeps]`; if editing, `bitbake -c clean memory-control-config` |

---

## Notes

The target kernel still needs cgroup v2 memory-controller support. How that is
enabled is platform-specific and should be owned by your platform's kernel
configuration — this recipe deliberately does not touch the kernel.

If your platform does not already enable the required options, keep a separate
platform-owned kernel fragment such as the example in
[`files/cgroups-v2.cfg`](recipes-support/memory-control-config/files/cgroups-v2.cfg).
At minimum it should enable `CONFIG_CGROUPS` and `CONFIG_MEMCG`.
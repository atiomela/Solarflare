# solarflare_update

Ansible role that detects Solarflare/Xilinx/AMD NICs (PCI vendor `1924`) and
performs **driver**, **Onload**, and/or **firmware** upgrades with automatic
rollback. Supports **RedHat** and **Debian** OS families.

## What it does

1. **Detect**: `lspci -d 1924:` sets `solarflare_present`; enumerates `sfc`
   interfaces; reads driver, firmware, and Onload versions.
2. **Resolve**: looks up the running kernel + OS in `solarflare_compat_matrix`
   to determine the target driver / firmware / Onload versions. Manual
   `*_src` / `*_url` / `*_image` variables override the matrix. Always prints
   a "next manual update" report with source URLs.
3. **Prepare**: creates `/tmp/solarflare_update/`, backs up the current
   initramfs, records installed SFC packages, stages packages from URL or
   controller path.
4. **Driver update** (RPM via `dnf` + `dracut`, or DEB via `apt` +
   `update-initramfs`): stops services, unloads modules, installs, regenerates
   initramfs (running kernel or every kernel), reloads driver, verifies
   version matches matrix target. Rolls back on any failure.
5. **Onload update**: same pattern as driver, separately controllable.
6. **Firmware update** via `sfupdate`: snapshots current image, flashes, asserts
   link healthy and version matches matrix target. Re-flashes snapshot on
   failure.

## Actions

The default action is **`rebuild`** — the role assumes some other process
updates the OS / kernel (it does **not** manage OS updates itself) and its job
is to bring the SFC driver and Onload back online against the new running
kernel.

| Value | What runs |
|---|---|
| `rebuild` *(default)* | rebuild driver (and Onload if it was installed) for the running kernel — DKMS-aware |
| `none` | detection + resolution only — prints the "next manual update" report |
| `driver` | full driver upgrade (matrix or manual package) |
| `firmware` | NIC firmware only, via `sfupdate` |
| `onload` | full Onload upgrade |
| `both` | driver + firmware |
| `all` | driver + firmware + Onload |

### How `rebuild` works

1. Runs `dkms status` to see whether `sfc` and/or `onload` are DKMS-packaged.
2. If they are, and the running kernel is **not already covered**
   (`<module>/<ver>, <ansible_kernel>, ...: installed`), runs
   `dkms install -m <mod> -v <ver> -k {{ ansible_kernel }} --force`.
3. Regenerates initramfs (`dracut` on RedHat, `update-initramfs` on Debian).
4. `depmod` + `modprobe sfc`, waits for the interface(s) to reappear,
   restarts the services that were running before.
5. **If the module isn't DKMS-managed**, falls back to the standard
   `driver_update.yml` / `onload_update.yml` flow using the matrix-resolved
   or manual package.
6. Onload is only touched if it was already installed
   (`solarflare_rebuild_onload_if_present: true`, default), so the role
   doesn't introduce Onload on hosts that don't use it.

If the running kernel is already covered by DKMS and
`solarflare_rebuild_skip_if_current: true` (default), the block is a no-op —
safe to run on every reboot/post-kernel-update playbook.

Rollback: failure in either rebuild block restores the saved initramfs and
falls back to reloading the previous `sfc` module, then fails the play with
a pointer to the relevant `/var/lib/dkms/.../make.log`.

## Version selection — three modes

`solarflare_version_selection` controls how target versions are picked:

- **`matrix`** — use only `solarflare_compat_matrix`. First entry whose
  `os_family`, optional `distribution_major_version`, and `kernel_regex` all
  match the host wins.
- **`manual`** — use only the explicit `*_src` / `*_url` / `*_image` variables.
- **`auto`** (default) — manual overrides win when set; otherwise the matrix
  is used.

You can also pin a specific named release with `solarflare_pin_release: el8-2024Q4`
to roll a known-good combo across staged groups.

## The compatibility matrix

`solarflare_compat_matrix` in `defaults/main.yml` is the source of truth for
which driver / firmware / Onload version belongs with which kernel. Each entry
looks like:

```yaml
- release: "el8-2024Q4"
  os_family: "RedHat"                       # RedHat | Debian
  distribution_major_version: "8"           # optional, narrows the match
  kernel_regex: "^4\\.18\\."                # matched against ansible_kernel
  driver:
    version: "5.3.14.1010"
    pkg: "kernel-module-sfc-RHEL8-5.3.14.1010-1.x86_64.rpm"
    url: ""                                 # optional, downloaded onto target
    source: "https://www.xilinx.com/..."    # for next-manual-update report
  firmware:
    version: "8.7.0.1004"
    image: "/opt/sf/firmware/SFA-bundle-8.7.0.1004.dat"
    source: "https://www.xilinx.com/..."
  onload:
    version: "8.1.2.26"
    pkg: "onload-8.1.2.26-1.x86_64.rpm"
    url: ""
    source: "https://www.xilinx.com/..."
```

Override the whole list in `group_vars/` to match your fleet. The shipped default has entries covering:

| OS family | Releases | Kernel regex |
|---|---|---|
| RHEL / Rocky / Alma / CentOS | 6, 7, 8, 9, 10 | `^2.6.32-*el6`, `^3.10.0-*el7`, `^4.18.0-*el8`, `^5.14.0-*el9`, `^6.12.` |
| Debian | 10, 11, 12, 13 | `^4.19.`, `^5.10.`, `^6.1.`, `^6.12.` |
| Ubuntu LTS (.04 only) | 16.04, 18.04, 20.04, 22.04, 24.04, 26.04 | GA + HWE kernel entries per release |

Ubuntu entries are split GA/HWE because Ubuntu LTS ships both a GA kernel
and a rolling Hardware Enablement (HWE) kernel from the next LTS — the role
matches whichever is actually running. Fill in the real package URLs and
stage the artefacts; the version strings shipped are placeholders based on
the public Solarflare/Xilinx/AMD release lineage and need to be verified
against the current vendor matrix.

> **EOL releases**: RHEL 6, Ubuntu 16.04, Debian 10/11 are listed but
> intentionally pinned to older driver/Onload lines (sfc 4.15.x, Onload 7.x).
> Onload 8.x and 9.x do not support those kernels.

When no entry matches the host, the role still runs but warns; you can then
fall back to manual variables, or fix the matrix.

## "Next manual update" report

On every run (including `solarflare_action: none`) the role prints:

```
===== Solarflare version resolution =====
OS family / version : RedHat / 8
Running kernel      : 4.18.0-553.16.1.el8_10.x86_64
Matrix release used : el8-2024Q4
Selection mode      : auto
----- Driver -----
  target version : 5.3.14.1010
  package        : kernel-module-sfc-RHEL8-5.3.14.1010-1.x86_64.rpm
  url            : (none)
  local src      : /srv/pkg/kernel-module-sfc-...rpm
  source page    : https://www.xilinx.com/support/download/nic-software-and-drivers.html
----- Firmware -----
  target version : 8.7.0.1004
  image          : /opt/sf/firmware/SFA-bundle-8.7.0.1004.dat
  source page    : https://www.xilinx.com/support/download/nic-firmware.html
----- Onload -----
  target version : 8.1.2.26
  package        : onload-8.1.2.26-1.x86_64.rpm
  source page    : https://www.xilinx.com/support/download/nic-software-and-drivers.html#onload
=========================================
```

This makes the role useful as a fleet-wide audit tool: run with
`solarflare_action=none --tags solarflare_resolve` to see what each host
*would* be upgraded to and where to fetch the next round of artefacts.

## Pre-flight compatibility checks

Before any action that touches the SFC stack (`driver`, `onload`, `firmware`,
`both`, `all`, `rebuild`), the role runs a pre-flight that **stops the play
before unloading anything** if it finds a problem. Loaded from
`vars/compat_rules.yml`, the checks cover:

1. **Matrix consistency** — the running kernel must still match the
   `kernel_regex` of the resolved matrix entry. Catches operator mistakes
   like a stale `solarflare_pin_release`.
2. **Driver ↔ kernel** — the target driver series (e.g. `5.3`) must support
   the running kernel's major.minor.
3. **Onload ↔ kernel** — the target Onload major (7/8/9) must support the
   running kernel.
4. **Onload ↔ driver** — the target Onload major must be compatible with the
   target driver series (Onload 7 needs sfc 4.15, Onload 8 needs sfc 5.3,
   Onload 9 needs sfc 5.3 or 5.4).
5. **Firmware ↔ driver** — older 6.x firmware only works with the 4.15
   driver line; modern 8.x firmware requires sfc 5.x.
6. **Build prerequisites** — `kernel-devel-$(uname -r)` /
   `linux-headers-$(uname -r)` and `gcc`/`make` must be present, otherwise
   any DKMS rebuild or out-of-tree module compile will fail mid-flight.
7. **Non-fatal warnings** — e.g. "Onload is installed but `solarflare_action`
   does not touch it; after a driver change Onload may need a rebuild —
   consider `all` or `rebuild`".

Failures are explicit and listed individually before the play stops. Set
`solarflare_compat_strict: false` to demote failures to warnings (use when
you know better than the rules — e.g. you're testing a new driver release
the matrix hasn't been updated for yet).

The rules file is overridable per-host or per-group; the shipped version
encodes the documented kernel ranges and driver/onload/firmware
combinations for the sfc 4.15, 5.3, 5.4 driver lines and Onload 7/8/9.

## Examples

Audit only:

```bash
ansible-playbook -i inv site.yml -e solarflare_action=none
```

Matrix-driven driver + firmware:

```bash
ansible-playbook -i inv site.yml \
  -e solarflare_action=both \
  -e solarflare_version_selection=matrix
```

Manual one-shot driver upgrade:

```bash
ansible-playbook -i inv site.yml --limit nic01 \
  -e solarflare_action=driver \
  -e solarflare_version_selection=manual \
  -e solarflare_driver_pkg_src=/srv/pkg/kernel-module-sfc-5.3.14.1010-1.x86_64.rpm
```

Pin a tested release across one group:

```yaml
# group_vars/canary.yml
solarflare_pin_release: "el8-2024Q4"
solarflare_action: "all"
```

## Facts the role sets

- `solarflare_present`, `solarflare_interfaces`
- `solarflare_driver_version_before` / `_after`
- `solarflare_firmware_version_before` / `_after`
- `solarflare_onload_installed`, `solarflare_onload_version_before` / `_after`,
  `solarflare_onload_source`, `solarflare_onload_loaded`
- `solarflare_matrix_match` — the full matrix entry chosen
- `solarflare_resolved` — flat dict of target `{driver, firmware, onload}`
- `solarflare_driver_update_ok`, `solarflare_onload_update_ok`,
  `solarflare_firmware_update_ok`

## Requirements

- RHEL/Rocky/Alma 8+ **or** Debian 11+/Ubuntu 20.04+
- `pciutils`, `ethtool`, `kmod` on every target
- `dracut` (RHEL) or `initramfs-tools` (Debian) for initramfs regeneration
- `sfutils` package (provides `sfupdate`) for firmware upgrades
- `community.general` collection (for `modprobe` module)


## Internal package repository (HTTP Basic auth, flat layout)

All artefacts live on one HTTP server with HTTP Basic auth, organised flat:
**one directory per OS, all package files in it.**

```yaml
solarflare_repo_base_url: "https://repo.example.local/solarflare"
solarflare_repo_user: ""                # vault these
solarflare_repo_pass: ""
solarflare_repo_validate_certs: true
solarflare_repo_force_basic_auth: true

# Per-host directory under base_url. Default: distribution+major lowercased
# (rhel8, ubuntu22, debian12). Override per group_var if your tree is named
# differently ("rhel-8", "el8", "rocky8", ...).
solarflare_repo_os_dir: "{{ (ansible_distribution ~ ansible_distribution_major_version) | lower }}"
```

Resulting URLs are simply `{base_url}/{os_dir}/{filename}` — for example
`https://repo.example.local/solarflare/rhel8/onload-8.1.2.26-1.x86_64.rpm`.

### Onload package set — autodetected from the matrix filename

The role takes the matrix entry's `pkg:` filename (e.g. `onload-8.1.2.26-1.x86_64.rpm`)
and derives sibling names by string substitution:

| Main filename | Derived siblings |
|---|---|
| `onload-<v>-<r>.x86_64.rpm` | `onload-kmod-<v>-<r>.x86_64.rpm`, `onload-devel-<v>-<r>.x86_64.rpm` |
| `onload_<v>-<r>_amd64.deb` | `onload-dkms_<v>-<r>_all.deb`, `onload-dev_<v>-<r>_amd64.deb` |

It then issues a `HEAD` request against each candidate URL (with Basic auth)
to confirm the file exists in the repo. Results:

- HTTP 200 → keep, will be downloaded and installed.
- HTTP 404 on a `required: true` sibling → fail the play with the missing URL.
- HTTP 404 on an optional sibling → silently skip.

You can override the sibling list per OS via `solarflare_onload_sibling_suffixes`
in `defaults/main.yml`, or supply explicit `kmod_pkg:` / `devel_pkg:` fields
in the matrix entry to bypass autodetect for a specific release.

Disable autodetect entirely with `solarflare_onload_autodetect_set: false` —
the role will then install ONLY the file listed in the matrix `pkg:` field.

## Onload is a package SET, not one file

Onload is installed as a *list* of related packages; defaults per OS family:

```yaml
solarflare_onload_packages_default:
  RedHat:
    - { name: onload,       filename_key: pkg,        required: true  }
    - { name: onload-kmod,  filename_key: kmod_pkg,   required: true  }
    - { name: onload-devel, filename_key: devel_pkg,  required: false }
  Debian:
    - { name: onload,       filename_key: pkg,        required: true  }
    - { name: onload-dkms,  filename_key: kmod_pkg,   required: true  }
    - { name: onload-dev,   filename_key: devel_pkg,  required: false }
```

Each matrix entry's `onload:` block carries the filenames for the keys
referenced by `filename_key`:

```yaml
onload:
  version:   "8.1.2.26"
  pkg:       "onload-8.1.2.26-1.x86_64.rpm"
  kmod_pkg:  "onload-kmod-8.1.2.26-1.x86_64.rpm"
  devel_pkg: "onload-devel-8.1.2.26-1.x86_64.rpm"
```

At runtime each is downloaded with HTTP Basic auth from
`{base_url}/{rendered_path}/{filename}`, dropped into
`/tmp/solarflare_update/`, then installed in a single transaction
(`dnf install pkg1 pkg2 pkg3` on RHEL, sequential `apt deb=` on Debian).
The rescue path removes the whole set on failure.

Set a package's `required: false` if your environment doesn't have it (e.g.
no `-devel` package) — the role then silently skips it. Required packages
with no resolved filename cause a pre-install failure with a clear pointer
to which matrix key needs to be filled in.
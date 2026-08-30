# 10 — Build System: How Images Are Produced

> Last updated: 2026-08-22
> All paths relative to `playos-refdistro/` unless stated otherwise.

This file explains, end-to-end, how the PlayOS build system turns source code
into bootable images for **QEMU**, the **ROG Ally**, and the **installer USB**.
The canonical implementation is the refdistro `Makefile` + `scripts/` +
`br2-external/`; the spec-side overview is `playos-spec/src/build-guide.md`.

---

## 1. The two-stage model

```
Stage A  ── Buildroot builds a target rootfs (initramfs, ext2, squashfs)
Stage B  ── PlayOS shell scripts assemble disk images from those artifacts
Stage C  ── Run/verify/flash scripts (QEMU boot, A/B pivot check, USB flash)
```

- **Stage A** is Buildroot: cross toolchain, kernel, packages, rootfs images.
  One Buildroot *output directory* (`O=`) per target, each configured by a defconfig.
- **Stage B** is plain `bash` + `sgdisk`/`sfdisk` + `losetup` + `mkfs.*` +
  `mount` — no genimage. The scripts copy the Stage A artifacts into GPT disk images.
- **Stage C** boots the result (QEMU) or writes it to USB (`dd`).

Everything is pinned by `versions.lock` (Buildroot commit, kernel version,
component SHAs) for reproducibility.

---

## 2. Entry points — the Makefile targets

```
make setup              clone+pins Buildroot, clones component repos into src/, verifies pins
make verify-pins        fail if required versions.lock pins are missing/inconsistent

# QEMU (dev)
make qemu-build         Buildroot build → output/qemu
make qemu-run           boot output/qemu with OVMF (scripts/qemu-boot-check.sh)
make qemu-pivot-check   synthetic A/B disk; verify slot pivot + forced rollback

# ROG Ally
make ally-build             dev image → output/ally
make ally-production-build  production image → output/ally-production
make ally-usb-image         assemble USB-bootable disk image (needs ally-build)
make ally-flash             print the dd/flash command

# Installer
make installer-build        installer rootfs → output/installer
make installer-image        assemble installer USB (needs installer-build + ally-build)
make installer-flash        print the dd/flash command

# Updates
make update-bundle          dev-signed .playosb from output/ally/images/rootfs.squashfs

make clean / distclean      remove outputs / remove everything
```

Every `*-build` target runs three Buildroot invocations:

1. `<defconfig>` — generate the `.config` for that output dir
2. `dirclean` every `playos-*` package — force re-sync of local `src/` sources
3. full build

> **Why dirclean?** Local-site packages (`SITE_METHOD = local`) are rsync'd
> into `output/<target>/build/` only once. Without dirclean, edits under
> `src/<component>` would be silently ignored on rebuilds.

---

## 3. The five defconfigs

| Defconfig (`br2-external/configs/`) | Output dir | Purpose |
|---|---|---|
| `playos_qemu_x86_64_defconfig` | `output/qemu` | dev target; generic x86_64, softpipe GL, GRUB2 EFI, separate kernel+initramfs |
| `playos_ally_defconfig` | `output/ally` | ROG Ally **dev** image (haswell, amdgpu, BusyBox+Dropbear+evtest, embedded initramfs) |
| `playos_ally_production_defconfig` | `output/ally-production` | ROG Ally **production** (Sprint 12: no BusyBox/Dropbear/evtest, post-build lint) |
| `playos_ally_installer_defconfig` | `output/installer` | one-shot installer (kernel cmdline `playos.mode=install`) |
| `playos_installer_qemu_defconfig` | *(no Makefile target)* | graphical installer UI under QEMU (virtio-gpu, softpipe) |

Key Buildroot knobs per defconfig:

- **Kernel:** `BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE` → `board/<target>/linux.config`;
  `BR2_GLOBAL_PATCH_DIR` → `board/patches`; kernel 6.12.103.
- **Initramfs:** `BR2_TARGET_ROOTFS_INITRAMFS=y`; Ally/installer defconfigs set
  `BR2_LINUX_KERNEL_INITRAMFS_SOURCE=$(BINARIES_DIR)/rootfs.cpio` (**embedded**
  in the bzImage → EFI stub boots with no external initrd). The QEMU defconfig
  does **not** embed it — QEMU passes `-initrd rootfs.cpio` explicitly.
- **Rootfs overlay:** `BR2_ROOTFS_OVERLAY` — all dev/installer defconfigs use
  `board/common/rootfs-overlay` + `board/dev/rootfs-overlay`; production uses
  only `common` (no SSH bring-up).
- **Users:** `BR2_ROOTFS_USERS_TABLES` → `board/ally/users-table.txt`
  (creates `playos-game`, uid/gid 1001).
- **Post-build:** only production sets `BR2_ROOTFS_POST_BUILD_SCRIPT` →
  `board/ally/post-build.sh` (fails the build if debug artifacts exist).
- **Images:** QEMU = `ext2` (+ GRUB2 for an EFI disk); Ally/installer =
  `ext2` + `squashfs` + `initramfs`.

---

## 4. br2-external structure

```
br2-external/
├── external.desc            # name/desc (Buildroot picks this up via BR2_EXTERNAL)
├── external.mk              # includes package/*/*.mk
├── Config.in                # "PlayOS packages" menu → package/*/Config.in
├── configs/                 # the five defconfigs above
├── board/
│   ├── common/              # busybox.config, rootfs-overlay/ (shared)
│   ├── dev/                 # rootfs-overlay/ (dev-only: playos-ssh-bringup)
│   ├── ally/                # linux.config, linux-installer.config, users-table.txt, post-build.sh
│   ├── qemu-x86_64/         # linux.config, grub.cfg
│   ├── qemu-installer/      # linux.config
│   └── patches/             # kernel patches
└── package/                 # one dir per PlayOS package: Config.in + <pkg>.mk
```

The nine PlayOS packages: `playos-init`, `playos-platform-api`, `playos-runtime`,
`playos-compositor`, `playos-shell`, `playos-raylib`, `playos-overlay`,
`playos-installer`, `playos-samples`.

**How a package is built** (`package/<pkg>/<pkg>.mk`):

- `SITE = $(BR2_EXTERNAL_PlayOS_PATH)/../src/<pkg>` — source cloned by `make setup`
- `SITE_METHOD = local` — no tarball; rsync'd into the output dir once
- `$(eval $(cmake-package))` — standard Buildroot cmake infrastructure
- Example: `playos-init.mk` installs the binary as `/init` via
  `PLAYOS_INIT_POST_INSTALL_TARGET_HOOKS`.

---

## 5. Stage B — how the disk images are assembled

### 5.1 ROG Ally USB image — `scripts/gen-ally-usb-image.sh`

Input: `output/ally` (needs `make ally-build`).
Output: `output/ally/images/playos-ally-usb.img` (~3.5 GiB sparse GPT).

```
truncate 3.3 GiB sparse file
sgdisk:   1: ESP          256 MiB  FAT32 (EF00)
          2: playos-a    2048 MiB  ext2  (8300, Linux)
          3: playos-data  ~1 GiB   ext4  (8300, fills rest)
losetup --partscan
mkfs.fat -F32 -n ESP        / mkfs.ext2 -L playos-a / mkfs.ext4 -L playos-data
ESP:   cp bzImage → EFI/BOOT/BOOTX64.EFI     ← EFI stub kernel, no GRUB
data:  optionally seed ~/.ssh/id_*.pub → ssh/authorized_keys
```

The kernel is a **bzImage with embedded initramfs** (Ally defconfig), placed
directly as `EFI/BOOT/BOOTX64.EFI`. UEFI boots it directly; `playos-init`
then finds the ESP + `playos-data` and starts the compositor/shell.

### 5.2 Installer USB image — `scripts/gen-installer-usb-image.sh`

Input: `output/installer` **and** `output/ally`.
Output: `output/installer/images/playos-ally-installer.img`.

Same partition layout, but:

- **ESP** carries the **installer kernel** (`playos.mode=install` in its
  `CONFIG_CMDLINE`) as `EFI/BOOT/BOOTX64.EFI`.
- **playos-a** carries the *install payload*:
  - `rootfs.squashfs` from the ally build (the system image to install)
  - `BOOTX64.EFI` from the ally build (the normal production kernel)
- **playos-data** is seeded with the developer SSH key (optional).

At install time `playos-installer` repartitions the target NVMe, then copies
`rootfs.squashfs` + `BOOTX64.EFI` into the target's `playos-a` slot and the
ESP, and seeds `ssh/authorized_keys` into the target `playos-data`.

### 5.3 QEMU — `scripts/qemu-boot-check.sh` and `qemu-pivot-check.sh`

QEMU does **not** use the disk-assembly scripts:

- `qemu-boot-check.sh` boots directly with
  `qemu-system-x86_64 -kernel bzImage -initrd rootfs.cpio` under OVMF,
  attaches a 256 MiB `data.img` (ext4, label `playos-data`) as a virtio disk,
  captures the serial console, and asserts the boot-stage marker appears.
- `qemu-pivot-check.sh` builds a *synthetic installed disk*
  (`output/qemu/images/installed.img` with ESP + `playos-a` + `playos-b` +
  `playos-data`) from the ally `rootfs.squashfs`, then boots it to verify
  **Scenario A** (normal A/B slot pivot into squashfs) and **Scenario B**
  (forced rollback when `boot_count >= 3 && health != good`, by reading
  `boot.json` back from the ESP).

The QEMU defconfig also builds a GRUB2 EFI disk (`output/qemu/images/efi-part`)
with `board/qemu-x86_64/grub.cfg` — available for manual EFI-disk testing,
though the default `make qemu-run` path uses direct `-kernel/-initrd`.

---

## 6. Stage C — flash & update artifacts

- `scripts/flash-usb.sh <image>` — lists removable devices, requires typing
  `YES`, then `dd`s the image. (`make ally-flash` / `installer-flash` just
  print this command.)
- `scripts/create-update-bundle.sh <rootfs.squashfs> <version> <out.playosb>`
  builds a dev-signed A/B update bundle:

```
[4B "PBS1"][4B header_len LE][JSON header][payload = raw squashfs]
[4B sig_len LE][sig_len bytes lowercase-hex HMAC-SHA256]
```

  HMAC key is a hardcoded **development** key; `playos-init` verifies it
  before applying (Sprint 11). `make update-bundle` wires this up.

---

## 7. Host dependencies (for native C builds only)

`scripts/build-host-deps.sh` builds a pinned Wayland/wlroots 0.20 stack into an
isolated prefix (`/opt/playos-deps`), because Ubuntu 24.04's versions are too
old for wlroots 0.20. Activate with `source /opt/playos-deps/env.sh` (or set
`PKG_CONFIG_PATH` as documented in `04-build-and-test.md`). This is **only**
for native `playos-compositor`/`playos-shell` builds; the shipped image is
unaffected (Buildroot provides its own stack from the pinned Buildroot commit).

`scripts/setup-ubuntu.sh` installs the distro-level host packages needed by
Buildroot.

---

## 8. Where artifacts land

```
output/<target>/
├── build/          # Buildroot per-package build dirs (linux-*, playos-*, ...)
├── host/           # host tools
├── target/         # the assembled rootfs (pre-image)
└── images/
    ├── bzImage              # kernel (EFI stub)
    ├── rootfs.cpio          # initramfs (embedded into bzImage on ally targets)
    ├── rootfs.ext2          # ext2 rootfs image
    ├── rootfs.squashfs      # squashfs rootfs (ally/installer only)
    ├── rootfs.tar           # tarball of the rootfs
    ├── playos-ally-usb.img        # ally: Stage B output
    ├── playos-ally-installer.img  # installer: Stage B output
    └── data.img / installed.img   # qemu: runtime/verify disks
```

---

## 9. Gotchas specific to the build system

- **`sudo` required** for the image scripts (`losetup`, `mount`, `mkfs`).
- **Sparse images:** `truncate` + minimal writes → the 3.5 GiB `.img` only
  uses a few hundred MiB of real disk until `dd`ed.
- **`sgdisk` fallback:** if `sgdisk` is missing the scripts fall back to
  `sfdisk` (same layout, different syntax).
- **Late-build edits:** always rebuild via the Makefile targets (they dirclean
  local packages). Calling `make -C buildroot` directly skips that.
- **`versions.lock` is authoritative** and `make setup`/`verify-pins` fail
  loudly on missing pins. Note: the header says *"Update via
  scripts/update-versions.sh"* — **that helper script does not currently
  exist**; pins are edited by hand (commit `versions.lock` after changing).
- **Two kernels in the installer image** — ESP installer kernel vs `playos-a`
  payload kernel; don't mix them up when debugging install failures.
- **`make clean` preserves `dl/`** (the source tarball cache); `distclean`
  removes it and the Buildroot clone.
- **First build is slow** (~40 min) mostly toolchain; subsequent builds are
  incremental per output dir. Output dirs are per-target, so
  `ally-production-build` does not reuse `output/ally` objects.

---

## 10. Quick reference — reproduce an image from scratch

```sh
cd playos-refdistro
make setup                    # once (Buildroot + src/ clones, pins)
make qemu-build && make qemu-run            # QEMU dev image + boot
make ally-build && make ally-usb-image      # Ally dev USB image
make ally-production-build                  # Ally production image
make installer-image                        # installer USB (needs ally-build)
make update-bundle VERSION=0.2.0            # dev-signed .playosb
```

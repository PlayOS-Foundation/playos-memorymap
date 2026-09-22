# 04 — Build & Test

> Last updated: 2026-09-22

## Per-repo builds (host, fast)

All C repos are CMake + C99. Default host builds use `build/`.

| Repo | Configure & build | Test | Notes |
|---|---|---|---|
| `playos-init` | `cmake -B build -DBUILD_TESTS=ON && cmake --build build` | `ctest --test-dir build` | static PID 1; tests: `init_state`, `boot_slot`, `security` |
| `playos-platform-api` | `cmake -B build -DPLAYOS_BACKEND=stub && cmake --build build` | — | backend switch: `stub` / `evdev` |
| `playos-runtime` | `cmake -B build && cmake --build build` | `ctest --test-dir build` | builds `libplayos-trusted`; test: `trusted-control` |
| `playos-compositor` | see note below | `ctest --test-dir build` (no tests currently) | needs wlroots 0.20 |
| `playos-shell` | `cmake -B build && cmake --build build` | — | run with `WAYLAND_DISPLAY=wayland-1` nested |
| `playos-samples` | per-sample `cmake -B build && cmake --build build` | — | each sample builds an executable named `game` |
| `playos-spec` | `mdbook build` | — | renders to `book/` |

### Compositor build on this machine

wlroots 0.20 headers/libs come from `/opt/playos-deps` (not system paths):

```sh
cd playos-compositor
export PKG_CONFIG_PATH=/opt/playos-deps/lib/x86_64-linux-gnu/pkgconfig:/opt/playos-deps/share/pkgconfig
cmake -B build && cmake --build build -j$(nproc)
```

If headers are missing, `find / -name 'wlroots*.pc'` and point `PKG_CONFIG_PATH` at that dir.

## OS image build (slow, via refdistro)

> Deep dive into how images are actually assembled (Stage A Buildroot →
> Stage B image scripts → Stage C run/flash) lives in
> [`10-build-system.md`](10-build-system.md). This section is the quick reference.

All targets live in `playos-refdistro/Makefile`:

```sh
cd playos-refdistro
make setup              # clone Buildroot (pinned by versions.lock), check deps
make qemu-build         # QEMU image  (~40 min first time)
make qemu-run           # boot it in QEMU/OVMF
make qemu-pivot-check   # A/B slot pivot + forced rollback in QEMU

make ally-build               # ROG Ally dev rootfs (with debug tools)
make ally-production-build    # ROG Ally production rootfs (Sprint 12, stripped)
make ally-dev-usb-image       # dev live+installer USB image (SSH; needs ally-build)
make ally-prod-usb-image      # prod live+installer USB image (no SSH)
make ally-flash               # flash dev image to USB (prompts for device)

make intel-build              # Intel PC dev rootfs
make intel-dev-usb-image      # Intel dev live+installer USB image
make update-bundle            # dev-signed .playosb from ally rootfs.squashfs
```

> There are **no `installer-build`/`installer-image` targets** (S13.7 + S14.5): the
> installer is now a PlayOS app the shell drives (Settings → System → Install PlayOS),
> carried inside the consolidated `*-dev-usb-image`/`*-prod-usb-image` images. The
> standalone `playos-installer` remains as the no-shell fallback.

Buildroot outputs go to `output/<target>` (`qemu`, `ally`, `ally-production`, `installer`).
Local PlayOS packages are `dirclean`ed before every build so `src/` edits are always picked up.

## What the host security test verifies (playos-init)

`ctest --test-dir playos-init/build` runs:

- `init_state` — state struct init
- `boot_slot` — boot.json parse, A/B verification, bundle apply (host seam)
- `security` — Sprint 12:
  - Ed25519: RFC 8032 vector + sign/verify roundtrip + tamper rejection
  - Landlock: allowlist honored, default-deny enforced (temp dirs)
  - seccomp: **SKIPs on this build host** (outer sandbox refuses
    `prctl(PR_SET_SECCOMP)` with `EACCES`) — the filter must be exercised
    on-device/in QEMU
  - manifest verify: missing / malformed / invalid signature return codes

## Typical verify workflow for a C change

```sh
cd <repo> && cmake -B build && cmake --build build -j$(nproc) && ctest --test-dir build --output-on-failure
```

Then, if the change is consumed by the OS image, bump `versions.lock` (edit
directly — the `scripts/update-versions.sh` helper referenced in its header
does not exist yet):

```sh
cd playos-refdistro
$EDITOR versions.lock   # set PLAYOS_<COMPONENT>_COMMIT=<sha>
git commit -m "versions.lock: bump <component> to <sha>" versions.lock
```

and finally `make qemu-build` (or `ally-build`) to validate the integrated image.

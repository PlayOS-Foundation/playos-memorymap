# 08 — Gotchas & Lessons Learned

> Last updated: 2026-09-12. These are the things that have already bitten
> someone on this project. Read before debugging.

## Wayland

- **A NULL listener slot aborts the process.** libwayland-client prints
  `listener function for opcode N of <interface> is NULL` and then calls
  `abort()`. It is not a warning. This killed the shell on hardware: we bound
  `zwlr_screencopy_manager_v1` at **version 3** but only filled the version-1
  events in the generated frame listener, so the compositor's `linux_dmabuf`
  offer (opcode 5) hit a NULL slot → SIGABRT → shell dead → empty compositor
  scene ("blue screen"). Two rules: **fill every member of a generated listener
  struct**, and **bind the protocol version you actually implement** (we now
  bind screencopy at v1, where wl_shm is guaranteed and the client copies
  directly from the `buffer` event; v3 requires waiting for `buffer_done`).
- **Honour the `wl_shm` format — it is not always `XRGB8888`.** The screencopy
  `buffer` event names the format the compositor will fill, and the byte order
  follows it: little-endian `XRGB8888`/`ARGB8888` memory is `B,G,R,X`, but
  `XBGR8888`/`ABGR8888` memory is `R,G,B,X`. Assuming `B,G,R,X` unconditionally
  swapped red and blue on hardware (the UI's navy `(20,41,76)` background was
  written as `(76,40,20)`, orange highlights came out blue). Verify colour
  paths by sampling a PNG pixel and comparing with the colour the UI source
  draws, not by eye.
- **Roles must be released on disconnect.** The compositor kept `shell_client`
  set after the client died, so the supervisor's restart was rejected with
  `shell role already taken` and ran untrusted for the whole session. Fixed with
  `wl_client_add_destroy_listener` (`3862e4d`). Any time a trusted role is
  claimed, add a destroy handler for it.
- **Append-only logs hide boot boundaries.** `/data/log/shell-stderr.log`
  survives restarts, so a crash is found via `grep -n "entering main loop"` to
  locate boots and `init.log`'s `shell PID N exited: code=-1 signal=6`
  (signal 6 = SIGABRT) to confirm a crash.

## Input / reserved buttons

- **The Ally's reserved buttons are momentary pulses — no hold gestures.** A raw
  evdev trace (2026-09-12) shows the Command Center button emitting `KEY_F16`
  press **and** release within the same shell poll (~1 ms apart), and Armoury
  Crate emitting `KEY_PROG1` the same way. Physically holding either button
  changes nothing, so `shell_input_button_held()` can never observe them — a
  "hold COMMAND" gesture is impossible on this hardware. Use the edge query
  `shell_input_button_pressed()`, which explicitly catches a press+release that
  lands inside one poll. Contrast: the volume keys on the vendor node
  (`KEY_VOLUMEUP/DOWN`) *do* report sustained presses (288–321 ms measured), so
  the shell can see holds when a device produces them.
- **"button 0x… pressed" log lines come from the query, not the event.** They
  are emitted inside `shell_input_button_pressed()`, so a button that no screen
  queries logs nothing. Absence of a line does not prove the button never fired.
  Log explicitly in the action handler instead (the screenshot path now logs
  `screenshot requested (ARMOURY CRATE|COMMAND)`).
- Reserved vendor keys are opened by the shell as *roles* (`asus`, `vendor`,
  `power`); the raw-debug lines print the role, not the device name.

## Boot / init

- **Kernel config drops dependencies silently.** The Ally kernel once lost its
  entire USB stack because a Kconfig edit dropped deps (fixed `9aed2a4`). When
  editing `linux.config`, re-check the generated `.config` and test USB/NIC in QEMU.
- **ESP can appear late on NVMe.** `playos-init` retries ESP find+mount for late
  partition registration (`f82a887`). Don't assume the ESP exists at first scan.
- **`/data` can mount the wrong partition.** Prefer the removable USB stick over
  the internal NVMe `playos-data` during install/first boot (`974e3ca`,
  `e81aaae`).
- **Rootfs is read-only squashfs.** Persistent state goes under `/data` ONLY.
  Runtime `mkdir` under `/` fails; create bind-mount targets in the overlay
  (e.g. `/root/.ssh` for Dropbear).
- **`/tmp` must be tmpfs** on the squashfs root (`9fb5a49`), or tools that
  mktemp in `/tmp` fail.

## Compositor / graphics

- Compositor builds need wlroots 0.20 from `/opt/playos-deps`:
  `PKG_CONFIG_PATH=/opt/playos-deps/lib/x86_64-linux-gnu/pkgconfig:/opt/playos-deps/share/pkgconfig`.
- wlroots pc files pull `wayland-protocols >= 1.47` — include
  `/opt/playos-deps/share/pkgconfig` in the path or configure fails.
- Black-screen root causes were render loop + initial configure ordering
  (`99309817`). The compositor readiness file is `/run/playos/compositor-ready`.

## Audio

- **ALSA card routing:** Ally card 0 is the GPU's HDMI audio; the internal
  speakers are card 1 (ALC294 → CS35L41). `etc/asound.conf` sets
  `defaults.pcm.card 1` / `defaults.ctl.card 1`. Games need `/dev/snd` in the
  Landlock allowlist and `playos-game` in the `audio` group or audio breaks.
- Shell retries audio init until card 1 registers late (`31bc6ed`).

## Security / sandbox (Sprint 12)

- **Sandbox order is fixed:** `no_new_privs → Landlock → setuid → seccomp → exec`.
  Landlock's `restrict_self` needs `no_new_privs`; seccomp denies `setuid`, so
  the credential drop must happen before the filter installs.
- **Landlock `add_rule` on a missing path fails** (`ENOENT`) — per-game
  saves/cache dirs are pre-created (`0700`, chowned 1001) before rules are added.
  An `allowed_access = 0` rule is rejected (`ENOMSG`); default-deny is the
  mechanism, not explicit deny rules.
- **`/etc/group` overlay replaces the generated file.** Adding a group
  membership means editing `board/common/rootfs-overlay/etc/group`, not just
  the users table.
- **seccomp install can be blocked by an outer sandbox.** On this dev host
  `prctl(PR_SET_SECCOMP)` returns `EACCES`, so `test_security` skips that check
  (exit 30). The filter must be validated in QEMU/on-device.
- **Games are dynamically linked** (musl + shared libraylib/libplayos) — the
  Landlock allowlist must include `/lib` and `/usr/lib` (read+execute) or every
  game exec fails on the interpreter. This is why a full seccomp *allowlist*
  was deferred in favor of a deny-list.
- **Ed25519 verify uses point `pack`, not raw field `pack25519`** for the R
  comparison — mixing them up silently fails verification.

## IPC / distro

- **The component repos are symlinked, not copied.** `playos-refdistro/src/playos-*`
  are symlinks to the sibling repos (`../../playos-init`, etc.), so
  `src/playos-init/ipc/ipc.h` IS `playos-init/ipc/ipc.h` — there is only one
  file. Edit and commit in the component repo; refdistro shows nothing for it
  (`src/playos-*` is gitignored). Exceptions that are real in-repo dirs:
  `src/playos-installer`, `src/playos-overlay`, `src/playos-raylib`.
- **Local packages are only rsync'd on first Buildroot configure.** The
  Makefile `dirclean`s all `playos-*` packages before each build so `src/`
  edits are picked up. Don't bypass that.
- **`versions.lock` pins are consumed by the build.** After merging a component
  change that the image should use, bump the pin
  (edit `versions.lock` directly — no helper script yet).
- Control socket is `0660 root:1000`; peer check accepts gid 1000 **or uid 0**.
  The shell/overlay currently pass via uid 0 (they still run as root).

## Tests

- `test_security` runs Landlock/seccomp in forked children and reports via exit
  codes — don't call `assert()` in those children.
- Host `ctest` for the compositor currently has **no tests**; it only proves
  compilation.
- Spec repo builds with `mdbook build`; a broken link or table fails it.

## Hardware validation is its own gate

QEMU/host passing ≠ done. Sprints 11.5–13.7 are on-device validated on the ROG
Ally and ZenBook. The remaining hardware gate is Sprint 14: the 19-criterion
MVP smoke test, the performance baseline, and SimpleDRM/low-graphics recovery
validation. See [`09-next-steps.md`](09-next-steps.md).
